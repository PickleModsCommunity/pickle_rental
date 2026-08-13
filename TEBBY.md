# TEBBY.md - pickle_rental

## Overview

`pickle_rental` is a FiveM resource (Lua, `fx_version 'cerulean'`, GTA V) that lets
players rent vehicles from configurable in-world locations. Each location has an
interact point (NPC ped or marker), a spawn point, and a list of vehicles with
prices. Players pay a deposit, drive the rented vehicle, and return it at the same
spot for a partial refund calculated by time elapsed and body health. Blips are added
to the map for every location. The resource supports ESX, QBCore, and a stub path for
other frameworks.

---

## Code design

### Architecture

The resource is split into three layers that follow a consistent Pickle Mods pattern:

| Layer | Path | Role |
|---|---|---|
| Config | `config.lua` | All user-editable settings (shared script) |
| Core utilities | `core/shared.lua`, `core/client.lua` | Generic helpers available everywhere |
| Module | `modules/rental/` | All rental-specific logic (shared, client, server) |
| Bridge | `bridge/esx/`, `bridge/qb/`, `bridge/standalone/` | Framework-specific adapters |
| Locales | `locales/` | Translation loader + per-language tables |

`fxmanifest.lua` wires everything with glob patterns
(`bridge/**/server.lua`, `modules/**/shared.lua`, etc.) so adding new bridge or
module files is automatic.

### Main flow

1. **Shared setup** – `config.lua` is loaded first, then `core/shared.lua` (utility
   functions), then `locales/`. `modules/rental/shared.lua` copies
   `Config.Locations` into a global `Locations` table.

2. **Bridge selection** – Each bridge file guards itself with
   `GetResourceState('es_extended') ~= 'started'` / `'qb-core'`. Only one bridge
   pair (ESX or QBCore) activates; the standalone bridge activates when neither is
   running and just prints a warning.

3. **Client loop** (`modules/rental/client.lua`) – A single `CreateThread` loop
   checks player distance to every location every frame (tight) or every second
   (idle). When inside `Config.RenderDistance` it renders NPC peds or markers, and
   when inside the interact radius it shows `lib.showTextUI` and listens for `[E]`.

4. **Renting** – `RentalMenu` uses `ox_lib`'s `lib.registerMenu` to list vehicles.
   Selecting one calls the `pickle_rental:rentVehicle` server callback, which
   validates cash and generates a plate. On success the client spawns the vehicle,
   then fires `pickle_rental:registerRental` (net event) so the server records it in
   `RentalVehicles[net_id]`.

5. **Returning** – `ReturnRental` calls the `pickle_rental:returnVehicle` callback.
   The server calculates the deposit refund from time remaining
   (`rental.timeLeft / Config.Rental.time`) and vehicle body health, then deletes
   the entity and gives money back.

6. **Display preview** – While the rental menu is open a local (non-networked)
   display copy of the highlighted vehicle is spawned at the spawn point (frozen,
   no collision). It is deleted before the actual rent call.

### Conventions

- Lua 5.4 (`lua54 'yes'`).
- Global functions are used throughout (no module/return pattern); globals follow
  `PascalCase` for functions and `camelCase` for locals.
- `dprint(...)` gates debug output behind `Config.Debug`.
- `_L(key, ...)` retrieves localised strings; missing keys return
  `"ERR_TRANSLATE_<key>_404"`.
- Bridge functions are named identically across ESX/QBCore adapters:
  `ShowNotification`, `ServerCallback`, `RegisterCallback`, `Search`, `AddItem`,
  `RemoveItem`, `CanAccessGroup`, `SetVehicleProperties`, `GetVehicleProperties`.
- Plate format uses `_` for a random letter and `.` for a random digit (defined in
  `Config.Rental.plateFormat`).

---

## Important files

| Path | Why it matters |
|---|---|
| `fxmanifest.lua` | Resource manifest; defines load order, Lua 5.4, and glob patterns that pull in all bridge/module files automatically. |
| `config.lua` | All user-editable settings: language, debug flag, render distance, rental time/plate format, `GiveKeys` callback, and all `Config.Locations` definitions. The primary file an operator edits. |
| `core/shared.lua` | Shared utility helpers: `v3`, `v2`, `GetRandomInt`, `dprint`. Loaded as a shared script before modules. |
| `core/client.lua` | Client utility helpers: `ModelRequest`, `CreateVeh`, `CreateNPC`, `CreateProp`, `CreateBlip`, `ShowInteractText`. Used by the rental client module. |
| `modules/rental/shared.lua` | Sets the global `Locations = Config.Locations`. Simple but required; without it `Locations` is nil on both client and server. |
| `modules/rental/client.lua` | All client-side rental logic: ped management, marker drawing, proximity loop, rental menu, vehicle spawn/return, display preview. The main client entry point. |
| `modules/rental/server.lua` | All server-side rental logic: plate generation, rental state (`RentalVehicles`), countdown thread, rent/return callbacks, refund calculation. |
| `bridge/esx/client.lua` | ESX client adapter. Guards itself with `GetResourceState`. Provides `ShowNotification`, `ServerCallback`, `CanAccessGroup`, `SetVehicleProperties`, `GetVehicleProperties`. |
| `bridge/esx/server.lua` | ESX server adapter. Provides `RegisterCallback`, `ShowNotification`, `Search`, `AddItem`, `RemoveItem`, `CanAccessGroup`. |
| `bridge/qb/client.lua` | QBCore client adapter — same surface as ESX client bridge. |
| `bridge/qb/server.lua` | QBCore server adapter — same surface as ESX server bridge. Note: `bank` balance deliberately falls back to `cash` (author left TODO comments). |
| `bridge/standalone/client.lua` | Activates only when neither ESX nor QBCore is running; prints a warning and stubs nothing — server will error if bridge functions are called. |
| `bridge/standalone/server.lua` | Same as standalone client — warning only. |
| `locales/locale.lua` | Defines the `_L(name, ...)` helper that formats strings from the active `Language[Config.Language]` table. |
| `locales/translations/en.lua` | English strings (default). Template for adding other languages. |
| `locales/translations/de.lua` | German translation. |
| `locales/translations/fr.lua` | French translation. |

