# GlusterFSを使ってみた

author
:   Kazuhiro NISHIYAMA

content-source
:    関西Debian + LILO + 東海道らぐ + openSUSE 合同 LT 大会

date
:   2020/01/26

allotted-time
:   10m

theme
:   lightning-simple

# 自己紹介

- Ruby コミッター
- Twitter, GitHub: `@znz`

# GlusterFS とは?

- 分散ファイルシステム
- 複数マシンで RAID のようなことが可能
  - レプリケーション
  - ストライピング

# 選択理由

- リアルタイムにレプリケーション可能
  - rsync だと実行時のみ
- Ceph より最小必要台数が少ない
- gluster の層が壊れても最悪 brick からファイル内容を取り出せる
- Debian 公式パッケージが存在

# brick

- xfs か ext4 のサブディレクトリを brick として使う
- ファイルの実体とメタ情報が入る
- メモリ消費が少ないかもと思って ext4 を選択

# brick の注意点

- `/` と同じパーティションは非推奨
- マウントポイント直接ではなくディレクトリを作って使う
- `/data` 以下に作っていたり `/export` 以下に作っていたり
- 例: `/export/sdb1/brick`

# trusted pool 作成

- peer nodes としてどちらかから登録
- 1台で登録すれば peer 全体が相互接続される
- 例: firewall で許可してから `gluster peer probe node02`

# volume

- brick を束ねて volume を作る
- volume ごとに replica 数などを設定
- 今回は2台でレプリカ数2で構成

```
gluster volume create gv0 \
  replica 2
  10.1.2.3:/export/sdb1/brick \
  10.1.2.4:/export/sdb1/brick
```

# クライアント側

- Linux では FUSE を使って mount
- nfs と同様に基本的に IP アドレスベースの制限のみ
- 基本的には firewall で制限する方が良さそう
- NFS-Ganesha を使うと NFS 経由でも mount できるらしい (今回は未使用)

# mount

```
sudo mount -t glusterfs \
10.1.2.3,10.1.2.4:gv0 /mnt/gv0
```

/etc/fstab:
`10.1.2.3,10.1.2.4:gv0 /mnt/gv0 glusterfs defaults,_netdev 0 0`

# 感想

- しばらく使ってみて glusterfs 由来のトラブルはまだない
- ディスクイメージファイルを LUKS で暗号化していたら、
  再起動時に ro になるというトラブルはあった

# 今後の予定

- 暗号化しているせいか、大量の rsync は遅いようなので、
  パフォーマンス計測などは今後の課題
- df でディスク使用量をみてみると、片方が落ちている間のデータがレプリケーションされていなさそうだったので、
  self-healing のためには brick を分割した方が効率がいいのかどうかなどを調べたい
