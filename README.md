# TCode Specification
**Author:** Tempest Multiaxis  
**Date:** 12 October 2026

<!--- credit to Tempest & the community. --->

## version `0.4` as of 12th October 2026

T-code is a protocol for implementing UART serial communications to an adult toy. It is partly influenced by G-code, which is an alphanumeric format used to drive CNC machines, including 3D-printers.

The toy or “device” will be capable of one or more functions, which are described as Linear motion, Vibration or Rotation “channels”, which can be addressed independently.

Commands are sent to the device in the form of alphanumeric ASCII phrases, which are interpreted by the device, stored in a buffer and executed on the receipt of new line or `/n` character. Commands can be sent over Serial, Bluetooth or WiFi (UDP).

Version 0.4 extends version 0.3 with support for smooth motion terms that enable cubic-spline and quadratic motion profiles, improving smoothness when chaining movements.

## Axes Conventions

The range of each axis is `[0, 1)`.

- the minimum position for each axis is `0`
- the maximum position is `0.9999...`

The default configuration for a Multi Axis Stroker Robot (or "MAxSR") is as follows:

- Linear motions in three axes:

  - `L0` is positive up relative to the user
  - `L1` is positive moving away from the user
  - `L2` is positive moving to the user's left

- Rotation in three axes:
  - `R0`, `R1`, `R2` are positive according to the right-hand rule around `L0`, `L1`, `L2` respectively

Visually:

![Axis Diagram][axis diagram]

## Basic Command Syntax

Basic or "short" commands for **L**inear move, **R**otate, **V**ibrate, **A**uxiliary take the form of a letter (`L`, `R`, `V`, or `A`) followed by a set of digits. Lower case letters (`l`, `r`, `v`, `a`) are also valid.

### Example Basic Commands

| Command   | Axis     | Channel ID | Magnitude |
| --------- | -------- | :--------: | --------- |
| `L277`    | Linear   |    `2`     | `0.77`    |
| `R09`     | Rotation |    `0`     | `0.90`    |
| `V317439` | Vibrate  |    `3`     | `0.17439` |

### Anatomy of a basic command: `X&$$`

The first digit `&` represents the channel ID (e.g. `0`-`9`).

Any subsequent digits (in this case `$$`) are the magnitude.

The magnitude for any channel is always a number greater than or equal to `0.0` and strictly less than `1.0`. The digits follow a leading decimal point that is omitted; they are effectively the fractional part after `0.`.

The magnitude corresponds to a linear position/speed, rotation position/speed, vibration level, or some auxiliary setting. In situations where position or speed is possible in a forward or reverse direction 0.5 is assumed to be central or at rest. 

## Extended Commands

Magnitude commands of the format `X&$$` can be augmented with additional terms to change the ramp time or speed.

| Command     | Axis    | Channel | Effect                                  |
| ----------- | ------- | :-----: | --------------------------------------- |
| `V199I2000` | Vibrate |   `1`   | Ramp to `0.99` over `2000` milliseconds |
| `L020S100`   | Linear  |   `0`   | Ramp to `0.2` at a rate of `0.1`/sec    |

![Interval and Speed][interval and speed diagram]

### Magnitude + Time Interval

Magnitude commands can be augmented with a speed term. This commands the toy to linearly ramp to the specified magnitude over an interval of time, given in milliseconds.

eg: `L&$$I£££`

Where `I` (or `i`) allows the effect to be ramped in over an interval of `£££` milliseconds, starting from the current value.

With this command the channel ramps to the specified level and continues at that level until given further instructions.

Interval commands are recommended for applications where a device is being used to follow a series of timed movements, such as a script.

### Magnitude + Speed

As an alternative, the rate at which the effect is ramped in can be specified.

`R&$$S£££`

Using `S` (or `s`) allows the effect to be ramped at a speed of `£££` per hundred milliseconds, ie a speed of `1000` will run one axis length in 1 second.

As with time interval, the channel ramps to the specified level and continues at that level until given further instructions.

Speed movements are most useful for applications such as control panels when single commands are issued to a device, such as for setup or telling a device to re-centre itself.

