# MyCobot 320 Pi: complete gripper, pickup, jogging, and IK session record

Document date: **2026-09-13**, using the session's Asia/Calcutta date.
Repository: `Sanket-sourav/Robotic-ARM`.
Companion: [Working scripts and confirmed successes](WORKING_SCRIPTS_AND_SUCCESSES.md).

## 1. Scope and evidence

This records the work requested, implemented, inspected, and tested throughout
the available chat, including unsuccessful experiments and corrections to earlier
explanations. It is a technical handoff, not a verbatim transcript.

Some early exchanges survive as user requests and saved source files rather than
complete terminal logs. They establish the intended workflow and the code that
was retained, but do not establish every intermediate execution result. The
later debugging, SSH, source inspection, and test outputs provide direct evidence.

Use these distinctions when interpreting this document:

| Evidence category | What it establishes |
|---|---|
| User request | Intended behavior; not evidence that it was implemented successfully |
| Saved source inspected | Exact behavior of the retained script; not proof of a completed physical operation |
| Recorded hardware feedback | Actual reported angles, coordinates, status, or error during a test |
| Software test | Behavior of the mathematical/controller implementation under tested inputs |
| User-described working script | Included in the working-workflow guide without inventing a missing success log |

No robot was moved while writing these two documentation files. Passwords,
authentication tokens, and other credentials are intentionally excluded.

The repository already contained a separate
[2026-09-11 startup and recovery runbook](../2026-09-11-sanket/MYCOBOT_320_PI_RUNBOOK.md).
That document is prior background, not evidence that its startup/calibration work
was repeated during this chat. It was left unchanged.

## 2. Hardware, software, and file inventory

| Item | Observed or retained value |
|---|---|
| Arm | Elephant Robotics MyCobot 320 Pi; six revolute joints |
| Gripper | Treated during this session as myGripper F100 / Pro force-controlled gripper |
| Gripper addressing | Pro API defaults to ID 14; no address change recorded |
| Pi account and most recent host | `er@172.20.10.2` |
| Earlier requested host | `er@10.190.238.43`; connection success not established by retained logs |
| Robot serial port | `/dev/ttyAMA0` |
| Baud rate | `115200` |
| Python robot API | `pymycobot 4.0.7`, `MyCobot320` class |
| Pi operating system | Ubuntu 20.04.4 LTS |
| Pi kernel | `5.4.0-1073-raspi`, AArch64 |
| System/Pico firmware | `1.5`, read from controller |
| Atom firmware | `5.2`, read from controller |
| Pi NumPy | `1.17.4` |
| Local development | Windows PowerShell; Python 3.12.10; NumPy 2.5.2 |
| Local working directory | `C:\Users\offic\OneDrive\Desktop\gripper_test` |
| Pi script directory | `/home/er` |

### Saved artifacts

| File | Purpose and status |
|---|---|
| `gripper.jpeg` | Close-up of the attached gripper, screen, wiring, and finger attachments |
| `situation.jpeg` | Arm on a table, gripper extending sideways, white block to the right of the base |
| `gripper1.py` | Minimal Pro-gripper open/close sequence with explicit three-second sleeps |
| `upright.py` | Request the all-zero joint pose at speed 10 |
| `pickup_box.py` | Accept XYZ, request a downward-oriented grasp, close gripper, return to zero joints |
| `jog.before_custom_ik.py` | Preserved native-jog revision with Cartesian/joint mode toggle |
| `jog.py` | Final custom FK/IK, joint-target-only keyboard controller |
| `test_jog.py` | Final 18-test software suite |
| `JOG_NOTES.md` | Detailed kinematics, sources, runtime behavior, and hardware test outcome |

The two photographs were re-inspected during documentation. Their contents help
describe the setup but do not yield a calibrated robot-frame block coordinate.
They are not uploaded by this documentation-only change. The F100 identification
comes from session context/API usage; the photograph is not a legible product
serial-number verification.

## 3. Chronological request and implementation record

### 3.1 Initial connection and gripper exercise

The user initially requested SSH access, identification of the gripper in
`gripper.jpeg`, and physical opening/closing after attachment to the MyCobot.
The user then asked for minimal `gripper1.py` code and an explanation of every line.

