## Prometheus простыми словами.

В двух словах, Prometheus — система мониторинга, обладающая возможностями тонкой настройки метрик. Она будет полезна для отслеживания состояния работы сервисов на низком уровне.

Основные преимущества — предоставление возможности создания гибких запросов к данным и хранение значений метрик в базе данных временных рядов, возможность автоматизации при администрировании. Разработана фондом облачных вычислений (Cloud Native Computing Foundation или CNCF).

Для получения метрик с удаленных узлов используется метод pull (сервер сам забирает данные). 

На узлы для сбора информации устанавливаются экспортеры (exporter) — пакеты, получающие данные для операционной системы или конкретного сервиса.

Также метрики могут собираться с помощью механизма push — для этого используется компонент pushgateway, который должен быть установлен дополнительно.

Довольно часто Prometheus настраивают в связке с Grafana, которая позволяет визуализировать показания наших метрик.   
В графане для этого есть уже настроенный источник, таким образом, настройка выполняется из коробки.

В сравнении с другими системами мониторинга обладает рядом отличий, например, в сравнении с Zabbix:

 - Zabbix является полностью законченной системой, которая предоставляет веб-инструмент для настройки и визуализации (dashboard). Prometheus — это всего лишь база данных со значениями для метрик. Для настройки и визуализации используются средства, которые являются сторонними или разработанными самостоятельно.  
 - Установка Prometheus не может быть выполнена из репозитория простыми командами. Нам необходимо скачать бинарник, а также создать скрипт для автозапуска или юнит в systemd. Также возможно установка из неофициального образа Docker.
 - Для отправки уведомлений Zabbix использует штатные средства. Для Prometheus устанавливается и настраивается Alertmanager.
 - Zabbix наполнен дополнительными инструментами, такими как графики, построение логической сети, автообнаружение и так далее. Prometheus этим не богат и для дополнительных возможностей необходима установка плагинов или доработка собственных модулей.

Prometheus является приложением от программистов для программистов. Prometheus более популярен для мониторинга сервисов в среде DevOps, например, Kubernetes. Zabbix же плохо подходит для динамических систем (например, в последнем могут постоянно удаляться и создаваться новые поды с сервисами) и уступает в этом плане prometheus.



## Установка Prometheus + Alertmanager + node_exporter на Linux

### Подготовка сервера

### Время

#### Для отображения событий в правильное время, необходимо настроить его синхронизацию. Для этого установим chrony:
```
# CentOS / Red Hat:
yum -y install chrony
systemctl enable chronyd
systemctl start chronyd

# Ubuntu / Debian:
apt-get install chrony
systemctl enable chrony
systemctl start chrony
```

#### Брандмауэр
#### На фаерволе, при его использовании, необходимо открыть порты: 
 - TCP 9090 — http для сервера prometheus.
 - TCP 9093 — http для алерт менеджера.
 - TCP и UDP 9094 — для алерт менеджера.
 - TCP 9100 — для node_exporter.

```
# с помощью firewalld:
firewall-cmd --permanent --add-port=9090/tcp --add-port=9093/tcp --add-port=9094/{tcp,udp} --add-port=9100/tcp
firewall-cmd --reload

# с помощью iptables:
iptables -I INPUT 1 -p tcp --match multiport --dports 9090,9093,9094,9100 -j ACCEPT
iptables -A INPUT -p udp --dport 9094 -j ACCEPT

# с помощью ufw:
ufw allow 9090,9093,9094,9100/tcp
ufw allow 9094/udp
ufw reload
```

#### SELinux

#### По умолчанию, SELinux работает в операционный системах на базе Red Hat. Проверяем, работает ли она в нашей системе:
```
getenforce
```

#### Если мы получаем в ответ: Enforcing, необходимо отключить его.
#### если же мы получим ответ The program 'getenforce' is currently not installed, то SELinux не установлен в системе.
```
setenforce 0

sed -i 's/^SELINUX=.*/SELINUX=disabled/g' /etc/selinux/config
```

## Prometheus

Prometheus имеет слабую поддержку установки из репозитория. Необходимо скачать исходник, создать пользователя, вручную скопировать нужные файлы, назначить права и создать юнит для автозапуска.

### Загрузка

Переходим на официальную страницу загрузки и копируем ссылку на пакет для Linux:
```
https://prometheus.io/download/

wget https://github.com/prometheus/prometheus/releases/download/v2.20.1/prometheus-2.20.1.linux-amd64.tar.gz
```

### Установка (копирование файлов)

После того, как мы скачали архив prometheus, необходимо его распаковать и скопировать содержимое по разным каталогам.

