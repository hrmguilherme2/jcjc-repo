
[![CI](https://github.com/stackb/bazel-jacocorunner/actions/workflows/ci.yaml/badge.svg)](https://github.com/stackb/bazel-jacocorunner/actions/workflows/ci.yaml)

# bazel-jacocorunner

- [bazel-jacocorunner](#bazel-jacocorunner)
  - [What is this?](#what-is-this)
- [Usage](#usage)
  - [`WORKSPACE`](#workspace)
  - [`.bazelrc`](#bazelrc)
  - [`coverage.sh`](#coveragesh)
- [NOTES](#notes)
  - [How do I know which `java_toolchain` is being used?](#how-do-i-know-which-java_toolchain-is-being-used)
  - [JacocoCoverageRunner Changes](#jacococoveragerunner-changes)
  - [Scala Coverage Caveats](#scala-coverage-caveats)

## What is this?

- Bazel's [`jacocorunner` source
  code](https://github.com/bazelbuild/bazel/blob/master/src/java_tools/junitrunner/java/com/google/testing/coverage/BUILD),
  copied here so we can more easily build it outside of bazel itself using
  different jacoco jar dependencies.
  (see `//java/com/google/testing/coverage`).
- A custom http_archive rule that downloads @gergelyfabian's [scala-improved
  fork of jacoco](https://github.com/gergelyfabian/jacoco), builds it with
  `mvn`, and exposes the maven artifacts as deps (`jacoco_http_archive`).
- A set of scala+java examples, shamelessly copied from
  https://github.com/gergelyfabian/bazel-scala-example, which serves as a
  test-base so we can check that things are working.
- A `coverage.sh` script that runs `bazel coverage` and then performs
  lcov/genhtml post-processing on the combined `_coverage_report.dat`.
- A github workflow `ci.yaml` that runs the tests on new PRs and the `master`
  branch.
- A github workflow `release.yaml` that publishes `jacocorunner-{VERSION}.jar`
  as a release asset.
- A set of `java_toolchain` and
  `toolchain<@bazel_tools//tools/jdk:toolchain_type>` in `//tools/jdk` (using
  the same naming patterns in `@bazel_tools//tools/jdk`) that consume the
  release asset jar in the `java_toolchain.jacocorunner` attribute.

# Usage

To use one of the custom java toolchains, you could do something like the
following:

## `WORKSPACE`

```bazel
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

# --------------------------------------------------------
# provides @build_stack_bazel_jacocorunner//tools/jdk:*
# --------------------------------------------------------

# Branch: master
# Commit: 6b66562185f2700dcdb33c7a5382bde0c7f7d15f
# Date: 2022-12-20 22:51:53 +0000 UTC
# URL: https://github.com/stackb/bazel-jacocorunner/commit/6b66562185f2700dcdb33c7a5382bde0c7f7d15f
# 
# expand toolchain completely
# Size: 443788 (444 kB)
http_archive(
    name = "build_stack_bazel_jacocorunner",
    sha256 = "8c940052ae59e2bf0d15d36e704222f3a9201cc6599b16e4cac2467c66083378",
    strip_prefix = "bazel-jacocorunner-6b66562185f2700dcdb33c7a5382bde0c7f7d15f",
    urls = ["https://github.com/stackb/bazel-jacocorunner/archive/6b66562185f2700dcdb33c7a5382bde0c7f7d15f.tar.gz"],
)

# --------------------------------------------------------
# provides @bazel_jacocorunner//:jar, needed by 
# toolchains in @build_stack_bazel_jacocorunner//tools/jdk
# --------------------------------------------------------

load("@build_stack_bazel_jacocorunner//:repositories.bzl", "bazel_jacocorunner")

bazel_jacocorunner()

# --------------------------------------------------------
# register a toolchain
# --------------------------------------------------------

register_toolchains("@build_stack_bazel_jacocorunner//tools/jdk:toolchain_java11_definition")
```

## `.bazelrc`

```conf
build:java11 --java_language_version=11
build:java11 --tool_java_language_version=11
build:java11 --java_runtime_version=remotejdk_11
build:java11 --tool_java_runtime_version=remotejdk_11
build:java11 --java_toolchain=@build_stack_bazel_jacocorunner//tools/jdk:toolchain_java11_definition
build:java11 --host_java_toolchain=@build_stack_bazel_jacocorunner//tools/jdk:toolchain_java11_definition

coverage:combined --combined_report=lcov
coverage:combined --coverage_report_generator="@bazel_tools//tools/test/CoverageOutputGenerator/java/com/google/devtools/coverageoutputgenerator:Main"
coverage:combined --experimental_fetch_all_coverage_outputs
coverage:combined --test_env=VERBOSE_COVERAGE=1

build --config=java11
coverage --config=combined
```

## `coverage.sh`

If you want to use lcov/genhtml without having to install them on the host, add
the following to your `WORKSPACE`:

```bazel
load("@build_stack_bazel_jacocorunner//:repositories.bzl", "lcov_repositories")

lcov_repositories()
```

This could be used something like the following in a shell script:

```sh
# prebuild some tools so they are in bazel-bin...
bazel build @linux_test_project_lcov//:genhtml_bin @build_stack_bazel_jacocorunner//tools/covbean

# run your coverage targets and generate the bazel-out/_coverage/_coverage_report.dat file...
bazel coverage //...

# make a temp directory for the report files
reportdir=$(mktemp -d /tmp/bazelcov.XXXXXX)

# run genhtml on the output
bazel-bin/external/linux_test_project_lcov/genhtml_bin \
  -branch-coverage \
  -o $reportdir \
  bazel-out/_coverage/_coverage_report.dat

# optionally, pack the report into an executable zip with an embedded webserver
local covbean_zip="$PWD/covbean.zip"
cp bazel-bin/external/build_stack_bazel_jacocorunner/tools/covbean/covbean.zip $covbean_zip
chmod +wx $covbean_zip
(cd $reportdir && zip $covbean_zip .)

# clean up
rm -rf $reportdir

```

Then you can view your report as follows:

```sh
sh ./covbean.zip &
open http://localhost:8080
```

# NOTES

## How do I know which `java_toolchain` is being used?

Try building a java target with `--toolchain_resolution_debug='@bazel_tools//tools/jdk:runtime_toolchain_type'`.

```
INFO: ToolchainResolution: Target platform @rules_nixpkgs_core//platforms:host: Selected execution platform @rules_nixpkgs_core//platforms:host, type @bazel_tools//tools/jdk:runtime_toolchain_type -> toolchain @remotejdk11_macos_aarch64//:jdk
```

`bazel query @remotejdk11_macos_aarch64//:jdk --output build` resolves to:

```bazel
java_runtime(
    name = "jdk",
    srcs = [
        "@remotejdk11_macos_aarch64//:jdk-bin",
        "@remotejdk11_macos_aarch64//:jdk-conf",
        "@remotejdk11_macos_aarch64//:jdk-include",
        "@remotejdk11_macos_aarch64//:jdk-lib",
        "@remotejdk11_macos_aarch64//:jre-default",
    ],
)
```

We can also see this in `aquery --output text`, which prints out the raw action prototext:

`bazel aquery --output=text //example/lib:hello-world`:

```conf
action 'Building example/lib/libhello-world.jar (1 source file)'
  Mnemonic: Javac
  Target: //example/lib:hello-world
  Configuration: darwin_arm64-fastbuild
  Execution platform: @local_config_platform//:host
  ActionKey: 944e1415223fe32d4855f27591467048e6e6f7dd768beb5d8e4a7dbfb0c3b253
  Inputs: [...]
  Environment: [LC_CTYPE=en_US.UTF-8]
  ExecutionInfo: {internal-inline-outputs: bazel-out/darwin_arm64-fastbuild/bin/example/lib/libhello-world.jdeps, supports-multiplex-workers: 1, supports-worker-cancellation: 1, supports-workers: 1}
  Command Line: (exec external/remotejdk11_macos_aarch64/bin/java \
    -XX:-CompactStrings \
    '--add-exports=jdk.compiler/com.sun.tools.javac.api=ALL-UNNAMED' \
    '--add-exports=jdk.compiler/com.sun.tools.javac.main=ALL-UNNAMED' \
    '--add-exports=jdk.compiler/com.sun.tools.javac.model=ALL-UNNAMED' \
    '--add-exports=jdk.compiler/com.sun.tools.javac.processing=ALL-UNNAMED' \
    '--add-exports=jdk.compiler/com.sun.tools.javac.resources=ALL-UNNAMED' \
    '--add-exports=jdk.compiler/com.sun.tools.javac.tree=ALL-UNNAMED' \
    '--add-exports=jdk.compiler/com.sun.tools.javac.util=ALL-UNNAMED' \
    '--add-opens=jdk.compiler/com.sun.tools.javac.code=ALL-UNNAMED' \
    '--add-opens=jdk.compiler/com.sun.tools.javac.comp=ALL-UNNAMED' \
    '--add-opens=jdk.compiler/com.sun.tools.javac.file=ALL-UNNAMED' \
    '--add-opens=jdk.compiler/com.sun.tools.javac.parser=ALL-UNNAMED' \
    '--add-opens=java.base/java.nio=ALL-UNNAMED' \
    '--add-opens=java.base/java.lang=ALL-UNNAMED' \
    '-Dsun.io.useCanonCaches=false' \
    -jar \
    external/remote_java_tools/java_tools/JavaBuilder_deploy.jar \
    --output \
    bazel-out/darwin_arm64-fastbuild/bin/example/lib/libhello-world.jar \
    --native_header_output \
    bazel-out/darwin_arm64-fastbuild/bin/example/lib/libhello-world-native-header.jar \
    --output_manifest_proto \
    bazel-out/darwin_arm64-fastbuild/bin/example/lib/libhello-world.jar_manifest_proto \
    --compress_jar \
    --output_deps_proto \
    bazel-out/darwin_arm64-fastbuild/bin/example/lib/libhello-world.jdeps \
    --bootclasspath \
    bazel-out/darwin_arm64-fastbuild/bin/external/bazel_tools/tools/jdk/platformclasspath.jar \
    --sources \
    example/lib/src/main/java/mypackage/Greeter.java \
    --javacopts \
    -source \
    11 \
    -target \
    11 \
    '-XDskipDuplicateBridges=true' \
    '-XDcompilePolicy=simple' \
    -g \
    -parameters \
    -Xep:ReturnValueIgnored:OFF \
    -- \
    --target_label \
    //example/lib:hello-world \
    --strict_java_deps \
    ERROR \
    --experimental_fix_deps_tool \
    add_dep \
    --reduce_classpath_mode \
    JAVABUILDER_REDUCED)
```

## JacocoCoverageRunner Changes

The `java/com/google/testing/coverage/JacocoLCOVFormatter.java` in this repo
differs a little bit from the one in the bazel repository.  In the original
repo, there is a function `.getExecPathForEntryName(String classPath)` that
takes the computed name of the `IClassCoverage` and tries to match it against a
set of paths that are known from scanning files named in the
`-paths-for-coverage.txt` file in instrumented jar `META-INF/`.  That function
does an `.endswith()` comparison to match files, and if
`.getExecPathForEntryName` return `null`, coverage will not be generated for
that file.  In our case, the layout of files does not exactly match the package
names.  Additional logic has been added: if the basenames match and the
`execPath` contains the package name, this is considered a match.

## Scala Coverage Caveats

When scala coverage is run, it creates instrumented jars like
`mylib-offline.jar` where the bytecode has been instrumented by jacoco.  This
creates a runtime dependency of the code to be tested/analyzed for coverage on
the jacoco agent code, which is provided by default in rules_scala by
`@bazel_tools//tools/jdk:JacocoCoverage`.  It turns out different jacoco
versions have unique internally shaded classnames, for example
`org/jacoco/agent/rt/internal_f3994fa/core/JaCoCo.class`.  So, if you try and
analyze code that has been instrumented by a different jacoco version used
under test, you'll get a ClassNotFoundError and coverage will fail
(will be looking for something like `.../internal_c0314ef/...`).

As a result, you need to ensure that the same jacoco version is used across the
`java_toolchain.jacocorunner`, `scala_toolchain.jacocorunner`, and the
`JacocoInstrumenter.deps`.  It is not possible to configure the `deps` of the
`rules_scala/src/java/io/bazel/rulesscala/coverage/instrumenter`, so you'll need
to patch rules_scala with your custom jacocorunner.# jcjc-repo
# jcjc-repo
