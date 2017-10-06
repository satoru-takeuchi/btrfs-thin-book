本章はBtrfs使用時に遭遇する様々なトラブルへの対処方法を逆引き式に紹介します。

# `btrfs filesystem show`を実行したら"*** Some devices missing"というメッセージが表示された。

メッセージの例を示します。

```
# btrfs filesystem show /mnt/
Label: none  uuid: b3355d36-38bf-43a0-bcda-f53ffc266da2
        Total devices 2 FS bytes used 640.00KiB
        devid    2 size 93.13GiB used 2.01GiB path /dev/sdc4
        *** Some devices missing

# 
```

RAID構成になっているBtrfsのストレージプールを構成するデバイスが故障したと考えられます。運用継続はできるものの、可及的速やかに故障したデバイスを正常なデバイスに交換する必要があります。上記の例では2つあるデバイス("Total devices 2")のうちデバイスIDが1であるデバイス(devid 1)が故障していますので、次のように別デバイスと交換します。

```
# btrfs replace start 1 /dev/sdc6 /mnt
```

# 書き込み時にENOSPCで異常終了した

他のファイルシステム同様、不要なファイルを削除して空き領域を作ってください。Btrfsにおいてはデータ、メタデータ共に複数ファイル/サブボリュームの間で共有されている可能性があるので、あるファイルを消したとしても必ずしもそのファイルのサイズぶんだけファイルシステムの空き領域が増えるとは限りません。不要なスナップショットを消したり、`btrfs filesystem du`を活用して、他のファイル/サブボリュームと共有されていないファイルを探して重点的に削除するのが効果的です。

# 空のディレクトリの削除が失敗する

恐らくディレクトリではなくサブボリュームを削除しようとしています。`btrfs subvolume list`コマンドによって削除しようとしているものがサブボリュームかどうかを確認してください。サブボリュームだった場合は`btrfs subvolume delete`サブコマンドによって削除してください。

# ストレージプールに十分な空き領域があるにも関わらず、バランス処理がENOSPCで異常終了した

ストレージプール内の空き領域(`btrfs filesystem df`によって表示される各行の"total - used"の値)がデータ、メタデータ共に十分あるにも関わらずバランス処理が異常終了することがあります。このとき`btrfs filesystem show`を実行すると、ストレージプール内のファイルシステムに予約されていない領域("size - used")がほぼ存在しないはずです。実装上の話になりますが、バランス処理は処理の途中で予約されていない領域の一部(典型的には1GB)を作業領域として使用します。この作業領域を確保できないとENOSPCEが発生するというわけです。

この問題を回避するためには、非常に不格好ですが、一時的に作業領域を確保するために他のデバイスをストレージプールに追加して、バランス処理が終われば削除するという方法が使えます。以下の例では、そのために、メモリ上に作成されたtmpfs(/tmp)上のファイルbalance-reserveというファイルをループバックマウントしています。

```
# dd if=/dev/zero of=/tmp/balance-temp bs=2G count=1
...
# losetup -f
/dev/loop1
# losetup /dev/loop1 /tmp/balance-temp
# btrfs device add /dev/loop1 /mnt
...
# btrfs balance start /mnt
...
# btrfs device remove /dev/loop1 /mnt
...
# rm /run/balance-temp
```

# "BTRFS error (device XXX): bdev YYY errs: wr 0, rd 1, flush 0, corrupt 0, gen 0"というメッセージが出る

Btrfsのストレージプールを構成するデバイスにおいて問題が発生した可能性が高いです。正常なデバイスにおいても一時的にこのような事象が発生することがあるため、一度出たくらいではそれほど気にする必要はありません。しかし、多発するようであればデバイスの交換を検討してください。

# ファイルシステムに書き込めなくなった

ファイルシステムの動作中に問題が発生したため、読み出し専用状態にフォールバックした可能性があります。この場合はdmesgに次のようなログが残ります。

```
BTRFS info (device /dev/sda2): forced readonly
```

バックアップを採取した上で、次回mount時に失敗した場合は後述の対処をしてください。

# dmesgにBtrfsに関するメッセージが出るようになった

## mount時に "BTRFS info (device XXX): The free space cache file (YYY) is invalid. skip it" というメッセージが出る

とくに対処の必要はありません。空き領域を管理するキャッシュ領域が壊れたことが原因です。Btrfsは壊れたキャッシュを自動的に廃棄、再構築します。再構築する間は空き領域の検索が必要なファイル作成、書き込みなどの各種処理の性能が劣化します。

## "BTRFS: could not find root 8" というメッセージが出る

quota機能を使っていないファイルシステムに対して出力されるものであり、とくに害はありません。最近のBtrfsにおいてはこのメッセージは出なくなっています。

# mountできなくなった

システムクラッシュの後などに次のようなメッセージと共にmountが失敗することがあります。

```
mount: wrong fs type, bad option, bad superblock on /dev/sdc4,
       missing codepage or helper program, or other error

       In some cases useful info is found in syslog - try
       dmesg | tail or so.
```

## まず確認すること

### balance中にumountした、あるいはシステムがクラッシュした後のmount時だった可能性

balance中にumountあるいはクラッシュした場合、次のmount時にbalance処理が再開されます。このとき、とくに後者がbalance処理のバグによって引き起こされた場合、balance処理の問題によってmount時に問題がが発生している可能性があります。次のようにskip_balanceオプションを指定してmountすると、この問題が避けられます。

```
# mount -o skip_balance /dev/sda2 /mnt
```

### RAIDを構成するデバイスが故障した可能性

