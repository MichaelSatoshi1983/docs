# 基本概念

LVM 全称是 Logical Volume Manager，也就是逻辑卷管理。它是在 Linux 分区的基础上创建了一个逻辑层，**可以灵活方便的去管理存储设备。**

LVM 利用的是 Linux 系统的 device-mapper 功能来实现存储系统的虚拟化，**也就是系统分区独立于底层的硬件。**通过 LVM 可以实现物理磁盘的抽象化，并且在上面建立虚拟分区。同时可以进行在线的变更。

## 基本组成部分

### 物理卷 PV

物理卷 PV 的全称是 Physical Volume。它是一个**可供存储LVM块的设备**。比如：

- 硬盘分区
- 回环设备
- SAN
- Raid 硬盘
- LUN
- 或者一个被内核映射的设备，比如 dm-encrypt

它包含了一个特殊的 LVM 头，同时也是构建 LVM 系统的基础部分和实际硬件。

### 卷组 VG

卷组 VG 的全称是 Volume Group。它是一个**对一个或多个物理卷的集合**。在文件系统中显示为`/dev/vg_name`

### 逻辑卷 LV

逻辑卷 LV 的全称是 Logical Volume。它是**可供系统使用的最终元设备**。它们被卷组管理，同时也由卷组去创建。**由物理块构成，但本质上是虚拟分区。**可以在上面创建文件系统，在文件系统中显示为`/dev/vg_name/lv_name`

### 物理块 PE

物理块 PE 的全称是 Physical Extends。**它是一个卷组中最小的连续区域（默认为4 MiB）**，多个物理块将被分配给一个逻辑卷。你可以把它看成物理卷的一部分，这部分可以被分配给一个逻辑卷。

### 关系图

![img](https://i-blog.csdnimg.cn/blog_migrate/cc27afedeb747c36f557e7ef83e6cc5a.png)

顺序依次为：Disk -> Partition -> PV -> VG -> LV -> Filesystem，也即磁盘->分区->物理卷->卷组->逻辑卷->文件系统。

# 优缺点

## 优点

- 多块硬盘看作一块大硬盘

- 使用逻辑卷（LV），可以创建跨越众多硬盘空间的分区。

- 可以创建小的逻辑卷（LV），在空间不足时再动态调整它的大小。

- 在调整逻辑卷（LV）大小时可以不用考虑逻辑卷在硬盘上的位置，不用担心没有可用的连续空间。

- 可以在线（Online）对逻辑卷（LV）和卷组（VG）进行创建、删除、调整大小等操作。不过LVM上的文件系统也需要重新调整大小，好在某些文件系统（例如ext4）也支持在线操作。

- 无需重新启动服务，就可以将服务中用到的逻辑卷（LV）在线（online）/动态（live）迁移至别的硬盘上。

- 允许创建快照，可以保存文件系统的备份，同时使服务的下线时间（downtime）降低到最小。

- 支持各种设备映射目标（device-mapper targets），包括透明文件系统加密和缓存常用数据（caching of frequently used  data）。这将允许你创建一个包含一个或多个磁盘、并用LUKS加密的系统，使用LVM on top 可轻松地管理和调整这些独立的加密卷 （例如. /, /home, /backup等) 并免去开机时多次输入密钥的麻烦。

## 缺点

- Windows 系统不支持 LVM 
- 系统恢复可能会遇到问题 

# 使用 LVM

## 物理卷

### 创建物理卷 pvcreate

创建物理卷的时候可以使用尚未分区的磁盘或者是一整个磁盘分区。

`语法：pvcreate 设备名（分区名）（支持多个）`

```shell
	sudo pvcreate /dev/sdc
```

### 列出所有物理卷 pvscan pvs pvdisplay

#### pvscan

不需要加任何参数

#### pvs

不需要加任何参数

#### pvdisplay

不需要加任何参数

### 删除物理卷 pvremove

`语法：pvremove 物理卷名字`

```shell
	sudo pvremove /dev/sdd2
```

## 卷组

### 创建卷组 vgcreate

创建卷组之前要确保以及创建了物理卷。

`语法：vgcreate 卷组名 包含的物理卷设备（支持多个）`

```shell
	sudo vgcreate lvm_tutorial /dev/sdc /dev/sdd1
```

### 列出所有卷组 vgscan vgs vgdisplay

同列出物理卷用法

### 列出附加到卷组的物理卷 pvdisplay

`语法：pvdisplay -S 卷组名 -C -o pv_name`

```shell
	sudo pvdisplay -S vgname=lvm_tutorial -C -o 
```

#### 列出有多少物理卷 vgdisplay

`语法：vgdisplay -S 卷组名 -C -o pv_count`

```shell
	sudo vgdisplay -S vgname=lvm_tutorial -C -o 
```

### 扩展卷组 vgextend

扩展卷组就是向卷组里面添加物理卷。

`语法：vgextend 卷组名 物理卷名（支持多个）`

```shell
	sudo vgextend lvm_tutorial /dev/sdd2
```

### 减小卷组 vgreduce

减小卷组就是从卷组里面删除一个物理卷设备。

`语法：vgreduce 卷组名 物理卷名（支持多个）`

```shell
	sudo vgreduce lvm_tutorial /dev/sdc /dev/sdd1
```

### 删除卷组 vgremove

从系统里面删除这个卷组。

`语法：vgremove 卷组名`

```shell
	sudo vgremove lvm_tutorial
```

## 逻辑卷

### 创建逻辑卷 lvcreate

`语法：lvcreate -L 大小 -n 逻辑卷名 从哪个卷组分配空间`

```shell
	sudo lvcreate -L 5GB -n lv1 lvm_tutorial
```

### 格式化创建之后的卷组

就像普通分区一样去格式化即可。

### 调整逻辑卷大小 lvextend

`语法：lvresize -L [+|-]大小 卷组名/逻辑卷名`

```shell
	sudo lvresize -L +2GB lvm_tutorial/lv1
	sudo lvresize -L -2GB lvm_tutorial/lv1
```

#### 调整文件系统大小

在逻辑卷大小调整完成后，**不要忘记再调整下文件系统的大小！**

#### ext4

```shell
    # 如果是ext4，使用resize2fs命令，后面参数是分区
    sudo resize2fs /dev/sdb5
```

#### xfs

```shell
    # 如果是xfs，使用xfs_growfs命令，后面参数是挂载点
    sudo xfs_growfs /mnt/sdb5
```

### 删除逻辑卷 lvremove

`语法：lvremove 卷组名/逻辑卷名`

```shell
	sudo lvremove lvm_tutorial/lv1
```