---

## Intended behaviors

- **`GiveKeys` is intentionally empty** (`config.lua`, `Config.GiveKeys`). This is a
  stub that every operator must fill in to wire up their own key/vehicle-lock resource
  (e.g. `qb-vehiclekeys`, `vehicles_keys`). An empty function is not a bug; it means
  the vehicle spawns unlocked until the operator implements it.

- **QBCore bank balance falls back to cash** (`bridge/qb/server.lua`). `Search`,
  `AddItem`, and `RemoveItem` for `"bank"` all redirect to `cash`. The author left
  explicit TODO comments. This is a known limitation, not a regression.

- **Standalone bridge does nothing** (`bridge/standalone/`). When neither ESX nor
  QBCore is present, the bridge files only print a warning. Calls to
  `ShowNotification`, `RegisterCallback`, etc. will error at runtime. This is the
  intended "you must implement it yourself" signal.

- **Display preview vehicle is local only** (`modules/rental/client.lua`,
  `SpawnLocalDisplay`). The preview vehicle is spawned with `networked = false` and
  deleted before the real rent. It is invisible to other players by design.

- **Refund formula** (`modules/rental/server.lua`, `Max` function): deposit is
  reduced by both time used and vehicle damage. The `Max` helper clamps to 0–price
  but its logic (`if number > max then return number` rather than `return max`) means
  values above `max` pass through unclamped — this appears intentional (the function
  is only called with ratios ≤ 1, so the branch is never triggered in normal
  operation, but the logic is unusual; do not "fix" it without understanding the full
  math).

- **`activeMenu` is an implicit global** (`modules/rental/client.lua`). The variable
  is used without a `local` declaration. This is consistent across the file and
  appears intentional for cross-function state sharing; it is not a leaked upvalue
  bug.

---

## Common issues & fixes

| Symptom | Likely cause | Fix |
|---|---|---|
| Resource name mismatch — locales or blips don't appear | Folder is named something other than `pickle_rental` | Rename the resource folder to exactly `pickle_rental` and update `server.cfg`. |
| `ERR_TRANSLATE_<key>_404` text shown to players | `Config.Language` does not match any key in `locales/translations/` | Set `Config.Language` to `"en"`, `"de"`, or `"fr"`, or add a matching translation file. |
| Vehicles spawn but are always unlocked | `Config.GiveKeys` is empty | Implement the `Config.GiveKeys(plate)` function in `config.lua` using your server's key resource export. |
| `attempt to call a nil value` on rent/return | Neither ESX nor QBCore detected; standalone bridge is active | Implement the required bridge functions in the standalone bridge files, or ensure ESX/QBCore starts before `pickle_rental`. |
| Players can rent while already renting | `rentalVehicleQueue` only guards the callback race; if queue is cleared abnormally (resource restart mid-rent), a second rent is possible | Restart the resource cleanly; add guard logic in `registerRental` if needed. |
| Return refund is always $0 | `Config.Rental.time` is set to 0 or time exceeded | Set `Config.Rental.time` to a positive minute value; the deposit is 0 once time runs out by design. |
| Ped disappears immediately after spawning | `Config.RenderDistance` is very small or player moved away | Increase `Config.RenderDistance` in `config.lua` (default `20.0`). |
| Blip shows wrong location | `location.locations.interact.coords` is incorrect in `Config.Locations` | Correct the `coords` vector in `config.lua` for that location entry. |
| `ox_lib` errors on start | `ox_lib` not started or wrong resource name | Ensure `ensure ox_lib` appears **before** `ensure pickle_rental` in `server.cfg`. |

---

## Conventions

- **File naming**: modules live under `modules/<name>/` with exactly `client.lua`,
  `server.lua`, and `shared.lua`. Bridge adapters live under `bridge/<framework>/`
  with `client.lua` and `server.lua`. Adding files following these patterns is picked
  up automatically by `fxmanifest.lua` globs.
- **Global function names**: `PascalCase` (e.g. `RentVehicle`, `CreateVeh`,
  `ShowNotification`). Local variables: `camelCase`.
- **Locale keys**: `snake_case` (e.g. `spawn_not_afford`, `return_success`). Always
  add new keys to all translation files simultaneously.
- **Bridge surface**: any new bridge must expose the same function names as the ESX
  bridge (`ShowNotification`, `RegisterCallback`, `Search`, `AddItem`, `RemoveItem`,
  `CanAccessGroup`, `SetVehicleProperties`, `GetVehicleProperties`). Do not rename
  them.
- **Debug output**: use `dprint(...)` (not `print`) for any debug logging so it
  respects `Config.Debug`.
- **`_L` for all player-facing strings**: never pass hardcoded strings to
  `ShowNotification`; always go through `_L(key, ...)`.
- **No `ox_target`**: this resource uses proximity markers + `lib.showTextUI`
  (`ox_lib`) for interaction; there is no target-system integration. Do not add
  target calls without a corresponding config flag.
- **Version field in `fxmanifest.lua`**: do not touch it; Tebby bumps it automatically
  after each fix.
