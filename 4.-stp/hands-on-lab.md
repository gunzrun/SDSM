# Практика

> Наверное, большинство ошибок в Packet Tracer допущено в части кода, отвечающего за симуляцию STP, будте готовы. В случае сомнения 
> сохранитесь, закройте PT и откройте заново

Итак, переходим к практике. Для начала внесем некоторые изменения в топологию — добавим избыточные линки. Учитывая сказанное в самом начале, вполне логично было бы сделать это в московском офисе в районе серверов — там у нас свич msk-arbat-asw2 доступен только через asw1, что не есть гуд. Мы отбираем \(пока, позже возместим эту потерю\) гигабитный линк, который идет от msk-arbat-dsw1 к msk-arbat-asw3, и подключаем через него asw2. Asw3 пока подключаем в порт Fa0/2 dsw1. Перенастраиваем транки:

> msk-arbat-dsw1\(config\)#interface gi1/2
> msk-arbat-dsw1\(config-if\)#description msk-arbat-asw2
> msk-arbat-dsw1\(config-if\)#switchport trunk allowed vlan 2,3
> msk-arbat-dsw1\(config-if\)#int fa0/2
> msk-arbat-dsw1\(config-if\)#description msk-arbat-asw3
> msk-arbat-dsw1\(config-if\)#switchport mode trunk
> msk-arbat-dsw1\(config-if\)#switchport trunk allowed vlan 2,101-104

> msk-arbat-asw2\(config\)#int gi1/2
> msk-arbat-asw2\(config-if\)#description msk-arbat-dsw1
> msk-arbat-asw2\(config-if\)#switchport mode trunk
> msk-arbat-asw2\(config-if\)#switchport trunk allowed vlan 2,3
> msk-arbat-asw2\(config-if\)#no shutdown

Не забываем вносить все изменения в документацию!

