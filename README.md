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

Конткйнеры опубликованы в docker-hub:<br/>
[temox/ui](https://hub.docker.com/r/temox/ui/)<br/>
[temox/comment](https://hub.docker.com/r/temox/comment/)<br/>
[temox/post](https://hub.docker.com/r/temox/post/)<br/>
[temox/prometheus](https://hub.docker.com/r/temox/prometheus/)<br/>


## Мониторинг контейнеров и инфраструктуры
### cAdvisor
[cAdvisor](https://github.com/google/cadvisor) собирает инфорацию о ресурсах и потребляемых контейнерами CPU, RAM, Network и др.

docker-compose.yml
```
  cadvisor:
    image: google/cadvisor:latest
    volumes:
      - '/:/rootfs:ro'
      - '/var/run:/var/run:rw'
      - '/sys:/sys:ro'
      - '/var/lib/docker/:/var/lib/docker:ro'
    ports:
      - '8080:8080'
    networks:
      back_net:
        aliases:
          - cadvisor
```
prometheus.yml
```
- job_name: 'cadvisor'
 static_configs:
 - targets:
 - 'cadvisor:8080'

```

### Визуализация метрик. Grafana.

docker-compose.yml
```
  grafana:
    image: grafana/grafana
    volumes:
      - grafana_data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=secret
    depends_on:
      - prometheus
    ports:
      - 3000:3000
    networks:
      back_net:
        aliases:
          - grafana
          
  volumes:
    grafana_data:
```

_Garafana_ позволяет подгружать дашборды из .json файлов:
```
 dashboards/
  ├── Business_Logic_Monitoring.json
  ├── DockerMonitoring.json
  └── ui_service_monitoring.json
```

[Встроенные функции _Grafana_](https://prometheus.io/docs/prometheus/latest/querying/functions/) позволяют производить различные операции над получаемыми из _Prometheus_ метриками.
Также, _Prometheus_ позволяет предоставлять метрики различных [типов](https://prometheus.io/docs/concepts/metric_types/)

### Алертинг. Alretmanager.

*Alertmanager* - дополнительный компонент для системы мониторинга Прометей, который отвечает за первичную обработку алертов и дальнейшую отправку оповещений по заданному назначению.

docker-compose.yml
```
  alertmanager:
    image: ${USER_NAME}/alertmanager
    command:
      - '-config.file=/etc/alertmanager/config.yml'
    ports:
      - 9093:9093
    networks:
      back_net:
        aliases:
          - alertmanager
```

Настройки _Alertmanager_ передаются через yaml-файл:
```
  alertmanager/
   ├── config.yml
   └── Dockerfile
```

Dockerfile
```
FROM prom/alertmanager
ADD config.yml /etc/alertmanager/
```

config.yml
```
global:
  slack_api_url: 'https://hooks.slack.com/services/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'

route:
  receiver: 'slack-notifications'

receivers:
- name: 'slack-notifications'
  slack_configs:
  - channel: '#temox1'
```

### Alert rules (Prometheus v2.x)

Правила описываются в виде yaml-файлов и добавляются в конфиг _Prometheus_:
 
alert.rules
```
groups:
- name: alrets
  interval: 5s
  rules:
  - alert: InstanceDown
    expr: up == 0
    for: 1m
    labels:
      severity: critical
    annotations:
      description : "{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 1 minute."
      summary: "Instance {{ $labels.instance }} down"
```

Dockerfile
```
...
ADD alert.rules /etc/prometheus/
```

prometheus.yml
```
rule_files:
  - "alert.rules"

alerting:
  alertmanagers:
  - scheme: http
    static_configs:
    - targets:
      - "alertmanager:9093"
```

Обновленные образы в docker.hub:<br/>
[ui](https://hub.docker.com/r/temox/ui/) <br/>
[comment](https://hub.docker.com/r/temox/comment/)<br/>
[post](https://hub.docker.com/r/temox/comment/)<br/>
[prometheus](https://hub.docker.com/r/temox/comment/)<br/>

## Docker Swarm

Кластер Docker Swarm состоит из **master-нод** и управляемых ими **worker-нод**:
![Docker swarm](https://docs.docker.com/engine/swarm/images/swarm-diagram.png)

### Инициализация. Добавление узлов кластера.
`docker swarm init`<br/>

Получение токена:<br/>
`docker swarm join-token manager/worker`<br/>

Полученный токен запускается на остальных узлах для добавления их в кластер.

### Docker swarm stack
Stack - группа сервисов и их хависимостей. Описываются в формате **docker-compose**.

Управление стэком:<br/>
`docker stack deploy/rm/services/ls STACK_NAME`<br/>

Деплой стэка с использованием _docker-compose.yml_:<br/>
`docker stack deploy --compose-file=<(docker-compose -f dockercompose.yml config 2>/dev/null) STACK_NAME`<br/>

### Labels. Размещение сервисов.
Ограничения размещения определяются значеинями **label'ов** и **docker-engine'ов**
```
- node.labels.reliability == high
- node.role != manager
- engine.labels.provider == google
```

Добавление **label'ов** к нодам:<br/>
`docker node update --label-add reliability=high NODE_NAME`<br/>

Просмотр информации:<br/>
`docker node ls -q | xargs docker node inspect -f '{{ .ID }} [{{ .Description.Hostname }}]: {{ .Spec.Labels }}'`<br/>

Размещение сервисов:
```
deploy:
 placement:
 constraints:
```
По label'ам:<br/>
`- node.labels.reliability == high`<br/>
По узлам:<br/>
`- node.role == worker`<br/>

### Масштабирование сервисов

**Replicated** - запуск определенного количества задач (_replicas_)
  ```
  deploy:
  mode: replicated
  replicas: 2
  ```
  Масштабирование "на лету":
    ```
    $ docker service scale DEV_ui=3
      or
    $ docker service update --replicas 3 DEV_ui
    ```
  Выключение всех задач:<br/>
    `docker service update --replicas 0 DEV_ui`<br/>

**Global** - запуск по одной задаче, на каждой ноде

### Rolling update

Для минимизации простоя и определения плана отката, используется **update_config**
```
deploy:
 update_config:
 parallelism: 2
 delay: 5s
 failure_action: rollback
 monitor: 5s
 max_failure_ratio: 2
 order: start-first
```
  - _parallelism_ - cколько контейнеров (группу) обновить одновременно?
  - _delay_ - задержка между обновлениями групп контейнеров
  - _order_ - порядок обновлений (сначала убиваем старые и запускаем новые или наоборот) (только в compose 3.4)
  - _monitor_ - сколько следить за обновлением, пока не признать его удачным или ошибочным
  - _max_failure_ratio_ - сколько раз обновление может пройти с ошибкой перед тем, как перейти к _failure_action_
  - _failure_action_ - что делать, если при обновлении возникла ошибка
    - _rollback_ - откатить все задачи на предыдущую версию
    - _pause_ (default) - приостановить обновление
    - _continue_ - продолжить обновление

### Ограничение ресурсов
```
deploy:
 resources:
 limits:
 cpus: ‘0.25’ - 25% процессорного времени
 memory: 150M - 105Мб ОЗУ
```

### Политика перезапуска контейнеров
```
 deploy:
   restart_policy:
     condition: on-failure
     max_attempts: 3
     delay: 3s
```
