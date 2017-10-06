本章では、前章で述べた各機能を具体的に使う方法について述べます。Btrfsは、用途別に次のような複数のコマンドを使い分けます。

- ファイルシステムの作成: mkfs.btrfsコマンド
- ほとんどの操作: btrfsコマンド。このコマンドには様々なサブコマンドが用意されています。
- その他復旧、トラブルシューティングのための一部コマンド: btrfs-* という名前のコマンド

上記すべてはbtrfs-progsというソフトウェアに含まれています。btrfs-progsのバージョンは一見カーネルバージョンに同期しているように見えますが、常に最新のものを使うのがベストです。古いカーネルで新しいbtrfs-progsを使うのは問題ありません。実際にはサポートの都合もあるので通常distroカーネルのものを使うことが多いと思いますが、それを使って何か問題が起きたときは、コミュニティの最新版で問題が修正されているかどうか確認するのがよいでしょう。

本章では、単に上記コマンドの使い方を羅列するのではなく、何をしたいときにどの操作をすればいいかという、運用ベースでBtrfsの使い方を説明します。

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

## ファイルシステムの新規作成時

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

## 運用中のファイルシステムのRAID構成を変更する

ファイルシステムを作ったときのRAID構成を、運用中に変更できます。次に示すのは、もともとデータはRAID0構成、メタデータはRAID1構成だったBtrfsファイルシステムを、データ、メタデータ共にRAID1構成にする例です。


```
# btrfs fi df /mnt/
btrfs fi df /mnt/
Data, RAID0: total=2.00GiB, used=768.00KiB		# データは元々RAID0構成
System, RAID1: total=8.00MiB, used=16.00KiB
Metadata, RAID1: total=1.00GiB, used=112.00KiB		# メタデータは元々RAID1構成
GlobalReserve, single: total=512.00MiB, used=0.00B
# btrfs balance start -dconvert=raid1 -mconvert=raid1 /mnt	# データ、メタデータともにRAID1構成にする
Done, had to relocate 3 out of 3 chunks
# btrfs fi df /mnt/
Data, RAID1: total=1.00GiB, used=512.00KiB		# データもRAID1構成になっている
System, RAID1: total=32.00MiB, used=16.00KiB
Metadata, RAID1: total=1.00GiB, used=112.00KiB		# メタデータは引き続きRAID1構成
GlobalReserve, single: total=512.00MiB, used=0.00B
#
```

この走査はファイルシステムの使用中のデータを全走査すると共に、必要に応じて再配置するため、使用中のデータ量に応じた時間がかかります。上記の例で使用している`btrfs balance`コマンドの詳細については後述します。

# 透過的圧縮

透過的圧縮機能はファイルシステム全体、あるいはファイルごとに有効か無効かを切り替えられます。Btrfsがデータを圧縮する単位は最小4KBです。このため、VMイメージのような無圧縮かつ大きなサイズのファイルに対しては有効でしょう。その一方、小さなファイルをたくさん含むソースツリーや、サイズは大きくても圧縮されているファイル(zipファイルやtar.{g,b,x}zファイルなど)では効果が見込めないでしょう。

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

# 状態確認

本節では、Btrfsが持っているファイルシステムを構成するデバイスやストレージ使用量などを確認するための方法を紹介します。

## 注意点

Btrfsの"マルチデバイス"および"コピーオンライト"という仕組みによって、ファイルシステム状態を確認にはある程度慣れが必要です。それに関して、Btrfsにおいては`df`コマンドの結果は信用してはいけません。このコマンドはファイルシステムが単一デバイスであることを前提としていますので、Btrfsについてこのコマンドの実行結果を見ても確かな情報は得られません。そのかわりに、これから述べるBtrfs固有の方法を使ってください。

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

## 個々のファイルのストレージ使用量を確認

Btrfsは後述のsnapshotの採取やreflinkによるコピーに起因して、ファイルの一部、ないし全てを他のファイルと(場合によっては自分自身とも)共有する事がよくあります。これによって、あるファイルを削除してもストレージプールの空き容量は削除した領域より少ない量しか減らない、あるいはまったく減らないという事象が発生します。このようなときは個々のファイルが共有せずに1ファイルのみで使っている容量を知る必要があります。それをするのが`btrfs filesystem du`コマンドです。このコマンドは文書だけではわかりにくいので実例を使って説明しましょう。

