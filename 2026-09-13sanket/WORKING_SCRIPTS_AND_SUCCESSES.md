# Working scripts and confirmed successes: MyCobot 320 Pi

Date: **2026-09-13** (Asia/Calcutta).
Companion: [Complete session record](FULL_SESSION_RECORD.md).

## 1. Purpose and confidence boundary

This is the practical guide to the earlier working workflow: `upright.py`,
`gripper1.py`, `pickup_box.py`, and the small native jogs with recorded successful
feedback. It includes exact saved source snapshots and explains each executable
statement. The experimental custom-IK development history belongs in the companion.

The user identified upright and pickup as the successful-workflow scripts.
They are documented on that basis, with an important evidence distinction:
their saved source was verified, but complete early physical execution logs are
not retained. The native jog successes below do have measured before/after
feedback. This guide does not claim that every pickup retained the block or that
all arbitrary target coordinates have been physically validated.

No code was changed and no robot was moved to produce this guide.

## 2. Connection and operating baseline

| Setting | Value used |
|---|---|
| Robot | MyCobot 320 Pi |
| Python class | `pymycobot.MyCobot320` |
| Installed library | `pymycobot 4.0.7` |
| Serial device | `/dev/ttyAMA0` |
| Baud rate | `115200` |
| Pi account/host | `er@172.20.10.2` |
| Pi working directory | `/home/er` |
| Gripper API | Pro/F100 force-controlled gripper methods |
| Default Pro-gripper ID | 14, as used by the retained API defaults |

Connect interactively:

```bash
ssh er@172.20.10.2
```

Use the account's credentials interactively; none are stored in this guide.
Only one program should use the robot serial port at a time. Run these movement
scripts with the arm physically clear and observable. They do not replace the
[existing startup runbook](../2026-09-11-sanket/MYCOBOT_320_PI_RUNBOOK.md).

The photographed tool has finger attachments. Flange coordinates describe the
robot's mechanical output frame, not automatically the grasp point between
those fingertips.

## 3. upright.py: request the default all-zero joint pose

### Purpose

Request J1-J6 = `[0, 0, 0, 0, 0, 0]` degrees at speed 10.
This is the session's default upright pose.

Run:

```bash
python3 /home/er/upright.py
```

### Exact saved source

```python
from pymycobot import MyCobot320


robot = MyCobot320("/dev/ttyAMA0", 115200, timeout=1)

try:
    if robot.is_power_on() != 1 or robot.is_controller_connected() != 1:
        raise RuntimeError("Robot is not powered and connected")

    error = robot.get_error_information()
    if error in (32, 33, 34, 35, 36):
        robot.clear_error_information()
    elif error not in (0, None):
        raise RuntimeError(f"Robot controller error: {error}")

    robot.sync_send_angles([0, 0, 0, 0, 0, 0], 10, timeout=30)
    print("Upright pose reached:", robot.get_angles())
finally:
    robot.close()
```

### Why each statement is present

| Statement | Purpose |
|---|---|
| `from pymycobot import MyCobot320` | Import the robot-specific communication class. |
| `robot = MyCobot320(..., timeout=1)` | Open the Pi UART at the verified baud rate with a one-second serial timeout setting. |
| `try:` | Ensure the matching `finally` runs after normal execution or an exception inside this block. |
| Power/controller `if` | Require both status methods to return 1 before proceeding. |
| First `raise RuntimeError(...)` | Abort if the preflight check fails. |
| `error = robot.get_error_information()` | Read the controller error value. |
| `if error in (32, 33, 34, 35, 36)` | Historical list of errors this script attempts to clear. |
| `robot.clear_error_information()` | Request clearing that selected error state. |
| `elif error not in (0, None)` | Reject other reported error values; note that this historical code permits `None`. |
| Second `raise RuntimeError(...)` | Show the error code and abort the motion request. |
| `robot.sync_send_angles([0,...,0], 10, timeout=30)` | Send the six zero-angle targets at speed 10 and use the library's synchronous wait helper. |
| `print(..., robot.get_angles())` | Display measured joint angles after the helper returns. |
| `finally:` | Begin cleanup that runs whether the block completes or raises. |
| `robot.close()` | Close the serial connection; this is not a gripper-close command. |

Blank lines only separate logical blocks.

### How to interpret the result

