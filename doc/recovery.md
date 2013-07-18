復旧手順
==========================

　
・シナリオ

DB-NODE1が故障してDB-NODE2にフェイルオーバーした

あとNODE1を再度スレーブとしてクラスタに参加させる。
　

1) mysql起動
<pre>
[DB-NODE1]# service mysql start
</pre>

2) 現マスターから取得したフルバックアップをリストア

<pre>
[DB-NODE1]# mysql -p < fullbackup.dump
</pre>

※LVMスナップショットの場合は 1)mysql起動前にデータディレクトリごと差し替えて起動すればOK


3) 初期化＆レプリケーション再設定

<pre>
[DB-NODE1]# mysql
mysql> RESET MASTER;
mysql> RESET SLAVE; ※5.5の場合は RESET SLAVE ALL;
mysql> CHANGE MASTER TO MASTER_LOG_FILE='BINLOGポジション', MASTER_HOST='DB-NODE2', MASTER_USER='レプリケーションユーザ', MASTER_PASSWORD='パスワード';
</pre>

4) mysql停止

<pre>
[DB-NODE1]# service mysql stop
</pre>


5) DB-NODE2のスレーブ時代の情報クリア

<pre>
[DB-NODE2]# mysql
mysql> RESET SLAVE; ※5.5の場合は RESET SLAVE ALL;
</pre>

RESET SLAVEを打つ理由はマスター昇格したサーバがスレーブへ降格する際に古いレプリケーションの情報を持ったままだと

意図しないポジションからスタートする可能性がある為です。

6) リソースクリーンアップ

<pre>
[DB-NODE2]# crm resource cleanup ms_mysql
</pre>

クリーンアップと同時にmysqlがpacemaker経由で起動されます。

この時5)を飛ばすとbinlogの指定が最初からになってしまい意図したポジションからレプリケーションが開始されません。

かならず手動でレプリケーションの再設定を施してからpacemaker経由で起動します。


7) 準同期レプリケーション再設定

<pre>
[DB-NODE1]# mysql
mysql> SET GLOBAL rpl_semi_sync_slave_enabled=1;STOP SLAVE; START SLAVE;
</pre>

<pre>
[DB-NODE2]# mysql
mysql> SET GLOBAL rpl_semi_sync_master_enabled=1;
</pre>


8) 準同期レプリケーションの確認

<pre>
[DB-NODE1]# less /var/log/mysq.err
</pre>

マスター側でこんなのがログに出てればOK

 > 130709 16:36:54 [Note] Start semi-sync binlog_dump to slave (server_id: 1), pos(mysqld-bi.....)


スレーブ側でこんなのがログに出てればOK

 > 130709 16:36:54 [Note] Slave I/O thread: Start semi-sync replication to master 'XXXXX@XXXXXX:3306' in


9) スレーブウォーミングアップ

<pre>
[DB-NODE1]# mysql
mysql> SELECT COUNT(*) FROM レコード数の多いテーブル
</pre>

10) スレーブVIP移動

<pre>
[DB-NODE2]# crm resource move p_slave_vip DB-NODE1 force
[DB-NODE2]# crm resource unmove p_slave_vip
</pre>


元の筐体へのフェイルバック
==================== 

・シナリオ

NODE1を再度スレーブとしてクラスタに参加させたあと

マスター、スレーブを逆転させて初期の状況に戻す。


11) 現在の状態の逆の情報をリセット

　現在、マスターであればスレーブの情報をクリア。

　現在、スレーブであればマスターの情報をクリア



<pre>
[DB-NODE1]# mysql
mysql> RESET MASTER;
</pre>

DB-NODE1はスレーブなのでマスターの情報をクリア


<pre>
[DB-NODE2]# mysql
mysql> RESET SLAVE ALL;
</pre>

DB-NODE2はマスターなのでスレーブの情報をクリア

12) マスタとスレーブ入れ替え

<pre>
[DB-NODE1]# crm resource move p_master_vip DB-NODE1 force
[DB-NODE1]# crm resource unmove p_master_vip
</pre>

13) 入れ替えした一つ前の状況の情報クリア

　現在、マスターであればスレーブの情報をクリア。

　現在、スレーブであればマスターの情報をクリア

※再度フェイルオーバした際に情報が残っていると意図しないポジションからレプリケーションが再開されてしまうのを防止する為です。


<pre>
[DB-NODE1]# mysql
mysql> RESET SLAVE ALL;
</pre>

DB-NODE1はスレーブ -> マスターへと入れ替わったのでスレーブの情報をクリア


<pre>
[DB-NODE2]# mysql
mysql> RESET MASTER;
</pre>

DB-NODE2はマスター -> スレーブへと入れ替わったのでマスターの情報をクリア


14) 準同期レプリケーション再設定

<pre>
[DB-NODE2]# mysql
mysql> SET GLOBAL rpl_semi_sync_slave_enabled=1;STOP SLAVE; START SLAVE;
</pre>

<pre>
[DB-NODE1]# mysql
mysql> SET GLOBAL rpl_semi_sync_master_enabled=1;
</pre>

