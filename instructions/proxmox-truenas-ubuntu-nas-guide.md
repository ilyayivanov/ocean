# Полное руководство по развертыванию домашнего сервера на базе Proxmox VE, TrueNAS и Ubuntu Server (v16)

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
3. **Виртуальная машина №2 (Ubuntu Server 24.04 LTS):** выделяется 4 ядра CPU, 8 ГБ ОЗУ и 64 ГБ на SSD. Здесь работают Docker-контейнеры (включая `qBittorrent` для загрузки файлов и будущий DLNA-сервер) и сервисы маршрутизации.
4. **Взаимодействие между ВМ:** Ubuntu монтирует изолированные вложенные ZFS-датасеты из TrueNAS (`media/download` и `media/collection`) по протоколу NFS через внутренний виртуальный коммутатор Proxmox на скорости 10–40 Гбит/с с разделением прав доступа по числовым `UID`/`GID`.

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
11. **Настройте сеть, пул, пользователей, изолированные датасеты и сетевые папки NFS в веб-интерфейсе TrueNAS (`http://192.168.89.202`):**
    * **Укажите DNS-серверы и шлюз (чтобы избежать ошибки `Failed to sync TRUENAS catalog`):** перейдите в **Network** -> блок **Global Configuration** -> **Settings**. Укажите в поле **Nameserver 1** адрес роутера (`192.168.89.1`) или `8.8.8.8`, убедитесь, что в **IPv4 Default Gateway** прописан `192.168.89.1`, и нажмите **Save**. Также проверьте часовой пояс (`Europe/Moscow`) и актуальное время в **System** -> **General Settings** -> **Localization**.
    * **Создайте ZFS-пул:** перейдите в **Storage** -> **Create Pool** и создайте пул (например, `tank`) из жестких дисков WD Red Plus (*Mirror* для 2 дисков или *RAIDZ1* для 3–4 дисков).
    * **Создайте локальных пользователей с согласованными `UID` и `GID`:** перейдите в **Credentials** -> **Local Users** -> **Add** и создайте трех пользователей для строгого разграничения прав:
      1. **Администратор медиатеки (`media_admin`):** задайте **UID = `1000`** (основная группа с **GID = `1000`**). Совпадает с основным пользователем в Ubuntu (`UID=1000`) и используется для управления и сортировки файлов.
      2. **Загрузчик торрентов (`media_downloader`):** задайте **UID = `1001`** (основная группа `media_downloader` с **GID = `1001`**). Используется только контейнером `qBittorrent` и не имеет доступа к другим датасетам.
      3. **Читатель коллекции (`media_collection_reader`):** задайте **UID = `1002`** (основная группа `media_collection_reader` с **GID = `1002`**). Используется для DLNA / медиасервера (только чтение коллекции).
      > **Важно о правах доступа по NFS:** протокол NFS в стандартной конфигурации проверяет права доступа исключительно по числовым идентификаторам (`UID` и `GID`), а не по текстовым именам. Поэтому монтировать нужно конкретные вложенные датасеты, а не общий родительский каталог.
    * **Добавьте администратора в группу загрузчика (чтобы `media_admin` мог просматривать и переносить файлы из `download`):** перейдите в **Credentials** -> **Local Groups**, найдите группу `media_downloader` (`GID=1001`), нажмите **Members** (или значок редактирования) и добавьте в неё пользователя `media_admin` (`UID=1000`).
    * **Создайте корневой и вложенные ZFS-датасеты и настройте права доступа:**
      1. Перейдите в **Datasets**, создайте корневой датасет `media` (`/mnt/tank/media`). В его блоке **Permissions** -> **Edit** задайте владельца **User** = `media_admin` (`1000`), **Group** = `media_admin` (`1000`) и права доступа `755` (чтобы остальные пользователи могли проходить сквозь корневую директорию к своим вложенным датасетам).
      2. Выберите датасет `media`, нажмите **Add Dataset** и создайте первый вложенный датасет **`download`** (`/mnt/tank/media/download`). Откройте его блок **Permissions** -> **Edit** и задайте:
         * **User:** `media_downloader` (`1001`)
         * **Group:** `media_downloader` (`1001`)
         * **Access Mode:** `770` (полный доступ `rwx` для `qBittorrent` и администратора `media_admin`, входящего в группу `1001`; полный запрет `---` для всех остальных).
      3. Выберите датасет `media`, нажмите **Add Dataset** и создайте второй вложенный датасет **`collection`** (`/mnt/tank/media/collection`). Откройте его блок **Permissions** -> **Edit** и задайте:
         * **User:** `media_admin` (`1000`)
         * **Group:** `media_collection_reader` (`1002`)
         * **Access Mode:** `750` (полный доступ `rwx` для администратора `1000`, права только на чтение и открытие папок `r-x` для DLNA `1002`, полный запрет `---` для всех остальных).
    * **Настройте изолированные общие папки NFS:** перейдите в **Shares** -> **UNIX (NFS) Shares** -> **Add** и создайте две отдельные шары (не публикуя корневой `/mnt/tank/media`):
      1. **Шара для загрузок:** *Path* = `/mnt/tank/media/download`, *Networks* = `192.168.89.0/24`.
      2. **Шара для коллекции:** *Path* = `/mnt/tank/media/collection`, *Networks* = `192.168.89.0/24`. В расширенных настройках (*Advanced Options*) отметьте **Read Only** (дополнительная защита от записи на уровне протокола NFS).
      * Включите службу NFS при запросе.

