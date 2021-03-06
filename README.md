# Go rules for [Bazel](https://bazel.build/)

Bazel 0.5.2 | Bazel HEAD
:---: | :---:
[![Build Status](https://travis-ci.org/bazelbuild/rules_go.svg?branch=master)](https://travis-ci.org/bazelbuild/rules_go) | [![Build Status](http://ci.bazel.io/buildStatus/icon?job=PR/rules_go)](http://ci.bazel.io/view/Bazel%20bootstrap%20and%20maintenance/job/PR/job/rules_go/)

## Announcements

* **August 28, 2017** Release
[0.5.4](https://github.com/bazelbuild/rules_go/releases/tag/0.5.4) is
now available!  This will be the last stable tag before requiring Bazel 0.5.4 and toolchains support.
* **August 9, 2017** Release
[0.5.3](https://github.com/bazelbuild/rules_go/releases/tag/0.5.3) is
now available!
* **July 27, 2017** Bazel 0.5.3 is now available. This includes a change which
is incompatible with rules\_go 0.5.1 and earlier. rules\_go 0.5.2 should work.
* **July 17, 2017** Release
[0.5.2](https://github.com/bazelbuild/rules_go/releases/tag/0.5.2) is now
available! This fixes an issue with Bazel at HEAD. Note that Bazel 0.5.2 is
now required.
* **July 12, 2017** The rules now require Bazel 0.5.2 or newer at HEAD. The
latest tagged version, 0.5.1, still works with Bazel 0.4.5 though.

## Contents

* [Overview](#overview)
* [Setup](#setup)
* [Generating build files](#generating-build-files)
* [Build modes](#build-modes)
* [FAQ](#faq)
* [Repository rules](#repository-rules)
  * [go_rules_dependencies](#go_rules_dependencies)
  * [go_register_toolchains](#go_register_toolchains)
  * [go_repository](#go_repository)
  * [new_go_repository](#new_go_repository)
* [Build rules](#build-rules)
  * [go_prefix](#go_prefix)
  * [go_library](#go_library)
  * [cgo_library](#cgo_library)
  * [go_binary](#go_binary)
  * [go_test](#go_test)
  * [go_proto_library](#go_proto_library)
  * [go_embed_data](#go_embed_data)

## Overview

The rules are in the alpha stage of development. They support:

* libraries
* binaries
* tests
* vendoring
* cgo
* auto generating BUILD files via [gazelle](go/tools/gazelle/README.md)
* protocol buffers (via extension //proto:go_proto_library.bzl)

They currently do not support (in order of importance):

* cross compilation
* bazel-style auto generating BUILD (where the library name is other than
  go_default_library)
* C/C++ interoperation except cgo (swig etc.)
* coverage
* test sharding

**Note:** The latest version of these rules (0.5.4) require Bazel ≥ 0.5.2 to
  work.

The `master` branch is only guaranteed to work with the latest version of Bazel.

## Setup

* Decide on the name of your package, eg. `github.com/joe/project`. It's
  important to choose a name that will match where others will download your
  code. This will be a prefix for import paths within your project.
* Create a file at the top of your repository named `WORKSPACE`, and add the
  following code, verbatim. This will let Bazel fetch necessary dependencies
  from this repository and a few others. You can add more external dependencies
  to this file later (see [go_repository](#go_repository) below).
  If you're using the latest stable release you can use the following contents:

    ```bzl
    git_repository(
        name = "io_bazel_rules_go",
        remote = "https://github.com/bazelbuild/rules_go.git",
        tag = "0.5.4",
    )
    load("@io_bazel_rules_go//go:def.bzl", "go_repositories")

    go_repositories()
    ```

  If you're using rules_go at or near the HEAD of master, you can use the
  following contents (optionally replacing the commit with something newer):

    ```bzl
    git_repository(
        name = "io_bazel_rules_go",
        remote = "https://github.com/bazelbuild/rules_go.git",
        commit = "d8d73c918ed7b59a5584e0cab4f5274d2f91faab",
    )
    load("@io_bazel_rules_go//go:def.bzl", "go_rules_dependencies", "go_register_toolchains")

    go_rules_dependencies()
    go_register_toolchains()
    ```

* Add a `BUILD` file to the top of your project. Declare the name of your
  workspace using `go_prefix`. This is used by Bazel to translate between build
  targets and import paths. Also add the gazelle rule.

    ```bzl
    load("@io_bazel_rules_go//go:def.bzl", "go_prefix", "gazelle")

    go_prefix("github.com/joe/project")
    gazelle(name = "gazelle")
    ```

* If your project can be built with `go build`, you can
  [generate your `BUILD` files](#generating-build-files) using Gazelle. If
  not, or if you just want to understand the things gazelle is going to
  generate for you, read on.

* For a library `github.com/joe/project/lib`, create `lib/BUILD`, containing
  a single library with the special name "`go_default_library`." Using this name
  tells Bazel to set up the files so it can be imported in .go files as (in this
  example) `github.com/joe/project/lib`. See the
  [FAQ](#whats-up-with-the-go_default_library-name) below for more information
  on this name.

    ```bzl
    load("@io_bazel_rules_go//go:def.bzl", "go_library")

    go_library(
        name = "go_default_library",
        srcs = ["file.go"]
    )
    ```

* Inside your project, you can use this library by declaring a dependency on
  the full Bazel name (including `:go_default_library`), and in the .go files,
  import it as shown above.

    ```bzl
    go_binary(
        ...
        deps = ["//lib:go_default_library"]
    )
    ```

* To declare a test,

    ```bzl
    go_test(
        name = "mytest",
        srcs = ["file_test.go"],
        library = ":go_default_library"
    )
    ```

* For instructions on how to depend on external libraries,
  see [Vendoring.md](Vendoring.md).

## Generating build files

If your project can be built with `go build`, you can generate and update
your `BUILD` files automatically using Gazelle, a tool included in this
repository. See the [Gazelle README](go/tools/gazelle/README.md)
for more information.

The `gazelle` rule in your root BUILD file gives you the ability to build and
run gazelle on your project using Bazel. This is the preferred way to run
Gazelle.

```
bazel run //:gazelle
```

By default, Gazelle assumes external dependencies are present in your
`WORKSPACE` file, following a certain naming convention. For example, it expects
the repository for `github.com/jane/utils` to be named
`@com_github_jane_utils`. If you prefer to use vendoring, add
`external="vendored"` to the `gazelle` rule. See [Vendoring.md](Vendoring.md).

## Build modes

### Building static binaries

You can build binaries in static linking mode using
```
bazel build --output_groups=static //:my_binary
```

You can depend on static binaries (e.g., for packaging) using `filegroup`:

```bzl
go_binary(
    name = "foo",
    srcs = ["foo.go"],
)

filegroup(
    name = "foo_static",
    srcs = [":foo"],
    output_group = "static",
)
```

### Using the race detector

You can run tests with the race detector enabled using
```
bazel test --features=race //...
```

You can build binaries with the race detector enabled using
```
bazel build --output_groups=race //...
```

The difference is necessary because the rules for binaries can produce both
race and non-race versions, but tools used during the build should always be
built in the non-race configuration. `--output_groups` is needed to select
the configuration of the final binary only. For tests, only one executable
can be tested, and `--features` is needed to select the race configuration.

## FAQ

### Can I still use the `go` tool?

Yes, this setup was deliberately chosen to be compatible with the `go`
tool. Make sure your workspace appears under

```sh
$GOPATH/src/github.com/joe/project/
```

eg.

```sh
mkdir -p $GOPATH/src/github.com/joe/
ln -s my/bazel/workspace $GOPATH/src/github.com/joe/project
```

and it should work.

### What's up with the `go_default_library` name?

This is used to keep import paths consistent in libraries that can be built
with `go build`.

In order to compile and link correctly, the Go rules need to be able to
translate between Bazel labels and Go import paths. Let's say your project name
is `github.com/joe/project`, and you have a library in the `foo/bar` directory
named `bar`. The Bazel label for this would be `//foo/bar:bar`. The Go import
path for this would be `github.com/joe/project/foo/bar/bar`.

This is not what `go build` expects; it expects
`github.com/joe/project/foo/bar/bar` to refer to a library built from .go files
in the directory `foo/bar/bar`.

In order to avoid this conflict, you can name your library `go_default_library`.
The full Bazel label for this library would be `//foo/bar:go_default_library`.
The import path would be `github.com/joe/project/foo/bar`.

`BUILD` files generated with Gazelle, including those in external projects
imported with [`go_repository`](#go_repository), will have libraries named
`go_default_library` automatically.

## Repository rules

### `go_rules_dependencies`

``` bzl
go_rules_dependencies()
```

Adds Go-related external dependencies to the WORKSPACE, including the Go
toolchain and standard library. All the other workspace rules and build rules
assume that this rule is placed in the WORKSPACE.
When [nested workspaces](https://bazel.build/designs/2016/09/19/recursive-ws-parsing.html) arrive this will be redundant.

### `go_register_toolchains`

``` bzl
go_register_toolchains(go_version)
```

Installs the Go toolchains. If `go_version` is specified, it sets the SDK version to use (for example, `"1.8.2"`). By default, the latest SDK will be used.


### `go_repository`

```bzl
go_repository(name, importpath, commit, tag, vcs, remote, urls, strip_prefix, type, sha256, build_file_name, build_file_generation, build_tags)
```

Fetches a remote repository of a Go project, and generates `BUILD.bazel` files
if they are not already present. In vcs mode, it recognizes importpath
redirection.

`importpath` must always be specified. This is used as the root import path
for libraries in the repository.

If the repository should be fetched using a VCS, either `commit` or `tag`
must be specified. `remote` and `vcs` may be specified if they can't be
inferred from `importpath` using the
[normal go logic](https://golang.org/cmd/go/#hdr-Remote_import_paths).

If the repository should be fetched using source archives, `urls` and `sha256`
must be specified. `strip_prefix` and `type` may be specified to control how
the archives are unpacked.

`build_file_name`, `build_file_generation`, and `build_tags` may be used to
control how BUILD.bazel files are generated. By default, Gazelle will generate
BUILD.bazel files if they are not already present.

<table class="table table-condensed table-bordered table-params">
  <colgroup>
    <col class="col-param" />
    <col class="param-description" />
  </colgroup>
  <thead>
    <tr>
      <th colspan="2">Attributes</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>name</code></td>
      <td>
        <code>String, required</code>
        <p>A unique name for this external dependency.</p>
      </td>
    </tr>
    <tr>
      <td><code>importpath</code></td>
      <td>
        <code>String, required</code>
        <p>The root import path for libraries in the repository.</p>
      </td>
    </tr>
    <tr>
      <td><code>commit</code></td>
      <td>
        <code>String, optional</code>
        <p>The commit hash to checkout in the repository.<br>
        Exactly one of <code>commit</code> or <code>tag</code> must
        be specified.</p>
      </td>
    </tr>
    <tr>
      <td><code>tag</code></td>
      <td>
        <code>String, optional</code>
        <p>The tag to checkout in the repository.<br>
        Exactly one of <code>commit</code> or <code>tag</code> must
        be specified.</p>
      </td>
    </tr>
    <tr>
      <td><code>vcs</code></td>
      <td>
        <code>String, optional</code>
        <p>The version control system to use for fetching the repository. Useful
        for disabling importpath redirection if necessary. May be
        <code>"git"</code>, <code>"hg"</code>, <code>"svn"</code>,
        or <code>"bzr"</code>.</p>
      </td>
    </tr>
    <tr>
      <td><code>remote</code></td>
      <td>
        <code>String, optional</code>
        <p>The URI of the target remote repository, if this cannot be determined
        from the value of <code>importpath</code>.</p>
      </td>
    </tr>
    <tr>
      <td><code>urls</code></td>
      <td>
        <code>List of Strings, optional</code>
        <p>URLs for one or more source code archives.<br>
        See
        <a href="https://bazel.build/versions/master/docs/be/workspace.html#http_archive"><code>http_archive</code></a>
        for more details.</p>
      </td>
    </tr>
    <tr>
      <td><code>strip_prefix</code></td>
      <td>
        <code>String, optional</code>
        <p>The internal path prefix to strip when the archive is extracted.<br>
        See
        <a href="https://bazel.build/versions/master/docs/be/workspace.html#http_archive"><code>http_archive</code></a>
        for more details.</p>
      </td>
    </tr>
    <tr>
      <td><code>type</code></td>
      <td>
        <code>String, optional</code>
        <p>The type of the archive, only needed if it cannot be inferred from
        the file extension.<br>
        See
        <a href="https://bazel.build/versions/master/docs/be/workspace.html#http_archive"><code>http_archive</code></a>
        for more details.</p>
      </td>
    </tr>
    <tr>
      <td><code>sha256</code></td>
      <td>
        <code>String, optional</code>
        <p>The expected SHA-256 hash of the file downloaded.<br>
        See
        <a href="https://bazel.build/versions/master/docs/be/workspace.html#http_archive"><code>http_archive</code></a>
        for more details.</p>
      </td>
    </tr>
    <tr>
      <td><code>build_file_name</code></td>
      <td>
        <code>String, optional</code>
        <p>The name to use for the generated build files. Defaults to
        BUILD.bazel.</p>
      </td>
    </tr>
    <tr>
      <td><code>build_file_generation</code></td>
      <td>
        <code>String, optional</code>
        <p>Used to force build file generation.<br>
        <code>"off"</code> means do not generate build files.<br>
        <code>"on"</code> means always run gazelle, even if build files are
        already present<br>
        <code>"auto"</code> is the default and runs gazelle only if there is
        no root build file</p>
      </td>
    </tr>
    <tr>
      <td><code>build_tags</code></td>
      <td>
        <code>String, optional</code>
        <p>The set of tags to pass to gazelle when generating build files.</p>
      </td>
    </tr>
  </tbody>
</table>

#### Example:

The rule below fetches a repository with Git. Import path redirection is used
to automatically determine the true location of the repository.

```bzl
load("@io_bazel_rules_go//go:def.bzl", "go_repository")

go_repository(
    name = "org_golang_x_tools",
    importpath = "golang.org/x/tools",
    commit = "663269851cdddc898f963782f74ea574bcd5c814",
)
```

The rule below fetches a repository archive with HTTP. GitHub provides HTTP
archives for all repositories. It's generally faster to fetch these than to
checkout a repository with Git, but the `strip_prefix` part can break if the
repository is renamed.

```bzl
load("@io_bazel_rules_go//go:def.bzl", "go_repository")

go_repository(
    name = "org_golang_x_tools",
    importpath = "golang.org/x/tools",
    urls = ["https://codeload.github.com/golang/tools/zip/663269851cdddc898f963782f74ea574bcd5c814"],
    strip_prefix = "tools-663269851cdddc898f963782f74ea574bcd5c814",
    type = "zip",
)
```

### `new_go_repository`

`new_go_repository` is deprecated. Please use [`go_repository`](#go_repository)
instead, which has the same functionality.

## Build rules

### `go_prefix`

```bzl
go_prefix(prefix)
```

`go_prefix` declares the common prefix of the import path which is shared by
all Go libraries in the repository. A `go_prefix` rule must be declared in the
top-level BUILD file for any repository containing Go rules. This is used by the
Bazel rules during compilation to map import paths to dependencies. See the
[FAQ](#whats-up-with-the-go_default_library-name) for more information.

<table class="table table-condensed table-bordered table-params">
  <colgroup>
    <col class="col-param" />
    <col class="param-description" />
  </colgroup>
  <thead>
    <tr>
      <th colspan="2">Attributes</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>prefix</code></td>
      <td>
        <code>String, required</code>
        <p>Global prefix used to fully qualify all Go targets.</p>
      </td>
    </tr>
  </tbody>
</table>

### `go_library`

```bzl
go_library(name, srcs, deps, data, library, gc_goopts)
```

`go_library` builds a Go library from a set of source files that are all part of
the same package. This library cannot contain cgo code (see
[`cgo_library`](#cgo_library)).

<table class="table table-condensed table-bordered table-params">
  <colgroup>
    <col class="col-param" />
    <col class="param-description" />
  </colgroup>
  <thead>
    <tr>
      <th colspan="2">Attributes</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>name</code></td>
      <td>
        <code>Name, required</code>
        <p>A unique name for this rule.</p>
      </td>
    </tr>
    <tr>
      <td><code>srcs</code></td>
      <td>
        <code>List of labels, required</code>
        <p>List of Go <code>.go</code> (at least one) or ASM <code>.s/.S</code>
        source files used to build the library</p>
      </td>
    </tr>
    <tr>
      <td><code>deps</code></td>
      <td>
        <code>List of labels, optional</code>
        <p>List of other libraries to linked to this library target</p>
      </td>
    </tr>
    <tr>
      <td><code>data</code></td>
      <td>
        <code>List of labels, optional</code>
        <p>List of files needed by this rule at runtime.</p>
      </td>
    </tr>
    <tr>
      <td><code>library</code></td>
      <td>
        <code>Label, optional</code>
        <p>A label of another rule with Go `srcs`, `deps`, and `data`. When this
        library is compiled, the sources from this attribute will be combined
        with `srcs`. This is commonly used to depend on Go sources in
        `cgo_library`.</p>
      </td>
    </tr>
    <tr>
      <td><code>gc_goopts</code></td>
      <td>
        <code>List of strings, optional</code>
        <p>List of flags to add to the Go compilation command. Subject to
        <a href="https://bazel.build/versions/master/docs/be/make-variables.html#make-var-substitution">Make
        variable substitution</a> and
        <a href="https://bazel.build/versions/master/docs/be/common-definitions.html#sh-tokenization">Bourne
        shell tokenization</a>.</p>
      </td>
    </tr>
  </tbody>
</table>

### `cgo_library`

```bzl
cgo_library(name, srcs, copts, clinkopts, cdeps, deps, data, gc_goopts)
```

`cgo_library` builds a Go library from a set of cgo source files that are part
of the same package. This library cannot contain pure Go code (see the note
below).

<table class="table table-condensed table-bordered table-params">
  <colgroup>
    <col class="col-param" />
    <col class="param-description" />
  </colgroup>
  <thead>
    <tr>
      <th colspan="2">Attributes</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>name</code></td>
      <td>
        <code>Name, required</code>
        <p>A unique name for this rule.</p>
      </td>
    </tr>
    <tr>
      <td><code>srcs</code></td>
      <td>
        <code>List of labels, required</code>
        <p>List of Go, C and C++ files that are processed to build a Go
        library.</p>
        <p>Those Go files must contain <code>import "C"</code>. C and C++ files
        can be anything allowed in <code>srcs</code> attribute of
        <code>cc_library</code>.</p>
      </td>
    </tr>
    <tr>
      <td><code>copts</code></td>
      <td>
        <code>List of strings, optional</code>
        <p>Add these flags to the C++ compiler</p>
      </td>
    </tr>
    <tr>
      <td><code>clinkopts</code></td>
      <td>
        <code>List of strings, optional</code>
        <p>Add these flags to the C++ linker</p>
      </td>
    </tr>
    <tr>
      <td><code>cdeps</code></td>
      <td>
        <code>List of labels, optional</code>
        <p>List of C/C++ libraries to be linked into the binary target.
        They must be <code>cc_library</code> rules.</p>
      </td>
    </tr>
    <tr>
      <td><code>deps</code></td>
      <td>
        <code>List of labels, optional</code>
        <p>List of other Go libraries to be linked to this library</p>
      </td>
    </tr>
    <tr>
      <td><code>data</code></td>
      <td>
        <code>List of labels, optional</code>
        <p>List of files needed by this rule at runtime.</p>
      </td>
    </tr>
    <tr>
      <td><code>gc_goopts</code></td>
      <td>
        <code>List of strings, optional</code>
        <p>List of flags to add to the Go compilation command. Subject to
        <a href="https://bazel.build/versions/master/docs/be/make-variables.html#make-var-substitution">Make
        variable substitution</a> and
        <a href="https://bazel.build/versions/master/docs/be/common-definitions.html#sh-tokenization">Bourne
        shell tokenization</a>.</p>
      </td>
    </tr>
  </tbody>
</table>

#### NOTE

`srcs` cannot contain pure-Go files, which do not have `import "C"`.
So you need to define another `go_library` when you build a go package with
both cgo-enabled and pure-Go sources.

```bzl
cgo_library(
    name = "cgo_enabled",
    srcs = ["cgo-enabled.go", "foo.cc", "bar.S", "baz.a"],
)

go_library(
    name = "go_default_library",
    srcs = ["pure-go.go"],
    library = ":cgo_enabled",
)
```

### `go_binary`

```bzl
go_binary(name, srcs, deps, data, library, linkstamp, x_defs, gc_goopts, gc_linkopts)
```

`go_binary` builds an executable from a set of source files, which must all be
in the `main` package. You can run the with `bazel run`, or you can run it
directly.

<table class="table table-condensed table-bordered table-params">
  <colgroup>
    <col class="col-param" />
    <col class="param-description" />
  </colgroup>
  <thead>
    <tr>
      <th colspan="2">Attributes</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>name</code></td>
      <td>
        <code>Name, required</code>
        <p>A unique name for this rule.</p>
      </td>
    </tr>
    <tr>
      <td><code>srcs</code></td>
      <td>
        <code>List of labels, required</code>
        <p>List of Go <code>.go</code> (at least one) or ASM <code>.s/.S</code>
        source files used to build the binary</p>
      </td>
    </tr>
    <tr>
      <td><code>deps</code></td>
      <td>
        <code>List of labels, optional</code>
        <p>List of other Go libraries to linked to this binary target</p>
      </td>
    </tr>
    <tr>
      <td><code>data</code></td>
      <td>
        <code>List of labels, optional</code>
        <p>List of files needed by this rule at runtime.</p>
      </td>
    </tr>
    <tr>
      <td><code>library</code></td>
      <td>
        <code>Label, optional</code>
        <p>A label of another rule with Go `srcs`, `deps`, and `data`. When this
        binary is compiled, the sources from this attribute will be combined
        with `srcs`. This is commonly used to depend on Go sources in
        `cgo_library`.</p>
      </td>
    </tr>
    <tr>
      <td><code>linkstamp</code></td>
      <td>
        <code>String; optional; default is ""</code>
        <p>The name of a package containing global variables set by the linker
        as part of a link stamp. This may be used to embed version information
        in the generated binary. The -X flags will be of the form
        <code>-X <i>linkstamp</i>.KEY=VALUE</code>. The keys and values are
        read from <code>bazel-bin/volatile-status.txt</code> and
        <code>bazel-bin/stable-status.txt</code>. If you build with
        <code>--workspace_status_command=<i>./status.sh</i></code>, the output
        of <code>status.sh</code> will be written to these files.
        <a href="https://github.com/bazelbuild/bazel/blob/master/tools/buildstamp/get_workspace_status">
        Bazel <code>tools/buildstamp/get_workspace_status</code></a> is
        a good template which prints Git workspace status.</p>
      </td>
    </tr>
    <tr>
      <td><code>x_defs</code></td>
      <td>
        <code>Dict of strings; optional</code>
        <p>Additional -X flags to pass to the linker. Keys and values in this
        dict are passed as <code>-X key=value</code>. This can be used to set
        static information that doesn't change in each build.</p>
        <p>If the value is surrounded by curly brackets (e.g.
        <code>{VAR}</code>), then the value of the corresponding workspace
        status variable will be used instead. Valid workspace status variables
        include <code>BUILD_USER</code>, <code>BUILD_EMBED_LABEL</code>, and
        custom variables provided through a
        <code>--workspace_status_command</code> as described in
        <code>linkstamp</code>.</p>
      </td>
    </tr>
    <tr>
      <td><code>gc_goopts</code></td>
      <td>
        <code>List of strings, optional</code>
        <p>List of flags to add to the Go compilation command. Subject to
        <a href="https://bazel.build/versions/master/docs/be/make-variables.html#make-var-substitution">Make
        variable substitution</a> and
        <a href="https://bazel.build/versions/master/docs/be/common-definitions.html#sh-tokenization">Bourne
        shell tokenization</a>.</p>
      </td>
    </tr>
    <tr>
      <td><code>gc_linkopts</code></td>
      <td>
        <code>List of strings, optional</code>
        <p>List of flags to add to the Go link command. Subject to
        <a href="https://bazel.build/versions/master/docs/be/make-variables.html#make-var-substitution">Make
        variable substitution</a> and
        <a href="https://bazel.build/versions/master/docs/be/common-definitions.html#sh-tokenization">Bourne
        shell tokenization</a>.</p>
      </td>
    </tr>
  </tbody>
</table>

### `go_test`

```bzl
go_test(name, srcs, deps, data, library, gc_goopts, gc_linkopts, rundir)
```

`go_test` builds a set of tests that can be run with `bazel test`. This can
contain sources for internal tests or external tests, but not both (see example
below).

To run all tests in the workspace, and print output on failure (the
equivalent of "go test ./..." from `go_prefix` in a `GOPATH` tree), run

```
bazel test --test_output=errors //...
```

You can run specific tests by passing the
[`--test_filter=pattern`](https://bazel.build/versions/master/docs/bazel-user-manual.html#flag--test_filter)
argument to Bazel. You can pass arguments to tests by passing
[`--test_arg=arg`](https://bazel.build/versions/master/docs/bazel-user-manual.html#flag--test_arg)
arguments to Bazel.

<table class="table table-condensed table-bordered table-params">
  <colgroup>
    <col class="col-param" />
    <col class="param-description" />
  </colgroup>
  <thead>
    <tr>
      <th colspan="2">Attributes</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>name</code></td>
      <td>
        <code>Name, required</code>
        <p>A unique name for this rule.</p>
      </td>
    </tr>
    <tr>
      <td><code>srcs</code></td>
      <td>
        <code>List of labels, required</code>
        <p>List of Go <code>.go</code> (at least one) or ASM <code>.s/.S</code>
        source files used to build the test</p>
      </td>
    </tr>
    <tr>
      <td><code>deps</code></td>
      <td>
        <code>List of labels, optional</code>
        <p>List of other Go libraries to linked to this test target</p>
      </td>
    </tr>
    <tr>
      <td><code>data</code></td>
      <td>
        <code>List of labels, optional</code>
        <p>List of files needed by this rule at runtime.</p>
      </td>
    </tr>
    <tr>
      <td><code>library</code></td>
      <td>
        <code>Label, optional</code>
        <p>A label of another rule with Go `srcs`, `deps`, and `data`. When this
        library is compiled, the sources from this attribute will be combined
        with `srcs`.</p>
      </td>
    </tr>
    <tr>
      <td><code>gc_goopts</code></td>
      <td>
        <code>List of strings, optional</code>
        <p>List of flags to add to the Go compilation command. Subject to
        <a href="https://bazel.build/versions/master/docs/be/make-variables.html#make-var-substitution">Make
        variable substitution</a> and
        <a href="https://bazel.build/versions/master/docs/be/common-definitions.html#sh-tokenization">Bourne
        shell tokenization</a>.</p>
      </td>
    </tr>
    <tr>
      <td><code>gc_linkopts</code></td>
      <td>
        <code>List of strings, optional</code>
        <p>List of flags to add to the Go link command. Subject to
        <a href="https://bazel.build/versions/master/docs/be/make-variables.html#make-var-substitution">Make
        variable substitution</a> and
        <a href="https://bazel.build/versions/master/docs/be/common-definitions.html#sh-tokenization">Bourne
        shell tokenization</a>.</p>
      </td>
    </tr>
    <tr>
      <td><code>rundir</code></td>
      <td>
        <code>String, optional</code>
        <p>Path to the directory the test should run in. This should be relative
        to the root of the repository the test is defined in. By default, the
        test will run in the directory of the BUILD file that defines it.
        Use "." to run the test at the repository root.</p>
      </td>
    </tr>
  </tbody>
</table>

#### Example

To write an internal test, reference the library being tested with the `library`
attribute instead of the `deps` attribute. This will compile the test sources
into the same package as the library sources.

``` bzl
go_library(
    name = "go_default_library",
    srcs = ["lib.go"],
)

go_test(
    name = "go_default_test",
    srcs = ["lib_test.go"],
    library = ":go_default_library",
)
```

To write an external test, reference the library being tested with the `deps`
attribute.

``` bzl
go_library(
    name = "go_default_library",
    srcs = ["lib.go"],
)

go_test(
    name = "go_default_xtest",
    srcs = ["lib_x_test.go"],
    deps = [":go_default_library"],
)
```

### `go_proto_library`

```bzl
go_proto_library(name, srcs, deps, has_services)
```
<table class="table table-condensed table-bordered table-params">
  <colgroup>
    <col class="col-param" />
    <col class="param-description" />
  </colgroup>
  <thead>
    <tr>
      <th colspan="2">Attributes</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>name</code></td>
      <td>
        <code>Name, required</code>
        <p>A unique name for the underlying go_library rule. (usually `go_default_library`)</p>
      </td>
    </tr>
    <tr>
      <td><code>srcs</code></td>
      <td>
        <code>List of labels, required</code>
        <p>List of Protocol Buffer <code>.proto</code>
        source files used to generate <code>.go</code> sources for a go_library</p>
      </td>
    </tr>
    <tr>
      <td><code>deps</code></td>
      <td>
        <code>List of labels, optional</code>
        <p>List of other go_proto_library(s) to depend on.
        Note: this also works if the label is a go_library,
        and there is a filegroup {name}+"_protos" (which is used for golang protobuf)</p>
      </td>
    </tr>
    <tr>
      <td><code>has_services</code></td>
      <td>
        <code>integer, optional, defaults to 0</code>
        <p>If 1, will generate with <code>plugins=grpc</code>
        and add the required dependencies.</p>
      </td>
    </tr>
    <tr>
      <td><code>ignore_go_package_option</code></td>
      <td>
        <code>integer, optional, defaults to 0</code>
        <p>If 1, will ignore the go_package option in the srcs proto files.
        Note: this will not work if the go_package options are specified in more
        than one line.
        </p>
      </td>
    </tr>
  </tbody>
</table>

### `go_embed_data`

```bzl
go_embed_data(name, src, srcs, out, package, var, flatten, string)
```
<table class="table table-condensed table-bordered table-params">
  <colgroup>
    <col class="col-param" />
    <col class="param-description" />
  </colgroup>
  <thead>
    <tr>
      <th colspan="2">Attributes</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>name</code></td>
      <td>
        <code>Name, required</code>
        <p>A unique name for the go_embed_data rule.</p>
      </td>
    </tr>
    <tr>
      <td><code>src</code></td>
      <td>
        <code>Label, optional</code>
        <p>A single file to embed. This cannot be used at the same time as
        <code>srcs</code>. The generated file will have a variable of type
        <code>[]byte</code> or <code>string</code> with the contents of
        this file.</p>
      </td>
    </tr>
    <tr>
      <td><code>srcs</code></td>
      <td>
        <code>List of labels, optional</code>
        <p>A list of files to embed. This cannot be used at the same time as
        <code>src</code>. The generated file will have a variable of type
        <code>map[string][]byte</code> or <code>map[string]string</code> with
        the contents of each file. The map keys are relative paths the files
        from the repository root. Keys for files in external repositories will
        be prefixed with "external/repo/" where "repo" is the name of the
        external repository.</p>
      </td>
    </tr>
    <tr>
      <td><code>out</code></td>
      <td>
        <code>String, required</code>
        <p>Name of the .go file to generated. This may be referenced by
        other rules, such as <code>go_library</code>.</p>
      </td>
    </tr>
    <tr>
      <td><code>package</code></td>
      <td>
        <code>String, optional, defaults to directory base name</code>
        <p>Go package name for the generated .go file. This defaults to the
        name of the directory containing the <code>go_embed_data</code> rule.
        This attribute is required in the repository root directory though.</p>
      </td>
    </tr>
    <tr>
      <td><code>var</code></td>
      <td>
        <code>String, optional, defaults to "Data"</code>
        <p>Name of the variable that will contain the embedded data.</p>
      </td>
    </tr>
    <tr>
      <td><code>flatten</code></td>
      <td>
        <code>Boolean, optional, defaults to false</code>
        <p>If true and <code>srcs</code> is used, map keys are file base names
        instead of relative paths.</p>
      </td>
    </tr>
    <tr>
      <td><code>string</code></td>
      <td>
        <code>Boolean, optional, defaults to false</code>
        <p>If true, the embedded data will be stored as <code>string</code>
        instead of <code>[]byte</code>.</p>
      </td>
    </tr>
  </tbody>
</table>

#### Example:

The `foo_data` rule below will generate a file named `foo_data.go`, which can
be included in a library. Gazelle will find and add these files
automatically.

```bzl
load("@io_bazel_rules_go//go:def.bzl", "go_embed_data", "go_library")

go_embed_data(
    name = "foo_data",
    src = "foo.txt",
    out = "foo_data.go",
    package = "foo",
    string = True,
    var = "Data",
)

go_library(
    name = "go_default_library",
    srcs = ["foo_data.go"],
)
```

The generated file will look like this:

```go
// Generated by go_embed_data for //:foo_data. DO NOT EDIT.

package foo



var Data = "Contents of foo.txt"
```
