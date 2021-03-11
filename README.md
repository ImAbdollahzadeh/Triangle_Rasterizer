# Triangle_Rasterizer
An overview on triangle drawing in C/C++ and acceleration with x86-SSE

# Prerequisites
- Knowledge of line equation and line drawing.
- Experinece with rasterization and concept on blit
- Good knowledge of x86 assembly and C programming languages

# Optimization
- SIMD (x86-SSE) optimization 

# Let's get started.
## Lines
A line can be formulated as ***Y = Y<sub>0</sub> + m(X - X<sub>0</sub>)***. Imagine a 2D coordinates system. A line is constructed between two points. Each point has its own **X** and **Y** components. For point P, the components are X and Y, while for point P<sub>0</sub>, these values are X<sub>0</sub> and X<sub>0</sub>. The slope of the line is given by **m**. If we rearrange the formula above, we can simply extract the value of ***m*** as being: 

m = (Y - Y<sub>0</sub>) / (X - X<sub>0</sub>)

Therefore, having two points, we can determine ***m*** and with this slope, we would be able to calculate other points (other XY pairs) falling on our line.

## Line drawing in C
Points can be given as ***"unsigned int"*** or ***"float"*** values. It totally depends on how we would like to interpret our coordinate system on our display.

Generally, for better precision, one uses floating point values and at the very last moment of drawing on screen, cast floating point values to integer values. We will follow this strategy to draw our lines.

With the following function, we are going to draw a line between two points.

    void draw_line(point* p0, point* p1) 
    {
		int           dx, dy, step_dx, step_dy, pixel_x, pixel_y, abs_dx, abs_dy, index = 0;
		float         m;
		float         x0          = (float)(p0->x);
		float         y0          = (float)(p0->y);
		float         x           = (float)(p1->x);
		float         y           = (float)(p1->y);
		unsigned int  color       = 0x00FFFF00; // ARGB
		unsigned int* framebuffer = (unsigned int*)fb;

		dx      = x - x0;
		dy      = y - y0;
		abs_dx  = ABS(dx); // absolute value of dx
		abs_dy  = ABS(dy); // absolute value of dy

		if (abs_dx >= abs_dy)
		{
			step_dx = sgn(dx); // +1 or -1
			m = (float)dy / (float)dx;
			while(index != dx)
			{
				pixel_x = index + x0;
				pixel_y = m * index + y0;
				framebuffer[pixel_x + WIDTH * pixel_y] = color;
				index += step_dx;
			}
		}
		else
		{
			step_dy = sgn(dy); // +1 or -1
			m = (float)dx / (float)dy;
			while (index != dy)
			{
				pixel_y = index + y0;
				pixel_x = m * index + x0;
				framebuffer[pixel_x + WIDTH * pixel_y] = color;
				index += step_dy;
			}
		}
	}

Let's break it down. 

A line in 2D coordinate system, has an angle greater (smaller) than ±45° with x axis. If this angle is below ±45°, our line is more horizontal and in case this angle is above ±45°, our line is more vertical. Having a more horizontal line, the x component of the adjacent pixel is either one pixel on right or left. The coressponding y component of this pixel must be calculated. For a more vertical line, on the other and, the y component of the adjacent pixel is either one pixel right or left. Then the coressponding x component of this pixel must be calculated. If we look at the function above, we see this fact by a conditional statement to follow the direction of the line.

*Note that we draw our line in wholepixel regime and not in subpixel regime. For subpixel regime explanation, please have a look at my True-open-free-Type repository.*
