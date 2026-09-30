# Домашнее задание: Сценарии iptables

## Текст задания

1. Реализовать knocking port — centralRouter попадает на ssh inetRouter через knock-скрипт.
2. Добавить inetRouter2, который виден с хоста (host-only) или форвардится через localhost.
3. Запустить nginx на centralServer.
4. Пробросить 80-й порт на inetRouter2:8080.
5. Дефолт в интернет оставить через inetRouter.
6. (*) Реализовать проход на 80-й порт без маскарадинга.

## Схема сети

```
host (192.168.50.1 / 192.168.1.x)
   |
   +-- inetRouter2 (host-only: 192.168.50.2)   <-- виден с хоста
   |        |
   |        +-- dir-net (192.168.0.3/28) --> centralServer (192.168.0.2/28, nginx :80)
   |
   +-- inetRouter (NAT, ssh под knock-защитой)
            |
            +-- router-net (192.168.255.1/30) --> centralRouter (192.168.255.2/30)
                                                       |
                                                       +-- dir-net (192.168.0.1/28)

Дефолт в интернет: centralServer -> centralRouter -> inetRouter (inetRouter2 в пути НЕ участвует).
```

## Реализация

Стенд развёрнут на **Proxmox** (ВМ Ubuntu 24.04)

### 1. Knocking port (inetRouter)

- `knockd` на inetRouter слушает интерфейс к centralRouter, последовательность **7000,8000,9000** (tcp SYN).
- Базовые правила: `-A INPUT -p tcp --dport 22 -j DROP` — ssh закрыт для всех, кроме mgmt-интерфейса.
- **Важно:** knockd добавляет правило через `iptables -I INPUT 1` (вставка ПЕРЕД DROP-правилом).
  С `-A` (append) ACCEPT оказался бы после DROP и соединение рубилось — это ключевая ошибка,
  которую пришлось исправить.
- Обратная последовательность 9000,8000,7000 удаляет правило.

Knock-скрипт на centralRouter (`/usr/local/bin/knock-ssh.sh`): стучится в порты через `/dev/tcp`,
затем подключается по ssh:

```bash
for p in 7000 8000 9000; do timeout 1 bash -c "echo > /dev/tcp/192.168.255.1/$p"; done
sleep 1; ssh root@192.168.255.1
```

Проверка (лог knockd на inetRouter):

```
192.168.255.2: openSSH: Stage 1 / Stage 2 / Stage 3 / OPEN SESAME
openSSH: running command: iptables -I INPUT 1 -s 192.168.255.2 -p tcp --dport 22 -j ACCEPT
```

После knock ssh-хендшейк проходит (`Permission denied` на этапе аутентификации — дальше firewall не блокирует);
до knock — `Connection timed out`.

### 2. inetRouter2 + nginx + проброс 80→8080

- ВМ inetRouter2: net0 — host-only (видна с хоста: `192.168.1.132`, в Vagrant-версии `192.168.50.2`), net1 — dir-net (`192.168.0.3/28`).
- nginx установлен и запущен на centralServer (`:80`, HTTP 200).
- На inetRouter2: `ip_forward=1` и DNAT:

```bash
iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 192.168.0.2:80
iptables -A FORWARD -p tcp -d 192.168.0.2 --dport 80 -j ACCEPT
```

Проверка с хоста:

```
$ curl -m 6 -o /dev/null -w '%{http_code}' http://192.168.1.132:8080/
200
```

### 3. Дефолт в интернет через inetRouter

inetRouter2 включён в схему «сбоку»: у centralServer/default маршрут не меняется.

```
traceroute 8.8.8.8 с centralServer:
 1  192.168.0.1     (centralRouter)
 2  192.168.255.1   (inetRouter)
 3  192.168.1.1     (NAT хоста)
```

### (*) 80-й порт без маскарадинга

- На inetRouter2 **нет** POSTROUTING MASQUERADE (`iptables -t nat -S POSTROUTING | grep -c MASQUERADE` → `0`).
- Вместо него на centralServer добавлен обратный маршрут: `192.168.1.0/24 via 192.168.0.3`
  (закреплён в netplan), поэтому ответы на хост уходят напрямую через inetRouter2 с
  сохранением исходных адресов.
- Проверка: `curl http://192.168.1.132:8080/` → **HTTP 200** без NAT-подмены адреса.

## Сохранение правил после перезагрузки

На inetRouter и inetRouter2 правила сохранены через `iptables-save` → `/etc/iptables_rules.ipv4`
и восстановливаются юнитом `iptables-restore.service` (systemd, аналог `if-pre-up.d` из методички).
knockd включён в автозагрузку.

## Особенности и заметки

- Ключевой момент knocking: правило от knockd должно вставляться **до** DROP (`-I INPUT 1`), иначе не работает.
- Для безмаскарадинговой схемы нужен **обратный маршрут** на сервере — иначе ответы уйдут через дефолт (centralRouter→inetRouter) и соединение не установится.
- В Vagrant-версии интерфейсы названы enp0s3/enp0s8 (VirtualBox), в Proxmox-версии enp6s18/enp6s19 — в плейбуке порядок задач идентичен.
- ufw на inetRouter отключён, чтобы не конфликтовал с ручными правилами iptables.

## Файлы репозитория

| Файл | Назначение |
|---|---|
| `Vagrantfile` | стенд: inetRouter, inetRouter2, centralRouter, centralServer |
| `ansible/hosts` | инвентарь |
| `ansible/provision.yml` | вся настройка: knocking, DNAT, nginx, обратный маршрут, персист правил |
| `README.md` | этот файл |
