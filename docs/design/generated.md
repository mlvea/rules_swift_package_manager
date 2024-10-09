This repository is a Bazel ruleset and Gazelle plugin called `rules_swift_package_manager` that aims to simplify the process of managing Swift Package Manager dependencies within a Bazel workspace. It provides tools to download, build, and consume Swift packages as first-class Bazel targets. 

Here's a breakdown of how it works:

1. **Gazelle Plugin:**  This part, written in Go, acts as a bridge between your `Package.swift` files and Bazel's build system. When you run Gazelle, it:
   - **`update-repos` mode:** Resolves your Swift package dependencies (direct and transitive) by executing Swift Package Manager commands. This generates a `Package.resolved` file (containing the exact versions of your dependencies), and a `swift_deps_index.json` file (mapping Swift modules to Bazel targets). It also creates repository rules (`swift_package` or `local_swift_package`) for each external dependency.
   - **`update` mode:** Reads the `swift_deps_index.json` file and analyzes your project's Swift source code to create Bazel build files (`BUILD.bazel`) for your project.

2. **Repository Rules (`swift_package` and `local_swift_package`):**  These are implemented in Starlark, which is Bazel's embedded language. They are used to:
   - **`swift_package`:** Download and build external Swift packages based on the information in `Package.resolved`. It also handles applying patches to packages (if configured in your `MODULE.bazel`).
   - **`local_swift_package`:**  Reference a Swift package directory on disk, similar to Bazel's `local_repository`. It generates Bazel build files for this local Swift package.

**The `swiftpkg` directory contains the Starlark code for the repository rules and their supporting modules.**  

Here's a progressive disclosure of complexity for understanding each file in the `swiftpkg` directory:

**Core Files:**

* **`defs.bzl`:** Defines the public API for the `swiftpkg` ruleset. It exports the `swift_package` and `local_swift_package` repository rules, along with helper rules for managing Swift dependency information (`swift_deps_info` and `swift_deps_index`).

* **`build_defs.bzl`:**  Exports rules that help with generating Bazel build files for specific package elements:
    - `generate_modulemap` for Objective-C modules.
    - `objc_resource_bundle_accessor` and `resource_bundle_accessor` for creating Objective-C and Swift accessors to resource bundles.
    - `resource_bundle_infoplist` for creating Info.plist files for resource bundles.

