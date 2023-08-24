Either provide a fake device via (v4l2loopback)[https://github.com/umlaeute/v4l2loopback] - use mainline rather than latest release because there's some exclusive-mode fixes there.

```
(sudo) modprobe v4l2loopback devices=4 video_nr=11,12,13,14 exclusive_caps=1,1,1,1 card_label=X_11,X_12,X_13,X_14
```

This then lets you run something like playwright in a docker container based on (mcr.microsoft.com/playwright)[https://playwright.dev/docs/docker] like so:
```
docker run -it --device /dev/video11 your-docker-image
```

And so X\_11 will be available as the only camera in this docker image. Celebration, we have a fake video camera we can write into via ffmpeg and it'll be available during the test.

I then wanted to be able to control when the image changed via a couple of static PNG image that i updated from within the test:

`/usr/bin/ffmpeg -stream_loop -1 -r 1 -re -i target%01d.png -vf realtime,format=yuv420p -f v4l2 <v4l2loopbackDevice>`

We use target%01d.png so we can have two images - if there's only one then ffmpeg won't re-open it each time it changes; this way it keeps on rotating between two images and the stream constantly updates.

Then if i want to change the static image i can swap out target0.png and target1.png to a different pair of images.

This is great; this works to be able to test that video isn't being looped somewhere, that it's being updated in a timely fashion and by having different PNG images, we can test in the receiving application that the right sources are being routed to the right places. 

EXCEPT: This doesn't work so great on non-linux systems.

So option two is to use chromes `--use-file-for-fake-video-capture="video.mjpg" --use-fake-device-for-media-stream` options to pass a mjpeg video stream in.

This is great and works everywhere, except you can't change the mjpeg on the fly - chrome initializes the fake video device as it starts, then assumes the file won't change framesize or file length (not video length) over time. So you can't change it, otherwise you'll end up with a black screen from chrome and need to restart the browser to make the fake video working again.
