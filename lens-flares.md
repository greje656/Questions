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

#Ray tracing
Ok let's get into it. To trace rays in an optical system we obviously need to build an optical system first. This part can be tedious. Not only have you got to find the "Lens Prescription" of a lens, you also need to manually parse it. For example, parsing the Nikon 28-75mm patent data might look something like this:

![](https://github.com/greje656/Questions/blob/master/images/lens-description.jpg)

There is no standard way of describing such systems. You may find all the information you need from a lens patent, but often (especially for older lenses) you end up staring at an old Russian document that seems to be missing important information required for the algorithm. For example, the Russian lens MIR-1 apparently produces beautiful flares, but the only lens description I could find for it was this:

![](https://github.com/greje656/Questions/blob/master/images/mir-1.jpg)

(from http://allphotolenses.com/public/files/pdfs/ce6dd287abeae4f6a6716e27f0f82e41.pdf)

A parsed optical system will look something like this:

Once you have parsed your lens description into something your trace algorithm can consume, you can then start to ray trace points. The idea is to start with a tesselated patch and trace through each of the points (these are called ray bundles in the paper). There are a couple of subtleties to note regarding the tracing algorithm.