---

## Этап 5. Настройка единого локального центра сертификации (Root CA) и доверенного HTTPS для Proxmox и TrueNAS

По умолчанию Proxmox VE и TrueNAS используют самоподписанные SSL-сертификаты (в TrueNAS 24.10+ сертификат по умолчанию выпущен на имя `localhost`, а встроенное создание CA удалено). Из-за этого браузеры выдают предупреждение `Your connection is not private`.

Чтобы убрать предупреждения навсегда для обоих серверов, создадим **единый локальный Центр Сертификации (`homeCA`)** прямо в консоли Proxmox, выпустим от его имени сертификаты с корректными полями SAN для **Proxmox** и **TrueNAS**, и добавим корневой сертификат в доверенные на ПК.

1. **Сгенерируйте корневой сертификат (Root CA) и сертификаты серверов в консоли Proxmox (`ocean` -> `Shell`):**
   ```bash
   mkdir -p /root/certs && cd /root/certs

   # 1. Создание корневого центра сертификации (Root CA) сроком на 10 лет
   openssl genrsa -out homeCA.key 2048
   openssl req -x509 -new -nodes -key homeCA.key -sha256 -days 3650 -subj "/CN=HomeLab Root CA" -out homeCA.crt

   # 2. Выпуск сертификата для Proxmox VE (ocean.home.arpa и 192.168.89.201)
   openssl genrsa -out pve.key 2048
   openssl req -new -key pve.key -subj "/CN=ocean.home.arpa" -out pve.csr
   cat <<EOF > pve.ext
   authorityKeyIdentifier=keyid,issuer
   basicConstraints=CA:FALSE
   keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
   subjectAltName = @alt_names

   [alt_names]
   DNS.1 = ocean.home.arpa
   IP.1 = 192.168.89.201
   EOF
   openssl x509 -req -in pve.csr -CA homeCA.crt -CAkey homeCA.key -CAcreateserial -out pve.crt -days 3650 -sha256 -extfile pve.ext

   # 3. Выпуск сертификата для TrueNAS (truenas.ocean.home.arpa и 192.168.89.202)
   openssl genrsa -out truenas.key 2048
   openssl req -new -key truenas.key -subj "/CN=truenas.ocean.home.arpa" -out truenas.csr
   cat <<EOF > truenas.ext
   authorityKeyIdentifier=keyid,issuer
   basicConstraints=CA:FALSE
   keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
   subjectAltName = @alt_names

   [alt_names]
   DNS.1 = truenas.ocean.home.arpa
   IP.1 = 192.168.89.202
   EOF
   openssl x509 -req -in truenas.csr -CA homeCA.crt -CAkey homeCA.key -CAcreateserial -out truenas.crt -days 3650 -sha256 -extfile truenas.ext
   ```
   *(Сохраните папку `/root/certs` — если в будущем развернёте новые локальные сервисы, вы сможете подписать для них сертификаты этим же ключом `homeCA.key`, не добавляя новых корневых сертификатов на компьютеры).*

