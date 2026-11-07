# Wreckfest Client Prediction Time slider

Adds the **CLIENT PREDICTION TIME** slider back into **Settings > NETWORK** in the current Steam version of Wreckfest. When it's set, the game uses the slider value to decide how many remote cars it predicts.

The mod is client-side only, so it works on normal (vanilla) servers. The server and the other players see an ordinary client. You don't need a server mod or a modded server, and nobody else has to install anything.

> **Playtest status:** the mod boots, the slider shows, moves and saves, and the prediction code has been verified offline. **It has not been tested online yet.** See [Status](#status).

## Why

_Everything below comes from reading the game's code through Ghidra/decomp so it may not be 100% correct. _

### The short version

The game has a hard limit on how many other cars it properly predicts at once, and that limit gets smaller the higher your ping is. On 300 ms you only get about 6. Everyone else is guessed at more roughly, and when the guess is wrong they jump. That jump is the warping you see on high-ping servers. The slider lets you raise the limit.

### How prediction works

The server is in charge of where every car is. It sends your game snapshots: car 7 was here, going this fast, pointing this way. By the time a snapshot reaches you it's already out of date, higher ping means that snapshot is more out of date.
Your game takes the snapshot and replays the physics for that car from then until now, to work out where it probably is at this moment. That replay is the "prediction". The higher your ping, the more time it has to replay, and the more work each car costs.

Because of that cost, the game doesn't replay every car. Every time a snapshot comes in it picks a few:

1. Your own car always gets replayed, and it doesn't count towards the limit.
2. Cars that are still partway through a replay from last time keep their place.
3. The rest of the places go to the cars that matter most: the nearest ones to the camera first. A car that's off screen counts as ten times further away than it really is, so cars you can see get priority.

Any cars on the server that don't fit into the prioritised list just pushes them forward along their current path, at most 480 ms. If the driver actually did anything in that 480ms it isn't reflected until the next update. When the next real snapshot arrives, the game compares it with where it had the car. If the gap is small, it eases the car across over a few frames. If the car is more than about 30 cm away from where it should be, it teleports the car to the right spot in one frame. 

### Where the limit comes from

The number of cars that get a proper replay is:

```
cars = 1800 / your ping      (never more than 24)
```

You can think of 1800 as a time budget in milliseconds. Each car costs about one ping's worth of replay, so the budget divided by your ping is roughly how many cars fit.

On a nearby server that's not a problem. At 75 ms you get all 24. But the budget is fixed, so it shrinks fast as ping goes up:

| Ping   | Stock (1800) | 3600 | 5000 | 7208 | 9200 (max) |
|--------|-------------:|-----:|-----:|-----:|-----------:|
| 100 ms | 18 | 24 | 24 | 24 | 24 |
| 200 ms |  9 | 18 | 24 | 24 | 24 |
| 250 ms |  7 | 14 | 20 | 24 | 24 |
| 300 ms |  6 | 12 | 16 | 24 | 24 |
| 350 ms |  5 | 10 | 14 | 20 | 24 |

If you're playing from Australia on a European server at around 300 ms, that's 6 cars out of a full lobby. The rest are running on the cheap guess, and on a track full of people wrecking each other, the cheap guess is wrong a lot.

### What happened to the slider

The 2018 version of the game had a CLIENT PREDICTION TIME slider in the network settings, and it worked a bit differently. The slider value was a time budget, and every car's replay was charged against it until it ran out. The cars that missed out one round went first the next, so it rotated. That version also had no hard snap; a car that was out got eased back in.

At some point the prediction code was rewritten. The slider vanished from the menu and the budget became the fixed 1800. The game still loads the old setting and stores it in memory, but never looks at it again.

### What this mod changes

It puts the slider back in the menu and makes the game use it in place of the 1800. Nothing else about prediction changes. The ranking, the cheap guess, the snapping and the 24-car cap all work exactly as before. You just get more cars on the proper replay.

Two small things to know:

- On the slider values, your ping is counted as 400 ms at most, so on a really bad connection you still get a sensible number of cars. 400 ms is the same replay cap the 2018 version used.
- DEFAULT (1800MS) is the game's own code path, untouched, so picking it is the same as not having the mod.

Every extra car you predict is more physics for your PC to run each time a snapshot lands. On a decent CPU you shouldn't notice it, but if your frame rate drops at the top of the slider, bring it down a few steps.

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
