---
layout: post
title: "そうだ、Intel DCIをしよう！"
description: "Intel DCIの接続に成功したのでその方法を書いておきます。"
category: Intel_DCI
tag: Intel_DCI JTAG Linux UEFI
locale: ja_JP
---

<blockquote class="twitter-tweet" data-conversation="none" data-lang="ja"><p lang="ja" dir="ltr">BINGO!<br>PC固まった！<br>やったぜ！ <a href="https://t.co/nULCPvDS2y">pic.twitter.com/nULCPvDS2y</a></p>&mdash; 高寺 俊喜(JP3BGY) (@JP3BGY) <a href="https://twitter.com/JP3BGY/status/1094117685183447041?ref_src=twsrc%5Etfw">2019年2月9日</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

先程Intel DCIへの接続に成功しましたので、その再現方法を記します。本来あまり公に書いていいものかと言われると疑問だけど、日本語だし、僕のBlog無名だし大丈夫でしょ？と言う感じで行きます。
なお、ここで書かれていることに関して僕は一切の責任を負いませんので悪しからず。

## ターゲットPCとなるCPUとマザーボード、その他パーツの用意
### 追記
SkylakeはUSB Debug Cableを用いた方法は使えないそうなので訂正

まずはIntel DCIが内蔵されている ~~Skylake~~ KabyLake以降のCPUとCPUのJTAGPinがチップセットと配線でつながっているマザーボードが必要になります。一応話を聞く限りではASRock社製はわりとあたりで、ASUSはセキュリティ上の理由でつながってなくて、GIGABYTEはワンちゃん行けるかもと言う感じみたい。他に情報があればTwitterかこのblogのGithub issueにてお話し聞かせていただけると嬉しいです。
私はIntel CoffeeLake Celeron & ASRock H310M-AC/ITXで成功しました。
ノートPCは当たりハズレが激しそうなのでお金のない人は動くことがわかっている環境を自作することを勧めます。

