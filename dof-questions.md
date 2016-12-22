Jorge Jimenez

Hi Jorge,

First of all, I'm a huge fan! I wanted to get that out of the way :P Huge huge fan of all the filtering work you've done in the last few years (aa, ssss, dof/motion blur)

I've recently been trying to implement the Depth of Field solution you presented at Siggraph (NEXT GENERATION POST PROCESSING IN CALL OF DUTY: ADVANCED WARFARE, SIGGRAPH 2014) and was wondering if you'd be ok with answering a few questions. Will try to keep them simple (kind of yes or no questions)

Question 1)
Is this really the raw results of what the foreground layer should look like? Or is this some kind of visualization mode of the foreground weights combined with the foreground result?

![](https://github.com/greje656/Questions/blob/master/images/foreground-original.png)

The results I seem to get for the foreground layer is more like (note this is using a "background range" of 2.5m so ~100in):
![](https://github.com/greje656/Questions/blob/master/images/foreground.jpg)

The weight of the samples:
![](https://github.com/greje656/Questions/blob/master/images/foreground-weights.jpg)

Similarly the background looks like:
![](https://github.com/greje656/Questions/blob/master/images/background.jpg)

And weights:
![](https://github.com/greje656/Questions/blob/master/images/background-weights.jpg)

Question 2)
Should the maxCoCMinDepth tiles be sampled using a linear interpolation? Or point sampling? Intuitively it feels like it should be point sampling but I see artifacts at the edges of the tiles if I do. Note that with tiles of 20x20 pixels I'm making the assumption that the maximum kernel size allowed should be 10x10px. Maybe that is a false assumption.

Point: ![](https://github.com/greje656/Questions/blob/master/images/tile-min-depth-point.jpg)

Linear (currently what I'm using): ![](https://github.com/greje656/Questions/blob/master/images/tile-min-depth-linear.jpg)
