---
layout: post
title: "Cutting Robot Video Latency from 86 ms to 22.8 ms"
date: 2026-10-23
slug: robot-video-latency
image: /assets/figures/robot_video_latency_still.jpg
---

![Operator in Dublin controlling the arm in London](/assets/figures/robot_video_latency_dublin_london.gif)

## Introduction:

It's hard to get any AI to do things with a 100% success rate. For example, Claude Code requires human input once in a while or it grows unhinged.

Robotics models can already do things right 70 - 80% of the time. Why wait? Take a model that works sometimes, deploy it, correct it and improve it. 

Having high quality corrections is important. One way to maximize the quality of these corrections is by Giving the human the most up to date information about the robot as possible.

In particular, video information is very important but latency must be minimized.

## How to minimize latency?

There's three distinct chunks. 

1. The time from photons hitting the camera until that image has been sent over the network. 

2.  network propagation. It is the time it takes for the image to travel across network.

3.  the time it takes for the photons of that image to hit the glass of the screen.

People call this total time glass to glass latency. Here we will just call it latency. 

## Photons to network

To minimize #1 , we use a high speed camera. We have 260 frames per second, which means the amount of time it takes for a frame to go from pixels to image is four milliseconds. 

We do decode (MJPEG). We encode it H.264 broadcasted over the network. 

This could likely be optimized much more by using camera that default provides H.264.


## net to net

We can change software and even hardware but physics is insurmountable. The speed of light is fixed. Every thousand kilometers costs about five milliseconds. 

The thing we want to optimize here for is jitter. We want our frames to be as consistent as possible so our remote operators can learn to trust them. 


## network to screen

This is actually surprisingly hard. With a 60 hertz screen we get a new frame every 16 ms. Plus browsers can be much worse. On Linux, Chrome takes 3 refreshes for an image to be displayed; at 60 hertz that's 50ms. 
On Android, Chrome holds about 1.7 refreshes.

By making a custom full screen app, we were able to bypass most of the window manager and write directly to a buffer that gets put onto the screen.

## How we measured

A high speed camera records

1. the real gripper jaw
2. a screen showing the live video of that jaw. 

We open and close the gripper, then line up the motion of the real jaw with the motion of its copy on the screen. The shift between the two curves is the latency. Because one camera sees both, there's only one clock, and the measurement includes everything: camera, encoder,
the trip through our London relay and back, decoding, drawing, and the screen itself. Each setting got 30 gripper moves per run and at least two runs, alternated with the other settings so slow drift doesn't favor any one. Numbers are medians. Repeats of the same setting agree within about ±1.5 ms.

We ran the same tests on two screens:

- **Desktop monitor:** an ordinary monitor on a Linux desktop, at 60 or 75 Hz.
- **Phone:** a 60 or 120 Hz phone.


## Results

We tested many different configurations. Here is a summary. 


### Desktop monitor

Adding one change at a time:

| Step | Latency | Change |
|---|---|---|
| Chrome, 60 Hz screen, sending 65 fps | 86.2 ms | |
| Send at the screen's refresh rate | 76.8 ms | −9.4 |
| Run the monitor at 75 Hz | 64.0 ms | −12.8 |
| Our own viewer (watch-video), ordinary window | 38.5 ms | −25.5 |
| Write straight into the buffer on screen (front buffer), send all 260 fps | 33.5 ms | −5.0 |
| Skip stale camera frames | 32.0 ms | −1.5 |
| Camera and encoder in a separate process | 32.5 ms | 0 |
| Decode the camera's JPEG at half size | 32.0 ms | 0 |
| Patched USB camera driver | **29.2 ms** | −2.8 |

Our best desktop setup with 75 hertz monitor (29.2 ms)

### Phone

Adding one change at a time:

| Step | Latency | Change |
|---|---|---|
| Chrome, 60 Hz screen, old sender | 64.2 ms | |
| Chrome at 120 Hz | 51.8 ms | −12.5 |
| Native front-buffer viewer, ~87 fps | 33.2 ms | −18.5 |
| Send all 260 fps | 25.0 ms | −8.2 |
| Camera and encoder in a separate process | 24.8 ms | 0 |
| Decode the camera's JPEG at half size | 25.0 ms | 0 |
| Patched USB camera driver | **22.8 ms** | −2.2 |

### What each change bought

| Change | Monitor | Phone |
|---|---|---|
| Getting out of the browser (our own viewer drawing to the screen) | −25.5 ms | −18.5 ms |
| Faster screen refresh | −12.8 ms (60 → 75 Hz) | −12.5 ms (60 → 120 Hz, Chrome) |
| Sending every camera frame | −5.0 ms | −8.2 ms |
| Sending at the screen's rate (Chrome) | −9.4 ms | — |
| Patched USB camera driver | −2.8 ms | −2.2 ms |
| Skipping stale frames | worst 1% of frames: 127 → 23 ms | — |
| Separate camera process, half-size decode | no change | no change |

The display path is most of it. The browser's display pipeline costs more than everything else put together. Some sender changes don't move the typical number but keep it steady: skipping stale frames stops a single stall from turning into a backlog. Those matter for how the video feels, even when the median doesn't change.

The sender improvements are worth almost nothing to someone watching in Chrome: 63.5 vs 64.2 ms at 60 Hz, and 49.0 vs 51.8 ms at 120 Hz. Chrome's display queue swamps them.

## Conclusion

To really deploy robots in the world, you need to have remote controllability. You need to correct problems and gather high quality data. This project allows you to do both.

We packaged this into a remote teleoperation tool that lets you control robots over the internet with low latency. 

Don't wait for 100% to deploy robots. We can deploy with humans to help when things go wrong.  
