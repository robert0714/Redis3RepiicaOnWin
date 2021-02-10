https://redis.io/topics/cluster-tutorial
## 關於 Redis Cluster
* 僅適合於 Redis3.0（包括3.0）以上版本  
* 每個 redis cluster 有 16384 個 hash slot，key 經過 CRC16 計算後決定儲存的 cluster node  
* 每個 redis node 都需要開放兩個 TCP port：一個是 redis client 連線使用的 port (常見為 6379) 另一個是 cluster bus 用來做 node 間的交互溝通 (固定為 client port 加上 10000)
* 僅支援單一 database  
* 不支援多個 key 的操作 (可能需要跨 node)  

* Redis 的 config ，預設只開放 Localhost，所以接下來，我們要修改一下設定。  
要去 Redis 的安裝目錄下，修改 redis.windows-service.conf 這個檔案， 並且在原本的 bind 127.0.0.1 後面加上這台的私有 ip。  
在預設的情況下，為了安全，Redis 只會監聽 127.0.0.1 這個 IP 位置，所以為了讓他能支援內網， 所以我們在 127.0.0.1 後面加上這台 ( Cache-1 ) 的私有 IP。

* 如要加上密碼， 所以將 requirepass foobared 給註解掉， 其中，foobared 就是密碼。
* Redis 目前有一個參數為　protected-mode yes ，這是一個 Redis 預設的保護參數， 如果把 bind 給註解掉 ( 表示開放 )，且又沒設定 Password，那外部怎樣連都會連不通喔。所以，要不就是把 protected-mode 設定 no ，要不就是乖乖地給個密碼吧。 ( 把 Protected-mode yes 給註解掉是沒用的喔～ )

## Source

```bash
https://github.com/microsoftarchive/redis/releases
wget https://github.com/microsoftarchive/redis/releases/download/win-3.0.504/Redis-x64-3.0.504.zip
```
## Configuration
basic
```conf
port 7000 # 每個 node 的 client port
cluster-enabled yes # 啟用 redis cluster
cluster-config-file nodes_7000.conf # 每個 node 需要獨立，cluster 自行維護使用，不需人為介入
cluster-node-timeout 5000 # node 判斷失效的時間
appendonly yes # 啟用 aof
```
advance
```conf
port 7000 # 每個 node 的 client port
cluster-enabled yes # 啟用 redis cluster
cluster-config-file nodes_7000.conf # 每個 node 需要獨立，cluster 自行維護使用，不需人為介入
cluster-node-timeout 5000 # node 判斷失效的時間
appendonly yes # 啟用 aof
daemonize yes # 背景執行
bind x.x.x.x # 允許 listen 特定 ip 的連線
requirepass password # 密碼設定
masterauth password # 從 master 的密碼
```

## Startup
Install redis instances for windows service 
```bash
cd Redis-x64-3.0.504
redis-server --service-install  --service-name redisService1 --port 7000 --cluster-config-file nodes-7000.conf  
redis-server --service-install  --service-name redisService2 --port 7001 --cluster-config-file nodes-7001.conf  --slaveof 127.0.0.1 7000

```

start redis instances for windows service
```bash
cd Redis-x64-3.0.504
redis-server  --service-start  --service-name  redisService1  
redis-server  --service-start  --service-name  redisService2  

```

To confirm the status when they were un connected.( 當redis.windows-service.conf有設定requirepass foobared 還需要額外參數 -a foobared )
```
redis-cli -h 127.0.0.1 -p 7000 info replication
# Replication
role:master
connected_slaves:0
master_repl_offset:0
repl_backlog_active:0
repl_backlog_size:1048576
repl_backlog_first_byte_offset:0
repl_backlog_histlen:0
```

