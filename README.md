### 1. Выбор темы
* Spotify (музыкальный сервис)

### 2. Определение возможного диапазона нагрузок

Возможный диапазон нагрузок может быть оценен относительно прямого конкурента с идентичным охватом аудитории - [Яндекс.Музыка](https://vc.ru/services/141928-gde-slushayut-muzyku-v-rossii-i-est-li-mesto-spotify-glavnoe-o-rossiyskom-rynke-i-ego-igrokah), таким образом:

* Месячная аудитория 15 млн

* Дневная аудитория 1.9 млн

* Пользователи в среднем слушают музыку ~90 минут в день

### 3. Выбор планируемой нагрузки

В силу того, что Spotify недавно начал свою деятельность на территории России, где уже работают крупные конкуренты, а также высокого уровня популярности 
сервиса в том числе в России, можно оценить планируемую нагрузку как 30% доля рынка в России.

Размер секунды аудиозаписи при битрейте в 192 Кбит/с составляет 24 Кб. Тогда можно получить следующий объем потребления трафика пользователем за час прослушивания:

`60 (мин) * 60 (сек) * 24 (Кб) / 1024 ~= 85Mb`

`Размер_трека = (4 мин * 60 с + 10 с) * 24 Кб = 6000 Кб ~= 5.86 Мб`

Объем медиатеки сервиса на момент 27 сентября 2017 года составлял более 35 млн треков <sup>[[2]](https://www.interfax.ru/business/580802)</sup>. Хранение 35 млн треков требует как минимум:

`Минимальный_объем_памяти_медиатеки = 35 * 10^6 треков * 5.86 Мб ~= 205 100 000 Мб ~= 195.6 Тб`

За час прослушивания музыки расходуется 85Mb трафика. Дневная аудитория 1.9 млн * 90 (минут) / 60 (час) * 85 (Mb) --> 242 Tb -- _дневная нагрузка_.

242 Tb * 8 / 864000 = 22.4 Гб/c

242 * 1014 Гб * 8 / 86400 = 22.8 гбит/c.

### 4. Логическая схема базы данных (без выбора СУБД)
![](./img/database.png)

### 5. Физическая системы хранения (конкретные СУБД, шардинг, расчет нагрузки, обоснование реализуемости на основе результатов нагрузочного тестирования)

Расчитаем необходимое RPS при регулярном пользовании сервисом.
Предположим, что пользователь во время пользования сервисом в течение дня совершает примерно 90 запросов (
добавление треков в плейлист, поисковые запросы, прослушивание музыки, лайк/дизлайк.
``` 
1 900 0000 активных пользователей
x
90 запросов в день
/
24 часа x 60 минут x 60 сек
------------------------------------------
~ 2 000 RPS
``` 
С такой нагрузкой справится 1 сервер PostgreSQL, однако для нашего проекта будем ориентироваться на использование кластера. Помимо того, 
понадобятся кэширующие сервисы - например, memcached, т.к специфика сервиса подразумевает неоднократное обращение к одним и тем же ресурсам поиска или 
метаданных. Основная база справится с планируемой нагрузкой и без кеширующих сервисов, однако следует учесть этот шаг при проектировании как средство для 
обработки резких скачков нагрузки в течение дня.

Аудио/фото контент целесообразно хранить в Amazon S3 с точки зрения цены/качества. Однако при данной постановке задачи можно ограничиться собственными ресурсами системы. Вспомогательный кэш для этих ресурсов подразумевается на 
наших бэкендах. Можно было бы воспользоваться Amazon CloudFront для решения этой проблемы, однако его нет в России.
Естественно речь идет о разработке SPA приложения на фронте, поэтому следует учесть использование методов/технологий минимизации 
скриптов и стилей.

Также следует предусмотреть большое количество записей пользователей о содержимом их библиотеки. Из схемы бд следует, что для хранения музыки пользователя потребуется 8 байт. Тогда, учитывая запас:

128 * 1024^3 (байты) * 0.5 / (8 * 200 (среднее кол-во песен у пользователя)) = 43 000 000 (хранение записей пользователей)

Дополнительных приемов для устройства структуры БД не требуется, т.к. ко всем данным доступ одинаковый.

### 6. Выбор прочих технологий: языки программирования, фреймфорки, протоколы взаимодействия, веб-сервера и т.д. (с обоcнованием выбора)

_web client_: JavaScript.

_backend_: Можно воспользоваться HAproxy.У Haproxy есть возможность указать dns resolvers (resolver обращается к локальному серверу имен) 
и настроить dns cache. Тем самым Haproxy будет сам обновлять dns cache, если записи в нем 
истекли, и заменять адреса для апстримов в том случае, если они изменились.

Язык бэкенда Golang, потому что зарекомендовал себя отличным языком для написания быстрого кода. Также следует 
отметить, что в силу того, что бизнес-логика сервиса не подразумевает крупных вычислений, то также нет необходимости 
работать с конфигурацией языка при работе в runtime с ресурсами системы (madvise и т.д).

Также можно прибегнуть к микросервисной архитектуре. Она имеет свои недостатки, но серьезно повышает характеристики масштабируемости и отказоустойчивости.

_app for smartphone_: Java для android, Swift для ios.

### 7. Расчет нагрузки и потребного оборудования

Итого, у нас есть следующие ключевые узлы системы:

* Фронтенд
* Бэкенд
* Балансировщик
* Статика
* Сборщик статистики

_Фронтенд_: 

Объем фронта можно оценивать примерно в 3 мб ->
3 * 80 000(количество пользователей в час) = 240 гб/ч = 533 Мб/c = 0.5 гигабита/c. Тогда для него возьмем 
64ГБ памяти 16 ядра RAID1 128 Гб SSD 10G Ethernet

Кол-во серверов:

RPS на отдачу фронтенда (при совершении обычных действий может потребоваться около 20 запросов, при учете отсутствия кеширования браузера) в пиковой нагрузке должен составить:

`Фронтенд_пиковый_rps = 1900000 * 0.7 * 20 * 1.3 / (12 ч * 60 мин * 60 с) ~= 2200 rps`

С такой нагрузкой справится и 1 сервер фронтенда на nginx. Но возьмем 2 сервера.

_Бэкенд_:

Для бэкенда возьмем:

128Гб памяти 32 ядра RAID1 64 Гб SSD 10G Ethernet должно хватить 4 сервера (с запасом)

_БД_ :

128Гб памяти 32 ядра RAID 10 1 Тб SSD
берем 2 для хранения информации о пользователях 

_Балансировка_ :

Возьмем 64Гб памяти 32 ядра RAID1 32 Гб SSD 10G Ethernet. 6(пропускная способность + пик нагрузки) для бэка.
Для каждого балансировщика предусмотрена реплика и сопровождающие методы обеспечения отказоустойчивости резервирования балансировщика. (пункт 10).

_Статика_ :

Для эффективной отдачи аудио контента, общий объем которого оценен в 195.6 Тб нам понадобится кластер серверов и шардирование контента.
Для этого разделим медиатеку на четыре сервера с размером диска в `128 Тб`. Диски формируем путем объединения 16 8 Тб дисков в RAID5 массив. 4 сервера данных снабдим 1 репликой каждый и получим 8 серверов для хранения контента. Кластер из 8 серверов снабдим балансировщиком nginx. Возьмем запас в 30 процентов на отдачу трафика по скорости (22.8 * 1.3 ~= 30) также 
при шардировании контента произойдет распределение нагрузки 30/4 сервера ~= 8 Гбит/с. Тогда будет достаточно 1x10G для нашего кластера. 

Шардирование хранимого контента будет обеспечиваться по идентификатору трека.

_Сборщик статиски_ :

8Гб памяти 32 ядра SATA без RAID 10G Ethernet - 2шт

 Сервер | CPU | RAM | NET | DISK | Кол-во
 -|-|-|-|-|-
 PostgreSQL | 16 | 128 Гб | 10Gx1 | (512ГБ SSD x 4 ) RAID 10 1 Тб SSD | 2
 Бекенд | 32 | 128 Гб | 10Gx1 | RAID1 64 Гб SSD | 6
 Фронтенд | 16 | 64 Гб | 10Gx1 | RAID1 128 Гб SSD | 2
 Балансировка | 32 | 64 Гб | 10Gx1 | RAID1 32 Гб SSD | 2 фронт + 2 бек + 2 статика 
 Статика | 32 | 256 Гб | 10Gx1 | (8ТБ SSD x 16) RAID 5 = 128 Тб| 8
 Сборщик статистики | 32 | 32 | 10Gx1 | RAID1 64 Гб SSD | 2


### 8. Выбор хостинга / облачного провайдера и расположения серверов

В качестве облачного провайдера для разворачивания серверов бекенда и фронтенда я решил выбрать mail cloud solutions, который обладает всем необходимым функционалом для развертывания нашего сервиса, плюс сервера расположены в России. На сайте MCS не оговаривается расположение ДЦ, однако предполагается соединенность всех информационных зон России через оптоволоконную сеть. Если предположить, что основная масса серверов находится в Европейской части России, то это нас устраивает, так как основная часть аудитории распологается именно здесь.

### 9. Схема балансировки нагрузки (входящего трафика и внутрипроектного, терминация SSL)
Балансируем нагрузку с помощью Nginx L7, терминация SSL -> Nginx. Так как будет больше одного Nginx, то воспользуемся
DNS-балансировкой. На одно доменное имя выделяется несколько IP-адресов. Сервер, на который будет направлен клиентский 
запрос, обычно определяется с помощью алгоритма Round Robin.

### 10. Обеспечение отказоустойчивости
Для обеспечения отказоустойчивости для каждого сервера БД храним две репликации (Master-Slave(2)). Таким образом Мастер 
сервер отвечает за изменения данных, а Слейв за чтение. Асинхронность репликации означает, что данные на Слейве могут 
появится с небольшой задержкой. Поэтому, в последовательных операциях необходимо использовать чтение с Мастера, чтобы 
получить актуальные данные. При выходе из строя Слейва, достаточно просто переключить все приложение на работу с Мастером. 
После этого восстановить репликацию на Слейве и снова его запустить.
Если выходит из строя Мастер, нужно переключить все операции (и чтения и записи) на Слейв. Таким образом он станет новым 
Мастером. После восстановления старого Мастера, настроить на нем реплику, и он станет новым Слейвом.

Кроме этого, собираем статистику о сервера и мониторим его: Grafana + Prometeus

Отказоустойчивость баланисровщиков обеспечивается сбором метрик и ручным переключением серверов в случае отказа главного балансировщика. Также возможен автоматизированный анализ системы с последующим перезапуском.

### 11. Использованные источники

[Spotify в России](https://www.forbes.ru/tehnologii/406063-glava-spotify-v-rossii-my-vidim-vzryvnoy-interes-polzovateley)

[Аудитория](https://trashbox.ru/link/2020-10-29-sptfy-320-million-monthly-active-users#:~:text=%D0%9D%D0%B0%20%D1%82%D0%B5%D0%BA%D1%83%D1%89%D0%B8%D0%B9%20%D0%BC%D0%BE%D0%BC%D0%B5%D0%BD%D1%82%20%D0%BC%D0%B5%D1%81%D1%8F%D1%87%D0%BD%D0%B0%D1%8F%20%D0%B0%D1%83%D0%B4%D0%B8%D1%82%D0%BE%D1%80%D0%B8%D1%8F,%D0%BF%D0%BB%D0%B0%D1%82%D0%BD%D1%8B%D0%B5%20%D0%BF%D0%BE%D0%B4%D0%BF%D0%B8%D1%81%D1%87%D0%B8%D0%BA%D0%B8%20(%2B27%25).)

[Среднее время](https://vc.ru/media/96460-chislo-podpischikov-yandeks-muzyki-vyroslo-v-tri-raza-za-poltora-goda-i-dostiglo-3-mln)

[Трафик](https://yandex.ru/support/music-app-winmobile/search-and-listen/cost.html)

[Фронт](https://habr.com/ru/company/tinkoff/blog/474632/)

[Аудиоконтент в мире](https://thequestion.ru/questions/82483/skolko_v_mire_pesen_64b53597)

[Оперативная память](https://ddriver.ru/kms_catalog+stat+cat_id-12+nums-78.html)

[Amazon](https://aws.amazon.com/ru/s3/pricing/)
