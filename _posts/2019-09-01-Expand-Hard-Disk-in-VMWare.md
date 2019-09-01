---
layout: post
title: ! "VMware에서 디스크 용량 확장하기 (Ubuntu)"
excerpt_separator: <!--more-->
comments : true
tags:
  - Tips
---

### Description

VMWare + Ubuntu 조합을 사용하다 보면, 미리 설정해 둔 디스크 용량이 부족해서 곤란한 경우가 있다.

VMware의 옵션으로 디스크 용량을 늘릴 수는 있지만, 이 경우 소프트웨어적으로 인식을 못하기 때문에 직접 우분투에서 설정을 변경해야 한다.

데스크톱 버전의 경우 간단하게 해결할 수 있는 GUI 소프트웨어가 있지만, 서버 버전의 경우에는 몇가지 명령어를 이용해 주어야 한다. 본 포스팅은 VMware Workstation 14 Pro 및 Ubuntu 18.04 Server 버전을 기준으로 작성하였다.


<!--more-->

### VMWare 설정

![]({{ site.baseurl }}/images/KRater/2019-09-01-Expand-Hard-Disk-in-VMWare/pic01.png)

먼저 가상 머신 설정에 들어가서 하드디스크 용량을 expand 해 준다. 여기서 나오는 메시지에서 보이듯, 가상 디스크 용량만 늘어날 뿐 파티션과 파일 시스템의 사이즈는 변화가 없다.

이번에는 포스팅을 위해서 5GB만 늘려주었다. (20GB -> 25GB)

### Ubuntu 설정

```bash
root@ubuntu_18_04:~# df -h
Filesystem      Size  Used Avail Use% Mounted on
udev            452M     0  452M   0% /dev
tmpfs            97M  1.3M   95M   2% /run
/dev/sda2        20G  4.8G   14G  26% /
tmpfs           481M     0  481M   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           481M     0  481M   0% /sys/fs/cgroup
/dev/loop0      219M  219M     0 100% /snap/microk8s/354
/dev/loop1       90M   90M     0 100% /snap/core/6130
/dev/loop2       87M   87M     0 100% /snap/core/4917
/dev/loop3      219M  219M     0 100% /snap/microk8s/383
tmpfs            97M     0   97M   0% /run/user/1000
```

설정을 바꿔주었지만 아직도 20G 그대로임을 알 수 있다. 다른 포스팅들을 보니까 fdisk, pv 등 여러가지 방법을 사용하던데 parted를 사용하는게 나에겐 직빵이었다.

```bash
root@ubuntu_18_04:~# parted /dev/sda
GNU Parted 3.2
Using /dev/sda
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) print free
Model: VMware, VMware Virtual S (scsi)
Disk /dev/sda: 26.8GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name  Flags
        17.4kB  1049kB  1031kB  Free Space
 1      1049kB  2097kB  1049kB                     bios_grub
 2      2097kB  21.5GB  21.5GB  ext4
        21.5GB  26.8GB  5370MB  Free Space

(parted)   
```

`print free` 명령어를 이용해서 Free 공간이 5GB만큼 잡혀있는걸 확인할 수 있다.

```bash
(parted) resizepart 2
Warning: Partition /dev/sda2 is being used. Are you sure you want to continue?
Yes/No? y                                                                 
End?  [26.8GB]? 27GB                                                      
(parted) print free
Model: VMware, VMware Virtual S (scsi)
Disk /dev/sda: 26.8GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name  Flags
        17.4kB  1049kB  1031kB  Free Space
 1      1049kB  2097kB  1049kB                     bios_grub
 2      2097kB  26.8GB  26.8GB  ext4

(parted)
```

`resizepart` 명령어를 이용해서 Free 공간을 전부다 먹어준다.

```bash
(parted)                                                                  
Information: You may need to update /etc/fstab.

root@ubuntu_18_04:~# resize2fs /dev/sda2
resize2fs 1.44.1 (24-Mar-2018)
Filesystem at /dev/sda2 is mounted on /; on-line resizing required
old_desc_blocks = 3, new_desc_blocks = 4
The filesystem on /dev/sda2 is now 6553083 (4k) blocks long.
```

그 뒤 parted를 빠져나와 쉘로 돌아온다. `resize2fs` 명령어를 이용해서 `/dev/sda2`를 resize해 준다.

```bash
root@ubuntu_18_04:~# df -h
Filesystem      Size  Used Avail Use% Mounted on
udev            452M     0  452M   0% /dev
tmpfs            97M  1.3M   95M   2% /run
/dev/sda2        25G  5.3G   19G  23% /
tmpfs           481M     0  481M   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           481M     0  481M   0% /sys/fs/cgroup
/dev/loop0      219M  219M     0 100% /snap/microk8s/354
/dev/loop1       90M   90M     0 100% /snap/core/6130
/dev/loop2       87M   87M     0 100% /snap/core/4917
/dev/loop3      219M  219M     0 100% /snap/microk8s/383
tmpfs            97M     0   97M   0% /run/user/1000
/dev/loop4       89M   89M     0 100% /snap/core/7396
/dev/loop5      184M  184M     0 100% /snap/microk8s/743
```

25GB로 용량이 늘어난 것을 확인할 수 있다.

### 후기

다른 포스팅들은 뭔가 복잡한 설정을 하던데 내 경우는 이정도만 해도 충분히 적용되었다. 뭔가 나중에 문제가 생길까봐 겁나지만 일단 돌아가면 되는거지 뭐 ...
