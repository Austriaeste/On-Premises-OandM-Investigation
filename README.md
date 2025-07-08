# On-Premises-O-M-Investigation

このREADMEでは、オンプレミス環境でのトラブルシューティングに役立つ一連のコマンドを紹介します。これらのコマンドは、主にLinux環境（RHEL6、RHEL7、RHEL8）で使用できます。おまけとして、CiscoおよびF5 BIG-IPロードバランサーにも触れています。各コマンドには期待される出力例を記載し、結果の解釈を補助します。

## アクセス数の確認

### 目的
Apacheサーバーのアクセス数を確認し、異常なアクセス増加を検出します。

### 環境
- Linux (RHEL6, RHEL7, RHEL8)

### コマンド

#### 1. httpdプロセスが再起動を起こしているかどうかを確認
```bash
ps -ef | grep httpd | grep root | grep -v grep
```
**出力例**:
```
root      1234     1  0  Sep03 ?        00:00:00 /usr/sbin/httpd -k start
```
**説明**: `httpd`プロセスがroot権限で実行中であることを確認。PID（例：1234）をメモして後続のコマンドで使用。

#### 2. 80番ポートがLISTEN状態であるかを確認
```bash
ss -tuln | grep :80
# または、RHEL6など古い環境では
netstat -anp | grep :80 | grep LISTEN
```
**出力例**:
```
tcp    LISTEN     0      128       0.0.0.0:80        0.0.0.0:*      prog=/httpd
```
**説明**: 80番ポートがLISTEN状態であれば、Apacheが正常に稼働中。出力がない場合は、`systemctl status httpd`でサービス状態を確認。

#### 3. 初めに確認したPIDに紐づくログの場所を確認
```bash
lsof -p <PID> | grep "log"
```
**出力例**:
```
httpd 1234 root  2w   REG   8,1    123456  /var/log/httpd/error_log
httpd 1234 root  3w   REG   8,1    789012  /var/log/httpd/access_log
```
**説明**: 指定したPID（例：1234）のプロセスが使用しているログファイルを確認。ログパスを特定し、後の解析に使用。

#### 4. エラーログを確認
```bash
cat /var/log/httpd/error_log | grep "Sep 03 23:1[3-6]"
```
**出力例**:
```
[Wed Sep 03 23:13:45.123456 2024] [core:error] [pid 1234] [client 192.168.1.1] File does not exist: /var/www/html/missing.html
```
**説明**: 指定した時間帯（例：23:13～23:16）のエラーログを確認。エラー原因（例：ファイル不存在など）を特定。

#### 5. アクセス数を確認
**注意**: ログパスは環境により異なる（例：RHELでは`/var/log/httpd/access_log`、Debian系では`/var/log/apache2/access.log`）。まず`head /var/log/httpd/access_log`でログフォーマットを確認。

```bash
# 10分単位
for N in {0..5} ; do echo "23:$N"; grep "28/Apr/2018:23:$N" /var/log/httpd/access_log | wc -l ; done
```
**出力例**:
```
23:0  150
23:1  200
23:2  180
23:3  250
23:4  300
23:5  175
```
**説明**: 各10分間（23:00～23:05）のアクセス数を表示。急増している時間帯を特定。

```bash
# 1時間単位
for N in {0..23} ; do printf "%02d:00 " $N; grep "25/Sep/2024:$N:" /var/log/httpd/access_log | wc -l ; done
```
**出力例**:
```
00:00  500
01:00  450
02:00  600
...
23:00  1200
```
**説明**: 各時間のアクセス数を表示。異常なスパイク（例：23:00の1200件）を調査。

```bash
# 1週間単位
for N in {18..24} ; do echo "2024-09-$N"; grep "2024-09-$N" /var/log/httpd/access_log | wc -l ; done
```
**出力例**:
```
2024-09-18  10000
2024-09-19  12000
2024-09-20  15000
...
2024-09-24  20000
```
**説明**: 1週間の日毎のアクセス数を表示。特定の日に異常な増加がないか確認。

