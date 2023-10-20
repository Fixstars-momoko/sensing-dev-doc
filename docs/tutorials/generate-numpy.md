---
sidebar_position: 8
---

# Use Sensor Data as Numpy Array

In this tutorial, we learn how to use data in GenDC saved in the [previous tutorial](save-gendc) as Numpy Array.

## Prerequisite

* numpy
* OpenCV

```bash
pip3 install -U pip
pip3 install opencv-python
pip3 install opencv-contrib-python
pip3 install numpy
```

* GenDC Separator module (included in SDK package)
* GenDC Descriptor information

While GenDC format is defined by emva, some ID or constant values are unique for each device. In kizashi 1.2, we need to know sourceId and where in the descriptor has audio sampling rate.

## Tutorial

### Load modules

The module we need is same as [the tutorial of generating video from raw-GenDC](save-gendc). Instead of generating video/audio file, we put data into numpy array so that you can easily process them. 

## Load and parse GenDC Data

The outline of the procesure to load and parse GenDC data are the same as [the tutorial of generating video from raw-GenDC](save-gendc). 

While audio data is 48kHz, the other non-image sensor may have different sampling rate; therefore, we need to get the sampling rate.

This varies device by device, so please refer the hardware specification.

In this tutorial, we introduce how to check the sampling rate of non-image sensor of kizashi 1.2.

Each component of GenDC is labeled by `SourceId` telling where its data comes from. If `SourceId` is `0x100*`, the data is from one of image sensors, while `0x200*` shows that its data is from an audio sensor. The framerate of audio data is constantly `48000` Hz.

Kizashi 1.2 supports analog sensors such as accelerometer and degital sensors like PMOD sensor. For either of sensor types, the number of sampling for 1 us in each container is contained in `struct1`, which is the 8-bit size with offset 8-bit of `Typespecific[3]`.

Sampling rate is equal to the number of samples in a second. i.e. we need to know  `1s / (sampling interval)` = `1000000us / (sampling interval)` = `1000000 / struct1` .


```python
# kizashi 1.2 unique (typespecific) >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>
audio_sample_rate = 48000

# sourceId (component)
SOURCEID_CAMERA1 = 0x1001
SOURCEID_CAMERA2 = 0x1002
SOURCEID_AUDIO1 = 0x2001
SOURCEID_AUDIO2 = 0x2002
SOURCEID_ANALOG1 = 0x3001
SOURCEID_ANALOG2 = 0x3002
SOURCEID_ANALOG3 = 0x3003
SOURCEID_PMOD1 = 0x4001
SOURCEID_EXT1_1 = 0x5001
SOURCEID_EXT1_2 = 0x5002
SOURCEID_EXT2_1 = 0x6001
SOURCEID_EXT2_2 = 0x6002


def kizashi_get_frequency(gendc_desc, ith_component):
    # return
    # - audio sampling rate (struct[4] of typespecifig 3)
    # - analog sampling rate per container (struct[4] of typespecifig 3)
    
    typespecific3 = gendc_desc.get("TypeSpecific", ith_component, 0)[3]
    struct4 = typespecific3.to_bytes(8, 'little')[4]
    if struct4 == 0x04:
        return audio_sample_rate
    else:
        struct1 = typespecific3.to_bytes(8, 'little')[1:3]
        freq_in_us = int.from_bytes(struct1, "little")
        return int(1000000 / freq_in_us)
# <<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<
```

Also, the data type varies for each sensor.

If the target sensor is an audio sensor, i.e. the `SourceId` is either `0x2001` or `0x2002`, the data type is `int16` and has 2-dimension (R and L).

```python
if nonimage_sensor == SOURCEID_AUDIO1 or nonimage_sensor == SOURCEID_AUDIO2:
    # if it is audio, the data has two channels
    bit_depth = np.int16
    if nonimage_datasize > 0:
        nonimage_nparr = np.frombuffer(nonimage_data, bit_depth).reshape((int(nonimage_datasize/np.dtype(bit_depth).itemsize/2), 2))
        nonimage.append(nonimage_nparr)
```

If the sensor is digital, i.e. the `SourceId` is `0x4001`, the data size is 64-bit depth but is has (x, y, z) data. The structure of data is 16-bit for each dimension (with 16-bit dummy data), so it needs to be reshaped as follows:

```python
Mono8 = 0x01080001
Mono10 = 0x01100003
Mono12 = 0x01100005
Data8 = 0x01080116
Data16 = 0x01100118 # analog
Data32 = 0x0120011a # audio L ch and R ch for respectively
Data64 = 0x0140011D

...

elif nonimage_sensor == SOURCEID_PMOD1:
    # if it is pmod, the data has three coordinates + dummy-ffff padding (x, y, z, dummy)
    # At this point, depth is fixed at 64-bit 
    bit_depth = np.int16
    num_coordinate = 1 if nonimage_format == Data16 \
        else 2 if nonimage_format == Data32 \
        else 4 if nonimage_format == Data64 else 0
    if nonimage_datasize > 0:
        nonimage_nparr = np.frombuffer(nonimage_data, bit_depth).reshape((int(nonimage_datasize/np.dtype(bit_depth).itemsize/num_coordinate), num_coordinate))
        if num_coordinate == 4:
            nonimage_nparr = nonimage_nparr[:, 0:3]
        nonimage.append(nonimage_nparr)
```

Analog sensor may have different bit-depth, so we need to know what is the data format of the sensor beforehand. It is stored in component header, so we can access it by get function of GenDC separator with the index of component 

```python

nonimage_format = gendc_descriptor.get("Format", nonimage_component)

...

else:
    # analog sensor
    bit_depth = np.int16 if nonimage_format == Data16 \
        else np.int32 if nonimage_format == Data32 \
        else np.int64 if nonimage_format == Data64 \
        else np.int8
    if nonimage_datasize > 0:
        nonimage_nparr = np.frombuffer(nonimage_data, bit_depth).reshape((int(nonimage_datasize/np.dtype(bit_depth).itemsize), 1))
        nonimage.append(nonimage_nparr)
```

### Write Numpy Array

Similar to the sequential audio data gained in [the tutorial of generating video from raw-GenDC](save-gendc), we now have a non-image sensor data in `single_nonimage`.

We simply can save these data into txt file by `np.savetxt` and you may want to use `np.loadtext` when you use the save data.

Note that the nubmer of generated file is the same as the number of channel (i.e. dimension); Audio has 2-channel, PMOD has 3-channel, and analog sensor has a single channel.

```python
for i in range(num_device):
    single_nonimage = np.concatenate( audios[i], axis=0 )

    for n_ch in  range(single_audio.shape[1]):
        np.savetxt(os.path.join(target_directory, "nonimage-sensor-data-in-numpy-" + str(i) + "-" + str(n_ch) + ".txt"), single_nonimage[:, n_ch])
```

## Complete code

Complete code used in the tutorial is [here](https://github.com/Sensing-Dev/tutorials/blob/main/python/tutorial5_save_numpy_to_txt.py)