---
date: '2025-03-12T23:26:01+01:00'
draft: false
title: 'Object Detection on FPV Drone'
slug: object-detection-fpv-drone
cover:
  image: "/images/object-detection-fpv-drone/cover.png"
  alt: "Cover image"
  relative: false
---

## Introduction

This project started with a simple question:

> How difficult would it be to get machine learning running locally on my FPV drone?

![(Gif showing EEPROM reading)](/images/object-detection-fpv-drone/drone_table.gif#center "700px")

After several iterations, I managed to run object detection on the video feed captured by my drone with a latency high enough not to introduce lag in the flying experience.

At this point I must take a pause. I know what you're wondering: are you building *smart killer drones*? Definitely not! The models that I'm using or I could use with my hardware are **too simple and faulty** to be of any practical use! These weapons exist, but still remain far away from what you can build at your home.

![(Misclassification from the ML model)](/images/object-detection-fpv-drone/coco_dog.png#center "700px")

> **Note:** The technical reason why I cannot run advanced ML models with my setup is that the hardware accelerator that I'm using only fits ML models with a maximum size of 8 MB. That's a pretty serious limitation for how good of a model you can use!

The motivation behind this project is not weapons but another: I wanted to get ML running live on a video feed so that I could show the result on my FPV goggles. For example, magine running a **DeepDream** kind of effect live on a camera stream. How cool would that be? I didn't get that far, but this project enables the technology for it.

```
# Altered perception with DeepDream and FPV goggles (idea)

Live video -> DeepDream ML model -> FPV goggles
```

![(DeepDream example)](/images/object-detection-fpv-drone/deepdream.jpg#center "700px")

Here is a recording of what I see on my FPV goggles (sorry for the low quality):

![(Object detection as seen in FPV)](/images/object-detection-fpv-drone/inside_goggles.gif#center "700px")

Interested? Let me show you how I built it!

## Hardware List

These are the pieces of hardware that I used in the project:

![(All the hardware I used in this project)](/images/object-detection-fpv-drone/hardware.jpeg#center "700px")

In the image above you can see:
- Raspberry Pi 5 (other models will work as well)
- Raspberry Pi Camera Module
- Coral Tensor Processing Unit (TPU) USB accelerator
- Analog FPV Goggles
- FPV drone

On a high-level, this is what happens:
1. The RaspberryPi camera gets the video feed
2. The Coral TPU gets the object detection done
3. The RaspberryPi outputs via analog video the video feed + object detection
4. The drone's Video Transmitter (VTX) sends the video to the FPV goggles

Analog FPV drones transmits video at 30 FPS. The image processing must happe at that latency (at least) to avoid lagginess.

The device handling the heavy ML is a Coral Tensor Processing Unit (TPU). It's an ML accelerator: a small USB device that can run ML models with high latency. For example, it can run object detection with a latency of ~17 ms. That means that it runs at ~60 FPS, which is more than enough for a stream of video.

![(Closeup of a Coral TPU)](/images/object-detection-fpv-drone/coral.jpeg#center "700px")

## Connecting the RaspberryPi to the drone

I needed a way to intercept the video frames, run ML on them and then have the output sent to the FPV goggles:

```
Video pipeline overview:
FPV video stream -> Processing on Coral TPU -> VTX -> FPV goggles
```

First I tried getting access to video feed of the drone via the drone firmware but then I realized that the drone's MCU does not have USB, hence I would not be able to connect the Coral TPU to it. I was happy about this because hacking the drone's firmware was no easy task.

I needed another approach. Less refined. More hacky. And then... I found it:

> What if I connected the RaspberryPi to the drone's VTX directly?

After a quick Google, I found that the RaspberryPi can output analog video. I knew that my VTX worked with analog video! I soldered some wires, connected everything... and *voilà*:

Simple as that, I had the Raspberry Pi desktop showing on my FPV Goggles!

![(Raspberry Pi Desktop in FPV goggles)](/images/object-detection-fpv-drone/apple_vision.gif#center "700px")

I soldered wires to the RaspberryPi to get access to the analog video output and enabled it in software following the official documentation.

![(Analog video hardware connections)](/images/object-detection-fpv-drone/rpi-video.png#center "700px")

> **Note:**
> In the second picture, if you look closely you'll see a purple cable soldered. It comes from the drone's *FPV camera*. For some reason that I don't understand, I had to keep it soldered to be able to inject the analog video from the Raspberry Pi into the drone analog video input.

Coding on the FPV goggles felt like having a very cheap version of Apple Vision 😂:

![(Raspberry Pi Desktop in FPV goggles)](/images/object-detection-fpv-drone/coding-fpv.gif#center "700px")

## Software

The software that I used on this project is available on my Github here: [inigoliz/rpi-meets-tpu](https://github.com/inigoliz/rpi-meets-tpu/).

I don't wan't to go much into details, but what it does is running a pretrained object detection model in the Coral TPU. Coral provides several trained models. The model I'm using is:

```
ssd_mobilenet_v2_coco_quant_postprocess_edgetpu.tflite
```

It is ~6.9 MB and it can detect 90 COCO objects (some of them are `human`, `fire hydrant` and `donut` - I guess this is what you get with American-centric AI...)

The distinctive feature of my software implementation of object detection in the Coral TPU is that, unlike the examples provided by Google, I **don't need to install any TPU-specifi libraries** to use the TPU. This has the advantage that I can use the TPU with devices that originally were not supported.

> The Raspberry Pi Zero W was released in 2016, 3 years before the Coral TPU. Despite it not being oficially supported, I can use it since my implementation does not use the official libraries.

![(Raspberry Pi Zero running object detection)](/images/object-detection-fpv-drone/rpi-zero.jpeg#center "700px")

![(Raspberry Pi Zero running object detection)](/images/object-detection-fpv-drone/requirements.png#center "700px")


## Closing remarks

Just another video...

![(Object detection on elephants)](/images/object-detection-fpv-drone/elephants.gif#center "700px")




