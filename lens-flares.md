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
