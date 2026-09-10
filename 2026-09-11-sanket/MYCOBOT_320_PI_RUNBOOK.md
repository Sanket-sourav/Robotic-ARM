# MyCobot 320 Pi Operations, Startup, Test, Calibration, and Recovery Runbook

**System covered:** Elephant Robotics MyCobot 320 Pi at `er@172.20.10.2`  
**Serial interface:** `/dev/ttyAMA0` at `115200` baud  
**Purpose:** Provide a safe, repeatable procedure for starting, validating, operating, testing, calibrating, recovering, and shutting down this robot.

> [!CAUTION]
> A robot can move unexpectedly and can pinch, strike, trap, or damage people and equipment. Secure the base, keep the workspace clear, keep the emergency-stop switch reachable, and never energize or move the arm when its physical state cannot be observed. Do not configure unattended automatic motor power-on at Linux boot.

## 1. Executive summary

The robot and Raspberry Pi are now operational. The original problem was a combination of robot power state and physical posture, not a failed UART or a broken `pymycobot` installation:

1. The Raspberry Pi UART was present and correctly configured.
2. The robot controller responded to a power-status request and reported `0`, meaning robot power was off.
3. While power was off, calls such as `get_angles()`, `get_coords()`, and `set_servo_calibration()` returned `-1` because valid telemetry or command acknowledgement was unavailable.
4. Calling `power_on()` returned `1` and restored controller, servo, angle, and coordinate responses.
5. The robot then reported error `17`, which the MyCobot API classifies within the `16`–`19` collision-protection range.
6. J3 was physically reported at `-147.12°`, beyond this controller's configured J3 minimum of `-145°`. Clear/resume and a slow command toward `-140°` were accepted at the protocol level but motion protection correctly prevented the joint from moving.
7. The operator used the emergency switch and manually placed the arm in a clear upright pose, then released the switch.
8. After `power_on()`, all controller, servo, voltage, temperature, error, angle, and coordinate checks passed.
9. Every joint completed a slow relative `+20°` movement and returned to its starting position with zero reported errors.

The dependable operating rule is therefore:

> **Safe physical pose first, emergency stop released, explicit API power-on second, complete health gate third, motion only after every gate passes.**

## 2. Known-good system baseline

These values were observed on the working system during the 2026-09-10/11 troubleshooting session.

| Item | Observed value | Meaning |
|---|---:|---|
| Robot model | MyCobot 320 Pi | Six-axis collaborative robot |
| Raspberry Pi OS | Ubuntu 20.04.4 LTS | Factory-era Pi image |
| Kernel | `5.4.0-1073-raspi` | AArch64 Raspberry Pi kernel |
| Python API | `pymycobot 4.0.7` | Installed Python library |
| System/Pico firmware | `1.5` | Returned by `get_system_version()` |
| Atom firmware | `5.2` | Returned by `get_atom_version()` |
| Serial device | `/dev/ttyAMA0` | Official 320 Pi serial port |
| Stable alias | `/dev/serial0 -> ttyAMA0` | Pi serial alias |
| Baud rate | `115200` | Official 320 Pi baud rate |
| Device owner/mode | `root:dialout`, `0660` | Serial access is group-controlled |
| Operator account | `er` | Member of `dialout` and `tty` |
| Serial login service | Disabled and inactive | Prevents a console from consuming the robot UART |
| Competing UART process | None observed | `fuser /dev/ttyAMA0` returned no owner |
| Motor bus voltage | `23.5`–`23.8 V` | Consistent with the 24 V system during the passing test |
| Servo temperatures | `30`–`40 °C` | Observed passing baseline, not a universal alarm threshold |
| Robot startup service | None | Linux does not automatically power on the robot |
| Other boot action | Cooling fan only | `/etc/rc.local` starts `/home/er/start_fan/fan.py` |

The boot configuration contains the following relevant settings in `/boot/firmware/config.txt`:

```ini
enable_uart=0
# ...later in the same file...
enable_uart=1
dtoverlay=miniuart-bt
```

The later `enable_uart=1` takes effect, and the working system exposes `/dev/ttyAMA0`. Because the current configuration is proven to work, do not edit it merely to remove the earlier duplicate line. If the image is rebuilt, use one unambiguous `enable_uart=1` setting, preserve the appropriate Bluetooth/UART overlay, reboot, and revalidate the device mapping.

`/boot/firmware/cmdline.txt` contains no `console=serial0` or `console=ttyAMA0` argument. Keep it that way: a serial console would compete with robot communications.