次に示すのはBtrfsファイルシステム上にあるbtrfs-du-testというディレクトリの情報を出力した結果です。

```
# ls -lR btrfs-du-test
btrfs-du-test:
total 0
drwxr-xr-x 1 root root 16 Oct  5 20:48 snap1
drwxr-xr-x 1 root root 16 Oct  5 20:48 sub1
drwxr-xr-x 1 root root 16 Oct  5 20:48 sub2

btrfs-du-test/snap1:
total 1024
-rw-r--r-- 1 root root 1048576 Oct  5 20:48 testfile

btrfs-du-test/sub1:
total 1024
-rw-r--r-- 1 root root 1048576 Oct  5 20:48 testfile

btrfs-du-test/sub2:
total 1024
-rw-r--r-- 1 root root 1048576 Oct  5 20:48 testfile
```

btrfs-du-test直下にあるsub1, sub2, snap1というのはすべてサブボリュームです。sub1とsub2は独立したサブボリュームであり、それぞれ1MBのサイズを持つtestfileというファイルを持っています。snap1はこの状態のsub1から採取したスナップショットです。つまりsub1とsub2の中のtestfileは共有されていないものの、sub1とsub2の中のtestfileは共有されているということです。これを踏まえて`btrfs filesystem du`コマンドを実行してみましょう。

```
# btrfs filesystem du btrfs-du-test
     Total   Exclusive  Set shared  Filename
   1.00MiB       0.00B           -  btrfs-du-test/sub1/testfile
   1.00MiB       0.00B           -  btrfs-du-test/sub1
   1.00MiB     1.00MiB           -  btrfs-du-test/sub2/testfile
   1.00MiB     1.00MiB           -  btrfs-du-test/sub2
   1.00MiB       0.00B           -  btrfs-du-test/snap1/testfile
   1.00MiB       0.00B           -  btrfs-du-test/snap1
   3.00MiB     1.00MiB     1.00MiB  btrfs-du-test
# 
```

出力における一番上の行がヘッダ情報、その下の各行が1つのファイルを表しています。各行の中の各列の意味を次に示します。

- 1列目(Total): ファイルの合計サイズ。ディレクトリの場合はそれに属する全てのファイルのサイズを積算した値
- 2列目(Exclusive): 上記合計サイズのうち、当該ファイル以外に共有されていないデータのサイズ
- 3列目(Set shared): 上記合計サイズのうち、ほかのファイルと共有している部分のサイズ。`btrfs filesystem df`の引数に直接指定したファイルのみ表示する。その他のファイルについては"-"を表示する。
- 4列目(Filename): ファイル名

この意味と上記の出力を照らし合わせると、次のことがわかります。

- sub1, sub2, snap1ともに、各スナップショット内のファイル(testfile)のサイズは1MB
- sub1とsub2は自分自身だけで使っているデータは無い。実際sub1/testfileとsub2/testfileが指す実データは1つだけ。
- 上記共有データの合計は1MB
- sub2は自分自身だけで1MBの領域を使っている(Exclusiveが1MB)
- snapshotやreflinkによって3 - (1 + 1) = 1MBの利用削減できている。

ファイルシステムの空き容量を増やしたければ、余計なsnapshotを削除したうえで、Exclusiveの欄の値が大きいファイルから重点的に削除するとよいでしょう。

# ストレージプールに対するデバイスの追加、削除、交換

## 追加

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

追加したデバイス(上記の例では/dev/sda2)はほぼ空の状態なので、ストレージプールを構成するデバイス間にデータを均等に配分するのが望ましいです。そのために、次に述べるバランス処理を実施します。このとき、単一デバイス構成から複数デバイス構成にする場合で、かつ、RAID構成にしたいことがあります。そのときは前述の`-dconver`オプション、および`-mconvert`オプションを使うとよいでしょう。

## バランス