The retained script opens a `MyCobot320` connection, sends
`set_pro_gripper_open()`, waits three seconds, sends
`set_pro_gripper_close()`, waits three seconds, and closes the serial connection.
Its exact source and explanation are in the companion guide.

The early source was kept intentionally short. It has no status verification and
no `try/finally`; it does not prove an object is held. A missing early terminal
log should not be replaced by a claim of a verified complete grasp.

### 3.2 Official documentation and command meanings

The user asked for official gripper documentation and explanations of its control
commands. Later direct inspection verified the installed Pro methods and their
existence on `MyCobot320`, including open, close, status, torque, and gripper stop.
The manufacturer references are collected in section 12.

The robot arm's `close()` method closes its serial connection. It does **not** close
the gripper. Closing the gripper requires `set_pro_gripper_close()`.

### 3.3 White-block pickup and coordinate acquisition

The user supplied `situation.jpeg`, asked to pick up the white block, requested a
script that goes to a position and returns, and asked how to obtain the block's
coordinates. The request developed into `pickup_box.py` and keyboard teaching.

`get_coords()` gives the arm's current reported flange/tool pose; it does not locate
the block. A block target must be measured in the base frame, taught by moving to
a suitable grasp pose, or estimated from a calibrated camera. This session did
not implement calibrated image-to-robot conversion or establish a retained numeric
XYZ target for the pictured block.

The request to reconnect changed to `er@172.20.10.2`, which is the host used in
the later successful SSH/SCP logs.

### 3.4 Keyboard jogging and coordinate explanations

The user asked what jog controls are, requested a minimal keyboard script uploaded
as `jog.py`, and asked for clear X/Y/Z and RX/RY/RZ explanations.

Translations describe flange position in millimetres. RX/RY/RZ describe orientation
in degrees when reporting a pose; the manufacturer conversion uses
`Rz(rz) @ Ry(ry) @ Rx(rx)`. Joint angles J1-J6 are different quantities: rotating
one joint generally changes multiple Cartesian position/orientation components.

The keyboard mapping that survived into later versions is:

| Keys | Cartesian function | Joint-mode function |
|---|---|---|
| `w` / `s` | X positive / negative | J1 positive / negative |
| `a` / `d` | Y positive / negative | J2 positive / negative |
| `r` / `f` | Z positive / negative | J3 positive / negative |
| `i` / `k` | Rotation about X | J4 positive / negative |
| `j` / `l` | Rotation about Y | J5 positive / negative |
| `u` / `n` | Rotation about Z | J6 positive / negative |
| `m` | Toggle Cartesian/joint modes | Same |
| `p` | Print measured pose/angles | Same |
| `h` | Show help | Same |
| `q` | Quit | Same |

Early native-jog revisions also had `o` and `c` for the gripper. The final custom-IK
task was arm-only, so those gripper actions are absent from its keyboard controller.

### 3.5 Return to upright

After reporting that jogging did not work properly, the user requested an
immediate return to the default upright state. The retained `upright.py` requests
`[0, 0, 0, 0, 0, 0]` through `sync_send_angles(..., 10, timeout=30)`.

Later recorded angles near zero establish that an approximately upright state
was reached during the overall workflow. The available record does not contain
a complete original `upright.py` execution log. Its print message alone is not
an independent completion check: the synchronous helper can finish its timeout
without certifying that the intended angles were actually achieved.

### 3.6 Pickup simplification, approach height, and gripping force

The requested pickup sequence evolved as follows:

1. Go to a coordinate, pick up a box, and return to the initial position.
2. Explain every line and keep the script lean.
3. Approach 50 mm above the target, point down, open, descend, close, and return.
4. Remove sleeps and check IK feasibility/restrictions.
5. Later remove the 50 mm approach: go directly to the location and grasp.
6. Increase grip strength because the block slipped; prefer a top-down grasp.
7. Increase grip strength again because slipping persisted.

The retained `pickup_box.py` reflects the direct-target version, not the earlier
50 mm approach. It uses `downward = [180, 0, 0]`, torque setting `60`, three
`robot.wait(2)` calls, and an all-zero return joint pose. Intermediate torque values
and every edit are not recoverable from the saved source; no values are invented here.

Important differences between requests and retained code:

