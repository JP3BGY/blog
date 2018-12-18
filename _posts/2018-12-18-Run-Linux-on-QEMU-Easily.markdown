---
layout: post
title: "OVMFを使った簡単なUEFI Linux環境の構築の仕方"
description: "この記事は[TSG Advent Calendar 2018](https://adventar.org/calendars/3450)の18日目の記事として書かれました。昨日はsatos氏の[駒場祭CTF作問の裏話](http://satos.hatenablog.jp/entry/2018/12/17/235940)でした。皆さん、どうも、TSGの中で一番新入部員っぽい雰囲気を醸し出してる1年のJP3BGYです。今日は秋学期にTSGで僕が担当を持っているLinux系Source Code Reading分科会に関連したお話をしようと思います。"
category: QEMU Linux UEFI
tag: QEMU Linux UEFI
locale: ja_JP
---

この記事は[TSG Advent Calendar 2018](https://adventar.org/calendars/3450)の18日目の記事として書かれました。
昨日はsatos氏の[駒場祭CTF作問の裏話](http://satos.hatenablog.jp/entry/2018/12/17/235940)でした。

皆さん、どうも、TSGの中で一番新入部員っぽい雰囲気を醸し出してる1年のJP3BGYです。今日は秋学期にTSGで僕が担当を持っているLinux系Source Code Reading分科会に関連したお話をしようと思います。

Linux系Source Code Reading分科会とは秋学期に僕がSlackで突然やろうと言って立ち上げた、glibcやcoreutils,Linux KernelなどのLinux関連のソフトウェアのソースコードを多読してC言語やLinuxシステムに強くなろうという分科会(TSG内部で毎週開かれる定期勉強会の類です)です。
当初はC言語入門したての人を想定してやろうと思っていたのですがメンツが[fiord氏](http://hyoga.hatenablog.com/),[kakinotane氏](https://kaki-no-tane.hatenablog.com/),[akouryy氏](http://akouryy.hatenablog.jp/)と最初っから最後までガチメンツだったため内容もガチになり紆余曲折あって現在Linux Kernelのmmap()を読みすすめています。
なんだかんだそれなりに上手く行っていて、それなりに好評っぽいので次の春学期にもやろうかなと企んでいます（味をしめた人）。
ですので、来年東大に入るであろう人やこれからプログラミングを始めようとしている東大生は是非TSGに入って僕と一緒にC言語を多読しましょう。

さて、閑話休題、そろそろ本題に入りましょう。先程話した分科会でLinux Kernelを読みすすめていると言いましたが、今現在はQEMUの-kernelや-appendなどの機能を用いたLegacy BIOS上でのLinux Kernelの挙動を見ています。mmap()等起動し終わったあとのKernelならば大抵の部分はUEFI環境とそこまで大差はないのですが、初期化プロセスなどはもろにそのあたりが関わっており、ソースを読む上では今後のことも考えてUEFI環境のものを見ておきたい、Legacy BIOSはクソだから読みたくない等の理由でUEFI環境を用意しておきたいと思うかもしれません。そこで、今回はQEMU上で動くOVMFを用いてBIOSのときとほぼ変わらない手間でUEFI Linuxを実行できる環境を用意しましょう。

## OVMF について

[OVMF](https://github.com/tianocore/tianocore.github.io/wiki/OVMF)とは[tianocore]()プロジェクトの一環で作られている[EDK II](https://github.com/tianocore/tianocore.github.io/wiki/EDK-II)というクロスプラットフォームなUEFI firmwareの中にあるQEMU用の firmwareです。
これを使うことでUEFIアプリケーションをQEMU上で実行、デバッグすることができるというすぐれものです。このOVMFの中にはLoadLinxLibと呼ばれるLinux Kernelローダーが存在してこれを使うことでLegacy BIOSの時と同じように```-kernel```, ```-append```, ```-initrd```オプションを用いてKernelを直に起動することができます。Grub2やらのUEFI対応ブートローダーの設定をしなくてもKernelが起動できるのはソースを読む側にとって非常に好都合です(無駄なソフトがない分デバッグが楽)。これを使わない手はないので是非使い方を覚えましょう。

## OVMF のビルド

OVMFを使うためにはまずOVMFをビルドする必要があります。Ubuntu等一部の環境ではパッケージマネージャの中にビルド済みOVMFを配布しているものも存在しますが無いものも多いのでここではビルドを用いた方法を書きます。

やり方はとっても簡単、まず前準備としてmake等のビルド環境(ubuntuならbuild-essential)とuuid-dev texinfo bison flex libgmp3-dev libmpfr-dev subversion iasl(acpica-tools)をパッケージマネージャからインストールします。
そして以下のコマンドを実行するだけ。

```bash
git clone https://github.com/tianocore/edk2/
cd edk2
./OvmfPkg/build.sh -a X64
```

これでx64用のOVMFがedk2/Build/なんたらかんたら/OVMF.fdという場所に出来上がります。
これを好きな場所にコピーして使いましょう。

## Linux のビルド

で、あとはLinux側の用意なんですが、当然Ubuntu等ディストリを用いてデバッグするのも悪くありませんが、デバッグというのはできる限り小さい環境でやるべきなのでここではちゃんとLinux Kernelをビルドしましょう。
といってもこれの大半は[kakinotane氏のアドベントカレンダーの記事](https://kaki-no-tane.hatenablog.com/entry/2018/12/03/135302)にかかれているのでここでは補足もとい追加すべきLinux Kernel parameterについて書きます。

kakinotane氏の記事の設定ではそもそもKernelがUEFIに対応していないとかVGAドライバがなくてGUIが立ち上がらないとかの問題があるのでそのへんをどうにかするべくいろいろいじったのですが、ブラックリストで消していくとうまく行くのに、ホワイトリストで設定をonにしていくととたんに動かなくなる（隠れたパラメタが原因っぽい？）のでここに僕が使ったできる限り項目を減らした[.config](https://raw.githubusercontent.com/JP3BGY/blog/master/data/.config)へのリンクを貼っときます。
これをダウンロードして.configをうまいこと差し替えてください。

また、当然ですが自分の見たいKernelの機能もyesにするようにしてください。ここで注意してほしいのは、この設定ではKernel Moduleを読み込むことができないということです。
読みたい機能は必ずMではなくyにするようにしてください。
Kernel Moduleをどうしても読み込みたいという方は[こちら](https://gist.github.com/chrisdone/02e165a0004be33734ac2334f215380e)のgistページの下の方に書かれているBuildrootを用いた方法を使ってみてください。

## Linux on UEFI の実行

後は実行してやるだけ。コマンドも簡単、いつものQEMUのオプションに ```-bios /path/to/copy_of_OVMF.fd``` を付け加えるだけ。これでちゃんと起動してやることもできるし、```-s```オプションをつかってgdbデバッグもできる。

## 終わりに

実はここまで使えるようになるまでにいろいろありまして、OVMFのソースデバッグをゴリゴリやりーのQEMUのソースを読みーのをしていました。そのあたりの話についてはアドベントカレンダー24日の記事に書きますので楽しみにしていてください。

明日はツバメプロの「Java の Deserialization 経由の RCE について書きます (予定)」です。お楽しみに。

## 参考にしたもの

https://github.com/tianocore/tianocore.github.io/wiki/How-to-build-OVMF  
https://gist.github.com/chrisdone/02e165a0004be33734ac2334f215380e   
https://github.com/tianocore/edk2/blob/master/OvmfPkg/README  
https://github.com/tianocore/edk2/commit/52fba28994e9d54e552264a76cda1834122f04d7  
OVMF及びQEMUのソースコード
