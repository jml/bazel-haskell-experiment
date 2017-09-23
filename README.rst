This has been migrated to `rules_haskell <https://github.com/jml/rules_haskell>`_

========================
Bazel Haskell experiment
========================

I want to build Haskell projects with Bazel_.
This repository contains a set of minimal experiments for doing so.

Goals
=====

To be able to build static executables for arbitrary Haskell programs for both Linux and macOS from my MacBook.

Why static executables?
-----------------------

The programs I want to make with Haskell mostly fall into two categories:

- command-line utilities
- backend services

If I want people to use the former, they have to be able to install them on their workstations.
The easiest way of enabling that is to distribute statically compiled executables –
this has served Go especially well (e.g. see minikube_).
When releasing difftodo_, I lost much time in figuring out how to distribute it to macOS users.

If I want to build the latter, I need to make Linux executables
and, these days, put those into a Docker image.
This means either making sure all of the external libraries are copied into the image
(an approach I pursue in `servant-template`_)
or statically linking the executable.

Why cross-compiling?
--------------------

Primary motivation is being able to do local deployments of Docker images
to a Kubernetes cluster
running on Minikube in a VM on my MacBook.

Why Bazel?
----------

* Fast, reproducible builds
* Thriving ecosystem for other languages, including Go and Rust
* Many architectural similarities to Nix, which I like
* Supports a mono-repo use case
* Committed to supporting cross-compilation, unlike Nix
* If I figure out how to do this in Bazel,
  I can encode that knowledge in build rules
  and then never have to figure it out ever again
* Although the developer community is tiny compared to Nix, it has the benefit of corporate sponsorship

Assumptions
===========

* Build environment is a MacBook Pro running macOS Sierra

Variables
=========

Problem domain
--------------

* "pure" Haskell vs Haskell with C dependencies (e.g. libpcre)
* target is macOS (native) vs Linux (cross-compiling)
* dynamically linked libraries vs statically linked

Solution domain
---------------

* Which Haskell build rules to use? (benley's, fyquah95's, or my own)

Method
======

Use this as a "monorepo", with many Haskell projects, each representing a stage of the experiment.

Estimate: 51 hours

0. Preparation
--------------

Estimate: 0.5 hours

- Create a minimal Haskell executable that depends on a minimal library
- Build it with ghc, documenting commands used

Results
~~~~~~~

This is what ``stack`` says it does:

.. code-block:: console

   $ ghc -XNoImplicitPrelude -XOverloadedStrings -o minimal-executable cmd/Main.hs

At least according to a naive reading of the verbose output, obtained using:

.. code-block:: console

   $ stack ghc -v -- -XNoImplicitPrelude -XOverloadedStrings -o minimal-executable cmd/Main.hs

However, if we run that GHC command outside of ``stack``, we see:

.. code-block:: console

   $ ghc -XNoImplicitPrelude -XOverloadedStrings -o minimal-executable cmd/Main.hs
   [1 of 1] Compiling Main             ( cmd/Main.hs, cmd/Main.o )

   cmd/Main.hs:3:1: error:
       Failed to load interface for ‘Protolude’
       Use -v to see a list of the files searched for.

   cmd/Main.hs:5:1: error:
       Failed to load interface for ‘BazelHaskellExperiment’
       Use -v to see a list of the files searched for.

Which means that ``stack`` is doing some hidden environment set up,
hinted at by the following debug log statements:

.. code-block:: console

   2017-08-24 16:12:47.993542: [debug] Resolving package entries
   @(Stack/Setup.hs:252:5)
   2017-08-24 16:12:48.002986: [debug] Starting to execute command inside EnvConfig
   @(Stack/Runners.hs:163:18)


Conclusions
~~~~~~~~~~~

I had to specify ``NoImplicitPrelude`` and ``OverloadedStrings`` on command-line,
and thus would have to in Bazel files as well.

Is it reasonable to insist that Haskell projects that use Bazel only use file-level pragma?


1. Pure, native, dynamic
------------------------

Estimate: 1.5 hours

- Build it with Bazel using fyquah95's build rules
- Build it with Bazel using benley's build rules
- Set up some way to easily toggle between them

Notes
~~~~~

This is taking far longer than expected, since neither of the pre-existing
Haskell rules files support libraries.

That is, they can build ``*.o`` and ``*.hi`` files, but not ``libHSfoo.a``
files.

I am now exploring how Bazel does this for C++. This is harder than expected
because the C++ rules are written in Java and are part of core Bazel.

Question that I'm trying to solve now is: how does the ``cc_library`` rule go
about compiling individual ``*.cpp`` files.

I'm using ``cpp-tutorial`` in ``github.com/bazel/examples`` as a starting
point.

.. code-block:: console

   $ bazel build --show_task_finish --subcommands //main:hello-world

Seems to do the trick.

.. code-block:: console

   gcc3 -Wno-builtin-macro-redefined '-D__DATE__="redacted"' '-D__TIMESTAMP__="redacted"' '-D__TIME__="redacted"' -c lib/hello-time.cc -o bazel-out/local-fastbuild/bin/lib/_objs/hello-time/lib/hello-time.pic.o
   external/local_config_cc/cc_wrapper.sh -U_FORTIFY_SOURCE -fstack-protector -Wall -Wthread-safety -Wself-assign -fcolor-diagnostics -fno-omit-frame-pointer '-std=c++0x' -MD -MF bazel-out/local-fastbuild/bin/main/_objs/hello-greet/main/hello-greet.pic.d '-frandom-seed=bazel-out/local-fastbuild/bin/main/_objs/hello-greet/main/hello-greet.pic.o' -fPIC -iquote . -iquote bazel-out/local-fastbuild/genfiles -iquote external/bazel_tools -iquote bazel-out/local-fastbuild/genfiles/external/bazel_tools -isystem external/bazel_tools/tools/cpp/gcc3 -Wno-builtin-macro-redefined '-D__DATE__="redacted"' '-D__TIMESTAMP__="redacted"' '-D__TIME__="redacted"' -c main/hello-greet.cc -o bazel-out/local-fastbuild/bin/main/_objs/hello-greet/main/hello-greet.pic.o
   /usr/bin/libtool -static -s -o bazel-out/local-fastbuild/bin/main/libhello-greet.a bazel-out/local-fastbuild/bin/main/_objs/hello-greet/main/hello-greet.pic.o

``_objs`` looks like something to watch for.

It's defined as a constant, ``OBJS`` in `CppHelper``.

Used in:

  - ``CppHelper.getObjDirectory``
    - ``CppHelper.getCompileOutputArtifact``
    - ``CppHelper.getCompileOutputTreeArtifact``
    - ``CppModel`` (ignoring this for now)
    - ``CppCompileActionBuilder.setOutputs``
  - ``CppLinkActionBuilder.mapLinkstampsToOutputs``
    - which is used in ``CppLinkActionBuilder.build``, which may be what we're looking for

Following the thread elsewhere, we see the following 'interesting' bits of code:

 - ``CcLibraryHelper.build``, which creates "the C++ compile and link actions"
 - ``CppModel.createCcCompileActions``, which "constructs the C++ compiler actions"
   (called by ``CcLibraryHelper``)


2. Pure, native, static
-----------------------

Estimate: 3 hours

- Try to statically link the minimal executable using ghc
- Encode that effort into Bazel rules, somehow
- Build statically with Bazel

3. C dependencies, native, dynamic
----------------------------------

Estimate: 2 hours

- Extend the example to depend on a Haskell library that depends on a C library
  (highlighter2 or cryptonite, perhaps)
- Build it with GHC
- Build it with Bazel

4. C dependencies, native, static
---------------------------------

Estimate: 3 hours

- Statically link that using GHC
  (this will probably require static versions of the dependent libraries)
- Encode that into Bazel rules
- Statically link with Bazel

5. Publish
----------

Estimate: 4 hours

If we get to this point, we'll have something interesting to other people.
It's unclear exactly how best to communicate, but some options are:

