# Real time river editor #

I've always been interested in water in games. I played From Dust a lot when it came out and it fascinated me. More rencetly I was also very impressed with the mud and water system of Spintires MudRunner and Uncharted 4. I was watching the Siggraph 2016 talk "Rendering Rapids in Uncharted 4" and something really caught my attention. There was a small demo of an idea called "Wave Particles", developped by Cem Yuksel and al.

![](images/slide1.png)

I was amazed at the complexity that could arise by this simple idea of stacking moving simple wave particles on top of each other and had to try it for myself. I setup a small d3d12 app and even without any lighting it was obvious that it was possible to build a system around this that was very intuitive and fun to use:

[![](http://img.youtube.com/vi/MIGhIPTaTDI/0.jpg)](http://www.youtube.com/watch?v=MIGhIPTaTDI)

Normally for open water the philip spectrum is used to distribute a number of waves lengths and amplitude, but I've found that using some logarithmic distribution made it easier to reason about the coverage of the waves on a given surface.

[[image of wave distribution]]

 One thing I tried early on was to combine the idea of wave particles and wave packets that was presented by Stefan Jeschke during the Siggrapgh of 2017. A wave packet roughly speaking is similar to a wave particle, but it carries with it multiple wave fronts:

[[image of wave packet]]

 The hope was that these complex particles could capture subtle water surface behaviors which would not be possible with simple wave particles. While it was an interesting idea I did not have much luck with them. Mainly because it was difficult to gather the longitudinal forces of the waves but also because they prevented the use of an important optimization that plain wave particles offered (http://www.cemyuksel.com/research/waveparticles/waveparticles_sketch_slides.pdf). So I ended up abandonning this effort early on and sticked to using plain wave particles:

[![](http://img.youtube.com/vi/I1l587DTKIE/0.jpg)](http://www.youtube.com/watch?v=I1l587DTKIE)

Once I had the lighting and modelling in a descent state, I turned my attention to generating an interesting vector field which could drive the motion of these waves.

I first looked at the distance field approach made popular by Valve with the game Portal. Unfortunatly this approach lacks the ability of creating interesting vortices which was a property I was hoping to have. I also looked at Jos Stam stable fluid method. Since this is an actual simulation the method captures some interesting fluid properties:

[![](http://img.youtube.com/vi/V9bH3QCu90w/0.jpg)](http://www.youtube.com/watch?v=V9bH3QCu90w)

But then I found out about LBM solvers. The more I looked into these the more they revealed interesting properties. They seemed straight forward to parallelize and port to GPU. They captured vorticity effects really well and on top of that they could be used to solve the Shallow Water Equation. Some configuration even appeared to converge to some stable state which seemed like a desirable property if the vector field needs to be stored offline.

I always like to start implementing these complex systems on CPU since it allows me to set break points and investigate the simulated data much easier than when ran on a GPU. But even at a very low resolution and frame rate, I could see that this method seemed promissing.

[![](http://img.youtube.com/vi/4aXSyiukvOI/0.jpg)](http://www.youtube.com/watch?v=4aXSyiukvOI)

For more information on how to solve the Shallow Water Equation using the LBM method, I recommend this fantastic book by Dr. Jian Guo Zhou (https://www.springer.com/gp/book/9783540407461).

I then ported the code to compute which allowed me to run the simulation at a much higher resolution:

[![](http://img.youtube.com/vi/seChiG6V94Q/0.jpg)](http://www.youtube.com/watch?v=seChiG6V94Q)

I was sold on the benifits of the LBM solver very early. The main drawback is the memory required to run it. The method needs to operate on 9 float components (so far 16F per component seems to be enough). To make it worst, I'm currently using a ping/pong scheme which means twice the amount of data. For example, if running the simulation on a 2d grid that's 256x128, then it's (256x128pixels) x (9 components) x (16 bits) x (2 for ping/ponging)

The vector field is then used to advect some data. Mostly foam related. At the moment the foam amount and 2 sets of uv coordinates are advected (5 more float components). TheWe need two sets of uvs because we will fetch the foam texture with each set and continually blend back and forth between the two samples as it stretches because of the advection.

[image of advected textures only]

Advecting the wave heights and normal maps didn't prove very successful. A very visble and annoying pusling appeared. The pulsing could be minimized using some perlin noise to add some variation but it was a difficult battle to fight. There is a lot of research done on this subject, especially from Fabrice Neyret and al. But I haven't seen anything that comes accross as a silver bullet. This idea
http://www-evasion.imag.fr/Publications/2003/Ney03/ basically advects 3 sets of uv coordinates and lets you choose which set would yeild less distortion. This idea https://www.youtube.com/watch?v=HXrCSJqXcl0 advects particles in the velocity fields and then splats out a uv set with very limited stretching. Incredible results but very expensive to calculate.

[![](http://img.youtube.com/vi/NqSOnmh2_do/0.jpg)](http://www.youtube.com/watch?v=NqSOnmh2_do)

Fortunatly, roughly at the same time I was tackling these issues, Stefan Jeschke presented yet another new idea called wave Profile Buffers during his Siggraph 2018 talk. As soon as I saw the presentation I started reading the paper. One key property of the Wave Profiles is that they don't require uv coordinates. To sample them you only need a world position, a time value, and a wave direction. This means that the wave fronts generated by sampling these wave profiles are spacially and temporally coherant. i.e. no stretching or tearing!

[![](http://img.youtube.com/vi/iGu_1Yvkukg/0.jpg)](http://www.youtube.com/watch?v=iGu_1Yvkukg)

## A note on lighting ##
The lighting of the water is fairly standard. It basically boils down to a specular term for the water surface and a scattering term which simulates the light scattered in the water volume coming back towards the eye. For the specular term I'm simply sampling a cube map which is generated using an atmospheric shader. There's a really good blog series on this subject that I love written by Alan Zucconi https://www.alanzucconi.com/2017/10/10/atmospheric-scattering-1/. But the specular term could be anything that your pipeline is currently using.

For the volumetric light scattering approximation, I define an imaginary point as the halfway point between the water surface being shaded and the floor directly underneath it. I then light this point with two imaginary infinite planes which acts as large area lights. The luminance of these two planes are defined as WaterColor*(N*L)*SunColor and GroundColor*(N*L)*SunColor*e^(-depth) respectively. I then solve the two exponential integrals  using an analytical solution and average them.

When the sun angle hits a certain grazing angle threshold, I also refract a ray and fetch a sky value and inject this as extra water scattering that can occur on the wave tips. Something I haven't tried yet but might be interesting would be to support self occlusion of the water surface by doing some screen space shadow tracing, but for now the wave heights I'm toying with are too small for it to see any interesting benefit. For larger waves it seems like self occlusion could definitly be interesting (http://www.rutter.ca/files/large/0405f593e9fabc4)

Future work:
There is a lot of work left to do for this to be production ready. The next step will be to implement a system which will let me paint large river canals. I have been thinking about using some kind of virtual texture setup where the velocity field could be baked along with some initial condition which could be used to bootstrap the advection for a tile as the camera approaches a certain area. This would elliminate the cost of running the simulation at runtime. 

Also, I'm interested to see how far one could go with pre-generating Wave Profile Buffers which would make them much more feasible to use on a tigheter gpu budget.