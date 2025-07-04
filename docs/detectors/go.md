# Go Detection

## Requirements

Go detection runs when one of the following files is found in the project:

-   `go.mod` or `go.sum`

## Detection strategy

### go.mod parsing
- All go.mod are parsed to detect dependencies. This parsing doesn't depend on presence of `go cli`.

### go cli (go list) or go.sum parsing
- If a `go.sum` file is found, detector first checks if go version in the adjacent `go.mod` ≥`1.17`. If it is `≥1.17`, the file is skipped. If it is `< 1.17`, the detector proceeds as follows. Read [Go Module Changes in Go 1.17](#go-module-changes-in-go-117) to understand why `1.17` is relevant.
- If `go cli` is found and not [disabled](#environment-variables), `go list` command is preferred over parsing `go.sum` file since `go.sum` files contains history of dependencies and including these dependencies can lead to [over-reporting](#known-limitations).
- If `go list` was not used or did not run successfully, detector falls back to parsing `go.sum` manually.

### Dependency graph generation
Full dependency graph generation is supported if Go v1.11+ is present
on the build agent. To generate the graph, the command
[go mod graph][2] is executed. This only adds edges between the components
that were already registered.

## Default Detection Strategy

The Go detector’s default behavior is optimized to reduce over-reporting by leveraging improvements introduced in Go 1.17.

- When a go.mod file declares a Go version ≥ 1.17, the detector analyzes only the go.mod file to determine dependencies.
- If the go.mod file specifies a Go version < 1.17, the detector uses a fallback strategy to ensure coverage.
Read more about this in the [Fallback Detection Strategy](#fallback-detection-strategy)

## Fallback Detection Strategy

The fallback detection strategy is used when the default strategy (based on `go.mod` files with `Go 1.17` or later) cannot be applied. 
In this mode, the detector uses `Go CLI` or manually parses `go.sum` to resolve dependencies. This strategy is known to overreport (see the [known limitations](#known-limitations)). Read through the [troubleshooting-section](#troubleshooting-failures-to-run-the-default-go-detection-strategy) for tips on how to ensure that the newer, more accurate default detection strategy runs successfully.

To force the fallback detection strategy, set the environment variable: `DisableGoCliScan=true`

### `go.mod` before go 1.17

Go detection is performed by parsing any `go.mod` files, and either invoking the `Go CLI` or manually parsing `go.sum` files found under the scan directory.

Only root dependency information is generated in the fallback detection
strategy. The full graph is not detected.

### `go.mod` after 1.17

Go detection is performed by only scanning the `go.mod` files. This
reduces over reporting dependencies. The `go.mod` file contains all
dependencies, including transitive ones. [<sup>3</sup>][3]

Similarly, no graph is generated.

## Troubleshooting failures to run the default Go detection strategy

The fallback detection strategy is known to overreport by nature of
parsing `go.sum` files which contain historical component information.

If you are experiencing overdetection from `go.sum` files or have
otherwise been made aware that you are using the fallback Go detection
strategy, search Component Detection output for the following to determine
what action is needed:

| CD output                                                          | Solution                                                                                                |
| ------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------- |
| `Go CLI was not found in the system`                               | [Ensure that Go v1.17+ is installed](#ensure-that-go-v117-is-installed-on-the-build-agent)              |
| `#[error]Go CLI command "go list -m -json all" failed with error:` | [Resolve `go list` errors](#resolve-go-list-errors)                                                     |
| `Go cli scan was manually disabled, fallback strategy performed.`  | [Remove the `DisableGoCliScan` environment variable](#remove-the-disablegocliscan-environment-variable) |
| _Timeouts in Go detection_                                         | [Fetch Go modules](#fetch-go-modules-before-the-component-detection-build-task-runs)                    |

If you do not believe that Go is used in your project but Go detection is running,
it is likely that you need to [clean up extraneous build files](#clean-up-extraneous-build-files).

### Ensure that Go v1.17+ is installed on the build agent

Install Go CLI tools v1.17+ and ensure that it is available at the
Component Detection step.

If Go CLI tools are installed in a prior build step but
Component Detection is not finding them, ensure that they
are not cleaned up or installed inside a container.

### Resolve `go list` errors.

Errors are logged in the Component Detection build task output and begin with
`#[error]Go CLI command "go list -m -json all" failed with error:`. These errors
are typically caused by version resolution problems or incorrectly formatted `go.mod`
files.

### Remove the `DisableGoCliScan` environment variable

The variable should not be set or should be set to `false`.

### Fetch Go modules before the Component Detection build task runs.

If modules are not fetched, `go list` will pull the modules and may
negatively impact performance at detection time.

If you are still experiencing timouts, the fallback strategy might be more
appropriate for your project:

1. Set `DisableGoCli=true`.
1. Run `go mod tidy` to clean your `go.mod` and `go.sum`s to reduce overreporting.
1. Install Go v1.17+ to bypass go.sum scanning.

### Clean up extraneous build files

Go detection runs when a `go.sum` or `go.mod` file is encountered. If
you do not use Go in your project, search for the following in Component
Detection build output to find the paths to the Go files CD has detected.
Component Detection must be running in debug mode to see these logs:

`##[debug]Found Go.mod:` or  
`##[debug]Found Go.sum:`

When you have found the affected files, you may:

1. Delete the files if they exist but are not necessary in your project.
2. Add a clean-up step to your build to remove the files if they are
   generated prior to the Component Detection build step but are not related
   to your project.
3. [Exclude the directory](../detector-arguments.md) if it should not be scanned by Component
   Detection.

## Known limitations

-   If the default strategy is used and go modules are not present in
    the system before the detector is executed, the go cli will fetch all
    modules to generate the dependency graph. This will incur additional
    detector time execution.

-   Dev dependency tagging is not supported.

-   Go detection will fallback if no Go v1.11+ is present.

-   (Prior to Go 1.17) Due to the nature of `go.sum` containing
    references for all dependencies, including historical,
    no-longer-needed dependencies; the fallback strategy can result in
    over detection. Executing [go mod
    tidy](https://go.dev/ref/mod#go-mod-tidy) before detection via the
    fallback strategy is encouraged.

-   Some legacy dependencies may report stale transitive dependencies in
    their manifests, in this case you can remove them safely from your
    binaries by using [exclude
    directive](https://go.dev/doc/modules/gomod-ref#exclude).

## Environment Variables

If the environment variable `DisableGoCliScan` is set to `true`, the
Go detector forcibly executes the [fallback strategy](#fallback-detection-strategy).

## Go Overview

### `go.mod` and `go.sum` Files

In the Go programming language, `go.mod` and `go.sum` files play a
vital role in managing dependencies and ensuring the reproducibility
and security of a Go project. These files are central to Go's module
system, introduced in Go version 1.11, which revolutionized how Go
manages external packages and dependencies.

### `go.mod` File

The `go.mod` file, short for "module file," is a fundamental component
of Go's module system.[<sup>4</sup>][4] It serves several crucial purposes:

1. **Module Definition**: The `go.mod` file defines the module name,
   which uniquely identifies the project. The module name typically
   follows the format of a version control repository URL or a custom
   path, such as `example.com/myproject`.

2. **Dependency Declaration**: Inside the `go.mod` file, you declare
   the specific versions of dependencies your project relies on.
   Dependencies are listed with their module paths and version
   constraints.

    ```
    go module example.com/myproject

    go 1.17

    require (
         github.com/somepackage v1.2.3 golang.org/x/someotherpackage v0.4.0
    )

    ```

    Here, `github.com/somepackage` and `golang.org/x/someotherpackage`
    are declared as project dependencies with specific version
    constraints.

3. **Semantic Versioning**: Go uses Semantic Versioning (Semver) to
   specify version constraints for dependencies. You can specify
   constraints such as `v1.2.3` (exact version) or `>=1.2.0, <2.0.0`
   (range of versions).

4. **Dependency Resolution**: When you build your project or import
   new dependencies, Go uses the `go.mod` file to resolve and download
   the exact versions of dependencies that satisfy the specified
   constraints.

5. **Dependency Graph**: The `go.mod` file implicitly constructs a
   dependency graph of your project's dependencies, allowing Go to
   ensure that all dependencies are compatible and can be built together.

### `go.sum` File

The `go.sum` file, short for "checksum file," is used for ensuring the
integrity and security of dependencies. It contains checksums
(cryptographic hashes) of specific versions of packages listed in the
`go.mod` file.[<sup>5</sup>][5] The `go.sum` file serves the following purposes:

1. **Cryptographic Verification**: When Go downloads a package
   specified in the `go.mod` file, it verifies the downloaded
   package's integrity by comparing its checksum with the checksum
   recorded in the `go.sum` file. If they don't match, it signals a
   potential security breach or data corruption.

2. **Dependency Pinning**: The `go.sum` file pins the exact versions
   of dependencies used in the project. It ensures that the same
   package versions are consistently used across different builds and
   development environments, which aids in reproducibility.

3. **Security**: By including checksums, the `go.sum` file helps
   protect against tampering with packages during transit or in case
   of compromised repositories. It adds a layer of trust in the packages
   being used.

Here's a simplified example of entries in a `go.sum` file:

```
github.com/somepackage v1.2.3 h1:jh2u3r9z0wokljwesdczryhtnu1xf6wl4h7h2us9rj0=
github.com/anotherpackage v0.4.0 h1:rn2iw0z7liy6d87dwygfawxqvx86jxd4m8hkw6yaj88=
```

Each line contains the package path, version, and a cryptographic hash
of the package contents.

#### Relevance in Dependency Scanning

1. **Dependency Resolution**: Dependency scanners use the information
   in `go.mod` to understand which packages and versions a Go project
   depends on.

2. **Security and Trust**: The `go.sum` file ensures that dependencies
   are downloaded securely and have not been tampered with during
   transit.

3. **Build Reproducibility**: `go.mod` and `go.sum` files contribute
   to build reproducibility by pinning exact versions of dependencies,
   making it possible to recreate the same build environment
   consistently.

### Go Module Changes in Go 1.17

Prior to Go 1.17, the `go.mod` file primarily contained information
about direct dependencies, but it didn't include information about
transitive (indirect) dependencies. This made it challenging to
accurately detect and manage all dependencies in a project.

In Go 1.17 and later, Go introduced an important change: the `go.mod`
file now includes information about both direct and transitive
dependencies. This improvement enhances the clarity and completeness
of dependency information within the `go.mod` file.

The completeness of `go.mod` file in `≥1.17` allows the detector to skip `go.sum` files entirely. 

#### Relevance of the Go Version Check

1. **Accuracy of Dependency Detection**: Checking the Go version in
   the `go.mod` file allows the Go Component Detector to determine
   whether the project is using the enhanced module system introduced in
   Go 1.17. If the Go version is 1.17 or higher, it indicates that the
   `go.mod` file contains information about transitive dependencies.
   Processing this updated `go.mod` file provides a more accurate and
   comprehensive view of the project's dependencies.

2. **Avoiding Over-Reporting**: In projects using Go 1.17 and later,
   transitive dependencies are already listed in the `go.mod` file,
   and processing the corresponding `go.sum` file could lead to
   over-reporting components. By not processing the `go.sum` file when
   it's not necessary (i.e., when the `go.mod` file includes transitive
   dependencies), the detector avoids redundant or incorrect component
   detection.

3. **Minimizing Noise**: Over-reporting components can result in
   unnecessary noise in the scan results.

## Detection Validation Strategy

The changes in the Go detector were validated by:

1. **Unit tests**: Sample `go.mod` and `go.sum` files were created
   and placed as unit tests. Other unit tests for go version less than
   1.17 were still maintained to ensure there were no regressions.

2. **Local testing**: Real `go.mod` and `go.sum` from the go CLI
   were created from a real test codebase and verified manually.

The main change for the go detector was not in the parsing of the
go.mod files, but rather simply filtering `go.sum` files if an
adjacent `go.mod` file specified a version higher than 1.17.

[1]: https://go.dev/ref/mod#go-list-m
[2]: https://go.dev/ref/mod#go-mod-graph
[3]: https://go.dev/doc/modules/gomod-ref#go-notes
[4]: https://go.dev/doc/modules/gomod-ref
[5]: https://go.dev/ref/mod#go-sum-files
