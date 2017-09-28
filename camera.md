# Physical Cameras in Stingray (Part 1) #
This is a quick blog to share some of the progress we made lately with Physical Cameras in Stingray. Our goal of implementing a solid physically based pipeline has always been split in three phases. First we validated our standard material. We then added physical lights. And now we are wrapping it up with a physical camera.

We define a physical camera as an entity controlled by the same parameters a real world camera would use. These parameters are split into two groups which corresponds to the two main parts of a camera. The camera _body_ is defined by it's sensor size, iso sensitivity, and a range of available shutter speeds. The camera _lens_ is defined by it's focal length, focus range, and range of aperture diameters. Setting all of these parameters should expose the inconming light the same way a real world camera would.

### Stingray Representation ###
Just like our physical light, our camera is expressed as an entity with a bunch of components. The main two components being the Camera Body and the Camera Lens. All other component values are driven by these first two. The mapping of these values is done through a script component which also belongs to the camera entity. Here is a glimpse of what the Physical Camera entity may look like (wip):

![](images/cameras/res6.jpg)

So while there are a lot of components that belongs to a camera, the user is expected to interact only with the body and lens component. The value of all the other components are derived from these main two through the "Properties Mapper" script component. 

### Post Effects ###
A lot of our post effects are didicated to simulate some sort of camera/lens artifact (DOF, motion blur, film grain, vignetting, bloom, chromatic aberation, ect). One thing we wanted was the ability for physical cameras to override some of the post processes defined in our global shading environments. We also wanted to let users easily opt out of the physically based mapping that occured between a camera and it's corresponding post-effect. For example a physical camera will generate a accurate circle of confusion for the depth of field effect, but a user might be frustrated by the limitations imposed by a physically correct dof effect. In this case a user can optout by simply deleting the "Depth Of Field" component from the camera entity.

It's nice to see how the expressiveness of the Stingray entity system is shaping up and how it enables us to build these complex entities without the need to change much of the engine.

### Properties Mapper ###
All of the mapping occurs in the properties mapper component which is basically just a lua script that gets executed whenever any of the entity properties are edited.

