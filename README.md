# control-chrome-video

Documenting and keeping files for controlling google-chrome's video without v4l2

Other options (which work but have limits)[other.md]

So where I got to is abusing chrome's y4m support - an old format that starts with a ASCII header representing the file followed by literal pixel data. Each frame is stored in full, and the next frame starts straight after.

That sounds ugly and has massive filesizes, but has one key benefit: If you overwrite the file with new pixel data while retaining the same frame size and video duration; you'll never have invalid data; the worst case is you'll end up having half and half data if you don't write a complete frame at once.

Demonstration time. In one session run:

`google-chrome --use-fake-device-for-media-stream --use-file-for-fake-video-capture=video.y4m`

In another session run:

FLASHING IMAGE VIDEO WARNING: Change the sleep 0.1 to a higher value if you or others watching are at risk from flashing images.

(bs magic number is length of blue.y4m here.)
```
while true; do dd conv=notrunc if=blue.y4m of=video.y4m bs=115283; sleep 0.1; dd conv=notrunc if=red.y4m of=video.y4m bs=115283; sleep 0.1; done
```

And as if by magic you'll have a video camera that you can control (within the bounds of changing the video loop only within the same size and limits).

Generating single-frame y4m videos is simple with ffmpeg - this is how i created the blue.y4m and red.y4m images. Framerate is up at 100 fps because this then forces chrome to re-read the y4m file every 1/100th of a second - so it becomes instantly responsive to changes in the file.

```
ffmpeg -r 1 -i blue.png -pix_fmt yuv420p -s 320x240 -r 30 output.y4m

```

(i make the image 320x240 to prevent it becoming too large; y4m is not an efficient file format; even if it does make this easier to use). I don't believe a larger image size would hinder this process, until we're unable to write atomically to the file.

And now you can control chrome's video, without v4l2 being a requirement, and create sufficiently rapid changes in the video to make this worthwhile for trying latency.

# Audio

Unfortunately the --use-fake-audio-file=file.wav does not work in the same way - chrome reads it entirely into memory (not as a mmapped file) and so it cannot be modified in the same way. A solution involving a fake audio device will still be required. 

My testing for this was:

```
google-chrome --use-fake-device-for-media-stream --use-file-for-fake-video-capture=video.y4m --use-file-for-fake-audio-capture=audio.wav
```

and this to attempt to change the sound:

```
dd if=noise.wav of=audio.wav bs=8898 conv=notrunc
```

Test WAV files were generated via:

```
ffmpeg -f lavfi -i "sine=frequency=1000:duration=0.1" noise.wav
ffmpeg -f lavfi -i "anullsrc=channel_layout=mono:sample_rate=44100:duration=0.1" silence.wav
```

These are two files that are length and header identical, just with a different 0.1s of sine wave or not. If this worked I'd want to adjust the output slightly so that the repeat was exactly on a sine wave cycle, atm there's a slight variance.

