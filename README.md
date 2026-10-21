# Airlaunch-RO

# ROStandaloneAirlaunch

**Author:** Suyash Jevaria w/ Claude and ChatGPT

A standalone version of RP-1's airlaunch for **KSP 1.12.x** with RSS / Realism Overhaul / AJE / FAR.
Build a plane in the SPH, open the Airlaunch window, set altitude, speed, heading and pitch, and
start the flight already airborne at those conditions. Works in **Sandbox** (and other modes) and needs
neither RP-1 nor KCT. There are no build times, research gates, funds or rollout steps.

Tested on KSP 1.12.5 with RSS, RO, AJE and FAR in Sandbox.

---

## Install

1. Copy `GameData/ROStandaloneAirlaunch/` from this ZIP into your KSP `GameData/` folder,
   giving `GameData/ROStandaloneAirlaunch/Plugins/ROStandaloneAirlaunch.dll`.
2. That's it. It has no dependencies beyond stock KSP 1.12. RSS/RO/AJE/FAR aren't required by the plugin
   itself; it just works with them.

To uninstall, delete `GameData/ROStandaloneAirlaunch/`. The mod writes nothing to your saves and keeps
its last-used values in `GameData/ROStandaloneAirlaunch/PluginData/LastUsed.cfg`.

## Use

1. In the SPH (or VAB), build or load the aircraft. Make sure the launch site selector shows the
   site you want (Runway by default in the SPH).
2. Click the **delta-wing toolbar button** to open the *RO Standalone Airlaunch* window.
3. Choose where to appear:
   - **Offset from launch site**: distance (km) and bearing (°) from where the craft would spawn.
     0 km puts you straight above the runway.
   - **Latitude / Longitude**: an absolute point on the launch body.
4. Set **altitude**, **speed**, **heading** and **pitch** (or click a preset).
5. Options:
   - *Velocity follows pitch*: off by default, which keeps the velocity horizontal so pitch only sets
     attitude. Turn it on for zero angle of attack.
   - *Throttle on release*, *Retract landing gear*, *Activate next stage on release* (stage the
     engines so AJE jets are running).
   - *Settle hold (s)*: how long the craft is held on rails at the start conditions while the scene
     settles. Use **0 for an instant launch** (still held for 5 physics frames); 1 s is the default.
6. Click **Airlaunch!**. KSP's normal launch runs (crew assignment, craft auto-save), and once the
   flight scene loads the craft is moved to the target and released.

After release a message such as `Airlaunch: 12000 m ASL, 350.0 m/s surface, heading 090.0` reports
what the craft is actually doing.

### What the numbers mean