- `home` is a fixed all-zero joint vector, not a captured initial pose.
- Built-in `solve_inv_kinematics()` is used in this older pickup script.
- Gripper opening occurs at home, before travel to the target.
- There is no separate approach/retreat waypoint or calibrated fingertip offset.
- It returns home before checking whether the previously read gripper status is 2.
- There is no explicit `sleep()` call, but `robot.wait(2)` is still a deliberate delay,
  and synchronous helpers may contain their own sleeps.
- The user reported slipping; the saved code is not proof of reliable load retention.

These observations were documented, not silently fixed during this documentation task.

### 3.7 Native jog API requested and uploaded

The user asked to use the library's jog functionality and accept a new key as soon
as the previous command finishes, without sleeps. The relevant revision used
`jog_increment_coord(axis, increment, speed)` and polled completion.

An SCP upload to `/home/er/jog.py` completed. Remote `python3 -m py_compile` passed.
Syntax checking establishes Python validity, not hardware API compatibility.

### 3.8 Attribute-error diagnosis

The user reported that `jog_stop` did not exist. SSH inspection of the actual
installed class confirmed:

```text
pymycobot version: 4.0.7
jog_stop: absent
stop: present
jog_increment_coord: present
jog_increment_angle: present
```

Both `robot.jog_stop()` calls were replaced with `robot.stop()`. Other used
robot methods were checked with `hasattr`. The corrected script was uploaded,
compiled on the Pi, started without motion, and exited successfully with `q`.
That was an API/startup test only; it did not validate motion completion.

### 3.9 Live tests of the native-jog script

The user explicitly requested a small physical movement to reproduce the errors.
The starting pose was approximately:

```text
[-5.3, -154.2, 523.8, -90.0, 0.52, 179.2]
```

An initially proposed upward test was changed to X +5 mm because Z was near the
upper workspace value. The X request timed out after 15 seconds. Subsequent
feedback was approximately `[-6.2, -154.2, 523.8, -90.0, 0.7, 179.2]`, idle with error 0.
There was no evidence that the desired X target was reached.

A Z -5 mm request then produced error 33. Joint readings were near zero:

```text
[-0.43, 0.35, 0.35, 0.17, -0.35, -0.35]
```

A direct native J2 +1 degree test produced movement. Its early sampling considered
the joint close enough to the target; later measurements showed J2 at 1.49 degrees.
A further Z -5 mm Cartesian request again produced error 33.

The script was then changed to:

- add `m` and 1-degree joint-mode increments;
- use measured pose/angle tolerances plus controller idle for completion;
- catch movement errors and remain at the keyboard prompt;
- retain native Cartesian and joint jog calls.

The revised script produced these recorded successful small steps:

| Test | Evidence |
|---|---|
| J2 +1 degree | J2 changed from 1.49 to 2.46 degrees |
| X +5 mm | Target X -13.5 mm; subsequent X -13.3 mm |
| X -5 mm | Target X -18.3 mm; subsequent X -17.3 mm, accepted by that revision's 1 mm tolerance |
| Quit | Serial connection closed and shell prompt returned |

Those successes belong to the native-jog revision, now
`jog.before_custom_ik.py`. They do not establish that every Cartesian direction
works from every pose or validate the later custom solver on hardware.

### 3.10 Correcting the interpretation of error 33

Earlier responses attributed error 33 specifically to a singular upright pose
and suggested that a small J2 move solved the problem generally. Later direct
source/documentation review corrected both statements:

| Error | Verified meaning |
|---|---|
| 0 | No reported controller error |
| 1-6 | Corresponding joint exceeds its limit |
| 16-19 | Collision protection |
| 32 | No inverse-kinematics solution |
| 33-34 | Linear motion has no adjacent solution |

An upright/near-singular posture can contribute to a Cartesian planning failure,
but error 33 alone does not prove singularity. Changing J2 alone is not a universal
escape: elbow extension and wrist-axis alignment also matter.

Likewise, changing an exact-position check did not by itself establish the cause
of the original timeout. Later tests showed genuine off-target outcomes as well.

### 3.11 Astra prompt and custom IK requirement

The user requested a clearer prompt for Astra describing game-like keyboard
control with custom inverse kinematics and joint commands only. A structured
prompt was produced. The user subsequently submitted that specification as the
implementation task in this chat.

The requested scope was arm-only: 5 mm translations, 2-degree rotations, no
Cartesian command APIs, no built-in IK, no sleep calls, bounded movement, responsive
keyboard input, software tests before hardware, and exactly one minimal hardware
test after validation.

