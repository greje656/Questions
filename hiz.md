# Notes on ssr HIZ tracing

The following is small gathering of notes and findings that we made through out the implementation of Hi-Z tracing in screen space for ssr in Stingray. I've recently heard a few claims regarding Hi-Z which motivated me to share some notes on the topic. Note that I also wrote about how we reproject reflections in a previous entry which might be of interest.

The original implementation was basically an implementation of the Hi-Z Screen-Space tracing described in GPU-Pro 5 by Yasin Uludag. The very first results we got looked something like this:
![](https://github.com/greje656/Questions/blob/master/images/ssr1.jpg)
![](https://github.com/greje656/Questions/blob/master/images/ssr2.jpg)
