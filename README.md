# DJI Mini 3 Pro — Gimbal calibration after gimbal/camera replacement

Field notes from fixing a **tilted horizon after replacing the gimbal arm assembly** on a
DJI Mini 3 Pro (WM162, USB `VID_2CA3&PID_0020`, aircraft firmware `01.00.0900`).

The goal of this repo is to save you the days I lost. **Read the "What actually matters"
section first** — most of the paths people point you to are dead ends for the Mini 3 Pro.

> ⚠️ Everything here writes to your drone or its firmware. You do it at your own risk.
> Nothing below requires paying random Telegram "calibration" sellers.

---

## The symptom

- Gimbal was replaced (new arm/motor assembly, **original camera module reused**).
- Mechanically perfect: all three axes drive, hold torque, stabilise.
- But the horizon is tilted by a **constant angle** and the camera doesn't sit centred.
- DJI Fly's gimbal auto-calibration reports **success but changes nothing**.
- Often **no error code at all** (no 40011 / 40021) — the calibration "passes" and does nothing.

## What is actually wrong

The gimbal holds a **rotated zero**. It regulates perfectly against its IMU — but the stored
angle between the gimbal IMU and the image sensor is wrong.

Measured on the actual aircraft:

| Photo | Gimbal reports (EXIF) | Real tilt in image |
|---|---|---|
| Before the crash | `GimbalRollDegree +0.00` | +0.19° (i.e. correct) |
| After replacement | `GimbalRollDegree +0.00` | **+10.2°** (measured across 2 different flight attitudes) |

So the tilt is a **fixed reference offset**, independent of flight attitude. It is not a
mechanical mounting error (that would cancel out, because the IMU moves with the cage). It is
the **factory calibration matrix** (IMU→sensor rotation) stored in the gimbal's non-volatile
memory. Your replacement arm set carries **another camera's calibration**.

**Consequences:**
- A firmware refresh does **not** fix it — the matrix lives in NVRAM, not in firmware.
- DJI Fly's calibration on stock firmware only does a fine trim on top of a wrong zero.
- You need to **rewrite the calibration data**, bound to your camera's serial number.

Tip: you can read the gimbal angle straight out of any DJI photo's XMP
(`drone-dji:GimbalRollDegree`, `GimbalPitchDegree`, `GimbalYawDegree`) — no tools needed.
That's how you measure the offset and later verify the fix.

---

## Anti-Rollback — the one thing that makes this safe

Firmware won't roll back below the highest Anti-Rollback (ARB) index the drone has ever seen.

- Stock `01.00.0900` → **ARB 7**
- Service/calibration firmware `V20.00.0800_wm162` → **ARB 7** (read from its `IM*H` header)

**Same index in both directions ⇒ flashing to the service firmware and back to stock is safe.**
Just **never install a newer stock firmware with ARB ≥ 8**, or the service firmware locks out
forever.

---

## What DOES NOT work (so you don't waste time)

- **o-gs/dji-firmware-tools** (`comm_og_service_tool.py GimbalCalib`): serial-only, and its
  model list stops at WM260 (Mavic 3). **No WM162/WM163.** The Mini 3 Pro exposes no serial
  port at all. Firmware decryption for the Mini-3 family isn't public either. Issues #286/#303/
  #316/#343 are open since 2022 for a reason.
- **DUMLdore**: Mavic/Phantom/Spark only, archived Aug 2024. Needs DUML over serial — the drone
  doesn't offer it.
- **SD-card offline flash**: Phantom 3 / Inspire 1 era only. Not this format.
- **Forcing a VCOM driver onto the drone (`usbser` on the bulk interfaces)**: **DON'T.** It
  creates COM ports but they don't work, **and it crashes the USB host controller**
  (`usbser` sends CDC control requests to a vendor/bulk interface → `error -71` →
  `xHCI host controller ... assume dead`). It took down the dock, audio and LAN every ~60 s.

### The NPI factory station is NOT for the Mini 3 Pro

The "`ET7LY32 WM163 Aftersale gimbal IMU data import`" station (DJI Test `V3.0.0.141`,
`mini3_calibration` package) that sellers hand out **cannot talk to a Mini 3 Pro**:

- It fails at `Get DUT Stable` after 15 s with `VCOM未连接上` (VCOM not connected).
- The Mini 3 Pro (`PID_0020`) has **no CDC/serial interface** — only RNDIS (`MI_00`),
  CDC-data, mass storage, and 5 vendor/bulk interfaces (class `ff`/sub `43`). It communicates
  over **RNDIS / TCP port 10000**, not over a serial VCOM.
- The station's `[HardwareIDs]` only list `PID_001D&MI_00` and `PID_001E&MI_02` — no `PID_0020`.
  The config files are encrypted (`*.encr`) and `Setting` / `Extract Config` are locked.
- Its bundled helpers are for other products: `CheckCali.exe` is built for **WM240** (Mavic 2),
  `commmgr.exe` is a **battery test-adapter** manager, `GetBoundSN.exe` phones a DJI server.

If a seller offers you this station "for the Mini 3 Pro" — its SN-prefix may match your drone,
but the software still physically cannot reach `PID_0020`. Don't pay for it.

**Do NOT run** `rpmb_clear.bat` from that package — it flashes a boot image and wipes the RPMB
area (guaranteed brick) and contains a factory employee's hardcoded server credentials.

---

## What actually matters (the accessible path)

You do **not** need the PC station. The workable route:

1. **Get the free service firmware.** `V20.00.0800_wm162_dji_system.bin` is mirrored publicly at
   **[N3JC222/DJI-Mini-3-Pro---Gimbal-Calibration-Software](https://github.com/N3JC222/DJI-Mini-3-Pro---Gimbal-Calibration-Software)**.
   Sellers charge ~€10 for this exact file — you don't have to pay. (This repo hosts no
   firmware itself, only these notes.)
2. **Flash it with Drone-Hacks V2.** Firmware flashing **within the same ARB index is free** —
   no licence needed, just a free account and *Flash From File*. (The €40 licence only unlocks
   NFZ/altitude/FCC, which you don't need here.)
3. **Calibrate the gimbal** on the service firmware, then
4. **Flash back to stock `01.00.0900`** (same ARB 7, so allowed).

> Note: step 3 is the part sellers gate behind remote sessions. On the service firmware the
> gimbal recalibration is what rewrites the IMU→sensor matrix bound to your camera SN. Verify
> success afterwards by shooting a photo of a doorframe and checking the tilt / EXIF
> `GimbalRollDegree` — it should track the real horizon again.

---

## Useful hardware facts

- The drone uses **one USB-C port for power and data**. It must be **powered on** to enumerate
  as a data device; plugged in while off, it only charges and is invisible.
- It **idle-disconnects after ~60 s** if nothing actively communicates with it — flash/connect
  promptly, don't let it sit.
- It gets **hot on a bench** (no prop airflow). Point a fan at it for anything longer than a
  minute.
- On a flaky USB controller the `-71` errors above can kill the whole controller. Use a
  motherboard USB-C port, avoid hubs, and keep it off shared controllers if you can.

---

## TL;DR

Replaced gimbal + old camera on a Mini 3 Pro → ~10° fixed roll offset because the arm set
carries the wrong camera's factory calibration. The IMU→sensor matrix must be rewritten.
The community serial tools and the leaked NPI VCOM station **cannot reach a Mini 3 Pro**
(`PID_0020`, RNDIS-only). The realistic free path is: flash `V20.00.0800` service firmware with
Drone-Hacks (free, same ARB 7), recalibrate, flash back to `01.00.0900`. Don't pay for the
`.bin`, don't run `rpmb_clear.bat`, and never force `usbser` onto the bulk interfaces.

---

## Links & sources

- **Service firmware `V20.00.0800_wm162`:** https://github.com/N3JC222/DJI-Mini-3-Pro---Gimbal-Calibration-Software
- **Drone-Hacks (flasher):** https://drone-hacks.com — free firmware flashing within the same ARB index
- **dji-firmware-tools (does NOT cover Mini 3 Pro):** https://github.com/o-gs/dji-firmware-tools
- Background threads: mavicpilots.com "Mini 3 Pro gimbal calibration after new gimbal replacement"

*This repository contains documentation only — no firmware or tools are hosted here.*
