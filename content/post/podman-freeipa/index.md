---
title: Как запустить и настроить FreeIPA сервер в Podman контейнере
date: 2025-02-10
slug: podman-freeipa
categories:
    - Заметки
---

## Шаги по настройке

### 1. Запуск команды для настройки образа

Запустим контейнер командой:

```sh
podman run --name freeipa-server-container -ti \
    -h ipa.example.test --read-only \
    -v /home/<USER>/.docker/freeipa/data:/data:Z freeipa/freeipa-server:fedora-41-4.12.2
```

### 2. Прохождение процесса настройки сервера

Пройдите через процесс настройки сервера.

### 3. Остановка процесса и подготовка Docker Compose файла

Остановите процесс (Ctrl + C) и подготовьте `docker-compose.yml` файл, который будет использовать данные из `/home/<USER>/.docker/freeipa/data`.

#### Мой `docker-compose.yml` файл

Обратите внимание, что имя хоста (`hostname`) должно совпадать с именем, указанным в первой команде.

```yml
services:
  freeipa:
    image: freeipa/freeipa-server:fedora-41-4.12.2
    container_name: freeipa-server-container-compose
    hostname: ipa.example.test
    read_only: true
    environment:
      - PASSWORD=YourStrongAdminPassword123
      - IPA_DOMAIN=example.test
      - IPA_REALM=EXAMPLE.TEST
      - NO_REDIRECT=true
    ports:
      - "5553:53/udp"        # DNS (UDP)
      - "5553:53"            # DNS (TCP)
      - "1088:88/udp"        # Kerberos (UDP)
      - "1088:88"            # Kerberos (TCP)
      - "3389:389"           # LDAP
      - "6636:636"           # LDAPS
      - "4464:464/udp"       # Kerberos Change/Set password (UDP)
      - "4464:464"           # Kerberos Change/Set password (TCP)
      - "7389:7389"          # Directory Server Port
      - "9443:9443"          # Web UI
      - "8080:80"            # HTTP
      - "4343:443"           # HTTPS
    volumes:
      - /home/igor/.docker/freeipa/data:/data:Z
    restart: always
```

### 4. Добавление записи в `/etc/hosts`

Добавьте следующую строку в ваш файл `/etc/hosts`:

```bash
127.0.0.1 ipa.example.test
```

### 5. Настройка перенаправления портов

Теперь настроим перенаправление портов. Из-за ограничений rootless режима Podman сразу указать системные порты невозможно, поэтому используем следующие правила:

```sh
sudo iptables -t nat -A OUTPUT -d 127.0.0.1 -p tcp --dport 443 -j REDIRECT --to-port 4343
sudo iptables -t nat -A OUTPUT -d 127.0.0.1 -p tcp --dport 80 -j REDIRECT --to-port 8080

# DNS
sudo iptables -t nat -A OUTPUT -d 127.0.0.1 -p tcp --dport 53 -j REDIRECT --to-port 5553
sudo iptables -t nat -A OUTPUT -d 127.0.0.1 -p udp --dport 53 -j REDIRECT --to-port 5553

# Kerberos
sudo iptables -t nat -A OUTPUT -d 127.0.0.1 -p tcp --dport 88 -j REDIRECT --to-port 1088
sudo iptables -t nat -A OUTPUT -d 127.0.0.1 -p udp --dport 88 -j REDIRECT --to-port 1088

# LDAP
sudo iptables -t nat -A OUTPUT -d 127.0.0.1 -p tcp --dport 389 -j REDIRECT --to-port 3389

# LDAPS
sudo iptables -t nat -A OUTPUT -d 127.0.0.1 -p tcp --dport 636 -j REDIRECT --to-port 6636

# Kerberos password change
sudo iptables -t nat -A OUTPUT -d 127.0.0.1 -p tcp --dport 464 -j REDIRECT --to-port 4464
sudo iptables -t nat -A OUTPUT -d 127.0.0.1 -p udp --dport 464 -j REDIRECT --to-port 4464
```

### 6. Доступ к веб-интерфейсу

После выполнения всех шагов вы можете зайти на веб-интерфейс FreeIPA по адресу:

[https://ipa.example.test/ipa/ui/#/e/group/search](https://ipa.example.test/ipa/ui/#/e/group/search)

Voilà!