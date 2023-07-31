---
title: "Visualizing Why Normals Need Tangent Space"
date: 2023-07-31T11:53:21+05:00
draft: false
---

This is not much of an article, but more of a visualization of why the TBN matrix is utilized, or simply the normal vector needs to be altered to respect the current vertex position. Lets take a look at an example normal map:

[image on normal map]

Since the map has a bluish tinge, you can analyze that the blue channel has a non-zero value throughout. This is because the normals are pointing out of the screen and towards you, in the Z (blue) direction. Now when we use this in our shader to display normals, the normal direction at the vertices will look something like:

![normal](/images/tangent/normal-1.png)

The direction that will be sampled from the image will essentially always be pointing in the same world z direction. If you have a flat plane whose normal always faces the z direction then you're good to go, but if you something thats even slightly more complex, you'll need to transform the point into tangent space.

The concept behind tangent space is simple as visualized below. 

![normal](/images/tangent/normal-2.png)

Apologies for the poor drawing, but without going into detail, it simply orients the reference frame to match the triangles orientation in object space. So the normals in the image that were pointing towards the world z coordinates, now point towards the face normals to give something like:

![normal](/images/tangent/normal-3.png)

Now lighting calculations will pickup the correct direction for the normals.