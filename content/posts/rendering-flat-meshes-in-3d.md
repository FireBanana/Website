---
title: "Ordering Renderable Meshes in a Flat Hierarchy in 3D Space"
date: 2023-01-30T18:35:18+05:00
draft: true
---

I recently was tasked with building a system to allow 3D meshes to be rendered on a single plane without any sort of z-fighting. This seemed like a simple, yet interesting topic to mess around with, but I'd have some constraints that I'd be working with. Unity is what I'd be working on, so I was at the mercy of the low-level API it offered and how far it would take me. One of the first steps was to upgrade the Unity project to URP, so that I could have access to Unity's Scriptable Render Pipeline. If you're on the built-in pipeline, you could possibly get similar results using just the `Camera` and `CommandBuffer` classes.

The scenes I was working on had the usual world items and an abundance of world-space UI, and was targeted specifically for VR and mobile hardware. These could be separate Unity canvases, meshes, renderers, you name it. Initially, my mind was set on utilizing depth offset, commonly known as `glPolygonOffset` in OpenGL and `vkCmdSetDepthBias` in Vulkan. This is conveniently exposed in ShaderLab using the `Offset` command. These have two parameters, namely <i>factor</i> and <i>units</i>. Factor is the constant value that the vertex depth is offset by, and units is an implementation specific slope value based on the camera's frustum. In simple terms, this function 'fakes' the depth of an object, and thus can be used to avoid Z-fighting while depth testing.

[image for polygonoffset]

This might look like the perfect approach but, as the eagle-eyed readers amongst you would have noticed, we'd still be writing to and reading from the depth buffer. Since you can stick an infinite amount of meshes onto said plane, the depth was incremented exponentially and many objects in front were occluded since they never passed the depth test. In addition, the depth buffer only had so much precision that the increment value between items would start falling off. This attempt quickly went down the drain. I needed something that wouldn't depend upon the depth buffer, so what options did I have?

Surprisingly, I didn't have to pressure my already steaming head much. I had recently read about the internal workings of Quake by Mike Abrash where he mentioned the use of the Painter's Algorithm, a fancy way of saying "to render from back to front". This seemed like the perfect approach; items could be ordered or sorted manually and the depth would not be affected.

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

As you might see, you don't get a lot of flexibility with this approach when it comes to custom rendering. You can essentially change some states with `RenderStateBlock` but when it comes to ordering, you're at the mercy of the engine. Therefore, I needed to make adjustments.

Before I go any deeper into the technical implementation, I'd like to go over the concept of how this process works.

[Image of multiple pinboards and items]

Multiple instances of these boards can exist in the scene, each holding an infinite amount of items. These boards also move in realtime, so in concept, the order of the stickied items should be updated accordingly. Sorting these items on the CPU every frame would be disastrous on mobile hardware, so we'll see how to delegate this task to the GPU soon.



Unity allows sending graphics commands via its `CommandBuffer` API. This has several helpful methods such as `DrawMesh` which we'll be using to draw individual meshes, as well as providing the ability to switch targets.