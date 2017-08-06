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

0. Preparation
--------------

Estimate: 0.5 hours

- Create a minimal Haskell executable that depends on a minimal library
- Build it with ghc, documenting commands used

1. Pure, native, dynamic
------------------------

Estimate: 1.5 hours

- Build it with Bazel using fyquah95's build rules
- Build it with Bazel using benley's build rules
- Set up some way to easily toggle between them

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

Future ideas
============

* An equivalent of gazelle_ that can automatically generate build rules, perhaps based on cabal or hpack files?
* A tool to one-off generate BUILD files based

References
==========

Static linking
--------------

* `Minimal example of static linking with Stack <https://github.com/jml/haskell-static-minimal-repro>`_
* `How can I create static executables on OS X with Stack? <https://stackoverflow.com/questions/39805657/how-can-i-create-static-executables-on-os-x-with-stack>`_
* `Build static Haskell executable with Nix <https://gist.github.com/teh/f4b45ba1ac46f0ae618c05739570d026>`_
* `Support for out of the box static linking <https://ghc.haskell.org/trac/ghc/ticket/10912>`_

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