## Smooth motion

### "Ease in" and "Ease out" modifiers

Extended commands can be augmented with "Ease in" - `<` and "Ease out" - `>` modifiers. These modifiers change the linear motion into a quadratic or cubic motion by setting velocity boundary conditions:

- `<` – quadratic curve that starts at zero speed and accelerates toward the end point.

- `>` – quadratic curve that decelerates to zero speed at the end of the move.

- `<>` – cubic-spline movement with zero speed at both the beginning and the end of the move.

These commands are useful for single movements. It is recommended that Gradient commands are used for sequences of movements.

eg: `L0999I2000<>`

![Ease in and Ease out][ease in and ease out]

### Gradient modifiers

Gradient or `G` commands initiate cubic spline based movements. They obey the same rules as extended commands but introduce a starting and ending velocity to the movement, which gives it a smooth acceleration. The starting velocity is always the current velocity of the axis, the end, arrival, velocity is specified by a parameter marked by `G`.

The `G` arrival velocity number can be positive or negative and has the same units as the `S` parameter described above - a speed of `1000` will run one axis length in 1 second.

eg `L09999I2000G-1000` - linear channel 0 to magnitude 0.9999 over 2000 ms, arriving with a velocity of -1000.

For most scripts the points represent turning points where motion stops and returns in the opposite direction, so practically speaking most `G` commands will be `G0`.

eg `R08I500G0` - rotation channel 0 to magnitude 0.8 over 500 ms, arriving with a velocity of 0.

If a `G` commanded movement with a non-zero value reaches the end of its movement and no further command has been received the axis enters an automatic deceleration phase, continuing at the current velocity but decelerating and returning to rest at the last commanded point over approximately two seconds. There is never an abrupt end to a `G` movement, so it is therefore not necessary to ensure that commands overlap with previous commands. On the scale of tens of milliseconds a gap between end of movement and a new command arriving are completely unnoticeable.

![Gradients][gradients]

Gradient commands can be used in many different ways, but predominantly they will be used for script playback. If a script consists of a series of position-at-time points the recommended algorithm for constructing `G` commands is as follows:

- For a reversal point, ie the motion stops and reverses direction, set the `G` value to `0`.

- For an intermediate point, where the movement does not change direction, set the `G` value to be the average value of the straight line gradient from the previous point to the current point, and the current point to the next point.

## Multiple Channels

Multiple channels can be operated in parallel, and will do so independently of each other.

Multiple commands can be sent at the same time, separated by a space character ‘ ‘, allowing instructions to be readied for multiple `L`, `R`, `V` & `A` channels before all are executed by the receipt of a `/n` character. Lag in scripted control can be minimised by sending the next instructions in advance, sending the new line to execute at the desired moment.

Commands addressed to the same channel before a new line character is sent will overwrite previous buffered commands to that channel.

## Live Control

To be able to keep up with a live feed, commands will need to have a short, quick fire format, for an instant response.

For the fastest response possible during live control, each command should be followed immediately by a newline (`/n`).

The TCode interpreter code usually contains on-board smoothing algorithms that can correct for small amounts of signal noise and smooth over gaps in the incoming data stream. These algorithms are activated by the arrival of a stream of short commands and disabled by extended commands. It is therefore recommended that short commands are consistently used for live control.

## Device Commands

Additional instructions to and from the device can be controlled through a predetermined list of commands prefixed with the letter `D` or `d`.

Device Commands currently available:

| Command | Action                                                    |
| ------- | --------------------------------------------------------- |
| `D0`    | Identify device & firmware version                        |
| `D1`    | Identify TCode version                                    |
| `D2`    | List available axes and associated user range preferences |
| `DSTOP` | Stop device                                               |

## Save commands

To save a user's preferences to the EEPROM on the OSR2/SR6 on an axis-by-axis basis a save command can be used. These commands are prefixed by the symbol "$".

These take the form `$TX-YYYY-ZZZZ`

Where:

`T` is the axis type (`L`, `R`, `V`, `A`)

`X` is the axis number (`0`-`9`)

