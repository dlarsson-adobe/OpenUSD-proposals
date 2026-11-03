# Reference: Consistent Plugin and Python Deployment for Exported CMake Targets

This file is the normative companion of [README.md](README.md), which explains the reasons behind these tables.

## Installable Target Kinds

Every kind below is an installable target in every consumption mode.

| Kind | Example | Target today | Proposed target | Inventories it joins |
|---|---|---|---|---|
| Library | `usdGeom` | Exported | Exported, gains deployment metadata | `PXR_LIBRARY_TARGETS`; also `PXR_PLUGIN_TARGETS` if it has registrations and `PXR_PYTHON_PACKAGE_TARGETS` if it owns a package |
| Monolithic aggregate | `usd_m` | Exported | Exported, unchanged; members name it with `PXR_DEPLOYMENT_BINARY` | `PXR_LIBRARY_TARGETS` |
| Monolithic member | `usdGeom` in a monolithic build | Per-library `OBJECT` target | Logical owner that carries the member's metadata | Same as the library or plugin it stands for |
| Plugin | `hdStorm`, `usdAbc` | Built, not exported | Exported | `PXR_PLUGIN_TARGETS` |
| Compiled Python extension | `_<lib>`, for example `_usdGeom` | Built and installed, not exported | Exported | `PXR_PYTHON_EXTENSION_TARGETS` |
| C++ tool | `sdfdump` | Built and installed, not exported | Exported | `PXR_TOOL_TARGETS` |
| Python tool | `usdview` | Custom target `<bin>_script` only | Metadata-only carrier; its content property is left to a later revision | `PXR_TOOL_TARGETS` |
| Pure-Python package | A package with no compiled code | None | Metadata-only carrier | `PXR_PYTHON_PACKAGE_TARGETS` |
| Resource-only plugin | A plugin with only a `plugInfo.json` and resources | None | Metadata-only carrier | `PXR_PLUGIN_TARGETS` |

Every installable target is also in `PXR_INSTALLABLE_TARGETS`.
Metadata-only carriers are exported `INTERFACE` libraries.

## Deployment Properties

These target properties describe what a target needs alongside its binary.
Producers set `PXR_INSTALLED_<X>` on their targets for export.
Consumers read only the `PXR_<X>` names; in an imported package they see the values resolved from `PXR_INSTALLED_<X>` when the package loads.

| Name | Set on | Value format | Exported as | Meaning |
|---|---|---|---|---|
| `PXR_RESOURCE_FILES` | Any content owner | List of `<source>:<destination>`; absolute source; bundle-relative destination | `PXR_INSTALLED_RESOURCE_FILES` | Resource files, including registration descriptors |
| `PXR_INSTALLED_RESOURCE_FILES` | Any content owner; set by the producer for export | List of `<source>:<destination>`; source relative to the installed prefix | Itself | Exported form of `PXR_RESOURCE_FILES`; resolved into it when the package loads |
| `PXR_PYTHON_FILES` | Package owner | List of `<source>:<destination>`; absolute source; package-relative destination | `PXR_INSTALLED_PYTHON_FILES` | Python files of the package |
| `PXR_INSTALLED_PYTHON_FILES` | Package owner; set by the producer for export | List of `<source>:<destination>`; source relative to the installed prefix | Itself | Exported form of `PXR_PYTHON_FILES`; resolved into it when the package loads |
| `PXR_PYTHON_PACKAGE` | Package owner | Importable package path relative to the Python root, for example `pxr/Usd` | As-is | Identity of the package; marks the package owner |
| `PXR_PYTHON_EXTENSION` | Package owner | One target name | As-is; resolved to the exported name | Compiled extension module of the package |
| `PXR_PLUGIN_REGISTRATIONS` | Registration owner | List of `<descriptor>` or `<descriptor>:<Name>`; bundle-relative | As-is | Registration entries the owner provides; marks a plugin |
| `PXR_PLUGIN_LOAD_MODE` | Registration owner; required | `LOADABLE`, `BUILTIN` or `RESOURCE_ONLY` | As-is | How the plugin's code is loaded |
| `PXR_DEPLOYMENT_BINARY` | Logical owner; optional | One target name | As-is; resolved to the exported name | Physical binary holding the owner's code |

Exported file lists contain no `no_subst` markers and no generator expressions.

## Plugin Load Modes

`PXR_PLUGIN_LOAD_MODE` decides how consumers treat `LibraryPath` in the declared registration entries.

| Mode | Meaning | `LibraryPath` treatment |
|---|---|---|
| `LOADABLE` | Code is in a runtime-loadable shared binary | Rewritten for that binary and the chosen layout |
| `BUILTIN` | Code is already linked into or loaded by the consumer | Kept empty, even when the code is in a shared aggregate |
| `RESOURCE_ONLY` | No code binary | Never added |