## USB 3.0 Debug Cableの用意
僕は[Data ProのUSB 3.0 Debug Cable](https://www.datapro.net/products/usb-3-0-super-speed-a-a-debugging-cable.html)を輸入しました。どういうケーブルかというと、USB 2.0互換用の配線とV+配線がつながっていないケーブルです。人によっては自作しているらしいですが、この辺非常にデリケートな部分なのでPC壊したくなければ購入することを勧めます。
どうしても作る場合はケーブルを切開する方法ではなく、USB端子の金属の接触部分を剥がす方法をやるとある程度楽で、ノイズの影響が少ないと思います。

## ホスト環境の用意
ホスト環境はUSB 3.0ポート以外特別なものが必要なわけではないですが、IvyBridge以前のUSB3.0ポートだとどういうわけかうまく動かない場合があるようです。僕はASUS Z77のUSB 3.0を使おうとしてうまく動きませんでした。自作PCであれば最近発売されたUSB3.X PCIe拡張ボードとかを増設すれば問題ないそうなので、もし以下に書く設定をしてUSB Debug Cableでつなげてもデバイス一覧にDCIの文字が一切なければそのようにすると良いです。

## Intel System Studioのインストール
[Intelのホームページ](https://software.intel.com/en-us/system-studio)からダウンロードしてインストールしましょう。学生と非営利なら使いたい放題できるのかな？おそらく。

## BIOS設定の書き換え

追記

BIOSにてIntel DCIを有効にする前に、Secure Bootの設定を無効化する必要があります。ASRock社製のUEFIはどうも標準で無効化されているようで書くことを失念していました。
皆さんは気をつけてください。

ここからがちょっと難関。環境によって微妙に変えるべき設定が変わるので注意してください。Skylakeについては参考資料の記述を見てください。ここではCoffeeLakeでのお話をします。CoffeeLake環境では以下の設定を変更すればDCIが使用可能になります。
```
Platform Debug Consent -> Enabled (DCI OOB+[DbC])
CPU Run Control -> Enabled
CPU Run Control Lock -> Disabled
PCH Trace Hub Enable Mode -> Host Debugger
```
ですが、ASRockのUEFI画面からは変更することができません。そこで、[EFI Shell](https://github.com/chipsec/chipsec/wiki/Creating-a-Bootable-USB-drive-with-UEFI-Shell)と[RU.EFI](http://ruexe.blogspot.com/)というツールを用いて無理やり書き換えていきます（ここでマザーボード及びCPUの会社からの保証・サポートは一切なくなるのでその点は注意してください）。まずは[このgist](https://gist.github.com/eiselekd/d235b52a1615c79d3c6b3912731ab9b2)に書いてあるようにBIOS ROMから変更箇所に該当するデータのオフセットを探します。細かい方法はgistの方に書かれているので書いてない情報について少し補足します。

まず、UEFIToolでUUIDって書かれてる部分はUEFIToolのGUID Searchで見つかります（このgistに書いてある情報は少し古そう？）。で、見つけた部分をExtractしますがBody onlyでもwholeでもどっちでも大丈夫です。で、Universal-IFR-Extractorを用いてBIOSの情報を探し出すのですが、だいたい[このファイル](https://github.com/JP3BGY/blog/blob/master/data/section.txt)みたいなものが出力されます。長ったらしい上に何書いているかわからんかもしれませんが、落ち着いてみればそうでもありません。
```

0x3D6E6 		One Of: Platform Debug Consent, VarStoreInfo (VarOffset/VarName): 0x1137, VarStore: 0x1, QuestionId: 0x27A1, Size: 1, Min: 0x0, Max 0x5, Step: 0x0 {05 91 77 15 78 15 A1 27 01 00 37 11 14 10 00 05 00}
0x3D6F7 			One Of Option: Disabled, Value (8 bit): 0x0 (default) {09 07 04 00 30 00 00}
0x3D6FE 			One Of Option: Enabled (DCI OOB+[DbC]), Value (8 bit): 0x1 {09 07 79 15 00 00 01}
0x3D705 			One Of Option: Enabled (DCI OOB), Value (8 bit): 0x2 {09 07 7A 15 00 00 02}
0x3D70C 			One Of Option: Enabled (USB2 DbC), Value (8 bit): 0x5 {09 07 7B 15 00 00 05}
0x3D713 			One Of Option: Enabled (USB3 DbC), Value (8 bit): 0x3 {09 07 7C 15 00 00 03}
0x3D71A 			One Of Option: Enabled (XDP/MIPI60), Value (8 bit): 0x4 {09 07 7D 15 00 00 04}
0x3D721 		End One Of {29 02}
0x3D723 		Ref: Advanced Debug Settings, VarStoreInfo (VarOffset/VarName): 0xFFFF, VarStore: 0x0, QuestionId: 0x3F8, FormId: 0x27A3 {0F 0F 71 15 72 15 F8 03 00 00 FF FF 00 A3 27}
0x3D732 	End Form {29 02}
0x3D734 	Form: Advanced Debug Settings, FormId: 0x27A3 {01 86 A3 27 71 15}
0x3D73A 		One Of: USB3 Type-C UFP2DFP Kernel/Platform Debug Support, VarStoreInfo (VarOffset/VarName): 0xA6D, VarStore: 0x1, QuestionId: 0x3F9, Size: 1, Min: 0x0, Max 0x2, Step: 0x0 {05 91 96 15 97 15 F9 03 01 00 6D 0A 10 10 00 02 00}
0x3D74B 			One Of Option: Disabled, Value (8 bit): 0x0 {09 07 04 00 00 00 00}
0x3D752 			One Of Option: Enabled, Value (8 bit): 0x1 {09 07 03 00 00 00 01}
0x3D759 			One Of Option: No Change, Value (8 bit): 0x2 (default) {09 07 95 15 30 00 02}
0x3D760 		End One Of {29 02}
0x3D762 		Suppress If {0A 82}
0x3D764 			QuestionId: 0x27A1 equals value 0x0 {12 06 A1 27 00 00}
0x3D76A 			One Of: SLP_S0# Override, VarStoreInfo (VarOffset/VarName): 0xA75, VarStore: 0x1, QuestionId: 0x3FA, Size: 1, Min: 0x0, Max 0x2, Step: 0x0 {05 91 7E 15 7F 15 FA 03 01 00 75 0A 10 10 00 02 00}
0x3D77B 				One Of Option: Disabled, Value (8 bit): 0x0 {09 07 04 00 00 00 00}
0x3D782 				One Of Option: Enabled, Value (8 bit): 0x1 {09 07 03 00 00 00 01}
0x3D789 				One Of Option: Auto, Value (8 bit): 0x2 (default) {09 07 06 00 30 00 02}
0x3D790 			End One Of {29 02}
0x3D792 			One Of: S0ix Override Settings, VarStoreInfo (VarOffset/VarName): 0xA76, VarStore: 0x1, QuestionId: 0x3FB, Size: 1, Min: 0x0, Max 0x3, Step: 0x0 {05 91 80 15 81 15 FB 03 01 00 76 0A 10 10 00 03 00}
0x3D7A3 				One Of Option: No Change, Value (8 bit): 0x0 {09 07 95 15 00 00 00}
0x3D7AA 				One Of Option: DCI OOB, Value (8 bit): 0x1 {09 07 82 15 00 00 01}
0x3D7B1 				One Of Option: USB2 DbC, Value (8 bit): 0x2 {09 07 83 15 00 00 02}
0x3D7B8 				One Of Option: Auto, Value (8 bit): 0x3 (default) {09 07 06 00 30 00 03}
0x3D7BF 			End One Of {29 02}
0x3D7C1 			One Of: USB Overcurrent Override for DbC, VarStoreInfo (VarOffset/VarName): 0xA6E, VarStore: 0x1, QuestionId: 0x3FC, Size: 1, Min: 0x0, Max 0x1, Step: 0x0 {05 91 98 15 99 15 FC 03 01 00 6E 0A 10 10 00 01 00}
0x3D7D2 				One Of Option: Disabled, Value (8 bit): 0x0 (default) {09 07 04 00 30 00 00}
0x3D7D9 				One Of Option: Enabled, Value (8 bit): 0x1 {09 07 03 00 00 00 01}
0x3D7E0 			End One Of {29 02}
0x3D7E2 			Suppress If {0A 82}
0x3D7E4 				QuestionId: 0xF46 equals value 0x0 {12 06 46 0F 00 00}
0x3D7EA 				One Of: CPU Run Control , VarStoreInfo (VarOffset/VarName): 0x65C, VarStore: 0x1, QuestionId: 0x3FD, Size: 1, Min: 0x0, Max 0x1, Step: 0x0 {05 91 84 15 85 15 FD 03 01 00 5C 06 10 10 00 01 00}
0x3D7FB 					One Of Option: Disabled, Value (8 bit): 0x0 {09 07 22 01 00 00 00}
0x3D802 					One Of Option: Enabled, Value (8 bit): 0x1 {09 07 21 01 00 00 01}
0x3D809 				End One Of {29 02}
0x3D80B 				One Of: CPU Run Control Lock , VarStoreInfo (VarOffset/VarName): 0x65D, VarStore: 0x1, QuestionId: 0x3FE, Size: 1, Min: 0x0, Max 0x1, Step: 0x0 {05 91 86 15 87 15 FE 03 01 00 5D 06 10 10 00 01 00}
0x3D81C 					One Of Option: Disabled, Value (8 bit): 0x0 {09 07 04 00 00 00 00}
0x3D823 					One Of Option: Enabled, Value (8 bit): 0x1 (default) {09 07 03 00 30 00 01}
0x3D82A 				End One Of {29 02}
0x3D82C 			End If {29 02}
0x3D82E 		End If {29 02}
0x3D830 		One Of: PCH Trace Hub Enable Mode, VarStoreInfo (VarOffset/VarName): 0x110F, VarStore: 0x1, QuestionId: 0x3FF, Size: 1, Min: 0x0, Max 0x2, Step: 0x0 {05 91 9A 15 9B 15 FF 03 01 00 0F 11 10 10 00 02 00}
0x3D841 			One Of Option: Disabled, Value (8 bit): 0x0 (default) {09 07 04 00 30 00 00}
0x3D848 			One Of Option: Target Debugger, Value (8 bit): 0x1 {09 07 A0 15 00 00 01}
0x3D84F 			One Of Option: Host Debugger, Value (8 bit): 0x2 {09 07 A1 15 00 00 02}
0x3D856 		End One Of {29 02}
```
今回関わってくる部分は上のような感じですが、例えば `One Of: ~ ` ってのがUEFIの設定項目で、 `VarOffset/VarName` というのが必要なデータの保存場所のオフセットです。 `One of Option:~` ってのがその設定項目の選択肢で同じ行にある `Value (): 0xYY` ってのがその設定を意味する値、Enumみたいなものです。 `Suppress If {0A 82} ~ ` ってのは後に大体 `QuestionID: ~ equals value ~` てのが続きますがこれはつまりQuestionIdで指定されたUEFIの設定項目の値がvalueである場合はこの選択肢は無効となる、と言う感じですね。上の例だとRun Controlの部分はPlatform Debug Consentの設定がEnabledになっていなければ無効になるのがわかります（もう一つ条件がありますがこれの示す先はUEFIの設定項目ではなかったので謎でしたがRU.EFIを見る限りDefaultで0ではなかったのでこっちが原因で無効になることはなさそうです）。

で、以上の方法で設定が無効になってしまうSuppress Ifの条件を含めて、必要な設定の保存オフセットが手に入ります。後はRU.EFIを使って書き換えるだけです。RU.EFIはUEFI Shellを起動可能状態にしているFAT32(ほかのファイルシステムはマザーボードによって対応しているかどうかが異なるのでオススメしません、ASRockはexFatは使えませんでした)で初期化されたUSBメモリの任意の場所にRU.EXE,RU.EFI,RUx32.EFIのすべてを同じディレクトリに保存し、UEFI Shellに入って以下の操作をすると起動します。
```
fs0:[Enter]
cd path/to/RU[Enter]
./RU.EFI
```
基本的にEFI ShellはUnix系ターミナルと同じ操作でプログラムの起動等ができます。

起動後は [Alt] and [-] and [=]の同時押し（注、RU.efiの操作は英字配列です）でUEFI configuration Listにはいって、Setupという文字を探してEnter、で、記録したオフセットの部分を記録したとおりの値に変更すれば終了です。念の為ちゃんと変更できているかどうか再起動した後RU.EFIをもう一度動かして確認すると良いです。

## Host PCからDebug PCに接続
まず、Guest PCからDebug機能がついているUSB 3.Xポートを探します。Linuxでの探し方はわからなかったのでWindows10評価版とUSB Viewを用いた方法を使いました。[MSDNのDoc](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/setting-up-a-usb-3-0-debug-cable-connection)を参考に探してみてください。H310M-ac/ITXはウラ面のUSB 3.1ポート2つともDebug機能が使えました。後は、USBケーブルでHostとGuestをつなげて、[eclypsiumの発表スライド](https://github.com/eclypsium/Publications/tree/master/2018/DEFCON26)を参考に設定をいじってやるだけです。Core iシリーズのCoffeeLakeの場合、Configuration Consoleで`CFL_CNP_OpenDCI_DBC_Only_ReferenceSettings`という項目を選んであげれば動きます(CFLはCoffeeLakeの略らしいですね)。接続ができたら`itp.hlt()`を呼んで、PCが固まれば成功です。お疲れ様でした。

## CPU/OSデバッグをしよう！
これで、OSやその下のレイヤであるUEFI,更にはCPU内部までデバッグすることが可能になります。QEMUやらのエミュレータではなく実機でデバッグすることができるのがとても魅力的です。マシンも安い構成なら3〜4万ほどで組み立てることができますので、低レイヤーの勉強を進める人はぜひご自宅に一台デバッグ専用機を作ってみてください。それでは、良いデバッグライフを！

## 参考にした資料
https://gist.github.com/eiselekd/d235b52a1615c79d3c6b3912731ab9b2

https://conference.hitb.org/hitbsecconf2017ams/sessions/commsec-intel-dci-secrets/

https://eclypsium.com/2018/07/23/evil-maid-firmware-attacks-using-usb-debug/

https://github.com/eclypsium/Publications

https://www.slideshare.net/phdays/tapping-into-the-core

https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/setting-up-a-usb-3-0-debug-cable-connection

https://github.com/ptresearch

