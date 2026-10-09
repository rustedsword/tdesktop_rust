# libtdesktop-rust for ppa:lightofmysoul/tg

Ubuntu 26.04 (resolute) and 26.10 (stonking) packaging of the Rust half of
Telegram Desktop for **amd64 + arm64**. Since 7.3.0, Telegram Desktop links
every Rust dependency as one static library, `tdesktop_rust`:

| crate | upstream | used for |
|---|---|---|
| `tlottie` | [dkaraush/tlottie](https://github.com/dkaraush/tlottie) | stickers, emoji and interface animations (since 7.2.7) |
| `wallet-engine` | [i582/wallet-engine](https://github.com/i582/wallet-engine) | the in-app TON wallet (since 7.3.0) |

They are one archive because a Rust staticlib embeds the Rust standard
library: two of them define its global runtime symbols twice and the linker
refuses the binary. This package supersedes `libtlottie-dev`, which shipped
tlottie alone.

There is no upstream repository for `tdesktop_rust`. Telegram Desktop's build
recipes (`Telegram/build/prepare/prepare.py`, the `centos_env` Dockerfile and
`snap/snapcraft.yaml`) generate it on the fly, so this branch is that
generated crate plus the two upstream trees as submodules, pinned where
Telegram Desktop pins them:

```
tlottie/          submodule, dkaraush/tlottie at the pin
wallet-engine/    submodule, i582/wallet-engine at the pin
tdesktop_rust/    the umbrella crate, exactly as the recipes generate it
vendor/           every crates.io dependency, see below
```

## Usage

```
git submodule update --init             # GIT_LFS_SKIP_SMUDGE=1 is fine
./publish.sh --test-build --no-upload   # validate locally (~5 min)
./publish.sh                            # sign and upload
./publish.sh --help
```

## What is built, and how

`debian/rules` runs upstream's recipe: `cargo rustc --lib --release
--crate-type staticlib --crate-type cdylib` in `tdesktop_rust/` with the
snap's profile, then wallet-engine's own `uniffi-bindgen-cpp` over the cdylib
to generate the C++ binding, then `objcopy --strip-debug` on the archive. The
cdylib exists only for the generator to read; it is not installed.

What is ours:

1. **`debian/patches/wallet-engine.patch`** — not a deviation. It is
   `wallet-engine.patch` from
   [desktop-app/patches](https://github.com/desktop-app/patches), unmodified,
   which every Telegram Desktop recipe applies on top of the pin. It adds the
   TL-B decoder for TON Connect `signData` cells.
2. **`debian/patches/tlottie-cpuid-rust-1.93.patch`** — upstream builds with
   Rust 1.96.1, resolute has 1.93.1, where `__cpuid`/`__cpuid_count` are
   still `unsafe`. Three `unsafe` blocks; redundant but harmless on 1.96.
3. **`--ignore-rust-version`** — wallet-engine declares `rust-version =
   "1.96.1"` but compiles unchanged on 1.93.1.
4. **`vendor/`**, see below.

Both patches target submodules, so they cannot be commits on this branch;
`override_dh_auto_configure` applies them, as the telegram-desktop package does
with its Qt patch.

The package is native (`3.0 (native)`), like the rest of this PPA:
`inside-container.sh` copies the checkout with the submodules' `.git` files
stripped, which leaves their contents as plain source.

## Vendored crates

Launchpad builders have no network. `vendor/` holds every crate the three
lock files involved need — `tdesktop_rust/Cargo.lock`,
`wallet-engine/bindgen/cpp/Cargo.lock` for the generator, and
`tlottie/Cargo.lock` for its test suite — with the ones no Linux amd64/arm64
build reaches replaced by stubs. `.cargo/config.toml` points cargo at it.

`tdesktop_rust/Cargo.lock` is wallet-engine's own lock file with the
dev-dependencies resolved away and tlottie added, so transitive versions are
the ones wallet-engine is tested with. Upstream's recipes run without
`--locked` and resolve afresh at build time instead.

Regenerate after any pin bump. The repository's own `.cargo/config.toml` must
be out of the way, or cargo looks for crates.io in the `vendor/` being
replaced:

```
cd tdesktop_rust
cp ../wallet-engine/Cargo.lock Cargo.lock
mv ../.cargo/config.toml ../.cargo/config.toml.off
cargo metadata --format-version 1 >/dev/null       # settle the lock file
rm -rf ../vendor
cargo vendor-filterer --platform=x86_64-unknown-linux-gnu \
                      --platform=aarch64-unknown-linux-gnu \
                      --sync "$PWD/../wallet-engine/bindgen/cpp/Cargo.toml" \
                      --sync "$PWD/../tlottie/Cargo.toml" ../vendor
mv ../.cargo/config.toml.off ../.cargo/config.toml
```

`--sync` needs absolute paths, or cargo reports the generator as "a member of
the wrong workspace". Then regenerate the `vendor/` paragraphs of
`debian/copyright` from each crate's declared `license`.

Every vendored crate keeps a `Cargo.toml.orig` that its `.cargo-checksum.json`
lists. `dh_clean` deletes `*.orig` as patch debris, so `debian/rules` passes it
`-XCargo.toml.orig`, and a lintian override covers
`source-contains-patch-failure-file`. For the same reason
`debian/source/options` must not strip `examples`: several crates ship one.
The two upstream `examples/` directories — Android, iOS and web demos with a
prebuilt `gradle-wrapper.jar` and `.wasm` blobs — are dropped by
`inside-container.sh` instead.

## Tests

tlottie's own suite runs from its manifest: the JSON parser and the SIMD blits
against their scalar reference. Five DoS tests are skipped: they assert the
render budgets tlottie 1.0.5 relaxed for 2000px canvases and were not updated,
and a bare upstream checkout of the pin fails exactly those five, on Rust 1.97
as well. wallet-engine's tests need dev-dependencies that are not vendored
(reqwest, proptest, ton_core, ...), so they do not run.

## Installed layout

```
/usr/lib/<triplet>/libtdesktop_rust.a
/usr/include/tlottie/tlottie.h
/usr/include/wallet_engine/wallet_engine.hpp
/usr/include/wallet_engine/wallet_engine.cpp
/usr/include/wallet_engine/wallet_engine_scaffolding.hpp
```

That is where Telegram Desktop's `cmake/external/{tdesktop_rust,tlottie,
wallet_engine}` look: `find_library(tdesktop_rust)`, then `find_path` with the
`tlottie` and `wallet_engine` suffixes, and it compiles `wallet_engine.cpp`
itself from next to the header. There is no `.so`, so `find_library` cannot
pick a shared one.

The archive embeds the Rust standard library, so nothing links it at runtime —
and **bumping this package requires rebuilding telegram-desktop**, which
build-depends on it with a version.

`cargo --print native-static-libs` reports `-lgcc_s -lutil -lrt -lpthread -lm
-ldl -lc`, all implicit for a `g++` link against glibc 2.34+.

## Moving to newer pins

Telegram Desktop pins both in `Telegram/build/prepare/prepare.py`:
`tlottieRevision` and `walletEngineRevision`, plus the `patches` commit that
supplies `wallet-engine.patch`.

```
git -C tlottie fetch && git -C tlottie checkout <tlottieRevision>
git -C wallet-engine fetch && git -C wallet-engine checkout <walletEngineRevision>
curl -fsSL -o debian/patches/wallet-engine.patch \
    https://raw.githubusercontent.com/desktop-app/patches/<patches-pin>/wallet-engine.patch
# re-vendor as above, if either Cargo.lock moved
dch -v 0~git<YYYYMMDD>.<short-sha>+ppa1~resolute1 --distribution resolute "..."
./publish.sh --test-build
```

Drop the cpuid patch once resolute's Rust reaches 1.96, and
`--ignore-rust-version` once it reaches wallet-engine's `rust-version`.

## Version scheme

`0~git<date>.<short-sha>+ppa1~resolute1`, where date and sha are those of
whichever of the two pins was committed last; the changelog names both. Two
shas do not fit: lintian's `package-has-long-file-name` fires above 80
characters, counted with the longest architecture name.

No hyphens: a native package version may not contain a Debian revision.
`+ppa1` marks it as ours. The changelog targets resolute; `publish.sh` uploads
the same version to stonking as `~stonking1`, which sorts above `~resolute1` and
is just as long.

## Prerequisites

- `podman`, `git`, `gpg` on the host
- `cargo-vendor-filterer` only when re-vendoring
- GPG key registered on Launchpad; fingerprint at the top of `publish.sh`
- The key must have **no passphrase**, or `debuild` hangs on pinentry inside
  the container