### 3.12 Source review and resolving model disagreements

The installed library and official documentation were inspected for joint limits,
serial behavior, available methods, status/error handling, coordinate conversion,
URDF transforms, and ROS joint sign mapping.

The Pi coordinate/DH page disagreed with the Pi URDF and live/API limits. The
implementation therefore used the official Pi URDF chain, rather than mixing
numbers from different models or silently accepting the conflicting DH table.

Its parent-to-joint fixed transforms are:

| Joint | Translation in mm | Fixed roll, pitch, yaw in degrees |
|---|---|---|
| J1 | 0, 0, 173.9 | 0, 0, 0 |
| J2 | 0, -88.78, 0 | 0, -90, 90 |
| J3 | 135, 0, -88.78 | 0, 0, 0 |
| J4 | 120, 0, 88.78 | 0, 0, 90 |
| J5 | 0, -95, 0 | 90, 0, 0 |
| J6 | 0, 65.5, 0 | -90, 0, 0 |

Each joint then rotates about its local positive Z axis. The official ROS slider
converts joint radians directly to command degrees, without extra sign changes.
The model uses exact quarter-turn rotations instead of the URDF's rounded 1.5708.

At zero joints, it predicts flange XYZ `[0, -154.28, 523.9]` mm and orientation
`[-90, 0, 180]` degrees. At a later measured nonzero configuration:

```text
Joints:       [-73.3, -0.96, -0.43, 0.17, 73.12, 0.17] degrees
FK XYZ:       [-83.17765, -97.88990, 522.48970] mm
Firmware XYZ: [-83.1, -97.9, 522.4] mm
FK RPY:       [-91.16743, -0.18426, 179.82015] degrees
Firmware RPY: [-91.17, -0.18, 179.82] degrees
Agreement:    0.119 mm and 0.005 degrees
```

This validates consistency with controller feedback, not independent physical
metrology or a gripper fingertip calibration.

The live library/firmware limits were also compared:

| Joint | Library limit, degrees | Firmware limit, degrees | Final software limit with margin |
|---|---|---|---|
| J1 | +/-168 | +/-168 | +/-167 |
| J2 | +/-135 | +/-135 | +/-134 |
| J3 | +/-145 | +/-145 | +/-144 |
| J4 | +/-145 | +/-148 | +/-144 |
| J5 | +/-168 | +/-168 | +/-167 |
| J6 | +/-180 | +/-180 | +/-179 |

No controller joint limits were widened or disabled.

### 3.13 Serial ownership and backup

During read-only inspection, another `python3 jog.py` process was found (PID 2948).
The user was asked to quit it and confirmed it was closed. A process check then
showed no matching jog process before continued validation.

The previous source was preserved locally and on the Pi as
`jog.before_custom_ik.py`. The current controller uses an application file lock
and pyserial exclusive access. These cannot prevent interference from every
unrelated program that ignores locks; only one control application should use
the UART at a time.

### 3.14 Implementing independent FK/IK

The new `jog.py` constructs forward kinematics from the transforms above. Its
geometric Jacobian describes how each joint's small rotation changes flange
translation and orientation. The orientation error is a rotation-matrix logarithm,
so Euler wraparound is not treated as a large physical rotation.

It then solves a bounded, damped least-squares problem from the measured joints:

```text
e = [target_position - current_position;
     Log(target_rotation * current_rotation_transpose)]

J = geometric Jacobian
Scale rotational rows of e and J by 100 mm/rad.

dq = solve(J.T*J + (damping^2 + preference)*I,
           J.T*e - preference*(q - start))
```

Line search accepts only improving trial steps. The preference term favors a
small change from the current joints. It searches a local branch; it does not
guarantee the globally nearest solution among all mathematical IK branches.

For translations, the target moves along a fixed base axis with orientation
held. For rotations, the flange centre stays fixed while the target orientation
rotates about a fixed base axis. Those rotation keys are not simple increments
to an Euler display field.

### 3.15 Limits, completion, and input handling

