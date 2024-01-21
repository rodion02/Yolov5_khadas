<h4 style="color:Black;">Данное руководство предназначенно для работы с khadas Vim3 Pro и запуска на этой модели нейронных сетей при помощи фреймфорка</h4>
<h1 style="text-align:center;color:Gray;">KSNN</h1>

<h4 style="text-align:justify">Оглавление:</h4>

<h6>Запуск примера</h6>
<h6>Работа с кастомным датасетом<h6>
<h6>Конвертация весов</h6>
<h6>Ускорение алгоритма</h6>

<br>

<h4 style="color:black;">Запуск примера</h4>
<details open>
<summary></summary>
Для начала нужно установить на khadas последнюю версию ubuntu с <a href="https://docs.khadas.com/products/sbc/vim3/os-images/start">официального сайта</a>.
Как только у нас появился образ, давайте установим все необходимые репозитории (он всего один):
<p><a href="https://github.com/khadas/ksnn"><img src=https://media.geeksforgeeks.org/wp-content/cdn-uploads/20191120192429/Top-10-Useful-Github-Repos-That-Every-Developer-Should-Follow.png style="width:30%;height:30%;"></a></p>

```sh
$ git clone https://github.com/khadas/ksnn
```
Следуем инструкциям:

```sh
$ sudo apt install python3-pip
$ pip3 install matplotlib
```
На данный момент актуальной является 1.4 версия, так что я поставил её
```sh
$ pip3 install ksnn/ksnn-1.4-py3-none-any.whl
```
В репозитории ksnn есть папка examples с примерами различных сеток, нас тут интересует yolov8n (полный путь ksnn/examples/yolov8n/). yolov8n-picture.py для одиночных изображений, а yolov8n-cap.py уже для потока видео с камеры. В папках ./libs/ и ./models/VIM3/ находятся примеры скомпилированной библиотеки - libnn_yolov8n.so и конвертированных весов оригональной модели yolov8n - yolov8n.nb.
yolov8n-picture.py:
```sh
$ python3 yolov8n-picture.py --model ./models/VIM3/yolov8n.nb --library ./libs/libnn_yolov8n.so --picture ./data/horses.jpg
```
yolov8n-cap.py
```sh
$ python3 yolov8n-cap.py --model ./models/VIM3/yolov8n.nb --library ./libs/libnn_yolov8n.so --device X
```
В параметр **model** мы передаем путь к .nb файлу
В параметр **library** мы передаем путь к .so файлу
**picture** ожидает путь к изображению, а **device** номер камеры
</details>

<h4 style="color:black;">Работа с кастомным датасетом</h4>
<details open>
<summary></summary>
Для обучения и последующей конвертации нам потребуется стороннее устройство, я лично использовал ноутбук с Ubuntu 22.04.

<h5 style="color:black;">Yolov8 Ultralitics</h5>

Для начала необходимо установить Yolov8:
<p><a href="https://github.com/ultralytics/ultralytics"><img src=https://media.licdn.com/dms/image/D4D22AQE514fzn-GsXQ/feedshare-shrink_800/0/1697097434163?e=2147483647&v=beta&t=Ngp5prtlbZom-T_XHPR1T4n-7PPBQSem6WS5-wgfq7U style="width:50%;height:50%;"></a></p>
Затем следуем инструкциям с <a href="https://docs.khadas.com/products/sbc/vim3/npu/ksnn/demos/yolov8n">официального сайта</a> и меняем по инструкции файл head.py (полный путь ultralytics/nn/modules/head.py) 

42 строчка (добавляем 2 строки):

```sh
def forward(self, x):
    """Concatenates and returns predicted bounding boxes and class probabilities."""
    if torch.onnx.is_in_onnx_export():
        return self.forward_export(x)
```

84 строчка (добавляем новый метод):

```sh
def forward_export(self, x):
    results = []
    for i in range(self.nl):
        dfl = self.cv2[i](x[i]).contiguous()
        cls = self.cv3[i](x[i]).contiguous()
        results.append(torch.cat([cls, dfl], 1))
    return tuple(results)
```

Или можно вставить уже измененный <a href="https://github.com/rodion02/yolov5_khadas/blob/KSNN/ultralytics/head.py">файл</a>.

Теперь нужно установить библиотеку Yolov8
```sh
pip install ultralytics
```

