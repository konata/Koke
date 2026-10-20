# Koke

Read one AOSP repository in its own Android Studio project, without building AOSP.
Koke connects an existing checkout to a small Gradle project and supplies source
roots, an Android SDK and the Maven dependencies you configure.

The normal workflow is **prepare the checkout → `make` → open and Sync**.
Optional version profiles select different configurations while sharing the same
checkout. Koke never clones repositories or switches Git revisions.

A project combines the initial files from `seed/`, the module's `profile/*.gradle`
configuration and links to its source checkout. Common version and dependency
catalogs live in `catalog/`; generated IDE projects live in `modules/`.

[Quick start](#quick-start-read-an-existing-module) ·
[Configuration recipes](#configuration-recipes) ·
[Version profiles](#advanced-version-profiles) ·
[Troubleshooting](#troubleshooting) ·
[Commands](#commands-and-layout)

## Quick start: read an existing module

### 1. Prepare your environment and checkout

You need Android Studio, Git, an Android SDK and a JDK compatible with the pinned
Gradle/AGP versions. The CLI workflow is tested with JDK 21. `java -version` should
work in the terminal; use a compatible Gradle JDK in Android Studio as well.

Make the SDK visible to command-line Gradle, for example on macOS:

```bash
export ANDROID_HOME="$HOME/Library/Android/sdk"
```

An instance's `local.properties` with `sdk.dir=...` is another option. Ordinary
instances currently select SDK 36; the V profile selects SDK 35. Install the SDK
selected by `catalog/versions.properties`, or override it in the module config.
The first invocation may download Gradle and plugins.

Place the AOSP checkout next to Koke, using the module name:

```text
workspace/
├── Koke/                          # this repository
│   └── modules/
│       └── MediaProvider/         # generated Android Studio project
│           └── externals/MediaProvider -> workspace/MediaProvider
└── MediaProvider/                 # your AOSP checkout
```

Choose the source revision yourself. A configured expected Git ref produces a
warning if the checkout does not match; Koke does not change it.

### 2. Prepare the IDE project

Run from the Koke root:

```bash
./gradlew make -Pmodule=MediaProvider
```

MediaProvider already has `profile/MediaProvider.gradle`. `make` creates its
instance if needed, then runs that instance's `link` task to connect the source.
Running it again preserves the instance's files and refreshes its links.

The preparation summary shows:

- Actual checkout locations and missing checkouts.
- Selected SDK and expected Git ref, if configured.
- Source and resource root counts, plus missing configured paths.
- How many requested pins map to Maven coordinates, and the names that do not.
- The exact project directory to open.

For example, a module with three discovered source roots and an unmapped internal
AOSP dependency reports lines like these:

```text
Koke: Example
  Source: /work/Example -> externals/Example
  SDK: android-36
  Sources: 3 roots (automatic)
  Resources: 1 roots
  Pins: 1 mapped, 1 skipped
    modules-utils-build: no Maven coordinate; symbols need source or another dependency
  Open in Android Studio: /work/Koke/modules/Example
```

Missing source trees or configured paths are reported as **Preparation incomplete**.
They remain warnings so you can prepare an instance before obtaining its source.
Invalid Gradle configuration or a failed child task makes `make` fail.

### 3. Open and Sync

Open `workspace/Koke/modules/MediaProvider/` in Android Studio, then Sync. The source
checkout at `workspace/MediaProvider/` is a different directory; open the generated
project to get Koke's configuration.

Preparation success confirms the setup tasks ran. It does **not** establish that
all dependencies downloaded or every symbol resolves. Most shipped module configs
use shared defaults; they are starting points, not complete AOSP dependency graphs.

## Daily workflow

| Change | What to do |
| --- | --- |
| Continue reading the same checkout | Open the existing IDE project |
| Change a module's sources, resources or dependencies | Sync; rerun `make` if you also want its summary |
| Add or change extra source links | Rerun `make -Pmodule=...`, then Sync |
| Switch the checkout's Git revision | Open the matching profile if applicable, then Sync |
| Update `seed.gradle` | Existing instances use it on their next Gradle configuration |
| Change `seed/` | New instances receive it; existing instance files are not overwritten |

`./gradlew make` prepares one **unversioned** instance for each `profile/*.gradle`
file whose name has no `@`. Version overrides are selected explicitly.
Use a named module during setup or diagnosis to keep the output focused.
`--offline` is forwarded to the instance build; its Gradle distribution and plugins
must already be cached.

## Add a module

Configuration is a one-time maintainer step. From the Koke root:

```bash
./gradlew config -Pmodule=Example
# Edit profile/Example.gradle only if defaults need changing.
# Place the source checkout at ../Example and select its Git revision.
./gradlew make -Pmodule=Example
```

The presence of `profile/Example.gradle` declares the module. A comment-only
file is enough when defaults work. `config` creates that flat file and preserves
existing files. It does not infer dependencies from `Android.bp`.

Start with the smallest config, inspect the preparation summary, then add only
known source roots and dependencies. Store changes in `profile/`, not in
the generated instance's build script, so the instance stays reproducible.

## Configuration recipes

`modules` is Koke's configuration map, available in each profile script.
Its fields are consumed by `seed.gradle`; they are not Android Gradle
Plugin properties.

### Use the repository's manifest

The shipped MediaProvider config contains only its differences:

```groovy
modules.applicationId = 'com.android.providers.media.module'
modules.manifest = 'AndroidManifest.xml'
```

Manifest, explicit source roots and explicit resource roots are relative to
`externals/<Module>/`. They never include a version suffix.

### Choose source and resource roots explicitly

Koke resolves `modules.sources` and `modules.resources` into the Android Gradle
Plugin's `android.sourceSets.main.java.srcDirs` and `res.srcDirs`. Use these Koke
fields for root overrides in profiles: Koke assigns the plugin's source roots
after loading the profiles, so direct `srcDirs` assignments there are overwritten.

IntentResolver uses:

```groovy
modules.sources = ['java/src']
```

For a repository with several relevant components, a config could contain:

```groovy
modules.sources = ['service/java', 'framework/java', 'shim_src']
modules.resources = ['ui/res']
```

Use only paths that exist in that checkout. An explicit list replaces automatic
discovery for that kind of root. `modules.resources = []` disables resource
discovery; `modules.sources = []` does the same for source discovery.

Automatic discovery matches directory names against `src|java` and looks for
Java/Kotlin files inside candidates. It searches without a fixed depth limit and
stops below a matching root. Resource discovery matches `res`.

`modules.exclude` defaults to a regex that skips names containing `test`, plus
`.git`, `.gradle`, `.idea`, `build` and `out`, while discovering roots. It does not
filter files inside a selected root. Unusual roots such as `legacy-src` or
`src_full_lib` need explicit selection or a module-specific `modules.src` regex:

```groovy
modules.src = 'src|java|src_full_lib'
```

This changes automatic source discovery; an explicit `modules.sources` list
still takes precedence. Choose roots for the implementation you want to read:
combining alternative implementations can introduce duplicate classes. Koke does
not evaluate `Android.bp` conditions or infer Soong target boundaries.

### Add Maven dependencies

Declare dependencies with AOSP-style names in the module script:

```groovy
dependencies {
    implementation pin('androidx.appcompat_appcompat')
    implementation pin('androidx.work_work-runtime')
}
```

Pins come from AOSP Maven repository directory paths in `catalog/pins/_.pin`:

```text
androidx/appcompat/appcompat/1.7.0-rc01
androidx/work/work-runtime/2.10.0-alpha02
```

For example, `pin('androidx.appcompat_appcompat')` maps to
`androidx.appcompat:appcompat:1.7.0-rc01` with that catalog. Use versions appropriate
to your checkout. A Maven dependency absent from the catalog can also be declared
with a normal Gradle coordinate.

A name without a Maven mapping contributes an empty file collection. The summary
lists it as skipped, rather than claiming its symbols are available. If those
symbols matter, add their source or a suitable dependency. The summary counts
distinct names passed to `pin()`, not direct Gradle dependencies or transitive
artifacts.

For detailed mappings and root paths, run from the instance:

```bash
./gradlew link --info
```

### Include another source checkout

Given `workspace/libs/Helper`, add:

```groovy
modules.extraSymlinks = [[name: 'Helper', path: 'libs/Helper']]
```

This creates `externals/Helper -> workspace/libs/Helper`. `name` is the link name;
`path` is relative to Koke's parent directory. Automatic discovery includes extra
links. To choose roots explicitly across both checkouts:

```groovy
modules.sources = ['src', '../Helper/java']
```

Koke normalizes these paths relative to the main link before following symlinks.
Rerun `make` after editing extra links. `externals/` is managed by Koke: `link`
replaces its symlinks and refuses to overwrite real files or directories. Keep
hand-written files in the source checkout, not under `externals/`.

### Override the SDK

```groovy
modules.sdk = 35
```

This overrides the selected catalog SDK for that module or profile. It does not
change the checkout or make its APIs compatible with a different Android release.

## Advanced: version profiles

Use `IntentResolver@B` and `IntentResolver@V` when different releases need different
SDKs, dependency versions or module settings. Ordinary modules can stay unversioned.

Create the optional configs once:

```bash
./gradlew config -Pmodule=IntentResolver@B
./gradlew config -Pmodule=IntentResolver@V
```

This creates `profile/IntentResolver.gradle` if absent, plus
`profile/IntentResolver@B.gradle` and `profile/IntentResolver@V.gradle`.
The base `IntentResolver.gradle` runs once, followed by the chosen version script.
Keep common dependencies in the base file and version-specific dependencies in
the version file. For example:

```groovy
// profile/IntentResolver@V.gradle
modules.sources = ['java/src']
dependencies {
    implementation pin('androidx.work_work-runtime')
}
```

`catalog/pins/V.pin` can override artifact versions. Koke loads `_.pin` first,
then overlays the chosen version file by Maven group/artifact. Unmentioned baseline
entries remain available; later entries for the same artifact win. These catalogs
are shared by all modules using that profile, so keep module-specific dependency
selection in the module script.

Prepare a profile explicitly:

```bash
# Manually switch ../IntentResolver to the release required by the B config first.
./gradlew make -Pmodule=IntentResolver@B
```

Open `modules/IntentResolver@B/` in Android Studio. Both the base profile and the
selected version file must exist; Koke reports a missing profile before creating
the project.

Both profiles link `externals/IntentResolver` to the **same** `workspace/IntentResolver`
checkout. Switching that checkout changes the source visible to both projects.
The profiles isolate IDE metadata and configuration, not source revisions. After
switching releases, open the intended profile and Sync; creating two profiles does
not preserve two source snapshots.

`catalog/versions.properties` supplies `<Version>.sdk`, or `_.sdk` for an unversioned
instance. A new version needs its own `<Version>.sdk` entry or a `modules.sdk`
override; versioned instances do not fall back to `_.sdk`. An optional
`<Version>.tag` names a local Git tag or branch whose commit is compared with
`HEAD`. A missing or mismatching ref is reported as a warning.

## Hidden platform APIs

If reading platform code requires hidden Android symbols, run from an instance,
then Sync:

```bash
./gradlew unseal
```

To return to the saved official `android.jar` later:

```bash
./gradlew restore
```

`unseal` uses [konata/UnsealedSdk](https://github.com/konata/UnsealedSdk) and affects
**every project using that SDK**. It is never run automatically by `make`, `link`
or Sync.

Koke validates the downloaded or cached JAR, saves the original as
`android.official.jar`, and replaces files through a temporary file in the same
directory. `restore` can also recover a missing `android.jar`. This supplies API
symbols; it does not generate AIDL, proto, resource or other build-generated code.

## Troubleshooting

| Symptom | Check |
| --- | --- |
| `module profile missing` | Run `config -Pmodule=...`; an explicit profile needs both common and version files |
| Source marked `[missing]` | Check the sibling checkout's name and location, then rerun `make` |
| No Java/Kotlin roots | Confirm the checkout contains JVM code; declare unusual roots with `modules.sources` |
| `Missing path` | Correct the relative source/resource/manifest path in the module config |
| Expected Git ref warning | Check the source revision and selected profile; Koke only checks local refs |
| SDK location or platform missing | Set `ANDROID_HOME` or the instance's `sdk.dir`, and install the selected SDK |
| Pins skipped | Supply the relevant source or Maven dependency if you need those symbols |
| Sync works but some symbols stay unresolved | Check missing roots, dependency declarations, hidden APIs and generated-code requirements separately |
| Existing project keeps old seed files | `make` preserves them; apply needed seed changes explicitly or generate a new instance after preserving local settings |
| An old version-suffixed link remains | Rerun `make` to recreate stable `externals/<Module>` links |

## Commands and layout

| Run from | Command | Effect |
| --- | --- | --- |
| Koke root | `config -Pmodule=Name[@Version]` | Create absent base and optional version profile files |
| Koke root | `make -Pmodule=Name[@Version]` | Prepare `modules/Name[@Version]`, link sources, print its summary |
| Koke root | `make` | Prepare the unversioned instance of every configured module |
| Instance | `link` | Refresh source links and print the same summary |
| Instance | `unseal` / `restore` | Explicitly change the shared SDK JAR |

Each command is invoked through that directory's `./gradlew`.

```text
Koke/
├── build.gradle                  # config / make orchestration
├── seed.gradle                   # shared IDE model, links and SDK tasks
├── gradle/
│   └── wrapper/                  # root Gradle wrapper
├── seed/                         # initial files copied into new instances
├── catalog/
│   ├── versions.properties       # SDK / expected Git ref per version
│   └── pins/{_,V}.pin            # baseline and optional version pins
├── profile/
│   ├── <Name>.gradle             # base module profile
│   └── <Name>@V.gradle           # optional version overrides
└── modules/
    └── <Name>[@<Version>]/        # generated, gitignored IDE instance
```

Each directory under `modules/` is an independent Gradle project. Open that
directory in Android Studio. Changes to profiles, catalogs and `seed.gradle`
take effect on the next Gradle configuration; `seed/` is copied only when a new
project is created. Keep IDE state and local settings in `modules/`, and commit
reusable module settings to `profile/`.

When moving an older instance from `Koke/<Name>` to `Koke/modules/<Name>`, preserve
its files and change its `build.gradle` script reference to:

```groovy
apply from: "${rootDir}/../../seed.gradle"
```

Then run `./gradlew make -Pmodule=Name` from the Koke root and reopen the new
project path in Android Studio. The sibling source checkout stays in place.
