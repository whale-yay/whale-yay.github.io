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

#### OSをインストールする際の参考URL
https://pctrouble.net/storage/bootsector.html \
https://ja.wikipedia.org/wiki/%E8%BB%BD%E9%87%8FLinux%E3%83%87%E3%82%A3%E3%82%B9%E3%83%88%E3%83%AA%E3%83%93%E3%83%A5%E3%83%BC%E3%82%B7%E3%83%A7%E3%83%B3 \
https://pc-kaizen.com/difference-between-mbr-and-gpt \
https://e-words.jp/w/%E3%83%96%E3%83%BC%E3%83%88%E3%82%BB%E3%82%AF%E3%82%BF.html 

#### タイムゾーン
`man timedatactl`を参考に設定をすすめる。
```
$ timedatectl status
              Local time: Thu 2022-10-27 05:06:17 UTC
           Universal time: Thu 2022-10-27 05:06:17 UTC
                 RTC time: Thu 2022-10-27 05:06:17
                Time zone: Etc/UTC (UTC, +0000)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no
```
タイムゾーンがEtc/UTCになっているので、JSTに変更する
```
$ timedatectl list-timezones | grep J
$ timedatectl set-timezone Japan
$ timedatectl status
```
リストにJapanがあることを確認し、タイムゾーンをセット。`timedatectl status`でTime zoneがJSTになっていることを確認する 

#### NTPサーバと時刻同期を行う
`man timedatectl`を参考
```
$ timedatectl timesync-status
```
Serverがntp.ubuntu.comになっているのでntp1.jst.mfeed.ad.jpに変更する 

`man timedatectl`にある`timesync-status`には`Show current status of systemd-timesyncd.service`と書いてある。
```
$ man systemd-timesyncd.service
DESCRIPTION
~~~略~~~
The NTP servers contacted are determined from the global settings in timesyncd.conf(5),
```
timesyncd.confに設定が書いている様ですが、ファイルの場所がわからないので検索する
```
$ man find
$ sudo find / -name "timesyncd.conf"
$ cat /etc/systemd/timesyncd.conf
~~~略~~~
[Time]
#NTP=
#FallbackNTP=ntp.ubuntu.com
~~~
```
ファイルを編集してNTPサーバを設定したあと、設定を反映させるためsystemd-timesyncd.serviceを再起動する。 
```
$ vi /etc/systemd/timesyncd.conf
[Time]
NTP=ntp1.jst.mfeed.ad.jp
$ timedatectl set-ntp false
$ timedatectl set-ntp true
$ timedatectl timesync-status
Server: 210.173.160.27 (ntp1.jst.mfeed.ad.jp)
```
タイムゾーンをJST、NTPサーバをntp1.jst.mfeed.ad.jpに設定することができた。

#### ファイヤウォール
`$ man ufw`を参考にファイヤウォールを有効にする
```
$ ufw status
ERROR: You need to be root to run this script
```
管理者権限で再実行する
```
$ sudo ufw status
Status: inactive
```
非アクティブ状態を確認して、アクティブにする。
```
$ sudo ufw enable
```
`$ man ufw`より、`enable reloads fire wall and enables firewall on boot`とあるため、次回以降の起動では必要なさそう \

手元のPCからSSHでサーバ機に接続していたが、
```
$ ssh **ip**
ssh: connect to host **ip** port 22: Connecttion timed out
```
タイムアウトするようになったので、22番ポートを開放
```
$ sudo ufw allow 22
```
#### SELinuxを無効