2. **Установите новый SSL-сертификат в Proxmox VE:**
   В консоли `ocean` -> `Shell` скопируйте сертификат и ключ в рабочую директорию узла (перезаписав старые файлы `pveproxy-ssl.*`) и перезапустите веб-интерфейс:
   ```bash
   cp /root/certs/pve.crt /etc/pve/nodes/ocean/pveproxy-ssl.pem
   cp /root/certs/pve.key /etc/pve/nodes/ocean/pveproxy-ssl.key
   systemctl restart pveproxy.service
   ```
   *(Примечание: В отличие от `pve-ssl.pem`, файлы `pveproxy-ssl.pem` и `pveproxy-ssl.key` предназначены именно для пользовательских сертификатов и не перезаписываются при обновлении кластера или самого Proxmox).*

3. **Импортируйте и активируйте SSL-сертификат в TrueNAS:**
   * Выведите содержимое сертификата и приватного ключа TrueNAS в консоли Proxmox:
     ```bash
     cat /root/certs/truenas.crt
     cat /root/certs/truenas.key
     ```
   * Откройте веб-интерфейс TrueNAS -> **Credentials** -> **Certificates** -> в блоке **Certificates** нажмите **Add**.
   * Задайте **Name:** `truenas-local`, выберите **Type:** `Import Certificate` -> **Next**.
   * Вставьте содержимое `truenas.crt` в поле **Certificate**, а содержимое `truenas.key` в поле **Private Key** (включая строки `-----BEGIN ...-----` и `-----END ...-----`). Нажмите **Next** -> **Save**.
   * Перейдите в **System** -> **General Settings** -> блок **GUI** -> **Settings**, выберите в поле **GUI SSL Certificate** новый сертификат `truenas-local` и нажмите **Save**.

4. **Установите корневой сертификат `homeCA.crt` в доверенные на ПК (Windows):**
   * Выведите содержимое `homeCA.crt` в консоли Proxmox (`cat /root/certs/homeCA.crt`), скопируйте его и сохраните на рабочем столе ПК в файл `homeCA.crt`.
   * Дважды кликните по файлу `homeCA.crt` -> нажмите **Установить сертификат...** (*Install Certificate*).
   * Выберите **Локальный компьютер** (*Local Machine*) -> **Далее**.
   * Выберите **Поместить все сертификаты в следующее хранилище** -> **Обзор** -> **Доверенные корневые центры сертификации** (*Trusted Root Certification Authorities*) -> **ОК** -> **Готово**.
   * Полностью перезапустите браузер. Теперь адреса `https://ocean.home.arpa:8006`, `https://192.168.89.201:8006`, `https://truenas.ocean.home.arpa` и `https://192.168.89.202` будут открываться с доверенным HTTPS-соединением без предупреждений.

---

## Этап 6. Создание и настройка виртуальной машины Ubuntu Server

1. **Нажмите `Create VM`** в правом верхнем углу Proxmox.
2. **Вкладка General:** *Node:* `ocean`, *VM ID:* `101`, *Name:* `ubuntu-docker`, включите **Start at boot** и задайте *Start/Shutdown order* = `2` (чтобы Ubuntu стартовала после поднятия хранилища TrueNAS).
3. **Вкладка OS:** Выберите ISO-образ **Ubuntu Server 24.04**.
4. **Вкладка System:** Оставьте по умолчанию (`i440fx`, `Default (SeaBIOS)`), включите **QEMU Agent**.
5. **Вкладка Disks:** *Storage:* `local-lvm`, *Disk size:* `64 GiB`, поставьте галочку **Discard**.
6. **Вкладка CPU:** *Cores:* `4`, *Type:* **host**.
7. **Вкладка Memory:** *Memory:* `8192` (8 ГБ).
8. **Вкладка Network:** *Model:* `VirtIO`.
9. **Установите Ubuntu.** Запустите ВМ, при установке задайте статический IP (`192.168.89.203/24`) и обязательно отметьте **Install OpenSSH server**. Первый созданный при установке пользователь автоматически получит `UID=1000` и `GID=1000` (проверить можно командой `id` после входа). После перезагрузки подключитесь к серверу по SSH:
   ```bash
   ssh имя_пользователя@192.168.89.203
   ```

