# SwiftPM 6.4 release compilation-mode spike

Experimental branch only. Do not install over the standard toolchain.

Base: SwiftPM `swift-6.4.0-RELEASE`, commit `18da3eb1e770679f6910890fbd52e95af53f67d2`.

This isolates the essential PIF-setting change from upstream PR #10632 (head
`8d2374c32a6f8f7e47ff0283ec985bddd9b1a6aa`): release optimization is `-O`, and
`SWIFT_COMPILATION_MODE` is `singlefile` (the upstream mapping for incremental).
The complete manifest API is not backported. Every release Swift target built by
this experimental executable gets these settings. Debug settings are unchanged.

The build engine and compiler agree on compilation mode and dependency outputs.
No compiler-only `-no-whole-module-optimization` override is required. The legacy
SwiftPM engine must never be invoked, including while bootstrapping this fork.

`Package.resolved` locks related dependencies to their `swift-6.4.0-RELEASE`
commits while retaining upstream branch requirements. Build with automatic
resolution disabled to preserve those pins. No Swift Build C/C++ optimization
patch or compiler upgrade is needed for this narrow experiment.

## Bootstrap

In the existing ARM64 Swift 6.4 Linux dependency image, with the source mounted
at `/spike` and the container limited to 20 GiB inside a 24 GiB Podman VM:

```sh
apt-get update
apt-get install -y --no-install-recommends libsqlite3-dev libncurses-dev uuid-dev
swift build --build-system swiftbuild --disable-automatic-resolution \
  --product swift-build -c debug -j 2
```

The fork executable itself is a debug build to keep bootstrap memory low. Its
consumer's `-c release` remains a release build, including compilation conditions.

## Invoke

Keep the fork's resource bundles beside its executable; mount its build tree at
`/spike` and invoke it explicitly rather than replacing `/usr/bin/swift-build`:

```sh
export SWIFTPM_CUSTOM_LIBS_DIR=/usr/lib/swift/pm
export SWIFT_EXEC=/usr/bin/swiftc
/spike/.build/out/Products/Debug-linux-aarch64/swift-build \
  --build-system swiftbuild -c release -j 2
```

A Linux executable/resource probe passed a clean release build, a no-op rebuild
with no compilation, and an incremental source edit. It rejects `DEBUG` at
compile time and checks optimized assertion configuration at runtime. The
compiler invocation includes `-O -enable-batch-mode -incremental`, no WMO flag,
no `DEBUG`, and no `-enable-testing`. Probe timings are not application benchmarks.

Full PulseServer validation is recorded separately in the platform branch's
`docs/reviews/shared-rust-remaining/podman-review/` directory. This spike does not
establish production suitability, performance equivalence, or binary-size savings.