バランス処理は大量のI/O処理の発生によって長時間を要することがままあります。そのため、処理の開始、中断、再開、中止、および進捗確認のためのコマンドが用意されています。一つのファイルシステムにおいて同時に実行できるバランス処理は1つだけです。


次のようにすればバランス開始できます。

```
# btrfs balance start /mnt/

```

デフォルトではフォアグラウンドで動作しますが、次のようにすればバックグラウンドで実行させられます。

```
# btrfs balance start -b /mnt/
...
```

実行中のバランス処理の進捗を表示するには次のようにします。


```
btrfs balance status /mnt/
```

バランス処理を中断するには次のようにします。

```
# btrfs balance pause /mnt
```

中断したバランス処理を再開するには次のようにします。

```
# btrfs balance resume /mnt
```

実行中のバランス処理を中止したい場合は次のようにします。


```
btrfs balance cancel /mnt/
```

## 削除

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

## 交換

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

# サブボリュームの操作

あるBtrfsファイルシステムにサブボリュームを作成、削除する方法、および、サブボリュームを一覧表示する方法を示します。

サブボリュームを作成するには次のようにします。

```
# btrfs subvolume create /mnt/sub
Create subvolume '/mnt/sub'

```

作成したサブボリュームはシステムからはただのディレクトリであるかのように見えます。

```
# ls -l /mnt
total 0
drwxr-xr-x 1 root root 0 Sep 15 15:03 sub
# 
```

サブボリュームはBtrfsファイルシステム上の任意のディレクトリ上に作れます。サブボリュームの配下にサブボリュームを作ることもできます。

あるBtrfsファイルシステム内に存在するサブボリュームを一覧表示するには次のようにします。

```
# btrfs subvolume list /mnt
ID 266 gen 55 top level 5 path sub
# btrfs subvolume create /mnt/sub2
Create subvolume '/mnt/sub2'
# btrfs subvolume create /mnt/sub3
Create subvolume '/mnt/sub3'
# btrfs subvolume list /mnt
ID 266 gen 55 top level 5 path sub
ID 267 gen 56 top level 5 path sub2
ID 268 gen 57 top level 5 path sub3
# 
```

作成済みのサブボリューム、およびその中の全てのファイルを削除するには次のようにします。

```
# btrfs subvolume delete /mnt/sub
Delete subvolume (no-commit): '/mnt/sub'
```

削除しようとするサブボリュームの配下に別のサブボリュームが存在する場合、削除は失敗します。

```
# btrfs subvolume create /mnt/sub/sub2
Create subvolume '/mnt/sub/sub2'
# btrfs subvolume delete /mnt/sub/
Delete subvolume (no-commit): '/mnt/sub'
ERROR: cannot delete '/mnt/sub': Directory not empty
# btrfs sub delete /mnt/sub/sub2
Delete subvolume (no-commit): '/mnt/sub/sub2'
# btrfs sub delete /mnt/sub
Delete subvolume (no-commit): '/mnt/sub'
# 
```

サブボリュームはmdirによって削除できません。


```
# rmdir /mnt/sub/
rmdir: failed to remove '/mnt/sub/': Operation not permitted
# 
```

# スナップショットの作成

サブボリューム単位でスナップショットを採取できます。

```
# btrfs subvolume list /mnt
ID 266 gen 61 top level 5 path sub
# ls -l /mnt/sub
total 0
# echo hello >/mnt/sub/test
# btrfs subvolume snapshot /mnt/sub /mnt/snap
Create a snapshot of '/mnt/sub' in '/mnt/snap'
# 
```

スナップショットの中身は採取元のサブボリュームと同じです。

```
# ls -l /mnt/snap
total 4
-rw-r--r-- 1 root root 6 Sep 15 15:31 test
# cat /mnt/snap/test
hello
# 
```

`btrfs subvolume snapshot`コマンドに`-r`オプションを付与すると、読み出し専用スナップショットを採取できます。

```
# btrfs subvolume delete /mnt/snap
Delete subvolume (no-commit): '/mnt/snap'
# btrfs subvolume snapshot -r /mnt/sub /mnt/snap
Create a readonly snapshot of '/mnt/sub' in '/mnt/snap'
# 
```

