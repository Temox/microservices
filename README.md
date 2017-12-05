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

### Сети и Docker-Compose

* *None* network driver
  - в контейнере присутствует только lo-интерфейс
  - отсутсыует возможность контактировать с внешним миром
  - можно использовать только для локальных тестов, внутри самого контейнера
    ` docker run --network none --rm -d --name net_test`
 
* *Host* network driver
  - Контейнер использует тот же интерфейс, что и хостовая машина
    `docker run --network host -d nginx`

* *Bridge* network driver
  - Для контейнеров создается своя сеть, в которой они могут взаимодействовать
  - Устанавливается по средствам bridg-интерфейса на самом хосте
     ```
     docker network create reddit --driver bridge
     docker run -d --network=reddit mongo:latest
     ```
  - Появляется возможность создать несколько подсетей
     ```
     docker network create back_net --subnet=10.0.2.0/24
     docker network create front_net --subnet=10.0.1.0/24
     
     docker run -d --network=front_net -p 9292:9292 --name ui temox/ui:1.0
     docker run -d --network=back_net --name comment temox/comment:1.0
     ```
     созданные сети изолированы друг от друга. Для взаимодействия контейнеров между подсетями, каждый из них добавляется в обе:
     ```
     docker network connect <network> <container>
     ```
 
 #### Docker-compose
 Позволяет создавать docker-контейнере на основе описания.
 Описание инфраструктуры помещается в файл _docker-compose.yml_.
 В самом файле допускается применение переменных окружения _{VARIABLE}_

## Системы мониторинга. Prometheus.

Запуск _Prometheus_ с помощью _Docker_:
`docker run --rm -p 9090:9090 -d --name prometheus prom/prometheus`

*Метрика* - собираемая информация. Формируются следующим образом:

`prometheus_build_info{branch="HEAD",goversion="go1.9.1",instance="localhost: 9090",job="prometheus",revision="3a7c51ab70fc7615cd318204d3aa7c078b7c5b20",version="1.8.1"} 1`

`название метрики{лэйбл="значение",лэйбл="значение",лэйбл="значение"} значение метрики`

*Target* - системы и процессы, за которыми следит _Prometheus_
*Endpoint* - адреса по которым обращаются _Target'ы_

### Конфигурация Prometheus

Файл конфигурации:
```
prometheus/
└── prometheus.yml
```

где,

scrape_interval - чачтота сбора метрик
job_name - имя джоба, группы enpoint'ов с одинаковыми функциями
targets - адреса для сбора метрик

### Exporters

Exporter - интерфейс для конвертации метрик в формат доступный для _Prometheus_

Для сбора статистики с docker-хоста используется node-exporter. Запускается отдельным контейнером _node-exporter_.
В конфигурацию _Prometheus_ добавляется дополнительный джоб _node_.

Конткйнеры опубликованы в docker-hub:

[temox/ui](https://hub.docker.com/r/temox/ui/)

[temox/comment](https://hub.docker.com/r/temox/comment/)

[temox/post](https://hub.docker.com/r/temox/post/)

[temox/prometheus](https://hub.docker.com/r/temox/prometheus/)