A `BUILTIN` or `RESOURCE_ONLY` entry that ships a nonempty `LibraryPath` is an error.
A `LOADABLE` entry that ships an empty or missing `LibraryPath` is accepted, because consumers always replace `LibraryPath` for `LOADABLE` entries.

## Inventories

The deployment inventories are global properties that list targets only.
They are available in every consumption mode and list only targets enabled in that USD build; an absent category is an empty list.

| Name | Members | Notes |
|---|---|---|
| `PXR_LIBRARY_TARGETS` | Libraries, including enabled monolithic logical owners and the applicable physical library targets | |
| `PXR_PLUGIN_TARGETS` | Targets with `PXR_PLUGIN_REGISTRATIONS`, including resource-only carriers | |
| `PXR_TOOL_TARGETS` | Tools: C++ executable targets and Python-tool metadata-only carriers | |
| `PXR_PYTHON_PACKAGE_TARGETS` | Targets with `PXR_PYTHON_PACKAGE` | |
| `PXR_PYTHON_EXTENSION_TARGETS` | Compiled Python extension modules | |
| `PXR_INSTALLABLE_TARGETS` | Union of the role inventories plus otherwise unclassified content carriers | Each target listed once |
| `PXR_LIBRARIES` | Unchanged | Not a deployment inventory; keeps its existing linking meaning and may hold raw system library names |

Roles overlap, so a target can appear in several role inventories; consumers deduplicate their selections.
Imported inventories reference exported targets; in-tree inventories reference the build or carrier targets.
Third-party targets join only through `pxr_register_targets(TARGETS ...)`.

## Metadata Validation Rules

A violation of any rule is an error.

1. A file-list entry without both a `<source>` and a `<destination>` is malformed.
2. A destination that is absolute or uses parent traversal to escape its bundle or package root is invalid.
3. A `PXR_PYTHON_PACKAGE` value that is absolute or contains parent traversal is invalid.
4. A registration owner without `PXR_PLUGIN_LOAD_MODE`, or with a value other than `LOADABLE`, `BUILTIN` or `RESOURCE_ONLY`, is invalid.
5. A `LOADABLE` owner without a runtime-loadable shared binary, of its own or through `PXR_DEPLOYMENT_BINARY`, is invalid.
6. A `RESOURCE_ONLY` owner that has a native code binary of its own, or that names a `PXR_DEPLOYMENT_BINARY`, is contradictory.
7. A `BUILTIN` or `RESOURCE_ONLY` registration entry with a nonempty `LibraryPath` is contradictory; a `LOADABLE` entry with an empty or missing `LibraryPath` is accepted.
8. A registration whose descriptor is not among the owner's resource destinations, or whose `<Name>` is not a `Plugins` entry in that descriptor, is invalid.
9. A registration entry claimed by more than one owner is contradictory.
10. A `PXR_DEPLOYMENT_BINARY` or `PXR_PYTHON_EXTENSION` alias is resolved to the target it names before the other rules apply.
11. A `PXR_DEPLOYMENT_BINARY` or `PXR_PYTHON_EXTENSION` that names a missing target is invalid.
12. A `PXR_DEPLOYMENT_BINARY` that names a target without a binary, such as an `INTERFACE` carrier, is invalid.
13. A `PXR_DEPLOYMENT_BINARY` that names the owner itself or forms a cycle is invalid.
14. A `PXR_DEPLOYMENT_BINARY` that names a target which itself has `PXR_DEPLOYMENT_BINARY` is an indirect chain and is invalid.
15. An inventory member that is not a known target is invalid.

## Install Functions

`pxrInstall.cmake` provides these consumer utilities, and each processes only the targets passed in `TARGETS`.

