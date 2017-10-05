Btrfs使用時に遭遇する様々なトラブルへの対処方法を逆引き式に紹介

# btrfs-progsのバージョンはどれを使えばいいのか

一見カーネルバージョンに同期しているようなバージョンに見えますが、常に最新のものを使うのがベストです。古いカーネルで新しいbtrfs-progsを使うのは問題ありません。実際にはサポートの都合もあるので通常distroカーネルのものを使うことが多いと思いますが、それを使って何か問題が起きたときは、コミュニティの最新版で問題が修正されているかどうか確認するのがよいでしょう。

btrfs-progsのソースは[こちら](git://git.kernel.org/pub/scm/linux/kernel/git/kdave/btrfs-progs.git)から入手できます。

# RAIDを構成するデバイスのうち1つが認識されなくなった

btrfs-replace

# ENOSPC

# mountできない

# dmesgに出力されるメッセージ

## "BTRFS: could not find root 8"
