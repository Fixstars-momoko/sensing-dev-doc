---
sidebar_position: 6
---
# GenDCからビデオを生成

 このチュートリアルでは、GenDCパーサーを使用して、[前のチュートリアル](save-gendc)で保存したGenDCデータを読み込む方法について学びます。

 このチュートリアルでは、kizashi 1.2（GenDCフォーマットのU3Vカメラ）を使用して、画像とオーディオの両方を記録し、ビデオを生成します。

 ## 必要条件

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

 * GenDCセパレーターモジュール（SDKパッケージに含まれています）
 * GenDC記述情報

 GenDCフォーマットはemvaによって定義されていますが、いくつかのIDや定数の値は各デバイスごとにユニークです。kizashi 1.2では、sourceIdと記述内のオーディオサンプリングレートの場所を知る必要があります。


 ## チュートリアル

 ### モジュールを読み込む

 保存したGenDCデータを読み込むには、以下のモジュールが必要です。

 ```python
 import sys
 import os
 import re
 ```

 GenDCの生データは `gendc_separator.descriptor` によって読み込まれ、デバイス情報（設定）はJSONモジュールによって読み込まれます。

 ```python
 import json

 import gendc_separator.descriptor as gendc
 ```

 また、読み込んだデータは `numpy` 配列として扱われます。解析後、画像データはOpenCVを使用して `.mp4` ビデオファイルに保存され、オーディオデータは `scipy` を使用して `wav` オーディオファイルに保存されます。

 ```python
 import numpy as np
 import cv2
 from scipy.io.wavfile import write
 from scipy.interpolate import interp1d
 ```

 ### デバイス情報を読み込む

 ディレクトリ `tutorial2_saved_gendc_YYYYmmDDHHMMSS` には、 `config.json` と `raw-X.bin` という2種類のファイルが含まれています。JSONフォーマットのファイルには、デバイスの情報が含まれており、デバイスの数、フレームレート、および画像の解像度が、センサーデータが記録された時点の設定として保存されています
。

 これらの値は、ビデオのサイズとフレームレートを定義するために必要です。

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

 ### GenDC binファイルを読み込む

 `raw-X.bin`（ここで `X` は0、1、2など）には、時系列順にGenDCのシリーズが含まれています。これらのファイルを読み込む順序に注意する必要があります。

 ファイルパスは文字列なので、`X=2` は `X=1` の後に来ますが、リソートせずに `X=11` も来ます。

 ```python
 bin_files = [f for f in os.listdir(target_directory) if f.startswith("raw-") and f.endswith(".bin")]
 bin_files = sorted(bin_files, key=lambda s: int(re.search(r'\d+', s).group()))
 ```

 すべての `raw-X.bin`（ここで `X=0, 1, 2、...、N`）を読み込み、ビデオとオーディオファイルを生成する予定です。

 `cv2.VideoWriter` オブジェクトを使用してフレームを1つずつ追加できる一方、 `scipy.io.wavfile.write` はオーディオファイルを生成するためにデータ全体が必要です。そのため、 `cv2.VideoWriter` を作成してビデオの書き込みを開始し、オーディオデータを保持するための `List` を作成して、データの読み込みが完了す 
