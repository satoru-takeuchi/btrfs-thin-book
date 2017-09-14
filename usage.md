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

# RAID

## 構成

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

## 復旧



# 透過的圧縮

## ファイルシステム全体

ファイルシステム内のすべてのデータについて透過的圧縮を有効にするにはmount時にcompressマウントオプションを指定します。

```
# mount -o compress /dev/sda1 /mnt/
# 
```

これ以降、アンマウントするまで、当該ファイルシステムに書き込まれたデータは透過的に圧縮されます。ここで注意点が2つあります。

- マウントオプションで-compressを有効化したからといって、ファイルシステム内の全データが圧縮されるわけではない。あくまで-compressを付けてマウントしている間に書き込んだデータのみが圧縮される
- 書き込み時にデータは必ず圧縮されるわけではありません。圧縮率が悪くなりそうだと判断されれば(例えば画像データや動画ファイル)圧縮はされません。そうではなく、圧縮率が悪かろうが圧縮させたい場合は-compressのかわりに-compress-forceを付けます。

-compressないし-comress-forceオプションに引数をつけることによって、透過的圧縮アルゴリズムを指定できます。指定できる値はzlib(デフォルト),lzo,noの3つです。lzoはzlibに比べて圧縮率が低いものの圧縮/展開速度が速いです。noはremount時に圧縮を無効化するときに使います。

次に示すのは透過的圧縮を有効にして、そのアルゴリズムをlzoにする例です。

```
# mount -o compress=lzo /dev/sda1 /mnt/
# 
```

## 個々のファイル

上述のマウントオプションに関わらず、個々のファイルに対して透過的圧縮を有効にできます。

次に示すのは/mnt/testというファイルの透過的圧縮を有効化して、圧縮アルゴリズムをzlibにする例です。


```
# touch /mnt/test
# btrfs property set /mnt/test compression zlib
# 
```

zlibの箇所をlzoにすれば圧縮アルゴリズムがLZOになり、""(空文字列)にすればファイルに対する圧縮を無効化できます。各ファイルの現在の圧縮設定は次のコマンドで確認できます。


```
# btrfs property get /mnt/test compression
compression=zlib
# 
```

> 実例

## 優先度

> mount時のフラグとpropertyのどちらが優先されるか

# 状態表示

本節では、Btrfsが持っているファイルシステムを構成するデバイスやストレージ使用量などを確認するためのコマンドを紹介します。これらコマンドはBtrfsの"マルチデバイス"および"コピーオンライト"の仕組みを反映して、少々とっつきにくいものになっていますので、ご注意ください。

## 最初の注意点: `df`コマンドの結果は信用してはいけない

Btrfsについては`df`コマンドの結果を信用しないでください。このコマンドはファイルシステムが単一デバイスであることを前提としていますので、Btrfsについてこのコマンドの実行結果を見ても確かな情報は得られません。そのかわりに、これから述べるBtrfs固有のコマンドを使ってください。

## ファイルシステムを構成するデバイスの表示

次に示すように、`btrfs filesystem show`コマンドによって、あるBtrfsファイルシステムを構成するデバイスの一覧を表示できます。

```
# btrfs filesystem show /mnt
Label: none  uuid: ...
	Total devices 2 FS bytes used 896.00KiB
	devid    1 size 93.13GiB used 2.01GiB path /dev/sdc1
	devid    2 size 93.13GiB used 2.01GiB path /dev/sdc3

```

このファイルシステムは/dev/sdc1と/dev/sdc3の2つから構成されていることがわかります。いずれのデバイスもサイズは"size"の後に表示される93.13GiBであり、かつ、Btrfsはそれらのうちそれぞれ"used"の後に表示される2.01GiBを切り出して使っています。使っているといっても、切り出した領域全てをファイルシステム内のファイルが使っているとは限りません。Btrfsがデバイスら切り出したものの、まだ実際のデータないしメタデータに割り当てられていない予約領域もこの数に含まれます。

## ファイルシステムのストレージの使用量

次に示すように、`btrfs filesystem df`コマンドによって、Btrfsがデバイスから切り出した領域のうち、実際に使用している量を確認できます。


```
# btrfs filesystem df /mnt
Data, RAID0: total=2.00GiB, used=768.00KiB
System, RAID1: total=8.00MiB, used=16.00KiB
Metadata, RAID1: total=1.00GiB, used=112.00KiB
GlobalReserve, single: total=16.00MiB, used=0.00B
```

各行の左端の列に表示されているのがデータの種類です。

- Data: 各ファイルの実データのための領域
- Metadata: 各ファイルの名前や日付などのメタデータのための領域
- System, GlobalReserve: Btrfsファイルシステム全体を管理するための特別なメタデータ。量が少ないのであまり気にする必要は無い

2番目の列に表示されているのがRAIDのタイプです。

3番目,4番目の列に表示されているtotal=の後、およびused=の後に表示されている数値が、それぞれのタイプのデータについてBtrfsがデバイスから切り出した量、および、その中で実際に使っている量です。ややこしいのですが、ここに出ている数値はRAID構成された上でのデータの論理的なサイズであり、デバイス上のサイズを示すとは限りません。表示されている数値をnとした場合の、RAIDのタイプとデバイス上のサイズの対応を次に示します。

|RAIDのタイプ|デバイス上のサイズ|
|:-----------|:-----------------|
|single|n|
|dup|2 x n|
|RAID0|n|
|RAID1|2 x n|
|RAID10|2 x n|

