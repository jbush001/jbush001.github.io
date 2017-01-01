---
layout: post
title: Quake Lightmaps
date: '2015-06-11T09:30:00.002-07:00'
author: Jeff
tags:
- 3d rendering
- lighting
modified_time: '2016-02-03T22:37:05.970-08:00'
thumbnail: https://2.bp.blogspot.com/-FWF21HtEsa4/VXiqaXZI-KI/AAAAAAAAB-c/AFvENpfTUeg/s72-c/texture-only.png
blogger_id: tag:blogger.com,1999:blog-5853447763770338628.post-4360517119255370373
blogger_orig_url: http://latchup.blogspot.com/2015/06/quake-lightmaps.html
---

I added support for lightmaps to the Quake renderer I discussed in the [last
post]({{ site.baseurl }}{% post_url 2015-06-05-not-so-fast %}).
[Lightmaps](http://www.jagregory.com/abrash-black-book/#chapter-68-quakes-
lighting-model) were a big innovation when they first appeared in Quake. By
precalculating light and shadow, they allowed much more realistic scenes than
would have been possible to compute in realtime. Lightmaps are still [used](ht
tp://docs.unity3d.com/460/Documentation/Manual/LightmappingInDepth.html) in
many games.

The tricky part of implementing this was that the Quake BSP format doesn't
store redundant information. Although it contains the file offsets for the
lightmaps, it doesn't have the dimensions of each one. The renderer must
compute these by walking the vertices of each polygon face, converting the
coordinates to texture space, and finding the bounding rectangle. As with the
texture data, I packed these into a single [atlas
texture](http://en.wikipedia.org/wiki/Texture_atlas) and stored the
coordinates as vertex attributes.

Here's a rendered frame with only texture mapping, no lighting:

![]({{ site.url }}/assets/2015-06-11-image-0000.png)

There is a relatively small amount of texture data for each level. Levels
reuse textures all over the place, repeating them across each surface. You can
see this with the floor tiles in the image above, for example. However,
lightmaps can't be reused like this. Every polygon in the level needs a
dedicated lightmap. To reduce the amount of data required, lightmaps are
stored at low resolution: one sample every 16 pixels in texture space. Also,
each sample is 8-bits. The next image shows just the lightmaps rendered for
the scene above.

![]({{ site.url }}/assets/2015-06-11-image-0001.png)

It's harder to verify lightmaps are working correctly than it was with
textures, because it's not as obvious when it is wrong. This looks mostly
correct. There is a shadow on the steps on the left side, and shadows cast by
the ductwork above. I can see lighter areas around the lights. There is this
odd light strip above the opening on the right side.  I think I have an off-
by-one in the lightmap atlas setup. Also, the opening on the right side
doesn't seem to show the light shining on the lower surface. There is a
flickering light there in the game, which the Quake engine renders
differently. I haven't implemented that, which I suspect is why the default
lightmap value doesn't look right.

Here are the same lightmaps with bilinear filtering enabled on the texture
sampler, which removes the sharp edges.

![]({{ site.url }}/assets/2015-06-11-image-0002.png)

Now that I have the lightmap data available, I need to multiply it by each
pixel in the texture. When id added hardware acceleration to Quake, desktop
GPUs didn't support programmable shaders, so they needed some machinations to
support lighting. Since my rendering engine does support shaders, the
implementation is simpler. The pixel shader is below, with new code **bolded.
**As described in previous posts, this works on 16 pixels at a time.

{% highlight c++ %}
{% raw %}
void shadePixels(vecf16_t outColor[4],
    const vecf16_t inParams[16],
    const void *_castToUniforms,
    const Texture * const sampler[kMaxActiveTextures],
    unsigned short mask) const override
{
    vecf16_t atlasU = wrappedAtlasCoord(
        inParams[kParamTextureU - 4],
        inParams[kParamAtlasLeft - 4],
        inParams[kParamAtlasWidth - 4]);

    vecf16_t atlasV = wrappedAtlasCoord(
        inParams[kParamTextureV - 4],
        inParams[kParamAtlasTop - 4],
        inParams[kParamAtlasHeight - 4]);

    sampler[0]->readPixels(atlasU, atlasV, mask, outColor);

+    vecf16_t lightmapValue[4];
+    sampler[1]->readPixels(
+        inParams[kParamLightmapU - 4],
+        inParams[kParamLightmapV - 4],
+        mask,
+        lightmapValue);
+
+    vecf16_t intensity = lightmapValue[0] + splatf(0.4);
+    outColor[kColorR] *= intensity;
+    outColor[kColorG] *= intensity;
+    outColor[kColorB] *= intensity;
}
{% endraw %}
{% endhighlight %}

In the code above, I've added two new interpolated parameters: the lightmap U
and V coordinates.  I don't need to worry about repeating lightmaps as I do
for the textures. My texture sampler only supports 32-bit textures now, so I
use the lowest channel for the intensity and ignore the others.  I add a
constant ambient value to the lightmap sample--it was too dark otherwise--and
multiply that intensity by the texel color.

Here is the original scene with the lightmap applied: The effect is subtle,
but looks much more realistic.

![]({{ site.url }}/assets/2015-06-11-image-0003.png)

*The four white dots on the right are a gap where the triangles don't fit
together tightly. This is probably an issue in my rasterizer.*

It also requires more instructions: about 30 million to render the first frame
compared to 22 million without lightmaps.  Two of the biggest contributors to
the profile in the previous post were parameter interpolation and texture
sampling, both of which lightmaps require. I could improve this by enabling an
8-bit texture format for the lightmap, which would remove the redundant
computations for the unused color channels.
