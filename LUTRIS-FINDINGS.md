# SCSKiller + Lutris 0.5.22 — findings and fixes

Analyzed: SCSKiller v1.4.7 (main), Lutris tag `v0.5.22`, and the local image/loader build
scripts.

## Problem 1 — games play, but nothing is recorded

Log line in `~/.local/share/SCSKiller/recorders.log`:

```
<game>: the recorder records nothing: its stamp couldn't be signed:
The folder where SCSKiller keeps its locks can't be trusted: every user can write to <dir>
```

**Cause.** SCSKiller signs the recorder's `scskiller.cfg`; signing is guarded by a named
mutex (`NamedMutex.Linux.cs`). Before using the lock, it walks every ancestor of the lock
directory up to `/` (`NamedMutex.Unsafe` → `OwnPath.Node`) and refuses to sign when any of
them is group- or other-writable, because another local user could then replace or race the
lock.

**Fix.**

- Every ancestor of the lock directory, including `/`, must be owned by root or the user and
  must not grant group/other write. In practice `/` must be `755` or stricter.
- Change only the offending directory itself:
  ```bash
  sudo chmod 755 /
  ```
- Never use `-R`: a recursive change rewrites the mode of every file on the system and breaks
  setuid/ownership semantics.
- Image builds: staging a rootfs by copying its children
  (`cp -a "$CHROOT"/* <mount>`, as in `docker/image-builder/entrypoint.sh:164` and
  `entrypoint-multi.sh:152`) does not set the destination root directory's own mode, so an
  over-permissive staging directory can become `/` on the flashed system. The image build
  must apply `chmod 755` to the root directory of the written rootfs.

## Problem 2 — Proton builds not found

Message: `its Wine version <v> isn't installed where Lutris keeps its builds`
(`LutrisSource.cs:246`).

**Cause.** SCSKiller probes (`LutrisSource.Build`, `LutrisSource.cs:265-276`):

1. `<lutris data>/runners/wine/<version>`
2. `<lutris data>/runners/proton/<version>`
3. `<SteamRoot>/compatibilitytools.d/<version>`, with
   `SteamRoot = realpath(~/.steam/root) ?? ~/.local/share/Steam`

Lutris 0.5.22 installs downloaded GE-Proton builds into
`$XDG_DATA_HOME/Steam/compatibilitytools.d`
(`lutris/util/wine/proton.py:get_steam_compat_dir`). With a distribution-packaged Steam,
`realpath(~/.steam/root)` can point to an installation folder that has no
`compatibilitytools.d`, so SCSKiller never looks where Lutris writes.

**Fix.** Make the compatibility-tools folder visible at the resolved Steam root:

```bash
ln -s ~/.local/share/Steam/compatibilitytools.d ~/.steam/root/compatibilitytools.d
```

Alternative: symlink a single build into `<lutris data>/runners/wine/<exact wine.version>`.
The name must match the yml exactly; SCSKiller has no `-x86_64` suffix fallback, while
Lutris's `find_proton_version` accepts both forms.

**Note.** Sharing Steam's `compatibilitytools.d` does not mean Steam runs the game. Lutris
runs the build directly through umu with `PROTONPATH=<path>` and its own WINEPREFIX; the
folder is simply the conventional Linux location for Proton builds (umu, ProtonUp-Qt and
Lutris all use it).

## Problem 3 — a game reports "not supported"

This is independent of the selected Proton.

**Cause.** The status comes from the Unreal reader (`UnrealReader.Detect`). The UI label
"Engine not supported yet" (`Format.cs:216`) summarizes one of:

- `Unreal <version> isn't supported yet` — engine build newer than SCSKiller knows
- `encrypted game files (needs the game's AES key)`
- `no shaders found (...)`
- `no D3D shaders (<platforms>)`

`RecorderSkip` (`ScsKiller.cs:6838-6852`) returns `SkipUnsupported` for these, so changing
Proton versions cannot fix it.

**Identify the exact reason:**

```bash
d=$(ls -d ~/.local/share/SCSKiller/games/*<game-id>* | head -1)
grep -oE '"(Unsupported|Version|Fork|Engine|GraphicsApi)":"[^"]*"' "$d/state.json" | sort -u
```

**Fix.** Depends on the reason: an unsupported engine version requires a newer SCSKiller;
encrypted files require the game's AES key; a missing-shader case means the build is not
supported.

## Proton version notes

- `ge-proton` ("GE-Proton (Latest)") is a virtual umu sentinel, not a directory. umu
  downloads and updates it at launch. Playable/recordable, but SCSKiller cannot resolve it
  to a Proton folder.
- A concrete build such as `GE-Proton11-12-x86_64` is a directory both Lutris and SCSKiller
  can resolve (after the Problem 2 fix).
- `proton-latest` is a Steam-only alias; Lutris has no such version key.
- Updates: `GE-Proton (Latest)` is automatic through umu; concrete builds are manual
  (ProtonUp-Qt, or releases extracted into `~/.local/share/Steam/compatibilitytools.d`);
  third-party forks are re-downloaded into `<lutris data>/runners/wine/`.
