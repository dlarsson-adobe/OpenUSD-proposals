# Consistent Plugin and Python Deployment for Exported CMake Targets

Status: Draft

Authors: David Larsson (Adobe)

## Contents

- [Summary](#summary)
- [Problem Statement](#problem-statement)
- [Glossary](#glossary)
- [Prior Art](#prior-art)
- [Proposal](#proposal)
  - [Installable Targets and Exports](#installable-targets-and-exports)
  - [Deployment Metadata](#deployment-metadata)
  - [Inventories](#inventories)
  - [Consumer Install Utilities](#consumer-install-utilities)
  - [Third-Party Plugins](#third-party-plugins)
  - [Build Modes](#build-modes)
  - [USD's Own Installs](#usds-own-installs)
- [Examples](#examples)
- [Alternatives Considered](#alternatives-considered)
- [Risks](#risks)
- [Rollout Plan](#rollout-plan)
- [Later Phase: Plugin Dependency Closure](#later-phase-plugin-dependency-closure)
- [Future Work](#future-work)
- [Out of Scope](#out-of-scope)

## Summary

A USD-based application needs more than the libraries it links.
At run time it also needs plugin binaries, their `plugInfo.json` registrations and resources, compiled Python modules and pure-Python packages.
Today USD's CMake build knows these files only as install rules for its own layout, so nothing a consumer can query describes them.
As a result, deploying USD inside an application means copying USD's install tree wholesale, setting environment variables such as `PXR_PLUGINPATH_NAME`, or maintaining hand-written file lists.

This proposal has three goals:

- **Consistent installation in every consumption mode.**
  A project installs USD's plugins and Python modules the same way whether it consumes USD as exported targets through `find_package(pxr)`, as a subproject through `add_subdirectory` or `FetchContent`, or as the top-level build.
- **Out-of-tree plugins installed like in-tree plugins.**
  Plugins built outside the USD source tree follow the same convention as USD's own, so an application deploys both with the same calls.
- **Flexible plugin locations.**
  Developers choose where plugins, resources and Python packages go in their own deployment, instead of reproducing USD's install layout.

To get there, USD would export every installable target and attach deployment metadata to each one: the resources, Python files and Python package it needs, and the plugin registrations it provides.
The metadata reads the same in every consumption mode.
USD would also ship CMake utilities in its config package that install selected plugins and Python packages into a layout the consumer chooses, and generate the registration files USD's plugin system needs to find them there.

The core of the proposal adds metadata alongside USD's existing install rules and does not change how USD installs itself.
Plugin dependency closure is a later phase, and CPS interoperability is future work.

## Problem Statement

A project can obtain USD's CMake targets in three ways, called consumption modes:

- **Top-level build:** USD is the project being built and installed.
- **Subproject:** a parent project adds USD with `add_subdirectory` or `FetchContent`, often with `EXCLUDE_FROM_ALL` so only the parts it links are built.
- **Imported:** a project calls `find_package(pxr)` against an installed USD.

Only the top-level build deploys correctly today, and only into USD's own layout.

### Imported consumers cannot see plugins or Python modules

`pxrTargets.cmake` exports libraries, but deliberately leaves out plugins such as `hdStorm` and `usdAbc`, the compiled Python modules (`_<lib>`), and the C++ tools.
Resource-only plugins and pure-Python packages have no target at all.
No exported target records which resources, `plugInfo.json` files or Python files it needs.
A consumer that wants to ship USD with its application therefore cannot ask CMake which files to install.

### Subproject consumers get all of USD or nothing

In subproject mode the targets exist, but they carry no deployment metadata either.
The only way to install USD's runtime files is to run USD's own install rules, which install everything USD configured, into USD's layout.
With `EXCLUDE_FROM_ALL`, those install rules still reference targets that were never built, so "build only what I link and install only that" does not work.

### Every consumer relies on workarounds

Because the build does not describe its runtime files, applications fall back on workarounds that are fragile and hard to share:

- Copying `lib/usd`, `plugin/usd` and `lib/python/pxr` wholesale, including plugins and modules the application never loads.
- Setting `PXR_PLUGINPATH_NAME` or `PYTHONPATH` at launch so USD finds files outside its expected layout.
- Maintaining hand-written lists of plugins, resources and Python files that drift as USD changes.
- Rewriting `plugInfo.json` `LibraryPath` entries by hand when the deployed layout differs from USD's own, since the installed files are configured against USD's install prefix.

### Third-party plugins have no convention

Plugins built outside the USD source tree cannot reuse USD's `pxr_plugin` macros, because USD does not install them.
Each project invents its own way to install binaries, resources, registrations and Python bindings, so an application that combines several plugins must handle each one differently.
