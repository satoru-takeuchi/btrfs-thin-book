本章では、前章で述べた各機能を具体的に使う方法について述べます。ファイルシステムの作成はmkfs.btrfsコマンドを使います。それ以外のほぼすべての操作にはbtrfsコマンドを使います。btrfsコマンドには目的別にサブコマンドが用意されています。

# ファイルシステムの作成

## 新規作成

1つのパーティションからbtrfsを作るには、次のようにします。

```
# mkfs.btrfs /dev/sda1 
btrfs-progs v4.4
See http://btrfs.wiki.kernel.org for more information.

Label:              (null)
UUID:               d5a5ac15-01bd-4747-b1b2-6d2e6726469c
Node size:          16384
Sector size:        4096
Filesystem size:    93.13GiB
Block group profiles:
  Data:             single            8.00MiB
  Metadata:         DUP               1.01GiB
  System:           DUP              12.00MiB
SSD detected:       no
Incompat features:  extref, skinny-metadata
Number of devices:  1
Devices:
   ID        SIZE  PATH
    1    93.13GiB  /dev/sda1

# 
```

複数パーティションからBtrfsを作る場合は、mkfs.btrfsにすべてのパーティションを指定します。

```
# mkfs.btrfs /dev/sda1 /dev/sda2
btrfs-progs v4.4
See http://btrfs.wiki.kernel.org for more information.

Label:              (null)
UUID:               129893fa-10b7-4bf8-ab80-528125073d76
Node size:          16384
Sector size:        4096
Filesystem size:    186.26GiB
Block group profiles:
  Data:             RAID0             2.01GiB
  Metadata:         RAID1             1.01GiB
  System:           RAID1            12.00MiB
SSD detected:       no
Incompat features:  extref, skinny-metadata
Number of devices:  2
Devices:
   ID        SIZE  PATH
    1    93.13GiB  /dev/sda1
    2    93.13GiB  /dev/sda2

# 
```

ファイルシステム作成時にRAID構成を指定できます。データについては-dオプションで、メタデータについては-mオプションで、それぞれに指定します。指定できる値はsingle,dup,raid0,raid1,raid10,raid5,raid6のいずれかです。singleはRAID構成にしないという意味です。dupは同じデバイスに2つのデータコピーを持つという意味です。dupは単一デバイスのときのみ指定できます。RAID構成のデフォルト値は、単一デバイスの場合、データはsingle,メタデータはdupです。複数デバイスの場合、データはraid0,メタデータはraid1です。

次に示すのは、2デバイス構成の場合に、データもメタデータもRAID1構成にする例です。

```
# mkfs.btrfs -d raid1 -m raid1 /dev/sda1 /dev/sda2
btrfs-progs v4.4
See http://btrfs.wiki.kernel.org for more information.

Label:              (null)
UUID:               9a686047-b2de-43d2-b1cd-c7ce95058358
Node size:          16384
Sector size:        4096
Filesystem size:    186.26GiB
Block group profiles:
  Data:             RAID1             1.01GiB
  Metadata:         RAID1             1.01GiB
  System:           RAID1            12.00MiB
SSD detected:       no
Incompat features:  extref, skinny-metadata
Number of devices:  2
Devices:
   ID        SIZE  PATH
    1    93.13GiB  /dev/sda1
    2    93.13GiB  /dev/sda2

# 
```

## ext4からの変換

> btrfs convert

# 透過的圧縮



# 状態表示

> btrfs filesystem [show|df|du]

# ストレージプールに対するデバイスの追加、削除、交換

> btrfs device [add|remove]
> btrsf replace

# バランス

> btrfs balance

# サブボリュームの操作

> btrfs subvolume

# スナップショットの作成

> btrfs subvolume snapshot

# quotaの設定

# 重複排除

> duperemove
