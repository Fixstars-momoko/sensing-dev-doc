---
sidebar_position: 3
---

# 画像の表示

このチュートリアルでは、ion-kitを使用してデバイスから画像データを取得し、OpenCVで表示する方法に
ついて学びます。

## 前提条件

* OpenCV（sensing-dev SDKと共にインストール済み）
* ion-kit（sensing-dev SDKと共にインストール済み）

## チュートリアル

### デバイス情報の取得

ionpyを使用して画像を表示するには、デバイスの次の情報を取得する必要があります。

* 幅
* 高さ
* ピクセルフォーマット

[前のチュートリアル](obtain-device-info.md)または[arv-tool-0.8](../../external/aravis/arv-tools.md)を参照してこれらの値を取得するのに役立ちます。

### パイプラインの構築

[イントロ](../intro.mdx)で学んだように、画像の入出力と処理のためのパイプラインを構築して実行します。 

このチュートリアルでは、U3Vカメラから画像を取得する唯一のビルディングブロックを持つ非常に単純な
パイプラインを構築します。

次のionpy APIは、パイプラインを設定します。

```c++
#define MODULE_NAME "ion-bb"
...
// パイプラインの設定
Builder b;
b.set_target(Halide::get_host_target());
b.with_bb_module(MODULE_NAME);
```

`set_target`は、ビルダーによって構築されたパイプラインが実行されるハードウェアを指定します。  

`ion-bb.dll` で定義されたBBを使用したいので、 `with_bb_module` 関数でモジュールをロードする必要
があります。

ここで使用するBBは `image_io_u3v_cameraN_u8x2` で、これは各ピクセルデータが8ビット深度で2次元の
U3Vカメラ向けに設計されています（たとえば、Mono8）。

デバイスのピクセルフォーマットがMono10またはMono12の場合は、それぞれ10ビットおよび12ビットのピ 
クセルデータを格納するために16ビット深度のピクセルが必要なので `image_io_u3v_cameraN_u16x2` を使
用する必要があります。

ピクセルフォーマットがRGB8の場合、ビットの深度は8で、次元は3です（幅と高さに加えて、カラーチャネルがあります）、そのため`image_io_u3v_cameraN_u8x3`を使用します。

これらのBBのいずれも、入力として `dispose`、`gain`、および `exposuretime` と呼ばれるものを必要 
とするため、これらの値をパイプラインに渡すためにポートを設定する必要があります。

```c++
// ポートの設定
Port dispose_p{ "dispose",  Halide::type_of<bool>() };
Port gain_p{ "gain", Halide::type_of<double>(), 1 };
Port exposure_p{ "exposure", Halide::type_of<double>(), 1 };
```

ポートの入力は動的です。つまり、各実行ごとに更新できます。入力値を文字列で静的に設定するには、 
`Param` を使用できます。

```c++
#define FEATURE_GAIN_KEY "Gain"
#define FEATURE_EXPOSURE_KEY "ExposureTime"
...
Param num_devices{"num_devices", std::to_string(num_device)};
Param pixel_format{"pixel_format_ptr", pixel_format};
Param frame_sync{"frame_sync", "true"};
Param gain_key{"gain_key", FEATURE_GAIN_KEY};
Param exposure_key{"exposure_key", FEATURE_EXPOSURE_KEY};
Param realtime_diaplay_mode{"realtime_diaplay_mode", "false"};
```