採取したスナップショットはサブボリュームと同様に扱えます。`btrfs subvolume list`によって、採取したスナップショットが出てきますし、`btrfs subvolume delete`で削除できます。

```
# btrfs subvolume list /mnt/
ID 266 gen 58 top level 5 path sub
ID 269 gen 58 top level 5 path snap
# btrfs subvolume delete /mnt/snap
Delete subvolume (no-commit): '/mnt/snap'
# 
```

ここまで見た範囲では`cp -r`との違いが見えません。スナップショットは採取元となるサブボリューム内のデータ量が多い場合に威力を発揮します。サブボリュームを`cp -r`でコピーする場合とスナップショットを取る場合とで、採取時間とsync後のストレージの使用量を比較してみましょう。ここではlinuxのgitリポジトリを持つサブボリュームをコピー元およびスナップショット元にします。


```
# btrfs subvolume list /mnt
ID 258 gen 9 top level 5 path sub
# ls -l /mnt/sub
total 0
drwxrwxr-x 1 sat sat 3386 Aug 31 12:04 linux
# du -h --summarize /mnt/sub/linux
4.5G	/mnt/sub/linux
# btrfs filesystem df /mnt
Data, RAID0: total=6.00GiB, used=4.32GiB
System, RAID1: total=8.00MiB, used=16.00KiB
Metadata, RAID1: total=1.00GiB, used=85.95MiB
GlobalReserve, single: total=32.00MiB, used=0.00B
```

まずはコピーの場合です。


```
# time cp -r /mnt/sub /mnt/cp

real	0m20.668s
user	0m0.096s
sys	0m2.788s
# sync
# btrfs filesystem df /mnt
Data, RAID0: total=10.00GiB, used=8.65GiB
System, RAID1: total=8.00MiB, used=16.00KiB
Metadata, RAID1: total=1.00GiB, used=171.23MiB
GlobalReserve, single: total=64.00MiB, used=0.00B
# 
```

終了までに20.6秒かかりました。増加したデータ使用量、およびメタデータの使用量はそれぞれ4.32GBと85.2MBです。

続いてスナップショットの場合です。


```
# time btrfs subvolume snapshot /mnt/sub /mnt/snap
Create a snapshot of '/mnt/sub' in '/mnt/snap'

real	0m0.157s
user	0m0.000s
sys		0m0.004s
# sync
# btrfs filesystem df /mnt/
Data, RAID0: total=10.00GiB, used=8.65GiB
System, RAID1: total=8.00MiB, used=16.00KiB
Metadata, RAID1: total=1.00GiB, used=171.25MiB
GlobalReserve, single: total=64.00MiB, used=0.00B
 

```

所要時間は0.157秒でした。増加したデータとメタデータの量はそれぞれ0, および0.02MBでした。

両者の違いを表にまとめてみましょう。

||コピー|スナップショット|
|:-------|:---------|:----------|
|所要時間|20.6s|0.157s|
|データ増加量|4.32GB|0|
|メタデータ増加量|85.2MB|0.02MB|

すべてにおいてスナップショットのほうが、はるかに良い値になっていることがわかります。これは以前に述べたように、スナップショットの"データはコピーせず、メタデータ(の中の必要なものだけ)のみをコピーする"という特徴によるものです。

最後にスナップショットの注意点を挙げておきます。

- スナップショット採取後にスナップショット、あるいスナップショットは採取元のサブボリューム上に書き込みをすると、書き込んだ領域がコピーオンライトの仕組みによって共有解除され、ストレージプールから新たなデータ、およびメタデータ領域を獲得します。このためBtrfsでは「既存ファイルへの書き込みしかしていないのにデータ使用量が増えていく」という事象が発生します。
- スナップショット採取時にはスナップショット上のデータに対応するダーティキャッシュ(ファイルへの変更がストレージに反映されていない領域)を全てストレージにライトバックされます。ダーティキャッシュが存在しない場合は上記のようにミリ秒オーダーでスナップショット採取が終わりますが、ダーティキャッシュが存在する場合は、その量に応じてスナップショット採取の所要時間は増加します。
- スナップショット採取時に、採取元のサブボリュームの中にさらに別のサブボリュームが入れ子になっている場合(このサブボリュームを便宜的に内部サブボリュームと表記します)、内部のサブボリュームのスナップショットは採取しません。採取したスナップショットの中の、内部サブボリュームに対応する部分は空ディレクトリになります。