| Policy | Final value or behavior |
|---|---|
| Translation step | 5 mm |
| Rotation step | 2 degrees about a base axis |
| Manual joint-mode step | 1 degree |
| Per-iteration solver cap | 2 degrees |
| Whole-command joint cap | 5 degrees per joint |
| Joint-limit margin | 1 degree inside strictest reviewed/live limits |
| IK residual tolerance | 0.15 mm / 0.1 degree |
| Wire precision | Targets rounded to 0.01 degree, then revalidated |
| Quantized residual gate | 0.25 mm / 0.15 degree |
| Orientation weighting | 100 mm/rad |
| Minimum scaled singular value | 0.5 mm/rad |
| Maximum scaled condition number | 1000 |
| Planned joint-path checks | 21 interpolation samples |
| Allowed sampled path deviation | 1 mm / 1 degree |
| Movement API | `send_angles()` at speed 10 |
| Completion | Controller idle and joint error <=0.25 degree |
| Final flange tracking gate | 2 mm / 1 degree |
| Overall movement timeout | 15 seconds |
| Stop confirmation deadline | 3 seconds |
| Stable off-target detection | At least 3 stable samples spanning at least 1 second |

These algorithm thresholds are application choices, not published manufacturer
performance guarantees. They cannot establish collision clearance.

A single worker owns robot I/O. The keyboard waits on terminal input and a worker
completion socket. Incoming movement keys while busy are discarded; successful
completion wakes the input loop without a fixed post-move delay. A pasted burst
does not become a stored sequence of future moves. Quit remains actionable while
work is in progress. Ctrl-C, SIGTERM, and SSH hangup also request cancellation.

The controller stops on movement errors. Only error codes 32-34 are automatically
cleared after stop verification. Collision, joint-limit, communication, unknown,
and malformed-feedback faults inhibit further movement until resolved. It never
releases motor torque as an error-handling shortcut.

No `sleep()` calls are present in the new controller. This does not mean zero
latency: keyboard repeat, Linux scheduling, UART round trips, and motor execution
take time. `send_angles()` still delegates servo execution/interpolation to
firmware; independent host IK does not replace the motor controller.

### 3.16 Software test development and results

Initial numerical tests included configurations where a 5 mm step could not be
accepted within the 5-degree joint cap. Those were not converted into successes
by widening runtime limits. Three better-conditioned numerical fixtures were
used to exercise all 12 directions, and difficult requests were retained as
explicit rejection cases.

The final test suite contains 18 tests, including:

- zero pose and recorded hardware-pose consistency;
- Jacobian comparison with finite differences;
- FK -> IK -> FK from perturbed joint seeds;
- all 12 Cartesian key directions from three numerical configurations;
- Euler wrap and 180-degree rotation handling;
- singular, unreachable, joint-limited, and excessive-joint-change rejection;
- repeated commands based on newly measured positions;
- recoverable controller errors and continued interaction;
- collision errors that are not cleared automatically;
- invalid acknowledgements and malformed/nonfinite telemetry;
- model mismatch preventing transmission;
- dry-run mode sending no joint commands;
- cancellation and verified stop behavior;
- idle status not being mistaken for successful completion;
- explicit stationary-off-target reporting.

All 18 passed locally and on the Pi. The final recorded Pi run took 5.933 seconds.
There are 36 direction subcases, not 36 physical robot movements. Numerical
fixtures are not approved real-arm operating poses.

### 3.17 Dry-run terminal test

The uploaded program was exercised interactively with `--dry-run`:

1. A Cartesian down key at the current poor-conditioning metric was rejected
   with a readable message, then `Ready` returned.
2. `m` switched to joint mode.
3. A burst `ffff` produced exactly one J3 -1 degree plan, not four queued plans.
4. `p` showed unchanged measured joints and pose.
5. `q` returned to the shell without movement.

The initial scaled Jacobian metric was sigma_min 0.3156, condition approximately
1500.3, beyond the application's acceptance thresholds.

### 3.18 Exactly one custom-controller hardware test

After software tests and dry-run validation, the following was executed:

```bash
python3 /home/er/jog.py --test-joint 3 --delta -1
```

It issued one six-joint target vector, changing only requested J3 by -1 degree,
at speed 10. Planned FK suggested about 4 mm flange displacement and a better
condition number of approximately 451.5.

The command was acknowledged and the arm moved, but did not meet the completion
tolerance. The script timed out, stopped, and confirmed idle. Five subsequent
stationary samples were identical:

| Joint | Requested degrees | Measured after stop | Measured minus requested |
|---|---:|---:|---:|
| J1 | -73.30 | -73.21 | +0.09 |
| J2 | -0.96 | -1.40 | -0.44 |
| J3 | -1.43 | -2.02 | -0.59 |
| J4 | +0.17 | -0.26 | -0.43 |
| J5 | +73.12 | +73.12 | 0.00 |
| J6 | +0.17 | +0.17 | 0.00 |

Controller feedback after stopping:

```text
XYZ/RPY:      [-80.3, -106.9, 519.4, -93.53, -0.89, 179.93]
is_moving:    0
error:        0
servo_status: [0, 0, 0, 0, 0, 0]
fresh mode:   0
movement type: 1
```

The last two settings were observed, not changed or established as a cause.
FK of the final measured joints differs from the planned target by approximately
5.99 mm and 1.46 degrees. The cause of the tracking discrepancy remains unknown.

This is **not a passing precision-motion test**. The absence of a reported
controller/servo fault does not prove that a target was reached. There was no
second movement or automatic return, in accordance with the one-test limit.

### 3.19 Final revision and deployment verification

The final revision added stationary-off-target detection so that stable, idle,
incorrect feedback is reported explicitly rather than always taking 15 seconds.
That change passed a simulated regression test. Tolerances were not loosened to
hide the mismatch. The final revision was uploaded with tests and notes.

Local and remote SHA-256 values agreed:

```text
jog.py
837560b32f24c562a31241f0d9218f38e5723a4b2dd84aca67c9c3991d005950

test_jog.py
3e00db9596ede051309490ffb33aa79a1f44e01ed66625eea4cb447ab3bc6021

JOG_NOTES.md
621cee854e5dc66ed4b83aaf5eb6638881759b420cec942f879026f5a12695e9
```

The final Pi self-test reported 18 passing tests. The SSH session was closed.
These hashes identify the deployed artifacts at that handoff, not arbitrary later
edits or the contents of this documentation directory.

## 4. Exact retained pickup sequence

The current `pickup_box.py` was inspected again during documentation. Its actual
sequence is:

```text
parse X Y Z
    -> target = [X, Y, Z, 180, 0, 0]
    -> power/controller check
    -> clear selected historical kinematic errors
    -> firmware IK seeded with zero joints
    -> go to zero joints at speed 20
    -> set gripper torque value 60
    -> wait 2 s
    -> open gripper
    -> wait 2 s
    -> go to solved joints at speed 15
    -> check joint target reached
    -> close gripper
    -> wait 2 s
    -> read gripper status
    -> return to zero joints at speed 20
    -> require the saved gripper status to equal 2
    -> close serial in finally
```

The torque value is a device/API setting, not a measured force in newtons. A
status interpreted as holding an object does not measure slip during travel.
The orientation is a requested flange pose, not proof that the specific finger
attachments are vertically aligned or that a top grasp is collision-free.

See the companion document for the exact retained code and line-by-line purpose.

## 5. What was actually successful

1. SSH/SCP communication with the Pi at the later address.
2. Inspection of actual installed API methods, versions, and joint limits.
3. Fixing the missing `jog_stop` attribute and passing a startup/quit test.
4. Specific small native J2 and X jogs, with recorded measured movement.
5. Preserving earlier scripts and documenting their actual command sequences.
6. Independent FK matching reported robot pose closely.
7. Custom IK and failure handling passing the numerical/mock test suite.
8. Real-terminal dry-run rejection, mode switching, burst handling, and quit.
9. Detection of a physical off-target outcome and confirmation of a stop.
10. Uploading and verifying the final custom-controller artifacts.

These are narrower claims than reliable arbitrary-axis physical teleoperation or
consistent successful pickup, which were not established.

## 6. Remaining limitations and open work

- Repeated block slipping was reported; a reliably retained top-down pickup was
  not demonstrated in retained logs.
- The white block has no retained calibrated XYZ/TCP teaching record.
- The latest custom-controller hardware test missed its joint target; investigate
  joint tracking before relying on its Cartesian accuracy.
- A single pose match to firmware is not independent calibration of all link
  geometry, encoder offsets, or gripper TCP.
- Joint interpolation sampling is not collision planning or exact swept-volume
  validation; the table, cables, block, and gripper geometry are not modeled.
- The older pickup and upright scripts have weaker timeout/error checking than
  the new controller, and still use synchronous helpers/delays.
