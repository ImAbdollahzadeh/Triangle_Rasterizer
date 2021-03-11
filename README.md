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

I'm not bothering with SSE acceleration of line drawing anymore. It would be left as an exercise for interested reader.

## Triangles

Okay, now it is ime to define a triangle as follows:

	typedef struct _triangle {
		point    * p0;
		point    * p1;
		point    * p2;
		rect     * bound_rect;
		COLORREF   color;
	} triangle;

There are three points to construct a tiangle. Then we draw three lines to highlight the borders of triangle. These lines are drawn with the **draw_line** function above. It is simply:

	// tr is an object of type triangle*
	draw_line(tr->p0, tr->p1);
	draw_line(tr->p1, tr->p2);
	draw_line(tr->p2, tr->p0);
	
<p align="center">
	<img src="https://github.com/ImAbdollahzadeh/Triangle_Rasterizer/blob/main/images/borders.PNG"/>
</p>

Now it is time to paint our triangle. For this, first of all, I need a macro to find the top point between points p0, p1, and p2. This macro also spits out left and rightside points (a.k.a side_point_1 and side_point_2). 

	#define FIND_TOP_POINT(TR, TPT, SPT1, SPT2) do {	\
		if(TR->p0->y < TR->p1->y)			\
		{						\
			if (TR->p0->y < TR->p2->y)		\
			{					\
				TPT  = TR->p0;			\
				SPT1 = TR->p1;			\
				SPT2 = TR->p2;			\
			}					\
			else					\
			{					\
				TPT  = TR->p2;			\
				SPT1 = TR->p1;			\
				SPT2 = TR->p0;			\
			}					\
		}						\
		else						\
		{						\
			if (TR->p1->y < TR->p2->y)		\
			{					\
				TPT  = TR->p1;			\
				SPT1 = TR->p0;			\
				SPT2 = TR->p2;			\
			}					\
			else					\
			{					\
				TPT  = TR->p2;			\
				SPT1 = TR->p1;			\
				SPT2 = TR->p0;			\
			}					\
		}						\
	} while(0)

Then I start from top point and steps through the line between this point and side_point_1. I go one pixel down, then calculate the coressponding pixel between top point and side_point_2. I have now two limit points on a scanline and all pixels in between have to be painted. I have a function called ***line_based_dword_fillup_triangle*** for this reason. It is insanely fast and efficient.

	void line_based_dword_fillup_triangle(triangle* tr)
	{
		COLORREF* framebuffer = (COLORREF*)fb;
	
		/* sort points of triangle */
		point* top_point    = NULL;
		point* side_point_1 = NULL;
		point* side_point_2 = NULL;
		FIND_TOP_POINT((tr), top_point, side_point_1, side_point_2);
	
		/* strat from top point towards side point 1 */
		float x0 = (float)(top_point->x);
		float y0 = (float)(top_point->y);
		float x  = (float)(side_point_1->x);
		float y  = (float)(side_point_1->y);
		float dx = x - x0;
		float dy = y - y0;
		float m  = dy / dx;
		float mprime = 1.0 / m;
		float opposite_m = ((float)(side_point_2->y) - y0) / ((float)(side_point_2->x) - x0);
		float gamma = (m / opposite_m);
		float next_y = y0;
		float next_x = x0;
		float opposite_x = 0;
		unsigned int this_pixel;
		unsigned int pixel_number;
		
		// the heart of the rasterizer
		while (next_y < y)
		{		
			next_x = x0 + ((next_y - y0) * mprime);
			opposite_x = (gamma * (next_x - x0)) + x0;
			this_pixel = ((unsigned int)next_x + ((unsigned int)next_y * WIDTH));
			pixel_number = ((unsigned int)opposite_x - (unsigned int)next_x);
			while (pixel_number--)
			{
				framebuffer[this_pixel] = tr->color;
				this_pixel++;
			}
			next_y++;
		}
	}

There is room for some minor optimization that I leave for readers to play with, but this function is the best to use scanline method for painting the triangle.

While we go one step in y direction down, we calculate the slope of the line between top_point and side_point_1 and thus the x component of the left side limit pixel. Based on this x component, we therefore calcualte the x component of the limit pixel falling on the line between top_point and side_point_2. Then fillup this scanline.

