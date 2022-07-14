---
layout: post
comments: true
---


clang-tidy is a wonderful tool to run static analysis on C/C++ code. The main
downside is that it takes quite a long time to run. For our project, it requires
around 25 minutes to do a full build and more than 40 minutes to do a full
clang-tidy check. This year, we began to move our build system to
[Bazel](https://bazel.build/). Naturally, we want to run clang-tidy with Bazel.
Leveraging Bazel’s great dependency tracking and remote cache support to reduce the clang-tidy check time.

There is a great lib for running clang-tidy with Bazel named
[bazel_clang_tidy](https://github.com/erenon/bazel_clang_tidy). It’s very easy
to set up and easy to use. The only downside of it is that the check is not
reproducible. In this article, we will share how we update it to support
reproducible check.

Before introducing our modification we will first explain how bazel_clang_tidy
works briefly. bazel_clang_tidy uses [aspects](https://bazel.build/rules/aspects)
to [extract source files and compilation flags](https://github.com/erenon/bazel_clang_tidy/blob/master/clang_tidy/clang_tidy.bzl)
from a C/C++ target. Then call clang-tidy to check the source files with the
compilation flags.

The main reason it’s not reproducible is that it uses the clang-tidy in `PATH`
and people may have different versions of clang-tidy installed. In order to have
reproducible results, we need to know which version of clang-tidy we are using
and make sure if the version changed the Bazel would detect it.

Inspired by `android_ndk_repository` rule, we wrote a `llvm_repo` repository
rule. The `llvm_repo` requires an `attr` named `llvm_version` . This is the
clang-tidy version we would expect. And an environment variable named
`LLVM_HOME`, which points to the directory that contains `bin/clang-t` .

```python
_llvm_repo_attrs = {
    "llvm_version": attr.string(doc='LLVM version', mandatory=True)
}

llvm_repo = repository_rule(
    implementation = _llvm_repo_impl,
    attrs = _llvm_repo_attrs,
    doc = "Setup llvm repo.",
    environ = ["LLVM_HOME"],
)
```

In the implementation of `llvm_repo` , we will first check if the clang-tidy in
`LLVM_HOME` is the same as`llvm_version`. If it doesn’t match, we will call `fail`.

```python
def _llvm_repo_impl(ctx):
    """Implementation of the llvm_repo rule."""
    llvm_home = ctx.os.environ["LLVM_HOME"]
    llvm_version = ctx.attr.llvm_version
    clang_tidy_version = ctx.execute(["{0}/bin/clang-tidy".format(llvm_home), "--version"])
    if clang_tidy_version.return_code != 0:
        fail("Failed to run clang-tidy.")
    if "LLVM version {0}".format(llvm_version) not in clang_tidy_version.stdout:
        fail("LLVM version not match.")

```

If the version matches, we will create a symlink inside `llvm_repo` points to
the `LLVM_HOME`.

```python
def _llvm_repo_impl(ctx):
    """Implementation of the llvm_repo rule."""
    ...
    ctx.symlink(llvm_home, "llvm-"+llvm_version)
```

In the origin `bazel_clang_tidy` lib, it has a `sh_binary` to run clang-tidy.
The wrapper script is needed to create the declared output file for Bazel even
if there is no error reported. We still need this script, but we need to update
it so it will use the `clang-tidy` in `llvm_repo` .

We first declare the `clang-tidy` in `llvm_repo` as a data dependence of
`run_clang_tidy` .

```python
sh_binary(
    name = "clang_tidy",
    srcs = ["run_clang_tidy.sh"],
    data = [":llvm-{llvm_version}/bin/clang-tidy", "//:clang_tidy_config"],
    tags = ["no-sandbox"],
    visibility = ["//visibility:public"],
    deps = ["@bazel_tools//tools/bash/runfiles"],
)
```

Then we use the [runfiles library](https://github.com/bazelbuild/bazel/blob/master/tools/bash/runfiles/runfiles.bash) to get the real path of `clang-tidy`

```bash
set -ue

# Usage: run_clang_tidy <OUTPUT> [ARGS...]

OUTPUT=$1
shift

# clang-tidy doesn't create a patchfile if there are no errors.
# make sure the output exists, and empty if there are no errors,
# so the build system will not be confused.
rm -f $OUTPUT
touch $OUTPUT

$(rlocation "{repo_name}/llvm-{llvm_version}/bin/clang-tidy") "$@"
```

Note in order to use the runfiles library you need to append the initialization
script of runfiles library before your script. It’s omitted here.

For runfiles library to find the file in an external repo, the repository name
is needed. So we use the `ctx.file` to dynamically create the build file and the
run_clang_tidy.sh in `llvm_repo` . 

```python
BUILD_CONTENT = """
sh_binary(
    name = "clang_tidy",
    srcs = ["run_clang_tidy.sh"],
    data = [":llvm-{llvm_version}/bin/clang-tidy", "//:clang_tidy_config"],
    tags = ["no-sandbox"],
    visibility = ["//visibility:public"],
    deps = ["@bazel_tools//tools/bash/runfiles"],
)

filegroup(
    name = "clang_tidy_config_default",
    data = [
        ".clang-tidy",
        # '//example:clang_tidy_config', # add package specific configs if needed
    ],
)

label_flag(
    name = "clang_tidy_config",
    build_setting_default = ":clang_tidy_config_default",
    visibility = ["//visibility:public"],
)
"""

RUNFILES_INIT = """
# --- begin runfiles.bash initialization ---
# Copy-pasted from Bazel's Bash runfiles library (tools/bash/runfiles/runfiles.bash).
set -euo pipefail
if [[ ! -d "${RUNFILES_DIR:-/dev/null}" && ! -f "${RUNFILES_MANIFEST_FILE:-/dev/null}" ]]; then
  if [[ -f "$0.runfiles_manifest" ]]; then
    export RUNFILES_MANIFEST_FILE="$0.runfiles_manifest"
  elif [[ -f "$0.runfiles/MANIFEST" ]]; then
    export RUNFILES_MANIFEST_FILE="$0.runfiles/MANIFEST"
  elif [[ -f "$0.runfiles/bazel_tools/tools/bash/runfiles/runfiles.bash" ]]; then
    export RUNFILES_DIR="$0.runfiles"
  fi
fi
if [[ -f "${RUNFILES_DIR:-/dev/null}/bazel_tools/tools/bash/runfiles/runfiles.bash" ]]; then
  source "${RUNFILES_DIR}/bazel_tools/tools/bash/runfiles/runfiles.bash"
elif [[ -f "${RUNFILES_MANIFEST_FILE:-/dev/null}" ]]; then
  source "$(grep -m1 "^bazel_tools/tools/bash/runfiles/runfiles.bash " \
            "$RUNFILES_MANIFEST_FILE" | cut -d ' ' -f 2-)"
else
  echo >&2 "ERROR: cannot find @bazel_tools//tools/bash/runfiles:runfiles.bash"
  exit 1
fi
# --- end runfiles.bash initialization ---
"""

RUN_CLANG_TIDY_SH = """
set -ue

# Usage: run_clang_tidy <OUTPUT> [ARGS...]

OUTPUT=$1
shift

# clang-tidy doesn't create a patchfile if there are no errors.
# make sure the output exists, and empty if there are no errors,
# so the build system will not be confused.
rm -f $OUTPUT
touch $OUTPUT

$(rlocation "{repo_name}/llvm-{llvm_version}/bin/clang-tidy") "$@"
"""

DOT_CLANG_TIDY = """
UseColor: true

Checks: >
    bugprone-*,
    cppcoreguidelines-*,
    google-*,
    performance-*,
HeaderFilterRegex: ".*"

WarningsAsErrors: "*"
"""

def _llvm_repo_impl(ctx):
    """Implementation of the llvm_repo rule."""
    llvm_home = ctx.os.environ["LLVM_HOME"]
    llvm_version = ctx.attr.llvm_version
    clang_tidy_version = ctx.execute(["{0}/bin/clang-tidy".format(llvm_home), "--version"])
    if clang_tidy_version.return_code != 0:
        fail("Failed to run clang-tidy.")
    if "LLVM version {0}".format(llvm_version) not in clang_tidy_version.stdout:
        fail("LLVM version not match.")
    ctx.file("BUILD",
             BUILD_CONTENT.format(llvm_version=llvm_version),
             executable=False)
    ctx.file("run_clang_tidy.sh",
             RUNFILES_INIT+RUN_CLANG_TIDY_SH.format(repo_name=ctx.name,
                                                    llvm_version=llvm_version),
             executable=True)
    ctx.file(".clang-tidy", DOT_CLANG_TIDY, executable=False)
    ctx.symlink(llvm_home, "llvm-"+llvm_version)
```

Finally, we tell the `clang_tidy_aspect` to use the `clang-tidy` from `llvm_repo` in `.bazelrc`.

```bash
build:clang-tidy --@bazel_clang_tidy//:clang_tidy=@llvm//:clang_tidy
```

Another problem is that the generated result file contains the absolute path of
the source code and build directory. We use a very simple way to fix this.
Since all Bazel actions are running in execution root and it’s the execution
root part we want to remove from the paths in result files. We simply call `pwd`
to get the current execution root path and use `sed` to replace it to a
reproducible value in file.

We replace it with the workspace name but you can choose whatever you want.

```bash
$(rlocation "{repo_name}/llvm-{llvm_version}/bin/clang-tidy") "${{ARGS[@]}}"
EXECUTION_ROOT="$(pwd)"
sed -i '' "s=$EXECUTION_ROOT=$WORKSPACE_NAME=g" "$OUTPUT"
```

That’s it. The modified bazel_clang_tidy is far from perfect but works great in
our project. The average analysis time reduced dramatically. Hope this can help
your project too. Thanks!
