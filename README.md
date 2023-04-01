# yolov5_khadas

# **Процесс установки всего необходимого для Khadas Vim3 Pro**

# Шаг 1

*Прежде чем начнём, проверь версию Galcore (Драйвер общения с NPU):*

```
dmesg | grep Galcore
```

*Зачем?*

*Вскоре нам понадобится устанавливать TIM-VX, и нам нужна будет соответствующая версия*

# Шаг 1.1

**На всякий случай если у нас нет Cmake:**

```
sudo apt-get install cmake
```

# Шаг 2

*Сперва установи библиотеку OpenCV:*

```
sudo apt install libopencv-dev
```
*На стороннем устройстве (мой ноутбук) пришлось также ставить библиотеку Protobuf:*
```
sudo apt-get install protobuf-compiler libprotobuf-dev
```

# Шаг3

*Теперь скачаем репозитории:*

1)
```
git clone https://github.com/OAID/Tengine tengine-lite
```

2)
```
git clone https://github.com/VeriSilicon/TIM-VX.git
```

# Шаг 4

*Переносим данные из* **TIM-VX** *в наш проект:*

```
cd tengine-lite
```

```
cp -rf ../TIM-VX/include ./source/device/tim-vx/
```

```
cp -rf ../TIM-VX/src ./source/device/tim-vx/
```

# Шаг 5

*А вот теперь займёмся моментами, связаннами с нашей версией* **Galcore**:

```
wget -c https://github.com/VeriSilicon/TIM-VX/releases/download/v1.1.34.fix/aarch64_A311D_6.4.8.tgz
```

*У меня версия Galcore* **6.4.8.7.1.1.1**, *в твоём случае лучше перейди на* [сайт](https://github.com/VeriSilicon/TIM-VX/releases) *и найди нужную тебе версию*

```
tar zxvf aarch64_A311D_6.4.8.tgz
```

```
mv aarch64_A311D_6.4.8 A311D
```

*Переименуем папку, чтобы с ней было легче работать, кстати* **Amlogic A311D** *это модель процессора, который стоит внутри Khadas Vim3* **Pro**, *а вот уже в Khadas Vim3* **L** *стоит процессор Amlogic S905D3, и при работе с ним естественно нужно брать* **aarch64_S905D3_6.4.8.tgz**

```
cd tengine-lite && mkdir -p ./3rdparty/tim-vx/include
```

```
cp -rf ../A311D/include/* ./3rdparty/tim-vx/include/
```

*Переносим данные 3rdparty к файлам TIM-VX в нашем проекте.*

# Шаг 6

*А теперь время компиляции:*

```
cd tengine-lite
```

```
mkdir build && cd build
```

*Не забываем указать Makefile-у чтобы он установил всё нужное для проектов в формате TIM-VX также уже на другом устройстве (не одноплатник Кадаса) устанавливаем Convert и Quant Tools:*

```
cmake -DTENGINE_ENABLE_TIM_VX=ON..
make -j`nproc` && make install
```
*(для кадаса)*

```
cmake -DTENGINE_BUILD_CONVERT_TOOL=ON -DTENGINE_BUILD_QUANT_TOOL=ON ..
make -j`nproc` && make install
```
*(Для стороннего устройства, где вы будете квантовать веса)*


# **Теперь у нас есть собранный tengine-lite проект**

**Как им пользоваться?**

*Внутри можно создать папки models и images, где будут храниться веса моделей и фотографии. Внутри build есть папка examples (рекомендую запускать модели именно оттуда) и есть папка install/bin, в которой также хранятс модели и 3 важных нам файла - convert_tool, quant_tool_int8 и quant_tool_uint8, предназначенных для конвертации весов любого формата в формат .tmfile.*

*А это значит можно найти модель с такой же архитектурой как на С++, получить веса в любом формате, а потом их просто конвертировать.*

```
./convert_tool -h
```
*для получения информации*

```
./convert_tool -f onnx -m net.onnx -o net.tmfile
```
*для конвертации модели*

*Подробнее [тут](https://github.com/OAID/Tengine/blob/tengine-lite/doc/docs_en/user_guides/convert_tool.md)*

```
./quant_tool_uint8 -h
```
 *для получения информации*

```
./quant_tool_uint8 -m ./net.tmfile -i ./dataset -o ./net_uint8.tmfile -g 3,640,640 -w 104.007,116.669,122.679 -s 0.017,0.017,0.017
```
*для квантования модели*

*Подробнее* [тут](https://github.com/OAID/Tengine/blob/tengine-lite/doc/docs_en/user_guides/quant_tool_uint8.md)

# **Как только мы получили веса для нашей готовой модели, самое время её запустить:**

```
./build/examples/net -m ./models/net_uint8.tmfile -i ./images/example.jpg -r 1 -t 1
```

**Вот и всё**

[Тут](https://github.com/OAID/Tengine/blob/tengine-lite/examples/README_EN.md) *более подробная документация.*

[Тут](https://drive.google.com/drive/folders/1hunePCa0x_R-Txv7kWqgx02uTCH3QWdS?usp=sharing) *гугл-диск с моделями и фотками для тренировок.*
