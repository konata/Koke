# Koke

Standalone Android Studio projects for reading individual AOSP packages /
modules at specific platform versions — no full AOSP build required.

Each instance is a self-contained AS project named `<Module>[@<Version>]`
(e.g. `MediaProvider@V`, `Profiling`). Instances are independent by
design — separate `.idea/`, indexes, and searches — so browsing one
module never pollutes another.

## Layout

```
Koke/                              # this repo
├── build.gradle                     # repo-level tasks: config, make
├── shared/
│   ├── app.gradle                   # per-instance: android{} + link / unseal / restore
│   ├── modules/
│   │   ├── MediaProvider/
│   │   │   ├── _.gradle             # common across versions
│   │   │   ├── V.gradle             # V-specific (sdk override, extra deps)
│   │   │   └── B.gradle
│   │   ├── Permission/{_,V}.gradle
│   │   └── Profiling/_.gradle       # single-version: just _.gradle
│   ├── prebuilts/
│   │   ├── _.pin                    # baseline dep-version catalog
│   │   └── V.pin                    # (optional) per-version overrides
│   └── catalog.properties           # per-version AOSP tag + compileSdk
├── Template/                        # pristine instance; never edit
├── MediaProvider@V/                 # instance (rubber-stamp of Template)
└── Profiling/
```

Every instance's tracked files (`build.gradle`, `settings.gradle`, wrapper,
etc.) are identical — the instance's identity comes from `rootDir.name`,
and its config is loaded from `shared/modules/<Module>/…`.

Instance directories are gitignored (regenerable via `:make`).

## Source-tree convention

Each instance's AOSP checkout lives as a **sibling of the Koke repo**,
named identically:

```
Koke/                 # this repo
MediaProvider/        # AOSP checkout  ─┐
Profiling/            # AOSP checkout  ─┴  matches instance names inside /
```

## Workflow

Repo-level tasks (run from the Koke repo root):

```bash
./gradlew config -Pmodule=MediaProvider@V   # scaffold _.gradle + V.gradle if absent
./gradlew make   -Pmodule=MediaProvider@V   # rubber-stamp Template → instance dir
./gradlew make                              # bulk: one instance per existing config
```

`config` declares a module, `make` materializes instances. Run `config`
**once per new module** (or new version of an existing one). Run `make`
whenever you want the instance dir (re)created — it's idempotent, and
skips dirs that already exist. `make` refuses to stamp an instance whose
module config doesn't exist (fail fast, no orphans).

Then, inside the instance:

1. Edit `shared/modules/<Module>/…` — list the `pin(...)` deps you need.
2. Ensure the sibling AOSP checkout exists.
3. Open `<Module>[@<Ver>]/` in Android Studio, Sync.
4. `./gradlew link` — materialize symlinks under `externals/`.
5. Sync again — AS picks up the source.

## Module config

`_.gradle` holds version-invariant fields; each `<Ver>.gradle` adds
version-specific overrides via `+=`. Adding a new version is append-only.

```groovy
// shared/modules/MediaProvider/_.gradle
ext.modules = [
    applicationId : 'com.android.providers.media.module',
    manifest      : 'MediaProvider/AndroidManifest.xml',
    src           : 'src|java',      // regex; matching dirs become java.srcDirs
    extraSymlinks : [],              // [[name: ..., path: ...], ...] for :link
]

dependencies {
    implementation pin('androidx.appcompat_appcompat')
}
```

```groovy
// shared/modules/MediaProvider/V.gradle
ext.modules += [sdk: 35]

dependencies {
    implementation pin('androidx.work_work-runtime')
}
```

## Platform catalog: `catalog.properties`

Per-version AOSP metadata. The `<Ver>` key (or `_` for single-version) drives
`compileSdk` and is the expected git tag for `:link`'s verification:

```properties
B.tag=android-16.0.0_r1
B.sdk=36
V.tag=android-15.0.0_r1
V.sdk=35
_.tag=android16-qpr2-release
_.sdk=36
```

On every Sync (and before any task), each symlink target's `git tag
--points-at HEAD` is checked against `<Ver>.tag` — mismatches and
non-git checkouts log a warning but never fail. Override `sdk` locally via
`modules.sdk` in a `<Ver>.gradle` if needed.

## Version catalog: `.pin` files

`shared/prebuilts/<Ver>.pin` is a verbatim dump of the AOSP
`prebuilts/sdk/current/androidx/m2repository/` layout:

```
androidx/appcompat/appcompat/1.7.0-beta01
androidx/work/work-runtime/2.9.0
```

Paste straight from AOSP — the loader splits `group/.../artifact/version`.
Lookup prefers `<Ver>.pin`, falls back to `_.pin`.

### `pin(name)`

Takes AOSP-style `<group>_<artifact>` (first `_` separates) and returns
the GAV coordinate. Missing entries silently no-op, so you can paste an
AOSP `Android.bp` / makefile dep list verbatim — AOSP-internal libs
without Maven coords drop out without breaking Sync.

```groovy
implementation pin('androidx.work_work-runtime')  // → androidx.work:work-runtime:2.9.0
implementation pin('modules-utils-build')         // not on Maven → skipped
```

## Tasks

**Repo-level** (`./gradlew …` from Koke repo root):

- `config -Pmodule=<Name>[@<Ver>]` — scaffold module config skeleton(s).
- `make [-Pmodule=<Name>[@<Ver>]]` — rubber-stamp Template into instance
  dir(s). With arg: one; without: bulk from every existing config.

**Per-instance** (`./gradlew …` from instance root):

- `link` — (re)create symlinks under `externals/` for the sibling AOSP
  source (and any `extraSymlinks` entries).
- `unseal` — replace the SDK's `android.jar` with a
  [konata/UnsealedSdk](https://github.com/konata/UnsealedSdk) variant so
  `@hide` / `@SystemApi` symbols resolve. **Global: affects every
  project on this machine using the same compileSdk.**
- `restore` — swap back to the official `android.jar`.

## Design principles

- **No compilation goal.** Sync must work; builds probably won't. Code
  browsing and symbol resolution are the only deliverables.
- **Instances are rubber-stamps.** All divergence lives in `shared/`;
  adding a module never edits existing instances.
- **AOSP-native vocabulary.** Dep names follow AOSP makefile syntax, not
  Maven. Source trees sit as siblings of the repo.