`pixel_format` は[デバイス情報の取得](#get-device-information)で取得したピクセルフォーマットで 
す。

::::caution
`gain_key` と `exposure_key` はデバイスのゲインと露光時間を制御するためのGenICamのフィーチャキ ーです。通常、これらはemvaによる**SFNC（Standard Features Naming Convention）**で `Gain` および `ExposureTime` として設定されていますが、一部のデバイスには異なるキーがあるかもし れません。

その場合、パラメータのキーを変更する必要があります。[このページ](../external/aravis/arv-tools#list-the-available-genicam-features)を参照して、利用可能なフィーチャをリストアッ 
プする方法を確認してください。

```c++
#define FEATURE_GAIN_KEY <デバイスのゲインを制御するフィーチャの名前>
#define FEATURE_EXPOSURE_KEY <露光時間を制御するフィーチャの名前>
```
::::

これで、ポートとパラメータを使用してパイプラインにBBを追加します。

```c++
Node n = b.add(bb_name[pixel_format])(dispose_p, gain_p, exposure_p)
    .set_param(
    num_devices,
    pixel_format,
    frame_sync,
    gain_key,
    exposure_key,
    realtime_diaplay_mode
    );
```

これが私たちのパイプラインの唯一のBBであるため、ノードの出力ポートはパイプラインの出力ポートに 
なり、その名前は `output_p` です。

BBとポートを含む私たちのパイプラインは次のようになります。

![tutorial1-pipeline](../img/tutorial1-pipeline.png)

入力値を渡し、ポートから出力データを取得するためには、入力用のバッファを用意し、そのバッファを 
入力用のポートにマッピングし、出力用のポートにバッファをマッピングする必要があります。

```c++
// 入力ポートのためのHalideバッファを作成
double *gains = (double*) malloc (sizeof (double) * num_device);
double *exposures = (double*) malloc (sizeof (double) * num_device);
for (int i = 0; i < num_device; ++i){
    gains[i] = 40.0;
    exposures[i] = 100.0;
}
Halide::Buffer<double> gain_buf(gains, std::vector< int >{num_device});
Halide::Buffer<double> exposure_buf(exposures, std::vector< int >{num_device});

// 出力ポートのためのHalideバッファを作成
std::vector< int > buf_size = std::vector < int >{ width, height };
if (pixel_format == "RGB8"){
    buf_size.push_back(3);
}
std::vector<Halide::Buffer<T>> output;
for (int i = 0; i < num_device; ++i){
    output.push_back(Halide::Buffer<T>(buf_size));
}

// I/Oポートの設定
PortMap pm;
pm.set(dispose_p, false);
pm.set(gain_p, gain_buf);
pm.set(exposure_p, exposure_buf);
pm.set(n["output"], output);
```

注意：ここでの`buf_size`は2Dイメージ用に設計されています。ピクセルフォーマットがRGB8の場合、カラーチャネルを追加するには`(width、height、3)`を設定する必要があります。

`T`は出力バッファのタイプです。たとえば、Mono8およびRGB8用に`uint8_t`、Mono10およびMono12用に`uint16_t`です。

### パイプラインの実行

パイプラインは実行の準備ができています。

チュートリアルコードでは、ゲインおよび露光時間の値を入力ポートにマッピングするのはオプションで 
すが、デバイスを破棄するには実行中に `false` に設定し、プログラムの実行が終了したら `true` に設 
定する必要があります。これにより、デバイスが安全に閉じられます。

```c++
for (int i = 0; i < loop_num; ++i){
    pm.set(dispose_p, i == loop_num-1);
    b.run(pm);
    ...
}
```

動的ポートを設定した後、 `b.run(pm);` を使用してパイプラインを実行します。

### OpenCVでの表示

出力データ（つまり、画像データ）が**Bufferのベクトル**にマップされているので、これをOpenCVバッ 
ファにコピーして画像処理または表示を行うことができます。

注意すべきは、OpenCVはバッファのチャネル（次元）の順序が異なることです。

```c++
for (int i = 0; i < num_device; ++i){
cv::Mat img(height, width, opencv_mat_type[pixel_format]);
std::memcpy(img.ptr(), output[i].data(), output[i].size_in_bytes());
img *= positive_pow(2, num_bit_shift_map[pixel_format]);
cv::imshow("image" + std::to_string(i), img);
}
```

`opencv_mat_type[pixel_format]` は、画像データのPixelFormat（ビットの深さと次元）に依存します。
例えば、Mono8の場合は `CV_8UC1` 、RGB8の場合は `CV_8UC3` 、Mono10およびMono12の場合は `CV_16UC1` です。

`output[i]` をOpenCV Matオブジェクトにコピーして表示できます。

```c++
cv::imshow("image" + std::to_string(i), img);
cv2.waitKey(1)
```

注意すべきは、 `output` はBufferのベクトルです（複数のデバイスを制御するために `num_device` を 
`1` より大きく設定できることを考慮しています）。画像データを取得するには、 `outpus[0]` にアクセ 
スします。

このプロセスを `for` ループで繰り返すと、カメラデバイスからの連続した画像が正常に表示されます。

:::tip カメラインスタンスが正確に解放されるのは
v23.11.01では、各外部関数に登録されたdispose()関数を呼び出すことで、u3vクラスを明示的に破棄しています。
ビルディングブロックは、デバイスを明示的に閉じるためにdisposeに入力ポートが必要です。
正確なタイミングを観察するには、ユーザーはWindowsコマンドラインでset ION_LOG_LEVEL=debug、またはUnixターミナルでexport ION_LOG_LEVEL=debugを設定できます。ユーザーは、ターミナルで以下の行を見た場合、aravis経由でカメラにアクセスできます:s
```
[2024-02-14 08:17:27.790] [ion] [debug] U3V::dispose() :: is called
[2024-02-14 08:17:27.791] [ion] [debug] U3V::dispose() :: AcquisitionStop
[2024-02-14 08:17:28.035] [ion] [debug] U3V::dispose() :: g_object_unref took 244 ms
[2024-02-14 08:17:28.109] [ion] [debug] U3V::dispose() :: g_object_unref took 72 ms
[2024-02-14 08:17:28.110] [ion] [debug] U3V::dispose() :: Instance is deleted
```
上記のデバッグ情報から、ユーザーはカメラインスタンスの解放にかかる時間を知ることができます。
:::

## 完全なコード

チュートリアルで使用された完全なコードは[こちら](https://github.com/Sensing-Dev/tutorials/blob/v23.11.01/cpp/src/tutorial1_display.cpp)にあります。


プログラムをコンパイルおよびビルドするために[こちら](https://github.com/Sensing-Dev/tutorials/blob/v23.11.01/cpp/CMAKELists.txt)で提供されているCMakeLists.txtを使用できます。