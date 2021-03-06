# Physical Cameras in Stingray (Part 1) #
This is a quick blog to share some of the progress [Olivier Dionne](https://twitter.com/olivier_dionne) and I made lately with Physical Cameras in Stingray. Our goal of implementing a solid physically based pipeline has always been split in three phases. First we [validated our standard material](http://bitsquid.blogspot.ca/2017/07/validating-materials-and-lights-in.html). We then added physical lights. And now we are wrapping it up with a physical camera.

We define a physical camera as an entity controlled by the same parameters a real world camera would use. These parameters are split into two groups which corresponds to the two main parts of a camera. The camera _body_ is defined by it's sensor size, iso sensitivity, and a range of available shutter speeds. The camera _lens_ is defined by it's focal length, focus range, and range of aperture diameters. Setting all of these parameters should expose the incoming light the same way a real world camera would.

### Stingray Representation ###
Just like our physical light, our camera is expressed as an entity with a bunch of components. The main two components being the Camera Body and the Camera Lens. We then have a transform component and a camera component which together represents the view projection matrix of the camera. After that we have a list of shading environment components which we deem relevant to be controlled by a physical camera (all post effects relevent to a camera). The state of these shading environment components is controled through a script component called the "Physical Camera Properties Mapper" (more on this later). Here is a glimpse of what the Physical Camera entity may look like (wip):

![](images/cameras/res11.jpg)

So while there are a lot of components that belongs to a physical camera, the user is expected to interact mainly with the body and the lens components.

### Post Effects ###
A lot of our post effects are dedicated to simulate some sort of camera/lens artifact (DOF, motion blur, film grain, vignetting, bloom, chromatic aberation, ect). One thing we wanted was the ability for physical cameras to override the post processes defined in our global shading environments. We also wanted to let users easily opt out of the physically based mapping that occurred between a camera and it's corresponding post-effect. For example a physical camera will generate an accurate circle of confusion for the depth of field effect, but a user might be frustrated by the limitations imposed by a physically correct dof effect. In this case a user can opt out by simply deleting the "Depth Of Field" component from the camera entity.

It's nice to see how the expressiveness of the Stingray entity system is shaping up and how it enables us to build these complex entities without the need to change much of the engine.

### Properties Mapper ###
All of the mapping occurs in the properties mapper component which I mentioned earlier. This is simply a lua script that gets executed whenever any of the entity properties are edited.

The most important property we wanted to map was the exposure value. We wanted the f-stop, shutter speed, and ISO values to map to an exposure value which would simulate how a real camera sensor reacts to incoming light. Lucky for us this topic is very well covered by Sebastien Lagarde and Charles de Rousiers in their awesome awesome awesome [Moving Frostbite to Physically Based Rendering](https://seblagarde.files.wordpress.com/2015/07/course_notes_moving_frostbite_to_pbr_v32.pdf) document. The mapping basically boils down to:

~~~
local function compute_ev(aperture, shutter_time, iso)
	local ev_100 = log2((aperture * aperture * 100) / (shutter_time * iso))
	local max_luminance = 1.2 * math.pow(2, ev_100)
	return (1 / max_luminance)
end
~~~

The second property we were really interested in mapping is the field of view of the camera. Usually the horizontal FOV is calculated as _2 x atan(h/2f)_ where _h_ is the camera sensor's width and _f_ is the current focal length of the lens. This by itself gives a good approximation of the FOV of a lens, but as was pointed out by the [MGS5 & Fox Engine presentation](https://youtu.be/FQMbxzTUuSg?t=50m12s), the focus distance of the lens should also be considered when calculating the FOV from the camera properties.

Intuitively we though that the change in the FOV was caused by a change in the effective focal length of the lens. Adjusting the focus usually shifts a group of lenses up and down the optical axis of a camera lens. Our best guess was that this shift would increase or decrease the effective focal length of the camera lens. Using this idea we we're able to simulate the effect that changing the focus point has on the FOV of a camera:

[https://www.youtube.com/watch?v=KDwUi-vYYMQ](https://www.youtube.com/watch?v=KDwUi-vYYMQ&feature=youtu.be)

~~~
local function compute_fov(focal_length, film_back_height, focus)
	local normalized_focus = (focus - 0.38)/(5.0 - 0.38)
	local focal_length_offset = lerp(0.0, 1.0, normalized_focus)
	return 2.0 * math.atan(film_back_height/(2.0 * (focal_length + focal_length_offset)))
end
~~~


While this gave us plausible results in some cases, it does not map accurately to a real world camera with certain lens settings. For example we can choose a focal length offset that gives good FOV mapping for a zoom lens set to 24mm but incorrect FOV results when it's set to 70mm (see [video](https://www.youtube.com/watch?v=KDwUi-vYYMQ&feature=youtu.be) above). This area of lens optics is one we would like to explore more in the future. 

In the future we will map more camera properties to their corresponding post-effects. More on this in a follow up blog.

### Validating Results ###
To validate our mappings we designed a small, controlled environment room that we re-created in stingray. This idea was inspired by the "Conference Room" setup that was presented by Hideo Kojima in the [MGS5 & Fox Engine presentation](https://youtu.be/FQMbxzTUuSg?t=20m22s). We used our simplified, environment room to compare our rendered results with real world photographs.

Controlled Environment:
![](images/cameras/res4.jpg)

Stingray Equivalent:
![](images/cameras/res3.jpg)

Since there is no convenient way to adjust the white balancing in Stingray, we decided to white balance our camera data and use a pure white light in our Stingray scene. We also decided to compare the photos and renders in linear space in the hope to minimize the source of potential error.

White balancing our photographs:
![](images/cameras/res6.gif)

Our very first comparison we're disapointing:
![](images/cameras/res10.jpg)

We tracked down the difference in brightness to a problem with how we expressed our light intensity. We discovered that we made the mistake of using the specified lumen value of our lights as it's light intensity. The total luminous flux is expressed in lumens, but the luminous intensity (what the material shader is interested in) is actually the luminous flux per solid angle. So while we let the users enter the "intensity" of lights in lumens, we need to map this value to luminous intensity. The mapping is done as _lumens/2π(1-cos(½α))_ where  _α_ is the apex angle of the light. Lots of details can be found [here](https://www.compuphase.com/electronics/candela_lumen.htm). This works well for point lights and spot lights. In the future our directional lights will be assumed to be the sun or moon and will be expressed in lux, perhaps with a corresponding disk size.

With this fix in place we started getting more encouraging results:
![](images/cameras/res1.jpg)

![](images/cameras/res2.jpg)

There is lots left to do but this feels like a very good start to our physically based cameras. 