```bash
# 1か月単位
for N in {01..30} ; do printf "2024-09-%02d " $N; grep "2024-09-$N" /var/log/httpd/access_log | wc -l ; done
```
**出力例**:
```
2024-09-01  30000
2024-09-02  31000
...
2024-09-30  35000
```
**説明**: 1か月の日毎のアクセス数を表示。長期的なトレンドを把握。

## ディスク使用量の確認

### 目的
ディスクの使用量を確認し、容量を圧迫しているファイルやディレクトリを特定します。

### 環境
- Linux (RHEL6, RHEL7, RHEL8)

### コマンド

#### 1. 現在の使用量を確認
```bash
df -h /var
```
**出力例**:
```
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1       100G   85G   15G  85% /var
```
**説明**: `/var`の使用量を確認。使用率（例：85%）が高い場合は、次のコマンドで詳細を調査。

#### 2. 容量を圧迫しているベスト10を表示
```bash
du -hx /var | sort -nr | head
```
**出力例**:
```
1.2G    /var/log
500M    /var/www/html
300M    /var/cache
...
```
**説明**: `/var`以下のディレクトリで容量を多く使用しているものを降順で表示。例：`/var/log`が1.2GBを占めている場合、ログファイルの肥大化を疑う。

#### 3. 容量を圧迫しているものがファイルかディレクトリかを特定
```bash
ls -lh /var/log
```
**出力例**:
```
-rw-r--r-- 1 root root 1.1G Sep 03 23:15 httpd/error_log
drwxr-xr-x 2 root root 4.0K Sep 03 22:00 httpd
```
**説明**: 容量の大きいディレクトリ（例：`/var/log`）を調査。大きなファイル（例：`error_log`）やディレクトリを特定し、必要に応じて削除や圧縮を検討。

## CPU使用率の確認

### 目的
CPU使用率を確認し、異常な負荷がかかっていないかを確認します。

### コマンド

#### 1. 今日の00:00から現在時刻までのCPU使用率の推移を確認
```bash
sar -u
```
**出力例**:
```
00:00:01    CPU  %user  %nice %system %iowait  %steal  %idle
00:10:01    all   10.0   0.0    5.0     2.0     0.0   83.0
00:20:01    all   15.0   0.0    7.0     3.0     0.0   75.0
...
```
**説明**: CPU使用率（例：`%user`や`%system`）が高い時間帯を特定。`%iowait`が高い場合はディスクI/Oがボトルネック可能性。

#### 2. cronジョブの確認
```bash
crontab -l
```
**出力例**:
```
0 0 * * * /usr/local/bin/backup.sh
```
**説明**: 現在のユーザのcronジョブを表示。深夜0時に実行されるスクリプト（例：`backup.sh`）がCPU負荷の原因かを調査。

#### 3. スケジューラを所有しているユーザ名を確認
```bash
ls -al /var/spool/cron/
```
**出力例**:
```
-rw------- 1 root root  123 Sep 03 10:00 root
-rw------- 1 user1 user1 456 Sep 03 09:00 user1
```
**説明**: cronファイルの所有者を確認。rootや他のユーザのジョブを後で調査。

#### 4. アラート検知時間帯にバッチ処理が走っていたかを確認
```bash
cat /var/spool/cron/root
```
**出力例**:
```
0 23 * * * /usr/local/bin/heavy_script.sh
```
**説明**: rootユーザのcronジョブを確認。例：23時に`heavy_script.sh`が実行されていれば、CPU負荷の原因の可能性。

#### 5. アラートの発生原因がcron実行によるものかを確認
```bash
cat /var/log/messages | grep "cron failed"
# または、RHEL7/8では
journalctl -u cron | grep "failed"
```
**出力例**:
```
Sep 03 23:00:01 hostname cron[5678]: (root) CMD (/usr/local/bin/heavy_script.sh) failed
```
**説明**: cronジョブの失敗ログを確認。失敗したジョブがCPU負荷やエラーの原因かを特定。

## メモリ使用率の確認

### 目的
メモリ使用率を確認し、異常な使用がないかを確認します。