* **`internal/`:** Contains supporting modules:

    **Package Information (`pkginfos`)**

    - **`pkginfos.bzl`:** The core module for handling Swift package information. It utilizes the `swift package dump-package` and `swift package describe` JSON files to extract package metadata: 
       - **`new()`:** Creates a package information struct.
       - **`get()`:** Reads the package information from disk. 
       - **`new_dependency()`:** Creates an external dependency struct.
       - **`new_product()`:** Creates a Swift product struct.
       - **`new_target()`:** Creates a Swift target struct.

    **Repository Rules (`repo_rules`)**

    - **`repo_rules.bzl`:** Provides shared functions and definitions for repository rules. It handles common tasks such as:
       - **`check_spm_version()`:** Ensures that Swift Package Manager meets a minimum version requirement.
       - **`get_exec_env()`:** Creates a dictionary of environment variables for the repository rule's execution environments.
       - **`gen_build_files()`:** Generates the Bazel build files for the Swift package based on extracted package information. 

    **Resource Files (`resource_files`)**

    - **`resource_files.bzl`:**  Evaluates paths to determine if they are automatically discoverable resource files. It is used to identify resources like nibs, xcassets, and xcstrings.

    **Dependency Index (`deps_indexes`)**

    - **`deps_indexes.bzl`:** Resolves Swift module names to Bazel labels using the `swift_deps_index.json` file. It performs various lookups, such as finding modules by name or label, and products by identity and name.

    **Bazel Repository Names (`bazel_repo_names`)**

    - **`bazel_repo_names.bzl`:**  Generates Bazel repository names from Swift package identities. 

    **Starlark Codegen (`starlark_codegen`)**

    - **`starlark_codegen.bzl`:**  A utility module for converting various Starlark data types (lists, dicts, structs) into Starlark code, including proper indentation. 

    **Build Declarations (`build_decls`)**

    - **`build_decls.bzl`:**  Creates and manages Bazel build file declarations (`struct` values), including:
       - **`new()`:** Creates a new build file declaration.
       - **`uniq()`:** Sorts and removes duplicate declarations from a list. 

    **Clang Files (`clang_files`)**

    - **`clang_files.bzl`:**  Handles Clang files (C, C++, Objective-C), including:
       - **`is_hdr()`:**  Determines if a file is a header file (based on extensions).
       - **`is_include_hdr()`:**  Determines if a file is a public header file.
       - **`collect_files()`:**  Collects headers and sources for a Clang target.

    **Bazel Selects (`bzl_selects`)**

    - **`bzl_selects.bzl`:** Converts Swift package manager conditionals (based on platforms, configurations, etc.) into Bazel `select()` statements. It uses a `struct` to represent a conditional, including its kind, value, and condition.

    **Objective-C Files (`objc_files`)**

    - **`objc_files.bzl`:**  Manages Objective-C files, including:
       - **`has_objc_srcs()`:**  Determines if any source files are Objective-C files.
       - **`collect_src_info()`:** Collects source information for a Clang target, including imported frameworks and header files.

    **Swift Files (`swift_files`)**

    - **`swift_files.bzl`:** Handles Swift source files, including:
       - **`has_objc_directive()`:**  Checks for `@objc` directives indicating the Swift file is meant for Objective-C consumption.
       - **`has_import()`:** Determines if a Swift file imports a specific module.

    **Module Maps (`module_maps`)**

    - **`module_maps.bzl`:**  Generates Clang module map files. It uses the `write_module_map()` function to create the module map file based on provided header information.

    **ObjC Resource Bundle Accessor (`objc_resource_bundle_accessor`)**

    - **`objc_resource_bundle_accessor.bzl`:**  Generates a header file (`.h`) and implementation file (`.m`) containing a macro for accessing resource bundles from Objective-C code.

    **Resource Bundle Accessor (`resource_bundle_accessor`)**

    - **`resource_bundle_accessor.bzl`:**  Generates a Swift file containing a `Bundle.module` accessor for accessing resource bundles from Swift code.

    **Artifact Info (`artifact_infos`)**

    - **`artifact_infos.bzl`:**  Creates structs for different artifact types (frameworks, xcframeworks), including information about the link type (dynamic or static).

    **Bazel Apple Platforms (`bazel_apple_platforms`)**

    - **`bazel_apple_platforms.bzl`:**  Determines the Bazel platform conditions (config settings) that should be used for a given Apple built-in framework.

    **Swift Dependency Index (`swift_deps_index`)**

    - **`swift_deps_index.bzl`:**  Generates the `swift_deps_index.json` file.

    **Swift Dependency Info (`swift_deps_info`)**

    - **`swift_deps_info.bzl`:**  Defines the repository rule that writes the `swift_deps_index.json` file to disk.

    **Validations (`validations`)**

    - **`validations.bzl`:** Provides functions for validating argument values.

    **Modulemap Parser:** 
    
    - **`modulemap_parser/`:** Contains the code for parsing the modulemap file:
        - **`errors.bzl`:**  A module that defines an error struct, which contains an error message and a list of child errors.
        - **`tokens.bzl`:** Provides functions for creating, managing, and validating tokens (identifiers, reserved words, string literals, comments, etc.) that are parsed from the module map.
        - **`tokenizer.bzl`:**  Tokenizes the content of the modulemap file. 
        - **`declarations.bzl`:**  Defines structs for various declarations that are parsed from the module map, such as modules, extern modules, headers, export declarations, and link declarations.
        - **`parser.bzl`:**  Parses the module map tokens. 
        - **`collection_results.bzl`:**  Creates a struct that encapsulates the results of token collection.
        - **`collect_module.bzl`:**  Collects top-level module declarations.
        - **`collect_module_members.bzl`:**  Collects module members from a list of tokens. 
        - **`collect_header_declaration.bzl`:**  Collects header declarations from a list of tokens. 
        - **`collect_link_declaration.bzl`:**  Collects link declarations from a list of tokens. 
        - **`collect_umbrella_dir_declaration.bzl`:**  Collects umbrella directory declarations.
        - **`collect_unprocessed_submodule.bzl`:**  Collects the submodule tokens for later processing. 
        - **`module_declarations.bzl`:**  Handles module declarations, including functions for replacing members in a module and checking if a declaration is a module.
        - **`character_sets.bzl`:**  A module that defines various character sets used for parsing the modulemap, including whitespace characters, decimal digits, letters, operators, and hexadecimal characters.
    
    **`bzlmod/`:**
    
    - **`swift_deps.bzl`:** The main module that implements the `swift_deps` bzlmod extension.

This high-level explanation helps to understand the key components of `rules_swift_package_manager` and the structure of the `swiftpkg` directory. 

The next step is to dig deeper into specific files and understand their functions in more detail. What particular file or aspect of the `swiftpkg` directory are you interested in learning more about?
