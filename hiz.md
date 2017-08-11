# Notes on ssr HIZ tracing

The following is small gathering of notes and findings that we made through out the implementation of Hi-Z tracing in screen space for ssr in Stingray. I've recently heard a few claims regarding Hi-Z which motivated me to share some notes on the topic. Note that I also wrote about how we reproject reflections in a previous entry which might be of interest.

The original implementation was basically an implementation of the Hi-Z Screen-Space tracing described in GPU-Pro 5 by Yasin Uludag. The very first results we got looked something like this:

Original scene:
![](https://github.com/greje656/Questions/blob/master/images/ssr1.jpg)

Traced ssr:
![](https://github.com/greje656/Questions/blob/master/images/ssr2.jpg)

The weird horizontal stripes we're reported when ssr was enabled in the Stingray editor. They only revealed themselves for certain resolution (they would appear and disappear as the viewport got resized). I started writing some tracing visualization views to help me track each hiz trace event:

![](https://github.com/greje656/Questions/blob/master/images/ssr-gif7.gif)

Using these kinds of debug views were invaluable for debugging hiz tracing (and ssr in general) through out the development process. For the artifact described above I was able to see that for some resolution, the starting position of a ray when traced at half-res happened to be exactly ad the edge of a hi-z cell. Which caused the cell intersection routine to fail.

~~~~
float3 intersect_cell_boundary(float3 pos, float3 dir, float2 cell_id, float2 cell_count, float2 cross_step, float2 cross_offset) {
    float2 cell_size = 1.0 / cell_count;
    float2 planes = cell_id/cell_count + cell_size * cross_step + cross_offset;
    float2 solutions = (planes - pos.xy)/dir.xy;

    float3 intersection_pos = pos + dir * min(solutions.x, solutions.y);
    return intersection_pos;
}
~~~~

To tackle this problem we can snap the origin of each traced rays to the center of a hiz cell: 
![](https://github.com/greje656/Questions/blob/master/images/ssr-gif6.gif)

However it didn't address all the tracing artifacts. The trace was still plagued by a lot of small pixels whose traced failed. When investigating these failing traces I noticed that they would sometimes incorrectly jump hiz cell incorrectly. The artifact also appeared more frequently for rays travelling in the screen space axis (±1,0) or (0,±1). After drawing a hundred ray diagrams on paper I realized that the cell intersection method proposed in GPU-Pro has a failing case. The proposed solution offsets the intersection planes by a small amount to make sure the traversal never gets stuck. This works in most cases but can result in a ray that which will not cross over into the next hiz cell. Hence the ray will waste the rest of it's allocated trace iterations intersecting the same cell over and over without ever crossing it. We can address this by modifying the method slightly. Instead of offsetting the planes we leave as is, find the closest solution for the ray intersection, and choose an offset accordingly:

~~~~
float3 intersect_cell_boundary(float3 pos, float3 dir, float2 cell_id, float2 cell_count, float2 cross_step, float2 cross_offset) {
    float2 cell_size = 1.0 / cell_count;
    float2 planes = cell_id/cell_count + cell_size * cross_step;
    float2 solutions = (planes - pos)/dir.xy;

    float3 intersection_pos = pos + dir * min(solutions.x, solutions.y);
    intersection_pos.xy += (solutions.x < solutions.y) ? float2(cross_offset.x, 0.0) : float2(0.0, cross_offset.y);
    return intersection_pos;
}
~~~~

Left: The ray to trace against a cell. Middle: Ray gets stuck due to incorrect intersection. Left: Ray crosses in cell properly and continues tracing. 
![](https://github.com/greje656/Questions/blob/master/images/ssr11.jpg)

Using this method we we're able to get rid of the left over trace artifacts:

![](https://github.com/greje656/Questions/blob/master/images/ssr-gif9.gif)

Final result:
![](https://github.com/greje656/Questions/blob/master/images/ssr6.jpg)

