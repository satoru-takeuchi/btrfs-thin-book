本章では、前章で述べた各機能を具体的に使う方法について述べます。ファイルシステムの作成はmkfs.btrfsコマンドを使います。それ以外のほぼすべての操作にはbtrfsコマンドを使います。btrfsコマンドには目的別にサブコマンドが用意されています。

# ファイルシステムの作成

## 新規作成

1つのパーティションからbtrfsを作るには、次のようにします。

```
# mkfs.btrfs /dev/sda1 
btrfs-progs v4.4
See http://btrfs.wiki.kernel.org for more information.

Label:              (null)
UUID:               ...
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
UUID:               ...
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

Btrfsファイルシステム作成後は、他のファイルシステムと同様、mountコマンドを使えば利用できます。複数パーティションから成るBtrfsファイルシステムの場合、すべてのパーティションのうちのいずれか1つを指定すればよいです。

ファイルシステム作成時にRAID構成を指定できます。データについては-dオプションで、メタデータについては-mオプションで、それぞれに指定します。指定できる値はsingle,dup,raid0,raid1,raid10,raid5,raid6のいずれかです。singleはRAID構成にしないという意味です。dupは同じデバイスに2つのデータコピーを持つという意味です。dupは単一デバイスのときのみ指定できます。RAID構成のデフォルト値は、単一デバイスの場合、データはsingle,メタデータはdupです。複数デバイスの場合、データはraid0,メタデータはraid1です。

次に示すのは、2デバイス構成の場合に、データもメタデータもRAID1構成にする例です。

```
# mkfs.btrfs -d raid1 -m raid1 /dev/sda1 /dev/sda2
btrfs-progs v4.4
See http://btrfs.wiki.kernel.org for more information.

Label:              (null)
UUID:               ...
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

## ext2/3/4からBtrfsへの変換と復旧

既存のext2/3/4ファイルシステムのbtrfsへの変換ができます。気に入らなければ再びbtrfsからext2/3/4に復旧もできます。

次に示すのは、ext2/3/4が入った/dev/sda1をbtrfsに変換する例です。このとき/dev/sda1はマウントされていない状態である必要があります。

```
# blkid /dev/sda1
/dev/sda1: UUID="..." TYPE="ext4" PARTUUID="..."                         # ファイルシステムタイプはext4
# btrfs-convert /dev/sda1 
create btrfs filesystem:
	blocksize: 4096
	nodesize:  16384
	features:  extref, skinny-metadata (default)
creating btrfs metadata.
copy inodes [o] [         0/        11]
creating ext2 image file.
cleaning up system chunk.
conversion complete.
# blkid /dev/sda1
/dev/sda1: UUID="..." UUID_SUB="..." TYPE="btrfs" PARTUUID="..."         # ファイルシステムタイプはbtrfs
# 
```

この後/dev/sda1はBtfsファイルシステムとしてアクセスできます。後でext2/3/4に復旧する予定がある場合は、それまでに次のことをしてはいけません。

- ファイルシステムのトップディレクトリにあるext2_savedサブボリュームの削除。ここには変換元のext2/3/4ファイルシステムの情報が記録されています。
- 後述のbtrfs balanceコマンド

ファイルシステムの変換後にBtrfsを使ってみたものの、気に入らなければ、アンマウントしてから次のコマンドでext4に復旧できます。

```
# btrfs-convert -r /dev/sda1
rollback complete.
# blkid /dev/sda1
/dev/sda1: UUID="..." TYPE="ext4" PARTUUID="..."                 # ファイルシステムがext4に戻っている
# 
```

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