10. **Настройте безопасность SSH (вход строго по ключу) и локального `root`:**
    * **Скопируйте свой публичный SSH-ключ с ПК на сервер** (если ключа еще нет, создайте его на ПК командой `ssh-keygen -t ed25519`):
      * *Linux / macOS:*
        ```bash
        ssh-copy-id имя_пользователя@192.168.89.203
        ```
      * *Windows (PowerShell):*
        ```powershell
        type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh имя_пользователя@192.168.89.203 "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
        ```
    * **Убедитесь, что вход по SSH-ключу работает** без запроса пароля (`ssh имя_пользователя@192.168.89.203`).
    * **Активируйте учетную запись `root` для локального доступа** (через консоль Proxmox или `su -`), задав ей пароль:
      ```bash
      sudo passwd root
      ```
    * **Отключите вход по паролю и запретите вход `root` по SSH:**
      ```bash
      sudo tee /etc/ssh/sshd_config.d/00-ssh-hardening.conf > /dev/null << 'EOF'
      PasswordAuthentication no
      KbdInteractiveAuthentication no
      ChallengeResponseAuthentication no
      PubkeyAuthentication yes
      PermitRootLogin no
      EOF
      ```
    * **Проверьте конфигурацию и перезапустите службу SSH:**
      ```bash
      sudo sshd -t && sudo systemctl restart ssh
      ```
      *(Примечание: после перезапуска службы подключение без SSH-ключа и любая попытка входа под `root` по сети будут отклоняться, при этом локальный вход под `root` через консоль Proxmox `101 (ubuntu-docker)` -> `Console` останется доступным).*

---

## Этап 7. Подключение изолированных папок NAS и запуск qBittorrent в Ubuntu

Выполните команды в терминале Ubuntu:

1. **Установите Docker и клиент NFS:**
   ```bash
   sudo apt update && sudo apt upgrade -y
   sudo apt install -y nfs-common qemu-guest-agent curl
   curl -fsSL https://get.docker.com | sh
   sudo usermod -aG docker $USER
   ```

2. **Синхронизируйте группы доступа в Ubuntu с TrueNAS (чтобы ваш пользователь `UID=1000` мог открывать папку `download` по SSH):**
   ```bash
   sudo groupadd -g 1001 media_downloader
   sudo groupadd -g 1002 media_collection_reader
   sudo usermod -aG media_downloader $USER
   ```
   Перезайдите в SSH-сессию, чтобы применились права групп `docker` и `media_downloader` (проверить можно командой `id`).

3. **Настройте автоматическое монтирование изолированных датасетов TrueNAS по требованию (`automount`):**
   Создайте локальные точки монтирования и добавьте оба вложенных датасета в `/etc/fstab` (корневой `/mnt/nas/media` не монтируется):
   ```bash
   sudo mkdir -p /mnt/nas/media/download /mnt/nas/media/collection
   cat << 'EOF' | sudo tee -a /etc/fstab
   192.168.89.202:/mnt/tank/media/download /mnt/nas/media/download nfs defaults,_netdev,x-systemd.automount,x-systemd.idle-timeout=60 0 0
   192.168.89.202:/mnt/tank/media/collection /mnt/nas/media/collection nfs ro,defaults,_netdev,x-systemd.automount,x-systemd.idle-timeout=60 0 0
   EOF
   sudo systemctl daemon-reload
   sudo systemctl restart mnt-nas-media-download.automount mnt-nas-media-collection.automount
   ```
   **Проверьте работу авто-монтирования:** обратитесь к обеим директориям и сразу проверьте список смонтированных разделов:
   ```bash
   ls -la /mnt/nas/media/download /mnt/nas/media/collection
   df -h
   ```
   *(Примечание 1: опция `x-systemd.automount` не держит сетевые папки подключенными в простое — система автоматически монтирует их в момент обращения и отключает через 60 секунд бездействия. Поэтому в спокойном состоянии `df -h` их не показывает — это нормальное поведение).*
   *(Примечание 2: флаг `ro` во второй строке дополнительно монтирует `collection` в режиме «только чтение» на стороне Ubuntu).*

