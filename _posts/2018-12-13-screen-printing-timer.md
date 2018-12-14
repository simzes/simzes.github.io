---
layout: post
author: simon
title: Screen Printing Exposure Unit
caption: A small program and circuit for powering lights for screen burning.
---
A timer program and circuit for controlling an AC relay with user input. This is intended for use in a screen printing photo exposure unit, with the facilities:
 * the set time is modified with buttons for up, down, and
  run/pause. The up and down buttons adjust the time by one second
  with a single press, or by ten seconds with held presses
 * the set time or run time is displayed with a 4-digit, seven-segment
   display
 * a piezo buzzer emits noises that correspond to events
 * the circuit and code support a "lid" switch, and a "lid override"
   switch, for shielding users from UV and other hazardous
   light-types.

The code, schematic, and print-outs are here:
<https://github.com/simzes/screen-printing-timer>

---
## Circuit
The circuit uses an arduino (328) to read user and system inputs from buttons and switches, display the set or run time, and switch the AC relay:<sup>[1](#myfootnote1)</sup>

![circuit diagram](https://raw.githubusercontent.com/simzes/screen-printing-timer/timer/media/screen_printing_timer.png)

### Run time display

The circuit as-is supports a common-anode display, where a single
digit is selected and powered by a one pin, and a segment is selected
and drained by one pin. Four pins are dedicated to powering
digits. Segment-selection is delegated to an 8-bit shift register:
this register is loaded with a clock and data line, and its output
sources a digit's drain by switching an npn transistor.

Display support is modular for both other annode-type displays, and for other types entirely; support for other anode-type displays can be added by changing the digit codes. Support for other displays can be added by overwriting the initialization and display time functions.

### Buttons and switches, and a debouncing algorithm

Internally, all buttons and switches are implemented by using the 328's input-pullup facility; a button is pressed, and a switch is tripped, by connecting a pin to ground.

Without debouncing, this configuration handles switches that change states infrequently--order 100ms or more. The lid and lid override are handled in this way.

All user-input buttons are mediated with a program-provided, configurable debouncing algorithm. For a button to enter a pressed or depressed state, the program must accumulate some _x_ observations of this state, with each observation sampled at an epoch of some fixed, _y_ milliseconds. Once a new state is entered, a transition event is issued and its timestamp is recorded; this transition corresponds to a single press. A button is held when a press state is entered, not left, and its press timestamp exceeds a certain threshold.<sup>[2](#myfootnote2)</sup>

## Program

The code is organized around three tasks: user inputs, display handling, and the program state.

### User inputs

User inputs are handled with a polling routine. At the start of each program
loop, a check is run to see if enough time has passed for another observation to
take place (see the debouncing description, above), and to take an observation
if this is the case. The loop always runs far quicker than is needed between observations.

With each observation, the current state of all pins tied to user input is read
in. After comparing the current pin state to the previously read state and the
internally held state, the program's debounced view is updated to notate press
events and press state. (State is how the button is, after debouncing; events
are held until they are consumed.)

### Display handling

The 7-segment display is driven by an interrupt timer. With some frequency, a routine runs to set the state of the digits so that they match the run or set time written by the program. The time itself is written into a shared array, on a per-digit basis.

The interrupt services one digit per run (because the drain for common-annode segments is shared across digits, multiple-digit displays need to energize one digit at a time). In a single run, the interrupt powers down the select pin for the last digit, sets the pattern in the shift register, and powers up the select pin for the next digit.

### Program state

The program is modelled as a series of states, where user actions and the timer state drive changes between states.

The program starts waiting for user input. Presses or holds of the up or down buttons modify the set time, and issue beeps and changes to the display time. When run/pause is held, the program checks that the lid is closed (unless the lid override is set), turns on the AC relay, and begins counting down. Once zero, pause, or panic is detected, it turns off the relay.

## Usage

### Set and run times

The "set time" is the duration over which the program will enable the AC relay. The program has a baked-in, default set time that is taken up on boot. Subsequent changes to the set time overwrite this value until the next power loss. The set time is changed by pressing or holding the up or down buttons.

The "run time" is the current duration of time left, when the timer is running or paused. The timer is started by holding down run/pause for a few seconds. The program can be paused with an additional press of run/pause, or by lifting the lid. Pauses or lid trips during the run do not change the set time, unless the run time is modified in these modes.

### Lid trip and override

The circuit has outlets for two switches, for governing the lid and a
lid override. These can be left unwired, and in this configuration
will have no effect on the circuit and program's operation.

To use the lid-trip facility, one or more normally-open switches can
be wired, in parallel, into the lid outlet. When the lid is tripped,
the AC relay is cut, the timer is paused, and the program enters a
"panic" state that requires acknowledgement to exit (one press of the
run/pause button).

The lid override is a normally-open switch that, when closed, tells
the program to ignore tripping of the lid. (This is intended to
support situations where the screen's frame is larger than the
interior space, and the lid cannot be closed, but the image area still
fits within the exposure area.)

---
## Footnotes

<a name="myfootnote1">1</a>: kicad schematic for this circuit: <https://github.com/simzes/screen-printing-timer/tree/timer/circuit>

<a name="myfootnote2">2</a>: a description of<http://www.ganssle.com/debouncing.htm>