# ファイル単位のスナップショット採取

Btrfsはファイルのコピー時に、コピー元のデータをコピー先にフルコピーすることなく、メタデータだけをコピーしてコピー先のメタデータからコピー元のデータを参照できます。ファイルシステムのスナップショットともいえる機能です。この機能は次のようにして使います。

```
$ cp --reflink a b
```

reflinkはデータコピーが不要なぶん、通常のcpコマンドよりもはるかに高速です。reflinkによるコピーはディレクトリに対して再帰的に採取もできます。

ならばサブボリューム単位のスナップショットは不要ではないかと思われるかもしませんが、それは違います。reflinkの場合は後述の高度なバックアップ/リストアに必要なsend/receive機能が使えない

# バックアップとリストア

## 通常の方法とBtrfsならではの方法の比較

BtrfsはXFSやext4において通常使われる方法に比べてファイルシステムを使う処理の中断時間が短い方法によるバックアップができます。両者の比較によって、Btrfsのバックアップの優位性について説明します。

XFSやext4において使われるバックアップの手順は次の通りです。

1. バックアップ元の領域へのアクセスを含む処理を中断して、この領域を更新しないことを保証した静止状態と呼ばれる状態にする。こうしないと後述のデータコピー中の更新によって、アプリにとって整合性のあるデータが採取できない
2. バックアップ元の領域をバックアップ用のファイルシステムにコピーする。
3. ファイルシステムの静止状態を解除して、手順1で中断した処理を再開する

この場合、2においてバックアップ用領域のフルコピーする必要があるため、バックアップ元領域のデータ量に比例して、処理の中断時間が長くなります。

これに対してBtrfsにおけるスナップショットを使うバックアップの場合の手順は次の通りです。

1. ファイルシステムを静止状態にする
2. バックアップ元のサブボリュームのスナップショットを採取する
3. ファイルシステムの静止状態を解除して、1において中断した処理を再開する
4. スナップショットのデータをバックアップ用のファイルシステムにコピーする。

この場合、バックアップ元のサブボリュームを静止状態にするのはスナップショット採取時のみです。上述の通りこの操作は最短ではmsオーダーの時間で終了します。手順4においてはバックアップ元のサブボリュームをいくら更新してもスナップショットのデータの中身は変化しませんので、バックアップ元サブボリュームを使用しながら、スナップショットのデータをバックアップ先にコピーできます。

## ファイルシステムの内部構造を意識したバックアップリストア

バックアップ元のデータをバックアップ先のファイルシステムにコピーする際は、Btrfsの内部構造を意識した上で動作する`btrfs {send,receive}`というコマンドを利用できます。このコマンドには次のような特徴があります。

- cpによるコピーよりも所要時間が短い
- 差分バックアップ、増分バックアップができる

なお、send/receiveによってコピーできるサブボリュームは読み出し専用のものだけです。典型的には上記手順2における`btrfs subvolume snapshot`コマンドを`-r`オプションを付けることによって作成します。既存の読み書き可能なサブボリュームを読み出し専用にするには次のようにします。

```
# btrfs property set test-subvolume ro true
```

再度読み書き可能モードにしたければ次のようにします。

```
# btrfs property set test-subvolume ro false
```

## フルバックアップ

send/receiveによるフルバックアップは次のように使います。

