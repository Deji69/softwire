# Etymology #

SoftWire is a word play on 'hard wire', which refers to a piece of hardware which has been wired to perform a single function. Hard-wired logic is very fast for what it is specialized at for, but useless for anything else. SoftWire attempts to combine the advantages of specialized functions while retaining the flexibility of software. It allows to generate highly specialized code specifically for the task at hand, and generate new ones with different functionality or specialization when circumstances change.

"The fastest instruction is the one that is never executed" - Michael Abrash

# Origin #

SoftWire was created for fast 3D rendering on the CPU. Graphics processing units (GPUs) consist of various hard-wired components (but have been becoming much more programmable) which form a configurable pipeline. The number of combinations for these configurations run into the billions, which makes it impossible to write specialized code to implement all this functionality on the CPU in an optimal manner. However, applications only make use of a limited number of pipeline configurations, and thus we can generate just these few specialized functions to perform exactly the expected functionality of the graphics pipeline.

# Details #

  * [Tutorial](Tutorial.md)

  * [Optimization](Optimization.md)