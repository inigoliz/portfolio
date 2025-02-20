---
date: '2024-11-27T12:36:05+01:00'
draft: false
title: 'Automatic Cat Feeder'
slug: janus-cat-feeder
cover:
  image: "/images/janus-cat-feeder/IMG_9995.gif"
  alt: "Cover image"
  relative: false
---
![(Janus header art)](/images/janus-cat-feeder/janus_header.png#center "500px")

---
## What is it?

![(Lilo getting her food)](/images/janus-cat-feeder/IMG_9965_2.gif#center "350px")
A camera-controlled cat feeder that only exposes the food when a certain pet is around.

---
## Motivation

I have two cats: one is black and slim and the other is white and chubby. The black one is called Lilo. The white one, Coco.

![(Lilo and Coco)](/images/janus-cat-feeder/IMG_9967.jpg#center "350px")

The problem is that Coco eats Lilo's wet food. I've built an automatic pet feeder that only exposes the food when Lilo approaches, thus stopping Coco from eating it.

![(Both using the feeder)](/images/janus-cat-feeder/lilo_coco_side_by_side.gif#center "600px")

## Design

The pet feeder is designed as a box with a drawer. Inside the box, a mechanic system controls the opening and closing of the drawer. A logic board decides when to open the drawer based on the images feed by a camera. A push button also allows for manual operation of the drawer.

![Cat Feeder overview](/images/janus-cat-feeder/IMG_0024.jpg#center "350px")

The box is designed such that when the drawer closes, it leaves no space for Coco to try to force the mechanism.

![(View of the door closing tightly)](/images/janus-cat-feeder/IMG_0005.gif#center "350px")

The design of the device went through several iterations. The biggest issue to overcome was that Coco, once aware that there was food inside, would try to force the cat-feeder to get access to the food.

## Inner Working
A servomotor operates a 3D-printed rack and pinion mechanism, which opens and closes the drawer.

![(Rack and pinion working)](/images/janus-cat-feeder/IMG_9995.gif#center "600px")

In order not to harm the cats in case of malfunction, the rack is coupled to the drawer thorugh a pair of magnets. If something blocks the drawer while it closes, the magnets detach, avoiding any possible harm both to the cats and to the device.

![(Closeup of the magnets)](/images/janus-cat-feeder/IMG_0026.jpg#center "350px")

A Raspberry Pi + Camera Module acts as the logic board. A custom shield wires together the different components.

![(Electronics)](/images/janus-cat-feeder/rpi_and_shield_2.png#center "1000px")

## Image Processing

I've tried different approaches to process the feed of the camera and decide whether to open the drawer or not. Even though in my heart I really wanted to use some object detection and classification ML model to detect the cats, the truth is that a simple pipeline that just analized the histograms of the frames is what worked best. The problem is that when the cats get close enough, the camera just sees a fuzzy image.

![(Closeup of the cats when eating)](/images/janus-cat-feeder/cats_collage_2.png#center)

The final algorithm was as simple as:
1. Transform frame to grayscale
2. Compute histogram
3. Check if the number of pixels with `pixel_intensity < 50` was above an experimentally determined threshold.

![(View of the door closing tightly)](/images/janus-cat-feeder/coco_lilo_hist_2.gif#center "800px")

The algorithm incorporates other features, such as toggle debouncing etc. The full code is available in [my github](https://github.com/ignigoliz/janus/).

![(Visitor Count)](https://komarev.com/ghpvc/?username=janus-hugo&style=pixel&label=VISITOR+COUNT)

