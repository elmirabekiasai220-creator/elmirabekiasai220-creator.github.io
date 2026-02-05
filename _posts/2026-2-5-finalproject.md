---
layout: posts
title: Creating a Mandelbrot fractal animation
---
# Project Report: Creating a Mandelbrot Fractal Animation

## Project Introduction
This project involves creating a program to produce animations of the Mandelbrot set. The project consists of two main parts: generating visual frames using C language and creating background music using Sonic Pi.

## Part 1: Generating Fractal Images in Python

### Initial Version (First Code)
```python
import matplotlib.pyplot as plt

W, H, I = 600, 400, 100

xmin, xmax = -2.5, 1.0
ymin, ymax = -1.25, 1.25

img = [[0] * W for _ in range(H)]

for py in range(H):
    y = ymin + (ymax - ymin) * py / (H - 1)
    for px in range(W):
        x = xmin + (xmax - xmin) * px / (W - 1)
        c = complex(x, y)
        z = 0j
        n = 0
        while (z.real * z.real + z.imag * z.imag) <= 4.0 and n < I:
            z = z * z + c
            n += 1
        img[py][px] = n

plt.imshow(img, cmap='hsv')
plt.axis("off")
plt.show()
```

**Explanation:**
- `W, H, I = 600, 400, 100`: Defining width, height, and maximum iterations
- `xmin, xmax, ymin, ymax`: Display range for the Mandelbrot set
- Nested loops for pixels: each pixel corresponds to a point in the complex plane
- Iterative calculation of the Mandelbrot formula: `z = z² + c`
- Storing iteration counts in the `img` array
- Display with `hsv` color mapping

### Second Version (Adding Custom Color Mapping)
```python
def hsv_to_rgb(h, s, v):
    # Function to convert color from HSV to RGB space
    ...

colors = []
for i in range(256):
    if i == 0 or i==255:
        colors.append((0, 0, 0, 1))  # Black color for points inside the set
    else:
        hue = (272 - (i * 180 / 256)) % 360 
        r, g, b = hsv_to_rgb(hue, 0.96, 0.59)
        colors.append((r, g, b, 1))

plt.imshow(img, cmap=plt.cm.colors.ListedColormap(colors))
```

**Changes:**
- Added `hsv_to_rgb` function for color conversion
- Created a custom color palette with purple-blue spectrum
- Points inside the set (iteration=255) are displayed in black

### Third Version (45-Degree Rotation)
```python
u = (px - W/2) / (W/2.5)
v = (py - H/2) / (H/2.5)
x = u * 0.707 - v * 0.707  # cos(45) = sin(45) ≈ 0.707
y = u * 0.707 + v * 0.707
```

**Changes:**
- Changed coordinate system for 45-degree rotation
- Used rotation matrix to create a diagonal view of the fractal

## Part 2: C Program for Animation Production

### File Structure

#### 1. File `defs.h` - Main Program Structures
```c
typedef struct _image_state
{
    double cx, cy;               // Image center
    double minx, maxx, miny, maxy; // Coordinate range
    double angle;                // Rotation angle
    int height, width;           // Image dimensions
    int image_count;             // Current frame number
    double hue_color;            // Current color shift
    double CR;                   // Real part of Julia constant
    double CI;                   // Imaginary part of Julia constant
    int top_rotation;            // Is rotation around top point?
    double top_rotation_angle;   // Top point rotation angle
    BitMapFile bmFileData;       // BMP image data
} ImageState;
```

**Explanation:** This structure maintains the complete state of the image at each moment of the animation.

#### 2. File `mandelbrotset.h` - Main Computational Functions

**Mandelbrot Calculation Function:**
```c
int get_mbs_iter(double cr, double ci, int max_iter)
{
    double zr = 0, zi = 0;
    int n = 0;
    
    while (n < max_iter) {
        double m = zr * zr + zi * zi;
        if (m > 4) {
            return n;
        }
        
        double zr_new = zr * zr - zi * zi + cr;
        double zi_new = 2 * zr * zi + ci;
        zr = zr_new;
        zi = zi_new;
        n++;
    }
    
    return max_iter;
}
```
**Function:** For each point `(cr, ci)` in the complex plane, it calculates the number of iterations of the formula `z = z² + c` until `|z| > 2` or maximum iterations are reached.

