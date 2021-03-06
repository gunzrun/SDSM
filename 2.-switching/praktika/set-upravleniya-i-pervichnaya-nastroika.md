# Сеть управления и первичная настройка

Настроим IP-адрес для управления.  
В наших лабах они не понадобятся, потому что мы настраиваем устройство через окно РТ. А вот в реальной жизни это вам жизненно необходимо.  
Для этого мы создаём виртуальный интерфейс и указываем номер интересующего нас влана. А далее работаем с ним, как с самым обычным физическим интерфейсом.

**msk-arbat-dsw1:**

```text
msk-arbat-dsw1(config)#interface vlan 2
msk-arbat-dsw1(config-if)#description Management
msk-arbat-dsw1(config-if)#ip address 172.16.1.2 255.255.255.0
```

**msk-arbat-asw3:**

```text
msk-arbat-asw3(config)#vlan 2
msk-arbat-asw3(config)#interface vlan 2
msk-arbat-asw3(config-if)#description Management
msk-arbat-asw3(config-if)#ip address 172.16.1.5 255.255.255.0
```

С msk-arbat-asw3 запускаем пинг до msk-arbat-dsw1:

```text
msk-arbat-asw3#ping 172.16.1.2

Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 172.16.1.2, timeout is 2 seconds:
..!!!
Success rate is 60 percent (3/5), round-trip min/avg/max = 4/4/4 ms
```

Первые пару пакетов могут потеряться на работу протокола [ARP](http://xgu.ru/wiki/ARP): определение соответствия IP-адрес — MAC-адрес. При этом MAC-адрес, порт и номер влана добавляются в таблицу коммутатора.

Самостоятельно настройте IP-адреса сети управления на остальных коммутаторах и проверьте их доступность

Собственно вот и вся магия. Зачастую к подобного рода действиям и сводится вся настройка, если вы не работаете в провайдере. С другой стороны, если вы работаете в провайдере, то, наверняка, такие вещи вам объяснять не нужно. Если желаете знать больше об этом, читайте: [VTP](http://xgu.ru/wiki/VTP), [QinQ](http://www.hilik.org.ua/q-in-q-настройка-в-cisco-catalyst-3750/), [зарезервированные номера VLAN](http://etherealmind.com/cisco-ios-vlan-1002-reserved-1005-purpose-function/)

Ещё один небольшой инструмент, который может немного увеличить удобство работы: banner. Это объявление, которое циска покажет перед авторизацией на устройство.

```text
Switch(config)#banner motd q
Enter TEXT message. End with the character 'q'.
It is just banner.
q

Switch(config)#
```

После motd вы указываете символ, который будет служить сигналом о том, что строка закончена. В это примере мы поставили “q”.

![](http://img-fotki.yandex.ru/get/4401/83739833.13/0_7fa45_63223d7b_XL.jpg)

> Относительно содержания баннера. Существует такая легенда: хакер вломился в сеть, что-то там поломал\украл, его поймали, а на суде оправдали и отпустили. Почему? А потому, что на пограничном роутере\(между интернет и внутренней сетью\), в banner было написано слово “Welcome”. “Ну раз просят, я и зашел”\)\). Поэтому считается хорошей практикой в баннере писать что-то вроде “Доступ запрещен!”.

## Резюме

Для упорядочивания знаний по пунктам разберём, что вам необходимо сделать:

1\) Настроить hostname. Это поможет вам в будущем на реальной сети быстро сориентироваться, где вы находитесь.

```text
Switch(config)#hostname HOSTNAME
```

2\) Создать все вланы и дать им название

```text
Switch(config)#vlan VLAN-NUMBER
Switch(config-vlan)#name NAME-OF-VLAN
```

3\) Настроить все access-порты и задать им имя

```text
Switch(config-if)#description DESCRIPTION-OF-INTERFACE
Switch(config-if)#switchport mode access
Switch(config-if)#switchport access vlan VLAN-NUMBER
```

Удобно иногда бывает настраивать интерфейсы пачками:

```text
msk-arbat-asw3(config)#interface range fastEthernet 0/6 — 10
msk-arbat-asw3(config-if-range)#description FEO
msk-arbat-asw3(config-if-range)#switchport mode access 
msk-arbat-asw3(config-if-range)#switchport access vlan 102
```

4\) Настроить все транковые порты и задать им имя:

```text
Switch(config-if)#description DESCRIPTION-OF-INTERFACE
Switch(config-if)#switchport mode trunk
Switch(config-if)#switchport trunk allowed vlan VLAN-NUMBERS
```

5\) **Не забывайте сохраняться:**

```text
Switch#copy running-config startup-config 
```

Итого: чего мы добились? Все устройства в одной подсети видят друг друга, но не видят устройства из другой. В следующем выпуске разбираемся с этим вопросом, а также обратимся к статической маршрутизации и L3-коммутаторам.  
В общем-то на этом данный урок можно закончить. В видео вы сможете ещё раз увидеть, как настраиваются вланы. В качестве домашнего задания настройте вланы на коммутаторах для серверов.

Здесь вы можете скачать конфигурацию всех устройств:  
[liftmeup\_configuration.zip](https://www.dropbox.com/s/sv5zwr037niwxgg/liftmeup_configuration.zip?dl=0).

**P.S.**  
_Важное дополнение: в предыдущей части, говоря о native vlan мы вас немного дезинформировали. На оборудовании cisco такая схема работы невозможна.  
Напомним, что нами предлагалось передавать на коммутатор msk-rubl-asw1 нетегированными кадры 101-го влана и принимать их там в первый.  
Дело в том, что, как мы уже упомянули выше, с точки зрения cisco с обеих сторон на коммутаторах должен быть настроен одинаковый номер влана, иначе начинаются проблемы с протоколом STP и в логах можно увидеть предупреждения о неверной настройке. Поэтому 101-й влан мы передаём на устройство обычным образом, кадры будут тегированными и соответственно, 101-й влан тоже необходимо создавать на msk-rubl-asw1._

Ещё раз хотим заметить, что при всём желании мы не сможем охватить все нюансы и тонкости, поэтому и не ставим перед собой такой задачи. Такие вещи, как принцип построения MAC-адреса, значения поля Ether Type или для чего нужен CRC в конце кадра, вам предстоит изучить самостоятельно.

Спасибо соавтору этого цикла, Максиму aka gluck.  
За предоставление дополнительных материалов хочу поблагодарить [Наташу Самойленко](http://xgu.ru/wiki/Категория:Автор_Наташа_Самойленко)

Читатели, не имеющие учётки на

