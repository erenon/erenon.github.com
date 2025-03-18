---
title: "Multidimensional Make Targets"
date: 2025-03-18
---

This is about describing targets in [GNU Make](https://www.gnu.org/software/make/manual/html_node/) that depend
on many parameters that need to be sampled.

**Example use-case 1**: There's a simulation with many input parameters,
e.g: weather simulation, with current temperature, humidity, date,
duration parameters. You want to run the simulation for some
combinations of these parameters.

**Example use-case 2**: There's a remote database the can be queried.
You want to sample larger batches, and apply a pipeline
of transformations on them, iteratively, as you figure out
what's really needed.

Describing these tasks in a `Makefile` is helpful because:

 - It makes dependencies and data-flow explicit (vs. manual changes and ad-hoc intermediate files)
 - Slow steps can be cached (simulation, database download)
 - `make` can execute independent steps parallel, greatly reducing execution time.

This post is about handling intermediate outputs, that depend on many parameters.
Consider this motivating example:

```make
DATE ?= 2025-03-15
LONGITUDE ?= 19.0527693
LATITUDE ?= 47.4943593
RESOLUTION ?= min

temperature: download_temperature.py
  ./$< --date $(DATE) --longitude $(LONGITUDE) --latitude $(LATIDUTE) --resolution $(RESOLUTION) > $@
```

In this case, by default, `make temperature` downloads the measurements for a pre-defined location and day.
The user can override the default using environment variables, e.g: `DATE=2025-03-16 make temperature`
will download it for the next day. However, there are multiple problems here, including:

 - After a successful download, changing the variables will not prompt `make` to re-execute the download.
   It doesn't know that the target depends on those variables, and it will consider it up-to-date.

 - We can't have multiple measurements (with different parameters) in our tree at the same time,
   e.g: to compare them.

 - If we have a similar "humidity" target, it gets easy to accidentally combine different measurements
   that are otherwise unrelated.

To fix these, we need to make the dependency of the target on the variables explicit.
The key idea is that we need to encode the variables in the target name.
For example, instead of simply having "temperature", we'll have "o/2025-03-15/19.0527693/47.4943593/min/temperature".
This immediately solves all of the above.

But how do we describe this in a Makefile? We can't enumerate all the possible combinations.
Ideally, make [pattern rules](https://www.gnu.org/software/make/manual/html_node/Pattern-Intro.html) could support something like this:

```make
# not actual make syntax
o/%DATE%/%LONGITUDE%/%LATITUDE%/%RESOLUTION%/temperature: download_temperature.py
  ./$< --date $(DATE) --longitude $(LONGITUDE) --latitude $(LATIDUTE) --resolution $(RESOLUTION) > $@
```

but it doesn't:

> A pattern rule contains the character ‘%’ (exactly one of them) in the target.

Instead, we'll emulate this, using [Pattern-specific Variable Values](https://www.gnu.org/software/make/manual/html_node/Pattern_002dspecific.html):

```make
# when requesting "o/2025-03-15/19.0527693/47.4943593/min/temperature",
# split the target name ($@) into a list, and set the variables using the directory names.
o/%: override DATE = $(word 2,$(subst /, ,$@))
o/%: override LONGITUDE = $(word 3,$(subst /, ,$@))
o/%: override LATITUDE = $(word 4,$(subst /, ,$@))
o/%: override RESOLUTION = $(word 5,$(subst /, ,$@))

o/%/temperature: download_temperature.py
  ./$< --date $(DATE) --longitude $(LONGITUDE) --latitude $(LATIDUTE) --resolution $(RESOLUTION) > $@

o/%/humidity: download_humidity.py
  ./$< --date $(DATE) --longitude $(LONGITUDE) --latitude $(LATIDUTE) --resolution $(RESOLUTION) > $@
```

## See Also

 - [Using Landlock to Sandbox GNU Make](https://justine.lol/make/)
 - [Your Makefiles are wrong](https://tech.davis-hansson.com/p/make/)


