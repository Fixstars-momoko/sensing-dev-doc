---
sidebar_position: 8
---

# センサーデータをNumpy配列として使用

このチュートリアルでは、[前のチュートリアル](save-gendc)でGenDCに保存されたデータをNumpy配列として使用する方法を学びます。

## 必要なもの

* numpy
* OpenCV

```bash
pip3 install -U pip
pip3 install opencv-python
pip3 install opencv-contrib-python
pip3 install numpy
```

* GenDCセパレータモジュール（SDKパッケージに含まれています）
* GenDCディスクリプタ情報

GenDCフォーマットはemvaによって定義されていますが、一部のIDや定数値は各デバイスごとに固有です。Kizashi 1.2では、sourceIdとディスクリプタ内のオーディオサンプリングレートの位置を知る必要があります。


## チュートリアル


### モジュールを読み込む

必要なモジュールは、[生のGenDCからビデオを生成するチュートリアル](save-gendc)と同じです。ビデオ/オーディオファイルを生成する代わりに、データをNumpy配列に入れて簡単に処理できるようにします。


### GenDCデータを読み込み、解析する

GenDCデータを読み込み、解析する手順の概要は、[生のGenDCからビデオを生成するチュートリアル](save-gendc)と同じです。

オーディオデータは48kHzですが、他の非画像センサーは異なるサンプリングレートを持つ場合があります。したがって、サンプリングレートを取得する必要があります。

これはデバイスごとに異なりますので、ハードウェア仕様を参照してください。

このチュートリアルでは、kizashi 1.2の非画像センサーのサンプリングレートを確認する方法を紹介します。

GenDCの各コンポーネントは、`SourceId`によってどこからデータが来たかを示しています。`SourceId`が`0x100*`である場合、データは画像センサーの1つから来ており、`0x200*`はオーディオセンサーのデータであることを示しています。オーディオデータのフレームレートは常に`48000` Hzです。

Kizashi 1.2は、加速度計などのアナログセンサーやPMODセンサーなどのデジタルセンサーをサポートしています。どちらのセンサータイプにも、各コンテナ内の1μsあたりのサンプリング間隔が`struct1`に含まれており、それは`Typespecific[3]`のオフセット8ビットの8ビットサイズです。

サンプリングレートは1秒あたりのサンプリング回数と動議です。1秒あたりのサンプリング回数を取得するには、`1s / (サンプリング間隔)` = `1000000us / (サンプリング間隔)` = `1000000 / struct1`を知る必要があります。

```python
# kizashi 1.2固有（タイプスペシフィック） >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>
audio_sample_rate = 48000

# ソースID（コンポーネント）
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
    # 戻り値
    # - オーディオサンプリングレート（typespecifig 3のstruct[4]）
    # - 1コンテナあたりのアナログサンプリングレート（typespecifig 3のstruct[4]）

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

また、各センサーのデータタイプも異なります。

対象センサーがオーディオセンサーである場合、つまり`SourceId`が`0x2001`または`0x2002`の場合、データタイプは`int16`で、2次元（RおよびL）です。

```python
if nonimage_sensor == SOURCEID_AUDIO1 or nonimage_sensor == SOURCEID_AUDIO2:
    # オーディオの場合、データは2つのチャンネルを持っています
    bit_depth = np.int16
    if nonimage_datasize > 0:
        nonimage_nparr = np.frombuffer(nonimage_data, bit_depth).reshape((int(nonimage_datasize/np.dtype(bit_depth).itemsize/2), 2))
        nonimage.append(nonimage_nparr)
```

センサーがデジタルである場合、つまり`SourceId`が`0x4001`の場合、データサイズは64ビットですが、（x、y、z）データが含まれています。データの構造は各次元ごとに16ビットであり（16ビットのダミーデータを含む）、次のようにリシェイプする必要があります。 

```python
Mono8 = 0x01080001
Mono10 = 0x01100003
Mono12 = 0x01100005
Data8 = 0x01080116
Data16 = 0x01100118 # アナログ
Data32 = 0x0120011a # オーディオのLチャンネルおよびRチャンネル
Data64 = 0x0140011D

...

elif nonimage_sensor == SOURCEID_PMOD1:
    # PMODの場合、データには3つの座標+ダミーffffパディング（x、y、z、ダミー）が含まれています
    # この時点での深度は固定で64ビットです
    bit_depth = np.int16
    num_coordinate = 1 if nonimage_format == Data16 \
        else 2 if nonimage_format == Data32 \
        else 4 if nonimage_format == Data64 else 0
    if nonimage_datasize > 0:
        nonimage_nparr = np.frombuffer(nonimage_data, bit_depth).reshape((int(nonimage_datasize/np.dtype(bit_depth).itemsize/num_coordinate), num_coordinate))
        if num_coordinate is 4:
            nonimage_nparr = nonimage_nparr[:, 0:3]
        nonimage.append(nonimage_nparr)
```

アナログセンサーは異なるビット深度を持つ場合があるため、センサーのデータ形式を事前に知る必要があります。これはコンポーネントヘッダに格納されており、GenDCセパレータのコンポーネントのインデックスでアクセスできます。

```python

nonimage_format = gendc_descriptor.get("Format", nonimage_component)

...

else:
    # アナログセンサー
    bit_depth = np.int16 if nonimage_format == Data16 \
        else np.int32 if nonimage_format == Data32 \
        else np.int64 if nonimage_format == Data64 \
        else np.int8
    if nonimage_datasize > 0:
        nonimage_nparr = np.frombuffer(nonimage_data, bit_depth).reshape((int(nonimage_datasize/np.dtype(bit_depth).itemsize), 1))
        nonimage.append(nonimage_nparr)
```

### Numpy配列に書き込む

[生のGenDCからビデオを生成するチュートリアル](save-gendc)で得られたシーケンシャルオーディオデータと同様に、`single_nonimage`に非画像センサーデータが含まれています。

これらのデータは単純に`np.savetxt`を使用してテキストファイルに保存でき、保存データを使用する場合には`np.loadtext`を使用したいと思うかもしれません。

生成されたファイルの数はチャンネルの数と同じです（つまり、オーディオは2チャンネル、PMODは3チャンネル、アナログセンサーは単一チャンネルです）。

```python
for i in range(num_device):
    single_nonimage = np.concatenate( audios[i], axis=0 )

    for n_ch in  range(single_audio.shape[1]):
        np.savetxt(os.path.join(target_directory, "nonimage-sensor-data-in-numpy-" + str(i) + "-" + str(n_ch) + ".txt"), single_nonimage[:, n_ch])
```

## 完全なコード

チュートリアルで使用される完全なコードは[こちら](https://github.com/Sensing-Dev/tutorials/blob/main/python/tutorial5_save_numpy_to_txt.py)で確認できます。