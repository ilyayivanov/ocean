# Полное руководство по развертыванию домашнего сервера на базе Proxmox VE, TrueNAS и Ubuntu Server (v6)

Данная инструкция составлена с учетом следующей конфигурации оборудования:
* **Процессор:** Intel Core i5-14400 (10 ядер / 16 потоков, LGA 1700)
* **Материнская плата:** Gigabyte B760M DS3H DDR4
* **Оперативная память:** 32 ГБ DDR4-3200 (2×16 ГБ Crucial)
* **Системный накопитель:** 512 ГБ NVMe SSD Kingston KC3000 (под гипервизор и диски ВМ)
* **Накопители для данных:** Жесткие диски WD Red Plus 4 ТБ (WD40EFPX)
* **Блок питания и корпус:** Cooler Master MWE Bronze 550W, корпус 1STPLAYER PROSTATION 8 MAX
* **ИБП:** CyberPower CP900EPFCLCD

---

## Архитектура решения

1. **Proxmox VE** устанавливается на быстрый NVMe SSD (Kingston KC3000).
2. **Виртуальная машина №1 (TrueNAS Community Edition / SCALE):** выделяется 4 ядра CPU и 16 ГБ ОЗУ. Встроенный SATA-контроллер материнской платы пробрасывается напрямую в эту ВМ (PCI Passthrough), благодаря чему ZFS работает с жесткими дисками напрямую без посредников.
3. **Виртуальная машина №2 (Ubuntu Server 24.04 LTS):** выделяется 4 ядра CPU, 8 ГБ ОЗУ и 64 ГБ на SSD. Здесь работают Docker-контейнеры (изолированная связка `Gluetun + qBittorrent` для скачивания торрентов строго через VPN) и сервисы маршрутизации.
4. **Взаимодействие между ВМ:** Ubuntu монтирует сетевую папку из TrueNAS по протоколу NFS через внутренний виртуальный коммутатор Proxmox на скорости 10–40 Гбит/с.

---

## Этап 1. Подготовка и настройка BIOS

Перед установкой системы необходимо включить аппаратную виртуализацию и проброс устройств на материнской плате **Gigabyte B760M DS3H DDR4**.

1. **Подключите накопители.** Установите SSD Kingston KC3000 в верхний слот M.2, а жесткие диски подключите к портам SATA на материнской плате.
2. **Зайдите в BIOS.** Включите сервер и нажимайте клавишу `Delete` при загрузке. Переключитесь в расширенный режим нажатием `F2` (*Advanced Mode*).
3. **Включите виртуализацию и IOMMU:**
   * Перейдите во вкладку **Tweaker** -> **Advanced CPU Settings** -> включите **Intel (VMX) Virtualization Technology** (`Enabled`).
   * Перейдите во вкладку **Settings** -> **IO Ports** -> включите **VT-d** (`Enabled`).
   * В этом же разделе найдите **Above 4G Decoding** и переведите в `Enabled`.
4. **Настройте автовключение после сбоя питания:**
   * В разделе **Settings** -> **Platform Power** установите параметр **AC BACK** в значение `Always On` (сервер автоматически запустится после восстановления электропитания, если батарея ИБП разрядится полностью).
5. **Сохраните настройки.** Нажмите `F10` и подтвердите выход.

---

## Этап 2. Установка Proxmox VE на SSD

1. **Скачайте ISO-образы систем на свой ПК:**
   * **Proxmox VE ISO** (`proxmox.com`)
   * **TrueNAS Community Edition / SCALE ISO** (`truenas.com`)
   * **Ubuntu Server 24.04 LTS ISO** (`ubuntu.com`)