- Update `compare-revisions`_ CI process to use Bazel
- Write and publish a blog post, focusing on results
- Update `servant-template`_ to use Bazel (possibly controversial),
  or at least whatever static linking techniques we discover
- Post to /r/haskell
- Tweet to @bazelbuild about it

6. Explore cross compiling
--------------------------

Estimate: 6 hours

- Follow the official GHC instructions to set up a cross-compiling GHC for macOS to Linux
- Use that GHC to cross-compile minimal binary
- Try to use the LLVM backend with a normal GHC to target linux amd64 from macOS
- Try Go cross compilation (perhaps on Cortex_?)
- Read up on how Go cross compilation works
- Update stack & ghc bugs with details

7. Pure, cross-compiled, dynamic
--------------------------------

Estimate: 4 hours

- Compile a dynamic Linux executable from my MacBook using Bazel
- Run it in a Docker image

8. Pure, cross-compiled, static
-------------------------------

Estimate: 4 hours

- Compile a static Linux executable from my MacBook using Bazel
- Compile it into a Docker image
  (technically out of scope, but generally useful, somewhat related, and hopefully not too hard)

9. C dependencies, cross-compiled, dynamic
------------------------------------------

Estimate: 4 hours

- Take the existing minimal example with C dependencies and compile it for Linux using Bazel

10. C dependencies, cross-compiled, static
------------------------------------------

Estimate: 4 hours

- Take the existing minimal example with C dependencies and compile it for Linux using Bazel
  making sure the resulting executable is statically linked

11. Review
----------

Estimate: 3 hours

* Can we factor out what we've learned into clean, re-usable Bazel rules?
* How would someone who had never used Bazel begin to use such a system?

12. Publish
-----------

Estimate: 4 hours

Again, details are unclear, but options include:

- Update `compare-revisions`_ core Makefile to use Bazel
- Write and publish a results-oriented blog post
- Write and publish a process-oriented blog post
- Update `servant-template`_
- Post to /r/haskell
- Post to Bazel mailing list

13. Profit
----------

Estimate: 8 hours

- Write rules for running Haskell tests
- Write rules for running Haskell benchmarks
- Migrate all my projects to bazel

  - difftodo (and then, release!)
  - holborn
  - graphql-api
  - haskell-spake2

Prior art
=========

There are two sets of published build rules for Haskell that I can find

* https://github.com/benley/bazel_rules_haskell
* https://github.com/fyquah95/haskell.bzl

Both are about the same age, have about the same activity, and have roughly equivalent documentation.

Questions
=========

* How does one best get a set of build rules into the official bazelbuild GitHub organization? What does this entail?
* Assuming that this results in me creating or contributing significantly to Bazel build rules for Haskell,
  how can I get others to maintain it? I realistically will not have much spare time to do so.
* Can cross-compiling be made easier by using LLVM somehow?
* Are there guidelines / best practices for writing Bazel rules for a language?
* Should build rules operate at cabal level or at GHC level?
  * Suspect GHC level is "cleaner" but more work, as it would end up re-implementing cabal

Future ideas
============

* An equivalent of gazelle_ that can automatically generate build rules, perhaps based on cabal or hpack files?
* A tool to one-off generate BUILD files based

Notes
=====

Stack appears to be a glorified cabal wrapper. This is what it runs on ``stack build --fast``

.. code-block:: console

   $ /Users/jml/.stack/setup-exe-cache/x86_64-osx/Cabal-simple_mPHDZzAJ_1.24.2.0_ghc-8.0.2 \
                --builddir=.stack-work/dist/x86_64-osx/Cabal-1.24.2.0 configure \
                --with-ghc=/Users/jml/.stack/programs/x86_64-osx/ghc-8.0.2/bin/ghc \
                --with-ghc-pkg=/Users/jml/.stack/programs/x86_64-osx/ghc-8.0.2/bin/ghc-pkg \
                --user \
                --package-db=clear \
                --package-db=global \
                --package-db=/Users/jml/.stack/snapshots/x86_64-osx/lts-9.0/8.0.2/pkgdb \
                --package-db=/Users/jml/src/bazel-haskell-experiment/.stack-work/install/x86_64-osx/lts-9.0/8.0.2/pkgdb \
                --libdir=/Users/jml/src/bazel-haskell-experiment/.stack-work/install/x86_64-osx/lts-9.0/8.0.2/lib \
                --bindir=/Users/jml/src/bazel-haskell-experiment/.stack-work/install/x86_64-osx/lts-9.0/8.0.2/bin \
                --datadir=/Users/jml/src/bazel-haskell-experiment/.stack-work/install/x86_64-osx/lts-9.0/8.0.2/share \
                --libexecdir=/Users/jml/src/bazel-haskell-experiment/.stack-work/install/x86_64-osx/lts-9.0/8.0.2/libexec \
                --sysconfdir=/Users/jml/src/bazel-haskell-experiment/.stack-work/install/x86_64-osx/lts-9.0/8.0.2/etc \
                --docdir=/Users/jml/src/bazel-haskell-experiment/.stack-work/install/x86_64-osx/lts-9.0/8.0.2/doc/bazel-haskell-experiment-0.0.1 \
                --htmldir=/Users/jml/src/bazel-haskell-experiment/.stack-work/install/x86_64-osx/lts-9.0/8.0.2/doc/bazel-haskell-experiment-0.0.1 \
                --haddockdir=/Users/jml/src/bazel-haskell-experiment/.stack-work/install/x86_64-osx/lts-9.0/8.0.2/doc/bazel-haskell-experiment-0.0.1 \
                --dependency=base=base-4.9.1.0 \
                --dependency=protolude=protolude-0.1.10-EbWghKT4Ra36YSCOzDFDKT \
                --ghc-options -O0 \
                --enable-tests \
                --enable-benchmarks
   $ /Users/jml/.stack/setup-exe-cache/x86_64-osx/Cabal-simple_mPHDZzAJ_1.24.2.0_ghc-8.0.2 \
                --builddir=.stack-work/dist/x86_64-osx/Cabal-1.24.2.0 build \
                lib:bazel-haskell-experiment \
                exe:minimal-executable \
                --ghc-options " -ddump-hi -ddump-to-file"


.. code-block::

   /Users/jml/.stack/programs/x86_64-osx/ghc-8.0.1/bin/ghc \
   --make \
   -fbuilding-cabal-package \
   -O \
   -static \
   -dynamic-too \
   -dynosuf dyn_o \
   -dynhisuf dyn_hi \
   -outputdir .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build \
   -odir .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build \
   -hidir .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build \
   -stubdir .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build \
   -i -i.stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build \
   -isrc -i.stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/autogen \
   -I.stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/autogen \
   -I.stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build \
   -optP-include \
   -optP.stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/autogen/cabal_macros.h \
   -this-unit-id protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW \
   -hide-all-packages \
   -no-user-package-db \
   -package-db /Users/jml/.stack/snapshots/x86_64-osx/lts-7.14/8.0.1/pkgdb \
   -package-db /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/pkgdb \
   -package-db .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/package.conf.inplace \
   -package-id array-0.5.1.1 \
   -package-id async-2.1.1-xFiBzw9xoB8HPZAuxUY2o \
   -package-id base-4.9.0.0 \
   -package-id bytestring-0.10.8.1 \
   -package-id containers-0.5.7.1 \
   -package-id deepseq-1.4.2.0 \
   -package-id ghc-prim-0.5.0.0 \
   -package-id hashable-1.2.4.0-EMu4H7FB10MAl6hwKw992G \
   -package-id mtl-2.2.1-6qsR1PHUy5lL47Hpoa4jCM \
   -package-id mtl-compat-0.2.1.3-CtLbq6xU5RAmphYnnSjKT \
   -package-id safe-0.3.10-1VyrsjWhmjvGnGud5lgW7f \
   -package-id stm-2.4.4.1-4z2NRWnB0NIIUvSJsHW0kF \
   -package-id text-1.2.2.1-5QpmrLQApEZ4Ly9nMHWY0s \
   -package-id transformers-0.5.2.0 \
   -XHaskell2010 \
   -XNoImplicitPrelude \
   -XOverloadedStrings \
   -XFlexibleContexts \
   -XMultiParamTypeClasses \
   Protolude \
   Unsafe \
   Debug \
   Protolude.Exceptions \
   Protolude.Base \
   Protolude.Applicative \
   Protolude.Bool \
   Protolude.List \
   Protolude.Monad \
   Protolude.Show \
   Protolude.Conv \
   Protolude.Either \
   Protolude.Functor \
   Protolude.Semiring \
   Protolude.Bifunctor \
   Protolude.CallStack \
   Protolude.Error \
   Protolude.Panic \
   -Wall \
   -fwarn-implicit-prelude \
   -O0 \
   -ddump-hi \
   -ddump-to-file

   /usr/bin/ar -r -s \
     .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/objs-3164/libHSprotolude-0.2-6HCoCsk9tmDFZ20MEUtzMW.a \
     .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude.o \
     .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Unsafe.o \
     .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Debug.o \
     .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Exceptions.o \
     .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Base.o \
     .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Applicative.o \
     .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Bool.o \
     .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/List.o \
     .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Monad.o \
     .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Show.o \
     .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Conv.o \
     .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Either.o \
     .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Functor.o \
     .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Semiring.o \
     .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Bifunctor.o \
     .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/CallStack.o \
     .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Error.o \
     .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Panic.o

