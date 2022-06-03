# highload_chat_design

## 1. Тип сервиса, функционал MVP и целевая аудитория

Тип сервиса: мессенджер

MVP:
- отправка сообщений
- список чатов
- каналы
- группы
- диалоги
- история сообщений
- регистрация по номеру телефона

Целевая аудитория:
- РФ и страны СНГ (10 мин в день)
- Европа
- Северная и Южная Америка
- Южная Азия

## 2. Расчет нагрузки
### Продуктовые метрики
 - Месячная аудитория: 500 млн активных пользователей
 - Дневная аудитория: 350млн со всего мира, 40 млн активных пользователей из России
 - Средний размер хранилища пользователя:
    * Данные о пользователе (номер телефона, никнейм, дата регистрации, последняя активность, аватар (в ср. 1-2 фотографии)) - 11Б + 10Б + 20КБ + 20КБ = 40кБ
    * Сообщения пользователя (40 сообщений в день 35 текстовых + 3 голосовых + 2 медиа) = (35 * 50Б) + (3*160кБ (1мин)) + (2*250кБ) = 880+1750 = 3630кБ
    * Сообщения пользователя будем хранить всегда (5 лет) - 3630 * 182*10 = 6гБ
    * Список чатов, групп и каналов пользователя [(Название элемента) 15Б + (последнее сообщение) 20Б + (время) 5Б] * 20 = 800Б = 0,8кБ
    * Всего на одного пользователя: 6гБ
   - Среднее количество действий пользователя по типам в день (типы запросов выбираем на основании выбраного функционала MVP) Оформить в виде сводной таблицы

     Действие                                          | Среднее кол-во в день
     ------------------------------------------------- | -------------
     Отправка сообщения (в том чилсе пост в канале)    | 20
     Получение сообщения                               | 40

     Действие             | Среднее кол-во в месяц
     -------------------- | ----------------------
     Создание чата        | 1
     Добавление в чат     | 2
   
### Технические метрики

 - Размер хранения в разбивке по типам данных (в Тб) - для существенных блоков данных

    На одного пользователя за 5 лет требуется хранить 6ГБ, соответственно на 500 млн пользователей:
    6ГБ * 500млн = 3 000 000 ТБ
    Для России и СНГ, соответственно на 40 млн пользователей:
    6ГБ * 40млн = 234 000 ТБ
    Файлы будут храниться в S3, а текст сообщений и остальная информация в текстовом виде
    То есть в БД это будет: 3МБ * 500млн = 1 430 ТБ
    Для России: 3МБ * 40млн = 114 ТБ
    
 - Сетевой трафик
    Основная нагрузка приходится на сообщения, рассмотрим трафик по типам сообщений

    Тип          | Отправка (дневаня аудитория 300 млн) | Отправка Тб/сутки | Получение (из-за какналов и групповых чатов возьмем доп. эмпирич. коэф.) Тб/сутки 
   ------------- |--------------------------------------|-------------------|-----------------------------------------------------------------------------------|
   Текстовое     | 35 * 300млн * 0.05кБ                 | 0.52              | 0.52 * 8 = 4.16                                                                   |
   Голосовое     | 3 * 300млн * 160кБ                   | 134               | 134 * 3 = 402                                                                     |
   Файл          | 2 * 300млн * 250кБ                   | 139               | 139 * 5 = 695                                                                     |

Для пиковой нагрузки (500млн активных пользователей):

Тип           | Отправка                             | Отправка Тб/сутки | Получение (из-за какналов и групповых чатов возьмем доп. эмпирич. коэф.) Тб/сутки 
------------- |--------------------------------------|-------------------|-----------------------------------------------------------------------------------|
Текстовое     | 35 * 500млн * 0.05кБ                 | 0.88              | 0.88 * 8 = 7.04                                                                   |
Голосовое     | 3 * 500млн * 160кБ                   | 240               | 240 * 3 = 720                                                                     |
Файл          | 2 * 500млн * 250кБ                   | 250               | 250 * 5 = 1250                                                                    |

 RPS в разбивке по типам запросов (запросов в секунду) - для основных запросов Оформить в виде сводной таблицы.
