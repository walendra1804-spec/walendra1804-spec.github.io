---
title: "Running an Old 32-bit Windows Unity Game on an Apple Silicon Mac, Then Fixing Its Lag"
description: "How we got a Unity 4.6 Windows game running on an A18 Pro MacBook with Wine and Rosetta 2: why two Wine builds crashed, how moving DirectX 11 to Metal with DXMT reduced lag, and how a byte patch to its audio settings ended the FPS drops."
date: 2026-11-01 13:00:00 +0700
tags: [wine, macos, apple-silicon, rosetta, unity, dxmt, metal, gaming, troubleshooting]
---

*Setup: MacBook with an Apple A18 Pro and 8 GB of RAM, macOS 26.5. The game is a 32-bit Windows game from 2015, built with Unity 4.6.3 and Mono, with no Mac version.*

## TL;DR

| Symptom | Cause | Fix |
|---|---|---|
| Game crashes or hangs while loading Mono on Wine Staging 11.18 | Most likely the 32-to-64-bit mode switch failing under Rosetta | Switch to **WineCX 24.0.7** (the CrossOver-based engine from Sikarugir) |
| Game crashes with an `alloc_pages_vprot` assertion on Game Porting Toolkit (Wine 7.7), and only runs with audio turned off | A CoreAudio thread trips an assertion in Wine's memory code | Same: WineCX 24.0.7 runs it with sound |
| Wrapper app says "damaged or incomplete", has a broken icon, is missing from Spotlight | The `&` in the game's name, written unescaped into `Info.plist` | Generate the plist with `plutil` |
| FPS collapses when a lot is on screen | DirectX 11 was being translated to OpenGL by `wined3d` | Translate to Metal with **DXMT** |
| FPS drops every time you collect a coin | Most likely the coin sounds being decoded from Ogg as they play | Patch those clips from `m_Stream` 1 to 0 |

The result is that the game now plays smoothly, with sound. That is based on playing it, because we never measured FPS.

---

## 1. Picking a Wine build

Homebrew is no longer an easy source for this: its Wine casks were disabled on 2026-09-01 for failing Gatekeeper. We used prebuilt engines from other sources instead, and we installed Rosetta 2 first, because these builds are x86_64:

```bash
softwareupdate --install-rosetta --agree-to-license
```

Every app gets its own Wine prefix, which is an isolated fake Windows folder. A failed experiment can't damage anything else that way.

The same game behaved very differently on each engine:

| Engine | Result | Our reading |
|---|---|---|
| Wine Staging 11.18 (Gcenx build) | Crash or hang when Mono loads | Its 32-bit mode (wow64) has to switch between 32-bit and 64-bit code, which appears to fail under Rosetta |
| Game Porting Toolkit (Wine 7.7) | Crash in CoreAudio with `alloc_pages_vprot` | A non-Wine audio thread calls into Wine's memory allocator. Disabling audio made the game run, but silently |
| **WineCX 24.0.7** (Wine 9.0) | **Runs, with sound** | |

The "our reading" column is our interpretation of the crash logs, not a confirmed root cause. The results themselves are what we observed.

**Lesson:** a newer Wine isn't automatically better for an old game. If a game fails, try another engine before you spend hours on settings.

The WineCX engine wouldn't start on its own at first, because of a missing `libinotify`. It needs the support libraries that ship in the Sikarugir wrapper template, so we copied the template's `Contents/Frameworks` next to the engine and pointed the loader at it:

```bash
export DYLD_FALLBACK_LIBRARY_PATH="$PWD/cx24/Frameworks:/usr/lib"
export WINEPREFIX="$HOME/wine-prefixes/mygame"
cx24/wswine.bundle/bin/wine "/path/to/Game.exe"
```

---

## 2. The "damaged or incomplete" app

We wrapped the launcher in a small `.app` so the game could be opened like any Mac app. macOS refused to open it, showed a prohibited icon, and Spotlight ignored it.

The cause was the game's name, which contains `&`. Our script pasted the name straight into `Info.plist`, and a bare `&` makes that XML invalid. So nothing was wrong with the signature. The app's description simply couldn't be read.

The fix is to let Apple's tool handle the escaping, and to check the result:

```bash
plist="MyGame.app/Contents/Info.plist"
plutil -create xml1 "$plist"
plutil -insert CFBundleName -string "Game&Name" "$plist"
plutil -insert CFBundleExecutable -string launcher "$plist"
plutil -insert CFBundlePackageType -string APPL "$plist"
plutil -insert CFBundleIconFile -string icon "$plist"
plutil -lint "$plist"
codesign -s - --force MyGame.app
```

For the icon, `wrestool` and `icotool` (from `icoutils`) extract it from the `.exe`, and `sips` and `iconutil` turn it into an `.icns`. Installing to `/Applications` and running `lsregister -f` and `mdimport` on the app made it show up in Spotlight.

---

## 3. Lag, part one: DirectX to Metal

The game was fine in quiet moments and collapsed as soon as many things appeared on screen. Unity writes the graphics device it found to `<Game>_Data/output_log.txt`, and that was the first clue:

```text
Version:  Direct3D 11.0 [level 10.1]
Renderer: NVIDIA GeForce 8800 GTX (ID=0x191)
```

There is no NVIDIA card in this MacBook. That name comes from Wine's `wined3d`, which translates Direct3D to OpenGL. OpenGL is a poor path on modern Macs, so we swapped in [DXMT](https://github.com/3Shain/dxmt), which translates Direct3D 10/11 to Metal. The Sikarugir template ships a build under `Contents/SharedSupport/renderer/dxmt`.

