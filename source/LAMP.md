# LAMP環境を構築

## 作業環境
使用したPC: EPSON Endevor ST125E \
卒検室に大量に置いてあったPC。30個程あったので一つ借りてLAMP環境を構築します。

## Linuxのインストール
Ubuntu Server 22.04.1 LTSのISOファイルを[ダウンロード](https://jp.ubuntu.com/download)した後、[Rufus](https://rufus.ie/ja/)を使ってBootableUSBを作成。\
BIOSの設定でUSBからブートするように設定して、[Ubuntu公式](https://ubuntu.com/tutorials/install-ubuntu-server#1-overview)にあるインストールガイドに沿ってインストールする。パーティション構成は[ArchWiki](https://wiki.archlinux.jp/index.php/%E3%83%91%E3%83%BC%E3%83%86%E3%82%A3%E3%82%B7%E3%83%A7%E3%83%8B%E3%83%B3%E3%82%B0)を参考にした \
今回は学内ネットワークを使用するため、追加でプロキシの設定も行った。\

#### USBブート
PCを起動すると、BOISは記憶装置の先頭セクタ(MBR)を読み込む。MBRにはパーティションスキームが収められており、その中の1つのパーティションの先頭セクタ(PBS)の内容にしたがってブートローダを読み込み、OSを起動する。\
どういった仕組みでISOファイルとUSBメモリからbootableUSBにしているのかは調べられなかった。(おそらくブートセクタが関係している) 時間があるときに[Rufusのソースコード](https://github.com/pbatard/rufus)を参考に勉強してみたい。

#### 反省点
やけにインストールが遅いと思ったら、10年以上前のPCを使用していることに気づいた。より軽量なOSを検討するべきだった。\

#### 参考URL
https://pctrouble.net/storage/bootsector.html \
https://ja.wikipedia.org/wiki/%E8%BB%BD%E9%87%8FLinux%E3%83%87%E3%82%A3%E3%82%B9%E3%83%88%E3%83%AA%E3%83%93%E3%83%A5%E3%83%BC%E3%82%B7%E3%83%A7%E3%83%B3 \
https://pc-kaizen.com/difference-between-mbr-and-gpt\
hhttps://e-words.jp/w/%E3%83%96%E3%83%BC%E3%83%88%E3%82%BB%E3%82%AF%E3%82%BF.html \