.. code-block:: console

   $ stack build --fast -v --cabal-verbose
   Version 1.5.1, Git revision 600c1f01435a10d127938709556c1682ecfd694e (4861 commits) x86_64 hpack-0.17.1
   [debug] Checking for project config at: /Users/jml/src/protolude/stack.yaml
   [debug] Loading project config file stack.yaml
   [debug] Trying to decode /Users/jml/.stack/build-plan-cache/x86_64-osx/lts-7.14.cache
   [debug] Success decoding /Users/jml/.stack/build-plan-cache/x86_64-osx/lts-7.14.cache
   [debug] Using standard GHC build
   [debug] Asking GHC for its version
   [debug] Getting Cabal package version
   [debug] Run process: /Users/jml/.stack/programs/x86_64-osx/ghc-8.0.1/bin/ghc --numeric-version
   [debug] Getting global package database location
   [debug] Run process: /Users/jml/.stack/programs/x86_64-osx/ghc-8.0.1/bin/ghc-pkg --no-user-package-db field --simple-output Cabal version
   [debug] Run process: /Users/jml/.stack/programs/x86_64-osx/ghc-8.0.1/bin/ghc-pkg --no-user-package-db list --global
   [debug] Process finished in 57ms: /Users/jml/.stack/programs/x86_64-osx/ghc-8.0.1/bin/ghc-pkg --no-user-package-db list --global
   [debug] Process finished in 62ms: /Users/jml/.stack/programs/x86_64-osx/ghc-8.0.1/bin/ghc-pkg --no-user-package-db field --simple-output Cabal version
   [debug] Process finished in 92ms: /Users/jml/.stack/programs/x86_64-osx/ghc-8.0.1/bin/ghc --numeric-version
   [debug] Resolving package entries
   [debug] Starting to execute command inside EnvConfig
   [debug] Parsing the cabal files of the local packages
   [debug] Parsing the targets
   [debug] Exception ignored when attempting to load /Users/jml/src/protolude/.stack-work/dist/x86_64-osx/Cabal-1.24.0.0/stack-build-cache: /Users/jml/src/protolude/.stack-work/dist/x86_64-osx/Cabal-1.24.0.0/stack-build-cache: openBinaryFile: does not exist (No such file or directory)
   [debug] Start: getPackageFiles /Users/jml/src/protolude/protolude.cabal
   [debug] Finished in 8ms: getPackageFiles /Users/jml/src/protolude/protolude.cabal
   [debug] Finding out which packages are already installed
   [debug] Run process: /Users/jml/.stack/programs/x86_64-osx/ghc-8.0.1/bin/ghc-pkg --global --no-user-package-db dump --expand-pkgroot
   [debug] Process finished in 61ms: /Users/jml/.stack/programs/x86_64-osx/ghc-8.0.1/bin/ghc-pkg --global --no-user-package-db dump --expand-pkgroot
   [debug] Ignoring package Cabal due to wanting version 1.24.2.0 instead of 1.24.0.0
   [debug] Run process: /Users/jml/.stack/programs/x86_64-osx/ghc-8.0.1/bin/ghc-pkg --user --no-user-package-db --package-db /Users/jml/.stack/snapshots/x86_64-osx/lts-7.14/8.0.1/pkgdb dump --expand-pkgroot
   [debug] Process finished in 119ms: /Users/jml/.stack/programs/x86_64-osx/ghc-8.0.1/bin/ghc-pkg --user --no-user-package-db --package-db /Users/jml/.stack/snapshots/x86_64-osx/lts-7.14/8.0.1/pkgdb dump --expand-pkgroot
   [debug] Ignoring package purescript, from (InstalledTo Snap,"/Users/jml/.stack/snapshots/x86_64-osx/lts-7.14/8.0.1/pkgdb/"), due to wrong location: (Just (InstalledTo Snap),Local)
   [debug] Ignoring package protolude, from (InstalledTo Snap,"/Users/jml/.stack/snapshots/x86_64-osx/lts-7.14/8.0.1/pkgdb/"), due to wrong location: (Just (InstalledTo Snap),Local)
   [debug] Run process: /Users/jml/.stack/programs/x86_64-osx/ghc-8.0.1/bin/ghc-pkg --user --no-user-package-db --package-db /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/pkgdb dump --expand-pkgroot
   [debug] Process finished in 29ms: /Users/jml/.stack/programs/x86_64-osx/ghc-8.0.1/bin/ghc-pkg --user --no-user-package-db --package-db /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/pkgdb dump --expand-pkgroot
   [debug] Constructing the build plan
   [debug] Checking if we are going to build multiple executables with the same name
   [debug] Executing the build plan
   [debug] Getting global package database location
   [debug] Run process: /Users/jml/.stack/programs/x86_64-osx/ghc-8.0.1/bin/ghc-pkg --no-user-package-db list --global
   [debug] Process finished in 30ms: /Users/jml/.stack/programs/x86_64-osx/ghc-8.0.1/bin/ghc-pkg --no-user-package-db list --global
   [info] protolude-0.2: unregistering (local file changes: protolude.cabal src/Debug.hs src/Protolude.hs src/Protolude/Applicative.hs src/Protolude/Base.hs ...)
   [debug] Run process: /Users/jml/.stack/programs/x86_64-osx/ghc-8.0.1/bin/ghc-pkg --no-user-package-db --package-db=/Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/pkgdb/ unregister --user --force --ipid protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW
   [debug] Process finished in 34ms: /Users/jml/.stack/programs/x86_64-osx/ghc-8.0.1/bin/ghc-pkg --no-user-package-db --package-db=/Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/pkgdb/ unregister --user --force --ipid protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW
   [debug] Exception ignored when attempting to load /Users/jml/src/protolude/.stack-work/dist/x86_64-osx/Cabal-1.24.0.0/stack-config-cache: /Users/jml/src/protolude/.stack-work/dist/x86_64-osx/Cabal-1.24.0.0/stack-config-cache: openBinaryFile: does not exist (No such file or directory)
   [debug] Exception ignored when attempting to load /Users/jml/src/protolude/.stack-work/dist/x86_64-osx/Cabal-1.24.0.0/stack-cabal-mod: /Users/jml/src/protolude/.stack-work/dist/x86_64-osx/Cabal-1.24.0.0/stack-cabal-mod: openBinaryFile: does not exist (No such file or directory)
   [info] protolude-0.2: configure (lib)
   [debug] Run process: /Users/jml/.stack/setup-exe-cache/x86_64-osx/Cabal-simple_mPHDZzAJ_1.24.0.0_ghc-8.0.1 --verbose --builddir=.stack-work/dist/x86_64-osx/Cabal-1.24.0.0 configure --with-ghc=/Users/jml/.stack/programs/x86_64-osx/ghc-8.0.1/bin/ghc --with-ghc-pkg=/Users/jml/.stack/programs/x86_64-osx/ghc-8.0.1/bin/ghc-pkg --user --package-db=clear --package-db=global --package-db=/Users/jml/.stack/snapshots/x86_64-osx/lts-7.14/8.0.1/pkgdb --package-db=/Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/pkgdb --libdir=/Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/lib --bindir=/Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/bin --datadir=/Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/share --libexecdir=/Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/libexec --sysconfdir=/Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/etc --docdir=/Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/doc/protolude-0.2 --htmldir=/Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/doc/protolude-0.2 --haddockdir=/Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/doc/protolude-0.2 --dependency=array=array-0.5.1.1 --dependency=async=async-2.1.1-xFiBzw9xoB8HPZAuxUY2o --dependency=base=base-4.9.0.0 --dependency=bytestring=bytestring-0.10.8.1 --dependency=containers=containers-0.5.7.1 --dependency=deepseq=deepseq-1.4.2.0 --dependency=ghc-prim=ghc-prim-0.5.0.0 --dependency=hashable=hashable-1.2.4.0-EMu4H7FB10MAl6hwKw992G --dependency=mtl=mtl-2.2.1-6qsR1PHUy5lL47Hpoa4jCM --dependency=mtl-compat=mtl-compat-0.2.1.3-CtLbq6xU5RAmphYnnSjKT --dependency=safe=safe-0.3.10-1VyrsjWhmjvGnGud5lgW7f --dependency=stm=stm-2.4.4.1-4z2NRWnB0NIIUvSJsHW0kF --dependency=text=text-1.2.2.1-5QpmrLQApEZ4Ly9nMHWY0s --dependency=transformers=transformers-0.5.2.0 --ghc-options -O0 --enable-tests --enable-benchmarks
   [info] Configuring protolude-0.2...
   [info] Dependency array ==0.5.1.1: using array-0.5.1.1
   [info] Dependency async ==2.1.1: using async-2.1.1
   [info] Dependency base ==4.9.0.0: using base-4.9.0.0
   [info] Dependency bytestring ==0.10.8.1: using bytestring-0.10.8.1
   [info] Dependency containers ==0.5.7.1: using containers-0.5.7.1
   [info] Dependency deepseq ==1.4.2.0: using deepseq-1.4.2.0
   [info] Dependency ghc-prim ==0.5.0.0: using ghc-prim-0.5.0.0
   [info] Dependency hashable ==1.2.4.0: using hashable-1.2.4.0
   [info] Dependency mtl ==2.2.1: using mtl-2.2.1
   [info] Dependency mtl-compat ==0.2.1.3: using mtl-compat-0.2.1.3
   [info] Dependency safe ==0.3.10: using safe-0.3.10
   [info] Dependency stm ==2.4.4.1: using stm-2.4.4.1
   [info] Dependency text ==1.2.2.1: using text-1.2.2.1
   [info] Dependency transformers ==0.5.2.0: using transformers-0.5.2.0
   [info] Using Cabal-1.24.0.0 compiled by ghc-8.0
   [info] Using compiler: ghc-8.0.1
   [info] Using install prefix: /Users/jml/.cabal
   [info] Binaries installed in:
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/bin
   [info] Libraries installed in:
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/lib/x86_64-osx-ghc-8.0.1/protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW
   [info] Private binaries installed in:
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/libexec
   [info] Data files installed in:
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/share/x86_64-osx-ghc-8.0.1/protolude-0.2
   [info] Documentation installed in:
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/doc/protolude-0.2
   [info] Configuration files installed in:
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/etc
   [info] Using alex version 3.1.7 found on system at:
   [info] /Users/jml/.stack/snapshots/x86_64-osx/lts-7.14/8.0.1/bin/alex
   [info] Using ar found on system at: /usr/bin/ar
   [info] No c2hs found
   [info] Using cpphs version 1.20.2 found on system at:
   [info] /Users/jml/.stack/snapshots/x86_64-osx/lts-7.14/8.0.1/bin/cpphs
   [info] Using gcc version 4.2.1 found on system at: /usr/bin/gcc
   [info] Using ghc version 8.0.1 given by user at:
   [info] /Users/jml/.stack/programs/x86_64-osx/ghc-8.0.1/bin/ghc
   [info] Using ghc-pkg version 8.0.1 given by user at:
   [info] /Users/jml/.stack/programs/x86_64-osx/ghc-8.0.1/bin/ghc-pkg
   [info] No ghcjs found
   [info] No ghcjs-pkg found
   [info] No greencard found
   [info] Using haddock version 2.17.2 found on system at:
   [info] /Users/jml/.stack/programs/x86_64-osx/ghc-8.0.1/bin/haddock
   [info] Using happy version 1.19.5 found on system at:
   [info] /Users/jml/.stack/snapshots/x86_64-osx/lts-7.14/8.0.1/bin/happy
   [info] Using haskell-suite found on system at: haskell-suite-dummy-location
   [info] Using haskell-suite-pkg found on system at: haskell-suite-pkg-dummy-location
   [info] No hmake found
   [info] Using hpc version 0.67 found on system at:
   [info] /Users/jml/.stack/programs/x86_64-osx/ghc-8.0.1/bin/hpc
   [info] Using hsc2hs version 0.68 found on system at:
   [info] /Users/jml/.stack/programs/x86_64-osx/ghc-8.0.1/bin/hsc2hs
   [info] Using hscolour version 1.24 found on system at:
   [info] /Users/jml/.stack/snapshots/x86_64-osx/lts-7.14/8.0.1/bin/HsColour
   [info] No jhc found
   [info] Using ld found on system at: /usr/bin/ld
   [info] No lhc found
   [info] No lhc-pkg found
   [info] Using pkg-config version 0.29.1 found on system at: /usr/local/bin/pkg-config
   [info] Using strip found on system at: /usr/bin/strip
   [info] Using tar found on system at: /usr/bin/tar
   [info] No uhc found
   [debug] Process finished in 1124ms: /Users/jml/.stack/setup-exe-cache/x86_64-osx/Cabal-simple_mPHDZzAJ_1.24.0.0_ghc-8.0.1 --verbose --builddir=.stack-work/dist/x86_64-osx/Cabal-1.24.0.0 configure --with-ghc=/Users/jml/.stack/programs/x86_64-osx/ghc-8.0.1/bin/ghc --with-ghc-pkg=/Users/jml/.stack/programs/x86_64-osx/ghc-8.0.1/bin/ghc-pkg --user --package-db=clear --package-db=global --package-db=/Users/jml/.stack/snapshots/x86_64-osx/lts-7.14/8.0.1/pkgdb --package-db=/Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/pkgdb --libdir=/Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/lib --bindir=/Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/bin --datadir=/Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/share --libexecdir=/Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/libexec --sysconfdir=/Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/etc --docdir=/Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/doc/protolude-0.2 --htmldir=/Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/doc/protolude-0.2 --haddockdir=/Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/doc/protolude-0.2 --dependency=array=array-0.5.1.1 --dependency=async=async-2.1.1-xFiBzw9xoB8HPZAuxUY2o --dependency=base=base-4.9.0.0 --dependency=bytestring=bytestring-0.10.8.1 --dependency=containers=containers-0.5.7.1 --dependency=deepseq=deepseq-1.4.2.0 --dependency=ghc-prim=ghc-prim-0.5.0.0 --dependency=hashable=hashable-1.2.4.0-EMu4H7FB10MAl6hwKw992G --dependency=mtl=mtl-2.2.1-6qsR1PHUy5lL47Hpoa4jCM --dependency=mtl-compat=mtl-compat-0.2.1.3-CtLbq6xU5RAmphYnnSjKT --dependency=safe=safe-0.3.10-1VyrsjWhmjvGnGud5lgW7f --dependency=stm=stm-2.4.4.1-4z2NRWnB0NIIUvSJsHW0kF --dependency=text=text-1.2.2.1-5QpmrLQApEZ4Ly9nMHWY0s --dependency=transformers=transformers-0.5.2.0 --ghc-options -O0 --enable-tests --enable-benchmarks
   [debug] Encoding /Users/jml/src/protolude/.stack-work/dist/x86_64-osx/Cabal-1.24.0.0/stack-config-cache
   [debug] Finished writing /Users/jml/src/protolude/.stack-work/dist/x86_64-osx/Cabal-1.24.0.0/stack-config-cache
   [debug] Encoding /Users/jml/src/protolude/.stack-work/dist/x86_64-osx/Cabal-1.24.0.0/stack-cabal-mod
   [debug] Finished writing /Users/jml/src/protolude/.stack-work/dist/x86_64-osx/Cabal-1.24.0.0/stack-cabal-mod
   [debug] Encoding /Users/jml/src/protolude/.stack-work/dist/x86_64-osx/Cabal-1.24.0.0/stack-build-cache
   [debug] Finished writing /Users/jml/src/protolude/.stack-work/dist/x86_64-osx/Cabal-1.24.0.0/stack-build-cache
   [info] protolude-0.2: build (lib)
   [debug] Run process: /Users/jml/.stack/setup-exe-cache/x86_64-osx/Cabal-simple_mPHDZzAJ_1.24.0.0_ghc-8.0.1 --verbose --builddir=.stack-work/dist/x86_64-osx/Cabal-1.24.0.0 build lib:protolude --ghc-options " -ddump-hi -ddump-to-file"
   [info] Component build order: library
   [info] creating .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build
   [info] creating .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/autogen
   [info] /Users/jml/.stack/programs/x86_64-osx/ghc-8.0.1/bin/ghc-pkg init .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/package.conf.inplace
   [info] Preprocessing library protolude-0.2...
   [info] Building library...
   [info] creating .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build
   [info] /Users/jml/.stack/programs/x86_64-osx/ghc-8.0.1/bin/ghc --make -fbuilding-cabal-package -O -static -dynamic-too -dynosuf dyn_o -dynhisuf dyn_hi -outputdir .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build -odir .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build -hidir .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build -stubdir .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build -i -i.stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build -isrc -i.stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/autogen -I.stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/autogen -I.stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build -optP-include -optP.stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/autogen/cabal_macros.h -this-unit-id protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW -hide-all-packages -no-user-package-db -package-db /Users/jml/.stack/snapshots/x86_64-osx/lts-7.14/8.0.1/pkgdb -package-db /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/pkgdb -package-db .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/package.conf.inplace -package-id array-0.5.1.1 -package-id async-2.1.1-xFiBzw9xoB8HPZAuxUY2o -package-id base-4.9.0.0 -package-id bytestring-0.10.8.1 -package-id containers-0.5.7.1 -package-id deepseq-1.4.2.0 -package-id ghc-prim-0.5.0.0 -package-id hashable-1.2.4.0-EMu4H7FB10MAl6hwKw992G -package-id mtl-2.2.1-6qsR1PHUy5lL47Hpoa4jCM -package-id mtl-compat-0.2.1.3-CtLbq6xU5RAmphYnnSjKT -package-id safe-0.3.10-1VyrsjWhmjvGnGud5lgW7f -package-id stm-2.4.4.1-4z2NRWnB0NIIUvSJsHW0kF -package-id text-1.2.2.1-5QpmrLQApEZ4Ly9nMHWY0s -package-id transformers-0.5.2.0 -XHaskell2010 -XNoImplicitPrelude -XOverloadedStrings -XFlexibleContexts -XMultiParamTypeClasses Protolude Unsafe Debug Protolude.Exceptions Protolude.Base Protolude.Applicative Protolude.Bool Protolude.List Protolude.Monad Protolude.Show Protolude.Conv Protolude.Either Protolude.Functor Protolude.Semiring Protolude.Bifunctor Protolude.CallStack Protolude.Error Protolude.Panic -Wall -fwarn-implicit-prelude -O0 -ddump-hi -ddump-to-file
   [info] [ 1 of 18] Compiling Protolude.CallStack ( src/Protolude/CallStack.hs, .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/CallStack.o )
   [info] [ 2 of 18] Compiling Protolude.Bifunctor ( src/Protolude/Bifunctor.hs, .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Bifunctor.o )
   [info] [ 3 of 18] Compiling Protolude.Error  ( src/Protolude/Error.hs, .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Error.o )
   [info] [ 4 of 18] Compiling Protolude.Base   ( src/Protolude/Base.hs, .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Base.o )
   [info] [ 5 of 18] Compiling Protolude.Semiring ( src/Protolude/Semiring.hs, .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Semiring.o )
   [info] [ 6 of 18] Compiling Protolude.Exceptions ( src/Protolude/Exceptions.hs, .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Exceptions.o )
   [info] [ 7 of 18] Compiling Protolude.Panic  ( src/Protolude/Panic.hs, .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Panic.o )
   [info] [ 8 of 18] Compiling Protolude.Conv   ( src/Protolude/Conv.hs, .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Conv.o )
   [info] [ 9 of 18] Compiling Protolude.Applicative ( src/Protolude/Applicative.hs, .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Applicative.o )
   [info] [10 of 18] Compiling Protolude.Either ( src/Protolude/Either.hs, .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Either.o )
   [info] [11 of 18] Compiling Protolude.Functor ( src/Protolude/Functor.hs, .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Functor.o )
   [info] [12 of 18] Compiling Protolude.Monad  ( src/Protolude/Monad.hs, .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Monad.o )
   [info] [13 of 18] Compiling Protolude.Bool   ( src/Protolude/Bool.hs, .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Bool.o )
   [info] [14 of 18] Compiling Protolude.Show   ( src/Protolude/Show.hs, .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Show.o )
   [info] [15 of 18] Compiling Protolude.List   ( src/Protolude/List.hs, .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/List.o )
   [info] [16 of 18] Compiling Debug            ( src/Debug.hs, .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Debug.o )
   [info] [17 of 18] Compiling Unsafe           ( src/Unsafe.hs, .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Unsafe.o )
   [info] [18 of 18] Compiling Protolude        ( src/Protolude.hs, .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude.o )
   [info] Linking...
   [info] [(SimpleUnitId (ComponentId "array-0.5.1.1"),PackageIdentifier {pkgName =
   [info] PackageName {unPackageName = "array"}, pkgVersion = Version {versionBranch =
   [info] [0,5,1,1], versionTags = []}},ModuleRenaming True []),(SimpleUnitId
   [info] (ComponentId "async-2.1.1-xFiBzw9xoB8HPZAuxUY2o"),PackageIdentifier {pkgName =
   [info] PackageName {unPackageName = "async"}, pkgVersion = Version {versionBranch =
   [info] [2,1,1], versionTags = []}},ModuleRenaming True []),(SimpleUnitId (ComponentId
   [info] "base-4.9.0.0"),PackageIdentifier {pkgName = PackageName {unPackageName =
   [info] "base"}, pkgVersion = Version {versionBranch = [4,9,0,0], versionTags =
   [info] []}},ModuleRenaming True []),(SimpleUnitId (ComponentId
   [info] "bytestring-0.10.8.1"),PackageIdentifier {pkgName = PackageName {unPackageName
   [info] = "bytestring"}, pkgVersion = Version {versionBranch = [0,10,8,1], versionTags
   [info] = []}},ModuleRenaming True []),(SimpleUnitId (ComponentId
   [info] "containers-0.5.7.1"),PackageIdentifier {pkgName = PackageName {unPackageName
   [info] = "containers"}, pkgVersion = Version {versionBranch = [0,5,7,1], versionTags
   [info] = []}},ModuleRenaming True []),(SimpleUnitId (ComponentId
   [info] "deepseq-1.4.2.0"),PackageIdentifier {pkgName = PackageName {unPackageName =
   [info] "deepseq"}, pkgVersion = Version {versionBranch = [1,4,2,0], versionTags =
   [info] []}},ModuleRenaming True []),(SimpleUnitId (ComponentId
   [info] "ghc-prim-0.5.0.0"),PackageIdentifier {pkgName = PackageName {unPackageName =
   [info] "ghc-prim"}, pkgVersion = Version {versionBranch = [0,5,0,0], versionTags =
   [info] []}},ModuleRenaming True []),(SimpleUnitId (ComponentId
   [info] "hashable-1.2.4.0-EMu4H7FB10MAl6hwKw992G"),PackageIdentifier {pkgName =
   [info] PackageName {unPackageName = "hashable"}, pkgVersion = Version {versionBranch
   [info] = [1,2,4,0], versionTags = []}},ModuleRenaming True []),(SimpleUnitId
   [info] (ComponentId "mtl-2.2.1-6qsR1PHUy5lL47Hpoa4jCM"),PackageIdentifier {pkgName =
   [info] PackageName {unPackageName = "mtl"}, pkgVersion = Version {versionBranch =
   [info] [2,2,1], versionTags = []}},ModuleRenaming True []),(SimpleUnitId (ComponentId
   [info] "mtl-compat-0.2.1.3-CtLbq6xU5RAmphYnnSjKT"),PackageIdentifier {pkgName =
   [info] PackageName {unPackageName = "mtl-compat"}, pkgVersion = Version
   [info] {versionBranch = [0,2,1,3], versionTags = []}},ModuleRenaming True
   [info] []),(SimpleUnitId (ComponentId
   [info] "safe-0.3.10-1VyrsjWhmjvGnGud5lgW7f"),PackageIdentifier {pkgName = PackageName
   [info] {unPackageName = "safe"}, pkgVersion = Version {versionBranch = [0,3,10],
   [info] versionTags = []}},ModuleRenaming True []),(SimpleUnitId (ComponentId
   [info] "stm-2.4.4.1-4z2NRWnB0NIIUvSJsHW0kF"),PackageIdentifier {pkgName = PackageName
   [info] {unPackageName = "stm"}, pkgVersion = Version {versionBranch = [2,4,4,1],
   [info] versionTags = []}},ModuleRenaming True []),(SimpleUnitId (ComponentId
   [info] "text-1.2.2.1-5QpmrLQApEZ4Ly9nMHWY0s"),PackageIdentifier {pkgName =
   [info] PackageName {unPackageName = "text"}, pkgVersion = Version {versionBranch =
   [info] [1,2,2,1], versionTags = []}},ModuleRenaming True []),(SimpleUnitId
   [info] (ComponentId "transformers-0.5.2.0"),PackageIdentifier {pkgName = PackageName
   [info] {unPackageName = "transformers"}, pkgVersion = Version {versionBranch =
   [info] [0,5,2,0], versionTags = []}},ModuleRenaming True [])]
   [info] /usr/bin/ar -r -s .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/objs-3164/libHSprotolude-0.2-6HCoCsk9tmDFZ20MEUtzMW.a .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude.o .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Unsafe.o .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Debug.o .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Exceptions.o .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Base.o .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Applicative.o .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Bool.o .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/List.o .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Monad.o .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Show.o .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Conv.o .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Either.o .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Functor.o .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Semiring.o .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Bifunctor.o .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/CallStack.o .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Error.o .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Panic.o
   [warn] ar: creating archive .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/objs-3164/libHSprotolude-0.2-6HCoCsk9tmDFZ20MEUtzMW.a
   [info] /Users/jml/.stack/programs/x86_64-osx/ghc-8.0.1/bin/ghc -shared -dynamic '-dynload deploy' -optl-Wl,-rpath,/Users/jml/.stack/programs/x86_64-osx/ghc-8.0.1/lib/ghc-8.0.1/array-0.5.1.1 -optl-Wl,-rpath,/Users/jml/.stack/snapshots/x86_64-osx/lts-7.13/8.0.1/lib/x86_64-osx-ghc-8.0.1/async-2.1.1-xFiBzw9xoB8HPZAuxUY2o -optl-Wl,-rpath,/Users/jml/.stack/programs/x86_64-osx/ghc-8.0.1/lib/ghc-8.0.1/base-4.9.0.0 -optl-Wl,-rpath,/Users/jml/.stack/programs/x86_64-osx/ghc-8.0.1/lib/ghc-8.0.1/binary-0.8.3.0 -optl-Wl,-rpath,/Users/jml/.stack/programs/x86_64-osx/ghc-8.0.1/lib/ghc-8.0.1/bytestring-0.10.8.1 -optl-Wl,-rpath,/Users/jml/.stack/programs/x86_64-osx/ghc-8.0.1/lib/ghc-8.0.1/containers-0.5.7.1 -optl-Wl,-rpath,/Users/jml/.stack/programs/x86_64-osx/ghc-8.0.1/lib/ghc-8.0.1/deepseq-1.4.2.0 -optl-Wl,-rpath,/Users/jml/.stack/programs/x86_64-osx/ghc-8.0.1/lib/ghc-8.0.1/ghc-prim-0.5.0.0 -optl-Wl,-rpath,/Users/jml/.stack/snapshots/x86_64-osx/lts-7.13/8.0.1/lib/x86_64-osx-ghc-8.0.1/hashable-1.2.4.0-EMu4H7FB10MAl6hwKw992G -optl-Wl,-rpath,/Users/jml/.stack/programs/x86_64-osx/ghc-8.0.1/lib/ghc-8.0.1/integer-gmp-1.0.0.1 -optl-Wl,-rpath,/Users/jml/.stack/snapshots/x86_64-osx/lts-7.13/8.0.1/lib/x86_64-osx-ghc-8.0.1/mtl-2.2.1-6qsR1PHUy5lL47Hpoa4jCM -optl-Wl,-rpath,/Users/jml/.stack/programs/x86_64-osx/ghc-8.0.1/lib/ghc-8.0.1/rts -optl-Wl,-rpath,/Users/jml/.stack/snapshots/x86_64-osx/lts-7.13/8.0.1/lib/x86_64-osx-ghc-8.0.1/safe-0.3.10-1VyrsjWhmjvGnGud5lgW7f -optl-Wl,-rpath,/Users/jml/.stack/snapshots/x86_64-osx/lts-7.13/8.0.1/lib/x86_64-osx-ghc-8.0.1/stm-2.4.4.1-4z2NRWnB0NIIUvSJsHW0kF -optl-Wl,-rpath,/Users/jml/.stack/snapshots/x86_64-osx/lts-7.13/8.0.1/lib/x86_64-osx-ghc-8.0.1/text-1.2.2.1-5QpmrLQApEZ4Ly9nMHWY0s -optl-Wl,-rpath,/Users/jml/.stack/programs/x86_64-osx/ghc-8.0.1/lib/ghc-8.0.1/transformers-0.5.2.0 -no-auto-link-packages -no-user-package-db -package-db /Users/jml/.stack/snapshots/x86_64-osx/lts-7.14/8.0.1/pkgdb -package-db /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/pkgdb -package-db .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/package.conf.inplace -package-id array-0.5.1.1 -package-id async-2.1.1-xFiBzw9xoB8HPZAuxUY2o -package-id base-4.9.0.0 -package-id bytestring-0.10.8.1 -package-id containers-0.5.7.1 -package-id deepseq-1.4.2.0 -package-id ghc-prim-0.5.0.0 -package-id hashable-1.2.4.0-EMu4H7FB10MAl6hwKw992G -package-id mtl-2.2.1-6qsR1PHUy5lL47Hpoa4jCM -package-id mtl-compat-0.2.1.3-CtLbq6xU5RAmphYnnSjKT -package-id safe-0.3.10-1VyrsjWhmjvGnGud5lgW7f -package-id stm-2.4.4.1-4z2NRWnB0NIIUvSJsHW0kF -package-id text-1.2.2.1-5QpmrLQApEZ4Ly9nMHWY0s -package-id transformers-0.5.2.0 .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude.dyn_o .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Unsafe.dyn_o .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Debug.dyn_o .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Exceptions.dyn_o .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Base.dyn_o .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Applicative.dyn_o .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Bool.dyn_o .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/List.dyn_o .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Monad.dyn_o .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Show.dyn_o .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Conv.dyn_o .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Either.dyn_o .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Functor.dyn_o .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Semiring.dyn_o .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Bifunctor.dyn_o .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/CallStack.dyn_o .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Error.dyn_o .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Panic.dyn_o -o .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/libHSprotolude-0.2-6HCoCsk9tmDFZ20MEUtzMW-ghc8.0.1.dylib -O0 -ddump-hi -ddump-to-file
   [info] /Users/jml/.stack/programs/x86_64-osx/ghc-8.0.1/bin/ghc-pkg update - --global --no-user-package-db '--package-db=/Users/jml/.stack/snapshots/x86_64-osx/lts-7.14/8.0.1/pkgdb' '--package-db=/Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/pkgdb' '--package-db=.stack-work/dist/x86_64-osx/Cabal-1.24.0.0/package.conf.inplace'
   [debug] Process finished in 2987ms: /Users/jml/.stack/setup-exe-cache/x86_64-osx/Cabal-simple_mPHDZzAJ_1.24.0.0_ghc-8.0.1 --verbose --builddir=.stack-work/dist/x86_64-osx/Cabal-1.24.0.0 build lib:protolude --ghc-options " -ddump-hi -ddump-to-file"
   [debug] Start: getPackageFiles /Users/jml/src/protolude/protolude.cabal
   [debug] Finished in 7ms: getPackageFiles /Users/jml/src/protolude/protolude.cabal
   [debug] Encoding /Users/jml/src/protolude/.stack-work/dist/x86_64-osx/Cabal-1.24.0.0/stack-build-cache
   [debug] Finished writing /Users/jml/src/protolude/.stack-work/dist/x86_64-osx/Cabal-1.24.0.0/stack-build-cache
   [info] protolude-0.2: copy/register
   [debug] Run process: /Users/jml/.stack/setup-exe-cache/x86_64-osx/Cabal-simple_mPHDZzAJ_1.24.0.0_ghc-8.0.1 --verbose --builddir=.stack-work/dist/x86_64-osx/Cabal-1.24.0.0 copy
   [info] directory .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/doc/html/protolude does
   [info] exist: False
   [info] creating
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/doc/protolude-0.2
   [info] Installing LICENSE to
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/doc/protolude-0.2/LICENSE
   [info] Installing library in
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/lib/x86_64-osx-ghc-8.0.1/protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW
   [info] creating
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/lib/x86_64-osx-ghc-8.0.1/protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW
   [info] creating
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/lib/x86_64-osx-ghc-8.0.1/protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW/Protolude
   [info] Installing .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude.hi to
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/lib/x86_64-osx-ghc-8.0.1/protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW/Protolude.hi
   [info] Installing .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Unsafe.hi to
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/lib/x86_64-osx-ghc-8.0.1/protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW/Unsafe.hi
   [info] Installing .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Debug.hi to
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/lib/x86_64-osx-ghc-8.0.1/protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW/Debug.hi
   [info] Installing
   [info] .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Exceptions.hi to
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/lib/x86_64-osx-ghc-8.0.1/protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW/Protolude/Exceptions.hi
   [info] Installing .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Base.hi
   [info] to
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/lib/x86_64-osx-ghc-8.0.1/protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW/Protolude/Base.hi
   [info] Installing
   [info] .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Applicative.hi to
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/lib/x86_64-osx-ghc-8.0.1/protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW/Protolude/Applicative.hi
   [info] Installing .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Bool.hi
   [info] to
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/lib/x86_64-osx-ghc-8.0.1/protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW/Protolude/Bool.hi
   [info] Installing .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/List.hi
   [info] to
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/lib/x86_64-osx-ghc-8.0.1/protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW/Protolude/List.hi
   [info] Installing .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Monad.hi
   [info] to
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/lib/x86_64-osx-ghc-8.0.1/protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW/Protolude/Monad.hi
   [info] Installing .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Show.hi
   [info] to
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/lib/x86_64-osx-ghc-8.0.1/protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW/Protolude/Show.hi
   [info] Installing .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Conv.hi
   [info] to
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/lib/x86_64-osx-ghc-8.0.1/protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW/Protolude/Conv.hi
   [info] Installing
   [info] .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Either.hi to
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/lib/x86_64-osx-ghc-8.0.1/protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW/Protolude/Either.hi
   [info] Installing
   [info] .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Functor.hi to
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/lib/x86_64-osx-ghc-8.0.1/protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW/Protolude/Functor.hi
   [info] Installing
   [info] .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Semiring.hi to
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/lib/x86_64-osx-ghc-8.0.1/protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW/Protolude/Semiring.hi
   [info] Installing
   [info] .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Bifunctor.hi to
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/lib/x86_64-osx-ghc-8.0.1/protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW/Protolude/Bifunctor.hi
   [info] Installing
   [info] .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/CallStack.hi to
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/lib/x86_64-osx-ghc-8.0.1/protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW/Protolude/CallStack.hi
   [info] Installing .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Error.hi
   [info] to
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/lib/x86_64-osx-ghc-8.0.1/protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW/Protolude/Error.hi
   [info] Installing .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Panic.hi
   [info] to
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/lib/x86_64-osx-ghc-8.0.1/protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW/Protolude/Panic.hi
   [info] creating
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/lib/x86_64-osx-ghc-8.0.1/protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW
   [info] creating
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/lib/x86_64-osx-ghc-8.0.1/protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW/Protolude
   [info] Installing .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude.dyn_hi
   [info] to
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/lib/x86_64-osx-ghc-8.0.1/protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW/Protolude.dyn_hi
   [info] Installing .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Unsafe.dyn_hi to
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/lib/x86_64-osx-ghc-8.0.1/protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW/Unsafe.dyn_hi
   [info] Installing .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Debug.dyn_hi to
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/lib/x86_64-osx-ghc-8.0.1/protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW/Debug.dyn_hi
   [info] Installing
   [info] .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Exceptions.dyn_hi
   [info] to
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/lib/x86_64-osx-ghc-8.0.1/protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW/Protolude/Exceptions.dyn_hi
   [info] Installing
   [info] .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Base.dyn_hi to
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/lib/x86_64-osx-ghc-8.0.1/protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW/Protolude/Base.dyn_hi
   [info] Installing
   [info] .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Applicative.dyn_hi
   [info] to
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/lib/x86_64-osx-ghc-8.0.1/protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW/Protolude/Applicative.dyn_hi
   [info] Installing
   [info] .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Bool.dyn_hi to
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/lib/x86_64-osx-ghc-8.0.1/protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW/Protolude/Bool.dyn_hi
   [info] Installing
   [info] .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/List.dyn_hi to
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/lib/x86_64-osx-ghc-8.0.1/protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW/Protolude/List.dyn_hi
   [info] Installing
   [info] .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Monad.dyn_hi to
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/lib/x86_64-osx-ghc-8.0.1/protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW/Protolude/Monad.dyn_hi
   [info] Installing
   [info] .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Show.dyn_hi to
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/lib/x86_64-osx-ghc-8.0.1/protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW/Protolude/Show.dyn_hi
   [info] Installing
   [info] .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Conv.dyn_hi to
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/lib/x86_64-osx-ghc-8.0.1/protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW/Protolude/Conv.dyn_hi
   [info] Installing
   [info] .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Either.dyn_hi to
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/lib/x86_64-osx-ghc-8.0.1/protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW/Protolude/Either.dyn_hi
   [info] Installing
   [info] .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Functor.dyn_hi to
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/lib/x86_64-osx-ghc-8.0.1/protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW/Protolude/Functor.dyn_hi
   [info] Installing
   [info] .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Semiring.dyn_hi to
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/lib/x86_64-osx-ghc-8.0.1/protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW/Protolude/Semiring.dyn_hi
   [info] Installing
   [info] .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Bifunctor.dyn_hi to
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/lib/x86_64-osx-ghc-8.0.1/protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW/Protolude/Bifunctor.dyn_hi
   [info] Installing
   [info] .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/CallStack.dyn_hi to
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/lib/x86_64-osx-ghc-8.0.1/protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW/Protolude/CallStack.dyn_hi
   [info] Installing
   [info] .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Error.dyn_hi to
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/lib/x86_64-osx-ghc-8.0.1/protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW/Protolude/Error.dyn_hi
   [info] Installing
   [info] .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/Protolude/Panic.dyn_hi to
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/lib/x86_64-osx-ghc-8.0.1/protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW/Protolude/Panic.dyn_hi
   [info] creating
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/lib/x86_64-osx-ghc-8.0.1/protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW
   [info] Installing
   [info] .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/libHSprotolude-0.2-6HCoCsk9tmDFZ20MEUtzMW.a
   [info] to
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/lib/x86_64-osx-ghc-8.0.1/protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW/libHSprotolude-0.2-6HCoCsk9tmDFZ20MEUtzMW.a
   [info] creating
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/lib/x86_64-osx-ghc-8.0.1/protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW
   [info] Installing executable
   [info] .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/libHSprotolude-0.2-6HCoCsk9tmDFZ20MEUtzMW-ghc8.0.1.dylib
   [info] to
   [info] /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/lib/x86_64-osx-ghc-8.0.1/protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW/libHSprotolude-0.2-6HCoCsk9tmDFZ20MEUtzMW-ghc8.0.1.dylib
   [debug] Process finished in 75ms: /Users/jml/.stack/setup-exe-cache/x86_64-osx/Cabal-simple_mPHDZzAJ_1.24.0.0_ghc-8.0.1 --verbose --builddir=.stack-work/dist/x86_64-osx/Cabal-1.24.0.0 copy
   [debug] Run process: /Users/jml/.stack/setup-exe-cache/x86_64-osx/Cabal-simple_mPHDZzAJ_1.24.0.0_ghc-8.0.1 --verbose --builddir=.stack-work/dist/x86_64-osx/Cabal-1.24.0.0 register
   [info] /Users/jml/.stack/programs/x86_64-osx/ghc-8.0.1/bin/ghc --abi-hash -fbuilding-cabal-package -O -outputdir .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build -odir .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build -hidir .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build -stubdir .stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build -i -i.stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build -isrc -i.stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/autogen -I.stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/autogen -I.stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build -optP-include -optP.stack-work/dist/x86_64-osx/Cabal-1.24.0.0/build/autogen/cabal_macros.h -this-unit-id protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW -hide-all-packages -no-user-package-db -package-db /Users/jml/.stack/snapshots/x86_64-osx/lts-7.14/8.0.1/pkgdb -package-db /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/pkgdb -package-id array-0.5.1.1 -package-id async-2.1.1-xFiBzw9xoB8HPZAuxUY2o -package-id base-4.9.0.0 -package-id bytestring-0.10.8.1 -package-id containers-0.5.7.1 -package-id deepseq-1.4.2.0 -package-id ghc-prim-0.5.0.0 -package-id hashable-1.2.4.0-EMu4H7FB10MAl6hwKw992G -package-id mtl-2.2.1-6qsR1PHUy5lL47Hpoa4jCM -package-id mtl-compat-0.2.1.3-CtLbq6xU5RAmphYnnSjKT -package-id safe-0.3.10-1VyrsjWhmjvGnGud5lgW7f -package-id stm-2.4.4.1-4z2NRWnB0NIIUvSJsHW0kF -package-id text-1.2.2.1-5QpmrLQApEZ4Ly9nMHWY0s -package-id transformers-0.5.2.0 -XHaskell2010 -XNoImplicitPrelude -XOverloadedStrings -XFlexibleContexts -XMultiParamTypeClasses Protolude Unsafe Debug Protolude.Exceptions Protolude.Base Protolude.Applicative Protolude.Bool Protolude.List Protolude.Monad Protolude.Show Protolude.Conv Protolude.Either Protolude.Functor Protolude.Semiring Protolude.Bifunctor Protolude.CallStack Protolude.Error Protolude.Panic -Wall -fwarn-implicit-prelude -O0
   [info] Registering protolude-0.2...
   [info] /Users/jml/.stack/programs/x86_64-osx/ghc-8.0.1/bin/ghc-pkg update - --global --no-user-package-db '--package-db=/Users/jml/.stack/snapshots/x86_64-osx/lts-7.14/8.0.1/pkgdb' '--package-db=/Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/pkgdb'
   [debug] Process finished in 212ms: /Users/jml/.stack/setup-exe-cache/x86_64-osx/Cabal-simple_mPHDZzAJ_1.24.0.0_ghc-8.0.1 --verbose --builddir=.stack-work/dist/x86_64-osx/Cabal-1.24.0.0 register
   [debug] Run process: /Users/jml/.stack/programs/x86_64-osx/ghc-8.0.1/bin/ghc-pkg --user --no-user-package-db --package-db /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/pkgdb describe --simple-output protolude --expand-pkgroot
   [debug] Process finished in 26ms: /Users/jml/.stack/programs/x86_64-osx/ghc-8.0.1/bin/ghc-pkg --user --no-user-package-db --package-db /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/pkgdb describe --simple-output protolude --expand-pkgroot
   [debug] Encoding /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/flag-cache/protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW
   [debug] Finished writing /Users/jml/src/protolude/.stack-work/install/x86_64-osx/lts-7.14/8.0.1/flag-cache/protolude-0.2-6HCoCsk9tmDFZ20MEUtzMW


