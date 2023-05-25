---
title: "Simple Rounded Corners"
date: 2023-05-25T12:04:38+05:00
draft: false
---

In most modern designs today, rounded corners are a staple in most UIs. I haven't seen a lot of simple explanations for it, so I'll share a simple approach to create one. The code is being run on ShaderToy, so any uniforms/vertex inputs should be converted for your engine of choice.

The concept is pretty simple. As you may have seen, rounded corners are more or less parts of a circle towards the edge of a quad. We can represent them as follows:

![Image](/images/rounded-corner/corner-1.png)

Basic enough. Now we need to fill in the areas that form the rest of the image. For this, we need only two extra primitives, both of them rectangles:

![Image](/images/rounded-corner/corner-2.png)

That's it. This is all you need to create rounded corners in a shader. Let's take a look at what the GLSL for this looks like:

```glsl
void mainImage( out vec4 fragColor, in vec2 fragCoord )
{
    vec2 uv = fragCoord/iResolution.xy;
    
    float mask = 0.;
    const float radius = 0.2;

    mask += step(radius, uv.x) * (1. - step(1. - radius, uv.x));
    mask += step(radius, uv.y) * (1. - step(1. - radius, uv.y));
    mask += 1. - step(radius, length(vec2(radius, radius) - uv)); 
    mask += 1. - step(radius, length(vec2(1. - radius, radius) - uv)); 
    mask += 1. - step(radius, length(vec2(1. - radius, 1. - radius) - uv)); 
    mask += 1. - step(radius, length(vec2(radius, 1. - radius) - uv)); 

    // Output to screen
    fragColor = vec4(mask,mask,mask,1.0);
}

```

We only need to work with a single-channel mask here. The first two lines after initializing the radius create the two rectangles that we discussed earlier. The following four lines create the four circles on the corners. All the distances are dependent on one value, which is the radius of the circle.

This would work great if your quad was always square-shaped. But in most cases, it won't be. Here's how it looks so far on ShaderToy:

![Image](/images/rounded-corner/corner-3.png)

The corners are stretched as the width is larger than the height. To mitigate this, we can use the aspect ratio of the quad to alter this transformation. <i>Note: I will only assume that the width is larger than the height. The opposite case is an exercise for you.</i>

```glsl
void mainImage( out vec4 fragColor, in vec2 fragCoord )
{
    float aspect = iResolution.x / iResolution.y;
    vec2 uv = fragCoord/iResolution.xy ;
    
    float mask = 0.;
    float radius = 0.2;
    vec2 rUv = (uv * vec2(aspect, 1.));

    mask += step(radius, uv.x * aspect) * (1. - step((1. * aspect) - radius, uv.x * aspect));
    mask += step(radius, uv.y) * (1. - step(1. - radius, uv.y));
    mask += 1. - step(radius, length(vec2(radius, radius) - rUv)); 
    mask += 1. - step(radius, length(vec2((1. * aspect) - radius, radius) - rUv)); 
    mask += 1. - step(radius, length(vec2((1. * aspect) - radius, 1. - radius) - rUv)); 
    mask += 1. - step(radius, length(vec2(radius, 1. - radius) - rUv)); 

    // Output to screen
    fragColor = vec4(mask,mask,mask,1.0);
}
```

We will only need to alter the scale on one axis, which in this case would be the x axis (as we're only assuming a larger width). For the rectangles, we change the first line to scale the `uv.x` down to match the `uv.y`, and scaling down the `1.` in `1. - radius` as we're sampling a smaller uv size.

For the circles, the `uv` when calculating length is also scaled; saved as `rUv`. The `1. - radius` is similarly scaled, and only on the x axis in both cases. This is the final image:

![Image](/images/rounded-corner/corner-4.png)

Now regardless of the width (as long as its larger than height), we'll get proper rounded corners.