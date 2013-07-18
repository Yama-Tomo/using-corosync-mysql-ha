
環境
======================

OS

* Ubuntu 12.04.1 LTS

MySQL

*  5.5.28

corosync

*  1.4.2-2

pacemaker

*  1.1.6-2

M/Wに関しては手抜きでパッケージインストールです。

MySQLはSemi-sync replicationが前提なので5.5を入れました。


設定
======================

マスターDB

* hostname: DB-NODE1
* ip: 192.168.162.10
* master vip: 192.168.162.100

スレーブDB

* hostname: DB-NODE2
* ip: 192.168.162.20
* slave vip: 192.168.162.200

MySQL

レプリケーションユーザと同じ名前でlocalhostに対しての権限を追加します。

<pre>
mysql> grant replication client, replication slave, SUPER, PROCESS, RELOAD on *.* to ユーザ名@'localhost' identified by 'パスワード';
mysql> FLUSH PRIVILEGES;
</pre>


pacemaker

<pre>
# crm configure load update /etc/corosync/crm.config
</pre>

ocfスクリプト修正

MySQL5.5以上のバージョンに対応させるためにスクリプトを差し替えます。

<pre>
# cp mysql /usr/lib/ocf/resource.d/heartbeat/mysql
</pre>