- The older scripts clear 35/36 without a verified meaning retained in this
  session. The final custom controller only automatically clears 32-34.
- No algorithm removes genuine reach, singularity, collision, or joint-limit
  restrictions. Those should not be bypassed to force a target.

Any future hardware investigation is a new observed test, not a reinterpretation
of the failed test as a success.

## 7. Operational command reference

These commands document available entry points; do not run concurrent UART users.

```bash
# Software only: no robot API import or serial access.
python3 /home/er/jog.py --self-test

# Read-only hardware/model checks.
python3 /home/er/jog.py --check

# Keyboard planning, no joint movement.
python3 /home/er/jog.py --dry-run

# Live keyboard movement; physical tracking issue remains open.
python3 /home/er/jog.py

# Earlier working-workflow entry points; these command real motion.
python3 /home/er/upright.py
python3 /home/er/gripper1.py
# Supply actual taught coordinates, not the literal placeholders below.
python3 /home/er/pickup_box.py X Y Z
```

## 8. Documentation publication scope

This directory adds exactly two Markdown files. It does not replace repository
scripts, upload the photographs, change robot settings, or execute another robot
test. The companion includes source snapshots of the three smaller scripts so
their behavior remains understandable even though executable files are not added
to this documentation-only commit.

## 9. Interpretation of upright versus startup

`upright.py` is a motion script, not a full startup/recovery service. It checks
that power and controller connection are already available, requests zero joint
angles, and closes its connection. It does not itself perform `power_on()`, set
encoder zeros, release servos, or guarantee collision-free homing from any pose.

The repository's older startup runbook covers different procedures. Its prior
successful tests must not be confused with this session's minimal upright script
or with the later off-target six-angle command.

## 10. Practical meaning of the coordinate controls

X/Y/Z belong to a chosen base frame, not necessarily the camera's left/right/up.
Z positive is upward for the normally mounted arm. The pictured block's position
in a photograph is insufficient to derive millimetres in that frame.

RX/RY/RZ are an orientation representation. A 2-degree rotation about fixed base
X, as used by the custom controller, generally changes more than one displayed
Euler component. That is expected and is different from rotating joint J4 by
2 degrees. Flange orientation also differs from an explicitly calibrated gripper
tip pose when finger attachments introduce an offset.

## 11. Audit boundary

The relevant manufacturer sources and installed methods were reviewed thoroughly
for this task. Proprietary firmware internals and every file in the manufacturer's
repositories were not exhaustively audited. Source discrepancies, measured
outcomes, test scope, and unknown causes are recorded explicitly rather than
filled with assumptions.

## 12. Manufacturer sources used in this work

- [MyCobot 320 Pi documentation](https://docs.elephantrobotics.com/docs/mycobot_320_pi_en/).
- [MyCobot 320 Pi API](https://docs.elephantrobotics.com/docs/mycobot_320_pi_en/10-ApplicationBasePython/10.1_320_PI-ApplicationPython/2_API.html): joint commands, Pro-gripper APIs, telemetry, and error definitions.
- [Pi coordinate/DH page](https://docs.elephantrobotics.com/docs/mycobot_320_pi_en/2-ProductFeature/2.2_320_PI_product/2.2.5-CoordinateSystem.html): reviewed, but conflicting values were not used as if verified.
- [Official Pi 2022 URDF](https://github.com/elephantrobotics/mycobot_ros/blob/noetic/mycobot_description/urdf/mycobot_320_pi_2022/new_mycobot_pro_320_pi_2022.urdf): geometric model actually used.
- [Official Pi ROS slider](https://github.com/elephantrobotics/mycobot_ros/blob/noetic/mycobot_320/new_mycobot_320_pi/scripts/mycobot_320_slider.py): direct radians-to-degrees joint mapping.
- [MyCobot320 Python source](https://github.com/elephantrobotics/pymycobot/blob/main/pymycobot/mycobot320.py): online reference; runtime behavior was also checked against installed 4.0.7.
- [Python command generator](https://github.com/elephantrobotics/pymycobot/blob/main/pymycobot/generate.py): inherited joint/error interfaces.
- [Manufacturer Euler conversions](https://github.com/elephantrobotics/pymycobot/blob/main/pymycobot/tool_coords.py): orientation convention.

Links identify the sources consulted; online branches can change. Recorded
installed versions, measured values, and file hashes above identify this session.