## Apacheのインストール
OSをインストールしたPCにApacheをインストールします。（なんだか癖でアパッチェって言ってしまう） \
[公式サイト](https://httpd.apache.org/docs/2.4/en/install.html)を参考
```Bash
$ wget https://dlcdn.apache.org/httpd/httpd-2.4.54.tar.gz
$ gzip -d httpd-2.4.54.tar.gz
$ tar xvf httpd-2.4.54.tar
$ cd httpd-2.4.54
$ ./configure
$ make
$ make install
$ /bin/apaechectl -k start
```

#### gzip
`man gzip` \
GzipはLempel-Zivコーディングを用いてファイルの圧縮や展開を行う。拡張子は.gz。`gzip -d`ではDecompress(展開)を行う。

#### tar
参考　[tar公式リファレンス](https://www.gnu.org/software/tar/) \
```
$ tar xvf httpd-2.4.54.tar
```
GNU tarは複数のファイルを一つのファイルに纏めるソフトウェア（アーカイブする） \
-x,　extract アーカイブを展開する。-v, verbose tarを実行しているファイルを表示する。vを増やすと詳細表示レベルが上がる。最大３。 -f, file アーカイブの内容を指定する

#### cd
`$ cd --help` \
指定したディレクトリにカレントディレクトリを移動させる。デフォルト（ディレクトリを指定しない）だとHOMEディレクトリに移動する。

#### Apacheのビルド
./configureを実行してソースツリーを構築する。今回はデフォルト設定で構築するためオプションはつけない。 \
実行してみたところ、エラーが発生した。
```
configure: Configuring Apache Portable Runtime library...
configure: 
checking for APR... no
configure: error: APR not found.  Please read the documentation.
```
APRが見つからないそうなので、[インストール](https://apr.apache.org/)する
```
$ wget https://dlcdn.apache.org//apr/apr-1.7.0.tar.gz
$ gzip -d apr-1.7.0.tar.gz
$ tar xvf apr-1.7.0.tar
$ cd apr-1.7.0
$ ./configure
```
./configureでエラーが発生した。Cのコンパイラのパスが通ってないようです。Cがインストールここで`apt install build-essentials`するのを忘れていたことに気づく。\
build-essentialsの中にAPRが含まれていないので、再度インストールを試みる。
```
$ ./configure
$ make
$ make install
```
`$ make install`でエラーが発生した。Permission deniedなのでsudoをつけて再実行する。 
```
$ sudo make install
```
APRのインストールが完了したので、apacheのconfigureを試みた 
```
$ cd ~/httpd-2.4.54
$ ./configure 
~~~略~~~~
configure: error: ARP-util not found. Please read the documentation
```

次はAPR-utilが無いと言われたので、こちらも[インストール](https://apr.apache.org/download.cgi)する 
```
$ wget https://dlcdn.apache.org//apr/apr-util-1.6.1.tar.gz
$ gzip -d apr-util-1.6.1.tar.gz
$ tar xvf apr-util-1.6.1.tar
$ cd apr-util-1.6.1
$ ./configure
~~~略~~~
configure: error: ARP could not be located. Please use the --with-apr option
```
APR-utilをconfigureする際にエラーが発生した. \
`./configure --help`からオプションの使い方を調べ、apr-configへのフルパスを指定して再実行 
```
$ ./configre --with-apr=/usr/local/apr/bin/apr-1-config
$ make
~~~略~~~
xml/apr_xml.c:35:10: fatal error: expat.h: No such file or directory
```
ARP-utilのmakeを行う際にエラーが発生した。[expatをインストール](https://github.com/libexpat/libexpat/releases)する
```
$ wget https://github.com/libexpat/libexpat/releases/download/R_2_5_0/expat-2.5.0.tar.gz
$ gzip -d expat-2.5.0.tar.gz
$ tar xvf expat-2.5.0.tar
$ cd expat-2.5.0
```
[公式リポジトリ](https://github.com/libexpat/libexpat)を元にインストールを進める
```
$ ./configure
$ make
$ sudo make install
```
expatのインストールが完了したので再度APR-utilをビルドする
```
$ cd ~/apr-util1.6.1
$ make
$ sudo make install
```
またまたApacheのConfigureを試みる
```
$ cd ~/httpd-2.5.54/
$ ./configure
~~~略~~~
configure: error: pcre(2)-config for libpcre not found. PCRE is required and available from http://pcre.org/
```
またもやエラーが発生する。pcreが無いみたいなので、エラーにある[サイト](http://pcre.org/)からインストールする \
pcreの[github](https://github.com/PCRE2Project/pcre2)にも[公式リファレンス](http://pcre.org/current/doc/html/)にもインストールの方法が記載されていない。 \
[README](http://pcre.org/readme.txt)の`Buliding PCRE2 using autotools`に載っていたため、実行する。
```
$ cd ~
$ wget https://github.com/PCRE2Project/pcre2/releases/download/pcre2-10.40/pcre2-10.40.tar.gz
$ gzip -d pcre2-10.40.tar.gz
$ tar xvf pcre2-10.40.tar
$ cd pcre2-10.40
$ ./configure
$ make
$ make check
$ sudo make install
$ pcre2-config --version
10.40
```
PCREのインストールが完了したので、ApacheのConfigureを試みる.
```
$ cd ~/httpd-2.4.54
$ ./configure
$ make
$ sudo make install
```
Apacheのインストールが完了！！！ \
#### しっかりApacheがインストールされたか確認する
```
$ /usr/local/apache2/bin/apachectl -k start
AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 127.0.1.1. Set the 'ServerName' directive globally to suppress this message
(13)Permission denied: AH00072: make_sock: could not bind to address [::]:80
(13)Permission denied: AH00072: make_sock: could not bind to address 0.0.0.0:80
no listening sockets available, shutting down
AH00015: Unable to open logs
$ sudo /usr/local/apache2/bin/apachectl -k start
AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 127.0.1.1. Set the 'ServerName' directive globally to suppress this message
```
`ServerName`が設定されていないが、127.0.1.1を代わりに使用しているので試してみる
```
$ curl 127.0.1.1
<html><body><h1>It works!</h1></body></html>
```
正常にサーバが起動しているみたいです。

#### Apacheのvirtualhostsを有効化する
/usr/local/apache2/conf/httpd.confを編集する
```
$ cat /usr/local/apache2/conf/httpd.conf
# Virtual hosts
Include conf/extra/httpd-vhosts.conf
```
インクルードしている.confファイルも編集する。 ファイル内に記載がある[リファレンス](http://httpd.apache.org/docs/2.4/en/vhosts/)を参考に設定する。（日本語版は情報が古いらしいので英語版）
```
$ cat /usr/local/apache2/conf/extra/httpd-vhosts.conf
<VirtualHost 127.0.1.2:80>
    ServerName tomishima-hbtask.local
    DocumentRoot "/usr/local/apache2/htdocs/tomishima"
</VirtualHost>

<VirtualHost 127.0.1.3:80>
    ServerName yusuke-hbtask.local
    DocumentRoot "/usr/local/apache2/htdocs/yusuke"
</VirtualHost>
```
DocumentRootに設定したファイルを作成して、Apacheを再起動させる
```
$ sudo vi /usr/local/apache2/docs/tomishima.html
$ sudo vi /usr/local/apache2/docs/yusuke.html
$ sudo /usr/local/apache2/bin/apachectl -k stop
$ sudo /usr/local/apache2/bin/apachectl -k start
AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 127.0.1.1. Set the 'ServerName' directive globally to suppress this message
$ curl 127.0.1.1
<html><body><h1>It works!</h1></body></html>
$ curl 127.0.1.2
<html><body><h1>こんにちは。ありがとう。</h1></body></html>
$ curl 127.0.1.3
<html><body><h1>ありがとう。こんにちは。</h1></body></html>
```
メインサーバもVirtualHostもうまく動作しているみたいなので、つぎは名前解決がしたいです。 \ 
Webに公開しないので、ローカルで`/etc/hosts`ファイルを編集します。[ubuntuマニュアル](https://manpages.ubuntu.com/manpages/impish/ja/man5/hosts.5.html)
```
$ cat /etc/hosts
127.0.0.1 localhost
127.0.1.2 tomishima-hbtask.local
127.0.1.3 yusuke-hbtask.local

$ curl yusuke-hbtask.local
<html><body><h1>ありがとう。こんにちは。</h1></body></html>
$ curl tomishima-hbtask.local
<html><body><h1>こんにちは。ありがとう。</h1></body></html>
```

#### Apacheを自動起動させる
`man systemctl`を参考
enableするためにunitを作成する必要があるので、`man systemd.unit`を参考に作成する。 \
Unit file load path tableからunitファイルを格納する場所を検討する。今回は`/etc/systemd/system`が良さそう。 \
他の.serviceと比較しながらhttpd.serviceを記述していく(`systemd.unit`と`systemd.service`が参考になった)
```
$ cat /etc/systemd/system/httpd.service
[Unit]
Description=Apache2 server

[Service]
Type=simple
ExecStart=/usr/local/apache2/bin/apachectl -k start
ExecStop=/usr/local/apache2/bin/apachectl -k stop

[Install]
WantedBy=multi-user.target
```
早速systemctlで起動してみる。
```
$ systemctl start httpd.service
$ curl 127.0.1.3
curl: (7) Failed to connect to yusuke-hbtask.local port 80 after 0 ms: Connection refused
$ systemctl status httpd.service
Oct 31 18:16:46 hogetaro systemd[1]: Started Apache2 server.
Oct 31 18:16:46 hogetaro apachectl[99861]: AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 127.0.1.1. Set the 'ServerName' directive globally to suppress this message
Oct 31 18:16:46 hogetaro apachectl[99939]: AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 127.0.1.1. Set the 'ServerName' directive globally to suppress this message
Oct 31 18:16:46 hogetaro systemd[1]: httpd.service: Deactivated successfully.
```
/usr/local/apache/conf/httpd.confにServerNameを設定してもう一度起動を試みる。
```
$ cat /usr/local/apache2/conf/httpd.conf
~~~
ServerName localhost:80
~~~
$ systemctl start httpd.service
$ systemctl status httpd.service
○ httpd.service - Apache2 server
     Loaded: loaded (/etc/systemd/system/httpd.service; disabled; vendor preset: enabled)
     Active: inactive (dead)
Oct 31 18:16:46 systemd[1]: httpd.service: Deactivated successfully.
Oct 31 18:20:49 systemd[1]: Started Apache2 server.
Oct 31 18:20:49 systemd[1]: httpd.service: Deactivated successfully.
```
`httpd.service: DEACTIVATED successfully.`しがうまくいかない。 \
起動はうまく行えているが、サーバが閉じてしまっている。`RemainAfterExit=yes`をhttpd.serviceに追記した。
```
$ sudo systemctl start httpd.service
$ curl 127.0.1.3
<html><body><h1>ありがとう。こんにちは。</h1></body></html>
$ sudo systemctl stop httpd.service
$ curl 127.0.1.3
curl: (7) Failed to connect to 127.0.1.3 port 80 after 0 ms: Connection refused
$ sudo systemctl enable httpd.service
Created symlink /etc/systemd/system/multi-user.target.wants/httpd.service → /etc/systemd/system/httpd.service.
```
サーバの起動時にapacheが起動するようフックを作成することができた。 \
### memo
./configureはGNUのautoconfに沿って書かれている。 \
`systemctl enable`で設定ファイルの変更を反映したいときは--nowか`systemctl start`を使う \
`systemsctl enable`はhookを作成するだけ(だいだいブート時)だが、`systemctl start`は実際にデーモンプロセスが生成される。 \
systemd.serviceの`WantedBy=multi-user.target`に関しては`man systemd.service`のExampleによく出ていたターゲットを指定しただけなので、他の`gretty.target`等との違いがわからない。要調査

## PHP インストール
[PHP公式](https://www.php.net/manual/ja/install.unix.apache2.php)に則ってインストールを進める。 \
[PHPのソース](https://www.php.net/downloads.php)をダウンロード

```
$ wget https://www.php.net/distributions/php-7.4.32.tar.gz
$ gzip php-7.4.32.tar.gz
$ tar xvf php-7.4.32.tar
$ cd php-7.4.32
$ ./configure --with-apxs2=/usr/local/apache2/bin/apxs --with-pdo-mysql
configure: error: in `/home/hogetaro/php-7.4.32':
configure: error: The pkg-config script could not be found or is too old.  Make sure it
is in your PATH or set the PKG_CONFIG environment variable to the full
path to pkg-config.

Alternatively, you may set the environment variables LIBXML_CFLAGS
and LIBXML_LIBS to avoid the need to call pkg-config.
See the pkg-config man page for more details.

To get pkg-config, see <http://pkg-config.freedesktop.org/>.
$ man pkg-config
No manual entry for pkg-config
```
pkg-configが無い、またはパスが通っていないらしい。man pkg-configをしてみたが見当たらないので、エラーメッセージの[URL](https://gitlab.freedesktop.org/pkg-config/pkg-config/-/blob/master/README)からダウンロードする
```
$ wget https://pkgconfig.freedesktop.org/releases/pkg-config-0.29.2.tar.gz
$ gzip pkg-config-0.29.2.tar.gz 
$ tar xvf pkg-config-0.29.2.tar 
```
[githubのREADME](https://gitlab.freedesktop.org/pkg-config/pkg-config/-/blob/master/README)を読みながらインストールをすすめる \
pkg-configのビルドはglibに依存していてglibのビルドはpkg-configに依存しているらしい。本来は環境変数をセットしたりいろいろするらしいが、面倒な場合は `--with-internal-glib`で最新版のglibを指定できる。（pkg-config内にバンドルコピー？が含まれているそうな）
```
$ cd pkg-config-0.29.2
$ ./configure --with-internal-glib
$ make
$ make check
$ sudo make install
$ cd ~/php-7.4.32
$ ./configure --with-apxs2=/usr/local/apache2/bin/apxs --with-pdo-mysql
~~~
configure: error: Package requirements (libxml-2.0 >= 2.7.6) were not met:

No package 'libxml-2.0' found

Consider adjusting the PKG_CONFIG_PATH environment variable if you
installed software in a non-standard prefix.

Alternatively, you may set the environment variables LIBXML_CFLAGS
and LIBXML_LIBS to avoid the need to call pkg-config.
See the pkg-config man page for more details.
```
pkg-configを標準でない方法でインストールした際はPKG_CONFIG_PATHを環境変数で指定したほうがいいらしい \
また、足りない[libxml-2.0](https://gitlab.gnome.org/GNOME/libxml2/-/tree/master)をインストールする
```
$ which pkg-config
$ export PKG_CONFIG_PATH=/usr/local/bin/pkg-config
$ wget https://gitlab.gnome.org/GNOME/libxml2/-/archive/v2.10.3/libxml2-v2.10.3.tar.gz
$ tar xf libxml2-v2.10.3.tar.gz
$ cd libxml2-v2.10.3
$ ./configure
checking for python extension module directory (pyexecdir)... ${PYTHON_EXEC_PREFIX}/lib/python3.10/site-packages
checking for PYTHON... no
configure: error: Package requirements (python-3.10) were not met:

No package 'python-3.10' found

$ python3 --version
Python 3.10.6
```
Python3.10がすでにインストールされているのに無いと言われる。  \
./configureにpythonがあることが伝わっていないかもしれない \
`./configure --help`からpythonを指定する
```
$ which python3
/usr/bin/python3
$ ./configure --with-python-prefix=/usr/bin/python3
~~
checking for python extension module directory (pyexecdir)... ${PYTHON_EXEC_PREFIX}/lib/python3.10/site-packages
checking for PYTHON... no
configure: error: Package requirements (python-3.10) were not met:

No package 'python-3.10' found
```
おなじエラーが出力された　\
pythonのsite-packageを探しているときにエラーがでてるっぽいので、site-packagesの場所を確認してみる
```
$ python3
>>> import site
>>> site.getsitepakcages()
['/usr/local/lib/python3.10/dist-packages', '/usr/lib/python3/dist-packages', '/usr/lib/python3.10/dist-packages']
```
`dist-packages`しかないことがわかった。[debianのwiki](https://wiki.debian.org/Python)によると、

> dist-packages instead of site-packages. Third party Python software installed from Debian packages goes into dist-packages, not site-packages. This is to reduce conflict between the system Python, and any from-source Python build you might install manually \

とあるので、python3.10をソースビルドして再度libxml2をconfigureする
```
$ wget https://www.python.org/ftp/python/3.10.8/Python-3.10.8.tgz
$ tar xf Python-3.10.8.tgz
$ cd Python3.10.8
$ ./configure
$ make
$ sudo make install
$ cd ~/libxm2-2.10.3
$ ./configure 
$ make
~~
ここでいくつかのdeprecated warnが出るが、make checkでテストが通ったので多分大丈夫
問題があったらここを見直す
$ sudo make install
```
libxml2がインストールできたので、phpを再度configureする
```
$ ./configure --with-apxs2=/usr/local/apache2/bin/apxs --with-pdo-mysql
checking for sqlite3 > 3.7.4... no
configure: error: Package requirements (sqlite3 > 3.7.4) were not met:

No package 'sqlite3' found
```
[sqlite3](https://www.sqlite.org/download.html)をインストールします
```
$ wget https://www.sqlite.org/2022/sqlite-autoconf-3390400.tar.gz
$ tar xf sqlite-autoconf-3390400.tar.gz
$ ./configure
$ make
$ sudo make install
$ ./configure --with-apxs2=/usr/local/apache2/bin/apxs --with-pdo-mysql
configure: error: Package requirements (zlib) were not met:

No package 'zlib' found
```
[zlib](https://www.zlib.net/)をインストール
```
$ wget https://www.zlib.net/zlib-1.2.13.tar.gz
$ tar xf zlib-1.2.13.tar.gz
$ cd zlip-1.2.13.tar.gz
$ ./configure
$ make
$ make check
$ sudo make install
$ cd ~/php-7.4.32
$ ./configure --with-apxs2=/usr/local/apache2/bin/apxs --with-pdo-mysql
$ make
$ sudo make install
```
PHP公式のガイドに沿ってインストールを進める
```
$ sudo cp php.ini-development /usr/local/lib/php.ini
$ cat /usr/local/apache2/conf/httpd.conf
~~
LoadModule php7_module modules/libphp7.so
~~~
<FilesMatch \.php$>
        SetHandler appication/x-httpd-php
</FilesMatch>
$ cat /usr/local/apache2/htdocs/tomishima/phpinfo.php
<?php phpinfo(1); ?>
$ systemctl restart httpd
$ curl tomishima-hbtask.local/phpinfo.php
<?php phpinfo(1); ?>
```
モジュールの動的ロードができていない。apacheを再インストールする
```
$ cd httpd-2.4.54
$ ./configure --enable-so
$ make
$ sudo make install
$ systemctl restart httpd
$ curl tomishima-hbtask.local/phpinfo.php
~~phpinfoのhtml~~
```
PHPのインストールが完了した。
# TODO
SELinuxを無効(apparmor)
mdが長すぎてよみずらいので分割する L A M P
