Ниже — итоговая последовательность, которая у вас сработала: VirtualBox на MacBook M5 → Windows 11 ARM → сеть через NAT, virtio-net и ручной DNS.

## Подготовка

1. Выключите виртуальную машину полностью (состояние «Выключена», не «Сохранена»). Иначе `VBoxManage` выдаст ошибку блокировки машины.
2. На Mac скачайте ISO `virtio-win` (в диалоге использовалась версия `0.1.302-1`). Этот драйвер подтверждён в отчётах пользователей для Windows ARM в VirtualBox.[1]
3. Узнайте точное имя ВМ:
   ```bash
   "/Applications/VirtualBox.app/Contents/MacOS/VBoxManage" list vms
   ```
   У вас она называется `Windows 10`.

## Настройка VirtualBox

4. Откройте «Настройки → Сеть → Адаптер 1». Включите адаптер, выберите тип подключения `NAT`, тип адаптера `Paravirtualized Network (virtio-net)` и оставьте включённым «Кабель подключён». 
5. Если virtio-net не выбирается в интерфейсе, задайте его командой (ВМ выключена):
   ```bash
   "/Applications/VirtualBox.app/Contents/MacOS/VBoxManage" modifyvm "Windows 10" --nictype1 virtio
   ```
6. Проверьте результат:
   ```bash
   "/Applications/VirtualBox.app/Contents/MacOS/VBoxManage" showvminfo "Windows 10" --machinereadable | grep -E "nic|cable"
   ```
   Нужно увидеть `nic1="nat"`, `nictype1="virtio"`, `cableconnected1="on"`. У вас так и было.
7. Подключите ISO `virtio-win` в «Настройки → Носители» к виртуальному оптическому приводу.
8. Необязательно: включите DNS-резолвер хоста:
   ```bash
   "/Applications/VirtualBox.app/Contents/MacOS/VBoxManage" modifyvm "Windows 10" --natdnshostresolver1 on
   ```
   У вас это не решило проблему, поэтому обязательным шагом он не является.

## Установка драйвера в Windows

9. Запустите ВМ. В «Диспетчере устройств» найдите Ethernet-контроллер без драйвера. Либо на экране «Давайте подключим вас к сети» нажмите «Установить драйвер».
10. Укажите драйверу папку `NetKVM\w11\ARM64` на подключённом виртуальном диске. Папка `amd64` не подойдёт, так как у вас ARM.[1]
11. После установки в списке адаптеров должен появиться `Red Hat VirtIO Ethernet Adapter` со статусом `Up`.

## Проверка сети

12. В PowerShell выполните:
    ```powershell
    Get-NetAdapter
    ipconfig /all
    ```
    Ожидаемые значения при NAT: IP `10.0.2.15`, шлюз `10.0.2.2`.[2]
13. Проверьте шлюз и интернет по IP:
    ```powershell
    Test-NetConnection 10.0.2.2
    Test-NetConnection 1.1.1.1 -Port 443
    ```
    У вас обе проверки прошли. Это показало, что NAT работает.
14. Проверьте DNS:
    ```powershell
    Resolve-DnsName microsoft.com
    ```
    У вас здесь был таймаут — это и была реальная причина отсутствия интернета.

## Исправление DNS

15. Узнайте имя адаптера: `Get-NetAdapter` (у вас `Ethernet 2`).
16. Задайте DNS вручную (PowerShell от администратора). Следите за написанием `-ServerAddresses`, одна «s» на конце:
    ```powershell
    Set-DnsClientServerAddress -InterfaceAlias "Ethernet 2" -ServerAddresses ("1.1.1.1","8.8.8.8")
    ipconfig /flushdns
    ```
17. Проверьте: `Resolve-DnsName microsoft.com` должна вернуть адреса. У вас вернулись `150.171.109.35` и IPv6-адрес.
18. Откройте Edge и проверьте сайты. При необходимости перезапустите браузер.

## Контроль после перезагрузки

19. Перезагрузите Windows и проверьте, что настройки сохранились:
    ```powershell
    Get-DnsClientServerAddress -InterfaceAlias "Ethernet 2" -AddressFamily IPv4
    ```
    В `ServerAddresses` должны остаться `1.1.1.1` и `8.8.8.8`.

## Если снова пропадёт сеть

Быстрая проверка по порядку: сначала IP-адрес (`ipconfig /all`), затем шлюз (`Test-NetConnection 10.0.2.2`), потом интернет по IP (`Test-NetConnection 1.1.1.1 -Port 443`) и в последнюю очередь DNS (`Resolve-DnsName`). Так вы сразу видите, на каком уровне проблема.

Источники
[1] Windows 11 ARM64 guest: usbnet adapter ("UsbNcm Host Device") fails with Code 10 ... https://github.com/VirtualBox/virtualbox/issues/877
[2] 9 Virtual Networking https://docs.oracle.com/en/virtualization/virtualbox/7.2/user/networkingdetails.html
