Btrfs使用時に遭遇する様々なトラブルへの対処方法を逆引き式に紹介

btrfs-progsのソースは[こちら](git://git.kernel.org/pub/scm/linux/kernel/git/kdave/btrfs-progs.git)から入手できます。

# RAIDを構成するデバイスのうち1つが認識されなくなった

btrfs-replace

# 書き込み時にENOSPCで異常終了した

不要なファイルを削除。データ、メタデータ共に複数ファイル/サブボリュームで共有されている可能性があるので、duコマンドの結果をもとに共有されていないファイルを探すのが効果的。

# 空のディレクトリを削除できない

それはsubvolume。subvolume deleteサブコマンドによって削除が必要

# ストレージプールに十分な空き領域があるにも関わらず、btrfs balanceがENOSPCで異常終了した

# 特定のストレージに関してI/Oエラーが多発するようになった

バックアップした上で、btrfs replace

# ファイルシステムに書き込めなくなった

Btrfsに問題が発生してromountになっている。dmesgを見て

# mount時に自動的に動作するbalanceに失敗してmountできない、あるいはシステムがクラッシュする

skip_balanceマウントオプション

# mountできなくなった

dmesgにBtrfsに関するエラーメッセージが表示されているはずです(典型的には行頭が"BTRFS")。メッセージに応じた対策をとってください。
それでもダメならバックアップからデータを復元。

バックアップが存在しない、なるべく新しいデータを復旧したいという場合はbtrfs-restore。

最終手段。btrfs check --repair

# dmesgにBtrfsに関するメッセージが出るようになった

## space cacheが壊れた

空き領域を管理するキャッシュ領域が壊れたのが原因。壊れたキャッシュは自動的に廃棄、再構築されるので、とくに対処の必要はない。再構築する間はデータ/メタデータ割り当てが必要なファイル作成、書き込みなどの各種処理の性能劣化する。

## "BTRFS: could not find root 8" というメッセージが出る

quota機能を使っていないファイルシステムに対して出力されるものであり、とくに害はありません。最近のBtrfsにおいてはこのメッセージは出なくなっています。

## chunkがどうこう言っている

btrfs-rescue chunk-recovery

## superblockが見つからないと言っている

btrfs-rescue chunk-super

## logがどういう言っている

btrfs-rescue zero-log

## bad tree rootがどうこう言っている

mount -o usebackuproot
