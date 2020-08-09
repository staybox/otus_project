# OTUS Проектная работа. Тема: Отказоустойчивый кластер веб-приложения (Centos 7)
----------------------------------------------------------------------- 

```
Проектная работа
Заключительный месяц курса посвящен проектной работе. Свой проект это то, что интересно писать студенту. То, что можно создать на основе знаний, полученных на курсе.
При этом не обязательно закончить его за месяц. В процессе написания по проекту можно получить консультации преподавателей. 
```

### Необходимое ПО для запуска проекта:

- Vagrant (Конфигурирует ВМ исходя из написанного кода)
- Ansible (Система управления конфигурациями)
- VirtualBox (или другой провайдер виртуализации)

Vagrant и Ansible работают в парадигме "Инфрастуктура как код"

### Состав проекта:

1. Балансировщики нагрузки (http), VIP: 192.168.10.39
- Balancer1 IP: 192.168.10.40 Установлено: KeepAlived, HAProxy (Master)
- Balancer2 IP: 192.168.10.41 Установлено: KeepAlived, HAProxy (Slave)

2. Веб-сервера
- Web1 IP: 192.168.10.42 Установлено: Nginx, служба GlusterFS для монтирования директории с файлами сайта (Master)
- Web2 IP: 192.168.10.43 Установлено: Nginx, служба GlusterFS для монтирования директории с файлами сайта (Master)

3. Кластерная файловая система (здесь храняться файлы сайта)
- GlusterFS1 IP: 192.168.10.44 Установлено: GlusterFS (Master)
- GlusterFS2 IP: 192.168.10.45 Установлено: GlusterFS (Master)
- GlusterFS3 IP: 192.168.10.46 Установлено: GlusterFS (Master)

4. Сервера приложений
- Backend1 IP: 192.168.10.47 Установлено: PHP, ProxySQL (Master)
- Backend2 IP: 192.168.10.48 Установлено: PHP, ProxySQL (Master)

5. Балансировщики нагрузки (Redis, MySQL) VIP: 192.168.10.53
- Balancer3 IP: 192.168.10.52 Установлено: KeepAlived, HAProxy (Master)
- Balancer4 IP: 192.168.10.54 Установлено: KeepAlived, HAProxy (Slave)

6. Сервера NoSQL БД Redis для хранения сессий и кэширования
- Redis1 IP: 192.168.10.49 Установлено: Redis, Sentinel (Master)
- Redis2 IP: 192.168.10.50 Установлено: Redis, Sentinel (Slave)
- Redis3 IP: 192.168.10.51 Установлено: Redis, Sentinel (Slave) 

7. Сервера БД (Percona XtraDB)
- Percona1 IP: 192.168.10.55 Установлено: Percona MySQL (Master)
- Percona2 IP: 192.168.10.56 Установлено: Percona MySQL (Master)
- Percona3 IP: 192.168.10.57 Установлено: Percona MySQL (Master)

### Описание ПО и проекта:

Проект представляет собой отказоустойчивый кластер на всех уровнях работы приложения. Таким образом, выход какой либо ноды не критичен для работы сервиса.

Используемое ПО:

1. KeepAlived - ПО, которое следит за состоянием приложения HAProxy, а также за состоянием другого хоста (в случае если Slave). Использует общий виртуальный IP адрес, который перетекает на ту ноду которая осталась жива либо на которой работает приложение HAProxy, необходимое для проксирования запросов.

2. HAProxy - проксирует пользовательские запросы. В проекте используется http, redis, MySQL проксирование (на balancer3 и balancer4 как запасное MySQL проксирование).

![Image 1](https://raw.githubusercontent.com/staybox/otus_project/master/screenshots/haproxy-web.png)

3. Nginx - высокопроизводительный веб-сервер. Выделен в отдельно стоящую группу серверов, для более эффективного размазывания нагрузки. Сервер в данном случае сжимает траффик, на что тратяться дополнительные ресурсы процессора.

4. Интерпретатор PHP 7.2 - установлен на отдельных серверах приложений

5. ProxySQL - ПО, которое позволяет проксировать SQL запросы по группам серверов (группы записи, группы чтения, запасная группа записи). Следить за доступностью SQL серверов, и автоматически выводит и вводит в строй. Имеет свой командный интерфейс управления (аналогичен работе с обычным SQL инстансом). Умеет определять тип запроса и отправлять на нужные сервера. В данном случае используется 1 сервер для записи, несмотря на то что у нас в схеме 3 мастер сервера. Сделано для того чтобы избежать взаимных блокировок (DeadLock). В данном проекте ProxySQL является основным проксирующим средством SQL запросов.

![Image 2](https://raw.githubusercontent.com/staybox/otus_project/master/screenshots/proxysql.png)

![Image 3](https://raw.githubusercontent.com/staybox/otus_project/master/screenshots/proxysql2.png)

6. GlusterFS - кластерная файловая система. Образует файловый кластер для файлового хранилища (где хранятся файлы сайта). Все веб-сервера и сервера приложений имеют доступ к файловому хранилищу для того чтобы сайт работал. На веб-серверах и серверах приложений установлен клиент GlusterFS.

7. Redis (6 версия) - NoSQL БД для хранения сессий и кэширования данных. В нашей схеме используется 1 мастер и 2 слэйва. В случае падения мастера, происходит переключение. За переключением следить Sentinel. Хранит данные в ОЗУ, за счет чего быстро записывает и отдает данные. Периодически сохраняет данные на диск.

8. Percona Multi-Master MySQL - сервера БД, которые хранят данные сайта в таблицах. Каждый читает и записывает с/на каждого.

![Image 4](https://raw.githubusercontent.com/staybox/otus_project/master/screenshots/percona.png)

Balancer3 и Balancer4 также могут проксировать SQL запросы, однако HAProxy не умеет понимать тип SQL запроса. Проксирование идет исходя из доступности нод.
Также же на этих нодах проксируется Redis, в данном случае HAProxy определяет кто из нод мастер и записывает только на мастер.

![Image 5](https://raw.githubusercontent.com/staybox/otus_project/master/screenshots/haproxy-redis.png)

![Image 6](https://raw.githubusercontent.com/staybox/otus_project/master/screenshots/wp.png)

![Image 7](https://raw.githubusercontent.com/staybox/otus_project/master/screenshots/wp3.png)

![Image 8](https://raw.githubusercontent.com/staybox/otus_project/master/screenshots/wp2.png)

### Схема:

![Image 9](https://raw.githubusercontent.com/staybox/otus_project/master/screenshots/schema_project.png)

### Как запустить:
- ```git clone git@github.com:staybox/otus_project.git && cd otus_project && vagrant up percona1 percona2 percona3 redis1 redis2 redis3 balancer3 balancer4 glusterfs1 glusterfs2 glusterfs3 && export ANSIBLE_CONFIG=$(pwd)/ansible-gluster/ansible.cfg && ansible-playbook ansible-gluster/provision.yml && vagrant up backend1 backend2 web1 web2 balancer1 balancer2```

