# Руководство по ручной установке **AmneziaVPN** с AmneziaWG на **VPS**

Подробное руководство по ручной установке AmneziaWG протокола и запуска AmneziaVPN на сервере с настройкой конфигурационных файлов на стороне сервера и клиента без каких-либо установочных скриптов, передачи данных и SSH-ключей вашего VPS клиенту Amnezia. Все компилируется и поднимается своими руками.

> [!IMPORTANT]
> Данное руководство предполагает исключительно настройку протокола amneziawg-go без возможности его смены на Xray / OpenVPN / Hysteria / SOCKS5 и любые другие возможные альтернативы, которые предлагает Amnezia из своего клиента.

## Предварительная подготовка

Предполагается, что к началу прочтения этого мануала вы выбрали хостера, оплатили аренду VPS и ваш сервер базово настроен (у вас есть стабильное подключение по паролю / SSH и есть пользователь с правами `sudo`) и готов к установке ПО.

> [!NOTE]
> В данном руководстве установка будет производиться на VPS с предустановленным и настроенным дистрибутивом Debian 12 (x86_64). Если у вас другой дистрибутив - команды могут отличаться, рекомендуется тщательно перепроверять по мере продвижения по мануалу.

## Установка Go и нужных пакетов

Если у вас чистый сервер, то можете воспользоваться данной [статьей](https://habr.com/ru/articles/756804/) по настройке вашего VPS для обеспечения базовой безопасности.

После чего возвращаемся сюда и производим установку и обновление нужных нам пакетов:

```
sudo apt update && sudo apt upgrade -y
sudo apt install -y nano make git build-essential wget
```

После обновления, необходимо включить пересылку пакетов между сетевыми интерфейсами, внеся изменения в конфигурационный файл. Открываем файл с помощью команды:

```
sudo nano /etc/sysctl.conf
```

В конец файла добавляем строчку `net.ipv4.ip_forward = 1`. Сохраняем сочетанием клавиш `Ctrl + O` и выходим с помощью `Ctrl + X`

Подхватываем изменения из файла командой:

```
sudo sysctl -p
```

AmneziaVPN использует собственный протокол amneziawg-go, который является форком [wireguard-go](https://github.com/WireGuard/wireguard-go) с возможностью настройки параметров обфускации трафика WireGuard в конфигурационных файлах. Так как эта реализация написана на языке программирования Go, для компиляции исходных файлов нам необходимо установить Go на наш сервер.

Для установки подходящего архива языка Go необходимо знать архитектуру процессора вашего сервера. Для определения этой характеристики используйте команду:

```
uname -m
```

Ответ должен быть в таком стиле: **x86_64**. Если вывод в консоли отличается, воспользуйтесь Google чтобы узнать ссылку на архив Go на основе архитектуры вашего процессора. Если же вывод совпадает, то скачиваем Go с помощью следующих команд:

```
cd ~ && mkdir go && cd go
wget https://go.dev/dl/go1.24.4.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.24.4.linux-amd64.tar.gz
```

> [!TIP]
> Учтите, что со временем кодовая база протокола может обновляться и возможно, вам нужна другая версия языка Go, отличная от той, что является актуальной на момент написания мануала. Это можно проверить в первой строчке Dockerfile [репозитория](https://github.com/amnezia-vpn/amneziawg-go/blob/master/Dockerfile) amneziawg-go. На сегодняшний день, актуальная версия - 1.24.4.

Теперь необходимо определить переменные окружения для Go:

```
sudo nano ~/.bashrc
```

Добавляем в конец файла следующие строчки:

```
export GOROOT=/usr/local/go
export GOPATH=$HOME/go
export PATH=$PATH:$GOROOT/bin:$GOPATH/bin
```

Сохраняем сочетанием клавиш `Ctrl + O` и выходим с помощью `Ctrl + X`

Подхватываем изменения из конфигурационного файла командой:

```
source ~/.bashrc
```

Проверяем, что все работает командой:

```
go version
```

Ожидаемый вывод:

```
go version go1.24.4 linux/amd64
```

## Сборка протокола amneziawg-go

Создаем папку для исходного кода и заходим в неё, после клонируем исходный код протокола amneziawg-go с помощью git:

```
cd ~ && mkdir amneziawg-go && cd amneziawg-go
git clone https://github.com/amnezia-vpn/amneziawg-go .
```

Компилируем исходный код командой:

```
make
```

Перемещаем скомпилированный бинарный файл в директорию _/usr/local/bin/_:

```
sudo mv amneziawg-go /usr/local/bin/
```

Выдаем права файлу на исполнение:

```
sudo chmod +x /usr/local/bin/amneziawg-go
```

## Сборка набора утилит amneziawg-tools

После успешной компиляции исходников самого протокола, необходимо установить утилиты для работы с AmneziaWG, они содержат команды **awg** и **awg-quick** (аналог **wg** и **wg-quick** у WireGuard):

```
cd ~ && mkdir amneziawg-tools && cd amneziawg-tools
git clone https://github.com/amnezia-vpn/amneziawg-tools.git .
```

Компилируем исходный код набора утилит командой:

```
cd src && make
```

Ожидаемый вывод компиляции должен быть следующим:

```
CC      wg.o
CC      config.o
CC      curve25519.o
CC      encoding.o
CC      genkey.o
CC      ipc.o
CC      pubkey.o
CC      set.o
CC      setconf.o
CC      show.o
CC      showconf.o
CC      terminal.o
LD      wg
```

Теперь устанавливаем утилиты в _/usr_ и _/ext_ директории для работы с протоколом командой:

```
sudo make install
```

Ожидаемый вывод:

```
'wg' -> '/usr/bin/awg'
'man/wg.8' -> '/usr/share/man/man8/awg.8'
'completion/wg.bash-completion' -> '/usr/share/bash-completion/completions/awg'
'wg-quick/linux.bash' -> '/usr/bin/awg-quick'
install: creating directory '/etc/amnezia'
install: creating directory '/etc/amnezia/amneziawg'
'man/wg-quick.8' -> '/usr/share/man/man8/awg-quick.8'
'completion/wg-quick.bash-completion' -> '/usr/share/bash-completion/completions/awg-quick'
'systemd/wg-quick.target' -> '/usr/lib/systemd/system/awg-quick.target'
'systemd/wg-quick@.service' -> '/usr/lib/systemd/system/awg-quick@.service'
```

## Установка модуля amneziawg-linux-kernel-module

> [!NOTE]
> Данный шаг можно пропустить, так как Amnezia может спокойно запускаться в Userspace и вполне работать без прямого общения с ядром Linux. Однако, если у вас много клиентов или вы хотите выжать максимум производительности, запуск Amnezia с помощью этого способа может существенно ускорить работу, приближая её к скорости WireGuard, который работает напрямую с ядром Linux.

Установим linux-kernel реализацию AmneziaWG. Этот модуль более низкоуровневый и работает напрямую через ядро Linux, а не полностью в виртуальном сетевом интерфейсе. В данном руководстве будет установка уже готовых релизов, но при желании все пакеты можно скомпилировать своими руками.

Устанавливаем модуль ядра Amnezia, выполняя все команды по очереди:

> [!IMPORTANT]
> Команды, используемые ниже актуальны только для дистрибутива Debian (рекомендуется перепроверить актуальные команды в README официального репозитория на момент установки). Команды установки ядра для других дистрибутивов можно найти в официальном [репозитории](https://github.com/amnezia-vpn/amneziawg-linux-kernel-module).

```
sudo apt install -y software-properties-common python3-launchpadlib gnupg2 linux-headers-$(uname -r)

sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 57290828

echo "deb https://ppa.launchpadcontent.net/amnezia/ppa/ubuntu focal main" | sudo tee -a /etc/apt/sources.list

echo "deb-src https://ppa.launchpadcontent.net/amnezia/ppa/ubuntu focal main" | sudo tee -a /etc/apt/sources.list

sudo apt-get update

sudo apt-get install -y amneziawg
```

## Настройка конфигурации WireGuard

Amnezia использует конфигурационные файлы WireGuard с добавлением параметров для шифрования трафика.

<details>
<summary>Пример шаблона конфигурации WireGuard для AmneziaVPN:</summary>

```ini
[Interface]
PrivateKey = <YOUR_PRIVATE_KEY>
Address = <YOUR_VPN_INTERFACE_IP>
DNS = 1.1.1.1 # Or your preferred DNS server

# AmneziaWG specific parameters (use your actual values)
Jc =
Jmin =
Jmax =
S1 =
S2 =
H1 =
H2 =
H3 =
H4 =

# PostUp: Commands executed after VPN interface (wg0) is up
PostUp = ip route add <YOUR_VPN_ENDPOINT_IP> via <YOUR_ETH0_GATEWAY_IP> dev eth0; \
         ip route del default via <YOUR_ETH0_GATEWAY_IP> dev eth0; \
         ip route add 0.0.0.0/0 dev wg0; \
         ip -6 route add ::/0 dev wg0

# PreDown: Commands executed before VPN interface (wg0) is taken down
PreDown = ip route del 0.0.0.0/0 dev wg0; \
          ip -6 route del ::/0 dev wg0; \
          ip route add default via <YOUR_ETH0_GATEWAY_IP> dev eth0; \
          ip route del <YOUR_VPN_ENDPOINT_IP> via <YOUR_ETH0_GATEWAY_IP> dev eth0

[Peer]
PublicKey = <YOUR_PEER_PUBLIC_KEY>
PresharedKey = <YOUR_PRESHARED_KEY>
AllowedIPs = 0.0.0.0/0, ::/0
Endpoint = <YOUR_VPN_ENDPOINT_IP>:<PORT>
PersistentKeepalive = 25
```

</details>
<br>

Это стандартный конфигурационный файл для захвата и перенаправления всего трафика на сервере в туннель. Однако такой шаблон переопределяет стандартный сетевой интерфейс, а означает, что весь трафик до нашего сервера будет проходить через туннель WireGuard. Если вы используете свой VPS исключительно для Amnezia - такой вариант подойдет вам, однако все же будет лучше если UDP-трафик Amnezia будет проходить только через конкретную подсеть.

Таким образом мы не рискуем потерять возможность стабильного подключения к VPS по SSH, в случае если конфигурация будет невалидная при первом запуске. А также, у нас будет возможность принимать и перенаправлять трафик на другие сетевые интерфейсы (например, на Xray, который будет стоять на этом же сервере как дополнительный способ обхода, или ваш собственный веб-сервер, раздающий какой-либо контент).

Создаем файл для конфигурации:

```
cd ~ && mkdir wg-config && cd wg-config
sudo touch wg0.conf
```

Теперь генерируем нужные ключи для конфигурации. Amnezia подменяет стандартные WireGuard утилиты своими алиасами, так команда **wg** становится `awg`, a \*\*wg-quick - `awg-quick`.

Генерируем публичные и приватные ключи в соответствующие файлы:

Серверные ключи:

```
sudo awg genkey | tee server_privatekey | awg pubkey > server_publickey
```

Клиентские ключи:

```
sudo awg genkey | tee client_privatekey | awg pubkey > client_publickey
```

Preshared ключ (по желанию - он является дополнительной мерой безопасности, так что его можно не указывать в конфигурации):

```
sudo awg genpsk
```

> [!NOTE]
> Обмен ключами происходит по следующей схеме: Сервер хранит свой приватный ключ (server_privatekey) и должен знать публичный ключ клиента (client_publickey). Клиент хранит свой приватный ключ (client_privatekey) и должен знать публичный ключ сервера (server_publickey). Для дополнительной защиты можно создать дополнительный общий ключ (Preshared key), который будет общим у сервера для клиента и у клиента для сервера. При заполнении конфигураций сервера и клиента PrivateKey - всегда будет являться приватным ключом текущей стороны (для сервера - ключ сервера, для клиента - ключ клиента), а PublicKey - публичным ключом противоположной стороны (для сервера - ключ клиента, для клиента - ключ сервера).

Если же вы все же хотите использовать ваш сервер исключительно для того, чтобы перенаправлять трафик в туннель WireGuard, то для команд **PostUp** и **PostDown** вместо значения подсети необходимо указать <YOUR_ETH0_GATEWAY_IP>.

> [!TIP]
> <YOUR_ETH0_GATEWAY_IP> - это IP-адрес шлюза вашего провайдера для интерфейса eth0 (или другого сетевого интерфейса вашей системы).

Чтобы узнать его выполните команду:

```
ip route show default
```

Ответ (IP-адрес после **via** - тот, что нам нужен):

```
default via <YOUR_ETH0_GATEWAY_IP> dev eth0 proto dhcp src <YOUR_VPN_ENDPOINT_IP> metric 100
```

> [!TIP]
> <YOUR_VPN_ENDPOINT_IP> - это публичный IP адрес сервера, который был выделен провайдером VPS и выслан вам на почту (естественно эти данные не должны быть вами утеряны). Также его можно узнать командой `sudo ip a` - IP-адрес будет указан в строке с вашим сетевым интерфейсом (например, eht0).

Редактируем конфиг на сервере по примеру ниже (команда для редактирования `sudo nano wg0.conf`):

```
[Interface]
Address = 10.10.10.1/24
ListenPort = 43287
PrivateKey = 6I/Bgspz99FeZR4hCkRPPCqvQBhsdxp1ySgF7Yw+Yno=

PostUp = iptables -I FORWARD 1 -i wg0 -j ACCEPT; iptables -I FORWARD 1 -o wg0 -j ACCEPT; iptables -t nat -I POSTROUTING 1 -s 10.10.10.0/24 -o eth0 -j MASQUERADE

PostDown = iptables -D FORWARD 1 -i wg0 -j ACCEPT; iptables -D FORWARD 1 -o wg0 -j ACCEPT; iptables -t nat -D POSTROUTING 1 -s 10.10.10.0/24 -o eth0 -j MASQUERADE

S1 = 0
S2 = 0
Jc = 4
Jmin = 40
Jmax = 70
H1 = 1
H2 = 2
H3 = 3
H4 = 4

[Peer]
# Client 1
PresharedKey = hzzjVCQuqgrJFNfpD8FK7HRC4gH8s1CeQJcioSZxxE8=
PublicKey = 99t441OC3kJyMTrKbewsGTLgTwM6JbIP/i5TgtGC2ws=
AllowedIPs = 10.10.10.2/32

[Peer]
# Client 2
PresharedKey = SMyr2dzx48scHrBsDIBuFIlglOeQJ8dotn5xSlv/os4=
PublicKey = psU7NT3InYY0XQmvkmqtrPCCmf/yTEBFstmHXuYYozM=
AllowedIPs = 10.10.10.3/32
```

> [!IMPORTANT]
> Обязательно используйте свои PrivateKey, PublicKey и PresharedKey ключи! Использование ключей, приводящихся в руководстве, повышает риск их компрометации. Также замените адрес (10.10.10.0) и порт (43287) подсети.

> [!IMPORTANT]
> Заметьте, что команды **PostUp** и **PostDown** устанавливают NAT-правила для интерфейса **wg0**. Имя конфигурационного файла должно совпадать с именем интерфейса, который вы поднимаете через `wg-quick` (например, **wg0.conf** → интерфейс **wg0**).

> [!NOTE]
> Для каждого клиента выделяется собственный IP-адрес в туннельной подсети, указанной в конфигурации сервера, использующийся на клиенте

> [!TIP]
> Значение Jmax не должно быть больше значения вашего **MTU**. Оставляйте его всегда меньше чем 1280, если не хотите разбираться и гуглить. Более подробно можно почитать [в этой статье](https://habr.com/ru/companies/amnezia/articles/807539/).

Конфигурация для клиента будет выглядеть следующим образом (Соответствует конфигурации Client 1):

```
[Interface]
Address = 10.10.10.2/32
PrivateKey = KGgeU6/uCJNyYVg7QSIhZZ/J1qHgJfYKAqdrOJLD8Xk=
DNS = 1.1.1.1

S1 = 0
S2 = 0
Jc = 4
Jmin = 40
Jmax = 70
H1 = 1
H2 = 2
H3 = 3
H4 = 4

[Peer]
PublicKey = QTw8CfqXcn3oZ6Dw7vdFHirHM6Z0Eg2g5w1Cp2D9uxM=
PresharedKey = hzzjVCQuqgrJFNfpD8FK7HRC4gH8s1CeQJcioSZxxE8=
Endpoint = [YOUR_VPN_ENDPOINT_IP]:43287
AllowedIPs = 0.0.0.0/0 # Захватываем весь трафик
PersistentKeepalive = 25
```

## Запуск AmneziaWG

Перемещаем серверный конфиг в директорию **/etc/amnezia/amneziawg/**:

```
sudo mv wg0.conf /etc/amnezia/amneziawg/
```

Запускаем Amnezia командой:

```
sudo awg-quick up wg0
```

Остановить процесс можно командой:

```
sudo awg-quick down wg0
```

Если в процессе вылезло предупреждение такого вида:

```
/usr/bin/awg-quick: line 32: resolvconf: command not found
```

То выполняем эти команды для установки пакета resolvconf:

```
sudo apt install -y resolvconf
sudo systemctl status resolvconf
```

После запуска AmneziaWG, просматриваем таблицу IP-роутов:

```
ip route show
```

Ищем строку, начинающуюся с 10.10.10.0/24:

Если сейчас мы отключим VPN-туннель AmneziaWG командой:

```
sudo awg-quick down wg0
```

То строка с 10.10.10.0/24 должна исчезнуть, как и сетевой интерфейс, который будет удален командой **PostDown** из **wg0.conf**.

> [!NOTE]
> Если у вас установлен **ufw** или любой другой firewall, то необходимо открыть порт для UDP-трафика до вашего сервера.

Настройка ufw:

```
sudo ufw allow 43287/udp comment "AmneziaWG"
sudo systemctl restart ufw
```

> [!NOTE]
> Если у вас настроен **fail2ban**, добавьте подсеть, куда перенаправляется трафик Amnezia (для надежности можно добавить также выделенные IP клиентов) в правило **ignoreip**.

Для подключения с клиента импортируйте файл с конфигурацией клиента в клиентское приложение Amnezia и тестируйте подключение. Enjoy!

## Приложение

### Если имеются какие-то проблемы после старта awg:

Полезные команды **iptables**:

- Список сетевых интерфейсов и их состояние: `ip link show`
- Вывод сетевых интерфейсов: `ip addr show`
- Вывод таблицы маршрутизации: `ip route show`
- Маршрут по умолчанию: `ip route show default`

---

Полезные команды **amnezia-tools** (<INTERFACE_NAME> в примерах выше - это **wg0**):

- Статус созданных awg туннелей: `sudo awg show`
- Создать туннель с конфигурацией: `sudo awg-quick up <INTERFACE_NAME>`
- Отключить туннель по имени интерфейса: `sudo awg-quick down <INTERFACE_NAME>`

---

Чтобы определить, работает Amnezia в userspace или через kernel-backend, можно посмотреть логи модуля командой `ip -d link show [CONF_FILENAME]`. Если в выводе содержится информация _tun type tun_ - это значит, что Amnezia работает через туннель в userspace, если же в выводе указан модуль _amneziawg_ - Amnezia работает напрямую с ядром.

---

Запуск отслеживания UDP-пакетов на определенном порту (полезно для понимания, доходят ли пакеты до сервера):

```
sudo tcpdump -n -i eth0 udp port [PORT] -vv
```

Попробуйте послать тестовый пакет вручную чтобы проверить, что порт открыт для UDP-трафика (Полезно в совокупности с предыдущей командой отслеживания пакетов на сервере). Команды для _PowerShell_:

```powershell
$udpClient = New-Object System.Net.Sockets.UdpClient
$payload = [System.Text.Encoding]::ASCII.GetBytes("hello from client")
$udpClient.Send($payload, $payload.Length, "[YOUR_VPN_ENDPOINT_IP]", [AMNEZIA_PORT])
$udpClient.Close()
```

---

### Использованные материалы при написании мануала

- https://habr.com/ru/articles/756804/
- https://habr.com/ru/companies/amnezia/articles/807539/
- https://github.com/amnezia-vpn/amnezia-client/issues/850
- https://ruvds.com/ru/helpcenter/nastroyka-vpn-s-ispolzovaniem-wireguard/
- https://gist.github.com/lanceliao/5d2977f417f34dda0e3d63ac7e217fd6

### Исходный код сервисов AmneziaVPN:

- [Исходный код протокола amneziawg-go](https://github.com/amnezia-vpn/amneziawg-go)
- [Утилиты для конфигурации AmneziaWG](https://github.com/amnezia-vpn/amneziawg-tools)
- [Kernel module](https://github.com/amnezia-vpn/amneziawg-linux-kernel-module)
