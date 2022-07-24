# Smarthome scripts
Скрипты на bach и других языках, которые я использую для автоматизации в Умном доме 

Здесь собраны скрипты, которые используются на серверах:
- [Master](https://github.com/dvit007/smarthome-bash-scripts#master-%D1%81%D0%B5%D1%80%D0%B2%D0%B5%D1%80)
- [Ubuntu16srv](https://github.com/dvit007/smarthome-bash-scripts#ubuntu16srv-%D0%B2%D0%B8%D1%80%D1%82%D1%83%D0%B0%D0%BB%D1%8C%D0%BD%D0%B0%D1%8F-%D0%BC%D0%B0%D1%88%D0%B8%D0%BD%D0%B0)
- [Raspberry Pi](https://github.com/dvit007/smarthome-bash-scripts#raspberry-pi-%D1%81%D0%B5%D1%80%D0%B2%D0%B5%D1%80)

Роутеры wifi:
 - [Tomato (OpenWrt)](https://github.com/dvit007/smarthome-bash-scripts#%D1%80%D0%BE%D1%83%D1%82%D0%B5%D1%80-tomato-openwrt)
 - [Mikrotik (Router OS 6)](https://github.com/dvit007/smarthome-bash-scripts#%D1%80%D0%BE%D1%83%D1%82%D0%B5%D1%80-tomato-openwrt)

Для каждого сервера, скрипты храняться в соответствующих папках. 
Они повторяют структуру каталогов на устройстве.

## Master сервер
На этом сервере работает OH3 и другие сервисы и службы необходимые для его работы.<br/>
Он запущен на отдельной виртаульной машине, внутри которой работе Docker.

### Скрипты для упраления виртуальными машинами QEMU
Расположение скриптов:
`etc/libvrt/hooks/`

**qemu**<br/>
Скрипт обеспечивает передачу информации в OH2 о состоняии гостевой ОС<br/>
и автоматическое ее отключение, по прошествии времени заданного в durationwork

Он вызывается автоматически службой libvirt.

Ему передаются параметры:
* `${1}` имя гостевой ОС
* `${2}` команда для этой виртуальки    

**shutdown_guest.sh**<br/>
Скрипт выключает гостевую ОС.

Он вызывается автоматически службой Cron. Задание Cron созадется в скрипте *qemu*

Ему передаются параметры:                                                        
* `${1}` имя гостевой ОС    

### Скрипты для управления UPS
Сервер подключен к ups Ippon. Скрипты автоматизируют выключение сервера при отключении электроэнерегии, и его автоматическому включению при возобновлении электроснабжения.<br/>
Перед отключением корректно завершается работа всех виртуальных машин. Что позволяет избежать потери данных из-за аварийного отключения.

Расположение настроек программы NUT для управления UPS:
`etc/nut/`

**nut.conf**,**ups.conf**,**upsd.conf**,**upsd.users**,**upsmon.conf**,**upssched.conf**

Расположение скриптов для управления UPS и сервером
`home/master/ups/`

**kvm_guest_shutdown.sh**<br/>
Скрипт реализующий корректное выключение гостевых ОС.<br/>
Он вызывается из скриптов службы NUT.

Ему передаются параметры:
* `--shutdown` отключение госетвых машин и выключение Хоста
* `--reboot` отключение гоствых машин и перезагрузка Хоста

**ups_host_start.sh**<br/>
Скрипт получает текущее состояние UPS  и отправляет его в OH2.<br/>
Вызывается из системеного скрипта *rc.local* при включении сервера.

Параметров передавать не надо.

**upssched-script.sh**<br/>
Скипт дублирует статус UPS в OH2. Меняет состояние прокси-items через Rest API

**/etc/rc.local**<br/>
Системный скрипт, который запускается при включении сервера. 
Обновлеяет статус UPS через 2 минуты, после старта.

### Скрипты для управления USB ключами
Для предоставления безопасного доступа к ключам USB, используется специальная виртуальная машина *ubuntu16srv*.<br/>
Она по команде через Telegram включается и автоматически отключается через 30 минут.

**manage_usb.sh**<br/>
Скрипт реализующий доступ к USB устройствам в зависимости от ролей пользователей.<br/>

Вызывается из сервера OpenHab. 

Ему надо передать один параметр - Роль пользовеля:
* Роль `1` - для всех неавторизванных пользователей
* Роль `2` - Бухгалтер 
* Роль `3` - Администратор ему доступны все устройства

Роли настраиваюся в правилах OpenHab        

### Скрипты для резервного копирования данных<br/>
Расположение скриптов `/vm`<br/>
Вызываются по расписанию из `cron`

**qemu_backup.sh**<br/>
Выключает виртуальные машины Qemu и делает резервные копии файла конфигурации и диска


**host_backup.sh**<br/>
Делает резервные копии системных папок сервера Master

**backup_copy.sh**<br/>
Копирует все резервные копии на второй HDD

## Роутер Mikrotik
Скрипты для роутера Mikrotik.<br/>
Версия Router OS 6

### Библиотека для скриптов
В ней собраны функции, которые облегчают разработку других скриптов.

**library-ros.rsc**<br/>
За основу взята идеия из репозитария<br/>
https://github.com/AleksovAnry/Edelweiss-ROS

Добавлены свои функции, связанные с определением подключения устройств к определенным серверам. 

Эту библиотеку надо сохранить на роутере в папке `flash/` чтобы она не удалилась после перезагрузки устройства.

Для автоматической загрузки бибилотеки надо добавить задание в System->Scheduler<br/>
`/system scheduler add comment="Load library for MikroTik Router OS" name=StartScriptLibrary \<br/>
    on-event="#The file must first be uploaded via File->Upload
    /import flash/library-ros.rsc"
    policy=ftp,reboot,read,write,policy,test,password,sniff,sensitive,romon \<br/>
    start-time=startup`

**netflixunlock.rsc**<br/>
Скрипт разблоиркует доступ к Netflix через VPN для любых устройств без установки клиентов VPN.<br/>
В том числе и для любых Smart TV

Его надо сохранить в роутере, и настроить задание в System->Scheduler<br/>
`/system scheduler add comment="Smart TV scripts" interval=1m name=NetflixUnlcok on-event="\<br/>
    /system script run NetflixUnlock_LGTV1
    /system script run NetflixUnlock_LGTV1,5"
    policy=ftp,reboot,read,write,policy,test,password,sniff,sensitive,romon`
    
## Raspberry Pi сервер
На этом сервере работает OH2

Скрипты расположены в `etc/openhab2/`

**qemu_guest_manage.sh**<br/>
Вызывается из правил OH2 для удаленного управления виртуальными машинами Qemu

**restartTelegram.sh**<br/>
Вызывается из правил OH2 для перезапкуска привязки Telegram. Этот "костыль" потребовался, так как бот периодически не получал команды от пользвателей.

**ups_state.sh**<br/>
Вызывется из правил OH2 для получения состояния UPS

## Роутер Tomato (OpenWRT)
Этот тип роутеов поддерживает bash-скрипты.

**wifi_users.sh**<br/>
Запускается на Tomato каждую минуту. Для этого его надо сохранить и добавить в scheduler роутера.<br/>
Получает mac адреса активных устройств.<br/>
Отправляет строку-массив с адресам в OH2, через Rest API

## Ubuntu16srv виртуальная машина
Эта виртуалка используется для следующих задач:
* предоставления доступа к USB ключам
* предоставление удаленного доступа в локальную сеть из интернета

Она запускается из OH2 командами из Telegram. Через 30 минут автоматически отключается.

**client-connect.sh**,**client-disconnect.sh**<br/>
Расположение скриптов `etc/openvpn`

Запускаются при подключении к серверу, и настраивают правила в iptables

**/home/dvit/iptables_ubuntu16srv.sh**<br/>
Управляет правилами iptables, вызывается из предыдущих скриптов

**server.conf**<br/>
Конфигурация моего сервера OpenVPN