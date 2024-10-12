### Systemd. Система инициализация в Linux.

#### <a name='toc'>Содержание</a>

1. [Система инициализации](#initialization_system)
2. [Управление сервисами при использовании systemd](#managing_services)
3. [Systemd — создание unit-файла](#create_unit_file) 
4. [Дополнительные источники](#recommended_sources)

#### Компоненты Systemd

![Systemd_components](https://github.com/user-attachments/assets/07e37ebf-f448-40dc-810f-d9323cb45010)


#### 1. [[⬆]](#toc) <a name='initialization_system'>Система инициализации</a>

После своей загрузки ядро передает управление системе инициализации, Цель этой системы - выполнить дальнейшую инициализацию системы, 
> _**Самая главная задача системы инициалиэации - запуск и управление системными службами. Служба (сервис, демон) - специальная программа, выполняющаяся в фоновом режиме и предоставляющая определенные услуги (или, как говорят, сервис - отсюда и второе название).**_

**Systemd** - подсистема инициализации и управления службами в Linux. Основная особенность - интенсивное **распараллеливание** запуска служб в процессе загрузки системы, что позволяет существенно ускорить запуск операционной системы.

##### Принцип работы
Система инициализации systemd используется во многих современных дистрибутивах, и на данный момент это самая быстрая система инициализации. Например, upstart - запускает службы параллельно. Но параллельный запуск - не всегда хорошо. Нужно учитывать зависимости служб. Например, сервис d-bus нужен многим другим сервисам. Пока сервис d-bus не будет запушен, нельзя запускать сервисы, которые от него зависят. И если сервис d-bus (или любой другой, от которого зависят какие-то другие сервисы) запускается долго, то все остальные службы будут ждать его загрузки, что скажется на загрузки системы в целом.

В systemd все по другому. При своем запуске службы проверяют, запущена ли необходимая им служба, по наличию файла сокета. Например, в случае с d-bus это файл `/var/run/dbus/system_bus_socket`. Если создать сокеты для всех служб, то можно запускать их параллельно, особо не беспокоясь, что произойдет сбой какой-то службы при запуске из-за отсутствия службы, от которой они зависят. Даже если несколько служб, которым нужен сервис d-bus, запустятся раньше, чем сам сервис d-bus, ничего страшного. Каждая из этих служб отправит в сокет (главное, что он уже открыт!) сообщение, которое обработает сервис d-bus после того, как он запустится. Вот, собственно, и все.

##### Конфигурационные файлы systemd
Systemd запускает сервисы описанные в его конфигурации. Конфигурация состоит из множества файлов, которые называют юнитами или модулями. Все эти юниты/модули разложены в трех каталогах:

| Путь | Описание |
| ------- | ----------- |
| `/etc/systemd/system` | юниты/модули, которые создаёт администратор системы. Обладают самым высоким приоритетом.|
| `/run/systemd/system` | юниты/модули, которые создаются в процессе работы системы. Приоритет этого каталога ниже, чем каталога /etc/systemd/system/, но выше, чем у /usr/lib/systemd/system |
| `/lib/systemd/system` | директория `/lib` ссылается на `/usr/lib` - `lib -> usr/lib` |
| `/usr/lib/systemd/system` | юниты/модули сервисов, установленных с помощью менеджера пакетов. Самый простой пример — веб-серверы: Apache или Nginx. |

Типичный файл модуля типа service  

![image](https://github.com/user-attachments/assets/0a56b4d5-3834-4be0-8e08-328dddbe27b5)

| Путь | Описание |
| ------- | ----------- |
| `[Unit]` | В секции `Unit` содержится описание модуля и его связи с другими модулями. Эта секция есть и в друrих модулях, а не только в сервисах. |
| `[Service]` | Секция `Service`содержит информацию о сервисе. Параметр `ExecStart` описывает команду, которую нужно запустить. Параметр `Туре` указывает, как сервис будет уведомлять systemd об окончании запуска. |
| `[Install]` | Секция Install содержит информацию о цели, в которой должен запускаться сервис. |

_**Раздел [Unit]**_ - содержит описание модуля и его связи с другими модулями.
* `Description` - имя и основные функции модуля.
* `Documentation` - указывает местоположение документации.
* `Requires` - перечисление модулей, от которых зависит это модуль (служба или устройство)
* `Wants` - директива похожа на Requires, но менее строгая, потому что не требует обязательного запуска зависимостей для работы модуля.
* `Before` - модули, перечисленные в этой директиве, не будут запущенны до тех пор, пока текущий модуль не будет отмечен как запущенный.
* `After` - модули, перечисленные в этой директиве, будут запущенны до запустка текущего модуля.
* `Conflicts` - перечень модулей, которые нельзя запускать одновременно с текущим.

_**Раздел [Service]**_ - содержит информацию о службе и о процессе, которым она управляет.
* `Type` - тип запуска модуля. Может принимать значения simple, forking, oneshot, dbus, notify или idle.
* `ExecStart` - команда со всеми аргументами, которая выполняется при запуске сервиса.
* `ExecStartPre, ExecStartPost` - дополнительные команды, которые выполняются перед и после команды из ExecStart.
* `ExecReload` - команда для перезапуска службы, если она есть.
* `RestartSec` - команда указывает timeuot после которого служба попробует запустится.
* `TimeoutSec` - время ожидания запуска или остановки процесса по истечению которого запуск/остановка службы считаются неудачными.
* `User, Group` - пользователь / группа от имени которых будет запущен процесс службы.

_**Раздел [Install]**_ - содержит информацию, используемую при включении и отключении автозапуска модуля (enable/disable).
* `WantedBy` - определяет как модуль должен быть включен. Эта директива позволяет указать зависимость аналогично директиве Wants, но при этом включение автозапуска создаст символическую ссылку в каталоге systemd в качестве связи.
* `Alias` - определяет псевдоним (или псевдонимы) модуля.
* `DefaultInstance` - для модулей шаблонов, указывает экземпляр, который должен быть включен если пользователь не указал конкретный экземляр.

Таким образом можно создать собственный сервис, который потом нужно поместить в файл `/etc/systemd/systern/имя_сервиса.service`. После этого нужно перезапустить саму systemd, чтобы она узнала о новом сервисе:
```
sudo systemctl daemon-reload
```

##### Типы модулей системы инициализации systemd

| Тип | Описание |
| ------- | ----------- |
| `.service` | Служба (сервис, демон), которую нужно/можно запустить и работать она будет в фоновом режиме. Пример имени модуля: network.service. Изначально systemd поддерживала сценарии SysV, но в последнее время в каталоге `/etc/init.d` систем, которые используют `systemd`, практически пусто (или вообще пусто), а управление сервисами осуществляется только посредством `systemd` |
| `.device` | Устройство. Представляет устройство в дереве устройств. Работает вместе с udev: если устройство описано в виде правила udev, то его можно представить в systemd в виде модуля device |
| `.target` | Используется для группировки других модулей/юнитов. В systemd нет уровней запуска, вместо них используются цели. Например, цель multi-user.target описывает, какие модули должны быть запущены в многопользовательском режиме |
| `.timer` | Представляет собой таймер системы инициализации systemd. Аналог cron (запуск другого юнита, *.service)  |
| `.mount` | Точка монтирования файловой системы |
| `.automount` | Автоматическая точка монтирования. Используется для монтирования сменных носителей - флешек, внешних жестких дисков, оптических дисков и т.д. |
| `.snapshot` | Точка монтирования. Представляет точку монтирования. Система инициализации systemd контролирует все точки монтирования. При использовании systemd файл /etc/fstab уже не главный, хотя все еще может использоваться для определения точек монтирования |
| `.socket` | Сокет. Представляет сокет, находящийся в файловой системе или в Интернете. Поддерживаются сокеты AF_INET, AF_INET6, AF_UNIX. Реализация довольно интересная. Например, если сервису service1.service соответствует сокет service1.socket, то при попытке установки соединения с service1.socket будет запущен service1.service  |
| `.path` | Файл или каталог, созданный где-то в файловой системе |
| `.scope` | Процесс, который создан извне |
| `.slice` | Управляет системными процессами. Представляет собой группу иерархически организованных модулей |
| `.swap` | Представляет область подкачки (раздел подкачки) или файл подкачки (свопа) |

##### Target (Цели)
> _**Target - позволяет группировать 1 и более сервисов в единый блок. Группруя сервисы в target, ими проще управлять. Systemd автоматически включает/выключает target по событиям. Включение target означает включение всех сервисов, которые он объединяет в себе. Если в systemd настроен target my-favorite.target, то при загрузке системы systemd включит все сервисы, которые прописаны внутри my-favorite.target**_

Файлы целей systemd *.target предназначены для группировки вместе других юнитов systemd через цепочку зависимостей. Например юнит graphical.target, использующийся для старта графической сессии, запускает системные сервисы GNOME Display Manager (gdm.service) и Accounts Service (accounts–daemon.service) и активирует multi–user.target. В свою очередь multi–user.target запускает другие системные сервисы, такие как Network Manager (NetworkManager.service) или D-Bus (dbus.service) и активирует другие целевые юниты basic.target.

В systemd имеются предопределенные цели, которые напоминают стандартный набор уровней запуска. Некоторые цели называются runlevelN.target, чтобы упростить переход бывших пользователей init на systemd, а именно:

| Путь | Описание |
| ------- | ----------- |
| `poweroff.target (runlevelO.target)` | завершение работы и отключение системы |
| `rescue.target (runlevel1.target)` | однопользовательский режим, среда восстановления |
| `multi-user.target (runlevel{2,3,4}.target)` | многопользовательский режим, без графического интерфейса |
| `graphical.target (runlevel5.target)` | многопользовательский режим с графическим интерфейсом |
| `reboot.target (runlevel6.target)` | завершение работы и перезагрузка системы |

Управление службами осуществляется с помощью программы systemctl. Например:

1. `systemctl start <uмя.service>` - запускает сервис
2. `systemctl stop <uмя.service>` - останавливает сервис
3. `systemctl restart <uмя.service>` - перезапускает сервис


#### 2. [[⬆]](#toc) <a name='managing_services'>Управление сервисами при использовании systemd</a>

В systemd целью большинства действий являются «юниты/модули», являющиеся ресурсами, которыми systemd знает, как управлять. Модули распределяются по категориям по типу ресурса, который они представляют, и определяются файлами, известными как файлы модулей. 

Systemd содержит инструмент `systemctl`, который позволяет управлять работой служб: запускать и останавливать, проверять состояние, обновлять конфигурацию и т.д.

##### Параметры программы systemctl

| Параметр | Описание |
| ------- | ----------- |
| `systemctl start <uмя.service>` | запускает сервис |
| `systemctl stop <uмя.service>` | останавливает сервис  |
| `systemctl restart <uмя.service>` | перезапускает сервис  |
| `systemctl try-restart <uмя.service>` | перезапуск сервиса, только если он запущен  |
| `systemctl reload <uмя.service>` | перезагружает конфигурацию сервиса  |
| `systemctl status <uмя.service>` | отображает подробное состояние сервиса |
| `systemctl is-active <uмя.service>` | отображает только строку active (сервис запущен) или inactive (остановлен) |
| `systemctl епаblе <uмя.service>` | включает сервис (обеспечивает его автоматический запуск) |
| `systemctl disable <uмя.service>` | отключает сервис (сервис не будет автоматически запускаться при запуске системы) |
| `systemctl rееnаblе <uмя.service>` | деактивирует сервис и сразу его использует |
| `systemctl list-units` | Выводит список всех сервисов |
| `systemctl list-unit-files --type service` | Выводит список всех сервисов и сообщает, какие из них активированы, а какие - нет |
| `systemctl list-units --type service --all` | выводит состояние всех сервисов |
| `systemctl list-sockets` | Выводит список всех служб сокетов |

Примеры:
```
sudo systemctl start httpd.service
sudo systemctl stop httpd
```
Первая команда запускает сервис httpd (веб-сервер), вторая - останавливает. Обратите внимание, что "`.service`" можно не указывать. Раньше, чтобы отключить службу на определенном уровне запуска, нужно было удалить ее символическую ссылку из определенного каталога. Аналогично, чтобы служба запускалась на определенном уровне запуска (например, в графическом режиме), нужно бьло создать символическую ссьшку. Сейчас всего этого нет, а есть только команды епаblе и disable, что гораздо удобнее. 


#### 3. [[⬆]](#toc) <a name='create_unit_file'>Systemd — создание unit-файла</a>
Напишем service, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова

##### Cоздаём файл с конфигурацией для сервиса
```
sudo nano /etc/default/watchlog
```
![image](https://github.com/user-attachments/assets/c1b95428-4be5-4bcf-8b6d-e655ceb20699)

##### Создаем файл /var/log/watchlog.log, пишем туда строки на своё усмотрение, плюс ключевое слово ‘ALERT’
```
sudo nano /var/log/watchlog.log
```
![image](https://github.com/user-attachments/assets/03edf7c8-ba4e-4504-85cf-3080f9ae6814)

##### Создадим скрипт и сделаем файл исполняемым. Команда logger отправляет лог в системный журнал.
```
sudo nano /opt/watchlog.sh
sudo chmod +x /opt/watchlog.sh
```
![image](https://github.com/user-attachments/assets/898c53fe-df12-4d8e-8ce3-41a3257d7c4c)

##### Создадим юнит для сервиса
```
sudo nano /etc/systemd/system/watchlog.service
```
![image](https://github.com/user-attachments/assets/967748f1-4235-4737-b721-db4446f5baa7)

##### Создадим юнит для таймера
```
sudo nano /etc/systemd/system/watchlog.timer
```
![image](https://github.com/user-attachments/assets/0b773d98-1d2e-4a8a-b700-d18bd3ed4f36)

_После всех этих действий достаточно запустить таймер, а он уже сам запустит сам сервис_
```
sudo systemctl start watchlog.timer
```
_Проверяем_
```
tail -n 1000 /var/log/syslog  | grep word
```
![image](https://github.com/user-attachments/assets/cd681d67-a9ea-4134-b8de-641ee0e00ad8)


#### 4. [[⬆]](#toc) <a name='recommended_sources'>Дополнительные источники</a>

1. [Systemd - Википедия](https://ru.wikipedia.org/wiki/Systemd)
2. [????????????????](https://www.alexgur.ru/articles/2275/)
3. [????????????????](https://losst.pro/nastrojka-zagruzchika-grub)
4. Весь Linux Для тех, кто хочет стать профессионалом, стр.324