#### Для начала создаем каталоги, в которые скопируем файлы для prometheus:
```
mkdir /etc/prometheus
mkdir /var/lib/prometheus
```
#### Распакуем наш архив и перейдем в каталог с распакованными файлами:
```
tar zxvf prometheus-*.linux-amd64.tar.gz
cd prometheus-*.linux-amd64
```

#### Распределяем файлы по каталогам:
```
cp prometheus promtool /usr/local/bin/
cp -r console_libraries consoles prometheus.yml /etc/prometheus
```

### Назначение прав

#### Создаем пользователя, от которого будем запускать систему мониторинга, пользователь без домашней директории и без возможности входа в консоль сервера.
```
useradd --no-create-home --shell /bin/false prometheus
```

#### Задаем владельца для каталогов, которые мы создали на предыдущем шаге:
```
chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus
```

#### Задаем владельца для скопированных файлов:
```
chown prometheus:prometheus /usr/local/bin/{prometheus,promtool}
```

#### Запуск и проверка, запускаем prometheus командой:
```
/usr/local/bin/prometheus --config.file /etc/prometheus/prometheus.yml --storage.tsdb.path /var/lib/prometheus/ --web.console.templates=/etc/prometheus/consoles --web.console.libraries=/etc/prometheus/console_libraries
```

#### Увидим лог запуска — в конце «Server is ready to receive web requests»:
```
level=info ts=2019-08-07T07:39:06.849Z caller=main.go:621 msg="Server is ready to receive web requests."
```

#### Открываем веб-браузер и переходим в консоль Prometheus:
```
http://<IP-адрес сервера>:9090
```

## Автозапуск

Prometheus после установка запускается вручную. Для настройки автоматического старта Prometheus создадим новый юнит в systemd.

Прерываем работу Prometheus с помощью комбинации Ctrl + C. 

#### Создаем файл prometheus.service:
```
vim /etc/systemd/system/prometheus.service

[Unit]
Description=Prometheus Service
After=network.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
--config.file /etc/prometheus/prometheus.yml \
--storage.tsdb.path /var/lib/prometheus/ \
--web.console.templates=/etc/prometheus/consoles \
--web.console.libraries=/etc/prometheus/console_libraries
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

#### Перечитываем конфигурацию systemd:
```
systemctl daemon-reload
```

#### Разрешаем автозапуск:
```
systemctl enable prometheus
```

#### После ручного запуска мониторинга, который мы делали для проверки, могли сбиться права на папку библиотек — снова зададим ей владельца:
```
chown -R prometheus:prometheus /var/lib/prometheus
```

#### Запускаем службу и проверяем, что она запустилась корректно:
```
systemctl start prometheus
systemctl status prometheus
```

## Alertmanager

Alertmanager нужен для сортировки и группировки событий. Он устанавливается по такому же принципу, что и prometheus.

### Загрузка

#### На официальной странице загрузки копируем ссылку на Alertmanager для Linux:
```
https://prometheus.io/download/#alertmanager
```

####  Теперь используем ссылку для загрузки alertmanager:
```
wget https://github.com/prometheus/alertmanager/releases/download/v0.21.0/alertmanager-0.21.0.linux-amd64.tar.gz
```

### Установка

#### Создаем каталоги для alertmanager:
```
mkdir /etc/alertmanager /var/lib/prometheus/alertmanager
```

#### Распакуем наш архив и перейдем в каталог с распакованными файлами:
```
tar zxvf alertmanager-*.linux-amd64.tar.gz
cd alertmanager-*.linux-amd64
```

#### Распределяем файлы по каталогам:
```
cp alertmanager amtool /usr/local/bin/
cp alertmanager.yml /etc/alertmanager

```

### Назначение прав

#### Создаем пользователя, от которого будем запускать alertmanager, без домашней директории и без возможности входа в консоль сервера.
```
useradd --no-create-home --shell /bin/false alertmanager
```

#### Задаем владельца для каталогов, которые мы создали на предыдущем шаге:
```
chown -R alertmanager:alertmanager /etc/alertmanager /var/lib/prometheus/alertmanager
```

#### Задаем владельца для скопированных файлов:
```
chown alertmanager:alertmanager /usr/local/bin/{alertmanager,amtool}
```

### Автозапуск

#### Создаем файл alertmanager.service в systemd:
```
vim/etc/systemd/system/alertmanager.service

[Unit]
Description=Alertmanager Service
After=network.target

