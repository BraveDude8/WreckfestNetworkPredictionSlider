# Wreckfest Client Prediction Time slider

Adds the **CLIENT PREDICTION TIME** slider back into **Settings > NETWORK** in the current Steam version of Wreckfest. When it's set, the game uses the slider value to decide how many remote cars it predicts.

The mod is client-side only, so it works on normal (vanilla) servers. The server and the other players see an ordinary client. You don't need a server mod or a modded server, and nobody else has to install anything.

> **Playtest status:** the mod boots, the slider shows, moves and saves, and the prediction code has been verified offline. **It has not been tested online yet.** See [Status](#status).

## Why

Between server updates, Wreckfest predicts where remote cars are. The number of cars it predicts each step is:

```
cars = budget / your ping      (never more than 24)
```

Older builds of the game had a slider for the budget. The current build still saves a value, but it never reads it and always uses **1800 ms**. At high ping that budget covers only a few cars, which is when remote cars warp the most:

| Ping   | Stock (1800) | 3600 | 5000 | 7208 | 9200 (max) |
|--------|-------------:|-----:|-----:|-----:|-----------:|
| 100 ms | 18 | 24 | 24 | 24 | 24 |
| 200 ms |  9 | 18 | 24 | 24 | 24 |
| 250 ms |  7 | 14 | 20 | 24 | 24 |
| 300 ms |  6 | 12 | 16 | 24 | 24 |
| 350 ms |  5 | 10 | 14 | 20 | 24 |

Players on long routes, for example from Australia to European servers at roughly 250 to 350 ms, are the ones it helps most.

## Install

1. Download `WreckfestPredictionSlider-playtest001.zip` from [Releases](../../releases).
2. Copy `XINPUT1_3.dll` into the Wreckfest folder, next to `Wreckfest_x64.exe`. To find the folder: Steam > Wreckfest > Manage > Browse local files.
3. Start the game and go to **Settings > NETWORK > CLIENT PREDICTION TIME**.

The slider runs from 800 ms to 9200 ms. **DEFAULT (1800MS)** sits where 1800 falls on the slider and gives you exactly the same behaviour as stock. Higher values cost more CPU, lower the value if it's causing performance issues.

## Uninstall

Delete `XINPUT1_3.dll`, and the game is stock again. The stock game ignores the saved slider value, so nothing else needs undoing.

To switch the mod off without deleting it, create `wfpred.ini` next to the DLL:

```ini
[wfpred]
enabled=0
```

## Safety and compatibility

- **Exe check.** On start, the DLL checks the game exe byte for byte. If Wreckfest has been updated, or anything doesn't match, it changes nothing and the game runs stock. `wfpred.log` then says `refused`.
- **What it writes.** The only file it writes is `wfpred.log`, next to the DLL. The game saves the slider value in its own settings file, as usual.
- **What it changes.** Only the prediction budget. The 24-car cap, the offline path and everything the server sees are untouched.
- **Other mods.** It doesn't work alongside other mods that also ship an `XINPUT1_3.dll`, such as some controller wrappers; only one of them can be installed. The HUD layout mod uses `VERSION.dll` instead, so both can be installed, but they haven't been tested together yet.
- **Unsigned.** The DLL isn't signed, so Windows or your antivirus may warn about it.

Supported game build:

| File | SHA-256 |
|------|---------|
| `Wreckfest_x64.exe` (Steam, November 2026) | `87b3c5a9253ca67aebf867b475b149e0964bc24aa1c117ff62cd5aec89dcb85b` |
| `XINPUT1_3.dll` (playtest 001) | `05ba9b9085860f0b5af48be5fc6fc351e9640345e794027d5bc4442295d58212` |

## How it works

`XINPUT1_3.dll` is a proxy that the game loads by itself. It forwards every XInput call to the system DLL. When it loads, it verifies the exe and then makes small in-memory patches:

- **Read hook.** It patches the divide that sets the predicted-car count. If the saved value is between 801 and 9200, the count becomes `min(24, value / min(ping, 400))`. Any other value takes the stock `1800 / ping` path.
- **Menu.** It restores the slider row in Settings > NETWORK. The row has 101 stops, from 800 to 9200 in steps of 84. The DEFAULT stop stores 800, a value the read hook treats as "use stock".

Nothing on disk is modified, and the patches go away when the game exits.

## Status

| What | Result |
|------|--------|
| Patch code, run against the real bytes in an emulator: read hook over a value × ping grid, menu round trip over all stops | Pass |
| Headless boot with the real game | Pass: no crash, all patches installed |
| In-game menu check | Pass: the slider shows, moves and saves (3908 ms, 1304 ms, DEFAULT) |
| Online play on a real server | **Not tested yet** |
| Together with the HUD layout mod | Not tested yet |

## Reporting back

Please open an issue with:

- whether the game crashed online, and if so, when;
- your ping to the server and the slider value you used;
- how the warping of other cars compared with DEFAULT: better, the same or worse;
- any change in frame rate;
- your `wfpred.log` from the Wreckfest folder.

## Building from source

You need the Visual Studio Build Tools (x64) and Python 3.

1. `Tools/gen_blob.py` generates the patch blob header (`wfp_blob.h`) for the pinned exe.
2. `build.ps1 -Gen <gen folder> -Out <out folder>` compiles `src/wfpred.c` into `XINPUT1_3.dll` and writes `hashes.json`.
3. `Tools/verify_blob.py` runs the offline checks.

The build is reproducible (`/Brepro`).

## Disclaimer

This is an unofficial fan mod. It isn't affiliated with or endorsed by THQ Nordic or Bugbear Entertainment. Use it at your own risk.
