---
layout: post
title: "OVMFをソースでバッグするお話"
description:"皆さん、どうも数日ぶり。プロのTSG新入生のJP3BGYです。今日は前回書いた[OVMFを使った簡単なUEFI Linux環境の構築の仕方](https://jp3bgy.github.io/blog/qemu%20linux%20uefi/2018/12/18/Run-Linux-on-QEMU-Easily.html)の続きっぽいもの(中身は独立してる)でこの記事を書くに至るまでに手にしてしまった(不本意)OVMF本体のソースでバッグをする方法です。"
category: QEMU Linux UEFI
tag: QEMU Linux UEFI
locale: ja_JP
---

この記事は[TSG Advent Calendar 2018](https://adventar.org/calendars/3450)の24日目の記事として書かれました。
昨日はakouryy氏の[言語実装分科会関連のつもりだったけど Hexagony 関連になりそう](http://akouryy.hatenablog.jp/entry/hexagony/converter-1)でした。

皆さん、どうも数日ぶり。プロのTSG新入生のJP3BGYです。
今日は前回書いた[OVMFを使った簡単なUEFI Linux環境の構築の仕方](https://jp3bgy.github.io/blog/qemu%20linux%20uefi/2018/12/18/Run-Linux-on-QEMU-Easily.html)の続きっぽいもの(中身は独立してる)でこの記事を書くに至るまでに手にしてしまった(不本意)OVMF本体のソースでバッグをする方法です。

上の記事の知識を得るまで紆余曲折ありまして、最初kakinotane氏が書いた方法でビルドしたKernelをそのままOVMFに投げたら何故か起動しなくて、その原因を探るためにudk-debuggerを用いてOVMFをひたすらソースデバッグしまくりまして、それでも原因がわからずQEMUをソースデバッグして、やっぱりわからずOVMFをデバッグしてようやく原因がKernelオプションにあるんだということに気が付きまして。Ubuntuのカーネルオプション(/bootディレクトリにあるやつ)をコピーしてオプションを削ってはBuild and Runを繰り返してようやく手に入れた情報というわけです。長かった・・・・・・・・・・・・・（白目）。

結局OVMFをソースでバッグをする必要はまったくなくてKernelが原因だったわけなんですが、誠に不本意ながらデバッグする能力を身に着けてしまったので供養の意味も込めて今後使うことのないであろうOVMFのソースでバッグの方法について書きます。 ~~Documentsが全く足りていないEDK IIはクソ~~ これ本当にどこに需要があるんだろうか・・・・・・。

## udk-debugger のインストール

EDK IIにはソースデバッグを可能にするライブラリが中に存在しています。なのでEDK IIが動くハードウェアであればどれであってもつないでソースでバッグをすることができるのですが少々UEFI環境は特殊なのでGDBやWinDBGにそのことを教えないといけません。そのためのソフトウェアである[UDK Debugger](https://firmware.intel.com/develop/intel-uefi-tools-and-utilities/intel-uefi-development-kit-debugger-tool)をインストールしてください。インストール方法は同梱してあるPDFにも書かれているのでここでは省略、インストールしたディレクトリは適当にメモっといてください。
また、今回は以下のコマンドで作成した/tmp/ovmf　PIPEを用いた方法を使うのでインストールするときはその設定を間違わないようにしてください。
```bash
mkfifo /tmp/ovmf.in
mkfifo /tmp/ovmf.out
#/tmpディレクトリは再起動すると消えるので継続的にデバッグをしたい人はどっか他のディレクトリを指定しましょう
#またこの場合udk-debuggerインストール時に設定するPIPEは/tmp/ovmfです、inやoutはつけないよう注意してください
```
なお、この子はWindows,Linux両対応しているわけですが今回はLinux環境を想定しています。Windowsで同じことがしたい人は同梱されているPDFをよく熟読してください。

## OVMFのビルドし直し

次にOVMFのビルドです。前の記事に書いたとおりにビルドするとソースデバッグする機能がOFFになっておりデバッグすることができません。なのでソースデバッグ用にビルドし直しましょう。
```bash
./OvmfPkg/build.sh -a X64 -D SOURCE_DEBUG_ENABLE
```

## GDBのビルド

次はGDBのビルドです。普通のGDBだとudk-debuggerの一部の機能が使えないので以下のコマンドでGDBをビルドしましょう。インストールするかどうかは自身で判断してください。僕はしませんでした。
```bash
wget http://ftp.gnu.org/gnu/gdb/gdb-8.2.tar.xz↲                           
tar xf gdb-8.2.tar.xz↲                                                    
mkdir build_gdb
cd build_gdb↲                                                             
../gdb-8.2/configure --prefix=`pwd`  --with-python=python3 --with-expat --
target=x86_64-w64-mingw32↲                                                
make 
```
configureのオプションは上に書いてあるものは必ず入れて、これらを打ち消すオプションは絶対に入れないでください。ココがないとうまく動きません。

## Debug開始

後はDebugを開始するだけです。
まずはじめに```/path/to/udk-debugger/bin/udk-gdbserver```を起動します。すると/tmp/ovmf{.in,.out}とつながって待機状態になるので、次にOVMFを起動します。
起動するときに``` -serial pipe:/tmp/ovmf```オプションをつけてつなぎましょう。いつも使う```-s```オプションは要らないです。
最後に自前のgdbでudk-gdbserverに接続します。gdbをオプション無しで起動して以下のコマンドを叩きます。なお、ポート番号はudk-gdbserverを起動したターミナルに表示されたものを使いましょう(大体1234のはずだけど)。
```
target remote :[port num]
source /path/to/udk-debugger/scripts/udk-gdb-script
```
これでちゃんと接続できたはずです、scriptをsourceするときに~ unsupported ~みたいな文言がでたらそれはgdbのビルドに失敗しているか他のgdbを使っています。今一度確かめてください。
デバッグするときに注意してほしいのはライブラリがロードされる前にbreakを挿入してそのままcontinueをしてしまうとなんでかbreakで引っかかってくれません。おそらくライブラリをロードされた後gdb側に制御を一旦渡さないと正しくbreakが挿入されないっぽいので注意してください。

## 終わりに

こんな技能**二度**と使わないだろうに一体何で身につけてしまったんだろうなぁ。
ちなみにOVMFで自作のUEFIアプリケーションをデバッグするときはこの機能は全く必要ないです。
[このリンク](https://github.com/tianocore/tianocore.github.io/wiki/How-to-debug-OVMF-with-QEMU-using-GDB)を参考にして頑張りましょう。
この機能は本当にOVMFやEDK IIの開発者のためのものなので多くの人にとってはほぼ無価値です。注意してください（遅い注意書き）。
明日はアドベントカレンダー最終日、博多市による[SECCONを巡るもう一つの戦いの話]()です。お楽しみに。

## 参考にしたもの

udk-debuggerの中に入ってるPDF Document
EDK IIのソース
https://github.com/tianocore/edk2/tree/master/SourceLevelDebugPkg
https://access.redhat.com/sites/default/files/attachments/ovmf-whtepaper-031815.pdf
