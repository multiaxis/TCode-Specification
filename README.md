# tcode-spec

<!--- credit to Tempest & the community. --->

## version `0.3` as of 10th May 2021

T-code is a protocol for implementing UART serial communications to an adult toy. It is partly influenced by G-code, which is an alphanumeric format used to drive CNC machines, including 3D-printers.

The toy or “device” will be capable of one or more functions, which are described as Linear motion, Vibration or Rotation “channels”, which can be addressed independently.

Commands are sent to the device in the form of alphanumeric ASCII phrases, which are interpreted by the device, stored in a buffer and executed on the receipt of new line or `/n` character.

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

Basic commands for **L**inear move, **R**otate, and **V**ibrate take the form of a letter (`L`, `R`, or `V`) followed by a set of digits. Lower case letters (`l`, `r`, `v`) are also valid.

### Example Basic Commands

| Command   | Axis     | Channel ID | Magnitude |
| --------- | -------- | :--------: | --------- |
| `L277`    | Linear   |    `2`     | `0.77`    |
| `R09`     | Rotation |    `0`     | `0.90`    |
| `V317439` | Vibrate  |    `3`     | `0.17439` |

### Anatomy of a basic command: `X&$$`

The first digit `&` represents the channel ID (e.g. `0`-`9`).

Any subsequent digits (in this case `$$`) are the magnitude.

The magnitude for any channel is always a number greater than or equal to `0.0` and strictly less than `1.0`. The magnitude digits are therefore effectively the digits following a `0.____`.

The magnitude corresponds to a linear position/speed, rotation position/speed, or vibration level. In situations where position or speed is possible in a forward or reverse direction 0.5 is assumed to be central or at rest.

## Advanced Commands

Magnitude commands of the format `X&$$` can be augmented with additional terms to change the ramp time or speed.

| Command     | Axis    | Channel | Effect                                  |
| ----------- | ------- | :-----: | --------------------------------------- |
| `V199I2000` | Vibrate |   `1`   | Ramp to `0.99` over `2000` milliseconds |
| `L020S10`   | Linear  |   `0`   | Ramp to `0.2` at a rate of `0.1`/sec    |

### Magnitude + Time Interval

Magnitude commands can be augmented with a speed term. This commands the toy to ramp to the specified magnitude over an interval of time, given in milliseconds.

eg: `L&$$I£££`

Where `I` (or `i`) allows the effect to be ramped in over an interval of `£££` milliseconds, starting from the current value.

With this command the channel ramps to the specified level and continues at that level until given further instructions.

### Magnitude + Speed

As an alternative, the rate at which the effect is ramped in can be specified.

`R&$$S£££`

Using `S` (or `s`) allows the effect to be ramped at a speed of `£££` per hundred milliseconds.

As with time interval, the channel ramps to the specified level and continues at that level until given further instructions.

## Multiple Channels

Multiple channels can be operated in parallel, and will do so independently of each other.

Multiple commands can be sent at the same time, separated by a space character ‘ ‘, allowing instructions to be readied for multiple `L`, `R` & `V` channels before all are executed by the receipt of a `/n` character. Lag in scripted control can be minimised by sending the next instructions in advance, sending the new line to execute at the desired moment.

Commands addressed to the same channel before a new line character is sent will overwrite previous buffered commands to that channel.

## Live Control

To be able to keep up with a live feed, commands will need to have a short, quick fire format, for an instant response.

For the fastest response possible during live control, each command should be followed immediately by a newline (`/n`).

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


<!---Images/Resources--->

[axis diagram]: ./axes.png
