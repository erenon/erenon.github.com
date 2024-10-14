---
title: "Restore file from editor memory"
date: 2024-10-14
---

I was editing a large text file over [sshfs][] using `nvim`.
Probably due to a lose network cable, during save, the connection broke,
and only half of the data reached the remote computer, the other
half remained in the send buffers of the kernel.

[sshfs]: https://github.com/libfuse/sshfs

The editor was configured to use no swapfiles, therefore it wrote
the target file directly: at this point the second half of the file
was _lost_, and the editor stuck in disk sleep.

The situation did not improve after the network connection was restored
and the share re-mounted (after it was forcefully unmounted...).
The last backup was too old, and I needed the data.

## Solution

Get the PIDs of the currently running editors:

```shell
$ ps -A | grep nvim
<PID>s
```

Let's find which editor instance was writing the file:

```shell
$ ls -l /proc/<PID>/fd/
13 -> path/to/our/file
```

We can check that this editor is indeed stuck, waiting for the
kernel to complete a write operation, that'll never happen:

```shell
cat /proc/<PID>/status | grep State
State: D (disk sleep)
```

In this state, the process cannot be attached to,
`gdb` or `gcore` will hang if tried. Fortunately, [`/proc/<PID>/mem`][ppm]
is still readable. The file is sparse, only offsets mapped
by that process are readable. [`/proc/<PID>/maps`][ppmaps] gives
the valid offsets. A [python script from StackExchange][soa] will save
the readable sections into a file. Slightly improved:

```python
# Original https://unix.stackexchange.com/questions/6301/how-do-i-read-from-proc-pid-mem-under-linux/6302#6302
import re

pid = '123456' # change this
maps_file = open(f"/proc/{pid}/maps", 'r')
mem_file = open(f"/proc/{pid}/mem", 'rb', 0)
output_file = open("self.dump", 'wb')
for line in maps_file.readlines():  # for each mapped region
    m = re.match(r'([0-9A-Fa-f]+)-([0-9A-Fa-f]+) ([-r])', line)
    if m.group(3) == 'r':  # if this is a readable region
        start = int(m.group(1), 16)
        end = int(m.group(2), 16)
        mem_file.seek(start)  # seek to region start
        try:
            chunk = mem_file.read(end - start)  # read region contents
            output_file.write(chunk)  # dump contents to standard output
        except OSError:
            print("failed to read from ", start)
maps_file.close()
mem_file.close()
output_file.close()
```

[ppm]: https://man7.org/linux/man-pages/man5/proc_pid_mem.5.html
[ppmaps]: https://man7.org/linux/man-pages/man5/proc_pid_maps.5.html
[soa]: https://unix.stackexchange.com/questions/6301/how-do-i-read-from-proc-pid-mem-under-linux/6302#6302

Running this script will save the memory of the editor,
that contains the complete file -- in chunks.
Fortunately, the internal structure is line-oriented, and tends to follow the
time of insertions. The lines in each chunk are in reverse order, use `tac` to
fix that:

```shell
strings nvim.dump | tac > nvim.strings
```

## Takeaway

 - `set swapfile`, when editing over sshfs
 - More frequent backups
 - Don't panic.


