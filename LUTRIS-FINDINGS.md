# SCSKiller + Lutris — problems and fixes

Covers SCSKiller 1.4.7, Lutris 0.5.22, and Proton 9–11 builds.

## 1. Games play, but nothing is recorded

Log line (`~/.local/share/SCSKiller/recorders.log`):

```
<game>: the recorder records nothing: its stamp couldn't be signed:
The folder where SCSKiller keeps its locks can't be trusted: every user can write to <dir>
```

**Cause.** SCSKiller signs the recorder's `scskiller.cfg`; signing is guarded by a named
mutex (`NamedMutex.Linux.cs`). Before using the lock it walks every ancestor of the lock
directory up to `/` (`NamedMutex.Unsafe` → `OwnPath.Node`) and refuses when any is group- or
other-writable, because another local user could replace or race the lock.

**Fix.** Every ancestor of the lock directory, including `/`, must be owned by root or the
user and must not be group/other-writable — `/` must be `755` or stricter:

```bash
sudo chmod 755 /
```

Never use `-R`: it rewrites the mode of every file on the system and breaks
setuid/ownership semantics.

**Image builds.** Staging a rootfs by copying its children
(`cp -a "$CHROOT"/* <mount>`, `docker/image-builder/entrypoint.sh:164`,
`entrypoint-multi.sh:152`) does not set the destination root directory's mode, so an
over-permissive staging directory can become `/` on the flashed system. The build must
`chmod 755` the root of the written rootfs.

## 2. Proton builds not found

Message: `its Wine version <v> isn't installed where Lutris keeps its builds`
(`LutrisSource.cs:246`).

**Cause.** SCSKiller probes only (`LutrisSource.Build`, `LutrisSource.cs:265-276`):

1. `<lutris data>/runners/wine/<version>`
2. `<lutris data>/runners/proton/<version>`
3. `<SteamRoot>/compatibilitytools.d/<version>`, with
   `SteamRoot = realpath(~/.steam/root) ?? ~/.local/share/Steam`

Lutris 0.5.22 installs downloaded GE-Proton builds into
`$XDG_DATA_HOME/Steam/compatibilitytools.d` (`lutris/util/wine/proton.py:get_steam_compat_dir`).
With a distribution-packaged Steam, `realpath(~/.steam/root)` can point to an installation
folder without `compatibilitytools.d`, so SCSKiller never looks where Lutris writes. Steam's
own builds in `steamapps/common` are likewise not probed.

**Fix.** Make the build visible at a probed path:

```bash
ln -s ~/.local/share/Steam/compatibilitytools.d ~/.steam/root/compatibilitytools.d
```

Or symlink a single build into `<lutris data>/runners/wine/<exact wine.version>`. The name
must match the yml exactly: SCSKiller has no `-x86_64` suffix fallback, while Lutris's
`find_proton_version` accepts both forms. For a Steam build not in `compatibilitytools.d`,
copy or link `steamapps/common/<Proton build>` under one of the probed paths.

**Note.** Sharing Steam's folder does not mean Steam runs the game. Lutris runs the build
through umu with `PROTONPATH=<path>` and its own WINEPREFIX; the folder is just the
conventional location for Proton builds (umu, ProtonUp-Qt and Lutris all use it).

## 3. A game reports "not supported"

Independent of the selected Proton.

**Cause.** The status comes from the Unreal reader (`UnrealReader.Detect`). The UI label
"Engine not supported yet" (`Format.cs:216`) summarizes one of:

- `Unreal <version> isn't supported yet` — engine newer than SCSKiller knows
- `encrypted game files (needs the game's AES key)`
- `no shaders found (...)`
- `no D3D shaders (<platforms>)`

`RecorderSkip` (`ScsKiller.cs:6838-6852`) returns `SkipUnsupported` for these, so changing
Proton cannot fix it.

**Identify the reason:**

