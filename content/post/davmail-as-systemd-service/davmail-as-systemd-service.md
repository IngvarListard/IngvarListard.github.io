---
title: noteName
fileName: davmail-as-systemd-service
tags:
- Linux
categories:
- Заметки
date: 2024-01-17
lastMod: 2024-01-17
---
## Введение

  + За последние 3 года я несколько раз прибегал к использованию davmail и конкретной статьи https://semanticlab.net/desktop/e-mail/linux/sysadmin/Managing-DavMail-with-systemd-and-preventing-service-timeouts-after-network-reconnects/. Поэтому решил сделать перевод в свои заметки и опубликовать здесь.

  + [DavMail](https://davmail.sourceforge.net)обеспечивает доступ к серверам Exchange по стандартным протоколам, таким как IMAP, SMTP и Caldav. Таким образом, он позволяет пользоваться корпоративной электронной почтой с помощью вашего любимого почтового клиента, например Mail Spring, Thunderbird, Geary.

  + Ниже описано, как автоматизировать запуск DavMail посредством создания systemd-сервиса и обеспечить его работоспособность даже после переподключения к сети.

## Запуск DavMail с помощью systemd

  + Если ваш дистрибутив не предоставляет конфигурационный файл DavMail для systemd, то вставьте сниппет ниже по пути `/etc/systemd/system/davmail.service`.
```ini
[Unit]
Description=Davmail Exchange gateway
Documentation=man:davmail
Documentation=https://davmail.sourceforge.net/serversetup.html
Documentation=https://davmail.sourceforge.net/advanced.html
Documentation=https://davmail.sourceforge.net/sslsetup.html
After=network.target

[Service]
Type=simple
User=davmail
PermissionsStartOnly=true
ExecStartPre=/usr/bin/touch /var/log/davmail.log
ExecStartPre=/bin/chown davmail:adm /var/log/davmail.log
ExecStart=/usr/bin/davmail -server /etc/davmail.properties
SuccessExitStatus=143
PrivateTmp=yes

[Install]
WantedBy=multi-user.target
```

  + После чего необходимо создать пользователя DavMail и запустить сервис systemd:
```sh
adduser --system davmail
systemctl daemon-reload
systemctl enable davmail
systemctl start davmail
```

## Обработка переключений сети

  + Одной из основных проблем DavMail является обработка переподключения к  к сети (например, если вы меняете сеть или подключаетесь к VPN), поскольку они требуют перезапуска службы, чтобы предотвратить тайм-ауты при доступе к вашей электронной почте. Одним из способов решения этой проблемы является использование службы NetworkManager-dispatcher. Её можно запустить так:
```sh
systemctl enable NetworkManager-dispatcher
systemctl start NetworkManager-dispatcher
```

  + После запуска эта служба позволяет запускать скрипты если доступ к сети пропал либо при повторном подключении. Скрипт ниже останавливает службу DavMail если сеть становится недоступной, и перезапускает службу после подключения к сети.
```sh
#!/bin/sh

# stop davmail, if no network connectivity is available and restart it once
# the network becomes available.

interface=$1 status=$2

case $status in
  up)
      systemctl restart davmail
      ;;
  down)
      systemctl stop davmail
      ;;
esac
```

  + Вы можете включить автоматический перезапуск службы DavMail поместив этот скрипт по пути `/etc/NetworkManager/dispatcher.sh` и сделать его исполняемым с помощью команды `chmod a+x /etc/NetworkManager/dispatcher.sh`
