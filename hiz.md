# Notes On SSR HIZ Tracing

The following is small gathering of notes and findings that we made through out the implementation of hiz tracing in screen space for ssr in Stingray. I've recently heard a few claims regarding hiz which motivated me to share some notes on the topic. Note that I also wrote about how we reproject reflections in a previous entry which might be of interest.

The original implementation was basically an implementation of the Hi-Z Screen-Space tracing described in GPU-Pro 5 by Yasin Uludag. The very first results we got looked something like this:

Original scene:
![](https://github.com/greje656/Questions/blob/master/images/ssr1.jpg)

Traced ssr:
![](https://github.com/greje656/Questions/blob/master/images/ssr2.jpg)

## Artifacts

The weird horizontal stripes we're reported when ssr was enabled in the Stingray editor. They only revealed themselves for certain resolution (they would appear and disappear as the viewport got resized). I started writing some tracing visualization views to help me track each hiz trace event:

![](https://github.com/greje656/Questions/blob/master/images/ssr-gif7.gif)

Using these kinds of debug views were invaluable for debugging hiz tracing (and ssr in general) through out the development process. For the artifact described above I was able to see that for some resolution, the starting position of a ray when traced at half-res happened to be exactly ad the edge of a hiz cell. Which caused the cell intersection routine to fail.

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

However it didn't address all the tracing artifacts. The trace was still plagued by a lot of small pixels whose traced failed. When investigating these failing traces I noticed that they would sometimes incorrectly jump hiz cell incorrectly. The artifact also appeared more frequently for rays travelling in the screen space axes (±1,0) or (0,±1). After drawing a hundred ray diagrams on paper I realized that the cell intersection method proposed in GPU-Pro has a failing case. The proposed solution offsets the intersection planes by a small amount to make sure the traversal never gets stuck. This works in most cases but can result in a ray that which will not cross over into the next hiz cell. Hence the ray will waste the rest of it's allocated trace iterations intersecting the same cell over and over without ever crossing it. We can address this by modifying the method slightly. Instead of offsetting the planes we leave as is, find the closest solution for the ray intersection, and choose an offset accordingly:

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

## Ray Marching Towards the Camera

At the end of the GPU-Pro chapter there is a small mention that raymarching towards the camera with hiz tracing would require storing both the minimum and maximum depth value in the hiz structure (requiring to bump the format to a R32G32F format. However if you visualize the trace of a ray leaving the surface and travelling towards the camera (i.e. away from the depth buffer plane) then you can augment the algorithm described GPU-Pro to find the first hit with a depth cell:

~~~
if(v.z > 0) {
  float min_minus_ray = min_z - ray.z;
  tmp_ray = min_minus_ray > 0 ? ray + v_z*min_minus_ray : tmp_ray;
  float2 new_cell_id = cell(tmp_ray.xy, current_cell_count);
  if(crossed_cell_boundary(old_cell_id, new_cell_id)) {
    tmp_ray = intersect_cell_boundary(ray, v, old_cell_id, current_cell_count, cross_step, cross_offset);
    level = min(HIZ_MAX_LEVEL, level + 2.0f);
  }
} else if(ray.z < min_z) {
  tmp_ray = intersect_cell_boundary(ray, v, old_cell_id, current_cell_count, cross_step, cross_offset);
  level = min(HIZ_MAX_LEVEL, level + 2.0f);
}
~~~

This has proven to be fairly solid and enables us to trace a wider range of the screen space:

https://www.youtube.com/watch?v=BjoMu-yI3k8&feature=youtu.be

## Ray Marching Behind Surfaces

Another alteration that can be made to the hiz tracing algorithm is to add support for rays to travel behind surface. Of course to do this you must define a thickness to the surface of the hiz cells. So instead of tracing against extruded hiz cells you trace against "floating" hiz cells.

~~~
if(level == HIZ_START_LEVEL && min_minus_ray > depth_threshold) {
  tmp_ray = intersect_cell_boundary(ray, v, old_cell_id, current_cell_count, cross_step, cross_offset);
  level = HIZ_START_LEVEL + 1;
}
~~~

Tracing Behind Surfaces disabled VS enabled:
![](https://github.com/greje656/Questions/blob/master/images/ssr13.jpg)
![](https://github.com/greje656/Questions/blob/master/images/ssr15.jpg)

Unfortunately this often means that the traced rays travelling behind a surface degenerate into a linear search and the cost can skyrocket for these traced pixels: 

Iterations count to complete trace (black=0, red=128)
![](https://github.com/greje656/Questions/blob/master/images/ssr14.jpg)


## The Problem of Tracing a Discrete Depth Buffer

For me the most difficult and hard to understand artifact of ssr is (by far) the implications of tracing a discreet depth buffer. Unless you can fully commit to the idea of tracing objects with infinite thicknesses, you will need to use some kind of depth threshold to mask a reflection if it's intersection with the object is not valid. If you do use a depth threshold then you can (will?) end up getting artifacts like these:

https://youtu.be/ZftaDG2q3D0

The problem as far as I understand it, is that rays can osciliate from hitting the surface back and forth when it's tracing rays that are almost parallelel to the extruded depth cells:
![](https://github.com/greje656/Questions/blob/master/images/ssr-depth-threshold01.jpg)

I have experimented with adapting the threshold based on different properties of the intersection (direction of reflected ray, angle of insidence at intersection, surface inclination at intersection) but I have never been able to find a silver bullet (or anything that resembles a bullet to be honest). I think that Mikkel Svendsen proposed a solution to this problem while presenting [https://youtu.be/RdN06E6Xn9E?t=40m27s](https://youtu.be/RdN06E6Xn9E?t=40m27s "Low Complexity, High Fidelity: The Rendering of INSIDE") but I have yet to wrap my head around the proposed solution and try it.
