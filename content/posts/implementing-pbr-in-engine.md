---
title: "Implementing PBR in-Engine"
date: 2023-06-07T10:27:13+05:00
draft: true
math: true
---

For the past few days I've been working on implementing simple PBR rendering for MiniEngine, I decided to follow Epic Game's design for a PBR renderer, [which can be found here](https://cdn2.unrealengine.com/Resources/files/2013SiggraphPresentationsNotes-26915738.pdf). This will only look at the BRDF, and for more details on the derivation, I'd recommend the book [Real Time Rendering](https://www.realtimerendering.com/).

I downloaded a free model that I'll be using, and I'll only be using the albedo texture for the most basic results. To start off, lets outline the theory that'll be used. This is only the initial implementation, further updates will be outlined in newer posts.

We'll be using the standard Cook-Torrence microfacet BRDF for specular, and a Lambertian approximation for surface diffuse. There are other approximations available as well but these two seem to be the most popular with good results. 

The albedo color is usually referred to as subsurface albedo. The value lies between 0 and 1, where 0 signifies that all light is absorbed and 1 where no light is absorbed. The Lambertian diffuse model is a smooth-surface model, and there a number of other models used in real-time rendering, such as a more accurate rendition by Ashikhmin and Shirley and Disney's diffuse model although they are quite compute intensive. The Lambertian approximation used is simply given by:

$$  f(l, v) = {c \over π} $$

where c is the diffuse albedo. The π term is a result of integrating a cosing factor across the surface hemisphere, which results in $$ 1 \over π $$