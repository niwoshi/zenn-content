---
title: "Dell Display and Peripheral ManagerのCLIでStream Deckからモニター入力を切り替える"
emoji: "🖥"
type: "tech"
topics: ["dell", "streamdeck", "cli", "windows"]
published: false
---

## はじめに

現在、自分の作業環境はDell U4919DWを含む4枚のディスプレイ環境で作業しています。一般的には快適な環境ではあるのですが、ここでPCを3台も使い分けていると入力切替がとにかく面倒です。実際現状ではWindows x 2台、Mac、Linux数台（とはいえこれは基本的にSSHで作業しているのでディスプレイ不要）の運用になると、もうどうしたらいいかわかりません。

EIZOのモニターはScreen InStyleという純正アプリでホットキーをいくらでもアサインできます。ホットキーを覚えるの事態は大変ですが、Stream Deckのようなデバイスや、なにかランチャーアプリのようなものがあればまぁ解決します。ところがDell U4919DWのようなDell Display and Peripheral Manager（旧Dell Display Manager 2）で管理されているディスプレイはホットキーの登録が少なすぎる！

![](https://storage.googleapis.com/zenn-user-upload/6dee6d3043a6-20260207.png)
*ホットキーが2つしか登録できない...*

今までなんとか運用してきたものの、Macの運用の機運が高まってきたというのもあって、何とかしたいと考え始めました。

ここでちょっと調べていたところ、Dell Display and Peripheral Manager（DDPM）はCLIのような機能も搭載されており、コマンドラインから入力を切り替えられることがわかりました。これをbatファイルにしてStream Deckに登録すれば、物理ボタンで一発切替できるという寸法です。

Stream Deckがない人も、デスクトップにbatファイルを置いておけば利用できるとは思います。

## 環境

- モニター: Dell U4919DW
- DDPM: Dell Display and Peripheral Manager
- Stream Deck（batファイル実行用）

直前までDDM2を利用していて、DDPMにしたらホットキー増えるかな？という淡い期待を抱いていました。  
DDPMのインストールは[Dell公式サイト](https://www.dell.com/support/kbdoc/ja-jp/000287285/windows%E5%90%91%E3%81%91dell-display-and-peripheral-manager)から行いました。

重要インストールの際にホットキーなど設定が消失する可能性があるのでスクリーンショットなどで保存しておくことをお勧めします。

## DDPMのCLI構文

DDPMの実行ファイルは、デフォルトのインストール先であれば下記のようになっていると思います。

### 実行ファイルのパス

```
"C:\Program Files\Dell\Dell Display and Peripheral Manager\Plugins\Subagent\CLI.Subagent.exe"
```

### 基本構文

```powershell
# 値の設定
.\CLI.Subagent.exe /set -Display=<パラメータ名> -value=<値>,forcewithnonotice

# 値の取得
.\CLI.Subagent.exe /get -Display=<パラメータ名>
```

**重要**
`-value=` の末尾に `,forcewithnonotice` を付けないと、Windowsの確認ダイアログが表示されて5分待たされます。Stream Deckから自動化する場合は必須のオプションです。

指定できる入力名はこのあたり。

| 入力名 | 説明 |
|---|---|
| `DP` / `DP1` / `DP2` | DisplayPort |
| `HDMI1` / `HDMI2` | HDMI |
| `USB-C` | USB-C |

モニターの機種によって番号付きが必要だったりするので、手元で試して確認するのが確実です。  
ちなみに、自分の場合はDPに関しては`DP`でした。この辺はディスプレイによって変わってくると思います。

PowerShellで実験してみる場合、batファイルとは少し異なるので注意してください。パスの実行などに、冒頭に`&`が必要です。

```ps1
PS > & "C:\Program Files\Dell\Dell Display and Peripheral Manager\Plugins\Subagent\CLI.Subagent.exe" /set -Display=ActiveInputSource -value="HDMI1,forcewithnonotice"
```

`,`などで叱られる場合、`"`でくくってやるのが基本です。

## batファイルを作ってStream Deckに登録する

### 管理者権限問題

DDPM CLIは管理者権限が必須です。普通にbatファイルを実行すると `"Result": "Process unelevated"` エラーが出ます。

管理者権限で実行する方法はいくつかありますが、セキュリティとのバランスを考えてWindowsタスクスケジューラを使う方法を採用します。一応現時点ではこれがよさそうです。（個人で利用する分には実行ファイルに管理者権限を付与してもいいかもしれません）

### タスクスケジューラでタスクを作成

入力ソースごとにタスクを作成します。自分の場合はHDMI1、HDMI2、DPの3つです。

#### 手順（HDMI1の例）
1. `Win + R` を押して `taskschd.msc` と入力してタスクスケジューラを開く
2. 右側の 「タスクの作成」 をクリック

3. 「全般」タブ
   - 名前: `DDPM_HDMI1`（てきとう）
   - 下部「セキュリティ オプション」にある「最上位の特権で実行する」にチェック
   - 「ユーザーがログオンしているときのみ実行する」を選択

4. 「操作」タブ
   - 「新規」をクリック
   - 操作：プログラムの開始
   - プログラム/スクリプト：`"C:\Program Files\Dell\Dell Display and Peripheral Manager\Plugins\Subagent\CLI.Subagent.exe"`
   - 引数の追加：`/set -Display=ActiveInputSource -value=HDMI1,forcewithnonotice`
     - `-value=`の後ろは適宜変更

5. 「条件」タブ:
   - すべてのチェックを外す

6. 「OK」をクリック

同様に `DDPM_HDMI2`、`DDPM_DP` タスクも作成します。引数だけ変更すればOKです。
:::message
タスク名は batファイルから `schtasks /run /tn "タスク名"` で呼び出すときに使うので、間違えないように注意してください。
:::

### Stream Deck用のbatファイルを作成

タスクスケジューラのタスクを呼び出すbatファイルを作ります。

**HDMI1に切替（`switch-hdmi1.bat`）
```bat
@echo off
schtasks /run /tn "DDPM_HDMI1"
```

**HDMI2に切替（`switch-hdmi2.bat`）
```bat
@echo off
schtasks /run /tn "DDPM_HDMI2"
```

**DisplayPortに切替（`switch-dp.bat`）
```bat
@echo off
schtasks /run /tn "DDPM_DP"
```

batファイルは適当なフォルダにまとめておいてください。自分は `C:\Users\[ユーザー名]\bin` に置いてます。

### Stream Deckへの登録

1. Stream Deckアプリを開く
2. 右側のアクション一覧から 「システム」>「開く」をドラッグしてボタンに配置
3. 「App / ファイル」欄にbatファイルのパスを指定
4. タイトルやアイコンをお好みで設定

これで、Stream Deckのボタンを押すだけでモニターの入力が切り替わります。一瞬コマンドプロンプトのようなウィンドウが表示されますがすぐに消えます。

## 輝度調整もCLIから可能

入力切替がメインで調べていたんですが、輝度やコントラストもCLIで変更できるようです。自分の場合は、日中用と夜用で輝度を使い分けているのでそれだけ登録しています。

### タスクスケジューラでタスクを作成

入力切替と同じ手順でタスクを作成します。

- タスク名: `DDPM_BrightnessLow`
- 引数: `/set -Display=BrightnessLevel -value=20,forcewithnonotice`

- タスク名: `DDPM_BrightnessHigh`
- 引数: `/set -Display=BrightnessLevel -value=80,forcewithnonotice`

詳しい手順は入力切替のところと同じなので省略します。

### Stream Deck用のbatファイル

輝度を低く（20%）（`brightness-low.bat`）

```bat
@echo off
schtasks /run /tn "DDPM_BrightnessLow"
```

輝度を高く（80%）（`brightness-high.bat`）

```bat
@echo off
schtasks /run /tn "DDPM_BrightnessHigh"
```

他にも使えるパラメータがあるので、メモしておきます。

| パラメータ | 説明 | 値の例 |
|---|---|---|
| `ContrastLevel` | コントラスト | 0-100 |
| `SpeakerVolume` | スピーカー音量 | 0-100 |
| `PowerSetting` | 電源 | On, Off, Standby |
| `ColorPreset` | カラープリセット | standard, movie, game |

### 応用：輝度を変数化する

上記の方法だと、輝度値ごとにタスクを作る必要があります。10%, 30%, 50%, 70%, 100%みたいに細かく調整したい場合、タスクが大量に増えてしまいます。

とりあえずPowerShellスクリプトと一時ファイルを使うことで、1つのタスクで任意の輝度値を設定できるようにしてみました。

#### PowerShellスクリプトを作成

`set-brightness.ps1` というファイルを作成します。自分は `C:\Users\[ユーザー名]\bin\ddm\` に置いています。

```powershell
$tempFile = 'C:\Users\[ユーザー名]\AppData\Local\Temp\ddpm_brightness.txt'
if (Test-Path $tempFile) {
    $brightness = (Get-Content $tempFile).Trim()
    $exe = 'C:\Program Files\Dell\Dell Display and Peripheral Manager\Plugins\Subagent\CLI.Subagent.exe'
    & $exe /set "-Display=BrightnessLevel" "-value=$brightness,forcewithnonotice"
    Remove-Item $tempFile
}
```

`.Trim()` が重要です。batファイルの `echo` コマンドは末尾にスペースを含めてしまうことがあって、これがないと `FormatException` で失敗します。

#### タスクスケジューラでタスクを作成

入力切替と同じ手順で、今度は1つだけタスクを作成します。

- タスク名: `DDPM_SetBrightness`
- プログラム/スクリプト: `powershell.exe`
- 引数: `-ExecutionPolicy Bypass -File "C:\Users\[ユーザー名]\bin\ddm\set-brightness.ps1"`

一応、他の設定（「最上位の特権で実行する」など）は入力切替と同じです。

#### batファイルで輝度値を指定

batファイルで輝度値を書き込んでから、タスクを実行します。

輝度10%（`brightness-10.bat`）

```bat
@echo off
echo 10 > C:\Users\[ユーザー名]\AppData\Local\Temp\ddpm_brightness.txt
schtasks /run /tn "DDPM_SetBrightness"
```

輝度50%（`brightness-50.bat`）

```bat
@echo off
echo 50 > C:\Users\[ユーザー名]\AppData\Local\Temp\ddpm_brightness.txt
schtasks /run /tn "DDPM_SetBrightness"
```

数値部分を変えるだけで、0〜100の任意の輝度値を設定できます。Stream Deckに複数のボタンを登録すれば、細かい輝度調整が可能です。

この方法のメリットは、タスクが1つで済むことと、後から輝度の値を変更するのが楽なことです。  
デメリットは、PowerShellスクリプトの管理が必要になることと、一時ファイルを経由するので少し複雑になることです。個人的には、よく使う輝度が2〜3段階なら固定値方式、5段階以上使いたいなら変数化がいい感じだと思いますが、後から変更しそうな気もするので変数化したものを利用しています。


## 旧バージョン（DDM 2.x）について

DDM 2.xは2025年5月にEOLになっています。旧バージョンのコマンド構文は以下の通りでした。

**DDM 2.x の構文例:```bat
"C:\Program Files\Dell\Dell Display Manager 2\DDM.exe" /writeactiveinput HDMI1
"C:\Program Files\Dell\Dell Display Manager 2\DDM.exe" /writebrightnesslevel 50
```

### DDM 2.xとDDPMのコマンド対応表

| 機能 | DDM 2.x | DDPM |
|---|---|---|
| 入力切替 | `/writeactiveinput HDMI1` | `/set -Display=ActiveInputSource -value=HDMI1,forcewithnonotice` |
| 輝度変更 | `/writebrightnesslevel 50` | `/set -Display=BrightnessLevel -value=50,forcewithnonotice` |
| コントラスト | `/writecontrastlevel 70` | `/set -Display=ContrastLevel -value=70,forcewithnonotice` |
| 音量 | `/writevolume 50` | `/set -Display=SpeakerVolume -value=50,forcewithnonotice` |
| 電源 | `/writepower off` | `/set -Display=PowerSetting -value=Off,forcewithnonotice` |

DDM 2.xが既にインストールされている環境ならそのまま使い続けてもOKですが、DDM 2.xはダウンロードリンクが削除されているので、PCを入れ替えたりする時にはDDPMに移行する必要があります。  
ちなみにDDM 2.xでは管理者権限は不要のはずでした。このせいでめっちゃメンドかった・・・

## 参考リンク

- [DDPM CLI for macOS (GitHub Gist)](https://gist.github.com/eliasschr/7889a0e988f3e5778a2de14ab4c83313)
- [Dell Display and Peripheral Manager for Windows](https://www.dell.com/support/kbdoc/en-us/000287285/dell-display-and-peripheral-manager-for-windows)
- [DDPM CLI confirmation dialog issue (Dell Community)](https://www.dell.com/community/en/conversations/monitors/u3421we-ddpm-21112-cli-command-requires-confirmation-from-user-via-gui/68baa9b8c5deda10cc1e1d11)
- [Dell Display Manager 2.0 CLI Commands (GitHub Gist)](https://gist.github.com/nebriv/cb934a3b702346c5988f2aba5ee39f0d)