| Setting | Definition |
|---|---|
| Altitude | **Above sea level** (the body's datum radius), not above terrain. Same as RP-1. The launch is refused if the target is less than `terrainClearance` (150 m) above the terrain there. |
| Speed | **Surface-relative** (the navball's *Surface* speed). Earth's rotation is added automatically: 350 m/s east at the Cape is ~758 m/s of orbital speed. |
| Heading | Compass degrees, 0 = north, 90 = east. |
| Pitch | Nose above the horizon in degrees. Roll is always wings-level. |
| Bearing / distance | Great-circle offset from the spawn point. |

### Config file

`GameData/ROStandaloneAirlaunch/PluginData/AirlaunchSettings.cfg` holds `LIMITS` (validation
bounds, no tech gating), `DEFAULTS`, and any number of `PRESET` blocks. It's re-read every time
you enter the editor. The *Acceptance test* preset is 12 000 m / 350 m/s / 090 / 0.

## How it works

1. **Editor:** settings are validated and stored as a one-shot `AirlaunchRequest`, then the stock
   `EditorLogic.launchVessel()` launches the craft normally. The request only arms when KSP actually
   starts loading the flight scene. It's discarded if the launch is cancelled, the ship is edited,
   or it isn't used within 2 minutes.
2. **Flight**, adapted from RP-1 4.7's `AirlaunchUtils.DoAirlaunchRoutine` and ROUtils'
   `HyperEdit_Utilities.DoAirlaunch`:
   - wait until the vessel has finished spawning;
   - resolve the target lat/lon and check terrain clearance (if it fails, the craft stays on the runway);
   - teleport prep from HyperEdit (clear landed state, remove launch clamps, hold unpack, put vessels on rails);
   - **settle hold:** every physics tick the packed vessel is placed on a straight, constant-speed
     surface track. Its orbit is set from the position and `body.getRFrmVel(pos) + surface velocity`,
     and its attitude is re-applied. It can't fall, slow down or clip terrain while scenery loads;
   - **release:** holds are dropped and the track keeps being applied until KSP unpacks the vessel;
   - **verify:** on the first physics frame the surface velocity is compared with the target, and if
     it's more than 2 m/s off a single `ChangeWorldVelocity` correction is applied;
   - gear up, throttle, optional staging, `PRELAUNCH → FLYING` event (so loggers notice).
3. **Attitude:** the requested nose/belly directions are applied to the craft's *control point*
   (`ReferenceTransform`: up = nose, forward = belly, KSP's aircraft convention). A rotated root part
   or a non-default control point therefore still ends up facing the right way.
4. East/North/Up come from the body's own lat/lon → world mapping, which is the navball's definition,
   so headings don't depend on coordinate-system handedness.

### Changes from RP-1

| RP-1 | This mod |
|---|---|
| KCT build list → rollout → launch | stock launch from the editor, instant |
| `AirlaunchTechLevel` limits from research | static `LIMITS` in a config file |
| Position = KSC (via KSCSwitcher `PQSCity`) + distance/azimuth | spawn point + distance/bearing, **or** lat/lon |
| Heading only | heading **and pitch** |
| Fixed 10 s on-rails countdown | configurable 0–30 s hold (5-frame minimum) |
| Hold propagation assumes KSP's inverse-rotation frame | explicit geographic track, valid in either frame |
| — | post-release velocity check / correction, terrain clearance check |
| Publicized `Assembly-CSharp` | stock assembly; the two non-public members (`Vessel.SetPhysicsHoldExpiryOverride`, `LoadingBufferMask`) are reached by guarded reflection and skipped if missing |

## Troubleshooting checklist

If something doesn't behave as expected, these are the things to check (a Sandbox test save works best).

- [ ] KSP.log contains `[ROStandaloneAirlaunch] Running on KSP 1.12.5` and no exceptions mentioning ROStandaloneAirlaunch.
- [ ] The toolbar button appears in the SPH and VAB, and the window opens and closes. Clicking in the window doesn't place or select parts.
- [ ] **Acceptance test:** load an F-22-style fighter with AJE engines and FAR, pick the *Acceptance test* preset, and click **Airlaunch!**
  - [ ] The craft appears in the air with no explosion and no falling through terrain.
  - [ ] Navball (Surface mode): about **350 m/s**, heading about **090**. Altitude about **12 000 m**.
  - [ ] The screen message and KSP.log line `After release: srf ... (target 350.00), error ...` show a small error (a correction line is OK).
  - [ ] Engines ignite (staged) and respond to throttle. Intakes supply air. Control surfaces respond (FAR).
  - [ ] Gear is retracted. The craft flies normally, and level flight is sustainable at 12 km / Mach ~1.2.
- [ ] Settle hold 0 vs 3 s: both release at the right speed. With 3 s a countdown is shown.
- [ ] Lat/Lon mode (e.g. 34.9, -117.9, Edwards area) puts the craft there. A target below terrain + 150 m is refused with a message.
- [ ] Heading 270 / 0 / 180 give the right navball headings. Pitch 10° gives a nose-up attitude.
- [ ] A normal (non-airlaunch) launch from the editor afterwards stays on the runway.
- [ ] Revert to Launch / Revert to Editor behave normally.

If anything fails, KSP.log lines prefixed `[ROStandaloneAirlaunch]` show each step.

## Known limitations

- Principia: RP-1 keeps the vessel in `PRELAUNCH` while packed for Principia's sake, and this mod does
  the same, but Principia hasn't been tested.
- Staging on release fires the *next* stage, whatever it contains. Turn it off if stage 0 holds decouplers.
- If KSP's "launch site not clear" dialog appears and you cancel it, the airlaunch request is discarded
  when you edit the ship, click *Cancel airlaunch*, or after 2 minutes.

## Building

Requirements: .NET SDK 8 (any edition) and KSP 1.12 assemblies, which aren't redistributable and so
aren't included. Copy these files from `<KSP>/KSP_x64_Data/Managed/` into a `References/` folder next to
`Source/` (or pass `-KSPManagedDir`):

```
Assembly-CSharp.dll            UnityEngine.dll                  UnityEngine.InputLegacyModule.dll
Assembly-CSharp-firstpass.dll  UnityEngine.CoreModule.dll       UnityEngine.TextRenderingModule.dll
                               UnityEngine.IMGUIModule.dll      UnityEngine.UI.dll
                               UnityEngine.PhysicsModule.dll    UnityEngine.AnimationModule.dll
```

Then from the folder containing `Source/` and `References/`:

```powershell
powershell -ExecutionPolicy Bypass -File Source\build.ps1
```

This builds `GameData/ROStandaloneAirlaunch/Plugins/ROStandaloneAirlaunch.dll`, runs the unit tests,
checks that the DLL references only mscorlib/System/Assembly-CSharp/UnityEngine (no RP-1, KCT,
ROUtils or Harmony), and writes `dist/ROStandaloneAirlaunch-<version>.zip`.

Source layout:

| Path | Purpose |
|---|---|
| `Source/ROStandaloneAirlaunch/AirlaunchMath.cs` | KSP-free math: local basis, velocities, attitude, great-circle track |
| `Source/ROStandaloneAirlaunch/AirlaunchSettings.cs` | KSP-free settings model, validation, serialization |
| `Source/ROStandaloneAirlaunch/AirlaunchConfig.cs` | cfg loading/saving, editor→flight request, logging |
| `Source/ROStandaloneAirlaunch/EditorAirlaunchUI.cs` | toolbar button, window, stock launch hand-off |
| `Source/ROStandaloneAirlaunch/FlightAirlaunch.cs` | flight-scene teleport, hold, release, verification |
| `Source/ROStandaloneAirlaunch/KspInterop.cs` | type conversions, guarded reflection, version check |
| `Source/ROStandaloneAirlaunch.Tests/` | NUnit tests (mock RSS Earth, both coordinate handednesses) |
| `Source/Tools/ApiInspector/` | dev tool: dumps KSP API signatures / checks DLL references |

## License and credits

Created by **Suyash Jevaria**. CC BY-NC-SA 4.0. See [LICENSE](LICENSE). Adapted from **RP-1** (KSP-RO team) and **ROUtils**, whose
teleport code derives from **HyperEdit** by khyperia and Ezriilc.

