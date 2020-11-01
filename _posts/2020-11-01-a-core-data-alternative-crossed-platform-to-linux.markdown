---
date: '2020-11-01 06:47:00'
layout: post
slug: a-core-data-alternative-crossed-platform-to-linux
status: publish
title: A Core Data Alternative Crossed Platform to Linux
categories:
- eyes
---

For the past a few months, I’ve worked on [Dflat](https://dflat.io), among [many](https://liuliu.me/eyes/migrating-ios-project-to-bazel-a-real-world-experience/) [other](https://liuliu.me/eyes/loading-csv-file-at-the-speed-limit-of-the-nvme-storage/) [things](https://github.com/liuliu/s4nnc/). The theme is to explore a workflow using [Bazel](https://bazel.build/) as the build and deployment system, and [Swift](https://swift.org/) as the main language for data exploration, modeling, and production system.

Dflat appears to be an outlier for this theme. I simply have the urge to implement what I considered to be the one-true-way of doing structured data persistence on mobile, and to share it with the wider community. It was designed squarely for mobile data persistence needs, much like Core Data before. Case in point, the backing data engine has always been SQLite.

Up until recently. As I dug deeper in this *Swift for data exploration, modeling and production system* theme, it is increasingly likely that I need some data persistence mechanism. It may not be as fancy as for mobile, where I can observe changes and rely on one-way data flow to update UI. But it is darned nice to specify schema explicitly, persisting to a proper database, and don’t worry about schema upgrade at all.

For the past couple of the days, I’ve worked on porting [Dflat to Linux](https://swiftpackageindex.com/liuliu/dflat). With some minimal work (mostly around [figuring out the right SQLite compilation flags](https://github.com/liuliu/dflat/blob/unstable/external/sqlite3.BUILD#L13) and moving code from one place to another to make Swift Linux runtime happy), it is done. I’ve also added some [Bazel rules](https://github.com/liuliu/dflat/blob/unstable/dflat.bzl#L33) for code generation, so if you use Bazel, Dflat would work wonders out of the box.

What does this mean? If you use [Swift in Linux](https://swift.org/download/#releases), you don’t need to interact with SQLite directly any more. Dflat handles SQLite concurrency control, proper SQLite configuration, strongly-typed queries, data schema management, transparent (and read-only) backward compatible upgrades. Beyond that, it offers query subscription so for a long-running program, you can simply subscribe to a query and update based on the changed results.

### An Example

If you happen to do Bazel + Swift on Linux, using Dflat is simple. In `WORKSPACE` file, add following:

```python
git_repository(
    name = "dflat",
    remote = "https://github.com/liuliu/dflat.git",
    commit = "3dc11274e8c466dd28ee35cdd04e84ddf7d420bc",
    shallow_since = "1604185591 -0400"
)

load("@dflat//:deps.bzl", "dflat_deps")

dflat_deps()
```

For the binary that requires to persist some user settings, you can start to edit the schema file: `user.fbs`.

```
table User {
    username: string (primary);
    accessTime: double;
}

root_type User;
```

In your `BUILD.bazel`:

```python
load("@dflat//:dflat.bzl", "dflatc")

dflatc(
    name = "user_schema",
    src = "user.fbs"
)

swift_binary(
    name = "main",
    srcs = ["main.swift", ":user_schema"],
    deps = [
        "@dflat//:SQLiteDflat"
    ]
)
```

Use them in `main.swift` file:

```swift
import Dflat
import SQLiteDflat

let workspace = SQLiteWorkspace(filePath: "main.db", fileProtectionLevel: .noProtection)

workspace.performChanges([User.self], changesHandler: { txnContext in
  let creationRequest = UserChangeRequest.creationRequest()
  creationRequest.username = "lliu"
  creationRequest.accessTime = Date().timeIntervalSince1970
  txnContext.try(submit: creationRequest)
})

workspace.shutdown()
```

Later read the data and update the `accessTime`:

```swift
import Dflat
import SQLiteDflat

let workspace = SQLiteWorkspace(filePath: "main.db", fileProtectionLevel: .noProtection)

let result = workspace.fetch(for: User.self).where(User.username == "lliu")
let user = result[0]
print(user.accessTime)

workspace.performChanges([User.self], changesHandler: { txnContext in
  let changeRequest = UserChangeRequest.changeRequest(user)
  changeRequest.accessTime = Date().timeIntervalSince1970
  txnContext.try(submit: changeRequest)
})

workspace.shutdown()
```

### iPhone 12 Pro v.s. Threadripper 3970x

Dflat benchmark is now available on both iPhone 12 Pro and TR 3970x. It enables some quick apple-to-orange comparisons.

The iPhone 12 Pro benchmark followed exactly [the previous benchmark](https://dflat.io/benchmark/).

The Linux benchmark runs with `bazel run --compliation_mode=opt app:BenchmarksBin`.


| Work | iPhone 12 Pro | TR 3970x |
|:---  |           ---:|      ---:|
| Insert 10,000 | 0.072s | 0.054s |
| Fetch 3,334 | 0.004s | 0.006s |
| Update 10,000 | 0.085s | 0.066s |
| 4-Thread Insert 40,000 | 0.452s | 0.219s |
| 4-Thread Delete 40,000 | 0.289s | 0.193s |
| 10,000 Updates to 1,000 Subscriptions* | 3.249s | 3.356s |

Each subscribed query contains roughly 1,000 objects. 

### Beyond Server / Client Relationship

The [Bazel](https://github.com/liuliu/dflat/blob/unstable/BUILD#L30) and [Swift Package Manager](https://github.com/liuliu/dflat/blob/unstable/Package.swift#L21) version of Dflat for Linux packaged [SQLite 3.33.0](https://swift.org/download/#releases) in source-form. It enables some interesting ideas. With a little bit more work, we may as well compile Dflat to run [in WebAssembly](https://swiftwasm.org/). Wasm can be the ideal form to be deployed at edge nodes. When worked on Snapchat app in the past, I’ve long theorized that we could deploy peering nodes for customers, and the server / device communication can be replaced with server / peering node / device with peering node and device [syncing materialized views incrementally](https://www.sqlite.org/c3ref/update_hook.html) through some kind of [binlog](https://dev.mysql.com/doc/internals/en/binary-log-overview.html) mechanism. It looks like this kind of architecture can be done, if we have some kind of uniformity between peering nodes and mobile devices.

### Beyond SQLite

Even Dflat now supports Linux, it doesn't mean this is a server software. Advanced features such as [live query subscription](https://dflat.io/runtime-api/#data-subscription) can only happen if you go through a single `Workspace` instance. To make Dflat work in Linux, we actually deepened the SQLite and Dflat relationship by promoting some code from SQLiteDflat to Dflat.

This doesn't mean we cannot change the backing data engine. Dflat could potentially help server-grade databases such as PostgreSQL in other ways. [The smooth schema evolution](https://dflat.io/notes/upgrade/) would be an interesting way to formalize what [people like Uber already did](https://eng.uber.com/schemaless-part-one-mysql-datastore/).

A server-grade database such as [PostgreSQL could also potentially support live query subscription](https://www.postgresql.org/docs/current/sql-listen.html) across many `Workspace` instances with some substantial redesign of Dflat internals.

### Exploration Ahead

Supporting Dflat on Linux enables some interesting design space exploration (for one, I need to change `Workspace.shutdown()` semantics to better work with short-lived command-line programs). Whether a mobile-focused database would work for a variety of Linux-based workflows is a question without definitive answer still. I would continue my exploration in the meantime and report back on any substantial findings.