<p align="center">
	<img src="https://github.com/ImAbdollahzadeh/Triangle_Rasterizer/blob/main/images/filled_in_c.PNG"/>
</p>

The heart of the rasterizer is where we can work on to accelerate the blitting. For this reason I use x86-SSE optimization. If we concentrate on the way both limit points defined, we come up with this fact that, it cannot use packed floating point instructions since a line pixel can be anywhere on screen and any attampt to make it 16-byte aligned, would be a speed killer, and better to stick with the un-aligned instructions. Also I used inline assembly and not a separate .asm file (to tolerate the overhead of function epilogue and prologue).

	void accelerated_line_based_xmmword_fillup_triangle(triangle* tr)
	{
		/* sort points of triangle */
		point* top_point    = NULL;
		point* side_point_1 = NULL;
		point* side_point_2 = NULL;
		FIND_TOP_POINT((tr), top_point, side_point_1, side_point_2);
	
		/* construct super color */
		__declspec(align(16)) COLORREF _super_color[4] = { tr->color, tr->color , tr->color , tr->color };
	
		/* strat from top point towards side point 1 */
		float x0         = (float)(top_point->x);
		float y0         = (float)(top_point->y);
		float x          = (float)(side_point_1->x);
		float y          = (float)(side_point_1->y);
		float dx         = x - x0;
		float dy         = y - y0;
		float m          = dy / dx;
		float mprime     = 1.0 / m;
		float opposite_m = ((float)(side_point_2->x) - x0) / ((float)(side_point_2->y) - y0);
		float gamma      = (m * opposite_m);
		float next_y     = y0;
		float next_x     = x0;
		float opposite_x = 0;
		unsigned int this_pixel;
		unsigned int pixel_number;
	
		while (next_y < y)
		{
			next_x       = x0 + ((next_y - y0) * mprime);
			opposite_x   = (gamma * (next_x - x0)) + x0;
			this_pixel   = ((unsigned int)next_x + ((unsigned int)next_y * WIDTH));
			pixel_number = ((unsigned int)opposite_x - (unsigned int)next_x);
	
	
			//--------------------------- SIMD OPTIMIZED LINE RASTERIZER -------------------------------
			/* 16 byte */
			unsigned int   total_bytes_in_line                  = pixel_number << 2;
			unsigned int   xmmwords_number                      = total_bytes_in_line >> 4;
			unsigned int   _16_bytes_blocks_in_line             = (xmmwords_number << 4);
			unsigned char* start_of_line_in_framebuffer         = fb + (this_pixel << 2);
			unsigned int*  start_of_final_dwords_in_framebuffer = NULL;
			__asm {
					mov    edi, start_of_line_in_framebuffer
					mov    ecx, xmmwords_number
				// sanity check
					cmp    ecx, 0
					je     _accelerated_blit_end_
					movaps  xmm0, _super_color
				_accelerated_blit_:
					movups xmmword ptr[edi], xmm0
					sub    ecx, 1
					add    edi, 16
					cmp    ecx, 0
					je     _accelerated_blit_end_
					jmp    _accelerated_blit_
				_accelerated_blit_end_:
					mov    start_of_final_dwords_in_framebuffer, edi
			}
			
			/* 4 byte */
			unsigned int dwords = (total_bytes_in_line - _16_bytes_blocks_in_line) >> 2;
			while (dwords--)
				*start_of_final_dwords_in_framebuffer++ = tr->color;
			//------------------------------------------------------------------------------------------
	
			next_y++;
		}
	}

This is, in my opinion, the fastest way to paint a triangle based on the scanline method with SIMD acceleration. Of course some very little optimizations can be done, but they are left for readers to play with.

In function ***accelerated_line_based_xmmword_fillup_triangle***, all the calculations are similar to those of ***line_based_dword_fillup_triangle***, but instead of DWORD blit, I separated blocks of XMMWORDs and used 16-byte blit and finally some DWORD blit at the very end. 

The benchmarking made it clear that ***accelerated_line_based_xmmword_fillup_triangle*** is simply 5.5 times faster than ***line_based_dword_fillup_triangle***. A single core CPU spent nearly ***0.43 ms*** to paint the represented triangle above with ***line_based_dword_fillup_triangle*** and the same with ***0.08 ms*** in ***accelerated_line_based_xmmword_fillup_triangle***.
