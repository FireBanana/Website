---
title: "Ordering Renderable Meshes in a Flat Hierarchy in 3D Space"
date: 2023-06-13T18:35:18+05:00
draft: false
---

I recently was tasked with building a system to allow 3D meshes to be rendered on a single plane without any sort of z-fighting. This seemed like a simple, yet interesting topic to mess around with, but I'd have some constraints that I'd be working with. Unity is what I'd be working on, so I was at the mercy of the low-level API it offered and how far it would take me. One of the first steps was to upgrade the Unity project to URP, so that I could have access to Unity's Scriptable Render Pipeline. If you're on the built-in pipeline, you could possibly get similar results using just the `Camera` and `CommandBuffer` classes.

The scenes I was working on had the usual world items and an abundance of world-space UI, and was targeted specifically for VR and mobile hardware. These could be separate Unity canvases, meshes, renderers, you name it. Initially, my mind was set on utilizing depth offset, commonly known as `glPolygonOffset` in OpenGL and `vkCmdSetDepthBias` in Vulkan. This is conveniently exposed in ShaderLab using the `Offset` command. These have two parameters, namely <i>factor</i> and <i>units</i>. Factor is the constant value that the vertex depth is offset by, and units is an implementation specific slope value based on the camera's frustum. In simple terms, this function 'fakes' the depth of an object, and thus can be used to avoid Z-fighting while depth testing.

![offset](/images/flat-mesh/1.png)

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

Multiple instances of these boards can exist in the scene, each holding an infinite amount of items. These boards also move in realtime, so in concept, the order of the stickied items should be updated accordingly. Sorting these items on the CPU every frame would be disastrous on mobile hardware, so we'll see how to delegate this task to the GPU soon.

I tried a couple of approaches initially that did not work well. First, as I have mentioned, polygon offset had poor results when the item count started amping up in the scene. The second approach was to enable multiple render targets and write some data to an `R32G32_SFLOAT` target, where `R` would be the colorID of items on a plane (that is, they would share a similar color), and `G` would hold depth information. Furthermore, I added a double-buffered approach for the MRTs, since you can't read and resolve a single target on the same frame. Although performance was decent on the Quest 2, the depth precision was quite off which I wasn't able to get to the bottom off, so I decided to try a better, simpler approach.

The current approach follows a much simpler concept and render loop, with just a small additional sort on the CPU side, which is:

```C#  
_sortedList = _renderableItemList.Select(x => x.Value)
        .OrderBy(x => x.PlaneIndex).ThenBy(x => x.Index).ToList();
```

The `_renderableItemList` holds each item that was attached to the plane in a dictionary, where `PlaneIndex` is a unique index of a plane in the level and `Index` is the order of the item on that plane. This runs whenever an item is added and it groups all items together by the plane's index first and then by its own index on the plane, neatly sorting the items.

So since the items are sorted, why not render them in order of the list? We can't, because the list has no information on where each plane is in the level, and in order to do that on the CPU would require sorting each plane every frame, which absolutely tanked the performance. There was a far easier approach where you can utilize the default depth target to do all the work for you. This is what the current render loop looks like:

```C#
var buffer = new CommandBuffer();

var rendererItemList = GetSortedList();

if (rendererItemList is null || rendererItemList.Count == 0) return;

buffer.SetRenderTarget(renderingData.cameraData.renderer.cameraColorTarget,
        renderingData.cameraData.renderer.cameraDepthTarget, 0, CubemapFace.Unknown, -1);

int counter = 0;
int currentId = rendererItemList[0].PlaneIndex;

//Data is grouped by planeindex and then index within
while( counter < rendererItemList.Count && currentId <= rendererItemList[^1].PlaneIndex )
{
        int counterState = counter;
        
        // Will render into default camera target
        while (counter < rendererItemList.Count &&
                currentId == rendererItemList[counter].PlaneIndex)
        {
                var item = rendererItemList[counter];
                
                ColorPass(ref item, buffer);
                counter++;
        }

        // reset state
        counter = counterState;

        // Will render into default depth target
        while (counter < rendererItemList.Count &&
                currentId == rendererItemList[counter].PlaneIndex)
        {
                var item = rendererItemList[counter];
                
                DepthPass(ref item, buffer);
                
                counter++;
        }

        currentId++;
}

context.ExecuteCommandBuffer(buffer);
```

A very simple and straightforward render pass. Everything of importance happens inside of the `while` loop, which essentially runs twice over each item, first rendering the color, and secondly the depth. Each item on a specific plane renders its color first, with the normal LEqual depth testing, and depth writing turned off. The depth pass turns of all the channels in the color mask, and writes the depth as it would have normally (I just set up a multi_compile inside the shader to switch between). Any subsequent renders will now test against this written depth, while not depth testing against the items on the same plane as themselves.

This is everything needed to render meshes flat, without any Z fighting.

{{< youtube WsY-r_EmyUY >}}