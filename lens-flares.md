# Lens Flares

While playing Horizon Zero Dawn I got inspired by the lens flares they supported and decided to look into implementing some basic flares in Stingray. There we're four types of flares I was particularly interested in.

1) Anisomorphic flares
2) Aperture diffraction (Starbursts)
4) Camera ghosts due to Sun or Moon (High quality)
3) Camera ghosts due to all other light sources (Low quality)

I intend to do a follow up blog posts once the Camera plugin once finished, but for now I wanted share the implementation details of the high-quality ghosts which are an implementation of "Physically-Based Real-Time Lens Flare Rendering" (http://resources.mpi-inf.mpg.de/lensflareRendering).

All the code used to generate the images and videos of this article can can be found here https://github.com/greje656/PhysicallyBasedLensFlare

### Ghosts

The basic idea of the "Physically-Based Lens Flare" paper is to ray trace "bundles" into a lens system which will end up on a sensor to form a ghost. A ghost here refers to the de-focused light that reaches the sensor of a camera due to the light reflecting off the lenses. Since a camera lens is not typicallt made of a single optical lens but many lenses there can be many ghosts that forms on it's sensor. If we only consider the ghosts that are formed from two bounces, that's a total of nCr(n,2) possible ghosts (where n is the number of lens components in a camera lens)

![](https://github.com/greje656/Questions/blob/master/images/ghost01.jpg)
![](https://github.com/greje656/Questions/blob/master/images/ghost02.jpg)
![](https://github.com/greje656/Questions/blob/master/images/ghost03.jpg)

### Lens Interface Description

Ok let's get into it. To trace rays in an optical system we obviously need to build an optical system first. This part can be tedious. Not only have you got to find the "Lens Prescription" you are looking for, you also need to manually parse it. For example parsing the Nikon 28-75mm patent data might look something like this:

![](https://github.com/greje656/Questions/blob/master/images/lens-description.jpg)

There is no standard way of describing such systems. You may find all the information you need from a lens patent, but often (especially for older lenses) you end up staring at an old document that seems to be missing important information required for the algorithm. For example, the Russian lens MIR-1 apparently produces beautiful flares, but the only lens description I could find for it was this:

![](https://github.com/greje656/Questions/blob/master/images/mir-1.jpg)

(http://allphotolenses.com/public/files/pdfs/ce6dd287abeae4f6a6716e27f0f82e41.pdf)

### Ray Tracing

Once you have parsed your lens description into something your trace algorithm can consume you can then start to ray trace points! The idea is to start with a tessellated patch and trace through each of the points (these are called ray bundles in the paper). There are a couple of subtleties to note regarding the tracing algorithm.

First, when a ray "misses" a lens component the raytracing routine isn't necessarily stopped. Instead if the ray can continue with a path that is meaningful the ray trace continues until it reaches the sensor.

![](https://github.com/greje656/Questions/blob/master/images/trace-01.jpg)

![](https://github.com/greje656/Questions/blob/master/images/trace-02.jpg)

Only if the ray misses the sphere formed by the radius of the lens do we break the raytracing routine. The idea behind this is to get as many traced points to reach the sensor so that the interpolated data can remain as continuous as possible. Rays track the maximum relative distance it had with a lens component while traching through the interface so this relative distance will be used in the pixel shader later to determine if a ray had left the interface (i.e. the relative distance tracked is > 1).

Secondly, a ray bundle carries a fixed amount of energy so it is important to consider the distortion of the bundle area that occurs while tracing them. In the paper, the author makes this small segment:

*"At each vertex, we store the average value of its surrounding neighbors. The regular grid of rays, combined with the transform feedback (or the stream-out) mechanism of modern graphics hardware, makes this lookup of neighboring quad values very easy"*

I don't understand how the transform feedback, along with the available adjacency information of the geometry shader could be enough to provide the information of the four surrounding quads of a vertex (if you know please leave a comment). Luckily we now have compute and UAVs which turns this problem into a fairly trivial one. Currently I only calculate an approximation of the surrounding areas by assuming they neighbouring quads are roughly parallelograms. I estimate their bases and heights as the average lengths o their top/bottom, left/right segments. The results are seen as caustics form where some bundles converge into tighter area patches while some other dilates:

![](https://github.com/greje656/Questions/blob/master/images/lens-area.jpg)

This works fairly well but is expansive. Something that I intend to improve in the future.

Now that we have a traced patch we need to make some sense out of it. The patch "as is" can look intimidating at first. Due to early exits of some rays the final vertices can sometimes look like something went terribly wrong:

![](https://github.com/greje656/Questions/blob/master/images/discard03.jpg)

The first thing to do is discard pixels that exited the lens system:

~~~~
float intensity1 = max_relative_distance < 1.0f;
float intensity = intensity1;
if(intensity == 0.f) discard;
~~~~

![](https://github.com/greje656/Questions/blob/master/images/discard04.jpg)

Then we can discard the rays that didn't have any energy as they entered to begin with (say outside the sun disk):

~~~~
float lens_distance = length(entry_coordinates.xy);
float sun_disk = 1 - saturate((lens_distance - 1.f + fade)/fade);
sun_disk = smoothstep(0, 1, sun_disk);
...
float intensity2 = sun_disk;
float intensity = intensity1 * intensity2;
if(intensity == 0.f) discard;
~~~~

![](https://github.com/greje656/Questions/blob/master/images/discard05.jpg)

Then we can discard the rays that we're blocked by the aperture:

~~~~
...
float intensity3 = aperture_sample;
float intensity = intensity1 * intensity2 * intensity3;
if(intensity == 0.f) discard;
~~~~

![](https://github.com/greje656/Questions/blob/master/images/discard06.jpg)

Finally we adjust the radiance of the beams based on their final areas:

~~~~
...
float intensity4 = (original_area/(new_area + eps)) * energy;
float intensity = intensity1 * intensity2 * intensity3 * intensity4;
if(intensity == 0.f) discard;
~~~~

![](https://github.com/greje656/Questions/blob/master/images/discard07.jpg)

The final value is the rgb reflectance value of the ghost modulated by the incoming light color:

~~~~
float3 color = intensity * input.reflectance.xyz * TemperatureToColor(INCOMING_LIGHT_TEMP);
~~~~

### Aperture
The aperture shape is built proceduraly. As suggested by Padraic Hennessy's blog I use a sign distance field confined by "n" segments and threshold it against some distance value. I also experimented with approximating the light diffraction that occurs at the edge of the apperture blades using a simple function: https://www.desmos.com/calculator/munv7q2ez3:

![](https://github.com/greje656/Questions/blob/master/images/apertures1.jpg)

Finally, I offset the signed distance field with a repeating sin function which can give curved aperture blades:

![](https://github.com/greje656/Questions/blob/master/images/apertures2.jpg)

### Starburst

The starburst phenomena is due to light diffraction that passes through the small aperture hole. It's a phenomena known as the "single slit diffraction of light". The author got really convincing results to simulate this using the Fraunhofer approximation. The challenge with this approach is that it requires bringing the aperture texture into fourrier space which is not trivial. In previous projects I used Cuda's math library to perform FFT on a signal but since the goal is to bring this into Stingray I didn't want to have such a dependency. Luckily I found this little gem posted by Joseph S. from intel (https://software.intel.com/en-us/articles/fast-fourier-transform-for-image-processing-in-directx-11). He provides a clean and elegant compute implementation of the butterfly passes method which bring a signal to and from fourier space. Using it I was able to feed in the aperture shape and extract a fourier power spectrum.

![](https://github.com/greje656/Questions/blob/master/images/starburst04.jpg)

This sprectrum needs to be filtered further in order to look like a starburst. This is where the Fraunhofer approximation comes in. The idea is to basically reconstruct the diffraction of white light by summing up the diffraction of multiple wavelengths. The key observation is that same fourrier signal can be used for all wavelength. The only thing needed is to scaled the sampling coordinates of the fourier signal based on the wavelength of the light (x0 , y0 ) = (u, v)*λ*z0.

![](https://github.com/greje656/Questions/blob/master/images/starburst01.jpg)

With this, you can also apply an extra filtering step that helps create a starburst that's a little more appealing. For example, I used a spiral pattern mixed with a small rotation to get a more intersting result (to be honest I think that's what the author is also doing to get the results he presents in his publication). This is another area that I would like to investigate a bit more in the future.

![](https://github.com/greje656/Questions/blob/master/images/starburst02.jpg)

### Anti Reflection Coating

While some appreciate the artistic aspect of lens flare, lens manufacturers work hard on trying to minimize these by coating lenses with anti-reflection. As Padraic Hennessy points out on his blog, great progress was made with this in the last 20 years. (more detail here)

### Optimisations

Currently the performance of the Nikon lens with an light source of x is ~blah. The performance degrades as the light source is made bigger since it results in more and more overshading. With a simplier lens interface the cost decreases. This makes sense since (in the current implementation at least) every possible "two bounces" ghosts is traced and drawn. For a lens system like the Nikon 28-75mm which has 27 lens components, that's n!/r!(n-r)! = 352 ghosts. It's easy to see that this number can increase dramatically with the number of component that are part of a lens:
https://www.desmos.com/calculator/rsrjo1mhy1
One obvious optimization would be to skip ghosts that have low enough intensities that they are not perceptible. Using Compute/Draw Indirect it would be possible to do a very coarse raytrace pass to decide which ghosts exceed a certain intensity threshold and hence reduce the computation and rasterization pressure. This is on my todo list.
