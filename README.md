# On-Premises-O-M-Investigation

このREADMEでは、オンプレミス環境でのトラブルシューティングに役立つ一連のコマンドを紹介します。これらのコマンドは、主にLinux環境（RHEL6、RHEL7、RHEL8）で使用でき、F5 BIG-IP、Cisco機器（RADIUS認証を含む）、Postfixメールサーバーに対応しています。各コマンドには期待される出力例と簡単な説明を記載し、問題の特定と解決を支援します。ApacheログパスはRHEL標準の`/var/log/httpd/`、Postfixログパスは`/var/log/maillog`、RADIUSログパスは`/var/log/radius/radius.log`を使用しますが、Debian系（`/var/log/apache2/`や`/var/log/mail.log`）の場合は適宜読み替えてください。

## 目次
- [アクセス数の確認](#アクセス数の確認)
- [ディスク使用量の確認](#ディスク使用量の確認)
- [CPU使用率の確認](#CPU使用率の確認)
- [メモリ使用率の確認](#メモリ使用率の確認)
- [ネットワーク接続性の確認](#ネットワーク接続性の確認)
- [サービスとデーモンの状態確認](#サービスとデーモンの状態確認)
- [ファイアウォールとSELinuxの確認](#ファイアウォールとSELinuxの確認)
- [システム全体のログ分析](#システム全体のログ分析)
- [Postfixメールキューの監視](#Postfixメールキューの監視)
- [DNSレコードとドメインの基礎](#DNSレコードとドメインの基礎)
- [送信ドメイン認証とドメインレピュテーション](#送信ドメイン認証とドメインレピュテーション)
- [LB（ロードバランサー）の確認](#LBロードバランサーの確認)
- [Cisco機器の確認](#Cisco機器の確認)
- [データベース調査](#データベース調査)

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
**注意**: ログパスやフォーマットは環境により異なる。まず`head /var/log/httpd/access_log`でログフォーマットを確認。

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

#### 4. ログローテーション設定の確認
```bash
cat /etc/logrotate.d/httpd
```
**出力例**:
```
/var/log/httpd/*log {
    weekly
    rotate 4
    compress
    delaycompress
    missingok
}
```
**説明**: Apacheログのローテーション設定を確認。`rotate 4`は4世代分保持することを意味し、ディスク圧迫の原因かを評価。必要に応じて`logrotate -f /etc/logrotate.d/httpd`で強制ローテーション。

#### 5. ログファイルサイズの確認
```bash
ls -lh /var/log/httpd/
```
**出力例**:
```
-rw-r--r-- 1 root root 1.5G Sep 03 23:15 access_log
-rw-r--r-- 1 root root 500M Sep 03 23:15 error_log
```
**説明**: ログファイルのサイズを確認。巨大なログファイルがあれば、ローテーション設定を見直すか、圧縮/削除を検討。

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

#### 6. プロセスごとのCPU/メモリ使用率
```bash
top -b -n 1
```
**出力例**:
```
  PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND
 1234 root      20   0  500m   200m   100m S  15.0  10.0   0:05.12 httpd
 5678 user1     20   0  300m   150m    80m S  10.0   7.5   0:03.45 mysqld
```
**説明**: 実行中のプロセスのCPUとメモリ使用率をリアルタイムで確認。`httpd`や`mysqld`など、高負荷プロセスを特定。

#### 7. 特定プロセスの詳細
```bash
ps -p <PID> -o pid,ppid,%cpu,%mem,cmd
```
**出力例**:
```
  PID  PPID %CPU %MEM CMD
 1234     1 15.0 10.0 /usr/sbin/httpd -k start
```
**説明**: 特定PIDのプロセス詳細を確認。親プロセス（PPID）やコマンドラインをチェックし、異常動作を調査。

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

## ネットワーク接続性の確認

### 目的
サーバやネットワーク機器間の接続性やパフォーマンスの問題を特定します。

### 環境
- Linux (RHEL6, RHEL7, RHEL8), Cisco, F5 BIG-IP

### コマンド

#### 1. 接続性の確認（ping）
```bash
ping -c 4 <target_ip>
```
**出力例**:
```
PING 192.168.1.1 (192.168.1.1) 56(84) bytes of data.
64 bytes from 192.168.1.1: icmp_seq=1 ttl=64 time=0.123 ms
64 bytes from 192.168.1.1: icmp_seq=2 ttl=64 time=0.134 ms
...
--- 192.168.1.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3004ms
```
**説明**: ターゲットIPへの到達性と応答時間を確認。パケットロスや高い遅延（例：100ms以上）があれば、ネットワーク問題を疑う。

#### 2. 経路の確認（traceroute）
```bash
traceroute <target_ip>
```
**出力例**:
```
traceroute to 192.168.1.1 (192.168.1.1), 30 hops max, 60 byte packets
 1  192.168.0.1  0.234 ms  0.245 ms  0.256 ms
 2  192.168.1.1  0.345 ms  0.356 ms  0.367 ms
```
**説明**: ターゲットまでの経路を確認。異常なホップやタイムアウトがあれば、ルーティング問題を調査。

#### 3. Ciscoでの接続性確認
```bash
ping <target_ip> repeat 5
```
**出力例**:
```
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.3.1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/2/5 ms
```
**説明**: CiscoデバイスからターゲットIPへのpingを行い、接続性を確認。「!」（成功）または「.」（失敗）を確認。

## サービスとデーモンの状態確認

### 目的
主要なサービス（例：Apache、cron、Postfix）の稼働状態を確認し、停止や異常を検出します。

### 環境
- Linux (RHEL6, RHEL7, RHEL8)

### コマンド

#### 1. サービス状態の確認（RHEL7/8）
```bash
systemctl status httpd
systemctl status postfix
```
**出力例**:
```
● httpd.service - The Apache HTTP Server
   Loaded: loaded (/usr/lib/systemd/system/httpd.service; enabled; vendor preset: disabled)
   Active: active (running) since Tue 2024-09-03 23:00:01 JST; 1 weeks ago
● postfix.service - Postfix Mail Transport Agent
   Loaded: loaded (/usr/lib/systemd/system/postfix.service; enabled; vendor preset: disabled)
   Active: active (running) since Tue 2024-09-03 23:00:01 JST; 1 weeks ago
```
**説明**: `httpd`および`postfix`が`active (running)`であることを確認。`failed`や`dead`の場合はログ（`/var/log/httpd/error_log`や`/var/log/maillog`）を調査。

#### 2. サービス再起動（必要時）
```bash
systemctl restart httpd
systemctl restart postfix
```
**出力例**:
```
[no output if successful]
```
**説明**: サービスに異常があれば再起動を試行。成功したか再び`systemctl status`で確認。

## ファイアウォールとSELinuxの確認

### 目的
ファイアウォールやSELinuxが通信やプロセスをブロックしていないかを確認します。

### 環境
- Linux (RHEL6, RHEL7, RHEL8)

### コマンド

#### 1. ファイアウォールルールの確認（RHEL7/8）
```bash
firewall-cmd --list-all
```
**出力例**:
```
public (active)
  services: http https smtp
  ports: 80/tcp 25/tcp
```
**説明**: 80番（HTTP）および25番（SMTP）ポートが許可されていることを確認。許可されていない場合は、`firewall-cmd --add-port=25/tcp --permanent`で追加。

#### 2. SELinuxの状態確認
```bash
getenforce
```
**出力例**:
```
Enforcing
```
**説明**: SELinuxが`Enforcing`の場合、ApacheやPostfixの動作を制限する可能性。必要に応じて`setenforce 0`で一時無効化し、動作を確認。

#### 3. SELinuxエラーログの確認
```bash
ausearch -m avc -ts today
```
**出力例**:
```
type=AVC msg=audit(1725326401.123:456): avc:  denied  { write } for  pid=1234 comm="httpd" path="/var/www/html/file"
```
**説明**: SELinuxによるアクセス拒否を調査。必要に応じてポリシー修正（`audit2allow`）を検討。

## システム全体のログ分析

### 目的
システム全体のログを調査し、アプリケーションやハードウェアの問題を特定します。

### 環境
- Linux (RHEL6, RHEL7, RHEL8)

### コマンド

#### 1. システムログの確認（RHEL7/8）
```bash
journalctl -p 3 -b
```
**出力例**:
```
Sep 03 23:00:01 hostname kernel: Out of memory: Kill process 1234 (httpd)
```
**説明**: エラーレベルのログ（`-p 3`）を確認。メモリ不足やカーネルエラーを特定。

#### 2. 最近のログをリアルタイム監視
```bash
journalctl -f
```
**出力例**:
```
Sep 03 23:15:01 hostname httpd[1234]: [error] client denied by server configuration
Sep 03 23:15:02 hostname postfix/smtpd[5678]: NOQUEUE: reject: RCPT from unknown[10.230.220.57]: 450 4.7.1 Client host rejected
```
**説明**: リアルタイムでログを監視し、問題発生時のエラーを即座に捕捉。

## Postfixメールキューの監視

### 目的
Postfixメールキューを確認し、滞留メールの数や原因（スパム、ブロックなど）を特定します。滞留状況を1分単位で監視し、メール送信の問題を調査します。

### 環境
- Linux (RHEL6, RHEL7, RHEL8), Postfix

### コマンド

#### 1. メールキューの全体状況を確認
```bash
mailq | awk 'BEGIN { RS = "" } { if ($0 !~ /^ *\(|^--/) { print $1, $7, $8, $9 } }'
```
**出力例**:
```
A1B2C3D4E5 [email protected] [email protected]
F6G7H8I9J0 [email protected] [email protected]
```
**説明**: `mailq`の出力を`awk`で整形し、キューID、送信元、送信先を表示。空行や区切り線（`--`）を除外し、見やすく整理。

#### 2. メールキューの滞留数を取得
```bash
mailq | tail -n 1
```
**出力例**:
```
-- 123 Kbytes in 45 Requests.
```
**説明**: キューの最後の行から、滞留メールの総数（例：45件）とデータサイズ（例：123KB）を確認。異常な増加（例：数百件以上）があれば、原因を調査。

#### 3. 1分単位で滞留状況を監視
```bash
while true; do date "+%Y-%m-%d %H:%M:%S"; mailq | tail -n 1; sleep 60; done
```
**出力例**:
```
2024-09-03 23:15:00
-- 123 Kbytes in 45 Requests.
2024-09-03 23:16:00
-- 130 Kbytes in 50 Requests.
2024-09-03 23:17:00
-- 145 Kbytes in 55 Requests.
```
**説明**: 1分ごとに現在時刻と滞留メール数を出力。急増（例：10分で100件増加）があれば、スパムやブロックの可能性を調査。

#### 4. 滞留メールの原因分析（ログ確認）
```bash
cat /var/log/maillog | grep "postfix/smtp.*status=deferred"
```
**出力例**:
```
Sep 03 23:15:01 hostname postfix/smtp[5678]: A1B2C3D4E5: to=<[email protected]>, relay=mx.google.com[172.217.194.26]:25, delay=3600, status=deferred (host mx.google.com[172.217.194.26] said: 450 4.7.1 Client host rejected: cannot find your reverse hostname)
```
**説明**: `status=deferred`で遅延メールを抽出し、理由を確認。例では、Googleが逆引きホスト名欠如で拒否（450 4.7.1）。他の可能性（例：`550 5.7.1`でスパム判定）も調査。

#### 5. スパムによる滞留の確認
```bash
cat /var/log/maillog | grep "reject.*spam"
```
**出力例**:
```
Sep 03 23:15:01 hostname postfix/smtpd[5678]: NOQUEUE: reject: RCPT from unknown[10.230.220.57]: 550 5.7.1 Service unavailable; client [10.230.220.57] blocked using zen.spamhaus.org
```
**説明**: スパムブロック（例：Spamhaus RBL）による拒否を特定。頻発するIPやドメインを調査し、必要に応じて`postfix/smtpd_recipient_restrictions`にRBL追加。

#### 6. Googleによるブロックの確認
```bash
cat /var/log/maillog | grep "google.*reject"
```
**出力例**:
```
Sep 03 23:15:01 hostname postfix/smtp[5678]: A1B2C3D4E5: to=<[email protected]>, relay=mx.google.com[172.217.194.26]:25, delay=3600, status=deferred (host mx.google.com[172.217.194.26] said: 550 5.7.26 Unauthenticated email from example.com is not accepted due to DMARC policy)
```
**説明**: GoogleのDMARCポリシーやSPF/DKIM不備によるブロックを確認。以下の認証設定を見直す。

## DNSレコードとドメインの基礎

### 目的
メール送信やウェブサーバーの設定において、DNSレコード（Aレコード、MXレコードなど）の役割を理解し、独自ドメインやサブドメインの設定を適切に行うことで、トラブルシューティングや認証設定（SPF、DKIM、DMARC）を円滑に進める。

### 環境
- Linux (RHEL6, RHEL7, RHEL8), Postfix, DNSサーバ（BINDなど）

### 概要
DNS（Domain Name System）は、ドメイン名（例: `example.com`）をIPアドレスや他のリソースに変換する仕組みです。AレコードやMXレコードなどのDNSレコードは、ドメインの役割（ウェブ、メールなど）を定義します。独自ドメインとサブドメインは、DNS設定を通じて柔軟に運用可能で、メール送信時の認証やサービスの分離に重要です。

#### DNSレコードの主要な種類
以下の表は、メールサーバーやウェブサーバー運用でよく使うDNSレコードの概要です。

| レコード種類 | 説明 | 例 | 用途 |
|-------------|------|----|------|
| **A** | ドメインまたはサブドメインをIPv4アドレスに紐づける。「住所」の役割を持ち、ウェブやメールサーバーの基盤となる。 | `example.com. IN A 192.168.1.1` | `example.com`にアクセスしたとき、192.168.1.1のサーバーに接続。 |
| **MX** | ドメインのメールを受信するサーバーを指定。優先順位（数値が小さいほど優先）付きで設定。 | `example.com. IN MX 10 mail.example.com.` | `example.com`宛のメールを`mail.example.com`（優先度10）で処理。 |
| **TXT** | テキスト情報を格納。SPF、DKIM、DMARCなどの認証設定に使用。 | `example.com. IN TXT "v=spf1 ip4:192.168.1.1 ~all"` | SPFで送信元IP（192.168.1.1）を認証。 |
| **CNAME** | ドメインまたはサブドメインを別のドメイン名にエイリアス（別名）として紐づける。 | `www.example.com. IN CNAME example.com.` | `www.example.com`を`example.com`と同じサーバーに接続。 |
| **PTR** | IPアドレスをドメイン名に逆引きする。メールサーバーの信頼性向上に必要。 | `1.1.168.192.in-addr.arpa. IN PTR mail.example.com.` | 192.168.1.1が`mail.example.com`として解決される。 |
| **NS** | ドメインのDNSサーバーを指定。ドメインの管理権限を定義。 | `example.com. IN NS ns1.example.com.` | `example.com`のDNSを`ns1.example.com`が管理。 |

**注意**: 
- Aレコードはドメインの基盤であり、ウェブやメールサーバーの接続に必須。MXレコードはメール専用で、Aレコードを参照する（例: `mail.example.com`のAレコードが必要）。
- TXTレコードはSPF/DKIM/DMARC設定に不可欠。設定ミスはメールのスパム判定や配送失敗の原因となる。
- PTRレコードは逆引き用で、プロバイダやDNS管理者に設定依頼が必要な場合が多い。

#### 独自ドメインとサブドメイン
- **独自ドメイン**: 自分で取得・管理するドメイン（例: `example.com`）。企業やサービスのアイデンティティを表し、ウェブサイト（`example.com`）やメール（`[email protected]`）に使用。DNSでA、MX、TXTレコードを設定し、サービスを運用。
- **サブドメイン**: 独自ドメインの階層下に作成されるドメイン（例: `mail.example.com`, `www.example.com`）。サービスを分離（メールサーバー、ウェブサーバーなど）したり、特定の機能を割り当てたりする。サブドメインごとにA、MX、TXTレコードを設定可能。
- **相関関係**:
  - サブドメインは独自ドメインのDNS設定に依存。`example.com`のNSレコードで指定されたDNSサーバーが、サブドメイン（`mail.example.com`）のレコードも管理。
  - メール送信では、独自ドメイン（`example.com`）にSPF/DKIM/DMARCを設定し、サブドメイン（`mail.example.com`）にAやMXレコードを設定するケースが一般的。
  - 例: `example.com`のMXレコードで`mail.example.com`を指定し、`mail.example.com`のAレコードでサーバーIP（192.168.1.1）を定義。
- **運用例**:
  - 独自ドメイン: `example.com` → Aレコードでウェブサーバー（192.168.1.1）、MXレコードでメールサーバー（`mail.example.com`）。
  - サブドメイン: `mail.example.com` → AレコードでメールサーバーIP、TXTレコードでDKIM公開鍵。
  - サブドメインのNSレコードを別DNSサーバーに委任可能（例: `shop.example.com`をクラウドDNSで管理）が、管理が複雑になる。

**トラブルシューティングのポイント**:
- Aレコードが未設定だと、ウェブやメールサーバーに接続できない。`dig +short example.com`でIPが返るか確認。
- MXレコードが誤っていると、メールが届かない。`dig +short MX example.com`で正しいメールサーバー（例: `mail.example.com`）が設定されているか確認。
- サブドメインの設定ミスは、サービス分離の失敗や認証エラーを引き起こす。`dig +short mail.example.com`で正しいIPが解決されるか確認。
- DKIMのTXTレコード（例: `default._domainkey.example.com`）がサブドメインに設定されている場合、PostfixのDKIM署名設定（`/etc/opendkim.conf`）と一致しているか確認。

### コマンド
以下のコマンドで、DNSレコードやドメイン設定を確認できます。詳細は「送信ドメイン認証とドメインレピュテーション」セクションも参照。

#### 1. Aレコードの確認
```bash
dig +short example.com
```
**出力例**:
```
192.168.1.1
```
**説明**: `example.com`のAレコードを確認。IPアドレスが返れば正しく設定済み。

#### 2. MXレコードの確認
```bash
dig +short MX example.com
```
**出力例**:
```
10 mail.example.com.
```
**説明**: `example.com`のMXレコードを確認。メールサーバー（`mail.example.com`）と優先度（10）が返る。

#### 3. サブドメインのAレコード確認
```bash
dig +short mail.example.com
```
**出力例**:
```
192.168.1.1
```
**説明**: サブドメイン`mail.example.com`のAレコードを確認。メールサーバーのIPが正しいかチェック。

#### 4. NSレコードの確認
```bash
dig +short NS example.com
```
**出力例**:
```
ns1.example.com.
ns2.example.com.
```
**説明**: `example.com`のDNSサーバーを確認。サブドメインも同じNSサーバーで管理されているか確認。

## 送信ドメイン認証とドメインレピュテーション

### 目的
独自ドメインのメール送信における信頼性を確保し、スパム判定やブロックを防ぎます。SPF、DKIM、DMARCの設定を確認し、ドメインレピュテーションを維持します。

### 環境
- Linux (RHEL6, RHEL7, RHEL8), Postfix, DNSサーバ（BINDなど）

### 概要
- **独自ドメイン**: 企業名やサービス名を含む独自ドメイン（例：`example.com`）は、フリーメール（例：`gmail.com`）に比べ信頼性が高く、ビジネスメールに必須。
- **SPF**: 送信元IPを認証し、なりすましを防止。DNS TXTレコードで設定。
- **DKIM**: 電子署名でメール改ざんを検知。公開鍵をDNSに、秘密鍵をサーバに設定。
- **DMARC**: SPF/DKIMの結果に基づき、受信側に動作（例：拒否、隔離）を指示。DNS TXTレコードで設定。
- **ドメインレピュテーション**: 送信実績、開封率、迷惑メール報告率に基づく評価。低い場合、メールがスパム扱いされる可能性。

### コマンド

#### 1. SPFレコードの確認
```bash
dig +short TXT example.com
```
**出力例**:
```
"v=spf1 ip4:192.168.1.1 include:_spf.google.com ~all"
```
**説明**: ドメイン`example.com`のSPFレコードを確認。`ip4:192.168.1.1`がサーバIP、`~all`はソフトフェイル（失敗時に隔離）を示す。設定不備ならDNS修正。

#### 2. DKIMレコードの確認
```bash
dig +short TXT default._domainkey.example.com
```
**出力例**:
```
"v=DKIM1; k=rsa; p=MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQC..."
```
**説明**: DKIM公開鍵を確認。`default._domainkey`はセレクタ名。署名エラーがログ（`/var/log/maillog`）にあれば、鍵の一致をチェック。

#### 3. DMARCレコードの確認
```bash
dig +short TXT _dmarc.example.com
```
**出力例**:
```
"v=DMARC1; p=reject; rua=mailto:[email protected]; ruf=mailto:[email protected]; fo=1;"
```
**説明**: DMARCポリシーを確認。`p=reject`は認証失敗時にメールを拒否。ログで`dmarc=fail`が頻発する場合、SPF/DKIM設定を見直す。

#### 4. 正引き（FQDN確認）
```bash
dig +short mail.example.com
```
**出力例**:
```
192.168.1.1
```
**説明**: メールサーバのFQDN（例：`mail.example.com`）が正しいIPに解決されるか確認。解決しない場合、DNS Aレコードを修正。

#### 5. 逆引き（PTRレコード確認）
```bash
dig +short -x 192.168.1.1
```
**出力例**:
```
mail.example.com.
```
**説明**: サーバIPの逆引きが正しいFQDNに解決されるか確認。Googleの`cannot find your reverse hostname`エラーがある場合、ISPにPTRレコード設定を依頼。

#### 6. ドメインレピュテーションの確認
```bash
# ブラックリスト確認
dig +short 1.1.168.192.zen.spamhaus.org
```
**出力例**:
```
127.0.0.2
```
**説明**: IP（例：`192.168.1.1`）がSpamhaus RBLに登録されているか確認。出力があればブラックリスト登録を示し、解除申請が必要。Google Postmaster Tools（https://postmaster.google.com）でレピュテーションも確認可能。

#### 7. Postfix設定の確認（SPF/DKIM/DMARC）
```bash
postconf -n | grep -E 'smtpd_recipient_restrictions|smtpd_milters'
```
**出力例**:
```
smtpd_recipient_restrictions = permit_mynetworks, permit_sasl_authenticated, reject_unauth_destination, check_policy_service unix:private/policyd-spf
smtpd_milters = inet:localhost:8891
non_smtpd_milters = inet:localhost:8891
```
**説明**: SPF（`policyd-spf`）、DKIM（`opendkim`）の設定を確認。`smtpd_milters`にDKIMが含まれ、`check_policy_service`でSPFが有効か確認。

## LB（ロードバランサー）の確認

### 目的
ロードバランサーのステータスを確認し、正常に動作しているかを確認します。

### 環境
- F5 BIG-IP

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

#### 3. ヘルスモニターの詳細確認
```bash
tmsh show ltm monitor http-monitor
```
**出力例**:
```
Ltm::Monitor: http-monitor
--------------------------------
Status: up
Interval: 5
Timeout: 16
Send String: GET /health HTTP/1.1\r\n
Receive String: OK
```
**説明**: ヘルスモニターの設定と状態を確認。`down`の場合は、モニター条件（例：`/health`の応答）を確認。

#### 4. トラフィック統計の詳細
```bash
tmsh show ltm virtual http-vs
```
**出力例**:
```
Ltm::Virtual Server: http-vs
--------------------------------
Status: available
Connections: 100
Bits In/Out: 1.2M/1.5M
```
**説明**: バーチャルサーバのトラフィック量を確認。不均衡な分配があれば、プool設定を調査。

## Cisco機器の確認

### 目的
Cisco機器のステータスを確認し、異常がないかを確認します。RADIUS認証のトラブルシューティングを含め、認証失敗や遅延の原因を特定します。

### 環境
- Cisco (IOS/IOS-XE), RADIUSサーバ（例：FreeRADIUS on Linux）

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

#### 7. インターフェースエラーの確認
```bash
show interfaces | include errors
```
**出力例**:
```
0 input errors, 0 CRC, 0 frame, 0 overrun, 0 ignored
0 output errors, 0 collisions, 0 interface resets
```
**説明**: エラー（例：CRCやドロップ）が多ければ、物理層の問題（ケーブル、ポート）を疑う。

#### 8. トラフィック統計
```bash
show interfaces | include packets
```
**出力例**:
```
Input packets: 123456, Output packets: 654321
```
**説明**: インターフェースのトラフィック量を確認。異常な偏りがあれば、ルーティングや負荷分散の問題を調査。

#### 9. RADIUSサーバの接続状態を確認
```bash
show aaa servers
```
**出力例**:
```
RADIUS: id 1, priority 1, host 192.168.1.100, auth-port 1812, acct-port 1813
        State: current UP, duration 3600s, previous duration 0s
        Dead: total time 0s, count 0
        Auth: request 100, timeouts 0, response time 50ms
        Acct: request 50, timeouts 0, response time 45ms
```
**説明**: RADIUSサーバ（例：192.168.1.100）の状態を確認。`State: UP`なら接続正常。`DOWN`や`timeouts`が多い場合は、ネットワーク接続やサーバの稼働状態を調査。

#### 10. RADIUS認証ログの確認
```bash
show logging | include AAA
```
**出力例**:
```
Sep 03 23:15:01: %AAA-3-AUTHFAIL: Authentication failure for user 'admin' from 192.168.1.50
```
**説明**: AAA（認証）関連のログを抽出し、認証失敗（例：ユーザー`admin`の失敗）を確認。ユーザー名/パスワード誤り、RADIUSサーバ設定不備、または接続問題を疑う。

#### 11. RADIUS設定の確認
```bash
show running-config | section aaa
```
**出力例**:
```
aaa new-model
aaa authentication login default group radius local
aaa authorization exec default group radius local
radius-server host 192.168.1.100 auth-port 1812 acct-port 1813 key 7 1234567890
```
**説明**: AAAおよびRADIUSサーバ設定を確認。`radius-server`のIP、ポート、共有鍵が正しいかチェック。鍵不一致は認証失敗の一般的な原因。

#### 12. RADIUSサーバへのテスト認証
```bash
test aaa group radius admin cisco123 new-code
```
**出力例**:
```
User successfully authenticated
```
**説明**: ユーザー`admin`とパスワード`cisco123`でRADIUSサーバにテスト認証を実施。失敗する場合、RADIUSサーバログ（`/var/log/radius/radius.log`）や共有鍵を調査。

#### 13. RADIUSパケットのデバッグ
```bash
debug aaa authentication
debug radius
```
**出力例**:
```
Sep 03 23:15:01: RADIUS: Authenticating user 'admin' to server 192.168.1.100
Sep 03 23:15:01: RADIUS: Received Access-Reject packet from 192.168.1.100
```
**説明**: RADIUS認証の詳細をデバッグ。`Access-Reject`があれば、サーバ側でユーザー認証が拒否されたことを示す。デバッグ後は`undebug all`で停止。

#### 14. Linux側RADIUSサーバログの確認（FreeRADIUSの場合）
```bash
cat /var/log/radius/radius.log | grep "Access-Reject"
```
**出力例**:
```
Sep 03 23:15:01 radiusd[1234]: Login incorrect: [admin/<via Auth-Type>] (from client cisco-router port 0 cli 192.168.1.50)
```
**説明**: RADIUSサーバのログを確認し、認証失敗の理由（例：パスワード誤り）を特定。必要に応じてユーザーDBや設定（`/etc/raddb/users`）を修正。

##

# データベース調査

## データベース接続の準備
MySQLでSQLを実行する前に、データベースに接続し、SQLを打てる状態にする必要があります。以下は一般的な接続手順です。

**コマンド:**
```bash
mysql -u [ユーザー名] -p
```
- 実行後、パスワードを入力します（例: `root`ユーザーなら`mysql -u root -p`）。
- 接続後、対象のデータベースを選択します：
```sql
USE app_db;
```

**説明:**
- `mysql`コマンドでMySQLサーバーにログインします。`-u`でユーザー名、`-p`でパスワード入力を指定。
- `USE`コマンドで、調査対象のデータベース（例: `app_db`）を選択し、SQLを実行可能な状態にします。

**確認する観点:**
- 正しいユーザーとパスワードで接続できているか確認。
- 対象データベースが選択され、クエリを実行する準備が整っているか確認。

**出力例:**
```bash
$ mysql -u root -p
Enter password: ******
Welcome to the MySQL monitor.  Commands end with ; or \g.
mysql> USE app_db;
Database changed
mysql>
```

## 1. アクティブな接続状況の確認
**なぜこのクエリが必要か:**  
データベースの接続数はリソース使用量やパフォーマンスに直結します。過剰な接続や長時間実行中のクエリは、システムの遅延やリソース枯渇を引き起こす可能性があります。このクエリでアクティブな接続を把握し、異常を素早く特定できます。

**コマンド:**
```sql
SHOW FULL PROCESSLIST;
```

**説明:**
現在実行中のすべての接続とクエリを詳細に表示します。ユーザー、ホスト、データベース、クエリ内容、実行時間、状態などが確認できます。

**確認する観点:**
- 実行中のクエリや接続数を把握し、負荷の高いユーザーやデータベースを特定。
- 異常な接続や長時間実行中のクエリを検出。

**出力例:**
```
+----+------+-----------+--------+---------+------+-------+-----------------------------+
| Id | User | Host      | db     | Command | Time | State | Info                        |
+----+------+-----------+--------+---------+------+-------+-----------------------------+
| 1  | app  | localhost | app_db | Query   |   10 | Sending data | SELECT * FROM users WHERE id = 123 |
| 2  | app  | localhost | app_db | Sleep   |  120 |       | NULL                        |
| 3  | root | localhost | NULL   | Query   |    0 | init  | SHOW FULL PROCESSLIST       |
+----+------+-----------+--------+---------+------+-------+-----------------------------+
```

## 2. スロークエリの確認
**なぜこのクエリが必要か:**  
スロークエリはデータベースのパフォーマンス低下の主な原因です。実行に時間がかかるクエリを特定することで、インデックス不足や非効率なクエリを改善し、アプリケーションの応答性を向上させられます。

**コマンド:**
```sql
SELECT * FROM mysql.slow_log WHERE start_time > NOW() - INTERVAL 1 DAY ORDER BY query_time DESC LIMIT 10;
```

**説明:**
おります。クエリの実行時間、ロック時間、処理した行数、クエリ内容を確認できます。

**確認する観点:**
- パフォーマンスボトルネックを引き起こすクエリを特定。
- インデックス追加やクエリ最適化の対象を洗い出し。

**出力例:**
```
+---------------------+------------+-----------+------------+---------------+---------------------------------+
| start_time          | query_time | lock_time | rows_sent | rows_examined | sql_text                        |
+---------------------+------------+-----------+------------+---------------+---------------------------------+
| 2025-07-08 10:00:00 | 00:00:05.2 | 00:00:00.1 |       100 |         10000 | SELECT * FROM orders WHERE date < '2025-01-01' |
| 2025-07-08 09:55:00 | 00:00:04.8 | 00:00:00.0 |        50 |          5000 | SELECT * FROM products         |
+---------------------+------------+-----------+------------+---------------+---------------------------------+
```

## 3. テーブル情報の確認
**なぜこのクエリが必要か:**  
テーブルのサイズや構造の変化は、ディスク使用量やクエリ性能に影響します。大きなテーブルや断片化したテーブルを特定することで、ストレージ管理やメンテナンスの必要性を判断できます。

**コマンド:**
```sql
SELECT * FROM information_schema.tables WHERE table_schema NOT IN ('mysql', 'information_schema', 'performance_schema', 'sys');
```

**説明:**
システムスキーマを除く全テーブルのメタデータを取得します。テーブル名、ストレージエンジン、データサイズ、インデックスサイズなどが含まれます。

**確認する観点:**
- テーブルのサイズや成長傾向を監視。
- 最適化やパーティショニングの必要性を評価。

**出力例:**
```
+----------------+------------+------------+-----------+-----------------+------------------+
| TABLE_SCHEMA   | TABLE_NAME | ENGINE     | ROW_COUNT | DATA_LENGTH     | INDEX_LENGTH     |
+----------------+------------+------------+-----------+-----------------+------------------+
| app_db         | users      | InnoDB     |      1000 |        16384    |         32768    |
| app_db         | orders     | InnoDB     |     50000 |      5242880    |       1048576    |
+----------------+------------+------------+-----------+-----------------+------------------+
```

## 4. ロック状況の確認
**なぜこのクエリが必要か:**  
ロック待機はクエリ遅延やデッドロックの原因となり、アプリケーションの停止やエラーを引き起こします。ロック状況を把握することで、問題のトランザクションを特定し、迅速な対応が可能です。

**コマンド:**
```sql
SELECT * FROM information_schema.innodb_trx WHERE trx_state = 'LOCK WAIT';
```

**説明:**
InnoDBのトランザクションでロック待機中のものを取得します。トランザクションID、実行中のクエリ、待機時間などが表示されます。

**確認する観点:**
- ロック競合によるクエリ遅延の原因を特定。
- 問題のあるトランザクションを終了させる判断材料に。

**出力例:**
```
+-----------------+-------------------+-----------------------------+-----------------+
| trx_id          | trx_state         | trx_query                   | trx_started     |
+-----------------+-------------------+-----------------------------+-----------------+
| 12345           | LOCK WAIT         | UPDATE users SET name = 'test' WHERE id = 1 | 2025-07-09 10:00:00 |
+-----------------+-------------------+-----------------------------+-----------------+
```

## 5. インデックス使用状況の確認
**なぜこのクエリが必要か:**  
インデックスはクエリ性能を向上させますが、不要なインデックスはディスクを無駄にし、書き込み性能を低下させます。使用状況を確認することで、インデックスの最適化や削除を判断できます。

**コマンド:**
```sql
SELECT * FROM information_schema.statistics WHERE table_schema NOT IN ('mysql', 'information_schema', 'performance_schema', 'sys') ORDER BY seq_in_index;
```

**説明:**
システムスキーマを除くテーブルのインデックス情報を取得します。インデックス名、カラムの順序、インデックスの種類などが確認できます。

**確認する観点:**
- 使用されていないインデックスを特定し、削除を検討。
- クエリパフォーマンス向上のためのインデックス最適化を評価。

**出力例:**
```
+----------------+------------+-------------+-----------+------------+
| TABLE_SCHEMA   | TABLE_NAME | INDEX_NAME  | COLUMN_NAME | SEQ_IN_INDEX |
+----------------+------------+-------------+-----------+------------+
| app_db         | users      | idx_user_id | user_id   |          1 |
| app_db         | orders     | idx_date    | order_date |         1 |
+----------------+------------+-------------+-----------+------------+
```

## 前提条件
- スロークエリログの有効化（`slow_query_log = ON`）が必要。
- `information_schema`および`mysql`スキーマへのアクセス権限。
- InnoDBストレージエンジンを使用（ロック関連クエリ用）。
