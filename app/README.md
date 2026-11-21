# Series app

A read-only Tauri v3 viewer for one classification-series repository. The
same `app/` directory is stamped unchanged into every repository of the
series; the repository name supplies the identity (app id, product name,
icon code, variant). The repository's own `blueprint.edn` is checked
against that name — its id must equal the name, or the name with the
`<scheme>-` part removed — and supplies the title verbatim. The product
name is the title with each of `\ / : * ? " < > |` replaced by a space
(whitespace runs collapsed, trimmed): the window title and the page show
the verbatim title while Tauri gets the sanitized product name. Control
characters in the name, or an empty product name, fail closed.

## What the app shows

Four tabs, all in Japanese UI labels, all content shown as it is in the
sources:

- 概要: every field of the repository's `blueprint.edn`.
- 操作卓の見本: the `docs/samples/operator-console.html` sample, shown only
  when the repository has one. The tab says it is a sample, not live data.
- 物理仕様: the `physics.edn` cases, shown only when the repository has
  `physics.edn`.
- この app について: id, version, source digests, and the statement that
  the app performs no external communication. For an ISCO-08 repository it
  also shows the ISCO-08 attribution (ja and en: source, code, and that the
  app is not produced, reviewed or endorsed by the ILO), built from the
  repository's own code, never from a per-repository literal. No ILO logo
  or ILO imagery.

With trusted-page-v1 the selected tab is remembered in localStorage (key
`itonami.series.<slug>.v1`, the tab id only); the app still renders when
storage throws. With restricted-wasm-v1 the tab lives in memory and resets
on each launch, because the sandboxed guest has no localStorage.

## Runtime profile (KOTOBA-159)

`itonami.runtime` in `package.json` names the client runtime profile.

- `restricted-wasm-v1`, for every new repository (`scripts/scaffold.mjs` writes
  it on a fresh stamp). `index.html` is a trusted host page
  (`src/host.html`, `src/host.js`) with the canonical runtime from
  `vendor/restricted-wasm-v1/`. The host verifies the guest document
  against the SHA-256 and source commit pinned at build time, then mounts it
  with `mountRestrictedClientApp` in an opaque-origin sandbox. The guest
  is one self-contained document (`src/guest.html`, plus `src/view.js` and
  `src/guest.js` inlined as one script). The page policy admits exactly that
  script, the runtime's preamble and the guest style, by hash. The
  descriptor's `local-state` adapter is `memory`. The build needs the source
  commit: build from a git checkout, or set `ITONAMI_SOURCE_COMMIT`.
- `trusted-page-v1`, only for the existing series apps listed in
  `scripts/trusted-page-v1-existing.txt`, and only through 2027-03-31 (UTC).
  A `package.json` without `itonami.runtime` predates the field and is read
  as trusted-page-v1; a re-stamp keeps the profile it finds. These apps keep
  the runtime in `vendor/` at its earlier pin. The build refuses
  trusted-page-v1 for an app that is not listed, and for every app from
  2027-04-01.

## Where its data comes from

`node scripts/build.mjs` reads the repository name from `itonami.source`
in `package.json`, plus `blueprint.edn`, `physics.edn` (when present)
and `docs/samples/operator-console.html` (when present) next to `app/`,
pins each source by SHA-256 digest in `dist/release.json`, and writes the
page's `dist/data.js`. `release.json` also says which commit was built:
`source_commit` is the 40-hex `git rev-parse HEAD` of the repository root
and `dirty` is true when `git status --porcelain` lists a tracked change or
an untracked file that is not ignored. Outside a git checkout the build
takes `ITONAMI_SOURCE_COMMIT` (40 hex) and `ITONAMI_SOURCE_DIRTY` (`true` or
`false`) when they are set, and writes `null` otherwise. The app carries no second, hand-written copy of the
data. The build refuses to run when `package.json` or
`src-tauri/tauri.conf.json` disagree with the derived identity, or when the
blueprint id does not match the repository name; re-run
`node scripts/scaffold.mjs` in that case.

## Consent and network

No external operation of any kind. The shipped descriptor declares no
operations, and the page only loads its own files (`script-src 'self'`,
`style-src 'self'`, plus the guest's hashed inline script and style under
restricted-wasm-v1). An operation added later must go through the vendored
canonical broker with per-operation consent. There is no product-local
consent logic and no remote fallback.

## Build and test

- `npm ci`
- `npm run build`: static build into `app/dist/`.
- `npm test`: builds, then runs `node --test test/*.test.mjs`.
- `npm run build:desktop`: static build, then a Tauri desktop build.
- `npm run build:android`: via `sh scripts/android-apk.sh` (needs the
  Android SDK + NDK 27, JDK 21, cargo/rustc on PATH, and the
  `aarch64-linux-android` Rust target). Each run starts from a fresh
  `tauri android init` (`src-tauri/gen/android/` is not in git) and writes the
  committed `android/MainActivity.kt` (edge-to-edge insets and system-bar
  colours from the page's `--bg`) into it, with the package taken from
  `tauri.conf.json`'s identifier. `scripts/android-manifest.mjs` then removes
  `android.permission.INTERNET` (`tools:node="remove"`): the app makes no
  network request, and the Store lists every permission an APK asks for. After
  the build, `aapt2 dump permissions` must show no INTERNET (aapt2 from the
  newest Android SDK build-tools is required).
- `src-tauri/Cargo.lock` is kept in the order cargo writes it
  (`test/cargo-lock.test.mjs`), so a build leaves the checkout clean. After a
  dependency change, let cargo rewrite it (for example
  `cargo update --workspace --offline` in `src-tauri/`, which changes no locked
  version) and commit the result. `cargo metadata --locked` does not detect an
  order-only difference.
- `tauri android dev` loads the page from a development server and needs
  INTERNET. It does not run `android-apk.sh`, so its manifest keeps the
  permission; only the APK built by `npm run build:android` drops it.
- `node scripts/scaffold.mjs <repo-root> [--repo <name>] [--no-icons]`:
  stamp this template into a series repository and rewrite its identity
  files from the repository name and its `blueprint.edn`. The name comes
  from `--repo`, else from the last path segment of the git remote
  origin URL (`.git` removed); the directory name is never used. The
  scaffold writes the name into `itonami.source` in `package.json`.

## Limits

- Read-only: the host exposes one command, `awai_host_info`, and no
  filesystem, network, shell or installer permission.
- The Android build is not yet verified in this repository.
