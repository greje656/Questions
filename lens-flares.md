#Lens Flares

While playing Horizon Zero Dawn I got inspired by the lens flares they supported and decided to look into implementing some basic flares in Stingray. There we're four types of flares I was particularly interested in supporting.

1) Anisomorphic flares
2) Aperture diffraction (Starbursts)
3) Camera ghosts (Low quality all other light sources in a scene)
4) Camera ghosts (High quality for Sun/Moon)

I intend to do a follow up blog posts once the Camera plugin once finished, but for now I wanted share the implementation details of the high-quality ghosts which are an implementation of "Physically-Based Real-Time Lens Flare Rendering" method (http://resources.mpi-inf.mpg.de/lensflareRendering).

All the code mentioned in this article can can be found here https://github.com/greje656/PhysicallyBasedLensFlare

#Ghosts

The basic idea of the "Physically-Based Lens Flare" paper is to ray trace "ray bundles" into a lens system which will end up on a sensor to form a "ghost". A ghost here refers to the de-focused light that reaches the sensor of a camera due to the light reflecting off the lens. Since camera lens are not made of a single optical lens but many lens there can be many ghosts that forms on the sensor. If we only consider the ghosts that are formed from two bouces, that's a total of nCr(n,2) ghosts (where n is the number of lens components in a lens)

![](https://github.com/greje656/Questions/blob/master/images/ghost01.jpg)
![](https://github.com/greje656/Questions/blob/master/images/ghost02.jpg)
![](https://github.com/greje656/Questions/blob/master/images/ghost03.jpg)

(By the way, implementing a 2d version on the cpu has proven an invaluable way of understanding and debugging the gpu version).

While some appreciate the artistic aspect of lens flare, lens manufacturers work hard on trying to minimize these by coating lenses with anti-reflection. As Padraic Hennessy points out on his blog, great progress was made with this in the last 20 years.

#Lens Interface Description

Ok let's get into it. To trace rays in an optical system we obviously need to build an optical system first. This part can be tedious. Not only have you got to find the "Lens Prescription" of a lens, you also need to manually parse it. For example, parsing the Nikon 28-75mm patent data might look something like this:

![](https://github.com/greje656/Questions/blob/master/images/lens-description.jpg)

There is no standard way of describing such systems. You may find all the information you need from a lens patent, but often (especially for older lenses) you end up staring at an old Russian document that seems to be missing important information required for the algorithm. For example, the Russian lens MIR-1 apparently produces beautiful flares, but the only lens description I could find for it was this:

![](https://github.com/greje656/Questions/blob/master/images/mir-1.jpg)

(from http://allphotolenses.com/public/files/pdfs/ce6dd287abeae4f6a6716e27f0f82e41.pdf)

#Ray Tracing

Once you have parsed your lens description into something your trace algorithm can consume you can then start to ray trace points! The idea is to start with a tessellated patch and trace through each of the points (these are called ray bundles in the paper). There are a couple of subtleties to note regarding the tracing algorithm.

First, when a ray "misses" a lens, the raytracing routine isn't necessary stopped. Instead if the ray can continue with a path that is "meaningful" the ray trace continues until it reaches the sensor.

![](https://github.com/greje656/Questions/blob/master/images/trace-01.jpg)

![](https://github.com/greje656/Questions/blob/master/images/trace-02.jpg)

Only if the ray misses the sphere formed by the radius of the lens do we break the raytracing routine. The idea is that each ray trace keeps track of the maximum relative distance it had with a lens component at an intersection. Continuing to trace the ray, and using it's interpolated data yields a more stable signal than would be achieved otherwise.

Secondly, a ray bundle carries a fixed amount of energy so it is important to consider the distortion of the bundle area that occurs while tracing them. In the paper there was this small segment:

*"At each vertex, we store the average value of its surrounding neighbors. The regular grid of rays, combined with the transform feedback (or the stream-out) mechanism of modern graphics hardware, makes this lookup of neighboring quad values very easy"*

I still don't really understand how the transform feedback, along with the available adjacency information of the geometry shade could be enough to provide the information of the four surrounding areas of a vertex (if you know please leave a comment). Luckily for me we now have compute and UAVs which turns this problem into a fairly trivial one. Currently I only calculate an approximation of the surrounding areas by assuming they neighbouring quads are roughly parallelograms. I estimate their bases and heights as the average lengths o their top/bottom, left/right segments. You can see the results as caustics where some bundles converge into tighter area patches while some other dilates:

![](https://github.com/greje656/Questions/blob/master/images/lens-area.jpg)

This seems to work fairly well, although it is expansive. Something that I will improve in the future.

Now that we have a traced patch we need to make some sense out of it. The patch "as is" can look intimidating at first. Due to early exits of some rays the final vertices can sometimes look like something went terribly wrong:

![](https://github.com/greje656/Questions/blob/master/images/discard03.jpg)

The first thing to do is discard pixels that exited the lens system. 

~~~~
float intensity1 = color.z < 1.0f;
float intensity = intensity1;
if(intensity == 0.f) discard;
~~~~

![](https://github.com/greje656/Questions/blob/master/images/discard04.jpg)
![](https://github.com/greje656/Questions/blob/master/images/discard05.jpg)
![](https://github.com/greje656/Questions/blob/master/images/discard06.jpg)
![](https://github.com/greje656/Questions/blob/master/images/discard07.jpg)