4. **Подготовьте локальные каталоги на SSD для Docker-контейнеров и назначьте права для `UID=1001`:**
   ```bash
   mkdir -p ~/docker/torrent/temp ~/docker/qbittorrent
   sudo chown -R 1001:1001 ~/docker/qbittorrent ~/docker/torrent/temp
   cd ~/docker
   ```

5. **Создайте файл `docker-compose.yml`:**
   ```bash
   nano docker-compose.yml
   ```
   Вставьте конфигурацию для запуска `qBittorrent` от имени изолированного пользователя `media_downloader` (`PUID=1001`, `PGID=1001`) с доступом исключительно к `/mnt/nas/media/download` (а также заготовку для будущего DLNA-сервера под `PUID=1002` с доступом только на чтение к `collection`). Для корректной работы с динамическим монтированием `x-systemd.automount` сетевые директории подключаются через `bind` с распространением монтирования `propagation: rshared`:
   ```yaml
   services:
     qbittorrent:
       image: lscr.io/linuxserver/qbittorrent:latest
       container_name: qbittorrent
       ports:
         - 8080:8080 # Веб-интерфейс qBittorrent
         - 6881:6881
         - 6881:6881/udp
       environment:
         - PUID=1001
         - PGID=1001
         - TZ=Europe/Moscow
         - WEBUI_PORT=8080
         - TORRENTING_PORT=6881
       volumes:
         - ./qbittorrent:/config
         - ./torrent/temp:/temp # Буфер активных закачек на SSD
         - type: bind
           source: /mnt/nas/media/download
           target: /download # Готовые файлы в датасете download (UID 1001)
           bind:
             propagation: rshared
       restart: unless-stopped

     # Заготовка для будущего DLNA-сервера (только чтение коллекции под UID 1002):
     # dlna:
     #   image: vladgh/minidlna:latest
     #   container_name: minidlna
     #   network_mode: host
     #   environment:
     #     - PUID=1002
     #     - PGID=1002
     #     - TZ=Europe/Moscow
     #   volumes:
     #     - type: bind
     #       source: /mnt/nas/media/collection
     #       target: /media
     #       read_only: true # Доступ только к collection (Read-Only)
     #       bind:
     #         propagation: rshared
     #   restart: unless-stopped
   ```

6. **Запустите контейнер, проверьте права доступа и узнайте временный пароль qBittorrent:**
   ```bash
   docker compose up -d
   docker exec -u 1001:1001 qbittorrent ls -la /download
   docker logs qbittorrent
   ```
   *(Примечание: проверять каталог `/download` внутри контейнера необходимо строго с флагом `-u 1001:1001`. По умолчанию `docker exec` запускается от имени `root` (`UID=0`), который из-за защиты NFS `Root Squash` преобразуется на сервере TrueNAS в анонимного пользователя `nobody` и получает ожидаемый отказ `Permission denied`, так как датасет защищен правами `770`).*

7. **Настройте пути в веб-интерфейсе qBittorrent (`http://192.168.89.203:8080`):**
   * Откройте *Options* -> *Downloads*.
   * *Default Save Path:* `/download`
   * Включите *Keep incomplete torrent in:* `/temp` (чтобы случайная запись кусков во время скачивания шла на быстрый SSD, а на жесткие диски TrueNAS переносился уже готовый файл).

---

## Этап 8. Подключение ИБП CyberPower CP900EPFCLCD в режиме Master / Slave и автозапуск ВМ при возврате питания

В данной схеме хост **Proxmox VE** выступает главным сервером (**Master / Netserver**), к которому по USB подключен ИБП, а виртуальные машины **TrueNAS** и **Ubuntu** работают как сетевые клиенты (**Slave**). 

При отключении электричества каждая система корректно завершает работу по собственному таймеру в строгом порядке: сначала останавливается **Ubuntu** (через 2 минуты), затем **TrueNAS** сбрасывает кэш ZFS и размонтирует диски (через 4 минуты), и последним выключается сам **Proxmox** (при снижении заряда батареи до 30%). Если электричество вернулось раньше, чем выключился Proxmox, гипервизор автоматически запустит остановленные виртуальные машины по событию возврата питания (`ONLINE`).

### Шаг 1. Настройка NUT-сервера (Master) и автозапуска ВМ на хосте Proxmox VE

