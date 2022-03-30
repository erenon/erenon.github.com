---
title: "Static storage shared between compilation units"
date: 2022-03-30
---

Suppose you want to create a static container compilation time,
that can be inserted into by different translation units,
and can be iterated runtime. For example:

```cpp
// a.cpp
void a() { STORE("names", "alice") }
```

```cpp
// b.cpp
void b() { STORE("names", "bob") }
```

```cpp
// c.cpp
void c() { STORE("names", "charlie") }

int main() {
  for (auto name : fetch("names"))
  {
    std::cout << name << ",";
    // prints alice,bob,charlie, in unspecified order
  }
}
```

I know of no way in standard C or C++ that'd allow this.
However, with platform specific tools, it is possible
to implement `STORE` and `fetch` - this post describes how.
The complete source code is available in the [companion repository][static-storage].

# Theory

The following discussion is constrained to the linux, macOS
and windows platforms. Each platform in scope has a mainstream
execution format, that defines how an executable binary should look like.
[ELF][] for linux, [Mach-O][] for macOS, [PE][] for windows.
Each of these formats define headers (metadata) and sections (data).
Sections can contain e.g: program code, exported symbol information debug symbols,
or even arbitrary data. We are going to create and populate a custom section
(`STORE`), and read to contents of that section runtime (`fetch`).

# Put Data into Custom Sections

On each platform, a different construct can be used
to put a given static variable into a specific section:

 - linux/ELF: [section attribute][]:

    ```cpp
    #define STORE(name, str) {               \
      __attribute__((section(name), used))   \
      static constexpr const char s[] = str; \
    }
    ```

   The `used` attribute instructs the compiler not to remove `s`,
   even if it appears unused. The curly braces around the definition
   prevent name conflicts (e.g: if there are multiple `STORE` invocations
   in the same scope).

 - macOS/Mach-O: sections are grouped into segments,
   the segment name must be also specified to the attribute:

    ```cpp
    #define STORE(name, str) {                               \
      __attribute__(("__DATA_CONST," section(name), used))   \
      static constexpr const char s[] = str;                 \
    }
    ```

 - windows/PE: [allocate specifier][]. I couldn't find the
   equivalent of the `used` attribute, so you have to convince
   the compiler otherwise, that the variable is used (e.g: by
   actually using it) - not shown here. Also each section need to be initialized first
   with a [const_seg][] pragma. The pragma must be in global/namespace scope,
   it can't be added to the `STORE` macro.

    ```cpp
    #define STORE(name, str) {               \
      __declspec(allocate(name))             \
      static constexpr const char s[] = str; \
    }

    #pragma const_seg("names") // user code, required once for each segment/translation unit
    ```

This definition makes it possible to use `STORE` inside functions.
Similarly, a `STORE_GLOBAL` macro can be created, that works in global/namespace scope.

One might be tempted to prefix the custom section name with a leading dot (ELF/PE)
or double underscore (Mach-O) to match conventions of the well known sections
(e.g: `.text` or `.rodata`). However, that'd be incorrect: the leading decoration
is there to separate platform specified sections from user defined custom sections.
For example:

> Section names with a dot (.) prefix are reserved for the system, although applications may use these sections if their existing meanings are satisfactory.
> Applications may use names without the prefix to avoid conflicts with system sections.
> <br>[Executable and Linkable Format][elfspec], 1-16

To verify that `STORE` works, create a test binary that invokes it,
and run a platform specific tool that dumps the contents of a specified section
of the specified binary:

  - linux/ELF: [readelf][]: `readelf -x names binary`
  - macOS/Mach-O: [otool][]: `otool -s __DATA_CONST names binary -V`
  - windows/PE: [dumpbin][]: `dumpbin.exe /SECTION:names binary /RAWDATA`

The strings we invoked `STORE` with should appear in the output of these commands.

[ELF]: https://en.wikipedia.org/wiki/Executable_and_Linkable_Format
[Mach-O]: https://en.wikipedia.org/wiki/Mach-O
[PE]: https://en.wikipedia.org/wiki/Portable_Executable

[section attribute]: https://gcc.gnu.org/onlinedocs/gcc/Common-Variable-Attributes.html#index-section-variable-attribute
[allocate specifier]: https://docs.microsoft.com/en-us/cpp/cpp/allocate
[const_seg]: https://docs.microsoft.com/en-us/cpp/preprocessor/const-seg

