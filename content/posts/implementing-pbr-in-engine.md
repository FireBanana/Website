---
title: "Implementing PBR in-Engine"
date: 2023-06-07T10:27:13+05:00
draft: true
math: true
---

For the past few days I've been working on implementing simple PBR rendering for MiniEngine, I decided to follow Epic Game's design for a PBR renderer, [which can be found here](https://cdn2.unrealengine.com/Resources/files/2013SiggraphPresentationsNotes-26915738.pdf). This will only look at the BRDF, and for more details on the derivation, I'd recommend the book [Real Time Rendering](https://www.realtimerendering.com/).

I downloaded a free model that I'll be using, and I'll only be using the albedo texture for the most basic results. To start off, lets outline the theory that'll be used. 

$x = {-b \pm \sqrt{b^2-4ac} \over 2a}$