An approximately upright state is supported by later recorded joint feedback
near zero. However, the text `Upright pose reached:` is printed by the script,
not independently certified by the hardware. Inspect the reported angles.
The inspected synchronous helper can return after its timeout; the saved script
does not add a separate target-tolerance assertion afterward.

This program does not call `power_on()`, calibrate encoder zeros, release servos,
or prove that a route to zero is clear from every starting posture. Its historical
35/36 clearing behavior is preserved here as source, not newly endorsed: the
session verified published meanings for 32-34, not 35/36.

## 4. gripper1.py: minimal open/close control

### Purpose

Open the Pro gripper, allow three seconds, close it, allow three seconds, then
close the serial connection.

Run:

```bash
python3 /home/er/gripper1.py
```

### Exact saved source

```python
from time import sleep

from pymycobot import MyCobot320


gripper = MyCobot320("/dev/ttyAMA0", 115200)

gripper.set_pro_gripper_open()
sleep(3)

gripper.set_pro_gripper_close()
sleep(3)

gripper.close()
```

### Why each statement is present

| Statement | Purpose |
|---|---|
| `from time import sleep` | Import the delay function used by this intentionally minimal early script. |
| `from pymycobot import MyCobot320` | Import the robot interface that relays Pro-gripper commands. |
| `gripper = MyCobot320(...)` | Open the robot serial connection; the variable name refers to the connection object, not a separate gripper driver. |
| `gripper.set_pro_gripper_open()` | Request gripper opening using the default gripper ID. |
| First `sleep(3)` | Allow time for the opening action. |
| `gripper.set_pro_gripper_close()` | Request gripper closing. |
| Second `sleep(3)` | Allow time for the closing action. |
| `gripper.close()` | Release the serial connection. |

This is the saved minimal command example. It predates the later no-sleep
keyboard requirement and explicitly contains sleeps. It does not read gripper
status, measure holding force, or guarantee object retention. It also lacks
`try/finally` around its sequence.

The installed class was verified to expose the Pro open/close methods.
A recorded open/close command sequence or photograph is not a calibrated force
measurement.

## 5. pickup_box.py: direct-target pickup workflow

### Purpose

Accept a taught X/Y/Z target, request a downward flange orientation, open the
gripper, move to the target, close it, and return to the fixed zero-joint pose.

Run only with actual measured or taught target values:

```text
python3 /home/er/pickup_box.py X Y Z
```

Replace `X Y Z` with three numeric millimetre values. The placeholders are not
literal shell arguments and this document does not supply an invented target
for the pictured block.

### Exact saved source

```python
#!/usr/bin/env python3
import sys

from pymycobot import MyCobot320


if len(sys.argv) != 4:
    raise SystemExit("Usage: python3 pickup_box.py X Y Z")

x, y, z = map(float, sys.argv[1:])
downward = [180, 0, 0]
target = [x, y, z] + downward
home = [0, 0, 0, 0, 0, 0]
robot = MyCobot320("/dev/ttyAMA0", 115200, timeout=1)

try:
    if robot.is_power_on() != 1 or robot.is_controller_connected() != 1:
        raise RuntimeError("Robot is not powered and connected")

    if robot.get_error_information() in (32, 33, 34, 35, 36):
        robot.clear_error_information()

    angles = robot.solve_inv_kinematics(target, home)
    if not isinstance(angles, list) or len(angles) != 6:
        robot.clear_error_information()
        raise RuntimeError("The pickup location is unreachable")

    robot.sync_send_angles(home, 20, timeout=30)
    robot.set_pro_gripper_torque(60)
    robot.wait(2)
    robot.set_pro_gripper_open()
    robot.wait(2)
    robot.sync_send_angles(angles, 15, timeout=30)
    if robot.is_in_position(angles, 0) != 1:
        raise RuntimeError("Robot did not reach the pickup location")
    robot.set_pro_gripper_close()
    robot.wait(2)
    status = robot.get_pro_gripper_status()
    robot.sync_send_angles(home, 20, timeout=30)

    if status != 2:
        raise RuntimeError(f"No object detected; gripper status was {status}")
finally:
    robot.close()
```

### What happens, in execution order