Расчет ведем по пиковой нагрузке (в 500млн пользовтелей).
 
 - Отправка сообщения: 500млн * 35/86400 = 202 500 RPS
 - Получение сообщения: 500млн * 35 * 8/86400 = 1 620 222 RPS
 - Добавление в групповой чат: 500млн * (2/30) /86400 = 400 RPS
 - Подписка на канал(в мес подписка на 3 канала): 500млн * (3/30) /86400 = 570 RPS
 - Создание чата/канала (в мес создание 1 канала/чата от пользователя): 500млн * (1/30) /86400 = 195 RPS

   Действие                            | RPS
   ------------------------------------| ---
   Отправка сообщения                  | 202 500
   Получение сообщения                 | 1 620 222
   Подписка на канал                   | 400
   Добавление в групповой чат          | 570
   Создание чата/канала                | 195

## 3. Логическая схема

![](internal/logic_db.png)

## 4. Физическая схема
![](internal/db-chat@2x.png)


### MessageDB

Для хранения сообщений предлагается написать кастомную БД с требуемой схемой хранения данных.
Помимо стандартных требований к БД по ACID, добавляется необходимость хранить сообщения по чатам таким образом,
чтобы данные чата находились физически рядом на диске.
Изначально требований к поиску сообщений и их редактированию нет, поэтому основной сценарий работы с базой - 
это добавление сообщения в конец чата.
Необходимо чтобы БД поддерживала механизм репликации и разбиение на шарды по схеме Virtual Sharding.

Хотелось чтобы была поддержка индексов, полезных для работы с сообщенями.
Это B+ tree + HASH + инвертированные списки.

По CAP теореме будем стремиться к реализации AP системы потому что проблем с консистентностью быть не должно,
потому что она решается до момента когда все сообщения флашатся на диск и нам достаточно eventual consistency.

Для решения задач консенсуса будет реализован механиззм RAFT.

### PostDB
Та же база, но схема данных немного другая.

### Шардирование

Tarantool Auth - шардирирование по `user_id`

MongoDB Users - шардирование по `user_id`

MongoDB Chat and Channels - шардирование по `_id`

MessagesDB - шардирование по `chat_id`

PostsDB - шардирование по `channel_id` канала

### Индексы

Tarantool Auth - HASH индекс по 'user_token' + HASH индекс по user_id

MongoDB Users - индекс по `name` для полнотекстового поиска

MongoDB Chat - кроме primary индекс не нужно

MongoDB Channels - индекс для полнотекстового поиска

MessagesDB - индекс для полнотекстового поиска по сообщениям

PostsDB - индекс для полнотекстового поиска по сообщениям 

### Схема работы движка чатов и каналов на базе Tarantool

Схема очереди для переписок:
![](internal/peer_to_peer.png)

Схема подписка на обновления чатов из чат-листа:
![](internal/chats_diff.png)

Схема очереди для общих чатов:
![](internal/common_chat.png)

- Amazon S3 используется для хранения и раздачи медиа файлов и всей статики.
- В Tarantool храним сессии пользователей, будет использоваться как key-value хранилище с шардингом по VShard.
- На базе Tarantool реализуем очередь сообщений для чатов, групп и каналов с репликацией и разбиением очередей сообщений по шардам.
Вешаем хранимую процедуру для снятия снапшота состояния очередей, чтобы из сервиса Flusher можно было запросить это состояние
и сохранить в персистентную базу. Реализуем репликацию между датацентрами для ускорения работы переписок между географически
разделенными пользователями.
- В mongoDB будем хранить данные пользователей, сообщения, списки чатов и посты в каналах. 
При заданной нагрузке базу необходимо шардировать со старта. Разбивать на базы и шарды будем по нагрузке. Схема репликации master-slave
Данные пользователей и списки чатов будем хранить в отдельной базе. Данные пользователей и списки чатов будем шардировать по id пользователя(upd по id сообщения).
Сообщения и посты будем также хранить в отдельной базе. Разбивать на шарды будем по id чата и канала.

