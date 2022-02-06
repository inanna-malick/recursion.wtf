+++
title = "Infinity Mirror HYPERCRYSTAL"
date = "2022-02-05"
author = "Inanna Malick"
authorTwitter = "inanna_malick"
tags = ["infinity", "mirrors", "meatspace"]
keywords = ["art", "infinity", "mirrors", "meatspace", "laser cut"]
showFullContent = false
+++

## Infinity Mirror HYPERCRYSTAL

{{< image src="/img/infinity_mirror_hypercrystal/powered_on_front_header.jpg" alt="hypercrystal powered on" position="center" style="border-radius: 8px;" >}}

This object is the result of a series of algorithmic/generative art techniques I've been working on, on and off, for the last decade. I was really excited to see my latest build get a lot of attention on [twitter](https://twitter.com/inanna_malick/status/1488927590207275010). I've written up a build log with details on how I generated and constructed this object:

<!--more--> 

<style>.embed-container { position: relative; padding-bottom: 56.25%; height: 0; overflow: hidden; max-width: 100%; } .embed-container iframe, .embed-container object, .embed-container embed { position: absolute; top: 0; left: 0; width: 100%; height: 100%; }</style><div class='embed-container'><iframe src='https://player.vimeo.com/video/674008248' frameborder='0' webkitAllowFullScreen mozallowfullscreen allowFullScreen></iframe></div>

## WHAT

Laser cut generative art. A non-platonic non-symmetrical solid with eight faces, each cut from 30% transparent plastic with each pair of connected faces held together by a plywood joinery system I developed.

## WHAT?

I used a generative algorithm of my own making to generate this geometry: plywood connective pieces, mirrored crystal faces, cardboard tray.

{{< image src="/img/infinity_mirror_hypercrystal/components.jpg" alt="hypercrystal components" position="center" style="border-radius: 8px;" >}}

I combined these pieces with a cardboard cigar box gifted to me by my sister (an amazing artist, check out her [portfolio](https://www.seankinsky.com/)). This box holds the electronics and the base of the LED lightbulbs that illuminate the piece, which allows me to maintain the illusion of a crystal sitting on top of a box, with each perceived as a standalone object.

{{< image src="/img/infinity_mirror_hypercrystal/under_construction.jpg" alt="hypercrystal under construction" position="center" style="border-radius: 8px;" >}}

The infinity effect was achieved, but it was insufficiently eldritch. The white LED bulbs are too obviously modern technological artifacts. I needed to obscure their form somehow.

{{< image src="/img/infinity_mirror_hypercrystal/powered_white_light_only.jpg" alt="hypercrystal powered on but with white LEDs only" position="center" style="border-radius: 8px;" >}}

I used partially transparent dichroic tape to obtain a lovely glitch rainbow effect. The color of dichroic materials shifts based on the angle of view, which means that they look really cool when light is filtered through them.

{{< image src="/img/infinity_mirror_hypercrystal/interior_shot_dichroic_detail.jpg" alt="interior shot with detail of dichroic tape" position="center" style="border-radius: 8px;" >}}

## HOW

### 3D Model

I start by creating a very low poly 3D model. In this case, a crystal:

{{< image src="/img/infinity_mirror_hypercrystal/3D_model.png" alt="3D model of hypercrystal" position="center" style="border-radius: 8px;" >}}

Then, I use a tool I've built using rustlang to generate a series of 2D vector drawings of the various physical components that make up the object. These 2D drawings are cut out of sheets of plywood and acrylic using a laser cutter.

### 2D Geometry

#### Faces

Let's start with the faces, each of which corresponds exactly to one of the faces in the original 3D model:

{{< image src="/img/infinity_mirror_hypercrystal/face_pieces.png" alt="generated faces" position="center" style="border-radius: 8px;" >}}

here's what the resulting laser cut pieces look like:

{{< image src="/img/infinity_mirror_hypercrystal/laser_cut_faces.jpg" alt="laser cut faces" position="center" style="border-radius: 8px;" >}}

The blue film protects the mirrored surface of the acrylic.

#### Edges

The algorithm also generates a series of edge connector pieces, each corresponding to the exact angle betwen the pair of faces that share an edge:

{{< image src="/img/infinity_mirror_hypercrystal/edge_pieces.png" alt="generated edges" position="center" style="border-radius: 8px;" >}}

These pieces are combined with a series of struts to form the connective tissue of the piece:

{{< image src="/img/infinity_mirror_hypercrystal/standardized_structure.png" alt="standardized struts" position="center" style="border-radius: 8px;" >}}

Here's what these pieces look like after laser cutting, and before assembly:

{{< image src="/img/infinity_mirror_hypercrystal/laser_cut_edges.jpg" alt="laser cut edges and struts" position="center" style="border-radius: 8px;" >}}


### 3D Assembly

First, I assembled the edge components using wood glue to hold them together. The resulting structure is surprisingly resilient due to the way it's put together.

{{< image src="/img/infinity_mirror_hypercrystal/initial_edge_assembly.jpg" alt="putting the edge pieces together" position="center" style="border-radius: 8px;" >}}

I used dark wood stain to give them an aged, charred effect.

{{< image src="/img/infinity_mirror_hypercrystal/wood_stain.jpg" alt="wood stain" position="center" style="border-radius: 8px;" >}}

Then I did a quick test fitting, with the blue protective film still present:

{{< image src="/img/infinity_mirror_hypercrystal/test_fitting.jpg" alt="test fitting" position="center" style="border-radius: 8px;" >}}

Here's the object, with all faces and edges present (I did forget to remove some of the protective film on the exterior before taking this photo). The mirrored surface is on the interior, and is thus protected from scratches and fingerprints. This also results in an extremely convincing infinity mirror illusion. 

To avoid marring the internal surface with fingerprints I wore nitrile gloves during the final assembly phase:

{{< image src="/img/infinity_mirror_hypercrystal/assembled_crystal.jpg" alt="assembled" position="center" style="border-radius: 8px;" >}}

The bottom face (otherwise known as pleading emoji) was customized to exactly fit two LED bulbs:

{{< image src="/img/infinity_mirror_hypercrystal/post_assembly_gloves.jpg" alt="post assembly detail on bottom face" position="center" style="border-radius: 8px;" >}}

It sits on a cardboard tray cut using the same geometry, with voids shaped to hold the protruding parts of the connective piece.

{{< image src="/img/infinity_mirror_hypercrystal/cardboard_tray_with_LEDS.jpg" alt="cardboard tray with LED bulbs" position="center" style="border-radius: 8px;" >}}

Here's a picture of the final glue-up:

{{< image src="/img/infinity_mirror_hypercrystal/final_glue_up.jpg" alt="final glue-up" position="center" style="border-radius: 8px;" >}}

I'm really happy with how it turned out - here's a picture showing what it looks like in daylight, from another angle:

{{< image src="/img/infinity_mirror_hypercrystal/daylight.png" alt="the object under daylight" position="center" style="border-radius: 8px;" >}}

That's pretty much it. I'll be making more art like this. Probably less abstract, more representative. Here's a sneak peek:

{{< image src="/img/infinity_mirror_hypercrystal/next_project.jpg" alt="next project" position="center" style="border-radius: 8px;" >}}
