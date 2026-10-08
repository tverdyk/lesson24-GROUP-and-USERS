# Домашнее Задание

# Описание домашнего задания
Ограничить доступ к системе для всех пользователей, кроме группы администраторов, в выходные дни (суббота и воскресенье), за исключением праздничных дней.

⭐️ Задание со звездочкой
Предоставить определённому пользователю доступ к Docker и право перезапускать Docker-сервис.

ОПИСАНИЕ ПО ЗАДАНИЮ 

Практическое задание выполняется без Vagrant на виртуалке KVM  OS Ubuntu 20.04.

Создание УЗ otus и otusadm с паролем
      
      root@ubuntu:~# useradd otusadm && useradd otus
      #
      root@ubuntu:~# passwd otusadm 
      New password: 
      Retype new password: 
      passwd: password updated successfully
      #  
      root@ubuntu:~# passwd otus 
      New password: 
      Retype new password: 
      passwd: password updated successfully

Создание Группы админов

    root@ubuntu:~# groupadd -f admin

Добавляем в группу УЗ root и otusadm

    root@ubuntu:~# usermod -aG admin otusadm 
    root@ubuntu:~# usermod -aG admin root 

Проврка что УЗ добавлены в Группу

    root@ubuntu:~# cat /etc/group | grep admin
    admin:x:1004:otusadm,root

Создане скрипта PAM-аутентификации для уз
Сам скрипт из методички

    #!/bin/bash
    #Первое условие: если день недели суббота или воскресенье
    if [ $(date +%a) = "Sat" ] || [ $(date +%a) = "Sun" ]; then
     #Второе условие: входит ли пользователь в группу admin
     if getent group admin | grep -qw "$PAM_USER"; then
            #Если пользователь входит в группу admin, то он может подключиться
            exit 0
          else
            #Иначе ошибка (не сможет подключиться)
            exit 1
        fi
      #Если день не выходной, то подключиться может любой пользователь
      else
        exit 0
    fi

 Добавим права на исполнение скрипта
   
    chmod +x /usr/local/bin/login.sh

Добавим стоку в /etc/pam.d/sshd под @include common-auth, чтоб правила авторизации работали.

    root@ubuntu:~# nano /etc/pam.d/sshd 
    # PAM configuration for the Secure Shell service

    # Standard Un*x authentication.
    @include common-auth

    auth required pam_exec.so debug /usr/local/bin/login.sh # НАША СТРОКА!!!

Настройка завершена. Для проверки необходимо авторизоватся в Субботу или Воскресенье на вм под УЗ.
Для проверки " сейчас " остановить сервис 

    # systemctl stop systemd-timesyncd.service

 Установить дату субботу 
     
     date 082712302022.00
     
 Проверка 

     oot@ubuntu:~# systemctl stop systemd-timesyncd.service 
    root@ubuntu:~# date 082712302022.00
    Sat Aug 27 12:30:00 UTC 2022
    root@ubuntu:~# date
    Sat Aug 27 12:30:02 UTC 2022


Проверка авторизации

     ssh otusadm@192.168.122.2
      updates could not be installed automatically. For more details,
     see /var/log/unattended-upgrades/unattended-upgrades.log

    Last login: Thu Oct  8 12:42:24 2026 from 192.168.122.1
Авторизация для otusadm успешна. 
Не забудьте вернуть сервис systemd-timesyncd.service в работу после проверки.

Авторизация для otus не проходит, как и запланировано. 

    ssh otus@192.168.122.2
    ** WARNING: connection is not using a post-quantum key exchange algorithm.
    ** This session may be vulnerable to "store now, decrypt later" attacks.
    ** The server may need to be upgraded. See https://openssh.com/pq.html
    otus@192.168.122.2's password: 
    Permission denied, please try again.
    otus@192.168.122.2's password: 
    Permission denied, please try again.
    otus@192.168.122.2's password: 
    otus@192.168.122.2: Permission denied (publickey,password).

Предоставить определённому пользователю доступ к Docker и право перезапускать Docker-сервис.

 Добавим УЗ otusadm в группу docker 

    usermod -aG docker otusadm 

Создадим в каталоге /etc/sudoers.d фаил с именем сервиса doker и добавить строку

    nano /etc/sudoers.d/docker
    
    otusadm  ALL=(root) NOPASSWD: /bin/systemctl restart docker

Теперь Уз otusadm может проверить статус контейнеров и перезапускать docker.service

    ssh otusadm@192.168.122.2
     updates could not be installed automatically. For more details,
    see /var/log/unattended-upgrades/unattended-upgrades.log
    
    Last login: Sat Aug 27 12:31:23 2022 from 192.168.122.1
    $ docker ps
    CONTAINER ID   IMAGE                       COMMAND                  CREATED                  STATUS                  PORTS                                       NAMES
    8658ab4de50d   grafana/grafana:latest      "/run.sh"                Less than a second ago   Up Less than a second   0.0.0.0:3000->3000/tcp, :::3000->3000/tcp   grafana
    8705b305ab16   prom/prometheus:latest      "/bin/prometheus --c…"   Less than a second ago   Up Less than a second   0.0.0.0:9090->9090/tcp, :::9090->9090/tcp   prometheus
    497ab6b2bb69   prom/node-exporter:latest   "/bin/node_exporter …"   Less than a second ago   Up Less than a second   0.0.0.0:9100->9100/tcp, :::9100->9100/tcp   node_exporter
     sudo systemctl restart docker       
    $ sudo systemctl status docker
    [sudo] password for otusadm: 
    ● docker.service - Docker Application Container Engine
         Loaded: loaded (/lib/systemd/system/docker.service; enabled; vendor preset: enabled)
         Active: active (running) since Sat 2022-08-27 12:43:16 UTC; 34s ago

Это успех.