1. **Подключите комплектный USB-кабель** от ИБП CyberPower CP900EPFCLCD в свободный USB-порт материнской платы и убедитесь, что система видит устройство:
   ```bash
   lsusb | grep -i cyber
   ```
   Вы увидите строку вида `Bus 001 Device 003: ID 0764:0501 Cyber Power System, Inc. CP1500 AVR UPS`.

2. **Установите пакеты NUT и `sudo` в консоли Proxmox (`ocean` -> `Shell`):**
   ```bash
   apt update && apt install -y nut nut-client nut-server sudo
   ```

3. **Включите режим сетевого сервера в `/etc/nut/nut.conf`:**
   ```bash
   echo "MODE=netserver" > /etc/nut/nut.conf
   ```

4. **Добавьте описание ИБП в `/etc/nut/ups.conf`:**
   ```bash
   cat << 'EOF' > /etc/nut/ups.conf
   pollinterval = 15

   [cyberpower]
       driver = usbhid-ups
       port = auto
       desc = "CyberPower CP900EPFCLCD"
       override.battery.charge.low = 30
       override.battery.runtime.low = 300
   EOF
   ```
   *(Параметр `pollinterval = 15` предотвращает спам в системный журнал о кратковременных задержках опроса USB у моделей CyberPower, а `override.battery.charge.low = 30` задает порог в 30% заряда для принудительного выключения самого гипервизора).*

5. **Разрешите сетевые подключения к серверу NUT в `/etc/nut/upsd.conf`:**
   ```bash
   cat << 'EOF' > /etc/nut/upsd.conf
   LISTEN 127.0.0.1 3493
   LISTEN 192.168.89.201 3493
   EOF
   ```

6. **Создайте учетные записи для локального и сетевого мониторинга в `/etc/nut/upsd.users`:**
   Замените `ПАРОЛЬ_ADMIN` и `ПАРОЛЬ_SLAVE` на надежные пароли без спецсимволов `#` и кавычек:
   ```bash
   cat << 'EOF' > /etc/nut/upsd.users
   [upsadmin]
       password = ПАРОЛЬ_ADMIN
       actions = SET
       actions = FSD
       instcmds = ALL
       upsmon primary

   [upsslave]
       password = ПАРОЛЬ_SLAVE
       upsmon secondary
   EOF
   ```

7. **Настройте локальный монитор `upsmon` в `/etc/nut/upsmon.conf` с отслеживанием возврата питания (`ONLINE`):**
   ```bash
   cat << 'EOF' > /etc/nut/upsmon.conf
   MONITOR cyberpower@localhost 1 upsadmin ПАРОЛЬ_ADMIN primary
   MINSUPPLIES 1
   SHUTDOWNCMD "/sbin/shutdown -h +0"
   NOTIFYCMD /usr/sbin/upssched
   POLLFREQ 5
   POLLFREQALERT 5
   HOSTSYNC 15
   DEADTIME 25
   POWERDOWNFLAG /etc/killpower
   RBWARNTIME 43200
   NOCOMMWARNTIME 300
   FINALDELAY 5
   NOTIFYFLAG ONLINE SYSLOG+WALL+EXEC
   EOF
   ```

8. **Настройте планировщик `upssched` и постоянную директорию `/var/run/nut`:**
   ```bash
   echo "d /var/run/nut 0755 nut nut -" > /etc/tmpfiles.d/nut-run.conf
   systemd-tmpfiles --create

   cat << 'EOF' > /etc/nut/upssched.conf
   CMDSCRIPT /usr/local/bin/upssched-cmd
   PIPEFN /var/run/nut/upssched.pipe
   LOCKFN /var/run/nut/upssched.lock

   AT ONLINE * EXECUTE power_restored
   EOF
   ```

9. **Разрешите пользователю `nut` запускать утилиту `qm` через `sudo` без пароля:**
   ```bash
   cat << 'EOF' > /etc/sudoers.d/nut-qm
   nut ALL=(root) NOPASSWD: /usr/sbin/qm
   EOF
   chmod 0440 /etc/sudoers.d/nut-qm
   visudo -cf /etc/sudoers.d/nut-qm
   ```
   *(Убедитесь, что команда `visudo` вернула `/etc/sudoers.d/nut-qm: parsed OK`).*

