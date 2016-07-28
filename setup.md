【OpenStack共同検証ラボ】
# OpenStack-Ansibleの監視・ログ分析基盤の作り方

## 変更履歴

|バージョン|更新日|更新内容|
|:---|:---|:---|
|0.8.0|2016/07/28|ベータ版公開|

````
筆者注:この本資料に対する提案や誤りの指摘は
Issue登録をお願いいたします
````

## はじめに

本資料はOpenStack共同検証ラボにて[OpenStack-Ansibleで作るOpenStack HA環境](http://enterprisecloud.jp/installguide-openstack-ansible/)を使用して構築した環境で検証を行っています。<br>
そのため、他ディストリビューション・他バージョンのOpenStack環境では監視項目・ログ分析内容について再設定が必要となる場合があります。

使用したOSのバージョンやOpenStack-Ansibleの設定などは上記手順書を参考してください。

### セットアップの流れ

本環境は以下の流れでセットアップを行います。

1. Zabbixセットアップ
2. Elasticsearch/kibanaセットアップ
3. fluentdセットアップ
4. Hatoholセットアップ

### ホスト命名規則について

本書の検証に使用した環境は下表の命名規則に従って物理サーバ・仮想サーバ・LXCコンテナのホスト名を設定しております。

|サーバ種別・用途|ホスト名　命名規則|例|
|---|---|---|
|OpenStack Controller用LXCホストサーバ|comp+数字3桁|cont001|
|各OpenStack Controller LXCコンテナ|cont+数字3桁_サービス名-ランダムなid|cont001_nova_console_container-e4b59b5c|
|OpenStack Compute|comp+数字3桁|comp001|
|KVM・CindervVolumeサーバ|zabi+数字3桁|zabi001|
|Zabbixサーバ|zabk+数字3字|zabk001|	
|CinderScheduler・CinderAPIコンテナ|zabk+数字3字_サービス名-ランダムなid|zabi001_cinder_scheduler_container-2fb6ea39|
|Elasticsearch・Kibanaサーバ|elas+数字3字|elas001|

### ネットワーク構成について

本環境のネットワークは下記の図のような構成となっています。
![ネットワーク構成図](images/network.png)

### 本書中の表記について

以下、&#35;マークは rootユーザ、&#36;マークは一般ユーザでコマンドを実行するものとします。<br>
また&#35;マーク及び&#36;マークの前に操作対象のサーバを記載しております。

## 1. Zabbixセットアップ

### 1.1 KVMのセットアップ

Zabbixサーバは、後述するHatoholと同一サーバにセットアップします。<br>
HatoholはUbuntuがサポート対象となっていないため、KVMでCentOS7.1のVMを作成します。<br>
KVMサーバで実施してください。

1.KVMパッケージインストール

KVM及び関連パッケージのインストールを行います。
 
```
kvm$ sudo apt-get install kvm libvirt-bin bridge-utils virtinst qemu-kvm
```

2.libvirt自動起動設定

libvirt-binの自動的起動設定を行います。

```
kvm$ sudo update-rc.d libvirt-bin defaults
```

### 1.2 VM作成

1.CentOSイメージダウンロード

```
kvm$ wget http://ftp.tsukuba.wide.ad.jp/Linux/centos/7/isos/x86_64/CentOS-7-x86_64-Minimal-1511.iso -P /tmp
```

2.VM作成

Zabbix Server用に8vCPU、16GBメモリ、80GB Disk、NIC2つ(br-mgmt,br-om)の仮想マシンを作成します。

```
kvm$ sudo virt-install --name centos7 --ram 16384 --vcpu 8  --file=/var/lib/libvirt/images/centos7.img  --file-size=80  --location=/tmp/CentOS-7-x86_64-Minimal-1511.iso  --os-variant=rhel7  --network bridge=br-mgmt  --network bridge=br-om  --nographics --extra-args='console=tty0 console=ttyS0,115200n8 keymap=ja' --accelerate  --autostart
```

コマンド実行後CLIセットアップが開始されます。<br>
しばらくすると下記のメニューが表示されます。<br>
設定を行う項目の数字を入力しエンターキーを押すことで、選択した項目の設定が行えます。<br>
以下の例は言語の設定(1)を入力しております。<br>
全設定が完了した後、bを入力することでOSのインストールが始まります。

```
Installation

 1) [x] Language settings                 2) [!] Timezone settings
        (English (United States))                (Timezone is not set.)
 3) [!] Installation source               4) [!] Software selection
        (Local media)                            (Minimal Install)
 5) [!] Installation Destination          6) [x] Kdump
        (No disks selected)                      (Kdump is enabled)
 7) [ ] Network configuration             8) [!] Root password
        (Not connected)                          (Password is not set.)
 9) [!] User creation
        (No user will be created)
  Please make your choice from above ['q' to quit | 'b' to begin installation | 'r' to refresh]:1
```

項目設定例として、以下に言語の設定例を記載します。<br>
設定を行う際には、項目選択時と同様に数字で設定を行っていきます。

```
Please select language support to install.　空エンター
[b to return to language list, c to continue, q to quit]: 32　####32を入力
================================================================================
================================================================================
Language settings

Available locales
 1)  Japanese (Japan)
Please select language support to install.
[b to return to language list, c to continue, q to quit]: 1　####1を入力
```

[インストールパラメータ例]

→マークは項目の入力が続いていくこと表しています。

|設定項目|設定値|備考|
|---|---|---|
|Language settings|32|Japanese|
|Timezone settings|5 → 70|Asia → Tokyo|
|Installation Destination|vda → Use All Space → LVM|全てデフォルトのためc(continue)を３回入力|
|Root password|password|任意の値|

3.IPアドレス設定

CentOS7からネットワーク設定はNetworkManagerから設定を行うことが推奨されています。<br>
そのため、nmcuiコマンドを使用して設定を行います。

eth0のネットワーク設定

```
zabbix# nmcli connection modify eth0 ipv4.method manual 
zabbix# nmcli connection modify eth0 connection.autoconnect "yes"
zabbix# nmcli connection modify eth0 ipv4.addresses "172.29.236.31/22"
zabbix# nmcli connection up eth0 
```

eth1のIPネットワーク設定

```
zabbix# nmcli connection modify eth1 ipv4.method manual 
zabbix# nmcli connection modify eth1 connection.autoconnect "yes"
zabbix# nmcli connection modify eth1 ipv4.addresses "172.28.1.69/24"
zabbix# nmcli connection modify eth1 ipv4.gateway "172.28.1.254"
zabbix# nmcli connection modify eth1 ipv4.dns "172.28.1.53"
zabbix# nmcli connection up eth1 
```

以降、SSHを使用してrootユーザにログインが可能となります。<br>
Ctrl+\^キーを押すことで、Zabbixサーバのコンソールから抜けることができます。

4./etc/hosts設定

Zabbixサーバの/etc/hostsにはOpenStack-Ansibleからデプロイされた各ホストのホスト名が設定されていないため、設定を行います。

Controllerノード#1(Ansibleを実行したノード)にログインし、/etc/hostsの設定を控えます。

```
controller#1$ tail -n +7 /etc/hosts
```

上記コマンドの実効結果をZabbixサーバのhostsに設定します。

```
zabbix# vi /etc/hosts/
```

### 1.3 Zabbix Serverセットアップ

Zabbix Serever,MariaDBのセットアップを行います。

1.パッケージインストール

Zabbixのレポジトリを登録し、Zabbix Serverのインストールを行います。

```
zabbix# rpm -ivh http://repo.zabbix.com/zabbix/3.0/rhel/7/x86_64/zabbix-release-3.0-1.el7.noarch.rpm
zabbix# yum install zabbix-server-mysql zabbix-web-mysql zabbix-web-japanese zabbix-agent mariadb mariadb-server
```

2.ZabbixサーバのDB設定

ここで指定するパスワードは後述する、MariaDBのzabbixユーザのものとなります。

```
zabbix# vi /etc/zabbix/zabbix_server.conf

#### コメントアウトを外す
DBHost=localhost

#### コメントアウトを外しパスワードを入力する。
DBPassword=zabbix
```

タイムゾーンを設定します。

```
zabbix# vi /etc/httpd/conf.d/zabbix.conf

#### コメントアウトを外し、Europe/RigaからAsia/Tokyoに設定する。
php_value date.timezone Asia/Tokyo
```

3.Mariadb設定

Zabbixで使用するMariaDBの設定を行います。

```
zabix# vi /etc/my.cnf.d/server.cnf

#### mysqldディレクティブに下記内容を追記
[mysqld]
character-set-server = utf8
collation-server     = utf8_general_ci
skip-character-set-client-handshake
innodb_file_per_table
```

設定後、MariaDBを起動します。

```
zabbix# systemctl start mariadb.service
```

4.MariaDB 起動確認

下記コマンドの実行結果がactive (running)となっていることを確認してください。

```
zabbix# systemctl status mariadb.service |grep Active
Active: active (running) since Fri 2016-07-01 10:12:18 JST; 5min ago
```

5.rootユーザーパスワード設定

MariaDBのrootユーザーパスワードはデフォルトで設定されないため、次のコマンドを使ってrootパスワードの設定を行います。

```
zabbix# mysql_secure_installation
...
Enter current password for root (enter for none): ←Enterキーを入力
OK, successfully used password, moving on...

Setting the root password ensures that nobody can log into the MariaDB
root user without the proper authorisation.

Set root password? [Y/n] y
New password:                    ←パスワードを2回入力
Re-enter new password:
Password updated successfully!
Reloading privilege tables..
 ... Success!

Remove anonymous users? [Y/n] y
Disallow root login remotely? [Y/n] n
Remove test database and access to it? [Y/n] y
Reload privilege tables now? [Y/n] y
```

6.MariaDBにZabbixDB作成

Zabbixが使用するDB及びユーザをMarriaDBに作成します。<br>
下記の例ではzabbixDBとzabbixユーザ(パスワードzabbix)を作成しています。

```
zabbix# mysql -u root -p
Enter password: ←MariaDBのrootユーザパスワードを入力

MariaDB [(none)]> CREATE DATABASE zabbix;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'localhost' IDENTIFIED BY 'zabbix';
```

zabbixDB及びzabbixユーザが作成されていることを確認します。

```
MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| zabbix             |
+--------------------+
4 rows in set (0.01 sec)

MariaDB [(none)]> show grants for 'zabbix'@'localhost';
+---------------------------------------------------------------------------------------------------------------+
| Grants for zabbix@localhost                                                                                   |
+---------------------------------------------------------------------------------------------------------------+
| GRANT USAGE ON *.* TO 'zabbix'@'localhost' IDENTIFIED BY PASSWORD '*DEEF4D7D88CD046ECA02A80393B7780A63E7E789' |
| GRANT ALL PRIVILEGES ON `zabbix`.* TO 'zabbix'@'localhost'                                                    |
+---------------------------------------------------------------------------------------------------------------+
2 rows in set (0.00 sec)
```

MariaDBからログアウトします。

```
MariaDB [(none)]> quit
```

ZabbixDBにZabbix Serverで使用するテーブルを作成します。<br>
zcatで指定するパスはインストールしたzabbix-server-mysqlのバージョンに依存するため、環境に合わせた値を使用してください。

```
zabbix# zcat /usr/share/doc/zabbix-server-mysql-3.0.3/create.sql.gz | mysql -uroot zabbix -p
Enter password: ←MariaDBのrootユーザパスワードを入力
```

テーブル作成後、show tablesの実行結果が空でないことを確認します。

```
zabbix# mysql -u root -p zabbix -e "show tables;"
Enter password: ←MariaDBのrootユーザパスワードを入力
+----------------------------+
| Tables_in_zabbix           |
+----------------------------+
| acknowledges               |
| actions                    |
| alerts                     |
| application_discovery      |
| application_prototype      |
~~~~~~~~~~~~省略~~~~~~~~~~~~~~~
| triggers                   |
| users                      |
| users_groups               |
| usrgrp                     |
| valuemaps                  |
+----------------------------+

```
7.サービス自動起動設定

Zabbix Server、Zabbix-agent、Apache、MariaDBの自動起動設定を行います。

```
zabbix# systemctl enable zabbix-server.service
zabbix# systemctl start zabbix-server
zabbix# systemctl enable zabbix-agent.service
zabbix# systemctl start zabbix-agent
zabbix# systemctl enable httpd.service
zabbix# systemctl start httpd
zabbix# systemctl enable mariadb.service
```

各サービスが起動していることを確認します。<br>
下記３コマンドの実行結果に Active: active (running) が出力されていることを確認してください。

```
zabbix# systemctl status zabbix-server.service |grep Active
zabbix# systemctl status zabbix-agent.service |grep Active
zabbix# systemctl status httpd.service |grep Active
```


### 1.4 Zabbix Server Web GUI初期セットアップ

1.ZabbixのWeb GUIにアクセス

ブラウザでZabbixのWEB GUIにアクセスします。

```
http://<ZabbixサーバのIPアドレス>/zabbix
```

2.初期セットアップ

初回ログイン時には下記画面が表示されます。<br>
Nextボタンをクリックすることでセットアップを進められます。<br>
画面の指示に従ってセットアップを進めてください。

![welcome](images/zabbix/welcome.png)

「2. Check of pre-requisites」は、システム要件を満たしている（全てOKとなっている）ことを確認します。

![バージョンチェック画面](images/zabbix/version_check.png)

「3. Configure DB connection」では下記を参考に設定を行ってください。

|項目|設定値|
|:---|:---|
|Database type|MySQL|
|Database host|localhost|
|Database Port|0|
|Database name|zabbix|
|User|zabbix|
|Password|zabbix|


「4. Zabbix server details」はZabbix Serverのインストール場所の指定です。本例ではそのまま次に進みます。

「5. Pre-Installation summary」で設定を確認し、問題なければ次に進みます。

「6. Install」で設定ファイルのパス（/etc/zabbix/zabbix.conf.php）が表示されるので確認し「Finish」ボタンをクリックします。

ログイン画面が表示されます。デフォルトユーザでログインを行えます。<br>
(ユーザ Admin / パスワード zabbix)

![ログイン画面](images/zabbix/login.png)

### 1.5 Zabbix Server 監視設定

1.Zabbixテンプレート登録

ZabbixのWebGUIから監視テンプレートのインポートを行います。
下記リンクからテンプレートのダウンロードを行ってください。

[テンプレート](configs/zabbix/OpenStack_templates_Kilo_OpenStack_ansible.xml)

テンプレートダウンロード後、ZabbixのGUIからテンプレートをインポートします。

「設定」→「テンプレート」の順にクリックし、「テンプレート」画面を表示します。<br>
画面右上の「インポート」ボタンをクリックします。

![テンプレート画面](images/zabbix/template.png)

「ファイルを選択」ボタンをクリックし、ダウンロードしたテンプレートファイルを選択します。<br>
その後、「インポート」ボタンをクリックし、テンプレートファイルをインポートします。

![インポート画面](images/zabbix/template_import.png)

2.ZabbixAgent 自動登録設定

Zabbixに未登録の監視対象ホストを自動的にZabbixへ登録するために、Zabbixの自動登録機能の設定を行います。<br>
この機能を使用することで、Zabbixが管理していないZabbix Agentが見つかった場合、自動的に監視対象としての登録・監視テンプレートの割り当てが行えます。<br>
本設定は、テンプレートのインポートでは自動的に設定されないため、手動で登録します。

[設定」→「アクション」の順にクリックします。<br>
表示された画面で画面右上に表示されている「イベントソース」のドロップダウンメニューから「自動登録」を選択します。<br>
その後、「アクションの作成」ボタンをクリックします。

![自動登録画面](images/zabbix/auto_registration.png)

アクションの作成画面では、下表を参考に各タブを入力します。

|アクションタブ<br>名前|アクションの実行条件タブ<br>アクションの実行条件(計算のタイプはAnd)|アクションの実行内容タブ<br>アクションの実行内容|
|:---|:---|:---|
|Automatic registration of Cinder\_API|・ホストメタデータ 含まれる BBTlab2016<br>・ホスト名 含まれる \_cinder\_api\_container|・ホストを追加<br>・ホストグループに追加: Cinder, OpenStack<br>・テンプレートとリンク: Template\_OpenStack\_Common<br>・テンプレートとリンク: Template\_OpenStack\_Cinder\_API|
|Automatic registration of Cinder\_Scheduler|・ホストメタデータ 含まれる BBTlab2016<br>・ホスト名 含まれる \_cinder\_scheduler\_container|・ホストを追加<br>・ホストグループに追加: Cinder, OpenStack<br>・テンプレートとリンク: Template\_OpenStack\_Common<br>・テンプレートとリンク: Template\_OpenStack\_Cinder\_Scheduler|
|Automatic registration of Glance|・ホストメタデータ 含まれる BBTlab2016<br>・ホスト名 含まれる \_glance\_container|・ホストを追加<br>・ホストグループに追加: Glance, OpenStack<br>・テンプレートとリンク: Template\_OpenStack\_Common<br>・テンプレートとリンク: Template\_OpenStack\_Glance|
|Automatic registration of Horizon|・ホストメタデータ 含まれる BBTlab2016<br>・ホスト名 含まれる \_horizon\_container|・ホストを追加<br>・ホストグループに追加: Horizon, OpenStack<br>・テンプレートとリンク: Template\_OpenStack\_Common<br>・テンプレートとリンク: Template\_OpenStack\_Horizon|
|Automatic registration of Keystone|・ホストメタデータ 含まれる BBTlab2016<br>・ホスト名 含まれる \_keystone\_container|・ホストを追加<br>・ホストグループに追加: Keystone, OpenStack<br>・テンプレートとリンク: Template\_OpenStack\_Common<br>・テンプレートとリンク: Template\_OpenStack\_Keystone|
|Automatic registration of Memcached|・ホストメタデータ 含まれる BBTlab2016<br>・ホスト名 含まれる \_memcached\_container|・ホストを追加<br>・ホストグループに追加: Memcached, OpenStack<br>・テンプレートとリンク: Template\_OpenStack\_Common<br>・テンプレートとリンク: Template\_OpenStack\_Memcached|
|Automatic registration of MySQL(Galera)|・ホストメタデータ 含まれる BBTlab2016<br>・ホスト名 含まれる \_galera\_container|・ホストを追加<br>・ホストグループに追加: MySQL, OpenStack<br>・テンプレートとリンク: Template\_OpenStack\_Common<br>・テンプレートとリンク: Template\_OpenStack\_MySQL|
|Automatic registration of Neutron_Agents|・ホストメタデータ 含まれる BBTlab2016<br>・ホスト名 含まれる \_neutron\_agents\_container|・ホストを追加<br>・ホストグループに追加: Neutron, OpenStack<br>・テンプレートとリンク: Template\_OpenStack\_Common<br>・テンプレートとリンク: Template\_OpenStack\_Neutron\_Network|
|Automatic registration of Neutron_Server|・ホストメタデータ 含まれる BBTlab2016<br>・ホスト名 含まれる \_neutron\_server\_container|・ホストを追加<br>・ホストグループに追加: Neutron, OpenStack<br>・テンプレートとリンク: Template\_OpenStack\_Common<br>・テンプレートとリンク: Template\_OpenStack\_Neutron\_Controller|
|Automatic registration of Nova\_API\_Metadata|・ホストメタデータ 含まれる BBTlab2016<br>・ホスト名 含まれる \_nova\_api\_matadata\_container|・ホストを追加<br>・ホストグループに追加: Nova\_Controller, OpenStack<br>・テンプレートとリンク: Template\_OpenStack\_Common<br>・テンプレートとリンク: Template\_OpenStack\_Nova\_API\_Metadata|
|Automatic registration of Nova\_API\_OS\_Compute|・ホストメタデータ 含まれる BBTlab2016<br>・ホスト名 含まれる \_nova\_api\_os\_compute|・ホストを追加<br>・ホストグループに追加: Nova\_Controller, OpenStack<br>・テンプレートとリンク: Template\_OpenStack\_Common<br>・テンプレートとリンク: Template\_OpenStack\_Nova\_API\_OS\_Compute|
|Automatic registration of Nova\_Cert|・ホストメタデータ 含まれる BBTlab2016<br>・ホスト名 含まれる \_nova\_cert\_container|・ホストを追加<br>・ホストグループに追加: Nova\_Controller, OpenStack<br>・テンプレートとリンク: Template\_OpenStack\_Common<br>・テンプレートとリンク: Template\_OpenStack\_Nova\_Cert|
|Automatic registration of Nova\_Compute|・ホストメタデータ 含まれる BBTlab2016<br>・ホスト名 含まれる comp<br>・ホスト名 ***【含まれない】*** \_container-|・ホストを追加<br>・ホストグループに追加: Nova\_Compute, OpenStack<br>・テンプレートとリンク: Template\_OpenStack\_Common<br>・テンプレートとリンク: Template\_OpenStack\_Nova\_Compute|
|Automatic registration of Nova\_Conductor|・ホストメタデータ 含まれる BBTlab2016<br>・ホスト名 含まれる \_nova\_conductor\_container|・ホストを追加<br>・ホストグループに追加: Nova\_Controller, OpenStack<br>・テンプレートとリンク: Template\_OpenStack\_Common<br>・テンプレートとリンク: Template\_OpenStack\_Nova\_Conductor|
|Automatic registration of Nova\_Cosole|・ホストメタデータ 含まれる BBTlab2016<br>・ホスト名 含まれる \_nova\_cosole\_container|・ホストを追加<br>・ホストグループに追加: Nova\_Controller, OpenStack<br>・テンプレートとリンク: Template\_OpenStack\_Common<br>・テンプレートとリンク: Template\_OpenStack\_Nova\_Cosole|
|Automatic registration of Nova\_Scheduler|・ホストメタデータ 含まれる BBTlab2016<br>・ホスト名 含まれる \_nova\_scheduler\_container|・ホストを追加<br>・ホストグループに追加: Nova\_Controller, OpenStack<br>・テンプレートとリンク: Template\_OpenStack\_Common<br>・テンプレートとリンク: Template\_OpenStack\_Nova\_scheduler|
|Automatic registration of RabbitMQ|・ホストメタデータ 含まれる BBTlab2016<br>・ホスト名 含まれる \_rabbit\_mq\_container|・ホストを追加<br>・ホストグループに追加: RabbitMQ, OpenStack<br>・テンプレートとリンク: Template\_OpenStack\_Common<br>・テンプレートとリンク: Template\_OpenStack\_RabbitMQ|

## 2. Elasticsearch/Kibana/Kophセットアップ
Elasticsearchはelastic社が開発しているビックデータの蓄積・解析が行えるOSSです。
KibanaやKophをインストールすることで、Elasticsearchに格納されたデータのKibanaによる格納データの分析・可視化及びKophによるElasticsearchのパフォーマンス情報の可視化を行えます。

以下の手順はElasticsearchサーバで実施します。

### 2.1 Elasticsearchセットアップ

1.Javaインストール
ElasticsearchはJavaアプリケーションのため、Javaをインストールします。

```
elastic$ sudo aptitude install software-properties-common
elastic$ sudo add-apt-repository -y ppa:webupd8team/java
elastic$ sudo apt-get update
elastic$ sudo apt-get -y install oracle-java8-installer
```

2.Elasticsearchパッケージインストール

Elasticsearchのインストールを行います。

```
elastic$ sudo apt-get install curl wget -qO - https://packages.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
elastic$ echo "deb http://packages.elastic.co/elasticsearch/2.x/debian stable main" | sudo tee -a /etc/apt/sources.list.d/elasticsearch-2.x.list
elastic$ sudo apt-get update && sudo apt-get install elasticsearch
```

3.Elasticsearch自動起動設定

Elasticsearchの自動起動設定を行います。

```
elastic$ sudo update-rc.d elasticsearch defaults 95 10
elastic$ sudo service elasticsearch start
```

4.bridge-utils、vlanパッケージインストール

他サーバとネットワークインターフェース構成を合わせるためにbridge-utils、vlanパッケージをインストールします。

```
elastic$ apt-get install bridge-utils
elastic$ apt-get install vlan
```

5.IPアドレス設定

```
elastic$ sudo vi /etc/network/interfaces
 
auto lo
iface lo inet loopback

auto em1
 iface em1 inet static
 address 172.28.1.67
 netmask 255.255.255.0
 gateway 172.28.1.254
 dns-nameservers 172.28.1.53

auto em2
 iface em2 inet manual

iface em2.2811 inet manual
 vlan-raw-device em2

auto br-mgmt
 iface br-mgmt inet static
 bridge_stp off
 bridge_waitport 0
 bridge_fd 0
 bridge_ports em2.2811
 address 172.29.236.17
 netmask 255.255.252.0
```

6.ホスト再起動

設定後ホストの再起動を行います。

```
elastic$ sudo reboot
```

### 2.2 その他Elasticsearch設定

Elastichsechに蓄積されていくログが格納されるindexのローテーションする設定を行います。<br>
ローテションはcurator(<https://github.com/elastic/curator>)を使用します。

1.curatorインストール

curatorのインストールにpipコマンドを使用します。pipのインストールを行った後、curatorをインストールします。

```
elastic$ sudo apt-get install python
elastic$ cd /tmp
elastic$ wget https://raw.github.com/pypa/pip/master/contrib/get-pip.py
elastic$ sudo python get-pip.py
elastic$ sudo pip install elasticsearch-curator
```

2.cron設定

curatorのインストール後、cronの設定を行います。念のため現在のcrontab設定をバックアップします。

```
elastic$ crontab -l > ~/crontab_`date "+%Y%m%d"`
```

3.スクリプト配置

indexのローテションを行うスクリプトを配置します。<br>
下記リンクからローテションスクリプトをダウンロードしてください。<br>
[indexローテションスクリプト](scripts/Elastichserch/curator_sh.tar.gz)

ダウンロード後、Elastichserchサーバにログインし、カレントディレクトリに圧縮ファイルをコピーします。<br>
コピー完了後、下記コマンドでファイルを展開します。

```
elastich$ sudo tar zxvf curator_sh.tar.gz -C / 
```

4.cron設定

スクリプトを配置後、crontabにスクリプトを登録します。

```
elastich$ sudo crontab -e

15 02 * * * /etc/cron.d/curator.sh 
```

crontabの初回実行時には下記のようなcrontabの編集に使用するエディタを選択するメニューが表示されます。<br>
使いやすいエディタの数字を入力してエンターを押してください。(下記例ではnanoを選択)

```
no crontab for root - using an empty one

Select an editor.  To change later, run 'select-editor'.
  1. /bin/ed
  2. /bin/nano        <---- easiest
  3. /usr/bin/vim.basic
  4. /usr/bin/vim.tiny

Choose 1-4 [2]: 2
```

下記コマンドを実行し、設定した内容が表示されることを確認します。

```
elastic$ sudo crontab -l
# Edit this file to introduce tasks to be run by cron.

〜〜〜省略〜〜〜

15 2 * * * root /etc/cron.d/curator.sh
```

### 2.3 Kibana/Kophセットアップ

1.Kibanaパッケージインストール

Kibanaのインストールを行います。

```
elastic$ echo "deb http://packages.elastic.co/kibana/4.5/debian stable main" | sudo tee -a /etc/apt/sources.list
elastic$ sudo apt-get update && sudo apt-get install kibana
```

2.Kophインストール

ElasticsearchのプラグインであるKophをインストールします。

```
elastic$ /usr/share/elasticsearch/bin/plugin install lmenezes/elasticsearch-kopf/v2.1.1
```

3.Kibana自動起動設定

```
elastic$ sudo update-rc.d kibana defaults 95 10
elastic$ sudo service kibana start
```

4.Kibana GUIにアクセス

ブラウザで下記URLにアクセスします。

```
http://<ElasticsearchサーバIP>:5601
```

アクセス後、下記のような画面が出力されます。

![KibanaのGUI](images/Elasticsearch/Kibana/Kibana_GUI.png)

5.Koph GUIにアクセス

ブラウザで下記URLにアクセスします。

```
http://<ElasticsearchサーバIP>:9200/_plugin/kopf/#!/cluster
```

アクセス後、下記のような画面が出力されます。

![KophのGUI](images/Elasticsearch/Koph/Koph_GUI.png)


## 3. fluentd/Zabbix Agentセットアップ

Controllerノード#1からAnsibleを使用して各ホストにfluentd、Zabbix Agentのインストールを行います。

以下の手順は指定がない限り、Controllerノード#1から実施します。

### 3.1 事前準備

OpenStack-Ansibleからデプロイされていないホスト(Elasticsearchサーバ、Zabbixサーバ、KVMサーバ)には、Controllerノード#1のrootユーザの公開鍵が登録されていないため手動でSSH鍵を登録します。

1.Controller#1の公開鍵確認

公開鍵を表示します。表示された公開鍵を控えてください。

```
Controller#1$ sudo cat /root/.ssh/id_rsa.pub
```

2.Elasticsearchサーバ、Zabbixサーバ、KVMサーバへ鍵登録

手順1で控えた実行結果を、Elasticsearchサーバ、Zabbixサーバ、KVMサーバの authorized_keys ファイルへ公開鍵を貼り付けます。<br>
それぞれのサーバにログインし下記コマンドを実行して公開鍵を書き込みます。

```
sudo mkdir /root/.ssh
sudo vi /root/.ssh/authorized_keys
```

3.SSHログイン確認

Controllerノード#1のrootユーザからElasticsearchサーバ、Zabbixサーバ、KVMサーバのrootユーザに鍵認証ログインが行えることを確認します。<br>
Controllerノード#1からそれぞれのサーバのIPアドレスに対して下記コマンドを実行し、パスワードなしでログインができることを確認します。

```
Controller#1$ sudo ssh <Elasticsearchサーバ、Zabbixサーバ、KVMサーバのIPアドレス>
```


4./etc/hosts編集

Zabbixサーバ等のOpenStack-Ansibleによってデプロイされていないサーバは手動にて/etc/hostsに追記を行います。<br>この時hostsに記載するIPアドレスはManagement NWセグメントの物を設定します。

```
Controller#1$ sudo vi /etc/hosts
```

5.Ansible実行事前作業

Ansibleで使用する作業用ディレクトリを作成します。

```
Controller#1$ mkdir ~/ansible
Controller#1$ cd ~/ansible
```

6.インベントリファイル作成

AnsibleのインベントリファイルをOpenStack-Ansibleで設定された/etc/hostsをから作成します。

```
Controller#1$ tail -n +8 /etc/hosts  | awk '{print $2}' > hostlist
```

### 3.2 fluentdセットアップ

fluentdとはTREASURE DATA社が開発しているログファイルの収集・転送・集約が行えるOSSです。<br>
ログに正規表現でタグを付けしElasticsearchに転送することで、Kibanaを使用したログ解析・可視化が行えます。<br>
本項でfluentdのセットアップを行います。

1.fluentd一斉インストール

Ansibleを使用してfluentdを各ホストへインストールを行うために下記のyamlファイルを作成します。

```
Controller#1$ vi fluent-install.yml

---
- hosts: all
  user: root
  tasks:
    - name: fluentd agent install
      shell: curl -L https://toolbelt.treasuredata.com/sh/install-ubuntu-trusty-td-agent2.sh | sh
    - wait_for: path=/var/run/td-agent/td-agent.pid  
    - name: multi-format-parser-plugin install
      shell: /opt/td-agent/embedded/bin/fluent-gem install fluent-plugin-multi-format-parser
    - name: Elasticsearch-plugin install
      shell: /opt/td-agent/embedded/bin/fluent-gem install fluent-plugin-elasticsearch
```

Playbook実行した際に対象ホストのSSH鍵を自動的にknown_hostsに登録する設定を行います。

```
$ vi ansible.cfg

[defaults]
host_key_checking = False
```

作成したyamlファイルを使用してfluentdをインストールします。

```
Controller#1$ ansible-playbook -i hostlist fluent-install.yml
```

2.flentd.conf配布準備

Controllerノード#1から各ホストに対してfluentdの設定ファイルをスクリプトで展開します。

はじめにControllerノード#1にスクリプト及び関連ファイルを配置します。<br>
下記リンクからAPI監視スクリプトをダウンロードしてください。

[fluentd設定ファイル](configs/fluetnd/fluentd_configs.tar.gz)

ダウンロード後、Controllerノード#1(Ansibleを実行したノード)にログインし、カレントディレクトリに圧縮ファイルをコピーします。<br>
コピー完了後、下記コマンドでファイルを展開します。

```
controller#1$ sudo tar zxvf fluentd_configs.tar.gz -C / 
```

上記コマンド実行後に/opt/ops-manager/fluentd/configs/配下に各コンポーネント・サービス毎のfluentdの設定ファイルが格納されたディレクトリが展開されるため、展開されたディレクトリに移動します。

```
controller#1$ /opt/ops-manager/fluentd/configs/
```

3.設定配布スクリプト使用前設定

環境によってホスト名が変化する可能性があるノードのホスト名を設定ファイルに記載します。<br>
NovaComputeノード、CinderVolumeノード及びElasticsearchが動作するサーバのホスト名を記載してください。。<br>
ここで設定するファイルは、それぞれのノードが増減するたびに設定変更をしてください。

OpenStackのController LXCが動作するノードのホスト名のみを記載します。<br>
複数台ノードが存在すつ場合は、全て記載してください。<br>
設定するホスト名は環境に合わせて変更してください。

```
controller#1$ vi controller_LXC_HOST.txt

cont001
cont002
cont003
```

Elasticsearchが動作するノードのホスト名のみを記載します。<br>
複数台ノードが存在すつ場合は、全て記載してください。<br>
設定するホスト名は環境に合わせて変更してください。

```
controller#1$ vi elasticsearch.txt

elas001
```

NovaComputeサーバのホスト名のみを記載します。<br>
複数台ノードが存在すつ場合は、全て記載してください。<br>
設定するホスト名は環境に合わせて変更してください。

```
controller#1$ vi nova_compute.txt

comp001
comp002
```

CinderVolumeが動作するノードのみを記載します。<br>
複数台ノードが存在すつ場合は、全て記載してください。<br>
設定するホスト名は環境に合わせて変更してください。

```
controller#1$ vi cinder_volume.txt

zabk001
```

4.設定配布スクリプト実行

スクリプト実行し全ホストに fluentdの設定ファイルを配布及び設定反映を行います。

```
controller#1$ sudo ./scp all 2>&1 | tee td_agent_conf_scp.log
```

以下はスクリプトの説明となります。

特定の種別のノードだけに対してfluentdの設定ファイルの配布及び設定反映を行いたい場合は、
スクリプトの引数に配布したいノードのfluentdの設定ファイルが格納されているディレクトリ名を指定してスクリプトを実行します。<br>

例) ３つのコンポーネント(cinder_api_container elasticsearch galera_container)に対して設定ファイルの配布・反映を行いたい

```
controller#1$ ls -1

cinder_api_container
cinder_scheduler_container
cinder_volume
controller_LXC_HOST
elasticsearch
galera_container
```

スクリプトの引数にfluentdの設定ファイルが格納されているディレクトリ名を指定し実行すると対象の設定ファイルが配布されます。

```
controller#1$ sudo ./scp cinder_api_container elasticsearch galera_container 
```

### 3.3 Zabbix Agent一斉インストール準備

Zabbix Agentのインストールは ansible-zabbix-agent(<https://github.com/dj-wasabi/ansible-zabbix-agent>)を使用して一括で行います。<br>
以下、Controllerノード#1で実行します。

1.ansible-zabbix-agentセットアップ

ansible-zabbix-agentのPlaybookをインストールします。

```
controller#1$ sudo ansible-galaxy install dj-wasabi.zabbix-agent
```

2.作業用ディレクトリに移動

Ansibleの設定ファイルを作成する作業用のディレクトリに移動します。

```
controller#1$ cd ~/ansible
```

3.ansible-galaxy yamlファイル編集

ansible-zabbix-agentで使用するyamlファイルを設定します。
agent\_listenip設定中のansible_eth1はインストール対象ホストでZabbix agentの通信で使用するインターフェース名を指定してください。<br>

```
controller#1$ vi zabbix-agent.yml

---
- hosts: all
  roles:
     - role: dj-wasabi.zabbix-agent
       agent_server: 172.29.236.31
       agent_serveractive: 172.29.236.31
       agent_listenip: "{{ ansible_eth1.ipv4.address }}"
       agent_hostmetadata: BBTlab2016
       zabbix_api_use: false 
```

**Zabbix Agentをインストールするホスト毎に監視の通信が流れるインターフェイス名が異なる場合は**、以下の手順を実施してください。<br>
インタフェース名が同一の場合、下記手順3,4を実施する必要はありません。

4.インベントリファイル分割
インベントリファイルの内容を同一インターフェイス名を持つホスト毎にグループ化します。

```
例)
controller#1$ vi ./hostlist

[lxc]###インターフェイス名eth1のホスト
cont001_keystone_container・・・
cont002_rabbitmq_container・・・

[hostos]###インターフェイス名br-mgmtのホスト
cont001
comp001
zabi001
```

5.zabbix-agent.yamlファイル修正

複数のインターフェイスパターンに対応するために、インターフェイス名を変数に格納します。<br>
手順1.で定義したホストグループ毎にインターフェイス名を定義します。

```
controller#1$ vi zabbix-agent.yml

---
[lxc]
agent_if: eth1
[hostos]
agent_if: br-mgmt
- hosts: all
  roles:
     - role: dj-wasabi.zabbix-agent
       agent_server: 172.29.236.31
       agent_serveractive: 172.29.236.31
       agent_listenip: "{{ ansible_[agent_if].ipv4.address }}"
       agent_hostmetadata: BBTlab2016
       zabbix_api_use: false 
```

### 3.4 Zabbix Agent一斉インストール

3.3項で**インタフェース名毎にインベントリファイルの内容を分けた場合は手順2のみ**を実施してください。<br>
分かれていない場合手順1のみを実施します。

1.Zabbix Agent一斉インストール

Playbookを実行しZabbix Agentを各ホストへインストールします。

```
controller#1$ sudo ansible-playbook -i ./hostlist ./zabbix-agent.yml
```

2.Zabbix Agentインストール

Ansibleの -lオプションでホストグループを指定してそれぞれに対して実行します。

```
controller#1$ sudo ansible-playbook -i ./hostlist -l lxc ./zabbix-agent.yml
controller#1$ sudo ansible-playbook -i ./hostlist -l hostos ./zabbix-agent.yml
```

### 3.5 Zabbix Agent個別設定

1.OpenStackにAPI監視用プロジェクト・ユーザ作成

OpenStackにZabbixでAPI監視を行うためのプロジェクトとユーザを作成します。<br>
Controller#1のutility_containerにログインします。<br>
ログイン先コンテナ名は環境に合わせてご変更ください。

```
controller#1$ sudo ssh cont001_utility_container-db4b0fe9
```

ログイン完了後、下記コマンドを実行します。

```
utility_container# . openrc
utility_container# openstack project create zabbix_monitoring
utility_container# openstack user create zabbix --password password
utility_container# openstack role add _member_ --user zabbix --project zabbix_monitoring
```

下記コマンドで、プロジェクト、ユーザが正常に作成されているか確認を行います。<br>
下記のような結果が返ってくることを確認します。

```
utility_container# openstack role list --user zabbix --project zabbix_monitoring
+----------------------------------+----------+-------------------+--------+
| ID                               | Name     | Project           | User   |
+----------------------------------+----------+-------------------+--------+
| 9fe2ff9ee4384b1894a90878d3e92bab | _member_ | zabbix_monitoring | zabbix |
+----------------------------------+----------+-------------------+--------+

```

2.API監視用スクリプト配置

Controller#1にAPI監視用のスクリプトを配置し、Zabbix Agentの設定変更を行います。

配置するスクリプト及び関連ファイルを下記リンクからダウンロードしてください。

[API監視スクリプト](scripts/Controller_1/api_sh.tar.gz)

ダウンロード後、Controllerノード#1(Ansibleを実行したノード)にログインし、カレントディレクトリに圧縮ファイルをコピーします。<br>
コピー完了後、下記コマンドでファイルを展開します。

```
controller#1$ sudo tar zxvf api_sh.tar.gz -C / 
```

3.スクリプト編集

展開後、下記コマンドで[1.OpenStackにAPI監視用プロジェクト・ユーザ作成]で作成したzabbixユーザのパスワードをスクリプトに反映します。

```
controller#1$　sed -i "s/OpenStack_Zabbix_User_Password/<OpenStackのAPI監視用Zabbixユーザパスワード>/" /etc/zabbix/scripts/api.sh
```

設定を反映させるためZabbix Agentサービスを再起動します。

```
controller#1$ sudo service zabbix-agent restart
```

4.API監視テンプレート紐付け設定

ZabbixのController#1ホストにAPI監視テンプレートの紐付けを行います。

ZabbixのWebGUIにログイン後、「設定」→「ホスト」の順にクリックします。<br>

![ホスト一覧画面](images/zabbix/host_list.png)

「名前」行に表示されているController#1のホスト名をクリックし、Controller#1ホストの設定画面が表示されます。<br>
表示された画面中程の[テンプレート]をクリックします。

![Controller#1覧画面](images/zabbix/cont001.png)

下記画像のような画面が表示されるので、「新規	テンプレートをリンク」のテキストボックスに
Template_OpenStack_API_Status　を入力し、「追加」→「更新」の順にクリックし、テンプレートを紐付けます。

![APIテンプレート割り当て](images/zabbix/API_template.png)

5.RabbitMQ監視スクリプト配置

RabbitMQの監視にはミラクル・リナックス社が公開しているスクリプトを使用します。<br>
[ミラクル・リナックス社サイト](https://www.miraclelinux.com/labs/download/openstack-monitoring)

RabbitMQが保持しているキューの長さを監視するため、RabbitMQホストにスクリプトを配置します。

スクリプト及び関連ファイルを下記リンクからダウンロードしてください。

[MessageQue監視スクリプト](scripts/rabbitmq/rabbitmq_script.tar.gz)

ダウンロード後、各RabbitMQノードにログインし、カレントディレクトリに圧縮ファイルをコピーします。<br>
コピー完了後、下記コマンドでファイルを展開します。

```
controller#1$ rabbit_mq_container$ sudo tar zxvf rabbitmq_script.tar.gz -C /
```

設定を反映させるためZabbix Agentサービスを再起動します。

```
controller#1$ rabbitmq#1#$ sudo service zabbix-agent restart
```


## 4. Hatoholセットアップ

HatoholとはMIRACLE LINUX社が開発している複数のZabbix・NagiosのGUIを１つにまとめて表示が行えるOSSです。<br>
本環境では、Zabbix Serverをセットアップしたサーバと同一サーバにHatoholをセットアップします。

以下の手順はZabbixサーバにて行います。

### 4.1 Hatoholインストール

Hatoholサーバのインストール及び、MariaDBの設定を行います。

1.リポジトリ登録

Hatoholプロジェクトののリポジトリを登録します。

```
zabbix# yum install wget
zabbix# wget -P /etc/yum.repos.d/ http://project-hatohol.github.io/repo/hatohol-el7.repo
```

2.epelリポジトリ登録

Hatoholのインストールにはepelリポジトリからパッケージ取得が必要となるため、epelリポジトリパッケージをインストールします。

```
zabbix# yum install epel-release
zabbix# yum update
```

3.Hatoholをインストールします。

```
zabbix# yum install hatohol-server hatohol-web
```

執筆時点でHatoholはyumコマンドを使用したバージョンアップが行えません。
<br>そのため、yumコマンドの対象から外すために、hatoholリポジトリーを無効化します。

```
zabbix# sed -i "s/enabled=1/enabled=0/g" /etc/yum.repos.d/hatohol-el7.repo
```

### 4.2 MariaDB設定

1.Hatohol データベースの初期化

MarriaDBにHatoholで使用するデータベースを作成します。

```
zabbix# hatohol-db-initiator --db_user root --db_password <MariaDBのrootパスワード>
```

上記コマンドを実行した場合、hatoholユーザ及び、hatoholデータベースがMariaDBに作成されます。
<br>これらを変更する場合、事前に/etc/hatohol/hatohol.confの編集が必要となります。

2.Hatohol Web用DBの作成

MarriaDBにHatoholのWeb GUIで使用するデータベースを作成します。

```
zabbix# mysql -u root -p
Enter password:   ←設定したMariaDBのrootユーザパスワードを入力
MariaDB > CREATE DATABASE hatohol_client DEFAULT CHARACTER SET utf8;
MariaDB > GRANT ALL PRIVILEGES ON hatohol_client.* TO hatohol@localhost IDENTIFIED BY 'hatohol';
MariaDB > quit
```

3.Hatohol Web用DBへのテーブル追加

手順2で作成したデータベースにHatoholのWeb GUIで使用するテーブルを作成します。

```
zabbix# /usr/libexec/hatohol/client/manage.py syncdb
```

### 4.3 SELinux設定

1.SElinux無効化

Hatoholは執筆時点ではSELinuxによる強制アクセス制御機能が有効化されていると正常に動作しません。<br>
そのため、SELinuxを無効化します。

```
zabbix# setenforce 0
```

SELinuxの実行モードをgetenforceコマンドにより確認します。<br>
この時Permissiveとなっていることを確認します。

```
zabbix# getenforce
Permissive
```

OS再起動後もSELinuxを無効化とする設定を行います。

```
zabbix# sed -i s/SELINUX=enforcing/SELINUX=disabled/ /etc/sysconfig/selinux
```

### 4.4 サービス自動起動設定

1.Hatoholサーバ及びApacheの自動起動設定

```
zabbix# systemctl enable hatohol
zabbix# systemctl start hatohol
zabbix# systemctl enable httpd
zabbix# systemctl start httpd

```

上記実行後、下記コマンドでhatoholプロセス起動確認を行います。

```
# systemctl status hatohol | egrep "Active|Main PID"
```

### 4.5 RabbitMQのインストール

1.パッケージインストール

Hatohol Arm Plugin Interface 2 (HAPI2)のセットアップに必要なRabbitMQのインストールを行います。<br>
EPELリポジトリーより、RabbitMQとErlangをインストールします。<br>
RabbitMQは本書の執筆時点(2016年6月現在)で最新のバージョン3.6.2をインストールします。

```
zabbix# yum install erlang
zabbix# rpm --import https://www.rabbitmq.com/rabbitmq-signing-key-public.asc
zabbix# yum install https://www.rabbitmq.com/releases/rabbitmq-server/v3.6.2/rabbitmq-server-3.6.2-1.noarch.rpm
```

2.RabbitMQサービス起動

Hatoholで使用するRabbitMQを起動します。

```
zabbix# systemctl enable rabbitmq-server
zabbix# systemctl start rabbitmq-server
```

3.RabbitMQの各種設定

RabbitMQにhatoholバーチャルホストを作成します。<br>
作成したバーチャルホストにアクセスするユーザーとしてhatoholユーザーを作成し、必要なパーミッションを設定します。<br>
以下はRabbitMQのパスワードをhatoholにする例です。

```
zabbix# rabbitmqctl add_vhost hatohol
zabbix# rabbitmqctl add_user hatohol hatohol
zabbix# rabbitmqctl set_permissions -p hatohol hatohol ".*" ".*" ".*"
```

### 4.6 HAP2 Zabbixプラグインのインストール

HatoholにZabbix用HAP2プラグインをインストールします。

```
zabbix# yum --enablerepo=hatohol install hatohol-hap2-zabbix
zabbix# systemctl restart hatohol
```

1.MariaDBにHAPI2設定追加

MariaDBにHAPI2のデータを追加します。

```
zabbix# hatohol-db-initiator --db_user root --db_password <MariaDBのrootパスワード>
zabbix# systemctl restart hatohol
```

2.Hatohol WEB GUIにアクセス

本環境ではZabbixサーバにHatoholをインストールしているため、ZabbixサーバのIPアドレスにブラウザでアクセスします。<br>
admin/hatohol（初期パスワード）でログインできます。

```
http://<zabbixサーバのIPアドレス>/hatohol/
```

3.HatoholにZabbixサーバーを登録

Zabbixサーバーの情報を追加します。Hatohol Webにログインしたら、上部のメニューバーの「設定」→「監視サーバー」の順にクリックします。<br>
「監視サーバー」の画面に切り替わったら「監視サーバー追加」ボタンをクリックしてノードを登録します。

|     項目       | 設定値 |
|---|---|
| 監視サーバータイプ | Zabbix (HAPI2) |
| ニックネーム       | zabbix |
| Zabbix API URL |  http://\<zabbixサーバーのIPアドレス>/zabbix/api_jsonrpc.php |
| ユーザー名 |  \<Zabbixのユーザ名(本環境ではAdmin)>|
| パスワード|  \<Zabbixに設定したユーザのパスワード(本環境ではzabbix)>|
| ポーリング間隔（秒）| 30 |
| リトライ間隔（秒）| 10 |
| パッシブモード  | オフ(チェックを入れない) |
| ブローカーURL | amqp://hatohol:hatohol@localhost/hatohol |
| その他項目 | 空白 |

設定が終わったら適用ボタンを押します。

ページを再読み込みして、通信状態が「初期状態」から「正常」になる事を確認します。