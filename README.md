# Graal AUR packages

AUR packages for the latest [GraalVM](https://github.com/oracle/graal) language
native-standalone releases (**25.2.4**, GraalVM 25 Innovation 2, 2026-07-28).

All distributions are **Native Standalone** tarballs: relocatable trees (the
launchers resolve their location via `/proc/self/exe` or `$ORIGIN` rpath) with
a pre-compiled native image — no JDK required.

## Packages

Each language has two editions: `-bin` (GraalVM Community Edition, **UPL v1.0**,
freely redistributable, AUR-submittable) and `-oracle-bin` (Oracle GraalVM,
**GraalVM Free Terms and Conditions (GFTC)**, redistribution restricted — for
local/internal use only). Each pair `conflicts` with its counterpart.

| Package                | Language / runtime         | Runtime in `/usr/` | Exposed commands                                          |
|------------------------|----------------------------|--------------------|------------------------------------------------------------|
| `graalpy-bin`          | Python 3.12                | `lib/graalpy`      | `graalpy`, `graalpy-config`, `graalpy-polyglot-get`          |
| `graalpy-oracle-bin`   | Python 3.12                | `lib/graalpy`      | _same_                                                    |
| `graaljs-bin`          | ECMAScript (GraalJS)       | `lib/graaljs`      | `graaljs`                                                   |
| `graaljs-oracle-bin`   | ECMAScript (GraalJS)       | `lib/graaljs`      | `graaljs`                                                   |
| `graalnode-bin`        | Node.js 24 (GraalNode)     | `lib/graalnode`    | `graalnode`, `graalnode-npm`, `graalnode-npx`                |
| `graalnode-oracle-bin` | Node.js 24 (GraalNode)     | `lib/graalnode`    | `graalnode`, `graalnode-npm`, `graalnode-npx`                |

The system `python`/`python3`, `node`/`npm`/`npx` are left untouched — the
Graal binaries are exposed under prefixed names. Create a GraalPy venv to get
`python3.12` + `pip`:

```sh
graalpy -m venv .venv && .venv/bin/pip install ...
```

GraalNode ships its own bundled npm (`graalnode-npm`) and node headers under
`/usr/lib/graalnode/include/` for node-gyp.

## Drop-in replacement packages

A second set that installs the **bare** system command names and replaces the
CPython / Node.js / SpiderMonkey packages at the pacman level:

| Package                    | Replaces at pacman level             | Exposed commands      |
|----------------------------|--------------------------------------|------------------------|
| `python-graalpy-bin`       | `python`, `python3`                  | `python`, `python3`    |
| `python-graalpy-oracle-bin`| `python`, `python3`                  | _same_                |
| `js-graaljs-bin`           | `js` (SpiderMonkey)                  | `js`                   |
| `js-graaljs-oracle-bin`    | `js` (SpiderMonkey)                  | _same_                |
| `node-graalnode-bin`       | `nodejs`, `npm`                      | `node`, `npm`, `npx`   |
| `node-graalnode-oracle-bin`| `nodejs`, `npm`                      | _same_                |

These use `provides=` (e.g. `python=3.12`, `node=24`, `npm=11.13`, `js`) and
`conflicts=` (e.g. `python`, `python3`, `nodejs`, `npm`, `js`) so other
packages' dependencies resolve to the Graal implementation. Install only **one**
set at a time — every `-bin`, `-oracle-bin`, and drop-in variant of a runtime
`conflicts` with the others (they share `/usr/lib/<name>`).

Notes:

- Creating a venv still gives you real `pip`: `python -m venv .venv`.
- GraalPy does **not** ship a `python3-config`; build systems that require one
  (e.g. some C-extension builds) will not work. The Graal launchers dispatch on
  `argv[0]`, so symlinking to names like `python3-config` does **not** behave
  like CPython's.
- The bundled `npm`/`npx` are self-contained; `npm` finds `node` via `PATH`.
- Arch's `python` provides a versioned `python=3.13`; providing `python=3.12`
  means dependencies requiring a newer CPython won't resolve to this package.

## Other Graal languages

- **TruffleRuby** — no standalone is published (JVM component only, maintenance mode).
- **FastR** — no current release (archived at 22.3.1).
- **LLVM / Sulong**, **WebAssembly**, **Espresso** — components of the full
  GraalVM JDK. Espresso *does* publish standalone tarballs on
  `gds.oracle.com`, but the latest referenced by the docs (25.1.3) is not yet
  available (max published: 25.0.4), so it is not packaged here.

## Updating

Bump `pkgver`, and recompute the per-arch sha256 sums (example for graaljs CE):

```sh
P=25.2.4
for A in amd64 aarch64; do
  curl -L "https://github.com/oracle/graaljs/releases/download/graal-$P/\
graaljs-community-$P-linux-$A.tar.gz" | sha256sum
done
```

## Layout notes

- Runtime goes to `/usr/lib/<name>`; `/usr/bin/<prefix>` entries are symlinks.
- GraalJS: `bin/js` + `lib/libjsvm.so`. GraalNode: `bin/node` (self-contained
  native image, `RPATH=$ORIGIN/../lib/`) + `lib/libgraal-nodejs.so`,
  `lib/libjsig.so`, `include/`, `npm/`.
- `*-polyglot-get` launchers are present in the trees but are **not** linked
  into `/usr/bin`: they only work in JVM-based distributions and print an error
  on native standalones.
- Runtime shared libraries only require `gcc-libs` and `zlib`.
- Stripping is disabled (`options=('!strip')`) to be safe with native-image
  binaries.