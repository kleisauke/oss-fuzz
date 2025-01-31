---
layout: default
title: New project guide
parent: Getting started
nav_order: 2
permalink: /getting-started/new-project-guide/
---

# Setting up a new project

- TOC
{:toc}
---

## Prerequisites
- [Integrate]({{ site.baseurl }}/advanced-topics/ideal-integration/) one or more [Fuzz Targets]({{ site.baseurl }}/reference/glossary/#fuzz-target)
  with the project you want to fuzz.
  Examples:
[boringssl](https://github.com/google/boringssl/tree/master/fuzz),
[SQLite](https://www.sqlite.org/src/artifact/ad79e867fb504338),
[s2n](https://github.com/awslabs/s2n/tree/master/tests/fuzz),
[openssl](https://github.com/openssl/openssl/tree/master/fuzz),
[FreeType](http://git.savannah.gnu.org/cgit/freetype/freetype2.git/tree/src/tools/ftfuzzer),
[re2](https://github.com/google/re2/tree/master/re2/fuzzing),
[harfbuzz](https://github.com/behdad/harfbuzz/tree/master/test/fuzzing),
[pcre2](http://vcs.pcre.org/pcre2/code/trunk/src/pcre2_fuzzsupport.c?view=markup),
[ffmpeg](https://github.com/FFmpeg/FFmpeg/blob/master/tools/target_dec_fuzzer.c).

- Install Docker using the instructions 
  [here](https://docs.docker.com/engine/installation).
  Googlers: [go/installdocker](https://goto.google.com/installdocker).
  [Why Docker?]({{ site.baseurl }}/faq/#why-do-you-use-docker)

  *NOTE: (Optional) If you want to run `docker` without `sudo`, follow the
  [Create a docker group](https://docs.docker.com/engine/installation/linux/ubuntulinux/#/create-a-docker-group) section.*
  *NOTE: Docker images can consume significant disk space. Run*
  *[docker-cleanup](https://gist.github.com/mikea/d23a839cba68778d94e0302e8a2c200f)*
  *periodically to garbage collect unused images.*

## Overview

To add a new OSS project to OSS-Fuzz, you need a project subdirectory
inside the [`projects/`](https://github.com/google/oss-fuzz/tree/master/projects) directory in [OSS-Fuzz repository](https://github.com/google/oss-fuzz).
Example: [boringssl](https://github.com/google/boringssl) project is located in
[`projects/boringssl`](https://github.com/google/oss-fuzz/tree/master/projects/boringssl).

The project directory needs to contain the following three configuration files:

* `projects/<project_name>/project.yaml` - provides metadata about the project.
* `projects/<project_name>/Dockerfile` - defines the container environment with information
on dependencies needed to build the project and its [fuzz targets]({{ site.baseurl }}/reference/glossary/#fuzz-target).
* `projects/<project_name>/build.sh` - build script that executes inside the container and
generates project build.

To *automatically* create a new directory for your project and
generate templated versions of these configuration files,
run the following set of commands:

```bash
$ cd /path/to/oss-fuzz
$ export PROJECT_NAME=<project_name>
$ python infra/helper.py generate $PROJECT_NAME
```

It is preferred to keep and maintain [fuzz targets]({{ site.baseurl }}/reference/glossary/#fuzz-target) in your own source code repository. If this is not possible due to various reasons, you can store them inside the OSS-Fuzz's project directory created above.

## project.yaml

This file stores the metadata about your project. The following attributes are supported:

### homepage
Project's homepage.

### primary_contact, auto_ccs
Primary contact and CCs list. These people get access to ClusterFuzz 
which includes crash reports, fuzzer statistics, etc and are auto-cced on newly filed bugs in OSS-Fuzz
tracker. To get full access to these artifacts, you should use a [Google account](https://support.google.com/accounts/answer/176347?hl=en)
here ([why?]({{ site.baseurl }}/faq/#why-do-you-require-a-google-account-for-authentication)).

### sanitizers (optional)
List of sanitizers to use. By default, it will use the default list of supported
sanitizers (currently -
["address"](https://clang.llvm.org/docs/AddressSanitizer.html),
["undefined"](https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html)).

If your project does not build with a particular sanitizer configuration and you need some time fixing
it, then you can use this option to override the defaults temporarily. E.g. For disabling 
UndefinedBehaviourSanitizer build, then you can just specify all supported sanitizers, except "undefined".

[MemorySanitizer ("memory")](https://clang.llvm.org/docs/MemorySanitizer.html) is also supported, but is not enabled by default due to likelihood of false positives.
For this to work, ensure that your project's runtime dependencies are listed in
[this file](https://github.com/google/oss-fuzz/blob/master/infra/base-images/msan-builder/Dockerfile#L20).
You may opt-in by adding "memory" to this list.

If you want to test a particular sanitizer (e.g. memory) and see what crashes it generates without filing
them in the issue tracker, you can set the experimental flag. The crashes can be accessed on [ClusterFuzz
homepage]({{ site.baseurl }}/furthur-reading/clusterfuzz#web-interface). Example:

```
sanitizers:
 - address
 - memory:
    experimental: True
 - undefined
 ```

Example: [boringssl](https://github.com/google/oss-fuzz/blob/master/projects/boringssl/project.yaml).

### help_url
Link to a custom help URL in bug reports instead of the
[default OSS-Fuzz guide to reproducing crashes]({{ site.baseurl }}/advanced-topics/reproducing/). This can be useful if you assign
bugs to members of your project unfamiliar with OSS-Fuzz or if they should follow a different workflow for
reproducing and fixing bugs than standard one outlined in the reproducing guide.

Example: [skia](https://github.com/google/oss-fuzz/blob/master/projects/skia/project.yaml).

### experimental
A boolean (either True or False) that indicates whether this project is in evaluation mode. This allows a project to be
fuzzed and generate crash findings, but not file them in the issue tracker. The crashes can be accessed on [ClusterFuzz homepage]({{ site.baseurl }}/furthur-reading/clusterfuzz#web-interface). This should be only used if you are not a maintainer of the project and have
less confidence in the efficacy of your fuzz targets. Example:

```
homepage: "{project_homepage}"
primary_contact: "{primary_contact}"
auto_ccs:
  - "{auto_cc_1}"
  - "{auto_cc_2}"
sanitizers:
  - address
  - memory
  - undefined
help_url: "{help_url}"  
experimental: True
```
## Dockerfile

This file defines the Docker image definition. This is where the build.sh script will be executed in.
It is very simple for most projects:
```docker
FROM gcr.io/oss-fuzz-base/base-builder    # base image with clang toolchain
MAINTAINER YOUR_EMAIL                     # maintainer for this file
RUN apt-get update && apt-get install -y ... # install required packages to build your project
RUN git clone <git_url> <checkout_dir>    # checkout all sources needed to build your project
WORKDIR <checkout_dir>                    # current directory for build script
COPY build.sh fuzzer.cc $SRC/             # copy build script and other fuzzer files in src dir
```
Expat example: [expat/Dockerfile](https://github.com/google/oss-fuzz/tree/master/projects/expat/Dockerfile)

In the above example, the git clone will check out the source to `$SRC/<checkout_dir>`. 

## build.sh

This file describes how to build binaries for [fuzz targets]({{ site.baseurl }}/reference/glossary/#fuzz-target) in your project.
The script will be executed within the image built from `Dockerfile`.

In general, this script will need to:

1. Build the project using your build system *with* correct compiler and its flags provided as
  *environment variables* (see below).
2. Build the [fuzz targets]({{ site.baseurl }}/reference/glossary/#fuzz-target), linking your project's build and libFuzzer.
   Resulting binaries should be placed in `$OUT`.

*Note*:

1. Please don't assume that the fuzzing engine is libFuzzer and hardcode in your build scripts.
We generate builds for both libFuzzer and AFL fuzzing engine configurations.
So, link the fuzzing engine using $LIB_FUZZING_ENGINE, see example below.
2. Please make sure that the binary names for your [fuzz targets]({{ site.baseurl }}/reference/glossary/#fuzz-target) contain only
alphanumeric characters, underscore(_) or dash(-). Otherwise, they won't run on our infrastructure.
3. Please don't remove source code files. They are needed for code coverage.

For expat, this looks like [this](https://github.com/google/oss-fuzz/blob/master/projects/expat/build.sh):

```bash
#!/bin/bash -eu

./buildconf.sh
# configure scripts usually use correct environment variables.
./configure

make clean
make -j$(nproc) all

$CXX $CXXFLAGS -std=c++11 -Ilib/ \
    $SRC/parse_fuzzer.cc -o $OUT/parse_fuzzer \
    $LIB_FUZZING_ENGINE .libs/libexpat.a

cp $SRC/*.dict $SRC/*.options $OUT/
```

### build.sh Script Environment

When build.sh script is executed, the following locations are available within the image:

| Location|Env| Description |
|---------| -------- | ----------  |
| `/out/` | `$OUT`         | Directory to store build artifacts (fuzz targets, dictionaries, options files, seed corpus archives). |
| `/src/` | `$SRC`         | Directory to checkout source files |
| `/work/`| `$WORK`        | Directory for storing intermediate files |

While files layout is fixed within a container, the environment variables are
provided to be able to write retargetable scripts.

### Requirements

Only binaries without any extension will be accepted as targets. Extensions are reserved for other artifacts like .dict, etc.

You *must* use the special compiler flags needed to build your project and fuzz targets.
These flags are provided in the following environment variables:

| Env Variable           | Description
| -------------          | --------
| `$CC`, `$CXX`, `$CCC`  | The C and C++ compiler binaries.
| `$CFLAGS`, `$CXXFLAGS` | C and C++ compiler flags.
| `$LIB_FUZZING_ENGINE`  | C++ compiler argument to link fuzz target against the prebuilt engine library (e.g. libFuzzer).

You *must* use `$CXX` as a linker, even if your project is written in pure C.

Most well-crafted build scripts will automatically use these variables. If not,
pass them manually to the build tool.

See [Provided Environment Variables](https://github.com/google/oss-fuzz/blob/master/infra/base-images/base-builder/README.md#provided-environment-variables) section in
`base-builder` image documentation for more details.

## Disk space restrictions

Our builders have a disk size of 70GB (this includes space taken up by the OS). Builds must keep peak disk usage below this.

In addition to this, please keep the size of the build (everything copied to `$OUT`) small (<10GB uncompressed) -- this will need be repeatedly transferred and unzipped during fuzzing and run on VMs with limited disk space.

## Fuzzer execution environment

[This page]({{ site.baseurl }}/furthur-reading/fuzzer-environment/) gives information about the environment that
your [fuzz targets]({{ site.baseurl }}/reference/glossary/#fuzz-target) will run on ClusterFuzz, and the assumptions that you can make.

## Testing locally

Use the helper script to build docker image and [fuzz targets]({{ site.baseurl }}/reference/glossary/#fuzz-target).

```bash
$ cd /path/to/oss-fuzz
$ python infra/helper.py build_image $PROJECT_NAME
$ python infra/helper.py build_fuzzers --sanitizer <address/memory/undefined> $PROJECT_NAME
```

This should place the built binaries into `/path/to/oss-fuzz/build/out/$PROJECT_NAME`
directory on your machine (and `$OUT` in the container).

*Note*: You *must* run these fuzz target binaries inside the base-runner docker
container to make sure that they work properly:

```bash
$ python infra/helper.py check_build $PROJECT_NAME
```

Please fix any failures pointed by the `check_build` command above. To test changes against
a particular fuzz target, run using:

```bash
$ python infra/helper.py run_fuzzer $PROJECT_NAME <fuzz_target>
```

If everything works locally, then it should also work on our automated builders and ClusterFuzz.
If it fails, check out [this]({{ site.baseurl }}/furthur-reading/fuzzer-environment/#dependencies) entry.

It's recommended to look at code coverage as a sanity check to make sure that
[fuzz target]({{ site.baseurl }}/reference/glossary/#fuzz-target) gets to the code you expect.

```bash
$ python infra/helper.py build_fuzzers --sanitizer coverage $PROJECT_NAME
$ python infra/helper.py coverage $PROJECT_NAME <fuzz_target>
```

*Note*: Currently, we only support AddressSanitizer (address) and UndefinedBehaviorSanitizer (undefined) 
configurations. MemorySanitizer is in development mode and not recommended for use. <b>Make sure to test each
of the supported build configurations with the above commands (build_fuzzers -> run_fuzzer -> coverage).</b>

## Debugging Problems

[Debugging]({{ site.baseurl }}/advanced-topics/debugging/) document lists ways to debug your build scripts or
[fuzz targets]({{ site.baseurl }}/reference/glossary/#fuzz-target)
in case you run into problems.


## Custom libFuzzer options for ClusterFuzz

By default, ClusterFuzz will run your fuzzer without any options. You can specify
custom options by creating a `my_fuzzer.options` file next to a `my_fuzzer` executable in `$OUT`:

```
[libfuzzer]
close_fd_mask = 3
only_ascii = 1
```

[List of available options](http://llvm.org/docs/LibFuzzer.html#options). Use of `max_len` is not recommended as other fuzzing engines may not support that option. Instead, if
you need to strictly enforce the input length limit, add a sanity check to the
beginning of your fuzz target:

```cpp
if (size < kMinInputLength || size > kMaxInputLength)
  return 0;
```

For out of tree [fuzz targets]({{ site.baseurl }}/reference/glossary/#fuzz-target), you will likely add options file using docker's
`COPY` directive and will copy it into output in build script.
(example: [woff2](https://github.com/google/oss-fuzz/blob/master/projects/woff2/convert_woff2ttf_fuzzer.options)).


### Seed Corpus

OSS-Fuzz uses evolutionary fuzzing algorithms. Supplying seed corpus consisting
of good sample inputs is one of the best ways to improve [fuzz target]({{ site.baseurl }}/reference/glossary/#fuzz-target)'s coverage.

To provide a corpus for `my_fuzzer`, put `my_fuzzer_seed_corpus.zip` file next
to the [fuzz target]({{ site.baseurl }}/reference/glossary/#fuzz-target)'s binary in `$OUT` during the build. Individual files in this
archive will be used as starting inputs for mutations. The name of each file in the corpus is the sha1 checksum (which you can get using the `sha1sum` or `shasum` comand) of its contents. You can store the corpus
next to source files, generate during build or fetch it using curl or any other
tool of your choice.
(example: [boringssl](https://github.com/google/oss-fuzz/blob/master/projects/boringssl/build.sh#L41)).

Seed corpus files will be used for cross-mutations and portions of them might appear
in bug reports or be used for further security research. It is important that corpus
has an appropriate and consistent license.

See also [Accessing Corpora]({{ site.baseurl }}/advanced-topics/corpora/) for information about getting access to the corpus we are currently using for your fuzz targets.


### Dictionaries

Dictionaries hugely improve fuzzing efficiency for inputs with lots of similar
sequences of bytes. [libFuzzer documentation](http://libfuzzer.info#dictionaries)

Put your dict file in `$OUT`. If the dict filename is the same as your target
binary name (i.e. `%fuzz_target%.dict`), it will be automatically used. If the
name is different (e.g. because it is shared by several targets), specify this
in .options file:

```
[libfuzzer]
dict = dictionary_name.dict
```

It is common for several [fuzz targets]({{ site.baseurl }}/reference/glossary/#fuzz-target)
to reuse the same dictionary if they are fuzzing very similar inputs.
(example: [expat](https://github.com/google/oss-fuzz/blob/master/projects/expat/parse_fuzzer.options)).

## Checking in to OSS-Fuzz repository

Fork OSS-Fuzz, commit and push to the fork, and then create a pull request with
your change! Follow the [Forking Project](https://guides.github.com/activities/forking/) guide
if you are new to contributing via GitHub.

### Copyright headers

Please include copyright headers for all files checked in to oss-fuzz:

```
# Copyright 2019 Google Inc.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#      http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#
################################################################################
```

If you are porting a fuzz target from Chromium, keep the original Chromium license header.

## The end

Once your change is merged, your project and fuzz targets should be automatically built and run on
ClusterFuzz after a short while (&lt; 1 day)!

Check your project's build status [here](https://oss-fuzz-build-logs.storage.googleapis.com/index.html).

Use [ClusterFuzz]({{ site.baseurl }}/furthur-reading/clusterfuzz) web interface [here](https://oss-fuzz.com/) to checkout the following items:
* Crashes generated
* Code coverage statistics
* Fuzzer statistics
* Fuzzer performance analyzer (linked from fuzzer statistics)

Note that your Google Account must be listed in [project.yaml](#projectyaml) for you to have access to the ClusterFuzz web interface.
