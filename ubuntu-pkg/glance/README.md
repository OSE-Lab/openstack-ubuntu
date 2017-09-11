# Glance 安裝與設定
OpenStack 的 Glance 提供了映像檔服務，該服務是整個 OpenStack 基礎架構的核心之一，其提供一套 API 來存取映像檔與詮釋資料(Metadata)的定義，並且支援多種儲存類型上的虛擬機映像檔，如 OpenStack 物件儲存等。

- [安裝前準備](#安裝前準備)
- [套件安裝與設定](#套件安裝與設定)
- [驗證服務](#驗證服務)

> <font color=red> 提醒! </font>以下操作將一律在 Controller 進行。

### 安裝前準備
在開始安裝前，要預先建立一個資料庫給 Glance 儲存相關資訊，使用以下指令建立資料庫：
```sh
$ mysql -u root -p
```

透過以下命令用來更新現有帳號資料或建立 Glance 資料庫：
```sql
CREATE DATABASE glance;
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost'  IDENTIFIED BY 'GLANCE_DBPASS';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'GLANCE_DBPASS';
```
> 這邊`GLANCE_DBPASS`可以隨需求修改。

完成後離開資料庫，接著要建立 Service 與 API Endpoint，首先導入`admin`環境變數：
```sh
$ . admin-openrc
```

接著透過以下流程來建立 Glance 的使用者、Service 以及 API Endpoint：
```sh
# 建立 Glance user
$ openstack user create --domain default --password GLANCE_PASS --email glance@example.com glance

# 新增 Glance 到 Admin Role
$ openstack role add --project service --user glance admin

# 建立 Glance service
$ openstack service create --name glance  --description "OpenStack Image service" image

# 建立 Glance public endpoints
$ openstack endpoint create --region RegionOne \
image public http://10.0.0.11:9292

# 建立 Glance internal endpoints
$ openstack endpoint create --region RegionOne \
image internal http://10.0.0.11:9292

# 建立 Glance admin endpoints
$ openstack endpoint create --region RegionOne \
image admin http://10.0.0.11:9292
```
> 在 v3 版本中，可以加入`-f <json, shell, table, yaml>`來檢視 keystone 資訊。

### 套件安裝與設定
在開始設定之前，首先要安裝相關套件與 OpenStack 服務套件，可以透過以下指令進行安裝：
```sh
$ sudo apt install -y glance
```

安裝完成後，編輯 `/etc/glance/glance-api.conf` 設定檔，在`[database]`部分修改使用以下方式：
```
[database]
# sqlite_db = /var/lib/glance/glance.sqlite
connection = mysql+pymysql://glance:GLANCE_DBPASS@10.0.0.11/glance
```
> 這邊`GLANCE_DBPASS`可以隨需求修改。

接下來，在`[keystone_authtoken]`部分加入以下內容：
```
[keystone_authtoken]
auth_uri = http://10.0.0.11:5000
auth_url = http://10.0.0.11:35357
memcached_servers = 10.0.0.11:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = glance
password = GLANCE_PASS
```
> 這邊`GLANCE_PASS`可以隨需求修改。

在`[paste_deploy]`部分加入以下內容：
```
[paste_deploy]
flavor = keystone
```

在`[glance_store]`部分加入以下內容：
```
[glance_store]
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/
```
> 其中`filesystem_store_datadir`是當映像檔上傳時檔案放置的目錄。

完成後，要接著編輯`/etc/glance/glance-registry.conf`並完成以下設定，在`[database]`部分修改使用以下方式：
```
[database]
# sqlite_db = /var/lib/glance/glance.sqlite
connection = mysql+pymysql://glance:GLANCE_DBPASS@10.0.0.11/glance
```
> 這邊`GLANCE_DBPASS`可以隨需求修改。

接下來，在`[keystone_authtoken]`部分加入以下內容：
```
[keystone_authtoken]
auth_uri = http://10.0.0.11:5000
auth_url = http://10.0.0.11:35357
memcached_servers = 10.0.0.11:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = glance
password = GLANCE_PASS
```
> 這邊`GLANCE_PASS`可以隨需求修改。

在`[paste_deploy]`部分加入以下內容：
```
[paste_deploy]
flavor = keystone
```

完成以上兩個檔案設定後，即可同步資料庫來建立資料表：
```sh
$ sudo glance-manage db_sync
```

完成後，重新啟動 Glance 的服務：
```sh
sudo service glance-registry restart
sudo service glance-api restart
```

最後刪除預設的 SQLite 資料庫：
```sh
$ sudo rm -f /var/lib/glance/glance.sqlite
```

### 驗證服務
首先接著導入 `admin` 帳號來驗證服務：
```sh
$ . admin-openrc
```

從網路上下載一個測試用映像檔 `cirros-0.3.5-x86_64`：
```sh
$ wget http://download.cirros-cloud.net/0.3.5/cirros-0.3.5-x86_64-disk.img
```

這邊透過指令將映像檔上傳，採用 `QCOW2` 格式，並且設定為公開的映像檔，來提供給雲端租戶們使用：
```sh
$ openstack image create "cirros-0.3.5-x86_64" \
--file cirros-0.3.5-x86_64-disk.img \
--disk-format qcow2 --container-format bare \
--public
```
> * 有關 ` openstack image create` 指令的參數，可以參考 [OpenStack command-line client](http://docs.openstack.org/developer/python-openstackclient/command-list.html) 的指令指南。
* 有關映像檔格式的資訊，可以參考 [Disk and container formats for images](http://docs.openstack.org/image-guide/image-formats.html)。

成功後會看到以下資訊：
```
+------------------+------------------------------------------------------+
| Field            | Value                                                |
+------------------+------------------------------------------------------+
| checksum         | f8ab98ff5e73ebab884d80c9dc9c7290                     |
| container_format | bare                                                 |
| created_at       | 2017-09-10T08:12:50Z                                 |
| disk_format      | qcow2                                                |
| file             | /v2/images/2419a4a9-927e-4f4c-bdd4-e14001b4deee/file |
| id               | 2419a4a9-927e-4f4c-bdd4-e14001b4deee                 |
| min_disk         | 0                                                    |
| min_ram          | 0                                                    |
| name             | cirros-0.3.5-x86_64                                  |
| owner            | 4b2b10e3bd9748ad8f6e000879e2efe6                     |
| protected        | False                                                |
| schema           | /v2/schemas/image                                    |
| size             | 13267968                                             |
| status           | active                                               |
| tags             |                                                      |
| updated_at       | 2017-09-10T08:12:50Z                                 |
| virtual_size     | None                                                 |
| visibility       | public                                               |
+------------------+------------------------------------------------------+
```

完成後，可以透過 Glance client 程式來查看所有映像檔，指令如下：
```sh
$ openstack image list
+--------------------------------------+---------------------+--------+
| ID                                   | Name                | Status |
+--------------------------------------+---------------------+--------+
| 2419a4a9-927e-4f4c-bdd4-e14001b4deee | cirros-0.3.5-x86_64 | active |
+--------------------------------------+---------------------+--------+
```
