## Начало работы

Установите Docker Engine и Docker Compose.

Выполните `docker compose up`. 

В результате будут подняты (в порядке запуска):

* Три Zookeeper
* Три Clickhouse
* HAProxy

## Сеть

Все поднятые узлы смотрят во внутреннюю сеть Docker. Единственный выставляемый наружу порт -- 9001. Это порт для подключения к кластеру Clickhouse по протоколу clickhouse-client через HAProxy. 

## Zookeeper

Кластер Zookeeper толерантен к потере половины узлов (с округлением вниз). Clickhouse использует Zookeeper как сервис конфигураций (метаданные о хранящихся в Clickhouse данных, блокировки и т.д.). Zookeeper почти не требует настройки. Данные о доменах и портах серверов Clickhouse будут ему анонсированы самими серверами Clickhouse при запуске кластера. 

## Clickhouse 

Кластер Clickhouse может потерять любое число узлов (кроме всех) и при этом сохранять работоспособность. 

Репликация асинхронная, мульти-мастер. И на запись и на чтение можно обращаться к любым из узлов кластера. Так как репликация асинхронная, то возможна потеря данных за краткий промежуток времени перед выходом из строя.

При запросах `SELECT` Zookeeper не используется. Соответственно отказ кластера Zookeeper приведёт к отказу на запись, но не к отказу на чтение.

## HAProxy

Используется для балансирования нагрузки и обеспечения доступности. Распределение нагрузки производится по алгоритму round-robin. Healthcheck проверяется у порта протокола clickhouse-client (просто проверяется доступность TCP порта). В случае HTTP есть документированный Healthcheck -- GET на порт 8123 должен вернуть OK.

HAProxy переадресовывает запросы на 9001 к одному из доступных узлов Clickhouse кластера, на порт 9000.

## Тестовый сценарий

Подключитесь к кластеру Clickhouse:

`clickhouse-client --port=9001`

Поочерёдно на каждой реплике выполните команду создания реплицируемой таблицы (замените `{replica-number}` на порядковый номер реплики) (обойти все реплики можно просто переподключаясь):

`CREATE TABLE ExampleTable (ExDate default today(), SomeText String) ENGINE = ReplicatedMergeTree('/clickhouse/tables/example', '{replica-number}', ExDate, (ExDate, SomeText), 8192)`

К сожалению , DDL на данный момент не реплицируются.

Теперь вы можете отключить некоторые из сервисов:

`sudo docker-compose kill {service-name}`

И проверить работу кластера:

`SELECT * FROM ExampleTable`

`INSERT INTO ExampleTable (SomeText) VALUES ('Wow its replicating');`

Также вы можете обратно поднять некоторые из сервисов:

`sudo docker-compose up {service-name}`

При этом данные, полученные кластером Clickhouse за время простоя данного сервиса, будут добавлены.

Заметьте, что в случае поломки Zookeeper кластера (отказ больше половины узлов с округлением вверх) запросы `SELECT` будут выполняться всё-равно. А вот `INSERT` уже нет.