## 3. Officially documented requirements

Elephant Robotics documents the following requirements for the 320 Pi:

- The power adapter and emergency-stop switch must be connected, and the emergency-stop switch must be released for normal operation.
- The arm should not start curled up or with joints touching/interfering.
- The 320 Pi API connection is `/dev/ttyAMA0` at `115200` baud.
- `power_on()` is the API operation used to enable robot communication/power, and a successful result is `1`.
- `is_power_on()` returns `1` for on, `0` for off, and `-1` for invalid/error data.
- `get_error_information()` returns `0` for no error; `1`–`6` for the corresponding joint limit; `16`–`19` for collision protection; `32` for no inverse-kinematics solution; and `33`–`34` for linear-motion solution problems.
- The manufacturer's first-use test moves joints one at a time around a safe zero pose.
- A released joint must be physically supported because it may move under gravity.

See [Official references](#16-official-references) for the source links.

## 4. What failed and why

### 4.1 Initial symptom

The initial read-only test returned:

```text
power_on: 0
joint_rotations_deg: -1
cartesian_coordinates: -1
```

An initial J1 calibration attempt also returned:

```text
j1_calibration_command_result: -1
```

`-1` did **not** mean the Raspberry Pi device file was necessarily missing. It meant the requested API call did not obtain valid robot data or acknowledgement.

### 4.2 Evidence that UART communication was healthy

The serial device existed, the user had permission, no process owned it, and the serial console was disabled. More importantly, debug mode showed the controller answering the power query:

```text
TX: fe fe 02 12 fa
RX: fe fe 03 12 00 fa
power: 0
```

Other commands timed out while the controller reported power off. This separated a power-state problem from a UART problem.

### 4.3 Power recovery

The following operation succeeded:

```python
mc.power_on()  # returned 1
```

After a three-second stabilization delay:

```text
power: 1
controller_connected: 1
all_servos_enabled: 1
```

Live angles and coordinates then became available.

### 4.4 Collision/limit protection

After power recovery, the controller reported:

```text
error: 17
angles: [0.0, 117.42, -147.12, -33.75, 74.61, 60.82]
```

This controller reported the configured J3 range as `-145°` to `+145°`; therefore J3 at `-147.12°` was outside the configured range. Attempts to clear the error, call `resume()`, and command J3 slowly to `-140°` all received protocol acknowledgements but did not produce physical motion. The safety system continued reporting error `17`.

This behavior was correct: an accepted command is not proof of completed motion. Always verify the measured angle and error state.

### 4.5 Physical recovery

The operator used the emergency switch, manually moved the arm to a clear upright pose, and released the switch. The next complete check reported:

```text
power: 1
controller_connected: 1
all_servos_enabled: 1
error: 0
robot_status: [0, 0, 0, 0, 0, 0]
next_error: [0, 0, 0, 0, 0, 0, 0]
servo_status: [0, 0, 0, 0, 0, 0]
angles: [2.81, 8.87, 2.63, -17.84, 87.09, 53.34]
coords: [35.1, -90.4, 512.0, -100.43, 52.56, -98.42]
```

This confirmed that the remaining failure had been the protected physical posture, not a servo electronics fault.

## 5. Correct procedure for every startup

Do these phases in order. Do not skip a gate because the previous run succeeded.

### Phase A — Physical preflight before energizing motion

1. Verify the robot base is securely bolted or clamped and cannot tip.
2. Confirm the correct 24 V power adapter and emergency-stop switch are connected.
3. Inspect cables for cuts, pinching, loose plugs, or routing through the arm's swept volume.
4. Remove tools, packaging, people, and other objects from the arm's workspace.
5. Confirm no joint is visibly against a hard stop, folded into another link, or trapped by an object.
6. If the arm must be manually repositioned, support its weight before disabling power or releasing any servo. Never let a released link fall.
7. Put the arm in a known clear pose. An upright pose worked for this robot.
8. Make sure every person nearby knows a test is about to begin.
9. Keep the emergency-stop switch in immediate reach.
10. Release the emergency stop only after the above checks are complete.

**Pass condition:** The base is secure, the arm is clear and supported, wiring is safe, and the emergency stop is released and reachable.

### Phase B — Start the Pi and connect

1. Apply robot power and allow Ubuntu to boot fully.
2. Connect locally or over SSH:

   ```powershell
   ssh er@172.20.10.2
   ```

3. Authenticate interactively. Do not put the password in scripts, shell history, this runbook, or source control.
4. Confirm the expected host and serial device:

   ```bash
   hostname
   ls -l /dev/ttyAMA0 /dev/serial0
   groups
   systemctl is-active serial-getty@ttyAMA0.service
   fuser -v /dev/ttyAMA0
   ```

Expected essentials:

```text
/dev/serial0 -> ttyAMA0
/dev/ttyAMA0 owned by root:dialout
user is in dialout
serial-getty is inactive
fuser shows no competing owner before the program opens the port
```

**Pass condition:** `/dev/ttyAMA0` exists, the user has access, and no unrelated process owns the port.

### Phase C — Explicit robot power-on

Linux boot does not currently power on the controller through `pymycobot`. Power-on must be explicit after the physical preflight.

```bash
python3 - <<'PY'
from pymycobot.mycobot320 import MyCobot320
import time

mc = MyCobot320('/dev/ttyAMA0', 115200, timeout=1)
try:
    state = mc.is_power_on()
    print('power_before:', state)
    if state == 0:
        print('power_on_result:', mc.power_on())
        time.sleep(3)
    print('power_after:', mc.is_power_on())
finally:
    mc.close()
PY
```

Interpretation:

- `power_after: 1` — continue.
- `power_after: 0` — power did not enable; inspect emergency stop and power hardware.
- `power_after: -1` — no valid response; diagnose serial ownership, wiring, power, and firmware.
- `power_on_result: 1` — command completed successfully.

**Pass condition:** `is_power_on()` returns `1` after the stabilization delay.

### Phase D — Mandatory health gate

Run the health checker in [Section 7](#7-reusable-startup-health-checker). Motion is allowed only when all of the following are true:

| Check | Required result |
|---|---:|
| `is_power_on()` | `1` |
| `is_controller_connected()` | `1` |
| `is_all_servo_enable()` | `1` |
| `get_error_information()` | `0` |
| `get_robot_status()` | all zero |
| `read_next_error()` | all zero |
| `get_servo_status()` | all zero |
| `get_angles()` | six numeric values, none `-1` |
| `get_coords()` | six numeric values, none `-1` |
| Each current joint angle | inside its live configured min/max |

Also record voltages, temperatures, firmware, angles, and coordinates. Compare voltage and temperature values to the known-good baseline, but use controller/servo fault status and the manufacturer's specifications as the authoritative decision points.

**Pass condition:** Every required result passes. If one fails, do not move the robot.

### Phase E — Start the intended application

1. Ensure only one process will open `/dev/ttyAMA0`.
2. Use the correct class and connection:

   ```python
   from pymycobot.mycobot320 import MyCobot320
   mc = MyCobot320('/dev/ttyAMA0', 115200)
   ```

3. Recheck `is_power_on()`, `is_controller_connected()`, and `get_error_information()` inside the application before its first motion.
4. Use conservative initial speed.
5. Add adequate delays or feedback-based waits between commands; a successful send result is not proof the physical target was reached.
6. After every motion, verify position and error state.
7. Close the serial port in a `finally` block.

## 6. Quick startup checklist

Use this only after becoming familiar with the detailed procedure.

- [ ] Base secure; cables intact and clear.
- [ ] No person or object in the workspace.
- [ ] Arm in a clear, non-interfering pose.
- [ ] Emergency stop connected, released, and reachable.
- [ ] Ubuntu booted and correct host reached.
- [ ] `/dev/ttyAMA0` exists; operator is in `dialout`.
- [ ] No serial getty or competing process owns the port.
- [ ] `power_on()` completed if power was off.
- [ ] Power, controller, and all servo checks return `1`.
- [ ] Robot, queued-error, and servo-status arrays are all zero.
- [ ] Angles and coordinates contain six valid values.
- [ ] Every angle is inside its live configured limit.
- [ ] Application starts at low speed and verifies feedback.

## 7. Reusable startup health checker

Save the following as `/home/er/mycobot_startup_check.py`. It does not move the arm. It powers on only when explicitly invoked with `--power-on`.

```python
#!/usr/bin/env python3
"""Read-only MyCobot 320 Pi health gate, with optional explicit power-on."""

import argparse
import json
import sys
import time
from pymycobot.mycobot320 import MyCobot320

PORT = "/dev/ttyAMA0"
BAUD = 115200


def six_numbers(value):
    return (
        isinstance(value, list)
        and len(value) == 6
        and all(isinstance(item, (int, float)) for item in value)
    )


def all_zero(value):
    return isinstance(value, list) and value and all(item == 0 for item in value)


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument(
        "--power-on",
        action="store_true",
        help="Power on the controller after the operator completes physical preflight",
    )
    args = parser.parse_args()

    mc = MyCobot320(PORT, BAUD, timeout=1)
    try:
        initial_power = mc.is_power_on()
        if initial_power == 0 and args.power_on:
            result = mc.power_on()
            print(f"power_on_result={result}", flush=True)
            time.sleep(3)

        limits = [
            [mc.get_joint_min_angle(joint), mc.get_joint_max_angle(joint)]
            for joint in range(1, 7)
        ]

        state = {
            "power": mc.is_power_on(),
            "controller_connected": mc.is_controller_connected(),
            "all_servos_enabled": mc.is_all_servo_enable(),
            "system_version": mc.get_system_version(),
            "atom_version": mc.get_atom_version(),
            "error": mc.get_error_information(),
            "robot_status": mc.get_robot_status(),
            "next_error": mc.read_next_error(),
            "servo_status": mc.get_servo_status(),
            "voltages": mc.get_servo_voltages(),
            "temperatures": mc.get_servo_temps(),
            "currents": mc.get_servo_currents(),
            "angles": mc.get_angles(),
            "coords": mc.get_coords(),
            "limits": limits,
        }

        angles_in_limits = six_numbers(state["angles"]) and all(
            isinstance(low, (int, float))
            and isinstance(high, (int, float))
            and low <= angle <= high
            for angle, (low, high) in zip(state["angles"], limits)
        )

        passed = all(
            [
                state["power"] == 1,
                state["controller_connected"] == 1,
                state["all_servos_enabled"] == 1,
                state["error"] == 0,
                all_zero(state["robot_status"]),
                all_zero(state["next_error"]),
                all_zero(state["servo_status"]),
                six_numbers(state["angles"]),
                six_numbers(state["coords"]),
                angles_in_limits,
            ]
        )

        state["angles_in_limits"] = angles_in_limits
        state["health_gate_passed"] = passed
        print(json.dumps(state, indent=2), flush=True)
        return 0 if passed else 2
    finally:
        mc.close()


if __name__ == "__main__":
    sys.exit(main())
```

Run it after physical preflight:

```bash
python3 /home/er/mycobot_startup_check.py --power-on
```

The command must exit with status `0` and print `"health_gate_passed": true` before motion begins:

```bash
python3 /home/er/mycobot_startup_check.py --power-on
echo $?
```

## 8. Repeatable six-joint ±20° functional test

This is a maintenance/self-check, not something that should run automatically on every Linux boot. Run it only with an operator watching the robot and after the health gate passes.

The script below:

1. Reads the live start angles and configured limits.
2. Selects `+20°`, or `-20°` when `+20°` would be too close to the configured maximum.
3. Moves only one joint at a time at speed `10`.
4. Verifies the measured angle rather than trusting command acknowledgement.
5. Returns that joint to its measured start angle.
6. Stops immediately on an error, timeout, or unexpected result.
7. Requires the operator to type `MOVE` before motion.

Save it as `/home/er/mycobot_joint_test_20deg.py`:

```python
#!/usr/bin/env python3
"""Observed, low-speed, one-joint-at-a-time MyCobot 320 Pi motion test."""

import json
import sys
import time
from pymycobot.mycobot320 import MyCobot320

PORT = "/dev/ttyAMA0"
BAUD = 115200
SPEED = 10
DELTA = 20.0
LIMIT_MARGIN = 3.0
POSITION_TOLERANCE = 1.5
MOVE_TIMEOUT = 10.0


def valid_six(value):
    return (
        isinstance(value, list)
        and len(value) == 6
        and all(isinstance(item, (int, float)) for item in value)
    )


def wait_for_joint(mc, joint, target):
    deadline = time.time() + MOVE_TIMEOUT
    last = None
    while time.time() < deadline:
        angles = mc.get_angles()
        if valid_six(angles):
            last = angles[joint - 1]
            if abs(last - target) <= POSITION_TOLERANCE:
                return True, last

        error = mc.get_error_information()
        if error not in (0, -1):
            return False, last
        time.sleep(0.35)
    return False, last


def fail(mc, message, code):
    try:
        mc.stop()
    finally:
        print(f"FAIL: {message}", file=sys.stderr, flush=True)
    raise SystemExit(code)


def main():
    print("Physical preflight and health gate must already be complete.")
    confirmation = input("Keep clear of the arm. Type MOVE to begin: ").strip()
    if confirmation != "MOVE":
        print("Cancelled.")
        return 1

    mc = MyCobot320(PORT, BAUD, timeout=0.7)
    try:
        power = mc.is_power_on()
        connected = mc.is_controller_connected()
        enabled = mc.is_all_servo_enable()
        error = mc.get_error_information()
        robot_status = mc.get_robot_status()
        servo_status = mc.get_servo_status()
        starts = mc.get_angles()

        if power != 1 or connected != 1 or enabled != 1:
            fail(mc, "power/controller/servo health gate failed", 2)
        if error != 0:
            fail(mc, f"controller error before motion: {error}", 3)
        if not all(item == 0 for item in robot_status):
            fail(mc, f"robot status before motion: {robot_status}", 4)
        if not all(item == 0 for item in servo_status):
            fail(mc, f"servo status before motion: {servo_status}", 5)
        if not valid_six(starts):
            fail(mc, f"invalid start angles: {starts}", 6)

        limits = [
            [mc.get_joint_min_angle(joint), mc.get_joint_max_angle(joint)]
            for joint in range(1, 7)
        ]

        results = []
        for joint in range(1, 7):
            start = starts[joint - 1]
            low, high = limits[joint - 1]
            if start + DELTA <= high - LIMIT_MARGIN:
                target = start + DELTA
            elif start - DELTA >= low + LIMIT_MARGIN:
                target = start - DELTA
            else:
                fail(mc, f"J{joint} has no safe 20-degree target", 10 + joint)

            target = round(target, 2)
            print(f"J{joint}: {start:.2f} -> {target:.2f}", flush=True)
            sent = mc.send_angle(joint, target, SPEED)
            reached, outward = wait_for_joint(mc, joint, target)
            error = mc.get_error_information()
            if sent != 1 or not reached or error != 0:
                fail(
                    mc,
                    f"J{joint} outward failed; sent={sent}, actual={outward}, error={error}",
                    20 + joint,
                )

            sent_back = mc.send_angle(joint, start, SPEED)
            returned, back = wait_for_joint(mc, joint, start)
            error = mc.get_error_information()
            if sent_back != 1 or not returned or error != 0:
                fail(
                    mc,
                    f"J{joint} return failed; sent={sent_back}, actual={back}, error={error}",
                    30 + joint,
                )

            results.append(
                {
                    "joint": joint,
                    "start": start,
                    "target": target,
                    "outward_actual": outward,
                    "return_actual": back,
                    "passed": True,
                }
            )
            print(f"J{joint}: PASS", flush=True)

        final = {
            "results": results,
            "final_angles": mc.get_angles(),
            "final_coords": mc.get_coords(),
            "final_error": mc.get_error_information(),
            "final_robot_status": mc.get_robot_status(),
            "final_servo_status": mc.get_servo_status(),
        }
        print(json.dumps(final, indent=2), flush=True)
        return 0
    finally:
        mc.close()


if __name__ == "__main__":
    sys.exit(main())
```

Run it only while watching the robot:

```bash
python3 /home/er/mycobot_joint_test_20deg.py
```

## 9. Passing test record from this incident

Each joint moved approximately `+20°` and returned within the test's `1.5°` feedback tolerance.

| Joint | Start | Commanded test target | Measured outward | Measured return | Result |
|---:|---:|---:|---:|---:|---|
| J1 | `2.37°` | `22.37°` | `21.26°` | `3.77°` | Pass |
| J2 | `9.84°` | `29.84°` | `29.61°` | `10.37°` | Pass |
| J3 | `2.81°` | `22.81°` | `22.41°` | `3.69°` | Pass |
| J4 | `-17.75°` | `2.25°` | `1.49°` | `-16.96°` | Pass |
| J5 | `87.53°` | `107.53°` | `106.69°` | `88.76°` | Pass |
| J6 | `53.26°` | `73.26°` | `72.15°` | `54.75°` | Pass |

Final settled state after all tests:

```text
power: 1
controller_connected: 1
all_servos_enabled: 1
angles: [2.72, 10.28, 2.72, -18.10, 87.89, 53.70]
coords: [26.8, -90.0, 512.4, -98.52, 53.20, -96.23]
error: 0
robot_status: [0, 0, 0, 0, 0, 0]
next_error: [0, 0, 0, 0, 0, 0, 0]
servo_status: [0, 0, 0, 0, 0, 0]
voltages: [23.5, 23.8, 23.5, 23.8, 23.6, 23.5]
temperatures: [37, 30, 39, 40, 37, 37]
```

## 10. Reading angles and Cartesian coordinates

Use read-only calls whenever diagnosing an unknown state:

```bash
python3 - <<'PY'
from pymycobot.mycobot320 import MyCobot320

mc = MyCobot320('/dev/ttyAMA0', 115200, timeout=1)
try:
    print('angles_deg:', mc.get_angles())
    print('coords_xyz_rpy:', mc.get_coords())
finally:
    mc.close()
PY
```

The coordinate list is:

```text
[x, y, z, rx, ry, rz]
```

The first three values are Cartesian position and the last three are end-effector orientation. Do not command an arbitrary coordinate merely because it is numerically inside a documented range; inverse kinematics, joint limits, singularities, and physical interference may still make it unreachable. Prefer teaching a verified pose and reading it back.

## 11. J1 calibration procedure

Calibration is **not** a normal startup action. It changes the stored zero reference and should be performed only when the physical zero is known to be wrong.

During this incident, J1 calibration initially returned `-1` while the robot reported power off. After power-on and after the operator confirmed the physical zero alignment, calibration returned `1` and J1 read `0.0°`.

### Preconditions

1. Secure the robot base.
2. Clear the workspace.
3. Identify the manufacturer's physical zero marks/grooves for J1.
4. Support the arm as necessary.
5. Place J1 precisely at the mechanical zero reference.
6. Release the emergency stop.
7. Establish serial communication and call `power_on()`.
8. Verify controller connection and servo health.

### Calibration command

```bash
python3 - <<'PY'
from pymycobot.mycobot320 import MyCobot320
import time

mc = MyCobot320('/dev/ttyAMA0', 115200, timeout=1)
try:
    assert mc.is_power_on() == 1, 'Robot is not powered on'
    assert mc.is_controller_connected() == 1, 'Controller is not connected'
    assert mc.is_all_servo_enable() == 1, 'A servo is unavailable'
    assert mc.get_error_information() == 0, 'Clear the physical fault first'

    print('angles_before:', mc.get_angles())
    result = mc.set_servo_calibration(1)
    time.sleep(2)
    print('calibration_result:', result)
    print('angles_after:', mc.get_angles())
    print('error_after:', mc.get_error_information())
finally:
    mc.close()
PY
```

Required result:

- `calibration_result` is `1`.
- J1 reads approximately `0.0°` at the physical J1 zero mark.
- Servo and robot status remain clean.

Do not use calibration to conceal an out-of-limit pose, collision error, damaged joint, or wrong mechanical alignment. Calibrating at an arbitrary pose creates a false coordinate system and can make later motion unsafe.

## 12. Fault diagnosis and recovery

### 12.1 Decision table

| Symptom | Meaning or likely cause | Correct action |
|---|---|---|
| `/dev/ttyAMA0` missing | UART/device-tree configuration or OS image problem | Inspect `/boot/firmware/config.txt`, device aliases, and kernel logs; do not run motion code |
| Permission denied opening port | User not in `dialout`, wrong device permissions | Add the intended user to `dialout`, log out/in, and recheck permissions |
| Port busy | Another Python, ROS, MyBlockly, server, or serial console process owns it | Identify with `fuser -v /dev/ttyAMA0`; stop only the known competing application |
| `is_power_on() == 0` | Controller replied, but robot power is off | Check physical power and emergency stop, then call `power_on()` after physical preflight |
| API returns `-1` | Invalid/no reply for that request | Check power first, then controller connection, port ownership, baud, wiring, and firmware |
| Error `1`–`6` | Corresponding joint is beyond a controller limit | Stop motion; support/reposition the affected joint safely inside its configured limits |
| Error `16`–`19` | Collision protection | Stop; remove interference; use a clear supported pose; clear/recheck only after the physical cause is removed |
| Error `32` | No inverse-kinematics solution | Use a reachable pose; prefer a previously taught/read coordinate |
| Error `33`–`34` | Linear-path solution problem | Choose another path/pose; do not repeatedly resend the same command |
| Servo status contains nonzero entries | Voltage, encoder, temperature, current, angle, or overload fault | Stop, record diagnostics, cool/check hardware, and consult the API mapping/support guidance |
| Send returns `1`, angle does not change | Command accepted but motion blocked or failed | Check measured angle, errors, pause state, collision/limit state, and servo health |

### 12.2 Minimal serial diagnostics

```bash
ls -l /dev/ttyAMA0 /dev/serial0
groups
systemctl is-active serial-getty@ttyAMA0.service
systemctl is-enabled serial-getty@ttyAMA0.service
fuser -v /dev/ttyAMA0
stty -F /dev/ttyAMA0 -a
```

The serial getty should remain disabled:

```bash
systemctl is-active serial-getty@ttyAMA0.service
# expected: inactive

systemctl is-enabled serial-getty@ttyAMA0.service
# expected: disabled
```

### 12.3 Protocol debug test

Use debug mode to distinguish a dead UART from a controller that is alive but powered off:

```bash
python3 - <<'PY'
from pymycobot.mycobot320 import MyCobot320

mc = MyCobot320('/dev/ttyAMA0', 115200, timeout=1, debug=True)
try:
    print('power:', mc.is_power_on())
    print('connected:', mc.is_controller_connected())
    print('system_version:', mc.get_system_version())
    print('atom_version:', mc.get_atom_version())
    print('angles:', mc.get_angles())
finally:
    mc.close()
PY
```

If a request produces an RX frame, the UART path is functioning for that request. An empty RX after retries indicates no valid reply to that particular command; interpret it together with power status.

### 12.4 Collision or out-of-limit recovery

1. Stop issuing movement commands.
2. Record angles, coordinates, error code, robot status, and servo status.
3. Check each angle against the live values from `get_joint_min_angle()` and `get_joint_max_angle()`.
4. Inspect the robot physically for a hard stop, self-interference, trapped cable, payload collision, or external obstruction.
5. Do not disable collision protection to force a move.
6. If the pose cannot be corrected under normal control, an operator must physically support the arm before any servo is released or emergency-stop/manual repositioning is used.
7. Move the affected joint into a clearly valid, non-interfering pose.
8. Release the emergency stop only after the arm is supported and clear.
9. Call `power_on()` and wait three seconds.
10. Call `clear_error_information()` once after the physical cause is gone.
11. Call `resume()` if the controller remains paused.
12. Repeat the complete health gate.
13. Begin with a small, low-speed, observed motion and verify actual feedback.

> [!WARNING]
> `release_servo(joint_id)` or `release_all_servos()` can allow links to fall under gravity. Do not call either remotely unless an operator is physically supporting the arm and has explicitly confirmed readiness.

### 12.5 Emergency-stop recovery

1. Treat an emergency-stop event as a new startup; never assume the previous software state remains valid.
2. Identify and remove the reason for the stop.
3. Support and reposition the arm if necessary while power is safely removed.
4. Clear the workspace.
5. Release the emergency stop.
6. Confirm the Pi is reachable and `/dev/ttyAMA0` is available.
7. Call `power_on()` explicitly.
8. Wait three seconds.
9. Run every health-gate check.
10. Resume normal operation only when errors and all status arrays are zero.

## 13. Safe shutdown procedure

### Stop only the current application

1. Stop sending new motion commands.
2. Wait for motion to finish or call `stop()` if an immediate controlled stop is required.
3. Read and record the final angles, coordinates, and error state.
4. Close the serial connection.

### De-energize the arm but leave the Pi running

With the arm in a supported pose:

```bash
python3 - <<'PY'
from pymycobot.mycobot320 import MyCobot320

mc = MyCobot320('/dev/ttyAMA0', 115200, timeout=1)
try:
    print('power_off_result:', mc.power_off())
    print('power_after:', mc.is_power_on())
finally:
    mc.close()
PY
```

Expected result: `power_off_result: 1` and `power_after: 0`.

### Shut down the Raspberry Pi

1. De-energize robot motion as above.
2. Close all applications using the serial port.
3. Save logs and work.
4. Use the OS shutdown command:

   ```bash
   sudo shutdown -h now
   ```

5. Wait until the Pi has fully halted before removing external power.

Do not remove power while the filesystem is active unless an emergency requires it.

## 14. Startup automation policy

### Current state

There is no systemd, cron, desktop-autostart, or `rc.local` entry that starts a MyCobot control program. `/etc/rc.local` starts only the cooling fan. This is why robot API power-on and validation must be performed each session.

### Recommended policy

Do **not** put `mc.power_on()` or a motion program in unattended OS startup. Linux cannot verify that:

- the arm is physically clear;
- the base is secure;
- a person is outside the workspace;
- the emergency stop is reachable;
- a payload or tool has changed;
- the arm was manually moved while de-energized.

For repeatability, automate diagnostics but retain a human-controlled transition to powered motion:

1. Boot the Pi and cooling fan automatically.
2. Perform physical preflight.
3. Run `mycobot_startup_check.py --power-on` manually.
4. Require `health_gate_passed: true`.
5. Start the intended robot application manually or through a supervised operator interface.

If unattended operation is a future requirement, it needs a separate safety design: guarded workspace, interlocks, risk assessment, known homing strategy, startup-state sensing, watchdog behavior, and a tested recovery plan. A systemd unit alone is not sufficient.

## 15. Maintenance and change control

### Before changing software or firmware

1. Record the currently working versions:

   ```bash
   python3 -c "import pymycobot; print(pymycobot.__version__)"
   ```

   ```python
   print(mc.get_system_version())
   print(mc.get_atom_version())
   ```

2. Save a copy of relevant boot and application configuration.
3. Read the release notes and confirm MyCobot 320 Pi compatibility.
4. Change one layer at a time: Python library, then firmware only if required—not both simultaneously.
5. Repeat the physical preflight, read-only health gate, and supervised low-speed motion test after each change.
6. Record the new version and results in a dated maintenance log.

### Do not change a working configuration casually

- Do not change the baud rate from `115200` for this arm without model-specific official instructions.
- Do not enable a serial console on `/dev/ttyAMA0`.
- Do not run multiple processes against the same serial device.
- Do not change joint min/max parameters to hide a calibration or collision problem.
- Do not flash firmware as the first response to a power-state or physical-pose problem.
- Do not recalibrate joints during routine startup.

### Suggested per-run log record

Record at least:

```text
Timestamp:
Operator:
Physical preflight passed: yes/no
Emergency stop tested/released: yes/no
System version:
Atom version:
pymycobot version:
Power/controller/all-servos status:
Robot error/status:
Servo status:
Voltages:
Temperatures:
Start angles:
Start coordinates:
Program/test executed:
End angles:
End coordinates:
End error/status:
Incidents or observations:
```

## 16. Official references

Accessed during the 2026-09-10/11 troubleshooting and documentation session:

1. [MyCobot 320 Pi first-time self-check](https://docs.elephantrobotics.com/docs/mycobot_320_pi_en/3-UserNotes/320_PI/1-first_time_self_check.html) — physical prerequisites, `/dev/ttyAMA0`, `115200`, and sequential joint test.
2. [MyCobot 320 Pi Python environment](https://docs.elephantrobotics.com/docs/mycobot_320_pi_en/10-ApplicationBasePython/10.1_320_PI-ApplicationPython/1_download.html) — Pi Python environment, Atom firmware prerequisite, and API initialization.
3. [MyCobot 320 API reference](https://docs.elephantrobotics.com/docs/mycobot_320_pi_cn/10-ApplicationBasePython/10.1_320_PI-ApplicationPython/2_API.html) — power, connection, error, angle, coordinate, limit, servo, calibration, stop, and resume methods.
4. [MyBlockly 320 Pi interface description](https://docs.elephantrobotics.com/docs/mycobot_320_pi_en/5-BasicApplication/5.2-ApplicationUse/5.2.1-myblockly/320pi/3-interface_description.html) — required model/serial/baud selection, power-on, and timing between actions.
5. [MyCobot 320 Pi hardware FAQ](https://docs.elephantrobotics.com/docs/mycobot_320_pi_en/3-UserNotes/320_PI/3_hardware.html) — zero-position marks, calibration guidance, firmware, and hardware troubleshooting.
6. [MyCobot 320 Pi system manual](https://docs.elephantrobotics.com/docs/mycobot_320_pi_en/5-BasicApplication/5.1-SystemUsageInstructions/320pi/5.1-SystemUsageInstructions.html) — Ubuntu/Pi environment, SSH/network behavior, and installed software context.
7. [MyCobot 320 safety instructions](https://docs.elephantrobotics.com/docs/mycobot_320_pi_en/3-UserNotes/320_M5/3.1.1-SafetyInstruction/1-SafetyInstruction.html) — secure mounting, safe motion, supported servo release, and electrical precautions.
8. [MyCobot 320 Pi specifications](https://docs.elephantrobotics.com/docs/mycobot_320_pi_en/2-ProductFeature/2.2_320_PI_product/2.2.1-MachineSpecification.html) — 24 V input, workspace, payload, environmental, and general joint specifications.

## 17. Final operational status

At the end of the documented recovery and test:

- J1 zero calibration had completed successfully.
- Raspberry Pi UART communication was working.
- Robot power, controller connection, and all servo enable states were healthy.
- All robot, queued-error, and servo-status values were zero.
- All six joints passed observed low-speed 20-degree movement and return tests.
- Angle and Cartesian telemetry were valid.
- The SSH session remained available for supervised operation.

The robot is ready for operation when—and only when—the physical preflight and complete software health gate are repeated successfully for the current run.