[elfspec]: https://refspecs.linuxfoundation.org/elf/elf.pdf

[readelf]: https://man7.org/linux/man-pages/man1/readelf.1.html
[otool]: https://www.manpagez.com/man/1/otool/
[dumpbin]: https://docs.microsoft.com/en-us/cpp/build/reference/dumpbin-command-line

# GCC Complications

All trivial so far. Unfortunately, if `STORE` is used in functions
with different linkage (e.g: in a normal function and in an inline or template function),
GCC will produce a section type conflict.
The situation is analyzed in a [stackoverflow question][].

[stackoverflow question]: https://stackoverflow.com/questions/35091862/inline-static-data-causes-a-section-type-conflict

The issue stems from the fact that GCC must give special care to
some function local static variables:

> Function-local static objects in all definitions of the same inline function
> (which may be implicitly inline) all refer to the same object defined in one
> translation unit, as long as the function has external linkage. <br>
> [cppreference](https://en.cppreference.com/w/cpp/language/storage_duration)

To ensure uniqueness, those variables are put into a comdat section,
other variables are not, and GCC is unable to resolve this situation.
To avoid GCC messing up the section type, it has to be specified explicitly.
Unfortunately, it cannot be specified using the section attribute,
and I found no way to convince GCC to forget about the standard requirements,
so inline assembly needs to be used:

```cpp
#define STORE(name, str)                              \
  __asm__ (                                           \
    ".pushsection \"" name "\",\"?\",@progbits" "\n"  \
    ".asciz \"" str "\""                        "\n"  \
    ".popsection"                               "\n"  \
  );
```

The [.pushsection][] and [.asciz][] assembler pseudo directives are used
to put a string value into a specific section, without actually creating
a variable that is restricted by C++ rules. The quoting mess is a bit
scary, but there's nothing too complicated here.

[.pushsection]: https://sourceware.org/binutils/docs/as/PushSection.html
[.asciz]: https://sourceware.org/binutils/docs/as/Asciz.html

# Get data from custom sections

We have a `STORE` macro, that takes two string literals,
and puts one into the section indicated by the other.
Now we only need to extract the data runtime that was inserted compile time.
To do that, we just need to replicate a small portion of what
readelf/otool/dumpbin does. The implementation is platform specific,
let's see the linux/ELF case for illustration:

```cpp
// Get every string STOREd in `name` in the binary at `path`.
std::vector<std::string> fetch(
  const char* name,
  const char* path = current_binary_path())
{
  // Create a random access view of `path` (error handling on read included)
  const MemoryMappedFile f(path);

  // Read the ELF header.
  // ElfW(Ehdr) expands to Efl32_Ehdr or Elf64_Ehdr,
  // according to the current platform.
  ElfW(Ehdr) ehdr;
  f.read(0, sizeof(ehdr), &ehdr);

  // Read the section headers from the location
  // indicated by the ELF header.
  std::vector<ElfW(Shdr)> shdrs(ehdr.e_shnum);
  f.read(ehdr.e_shoff, shdrs.size() * sizeof(ElfW(Shdr)), shdrs.data());

  // The section headers do not contain their name,
  // only an offset into the section header string table.
  // Locate this string table:
  const ElfW(Off) string_table_offset = shdrs.at(ehdr.e_shstrndx).sh_offset;

  std::vector<std::string> result;

  // For each section
  for (const ElfW(Shdr)& shdr : shdrs)
  {
    // Get the section name from the string table
    const std::string_view section_name = f.string(string_table_offset + shdr.sh_name);
    if (section_name == name)
    {
      // This is the section we are looking for.
      // Get all the strings.
      for (ElfW(Xword) offset = 0; offset < shdr.sh_size;)
      {
        const std::string_view str = f.string(shdr.sh_offset + offset);
        result.emplace_back(str);
        offset += str.size() + 1;
      }
      // no break or early return here.
      // section names do not need to be unique.
    }
  }

  return result;
}
```

The full source is available in the [companion repository][static-storage].

[static-storage]: https://github.com/erenon/static-storage

# Use case

This looks cool (?), but why? The [binlog][] high performance log library
uses a more advanced variation of this technique to store the metadata
of log invocations in the binary, avoiding one-time setup for each
invocation that'd be otherwise necessary.

[binlog]: https://github.com/morganstanley/binlog/blob/hiperf/include/binlog/create_source.hpp