2. **Запишите образ Proxmox на USB-флешку.** Используйте программу **Rufus** (в режиме *DD Image*) или **BalenaEtcher**.
3. **Загрузитесь с флешки.** Нажмите `F12` при старте сервера и выберите загрузочную флешку. В меню установки выберите `Install Proxmox VE (Graphical)`.
4. **Выберите целевой диск.** Укажите накопитель **Kingston KC3000 512GB**. Нажмите `Options` и убедитесь, что выбрана файловая система `ext4` (для одиночного системного SSD она оптимальна и не расходует лишнюю ОЗУ).
5. **Укажите регион и пароль.** Выберите страну, часовой пояс (`Europe/Moscow`) и задайте надежный пароль администратора `root`.
6. **Настройте сеть:**
   * *Hostname:* `ocean.home.arpa`
   * *IP Address (CIDR):* статический адрес из вашей сети, например `192.168.89.201/24`
   * *Gateway:* IP-адрес вашего роутера, например `192.168.89.1`
   * *DNS Server:* `192.168.89.1`
7. **Завершите установку.** После перезагрузки извлеките флешку. Веб-интерфейс Proxmox станет доступен по адресу `https://192.168.89.201:8006`.

---

## Этап 3. Первичная настройка Proxmox и включение проброса дисков (IOMMU)

1. **Войдите в веб-интерфейс Proxmox.** Откройте в браузере `https://192.168.89.201:8006`, введите логин `root` и пароль, заданный при установке.

2. **Настройте репозитории без подписки (`no-subscription`) и обновите систему.** По умолчанию в Proxmox включены корпоративные репозитории (`enterprise`), без платной лицензии `apt update` выдает ошибку доступа `401 Unauthorized`. Проще всего переключить их через веб-интерфейс (работает одинаково в Proxmox VE 8 и Proxmox VE 9):
   * В левом меню выберите узел `ocean` -> **Updates** -> **Repositories**.
   * Выделите по очереди строки с пометкой `enterprise` (для Proxmox и Ceph) и нажмите кнопку **Disable** сверху.
   * Нажмите **Add**, выберите в выпадающем списке **No-Subscription** и нажмите **Add** для добавления бесплатного репозитория.
   * Откройте консоль узла (`ocean` -> **Shell**) и обновите систему:
     ```bash
     apt update && apt dist-upgrade -y
     ```

3. **Отключите всплывающее окно «No valid subscription» при входе.** В консоли `ocean` -> **Shell** выполните команду, которая отредактирует скрипт интерфейса (сделав резервную копию) и перезапустит веб-сервер:
   ```bash
   sed -Ezi.bak "s/(Ext.Msg.show\(\{\s+title: gettext\('No valid sub)/void\(\{ \/\/ /g" /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js && systemctl restart pveproxy.service
   ```
   Чтобы уведомление не возвращалось после обновлений пакета `proxmox-widget-toolkit`, вынесите команду в отдельный исполняемый скрипт и добавьте его вызов в хук `apt` (это исключает синтаксические ошибки экранирования кавычек в конфигурации `apt.conf.d`):
   ```bash
   cat << 'EOF' > /usr/local/bin/pve-no-nag.sh
   #!/bin/bash
   if dpkg -V proxmox-widget-toolkit | grep -q '/proxmoxlib\.js$'; then
       sed -Ezi.bak "s/(Ext.Msg.show\(\{\s+title: gettext\('No valid sub)/void\(\{ \/\/ /g" /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js
       systemctl restart pveproxy.service
   fi
   EOF
   chmod +x /usr/local/bin/pve-no-nag.sh
   echo 'DPkg::Post-Invoke { "/usr/local/bin/pve-no-nag.sh || true"; };' > /etc/apt/apt.conf.d/no-nag-script
   ```
   *(Примечание: после выполнения обновите страницу в браузере с очисткой кэша через `Ctrl + F5`).*

4. **Загрузите ISO-образы.** В левом меню выберите `local (ocean)` -> `ISO Images` -> `Upload` и загрузите скачанные ранее образы **TrueNAS** и **Ubuntu Server**.

5. **Активируйте IOMMU в параметрах загрузчика GRUB.** В консоли `ocean` -> **Shell** отредактируйте конфигурацию GRUB:
   ```bash
   nano /etc/default/grub
   ```
   Найдите строку `GRUB_CMDLINE_LINUX_DEFAULT="quiet"` и замените её на:
   ```text
   GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt"
   ```
   Сохраните файл (`Ctrl + O`, затем `Enter`) и выйдите из редактора (`Ctrl + X`).

