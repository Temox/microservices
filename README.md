# Microservices. Docker.
### Создание контейнера с запущенным приложением.

С помощью *docker-machine* создана ВМ в GCP для хостинга docker-контейнеров.

* *Dockerfile* - описание образа
* *mongod.conf* - преподготовленный конфиг для mongodb
* *db_config* - ссылка на mongodb
* *start.sh* - скрипт запуска приложения

Создание образа:
` docker build -t reddit:latest . `

Выгрузка созданного образа в DockerHub:
```
 docker tag reddit:latest temox/otus-reddit:1.0
 docker push temox/otus-reddit:1.0
```
Выгруженный [образ](https://hub.docker.com/r/temox/otus-reddit)


### Разделение приложения на компоненты
Создание работающего приложения состоящего из нескольких контейнеров, связанных между собой по средствам Docker-сети.

Приложение состоит из трех компонентов
```
+-- comment            - сервис написания комментов
¦   +-- Dockerfile
+-- post-py            - сервис написания постов
¦   +-- Dockerfile
L-- ui                 - вэб-интерфейс
    +-- Dockerfile
```

В дирректории каждого сервиса расположен _Dockerfile_ с описанием сборки образа.
После сборки:
```
docker build -t temox/comment:1.0 ./comment
docker build -t temox/post:1.0 ./post-py
docker build -t temox/ui:1.0 ./ui
```

Создаем сеть для связи контейнеров:
```
docker network create reddit
```

И запускаем контейнеры (ключ _--env_ позволяет переопределить, заданные в Dockerfile, переменные окружения):
```
docker run -d --network=reddit --network-alias=post_dbserver --network-alias=comment_dbserver mongo:latest
docker run -d --network=reddit --network-alias=reddit_post --env POST_DATABASE_HOST=post_dbserver temox/post:1.0 
docker run -d --network=reddit --network-alias=reddit_comment --env COMMENT_DATABASE_HOST=comment_dbserver temox/comment:1.0
docker run -d --network=reddit -p 9292:9292 --env POST_SERVICE_HOST=reddit_post --env COMMENT_SERVICE_HOST=reddit_comment temox/ui:1.0
```
Для уменьшения объема получаемых образов, в качестве основных образов можем использовать "чистые" образы, такие как _ubuntu:16.04_ или _alpine linux_


