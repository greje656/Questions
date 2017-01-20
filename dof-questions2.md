The "scene":
![](https://github.com/greje656/Questions/blob/master/images/scene3.jpg)

The different layers:
![](https://github.com/greje656/Questions/blob/master/images/layers.gif)

I'm confused about weather or not the background samples are weighted correctly?
![](https://github.com/greje656/Questions/blob/master/images/confusion.jpg)

Looking at the slides, it's almost is if any pixel's classified as 100% background should use a coc defined as:
lerp(tileMaxCoc, currentPixel'sCoC, DepthCmp2(currentPixel'sDepth, closestTileDepth));
I'm not sure!? Or maybe this is some kind of debug view and not the background layer "as is"?