**Julia Set Calculation Function:**
```c
int get_julia_iter(double zr, double zi, double cr, double ci, int max_iter)
{
    int n = 0;
    
    while (n < max_iter) {
        double m = zr * zr + zi * zi;
        if (m > 4) {
            return n;
        }
        
        double zr_new = zr * zr - zi * zi + cr;
        double zi_new = 2 * zr * zi + ci;
        zr = zr_new;
        zi = zi_new;
        n++;
    }
    
    return max_iter;
}
```
**Difference from Mandelbrot:** In the Julia set, `c` is constant and `z` is the starting variable. This creates different patterns.

**Color Conversion Function:**
```c
void hsv_to_rgb(int hue, int saturation, int value, COLORINDEX *p)
{
    double h = hue / 360.0;
    double s = saturation / 100.0;
    double v = value / 100.0;
    
    double r, g, b;
    
    double hh = h * 6.0;
    int i = (int)hh;
    double f = hh - i;
    double p_val = v * (1 - s);
    double q = v * (1 - f * s);
    double t = v * (1 - (1 - f) * s);
    
    if (i == 0) { r = v; g = t; b = p_val; }
    else if (i == 1) { r = q; g = v; b = p_val; }
    else if (i == 2) { r = p_val; g = v; b = t; }
    else if (i == 3) { r = p_val; g = q; b = v; }
    else if (i == 4) { r = t; g = p_val; b = v; }
    else { r = v; g = p_val; b = q; }
    
    p->r = (unsigned char)(r * 255.0);
    p->g = (unsigned char)(g * 255.0);
    p->b = (unsigned char)(b * 255.0);
    p->junk = 0;
}
```
**Function:** Converts color from HSV (Hue, Saturation, Value) space to RGB (Red, Green, Blue) space.

**UpdateImageData Function - The Heart of the Program:**
```c
void UpdateImageData(ImageState *state)
{
    double cx, cy, sin_a, cos_a;
    
    // Determining rotation center
    if (state->top_rotation == 1) {
        cx = (state->minx + state->maxx) / 2;
        cy = state->maxy;  // Rotation around top point
        cos_a = cos(state->top_rotation_angle);
        sin_a = sin(state->top_rotation_angle);
    } else {
        cx = (state->minx + state->maxx) / 2;
        cy = (state->miny + state->maxy) / 2;  // Rotation around center
        cos_a = cos(state->angle);
        sin_a = sin(state->angle);
    }
    
    // Parallel computation for each pixel
    #pragma omp parallel for
    for (int x = 0; x < state->width; x++) {
        for (int y = 0; y < state->height; y++) {
            // Convert pixel coordinates to mathematical coordinates
            double nx = state->minx + (state->maxx - state->minx) * x / (state->width - 1);
            double ny = state->miny + (state->maxy - state->miny) * y / (state->height - 1);
            
            // Apply rotation
            double dx = nx - cx;
            double dy = ny - cy;
            double rx = dx * cos_a - dy * sin_a + cx;
            double ry = dx * sin_a + dy * cos_a + cy;
            
            // Choose calculation type: Mandelbrot or Julia
            int iter;
            if (state->CR == 0 && state->CI == 0) {
                iter = get_mbs_iter(rx, ry, 256);
            } else {
                iter = get_julia_iter(rx, ry, state->CR, state->CI, 256);
            }
            
            // Store result
            if (iter == 256) {
                state->bmFileData.bmData[y * state->width + x] = 0;  // Inside the set
            } else {
                state->bmFileData.bmData[y * state->width + x] = iter;  // Outside the set
            }
        }
    }
    
    // Create color palette with dynamic color shift
    for (int i = 0; i < 256; i++) {
        int base_hue = 272 - (i * 180 / 256);
        int hue = ((int)(base_hue + state->hue_color)) % 360;  // Apply color shift
        hsv_to_rgb(hue, 96, 59, &(state->bmFileData.bmHeader.colorIdx[i]));
    }
    
    // Black color for points inside the set
    state->bmFileData.bmHeader.colorIdx[0].r = 0;
    state->bmFileData.bmHeader.colorIdx[0].g = 0;
    state->bmFileData.bmHeader.colorIdx[0].b = 0;
    state->bmFileData.bmHeader.colorIdx[0].junk = 0;
}
```

**State Change Functions:**

**Changing Julia Parameters:**
```c
void JuliaChange(ImageState *state, double t_cr, double t_ci, int steps)
{
    double cr_step = (t_cr - state->CR) / steps;
    double ci_step = (t_ci - state->CI) / steps;
    
    for (int i = 0; i < steps; i++) {
        state->CR += cr_step;
        state->CI += ci_step;
        UpdateImageData(state);
        WriteBitmapFile(state->image_count++, &state->bmFileData);
    }
}
```