To confirm the status when they were connected.( 當redis.windows-service.conf有設定requirepass foobared 還需要額外參數 -a foobared )
```
redis-cli -h 127.0.0.1 -p 7000  info replication
# Replication
role:master
connected_slaves:1
slave0:ip=::c0f3:fa00:0:0,port=7001,state=online,offset=239,lag=1
master_repl_offset:239
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:2
repl_backlog_histlen:238

redis-cli -h 127.0.0.1 -p 7001  info replication
# Replication
role:slave
master_host:localhost
master_port:7000
master_link_status:up
master_last_io_seconds_ago:1
master_sync_in_progress:0
slave_repl_offset:57
slave_priority:100
slave_read_only:1
connected_slaves:0
master_repl_offset:0
repl_backlog_active:0
repl_backlog_size:1048576
repl_backlog_first_byte_offset:0
repl_backlog_histlen:0
```
Test case:
接下來，我們從 Master 塞一個 HelloWorld ，然後連到 Slave，取得資料看看。 我們全部都用 Slave 這台來操作，結果可以參考如下圖；所以這樣設定完，兩台 Redis 就會擁有複寫的功能了， 而且我們可以透過 Master 來讀寫，而 Slave 來進行讀的動作了，不過這邊要特別注意， Master 提供了讀寫，但 Slave 是僅供讀取而已，是不能寫入的...
```
redis-cli -h 127.0.0.1 -p 7000
127.0.0.1:7000> set Test HelloRobert
OK
127.0.0.1:7000> get Test
"HelloRobert"
127.0.0.1:7000> exit

redis-cli -h 127.0.0.1 -p 7001
127.0.0.1:7001> get Test
"HelloRobert"
127.0.0.1:7001>
```

所以，到這邊，如果 Master 掛掉了，會自動把 Slave 切換成 Master 嗎!? 當然不...所以下一個章節，就是第三台機器登場。

# 設定　Sentinel
完成了 Master 和 Slave 的複寫，接下來，就是要建構 Sentinel，Sentinel 的其中兩個功能， 就是監控( monitor )和切換 (failover)，他會監控 Master 是否死掉，死了之後，就會把 Slave 變成 Master ( 這樣就可以持續寫入 )， 而原本的 Master 復活後，就會被改成 Slave。

啟動 Sentinel ，需要不同的參數，所以我們就不使用 msi 了， 然果真的覺得 Windows 服務比較威，那可以參考這邊，註冊進 Windows 服務(https://hadb.me/redis-windows-sentinel/)。

增加一個 sentinel.conf 檔案，到時候 Sentinel 的設定就會寫在裡面，如下。
```
port 26379
bind 127.0.0.1 10.0.0.6
sentinel monitor master1 10.0.0.4 7000 1
sentinel down-after-milliseconds master1 5000
sentinel failover-timeout master1 900000
sentinel parallel-syncs master1 2
sentinel auth-pass master1 foobared
```
* port : 代表 Sentinel Port 的位置，通常大家都會用 26379
* bind : 這很重要，至少要加上 127.0.0.1，不然會因為通訊的原因，而造成不斷的去嘗試切換 Master 和 Slave，如果有三台 Sentinel ，後面也要加上這台的 IP，真的不想加這行，也可以把安全模式關閉，改成　protected-mode no
* sentinel monitor : 要監控的 Master ip 和 port，其中 master1 是自訂名稱。
* down-after-milliseconds : 如果在幾毫秒內， 没有回應 sentinel ， 那麼 Sentinel 就會將這台標記為下線 SDOWN （ subjectively down )
* failover-timeout : 轉移過程多久沒完成，算失敗。
* parallel-syncs 在轉移過程，有多少台會和 Master 同步資料， 數字越小，完成轉移的時間越長

上面完成後，我們還要回到 Master 的機器，設定 redis.windows-service.conf， 找到 # masterauth，改成 masterauth foobared ( 代表 Master 的密碼 )。 需要這樣的原因，是因為，到時候 Master 會被切換成 Slave，如果沒有設定 masterauth foobared， 到時候這台機器會無法和未來的 Master 進行連線。 ( 設定完也不忘記重新啟動 ) <安裝windows service增加此參數會造成slave無法啟動，若slave無此參數，則會發生failover失敗>

完成後，我們就可以啟動 Sentinel
```
redis-server.exe sentinel.conf --sentinel

[18088] 10 Feb 12:39:50.916 # Creating Server TCP listening socket localhost:26379: bind: No such file or directory

D:\webatm\Redis-x64-3.0.504>redis-server.exe sentinel.conf --sentinel
                _._
           _.-``__ ''-._
      _.-``    `.  `_.  ''-._           Redis 3.0.504 (00000000/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._
 (    '      ,       .-`  | `,    )     Running in sentinel mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 26379
 |    `-._   `._    /     _.-'    |     PID: 15760
  `-._    `-._  `-./  _.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |           http://redis.io
  `-._    `-._`-.__.-'_.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |
  `-._    `-._`-.__.-'_.-'    _.-'
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

