---
date: '2020-08-07 19:02:00'
layout: post
slug: migrating-ios-project-to-bazel-a-real-world-experience
status: publish
title: Migrating iOS Project to Bazel, a Real-World Experience
categories:
- eyes
---

During the past few years, I've been involved in migrations from Xcode workspace based build system to Buck twice (Facebook and Snapchat). Both of these experiences took talented engineers multi-months work with many fixes upstreamed to get it work. Recently, I've been helping another company to migrate their iOS project from Xcode workspace based build system to Bazel. This experience may be relevant to other people, considering the [Bazel guide](https://docs.bazel.build/versions/master/bazel-and-apple.html#migrating-to-bazel) is quite light on details.

## Why

Why in these companies, people choose to use alternative build tools such as Buck or Bazel but not Xcode? There are a few reasons:

### Better Code Review and Dependency Management

Xcode stores its build configurations in both xcscheme files and pbxproj files. None of them are text-editor friendly and heavily rely on Xcode GUI for its configurations. There are ways to only use xcconfig, a text-based configuration file for pbxproj. But for many real-world projects, that is just the third place for build configurations rather than the only source of truth. As a result, configuration changes are impossible to review effectively in any popular web tools (GitHub / Gitlab / Gerrit).

Xcode does dependency management in implicit ways. Thus, you have to create xcworkspace to include all dependent sub-projects for build. This is problematic if you want to split your projects and tests into smaller manageable units. That often ends up with many xcworkspace projects and each need to have manual dependency management and being included in CI tool.

### Better Caching and Faster Build Speed

Xcode's build cache management is quite incompetent. A change in the scheme can result in rebuilding from scratch. Switching between branches can often result in full rebuilding as well.

Bazel is a [hermetic build system](https://landing.google.com/sre/sre-book/chapters/release-engineering/#hermetic-builds-nqslhnid). If a file or its dependency doesn't change, a rebuild won't be triggered. Switching between branches won't slow down the build because artifacts are cached by its content.

Bazel provided an upgrade path from its primitive disk-based cache system to [a distributed remote cache](https://docs.bazel.build/versions/master/remote-caching.html). You can make a switch when the codebase grows or the team grows.

### Better Modularization

Swift likes to have smaller modules. Having each module as its own project poses challenges in code review and dependency management. There are alternative solutions such as Swift Package Manager or CocoaPods. Both of them have their unique set of problems (SPM is often too opinionated, while CocoaPods is slow and invasive).

## When

Engineering teams often have competing priorities. For many teams, it is unlikely that their starting point will be a Bazel-based project (hopefully this will change in the future). When to prioritize the migration to Bazel? I've read it somewhere a good summary on this, and will just put it here: *a good time to migrate to Bazel is when you about to need it*. If the team starts to feel the pain of Xcode-based build system (multi-minute build time, a ball of mud mono-project, or many small projects but with CI brokages many times every week), it can often take months to do the migration. On the other hand, when you have only 2 people developing an app, it is simply hard to judge the value proposition.

A semi mono-project, with 3 to 4 developers, and some external dependencies (~10 CocoaPods libraries) would be a good place to allocate 1 engineer-week to do the transition. That's what I did here.

## Setup Tooling

The first step of the migration is to set up the tools. We will use [Bazelisk](https://github.com/bazelbuild/bazelisk) to manage Bazel versions, and will symlink Bazelisk to `/usr/local/bin/bazel` for ease of access. For Xcode integration, we will use [Tulsi](https://github.com/bazelbuild/tulsi). It will be installed from source. Both tools are checked out under: `$HOME/${Company_Name}/devtools` directory. The installation is automated through scripts inside the repository.

## Setup Repository Directory Structure

While it is possible to manage external dependencies through `WORKSPACE`, Bazel loves monorepo. Thus, we are going to vendoring almost all our dependencies into our repository. The repository will be reorganized from a one-project centric layout to a monorepo layout.

```yaml
$HOME/${Company_Name}/${Repository}/
 - common
 - common/bazel_tools
 - common/scripts
 - common/vendors
 - ios/bazel_tools
 - ios/Apps
 - ios/Features
 - ios/Libraries
 - ios/Scripts
 - ios/Vendors
```

## The Migration

### Bazel Basics

If you haven't, now is a good time to read the [Bazel guide for iOS](https://docs.bazel.build/versions/master/bazel-and-apple.html). We first setup an [ordinary `WORKSPACE` file](https://gist.github.com/liuliu/5127262a28f77946908591fd176b90a8) that has `rules_apple`, `rules_swift` and `xctestrunner` imported.

This will allow us to start to use `swift_library`, `swift_binary` and `ios_application` to quickly build iOS app using Bazel.

For Xcode integration, we use the [`generate_xcodeproj.sh`](https://github.com/bazelbuild/tulsi/blob/master/src/tools/generate_xcodeproj.sh) script to create Xcode project with Tulsi. The `tulsiproj` however, was never checked into our repository. This keeps our Bazel `BUILD` file the only source of truth for our build configurations.

At this step, we exhausted what we learned from the [Bazel guide for iOS](https://docs.bazel.build/versions/master/bazel-and-apple.html) and need to face some real-world challenges.

### CocoaPods Dependencies

It is hard to avoid CocoaPods if you do any kind of iOS development. It is certainly not the best, but it has accumulated enough popularity that everyone has a `.podspec` file somewhere in their open-source repository. Luckily, there is the [PodToBUILD](https://github.com/pinterest/PodToBUILD) project from Pinterest to help alleviate the pain of manually converting a few CocoaPods dependencies to Bazel.

First, I use `pod install --verbose` to collect some information about the existing project's CocoaPods dependency. [This script](https://gist.github.com/liuliu/0bba51a54ef8b1308cf74ef0769f4f82) is used to parse the output and generate `Pods.WORKSPACE` file that PodToBUILD want to use. We use the `bazel run @rules_pods//:update_pods` to vendoring CocoaPods dependencies into the repository.

Some dependencies such as Google's Protobuf already have [Bazel support](https://github.com/protocolbuffers/protobuf/blob/master/BUILD). After vendoring, we can switch from PodToBUILD generated one to the official one. Some of the dependencies are just static / dynamic binary frameworks. We can just use `apple_dynamic_framework_import` / `apple_static_framework_import`. Pure Swift projects support in PodToBUILD is something to be desired. But luckily, we can simply use `swift_library` for these dependencies. They usually don't have complicated setup.

Some do. Realm is a mix of C++17, static libraries (`librealm.a`) and Swift / Objective-C sources. We can still use PodToBUILD utilities to help, and it is a good time to introduce [`swift_c_module`](https://github.com/bazelbuild/rules_swift/blob/master/doc/rules.md#swift_c_module) that can use `modulemap` file to create proper Swift imports.

The **C++17** portion is interesting because until this point, we used Bazel automatically created [toolchains](https://docs.bazel.build/versions/master/toolchains.html). This toolchain, unfortunately, forced C++11 (at least for Bazel 3.4.1 on macOS 10.15). The solution is simple. You need to copy `bazel-$(YourProject)/external/local_config_cc` out into your own `common/bazel_tools` directory. Thus, we will no longer use the automatically generated toolchains configuration. You can modify C++11 to C++17 in the forked `local_config_cc` toolchain.

Here is what my `.bazelrc` looks like after CocoaPods dependencies migration:
```bash
build --apple_crosstool_top=//common/bazel_tools/local_config_cc:toolchain
build --strategy=ObjcLink=standalone
build --symlink_prefix=/
build --features=debug_prefix_map_pwd_is_dot
build --experimental_strict_action_env=true
build --ios_minimum_os=11.0
build --macos_minimum_os=10.14
build --disk_cache=~/${Company_Name}/devtools/.cache
try-import .bazelrc.variant
try-import .bazelrc.local
```

### The Application

If everything goes well, depending on how many CocoaPods dependencies you have, you may end up on day 3 or day 5 now. At this point, you can build each of your CocoaPods dependencies individually. It is time to build the iOS app with Bazel.

There actually aren't many gotchas for building in the simulator. Following the [Bazel guide for iOS](https://docs.bazel.build/versions/master/bazel-and-apple.html) and set up your dependencies properly, you should be able to run the app inside the simulator.

If you have any bridging header (which you should avoid as much as possible!), you can add `["-import-objc-header". "$(location YourBridgingHeader.h)"]` to your `swift_library`'s `copts`.

To run the app from the device, it may need some extra work. First, Bazel needs you to tell it the exact location of provisioning files. I elected to store development provisioning files directly in `ios/ProvisioningFiles` directory. With more people, this may be problematic to update, since each addition of device or developer certificate requires a regeneration of provisioning files. Alternatively, you can manage them through [Fastlane tools](https://fastlane.tools/).

iOS devices are often picky about the entitlements. Make sure you have the proper `application-identifier` key-value pair in your entitlements file.

If you use Xcode, now is a good time to introduce the `focus.py` script. This script will take a Bazel target, and generate / open the Xcode project for you. It is a good idea to have such a script to wrap around [`generate_xcodeproj.sh`](https://github.com/bazelbuild/tulsi/blob/master/src/tools/generate_xcodeproj.sh). You will inevitably need some light modifications around the generated Xcode project or scheme files beyond what Tulsi is capable of. [Here is mine](https://gist.github.com/liuliu/ccbbfe94fed7bfe07148a6ffd200b06e).

You can use such script like this:
```bash
./focus.py ios/Apps/Project:Project
```

#### Dynamic Frameworks

`rules_apple` in 04/2020 [introduced a bug that won't codesign dynamic frameworks properly](https://github.com/bazelbuild/rules_apple/issues/746). It is not a big deal for people that have no dynamic framework dependency (you should strive to be that person!). For many mortals, this is problematic. Simply switching to `rules_apple` prior to that commit will fix the issue.

### Tests

Bazel, as it turns out, has fantastic support for simulator-based tests. I still remember the days to debug Buck issues around hosted unit tests and alike.

Many projects may start with something called *Hosted Tests* in the Xcode world. It is quick and dirty. With *Hosted Tests*, you have full access to UIKit, you can even write fancy [SnapshotTesting](https://github.com/pointfreeco/swift-snapshot-testing). However, now is a good time for you to separate your tests out into two camps: a library test and a hosted test.

A library test in Bazel is a `ios_unit_test` without `test_host` assigned. It can run without the simulator, and often faster. It is restrictive too. Your normal way of accessing `UIImage` from the application bundle won't work. Some UIKit components will not initialize properly without an `UIApplicationDelegate`. These are not bad! It is great to isolate your test to what you really care about: your own component!

You should move most of your existing tests to library tests.

SnapshotTesting has to be a hosted test. There are also implicit assumptions within that library about where to look for snapshot images. Luckily, we can pass `--test_env` from Bazel to our test runner and write [a wrapper around `assertSnapshot` method](https://github.com/pointfreeco/swift-snapshot-testing/blob/bcbbbe28bfd38970d4f4ae27da4427ceb932b397/Sources/SnapshotTesting/AssertSnapshot.swift#L137).

The `ios_ui_test` will just work for your UI tests. The only bug we encountered is about `bundle_name`. Just don't modify `bundle_name` in your `ios_application`. The `ios_ui_test` is not happy to run a bundle that is not named after the Bazel target name.

#### Code Coverage

Bazel in theory have good support for code coverage. You should be able to simply `bazel test --collect_code_coverage` and it is done. However, at least for the particular `rules_apple` and 3.4.1 Bazel, I have trouble doing that.

The code coverage is not hard though. Under the hood, Xcode simply uses [source based code coverage available from Clang / Swift](https://clang.llvm.org/docs/SourceBasedCodeCoverage.html). We can pass the right compilation parameters through Bazel and it will happily build tests with coverage instrumentation.
```bash
build --copt="-fprofile-instr-generate"
build --cxxopt="-fprofile-instr-generate"
build --linkopt="-fprofile-instr-generate"
build --swiftcopt="-profile-generate"
build --copt="-fcoverage-mapping"
build --cxxopt="-fcoverage-mapping"
build --linkopt="-fcoverage-mapping"
build --swiftcopt="-profile-coverage-mapping"
```

To run tests with the coverage report, we need to turn off Bazel sandbox to disable Bazel's tendency of deleting files generated from test runs. `LLVM_PROFILE_FILE` environment variable needs to be passed through `--test_env` as well. Here are four lines how I generated coverage report that [Codecov.io](https://codecov.io/) will be happy to process:
```bash
bazel test --test_env="LLVM_PROFILE_FILE=\"$GIT_ROOT/ProjectTests.profraw\"" --spawn_strategy=standalone --cache_test_results=no ios/Apps/Project:ProjectTests
xcrun llvm-profdata merge -sparse ProjectTests.profraw -o ProjectTests.profdata
BAZEL_BIN=$(bazel info bazel-bin)
xcrun llvm-cov show $BAZEL_BIN/ios/Apps/Project/ProjectTests.__internal__.__test_bundle_archive-root/ProjectTests.xctest/ProjectTests --instr-profile=ProjectTests.profdata > Project.xctest.coverage.txt 2>/dev/null
```

### CI Integration

It is surprisingly simple to integrate Bazel with CI tools we use. For the context, we use [Bitrise](https://www.bitrise.io/) for unit tests and ipa package submission / deployment. Since we scripted our Bazel installation, for unit tests, it is as simple as running `bazel test` from the CI. Both hosted tests and UI tests worked uniformly that way.

There are some assumptions from Bitrise about how provisioning files are retrieved. If you use [Fastlane tools](https://fastlane.tools/) throughout, you may not have the same problem.

We end up [forked its *Auto Provision* step](https://github.com/liuliuvs/steps-ios-auto-provision) to make everything they retrieved from Xcode project the pass-in parameters. At later stage, we simply copied out the provisioning file to replace the development provisioning file from the repo.

## Benefits after the Bazel Migration

### Apollo GraphQL Integration

Prior to Bazel migration, Apollo GraphQL integration relies on Xcode build steps to generate the source. That means we have tens of thousands lines of code need to be recompiled every time when we build. People also need to install apollo toolchain separately on their system, with node.js and npm dependencies.

We were able to integrate the packaged apollo cli further into Bazel.
```python
http_archive(
  name = "apollo_cli",
  sha256 = "c2b1215eb8e82ec9d777f4b1590ed0f60960a23badadd889e4d129eb08866f14",
  urls = ["https://install.apollographql.com/legacy-cli/darwin/2.30.1"],
  type = "tar.gz",
  build_file = "apollo_cli.BUILD"
)
```

The toolchain itself will be managed as a `sh_binary` from Bazel perspective.
```python
sh_binary(
  name = "cli",
  srcs = ["run.sh"],
  data = ["@apollo_cli//:srcs"],
  visibility = ["//visibility:public"]
)
```
```bash
#!/usr/bin/env bash
RUNFILES=${BASH_SOURCE[0]}.runfiles
"$RUNFILES/__main__/external/apollo_cli/apollo/bin/node" "$RUNFILES/__main__/external/apollo_cli/apollo/bin/run" "$@"
```

With `genrule` and apollo cli, we were able to generate the source code from Bazel as a separate module. In this way, unless the query changed or schema changed, we don't need to recompile the GraphQL module any more.
```python
filegroup(
  name = "GraphQL_Files",
  srcs = glob(["*.graphql"])
)

filegroup(
  name = "GraphQL_Schema",
  srcs = ["schema.json"]
)

genrule(
  name = "GraphQLAPI_Sources",
  srcs = [":GraphQL_Files", ":GraphQL_Schema"],
  outs = ["GraphQLAPI.swift"],
  # Apollo CLI is not happy to see bunch of symlinked files. So we copied the GraphQL files out
  # such that we can use --includes properly.
  cmd = """
mkdir -p $$(dirname $(location GraphQLAPI.swift))/SearchPaths && \
cp $(locations :GraphQL_Files) $$(dirname $(location GraphQLAPI.swift))/SearchPaths && \
$(location //common/vendors/apollo:cli) codegen:generate --target=swift --includes=$$(dirname $(location GraphQLAPI.swift))/SearchPaths/*.graphql --localSchemaFile=$(location :GraphQL_Schema) $(location GraphQLAPI.swift)
""",
  tools = ["//common/vendors/apollo:cli"]
)
```

### CI Build Time

Even Bazel's disk cache is primitive, we were able to reap the benefit from our CI side. Bitrise CI allows you to push and pull build artifacts. We were able to leverage that to cut our build time by half from Bitrise.

### Build for Different Flavors

A `select_a_variant` Bazel function is introduced. Under the hood, it is based on [`select`](https://docs.bazel.build/versions/master/be/functions.html#select) and [`config_setting`](https://docs.bazel.build/versions/master/be/general.html#config_setting) primitives from Bazel. A [simple `variant.py` script](https://gist.github.com/liuliu/6172e7535b4218e27c829492d2499f98) can be added to switch between different flavors.

For different flavors, a different set of Swift macros will be passed in (we wrapped `swift_library` with a new Bazel macro). Different sets of dependencies can be selected as well. These build options, particularly for dependencies, are difficult to manage with the old Xcode build system.

### Code Generation and More Code Generations

We've changed the app runtime static configurations from reading a bundled-JSON file at startup to generating Swift code from the same JSON file. I am looking forward to having more code generations and even try a bit of [Sourcery](https://github.com/krzysztofzablocki/Sourcery) now after the Bazel migration. The vanilla `genrule` is versatile enough and supports multiple outputs (comparing to Buck). Once figured out that `swift_binary` should belong to the `tools` parameter of `genrule`, it is a breeze to write code generation tools in Swift.

## Conclusion

Even though there are still some workarounds needed for Bazel. It is night and day compared to my experience with Buck a few years ago. It is relatively simple to migrate a somewhat complicated setup in a few days.

Looking forward, I think a few things can be improved.

[PodToBUILD](https://github.com/pinterest/PodToBUILD) was built with Swift. It cannot parse many Ruby syntax, and can cause failures from that. In retrospect, we probably should have such tool to be built with Ruby. At the end of the day, the CocoaPods build syntax is not complicated. Once you run Ruby through that DSL, everything should be neatly laid out.

Although Bazel is language agnostic. I hope that in the future, we can have a Bazel package manager that is as easy to use as CocoaPods. That probably can be the final nag for many people to use Bazel to start new iOS projects. The `WORKSPACE` alternative with `git_repository` is not a real solution. For one, it doesn't traverse dependencies by default. This is for a good reason if you understand the philosophy behind it. But still, it makes Bazel a harder hill to climb.

Let me know if you have more questions.