Btrfsのストレージプールを構成するデバイスの一つを引数として`btrfs filesystem show`を実行してください。以下の例ではdevice id 1のストレージが故障してカーネルに認識されていません。

```
# btrfs filesystem show /dev/sdc4
warning, device 1 is missing
Label: none  uuid: def851dd-8329-490b-9625-6ec8c6c8a989
        Total devices 2 FS bytes used 896.00KiB
        devid    2 size 93.13GiB used 2.01GiB path /dev/sdc4
        *** Some devices missing

```

この状態であれば、mount時に`-o degraded`オプションを付加すれば、冗長性は失なった状態ではあるものの、マウントはできます。

```
# mount -o degraded /dev/sdc4 /mnt/
```

この後の対処は"`btrfs filesystem show`を実行したら"*** Some devices missing"というメッセージが表示された"という節において説明していますので、そちらをごらんください。

## その後の対処

基本的にはシステムの整合性が採れている状態に採取されたバックアップからファイルシステムを復旧してください。ここに記載するのは、バックアップをとっていない、あるいは最終バックアップ後の可能な限り最新のデータを復旧したいという場合の対処法です。この節に記載の対処は前から順番に試してください。

## mountできないファイルシステムからファイルを吸い出す

btrfs-restoreコマンドによって、ファイルシステムの中に存在するファイルを、mountすることなく吸い出せます。次に示すのは`<device name>`というデバイス上に構成されたBtrfsファイルシステムのファイルを`<path>`以下に吸い出す例です。

```
# btrfs-restore <device name> <path>

```

# mountに必要なメタデータの復元

過去のカーネルログ、およびdmesgにBtrfsに関するエラーメッセージ(典型的には行頭が"BTRFS")をもとに対処してください。

## replay, recover, log_tree という文字列を含んだバックトレースが出ている

バックトレースはたとえば以下のようなものです。

```
? replay_one_dir_item+0xb5/0xb5 [btrfs]
? walk_log_tree+0x9c/0x19d [btrfs]
? btrfs_read_fs_root_no_radix+0x169/0x1a1 [btrfs]                                                                                                                                           ? btrfs_recover_log_trees+0x195/0x29c [btrfs]
? replay_one_dir_item+0xb5/0xb5 [btrfs]                                                                                                                                                     ? btree_read_extent_buffer_pages+0x76/0xbc [btrfs]                                                                                                                                          ? open_ctree+0xff6/0x132c [btrfs]
```
これはBtrfsが内部で管理しているログツリーと呼ばれるメタデータが壊れているためマウントが失敗したことを示しています。ファイルシステムがクラッシュした直後のmountがこの理由によって失敗することがあります。壊れたログは次のコマンドによって削除できます。その後もう一度ファイルシステムのmountを試してください。

```
# btrfs-rescue zero-log /dev/sda2
```

このログツリーの削除によってどのような影響があるかについて説明します。Btrfsはメモリ上にキャッシュされたファイルシステムのデータを定期的(デフォルトでは30秒に一回)にストレージに同期させています。この同期操作のことをトランザクションコミットと呼びます。fsync()やO_SYNCで開いたファイルへの書き込みなどの同期書き込みについては、一旦ストレージ上のログツリーに保存した上で、トランザクションコミット時にデータ領域に反映させます。ログツリーを削除すると、最後のトランザクションコミットからクラッシュ時までの同期書き込みの結果を失います。

## 過去のsysylogに"chunk tree"という文字列を含むエラーが出力されている。

どのデバイスにどのデータを配置するかという情報が入っているchunk treeというメタデータが壊れている可能性があります。次のコマンドによって、ファイルシステムをスキャンしてchunk treeを復元できます。

```
# btrfs-rescue chunk-recover /dev/sdb2
```

このコマンドはストレージプールのすべてを走査するため、非常に時間がかかる恐れがあります。

## それ以外のログが出ている、あるいはとくにログが出ていない

大きく分けて次の2つの可能性が考えられます。

- スーパーブロック(後述)が壊れている
- root tree(後述)が壊れている

それぞれについて対処方法を説明します。

## スーパーブロックが壊れている場合

スーパーブロックとは、mountしようとしたストレージがBtrfsに属していることを示すと共に、各種管理情報を保持しているメタデータです。このデータにはバックアップされていますので、スーパーブロックが壊れている場合は次のコマンドによって復元できます。

```
# btrfs-rescue super-recover /dev/sda2
```

## root treeが壊れている場合

どの場所にどんなデータ/メタデータが配置されているかを管理するroot treeというメタデータが壊れている場合があります。次のコマンドを実行してください。


```
# mount -o usebackuproot /dev/sdb2 /mnt
```

root treeの内容は定期的に更新されていきますが、過去のroot treeをバックアップとして保存しています。上記コマンドは、最新のroot treeのかわりにバックアップ用のroot treeを使用してBtrfsファイルシステムをmountします。mountに成功したとしても、root treeが過去のものである以上、データも過去のものになっていることにご注意ください。

## ファイルシステムの復元(最終手段)

次のコマンドによってファイルシステムの復元を試みます。

```
# btrfs check --repair /dev/sda2
```

このコマンドは矛盾のあるデータは容赦無く削除するなどして無理矢理にでもmountできるようにするためのものであり、かつ、必ずしも成功するとは限りません。このため、他に打つ手がまったく無くなったときの最後の手段として使用してください。このコマンドの実行にはストレージプールの全走査が必要なため、ストレージプールのサイズに比例して実行時間が延びます。
