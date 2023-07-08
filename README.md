# ha-mikrotik (original https://github.com/svlsResearch/ha-mikrotik)
High availability code for Mikrotik routers

# Status: December 10th 2019
**RouterOS 6.44.6** is the only version that the author runs and tests with as of now. Anything newer is unknown and not recommended until tested extensively. Do not just upgrade RouterOS expecting it to work, Mikrotik has broken a series of features that ha-mikrotik relies on at various points in time. If you are not comfortable testing new versions that make break your entire setup, wait for the author or another party to confirm compatibility.

This has been tested stable across 6 different pairs of CCR1009s for over a year, there have been multiple adminstrative failover events and a few cases of hardware failovers. Please ensure you are using compatible hardware, RouterOS, and ha-mikrotik releases.

# Warning
Please do not test this on production routers. This should be tested in a lab setup with complete out of band serial access.
This was developed on the CCR1009-8g-1s-1s+ and is in use in our production environment. Proceed at your own risk, the code can potentionally wipe out 
all of your configuration and files on your device.

Extensive documentation is still needed. This is being delivered as a proof of concept.
You will need to do a bit of code reading and testing to figure out how it works.

# Issues
The #1 issue is a race condition during the startup of the secondary after it gets an updated configuration. It needs to quickly disable all of the interfaces
so that it doesn't end up taking traffic (MACs are cloned) from the active router. If you use spanning tree on your switches, it is likely that this
will happen fast enough and the Layer2/3 won't have time to come up and cause issues. Test this very carefully, you will get very strange results if your ports
start forwarding instantly from your upstream switch.

# Concept
Using a dedicated interface, VRRP, scripts, and backups, we can make a pair of Mikrotik routers highly available.
Configuration and files are actively synchronized to the standby and the standby remains ready to takeover when the VRRP heartbeat fails.

# Hardware originally developed for
Pair of CCR1009-8g-1s-1s+
RouterOS v6.33.5
Routerboard firmware 3.27
Bootstrapped from complete erased routers and then config built up once HA installed.