### コマンド

#### 1. メモリ使用率を確認
```bash
# RHEL6
free -k | grep "Mem" | awk '{print ($2-$4-$6-$7)/$2*100}'
# RHEL7, RHEL8
free -k | grep "Mem" | awk '{print ($2-$4-$6)/$2*100}'
```
**出力例**:
```
75.5
```
**説明**: メモリ使用率をパーセンテージで表示（例：75.5%）。80%を超える場合は、メモリリークやプロセス過多を疑い、`top`や`htop`で詳細を確認。

## LB（ロードバランサー）の確認

### 目的
ロードバランサーのステータスを確認し、正常に動作しているかを確認します。

### 環境
F5 BIG-IP

### コマンド

#### 1. 5秒後にupしている事を確認
```bash
cat /var/log/ltm | grep "monitor status"
```
**出力例**:
```
Sep 03 23:00:01 bigip monitor status: pool http-pool member 10.10.40.40:80 up
```
**説明**: プールメンバーが「up」であることを確認。`down`が表示された場合は、該当サーバのヘルスチェック失敗を疑う。

#### 2. available状態である事を確認
```bash
tmsh show ltm pool http-pool members | grep -A30 "Ltm::Pool:http-pool"
```
**出力例**:
```
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
Ltm::Pool Member: http-pool  10.10.40.40:80
-------------------------------------------
Status
  Availability : available
  State        : enabled
  Reason       : Pool member is available
```
**説明**: プールとメンバーが`available`であることを確認。`unavailable`の場合は、サーバの接続性やアプリケーションエラーを調査。

## Cisco機器の確認

### 目的
Cisco機器のステータスを確認し、異常がないかを確認します。

### 環境
Cisco

### コマンド

#### 0. 検知メッセージより、neighborルータのIPアドレスが192.168.3.1である事を確認

#### 1. Tunnelのステータスが「up」している事を確認
```bash
show ip interface brief
```
**出力例**:
```
Interface              IP-Address      OK? Method Status                Protocol
Tunnel0                192.168.1.1     YES manual up                    up
```
**説明**: `Status`と`Protocol`が両方`up`であることを確認。`down`の場合は、トンネル設定や接続性を調査。

#### 2. Routerの稼働系と待機系にて再起動が発生していない事を確認
```bash
show version | inc uptime
```
**出力例**:
```
uptime is 3 weeks, 2 days, 5 hours, 10 minutes
```
**説明**: 長期間の稼働時間を確認。短い場合は、予期しない再起動を疑い、`show log`で詳細を確認。

#### 3. CPU負荷の高騰は見受けられない事を確認
```bash
show process cpu history
```
**出力例**:
```
    11111
100
 90
 80
 70
 60
 50
 40
 30
 20
 10 #####
   0....5....1....1....2....2....3....3....4....4....5....5....6
         0    5    0    5    0    5    0    5    0    5    0
       CPU% per hour (last 60 hours)
```
**説明**: CPU使用率の履歴を確認。高いスパイク（例：50%超）があれば、原因プロセスを`show process cpu sorted`で調査。

#### 4. メモリ負荷の高騰は見受けられない事を確認
```bash
show memory summary
```
**出力例**:
```
                Used    Free
Processor      500M    1.5G
I/O            200M    300M
```
**説明**: `Free`メモリが十分か確認。`Free`が極端に少ない場合は、メモリリークやプロセス過多を疑う。

#### 5. neighborの機器はXXである事を確認
```bash
show run | inc 192.168.3.1
```
**出力例**:
```
neighbor 192.168.3.1 remote-as 65001
```
**説明**: 指定したIP（192.168.3.1）がBGPネイバーとして設定されていることを確認。

#### 6. 現状BGPのステータスは「up」している事を確認
```bash
show logging
```
**出力例**:
```
Sep 03 23:00:01: %BGP-5-ADJCHANGE: neighbor 192.168.3.1 Up
```
**説明**: BGPセッションが「Up」であることを確認。「Down」があれば、ネイバー設定やネットワーク接続を調査。