6. **Добавьте модули ядра VFIO в автозагрузку.** Создайте отдельный конфигурационный файл в директории `/etc/modules-load.d/`:
   ```bash
   cat <<EOF > /etc/modules-load.d/vfio.conf
   vfio
   vfio_iommu_type1
   vfio_pci
   EOF
   ```

7. **Обновите конфигурацию GRUB и перезагрузите хост.** Используйте полный путь `/usr/sbin/update-grub` (на случай, если переменные окружения `PATH` не содержат `/usr/sbin`):
   ```bash
   /usr/sbin/update-grub
   reboot
   ```

8. **Проверьте, что IOMMU и модули VFIO успешно заработали.** После перезагрузки снова откройте `Shell` и выполните проверочные команды:
   ```bash
   cat /proc/cmdline | grep intel_iommu
   lsmod | grep vfio
   ```
   В выводе первой команды должны присутствовать параметры `intel_iommu=on iommu=pt`, а во второй — загруженные модули `vfio_pci`, `vfio_iommu_type1`, `vfio` и `iommufd`.

9. **Найдите адрес вашего SATA-контроллера.** Выполните команду:
   ```bash
   lspci -nn | grep -i sata
   ```
   Вы увидите строку вида `00:17.0 SATA controller: Intel Corporation Alder Lake-S PCH SATA Controller [8086:7ae2]`. Запомните PCI-адрес в начале строки (`00:17.0`) — он понадобится на следующем этапе для проброса дисков в TrueNAS.
   *(Примечание: Так как система Proxmox установлена на NVMe SSD, который работает по шине PCIe, проброс встроенного SATA-контроллера целиком в TrueNAS абсолютно безопасен и не затронет работу гипервизора).*

---

## Этап 4. Создание и настройка виртуальной машины TrueNAS

1. **Нажмите `Create VM`** в правом верхнем углу Proxmox.
2. **Вкладка General:**
   * *Node:* `ocean`
   * *VM ID:* `100`
   * *Name:* `truenas`
   * Поставьте галочку **Start at boot** и задайте *Start/Shutdown order* = `1`.
3. **Вкладка OS:** Выберите загруженный ISO-образ TrueNAS.
4. **Вкладка System:**
   * *Machine:* **q35** (обязательно для проброса PCIe-устройств).
   * *BIOS:* **OVMF (UEFI)**.
   * Снимите галочку с `Pre-Enroll keys`.
   * *Graphic card (Display):* выберите **VirtIO-GPU** (или **Standard VGA**), чтобы избежать искажения изображения в виде черно-белой сетки в консоли noVNC при загрузке в режиме UEFI.
5. **Вкладка Disks:**
   * *Storage:* `local-lvm`, *Disk size:* `32 GiB`, *Bus/Device:* `VirtIO Block`.
   * Поставьте галочку **Discard** (поддержка TRIM для SSD).
6. **Вкладка CPU:** *Cores:* `4`, *Type:* **host**.
7. **Вкладка Memory:** *Memory:* `16384` (16 ГБ). Снимите галочку **Ballooning Device** (ZFS требует фиксированного объема памяти).
8. **Вкладка Network:** *Model:* `VirtIO (paravirtualized)`.
9. **Пробросьте SATA-контроллер.** Завершите создание ВМ (не запуская её), перейдите в `100 (truenas)` -> **Hardware**:
   * Нажмите `Add` -> **PCI Device**.
   * Выберите ваш SATA-контроллер (`00:17.0`).
   * Отметьте **All Functions** и **PCI-Express**.
   *(Примечание: если на шаге 4 не был выбран видеоадаптер `VirtIO-GPU`, измените здесь параметр **Display** с `Default` на **VirtIO-GPU**).*