Auto-format Bazel files with `buildifier <https://github.com/bazelbuild/buildtools>`_:

.. code-block:: console

   $ buildifier -showlog -mode=fix $(find . \( -name '*.bzl' -o -name '*.BUILD' -o -name 'WORKSPACE' -o -name 'BUILD' \) -type f)


https://github.com/bazelbuild/rules_rust/blob/master/rust/rust.bzl

.. code-block::

       fragments = ["cpp"],

What's all this about then?


References
==========

Static linking
--------------

* `Minimal example of static linking with Stack <https://github.com/jml/haskell-static-minimal-repro>`_
* `How can I create static executables on OS X with Stack? <https://stackoverflow.com/questions/39805657/how-can-i-create-static-executables-on-os-x-with-stack>`_
* `Build static Haskell executable with Nix <https://gist.github.com/teh/f4b45ba1ac46f0ae618c05739570d026>`_
* `Support for out of the box static linking <https://ghc.haskell.org/trac/ghc/ticket/10912>`_
* `Statically linked binaries on Mac OS X <https://developer.apple.com/library/content/qa/qa1118/_index.html>`_

macOS
-----

* `Workflow/tools for installing command line application on OS X (Yosemite or later) <https://apple.stackexchange.com/questions/234979/workflow-tools-for-installing-command-line-application-on-os-x-yosemite-or-late>`_
* `Distributing Your Application <https://developer.apple.com/library/content/documentation/Porting/Conceptual/PortingUnix/distributing/distibuting.html#//apple_ref/doc/uid/TP40002855-TPXREF101>`_

Cross compiling
---------------

* `How to do cross-compilation with GHC <https://ghc.haskell.org/trac/ghc/wiki/Building/CrossCompiling>`_
* `Cross-compilation using Clang <https://clang.llvm.org/docs/CrossCompilation.html>`_

.. _bazel: https://bazel.build/
.. _`cross-compiling support`: https://github.com/bazelbuild/rules_go/issues/70
.. _gazelle: https://github.com/bazelbuild/rules_go#generating-build-files
.. _servant-template: https://github.com/jml/servant-template/
.. _minikube: https://github.com/kubernetes/minikube/
.. _difftodo: https://github.com/jml/difftodo/
.. _compare-revisions: https://github.com/weaveworks-experiments/compare-revisions
.. _cortex: https://github.com/weaveworks/cortex
