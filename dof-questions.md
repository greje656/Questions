####Question 1)
Is this really the raw results of what the foreground layer should look like? Or is this some kind of visualization mode of the foreground weights combined with the foreground result?

![](https://github.com/greje656/Questions/blob/master/images/foreground-original.png)

The results I seem to get for the foreground layer is more like (note this is using a "background range" of 2.5m so ~100in).

foreground.rgb/foreground.a:
![](https://github.com/greje656/Questions/blob/master/images/foreground.jpg)
Notice how I don't get any samples pixels whose weight is low? I'm wondering if I miss-understood something crucial here (i.e. no black pixels like you're result). 

The weight of the samples looks like this:
![](https://github.com/greje656/Questions/blob/master/images/foreground-weights.jpg)

Similarly the background looks like this:

background.rgb/background.a
![](https://github.com/greje656/Questions/blob/master/images/background.jpg)

And the background weights:
![](https://github.com/greje656/Questions/blob/master/images/background-weights.jpg)

The normalized alpha value I get looks like:
![](https://github.com/greje656/Questions/blob/master/images/alpha.jpg)

And finally "lerp(background/background.a, foreground/foreground.a, alpha)":
![](https://github.com/greje656/Questions/blob/master/images/results.jpg)

I'm wondering maybe the foreground/background layers should be weighted like this?

foreground.rgb/SAMPLE_COUNT:
![](https://github.com/greje656/Questions/blob/master/images/foreground2.jpg)

background.rgb/SAMPLE_COUNT:
![](https://github.com/greje656/Questions/blob/master/images/background2.jpg)

####Question 2)
Should the maxCoCMinDepth tiles be sampled using linear interpolation? Or point sampling? Intuitively it feels like it should be point sampling but I see artifacts at the edges of the tiles if I do. Note that with tiles of 20x20 pixels I'm making the assumption that the maximum kernel size allowed should be 10x10px. Maybe that is a false assumption?

Linear (currently what I'm using):
![](https://github.com/greje656/Questions/blob/master/images/tile-min-depth-linear.jpg)

Point:
![](https://github.com/greje656/Questions/blob/master/images/tile-min-depth-point.jpg)

Point sampling artifacts:
![](https://github.com/greje656/Questions/blob/master/images/artifacts.jpg)

####Questions 3)
Finally (this one might be really dumb!). Does this need to differentiate samples that are behind\infront of the camera focus point? i.e. does this need signed CoCs? I think it does but that's not something that's 100% clear to me. Currently I've implemented a solution that uses abs(coc), but this doesn't work for focused objects on top of unfocused:
![](https://github.com/greje656/Questions/blob/master/images/results-bad.jpg)
It's probably that we DO need to use a signed circle of confusions but I just wanted to confirm that I'm not missing something obvious here (I'm sure I am)

####Code used to generate images
	#define NUM_SAMPLES 5
	#define COC_SIZE_IN_PIXELS 10
	#define BACKGROUND_RANGE 2.5

	struct PresortParams {
		float coc;
		float backgroundWeight;
		float foregroundWeight;
	};
	
	float2 DepthCmp2(float depth, float closestTileDepth){
		float d = saturate((depth - closestTileDepth)/BACKGROUND_RANGE);
		float2 depthCmp;
		depthCmp.x = smoothstep( 0.0, 1.0, d ); // Background
		depthCmp.y = 1.0 - depthCmp.x; // Foreground
		return depthCmp;
	}
	
	float SampleAlpha(float sampleCoc) {
		const float DOF_SINGLE_PIXEL_RADIUS = length(float2(0.5, 0.5));
		return min(
			rcp(PI * sampleCoc * sampleCoc),
			rcp(PI * DOF_SINGLE_PIXEL_RADIUS * DOF_SINGLE_PIXEL_RADIUS)
		);
	}
	
	PresortParams GetPresortParams(float sample_coc, float sample_depth, float closestTileDepth){
		PresortParams presort_params;
	
		presort_params.coc = sample_coc;
		presort_params.backgroundWeight = SampleAlpha(sample_coc) * DepthCmp2(sample_depth, closestTileDepth).x;
		presort_params.foregroundWeight = SampleAlpha(sample_coc) * DepthCmp2(sample_depth, closestTileDepth).y;
	
		return presort_params;
	}
	
	float4 ps_main(PS_INPUT input) : SV_TARGET0 {
	
		float tileMaxCoc = TEX2DLOD(input_texture3, input.uv, 0).g * COC_SIZE_IN_PIXELS;
	
		float SAMPLE_COUNT = 0.0;
		float4 background = 0;
		float4 foreground = 0;
	
		for (int x = -NUM_SAMPLES; x <= NUM_SAMPLES; ++x) {
			for (int y = -NUM_SAMPLES; y <= NUM_SAMPLES; ++y) {
	
				float2 samplePos = get_sampling_pos(x, y);
				float2 sampleUV = input.uv + samplePos/back_buffer_size * tileMaxCoc;
	
				float3 sampleColor = TEX2DLOD(input_texture0, sampleUV, 0).rgb;
				float sampleCoc = decode_coc(TEX2DLOD(input_texture1, sampleUV, 0).r) * COC_SIZE_IN_PIXELS;
				float sampleDepth = linearize_depth(TEX2DLOD(input_texture2, sampleUV, 0).r);
				float closestTileDepth = TEX2DLOD(input_texture3, input.uv, 0).r;
	
				PresortParams samplePresortParams = GetPresortParams(sampleCoc, sampleDepth, closestTileDepth);
	
				background += samplePresortParams.backgroundWeight * float4(sampleColor, 1.f);
				foreground += samplePresortParams.foregroundWeight * float4(sampleColor, 1.f);
	
				SAMPLE_COUNT += 1;
			}
		}
		
		float alpha = saturate( 2.0 * ( 1.0 / SAMPLE_COUNT ) * ( 1.0 / SampleAlpha(tileMaxCoc)) * foreground.a);
		float3 finalColor = lerp(background/background.a, foreground/foreground.a, alpha);
	
		return float4(finalColor, 1);
	}