# IoT lab work #1

# Описание

## Настройка

Предварительно нам нужно создать overlay сеть, для чего проинициализируем `docker swarm`:

```sh
docker swarm init
```

Также нужно создать и пользовательскую сеть. Ее будут использовать сервисы, которые мы опишем ниже:

```sh
docker network create --driver overlay --attachable my-overlay
```

## Запуск файла `docker-compose.yml`

Выполним простейший shell-скрипт для запуска этого файла:

```sh
docker compose up
```

## Добавление подключающий скриптов и проверка их на работу

Первым делом добавляем подключающие скрипты с помощью сервиса `basic-client` следующим образом:

```sh
docker compose exec -it basic-client sh
cd ~
./source.sh
./target.sh
```

Для этого и был предназначен предназначен сервис `basic-client`. Посколько `docker-compose` и директория `connectors` находятся в одной папке, то в `docker-compose`  Внутри него маунтится директория connectors, если `docker-compose.yml` можно замаппить все это и запускать достаточно просто.
Для этого уже был соответствующим образом настроен `docker-compose`, поэтому теперь достаточно сделать следующее для подключения к уже запущенному сервису:

```sh
docker compose exec -it basic-client sh
```

Теперь проверим, что все прошло хорошо и наши коннекторы работают:

```sh
~ $ ./get-topics.sh
["mongodb-test-users-source","mongo-test-users-sink"]
```

## Взаимодействие с БД

Для начала осуществим в нее вход:

```sh
docker compose exec -it mongodb-<source/target> bash
```

* `mongodb-<source/target>` - база данных на выбор, где `source` - ее источник

Подключение к `mongosh` будет происходить аналогично:

```sh
mongosh
```

При входе увидим следующую запись:

```
rs0 [direct: primary] test>
```

### Вставка данных в базу

Добавление данных в коллекцию users базы данных test:

```
db.users.insertOne({
  "_id": Number(42),
  "firstname": String("ivan"),
  "lastname": String("nesterov"),
  "age": Number(21),
  "email": String("me@wannapass.this")
})

db.users.insertOne({
  "firstname": String("qwerty"),
  "lastname": String("qwerty"),
  "age": Number(42),
  "email": String("qwerty@qqwert.com")
})
```

Для `target`, что ожидаемо, в качестве результата должно отобразиться дублирование:

```
db.users.find()
```

### Удаление данных из БД

Было найдено странное ограничение драйвера MongoDB connector. Операция `DROP` из-за этого не работает:
```
WARN Unsupported change stream operation: drop (com.mongodb.kafka.connect.sink.cdc.mongodb.ChangeStreamHandler)
```

### Внесение изменений в схему данных:

Протестируем это следующим образом:

```
db.users.updateOne( {},
[
  { $set: { firstname: "nesterrovv" } },
])
```

### Другие манипуляции с данными:

Например, можно написать агрегирование с целью получение типов и ключей нашего набора элементов, причем сама БД от этого меняться не будет. Запрос будет выглядеть так: 

```sh
db.users.aggregate([
  { $sample: { size: 1 } },
  { $project: { 
    fields: { $objectToArray: "$$ROOT" } 
  } },
  { $unwind: "$fields" },
  { $group: {
    _id: "$fields.k",
    types: { $addToSet: { $type: "$fields.v" } }
  } }
]);
```

## Пара слов о `kafka-ui`

В `docker-compose` укажем, что она работает на порту `8080`. Станадартный порт для таких целей.

## Завершение работы

Остановка `docker-compose` и  очистка volumes:

```sh
docker compose down --volumes
```

Почти все. Теперь покинем еще и созданную на первых этапах сеть:

```sh
docker swarm leave [--force]
```