10. **Создайте скрипт `/usr/local/bin/upssched-cmd` для автоматического запуска ВМ при восстановлении питания:**
    ```bash
    cat << 'EOF' > /usr/local/bin/upssched-cmd
    #!/bin/bash
    case $1 in
        power_restored)
            logger -t upssched-cmd "Питание от сети восстановлено. Проверяем и запускаем ВМ..."
            sleep 10
            
            if sudo /usr/sbin/qm status 100 | grep -q "stopped"; then
                logger -t upssched-cmd "Запуск TrueNAS (VM 100)..."
                /usr/bin/script -q -c "sudo /usr/sbin/qm start 100" /dev/null 2>&1 | logger -t upssched-cmd
            else
                logger -t upssched-cmd "VM 100 не в статусе stopped (текущий: $(sudo /usr/sbin/qm status 100))"
            fi
            
            sleep 30
            
            if sudo /usr/sbin/qm status 101 | grep -q "stopped"; then
                logger -t upssched-cmd "Запуск Ubuntu (VM 101)..."
                /usr/bin/script -q -c "sudo /usr/sbin/qm start 101" /dev/null 2>&1 | logger -t upssched-cmd
            else
                logger -t upssched-cmd "VM 101 не в статусе stopped (текущий: $(sudo /usr/sbin/qm status 101))"
            fi
            ;;
    esac
    EOF
    chmod +x /usr/local/bin/upssched-cmd
    ```
    *(Примечание: обертка `/usr/bin/script -q -c "..." /dev/null` создает виртуальный псевдотерминал (PTY), который требуется команде `qm start` при запуске из фонового демона, чтобы избежать ошибки `failed to tcsetpgrp: Inappropriate ioctl for device`).*

11. **Ограничьте права доступа к конфигурационным файлам с паролями и перезапустите службы:**
    ```bash
    chown root:nut /etc/nut/*
    chmod 640 /etc/nut/*
    systemctl restart nut-driver.target nut-server nut-monitor
    systemctl enable nut-server nut-monitor
    ```

12. **Проверьте работу ИБП и тестовый запуск скрипта:**
    ```bash
    upsc cyberpower@localhost
    ```
    В выводе должны отобразиться параметры батареи (`battery.charge`, `battery.runtime`, `ups.status: OL`). Проверить скрипт автозапуска ВМ можно командой (в условиях отсутствия терминала, аналогично демону):
    ```bash
    su -s /bin/bash nut -c "/usr/local/bin/upssched-cmd power_restored" < /dev/null
    journalctl -t upssched-cmd -n 10 --no-pager
    ```

---

### Шаг 2. Настройка клиента (Slave) в TrueNAS

1. **Проверьте сетевую доступность NUT-сервера из консоли TrueNAS.** Откройте в веб-интерфейсе TrueNAS **System** -> **Shell** и выполните:
   ```bash
   upsc cyberpower@192.168.89.201
   ```
   Если вывод возвращает данные о состоянии ИБП, сетевой доступ работает корректно.

2. **Настройте службу UPS в веб-интерфейсе TrueNAS:**
   * Перейдите в **System** -> **Services**, найдите службу **UPS** и нажмите на иконку карандаша (**Edit**).
   * **UPS Mode:** выберите `Slave`.
   * **Identifier:** введите `cyberpower` (имя должно строго совпадать с именем секции в `[cyberpower]` на Proxmox).
   * **Remote Host:** укажите IP-адрес Proxmox `192.168.89.201`.
   * **Remote Port:** `3493`.
   * **Monitor User:** введите `upsslave`.
   * **Monitor Password:** укажите ваш `ПАРОЛЬ_SLAVE`.
   * **Shutdown Mode:** выберите `UPS goes on battery` (выключение по таймеру после перехода на батарею).
   * **Shutdown Timer:** укажите `240` (система начнет корректное завершение работы через 4 минуты после отключения электричества, если питание не восстановится).
   * **Shutdown Command:** оставьте по умолчанию (`/sbin/shutdown -P now` или `midclt call system.shutdown`).
   * **Power Off UPS:** снимите галочку (отключать питание розеток ИБП должен только Master-хост Proxmox в самом конце).
   * Нажмите **Save**.

3. **Запустите службу UPS:**
   * В списке **System** -> **Services** переведите переключатель напротив **UPS** в положение **Running** и поставьте галочку **Start Automatically**.
   * Теперь в разделе **Reporting** -> **UPS** начнет собираться статистика заряда батареи, нагрузки и напряжения.