1. Parse the three supplied coordinates.
2. Build `target = [x, y, z, 180, 0, 0]`.
3. Check power and controller connection.
4. Clear an error in the historical kinematic-error list if present.
5. Ask firmware IK for a joint solution, seeded from zero joint angles.
6. Validate that the returned solution is a six-element list.
7. Go to all-zero joints at speed 20.
8. Set Pro-gripper torque parameter to 60 and wait two seconds.
9. Open the gripper and wait two seconds.
10. Move to the solved joint target at speed 15.
11. Require the robot to report that joint target reached.
12. Close the gripper, wait two seconds, and read its status.
13. Return to all-zero joints at speed 20.
14. Require the previously captured gripper status to be 2.
15. Close the serial connection in the `finally` block.

### Why each statement is present

| Statement | Purpose |
|---|---|
| `#!/usr/bin/env python3` | Select Python 3 when the file is invoked as an executable on Linux. |
| `import sys` | Access command-line arguments. |
| `from pymycobot import MyCobot320` | Import the robot interface. |
| `if len(sys.argv) != 4:` | Require the script name plus exactly three coordinate arguments. |
| `raise SystemExit("Usage: ...")` | Print usage and exit if the count is wrong. |
| `x, y, z = map(float, sys.argv[1:])` | Parse the three coordinates into numeric values; invalid text raises before opening serial. |
| `downward = [180, 0, 0]` | Choose the requested flange orientation in degrees. |
| `target = [x, y, z] + downward` | Combine position and orientation into a six-value target pose. |
| `home = [0, 0, 0, 0, 0, 0]` | Define the fixed return joint pose in degrees. |
| `robot = MyCobot320(..., timeout=1)` | Open the robot UART. |
| `try:` | Protect cleanup for the following robot operations. |
| Power/controller `if` and `raise` | Require reported power and controller readiness. |
| Error-code `if` | Check whether an error belongs to the historical clearing list. |
| First `clear_error_information()` | Clear that selected error state. |
| `angles = robot.solve_inv_kinematics(target, home)` | Ask the controller to solve the desired pose using zero joints as the reference seed. |
| Result-shape `if` | Require a list of six joint values before using it. |
| Second `clear_error_information()` | Attempt to clear error information after a rejected IK result. |
| `raise RuntimeError("The pickup location is unreachable")` | Abort this attempt. This message is the script's interpretation; an invalid reply can also indicate communication trouble. |
| First `sync_send_angles(home, 20, timeout=30)` | Move to the fixed home pose using the synchronous library helper. |
| `set_pro_gripper_torque(60)` | Set the gripper's API torque parameter to 60. It is not a force value in newtons. |
| First `robot.wait(2)` | Delay after setting the gripper parameter. |
| `set_pro_gripper_open()` | Open the gripper before the approach. |
| Second `robot.wait(2)` | Delay for opening. |
| `sync_send_angles(angles, 15, timeout=30)` | Move to the solved grasp joint configuration. |
| `if robot.is_in_position(angles, 0) != 1` | Check the grasp position in joint space; flag 0 means joint angles. |
| Corresponding `raise RuntimeError(...)` | Do not issue the close command if grasp-position confirmation fails. |
| `set_pro_gripper_close()` | Close the gripper at the requested grasp location. |
| Third `robot.wait(2)` | Delay before checking the gripper state. |
| `status = robot.get_pro_gripper_status()` | Save the gripper state reported before return travel. |
| Final `sync_send_angles(home, 20, timeout=30)` | Return to the fixed home joint pose. |
| `if status != 2:` | Apply this script's holding-object status criterion to the saved state. |
| Final `raise RuntimeError(...)` | Report the saved gripper status if it did not indicate an object. |
| `finally:` and `robot.close()` | Release serial even if a robot operation raises. |

### Terms that matter when reusing this script

- **Home is fixed:** it does not remember whatever pose the arm had at startup.
- **XYZ is taught/measured:** the script does not detect the white block in an image.
- **No 50 mm waypoint:** the earlier approach-height idea was removed.
- **Downward is requested flange orientation:** finger orientation and clearance
  still depend on the actual mounting and the chosen target.
- **Delays remain:** `robot.wait(2)` and synchronous helpers mean this is not a
  no-wait or no-latency implementation.
- **Object status is sampled before return:** the script does not continuously
  check grip retention while transporting the object.
- **Return completion is not separately asserted:** unlike the grasp-position
  check, the home commands rely on the synchronous helper.
- **Historical error handling is limited:** the script clears its listed codes
  but does not comprehensively reject all other faults or malformed numeric data.

These distinctions make the saved workflow explicit without claiming more than
its source and available evidence support. They are not edits to the script.

