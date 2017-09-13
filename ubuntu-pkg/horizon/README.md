# Horizon 安裝與設定
OpenStack 的 Horizon 提供了一套 Web-based 介面的管理儀表板，其提供雲端管理人員夠透過該儀表板進行管理 OpenStack 中各種服務與資源。

- [系統基本需求](#系統基本需求)
- [套件安裝與設定](#套件安裝與設定)
- [驗證服務](#驗證服務)

### 系統基本需求
在安裝 Horizon 前，叢集需要滿足以下幾點：
* OpenStack Nova Compute 已被安裝。並啟動了 Keystone 身份認證服務。
> P.S. 需注意`身份認證服務`與`Nova Compute`的 endpoints

* `身份認證服務`的主機使用者可以利用 sudo 權限。因為 Apache 不提供從 Root 使用者來取得相關內容，管理者若要執行 Horizon  的話，必須讓 Keystone 擁有`sudo`使用權。
* Python 必須在 2.7 以上版本。Python 的版本能夠支援 Django。該 Python 的版本應該可以執行於任何作業系統，諸如：Linux、Mac OS X 等。

### 套件安裝與設定
假設基本的 OpenStack 環境都已經完成後，就可以到`Controller`節點上安裝套件：
```sh
$ sudo apt install openstack-dashboard
```
> 預設下 OpenStack Dashboard 會是 Ubuntu theme，可以修改`/etc/openstack-dashboard/local_settings.py`設定檔來使用官方預設樣板：
```
AVAILABLE_THEMES = [
    ('default', 'Default', 'themes/default'),
    ('material', 'Material', 'themes/material'),
]
```

完成安裝後，編輯`/etc/openstack-dashboard/local_settings.py`設定檔，部分加入以下內容：
```
OPENSTACK_HOST = "10.0.0.11"
ALLOWED_HOSTS = '*'

CACHES = {
   'default': {
       'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
       'LOCATION': '10.0.0.11:11211',
   }
}

SESSION_ENGINE = 'django.contrib.sessions.backends.cache'

OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST
OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"
OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True
OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = 'default'

OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "image": 2,
    "volume": 2,
}
```
> 其中`TIME_ZONE`可以依需與地區更改，可參考 [list of time zones](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones) 來正確設定。

完成後，重新載入與啟動 Apache2：
```sh
sudo service apache2 reload
sudo service apache2 restart
```

### 驗證服務
這個部分將描述如何進行儀表板的驗證操作，依照以下兩個簡單步驟：
* 開啟 Web 瀏覽器進入儀表板: [Horizon Dashboard](http://10.0.0.11/horizon)。
* 使用 admin 或 demo 的使用者登入。

![horizon](images/horizon.png)