| Function | Processes | Accepts |
|---|---|---|
| `pxr_install_targets` | Every advertised category on the selected targets: runtime binaries, resources, plugin registrations and Python content, including content on otherwise unclassified metadata-only carriers | Every argument in [Install Arguments](#install-arguments) |
| `pxr_install_plugins` | Selected targets with plugin registrations: their runtime binaries, resources and registrations; other targets contribute nothing | `TARGETS`, `DESTINATION`, `RUNTIME_DESTINATION`, `LIBRARY_DESTINATION`, `RESOURCE_DESTINATION`, `USD_RUNTIME_TARGET`, `USD_RUNTIME_DESTINATION` |
| `pxr_install_python_packages` | Selected package owners: their Python files and compiled extension modules, plus advertised parent support content; other targets contribute nothing | `TARGETS`, `DESTINATION`, `PYTHON_DESTINATION` |

## Install Arguments

Each argument means the same in every function that accepts it.

| Argument | Meaning | Default | Required when | Accepted by |
|---|---|---|---|---|
| `TARGETS` | Targets to process | None | Always | All three functions |
| `DESTINATION` | Deployment root, relative to the install prefix | `.` | Never | All three functions |
| `RUNTIME_DESTINATION` | Directory for runtime binaries, including Windows runtime DLLs, relative to the deployment root | `bin` | Never | `pxr_install_targets`, `pxr_install_plugins` |
| `LIBRARY_DESTINATION` | Directory for shared libraries, relative to the deployment root | The GNUInstallDirs library directory | Never | `pxr_install_targets`, `pxr_install_plugins` |
| `RESOURCE_DESTINATION` | Exact resource bundle root, relative to the deployment root; no target-name suffix is added | Per target, `usd/<name>` under the library directory, where `<name>` is the target's unqualified name | Never | `pxr_install_targets`, `pxr_install_plugins` |
| `PYTHON_DESTINATION` | Python root, relative to the deployment root; extension modules go in their package beneath it | None | Python content is selected | `pxr_install_targets`, `pxr_install_python_packages` |
| `USD_RUNTIME_TARGET` | Target whose binary contains Plug discovery code; never an archive; selects nothing for installation | None | Plugin registrations are selected | `pxr_install_targets`, `pxr_install_plugins` |
| `USD_RUNTIME_DESTINATION` | Deployed directory of that binary, relative to the deployment root | None | Plugin registrations are selected | `pxr_install_targets`, `pxr_install_plugins` |

An unknown keyword, or an argument the function does not accept, is an error, and every destination, defaults included, must stay beneath the deployment root.

## Registration Rewriting

Deployed descriptor copies treat each selected registration entry according to its [load mode](#plugin-load-modes).

| Aspect | `LOADABLE` | `BUILTIN` | `RESOURCE_ONLY` |
|---|---|---|---|
| `LibraryPath` | Rewritten for the selected binary and the chosen layout | Kept empty | Never added |
| `Root` and `ResourcePath` | Rewritten for the deployed bundle and layout | Rewritten for the deployed bundle and layout | Rewritten for the deployed bundle and layout |
| Other fields of retained entries | Preserved | Preserved | Preserved |
| Unselected entries in a selected descriptor | Omitted from the deployed copy | Omitted from the deployed copy | Omitted from the deployed copy |
| Descriptor-level fields | Preserved | Preserved | Preserved |

Templates and ordinary resources are copied unchanged.
Producer descriptors are never modified.
The discovery index is deployment-owned, lists the installed descriptors with relative `Includes`, and sits at a location USD's plugin registry searches relative to `USD_RUNTIME_DESTINATION`.

## Python Parent Packages

A parent package of a selected package is deployed according to its advertised owner.

| Case | Meaning | Rule |
|---|---|---|
| Advertised owner with `__init__.py` | Regular parent package | Exactly one authoritative provider; its initializer is copied unchanged |
| Advertised owner without `__init__.py` | Initializer-free namespace parent package | Nonconflicting contributions may coexist; no initializer is synthesized |
| Parent with no advertised owner | Missing ownership | Error; never inferred as a namespace |
| Selected child package | Package owner passed in `TARGETS` | Brings its advertised parent support content but not unselected siblings; shared parent content is deduplicated |

## Install Diagnostics

Every condition below is an error, detected as early as the information allows.

| Condition | Detected at | Result |
|---|---|---|
| Unknown keyword | Configure time | Error |
| Argument the function does not accept | Configure time | Error |
| Unknown or non-installable target | Configure time | Error |
| Metadata that breaks a [metadata validation rule](#metadata-validation-rules) | Configure time; install time, before copying, for rules that need descriptor or generated-input contents not yet resolved at configure time | Error |
| Absolute or escaping destination, including defaults | Configure time | Error |
| Python content selected without `PYTHON_DESTINATION` | Configure time | Error |
| Plugin registrations selected without `USD_RUNTIME_TARGET` or `USD_RUNTIME_DESTINATION` | Configure time | Error |
| Runtime target that is an archive or, as far as CMake can tell, does not contain Plug discovery code | Configure time | Error |
| Runtime placement inconsistent with the same deployment's install of that target, or discovery that cannot be validated | Configure time | Error; `PXR_PLUGINPATH_NAME` is not assumed |
| Index location already holding an unrelated index | Configure time when known, otherwise install time | Error |
| Undeclared or conflicting registration ownership in a selected descriptor | Configure time when the descriptor contents are resolved, otherwise install time, before copying | Error |
| Parent package without an advertised owner | Configure time | Error |
| Conflicting regular parent providers or namespace content | Configure time when known, otherwise install time | Error |
| Incompatible content or transformations at one output path, including configuration collisions and default bundle-name collisions | Configure time when known, otherwise install time | Error |
| Missing input file | Install time | Error; the message names the target and the file and the required preparation, such as building the target or adding `add_dependencies` under `EXCLUDE_FROM_ALL` |

<!-- D03: replace this line with the third-party plugin function tables -->

<!-- D04: replace this line with the build mode tables -->
