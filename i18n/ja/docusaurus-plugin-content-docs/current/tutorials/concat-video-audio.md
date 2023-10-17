---
sidebar_position: 5
---

# 動画と音声の連結

このチュートリアルでは、[前のチュートリアル](generate-video)で生成した動画と音声ファイルを連結する方法を学びます。

:::note Python 3.11向けのffmpeg-pythonサポート

このチュートリアルでは、まだPython 3.11をサポートしていないためPythonラッパーモジュールである `ffmpeg-python` のかわりに`subprocess` を介して `ffmpeg.exe` を使用します。

`ffmpeg-python` がPython 3.11でサポートされるようになったら、このチュートリアルを更新します。
:::

## 必要なもの

* [ffmpeg](https://ffmpeg.org/download.html)

## チュートリアル

### モジュールの読み込み

Pythonコードを介して `ffmpeg.exe` を実行するために、subprocessモジュールをインポートします。

```python
import subprocess
```

### 動画と音声の連結

このチュートリアルでは、動画と音声が[前のチュートリアル](generate-video)で生成されたことを前提としています。もし別の方法で生成した場合は、両方のファイルの長さが同じであることを確認してください。

```python
audio_file = 'audio0.wav'
video_file = 'without_audio0.mp4'
```

ffmpegはマルチメディア処理フレームワークです。異なる種類のメディアを連結したいため、いくつかのオプションを指定する必要があります。

入力ファイルは `-i` オプションで設定できます。

コーデックはデータをエンコードするためのプログラム/ドライバです。出力ビデオのコーデックは入力のコーデックに従うことを希望するため、 `-c:v` (ビデオのコーデックオプション) を `copy` に設定し、 `-c:a` (音声のコーデックオプション) を `aac` (Advanced Audio Coding) に設定します。 AACは最大96kHzで使用でき 
、多くのプラットフォームで広く使用されています。

単一のメディアファイルには複数のストリームが含まれることがあります。そのため、`-map` オプションを使用して連結したいストリームを選択する必要があります。すべてのストリームを使用する場合、最初の2つの要素（ファイルのインデックスとメディアタイプ）を設定すれば十分です。

ffmpegのオプションは次のようになります：

```python
ffmpeg_option = ' -i without_audio0.mp4 -i audio0.wav -c:v copy -c:a aac -map 0:v:0 -map 1:a:0 concat.mp4'
```

`concat.mp4` は出力ファイルの名前です。

次に、subprocessを介してffmpegを実行します。

```python
proc0 = subprocess.Popen('ffmpeg' +  ffmpeg_option, shell=True, stdout=subprocess.PIPE, stderr=subprocess.STDOUT)
```

注意: `ffmpeg.exe` は、このチュートリアルコードを実行するディレクトリにあるか、環境変数 `PATH` に含まれている必要があります。

## 完全なコード

このチュートリアルで使用される完全なコードは[こちら](https://github.com/Sensing-Dev/tutorials/blob/main/python/tutorial4_concat_video_and_audio.py) にあります。