[Service]
EnvironmentFile=-/etc/default/alertmanager
User=alertmanager
Group=alertmanager
Type=simple
ExecStart=/usr/local/bin/alertmanager \
          --config.file=/etc/alertmanager/alertmanager.yml \
          --storage.path=/var/lib/prometheus/alertmanager \
          $ALERTMANAGER_OPTS
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure

[Install]
WantedBy=multi-user.target

```

#### Перечитываем конфигурацию systemd:
```
systemctl daemon-reload
```

#### Разрешаем автозапуск:
```
systemctl enable alertmanager
```

#### Запускаем службу:
```
systemctl start alertmanager
```


#### Открываем веб-браузер и переходим в консоль alertmanager:
```
http://<IP-адрес сервера>:9093
```

## node_exporter

Для получения метрик от операционной системы, установим и настроим node_exporter на тот же сервер прометеуса (и на все клиентские компьютеры). Процесс установки такой же, как у Prometheus и Alertmanager.

Если мы устанавливаем node_exporter на клиента, необходимо проверить наличие брандмауэра и, при необходимости, открыть tcp-порт 9100.




### Загрузка

#### Заходим на страницу загрузки и копируем ссылку на node_exporter.
#### обратите внимание, что для некоторых приложений есть свои готовые экспортеры.
```
https://prometheus.io/download/#node_exporter
```

#### используем ссылку для загрузки node_exporter:
```
wget https://github.com/prometheus/node_exporter/releases/download/v1.0.1/node_exporter-1.0.1.linux-amd64.tar.gz
```

### Установка

#### Распакуем скачанный архив:
```
tar zxvf node_exporter-*.linux-amd64.tar.gz
```

#### перейдем в каталог с распакованными файлами:
```
cd node_exporter-*.linux-amd64
```

#### Копируем исполняемый файл в bin:
```
cp node_exporter /usr/local/bin/
```

### Назначение прав

#### Создаем пользователя nodeusr:
```
useradd --no-create-home --shell /bin/false nodeusr
```

#### Задаем владельца для исполняемого файла:
```
chown -R nodeusr:nodeusr /usr/local/bin/node_exporter
```

#### Автозапуск, Создаем файл node_exporter.service в systemd:
```
vim /etc/systemd/system/node_exporter.service

[Unit]
Description=Node Exporter Service
After=network.target

[Service]
User=nodeusr
Group=nodeusr
Type=simple
ExecStart=/usr/local/bin/node_exporter
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure

[Install]
WantedBy=multi-user.target

```

#### Перечитываем конфигурацию systemd:
```
systemctl daemon-reload
```

#### Разрешаем автозапуск:
```
systemctl enable node_exporter
```

#### Запускаем службу:
```
systemctl start node_exporter
```

#### Открываем веб-браузер и переходим по адресу  — мы увидим метрики, собранные node_exporter:
```
http://<IP-адрес сервера или клиента>:9100/metrics
```


## Отображение метрик с node_exporter в консоли prometheus

#### Открываем конфигурационный файл prometheus:
```
vim /etc/prometheus/prometheus.yml
```

#### В разделе scrape_configs добавим:
```
scrape_configs:
  ...
  - job_name: 'node_exporter_clients'
    scrape_interval: 5s
    static_configs:
      - targets: ['192.168.0.14:9100','192.168.0.15:9100']
```

В данном примере мы добавили клиента с IP-адресом 192.168.0.14, рабочее название для группы клиентов node_exporter_clients. Для примера, мы также добавили клиента 192.168.0.15 — чтобы продемонстрировать, что несколько клиентов добавляется через запятую.

#### Чтобы настройка вступила в действие, перезагружаем наш сервис prometheus:
```
systemctl restart prometheus
```

#### Заходим в веб-консоль prometheus и переходим в раздел Status - Targets:
#### в открывшемся окне мы должны увидеть нашу группу хостов и сам компьютер с установленной node_exporter, cтатус также должен быть UP.



## Отображение тревог

#### Создадим простое правило, реагирующее на недоступность клиента.
```
vi /etc/prometheus/alert.rules.yml

groups:
- name: alert.rules
  rules:
  - alert: InstanceDown
    expr: up == 0
    for: 1m
    labels:
      severity: critical
    annotations:
      description: '{{ $labels.instance }} of job {{ $labels.job }} has been down
        for more than 1 minute.'
      summary: Instance {{ $labels.instance }} down

```

#### Теперь подключим наше правило в конфигурационном файле prometheus:
```
vim /etc/prometheus/prometheus.yml

...
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"
  - "alert.rules.yml"
