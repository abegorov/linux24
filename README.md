# Настраиваем центральный сервер для сбора логов

## Задание

1. В **vagrant** поднимаем 3 машины **web**, **log**, **elk**.
2. На **web** поднимаем **nginx**.
3. На **log** настраиваем центральный лог сервер на любой системе на выбор.
4. Настраиваем аудит, следящий за изменением конфигов **nginx**.

- Все критичные логи с **web** должны собираться и локально и удаленно.
- Все логи с **nginx** должны уходить на удаленный сервер (локально только критичные).
- Логи аудита должны также уходить на удаленную систему.
- В **elk** должны уходить только логи **nginx**.

## Реализация

Задание сделано на **debian/bookworm64** версии **v12.20240905.1**. Для автоматизации процесса написаны следующие роли **Ansible**, переменные для которых хранятся в [group_vars/all.yml](group_vars/all.yml), [hostvars/web.yml](host_vars/web.yml), [hostvars/log.yml](host_vars/log.yml), [hostvars/elk.yml](host_vars/elk.yml):

- **rsyslog** - настраивает **rsyslog**. На узлах **web** и **elk** стандартный конфиг дополняется отправкой лога аудита ([audit.conf](roles/rsyslog/templates/audit.conf)) и отправкой логов на централизованный сервер ([client.conf](roles/rsyslog/templates/client.conf)). На узле **log** лог добавляется настройками приёма логов по **TCP** и **UDP** в файле [server.conf](roles/rsyslog/templates/server.conf).
- **auditd** - устанавливает **auditd**. На узле **nginx** также настраивает мониторинг изменний файлов в `/etc/nginx` с помощью правил [nginx.rules](roles/auditd/templates/nginx.rules).
- **nginx** - устанавливает **nginx** и настраивает его для отправки логов на серер **log** через его конфигурацию в файл [nginx.conf](roles/nginx/templates/nginx.conf)/

## Запуск

Необходимо скачать **VagrantBox** для **debian/bookworm64** версии **v12.20240905.1** и добавить его в **Vagrant** под именем **debian/bookworm64/v12.20240905.1**. Сделать это можно командами:

```shell
curl -OL https://app.vagrantup.com/debian/boxes/bookworm64/versions/12.20240905.1/providers/virtualbox/unknown/vagrant.box
vagrant box add vagrant.box --name "debian/bookworm64/v12.20240905.1"
rm vagrant.box
```

Для того, чтобы **vagrant 2.3.7** работал с **VirtualBox 7.1.0** необходимо добавить эту версию в **driver_map** в файле **/usr/share/vagrant/gems/gems/vagrant-2.3.7/plugins/providers/virtualbox/driver/meta.rb**:

```ruby
          driver_map   = {
            "4.0" => Version_4_0,
            "4.1" => Version_4_1,
            "4.2" => Version_4_2,
            "4.3" => Version_4_3,
            "5.0" => Version_5_0,
            "5.1" => Version_5_1,
            "5.2" => Version_5_2,
            "6.0" => Version_6_0,
            "6.1" => Version_6_1,
            "7.0" => Version_7_0,
            "7.1" => Version_7_0,
          }
```

После этого нужно сделать **vagrant up**.

Протестировано в **OpenSUSE Tumbleweed**:

- **Vagrant 2.3.7**
- **VirtualBox 7.1.0_SUSE r164728**
- **Ansible 2.17.4**
- **Python 3.11.10**
- **Jinja2 3.1.4**

## Проверка

Выполним:

```shell
curl -s -o /dev/null http://192.168.56.10
vagrant ssh log -c 'sudo ls -l /var/log/rsyslog/web':
```

```text
total 228
-rw-r--r-- 1 root root    104 Oct  5 20:20 '(sd-pam).log'
-rw-r--r-- 1 root root    562 Oct  5 20:19  ansible-ansible.builtin.apt.log
-rw-r--r-- 1 root root    427 Oct  5 20:19  ansible-ansible.builtin.systemd_.log
-rw-r--r-- 1 root root    274 Oct  5 20:19  ansible-ansible.legacy.command.log
-rw-r--r-- 1 root root    538 Oct  5 20:19  ansible-ansible.legacy.copy.log
-rw-r--r-- 1 root root    426 Oct  5 20:19  ansible-ansible.legacy.file.log
-rw-r--r-- 1 root root    126 Oct  5 20:19  ansible-ansible.legacy.slurp.log
-rw-r--r-- 1 root root    203 Oct  5 20:19  ansible-ansible.legacy.stat.log
-rw-r--r-- 1 root root 168354 Oct  5 20:20  audit.log
-rw-r--r-- 1 root root    106 Oct  5 20:19  nginx.log
-rw-r--r-- 1 root root   1069 Oct  5 20:31  nginx_access.log
-rw-r--r-- 1 root root    434 Oct  5 20:19  rsyslogd.log
-rw-r--r-- 1 root root    401 Oct  5 20:20  sshd.log
-rw-r--r-- 1 root root   4045 Oct  5 20:19  sudo.log
-rw-r--r-- 1 root root    348 Oct  5 20:20  systemd-logind.log
-rw-r--r-- 1 root root   3110 Oct  5 20:20  systemd.log
```

Как видно из вывода, на сервере присутствуют лог **nginx** и **audit**, который мы можем дополнительно посмотреть:

```text
❯ vagrant ssh log -c 'sudo cat /var/log/rsyslog/web/audit.log' | grep '/etc/nginx/nginx.conf'
2024-10-05T20:19:04+00:00 web audit type=PATH msg=audit(1728159544.101:105): item=1 name="/etc/nginx/nginx.conf.dpkg-new" inode=3408936 dev=08:01 mode=0100000 ouid=0 ogid=0 rdev=00:00 nametype=CREATE cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0#035OUID="root" OGID="root"
2024-10-05T20:19:04+00:00 web audit type=PATH msg=audit(1728159544.189:148): item=2 name="/etc/nginx/nginx.conf.dpkg-new" inode=3408936 dev=08:01 mode=0100644 ouid=0 ogid=0 rdev=00:00 nametype=DELETE cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0#035OUID="root" OGID="root"
2024-10-05T20:19:04+00:00 web audit type=PATH msg=audit(1728159544.189:148): item=3 name="/etc/nginx/nginx.conf" inode=3408936 dev=08:01 mode=0100644 ouid=0 ogid=0 rdev=00:00 nametype=CREATE cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0#035OUID="root" OGID="root"
2024-10-05T20:19:05+00:00 web audit type=PATH msg=audit(1728159545.650:218): item=3 name="/etc/nginx/nginx.conf" inode=3408936 dev=08:01 mode=0100644 ouid=0 ogid=0 rdev=00:00 nametype=DELETE cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0#035OUID="root" OGID="root"
2024-10-05T20:19:05+00:00 web audit type=PATH msg=audit(1728159545.650:218): item=4 name="/etc/nginx/nginx.conf" inode=6291472 dev=08:01 mode=0100644 ouid=0 ogid=0 rdev=00:00 nametype=CREATE cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0#035OUID="root" OGID="root"
```