## 6. Obtaining a usable pickup coordinate

The successful building block is reading robot feedback at a deliberately taught
pose. `get_coords()` reads the flange/tool pose currently selected by the robot;
it does not detect objects. `get_angles()` reads the six measured joint angles.

For a future taught pickup, position the gripper under observation at the intended
grasp pose, record both kinds of feedback, and account for finger/TCP offsets.
Use that recorded position only if its frame and orientation agree with the
pickup script. No valid numeric target for the block was retained in this chat.

A picture by itself does not provide the camera calibration, depth, or base-frame
transform required for an automatic coordinate estimate.

## 7. Native keyboard jogging that was physically demonstrated

The successful small-motion tests used the revision saved as
`jog.before_custom_ik.py`, not the final custom solver.

| Test | Recorded result |
|---|---|
| Joint mode, J2 +1 degree | J2 changed from 1.49 to 2.46 degrees |
| Cartesian X +5 mm | Target X -13.5 mm; measured X -13.3 mm |
| Cartesian X -5 mm | Target X -18.3 mm; measured X -17.3 mm, within that revision's tolerance |
| Startup and quit | Help appeared and `q` closed serial successfully |

That version used `jog_increment_angle()` for joint steps,
`jog_increment_coord()` for Cartesian steps, and `stop()` on its movement-error
paths. The installed library supports `stop()`; it does not provide `jog_stop()`.

Its step settings were 5 mm translation, 2 degrees orientation, 1 degree joint
increments, speed 10, with `m` toggling the two modes. These individual tests
support the listed movements, not unrestricted Cartesian control from any posture.

The filename `/home/er/jog.py` now refers to the later custom-IK controller.
Consult the full record before choosing a version; do not assume a historical
success log describes the current file.

## 8. Other confirmed reusable results

These successful components can be reused without treating the experimental
controller as physically validated:

- The official Pi URDF forward model agreed with reported flange feedback within
  0.119 mm and 0.005 degrees at the measured validation configuration.
- The final local FK/IK and controller test suite passed all 18 tests on Windows
  and on Pi NumPy 1.17.4, including 36 Cartesian direction subcases.
- Actual terminal dry-run mode handled mode changes, pose printing, rejected
  plans, a repeated-key burst, and quit without issuing a joint move.
- Final file uploads were verified by matching local/remote SHA-256 hashes.
- The previous jogging source was preserved as a backup.

The custom controller's unsuccessful physical precision test is deliberately
not presented as a successful motion example here. Its full result is in the
companion record.

## 9. Choosing an entry point

| Need | Entry point | What to expect |
|---|---|---|
| Request upright zero joints | `python3 /home/er/upright.py` | Real arm motion; inspect measured final angles |
| Exercise open/close | `python3 /home/er/gripper1.py` | Real gripper motion; simple timed sequence |
| Run the retained direct pickup workflow | `python3 /home/er/pickup_box.py X Y Z` | Requires real taught numeric coordinates |
| Inspect the custom solver without hardware | `python3 /home/er/jog.py --self-test` | Numerical and mock-controller checks |
| Check live model/feedback without movement | `python3 /home/er/jog.py --check` | Read-only preflight |
| Preview keyboard plans | `python3 /home/er/jog.py --dry-run` | No joint commands |

The script code blocks above are snapshots of the local files inspected for this
document. The documentation upload does not install or replace executable
programs in this GitHub repository.

## 10. Official references

- [MyCobot 320 Pi Python API](https://docs.elephantrobotics.com/docs/mycobot_320_pi_en/10-ApplicationBasePython/10.1_320_PI-ApplicationPython/2_API.html): arm and Pro-gripper commands, coordinate feedback, status, and error definitions.
- [Official MyCobot320 implementation](https://github.com/elephantrobotics/pymycobot/blob/main/pymycobot/mycobot320.py): online source; the installed version was separately inspected.
- [Official Pi URDF](https://github.com/elephantrobotics/mycobot_ros/blob/noetic/mycobot_description/urdf/mycobot_320_pi_2022/new_mycobot_pro_320_pi_2022.urdf): validated model used in the later mathematical work.
- [Official Pi ROS slider](https://github.com/elephantrobotics/mycobot_ros/blob/noetic/mycobot_320/new_mycobot_320_pi/scripts/mycobot_320_slider.py): joint radians-to-degrees mapping.
