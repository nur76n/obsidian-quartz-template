---
layout: default
title: Proxmox Cloud-Init
nav_order: 70
---

# Proxmox Cloud-Init

## Подготовка шаблона

Скачиваем образ cloud image Ubuntu 20.04:
```
wget https://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-amd64.img
```

Создаем ВМ
```bash
qm create 9020 --memory 2048 --net0 virtio,bridge=vmbr0,tag=18 --agent enabled=1 --cores 2 --ostype l26 --name ubuntu20-template
```

импортируем скачанный образ в local-lvm
```bash
qm importdisk 9020 focal-server-cloudimg-amd64.img local-lvm
```

подключаем диск в созданную машину
```bash
qm set 9020 --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-9020-disk-0
```

увеличиваем размер диска
```bash
qm resize 9020 scsi0 +8G
```

Не забываем установить ограничение на ввод-вывод, чтобы машина не смогла забрать все ресурсы
```bash
qm set 9020 --scsi0 local-lvm:base-9120-disk-0,mbps_rd=100,mbps_wr=100,iops_rd=100,iops_wr=100
```

Добавляем устройство CloudInit Drive 
```bash
qm set 9020 --scsi2 local-lvm:cloudinit
```

устанавливаем загрузочное устройство
```bash
qm set 9020 --boot c --bootdisk scsi0
```

устанавливаем отображание консоли через xterm.js
```bash
qm set 9020 --serial0 socket --vga serial0
```

---
Далее через веб интерфейс настраиваем остальные параметры.  
Во вкладке Cloud-Init:
- User: `srv-admin`
- Password: `superpassword`
- DNS domain: `corp.company.com`
- DNS servers: `10.0.1.11,10.0.1.12`
- SSH pubic key: `<открытый ключ>`
- IP Config: IPv4: `Static`, IPv4/CIDR: `10.0.8.251/24`, Gateway: `10.0.8.1`

Либо командой:
```bash
qm set 9020 --ciuser srv-admin --cipassword srv-password --searchdomain corp.company.kz --nameserver 10.0.1.11,10.0.1.12 --ipconfig0 ip=10.0.8.251/24,gw=10.0.8.1 --sshkeys /path/to/id_rsa.pub
```


## Донастраиваем систему
---
Включаем машину 

Обновляем систему
```bash
sudo apt update && sudo apt dist-upgrade -y
```
Включаем bash autocompletion root-пользователя, раскомментировав эти строки в файле `/root/.bashrc`:
```bash
if [ -f /etc/bash_completion ] && ! shopt -oq posix; then
    . /etc/bash_completion
fi
```
Устанавливаем правильный часовой пояс, в моем случае `Asia/Almaty`
```bash
sudo dpkg-reconfigure tzdata
```

Выключаем машину

Конвертируем в шаблон