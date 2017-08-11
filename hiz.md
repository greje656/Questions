# Notes on ssr HIZ tracing

The following is small gathering of notes and findings that we made through out the implementation of Hi-Z tracing in screen space for ssr in Stingray. I've recently heard a few claims regarding Hi-Z which motivated me to share some notes on the topic. Note that I also wrote about how we reproject reflections in a previous entry which might be of interest.

The original implementation was basically an implementation of the Hi-Z Screen-Space tracing described in GPU-Pro 5 by Yasin Uludag. The very first results we got looked something like this:

Original scene:
![](https://github.com/greje656/Questions/blob/master/images/ssr1.jpg)

Traced ssr:
![](https://github.com/greje656/Questions/blob/master/images/ssr2.jpg)

The weird horizontal stripes we're reported when ssr was enabled in the Stingray editor. They only revealed themselves for certain resolution (they would appear and disappear as the viewport got resized). I started writing some tracing visualization views to help me track each hiz trace event:

![](https://github.com/greje656/Questions/blob/master/images/ssr-gif2.gif)

Using these kinds of debug views were invaluable for debugging hiz tracing (and ssr in general) through out the development process. For the artifact described above I was able to see that for some resolution, the starting position of a ray when traced at half-res happened to be exactly ad the edge of a hi-z cell. Which caused the cell intersection routine to fail.

~~~~
float3 intersect_cell_boundary(float3 pos, float3 dir, float2 cell_id, float2 cell_count, float2 cross_step, float2 cross_offset) {
    float2 cell_size = 1.0 / cell_count;
    float2 planes = cell_id/cell_count + cell_size*cross_step + cross_offset;
    float2 solutions = (planes - pos.xy)/dir.xy;
    return pos + dir * min(solutions.x, solutions.y);
}
~~~~

To tackle this problem I made sure that the trace always started from the center of a hiz cell. 

![](https://github.com/greje656/Questions/blob/master/images/ssr-gif4.gif)
