---
layout: post
title: "Intel DCI 続編（資料まとめ）"
description: "この記事は[TSG Advent Calendar 2019](https://adventar.org/calendars/4182)の1日目の記事として書かれました。今日はTSGの分科会の内容と関連した、Intel DCIの続編について書こうと思います。"
category: Linux
tag: Linux JTAG Intel
locale: ja_JP
---
## 挨拶

この記事は[TSG Advent Calendar 2019](https://adventar.org/calendars/4182)の1日目の記事として書かれました。

皆さん、どうも、いつの間にかTSGの部長になって一年が経っていたJP3BGYです。
どういうわけか、今年一年は何やらいろいろイベントがありまして。3月にサークル代表として総長賞をいただくことになったり、4月から新入生向けの分科会を開きながら5月にTSG CTF（なお作問に失敗して僕の問題はお蔵入りですが）と五月祭、6月はKMCと合同のゴルフ大会が開催され、8月9月はSECCONの作問に追われながらTWCTFでおしいところまで行って、11月に作問した問題でSECCONが行われた後に駒場祭が来て、そして年末のTSG Advent Calendarの時期が来て今に来たるという感じです。

CTFだけでなく競プロやら自作のソフトウェアやらにも進捗がでてTWCTFの本戦はいけなくて残念だったが今年は結構いい年だったんじゃないかと思う次第です。

## Intel DCI 続編

さて、挨拶はこの辺にして本題のIntel DCIのお話をしていきましょう。昨年度のAdvent Calendar記事に書きましたが、Source Code Reading分科会を今年もやりたいと言っていたことが実現して、今現在昨年度の内容をそのまま下級生にやっているEasyとTSG内で不足しているPWN成分を補うために行われているLinux Kernel,Browser,VMのソースコードを読んで攻撃方法を探るHardの2つの難易度でやっております。
今回はそのうちHardにあたる内容を少しばかり一般に公開しようかなと思います。

こちらは今年度初めのあたりの話ですが、[このようなIntel DCIの記事](https://jp3bgy.github.io/blog/intel_dci/2019/02/09/Do-Intel-DCI.html)を出しました。Intel DCIに関する資料はあまり多く出回ってなかったためそれなりに反響があり書いた甲斐があったと思うのですが、この記事の内容は接続方法までしか書かれていなくて、ここから実際にどうやってデバッグをしていくのかとか、新しくでたCannon Lakeとかとは違いがあるのかと言った、技術的な詳細に関してほとんどなにも書かれていません。
これは僕と同じことを試した方ならわかってもらえるでしょうが、Intel System Studioに関しての資料って[公式Documents](https://software.intel.com/en-us/system-studio/documentation/view-all)には大した情報が書かれていなくて、ちゃんとしたことを知ろうと思ったらあちこち探し回らないといけないんですね。以前は調べる方法が全くわからず、とりあえずいろいろ試して動いたからBlogにまとめたという感じだったんですが、今回改めて資料を探し直してある程度のことがわかったのでここに書いていこうかなと思います。

## Intel DCI を使ったOSやUEFI Debugのやり方
とりあえず、DCIってそもそもなんだよってのと、接続の仕方については前回の記事を参照してください。ここでは前回との変更点とか追加情報を話していきます。

前回使ったのはIntel DFx Abstraction Layerとよばれるものだったのですが、どうもIntelでの扱いが変わったらしく最近はもう一方のOpen IPCの方を使うらしいです。なので前回のDALを使った使い方は非推奨らしいです。実際にDALを使ってもうまく接続ができませんでした。

なので、接続するときはIntel System Debuggerから直接つなぐようにしてください。Intel System Debuggerからのつなぎ方は[Eclypsiumのslide](https://github.com/eclypsium/Publications)が参考になると思います。ちなみになんですが、この記事を書いている11月では何でかIntel System Studio for Linux Targetのみをインストールした場合どう頑張ってもDCIにつなげることができませんでした。基本 gdbみたいなLinux専用ツールは使わないので for Windows Target版でも問題ないと思うのでこっちをインストールするといいと思います。それと、自分が書いたかどうかわからないので一応ここに書きますが、Linux HostバージョンのSystem StudioではDCIの機能は使えません。必ずWindowsを用意して、Windows Host版のSystem Studioを使うようにしてください。

さて、前回話すことのなかったOSやUEFIの具体的なDebugの仕方なのですが、Intel System Debuggerはどうも.pdbもdwarfも対応しているらしいので、とりあえず好きな環境でデバッグ情報付きでデバッグしたいプログラムをビルドしてください。そしてデバッグホスト側からそれらのファイルにアクセスできるような環境を用意してください。これちょっと苦戦したんですが、最近どうもext2fsdとかのドライバがWindows 10ではうまく動かないらしく、ext4から直接ファイルを読み取るみたいなことができませんでした。なので仕方なく僕はSambaによるネットワーク共有をしています。[WSL2ではどうもLinux Partitionのmountもサポートするとの情報](https://www.reddit.com/r/bashonubuntuonwindows/comments/bqcc6z/using_wsl2_to_access_a_linux_drivepartition_eg_a/)もあるらしいので、そちらを期待しましょう。ちなみに現在のPreview版ではまだその機能はなさそうでした。

ビルドしたファイル群にアクセスできるようにしたら、ビルドしたファイルをターゲットにインストールして起動しましょう。ここで注意してほしいことがありまして、後ほど話しますがIntel DCI DbC3は制約がありまして、CPU Resetからプログラムを眺めることができません。なので例えば起動の一番最初からDebugしたいという場合は自身でプログラムに無限ループっぽいものを埋め込む必要があります。[Intel公式情報](https://osfc.io/uploads/talk/paper/18/Debugging_Intel_Firmware_using_DCI___USB_3.0.pdf)によると、それ用のCPUの機能とかがあったり、edk2にはそれ用の関数があったりするらしいです。まあ、こういうのが面倒だったりするなら、UEFIに入った後にDCIで接続してみたいなことをするといいかもしれません。あと、kASLRみたいな機能は切っておくことを推奨します。一応kASLRみたいな機能が使われる前にはDebugをすることは可能なのでkASLRに切り替わる前にその情報を集めてDebugのmemory mapping情報のオフセットを変えるみたいなことはできるとは思うのですが、面倒極まりないので辞めときましょう。

さて、ここまでで接続の準備はできたので接続を初めましょう。接続した後アセンブラっぽいものが表示されたら正しく接続されています。表示されない場合は何かしらの理由で接続がうまく言っていません。理由としては、
1. USB DbCケーブルがおかしい
2. CPUがまだ起動されていない
3. System Studio for Linuxだけしかインストールしてない

とかが僕の環境ではありました。

接続後、デバッグ情報を読み込みましょう。この時LinuxなどのDWARF情報を読み込むものはファイルが見つからないとエラーを出すかと思います。これはDWARFの情報は絶対パスで保存されているため、なにも設定をしなければ絶対パスで同じところにファイルが存在しないと怒られるからです。当然ビルド時に相対パスにするオプションは存在するのですが面倒なのでデバッガ側の設定をしてそっちで対応させましょう。Options->Source Directoriesと選択をするとRulesとか書かれている画面が表示されます。Rulesというのはデバッグ情報に書かれているパスが実際のパスとしてどこと対応するのかというものを書くことができます。とりあえずルートパスがどこと対応するかだけ書いておけばなんとかなります。他に、Directoriesと書かれたタブもあります。こちらは相対パスで保存されていたりする場合に相対パスの根本はどこのディレクトリなのか等を設定する場所です。自身のビルド環境に対応した方を適切に設定してください。

ここまできて、ようやくデバッグを進めることができます。QEMUの場合と比べてずいぶんと手間がかかりますが、その分利点も多く、例えばgdbが対応してないようなMMUやもろもろのDescriptor Tableの状況などシステムの状態がGUIですぐに見ることができます。一応Linuxをデバッグしている場合はvmlinux-gdb.pyと呼ばれるようなgdb拡張機能によって一部これらの情報を表示することができますが、GUIでこれらが簡単に行えるのは大きな利点ではないでしょうか。あとはやはり実機でのデバッグがやりやすいという点ですね、QEMUとかではデバイスのエミュレーションは多少はありますがエミューレーターが作られているものはそこまで多くないことも考えると、とりわけデバイスが関わってくる部分は実機でやったほうが良いと思われます。

## Intel DCIの種類と違い

とりあえず、前回の記事に書かなかったOSデバッグまでの流れを書きましたが、次にIntel DCIの種類について話します。先にDbC3では制限があると書いた話です。そもそも、Intel DCIで使えるデバイスは4つ（環境によっては3つ）あります。
1. Intel ITP-XDP マザーボードにあるJTAGに直接差し込むタイプ
2. Intel SVT CCA USBでつなぐが、ホストとゲストの間に専用デバイスが必要になるタイプ、ソフトウェア側ではOOBとか表記されてる
3. Intel DCI DbC3 前回から話しているやつ
4. Intel DCI DbC2 資料が全然ない、おそらくインターネット上には存在していなさそう ~fuckin Intel~

これらのうちSkylake以降すべてに対応しているのが1~3なのですが、最近発売したCannon Lake、つまり第9世代以降ではおそらく4つ目のDbC2にも対応していると思います。

とりあえず1~3のうち僕らが使うDbC3が負っている制約について話します。まず、DbC3はUSB 3.0のデバイスの機能を使っているため、USB 3.0が使えるようになるまで接続することができません。具体的にはS0 Stateになった後USB 3.0が動くようになるまでの間のデバッグはできません。USB 3.0が動くようになる前にCPU Resetは動いてしまうためこれは結構大きな痛手で、例えばIntel System Debuggerにあるreset/restartコマンド、いわゆるBIOS resetコマンドが使えません。つまり、起動の一番最初っからデバッグをすることができません。これをするには1か2の方法を使う必要がありますが、少なくとも個人向けにこれらのデバイスをIntelは販売していないので現状研究機関かハードウェア開発会社に就職する以外でこれらを使う方法は無いと言って良いでしょう。

ところが、どうも4が対応されているであろうと思われるCannon Lakeでは少し話が変わる可能性があり、DbC2はUSB 2.0さえ動いていれば接続ができて、しかもUSB 2.0はCPU Resetが走る前に接続可能状態になります。これは僕は実機を持っていないので確認することができないのですが、もしかしたらBIOS resetも動く可能性があります。しかし、これらに関する確たる情報は現状見つかっておらず、例えばこの話をするにあたって参考にした[Intelの動画](https://software.intel.com/en-us/videos/introduction-of-system-debug-and-trace-in-intel-system-studio-2018)および、[Intelが出しているファームウェアの仕様](https://www.intel.com/content/www/us/en/intelligent-systems/intel-firmware-support-package/intel-fsp-overview.html)を見てみると、前者では次のCPUから対応だよー、つまりCoffee Lakeから対応していますよ、と言われているんですが、Coffee Lakeのファームウェアの仕様を見てみると、確かに設定項目としてDbC2のはあるにはあるんですが、よく見てみるとこの設定はOOB+[DbC]と同じ設定であると書かれていて、おそらくなのですがサポートがされていないのではないかと思います。実際、Coffee LakeでDbC2を試すために、USB 3.0 A-Aケーブルを購入しまして、電源端子のみを剥がしてDbC3/2両対応させたケーブルを作って試してみたのですが、おそらくUSB 3.0として接続されているような感じがありました。ちなみになんですがData Proが出しているUSB 3.0 Debug cableはUSB 2.0の配線がされていないのでDbC2のケーブルとしては使えません。DbC2のケーブルとしてはIntelが公式で売っているケーブルの情報を見る限りUSB 3.0のA-Aケーブルから電源端子だけ接続してないようなものであれば良いと思われます。話を戻して、Cannon Lakeのファームウェアを見てみると、Coffee LakeにもあったDebuggerの設定項目以外にDbCとして2と3どちらを使うかみたいな設定項目が増えています。Coffee Lakeと同じくこれも単純にファームウェアとして設定を作っただけでハードウェアは対応していないよという可能性はありますが少なくともCoffee Lakeとは違ってハードウェアでも対応がなされている可能性は高いです。ただ、公式がうんともすんとも言っていないので、こればっかりは実際に試してみるしか無いのではないかと思います。僕は第10世代で対応がされると噂されているIntel CETが試したいので誰かCannon Lakeで人柱になってください。それか僕にPCを恵んでください。

## 終わりに

とりあえず、前回調べきることができなかった情報について軽くまとめました。Intelのドキュメントがクソみたいになにも書かれていない上に、一応ちゃんと調べればそれなりの資料が存在するという本当にクソみたいな状況だったので調べきるのに時間がかかりました。調べる課程で記事の参考にしなかったが便利そうなリンクについては下の参考リンク集に張っておくので、是非活用してください。

## 参考にした資料まとめ
あまり記事の参考にしなかったが、便利な資料もまとめて張っておきます。
https://software.intel.com/en-us/videos/introduction-of-system-debug-and-trace-in-intel-system-studio-2018
https://osfc.io/uploads/talk/paper/18/Debugging_Intel_Firmware_using_DCI___USB_3.0.pdf
https://software.intel.com/security-software-guidance/
https://casualhacking.io/blog/2019/6/2/debug-uefi-code-by-single-stepping-your-coffee-lake-s-hardware-cpu
http://www.freepatentsonline.com/y2016/0283423.html
https://github.com/eclypsium/Publications
https://www.intel.com/content/www/us/en/intelligent-systems/intel-firmware-support-package/intel-fsp-overview.html<Paste>
