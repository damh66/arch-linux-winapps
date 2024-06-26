# Установка Winapps на Arch Linux

**[Официальный репозиторий WinApps](https://github.com/Fmstrat/winapps)**

***Полностью исправная работа WinApps возможна только на Linux с GNOME и KDE***

### 1. Настройка виртуальной машины Windows

Создаем и настраиваем виртуальную машину по инструкции с [официального репозитория WinApps](https://github.com/Fmstrat/winapps/blob/main/docs/KVM.md)

### 2. Клонирование репозитория и установка необходимых пакетов

Клонируем официальный репозиторий:
```
git clone https://github.com/Fmstrat/winapps.git
```

Устанавливаем пакеты:
```
sudo pacman -S freerdp qemu virt-viewer dnsmasq vde2 bridge-utils openbsd-netcat libguestfs ebtables iptables bc apparmor libgovirt libvirt
```

### 3. Правка конфигурационных файлов

Создаем файл конфигурации **WinApps** по пути **`~/.config/winapps/winapps.conf`**:
```
RDP_USER="MyWindowsUser"
RDP_PASS="MyWindowsPassword"
#RDP_DOMAIN="MYDOMAIN"
#RDP_IP="192.168.123.111"
#RDP_SCALE=100
#RDP_FLAGS=""
#MULTIMON="true"
#DEBUG="true"
```
> * `RDP_USER` и `RDP_PASS` - имя пользователя и пароль должны представлять собой полную учетную запись и пароль пользователя
>
> * `RDP_DOMAIN` - для пользователей в домене вы можете раскомментировать и изменить
>
> * `RDP_IP` - если вы используете виртуальную машину с включенным NAT, оставьте  закомментированным, и WinApps автоматически определит правильный локальный IP-адрес
>
> * `RDP_SCALE` - установка желаемого масштаба [100|140|160|180] на дисплеях с высоким разрешением (UHD) 
>
> * `RDP_FLAGS` - добавление флагов к FreeRDP
>   > Например, `/audio-mode:1` для передачи микрофона
>
> * `MULTIMON` - использование WinApps с несколькими мониторами, однако, если вы увидите черный экран (ошибка FreeRDP), следует отключить это
>
> * `DEBUG` - журнал будет создаваться при каждом запуске приложения в `~/.local/share/winapps/winapps.log`

В конфигурационном файле `/etc/libvirt/libvirtd.conf` приводим эти строки к данному значению:
```
#unix_sock_group = "libvirt"
#unix_sock_rw_perms = "0770"
```

В файле `~/.config/libvirt/libvirt.conf` прописываем:
```
uri_default = "qemu:///system"
```

Переходим в раннее клонированный репозиторий и прописываем:
```
cd winapps
virsh define kvm/RDPWindows.xml
virsh start RDPWindows
virsh autostart RDPWindows
```
> `virsh define` - определение файла конфигурации XML для виртуальной машины
>
> `virsh start` - запуск виртуальной машины
>
> `virsh autostart` - автозапуск виртуальной машины при загрузке хоста KVM

Проверяем список виртуальных интерфейсов:
```
virsh net-list --all
```

Включаем виртуальный интерфейс и добавляем его в автозапуск:
```
virsh net-start default
virsh net-autostart --network default
```

Настройка KVM для возможности запуска от имени обычного пользователя:
```
sudo sed -i "s/#user = "root"/user = "$(id -un)"/g" /etc/libvirt/qemu.conf
sudo sed -i "s/#group = "root"/group = "$(id -gn)"/g" /etc/libvirt/qemu.conf
sudo usermod -aG kvm $(id -un)
sudo usermod -aG libvirt $(id -un)
sudo systemctl restart libvirtd
sudo ln -s /etc/apparmor.d/usr.sbin.libvirtd /etc/apparmor.d/disable/
```
> Если группы **libvirt** не существует, то создаем ее:
> ```
> newgrp libvirt
> ```

Перезагружаем ПК, чтобы изменения вступили в силу
```
systemctl reboot
```

### 4. Запуск установщика WinApps

Переходим в папку клонированного репозитория:
```
cd winapps
```

Проверяем способность подключения **FreeRDP** к виртуальной машине:
```
bin/winapps check
```

Среди выходных данных нужно будет **принять** предложенный сертификат безопасности. После этого должно появиться окно **Проводника Windows**. Вы можете закрыть это окно и нажать `Ctrl-C`, чтобы выйти из FreeRDP

Если появились ошибка, попробуйте перезапустить виртуальную машину. Если перезапуск ВМ не помог, причины могут быть в этом:
* Не принят **сертификат безопасности** при первом подключении
* Не включен **RDP**(удаленный рабочий стол) в виртуальной машине
* Указан **неверный IP-адрес** в `~/.config/winapps/winapps.conf`
  > если на виртуальной машине задан статический **IP-адрес**
* Неправильные **учетные данные пользователя** в `~/.config/winapps/winapps.conf`
* Не объединены **данные реестра** `winapps/install/RDPApps.reg` в виртуальной машине

Если ошибки не возникают, запускаем **установщик**:
```
./installer.sh
```

После сканирования всех приложений установщиком на виртуальной машине, выбираем:
* Установка только для **текущего пользователя** или для **всей системы**
* Установка предварительно сконфигурированных программ: **всех найденных**, **только выбранных** или **ничего не устанавливать**
* Установка других найденных программ: **всех найденных**, **только выбранных** или **ничего не устанавливать**

<img src="https://raw.githubusercontent.com/Fmstrat/winapps/main/demo/installer.gif">

После этого установка будет завершена. Установщик можно запускать несколько раз, при последующем запуске установщик удалит все текущие приложения и обновит приложения.

#### Необязательные аргументы установщика

```
./installer.sh --user                # Установка приложений для текущего пользователя
./installer.sh --system              # Установка приложений для всей системы
./installer.sh --user --uninstall    # Удаление приложений для текущего пользователя
./installer.sh --system --uninstall  # Удаление приложений для всей системы
```