10. **Установите TrueNAS и задайте статический IP-адрес:**
    * Запустите ВМ (`Start`), откройте `Console` и установите систему на виртуальный диск `VirtIO` (32 ГБ).
    * После перезагрузки в консольном меню выберите пункт `1) Configure network interfaces`:
      1. Выберите сетевой интерфейс (например, `ens18`).
      2. На вопрос об удалении настроек (`Delete interface?`) ответьте `n`.
      3. Отключите DHCP (`Configure interface for DHCP?` / `ipv4_dhcp`) — введите **`n`**.
      4. В разделе настройки адресов (`Configure IPv4?` / `Aliases`) введите **`y`** и укажите статический IP с маской подсети: `192.168.89.202/24`.
      5. На вопрос об IPv6 (`Configure IPv6?`) ответьте **`n`**.
      6. Нажмите **`a`** (*Apply* / *Save*) для применения настроек и вернитесь в главное меню (`q`).
    * В главном консольном меню выберите пункт `2) Configure static routes` и задайте шлюз по умолчанию (`Default Route`) — IP-адрес вашего роутера (`192.168.89.1`).
11. **Настройте сеть, пул и сетевую папку NFS в веб-интерфейсе TrueNAS (`http://192.168.89.202`):**
    * **Укажите DNS-серверы и шлюз (чтобы избежать ошибки `Failed to sync TRUENAS catalog`):** перейдите в **Network** -> блок **Global Configuration** -> **Settings**. Укажите в поле **Nameserver 1** адрес роутера (`192.168.89.1`) или `8.8.8.8`, убедитесь, что в **IPv4 Default Gateway** прописан `192.168.89.1`, и нажмите **Save**. Также проверьте часовой пояс (`Europe/Moscow`) и актуальное время в **System** -> **General Settings** -> **Localization**.
    * **Создайте ZFS-пул:** перейдите в **Storage** -> **Create Pool** и создайте пул (например, `tank`) из жестких дисков WD Red Plus (*Mirror* для 2 дисков или *RAIDZ1* для 3–4 дисков).
    * **Создайте датасет:** перейдите в **Datasets** и создайте датасет `media`.
    * **Настройте общий доступ NFS:** перейдите в **Shares** -> **UNIX (NFS) Shares** -> **Add**. Укажите путь `/mnt/tank/media`, а в разделе *Networks* разрешите доступ для вашей подсети `192.168.89.0/24`. Включите службу NFS при запросе.

---

## Этап 5. Создание и настройка виртуальной машины Ubuntu Server

1. **Нажмите `Create VM`** в Proxmox.
2. **Вкладка General:** *Node:* `ocean`, *VM ID:* `101`, *Name:* `ubuntu-docker`, включите **Start at boot** и задайте *Start/Shutdown order* = `2` (чтобы Ubuntu стартовала после поднятия хранилища TrueNAS).
3. **Вкладка OS:** Выберите ISO-образ **Ubuntu Server 24.04**.
4. **Вкладка System:** Оставьте по умолчанию (`i440fx`, `Default (SeaBIOS)`), включите **QEMU Agent**.
5. **Вкладка Disks:** *Storage:* `local-lvm`, *Disk size:* `64 GiB`, поставьте галочку **Discard**.
6. **Вкладка CPU:** *Cores:* `4`, *Type:* **host**.
7. **Вкладка Memory:** *Memory:* `8192` (8 ГБ).
8. **Вкладка Network:** *Model:* `VirtIO`.
9. **Установите Ubuntu.** Запустите ВМ, при установке задайте статический IP (`192.168.89.203/24`) и обязательно отметьте **Install OpenSSH server**. После перезагрузки подключитесь к серверу по SSH:
   ```bash
   ssh имя_пользователя@192.168.89.203
   ```

---

## Этап 6. Подключение папок NAS и запуск связки VPN + qBittorrent в Ubuntu

Выполните команды в терминале Ubuntu:

1. **Установите Docker и клиент NFS:**
   ```bash
   sudo apt update && sudo apt upgrade -y
   sudo apt install -y nfs-common qemu-guest-agent curl
   curl -fsSL https://get.docker.com | sh
   sudo usermod -aG docker $USER
   ```
   Перезайдите в SSH-сессию, чтобы применились права группы `docker`.

