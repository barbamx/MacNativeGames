# CLAUDE.md

## What this repo is

MacNativeGames ports The Legend of Zelda: Tears of the Kingdom (TOTK) to run natively on Apple silicon Mac, iPhone and iPad. It is not an emulator: the game's ARM64 code runs directly and is rewritten where the OS requires it, Switch services are answered by a small host, and NVN graphics are rebuilt on Metal. The repo holds no game content; `build_app.py` makes the app from the user's own game file and keys. See `README.md` for the full instructions.

## Rules for this user's session

- Keep everything inside this repo. Do not read, write or log to anything outside it (no `/tmp`, no `~/Games`). Ask first if something outside is needed. Use `files/` for inputs, work folder and logs.
- Never commit or print game files, keys or firmware. `.gitignore` covers `*.keys *.nsp *.nca *.app/ TOTK_build/`, but **not** the `files/` folder itself or `files/build.log`, so check `git status` before any commit.

## Local layout (this checkout)

- `files/The Legend of Zelda Tears of the Kingdom (NSP)(eShop)/*.nsp`: base game 1.0.0 (v0), NSP.
- `files/ProdKeys22.1.0/prod.keys`: keys (also `title.keys`).
- `Firmware22.5.0/` (repo root): unpacked firmware NCAs, used for the shared fonts.
- `files/TOTK_build/`: build work folder (intermediate files, can be deleted to free space).
- `files/Tears of the Kingdom.app`: the built app (about 17 GB).
- `files/build.log`: log of the last build.

## How the app was built (worked, 28 min, exit 0)

```sh
brew install python@3.14   # opus and libvpx were already installed; Xcode CLT present
python3.14 -u build_app.py \
  "files/The Legend of Zelda Tears of the Kingdom (NSP)(eShop)/The Legend of Zelda Tears of the Kingdom [0100F2C0115B6000][v0][US].nsp" \
  -k files/ProdKeys22.1.0/prod.keys \
  --firmware Firmware22.5.0 \
  --work files/TOTK_build \
  -o "files/Tears of the Kingdom.app" > files/build.log 2>&1
```

Run it in the background and read `files/build.log`; it prints a progress line about once a minute.

## What was learned

- The macOS system `python3` is 3.9; the script needs `python3.14` (Homebrew `python@3.14`).
- An NSP has no system shared fonts, so the build stops immediately with "NSP files do not include system shared fonts" unless `--firmware <dir of .nca>` or `--shared-fonts` is given. An XCI would not need this.
- Keys 22.1.0 with firmware 22.5.0 worked together; the key/firmware version mismatch was not a problem.
- `main.npdm` is accepted via its Nintendo signature when the dump differs from the pinned hash ("from another dump; using it"). That is expected, not an error.
- Passing `-o` to the final location matters: the 19,892 startup shaders are compiled into macOS's Metal cache for that exact path, and moving the app (even to `/Applications`) makes them compile again on first start. The Xcode Metal toolchain (`xcodebuild -downloadComponent MetalToolchain`) would bake them into the app instead.
- Disk: the volume had 41 GB free and the build fit, but only just. Plan on about 17 GB for the app plus the work folder.
- Phases: code rewrite (about 2 min), RomFS extraction and verify (17 GB), then shader preparation (about 15 min compile), then the macOS check.
- Verified working: the user launched the built app from `files/` and it runs.
- Moving the app after the build (the user moved it to `/Applications`) makes Metal's shader cache cold: the game showed "warming shaders and pipeline" for minutes. Moving is a move, not a copy, so deleting it afterwards removes the only copy (the 17 GB app ends up in the Trash).
- Recovery: re-running the same build command reuses `files/TOTK_build` and rebuilt the app in 8 min instead of 28, with the shaders compiled in about 3 s because the cache for that path was still warm. Keep the app in `files/` and open it there.
- Free disk dropped from 41 GB to 22 GB after the build, so keep the work folder only if more rebuilds are expected.