---

### Шаг 3. Настройка клиента (Slave) в Ubuntu Server

Чтобы виртуальная машина с Docker-контейнерами и сетевыми дисками NFS выключалась самой первой (через 2 минуты после пропажи сети), установите клиент NUT в Ubuntu:

1. **Подключитесь к Ubuntu по SSH и установите `nut-client`:**
   ```bash
   sudo apt update && sudo apt install -y nut-client
   ```

2. **Задайте режим клиента в `/etc/nut/nut.conf`:**
   ```bash
   echo "MODE=netclient" | sudo tee /etc/nut/nut.conf
   ```

3. **Настройте подключение к Proxmox и таймер выключения в `/etc/nut/upsmon.conf`:**
   ```bash
   sudo tee /etc/nut/upsmon.conf > /dev/null << 'EOF'
   MONITOR cyberpower@192.168.89.201 1 upsslave ПАРОЛЬ_SLAVE secondary
   MINSUPPLIES 1
   SHUTDOWNCMD "/sbin/shutdown -h +0"
   NOTIFYCMD /usr/sbin/upssched
   POLLFREQ 5
   POLLFREQALERT 5
   HOSTSYNC 15
   DEADTIME 25
   NOTIFYFLAG ONBATT SYSLOG+WALL+EXEC
   NOTIFYFLAG ONLINE SYSLOG+WALL+EXEC
   EOF
   ```

4. **Настройте таймер отключения через 120 секунд работы от батареи в `/etc/nut/upssched.conf`:**
   ```bash
   echo "d /var/run/nut 0755 nut nut -" | sudo tee /etc/tmpfiles.d/nut-run.conf
   sudo systemd-tmpfiles --create

   sudo tee /etc/nut/upssched.conf > /dev/null << 'EOF'
   CMDSCRIPT /bin/upssched-cmd
   PIPEFN /var/run/nut/upssched.pipe
   LOCKFN /var/run/nut/upssched.lock
   AT ONBATT * START-TIMER earlyshutdown 120
   AT ONLINE * CANCEL-TIMER earlyshutdown
   EOF
   ```

5. **Создайте скрипт выполнения команды выключения `/bin/upssched-cmd`:**
   ```bash
   sudo tee /bin/upssched-cmd > /dev/null << 'EOF'
   #!/bin/bash
   case $1 in
       earlyshutdown)
           logger -t upssched-cmd "Отключение питания более 2 минут — выключаем Ubuntu Server"
           /sbin/upsmon -c fsd
           ;;
   esac
   EOF
   sudo chmod +x /bin/upssched-cmd
   ```

6. **Перезапустите службу мониторинга в Ubuntu:**
   ```bash
   sudo chown root:nut /etc/nut/* && sudo chmod 640 /etc/nut/*
   sudo systemctl restart nut-monitor
   sudo systemctl enable nut-monitor
   ```

---

### Итоговый порядок действий при отключении и возврате электричества

1. **0:00** — Пропадает электричество, ИБП переходит на батарею (`OB`).
2. **2:00** — Ubuntu Server корректно останавливает Docker-контейнеры, отмонтирует папки TrueNAS по NFS и выключается.
3. **4:00** — TrueNAS останавливает службы, сбрасывает кэш ZFS на диски, экспортирует пул `tank` и выключается.
4. **Сценарий А (электричество вернулось до разряда ИБП до 30%):** Proxmox VE фиксирует переход на сеть (`ONLINE`), выдерживает паузу 10 секунд, запускает TrueNAS (`VM 100`), ждет 30 секунд и запускает Ubuntu (`VM 101`). Ubuntu автоматически монтирует NFS-датасеты при обращении Docker-контейнеров.
5. **Сценарий Б (длительное отключение, заряд ИБП < 30% или остаток < 5 минут):** Proxmox VE отправляет команду на полное завершение работы, выключается сам и отключает подачу питания на розетки ИБП. После восстановления электропитания BIOS материнской платы (`AC BACK: Always On`) автоматически включает сервер, Proxmox загружается и запускает ВМ согласно порядку `Start/Shutdown order` (`1` -> `2`).
