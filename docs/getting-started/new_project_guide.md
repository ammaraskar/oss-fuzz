---
layout: default
title: Setting up a new project
parent: Getting started
nav_order: 2
permalink: /getting-started/new-project-guide/
---

# Setting up a new project
{: .no_toc}

- TOC
{:toc}
---

## Prerequisites

Before you can start setting up your new project for fuzzing, you must do the following:

- [Integrate]({{ site.baseurl }}/advanced-topics/ideal-integration/) one or more [fuzz targets]({{ site.baseurl }}/reference/glossary/#fuzz-target)
  with the project you want to fuzz.
  For examples, see
[boringssl](https://github.com/google/boringssl/tree/master/fuzz),
[SQLite](https://www.sqlite.org/src/artifact/ad79e867fb504338),
[s2n](https://github.com/awslabs/s2n/tree/master/tests/fuzz),
[openssl](https://github.com/openssl/openssl/tree/master/fuzz),
[FreeType](http://git.savannah.gnu.org/cgit/freetype/freetype2.git/tree/src/tools/ftfuzzer),
[re2](https://github.com/google/re2/tree/master/re2/fuzzing),
[harfbuzz](https://github.com/behdad/harfbuzz/tree/master/test/fuzzing),
[pcre2](http://vcs.pcre.org/pcre2/code/trunk/src/pcre2_fuzzsupport.c?view=markup), and
[ffmpeg](https://github.com/FFmpeg/FFmpeg/blob/master/tools/target_dec_fuzzer.c).

- [Install Docker](https://docs.docker.com/engine/installation)
  (Googlers can visit [go/installdocker](https://goto.google.com/installdocker)).
  [Why Docker?]({{ site.baseurl }}/faq/#why-do-you-use-docker)

  If you want to run `docker` without `sudo`, you can
  [create a docker group](https://docs.docker.com/engine/installation/linux/ubuntulinux/#/create-a-docker-group).

  **Note:** Docker images can consume significant disk space. Run
  [docker-cleanup](https://gist.github.com/mikea/d23a839cba68778d94e0302e8a2c200f)
  periodically to garbage-collect unused images.

## Creating the file structure

Each OSS-Fuzz project has a subdirectory
inside the [`projects/`](https://github.com/google/oss-fuzz/tree/master/projects) directory in the [OSS-Fuzz repository](https://github.com/google/oss-fuzz). For example, the [boringssl](https://github.com/google/boringssl)
project is located in [`projects/boringssl`](https://github.com/google/oss-fuzz/tree/master/projects/boringssl).

Each project directory also contains the following three configuration files:

* [project.yaml](#project.yaml) - provides metadata about the project.
* [Dockerfile](#Dockerfile) - defines the container environment with information
on dependencies needed to build the project and its [fuzz targets]({{ site.baseurl }}/reference/glossary/#fuzz-target).
* [build.sh](#build.sh) - defines the build script that executes inside the Docker container and
generates the project build.

You can automatically create a new directory for your project in OSS-Fuzz and
generate templated versions of the configuration files
by running the following commands:

```bash
$ cd /path/to/oss-fuzz
$ export PROJECT_NAME=<project_name>
$ python infra/helper.py generate $PROJECT_NAME
```

Once the template configuration files are available, you can modify them to fit your project.

**Note:** We prefer that you keep and maintain [fuzz targets]({{ site.baseurl }}/reference/glossary/#fuzz-target) in your own source code repository. If this isn't possible, you can store them inside the OSS-Fuzz project directory you created.

## project.yaml

This configuration file stores project metadata. The following attributes are supported:

- [homepage](#homepage)
- [primary_contact](#primary)
- [auto_ccs](#primary)
- [sanitizers](#sanitizers) (optional)
- [help_url](#help_url)
- [experimental](#experimental)

### homepage
You project's homepage.

### primary_contact, auto_ccs {#primary}
The primary contact and list of other contacts to be CCed. Each person listed gets access to ClusterFuzz, including crash reports and fuzzer statistics, and are auto-cced on new bugs filed in the OSS-Fuzz
tracker. If you're a primary or a CC, you'll need to use a [Google account](https://support.google.com/accounts/answer/176347?hl=en) to get full access. ([why?]({{ site.baseurl }}/faq/#why-do-you-require-a-google-account-for-authentication)).

### sanitizers (optional) {#sanitizers}
The list of sanitizers to use. If you don't specify a list, `sanitizers` uses a default list of supported
sanitizers (currently ["address"](https://clang.llvm.org/docs/AddressSanitizer.html) and
["undefined"](https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html)).

[MemorySanitizer](https://clang.llvm.org/docs/MemorySanitizer.html) ("memory") is also supported, but is not enabled by default due to the likelihood of false positives.
If you want to use "memory," first make sure your project's runtime dependencies are listed in the OSS-Fuzz
[msan-builder Dockerfile](https://github.com/google/oss-fuzz/blob/master/infra/base-images/msan-builder/Dockerfile#L20).
Then, you can opt in by adding "memory" to your list of sanitizers.

If your project does not build with a particular sanitizer configuration and you need some time to fix
it, you can use `sanitizers` to override the defaults temporarily. For example, to disable the 
UndefinedBehaviourSanitizer build, just specify all supported sanitizers except "undefined".

If you want to test a particular sanitizer to see what crashes it generates without filing
them in the issue tracker, you can set an `experimental` flag. For example, if you want to test "memory", set `experimental: True` like this:

```
sanitizers:
 - address
 - memory:
    experimental: True
 - undefined
 ```

Crashes can be accessed on the [ClusterFuzz
homepage]({{ site.baseurl }}/furthur-reading/clusterfuzz#web-interface).

`sanitizers` example: [boringssl](https://github.com/google/oss-fuzz/blob/master/projects/boringssl/project.yaml).

### help_url
A link to a custom help URL that appears in bug reports instead of the default
[OSS-Fuzz guide to reproducing crashes]({{ site.baseurl }}/advanced-topics/reproducing/). This can be useful if you assign
bugs to members of your project unfamiliar with OSS-Fuzz, or if they should follow a different workflow for
reproducing and fixing bugs than the standard one outlined in the reproducing guide.

`help_url` example: [skia](https://github.com/google/oss-fuzz/blob/master/projects/skia/project.yaml).

### experimental
A boolean (either True or False) that indicates whether this project is in evaluation mode. `experimental` allows you to fuzz your project and generate crash findings, but prevents bugs from being filed in the issue tracker. The crashes can be accessed on the [ClusterFuzz homepage]({{ site.baseurl }}/furthur-reading/clusterfuzz#web-interface). Here's an example:

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

**Note:** You should be only use `experimental` if you are not a maintainer of the project and have
low confidence in the efficacy of your fuzz targets. 

## Dockerfile

This configuration file defines the Docker image for your project. Your [build.sh](#build.sh) script will be executed in inside the container you define.
For most projects, the image is simple:
```docker
FROM gcr.io/oss-fuzz-base/base-builder    # base image with clang toolchain
MAINTAINER YOUR_EMAIL                     # maintainer for this file
RUN apt-get update && apt-get install -y ... # install required packages to build your project
RUN git clone <git_url> <checkout_dir>    # checkout all sources needed to build your project
WORKDIR <checkout_dir>                    # current directory for build script
COPY build.sh fuzzer.cc $SRC/             # copy build script and other fuzzer files in src dir
```
In the above example, the git clone will check out the source to `$SRC/<checkout_dir>`.

For an example in Expat, see [expat/Dockerfile](https://github.com/google/oss-fuzz/tree/master/projects/expat/Dockerfile) 

## build.sh

This file defines how to build binaries for [fuzz targets]({{ site.baseurl }}/reference/glossary/#fuzz-target) in your project.
The script is executed within the image built from your [Dockerfile](#Dockerfile).

In general, this script should do the following:

- Build the project using your build system with the correct compiler.
- Provide compiler flags as [environment variables](#Requirements).
- Build your [fuzz targets]({{ site.baseurl }}/reference/glossary/#fuzz-target) and link your project's build with libFuzzer.
   
Resulting binaries should be placed in `$OUT`.

Here's an example from Expat ([source]((https://github.com/google/oss-fuzz/blob/master/projects/expat/build.sh)):

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
**Notes:**

1. Don't assume the fuzzing engine is libFuzzer by default, because we generate builds for both libFuzzer and AFL fuzzing engine configurations. Instead, link the fuzzing engine using $LIB_FUZZING_ENGINE.
2. Make sure that the binary names for your [fuzz targets]({{ site.baseurl }}/reference/glossary/#fuzz-target) contain only
alphanumeric characters, underscore(_) or dash(-). Otherwise, they won't run on our infrastructure.
3. Don't remove source code files. They are needed for code coverage.

### build.sh script environment

When your build.sh script is executed, the following locations are available within the image:

| Location|Env| Description |
|---------| -------- | ----------  |
| `/out/` | `$OUT`         | Directory to store build artifacts (fuzz targets, dictionaries, options files, seed corpus archives). |
| `/src/` | `$SRC`         | Directory to checkout source files. |
| `/work/`| `$WORK`        | Directory for storing intermediate files. |

Although the files layout is fixed within a container, environment variables are
provided so you can write retargetable scripts.

### build.sh requirements {#Requirements}

Only binaries without an extension are accepted as targets. Extensions are reserved for other artifacts, like .dict.

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

See the [Provided Environment Variables](https://github.com/google/oss-fuzz/blob/master/infra/base-images/base-builder/README.md#provided-environment-variables) section in
`base-builder` image documentation for more details.

## Disk space restrictions

Our builders have a disk size of 70GB (this includes space taken up by the OS). Builds must keep peak disk usage below this.

In addition, please keep the size of the build (everything copied to `$OUT`) small (<10GB uncompressed). The build is repeatedly transferred and unzipped during fuzzing and runs on VMs with limited disk space.

## Fuzzer execution environment

For more on the environment that
your [fuzz targets]({{ site.baseurl }}/reference/glossary/#fuzz-target) run in, and the assumptions you can make, see the [fuzzer environment]({{ site.baseurl }}/furthur-reading/fuzzer-environment/) page.

## Testing locally

You can build your docker image and fuzz targets locally, so you can test them before you push them to the OSS-Fuzz repository. 

1. Run the same helper script you used to create your directory structure, this time using it to build your docker image and [fuzz targets]({{ site.baseurl }}/reference/glossary/#fuzz-target):

    ```bash
    $ cd /path/to/oss-fuzz
    $ python infra/helper.py build_image $PROJECT_NAME
    $ python infra/helper.py build_fuzzers --sanitizer <address/memory/undefined> $PROJECT_NAME
    ```

    The built binaries appear in the `/path/to/oss-fuzz/build/out/$PROJECT_NAME`
    directory on your machine (and `$OUT` in the container).

    **Note:** You *must* run your fuzz target binaries inside the base-runner docker
    container to make sure that they work properly.

2. Find failures to fix by running the `check_build` command: 
    
    ```bash
    $ python infra/helper.py check_build $PROJECT_NAME
    ```

3. If you want to test changes against a particular fuzz target, run the following command:

    ```bash
    $ python infra/helper.py run_fuzzer $PROJECT_NAME <fuzz_target>
    ```

4. We recommend taking a look at your code coverage as a sanity check to make sure that your
fuzz targets get to the code you expect:

    ```bash
    $ python infra/helper.py build_fuzzers --sanitizer coverage $PROJECT_NAME
    $ python infra/helper.py coverage $PROJECT_NAME <fuzz_target>
    ```

**Note:** Currently, we only support AddressSanitizer (address) and UndefinedBehaviorSanitizer (undefined) 
configurations. MemorySanitizer is in development mode and not recommended for use. <b>Make sure to test each
of the supported build configurations with the above commands (build_fuzzers -> run_fuzzer -> coverage).</b>

If everything works locally, it should also work on our automated builders and ClusterFuzz. If you check in
your files and experience failures, review your [dependencies]({{ site.baseurl }}/furthur-reading/fuzzer-environment/#dependencies).

## Debugging Problems

If you run into problems, our [Debugging page]({{ site.baseurl }}/advanced-topics/debugging/) lists ways to debug your build scripts and
[fuzz targets]({{ site.baseurl }}/reference/glossary/#fuzz-target).

## Efficient fuzzing

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

### Custom libFuzzer options for ClusterFuzz

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

## Checking in to the OSS-Fuzz repository

Once you've tested your fuzzing files locally, fork OSS-Fuzz, commit, and push to the fork. Then create a pull request with
your change! Follow the [Forking Project](https://guides.github.com/activities/forking/) guide
if you're new to contributing via GitHub.

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

**Exception:** If you're porting a fuzz target from Chromium, keep the original Chromium license header.

## Reviewing results

Once your change is merged, your project and fuzz targets should be automatically built and run on
ClusterFuzz after a short while (&lt; 1 day). If you think there's a problem, you can check your project's [build status](https://oss-fuzz-build-logs.storage.googleapis.com/index.html).

Use the [ClusterFuzz web interface](https://oss-fuzz.com/) to review the following:
* Crashes generated
* Code coverage statistics
* Fuzzer statistics
* Fuzzer performance analyzer (linked from fuzzer statistics)

**Note:** Your Google Account must be listed in [project.yaml](#projectyaml) for you to have access to the ClusterFuzz web interface.
