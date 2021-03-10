# Triangle_Rasterizer
An overview on triangle drawing in C/C++ and acceleration with x86-SSE

# Aims
1. Indroducing the concept of Bezier curve
2. Design of True (open, free) Type glyphs based on curves and lines
3. Rendering glyphs in subpixel regime
4. Implementation of smoothing low pass filters

# Prerequisites
- Knowledge of line equation and line drawing.
- Experinece with rasterization and concept on blit
- Good knowledge of x86 assembly and C programming languages

# Optimization
- SIMD (x86-SSE) optimization 

# Let's get started.
## Lines
A line can be formulated with as ***Y = Y<sub>0</sub> + m(X - X<sub>0</sub>)***. Imagine a 2D coordinates system. A line is constructed between two points. Each point has its own **X** and **Y** components. For point P, the components are X and Y, while for point P<sub>0</sub>, these values are Y<sub>0</sub> and X<sub>0</sub>. The slope of the line is given by **m**. If we rearrange theformula above, we can simply extract the value of m as being: 

m = Y - Y<sub>0</sub> / X - X<sub>0</sub>

Therefore, having two points, we can determine the m and with this slope, we are able to calculate other points (other XY pairs) on this line.