Installing it took three attempts:

1. **Copying the DLLs into the prefix and setting registry overrides to `native,builtin`.** Wine ignored this and kept loading its own `d3d11.dll`, even though the override and the files were correct. DXMT's DLLs seem to be marked as Wine builtins, in which case Wine prefers its engine folder.
2. **Copying the DLLs into the engine.** Now `winemetal.dll` wasn't found, DXMT failed to load, and Unity quietly fell back to Direct3D 9.
3. **Also copying `winemetal.dll` into the prefix.** This worked.

To leave the original engine untouched, we made a copy. On APFS, `cp -c` creates a clone, so this takes almost no extra disk space:

```bash
cp -cR cx24 cx24+dxmt
LIB=cx24+dxmt/wswine.bundle/lib/wine
cp dxmt/wine/i386-windows/*   "$LIB/i386-windows/"
cp dxmt/wine/x86_64-windows/* "$LIB/x86_64-windows/"
cp dxmt/wine/x86_64-unix/*    "$LIB/x86_64-unix/"

WIN="$WINEPREFIX/drive_c/windows"
cp dxmt/wine/i386-windows/winemetal.dll   "$WIN/syswow64/"
cp dxmt/wine/x86_64-windows/winemetal.dll "$WIN/system32/"
```

Unity's log confirmed the change:

```text
Version:  Direct3D 11.0 [level 11.0]
Renderer: Apple A18 Pro (ID=0x0)
```

You may also see `Failed to get metal layer via MACDRV_ESCAPE_GET_SURFACE` in the log. The game still drew correctly for us.

We turned on `WINEMSYNC=1` at the same time, which replaces some thread synchronization with a lighter mechanism. Because both changes landed together, we can't say how much each one contributed. General lag improved a lot, but one problem remained.

---

## 4. Lag, part two: the coin

Now the only drop left was very specific: collecting coins.

**First guess: shaders.** A hitch when an effect appears is often shader compilation, so we gave DXMT a permanent cache:

```bash
export DXMT_SHADER_CACHE_PATH="$HOME/wine-cache/mygame"
```

The drop felt smaller, but it still happened on *every* coin. A compile only happens once, so shaders couldn't be the main cause.

**Second guess: the garbage collector.** Unity 4's Mono stops all threads to clean up memory, which we suspected would be slow under Wine. We couldn't get Mono to log its collections, so this stayed unproven, and we moved on to reading the game's code.

**Reading the code.** Unity games ship their C# in `Managed/Assembly-CSharp.dll`, which is easy to inspect. With [dnfile](https://github.com/malwarefrank/dnfile) we listed the calls inside the coin methods. `Sound_Manager.Get_Coin` turned out to create a new sound object with `Object.Instantiate` each time.

Next we read the clips' settings with [UnityPy](https://github.com/K0lb3/UnityPy). Of the game's 125 audio clips, 109 were WAV and only 16 were Ogg Vorbis, and two clips with coin-handling names were among those 16. All of the Ogg clips had `m_Stream = 1`. We matched those clips to the coin pickup by name only, which is another reason we changed all 16 rather than just those two.

That suggested a theory: Ogg clips kept compressed in memory have to be decoded as they play, and here that happens through both Wine and Rosetta on every coin. So we set those 16 clips to `m_Stream = 0`, which we understood to mean "decompress when loading".

We changed only those bytes, so the rest of each file stayed identical. First, back up the assets. Then locate the field:

```python
import struct, UnityPy

env = UnityPy.load(".")  # run inside <Game>_Data
for obj in env.objects:
    if obj.type.name != "AudioClip":
        continue
    t = obj.read_typetree()
    if t["m_Type"] != 14:  # 14 = Ogg Vorbis
        continue
    raw = obj.get_raw_data()
    pattern = struct.pack("<iiBBxxii", t["m_Format"], t["m_Type"],
                          int(t["m_3D"]), int(t["m_UseHardware"]),
                          t["m_Stream"], len(t["m_AudioData"]))
    i = raw.find(pattern)
    print(obj.assets_file.name, obj.byte_start + i + 12, t["m_Name"])
```

Then write a little-endian `0` at each printed offset, after checking that the current value is `1`. This layout is what we found in this Unity 4.6 game, so check yours before writing to it.

The game loaded with no new errors, and the coin drops were gone. It has played smoothly in everything tried since.

> **Honest note:** the coin fix worked, but we didn't prove the mechanism. We didn't profile the game before and after, and the meaning of `m_Stream = 0` comes from our understanding of Unity 4's load types, not from documentation we checked. What we can say is that this change came right before the drops disappeared.

---

## 5. What to try with your own old game

1. Read `<Game>_Data/output_log.txt`. It tells you the Unity version and which graphics device the game thinks it has.
2. If you see a fake GPU name like `GeForce 8800 GTX`, you're on `wined3d`. For a DirectX 10/11 game, try DXMT.
3. If the game crashes, change the Wine engine before changing settings.
4. If the lag is tied to one action, ask whether it happens only the first time or every time. First-time-only points to shaders. Every time points to something the game does on each occurrence, so go and read what that action does.
5. Back up any game file before you patch it.

---

*Written by Panca Walendra Putra and Claude Opus 5.5 (Anthropic), from a real troubleshooting session.*
