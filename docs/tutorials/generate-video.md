---
sidebar_position: 6
---

# Generate Video from GenDC

In this tutorial, we learn how to use GenDC parser to load GenDC data saved in the [previous tutorial](save-gendc).

We use kizashi 1.2 (U3V camera with GenDC format) to record both image and audio to generate the video in this tutorial.

## Prerequisite

* scipy 
* numpy
* OpenCV

```bash
pip3 install -U pip
pip3 install opencv-python
pip3 install opencv-contrib-python
pip3 install numpy
pip3 install scipy
```

* GenDC Separator module (included in SDK package)
* GenDC Descriptor information

While GenDC format is defined by emva, some ID or constant values are unique for each device. In kizashi 1.2, we need to know sourceId and where in the descriptor has audio sampling rate.


## Tutorial

### Load modules

To load the saved GenDC data, we need the following modules.

```python
import sys
import os
import re
```
GenDC raw data is loaded by `gendc_separator.descriptor`, and device info(config) is loaded by JSON module.

```python
import json

import gendc_separator.descriptor as gendc
```

Also, loaded data would be treated as `numpy` array. After parsing, image data will be saved into `.mp4` video with OpenCV and audio data will be saved into `wav` audio file with `scipy`.

```python
import numpy as np
import cv2
from scipy.io.wavfile import write
from scipy.interpolate import interp1d
```

### Load Device Info

The directory `tutorial2_saved_gendc_YYYYmmDDHHMMSS` contains two types of files: `config.json` and `raw-X.bin`. The JSON format file has device info, such as number of devices, framerate, and resolution of images at the time the sensor data was recorded.

Those values are required to generate video to define its size and framerate.

```python
f = open(os.path.join(target_directory, "config.json"))
config = json.loads(f.read())
f.close()
num_device = config["num_device"]

image_framerate = [ config[key]["framerate"] for key in config if "sensor" in key]
widths = [ config[key]["width"] for key in config if "sensor" in key]
heights = [ config[key]["height"] for key in config if "sensor" in key]
bit_depths = []

```

### Load GenDC bin files

Since `raw-X.bin` where `X = 0, 1, 2, ...` contains the series of GenDC in chronological order, we need to pay attention to the order of loading these files.

File path is a string, so `X=2` is coming after `X=1` but also `X=11` without resorting.

```python
bin_files = [f for f in os.listdir(target_directory) if f.startswith("raw-") and f.endswith(".bin")]
bin_files = sorted(bin_files, key=lambda s: int(re.search(r'\d+', s).group()))
```

We are planning to load all `raw-X.bin` where `X = 0, 1, 2, ..., N` and generate video and audio files.

While `cv2.VideoWriter` object allows to adding frame one by one with `write()` API, `scipy.io.wavfile.write` requires the whole data to generate audio file. Therefore, we create `cv2.VideoWriter` to start to write a video and `List` to keep storing audio data so that we can use `scipy.io.wavfile.write` until we finish loading data.

```python
videos = {} #videowriter for each device
audios = {} #list of numpy of audio file for each device

...

for i in range(num_device):
    audios[i] = list()
    videos[i] = cv2.VideoWriter(os.path.join(target_directory, 'without_audio' + str(i) + '.mp4'), cv2.VideoWriter_fourcc(*'mp4v'), image_framerate[i], (widths[i], heights[i]), False)
```

Single `raw-X.bin` has multiple GenDC. To parse one by one, we need to know which part of the bin file is the beginning of the GenDC (offset) and which part of the bin file is the end of the GenDC (offset + container size).

We manage this with `cursor` and it should be `0` at the beginning for each bin file. Since the GenDC Descriptor contains the size of GenDC, it increments by `gendc_descriptor.get_container_size()`.

```python
cursor = 0
filecontent = f.read()
...
gendc_descriptor = gendc.Container(filecontent[cursor:])
...
cursor = cursor + gendc_descriptor.get_container_size()
```

### Parse GenDC

In the section above, we get a single GenDC data in `gendc_descriptor`.

Now we need to parse it component by component, and part by part.

```python
# TypeId
GDC_INTENSITY   = 0x0000000000000001
GDC_METADATA    = 0x0000000000008001

image_component = gendc_descriptor.get_first_get_datatype_of(GDC_INTENSITY)
nonimage_component = gendc_descriptor.get_first_get_datatype_of(GDC_METADATA)
```
The snippet above get the component index where the data type of the component is image and non-image.

### Write video

To access the data, excluding descriptor, we need to know where the data starts (data offset) and where the data ends (data offset + its size). Please note that image offset is `cursor` (offset of GenDC in a bin-file) + `DataOffset` (offset of data in GenDC).

```python
image_datasize = gendc_descriptor.get("DataSize", image_component, 0)
image_offset = cursor + gendc_descriptor.get("DataOffset", image_component, 0)
image_data = filecontent[image_offset: image_offset+image_datasize]
```

To loaded data is Uint8 1D shape as it is stated in the previous tutorial. We need to convert this into the image format. `height` and `width` are coming from JSON object that we learned in [Load Device Info](#load-device-info).

```python
image_offset = cursor + gendc_descriptor.get("DataOffset", image_component, 0)
image_nparr = np.frombuffer(image_data, np.uint8 if pixel_format == Mono8 else np.uint16).reshape((height, width))
video.write(image_nparr)
```

After parsing all `raw-X.bin` files, you can release the `cv2.VideoWriter` object to close the video.

```python
for i in range(num_device):
    videos[i].release()
```

### Write Audio

As we learned in [Write Video](#write-video), we need to obtain audio data's offset and size.

```python
if nonimage_sensor == SOURCEID_AUDIO1 or nonimage_sensor == SOURCEID_AUDIO2:

    nonimage_datasize = gendc_descriptor.get("DataSize", nonimage_component, 0)
    nonimage_offset = cursor + gendc_descriptor.get("DataOffset", nonimage_component, 0)
    nonimage_data = filecontent[nonimage_offset: nonimage_offset+nonimage_datasize]
```

Audio sensor of kizashi 1.2 has L and R channel which split int32. Therefore, the dimension of audio data is 2 and each bit-depth is int16.

```python
    # if it is audio, the data has two channels
    bit_depth = np.int16
    num_channel = 2
    if nonimage_datasize > 0:
        nonimage_nparr = np.frombuffer(nonimage_data, bit_depth).reshape((int(nonimage_datasize/np.dtype(bit_depth).itemsize/num_channel), num_channel))
        audio.append(nonimage_nparr)
```

You keep add the reshaped audio file into the List, and once you finish it, concatinate the all audio data in the List so that scipy can generate the sequential audio file.

```python
for i in range(num_device):
    videos[i].release()
    single_audio = np.concatenate( audios[i], axis=0 )

    audio_sample_rate = 48000

    write(os.path.join(target_directory, "audio" + str(i) + ".wav"), audio_sample_rate, single_audio.astype(bit_depth))
```

Note that you need to know either the audio sampling rate or the offset of audio sampling rate in GenDC.

## Complete code

Complete code used in the tutorial is [here](https://github.com/Sensing-Dev/tutorials/blob/main/python/tutorial3_generate_video.py)