The most important property we wanted to map was the exposure value. We wanted the f-stop, shutter speed, and ISO values to map to an exposure value which would simulate how a real camera sensor reacts to incoming light. Lucky for is, this is a topic that was very well covered by Sebastien Lagarde and Charles de Rousiers in their awesome awesome awesome [Moving Frostbite to Physically Based Rendering](https://seblagarde.files.wordpress.com/2015/07/course_notes_moving_frostbite_to_pbr_v32.pdf) document. The mapping basically boils down to:

~~~
local function compute_ev(aperture, shutter_time, iso)
	local ev_100 = log2((aperture * aperture * 100) / (shutter_time * iso))
	local max_luminance = 1.2 * math.pow(2, ev_100)
	return (1 / max_luminance)
end
~~~

The second property we were really keen on mapping is the field of view of the camera. Usually the horizontal FOV is calculated as _2 x atan(h/2f)_ where _h_ is the camera sensor's with and _f_ is the current focal length of the lens. This by itself gives a good approximation of the FOV of a lens, but as was pointed out by the [MGS5 & Fox Engine presentation](https://youtu.be/FQMbxzTUuSg?t=50m12s), the focus distance of the lens should also be considered when mapping calculating FOV from the camera properties.

Intuitively we though that the change in the FOV was caused by a change in the effective focal length of the lens. Adjusting the focus usually shifts a group of lens up and down the optical axis of a lens and our understanding/guess was that this shift increased and decreased the effective focal length of the lens. Using this idea we we're able to simulate the effect changing the focus point has on the FOV of a camera:

[https://www.youtube.com/watch?v=KDwUi-vYYMQ](https://www.youtube.com/watch?v=KDwUi-vYYMQ&feature=youtu.be)

~~~
local function compute_fov(focal_length, film_back_height, focus)
	local normalized_focus = (focus - 0.38)/(5.0 - 0.38)
	local focal_length_offset = lerp(0.0, 1.0, normalized_focus)
	return 2.0 * math.atan(film_back_height/(2.0 * (focal_length + focal_length_offset)))
end
~~~


While this gave us plausible results in some cases it didn't exactly map accuratly with the behavior of a real world lens for some lens setting. This is an area we would like to improve on (we will need to think about the optics of the lens a little bit more).

In the future we will continue to map more and more of the camera's properties to their corresponding post-effects. More on this in a follow up blog.

### Validating Results ###
To validate our mappings we designed a small controlled environment room which we then re-created in stingray. This would allow us to compare renders and actual photographs. It's really a "poor programer" equivalent of the cool "Conference Room" setup that was presented by Hideo Kojima in the [MGS5 & Fox Engine presentation](https://youtu.be/FQMbxzTUuSg?t=20m22s).

Here's a checklist of what has proven useful:
- Access to a dark room (ideally with farily dark diffuse walls)
- Camera
- Tripod
- ColorChecker
- Tape measure
- Light bulb (with known Lumens and Color Temperature values)
- Light Meter

![](images/cameras/res4.jpg)

![](images/cameras/res3.jpg)

Our very first comparaison we're disapointing:

Findings:
One thing we observed early on is that we had to scale our exposure by a fudge factor to get a similar exposure.

We eventually tracked down the exposure difference to a problem with how we express our lights intensity. Originally we had light intensity = value in lumens. This isnt right. The total luminoud flux is expressed in lumens, but the luminous intensity the.material shader is interested in is actually thr luminous flux per solid.angle. So while we let the users enter the intensity of lights in lumens, we need to map thr luminous intensity as blahblahbalh. This works for pointlights and spotlights. Directional lights will be assumed to be suns or moons and will be expressed in lux. Ies profiles will have to be considered (not done at the time.of writting this).

Now we are getting far more encouraging results!

![](images/comp-01.gif)

Stingray 1.9 is just around the corner and with it will come our new physical lights. I wanted to write a little bit about the validation process that we went through to increase our confidence in the behaviour of our materials and lights.

Early on we were quite set on building a small controlled "light room" similar to what the [Fox Engine team presented at GDC](https://youtu.be/FQMbxzTUuSg?t=19m25s) as a validation process. But while this seemed like a fantastic way to confirm the entire pipeline is giving plausible results, it felt like identifying the source of discontinuities when comparing photographs vs renders might involve a lot of guess work. So we decided to delay the validation process through a controlled light room and started thinking about comparing our results with a high quality offline renderer. Since [SolidAngle](https://www.solidangle.com/) joined Autodesk last year and that we had access to an [Arnold](https://www.solidangle.com/arnold/) license server it seemed like a good candidate. Note that the Arnold SDK is extremely easy to use and can be [downloaded](https://www.solidangle.com/arnold/download) for free. If you don't have a license you still have access to all the features and the only limitation is that the rendered frames are watermarked. 

We started writing a Stingray plugin that supported simple scene reflection into Arnold. We also implemented a custom Arnold Output Driver which allowed us to forward Arnold's linear data directly into the Stingray viewport where they would be gamma corrected and tonemapped by Stingray (minimizing as many potential sources of error).

### Material parameters mapping ###
The trickiest part of the process was to find an Arnold material which we could use to validate. When we started this work we used Arnold 4.3 and realized early that the Arnold's [Standard shader](https://support.solidangle.com/display/AFMUG/Standard) didn't map very well to the Metallic/Roughness model. We had more luck using the [alSurface shader](http://www.anderslanglands.com/alshaders/alSurface.html) with the following mapping:

~~~~
// "alSurface"
// ==============================================================================================
AiNodeSetRGB(surface_shader, "diffuseColor", color.x, color.y, color.z);
AiNodeSetInt(surface_shader, "specular1FresnelMode", 0);
AiNodeSetInt(surface_shader, "specular1Distribution", 1);
AiNodeSetFlt(surface_shader, "specular1Strength", 1.0f - metallic);
AiNodeSetRGB(surface_shader, "specular1Color", white.x, white.y, white.z);
AiNodeSetFlt(surface_shader, "specular1Roughness", roughness);
AiNodeSetFlt(surface_shader, "specular1Ior", 1.5f); // solving ior = (n-1)^2/(n+1)^2 for 0.04 gives 1.5
AiNodeSetRGB(surface_shader, "specular1Reflectivity", white.x, white.y, white.z);
AiNodeSetRGB(surface_shader, "specular1EdgeTint", white.x, white.y, white.z);

AiNodeSetInt(surface_shader, "specular2FresnelMode", 1);
AiNodeSetInt(surface_shader, "specular2Distribution", 1);
AiNodeSetFlt(surface_shader, "specular2Strength", metallic);
AiNodeSetRGB(surface_shader, "specular2Color", color.x, color.y, color.z);
AiNodeSetFlt(surface_shader, "specular2Roughness", roughness);
AiNodeSetRGB(surface_shader, "specular2Reflectivity", white.x, white.y, white.z);
AiNodeSetRGB(surface_shader, "specular2EdgeTint", white.x, white.y, white.z);
~~~~

Stingray VS Arnold: roughness = 0, metallicness = [0, 1]
![](images/res1.jpg)

Stingray VS Arnold: metallicness = 1, roughness = [0, 1]
![](images/res3.jpg)

Halfway through the validation process Arnold 5.0 got released and with it came the new [Standard Surface shader](https://support.solidangle.com/display/A5AFMUG/Standard+Surface) which is based on a Metalness/Roughness workflow. This allowed for a much simpler mapping:

~~~~
// "aiStandardSurface"
// ==============================================================================================
AiNodeSetFlt(standard_shader, "base", 1.f);
AiNodeSetRGB(standard_shader, "base_color", color.x, color.y, color.z);
AiNodeSetFlt(standard_shader, "diffuse_roughness", 0.f); // Use Lambert for diffuse

AiNodeSetFlt(standard_shader, "specular", 1.f);
AiNodeSetFlt(standard_shader, "specular_IOR", 1.5f); // solving ior = (n-1)^2/(n+1)^2 for 0.04 gives 1.5
AiNodeSetRGB(standard_shader, "specular_color", 1, 1, 1);
AiNodeSetFlt(standard_shader, "specular_roughness", roughness);
AiNodeSetFlt(standard_shader, "metalness", metallic);
~~~~

### Investigating material differences ###

The first thing we noticed is an excess in reflection intensity for reflections with large incident angles. Arnold supports [Light Path Expressions](https://support.solidangle.com/display/A5AFMUG/Introduction+to+Light+Path+Expressions) which made it very easy to compare and identify the term causing the differences. In this particular case we quickly identified that we had an energy conservation problem. Specifically the contribution from the Fresnel reflections was not removed from the diffuse contribution:

![](images/fix1.jpg)

Scenes with a lot of smooth reflective surfaces demonstrates the impact of this issue noticeably:

![](images/fixc.gif)
![](images/fixa.gif)
![](images/fixb.gif)

Another source of differences and confusion came from the tint of the Fresnel term for metallic surfaces. Different shaders I investigaed had different behaviors. Some tinted the Fresnel term with the base color while some others didn't:

![](images/metal3.jpg)

It wasn't clear to me how Fresnel's law of reflection applied to metals. I asked on Twitter what peoples thoughts were on this and got this simple and elegant [claim](https://twitter.com/BrookeHodgman/status/884532159331028992) made by Brooke Hodgman: *"Metalic reflections are coloured because their Fresnel is wavelength varying, but Fresnel still goes to 1 at 90deg for every wavelength"*. This convinced me instantly that indeed the correct thing to do was to use an un-tinted Fresnel contribution regardless of the metallicness of the material. I later found this [graph](https://en.wikipedia.org/wiki/Reflectance) which also confirmed this: 

![](images/reflectance.jpg)

For the Fresnel term we use a pre filtered Fresnel offset stored in a 2d lut (as proposed by Brian Karis in [Real Shading in Unreal Engine 4](http://blog.selfshadow.com/publications/s2013-shading-course/karis/s2013_pbs_epic_slides.pdf)). While results can diverge slightly from Arnold's Standard Surface Shader (see "the effect of metalness" from Zap Andeson's [Physical Material Whitepaper](https://www.dropbox.com/s/jt8dk65u14n2mi5/Physical%20Material%20-%20Whitepaper%20-%201.01.pdf?dl=0)), in most cases we get an edge tint that is pretty close:

![](images/metal4.jpg)

### Investigating light differences ###

With the brdf validated we could start looking into validating our physical lights. Stingray currently supports point, spot, and directional lights (with more to come). The main problem we discovered with our lights is that the attenuation function we use is a bit awkward. Specifically we attenuate by I/(d+1)^2 as opposed to I/d^2 (Where 'I' is the intensity of the light source and 'd' is the distance to the light source from the shaded point). The main reason behind this decision is to manage the overflow that could occur in the light accumulation buffer. Adding the +1 effectively clamps the maximum value intensity of the light as the intensity set for that light itself i.e. as 'd' approaches zero 'I' approaches the intensity set for that light (as opposed to infinity). Unfortunatly this decision also means we can't get physically [correct light falloffs](https://www.desmos.com/calculator/jydb51epow) in a scene:

![](images/graph.gif)

Even if we scale the intensity of the light to match the intensity for a certain distance (say 1m) we still have a different falloff curve than the physically correct attenuation. It's not too bad in a game context, but in the architectural world this is a bigger issue:

![](images/fix-int5.gif)

This issue will be fixed in Stingray 1.10. Using I/(d+e)^2 (where 'e' is 1/max_value along) with an EV shift up and down while writting and reading from the accumulation buffer as described by [Nathan Reed](http://www.reedbeta.com/blog/artist-friendly-hdr-with-exposure-values/) is a good step forward.

Finally we were also able to validate our ies profile parser/shader and our color temperatures behaved as expected:

![](images/ies2.gif)

### Results and final thoughts ###

![](images/comp-01.gif)
![](images/comp-03.gif)

Integrating a high quality offline renderer like Arnold has proven invaluable in the process of validating our lights in Stingray. A similar validation process could be applicable to many other aspects of our rendering pipeline (antialiasing, refractive materials, fur, hair, post-effects, volumetrics, ect)

I also think that it can be a very powerful tool for content creators to build intuition on the impact of indirect lighting in a particular scene. For example in a simple level like this, adding a diffuse plane dramatically changes the lighting on the Buddha statue:

![](images/diffuse.gif)

The next step is now to compare our results with photographs gathered from a controlled environments. To be continued...