# Image captioning

Проект по построению и практическому использованию модели автоматической 
текстовой аннотации
изображений для поиска непохожих моментов в видео.

## Идея

1. С помощью нейронной сети создается аннотация кадра из видео
2. Аннотация сравнивается с аннотациями предыдущих кадров видео и считается
некоторая метрика непохожести текущего кадра на предыдущие

## Имплементация

1. Для логирования результатов экспериментов
(разные размерности эмбеддингов слов, скрытых слоев и т.д.) и соответствующих
кодов использовался [neptune](https://neptune.ai/)
2. Модель, аннотирующая изображения была написана с нуля на pytorch, 
в качестве архитектуры 
использовалась модель с soft attention из статьи 
[Show, Attend and Tell](https://arxiv.org/abs/1502.03044).
3. В качестве датасета использоваться 
[Microsoft COCO](https://cocodataset.org/#home)
3. Изображения кодировались с помощью Resnet 152 и 
затем подавались в однонаправленную LSTM
4. Для создания интерфейса была использована следующая связка:
- flask сервер с моделью, обрабатывающий все запросы и 
возвращающий текстовые описания, закодированные сверточной сетью
изображения и эмбеддинги слов из текстовых описаний.
- shiny app для создания интерфейса
- flask сервер и shiny app завернуты в докер контейнеры
- докер контейнеры поднимаются с помощью docker compose up, 
с уже прописанным пробросом необходимых портов


## Результаты

При обучении модели, аннотирующей изображения, удалось достичь BLEU4 = 20.14.
При этом для многих изображений модель выдает адекватные описания\
Примеры:
1. A group of cows are grazing in a field
![A group of cows are grazing in a field](https://sun9-31.userapi.com/impg/pb6wXXMsVoh1DtPo_pE-3ZnDOSskyLQ5PkjAgw/z7pmZYR-EcM.jpg?size=2560x1440&quality=96&sign=d9f1451eefca4918d0945d0f453a5ca3&type=album)
2. A city street with a car parked on the side of the road
![A city street with a car parked on the side of the road](https://sun9-59.userapi.com/impg/7zIOiTroHQiYArLx_c3_jpCMlN8D_RHd5T-1Mw/iLM1FbDC9eU.jpg?size=913x567&quality=96&sign=c5e4ceb6165880a72fa19658c077dad3&type=album)

При этом для изображений под углом часто получаются плохие описания. Это происходит, потому что модель
была обучена на изображениях, снятых параллельно земле.

Пример:
1. A train on a track with a train on it
![A train on a track with a train on it](https://sun9-41.userapi.com/impg/wEHEAlN7J0IT6SNXBmZK8skNKrmTHqc578qZ1A/5sQR7A9pyHQ.jpg?size=1920x1080&quality=96&sign=689f532cab1c5fb9a3c9a30ee5d59078&type=album)


При анализе видео для нахождения непохожих друг на друга кадров 
используются не сами слова из текстовых аннотаций, а эмбеддинги слов, т.к. 
для большинства кадров слова в описании не повторяются с предыдущим кадром,
поэтому текстовый анализ невозможен. Эмбеддинги же представляют собой вектор,
при этом обладающий смыслом (king - man + women = queen и тд). Поэтому,
для похожих кадров, имеющих разное описание, эмбеддинги слов все равно близки друг к
другу.

С помощью алгоритма удается находить непохожие кадры, но при этом многие похожие кадры также
отмечаются как похожие. Скорее всего это происходит из-за того, что сеть чувствительна к незначительным 
изменения изображения, то есть выдает кардинально другое описание для похожего изображения.


## Репозиторий

### Имеющиеся файлы

- *dataset.py* pytorch датасет для использования в loaders
- *model.py* pytorch модель
- *model_training.py* функции с train и validation loops
- *training.ipynb* ноутбук с обучением модели и логированием параметров
- *utils.py* вспомогательные функции для создания датасета, загрузки и логирования 
моделей
- *server/docker-compose.yaml* файл для поднятия сервисов через docker compose up
- *server/captioning_model/Dockerfile* докер файл для сервиса с моделью
- *server/captioning_model/model.py* pytorch модель, которая будет работать на сервере
- *server/captioning_model/requirements.txt* библиотеки, необходимые для работы сервера
с моделью
- *server/captioning_model/server.py* flask сервер с моделью
- *server/captioning_model/server_utils.py* вспомогательные функции для работы сервера
- *server/shiny_app/app.R* shiny app для интерфейса
- *server/shiny_app/Dockerfile* докер файл для сервиса с shiny app

### Инструкция по работе

Для обучения модели:
1. В файле *training.ipynb* в 4 ячейке поставить путь к своему
проекту на neptune, в 5 строчке прописать свои ключи для neptune. Все это нужно для
логгирования результатов

2. Создать тренировочные файлы в 6 ячейке, задав название датасета 
(COCO, flickr30k или flickr8k)

3. Загрузить необходимые словари, указав в 7 строке название датасета с _ в начале

4. Инициализировать/загрузить модель, указав необходимые параметры. 
Если checkpoint_name = None, то модель инициализируется, если checkpoint_name указан,
то модель загрузится из указанного файла

5. Инициализировать загрузчики данных (можно изменить размер батчей)

6. Обучить модель, предварительно создав эксперимент в neptune для логгирования

После обучения в папке models будут лежать чекпоинты моделей

Для запуска сервисов:

1. Находясь в /server поднять сервисы используя команду docker compose up.

Предварительно
необходимо изменить пути в docker-compose.yaml на свои. Эти пути используются для маунта 
папок из контейнера, что позволяет синхронизировать указанные директории между хост машиной
(компьютер на котором поднимается докер контейнер) и самим докер контейнером.

Также можно изменить порты на хост машине 
в формате <порт на хост машине>:<порт докер контейнера>

Можно настроить количество оперативной памяти, выделяемой для сервисов.

2. После того, как сервисы поднимутся можно получить к ним доступ: для интерфейса shiny app
по адресу localhost:<порт на хост машине> (по умолчанию 3838), для flask сервера с моделью
по адресу localhost:<порт на хост машине> (по умолчанию 8888).

В интерфейсе shiny app необходимо выбрать видео для анализа (.mp4), с каким количеством предыдущих
кадров необходимо сравнивать текущий и каждую какую секунду видео вырезать в качестве кадра. 
После того как пройдет
загрузка, будет выведен график непохожести (подсчитана как евклидово расстояние
между средним эмбеддингов текущего кадра и средним эмбеддингов предыдущих N кадров )
текущего момента на предыдущие.

При работе с flask сервером:
- по адресу */get_captions* можно загрузить картинку в формате png/jpg (с 3 каналами)
и получить ее описание
- по адресу */get_video_captions* можно загрузить mp4 видео с названием файла video 
и получить текстовое описание 
кадров и их непохожесть на предыдущие. Доступные параметры: every_n_second - каждую какую
секунду видео необходимо описывать, metric - какую метрику непохожести считать ("euclidian"
 для евклидова расстояни и "cosine" для метрики, обратной cosine similarity),
 rolling_window_size - с каким количеством предыдущих кадров сравнивать текущий (то есть
 считать метрику непохожести)
 - по адресу */get_image_caption* можно отправить картинку в запросе с файлом image в формате
 png/jpg (с 3 каналами) и ответе получить текстовое описание картинки, эмбеддинги слов
 и выход сверточной нейронной сети.
 
 ## Участники проекта
- Илья Авилов (www.linkedin/in/ieavilov) - написание и обучение модели, написание flask сервера, обертка flask сервера и shiny
app в docker и поднятие сервисов
- Варвара Захарченко (ruslan.basyrov.id@gmail.com) - shiny app
- Семен Забродин (semenzabrodin@mail.ru) - shiny app
- Руслан Басыров (ruslan.basyrov.id@gmail.com) - shiny app, связывание shiny app с flask сервисом с помощью запросов, 
нарезка видео по кадрам