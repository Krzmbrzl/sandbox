# C++ modules

This is a (simple) test example of how C++20's modules work. This requires a compiler that actually supports modules and since we're using CMake as a
build system, the used compilers also have to have support for [p1689](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p1689r5.html)
(see [this blog post](https://www.kitware.com/import-cmake-the-experiment-is-over/)). The following compilers are known to have the necessary support:
- MSVC 14.34 or newer with Visual Studio 2022 or newer
- GCC 14 or newer
- Clang 16 or newer

Additionally, CMake 3.28 or newer is required for the module support to be available (in a non-experimental state).


## Terminology

There is a hand full of terminology to know when working with modules. All of that is rather well explained [on
cppreference](https://en.cppreference.com/w/cpp/language/modules) but this is my short summary:
- **Module declaration**: Everything that involves the `module` keyword. The module declaration must always be the first declaration in the
  translation unit.
- **Module unit**: Every translation unit that contains a module declaration
- **Global module fragment**: Section between a plain `module;` and a mandatory subsequent module declaration marking the end of the fragment. If
  present, the global module fragment must always be the first thing in a translation unit and it may only contain preprocessor statements. It is also
  the only part in a module unit where `#include` may be used.
- **Private module fragment**: Everything after `module : private;`. All contents inside the private fragment are unreachable from importing
  translation units.
- **Module partition**: Module unit whose name contains a colon. The syntax is `module <moduleName>:<partitionName>`. That is, a module partition
  `<partitionName>` always belongs to a given module `<moduleName>`.
- **Named module**: Collection of module units with the same module name
- **Module interface unit**: Module units that use the `export` keyword in their module declaration, e.g. `export module Dummy`
- **Module implementation unit**: All module units that are not module interface units
- **Primary module interface unit**: A module interface unit that doesn't specify a module partition. Every named module must have exactly one primary
  module interface unit.


## How things work together

Every named module must have exactly one primary interface unit. Its exported declarations is what an importer of that module will have access to.

It is possible to combine function declaration and definition by having the declaration be part of the module interface unit. This is then
semantically similar to having the function definition inside a header file (minus the potential ODR-violation issues). That is, any change to the
definition leads to a recompilation of all importing code. Instead, it is usually desirable to split declaration and definition by having a module
interface unit that only contains the declarations and an associated module implementation unit that contains the definitions. This way, changes in
the definition does not require recompilation of importing code.

If separate files are not desired, a private module fragment can be used instead. Having function definitions in the private module fragment is
semantically equivalent to having a separate module implementation unit. Note however, that only a primary module interface unit can have a
private module fragment and in this case, there can't be any other module units within that named module. So a private module fragment is really meant
to be used only if you want to implement your module entirely within a single file.

Module implementation units implicitly import all declarations from the associated module interface unit (whether or not they are exported). They
themselves can't export any declarations.


## Current limitations

The implementation status of modules in different compilers still varies and is generally not yet complete (as of beginning 2025):
- [GCC module status](https://gcc.gnu.org/onlinedocs/gcc/C_002b_002b-Modules.html):
    - Private module fragments are not yet supported
    - Definitions in module partitions leak into different named modules (or non-module code)
    - Standard library headers can't imported (have to be included)
    - Module support must be explicitly enabled via the `-fmodules-ts` compiler option
- [Clang module status](https://clang.llvm.org/docs/StandardCPlusPlusModules.html#known-issues):
    - Importable module files have to have file extension `.cppm`, `.ccm`, `.cxxm` or `c++m`
- MSVC:
    - Module interface units should have file extension `.ixx`
- Apple Clang: No module support whatsoever

Additionally, CMake also doesn't support modules for all generators. Most notably, the `Unix Makefiles` generator **does not support modules**.
Instead, only the `Ninja` and `Visual Studio` generators are supported at the moment. In order for Clang to work with cmake, `clang-scan-deps` needs
to be installed separately. This binary either needs to be in `PATH` or one has to set the `CMAKE_CXX_COMPILER_CLANG_SCAN_DEPS` CMake variable to the
path of the respective binary. On Ubuntu (Debian) this is part of the `clang-tools` package.


## Tooling support

### Clangd

[Clangd](https://clangd.llvm.org/) does not yet support modules officially (see [feature request](https://github.com/clangd/clangd/issues/1293)).
However, there exists experimental module support in Clangd v19 that can be enabled via the `--experimental-modules-support` CLI flag.