[15760] 10 Feb 12:40:40.447 # Sentinel runid is ddcc74b24b55eb2b10531cc372cad9879e77c1d7
[15760] 10 Feb 12:40:40.451 # +monitor master master1 127.0.0.1 7000 quorum 1
[15760] 10 Feb 12:40:41.460 * +slave slave [::d486:1540:100:0]:7001 ::d486:1540:100:0 7001 @ master1 127.0.0.1 7000
[15760] 10 Feb 12:40:46.489 # +sdown slave [::d486:1540:100:0]:7001 ::d486:1540:100:0 7001 @ master1 127.0.0.1 7000
```
# Test Failover
閉 Master，然後可以從 Sentinel 看到此變化。 馬上就可以看到，他已經把 Salve 切換到 Master 了。  
slave無此參數(masterauth foobared)以及密碼設定(requirepass foobared)，slaveof 參數 只能用ip 則會發生failover失敗
```
                _._
           _.-``__ ''-._
      _.-``    `.  `_.  ''-._           Redis 3.0.504 (00000000/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._
 (    '      ,       .-`  | `,    )     Running in sentinel mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 26379
 |    `-._   `._    /     _.-'    |     PID: 15760
  `-._    `-._  `-./  _.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |           http://redis.io
  `-._    `-._`-.__.-'_.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |
  `-._    `-._`-.__.-'_.-'    _.-'
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

[15760] 10 Feb 12:40:40.447 # Sentinel runid is ddcc74b24b55eb2b10531cc372cad9879e77c1d7
[15760] 10 Feb 12:40:40.451 # +monitor master master1 127.0.0.1 7000 quorum 1
[15760] 10 Feb 12:40:41.460 * +slave slave [::d486:1540:100:0]:7001 ::d486:1540:100:0 7001 @ master1 127.0.0.1 7000
[15760] 10 Feb 12:40:46.489 # +sdown slave [::d486:1540:100:0]:7001 ::d486:1540:100:0 7001 @ master1 127.0.0.1 7000
[15760] 10 Feb 12:47:28.119 # +sdown master master1 127.0.0.1 7000
[15760] 10 Feb 12:47:28.119 # +odown master master1 127.0.0.1 7000 #quorum 1/1
[15760] 10 Feb 12:47:28.121 # +new-epoch 1
[15760] 10 Feb 12:47:28.122 # +try-failover master master1 127.0.0.1 7000
[15760] 10 Feb 12:47:28.126 # +vote-for-leader ddcc74b24b55eb2b10531cc372cad9879e77c1d7 1
[15760] 10 Feb 12:47:28.126 # +elected-leader master master1 127.0.0.1 7000
[15760] 10 Feb 12:47:28.127 # +failover-state-select-slave master master1 127.0.0.1 7000
[15760] 10 Feb 12:47:28.226 # -failover-abort-no-good-slave master master1 127.0.0.1 7000
[15760] 10 Feb 12:47:28.302 # Next failover delay: I will not start a failover before Wed Feb 10 13:17:28 2021
```
正常failover如下
```
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |
  `-._    `-._`-.__.-'_.-'    _.-'
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

[19660] 10 Feb 13:16:08.967 # Sentinel runid is ef41d784ec1ef1cc07f9e5aebfde8ddb370f93a0
[19660] 10 Feb 13:16:08.979 # +monitor master master1 127.0.0.1 7000 quorum 1
[19660] 10 Feb 13:16:09.977 * +slave slave 127.0.0.1:7001 127.0.0.1 7001 @ master1 127.0.0.1 7000
[19660] 10 Feb 13:16:14.028 # +sdown slave [::d486:1540:100:0]:7001 ::d486:1540:100:0 7001 @ master1 127.0.0.1 7000
[19660] 10 Feb 13:16:23.479 # +sdown master master1 127.0.0.1 7000
[19660] 10 Feb 13:16:23.479 # +odown master master1 127.0.0.1 7000 #quorum 1/1
[19660] 10 Feb 13:16:23.488 # +new-epoch 4
[19660] 10 Feb 13:16:23.496 # +try-failover master master1 127.0.0.1 7000
[19660] 10 Feb 13:16:23.499 # +vote-for-leader ef41d784ec1ef1cc07f9e5aebfde8ddb370f93a0 4
[19660] 10 Feb 13:16:23.508 # +elected-leader master master1 127.0.0.1 7000
[19660] 10 Feb 13:16:23.509 # +failover-state-select-slave master master1 127.0.0.1 7000
[19660] 10 Feb 13:16:23.588 # +selected-slave slave 127.0.0.1:7001 127.0.0.1 7001 @ master1 127.0.0.1 7000
[19660] 10 Feb 13:16:23.588 * +failover-state-send-slaveof-noone slave 127.0.0.1:7001 127.0.0.1 7001 @ master1 127.0.0.1 7000
[19660] 10 Feb 13:16:23.696 * +failover-state-wait-promotion slave 127.0.0.1:7001 127.0.0.1 7001 @ master1 127.0.0.1 7000
[19660] 10 Feb 13:16:24.583 # +promoted-slave slave 127.0.0.1:7001 127.0.0.1 7001 @ master1 127.0.0.1 7000
[19660] 10 Feb 13:16:24.583 # +failover-state-reconf-slaves master master1 127.0.0.1 7000
[19660] 10 Feb 13:16:24.659 * +slave-reconf-sent slave [::d486:1540:100:0]:7001 ::d486:1540:100:0 7001 @ master1 127.0.0.1 7000
[19660] 10 Feb 13:16:24.659 # +failover-end master master1 127.0.0.1 7000
[19660] 10 Feb 13:16:24.667 # +switch-master master1 127.0.0.1 7000 127.0.0.1 7001
[19660] 10 Feb 13:16:24.677 * +slave slave [::d486:1540:100:0]:7001 ::d486:1540:100:0 7001 @ master1 127.0.0.1 7001
[19660] 10 Feb 13:16:24.678 * +slave slave 127.0.0.1:7000 127.0.0.1 7000 @ master1 127.0.0.1 7001
[19660] 10 Feb 13:16:29.740 # +sdown slave [::d486:1540:100:0]:7001 ::d486:1540:100:0 7001 @ master1 127.0.0.1 7001
[19660] 10 Feb 13:16:29.740 # +sdown slave 127.0.0.1:7000 127.0.0.1 7000 @ master1 127.0.0.1 7001
```
使用Cli驗證
```
redis-cli -h 127.0.0.1 -p 7001  -a foobared info replication
# Replication
role:master
connected_slaves:0
master_repl_offset:0
repl_backlog_active:0
repl_backlog_size:1048576
repl_backlog_first_byte_offset:0
repl_backlog_histlen:0
```

測試資料交換
```
>  redis-cli -h 127.0.0.1 -p 7001
127.0.0.1:7001> set Test HelloRobert
OK
127.0.0.1:7001> get Test
"HelloRobert"
127.0.0.1:7001> exit

>  redis-cli -h 127.0.0.1 -p 7000
Could not connect to Redis at 127.0.0.1:7000: 無法連線，因為目標電腦拒絕連線。
not connected> exit

>  redis-cli -h 127.0.0.1 -p 7000
127.0.0.1:7000> get Test
(nil)
127.0.0.1:7000> get Test
(nil)
127.0.0.1:7000> get Test
(nil)
127.0.0.1:7000> get Test
"HelloRobert"
127.0.0.1:7000> get Test
"HelloRobert"
127.0.0.1:7000> info replication
# Replication
role:slave
master_host:127.0.0.1
master_port:7001
master_link_status:up
master_last_io_seconds_ago:1
master_sync_in_progress:0
slave_repl_offset:2442
slave_priority:100
slave_read_only:1
connected_slaves:0
master_repl_offset:0
repl_backlog_active:0
repl_backlog_size:1048576
repl_backlog_first_byte_offset:0
repl_backlog_histlen:0
127.0.0.1:7000>
```

## Appendix

1、Install Sentinel Service
```
redis-server --service-install sentinel.conf --service-name redis-sentinel-service --sentinel
```
2、Start Sentinel Service
```
redis-server --service-start --service-name redis-sentinel-service
```
3、Start  Sentinel Service
```
redis-server --service-stop --service-name redis-sentinel-service
```

4、Uninstall Sentinel Service
```
redis-server --service-stop --service-name redis-sentinel-service
redis-server --service-uninstall --service-name redis-sentinel-service
```