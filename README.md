# DJI Mini 3 Pro — gimbal calibration after replacement

Tilted horizon after swapping the gimbal arm and reusing the old camera. Here's the fix.

## Fix
1. Get service firmware `V20.00.0800_wm162` (free, don't pay for it):
   https://github.com/N3JC222/DJI-Mini-3-Pro---Gimbal-Calibration-Software
2. Flash it with **Drone-Hacks V2** — free, no licence needed (*Flash From File*).
3. Recalibrate the gimbal on the service firmware.
4. Flash back to stock `01.00.0900`.

Stock and service firmware are **both Anti-Rollback 7**, so flashing back and forth is safe.
Never install newer stock firmware (ARB ≥ 8).

## Don't waste time / money on
- The **NPI VCOM station** ("`WM163 ... gimbal IMU data import`") — can't reach a Mini 3 Pro
  (`PID_0020` has no serial port, only RNDIS). Fails at "VCOM not connected".
- **dji-firmware-tools / DUMLdore** — no Mini 3 Pro support.
- **Forcing a `usbser`/VCOM driver** onto the drone — crashes the USB controller.
- **`rpmb_clear.bat`** from the calibration package — bricks the drone.
