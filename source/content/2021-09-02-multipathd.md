# Multipath

## Установка
---

```bash
apt install multipath-tools
```
Включаем автозапуск и стартуем службу
```bash
systemctl enable multipath-tools
systemctl start multipath-tools
```
Теперь приступаем к настройке.
  
  

## Настройка
---

Редактируем файл `/etc/multipath.conf `
```bash
blacklist {
    wwid .*
}

blacklist_exceptions {
    wwid "0x201.0013780....0"
}


defaults {
    polling_interval        2
    prio                    alua
    uid_attribute           ID_WWN
    rr_min_io               100
    user_friendly_names     yes
    path_grouping_policy    failover
    path_selector           "round-robin 0"
    no_path_retry           5
    failback                immediate
    rr_weight               priorities
    find_multipaths         on
}

```

Применяем настройки
```bash
multipathd -k"reconfigure"
```
Проверяем состояние и дебаг информацию
```bash
multipath -ll
multipath -v4
```


## Настройка LVM
---
При запуске выполняется команда vgscan, которая выполнит поиск меток LVM на блочных устройствах системы с целью определения того, какие из них представляют собой физические тома, а также получения метаданных и создания списков групп томов. Имена физических томов хранятся в файле `/etc/lvm/.cache` на каждом узле в системе. Последующие команды будут обращаться к этому файлу, при этом не будет необходимости в повторном сканировании.

С помощью фильтров, определяемых в `lvm.conf`, можно управлять тем, какие устройства будут сканироваться. Фильтры представляют собой набор регулярных выражений, применяемых к именам устройств в каталоге `/dev` с целью разрешения или запрета определения блочного устройства.

Приведенные ниже примеры демонстрируют использование фильтров. Следует отметить, что некоторые примеры не являются лучшими решениями, так как регулярные выражения свободно сопоставляются с полными путями. Например, `a/.*loop.*/` соответствует не только `a/loop/`, но и `/dev/solooperation/lvol1`.

Следующий фильтр добавит все найденные устройства, что является поведением по умолчанию в случае, если фильтры не заданы.

Редактируем файл `/etc/lvm/lvm.conf`: 

Комментим эту строку
```bash
global_filter = [ "r|/dev/zd.*|", "r|/dev/mapper/pve-.*|" "r|/dev/mapper/.*-(vm|base)--[0-9]+--disk--[0-9]+|"]
```

И добавляем эту
```bash
global_filter = [ "a|/dev/sda|", "a|/dev/sdb|", "a|/dev/mapper/mpath.*|", "r|/dev/sd.*|", "r|/dev/zd.*|", "r|/dev/mapper/pve-.*|", "r|/dev/mapper/.*-(vm|base)--[0-9]+--disk--[0-9]+|"]
```
или эту (где запрещаем по модели SAN)
```bash
global_filter = ["r|/dev/disk/by-id/scsi-SQsan_XS3226.*|", "r|/dev/zd.*|", "r|/dev/mapper/pve-.*|", "r|/dev/mapper/.*-(vm|base)--[0-9]+--disk--[0-9]+|"]
```
или блочим по ip
```bash
global_filter = ["r|/dev/disk/by-path/ip-.*|", "r|/dev/zd.*|", "r|/dev/mapper/pve-.*|" "r|/dev/mapper/.*-(vm|base)--[0-9]+--disk--[0-9]+|"]
```


```bash
pvscan
vgscan
lsblk
```

## Увеличение размера
---
```bash
iscsiadm -m node --targetname target_name -R
multipathd -k"resize map multipath_device"
cfdisk /dev/mapper/multipath_disk
pvresize /dev/mapper/multipath_disk_part_n
```

## Отключение зависших дисков

```bash
dmsetup info -c | grep san1
vg--san1--vol1-vm--148--disk--1 253  27 L--w    0    3      0 LVM-6ZO29Qr8HJuM3YzL23SNNhoKHqXbmFA2V9iiZeCzqRVyy0I8X1EW7Mu1y7Y9UlIy
vg--san1--vol1-vm--133--disk--0 253  29 L--w    0    1      0 LVM-6ZO29Qr8HJuM3YzL23SNNhoKHqXbmFA2Vl7REiJbdkl98V6DHqU3LAhGNOGbpKfG
vg--san1--vol1-vm--148--disk--0 253  25 L--w    0    2      0 LVM-6ZO29Qr8HJuM3YzL23SNNhoKHqXbmFA2QCIyXwBzBdN102UYoyHkcfb1ekLjCMRB
vg--san1--vol1-vm--147--disk--0 253  26 L--w    0    1      0 LVM-6ZO29Qr8HJuM3YzL23SNNhoKHqXbmFA2eKK6h60xG3VJ2ZkfwU3DcvG3uuKUSBMO
mpath_san1_vol1                 253  24 L--w    5    1     33 mpath-0x20200013780d74c0
vg--san1--vol1-vm--144--disk--0 253  28 L--w    0    1      0 LVM-6ZO29Qr8HJuM3YzL23SNNhoKHqXbmFA2roNDaYIhlsypfMPo3e4e6qnxOIx3z2Yc
```

```bash
ls -la /sys/dev/block/253\:27
lrwxrwxrwx 1 root root 0 Jun  6 20:09 /sys/dev/block/253:27 -> ../../devices/virtual/block/dm-27
```

```bash
dmsetup remove /dev/dm-27
```


## Полезные ссылки
---
* [Статья от RedHat о фильрах LVM](https://access.redhat.com/documentation/ru-ru/red_hat_enterprise_linux/5/html/logical_volume_manager_administration/lvm_filters)
* [Корректность фильтра можно проверить на сайте](https://regex101.com/)
* [Конфигурационный файл DM-Multipath
](https://help.ubuntu.ru/wiki/%D1%80%D1%83%D0%BA%D0%BE%D0%B2%D0%BE%D0%B4%D1%81%D1%82%D0%B2%D0%BE_%D0%BF%D0%BE_ubuntu_server/%D0%BC%D0%BD%D0%BE%D0%B6%D0%B5%D1%81%D1%82%D0%B2%D0%B5%D0%BD%D0%BD%D0%BE%D0%B5_%D1%81%D0%B2%D1%8F%D0%B7%D1%8B%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5_%D1%83%D1%81%D1%82%D1%80%D0%BE%D0%B9%D1%81%D1%82%D0%B2/configuration)
