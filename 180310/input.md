# デーモンよもやま話

## はじめに

- [daemon(7)](https://www.freedesktop.org/software/systemd/man/daemon.html) を頑張って解説します

- 完全に理解してる人は説明要らないと思うし、むしろ解説してほしい

## はじめに

- 内容への質問はいつでもどうぞ

- 内容外の質問は最後にお願いします

## デーモンとは何か

突然ですが「デーモン」って説明できますか?

## こっちではないです

![デーモン小暮閣下](./images/kogure.jpeg)

## `Wikipedia`先生

「デーモン (英語: Daemon) は、UNIX, Linux, MacOSXなどUnix系ののマルチタスクオペレーティングシステム (OS) において動作するプロセス（プログラム）で、主にバックグラウンドで動作するプロセス[^wikipedia_daemon]」

[^wikipedia_daemon]: https://ja.wikipedia.org/wiki/%E3%83%87%E3%83%BC%E3%83%A2%E3%83%B3_(%E3%82%BD%E3%83%95%E3%83%88%E3%82%A6%E3%82%A7%E3%82%A2)

## じゃあこういうこと?

```
nohup /path/to/your/daemon &
```

「コマンドを実行している際に、仮想端末（Terminal）の画面を閉じたりログアウトしたりすると、実行中のコマンドも終了してしまいます（コマンドをバックグラウンド実行していても終了する）。
コマンド起動時に「nohup コマンド &」と指定することで、このような場合でもそのままコマンドの実行を続けることができます。[^nohup]」

[^nohup]: http://www.atmarkit.co.jp/ait/articles/1708/24/news022.html

## じゃあこういうこと?

```
nohup /path/to/your/daemon &
```

まぁ、事足りるならええんやけど……。

## 問題点
例えば、こういう不都合もありまして。

    $ nohup sleep 500 >/dev/null 2>&1 &
    [1] 21924
    $ ps
    PID TTY          TIME CMD
    19643 pts/3    00:00:00 bash
    21924 pts/3    00:00:00 sleep
    21964 pts/3    00:00:00 ps
    $ fg
    nohup sleep 500 > /dev/null 2>&1
    ^C
    $ ps
    PID TTY          TIME CMD
    19643 pts/3    00:00:00 bash
    22079 pts/3    00:00:00 ps

## 守護神の親は?

「Unix系の環境では、常にではないが、デーモンの親プロセスはinitプロセスとなっていることが多い。[^wikipedia_daemon]」

じゃあ、initって何なのさ?

# `init`

## PID = 1

initはPID(プロセス)が1のプロセスである

## PID = 1

じゃあ、こういうのもありですね!!![^emergency]

```
$ grep "\slinux\s" /boot/grub/grub.cfg
...
linux [...] init=/bin/bash
...
```

[^emergency]: http://www.atmarkit.co.jp/ait/articles/0809/30/news148.html

## `init`とは

- カーネルから直接起動させるプロセスである
- すべてのプロセスの親である
- 各種プログラムやデーモンを起動させる

## `init`とは

### カーネルから直接起動させるプロセスである

ここで、Linuxが起動(ブート)するまでをおさらい

## ブートストラップ

ジーニアス和英大辞典

    bootstrap [名]
    1. (長靴の口の後ろにある)つまみ皮.

![ブートストラップイメージ図](./images/bootstrap.jpg)

## ざっくり版`Linux`の起動

1. BIOS/UEFIがハードウェアをいい感じに初期化する
1. BIOS/UEFIが所定の位置にあるブートストラップローダーを読み込む
1. ブートストラップローダ―がカーネルを起動させる
1. initramfs(「ミニルート」)をメモリ上に展開する[^weekly_recipe_0348]

[^weekly_recipe_0348]: http://gihyo.jp/admin/serial/01/ubuntu-recipe/0384

## ざっくり版`Linux`の起動(続き)
5. 各種カーネルモジュール(デバイスドライバ)をロードする
1. 本物のルートファイルシステムに移動する[^weekly_recipe_0348]
1. カーネルがinitを起動させる
1. あとはよしなに

## `init`とは

### カーネルから直接起動させるプロセスである

だから、PIDが1になる(と思っておけば良い)

## `init`とは

### すべてのプロセスの親である

## プロセスの誕生

プロセス = 

メモリ上に展開された実行可能なバイナリー

\+

動的な環境

(超ザックリ理解)

## プロセスの誕生

動的な環境とは

- 環境変数
- ファイル記述子
- 自分のプロセスID
- 親プロセスのID

...etc

## プロセスの誕生

新しいプロセスに動的な環境を引き渡す必要がある

UNIXでは「既存プロセスの環境をまるごとコピーする」という手法で実装している

## プロセスの誕生

具体的には、以下の2つのシステムコールで実現:

- `fork()`: 親プロセスが自分自身をコピーして子プロセスを生成する
- `exec()`: 親プロセスが生成した子プロセスを書き換え、目的のプロセスを実行する

## プロセスの誕生

これにより「新規プロセスは既存プロセスの子プロセスとして生成されるため、親プロセスなしで新規プロセスを生成することはできない」という制約が生まれる[^unix_mag]

[^unix_mag]: 「次世代のプロセス管理デーモン 進化するinit」UNIX magazine 2009年1月号。

## `init`とは

### すべてのプロセスの親である

ちなみに、親プロセスを失ったみなしごプロセスの親はinitになる

## `init`とは

### 各種プログラムやデーモンを起動させる

## `BSD init`の場合

システムの設定ファイルとして2つのシェルスクリプトが使われる

- `/etc/rc`
- `/etc/rc.local`
    - システム起動時は`/etc/rc`にキックされる

## `BSD init`の場合

さすがにハードコアすぎる

## `System V init`の場合

ランレベルの概念が導入されている

ランレベルのモード、全部言えるかな?

## `System V init`の場合

システム起動時

- `/etc/rc.${runlevel}`にあるスクリプトのうち
- 頭文字がSのものを
- その後の数字の順番に読み込み
- 実行する

## `System V init`の場合

終了もサポートしている

SがKに変わるだけ

## `System V init`の場合

BSD initより柔軟

- ランレベルも簡単に使えるしね!

## `systemd`の場合

「SysV initはそりゃ、BSD initよりは柔軟だけど……

ブート時にすべて、一律にやる必要があるのか?」

→もう少し柔軟性・速度がほしい

## `systemd`の場合

終了ってどうすんの?

→各プログラムに任せる

そもそも、initの仕事でなかったりする

## `systemd`の場合

デーモンの終了も面倒を見てくれる[^systemd_for_admin_2]

- SysV initの場合、デーモンの子プロセスのkillはデーモン側の仕事
    - 行儀の悪いデーモンは子プロセスを残す
    - SysV initはどれがどの子かがわからない
- cgroupを使う

[^systemd_for_admin_2]: http://popopopoon.hatenadiary.jp/entry/2017/11/19/005612

## `init`とは

### 各種プログラムやデーモンを起動させる

- systemdは終了も面倒をみるよ
- systemdは起動のさせ方も柔軟だよ

# デーモンづくり

## `System V init`でデーモン

### `stdin` `stdout` `stderr`を除くすべてのファイル記述子を閉じる

これにより、想定外のファイル記述子がデーモンプロセスにとどまってしまうのを防ぐ

## `System V init`でデーモン

### シグナルハンドラ―をすべてデフォルトに戻す

シグナルとは

- SIGKILL
- SIGHUP

とか[^signals]。
これらを受信したときに特定の動作をおこなわせるのがシグナルハンドラ[^signal_handler]

[^signals]: https://linuxjm.osdn.jp/html/LDP_man-pages/man7/signal.7.html
[^signal_handler]: https://codezine.jp/article/detail/4700

## `System V init`でデーモン
### シグナルマスクのリセット

シグナルはブロックすることができる

シグナルマスクとはブロックしているシグナルの集合を示すもの

## `System V init`でデーモン
### 動的環境のクリーンアップ

デーモンの実行に悪影響を及ぼしそうなものを除外する


## `System V init`でデーモン
### `fork()`システムコール

## `System V init`でデーモン
### 子プロセス内で`setid()`システムコール

すべての制御端末からプロセスを切り離し、独立したセッションを作成する[^setsid]

[^setsid]: https://linuxjm.osdn.jp/html/LDP_man-pages/man2/setsid.2.html 「呼び出したプロセスは、 新しいプロセスグループと新しいセッションの唯一のプロセスとなる。 新しいセッションは制御端末を持たない。」

## `System V init`でデーモン
### 子プロセス内で2回目の`fork()`システムコール

デーモンは完全に制御端末を獲得することができなくなる

## `System V init`でデーモン
### 最初の子プロセスで`exit()`システムコール

- 呼び出し元からみて子プロセスが死に、孫プロセスはみなしごになる
- 親を失った孫プロセスはinitを親にする

## `System V init`でデーモン
### `stdin` `stdout` `stderr` を `/dev/null` に接続

## `System V init`でデーモン
### `umask`を0に設定

`open()`や`mkdir()`といった呼び出しへ`umask`が影響を及ぼさないようにする

## `System V init`でデーモン
### `cd /`を実行

ディレクトリが使用中にならないようにする

## `System V init`でデーモン
### PIDファイルの作成

デーモンの二重起動を防止させる

## `System V init`でデーモン
### 適切なレベルまで特権を降格

## `System V init`でデーモン
### 起動完了を呼び出し元プロセスに通知

この通知は最初の`fork()`より前につくられている

(この辺からわからんw)

## `System V init`でデーモン
### 呼び出し元プロセスで`exit()`

## `System V init`でデーモン
以上

## `System V init`でデーモン
ちょっと強引すぎやしませんかね……

(小並感)

## `systemd`先生登場

上に書いたようなややこしいことを全部、systemdがやってくれる

- 動的環境のクリーンアップ
- シグナルハンドラ―とマスクのリセット
- `stdin`を`/dev/null`に接続
- `stdout` `stderr` を`systemd-journald.service`につなぐ

## ユニット

脱・シェルスクリプト

→ユニットファイルとして諸々の設定を記述・管理

- .service
- .target
- .socket

...etc

## ユニット

```
$ cat /lib/systemd/system/ssh.service
...
[Unit]
Description=OpenBSD Secure Shell server
After=network.target auditd.service
ConditionPathExists=!/etc/ssh/sshd_not_to_be_run

[Service]
EnvironmentFile=-/etc/default/ssh
ExecStartPre=/usr/sbin/sshd -t
ExecStart=/usr/sbin/sshd -D $SSHD_OPTS
ExecReload=/usr/sbin/sshd -t
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
Restart=on-failure
RestartPreventExitStatus=255
Type=notify
RuntimeDirectory=sshd
RuntimeDirectoryMode=0755

[Install]
WantedBy=multi-user.target
Alias=sshd.service
```

## デーモンの起動

デーモンの起動方法が選べる

- ブート時
- ソケットベース
- バスベース
- デバイスベース
- パスベース
- タイマーベース

## ユニットの依存関係

```
systemctl list-dependencies graphical.target
...
(ずらー)
```
[bootup(7)](https://www.freedesktop.org/software/systemd/man/bootup.html)

# 終了!

## 多分話しすぎているはず!w

- 質問・雑談タイム