**Dynamic Color Shift:**
```c
void ColorShift(ImageState *state, int steps)
{
    double hue_s = 120 / steps;  // 120-degree change throughout animation
    
    for (int i = 0; i < steps; i++) {
        state->hue_color += hue_s;
        if (state->hue_color >= 360)
            state->hue_color -= 360;
        UpdateImageData(state);
        WriteBitmapFile(state->image_count++, &state->bmFileData);
    }
}
```

**Zoom with Rotation:**
```c
void Zoom_TopRotate(ImageState *state, double zoom, double angle, int steps)
{
    state->top_rotation = 1;  // Enable rotation around top point
    
    for (int i = 0; i < steps; i++) {
        // Apply zoom
        spanx -= dx;
        spany -= dy;
        state->minx = cx - spanx / 2;
        state->maxx = cx + spanx / 2;
        state->miny = cy - spany;
        state->maxy = cy;
        
        // Apply rotation
        c_angle += as;
        state->top_rotation_angle = c_angle;
        
        // Simultaneous color change
        c_hue += hs;
        if (c_hue >= 360) 
            c_hue -= 360;
        state->hue_color = c_hue;
        
        UpdateImageData(state);
        WriteBitmapFile(state->image_count++, &state->bmFileData);
    }
    
    state->top_rotation = 0;  // Disable special mode
}
```

## Part 3: Music with Sonic Pi

### Music Structure:
1. **Main Melody:** Repeating pattern with notes G4, B3, E4
2. **Special Effects:**
   - Glissando (sliding from one note to another)
   - Gradual tempo changes
   - Multiple audio layers with `in_thread`

### Key Parts of Sonic Pi Code:
```ruby
# Creating ascending glissando
in_thread do
  start_note = note(:e4)
  end_note = note(:f7)
  steps = 150
  duration = 7.0
  
  (0..steps).each do |i|
    t = i.to_f / steps
    acceleration_factor = (t ** 0.3) * 0.8 + 0.2
    current_note_midi = start_note + (end_note - start_note) * t
    play current_note_midi, amp: 0.3, release: 0.05
    sleep_time = (duration / steps) * (1.5 - (t * 0.5))
    sleep sleep_time
  end
end

# Changing tempo to create movement sensation
use_bpm 93  # Change from 108 to 93
```

## How Audio and Video Are Synchronized

### Sample Configuration File:
```
800 600          # Width and height
-2.5 1.0 -1.25 1.25  # Initial range

center 0.0 -0.5 100    # Move to new center in 100 frames
zoom 10.0 200          # 10x zoom in 200 frames
rotate 360.0 150       # 360-degree rotation in 150 frames
julia -0.8 0.156 120   # Change to Julia set with c = -0.8 + 0.156i
colorshift 80          # Color change in 80 frames
zoomtoprotate 5.0 180.0 100  # Simultaneous zoom and rotation
```

### Timing Calculation:
```c
// In ProcessArgs function
double seconds = (Commands[cmdno].steps) / 24;  // 24 frames per second
fprintf(newfp, "zoom  %d steps  %f seconds\n", Commands[cmdno].steps, seconds);
```

## Advanced Technical Features

### 1. Parallelization:
```c
#pragma omp parallel for
for (int x = 0; x < state->width; x++) {
    // Independent computations for each column
}
```
**Advantage:** Execution speed increases several times.

### 2. Dynamic Memory Management:
```c
outcfg->Commands = (Cmd*)malloc(sizeof(Cmd) * cmdno);
// ...
free(cfg.Commands);
free(state.bmFileData.bmData);
```

### 3. Intelligent Coordinate System:
- Automatic conversion between pixel and mathematical coordinates
- Application of rotation matrix
- Support for multiple rotation centers

## Final Video Production Process

1. **Scenario Preparation:** Designing movement sequences in the configuration file
2. **Frame Production:** Running the C program with scenario input
3. **Audio Production:** Executing Sonic Pi code
4. **Combination:** Using FFmpeg to convert frames to video
5. **Merging:** Combining video and audio in editing software

## Conclusion

This project combines mathematical concepts (fractals), programming (C and Python), and art (music and animation). Using optimized algorithms and parallelization techniques, we were able to create a complex and beautiful animation of the Mandelbrot and Julia sets that is both visually appealing and technically efficient to execute.