`YYYY` is the preferred minimum (`0000`-`9999`)

`ZZZZ` is the preferred maximum (`0000`-`9999`)

Note that saved preferences do not change the behaviour of the device itself. They exist as a reference for the driving app or plugin, accessed via the D2 command.

---

# TCode over Bluetooth
**Companion to the TCode Specification v0.4**  
**Author:** Tempest Multiaxis  
**Date:** 16 October 2026

Notes for application authors who wish to connect to a TCode device over BLE.

For TCode over Bluetooth command syntax, axis model, and device commands remain exactly as defined in the main specification. Only the transport differs.

## Service and Characteristic

Devices that implement this extension advertise the following custom service and characteristic. These identifiers were chosen so that existing web-based controllers can discover TCode devices without relying on the generic Nordic UART Service.

| Item | UUID | Properties |
|------|------|------------|
| Service | `ff1b451d-3070-4276-9c81-5dc5ea1043bc` | — |
| Characteristic | `c5f1543e-338d-47a0-8525-01e3c621359d` | READ, WRITE_NR |

- **WRITE_NR** (Write Without Response) is used for all command traffic. It avoids a per-packet ATT acknowledgement and is therefore suitable for streaming.
- **READ** allows the central to retrieve the most recent value written to the characteristic (typically the last response or status the device chose to store).
- Notifications are not required and are not part of this standard.

The service UUID must appear in the advertising packet (and preferably also in the scan response) so that Web Bluetooth and other central implementations can filter on it.

### Connection Interval Hints

Devices may advertise preferred connection interval values of `0x06`–`0x12` (approximately 7.5–15 ms). These are advisory only; the central (phone or browser) ultimately selects the interval. In practice most phone connections settle between 15 ms and 30 ms.

## Framing

TCode commands are sent as UTF-8 text exactly as they would be over serial. Multiple commands may be space-separated on a single line. A line is terminated by `\n` or `\r` (or both).

Because BLE delivers data in bursts (typically every 15–30 ms), a single characteristic write often contains one or more complete lines. 

## Short Commands over Bluetooth

Short commands (those that carry no `I`, `S`, or `G` parameter) are legal, but they have an important limitation on this transport.

On-board live-mode smoothing measures the real elapsed time between successive short commands for the same axis. When a BLE burst arrives, many short commands can be processed within a few milliseconds of one another. Intervals of that length are not meaningful to the motion planner.

**Consequently, a device implementing this extension will ignore any short command that arrives for a given axis within 5 ms of the previous short command for that same axis.**

In practical terms:

- If a single BLE burst contains several short commands for the same axis, only the first one is acted upon.
- Subsequent short commands for that axis inside the same burst are discarded.
- Commands for other axes, and any extended commands (`I`, `S`, or `G`), are unaffected.

This behaviour keeps the device stable, but it also means that a high-rate stream of short commands (for example 200–300 Hz) will lose the majority of its samples. The on-board smoothing algorithms can still produce continuous motion from the surviving points, yet the result is coarser than the same stream delivered over serial.

### Recommendation

Short commands should be used over Bluetooth only when no alternative exists.

For scripted or high-quality motion **gradient (`G`) commands are strongly preferred**. A gradient command supplies both a target position and an arrival velocity, allowing the device to construct a smooth cubic segment. Gradient commands can therefore be used on the 100-1000ms scale and produce smoother, more predictable motion over BLE than a much higher rate of short commands.

eg: `L05000I300G200 L15000I1000G-100`

---

## Summary for Implementers

1. Discover and connect to the service UUID `ff1b451d-3070-4276-9c81-5dc5ea1043bc`.
2. Write TCode lines (including the terminating newline) to the characteristic using Write Without Response.
3. Expect that short commands arriving inside the same BLE burst will be reduced to the first command per axis.
4. Prefer gradient (`G`) commands whenever the application can generate them.

<!---Images/Resources--->

[axis diagram]: ./axes.png
[interval and speed diagram]: ./intervalspeed.png
[ease in and ease out]: ./easeineaseout.png
[gradients]: ./gradients.png