Btrfsは各種データを使用するときに、おおよそ次の論理で割り当てをすると考えてよいでしょう。

1. Btrfsが切り出した領域(total)からデータを割り当てる。使ったぶんだけusedが増える。空き領域(total-割り当て前のused)が足りなければ2へ
2. デバイスから領域を切り出してtotalを増やした上で1に戻る。切り出す際に、RAID1, RAID10については2つのデバイスにそれぞれnの空き容量が必要。デバイスの残り空き領域(`btrfs filesystem show`におけるsize - used)が不十分ならエラー終了する(ENOSPCを返す)。

## usage

> btrfs filesystem usage
> df の代替品

## ファイルのディスク使用量

> btrfs filesystem du

# ストレージプールに対するデバイスの追加、削除、交換

btrfs device addサブコマンドによって、ストレージプールにデバイスを追加できます。次に示すのは/dev/sda2を/mntにマウントされたBtrfsファイルシステムに追加する例です。

```
# btrfs filesystem show /mnt 
Label: none  uuid: 43e7bf1d-f6cb-41eb-b2b3-bc073d61bafd
	Total devices 1 FS bytes used 384.00KiB
	devid    1 size 93.13GiB used 2.02GiB path /dev/sda1

# btrfs device add /dev/sda2 /mnt/
# btrfs filesystem show /mnt 
Label: none  uuid: 43e7bf1d-f6cb-41eb-b2b3-bc073d61bafd
	Total devices 2 FS bytes used 384.00KiB
	devid    1 size 93.13GiB used 2.02GiB path /dev/sda1
	devid    2 size 93.13GiB used 0.00B path /dev/sda2		# /dev/sda2が追加されている

# 
```

追加したデバイス(上記の例では/dev/sda2)はほぼ空の状態なので、この後に後述のbtrfs balanceサブコマンドを実行するのが望ましいです。


btrfs device delサブコマンドによって、ストレージプールからデバイスを取り外せます。この処理は、取り外すデバイスに存在するデータをストレージプール上の他のデバイスに移動させる必要があるため、取り外すデバイスに存在していたデータの量に応じて処理の所要時間が長くなります。


次に示すのは、直前の例で追加した/dev/sda2を/mntから取り外す例です。

```
# btrfs device remove /dev/sda2 /mnt
# btrfs filesystem show /mnt
Label: none  uuid: 43e7bf1d-f6cb-41eb-b2b3-bc073d61bafd
	Total devices 1 FS bytes used 384.00KiB
	devid    1 size 93.13GiB used 2.02GiB path /dev/sda1

# 
```
btrfs replaceサブコマンドによってストレージプール内のデバイスを、ストレージプール外の他のデバイスと交換できます。この処理は、交換前のデバイスに存在するデータを交換後のデバイスに移動させる必要があるため、取り外すデバイスに存在していたデータの量に応じて処理の所要時間が長くなります。replaceはaddやremoveと異なり、開始処理(start)、進捗確認処理(status)、取り消し処理(cancel)の3つに分割されています。

次に示すのは/mnt以下に存在するBtrfsファイルシステムにおいて、ストレージプールに存在する/dev/sda2と、ストレージプール外にある/dev/sda3を交換する例です。

```
# btrfs filesystem show /mnt/
Label: none  uuid: f631e109-ad19-4fc2-a3b2-16ac7724f76f
	Total devices 2 FS bytes used 1.00GiB
	devid    1 size 93.13GiB used 2.02GiB path /dev/sda1
	devid    2 size 93.13GiB used 2.00GiB path /dev/sda2

# btrfs replace start /dev/sda2 /dev/sda3 /mnt/
# 
```

進捗は次のようにbtrfs replace statusコマンドによって確認できます。


```
# btrfs replace status /mnt/
Started on 30.May 11:03:16, finished on 30.May 11:03:28, 0 write errs, 0 uncorr. read errs	# 正常終了(finished)している
# btrfs filesystem show /mnt/
Label: none  uuid: f631e109-ad19-4fc2-a3b2-16ac7724f76f
	Total devices 2 FS bytes used 1.00GiB
	devid    1 size 93.13GiB used 2.02GiB path /dev/sda1
	devid    2 size 93.13GiB used 2.00GiB path /dev/sda3		# /dev/sda2が/dev/sda3になっている

# 
```

このコマンドはデバイスの交換が完了するまで動作し続けますが、途中でC-cを押せばコマンドから抜けられます。その後再び同じコマンドを実行すれば再度進捗を確認できます。

次に示すのは/dev/sda2と/dev/sdb3の交換処理を途中で中断する例です。


```
# btrfs replace start /dev/sda2 /dev/sda3 /mnt/
# btrfs replace cancel /mnt
# btrfs replace status /mnt/
Started on 30.May 11:09:28, canceled on 30.May 11:09:33 at 0.0%, 0 write errs, 0 uncorr. read errs	# 取り消しされた(canceled)
# btrfs fi show /mnt/
Label: none  uuid: f631e109-ad19-4fc2-a3b2-16ac7724f76f
	Total devices 2 FS bytes used 1.00GiB
	devid    1 size 93.13GiB used 2.02GiB path /dev/sda1
	devid    2 size 93.13GiB used 2.00GiB path /dev/sda2		# /dev/sda2のまま

# 
```

# バランス

> btrfs balance

# サブボリュームの操作

> btrfs subvolume

# スナップショットの作成

> btrfs subvolume snapshot

# quotaの設定

# 重複排除

> duperemove
