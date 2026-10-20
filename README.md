# Koke

AOSP in Android Studio, without the platform build.

Koke wraps source checkouts in small Gradle projects for browsing and navigation.
You handle Git. Koke handles the IDE setup.

## Use it

Keep checkouts next to Koke:

```text
workspace/
├── Koke/
│   └── modules/
│       ├── MediaProvider/       # IDE project
│       └── IntentResolver/      # IDE project
├── MediaProvider/              # source checkout
└── IntentResolver/             # source checkout
```

From `Koke/`:

```bash
./koke apply                 # create missing projects from base profiles
./koke apply MediaProvider   # set up or refresh one project
```

Open `modules/MediaProvider/` in Android Studio and Sync. Same deal for
`IntentResolver`.

Named `apply` creates missing profiles and projects, then refreshes links.
Existing files stay put. Batch `apply` only creates missing, unversioned projects.

## Make it yours

Settings live in `profile/<Name>.gradle`. An empty profile uses the defaults.
For IntentResolver-style layouts:

```groovy
self {
    sources = ['java/src']
    resources = ['java/res']
}
```

`self.sources = ['java/src']` works too. Both forms update Koke's configuration.
Profiles run normally; Koke reads the final `self` settings afterward and applies
them to AGP. Put source roots here, since Koke overwrites direct `srcDirs`
assignments made in profiles.

Paths are relative to `externals/<Name>/`, the checkout link inside the project.
Each explicit list **replaces** discovery; `[]` selects nothing.

By default, Koke finds `src` or `java` directories containing Java/Kotlin, plus `res`
directories. Discovery skips test directories and build metadata, and stops
below a matching root. Spell out unusual roots yourself. `self.src` and
`self.exclude` change discovery's directory-name regexes; exclusions don't
filter files inside a selected root.

Other settings:

| Field | Example |
| --- | --- |
| `applicationId` | `'com.android.providers.media.module'` |
| `manifest` | `'AndroidManifest.xml'` |
| `sdk` | `35` |
| `extraSymlinks` | `[[name: 'Helper', path: 'libs/Helper']]` |

Extra checkout paths start at Koke's parent directory. Their links go under
`externals/` and join automatic discovery. An explicit root like
`'../Helper/java'` selects that link. Rerun `./koke apply Name` after changing links.

## Dependencies

Use regular Gradle dependencies, with `pin()` for AOSP-style names:

```groovy
dependencies {
    implementation pin('androidx.appcompat_appcompat')
}
```

In `catalog/pins/_.pin`, this line:

```text
androidx/appcompat/appcompat/1.7.0-rc01
```

maps to `androidx.appcompat:appcompat:1.7.0-rc01`. Unmapped names are reported
and skipped. Bring the missing source or dependency yourself.

## Versions, if you need them

Stick to one profile until releases actually need different settings.

```bash
# Switch ../IntentResolver to the V release yourself, then:
./koke apply IntentResolver@V
```

Open `modules/IntentResolver@V/`. Koke loads the base profile, then
`profile/IntentResolver@V.gradle`; later `self` assignments win.
`catalog/pins/V.pin` overrides baseline pins by group and artifact.

SDKs come from `catalog/versions.properties`: `_.sdk` for plain projects,
`V.sdk` for `@V`. A new version needs its own entry or `self.sdk`; there is no
`_.sdk` fallback. Optional `<Version>.tag` entries warn when the checkout doesn't match.

**Versions share one checkout.** Switch Git yourself, open the matching project,
then Sync. Separate projects give you separate IDE settings, not separate source.

## Hidden APIs

Inside the generated project:

```bash
./gradlew unseal   # install the unsealed android.jar
./gradlew restore  # restore the saved original
```

`unseal` uses [UnsealedSdk](https://github.com/konata/UnsealedSdk) and changes
**the shared SDK**, affecting every project using it. Run it explicitly, then Sync.

## Keep expectations straight

Koke is a reading setup. It doesn't evaluate Soong targets or generate missing
code. Choose your source roots; dumping every variant into one project creates
duplicate classes. A successful Sync doesn't promise every symbol will resolve.

For resolved roots and pins, run `./gradlew link --info` inside the project.