# Video demonstrating/explaining ha-mikrotik by The Network Berg
[MIkrotik - REAL HA Configuration](https://www.youtube.com/watch?v=GEef9P8wwxs)

# Installing
1. Source a pair of matching routers, ideally CCR1009-8g-1s-1s+.
2. Install RouterOS v6.44.6 and make sure the Routerboard firmware is up date.
3. Ensure you have serial connections to both devices.
4. Reset both routers using the command:
`/file remove [find]; /system reset-configuration keep-users=no no-defaults=yes skip-backup=yes`
5. Connect an ethernet cable between ether8 and ether8.
6. On one router, configure a basic network interface on ether1 with an IP address of your choosing. Just enough to be able to copy a file.
7. Upload HA_init.rsc and import it:
`/import HA_init.rsc`
8. Install HA (note to replace the fields of macA, macB, and password. I suggest a sha1 hex hash for the password.
`$HAInstall interface="ether8" macA="[MAC_OF_A_ETHER8]" macB="[MAC_OF_B_ETHER_8]" password="[A RANDOM PASSWORD OF YOUR CHOOSING]"`
9. Follow the instructions given by $HAInstall to bootstrap the secondary. I use the MAC telnet that is suggested at the top but any other method is sufficient.
10. Once router B is bootstrapped, it will reboot itself into a basic networking mode. It needs to be pushed the current configuration.
`$HASyncStandby`
11. B will now reboot and when it returns, it should be in standby mode. A should be the active router. You can now reboot A and B will takeover.

# Upgrading ha-mikrotik
1. Download a new release of ha-mikrotik
2. Upload HA_init.rsc to the active and import it:
`/import HA_init.rsc`
3. Run `$HAPushStandby` on the active, this should push the new code and reboot the standby.
4. Wait for the standby to come back, login and make sure everythig looks good. (/log print).
5. Run `$HASyncStandby` on the active, there should be no changes (unless something else changed on the active inbetween).
6. **THIS WILL REBOOT THE ACTIVE** Run `$HASwitchRole` on the active.
7. Your active is now the previous standby and both are upgraded once the standby boots.

# Rebuilding a hardware failed standby
Rebuilding failed hardware is similar to a new installation except that we don't need to reset both and don't need to bring in a new HA_init, assuming both RouterOS are compatible.

Install a compatible version of RouterOS on the new hardware and factory reset the configuration. Connect your new hardware the same exact way the old one was. We assume you have used the default of ether8 for the $haInterface.

**If A is active, run from A:**
1. `$HAInstall interface=$haInterface macA=$haMacMe macB="[NEW_MAC_OF_B_ETHER8]" password=$haPassword`
2. Follow on screen instructions just like original install.
3. Once the standby comes back, `$HASyncStandby`.
4. Done.

**If B is active, run from B:**
1. `$HAInstall interface=$haInterface macB=$haMacMe macA="[NEW_MAC_OF_A_ETHER8]" password=$haPassword`
2. Follow on screen instructions just like original install.
3. Once the standby comes back, `$HASyncStandby`.
4. Done.

# Скрипт не работал с CCR2004-1G-12S+2XS, Firmware 7.*, чтобы решить вопрос, я чуть чуть переделал скрипт и добавил еще один скрипт для нормальной работы скрипта.
# Как новые скрипты работают 
  Каждое утро конфиг с основного микротика заливается на резервный. 
  Если вам надо в ручном режиме залить конфиг на резервный конфиг то вам достаточно запустить скрипт **ha_pushbackup**
  
  **Если с вашим микротиком все будет иначе и стандартный скрипт заработает то мою доработку не надо применять!**
  
# Как установить
  1. Подготовьте 2 одинаковых микротика, обновите их чтобы оба микротика были с одинаковой версией **RouterOS** и **Routerboard firmware**
  2. убедитесь что у вас есть консольное подключение к обоим микротикам
  3. Сбросьте оба маршрутизатора с помощью команды: 
  ```bash
  /file remove [find]; /system reset-configuration keep-users=no no-defaults=yes skip-backup=yes
  ```
  4. Подключите кабель ethernet между ether10 и ether10 (я выбираю последний порт микротика)
  5. На одном маршрутизаторе настройте IP адрес на интерфейс ether1. Нужно для того чтобы скопировать файл.
  6. Загрузите файл HA_init.rsc и импортируйте его:
  ```bash
  /import HA_init.rsc
  ```
  7. Установите HA (обратите внимание, что необходимо заменить поля macA, MacB и пароль.
  ```bash
  $HAInstall interface="ether10" macA="[MAC_OF_A_ETHER10]" macB="[MAC_OF_B_ETHER_10]" password="[ВАШ ПАРОЛЬ]"
  ```
  8. Следуйте инструкциям, приведенным в $HAInstall, для начальной загрузки дополнительного устройства. Я использую MAC telnet.
  (Скрипт выдаст конфиг которую надо скопировать и вставить на второй микротик, сохраните этот текст конфига)
  9. Вот отсюда начинаются мои изменения
  10. Откройте скрипт **ha_startup** и замените код на код из файла **ha_startup_fix** (в папке fix) и сохраните скрипт. Внимание, название скрипта на микротике не менять!
  11. Создайте новый скрипт, название скрипта **fix_me** установите галку **Don't Require Permissions**
  содержание скрипта:
  ```bash
  :global haInterface; 
  /log warning "fix_me: start";
  /interface ethernet enable $haInterface;
  /interface ethernet reset-mac-address [find default-name="$haInterface"];
  /interface vrrp enable HA_VRRP;
  /ip service enable ftp; 
  /log warning "fix_me: end";
  ```
  сохраните скрипт. 
  
  12. подключитесь к микротику 2 и там тоже создайте этого скрипта. 
  
  13. откройте терминал и введите код которую выдал скрипт из 8 пункта.

  Готово.
