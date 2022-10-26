# LAMP環境を構築

## 作業環境
使用したPC: EPSON Endevor ST125E \
卒検室に大量に置いてあったPC。30個程あったので一つ借りてLAMP環境を構築します。

## Linuxのインストール
Ubuntu Server 22.04.1 LTSのISOファイルを[ダウンロード](https://jp.ubuntu.com/download)した後、[Rufus](https://rufus.ie/ja/)を使ってBootableUSBを作成。\
BIOSの設定でUSBからブートするように設定して、[Ubuntu公式](https://ubuntu.com/tutorials/install-ubuntu-server#1-overview)にあるインストールガイドに沿ってインストールする。パーティション構成は[ArchWiki](https://wiki.archlinux.jp/index.php/%E3%83%91%E3%83%BC%E3%83%86%E3%82%A3%E3%82%B7%E3%83%A7%E3%83%8B%E3%83%B3%E3%82%B0)を参考にした \
今回は学内ネットワークを使用するため、追加でプロキシの設定も行った。

#### USBブート
PCを起動すると、BOISは記憶装置の先頭セクタ(MBR)を読み込む。MBRにはパーティションスキームが収められており、その中の1つのパーティションの先頭セクタ(PBS)の内容にしたがってブートローダを読み込み、OSを起動する。\
どういった仕組みでISOファイルとUSBメモリからbootableUSBにしているのかは調べられなかった。(おそらくブートセクタが関係している) 時間があるときに[Rufusのソースコード](https://github.com/pbatard/rufus)を参考に勉強してみたい。

#### プロキシ
学内では、プロキシサーバを経由しなければ外部サーバに接続ができないようになっている。卒検室にあるPCではDHCPを利用することが禁止されているので、今回はIPアドレスを固定してネットワークにアクセスした。 \
IP, ネットマスク,　ゲートウェイ, DNSを設定した。

#### 反省点
やけにインストールが遅いと思ったら、10年以上前のPCを使用していることに気づいた。より軽量なOSを検討するべきだった。

#### 参考URL
https://pctrouble.net/storage/bootsector.html \
https://ja.wikipedia.org/wiki/%E8%BB%BD%E9%87%8FLinux%E3%83%87%E3%82%A3%E3%82%B9%E3%83%88%E3%83%AA%E3%83%93%E3%83%A5%E3%83%BC%E3%82%B7%E3%83%A7%E3%83%B3 \
https://pc-kaizen.com/difference-between-mbr-and-gpt \
https://e-words.jp/w/%E3%83%96%E3%83%BC%E3%83%88%E3%82%BB%E3%82%AF%E3%82%BF.html 


## Apacheのインストール
OSをインストールしたPCにApacheをインストールします。（なんだか癖でアパッチェって言ってしまう）
[公式サイト](https://httpd.apache.org/docs/2.4/en/install.html)を参考
```Bash
wget https://dlcdn.apache.org/httpd/httpd-2.4.54.tar.gz
gzip -d httpd-2.4.54.tar.gz
tar xvf httpd-2.4.54.tar
cd httpd-2.4.54
./configure
make
make install
/bin/apaechectl -k start
```

#### gzip
`man gzip`
GzipはLempel-Zivコーディングを用いてファイルの圧縮や展開を行う。拡張子は.gz。`gzip -d`ではDecompress(展開)を行う。

#### tar
参考　[tar](https://www.gnu.org/software/tar/)
`tar xvf httpd-2.4.54.tar`
GNU tarは複数のファイルを一つのファイルに纏めるソフトウェア（アーカイブする） \
-x,　extract アーカイブを展開する。-v, verbose tarを実行しているファイルを表示する。vを増やすと詳細表示レベルが上がる。最大３。 -f, file アーカイブの内容を指定する

#### cd
`cd --help` \
指定したディレクトリにカレントディレクトリを移動させる。デフォルト（ディレクトリを指定しない）だとHOMEディレクトリに移動する。

#### ./configure
./configureを実行してソースツリーを構築する。今回はデフォルト設定で構築するためオプションはつけない。 \
実行してみたところ、エラーが発生した。
```
configure: Configuring Apache Portable Runtime library...
configure: 
checking for APR... no
configure: error: APR not found.  Please read the documentation.
```
APRが見つからないそうなので、[インストール](https://apr.apache.org/)をする
```
wget https://dlcdn.apache.org//apr/apr-1.7.0.tar.gz
gzip -d apr-1.7.0.tar.gz
tar xvf apr-1.7.0.tar
cd apr-1.7.0
./configure
```
./configureでエラーが発生した。Cのコンパイラのパスが通ってないみたい。ここで`apt install build-essentials`するのを忘れていたことに気づく。\
build-essentialsの中にAPRが含まれていないので、再度インストールを試みる。
```
./configure
make
make install
```


