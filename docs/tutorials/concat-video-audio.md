---
sidebar_position: 7
---

# Concat Video and Audio

In this tutorial, we learn how to use `import subprocess` and `ffmpeg` to concat video and audio file that you generated in the [previous tutorial](generate-video).

:::note ffmpeg-python support for Python 3.11

In this tutorial, we use `ffmpeg.exe` via `subprocess` module since python-wrapper (module) called `ffmpeg-python` which provides API to access `ffmpeg.exe` is not supporting Python 3.11 yet.

We will update this tutorial once `ffmpeg-python` is available on Python 3.11.
:::

## Prerequisite

* [ffmpeg](https://ffmpeg.org/download.html)

## Tutorial

### Load modules

Import subprocess module to run `ffmpeg.exe` via Python code.

```python
import subprocess
```

### Concat Video and Audio

This tutorial assume that both video and audio were generated from the [previous tutorial](generate-video).

If they are generated in the other method, please make sure that duration of both files are the same.

```python
audio_file = 'audio0.wav'
video_file = 'without_audio0.mp4'
```

ffmpeg is multi-media processing framework. Since we want to concat two different types of media, we need to specify some options.

Input files can be set with `-i` option.

Codec is the program/driver to encode data. Since the codec of output video would like to follow the codec of input, we set `-c:v` (codec option for video) to `copy`, and set `-c:a` (codec option for audio) to `aac` (Advanced Audio Coding) which allows to have max-96kHz (grater than 48kHz of kizashi audio sensor) and to be widely used in many platform.

Sometimes, single media file has multiple stream; therefore we need to pick which ones to be concatenated with `-map` option with `<index of the file>:<media type>:<which stream>`. If you are sure that you would like to use all stream, only the first two (index of the file and media type) are required to set.

Now, we have ffmpeg option like this:
```python
ffmpeg_option = ' -i without_audio0.mp4 -i audio0.wav -c:v copy -c:a aac -map 0:v:0 -map 1:a:0 concat.mp4'
```

`concat.mp4` is the name of output file.

Now we run ffmpeg via subprocess.

```python
proc0 = subprocess.Popen('ffmpeg' +  ffmpeg_option, shell=True, stdout=subprocess.PIPE, stderr=subprocess.STDOUT)
```

Note that `ffmpeg.exe` needs to be in the same directory where you run this tutorial code or in the environment variable `PATH`. 

## Complete code

Complete code used in the tutorial is [here](https://github.com/Sensing-Dev/tutorials/blob/main/python/tutorial4_concat_video_and_audio.py)