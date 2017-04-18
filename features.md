Btrfsは(ext4やXFSなどの)従来型ファイルシステムが提供する機能に加えて、次のような豊富な機能を提供します。

- ストレージプール: LVMなどのように、複数のデバイス(パーティション)から、一つの大きなストレージプールを構築できます
- サブボリューム: サブボリュームとは,上記ストレージプールから切り出したマウント可能な領域です
- スナップショット: ある時点のサブボリュームの内容をコピーしたものです。cpコマンドなどによるコピーよりもはるかに軽量です
- RAID: ストレージプールはRAID0を用いてデータを保護できます(RAID0,1,10,5,6をサポート)
- 障害検知/修復: ストレージからのデータの読み出し時にデータの破壊を検出できます。ストレージプールのRAID構成が、データのコピーやパリティを持つもの(1,10,5,6)である場合、正しいデータを修復できます。修復できない構成であればデータを捨ててエラーを返します。
- quota: サブボリュームごとにストレージの使用量の制限をかけられます
- 透過的圧縮: データを圧縮した状態でストレージに書き込むことによってストレージの容量、および帯域を節約可能です(CPU使用量は増えます)

上記すべての機能はアンマウントすることなく、オンラインで実現できるのも大きな特徴です。

本章では、上記の機能のそれぞれについて説明をします。

# ストレージプール


# サブボリューム


# スナップショット

# RAID

＃ 障害検知/修復

# quota

# 透過的圧縮