Теперь начнём обучение:
```sh
from ultralytics import YOLO

# Загружаем модель
model = YOLO('yolov8n.pt')  # берем предобученную
```
Обучать можно как на GPU:
```sh
results = model.train(data='coco128.yaml', epochs=100, imgsz=640, batch=4, imgsz=640, device=0)
```
(по умолчанию batch=16, так что если система не особо производительная, то лучше его взять меньше)

Так и с помощью CPU:
```sh
results = model.train(data='coco128.yaml', epochs=100, imgsz=640)
```

Более подробно всё описано <a href="https://docs.ultralytics.com/modes/train/#resuming-interrupted-trainings">тут</a>.

<h5 style="color:black;">Конвертация весов</h5>
<details open>
<summary></summary>

После обучения мы получим .pt веса, которые нам нужно конвертировать в .onnx
```sh
from ultralytics import YOLO
model = YOLO("./runs/detect/train/weights/best.pt")
results = model.export(format="onnx")
```

Теперь по <a href="https://docs.khadas.com/products/sbc/vim3/npu/ksnn/ksnn-convert">инструкции</a> устанавливаем репозиторий:

<p><a href="https://github.com/khadas/aml_npu_sdk"><img src=https://media.geeksforgeeks.org/wp-content/cdn-uploads/20191120192429/Top-10-Useful-Github-Repos-That-Every-Developer-Should-Follow.png style="width:30%;height:30%;"></a></p>

```sh
$ git clone --recursive https://github.com/khadas/aml_npu_sdk.git
```
Переходим в
```sh
$ cd aml_npu_sdk/acuity-toolkit/python
```
тут нас интересует convert, с помощью которого мы и будем конвертировать наши веса.

Сразу подготовьте .onnx файл с весами и папку с колибрационным набором изображений для датасета (5% от всего набора, я беру не больше 100) и .txt файл, в котором будут прописаны пути к каждому изображению (.../data/image1.jpg итд)

```sh
$ ./convert \
--model-name yolov8n \
--platform onnx \
--model ./yolov8n.onnx \
--mean-values '0 0 0 0.00392156' \
--quantized-dtype asymmetric_affine \
--source-files ./data/dataset.txt \
--kboard VIM3 \
--print-level 1
```
**model-name** - название для выходного файла
**platform** - формат весов (для yolo это onnx)
**model** - путь к весам
**mean-values** - средние значения R, G и B каналов по вашему датасету. Про то, как их вычислить можно прочитать <a href="https://kozodoi.me/blog/20210308/compute-image-stats">тут</a>, но помните что эти значения.
**quantized-dtypes** - способы квантования (оставьте лучше тот, что в примере).
**source-files** - путь к файлу с путями к изображениям из колибрационного датасета.

В результате мы получим yolov8n.nb и yolov8n.so файлы в папке outputs (полный путь aml_npu_sdk/acuity-toolkit/python/outputs)

Теперь можно запустить и посмотреть на примеры работы алгоритма с вашими данными как в самом начале.
</details>
</details>

<h4 style="color:black;">Ускорение алгоритма</h4>
<details open>
<summary></summary>

Основной алгоритм обрабатывает изображение за 300ms, из которых 260ms уходят на post-process. Я это ускоряю при помощи библиотеки **numba** и некоторого изменения исходного кода.
Чтобы запустить мой алгоритм необходимо передать ему некоторые параметры:

```sh
$ python3 ./khadas_stream.py \
--library ./yolov8n.so \
--model ./yolov8n.nb \
--source 0 \
--visualize 0 \
--conf ./data.json \
--level 1
```
**library** - путь к .so библиотеке.
**model** - путь к .nb весам.
**source** - либо номер камеры (0, 1 итд) или ссылка на stream/ip камеру.
**visualize** - демонмтрировать ли результат работы камеры, по умолчанию False, чтобы не тпатить на это ресурсы.
**conf** - путь к конфигурационному файлу .json, который содержит имена классов и доп. настройки.
data.json:
```
{
    "CLASSES":["class1", "class2'...],
    "SETTINGS":{
        "SPAN": 1, 
        "MAX_BOXES": 500, 
        "OBJ_THRESH": 0.4, 
        "NMS_THRESH": 0.5}
}
```
То есть для внесения изменений вам не нужно трогать .py файл, достаточно внести изменения в .json файл.
С файлом обязательно должен быть res.py так как при запуске алгоритм компилируется и ему нужны данные из этого Или можно вставить уже измененный <a href="https://github.com/rodion02/yolov5_khadas/blob/KSNN/yolov8n/khadas_stream.py">файл</a>.
</details>