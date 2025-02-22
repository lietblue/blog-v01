---
slug: proxmox-cloudinit
title: 在 Proxmox 中使用 Cloud-init 虚拟机模板
description: 好吃的 Cloud-init 一口可以吃完
date: 1740238334

---


#### 下载镜像
debian cloud-init 镜像有多种类型   
azure - 微软云 Azure 优化版本  
ec2 - 亚马逊云 EC2 优化版本  
generic - 通用镜像  
genericcloud - 通用云镜像 删除的全部的物理硬件驱动  
nocloud - 测试用 没有cloud-init 可以无密码登录 root 账户  

下载链接(官方): https://cloud.debian.org/images/cloud/  
下载链接(USTC镜像源): https://mirrors.ustc.edu.cn/debian-cdimage/cloud/  

进去之后点击 debian 的版本如 bookworm 之后点 latest 选你要下的镜像即可  

下载镜像有两种: raw/qcow2 喜欢哪个就吃哪个  
其中与 下载镜像 前辍同名结尾为json 的文件中包含 该镜像的软件包及版本 与 该镜像的hash 可以核对一下镜像的完整性  

本教程以 [bookworm/latest/debian-12-genericcloud-amd64.raw](https://cloud.debian.org/images/cloud/bookworm/latest/debian-12-genericcloud-amd64.raw) 为例  
```bash
wget https://cloud.debian.org/images/cloud/bookworm/latest/debian-12-genericcloud-amd64.raw
```
#### 核对下载内容(可选)  
`shasum -a 512 debian-12-genericcloud-amd64.raw | xxd -p -r | base64` 与 json 里面的 **sha512:base64** 比较即可 

#### 修改镜像 (可选):
安装镜像修改工具 给镜像安装 qemu-guest-agent和fastfetch
```bash
apt update -y && apt install libguestfs-tools -y  
virt-customize -a debian-12-genericcloud-amd64.raw \
    --install qemu-guest-agent,fastfetch \
    --truncate /etc/machine-id
```

#### 创建VM模板  
```bash
qm create 9000 --name "debian-cloudinit" \
  --memory 2048 \
  --cores 2 \
  --net0 virtio,bridge=vmbr0 \
  --ostype l26
qm importdisk 9000 debian-12-genericcloud-amd64.raw local-lvm  
```
我更习惯于用 virtio-scsi-single 你也可以用 virtio-scsi-pci  
--scsi0
```bash
qm set 9000 --scsihw virtio-scsi-single --scsi0 local-lvm:vm-9000-disk-0,discard=on,ssd=1  
qm set 9000 --boot order=scsi0  
qm set 9000 --ide2 local-lvm:cloudinit  
qm set 9000 --agent enabled=1  
qm set 9000 --serial0 socket --vga serial0  
qm template 9000  
```

#### 用模板创建VM  
设置ssh key 当然你也可以在pve cloudinit页面也可以设置
```bash
qm clone 9000 201 --name debian-cloudinit-test  
qm set 201 --sshkey ~/.ssh/authorized_keys.pub  
qm set 201 --sshkeys:ssh-rsa...  
```

你也可以创建一个 cloud-init 的配置文件 cloudinit-config.yaml  并放一些自定义配置 如 k3sup所需的 sudo: ALL=(ALL) NOPASSWD:ALL  
```yml
users:  
- name: calvin  
  groups: sudo  
  sudo: ALL=(ALL) NOPASSWD:ALL  
  ssh_authorized_keys:  
  - ssh-rsa...  
```
设置vm使用该文件  
```
qm set 9000 --cicustom "user=local:snippets/cloudinit-config.yaml"  
```
注意:  
该文件并不会导入到vm当中  
如果要迁移到其他主机请确保目标机器有该config 不然vm无法启动  