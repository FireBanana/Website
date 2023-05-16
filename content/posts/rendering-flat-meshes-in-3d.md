---
title: "Rendering Meshes in a Plane With Ordering"
date: 2023-01-30T18:35:18+05:00
draft: true
---

I recently was tasked with building a system to allow 3D meshes to be rendered on a single plane without any sort of z-fighting. This seemed like a simple, yet interesting topic to mess around with, but I'd have some constraints that I'd be working with. Unity is what I'd be working with, so I was at the mercy of the low-level API it offered and how far it would take me. One of the first steps was to upgrade the Unity project to URP, so that I could have access to Unity's Scriptable Render Pipeline. If you're on the built-in pipeline, you could possibly get similar results using just the `Camera` and `CommandBuffer` classes.

The scenes I was working on had the usual world items and an abundance of world-space UI, with some occasional meshes that were not flat. These could be separate Unity canvases, meshes, renderers, you name it. Initially, my mind was set on depth offset, commonly known as `glPolygonOffset` in OpenGL and `vkCmdSetDepthBias` in Vulkan. This is conveniently exposed in ShaderLab using the `Offset` command. These have two parameters, namely <i>factor</i> and <i>units</i>. Factor is the constant value that the vertex depth is offset by, and units is an implementation specific slope value based on the camera's frustum. In simple terms, this function 'fakes' the depth of an object, and thus can be used to avoid Z-fighting while depth testing.

[image for polygonoffset]

This might look like the perfect approach but, as the eagle-eyed readers amongst you would have noticed, we'd still be writing to and reading from the depth buffer. Since you can stick an infinite amount of meshes onto said plane, the depth was incremented exponentially and many objects in front were occluded since they never passed the depth test.
This attempt quickly went down the drain. I needed something that wouldn't depend upon the depth buffer, so what options did I have?

Surprisingly, I didn't have to pressure my already steaming head much. I had recently read about the internal workings of Quake by Mike Abrash where he mentioned the use of the Painter's Algorithm, a fancy way of saying "to render from back to front". This seemed like the perfect approach; items could be ordered or sorted manually and the depth would not be affected. Alright, the mist around this matter seems to have thinned out, but how should I start?

Fortunately, URP has a helpful system called `Render Feature` which allows you to inject your own passes into the pipeline, for all cameras rendering in a scene. Thats where I decided to pool my effort into. First I tried to use the rendering API provided by the pipeline to set up a separate pass, but that had its weaknesses. Heres a sample of the couple of functions it uses to render inside the `Execute()` override.

```
        SortingSettings sortingSettings = new SortingSettings(camera);
        sortingSettings.criteria = SortingCriteria.All;

        CullResults = renderContext.Cull(ref cullingParameters);

        DrawingSettings drawSettings = new DrawingSettings(new ShaderTagId("SRPDefaultUnlit"), sortingSettings);
        drawSettings.enableDynamicBatching = false;
        drawSettings.enableInstancing = true;
        drawSettings.sortingSettings = sortingSettings;
        drawSettings.perObjectData = 0;
        FilteringSettings filterSettings = new FilteringSettings(RenderQueueRange.opaque);
        renderContext.DrawRenderers(CullResults, ref drawSettings, ref filterSettings);
```

As you might see, you don't get a lot of flexibility with this approach. You can essentially change some states with `RenderStateBlock` but when it comes to ordering, you're at the mercy of the engine. Therefore, I needed to make adjustments.

Enter the `CommandBuffer`, or Unity's `CommandBuffer` API to be more exact. 