2. **Настройте автоматическое монтирование сетевой папки TrueNAS:**
   ```bash
   sudo mkdir -p /mnt/nas/media
   echo "192.168.89.202:/mnt/tank/media /mnt/nas/media nfs defaults,_netdev 0 0" | sudo tee -a /etc/fstab
   sudo mount -a
   ```
   Убедитесь командой `df -h`, что папка `/mnt/nas/media` успешно примонтирована.

3. **Подготовьте каталоги для Docker-контейнеров:**
   ```bash
   mkdir -p ~/docker/torrents/temp ~/docker/gluetun ~/docker/qbittorrent
   cd ~/docker
   ```

4. **Создайте файл `docker-compose.yml`:**
   ```bash
   nano docker-compose.yml
   ```
   Вставьте конфигурацию (весь трафик `qbittorrent` направляется строго через сетевой интерфейс контейнера `gluetun` со встроенным Kill Switch):
   ```yaml
   services:
     gluetun:
       image: qmcgaw/gluetun:latest
       container_name: gluetun
       cap_add:
         - NET_ADMIN
       devices:
         - /dev/net/tun:/dev/net/tun
       ports:
         - 8080:8080 # Веб-интерфейс qBittorrent
       environment:
         - VPN_SERVICE_PROVIDER=custom
         - VPN_TYPE=wireguard
         - WIREGUARD_PRIVATE_KEY=ВАШ_ПРИВАТНЫЙ_КЛЮЧ_WIREGUARD
         - WIREGUARD_ADDRESSES=10.0.0.2/32
         - VPN_ENDPOINT_IP=IP_ВАШЕГО_VPN_СЕРВЕРА
         - VPN_ENDPOINT_PORT=51820
         - WIREGUARD_PUBLIC_KEY=ПУБЛИЧНЫЙ_КЛЮЧ_СЕРВЕРА
         - FIREWALL_OUTBOUND_SUBNETS=192.168.89.0/24 # Доступ из локальной сети
       restart: unless-stopped

     qbittorrent:
       image: lscr.io/linuxserver/qbittorrent:latest
       container_name: qbittorrent
       network_mode: "service:gluetun"
       depends_on:
         - gluetun
       environment:
         - PUID=1000
         - PGID=1000
         - TZ=Europe/Moscow
         - WEBUI_PORT=8080
       volumes:
         - ./qbittorrent:/config
         - ./torrents/temp:/temp # Буфер активных закачек на SSD
         - /mnt/nas/media:/downloads # Готовые файлы на дисках TrueNAS
       restart: unless-stopped
   ```

5. **Запустите контейнеры и узнайте временный пароль qBittorrent:**
   ```bash
   docker compose up -d
   docker logs qbittorrent
   ```

6. **Настройте пути в веб-интерфейсе qBittorrent (`http://192.168.89.203:8080`):**
   * Откройте *Options* -> *Downloads*.
   * *Default Save Path:* `/downloads`
   * Включите *Keep incomplete torrents in:* `/temp` (чтобы случайная запись кусков во время скачивания шла на быстрый SSD, а на жесткие диски TrueNAS переносился уже готовый файл).

---

## Этап 7. Подключение ИБП CyberPower CP900EPFCLCD

1. Подключите комплектный USB-кабель от ИБП в USB-порт материнской платы.
2. В консоли Proxmox (`ocean` -> `Shell`) установите службу мониторинга ИБП:
   ```bash
   apt update && apt install -y nut
   ```
3. В файле `/etc/nut/nut.conf` задайте режим `MODE=standalone`, а в `/etc/nut/ups.conf` добавьте секцию для CyberPower:
   ```ini
   [cyberpower]
       driver = usbhid-ups
       port = auto
       desc = "CyberPower CP900EPFCLCD"
   ```
4. Перезапустите службы (`systemctl restart nut-server nut-monitor`). При длительном отключении электричества Proxmox автоматически корректно завершит работу обеих виртуальных машин (сначала Ubuntu, затем TrueNAS) и выключится сам.
