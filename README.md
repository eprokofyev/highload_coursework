# Курсовая работа по курсу HighLoad
## 1. Тема: Онлайн-кинотеатр
Сервис для просмотра фильмов по подписке.
## 2. Определение возможного диапазона нагрузок подобного проекта
В качестве примера онлайн-кинотеатра я выбрал [ivi.ru](http://ivi.ru). Месячная аудитория сервиса составляет 50 млн. уникальных пользователей, а общая длительность просмотров в 2019 году составила 1 млрд. часов.
## 3. Планируемая нагрузка
Основная нагрузка для сервиса это стриминг видео-файлов. Клиентами могут быть стационарные компьютеры, смартфоны и тв-приставки. Сервисом будет пользоваться примерно 50 млн. человек в месяц, общая длительность просмотров составит 70 млн. часов, а их суммарное количество 250 млн.
## 4. Логическая схема базы данных
Для проекта онлайн-кинотеатра выделим несколько основных сущностей: "users", "movies", "serials", "actors", "reviews" и "recommendations". Таблица "movies" содержит в себе данные о фильмах и об отдельных эпизодах сериалов. А в таблице "serials" предоставляет информацию о сериалах в целом, а не об отдельных сериях. Изначально схема базы данных была приведена к нормальной форме Бойса-Кодда, но была денормализована и в некоторые таблицы были внесены избыточные данные. Это сделано для уменьшения количества джойнов с крупными таблицами. Так, например, таблицы "reviews_serial", "reviews_movies" и "reviews_actors" дублируют информацию о пользователях из таблицы "users". В схеме также содержатся таблицы с рекомендациями для авторизованных пользователей на основе ранее просмотренных фильмов и для неавторизованных пользователей.
![alt text](./schema.png)
## 5. Физическая система хранения
Предположим, что выбор фильма для просмотра занимает у пользователя в среднем 5 переходов по сайту. На каждой странице происходят запросы рекомендаций по фильмам, получения по ним дополнительной информации и запросы пользовательских данных. Тогда пусть на каждый переход приходятся 10 запросов. Таким образом,

![\Large](https://latex.codecogs.com/svg.latex?\Large&space;rps=\frac{5*10*250,000,000}{30*24*60*60}\approx5000)

Сделаем поправку на то, что ночью трафик падает, а днём и вечером растет. Увеличим среднее RPS в 2.5 раза (~12,500). Основная часть запросов это рекомендации по фильмам (пусть это будет 80%, то есть 10,000).

Таблицы с рекомендациями будут размещаться в базе данных MongoDB на 4 серверах.
То есть в среднем на каждую машину будет приходиться по 2,500 RPS. Главными преимуществами MongoDB для моей задачи является шардинг, быстрый доступ и то, что данные для рекомендаций не ложатся на реляционную модель.  Пусть для каждого пользователя есть 100 рекомендаций фильмов. Размер одной рекомендации составляет 100 байт. Получается, что общий размер сервиса рекомендаций составляет 500 ГБ. На каждом сервере будет храниться 10 ГБ данных.

Информация связанная с пользователями, фильмами и сериалами будут находиться в PostgreSQL на 1 сервере с нагрузкой 2,500 RPS. Я выбрал PostgreSQL, т. к. необходимо надёжное хранение для важных данных, это обеспечивается за счёт реплик для каждого сервера. На одного пользователя приходится примерно 1 КБ места на диске. Сюда входит личная информация, данные о платежах и учёт просмотренных фильмов. Это число в дальнейшем будет только расти. Таким образом, для всех текущих пользователей необходимо 50 ГБ. Оценим место для хранения информации о фильмах и сериалах. На каждое видео приходится примерно 5 КБ данных. Сюда включено описание фильма, отзывы, список актеров и жанров. В нашем сервисе будет доступно 50,000 видеоматериалов. Суммарно получается 500 МБ данных связанных с фильмами. Суммарно выходит 5,5 ГБ данных.

Оценим размер библиотеки сервиса. Пусть половину видео составляют сериалы, а другую половину фильмы. Общий размер занимаемой памяти одним фильмом примерно 20 ГБ, а сериала 10 ГБ (сюда входят различные разрешения). Тогда размер библиотеки в 50,000 видео - 750 ТБ.

Информацию о сессиях будем хранить в Redis. Размер одной записи составляет примерно 50 байт. При условии, что у всех пользователей будет 5 активных сессий, размер базы данных составит 12.5 ГБ. 

## 6. Выбор прочих технологий: языки программирования, фреймворки, протоколы взаимодействия, веб-сервера и т.д.

**Бэкенд**: Языком разработки выберем Golang. Этот язык доказал, что отлично подходит для систем с высокой нагрузкой: concurrency из коробки, эффективное использование ресурсов, высокая скорость разработки, обширная стандартная библиотека, популярность языка.

**Фронтенд**: Выбираю самый распространенный стек технологий HTML,CSS, JavaScript.

**Мобильные клиенты**: Мобильное приложение для Android будет написано на языках Java и Kotlin. Kotlin является официальным языком разработки для платформы, но Java также необходим, т. к. на Java создано очень много библиотек для Android и далеко не все разработчики ещё освоили Kotlin. Приложение для iOS будет написано на языке Swift.

**Протоколы**: HTTPS - для безопасного соединения клиента и сервера. HTTP внутри датацентра. Protobuf - стандартный протокол для обмена сообщениями между микросервисами в фреймворке gRPC.

**Веб-сервер**: Выбираю nginx из-за быстроты его работы

## 7. Расчет нагрузки и потребного оборудования
1. PostgreSQL

   Сервер базы данных не требует больших затрат CPU и RAM, поэтому возьмём стандартную конфигурацию сервера и обеспечим запас по пользователям.

   CPU(cores) | RAM(GB) | SSD(GB)
   --- | --- | ---
   8 | 16 | 20


2. MongoDB

   Требования здесь аналогичные, но, чтобы разделить данные на 4 серверы, необходимо использовать шардинг по user_id.


   CPU(cores) | RAM(GB) | SSD(GB)
   --- | --- | ---
   8 | 16 | 300


3. Backend

   Единственное чем занимаются backend-сервера это ходят в БД за данными, поэтому их количество равно количеству серверов баз данных, то есть 5.


   CPU(cores) | RAM(GB) | SSD(GB)
   --- | --- | ---
   8 | 16 | 5


4. Frontend

   Для Frontend сервера самый важный CPU и диск для раздачи статики. На случай неполадок рядом будет располагаться такой же сервер. 


   CPU(cores) | RAM(GB) | SSD(GB)
   --- | --- | ---
   16 | 32 | 256


5. Redis

   Количество запросов будет соответсвовать суммарным запросам к backend - 12,500 RPS, поэтому достаточно двух серверов - основного и реплики

      CPU(cores) | RAM(GB) | SSD(GB)
   --- | --- | ---
   16 | 32 | 128

6. CDN

   Определим количество серверов для хранения контента. Основным ограничением здесь является пропускная способность.
   Сначала найдём среднее время просмотра фильмов одним пользователями в часах.

   ![\Large ](https://latex.codecogs.com/svg.latex?\Large&space;t=\frac{70,000,000}{250,000,000}=0.28)

   Теперь необходимо вычислить количество пользователей, одновременно использующих сервис в течение среднего времени просмотра

   ![\Large ](https://latex.codecogs.com/svg.latex?\Large&space;users=\frac{250,000,000*0,28}{30*24}\approx100,000)

   Скорость скачивания для качества 1080р составляет 4.5 МБит/с, для 720р - 2.77 МБит/с, для 576р - 1.83 МБит/с, 480р - 1.3 МБит/с, для 360р - 0.79 МБит/с. Тогда средняя скорость - 2.238 МБит/с. Рассчитаем пропускную способность в ГБит/с, которую необходимо обеспечить с учётом поправки на неравномерность трафика.

   ![\Large ](https://latex.codecogs.com/svg.latex?\Large&space;speed={5*100,000*2.24}\approx1120)

   Один сервер может отдавать примерно 40 ГБит/с. Но пропускная способность это нестабильный ресурс, поэтому обеспечим 50% резервирование. Таким образом, для онлайн-просмотра необходимо 42 сервера.

   Раздавать контент будем с помощью технологии CDN. Для этого распределим контент на 5 origin-серверов, но т. к. размер библиотеки велик, то необходим шардинг. В конечном итоге библиотека будет находиться на 5 origin-кластерах с 4 серверами с данными в каждом. Библиотека будет располагаться на SSD для быстрого доступа к фильмам. Для надёжного хранения используем схему RAID 10. Тогда один сервер будет обладать следующими характеристиками

   CPU(cores) | RAM(GB) | SSD(GB)(RAID 10)
   --- | --- | ---
   8 | 16 | 90 * 4096

   На каждом контент-сервере будет храниться 2/3 видеотеки ~ 500 ГБ. В "горячем" состояние будем держать 10% видеотеки
   CPU(cores) | RAM(GB) | HDD(GB)(RAID 0) | SSD(GB)(RAID 0)
   --- | --- | --- | ---
   8 | 16 | 125 * 4096 | 4 * 4096


Итоговая таблица
Назначение | Количество | CPU(cores) | RAM(GB) | SSD(GB) | HDD(GB)
| --- | --- |  --- | --- | --- | ---
PostgreSQL | 2 * 1 | 8 | 16 | 25 |
MongoDB | 4 + 1 | 8 | 16 | 300
Backend | 5 | 8 | 16 | 16
Frontend | 2 * 1 | 16 | 32 | 256
Redis | 2 * 1 | 16 | 32 | 128
Origin-сервер | 20 | 8 | 16 |  90 * 4096
Контент-сервер | 42 | 8 | 16 | 125 * 4096 | 4 * 4096


## 8. Выбор хостинга/облачного провайдера и расположения серверов
Для хостинга серверов баз данных, бэкенда и фронтенда выберем MCS. MCS обладает всеми удобствами и функционалом современного облачного провайдера. Помимо этого сервис имеет множество серверов на территории России.

Контент-сервера будут располагаться в крупных городах России в сетях провайдеров. Но основная часть также будет находиться в Москве в облаке MCS.


## 9. Схема балансировки нагрузки (входящего трафика и внутрипроектного, терминация SSL)
Для балансировки API используем nginx. В nginx реализованы стандартные алгоритмы балансировки, предусмотрена терминация SSL и это самый распространенный веб-сервер - без труда можно найти специалистов.

Для эффективной работы CDN будем использовать Geo-Based DNS для этого разделим территорию России на 6 зон: Дальный Восток - 2 сервера в Хабаровске, Сибирь - 4 сервера в Новосибирске, Урал - 4 сервера в Екатеринбурге, Юг - 7 серверов в Ростове-на-Дону, Север - 4 сервера в Санкт-Петербурге и Центр - 21 сервер в Москве. Для каждой зоны укажем IP всех контент-серверов относящихся к ней и по алгоритму round robin будем распределять нагрузку внутри зоны.

## 10. Обеспечение отказоустойчивости
* Отказоустойчивость PostgreSQL будем обеспечивать за счёт реплик исходной базы данных.
* Для бэкенд серверов воспользуемся преимуществами kubernetes - в случае падения одного из бэкендов на его место тут же появится новый. Необходимо ещё настроить тайм-ауты в nginx и rate limit на запросы.
* Устойчивость фронтенд-сервера обеспечивается за счёт CARP протокола с запасным фронтенд-сервером.
* Видеофайлы хранятся на серверах с массивом RAID 10, который обеспечивает высокую отказоустойчивость и производительность
* Настройка мониторинга сервисов и отслеживание основных показателей нагрузки
* Для обеспечения отказоустойчивости контент-серверов используем CARP протокол, тогда в случае падения сервера его запросы берёт на себя другой.