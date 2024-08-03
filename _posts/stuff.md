




## Resources/Appendix
- physical
- circuit
- driver program
- analysis program
- raw data
- graphs

**Electro-Mechanical System**
The mechanical system begins with a rectangular aluminum extrusion frame.

Inside the frame, a T-shaped boom mounts across its width, and can slide back and forth; on either end of the T, two sliders bolt onto the boom and fit into the frame’s trapezoidal slots. These lock it into the frame. On the other end, a flat slider holds the other end up.

On the bottom of T-boom, an LED and its heat sink mount, using thermal-transfer adhesive. On either side of the T-boom, two extrusion nuts mount a lens holder. Lens holders are designed and printed for each lens, based on the diameter, focal length, and retaining ring.

A drive belt attaches to either end of the T-boom; one belt loop passes through an idler, and the other loop through a geared stepper shaft. The stepper supports 1600 steps per rotation; with a 20-tooth shaft and a 3mm belt pitch, there are 80 steps per 3mm. To move the T-boom at 1mm steps (most of the measurements were at 1mm steps)

1mm/26.6 → .0375mm per step
26 * .0375 → 0.975
27 * .0275 → 1.0125

The sensor lead screw has a 1mm pitch, minimizing variations to the manufacturing tolerance of the screw itself. The light belt has a 3mm pitch with a 1600-position stepper motor; at a 1mm step, the stepper must alternate between 26 and 27 steps, making the actual position within .025mm of its goal.

The frame hovers above a table, supported by 4 adjustable-height stilts. These unlock and adjust to measure different proposed exposure unit heights.

On the table, the UV intensity sensor is mounted on a brass plate sitting a track. A stepper turns a lead screw, moving the sensor plate back and forth using an attached nut. The lead screw is a 6mm x 1.0 pitch screw, making for one rotation per 1mm of movement.

**Embedded Controls System**

******Analysis Program**