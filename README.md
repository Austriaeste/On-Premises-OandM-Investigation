# On-Premises-O-M-Investigation
このREADMEでは、オンプレミス環境でのトラブルシューティングに役立つ一連のコマンドを紹介します。
これらのコマンドは、主にLinux環境（RHEL6, RHEL7, RHEL8）で使用できます。

## アクセス数の確認

### 目的
Apacheサーバーのアクセス数を確認し、異常なアクセス増加を検出します。

### 環境
- Linux (RHEL6, RHEL7, RHEL8)
- Cisco、BIG-IP

### コマンド

#### 1. httpd プロセスが再起動を起こしているかどうかを確認
```bash
ps -ef | grep httpd | grep root | grep -v grep
```
#### 2. 80番ポートが LISTEN 状態であるかを確認
```bash
netstat -anp | grep :80 | grep LISTEN
```
#### 3. 初めに確認した PID に紐づくログの場所を見てみましょう
```bash
lsof -p <PID> | grep "log"
```
#### 4.エラーログを確認
```bash
cat /var/log/httpd/error_log | grep "Sep 03 23:1[3-6]"
```
#### 5.アクセス数を確認
```bash
<10分単位>
for N in {0,1,2,3,4,5} ;do echo "23:$N";grep "28/Apr/2018:23:$N" /etc/httpd/logs/access_log | wc -l ;done

<1時間単位>
for N in {0..23} ;do echo "$N:00";grep "25/Sep/2024:$N:" /var/log/apache2/access.log | wc -l ;done

<1週間単位>
for N in {18..24} ;do echo "2024-09-$N";grep "2024:09:$D" /var/log/apache2/access.log | wc -l ;done

<1か月単位>
for N in {01..30} ;do echo "2024-09-$N";grep "2024:09:$N" /var/log/apache2/access.log | wc -l ;done
```
## ディスク使用量の確認

### 目的
ディスクの使用量を確認し、容量を圧迫しているファイルやディレクトリを特定します。

### 環境
- Linux (RHEL6, RHEL7, RHEL8)

### コマンド

#### 1. 現在の使用量を確認
```bash
df -h <対象領域>
```
#### 2. 容量を圧迫しているベスト10位までを表示
```bash
du -hx <対象領域> | sort -nr | head
```
#### 3. 容量を圧迫しているものがファイルかディレクトリかを特定
```bash
ls -lt <前項で表示したもの>
```


## LB（ロードバランサー）の確認

### 目的
ロードバランサーのステータスを確認し、正常に動作しているかを確認します。

### 環境
F5 BIG-IP

### コマンド

#### 1. 5秒後に upしている事を確認
```bash
cat /var/log/ltm | grep "monitor status"
```
#### 2. available 状態である事を確認
```bash
[root@bigip:Active] config#tmsh show ltm pool http-pool members | grep -A30 "Ltm"::Pool:http-pool
Ltm::Pool: http-pool
--------------------------------------
Status
  Availability : available
  State        : enabled
  Reason       : The pool is available

Traffic                ServerSide
  Bits In                   37.3K
  Bits Out                  52.0K
  Packets In                   64
  Packets Out                  43
  Current Connections           0
  Maximum Connections           2
  Total Connections            15

Ltm::Pool Member: http-pool  10.10.40.40:012345
-------------------------------------------
Status
  Availability : available
  State        : enabled
  Reason       : Pool member is available

Traffic                ServerSide  General
  Bits In                   37.3K        -
  Bits Out                  52.0K        -
  Packets In                   64        -
  Packets Out                  43        -
  Current Connections           0        -
  Maximum Connections           2        -
  Total Connections            15        -
  Total Requests                -       10
  [root@bigip:Active] config#
  ```
## Cisco機器の確認

### 目的
Cisco機器のステータスを確認し、異常がないかを確認します。

### 環境
Cisco

### コマンド

#### 0. 検知メッセージより、neighbor ルータの IP アドレスが 192.168.3.1 である事を確認

#### 1. Tunnel のステータスが「up」している事を確認
```bash
show ip interface brief

```
#### 2. Routerの稼働系と待機系にて再起動が発生していない事を確認
```bash
show version | inc uptime
```
#### 3. CPU 負荷の高騰は見受けられない事を確認
```bash
show process cpu history
```
#### 4. メモリ負荷の高騰は見受けられない事を確認
```bash
show mem summary
```
#### 5. neighbor の機器は XX である事を確認
```bash
show run | inc 192.168.3.1
```
#### 6. 現状 BGP のステータスは「up」している事を確認
```bash
show logging
```