```bash
d=$(ls -d ~/.local/share/SCSKiller/games/*<game-id>* | head -1)
grep -oE '"(Unsupported|Version|Fork|Engine|GraphicsApi)":"[^"]*"' "$d/state.json" | sort -u
```

**Fix.** An unsupported engine needs a newer SCSKiller; encrypted files need the game's AES
key; a missing-shader case means that build is not supported.

## 4. Compile refused: prefix setting mismatch

Messages when adding a game to the queue:

```
failed, not ready: its prefix was last set up with use_dxvk_d3d8 True and a compile's run
has False, which would set it up again
failed, not ready: its prefix was last set up with another builtin_dll_copy than a compile's
run has, which would set it up again
failed, not ready: its prefix's config_info isn't what its Proton writes: start it once from Steam
```

These are prefix bookkeeping, unrelated to the game's graphics API.

**Cause.** Before compiling, SCSKiller compares every line of the prefix's `config_info`
(written by Proton) with `prefix_info`, which it computes by statically parsing the Proton
script (`ProtonScript.Linux.cs`, `ConfigInfoChange`), and refuses when they differ — Proton
would otherwise re-run prefix setup mid-compile. It does not model what launchers and Proton
forks change at runtime:

- **Proton 11 under Lutris:** Lutris exports `PROTON_DXVK_D3D8=1` for every DXVK-enabled
  launch (`lutris/runners/wine.py:1381-1382`), so Proton records `True` for
  `use_dxvk_d3d8` (the 14th and last `config_info` line). A compile's launch drops
  `PROTON_DXVK_D3D8` (`ProtonLaunch.Linux.cs:243`) and SCSKiller hardcodes the expected
  value to `False` (`ProtonScript.Linux.cs:203`). Every such prefix mismatches.
- **Proton 10 via UMU:** the Proton script itself sets
  `os.environ["PROTON_DLL_COPY"] = "*"` before reading it, so the prefix records `*` for
  `builtin_dll_copy`; SCSKiller computes the script's fallback DLL list.

Only the named line diverges; relaunching or switching Proton 10/11 builds does not help.
SCSKiller 1.4.7 is affected.

**Fix (Proton 11 under Lutris).** Set `PROTON_DXVK_D3D8=0` in the game's environment
(Lutris → Configure → System options → Environment variables), launch the game once so
Proton writes `config_info` itself, then compile. Lutris only sets variables that are
absent, so this persists across launches; the compile run's `False` then matches. It only
disables DXVK's D3D8 path, so it does not affect D3D9/10/11/12 titles. Launch the game
again only after compiling.

**Do not hand-edit `config_info`.** Proton writes it one value per line with no trailing
newline; an editor-added newline makes the line count differ from the computed 14 and
triggers the "isn't what its Proton writes" refusal.

**Report upstream** with: SCSKiller version, launcher, Proton builds tried, the exact
message, and the prefix's recorded value versus the compile run's computed value.

## Proton version notes

- `ge-proton` ("GE-Proton (Latest)") is a virtual umu sentinel, not a directory; umu
  downloads and updates it at launch. Playable/recordable, but SCSKiller cannot resolve it
  to a Proton folder.
- A concrete build such as `GE-Proton11-12-x86_64` is a directory both Lutris and
  SCSKiller can resolve (after the Problem 2 fix).
- `proton-latest` is a Steam-only alias; Lutris has no such version key.
- Updates: `GE-Proton (Latest)` is automatic through umu; concrete builds are manual
  (ProtonUp-Qt, or release archives extracted into
  `~/.local/share/Steam/compatibilitytools.d`); third-party forks are re-downloaded into
  `<lutris data>/runners/wine/`.
- Use the newest concrete build, e.g. `GE-Proton11-12-x86_64`, together with the Problem 4
  fix. Protons 10 and 11 are the builds current games need; older builds only avoid the
  mismatch by lacking the newer prefix fields.
