---
layout: post
author: simon
---
## Summary

**Title:** Walter Benjamin Meets Steve Reich: a Minimalist Digital Logic Implementation of “Clapping Music”  
**Years:** 2015-2016  
**Materials:** 7400-series digital logic, analog clock, lever solenoids

---
## Abstracts
### Musical Abstract
Steve Reich’s 1972 work, “Clapping Music,” is a minimalist percussion work for two clapping individuals.

The fundamental unit of the piece is a single, 12:8 bar:
![](public/images/clapping-music/basic-bar.png)

When the piece starts, both parts clap the measure together, repeating it eight times. When the ninth measure is reached, a new cycle begins: the second part rotates its pattern to the left to form a new one; the second beat in the pattern bar becomes the first beat; and the first beat is wrapped around to become the last beat. The first part continues to play the original pattern against this new phase.

In this cycle, the two parts together are notated like this:
![](public/images/clapping-music/both-parts.png)

After another eight bars, the second part applies the same shift operation to its pattern, while the first part again plays the original pattern.

This cycle continues until the parts are once again aligned, after twelve cycles; the piece concludes at the end of this thirteenth cycle.

#### Musical Performance
A youtube video of a recording can be found here (note: as is done in the video, the piece can be played with six repeats of the measure pattern in a cycle, rather than 8. This circuit was timed to Pierre-Laurent Aimard's recording in "African Rhythms"):

<div class="youtube-container">
<iframe src="https://www.youtube.com/embed/liYkRarIDfo?rel=0" 
frameborder="0" allowfullscreen class="video"></iframe>
</div>

### Technical Abstract
A 555 clock, operating at twice the frequency of the piece’s eighth note, generates a downbeat for eighth note. The measure pattern is represented as the binary pattern, “111 011 010 110,” with one bit per clock downbeat. Each part’s state is held in a 12-bit shift register configured to loop its highest bit back as its serial input; on each downbeat, the shift register rotates by one cell, and a tap bit on the serial input saves the value into a 1-bit cell that drives the solenoid for each beat. The pattern’s position in each shift register forms a part’s current position in the measure, and the measure rotates around a cursor. (A more musical conception might use a cursor that moves through memory addresses and selects the current beat.) On boot, the part registers are fed serially by a 12-bit parallel-load shift register; this boot register is wired to load the measure pattern from power and ground ties on reset.

An 8-bit counter holds the circuit’s position in each cycle. At the end of each cycle, when the measure counter reads eight bars, for 96 beats, a 4-bit cycle counter is incremented, and the shift register for the second, rotating part is clocked an additional time. When the cycle counter reads 13, the circuit halts.

### Circuitry Performance

<div class="youtube-container">
<iframe src="https://www.youtube.com/embed/jw6vcDOxskQ?rel=0" 
frameborder="0" allowfullscreen class="video"></iframe>
</div>
  
  
The lights-only version can be easier to watch:

<div class="youtube-container">
<iframe src="https://www.youtube.com/embed/4rYDtvjl7-0?rel=0" 
frameborder="0" allowfullscreen class="video"></iframe>
</div>

---
## Resources and Documentation
### Early Version Full Notes
Complete notes on an earlier version can be found [here](https://github.com/simzes/clapping-music-circuit/blob/master/docs/prototype_notes.pdf). This version was not implemented in full, and contains a number of excess states and circuit variables. As an example, the “load” state was removed after it was realized that the reset → normal transition could be seamless and stateless if the parts’ tap bits were moved to the serial input/high bit.

## Servo Prototype
A prototype was first developed with an arduino, driving servo motors. A video of this prototype, striking drinking cups with taped-on pencils, is below:

<div class="youtube-container">
<iframe src="https://www.youtube.com/embed/_mvR-xaQNZ4?rel=0"
frameborder="0" allowfullscreen class="video"></iframe>
</div>

The arduino code for this is
[here](https://raw.githubusercontent.com/simzes/clapping-music-circuit/master/clapping_music_servo/clapping_music_servo.ino).

### Kicad Schematic
A kicad schematic for this project, hosted in github, can be found here:
<https://github.com/simzes/clapping-music-circuit>

Note that the 74xx-eu.lib library, as accessed during the project, contained an error for the JK flip-flop pinout declaration. A fixed version has been included.

### Layouts
A PDF and PNG schematic can be found
[here](https://github.com/simzes/clapping-music-circuit/blob/master/printouts/clapping-music.pdf)
and
[here](https://raw.githubusercontent.com/simzes/clapping-music-circuit/master/printouts/clapping-music.png).
A floorplan of a constructed circuit can be found
[here](https://raw.githubusercontent.com/simzes/clapping-music-circuit/master/printouts/floorplan-annot.png).

---
## Full Notes
A 555 timer generates a clock pulse at twice the frequency of the piece’s eight-note; each eighth-note beat in the piece has a downbeat and upbeat pair, CLK_0 and CLK_1.

A JK flip-flop further divides the clock into a downbeat-and, and an upbeat-and; the JK toggles between high during the downbeat, and low during the upbeat; the output is AND-ed with an inverted clock line to produce these two sub-beats, CLK_0_bar and CLK_1_bar.

On boot, a 12-bit parallel-in, serial-out shift register pulls the binary pattern “111 011 010 110” into memory from power and ground rail ties. Then, with each downbeat clock signal, the highest bit of the remaining pattern is strobed into two 12-bit serial-in, parallel-out shift registers; these hold the bit pattern for parts one and two. The boot PISO register is followed, or flushed, with 0 as its serial input. (This PISO register can function as serial-in, serial-out when not in parallel-load mode.)

On each clock downbeat (CLK_0), the lowest bit in each part’s register is saved into a flip-flop cell, and the registers for both parts are advanced. The highest bit in each part’s register is fed back into its serial input; a part register’s serial input is defined as the OR of the register’s highest bit and the highest bit of the boot register. Because the boot register is flushed with a 0, once the pattern is loaded, it is cycled and preserved.

The value of each flip-flop cell holds the 1/0 value of a part for each downbeat clock cycle. Its output switches a lever solenoid for each part; CLK_0 turned out to be a decent duration for switching the solenoids; the switch control line is defined as the AND of the player bit and CLK_0. Previous versions had used a 555 for generating a single pulse for each solenoid “clap.”

An 8-bit and a 4-bit counter hold the circuit’s position in the 8 measures of each cycle (96 beats in total), and in the 13 cycles of the piece. On each clock downbeat, the measure counter is incremented. When its value equals 96: the second part’s register is shifted once on the upbeat, implementing the phase shift for the next cycle; the cycle counter advances by one; and the measure counter is cleared. When the cycle counter value equals 13, the DONE signal is stored, and the buffer for the clock lines is switched off. The circuit halts as a consequence.

On reset, the done cell, measure counter, cycle counter, and part registers are cleared. 

The reset logic converts an input signal, generated from a button press, into a logical phase synchronized with the main clock. A 555 chip is used to debounce the reset button; for each distinct press, its output is a single pulse that is longer than any downbeat-to-downbeat span. On CLK_1, the 555 output ("reset store") is saved into a flip-flip. The lines "R" and "R_bar" are run into a buffer, and passed as the reset signals to the measure and cycle counters, the load register, and the parts registers, depending on an IC's active high or active low function. Once the reset store signal goes low, the reset flip-flop will be cleared on the first rising CLK_1 edge, and the circuit will begin playing at CLK_0.

Note: an earlier mention of the clock signals mentioned that when the "done" state is reached at the conclusion of the piece, the signals are disabled with the buffer's output enable function. To avoid this lockup, the reset logic uses the CLK signal being passed into the buffer.

### Signal Definitions
Clock Signals:

| CLK   | output of the 555 timer; generates a square wave at twice the eighth note frequency  |
| ----- | ------------------------------------------------------------------------------------ |
| CLK_0 | downbeat clock signal; every first of the CLK pair (CLK_0 = AND(CLK, CLK_TOGGLE))    |
| CLK_1 | upbeat clock signal; every second of the CLK pair (CLK_1 = AND(CLK, CLK_TOGGLE_BAR)) |

Reset signals:

| R     | reset signal, synchronized with CLK_1                   |
| ----- | ------------------------------------------------------- |
| R_B   | reset signal inversion, for active low logic            |
| R_STR | reset signal from the 555 pulse, before synchronization |

Load and Part Register Signals:

| LD_b    | load bit; leading bit of the piece’s pattern, as fed from the boot register                                    |
| ------- | -------------------------------------------------------------------------------------------------------------- |
| L_b_OUT | left part’s shift register high bit                                                                            |
| L_b_STR | left part serial input: OR(LD_b, L_b_OUT). L_b_STR is tapped and stored for driving the left part’s solenoid   |
| R_b_OUT | right part’s shift register high bit                                                                           |
| R_b_STR | right part serial input: OR(LD_b, R_b_OUT). R_b_Str is tapped and stored for driving the right part’s solenoid |

Measure Counter and Shift Signals:

| SHFT_STR | cycle shift signal. Follows AND(M_5, M_6), where M_x is bit x of the
measure counter                                                                                                                                                                      |
| -------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| SHFT     | cycle shift signal, but synchronized with the clock; SHFT_STR is latched into a flip-flop
on CLK                                                                                                                                                          |
| CYC_R    | cycle reset; AND(SHFT, CLK_1)                                                                                                                                                                                                                             |
| L_CLK    | clock for driving the left part’s shift register: XNOR(CLK_0, CYC_R)                                                                                                                                                                                      |
| R_CLK    | clock for driving the right part’s shift register: CLK_0, passed through                                                                                                                                                                                  |
| M_R      | measure reset; NOR(CYC_R, R). Once SHFT_STR is high, on the 96th beat (of a cycle), it is latched into the SHFT flip-flop. CYC_R will then be high for CLK_1, resetting the measure counter. SHFT will be relatched as low on the next CLK signal (CLK_0) |

Cycle and Done Signals:

| DONE_STR | done signal, for counting cycles and ending the performance. Equal to: XNOR(OR(C_2_B, C_3_B), C_0_B), where C_x is bit x of the cycle counter |
| -------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| DONE     | done signal, but latched with CLK                                                                                                             |
| CYC_R    | defined above; used as the cycle counter’s increment clock                                                                                    |

**Parts List**
A parts list for most of the components in the circuit can be found below:

| **Parts**                   | Count |
| --------------------------- | ----- |
| PISO Shifter, 12 bit        | 1     |
| SIPO Shifter, 8 bit         | 2     |
| SIPO Shifter, 4 bit         | 2     |
| 555 Timer                   | 2     |
| 8 Bit Adder/Counter         | 1     |
| 4 Bit Adder/Counter         | 1     |
| JK Flip Flop                | 4     |
| 2in AND                     | 9     |
| 4in AND                     | 2     |
| 1in Inverter                | 4     |
| 2in NOR                     | 1     |
| 2in OR                      | 5     |
| 4 in NOR                    | 1     |
| NPN Transistor              | 4     |
| Main Clock Capacitor: 2.2uF | 1     |
| Main Clock R1: 10K          | 1     |
| Main Clock R2: 20-22K       | 1     |
| Reset Pulse Capacitor: 1 uF | 1     |
| Reset Pulse R: 100K         | 1     |
| MOSFET (N-channel)          | 2     |
