本章では、前章で述べた各機能を具体的に使う方法について述べます。ファイルシステムの作成はmkfs.btrfsコマンドを使います。それ以外のほぼすべての操作にはbtrfsコマンドを使います。btrfsコマンドには目的別にサブコマンドが用意されています。

# ファイルシステムの作成

## 新規作成

mkfs.btrfs

- RAID構成
- 透過的圧縮

## ext4からの変換

> btrfs convert

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
