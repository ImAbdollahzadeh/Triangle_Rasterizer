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

Generally, for better precision, one uses floating point values and at the very last moment of drawing on screen, cast floating point values to unsigned int values. We will follow this strategy to draw our lines.

With the folloeing function, we are going to draw a line between two points.

    void draw_line(point* p0, point* p1) 
    {
	unsigned int* framebuffer = (unsigned int*)fb;
	float x0                  = (float)(p0->x);
	float y0                  = (float)(p0->y);
	float x                   = (float)(p1->x);
	float y                   = (float)(p1->y);
	float dx                  = x - x0;
	float dy                  = y - y0;
	float m                   = dy / dx;
	float mprime              = 1.0 / m;
	float next_y              = y0;
	float next_x              = x0;
	unsigned int this_pixel   = 0;
	unsigned int pixel_number = 0;
      
	/* rasterize as scanline method */
	while (next_y < y)
	{
		next_x       = x0 + ((next_y - y0) * mprime);
		this_pixel   = ((unsigned int)next_x + ((unsigned int)next_y * WIDTH));
		pixel_number = ((unsigned int)next_x - (unsigned int)x0);
		while (pixel_number--)
		{
		    framebuffer[this_pixel] = tr->color;
		    this_pixel++;
		}
		next_y++;
	    }
    }
