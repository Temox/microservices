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
