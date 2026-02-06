# SSH
### Проброс портов
Знакома ситуация? На сервере крутится PostgreSQL, Redis или веб-интерфейс, но они закрыты и слушают только 127.0.0.1. В целях безопасности — правильно. Но как подключиться?

💡Решение — SSH-туннель. 1 команда:
```bash
ssh -L 6543:localhost:5432 user@backend-prod-01
```
Что произошло:
🔹На твоём ноуте открылся порт 6543
🔹Весь трафик с него шифрованно идёт на backend-prod-01:5432
🔹Подключайся в DBeaver к localhost:6543 — работает!

Делаем навсегда удобным в ~/.ssh/config:
```bash
Host backend-tunnel
    HostName backend-prod-01
    User deploy
    LocalForward 6543 localhost:5432  # PostgreSQL
    LocalForward 9100 localhost:3000  # Grafana
```
Теперь просто: 
```bash
ssh backend-tunnel -Nf
```
и оба порта доступны локально.
Обратная задача — дать коллеге доступ к твоему localhost:
```bash
ssh -R 9090:localhost:8080 bastion-server
```
Почему это безопасно?
1️⃣Весь трафик шифруется через SSH
2️⃣Доступ есть только у тех, у кого есть SSH-ключ
3️⃣Не нужно открывать порты в firewall

### Использование джамп-хоста
У нас есть бастион bastion.mycompany.com, а за ним сервер *app-server-1* с серыми IP 10.0.5.10.

Открываем _~/.ssh/config_ и пишем:
```bash
# Наш прыжковый хост (бастион)
Host bastion
    HostName bastion.mycompany.com
    User vasya
    IdentityFile ~/.ssh/id_ed25519_work

# Конечный сервер, доступный ТОЛЬКО через бастион
Host app1
    HostName 10.0.5.10
    User deploy
    IdentityFile ~/.ssh/id_ed25519_work
    ProxyJump bastion  # <--- Вся магия здесь
```
Теперь, чтобы попасть на app-server-1, просто пишешь одну команду:
```bash
ssh app1
```
OpenSSH сам установит соединение с бастионом и пробросит туннель до конечного сервера. Всё прозрачно. Ключи подтягиваются автоматически. Никаких лишних действий.

#### Ускоряем всё в разы с ControlMaster

Если вам часто требуется открывать параллельные сессии по одним и тем же хостам, добавьте мультиплексирование.

Первое соединение станет "мастером", а все последующие будут летать через уже открытый сокет, экономя время на handshake (рукопожатии).

Дополним наш конфиг для бастиона:
```bash
Host bastion
    HostName bastion.mycompany.com
    User vasya
    IdentityFile ~/.ssh/id_ed25519_work
    # Включаем мультиплексирование
    ControlMaster auto
    # Файл-сокет для управления сессией
    ControlPath ~/.ssh/cm-%r@%h:%p
    # Держим мастер-сессию 10 минут после выхода
    ControlPersist 10m
```
Теперь повторные подключения к бастиону или через него будут мгновенными.