![](https://habrastorage.org/getpro/habr/post_images/6cf/e25/20f/6cfe2520ffacd459babe1bad42beeb6c.png)

[Скачать актуальную версию документа](http://dl.dropbox.com/u/47476169/Habr/V4/Network_Planning_v4.xlsx)

Теперь посмотрим, как в данный момент у нас *самонастроился* STP. Нас интересует только VLAN0003, где у нас, судя по схеме, петля.

```
msk-arbat-dsw1>en
msk-arbat-dsw1#show spanning-tree vlan 3
```

Разбираем по полочкам вывод команды

![](https://habrastorage.org/getpro/habr/post_images/2f3/4f1/c0f/2f34f1c0fbf7a97e7b970aa692565606.png\)

Итак, какую информацию мы можем получить? Так как по умолчанию на современных цисках работает PVST+ \(т.е. для каждого влана свой процесс STP\), и у нас есть более одного влана, выводится информация по каждому влану в отдельности, каждая запись предваряется номером влана. Затем идет вид STP: ieee значит PVST, rstp — Rapid PVST, mstp то и значит. Затем идет секция с информацией о корневом свиче: установленный на нем приоритет, его mac-адрес, стоимость пути от текущего свича до корневого, порт, который был выбран в качестве корневого \(имеет лучшую стоимость\), а также настройки таймеров STP. Далее- секция с той же информацией о текущем свиче \(с которого выполняли команду\). Затем- таблица состояния портов, которая состоит из следующих колонок \(слева направо\):

* собственно, порт
* его роль \(Root- корневой порт, Desg- назначенный порт, Altn- дополнительный, Back- резервный\)
* его статус \(FWD- работает, BLK- заблокирован, LIS- прослушивание, LRN- обучение\)
* стоимость маршрута до корневого свича
* Port ID в формате: приоритет порта.номер порта
* тип соединения

Итак, мы видим, что Gi1/1 корневой порт, это дает некоторую вероятность того, что на другом конце линка корневой свич. Смотрим по схеме, куда ведет линк: ага, некий msk-arbat-asw1.

```
msk-arbat-asw1#show spanning-tree vlan 3
```

И что же мы видим?
```
VLAN0003
 Spanning tree enabled protocol ieee
 Root ID    Priority    32771
            Address     0007.ECC4.09E2
            This bridge is the root
            Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
```

Вот он, наш корневой свич для VLAN0003.

А теперь посмотрим на схему. Ранее, мы увидели в состоянии портов , что dsw1 блокирует порт Gi1/2, разрывая таким образом петлю. Но является ли это оптимальным решением? Нет, конечно. Сейчас наша новая сеть работает точь-в-точь как старая- трафик от asw2 идет только через asw1. Выбор корневого маршрутизатора никогда не нужно оставлять на совесть глупого STP. Исходя из схемы, наиболее оптимальным будет выбор в качестве корневого свича dsw1- таким образом, STP заблокирует линк между asw1 и asw2. Теперь это все надо объяснить недалекому протоколу. А для него главное что? Bridge ID. И он неслучайно складывается из двух чисел. Приоритет- это как раз то слагаемое, которое отдано на откуп сетевому инженеру, чтобы он мог повлиять на результат выбора корневого свича. Итак, наша задача сводится к тому, чтобы уменьшить \(меньше-лучше, думает STP\) приоритет нужного свича, чтобы он стал Root Bridge.  Есть два пути:

1\) вручную установить приоритет, заведомо меньший, чем текущий:

```
msk-arbat-dsw1>enable
msk-arbat-dsw1#configure terminal
msk-arbat-dsw1\(config\)#spanning-tree vlan 3 priority ?
 <0-61440>  bridge priority in increments of 4096
msk-arbat-dsw1\(config\)#spanning-tree vlan 3 priority 4096
```

Теперь он стал корневым для влана 3, так как имеет меньший Bridge ID:

```
msk-arbat-dsw1#show spanning-tree vlan 3
VLAN0003
 Spanning tree enabled protocol ieee
 Root ID    Priority    4099
            Address     000B.BE2E.392C
            This bridge is the root
            Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
```

2\) дать умной железке решить все за тебя:

```
msk-arbat-dsw1\(config\)#spanning-tree vlan 3 root primary
```

Проверяем:

```
 msk-arbat-dsw1#show spanning-tree vlan 3
 VLAN0003
  Spanning tree enabled protocol ieee
  Root ID    Priority    24579
             Address     000B.BE2E.392C
             This bridge is the root
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
```

Мы видим, что железка поставила какой-то странный приоритет. Откуда взялась эта круглая цифра, спросите вы? А все просто- STP смотрит минимальный приоритет \(т.е. тот, который у корневого свича\), и уменьшает его на два шага инкремента \(который составляет 4096, т.е. в итоге 8192\). Почему на два? А чтобы была возможность на другом свиче дать команду spanning-tree vlan n root secondary \(назначает приоритет=приоритет корневого-4096\), что позволит нам быть уверенными, что, если с текущим корневым свичом что-то произойдет, его функции перейдут к этому, “запасному”. Вероятно, вы уже видите на схеме, как лампочка на линке между asw2 и asw1 пожелтела? Это STP разорвал петлю. Причем именно в том месте, в котором мы хотели. Sweet! Зайдем проверим: лампочка — это лампочка, а конфиг — это факт.

```
 msk-arbat-asw2#show spanning-tree vlan 3
 VLAN0003
  Spanning tree enabled protocol ieee
  Root ID    Priority    24579
             Address     000B.BE2E.392C
             Cost        4
             Port        26\(GigabitEthernet1/2\)
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    32771  \(priority 32768 sys-id-ext 3\)
             Address     000A.F385.D799
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  20
 
 Interface        Role    Sts    Cost      Prio.Nbr Type
 ---------------- ---- --- --------- -------- --------------------------------
 Fa0/1            Desg    FWD    19        128.1    P2p
 Gi1/1            Altn    BLK    4         128.25   P2p
 Gi1/2            Root    FWD    4         128.26   P2p
```

Теперь полюбуемся, как работает STP: заходим в командную строку на ноутбуке PTO1 и начинаем бесконечно пинговать наш почтовый сервер \(172.16.0.4\). Пинг сейчас идет по маршруту ноутбук-asw3-dsw1-gw1-dsw1\(ну тут понятно, зачем он крюк делает — они из разных вланов\)-asw2-сервер. А теперь поработаем Годзиллой из SimСity: нарушим связь между dsw1 и asw2, вырвав провод из порта \(замечаем время, нужное для пересчета дерева\).

Пинги пропадают, STP берется за дело, и за *каких-то* 30 секунд коннект восстанавливается. Годзиллу прогнали, пожары потушили, связь починили, втыкаем провод обратно. Пинги опять пропадают на 30 секунд! Мда-а-а, как-то не очень быстро, особенно если представить, что это происходит, например, в процессинговом центре какого-нибудь банка.

Но у нас есть ответ медленному PVST+! И ответ этот — Быстрый PVST+ \(так и называется, это не шутка: Rapid-PVST\). Посмотрим, что он нам дает. Меняем тип STP на всех свичах в москве командой конфигурационного режима: spanning-tree mode rapid-pvst

Снова запускаем пинг, вызываем Годзиллу... Эй, где пропавшие пинги? Их нет, это же Rapid-PVST. Как вы, наверное, помните из теоретической части, эта реализация STP, так сказать, “подстилает соломку” на случай падения основного линка, и переключается на дополнительный \(alternate\) порт очень быстро, что мы и наблюдали. Ладно, втыкаем провод обратно. Один потерянный пинг. Неплохо по сравнению с 6-8, да?

## EtherChannel

Помните, мы отобрали у офисных работников их гигабитный линк и отдали его в пользу серверов? Сейчас они, бедняжки, сидят, на каких-то ста мегабитах, прошлый век! Попробуем расширить канал, и на помощь призовем EtherChannel. В данный момент у нас соединение идет от fa0/2 dsw1 на Gi1/1 asw3, отключаем провод. Смотрим, какие порты можем использовать на asw3: ага, fa0/20-24 свободны, кажется. Вот их и возьмем. Со стороны dsw1 пусть будут fa0/19-23. Соединяем порты для EtherChannel между собой. На asw3 у нас на интерфейсах что-то настроено, обычно в таких случаях используется команда конфигурационного режима default interface range fa0/20-24, сбрасывающая настройки порта \(или портов, как в нашем случае\) в дефолтные. Packet tracer, увы, не знает такой хорошей команды, поэтому в ручном режиме убираем каждую настройку, и тушим порты \(лучше это сделать, во избежание проблем\)

```
msk-arbat-asw3\(config\)#interface range fa0/20-24
msk-arbat-asw3\(config-if-range\)#no description
msk-arbat-asw3\(config-if-range\)#no switchport access vlan
msk-arbat-asw3\(config-if-range\)#no switchport mode
msk-arbat-asw3\(config-if-range\)#shutdown
```

ну а теперь волшебная команда
```
msk-arbat-asw3\(config-if-range\)#channel-group 1 mode on
```

то же самое на dsw1:
```
msk-arbat-dsw1\(config\)#interface range fa0/19-23
msk-arbat-dsw1\(config-if-range\)#channel-group 1 mode on
```

поднимаем интерфейсы asw3, и вуаля: вот он, наш EtherChannel, раскинулся аж на 5 физических линков. В конфиге он будет отражен как interface Port-channel 1. Настраиваем транк \(повторить для dsw1\):
```
msk-arbat-asw3\(config\)#int port-channel 1
msk-arbat-asw3\(config-if\)#switchport mode trunk
msk-arbat-asw3\(config-if\)#switchport trunk allowed vlan 2,101-104
```

Как и с STP, есть некая трудность при работе с etherchannel в Packet Tracer’e. Настроить-то мы, в принципе, можем по вышеописанному сценарию, но вот проверка работоспособности под большим вопросом: после отключения одного из портов в группе, трафик перетекает на следующий, но как только вы вырубаете второй порт — связь теряется и не восстанавливается даже после включения портов.

Отчасти в силу только что озвученной причины, отчасти из-за ограниченности ресурсов мы не сможем раскрыть в полной мере эти вопросы и посему оставляем бОльшую часть на самоизучение.