るまで `scipy.io.wavfile.write` を使用できるようにします。

 ```python
 videos = {} #各デバイス用のvideowriter
 audios = {} #各デバイス用のオーディオファイルのnumpyリスト

 ...

 for i in range(num_device):
     audios[i] = list()
     videos[i] = cv2.VideoWriter(os.path.join(target_directory, 'without_audio' + str(i) + '.mp4'), cv2.VideoWriter_fourcc(*'mp4v'), image_framerate[i], (widths[i], heights[i]), False)
 ```

 単一の `raw-X.bin` には複数のGenDCが含まれています。1つずつ解析するために、GenDCの開始部分（オフセット）とGenDCの終了部分（オフセット + コンテナサイズ）を知る必要があります。

 これは `cursor` で管理され、各binファイルの開始時には `0` であるべきです。GenDCディスクリプタにはGenDCのサイズが含まれているため、 `gendc_descriptor.get_container_size()` によって増加します。

 ```python
 cursor = 0
 filecontent = f.read()
 ...
 gendc_descriptor = gendc.Container(filecontent[cursor:])
 ...
 cursor = cursor + gendc_descriptor.get_container_size()
 ```

 ### GenDCを解析する

 上記のセクションでは、 `gendc_descriptor` に単一のGenDCデータが含まれています。

 これをコンポーネントごとに、パーツごとに解析する必要があります。

 ```python
 # TypeId
 GDC_INTENSITY   = 0x0000000000000001
 GDC_METADATA    = 0x0000000000008001

 image_component = gendc_descriptor.get_first_get_datatype_of(GDC_INTENSITY)
 nonimage_component = gendc_descriptor.get_first_get_datatype_of(GDC_METADATA)
 ```
 上記のスニペットは、データのコンポーネントのデータ型が画像と非画像の場合のコンポーネントのインデックスを取得します。

 ### ビデオを書き込む

 ディスクリプタを除くデータにアクセスするには、データの開始位置（データオフセット）とデータの終了位置（データオフセット + サイズ）を知る必要があります。画像オフセットは `cursor`（binファイル内のGenDCのオフセット）+ `DataOffset`（GenDC内のデータのオフセット）です。

 ```python
 image_datasize = gendc_descriptor.get("DataSize", image_component, 0)
 image_offset = cursor + gendc_descriptor.get("DataOffset", image_component, 0)
 image_data = filecontent[image_offset: image_offset+image_datasize]
 ```

 読み込まれたデータは、前のチュートリアルで述べられているようにUint8の1次元形状です。これを画像形式に変換する必要があります。 `height` と `width` は [デバイス情報の読み込み](#load-device-info) で学んだJSONオブジェクトから取得されます。

 ```python
 image_offset = cursor + gendc_descriptor.get("DataOffset", image_component, 0)
 image_nparr = np.frombuffer(image_data, np.uint8 if pixel_format == Mono8 else np.uint16).reshape((height, width))
 video.write(image_nparr)
 ```

 すべての `raw-X.bin` ファイルを解析した後、 `cv2.VideoWriter` オブジェクトを解放してビデオを閉じることができます。

 ```python
 for i in range(num_device):
     videos[i].release()
 ```

 ### オーディオを書き込む

 [ビデオの書き込み](#write-video) で学んだように、オーディオデータのオフセットとサイズを取得する必要があります。

 ```python
 if nonimage_sensor == SOURCEID_AUDIO1 or nonimage_sensor == SOURCEID_AUDIO2:

     nonimage_datasize = gendc_descriptor.get("DataSize", nonimage_component, 0)
     nonimage_offset = cursor + gendc_descriptor.get("DataOffset", nonimage_component, 0)
     nonimage_data = filecontent[nonimage_offset: nonimage_offset+nonimage_datasize]
 ```

 kizashi 1.2のオーディオセンサーはLとRチャンネルを持っており、int32に分割されています。したがって、オーディオデータの次元は2で、ビット深度はint16です。

 ```python
     # オーディオの場合、データには2つのチャンネルが含まれています
     bit_depth = np.int16
     num_channel = 2
     if nonimage_datasize > 0:
         nonimage_nparr = np.frombuffer(nonimage_data, bit_depth).reshape((int(nonimage_datasize/np.dtype(bit_depth).itemsize/num_channel), num_channel))
         audio.append(nonimage_nparr)
 ```

 リシェイプされたオーディオファイルをリストに追加し続け、完了したらリスト内のすべてのオーディオデータを連結して、scipyが連続したオーディオファイルを生成できるようにします。

 ```python
 for i in range(num_device):
     videos[i].release()
     single_audio = np.concatenate( audios[i], axis=0 )

     audio_sample_rate = 48000

     write(os.path.join(target_directory, "audio" + str(i) + ".wav"), audio_sample_rate, single_audio.astype(bit_depth))
 ```

 オーディオのサンプリングレートまたはGenDC内のオーディオサンプリングレートのオフセットを知る必要があることに注意してください。

 ## 完全なコード

 このチュートリアルで使用される完全なコードは[こちら](https://github.com/Sensing-Dev/tutorials/blob/main/python/tutorial3_generate_video.py) で提供されています。