...
```

В данном примере мы добавили наш файл alert.rules.yml в секцию rule_files. Закомментированные файлы first_rules.yml и second_rules.yml уже были в файле в качестве примера.

#### Перезапускаем сервис:
```
systemctl restart prometheus
```

#### Открываем веб-консоль прометеуса и переходим в раздел Alerts. Если мы добавим клиента и попробуем его отключить для примера, мы увидим тревогу:

## Отправка уведомлений

#### Настроим связку с алерт менеджером для отправки уведомлений на почту. Настроим alertmanager, в секцию global добавим:
#### мы будем отправлять сообщения от email monitoring@dmosk.ru.
```
vim /etc/alertmanager/alertmanager.yml

global:
  ...
  smtp_from: monitoring@dmosk.ru

```

#### Приведем секцию route к виду:
#### в данном примере нами был добавлен маршрут, который отлавливает событие InstanceDown и запускает ресивер send_email.
```
route:
  group_by: ['alertname', 'instance', 'severity']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'web.hook'

  routes:
    - receiver: send_email
      match:
        alertname: InstanceDown
```


#### добавим еще один ресивер:
#### * в данном примере мы отправляем сообщение на почтовый ящик alert@dmosk.ru с локального сервера. Обратите внимание, что для отправки почты наружу у нас должен быть корректно настроенный почтовый сервер (в противном случае, почта может попадать в СПАМ).
```
receivers:
...
- name: send_email
  email_configs:
  - to: alert@dmosk.ru
    smarthost: localhost:25
    require_tls: false
```

#### Перезапустим сервис для алерт менеджера:
```
systemctl restart alertmanager
```

#### Теперь настроим связку prometheus с alertmanager — открываем конфигурационный файл сервера мониторинга: Приведем секцию alerting к виду:
#### * где 192.168.0.14 — IP-адрес сервера, на котором у нас стоит alertmanager.
```
vi /etc/prometheus/prometheus.yml

alerting:
  alertmanagers:
  - static_configs:
    - targets:
      - 192.168.0.14:9093
      
      
```

#### Перезапускаем сервис:
```
systemctl restart prometheus
```


Немного ждем и заходим на веб интерфейс алерт менеджера — мы должны увидеть тревогу:

... а на почтовый ящик должно прийти письмо с тревогой.



## Мониторинг служб Linux

Для мониторинга сервисов с помощью Prometheus мы настроим сбор метрик и отображение тревог.

Сбор метрие с помощью node_exporter


#### Открываем сервис, созданный для node_exporter и добавим к ExecStart:
#### данная опция указывает экспортеру мониторить состояние каждой службы.

```
vim /etc/systemd/system/node_exporter.service

...
ExecStart=/usr/local/bin/node_exporter --collector.systemd
...

```

#### При необходимости, мы можем либо мониторить отдельные службы, добавив опцию collector.systemd.unit-whitelist:
#### в данном примере будут мониториться только сервисы chronyd, mariadb и nginx.

```
ExecStart=/usr/local/bin/node_exporter --collector.systemd --collector.systemd.unit-whitelist="(chronyd|mariadb|nginx).service"
```

#### либо наоборот — мониторить все службы, кроме отдельно взятых:
#### при такой настройке мы запретим мониторинг сервисов auditd, dbus и kdump.
```
ExecStart=/usr/local/bin/node_exporter --collector.systemd --collector.systemd.unit-blacklist="(auditd|dbus|kdump).service"
```

#### Чтобы применить настройки, перечитываем конфиг systemd:
```
systemctl daemon-reload
```

#### Перезапускаем node_exporter:
```
systemctl restart node_exporter
```

#### Отображение тревог
Настроим мониторинг для службы NGINX.

#### Создаем файл с правилом:
```
vi /etc/prometheus/services.rules.yml

groups:
- name: services.rules
  rules:
    - alert: nginx_service
      expr: node_systemd_unit_state{name="nginx.service",state="active"} == 0
      for: 1s
      annotations:
        summary: "Instance {{ $labels.instance }} is down"
        description: "{{ $labels.instance }} of job {{ $labels.job }} is down."
```


#### Подключим файл с описанием правил в конфигурационном файле prometheus:
#### в данном примере мы добавили наш файл services.rules.yml к уже ранее добавленному alert.rules.yml в секцию rule_files.
```
vi /etc/prometheus/prometheus.yml

...
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"
  - "alert.rules.yml"
  - "services.rules.yml"
...
```

#### Перезапускаем prometheus:
```
systemctl restart prometheus
```

#### Для проверки, остановим наш сервис:
```
systemctl stop nginx
```

В консоли Prometheus в разделе Alerts мы должны увидеть тревогу:



/opt/node-exporter/current/node_exporter --collector.systemd --web.listen-address=0.0.0.0:9100