## 5. Технологии
   - JavaScript - для написания фронтенда приложения
   - React - для упрощения разработки фронтенда 
   - Golang - для разработки всех сервисов 
   - Tarantool Cartridge - для кеша, хранения сессий, in-memory платформа для реализации кастомной очереди сообщений для чатов и каналов
   - MongoDB - для персистентного хранения данных пользователей, чатов и каналов
   - Nginx - reverse proxy, раздача статики, балансинг на уровне L7, проксирование websocket

## 6. Схема проекта
![img.png](internal/arch.png)
## 7. Список серверов
Для распределения нагрузки предлагается завести 4 датацентра
- Северная Америка (Нью-Йорк)
- Южная Америка (Сан Пауло)
- Европа (Амстердам)
- Индия (Мумбаи)

Amazon S3
- Необходимо хранить файлы и аудио файлы, их размер составляет около 220 000 Тб
- В одном бакете хранится 5 Тб данных, значит потребуется 44к бакетов

Сервера для сервисов:

- Для баз данных нужны сервера с увеличенным относительно остальных кофнигураций размером диска.
- Для тарантула можно взять сервер с небольшим количестом ядер и размером диска, но увеличенным объемом памяти.
- Для остальных сервисов достаточно стандартной конфигурации.

Расчет количества и конфигурации серверов производим для региона Россия и СНГ как эталонный.
К остальным регионам числа приводим в соответствии с поправочным коэфициентом, зависящим от разницы в DAU.

### Peer service
Этот сервис должен уметь держать Websocket коннекты с пользователями, поэтому нужно учесть
максимально возможное количество коннектов на один сервер. При должном тюнинге Linux машины (выкручиваем макс. кол-во открытых файловых дескрипторов на один процесс)
можно настроить так, чтобы макс количество коннектов к серверу могло быть около 65тыс. Тогда рассчитаем кол-во серверов на регион
по пиковой нагрузке то есть по DAU: 40 000 000/ 65 000 ~= 615.

Размер одного сообщения примерно равен 50Б тогда при самой пиковой нагрузке (если каждому пользователю придет за секунду 3 сообщения) через один сервер всего будет проходить 
50Б * 3 * 65 000 = 72 Мбит/с для сервера достаточно стандартного интерфейса 10 Гбит/с

### Chat diff service

Этот сервис будет также держать Websocket коннекты с пользователем.

Количество серверов принимаем примерно то же - 615.

Трафик примерно следующий (в среднем одновременно обновляются 8 чатов):

50Б * 8 * 65 000 = 72 Мбит/с для сервера достаточно стандартного интерфейса в 10 Гбит/с

### Chat and Chanel List service

RPS: 100 000

Объем одного запроса - это вытаскивание списка сообщений для одного пользователя.

Объем - запись о чате * (кол-во чатов пользователя + кол-во каналов) = 120Б * (30 + 20) = 5,7 Кб

Траффик - 4 МБит/с

Кол-во - 4

### Chat and Chanel history service
Рассчитываем по нагрузке на этот сервис.

В сутки пользователь запрашивает историю чата в среднем 50 раз.
На каждую историю чата запрашивается по 5 сообщений. В среднем запросов 20 на историю.
Получаем примерный RPS = (40млн * 50 *  20) / 24 * 3600 ~= 460 000 RPS

Принимаем RPS: ~500 000

Объем запроса - (10 сообщений * 50Б) = 500Б

Необходимая пропускная способности: 3,73 Гбит/сек - хватит интерфейса в 10 Гбит/с

Кол-во: 4

### User Data
В среднем пользователь вытаскивает свои данные 1 раз.

Информацию о других пользователях в среднем 20 раз.

RPS: 10 000

Кол-во серверов: 2

Берем станадртную конфигурацию с интерфейсом в 10 ГБит/с 32 Гб RAM и 256 ГБ диска

### Auth
Обращение к сервису авторизации идет из разных сервисов системы для проверки валидности пользовательского токена.

Обращение происходит на каждый запрос к Gateway + на каждый коннект к Chat diff и Peer Service.

В итоге получаем примерное количесвто RPS.

RPS: 1 000 000

На вход идет строка c payload в 16 байт

Необходимая пропускная способность 0,12 Гбит/с (на самом деле чуть больше с учетом http)

