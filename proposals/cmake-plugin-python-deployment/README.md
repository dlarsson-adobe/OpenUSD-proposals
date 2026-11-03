# Consistent Plugin and Python Deployment for Exported CMake Targets

Status: Draft

Authors: David Larsson (Adobe)

## Contents

- [Introduction](#introduction)
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

## Introduction

Applications built on USD need more than the libraries they link against.
At run time USD also loads plugin binaries such as `hdStorm` and `usdAbc`, reads their `plugInfo.json` registrations and resources, and imports compiled and pure-Python modules.
USD's CMake build knows about every one of these files, but only as install rules for its own install tree, so a project that consumes USD cannot ask which files it needs to ship.
In practice, teams copy USD's install tree wholesale, point `PXR_PLUGINPATH_NAME` at it, or maintain a hand-written file list that drifts with every release.

We propose that USD describe these runtime files to CMake as metadata on its exported targets, and ship a small set of CMake functions that install selected plugins and Python packages into a layout the consumer chooses.
The same calls work whether a project imports USD with `find_package(pxr)`, adds it as a subproject, or builds it on its own, and plugins built outside USD's source tree can follow the same convention.
The core of the proposal adds this metadata alongside USD's existing install rules and leaves USD's own install unchanged; plugin dependency closure follows in a later phase.

## Problem Statement

A project can obtain USD's CMake targets in three ways, which we call consumption modes:

- **Top-level build:** USD is the project being built and installed.
- **Subproject:** a parent project adds USD with `add_subdirectory` or `FetchContent`, often with `EXCLUDE_FROM_ALL` so that only the parts it links are built.
- **Imported:** a project calls `find_package(pxr)` against an installed USD.

Today only the top-level build deploys USD correctly, and only into USD's own layout.
The other two modes each fail for a different reason, and the workarounds teams use instead have costs of their own.

### Imported consumers cannot see plugins or Python modules

`pxrTargets.cmake` exports USD's libraries but deliberately leaves out plugins such as `hdStorm` and `usdAbc`, the compiled Python modules, and the C++ tools.
Resource-only plugins and pure-Python packages have no target at all, and no exported target records which resources, `plugInfo.json` files or Python files it needs.
A consumer that wants to ship USD with its application therefore has nothing to ask CMake about.

### Subproject consumers get all of USD or nothing

In subproject mode the targets exist, but they carry no deployment metadata either.
The only way to install USD's runtime files is to run USD's own install rules, which install everything USD configured into USD's layout.
Combined with `EXCLUDE_FROM_ALL`, those rules still reference targets that were never built, so "build only what I link and install only that" does not work.

### Every consumer relies on workarounds

Because nothing in the build describes USD's runtime files, each application reconstructs that knowledge in its own way.
The common approaches all trade one problem for another:

- Copying `lib/usd`, `plugin/usd` and `lib/python/pxr` wholesale ships plugins and modules the application never loads.
- Setting `PXR_PLUGINPATH_NAME` or `PYTHONPATH` at launch lets USD find files outside its expected layout, but ties the application to environment setup that every launcher must repeat.
- Hand-written lists of plugins, resources and Python files work until USD adds, renames or splits a plugin, and then drift silently.
- Rewriting `plugInfo.json` `LibraryPath` entries by hand is needed whenever the deployed layout differs from USD's own, because the installed files are configured against USD's install prefix.

None of this knowledge is shared, so every studio and application vendor maintains its own copy.

### Third-party plugins have no convention

Plugins built outside the USD source tree face the same problem with no shared answer.
They cannot reuse USD's `pxr_plugin` macros, because USD does not install them, so each project invents its own way to install binaries, resources, registrations and Python bindings.
An application that combines several such plugins must then learn and handle each convention separately, on top of the work it already does for USD itself.

## Glossary

<!-- D07: replace this line with the Glossary section -->

## Prior Art

<!-- D07: replace this line with the Prior Art section -->

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

USD ships the consumer utilities as `pxrInstall.cmake` in its config package.
The module is available automatically in every consumption mode: through `pxrConfig.cmake` when USD is imported, and through USD's own setup in the top-level build and subproject modes.
Loading it creates no install rules, and the minimum CMake version stays 3.27.

The module provides three functions, and every call names its targets explicitly:

- `pxr_install_targets(TARGETS ...)` processes every category of content the selected targets advertise: runtime binaries, resources, plugin registrations and Python content, including content on otherwise unclassified metadata-only carriers.
- `pxr_install_plugins(TARGETS ...)` processes plugin content: the binaries, resources and registrations of selected targets that have registrations.
- `pxr_install_python_packages(TARGETS ...)` processes Python content: the files and compiled extension modules of selected package owners.

[Install functions](reference.md#install-functions) lists what each function processes and which arguments it accepts.
The two category functions filter a mixed selection by advertised content, so a valid target without matching content contributes nothing.
Plugin installation does not silently install Python content, and Python installation does not silently install plugins.
Unknown targets and malformed metadata are errors.

There is no install-all mode.
Bulk selection reads an inventory at the selection site, after the producer has registered its targets, and passes it to a function:

```cmake
get_property(plugins GLOBAL PROPERTY PXR_PLUGIN_TARGETS)
pxr_install_plugins(
  TARGETS ${plugins}
  DESTINATION app
  USD_RUNTIME_TARGET plug
  USD_RUNTIME_DESTINATION bin)
```

Here `plug` is the library that contains Plug discovery code in a modular shared build.
`bin` states that the consumer deploys it to `app/bin`, as on Windows, but this call does not install it.
The inventories are authoritative global properties, so the functions take target lists and never rely on ordinary-variable snapshots of them.

All destinations share one vocabulary, and each argument means the same in every function that accepts it.
`DESTINATION` names the deployment root relative to the install prefix, and every other destination is relative to that root.
`RUNTIME_DESTINATION` places runtime binaries, including Windows runtime DLLs, and `LIBRARY_DESTINATION` places shared libraries.
`RESOURCE_DESTINATION` places a target's resource bundle, and `PYTHON_DESTINATION` names the Python root.
[Install arguments](reference.md#install-arguments) gives the defaults and says when each argument is required.
Bundle-relative resource paths and advertised Python package paths are preserved beneath these roots.
Compiled Python extension modules go in their package under the Python root, not in the general binary directory.

A consumer that wants different destinations for some targets or categories makes separate calls that select those targets.
An explicit `RESOURCE_DESTINATION` is the exact bundle root, with no hidden target-name suffix.
Several targets can share it only when their outputs are compatible.
Without it, each target gets its own bundle root under the library directory, named after the target's unqualified name, like USD's own bundle roots under `lib/usd`.
Aliases are resolved before that name is derived, and default names that collide incompatibly require explicit overrides.
Every output path stays beneath the deployment root.
An absolute destination, or parent traversal that escapes the root, is an error, and the GNUInstallDirs defaults get the same validation.

For a target built in the same project, the functions use native target installation.
For an imported target, they use CMake's imported runtime artifact installation, `install(IMPORTED_RUNTIME_ARTIFACTS)`, available since CMake 3.21.
Only runtime artifacts are deployed: no import libraries, headers or CMake package files.
Imported binaries are installed as they are.
A logical owner with `PXR_DEPLOYMENT_BINARY` deploys the binary that property names, without the content of the binary's other members.
[Deployment Metadata](#deployment-metadata) and [deployment properties](reference.md#deployment-properties) define the property.
Copies of a shared aggregate are deduplicated across calls, and aggregate membership never implies plugin closure.
Tools deploy their runtime binary; Python-tool content follows once the property that describes it is defined.

USD's plugin registry has built-in relocatable fallback locations that it searches relative to the binary that contains Plug discovery code.
`PXR_PLUGINPATH_NAME` and `PXR_INSTALL_LOCATION` also feed discovery, but a relocatable deployment relies on these relative locations.
The deployed layout therefore matters relative to that binary, not relative to the producer's install prefix.
`USD_RUNTIME_TARGET` names the target that contains this code, for example `plug` in a modular shared build, the shared monolithic aggregate, or a host executable that embeds static USD.
A static archive is never a valid anchor.
`USD_RUNTIME_DESTINATION` names the directory where that binary is deployed, relative to the deployment root.
Both arguments describe the deployment; they neither select, build nor install the runtime target.
The consumer installs that target through its own selection, which may be an explicitly selected member whose physical binary is the same target.

The functions write a discovery index: a deployment-owned, relocatable top-level `plugInfo.json` at a location that USD's plugin registry searches relative to the runtime binary's directory.
Its relative `Includes` point at the descriptors actually installed, including those under overridden resource roots.
It does not assume one wildcard bundle layout like USD's own `lib/usd/plugInfo.json`, and it never overwrites an unrelated index.
When the same deployment also installs the runtime target, the declared runtime placement is checked against that target's binary destination.
When discovery cannot be validated, configuration fails with a clear message instead of silently requiring `PXR_PLUGINPATH_NAME` at run time.
This check covers the layout only: it does not prove that the host contains or loads the code, or that static registrations survive linking.
[Build Modes](#build-modes) gives the anchor rules for static and monolithic builds.

Producer descriptors are never modified.
Instead, the functions aggregate the selected registration entries across calls into deployment-owned descriptor copies.
A deployed copy omits the entries of unselected plugins and keeps the descriptor-level fields and every other field of the retained entries.
This refines the metadata model for descriptors that mix owners: the deployed copy does not retain registrations for unselected plugins.
`Root` and `ResourcePath` are rewritten for the deployed bundle and layout in every load mode.
`LibraryPath` follows the [load mode](reference.md#plugin-load-modes).
A `LOADABLE` entry gets the path of its selected binary in the chosen layout.
A `BUILTIN` entry keeps it empty, even when the code is in the monolithic aggregate, and a `RESOURCE_ONLY` entry never gains one.
[Registration rewriting](reference.md#registration-rewriting) lists each case.
Shared binaries, several entries per descriptor or per binary, and resource-only registrations are all supported.
Undeclared or conflicting ownership in a selected descriptor is an error.
Templates such as `usd`'s `codegenTemplates/plugInfo.json` and ordinary resources are copied unchanged.

Python packages nest, so deploying `pxr/Usd` also needs its parent package `pxr`.
Every parent package is advertised through the same package-owner contract as any other package, with `PXR_PYTHON_PACKAGE` and `PXR_PYTHON_FILES`.
The `pxr` initializer can use a metadata-only owner, and no namespace flag or other new property is needed.
An owner whose files include `__init__.py` makes a regular parent package.
An owner without one declares an initializer-free namespace parent package.
A parent without an advertised owner is an error, never an inferred namespace.
Selecting a child package brings its advertised parent support content, not unselected sibling packages, and shared parent content is deduplicated.
Producer initializers are copied unchanged, including USD's generated `pxr/__init__.py`, and empty initializers are never synthesized.
A regular parent has exactly one authoritative provider, nonconflicting namespace contributions may coexist, and missing or conflicting ownership or content is diagnosed.
[Python parent packages](reference.md#python-parent-packages) summarizes the cases.
Only Python outputs that the producer's build enables are advertised and deployed.
The consumer remains responsible for putting its Python root on the import search path.

Overlapping selections are unioned, and identical output operations are deduplicated across calls.
Compatible descriptor and index contributions aggregate.
Incompatible content or transformations at one output path are errors, while deploying the same target into distinct layouts stays allowed.
Configuration-specific binaries and descriptors need distinct output paths, and configuration-independent content may share a path when it is identical.
Collisions are diagnosed without requiring a literal `$<CONFIG>` in destinations.

Validation happens as early as the information allows.
At configure time the functions check metadata, selections, destinations and known collisions.
This is where the [metadata validation rules](reference.md#metadata-validation-rules) are enforced for the selected targets.
Rules that need descriptor or generated-input contents not yet resolved at configure time run at install time, before copying.
At install time they check resolved inputs and configuration-dependent outputs, before copying where possible.
Installation never builds anything implicitly and never skips a missing advertised file.
A missing input fails with a message that names the target and the file and explains the required preparation.
The consumer builds what it selects, which under `EXCLUDE_FROM_ALL` means adding `add_dependencies` on the selected targets.
Producers attach generated-content prerequisites to their own targets.
[Install diagnostics](reference.md#install-diagnostics) lists every error condition.

Third-party targets are installed by the same calls once they are registered, as described in [Third-Party Plugins](#third-party-plugins).
Plugin dependency closure is a later-phase opt-in described in [Later Phase: Plugin Dependency Closure](#later-phase-plugin-dependency-closure); without it the functions deploy explicit selections only, apart from Python parent support.

The functions stop at the binaries the consumer supplies.
The consumer supplies loader-compatible binaries and layouts, because correct registry paths do not guarantee that the operating system's dynamic loader can load a binary.
The functions do not repair the RPATHs or install names of arbitrary imported binaries.
They do not scan for non-USD runtime dependencies, stage files into the build tree or generate Python bytecode.
There is no general guarantee for shared plugins loaded by a static-USD host, and no guarantee that static code is retained; [Build Modes](#build-modes) covers both.

Why no install-all mode, and no install rules when the package loads?
Loading a package must stay free of side effects in every consumption mode, and a consumer deploys only what it selects.
An inventory query keeps bulk selection to one explicit line.
Why separate category functions rather than a Python forwarding option on plugin installation?
Plugin and Python content go to independent roots with different required arguments, and a forwarding option would let one call quietly install the other category.
Why separate calls rather than a nested per-target override map?
Separate calls reuse one argument vocabulary and ordinary target lists, while a nested map would need its own syntax and parsing.
Why a generated relocatable index rather than a wildcard layout or `PXR_PLUGINPATH_NAME`?
A wildcard `Includes` finds bundles only in one fixed layout, and an environment variable must be set at every launch.
A generated index with relative `Includes` covers any chosen layout and moves with the deployment.
Why copy `pxr/__init__.py` unchanged rather than regenerate it for the selected subset?
The initializer belongs to its producer, and regenerating it would give the same package different contents in different deployments.
Why deployed descriptor copies rather than modifying producer descriptors?
Producer descriptors are shared by the build tree, the installed USD and every deployment, so each deployment writes its own copies and leaves the originals intact.

### Third-Party Plugins

A third-party plugin deploys like a USD plugin when it publishes the same deployment metadata.
The convention has three independent parts: authoring the metadata, resolving it when the package loads, and registering the targets in the inventories.
Installed discovery then works as for USD's own plugins.
The three functions involved are shipped with USD's config package.
The [reference](reference.md) lists the [producer obligations](reference.md#third-party-producer-obligations) and the [third-party functions](reference.md#third-party-functions) with their arguments.

The documented `PXR_*` property contract is sufficient on its own.
A producer that sets the properties with its own CMake code is a first-class participant, and USD requires nothing more.
The producer publishes the [deployment properties](reference.md#deployment-properties) on each deployable target.
Resources and Python files go in `PXR_RESOURCE_FILES` and `PXR_PYTHON_FILES`, with the `PXR_INSTALLED_RESOURCE_FILES` and `PXR_INSTALLED_PYTHON_FILES` siblings for export, whose sources are relative to the producer's installed prefix.
Package and extension ownership go in `PXR_PYTHON_PACKAGE` and `PXR_PYTHON_EXTENSION`.
Registration entries go in `PXR_PLUGIN_REGISTRATIONS`, and each registration owner declares the required `PXR_PLUGIN_LOAD_MODE`.
A logical owner whose code lives in another binary names it with `PXR_DEPLOYMENT_BINARY`.
A binary-less plugin or Python package uses an exported metadata-only `INTERFACE` carrier, as proposed for USD's own targets.
Generators of deployable content are attached to the deployable target with `add_dependencies`.
The export lists in `EXPORT_PROPERTIES` the properties that are exported as-is: `PXR_INSTALLED_RESOURCE_FILES`, `PXR_INSTALLED_PYTHON_FILES`, `PXR_PYTHON_PACKAGE`, `PXR_PYTHON_EXTENSION`, `PXR_PLUGIN_REGISTRATIONS`, `PXR_PLUGIN_LOAD_MODE` and `PXR_DEPLOYMENT_BINARY`.
The in-tree `PXR_RESOURCE_FILES` and `PXR_PYTHON_FILES` are not exported, because the package-load resolution below derives them from their `PXR_INSTALLED_<X>` siblings.
Exported values carry no generator expressions and no `no_subst` markers.

USD also offers an optional helper, `pxr_set_deployment_metadata`.
It sets the same properties on an existing target and validates them against the [metadata validation rules](reference.md#metadata-validation-rules).
It does not create libraries or replace the producer's build system.
It creates no install rules, exports no targets and enrolls nothing in the inventories.
Producers remain responsible for their own installation, for complete installed content and for their CMake target exports.
Installed content never depends on the producer's source or build tree.

Why not the contract alone?
The properties have several interacting rules, and a thin helper that sets and checks them reduces producer errors.
Why not a mandatory helper, or a public `pxr_plugin`?
Third-party projects build plugins in many ways today: they copy USD's macros, re-implement them or hand-roll their own install rules.
A required helper that builds or installs targets would constrain those existing build systems, while metadata on existing targets fits all of them.

An imported package must turn its exported metadata into the values consumers read.
After loading its exported targets, the package config calls `pxr_resolve_installed_metadata`.
It passes the package's relocated prefix, derived from the location of the config file, and an explicit mapping from build target names to exported target names.
The resolver resolves the sources of `PXR_INSTALLED_RESOURCE_FILES` and `PXR_INSTALLED_PYTHON_FILES` against that prefix into `PXR_RESOURCE_FILES` and `PXR_PYTHON_FILES`.
Bundle-relative and package-relative destinations stay unchanged.
It also resolves the target references in `PXR_PYTHON_EXTENSION` and `PXR_DEPLOYMENT_BINARY` through the mapping, honoring export namespaces and `EXPORT_NAME`.
This mapping is needed because CMake does not rewrite target names inside custom exported properties.
Metadata-only carriers follow the same contract.
The resolver is idempotent and creates neither install rules nor inventory enrollment.
Under this proposal, USD's package config does the same for USD's targets after it includes them.
USD exports its targets without a namespace, so its mapping is the identity.
Later-phase required-plugin references follow the same resolution; see [Later Phase: Plugin Dependency Closure](#later-phase-plugin-dependency-closure).

A reference to a target in another package names a target supplied by a normally declared package dependency.
The resolver never guesses namespaces and never discovers or loads dependency packages itself.
Producers keep this export bookkeeping in their own package-generation support, not in each consuming application.
A contract-only producer may supply equivalent package-load setup instead of calling the resolver.
Either way, consumers see resolved `PXR_<X>` values and exported target names.

Why not require identical build and import names?
Namespaces and `EXPORT_NAME` are valid and common in exports, and forbidding them would exclude many existing packages.
Why not leave reference resolution to consumers, or to a later revision?
Every consumer would have to know each producer's export naming, and an unresolved reference makes the metadata unusable after import.
Why not guess namespaces or load dependency packages implicitly?
A guess can pick the wrong target silently, and implicit package loading hides dependencies that the package config should declare.

A third-party package also publishes package-scoped role lists of its own exported targets, with resolved references.
These lists are registration inputs.
They are not snapshots of the project-wide union and do not replace the global [inventories](reference.md#inventories).
To make the targets available for selection, the consumer registers them explicitly:

```cmake
find_package(acmeUsdPlugins REQUIRED)
pxr_register_targets(TARGETS acme::usdAcme acme::acmeResources)
```

`pxr_register_targets` validates the named existing targets and their metadata, resolves aliases, and deduplicates its additions to the project-global `PXR_*_TARGETS` properties.
Roles come from the metadata and the native target kind, exactly as for USD's own targets, so there is no role marker.
Otherwise unclassified content carriers join `PXR_INSTALLABLE_TARGETS`.
A package's metadata is resolved completely before its targets are registered, and unknown targets or invalid metadata are errors.
Loading a third-party package never enrolls its targets; the consumer opts in.
A third-party subproject may make the call itself, including under `EXCLUDE_FROM_ALL`.
USD's own producer list stays distinct from the project-wide union, and `PXR_LIBRARIES` keeps its linking meaning.
Registration is independent of authoring and package-load resolution.
It does not build, install or generate install rules, it enrolls no unrelated targets, and it does not expand runtime plugin dependencies.
Consumers then query the global inventories at the selection site and pass their selections to the functions in [Consumer Install Utilities](#consumer-install-utilities).

Why not enroll targets as a side effect of authoring?
Authoring happens wherever the producer defines its targets, and inventory changes there would surprise consumers that never asked for those targets.
Why not let only consumers enroll targets?
A third-party subproject knows its own deployable targets best, and consumer-only enrollment would prevent it from registering them.
Why not enroll on package load?
A consumer may load a package only to link a library, and every loaded package would then grow the inventories that bulk selection reads.

Registration makes a plugin available for selection; it is not registration with USD's runtime plugin registry.
The producer declares its actual registration entries with `PXR_PLUGIN_REGISTRATIONS`.
The owner uses its native binary or an explicit `PXR_DEPLOYMENT_BINARY`, and declares its [plugin load mode](reference.md#plugin-load-modes).
An `INTERFACE` owner can stand for a built-in member of an aggregate, not only for a resource-only plugin.
Several entries may share one binary, and mixed load modes need separate logical owners.
From there the consumer install functions treat the plugin exactly like a USD plugin; [Consumer Install Utilities](#consumer-install-utilities) describes how they generate descriptors and discovery indexes for the selected entries.
Static and monolithic discovery follows [Build Modes](#build-modes).
This gives installed discovery without `PXR_PLUGINPATH_NAME`, subject to the runtime-placement validation of the consumer install functions.
It does not promise to repair arbitrary loader or RPATH settings.

Python follows the same explicit package and extension contract as USD's own packages.
Consumers never infer an extension from its name, a package has one owner, and registering one package does not enroll its siblings.

### Build Modes

The deployment metadata covers the four build modes USD already supports: modular shared, modular static, shared monolithic and static monolithic.
It adds no build combination and no new way to load code at run time.
The rules below are contract rules, not a claim that upstream already exports every target or property they need.
Monolithic scope is the core under `pxr/` only; plugins outside the core, such as those under `extras/`, stay separate targets.
[Build mode coverage](reference.md#build-mode-coverage) summarizes each mode.

Each mode builds a different set of binaries today.
Without shared libraries, USD skips plugins that would be separate binaries, compiled Python modules and Python tools.
Monolithic builds compile the core libraries and core plugins as per-library `OBJECT` targets and link them into the aggregate `usd_m`, whose file is named `usd_ms` when it is shared.
USD's build leaves `LibraryPath` empty for static libraries and for monolithic core members, because that code is not separately loadable.

Logical owners exist in every mode.
Every enabled, independently selectable library, plugin and content owner has an exported logical target, as [Installable Targets and Exports](#installable-targets-and-exports) describes.
In monolithic builds the existing per-library `OBJECT` targets carry the metadata, and metadata-only carriers stand in where content has no target.
Resources, registrations and Python ownership stay on the logical owners and are not duplicated on the aggregate.
The inventories list the enabled logical owners and the applicable physical library targets, as [Inventories](#inventories) describes.
Plugins and Python outputs that a mode skips are not advertised, and a category the mode does not build is an empty list.
`PXR_LIBRARIES` keeps its linking meaning, and inventory membership does not make a logical carrier something a consumer can link on its own.

A logical owner reaches its physical binary through `PXR_DEPLOYMENT_BINARY`, defined in [Deployment Metadata](#deployment-metadata).
In modular builds owners leave it unset and use their own binary, and carriers have none.
In monolithic builds each core member names the aggregate `usd_m`, and the aggregate lists no members.
A reference to a target in another package follows the existing declared-dependency rules, and resolving it does not load any package.
Selecting a member includes its physical binary, but not the resources, registrations or Python packages of other members.
Selecting the aggregate alone does not select the content of any member.
When several selections bring the same shared aggregate, even across separate calls, it is deployed once; [Consumer Install Utilities](#consumer-install-utilities) covers this deduplication.
Incompatible destinations or transformations for the same binary remain errors.
The association is neither required-plugin closure nor link-dependency scanning; required-plugin traversal is a separate later phase.
Supplying `USD_RUNTIME_TARGET` selects no binary for installation.
A member the consumer selects explicitly can still name that same target as its physical binary.
[Physical binary resolution](reference.md#physical-binary-resolution) lists each case.

Each mode implies a load mode for its registrations; [Plugin load modes](reference.md#plugin-load-modes) defines the values.
In modular shared builds, plugins and libraries with registrations have their own shared binaries and are `LOADABLE`.
In modular static builds, the registrations of static libraries are `BUILTIN`, and ordinary plugins are skipped and not advertised.
In shared monolithic builds, core members are `BUILTIN` with an empty `LibraryPath`, even though `usd_m` is a shared library.
Separately loadable non-core plugins and Python extensions keep their own binaries and associations, and those plugins are `LOADABLE`.
In static monolithic builds, core members are `BUILTIN`.
Owners without code are `RESOURCE_ONLY` in every mode.
An `INTERFACE` carrier can describe a built-in logical member when it names the aggregate explicitly with `PXR_DEPLOYMENT_BINARY`.

Static and static monolithic builds deploy runtime content only.
That content is resources, `BUILTIN` registrations with an empty `LibraryPath`, and a validated discovery anchor.
The consumer utilities do not deploy static archives.
A static monolithic member still names the static `usd_m` with `PXR_DEPLOYMENT_BINARY`, but that archive is not deployed.
Archives are link-time and SDK artifacts, and USD's own install handles them as it does today.
A built-in registration does not turn an archive into a runtime plugin payload.

Plug anchors its registration search on the directory of the binary that contains its discovery code.
When it cannot determine that binary, it falls back to the executable.
In modular shared builds that binary is the shared `plug` library, and in shared monolithic builds it is `usd_m`.
In static builds the discovery code is embedded in whatever executable or shared library links USD, so the anchor is that actual host, not always the executable.
`USD_RUNTIME_TARGET` names the binary that contains the discovery code, and `USD_RUNTIME_DESTINATION` names the directory where that binary is deployed; [Consumer Install Utilities](#consumer-install-utilities) defines both arguments.
A static archive is never an anchor.
The declared directory and the supported search layout are validated, including consistency when the same deployment also installs that binary.
These arguments alone neither select nor install the host.
Deployment-owned registration descriptors and relocatable discovery indexes are generated only for supported, validated placements.
When the placement cannot be established, deployment fails clearly instead of silently requiring `PXR_PLUGINPATH_NAME`.
[Discovery anchors](reference.md#discovery-anchors) lists the anchor for each mode.

Layout validation has a boundary.
It does not prove that the host contains or loads the required code, or that a static registration survived linking.
Consumers provide that integration, for example by whole-archive linking, and they provide binaries their loader can use.
Arbitrary linker retention, RPATH or install-name repair, and deployment of non-USD dependencies stay outside the guarantee.
The proposal makes no general portability promise for shared plugins loaded by a host that links USD statically.
This is not a universal prohibition: a configuration that a producer explicitly advertises and supports can still describe and deploy its artifacts, without the proposal certifying it at run time.

Consumers prepare their selections explicitly.
They build both the selected logical content and its physical binary, and installation never builds anything or silently skips a missing selected input.
Members get no build dependency on the aggregate, because it would form a cycle with the aggregate's dependencies on the member objects.
Prerequisites for generated content stay on the logical owners.
Python package and extension relationships are kept for the outputs the producer actually enables.
A shared aggregate does not imply ownership of any Python package, and packages and extensions that static builds skip are not synthesized.
The presence of a binary does not mean that sibling content is deployed, or that every subset a consumer selects satisfies all runtime dependencies.

Why not let the aggregate list its members?
A reverse list would be a second source of truth beside the members' own metadata, and the direct reference from each member is enough to find the binary.
Why not rewrite `LibraryPath` of core members to the shared aggregate?
The aggregate is already loaded when its code runs, and USD's build already treats monolithic core members as not separately loadable, so `BUILTIN` keeps that meaning.
Why not deploy static archives, for example with an `ARCHIVE_DESTINATION` argument?
It was considered and dropped, because archive deployment would be a new feature rather than metadata for existing behavior, and USD's own install already handles archives.
Why not assume discovery is always relative to the executable?
A static USD can be embedded in a shared library, and Plug then anchors its search on that library, so the consumer must name the actual host.

### USD's Own Installs

USD keeps its existing install rules, and so its install layout, unchanged.
Its build sets the deployment metadata alongside those rules, describing the files they install.
A CI check in USD verifies that the metadata reproduces the existing install tree, so the metadata and the install rules cannot drift apart.
Switching USD to install itself through its own metadata is not part of this proposal's acceptance; Pixar may schedule it later as cleanup.
[Rollout Plan](#rollout-plan) lists the core delivery steps.

Why not switch USD's install rules to the metadata now?
It would change a working install for every existing user at the same time as the contract is introduced, while the check already gives the same confidence that the metadata is complete.
Why not add the metadata without a check?
Nothing would then catch metadata that falls behind the install rules, and consumers would deploy an incomplete tree without any warning.

## Examples

<!-- D05: replace this line with the Examples section -->

<!-- D06: replace this line with the Alternatives Considered, Risks, Rollout Plan, Later Phase: Plugin Dependency Closure, Future Work and Out of Scope sections -->
