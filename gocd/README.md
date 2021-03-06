gocd
----

`gocd` is a Go utility that makes it easy to track, review and enforce the imports used by Go packages. It ensures that
the organization of imports in a Go project is handled meticulously and can be kept perfect.

## Usage

### Generate gocd_imports.json file

Run `./gocd [dir]`, where `dir` the root directory of a Go package hierarchy. This generates a `gocd_imports.json` file
in the specified directory that contains information about of the imports in the Go files in the directory structure.

### Verify that a package hierarchy conforms to its imports file

Run `./gocd --verify [dir]` to verify that the specified directory contains a `gocd_imports.json` file and that the
content of that file matches the content that would be generated by running `./gocd [dir]`. If this is not the case, an
error is printed and the program returns with a non-zero exit code. This can be used in CI environments.

### Reviewing the file

Here is example output:

```json
{
    "imports": [
        {
            "path": "github.com/palantir/checks/vendor/github.com/palantir/pkg/cli",
            "numGoFiles": 17,
            "numImportedGoFiles": 17,
            "importedFrom": [
                "github.com/palantir/checks/gocd/cmd",
                "github.com/palantir/checks/gocd/cmd/gocd"
            ]
        },
        {
            "path": "github.com/palantir/checks/vendor/github.com/palantir/pkg/cli/cfgcli",
            "numGoFiles": 1,
            "numImportedGoFiles": 34,
            "importedFrom": [
                "github.com/palantir/checks/gocd/cmd",
                "github.com/palantir/checks/gocd/cmd/gocd"
            ]
        }
    ],
    "mainOnlyImports": [],
    "testOnlyImports": [
        {
            "path": "github.com/palantir/checks/vendor/github.com/stretchr/testify/assert",
            "numGoFiles": 6,
            "numImportedGoFiles": 9,
            "importedFrom": [
                "github.com/palantir/checks/gocd_test"
            ]
        }
    ]
}
```

The output has 3 sections: `imports`, `mainOnlyImports` and `testOnlyImports`.

* `imports` lists the packages that are imported by the non-main, non-test packages
* `mainOnlyImports` lists the packages that are imported by main packages, but are not imported by non-main or non-test
  packages. Represents imports that are present only because of main packages.
* `testOnlyimports` lists the packages that are imported by test packages, but are not imported by non-main or
  main packages. Represents imports that are present only because of test packages.

`numGoFiles` is the total number of `.go` files in the package itself and `numImportedGoFiles` is the number of
additional `.go` files imported by the package. The total of these two values is the total number of `.go` files for
this import (including transitive dependencies), and is a good indicator of the size/scope of the import.

Note that the count of different top-level packages may include the same libraries, so the total number of Go files
required is not necessarily the sum of all of the values. In the example above, `go-palantir/cli` imports
`go-palantir/cli/flag`, so the 17 `numImportedGoFiles` include the 10 files reported by `go-palantir/cli/flag`. Because
of this, the total of all of the counts is an upper bound rather than an exact count. The count for a specific package
does not double-count dependencies.

`importedFrom` lists the packages in the project that import the package directly.

## Motivation

The Go language has a very simple and well-defined import mechanism. However, this mechanism can sometimes work against
encapsulation. A single Go package may have a huge dependency tree (that is, importing that one package may necessitate
importing many more), but it is hard to discern the scope of the imports without manually examining the imports.

This can pose a challenge when trying to determine whether or not it's reasonable to add a new dependency. A project
maintainer may be okay with adding a package that consists of a few Go files that imports only the standard library as a
dependency, but not okay with adding a dependency on a package that transitively imports 20 other packages.

This scenario is especially challenging for "pkg"-style projects that consist of many independent packages that are
meant to have a small footprint. There may be some packages that must have a large import footprint because they depend
on third-party packages, but others that are meant to be light-weight. Currently, it is hard for an author or reviewer
on such a project to properly keep track of the scope of dependencies.

`gocd` provides tooling that allows projects to track the imports of their Go packages. More importantly, it provides an
enforcement mechanism that allows maintainers to verify the changes that they care about and track package dependencies
in a sane manner. It reduces the chance of inadvertently introducing large dependencies into packages by writing the
most important aspects of the change to a separate file that can be easily reviewed.