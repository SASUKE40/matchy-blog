---
title: Community Bonding Period
excerpt: Birthday of this page! 🎉 Warm-up and greet the team
tags:
  - gsoc
date: 2022-05-29 21:54:00
updated: 2022-05-29 23:22:00
banner_img: /img/gsoc.png
banner_img_height: 30
hide: true
---

{% note danger %}
Occupied with graduation related stuff during the community bonding period. Cannot be very productive.
{% endnote %}

## Week 1

**May 22nd - May 28th**: Community bonding period Week 1

### Create the GSoC page for weekly dev report

I create this page using `hexo new page gsoc` and add customized layout `gsoc.ejs` for presenting both a short introduction and a list of blog posts in the same page.

### Sync and Build OpenMS again

I develop on **Debian 11** and **Ubuntu 20.04** machines. Although I have built OpenMS successfully before, it is good to record the steps here for future reproducibility.

#### Checkout OpenMS

```bash
git clone https://github.com/OpenMS/OpenMS.git
```

#### Install dependencies

OpenMS has some trouble when there are both local and contrib dependencies. Since I have `zlib`, `bzlib` and `svm` installed locally, I decided to install all dependencies to get rid of the need of building `contrib`

```bash
# if universe is not already included, include the ubuntu universe repository and update
# sudo add-apt-repository universe
sudo apt update
sudo apt-get install build-essential cmake autoconf patch libtool git automake
sudo apt-get install qtbase5-dev libqt5svg5-dev libqt5opengl5-dev
sudo apt-get install libeigen3-dev libsqlite3-dev libwildmagic-dev libboost-random1.67-dev \
    libboost-regex1.67-dev libboost-iostreams1.67-dev libboost-date-time1.67-dev libboost-math1.67-dev \
    libxerces-c-dev libglpk-dev zlib1g-dev libsvm-dev libbz2-dev coinor-libcoinmp-dev libhdf5-dev
    # this should eliminate the need for building contrib libraries.
```

{% note info %}
`libboost` probably has different versions than listed about in the source. For **Debian 11**, the version of `libboost` in the source is `1.74`.
{% endnote %}

#### Configuring and building OpenMS/TOPP

Make the build directory.

```bash
mkdir OpenMS-build
cd OpenMS-build
```

Run `cmake` to configure the build process.

```bash
cmake -DBOOST_USE_STATIC=OFF /path/to/OpenMS
```

{% note info %}
`-DBOOST_USE_STATIC=OFF` is necessary for **Debian/Ubuntu** (or similar OSs) since the static versions of the boost library are compiled with different settings on them[^1].
{% endnote %}

Build by running `make`.

```bash
make -j 64
```

#### Test the OpenMS/TOPP installation

Run `ctest` in the build directory

```bash
ctest -j 64
```

{% note warning %}
On servers without GUI, the GUI related tests will always fail.
On **Debian/Ubuntu**, some of the tests always fail due to difference in floating point precision and library implementation (e.g. `libsvm`).
The always-"failed" tests are listed below (GUI tests marked in green; Debian/Ubuntu specific failure marked in red)

```diff
-        495 - PrecursorIonSelectionPreprocessing_test (Failed)
-        575 - DetectabilitySimulation_test (Failed)
-        615 - MRMFeatureSelector_test (Failed)
!        643 - TOPPView_test (Child aborted)
!        644 - TSGDialog_test (Child aborted)
-        2283 - TOPP_OpenPepXL_1_out_2 (Failed)
-        2288 - TOPP_OpenPepXLLF_1_out_2 (Failed)
!        2373 - UTILS_INIUpdater_1 (Child aborted)
!        2374 - UTILS_INIUpdater_1_out (Failed)
!        2376 - UTILS_INIUpdater_3 (Child aborted)
!        2377 - UTILS_INIUpdater_3_out (Failed)
!        2472 - TOPP_ExecutePipeline_1 (Child aborted)
```
{% endnote %}

## To-Dos in the coming weeks in community bonding period

{% cb false %} Resume my attempts on creating BLOSC-encoded mzMLb file (prototype write module) in <a href='https://github.com/matchy233/OpenMS-playground'>matchy233/OpenMS-playground</a>
{% cb false %} Discuss with mentors about how to integrate <a href='https://github.com/Blosc/hdf5-blosc'>Blosc/hdf5-blosc</a> plugin into OpenMS
{% cb false %} Fix deprecation and warnings in <a href='https://github.com/Blosc/hdf5-blosc'>Blosc/hdf5-blosc</a> (Optionally raise a PR to that repo)

## Bibliography

[^1]: [OpenMS documentation > Building OpenMS on GNU/Linux](https://abibuilder.informatik.uni-tuebingen.de/archive/openms/Documentation/nightly/html/install_linux.html)