```
# ls -lR test-subvolume		# バックアップ元サブボリュームの中身を確認
test-subvolume:
total 1024
-rw-r--r-- 1 root root 1048576 Oct  6 06:44 testfile

# btrfs subvolume snapshot -r test-subvolume test-snapshot	# test-subvolumeの読み出し専用スナップショット(test-snapshot)を作成
Create a readonly snapshot of 'test-subvolume' in './test-snapshot'
# btrfs send test-snapshot | btrfs receive /mnt/		# test-subvolumeが存在するのとは別のファイルシステム(/mnt)にtest-snapshotをコピー
At subvol test-snapshot
At subvol test-snapshot
# ls -lR /mnt/test-snapshot					# コピーされたsnapshotの確認
/mnt/test-snapshot:
total 1024
-rw-r--r-- 1 root root 1048576 Oct  6 06:44 testfile
# 
```

`btrfs receive`のターゲットである/mntはBtrfsでなくてはいけません。

receiveによって受け取ったサブボリュームを使って/mntからbackup元のBtrfsファイルシステムに対して(バックアップとは逆方向の)send/receiveすれば、バックアップからのリストアができます。

`btrfs-send`の出力はサブボリュームとして復元するのではなく、ファイルへの保存もできます。

```
# btrfs send test-snapshot | xz - >/mnt/test-snapshot.xz
At subvol test-snapshot
# 
```

これについては/mntはBtrfsでなくても構いません。このファイルを展開してBtrfsファイルシステム上でreceiveすればリストアできます。

## 差分/増分バックアップ

send/receiveには、あるサブボリュームについて、以前にコピーしたサブボリュームのデータからのの差分だけをコピーする機能があります。前節で例示した、test-subvolumeのバックアップを/mnt/test-snapshotに採取した処理の続きとして、差分バックアップを採取する例を示します。

```
# dd if=/dev/zero of=test-subvolume/testfile2 bs=1M count=1		# バックアップ後のtest-subvolumeに、新たにtestfile2というファイルを作成する
1+0 records in
1+0 records out
1048576 bytes (1.0 MB, 1.0 MiB) copied, 0.000691068 s, 1.5 GB/s
# btrfs subvolume snapshot -r test-subvolume test-snapshot2		# test-subvolumeのスナップショットを作成。このときtest-subvolume/test-snapshotとtest-snapshot2の間にはtestfile2が新に作成されているという差分がある
Create a readonly snapshot of 'test-subvolume' in './test-snapshot2'
# btrfs send -p test-snapshot test-snapshot2 | btrfs receive /mnt	# test-snapshot2のコピー時に親スナップショットの(`-p`オプションによる)指定によって、両者の差分(testfile2)だけをコピー
At subvol test-snapshot2
At snapshot test-snapshot2
# ls -lR /mnt/test-snapshot2
/mnt/test-snapshot2:
total 2048
-rw-r--r-- 1 root root 1048576 Oct  6 06:44 testfile
-rw-r--r-- 1 root root 1048576 Oct  6 07:08 testfile2
```

sendにおいて`-p`によって指定された親サブボリュームはreceive時のターゲットとなるディレクトリ以下に同じ名前で存在している必要があります。

この後test-subvolumeを親サブボリュームとしてsend/receiveを繰り返すと差分バックアップが実現できます。その一方、常に一つ前にコピーしたスナップショットを親サブボリュームとしてsend/receiveを繰り返すと増分バックアップが実現できます。

# quotaの設定

# 重複排除

> duperemove

# データ破壊の明示的な検出と修復

Btrfsがデータ破壊を検出するのは通常ストレージからデータを読み出すときです。そのときに読み出したデータが破壊されていればI/Oエラーが発生するか、あるいはRAID構成によっては正しいデータに修復します。それだけではなく、scrubと呼ばれる処理によって、読み出しを待たずにファイルシステムの走査によるデータ破壊の検出/修復もできます。このコマンドはファイルシステムの全データを走査するため、ストレージプール内のファイルシステムが使用している量に応じて時間がかかります。データ破壊を可能な限り早く検出をしたいかたは、cronなどによって定期的にscrubをするとよいでしょう。

次のようにすれば処理を開始できます。


```
# btrfs scrub start /mnt
```

処理の進捗は次のように確認します。

```
# btrfs scrub status /mnt
```

処理を途中で中断するには次のようにします。

```
# btrfs scrub cancel /mnt

```

中断した状態から再開するには次のようにします。

```
# btrfs scrub resume /mnt
```