Кол-во: 4

### Nginx к S3
RPS: 1 000 000

Один сервер при 100Кб держит ~90 000 RPS.

k = 1,8 - эмпир. коэф. запаса

Кол-во: 1 000 000 * k / 90 000 = 14

Для передачи разных данных от картинок до сообщений берем бондинг в 40 ГБит/с

### Gateway
Суммарно запросов от одного пользователя за один день: 
50 * 20(Chat history) + 50 * 20(Channel history) + 21(User Data) + 1(Chat list) + 1(Channel list) = 2024 запроса

Примерный RPS: (40млн * 2024)/24 * 3600 ~= 1 000 000  RPS

RPS: 1 000 000

k = 20

Кол-во: 1 000 000 * k / 90 000 = 20

### Tarantool auth
Необходимо хранить: 40млн * (100Б - запись в бд) = 3 ГБ

Хватит конфигурации с памятью объемом в 16 ГБ, можно взять 1 мастера + 2 реплики

### Tarantool queue
Нужно учесть максимальное количество данных которое система должна в себя вмещать и как долго.

За половину суток (12ч - принимаемое время для flusher) в сервис поступает (40млн * 40 сообщений)/2 = 800 млн сообщений
в среднем по 50Б. Значит необходимый объем хранения данных в памяти равен ~40 ГБ то есть для одного ЦОД хватит 
с запасом 6 (2 шарда-мастера + 2 реплики) серверов Tarantool с памятью по 32 ГБ и стандартным сетевым интерфейсом, дисковой памяти нужно также не много, только для
хранения снапшотов и xlog.

### Mongo DB
Рассчитываем по объему данных, которое необходимо хранить.

Для хранения 114 ТБ (объем текстовых данных на пользователя) берем сервер с RAID 10 по 4ТБ NVMe 16шт

Кол-во: 4 шарда мастера + 2 реплики на каждого = 8


Сервис                                  | Кол-во серверов на один ЦОД | RAM | CPU     | Storage               | Network
----------------------------------------|-----------------------------|-----|---------|-----------------------| -------
Nginx                                   | 20                          | 32  | 8 cores | 512ГБ SSD             | 40 ГБит/с
Auth                                    | 4                           | 32  | 8 cores | 256ГБ HDD             | 10 Гбит/с
Gateway                                 | 20                          | 32  | 8 cores | 256ГБ HDD             | 10 Гбит/с
Tarantool auth                          | 3                           | 32  | 8 cores | 256ГБ SSD             | 10 Гбит/с
Tarantool queue                         | 5                           | 32  | 8 cores | 256ГБ SSD             | 10 Гбит/с
Peer service                            | 615                         | 16  | 8 cores | 256ГБ HDD             | 10 Гбит/с
Chat diff service                       | 615                         | 16  | 8 cores | 256ГБ HDD             | 10 Гбит/с
Chanel posts service                    | 4                           | 16  | 8 cores | 256ГБ HDD             | 10 Гбит/с
User Data                               | 4                           | 16  | 8 cores | 256ГБ HDD             | 10 Гбит/с
Chat and Chanel List service            | 4*2                         | 16  | 8 cores | 256ГБ HDD             | 10 Гбит/с
Chat and Chanel history service         | 4*2                         | 16  | 8 cores | 256ГБ HDD             | 10 Гбит/с
Flusher service                         | 1                           | 16  | 8 cores | 256ГБ HDD             | 10 Гбит/с
Mongo DB                                | 8                           | 64  | 8 cores | RAID 10 4ТБ NVMe x 16 | 10 Гбит/с



## 8. Источники
1. https://docs.nats.io/nats-concepts/jetstream
2. https://www.tarantool.io/ru/doc/latest/reference/reference_rock/vshard/
3. https://habr.com/ru/company/vk/blog/436916/
4. https://github.com/tarantool/crud/#insert
5. https://www.statista.com/statistics/272014/global-social-networks-ranked-by-number-of-users/
6. https://habr.com/ru/company/oleg-bunin/blog/522744/
7. https://www.nginx.com/blog/testing-the-performance-of-nginx-and-nginx-plus-web-servers/
