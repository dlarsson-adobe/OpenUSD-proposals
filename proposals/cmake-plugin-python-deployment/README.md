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

## Proposal

The core of the proposal adds exports, deployment metadata and inventories alongside USD's existing install rules.
This section explains the model and its reasons; the companion [reference](reference.md) holds the normative tables.

### Installable Targets and Exports

Every item a consumer may deploy on its own becomes an installable target, addressable by name in every consumption mode.
The installable target kinds are libraries, plugins, compiled Python extension modules, tools, pure-Python packages and resource-only plugins.
[Installable target kinds](reference.md#installable-target-kinds) lists each kind with what upstream exports today and what this proposal adds.

Today `pxrTargets.cmake` exports libraries only.
Plugins such as `hdStorm` and `usdAbc` are built as targets but deliberately left out of the export set.
The compiled Python extension modules, named `_<lib>` (or `_<lib>_d` on Windows with `PXR_USE_DEBUG_PYTHON`), are installed without an export.
C++ tools are installed without an export as well.
Python tools have only a custom target, `<bin>_script`, which cannot be exported.
Resource-only plugins and pure-Python packages have no target at all, only installed files.

This proposal exports plugins, compiled Python extension modules and C++ tools as ordinary targets.
In the top-level build and subproject modes the same targets already exist, so they only gain metadata.

Items without a binary of their own need a target to carry their metadata.
For these, USD adds metadata-only carriers: exported `INTERFACE` libraries that identify deployable content but contain no code.
Resource-only plugins and pure-Python packages each get a carrier.
Python tools also need a carrier, so that they appear among the tools; the property that describes the script is left to a later revision of this proposal.

A carrier joins the deployment inventories, but it does not become something a consumer is expected to link.
Exporting a target for deployment does not change linking semantics for any existing target.

Each target is a logical owner of its content.
Usually the logical owner is also the physical binary that holds the code.
In monolithic builds they differ: the code of every library and core plugin is linked into the aggregate `usd_m`.
Every enabled, independently selectable library, plugin and content owner keeps a logical target in that case as well.
The existing per-library `OBJECT` targets can carry the metadata.
Resources, registrations and Python ownership stay on the logical owners and are not duplicated on the aggregate.
[Deployment Metadata](#deployment-metadata) describes how a logical owner names its physical binary.

The change adds exports and metadata only.
USD's own install layout and install rules stay as they are; [USD's Own Installs](#usds-own-installs) covers how they relate.

Why carriers rather than attaching everything to the nearest library?
A pure-Python package or a resource-only plugin can be selected without any particular library, and attaching it to one would make selection pull in unrelated content.
Why not export the existing custom targets?
CMake cannot export custom targets, so an `INTERFACE` library is the only exportable target without a binary.

### Deployment Metadata

Deployment metadata is a small family of target properties that describe what a target needs alongside its binary.
[Deployment properties](reference.md#deployment-properties) lists each property, its value format and how it is exported.

Public properties and variables follow the form `PXR_<NOUN>`, public functions follow `pxr_<verb>_<noun>`, and internal names use `_PXR_*` or `_pxr_*`.
No name starts with `INTERFACE_` or `IMPORTED_`, because `EXPORT_PROPERTIES` rejects those prefixes.
These are the first custom target properties USD exports.
The names avoid the existing global property `PXR_PYTHON_MODULES`.

Resource files, Python files and Python package identity are separate properties: `PXR_RESOURCE_FILES`, `PXR_PYTHON_FILES` and `PXR_PYTHON_PACKAGE`.
A consumer can then install resources and Python files to independent roots.

File lists use `<source>:<destination>` entries.
For `PXR_RESOURCE_FILES` the destination is relative to the target's resource bundle.
For `PXR_PYTHON_FILES` it is relative to the Python package.
Windows drive letters in sources are preserved, and entries follow documented CMake list escaping.
The producer macro option `no_subst` stays a producer detail and does not appear in these lists.
It is not needed there: ordinary resources are copied unchanged, and declared registrations are rewritten.

In the top-level build and subproject modes, `PXR_<X>` holds absolute source locations.
Authored files point into the source tree, and configured files such as `plugInfo.json` point into the build tree.
`EXPORT_PROPERTIES` exports a property with the same name and value, and it rejects generator expressions.
So the exported value cannot be the in-tree one.
Each file-list property therefore has an exported sibling `PXR_INSTALLED_<X>`: `PXR_INSTALLED_RESOURCE_FILES` and `PXR_INSTALLED_PYTHON_FILES`.
Producers set it on their targets for export, with sources relative to the installed USD prefix.
When the package loads, a helper named `pxr_resolve_installed_metadata` resolves `PXR_INSTALLED_<X>` into `PXR_<X>` with absolute locations; [Third-Party Plugins](#third-party-plugins) gives its details.
Consumers read only `PXR_<X>`, in every consumption mode; in an imported package they see the resolved values.
`PXR_INSTALLED_<X>` is part of the documented producer contract, because third-party producers emit it too.
Installed content never depends on the producer's source or build tree.

Consumers choose independent destination roots per target and per content category.
Bundle-relative and package-relative layout is preserved under those roots.
Arbitrary remapping of single files is not promised.
Metadata that breaks the [validation rules](reference.md#metadata-validation-rules), such as a destination that escapes its root, is an error.

`PXR_PLUGIN_REGISTRATIONS` names the registration descriptors a target owns.
Each entry is bundle-relative and takes one of two forms:

- `<descriptor>`: every registration entry in that descriptor belongs to the owner.
- `<descriptor>:<Name>`: only the `Plugins` entry named `<Name>` belongs to the owner, for descriptors that mix owners.

The property is exported as-is.
A registration is what makes a target a plugin; there is no separate role marker.

Registrations are explicit for three reasons:

- Not every file named `plugInfo.json` is a registration descriptor; `usd`'s `codegenTemplates/plugInfo.json` is a template that must not be rewritten.
- One bundle can register several plugins on one binary, so registration entries cannot be derived from targets one to one.
- Resource-only plugins have no `LibraryPath`, so the absence of a binary must be stated rather than guessed.

Consumers rewrite only the declared registration entries.
They keep every other JSON field, update `Root` and `ResourcePath` consistently when the chosen layout requires it, and copy every other resource unchanged.
Producer descriptors are never modified in place.
An installed USD contains fully specified paths, so consumers do not rely on unexpanded `@...@` placeholders.

`PXR_PLUGIN_LOAD_MODE` is required on every registration owner and is exported as-is.
It states how the plugin's code reaches the process, which decides what happens to `LibraryPath`:

- `LOADABLE`: the code is in a runtime-loadable shared binary; consumers rewrite `LibraryPath` for that binary and layout.
- `BUILTIN`: the code is already linked into or loaded by the consumer; `LibraryPath` stays empty, even when the code lives in a shared aggregate.
- `RESOURCE_ONLY`: there is no code binary; consumers never invent a `LibraryPath`.

A `BUILTIN` or `RESOURCE_ONLY` entry that ships a nonempty `LibraryPath` contradicts its mode and is an error.
A `LOADABLE` entry that ships an empty or missing `LibraryPath` is accepted, because consumers always replace `LibraryPath` for `LOADABLE` entries.
A `RESOURCE_ONLY` owner with a native code binary or a `PXR_DEPLOYMENT_BINARY` is also an error.

All registrations on one owner share a mode, so a producer splits mixed modes across logical owners.
An `INTERFACE` owner is not by itself proof of a resource-only plugin.
The mode describes loading, not role.
[Plugin load modes](reference.md#plugin-load-modes) summarizes the three values.

`PXR_DEPLOYMENT_BINARY` is an optional reference from a logical owner to the target that holds its code, for example from a monolithic member to `usd_m`.
Without it, the owner's own binary is used, or no binary for a carrier.
There is no reverse list from an aggregate to its members.
The property is exported as-is with no installed sibling.
Its target name is resolved through the producer's explicit mapping from build targets to exported names, including namespaces and `EXPORT_NAME`.
Selecting a member brings its physical binary only, not the content of its siblings.
Selecting the aggregate alone does not select the content of its members.
This is not dependency closure, which is a later phase.

The package owner carries the Python metadata for one package.
It is normally the library target, or a metadata-only carrier for a pure-Python package.
It sets `PXR_PYTHON_PACKAGE` to the importable package path relative to the consumer's Python root, for example `pxr/Usd`.
Absolute paths and parent traversal are invalid.
It sets `PXR_PYTHON_FILES` to the package files.
It names its compiled extension target, if any, with `PXR_PYTHON_EXTENSION`, which is exported as-is and resolved like other target references.
Consumers never infer the extension from `_<lib>` naming.
Each package file list has one authoritative owner, and the extension target carries no copy.
Only Python outputs that the build enables are advertised.
Policy for parent and namespace package initializers belongs to [Consumer Install Utilities](#consumer-install-utilities).

There is no prerequisite property.
A producer attaches any generator of deployable content to the deployable target with `add_dependencies`, which works on `INTERFACE` carriers since CMake 3.3.
A subproject consumer adds its own dependencies on the targets it selects.
Installation never builds anything implicitly and never silently omits missing inputs.

Configurations share content lists.
Per-configuration binaries use native CMake per-configuration selection.
Configuration-specific content variants are not proposed without a concrete case.

Producers exclude tests and bytecode from the metadata and keep runtime data such as `codegenTemplates`.

Why not one combined property, or a JSON encoding?
Separate list properties let consumers send each category to its own root with ordinary CMake list handling, while a combined or JSON value would need parsing in every consumer.
Why not a role marker property?
It would be a second source of truth beside the metadata, and third-party targets get their roles automatically from their metadata.
Why not a prerequisite property?
USD generates almost no deployable content at build time, and `add_dependencies` already expresses the rest, including on carriers.
Why not generator expressions in exported values?
`EXPORT_PROPERTIES` rejects them, which is why exported file lists use the `PXR_INSTALLED_<X>` siblings.

### Inventories

Consumers also need to discover which installable targets exist.
USD publishes six deployment inventories as global properties, each a list of targets:

- `PXR_LIBRARY_TARGETS`: libraries.
- `PXR_PLUGIN_TARGETS`: plugins, including resource-only carriers.
- `PXR_TOOL_TARGETS`: tools, both C++ executable targets and Python-tool metadata-only carriers.
- `PXR_PYTHON_PACKAGE_TARGETS`: Python package owners.
- `PXR_PYTHON_EXTENSION_TARGETS`: compiled Python extension modules.
- `PXR_INSTALLABLE_TARGETS`: the union of the role inventories plus otherwise unclassified content carriers.

[Inventories](reference.md#inventories) gives the membership of each.

Roles derive from metadata and native target kind.
A target with a registration is a plugin, and a target with `PXR_PYTHON_PACKAGE` is a package owner.
Roles overlap, so a target can appear in several inventories, and consumers deduplicate their selections.

The inventories are available in every consumption mode, including to the parent of an `EXCLUDE_FROM_ALL` subproject.
Imported inventories reference exported targets, and in-tree inventories reference the corresponding build or carrier targets.
They list only targets enabled in that USD build, and an absent category is an explicitly empty list.
In monolithic builds they list the enabled logical owners and the applicable physical library targets.

Membership neither builds nor installs anything.
Bulk selection is explicit at the selection site, by querying the inventory there and passing it to a consumer utility:

```cmake
get_property(plugins GLOBAL PROPERTY PXR_PLUGIN_TARGETS)
pxr_install_plugins(TARGETS ${plugins} ...)
```

[Consumer Install Utilities](#consumer-install-utilities) specifies the function arguments.
With `EXCLUDE_FROM_ALL`, the consumer adds `add_dependencies` on what it selects so that it is built.
Third-party targets join the inventories only through an explicit `pxr_register_targets(TARGETS ...)` call, described in [Third-Party Plugins](#third-party-plugins); loading a package never enrolls its targets.

`PXR_LIBRARIES` is unchanged.
It keeps its existing linking meaning in imported mode: `usd_m` or all libraries, plus raw system libraries for static Windows builds.
The `_TARGETS` suffix marks lists that contain only targets, never raw library names.

Why not extend `PXR_LIBRARIES`?
It is a link list that may contain raw system library names, and changing its contents would break existing consumers.
Why not `PXR_BINARIES`?
It would mean tools only, which the name does not convey; `PXR_INSTALLABLE_TARGETS` allows "install everything" without deduplicating overlaps and stays forward-compatible with new roles.
Why global properties rather than variables?
A global property is visible from any directory, including the parent of a subproject, and reading it at the selection site avoids stale variable snapshots.

### Consumer Install Utilities

<!-- D02: replace this line with the section -->

### Third-Party Plugins

<!-- D03: replace this line with the section -->

### Build Modes

<!-- D04: replace this line with the Build Modes section -->

### USD's Own Installs

<!-- D04: replace this line with the USD's Own Installs section -->
