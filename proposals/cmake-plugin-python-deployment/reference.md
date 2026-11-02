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

Consumer-function arguments and build-mode rules are added to this file by their sections.
