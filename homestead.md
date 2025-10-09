# Laravel Homestead

- [介紹](#introduction)
- [安裝與設定](#installation-and-setup)
    - [首次步驟](#first-steps)
    - [設定 Homestead](#configuring-homestead)
    - [設定 Nginx 網站](#configuring-nginx-sites)
    - [設定服務](#configuring-services)
    - [啟動 Vagrant Box](#launching-the-vagrant-box)
    - [依專案安裝](#per-project-installation)
    - [安裝選用功能](#installing-optional-features)
    - [別名](#aliases)
- [更新 Homestead](#updating-homestead)
- [日常使用](#daily-usage)
    - [透過 SSH 連線](#connecting-via-ssh)
    - [新增額外網站](#adding-additional-sites)
    - [環境變數](#environment-variables)
    - [連接埠](#ports)
    - [PHP 版本](#php-versions)
    - [連接資料庫](#connecting-to-databases)
    - [資料庫備份](#database-backups)
    - [設定 Cron 排程](#configuring-cron-schedules)
    - [設定 Mailpit](#configuring-mailpit)
    - [設定 Minio](#configuring-minio)
    - [Laravel Dusk](#laravel-dusk)
    - [分享您的環境](#sharing-your-environment)
- [偵錯與效能分析](#debugging-and-profiling)
    - [使用 Xdebug 偵錯 Web 請求](#debugging-web-requests)
    - [偵錯 CLI 應用程式](#debugging-cli-applications)
    - [使用 Blackfire 分析應用程式效能](#profiling-applications-with-blackfire)
- [網路介面](#network-interfaces)
- [擴展 Homestead](#extending-homestead)
- [供應商特定設定](#provider-specific-settings)
    - [VirtualBox](#provider-specific-virtualbox)

<a name="introduction"></a>
## 介紹

> [!WARNING]
> Laravel Homestead 是一個不再積極維護的傳統套件。[Laravel Sail](/docs/{{version}}/sail) 可作為現代替代方案使用。

Laravel 致力於讓整個 PHP 開發體驗愉快，包括您的本機開發環境。[Laravel Homestead](https://github.com/laravel/homestead) 是一個官方預先打包的 Vagrant box，它提供您一個絕佳的開發環境，無需在您的本機電腦上安裝 PHP、Web 伺服器或任何其他伺服器軟體。

[Vagrant](https://www.vagrantup.com) 提供一個簡單優雅的方式來管理和佈建虛擬機器。Vagrant box 是完全可拋棄的。如果出了問題，您可以在幾分鐘內銷毀並重新建立 box！

Homestead 可以在任何 Windows、macOS 或 Linux 系統上執行，並包含 Nginx、PHP、MySQL、PostgreSQL、Redis、Memcached、Node，以及所有您需要用來開發出色 Laravel 應用程式的其他軟體。

> [!WARNING]
> 如果您正在使用 Windows，您可能需要啟用硬體虛擬化 (VT-x)。這通常可以透過您的 BIOS 啟用。如果您正在 UEFI 系統上使用 Hyper-V，您可能還需要停用 Hyper-V 才能存取 VT-x。

<a name="included-software"></a>
### 內含軟體

<style>
    #software-list > ul {
        column-count: 2; -moz-column-count: 2; -webkit-column-count: 2;
        column-gap: 5em; -moz-column-gap: 5em; -webkit-column-gap: 5em;
        line-height: 1.9;
    }
</style>

<div id="software-list" markdown="1">

- Ubuntu 22.04
- Git
- PHP 8.3
- PHP 8.2
- PHP 8.1
- PHP 8.0
- PHP 7.4
- PHP 7.3
- PHP 7.2
- PHP 7.1
- PHP 7.0
- PHP 5.6
- Nginx
- MySQL 8.0
- lmm
- Sqlite3
- PostgreSQL 15
- Composer
- Docker
- Node (包含 Yarn、Bower、Grunt 和 Gulp)
- Redis
- Memcached
- Beanstalkd
- Mailpit
- avahi
- ngrok
- Xdebug
- XHProf / Tideways / XHGui
- wp-cli

</div>

<a name="optional-software"></a>
### 選用軟體

<style>
    #software-list > ul {
        column-count: 2; -moz-column-count: 2; -webkit-column-count: 2;
        column-gap: 5em; -moz-column-gap: 5em; -webkit-column-gap: 5em;
        line-height: 1.9;
    }
</style>

<div id="software-list" markdown="1">

- Apache
- Blackfire
- Cassandra
- Chronograf
- CouchDB
- Crystal & Lucky Framework
- Elasticsearch
- EventStoreDB
- Flyway
- Gearman
- Go
- Grafana
- InfluxDB
- Logstash
- MariaDB
- Meilisearch
- MinIO
- MongoDB
- Neo4j
- Oh My Zsh
- Open Resty
- PM2
- Python
- R
- RabbitMQ
- Rust
- RVM (Ruby 版本管理器)
- Solr
- TimescaleDB
- Trader <small>(PHP 擴充功能)</small>
- Webdriver & Laravel Dusk 實用程式

</div>

<a name="installation-and-setup"></a>
## 安裝與設定


<a name="first-steps"></a>
### 首次步驟

在啟動您的 Homestead 環境之前，您必須安裝 [Vagrant](https://developer.hashicorp.com/vagrant/downloads) 以及以下其中一個支援的供應商：

- [VirtualBox 6.1.x](https://www.virtualbox.org/wiki/Download_Old_Builds_6_1)
- [Parallels](https://www.parallels.com/products/desktop/)

所有這些軟體套件都為所有常見的作業系統提供了易於使用的視覺化安裝程式。

若要使用 Parallels 供應商，您將需要安裝 [Parallels Vagrant plug-in](https://github.com/Parallels/vagrant-parallels)。它是免費的。


<a name="installing-homestead"></a>
#### 安裝 Homestead

您可以透過將 Homestead 儲存庫複製到您的主機機器上來安裝 Homestead。考慮將儲存庫複製到您的「家」目錄中的 `Homestead` 資料夾內，因為 Homestead 虛擬機器將作為您所有 Laravel 應用程式的主機。在整份文件中，我們將此目錄稱為您的「Homestead 目錄」：

```shell
git clone https://github.com/laravel/homestead.git ~/Homestead
```

複製 Laravel Homestead 儲存庫後，您應該切換到 `release` 分支。此分支始終包含最新穩定的 Homestead 版本：

```shell
cd ~/Homestead

git checkout release
```

接下來，從 Homestead 目錄執行 `bash init.sh` 命令以建立 `Homestead.yaml` 設定檔。`Homestead.yaml` 檔案是您將設定所有 Homestead 安裝設定的地方。此檔案將放在 Homestead 目錄中：

```shell
# macOS / Linux...
bash init.sh

# Windows...
init.bat
```


<a name="configuring-homestead"></a>
### 設定 Homestead


<a name="setting-your-provider"></a>
#### 設定您的供應商

您的 `Homestead.yaml` 檔案中的 `provider` 鍵表示應該使用哪個 Vagrant 供應商：`virtualbox` 或 `parallels`：

    provider: virtualbox

> [!WARNING]
> 如果您使用的是 Apple Silicon，則必須使用 Parallels 供應商。


<a name="configuring-shared-folders"></a>
#### 設定共用資料夾

`Homestead.yaml` 檔案的 `folders` 屬性列出了您希望與 Homestead 環境共用的所有資料夾。隨著這些資料夾中的檔案變更，它們將在您的本機機器和 Homestead 虛擬環境之間保持同步。您可以視需要設定任意數量的共用資料夾：

```yaml
folders:
    - map: ~/code/project1
      to: /home/vagrant/project1
```

> [!WARNING]
> Windows 使用者不應使用 `~/` 路徑語法，而應使用其專案的完整路徑，例如 `C:\Users\user\Code\project1`。

您應該始終將個別應用程式映射到其自己的資料夾映射，而不是映射包含所有應用程式的單一大型目錄。當您映射資料夾時，虛擬機器必須追蹤該資料夾中*每個*檔案的所有磁碟 I/O。如果資料夾中檔案數量過多，您可能會遇到效能下降的問題：

```yaml
folders:
    - map: ~/code/project1
      to: /home/vagrant/project1
    - map: ~/code/project2
      to: /home/vagrant/project2
```

> [!WARNING]
> 使用 Homestead 時，切勿掛載 `.` (目前目錄)。這會導致 Vagrant 不將目前資料夾映射到 `/vagrant`，並會破壞選用功能，並在佈建時導致意外結果。

若要啟用 [NFS](https://developer.hashicorp.com/vagrant/docs/synced-folders/nfs)，您可以在資料夾映射中加入 `type` 選項：

```yaml
folders:
    - map: ~/code/project1
      to: /home/vagrant/project1
      type: "nfs"
```

> [!WARNING]
> 在 Windows 上使用 NFS 時，您應該考慮安裝 [vagrant-winnfsd](https://github.com/winnfsd/vagrant-winnfsd) 外掛程式。此外掛程式將維護 Homestead 虛擬機器中檔案和目錄的正確使用者/群組權限。

您也可以透過在 `options` 鍵下依序列出，傳遞 Vagrant [同步資料夾 (Synced Folders)](https://developer.hashicorp.com/vagrant/docs/synced-folders/basic_usage) 支援的任何選項：

```yaml
folders:
    - map: ~/code/project1
      to: /home/vagrant/project1
      type: "rsync"
      options:
          rsync__args: ["--verbose", "--archive", "--delete", "-zz"]
          rsync__exclude: ["node_modules"]
```


<a name="configuring-nginx-sites"></a>
### 設定 Nginx 網站

不熟悉 Nginx 嗎？沒問題。您的 `Homestead.yaml` 檔案的 `sites` 屬性允許您輕鬆地將「網域」映射到 Homestead 環境上的資料夾。`Homestead.yaml` 檔案中包含一個範例網站設定。同樣，您可以根據需要向 Homestead 環境添加任意數量的網站。Homestead 可以作為您正在開發的每個 Laravel 應用程式的便捷虛擬化環境：

```yaml
sites:
    - map: homestead.test
      to: /home/vagrant/project1/public
```

如果您在佈建 Homestead 虛擬機器後變更了 `sites` 屬性，您應該在終端機中執行 `vagrant reload --provision` 命令以更新虛擬機器上的 Nginx 設定。

> [!WARNING]
> Homestead 指令碼設計為盡可能冪等。但是，如果您在佈建時遇到問題，您應該透過執行 `vagrant destroy && vagrant up` 命令來銷毀並重建機器。


<a name="hostname-resolution"></a>
#### 主機名稱解析

Homestead 使用 `mDNS` 發布主機名稱以進行自動主機解析。如果您在 `Homestead.yaml` 檔案中設定 `hostname: homestead`，則該主機將在 `homestead.local` 可用。macOS、iOS 和 Linux 桌面發行版預設包含 `mDNS` 支援。如果您使用的是 Windows，則必須安裝 [Bonjour Print Services for Windows](https://support.apple.com/kb/DL999?viewlocale=en_US&locale=en_US)。

使用自動主機名稱最適用於 [依專案安裝](#per-project-installation) 的 Homestead。如果您在單一 Homestead 實例上託管多個網站，您可以將您的網站「網域」添加到您機器上的 `hosts` 檔案中。`hosts` 檔案會將您 Homestead 網站的請求重新導向到您的 Homestead 虛擬機器。在 macOS 和 Linux 上，此檔案位於 `/etc/hosts`。在 Windows 上，它位於 `C:\Windows\System32\drivers\etc\hosts`。您添加到此檔案中的行將如下所示：

```text
192.168.56.56  homestead.test
```

請確保列出的 IP 位址與您的 `Homestead.yaml` 檔案中設定的 IP 位址一致。一旦您將網域添加到 `hosts` 檔案並啟動 Vagrant box，您就可以透過網路瀏覽器存取該網站：

```shell
http://homestead.test
```


<a name="configuring-services"></a>
### 設定服務

Homestead 預設會啟動多個服務；但是，您可以在佈建期間自訂啟用或禁用哪些服務。例如，您可以透過修改 `Homestead.yaml` 檔案中的 `services` 選項來啟用 PostgreSQL 並禁用 MySQL：

```yaml
services:
    - enabled:
        - "postgresql"
    - disabled:
        - "mysql"
```

指定的服務將根據它們在 `enabled` 和 `disabled` 指令中的順序啟動或停止。


<a name="launching-the-vagrant-box"></a>
### 啟動 Vagrant Box

一旦您編輯 `Homestead.yaml` 到您喜歡的程度，請從您的 Homestead 目錄執行 `vagrant up` 命令。Vagrant 將啟動虛擬機器並自動設定您的共用資料夾和 Nginx 網站。

要銷毀機器，您可以使用 `vagrant destroy` 命令。

<a name="per-project-installation"></a>
### 依專案安裝

您可以為每個專案設定一個 Homestead 實例，而不是全域安裝 Homestead 並在所有專案中共用同一個 Homestead 虛擬機器。如果希望在專案中附帶 `Vagrantfile`，使其他開發人員在複製專案儲存庫後可以立即執行 `vagrant up`，那麼依專案安裝 Homestead 可能會很有幫助。

您可以使用 Composer 套件管理工具將 Homestead 安裝到您的專案中：

```shell
composer require laravel/homestead --dev
```

Homestead 安裝完成後，請執行 Homestead 的 `make` 命令，為您的專案產生 `Vagrantfile` 和 `Homestead.yaml` 檔案。這些檔案將放在您專案的根目錄中。`make` 命令會自動設定 `Homestead.yaml` 檔案中的 `sites` 和 `folders` 指令：

```shell
# macOS / Linux...
php vendor/bin/homestead make

# Windows...
vendor\\bin\\homestead make
```

接著，在您的終端機中執行 `vagrant up` 命令，並在瀏覽器中透過 `http://homestead.test` 存取您的專案。請記住，如果您沒有使用自動 [hostname resolution](#hostname-resolution)，仍需要在 `/etc/hosts` 檔案中為 `homestead.test` 或您選擇的網域新增一個條目。

<a name="installing-optional-features"></a>
### 安裝選用功能

選用軟體是透過 `Homestead.yaml` 檔案中的 `features` 選項來安裝的。大多數功能可以透過布林值啟用或禁用，而有些功能則允許多個設定選項：

```yaml
features:
    - blackfire:
        server_id: "server_id"
        server_token: "server_value"
        client_id: "client_id"
        client_token: "client_value"
    - cassandra: true
    - chronograf: true
    - couchdb: true
    - crystal: true
    - dragonflydb: true
    - elasticsearch:
        version: 7.9.0
    - eventstore: true
        version: 21.2.0
    - flyway: true
    - gearman: true
    - golang: true
    - grafana: true
    - influxdb: true
    - logstash: true
    - mariadb: true
    - meilisearch: true
    - minio: true
    - mongodb: true
    - neo4j: true
    - ohmyzsh: true
    - openresty: true
    - pm2: true
    - python: true
    - r-base: true
    - rabbitmq: true
    - rustc: true
    - rvm: true
    - solr: true
    - timescaledb: true
    - trader: true
    - webdriver: true
```

<a name="elasticsearch"></a>
#### Elasticsearch

您可以指定支援的 Elasticsearch 版本，該版本必須是精確的版本號碼 (major.minor.patch)。預設安裝將建立一個名為 'homestead' 的叢集。您永遠不應將超過作業系統一半的記憶體分配給 Elasticsearch，因此請確保您的 Homestead 虛擬機器至少有兩倍於 Elasticsearch 分配的記憶體。

> [!NOTE]
> 查閱 [Elasticsearch 文件](https://www.elastic.co/guide/en/elasticsearch/reference/current) 以了解如何自訂您的設定。

<a name="mariadb"></a>
#### MariaDB

啟用 MariaDB 將會移除 MySQL 並安裝 MariaDB。MariaDB 通常作為 MySQL 的即時替代品，因此您仍然應該在應用程式的資料庫設定中使用 `mysql` 資料庫驅動程式。

<a name="mongodb"></a>
#### MongoDB

預設的 MongoDB 安裝會將資料庫使用者名稱設定為 `homestead`，對應的密碼設定為 `secret`。

<a name="neo4j"></a>
#### Neo4j

預設的 Neo4j 安裝會將資料庫使用者名稱設定為 `homestead`，對應的密碼設定為 `secret`。若要存取 Neo4j 瀏覽器，請透過您的網頁瀏覽器造訪 `http://homestead.test:7474`。連接埠 `7687` (Bolt)、`7474` (HTTP) 和 `7473` (HTTPS) 已準備好為 Neo4j 客戶端提供請求服務。

<a name="aliases"></a>
### 別名

您可以透過修改 Homestead 目錄中的 `aliases` 檔案，來為您的 Homestead 虛擬機器新增 Bash 別名：

```shell
alias c='clear'
alias ..='cd ..'
```

更新 `aliases` 檔案後，您應該使用 `vagrant reload --provision` 命令重新佈建 Homestead 虛擬機器。這將確保您的新別名在該機器上可用。

<a name="updating-homestead"></a>
## 更新 Homestead

在開始更新 Homestead 之前，您應該先在您的 Homestead 目錄中執行以下指令，確保已移除您目前的虛擬機器：

```shell
vagrant destroy
```

接下來，您需要更新 Homestead 原始碼。如果您是透過複製 (clone) 儲存庫來安裝，請在您最初複製儲存庫的位置執行以下指令：

```shell
git fetch

git pull origin release
```

這些指令會從 GitHub 儲存庫拉取最新的 Homestead 程式碼，抓取最新的標籤 (tags)，然後切換到最新標記的發行版 (release)。您可以在 Homestead 的 [GitHub 發行頁面](https://github.com/laravel/homestead/releases) 找到最新的穩定發行版本。

如果您是透過專案的 `composer.json` 檔案安裝 Homestead，則應該確保您的 `composer.json` 檔案包含 `"laravel/homestead": "^12"` 並更新您的依賴套件：

```shell
composer update
```

接下來，您應該使用 `vagrant box update` 指令來更新 Vagrant box：

```shell
vagrant box update
```

更新 Vagrant box 之後，您應該從 Homestead 目錄執行 `bash init.sh` 指令，以更新 Homestead 的其他設定檔。系統會詢問您是否要覆寫現有的 `Homestead.yaml`、`after.sh` 和 `aliases` 檔案：

```shell
# macOS / Linux...
bash init.sh

# Windows...
init.bat
```

最後，您需要重新生成您的 Homestead 虛擬機器，以利用最新的 Vagrant 安裝：

```shell
vagrant up
```

<a name="daily-usage"></a>
## 日常使用


<a name="connecting-via-ssh"></a>
### 透過 SSH 連線

您可以透過在您的 Homestead 目錄中執行 `vagrant ssh` 終端機指令，來 SSH 進入您的虛擬機器。


<a name="adding-additional-sites"></a>
### 新增額外網站

一旦您的 Homestead 環境已佈建並執行中，您可能想為其他的 Laravel 專案新增額外的 Nginx 網站。您可以在單一 Homestead 環境中執行任意數量的 Laravel 專案。要新增額外網站，請將該網站新增到您的 `Homestead.yaml` 檔案中。

```yaml
sites:
    - map: homestead.test
      to: /home/vagrant/project1/public
    - map: another.test
      to: /home/vagrant/project2/public
```

> [!WARNING]
> 在新增網站之前，您應該確保已經為專案目錄設定了 [資料夾映射](#configuring-shared-folders)。

如果 Vagrant 未自動管理您的「hosts」檔案，您可能還需要將新網站新增到該檔案中。在 macOS 和 Linux 上，此檔案位於 `/etc/hosts`。在 Windows 上，它位於 `C:\Windows\System32\drivers\etc\hosts`。

```text
192.168.56.56  homestead.test
192.168.56.56  another.test
```

網站新增後，從您的 Homestead 目錄執行 `vagrant reload --provision` 終端機指令。


<a name="site-types"></a>
#### 網站類型

Homestead 支援多種網站「類型」，讓您可以輕鬆執行非基於 Laravel 的專案。例如，我們可以透過使用 `statamic` 網站類型輕鬆地將 Statamic 應用程式新增至 Homestead：

```yaml
sites:
    - map: statamic.test
      to: /home/vagrant/my-symfony-project/web
      type: "statamic"
```

可用的網站類型包括：`apache`、`apache-proxy`、`apigility`、`expressive`、`laravel` (預設)、`proxy` (用於 Nginx)、`silverstripe`、`statamic`、`symfony2`、`symfony4` 和 `zf`。


<a name="site-parameters"></a>
#### 網站參數

您可以透過 `params` 網站指令為您的網站新增額外的 Nginx `fastcgi_param` 值：

```yaml
sites:
    - map: homestead.test
      to: /home/vagrant/project1/public
      params:
          - key: FOO
            value: BAR
```


<a name="environment-variables"></a>
### 環境變數

您可以透過將全域環境變數新增到您的 `Homestead.yaml` 檔案來定義它們：

```yaml
variables:
    - key: APP_ENV
      value: local
    - key: FOO
      value: bar
```

更新 `Homestead.yaml` 檔案後，務必執行 `vagrant reload --provision` 指令來重新佈建機器。這將更新所有已安裝 PHP 版本的 PHP-FPM 設定，並更新 `vagrant` 使用者的環境。


<a name="ports"></a>
### 連接埠

依預設，以下連接埠會轉發到您的 Homestead 環境：

<div class="content-list" markdown="1">

- **HTTP:** 8000 &rarr; 轉發至 80
- **HTTPS:** 44300 &rarr; 轉發至 443

</div>


<a name="forwarding-additional-ports"></a>
#### 轉發額外連接埠

如果您願意，可以透過在您的 `Homestead.yaml` 檔案中定義 `ports` 設定項目來轉發額外連接埠到 Vagrant box。更新 `Homestead.yaml` 檔案後，務必執行 `vagrant reload --provision` 指令來重新佈建機器：

```yaml
ports:
    - send: 50000
      to: 5000
    - send: 7777
      to: 777
      protocol: udp
```

以下是您可能希望從主機映射到 Vagrant box 的額外 Homestead 服務連接埠列表：

<div class="content-list" markdown="1">

- **SSH:** 2222 &rarr; 至 22
- **ngrok UI:** 4040 &rarr; 至 4040
- **MySQL:** 33060 &rarr; 至 3306
- **PostgreSQL:** 54320 &rarr; 至 5432
- **MongoDB:** 27017 &rarr; 至 27017
- **Mailpit:** 8025 &rarr; 至 8025
- **Minio:** 9600 &rarr; 至 9600

</div>


<a name="php-versions"></a>
### PHP 版本

Homestead 支援在同一虛擬機器上執行多個 PHP 版本。您可以在 `Homestead.yaml` 檔案中為指定網站指定要使用的 PHP 版本。可用的 PHP 版本包括：「5.6」、「7.0」、「7.1」、「7.2」、「7.3」、「7.4」、「8.0」、「8.1」、「8.2」和「8.3」（預設）：

```yaml
sites:
    - map: homestead.test
      to: /home/vagrant/project1/public
      php: "7.1"
```

[在您的 Homestead 虛擬機器中](#connecting-via-ssh)，您可以透過 CLI 使用任何支援的 PHP 版本：

```shell
php5.6 artisan list
php7.0 artisan list
php7.1 artisan list
php7.2 artisan list
php7.3 artisan list
php7.4 artisan list
php8.0 artisan list
php8.1 artisan list
php8.2 artisan list
php8.3 artisan list
```

您可以透過在您的 Homestead 虛擬機器中發出以下指令，來更改 CLI 使用的預設 PHP 版本：

```shell
php56
php70
php71
php72
php73
php74
php80
php81
php82
php83
```


<a name="connecting-to-databases"></a>
### 連接資料庫

一個 `homestead` 資料庫已為 MySQL 和 PostgreSQL 開箱即用設定。要從主機的資料庫用戶端連接到您的 MySQL 或 PostgreSQL 資料庫，您應該連接到 `127.0.0.1`，連接埠為 `33060` (MySQL) 或 `54320` (PostgreSQL)。這兩個資料庫的使用者名稱和密碼都是 `homestead` / `secret`。

> [!WARNING]
> 僅在從主機連接到資料庫時才應使用這些非標準連接埠。由於 Laravel 在虛擬機器「內部」執行，因此您將在 Laravel 應用程式的 `database` 設定檔中使用預設的 3306 和 5432 連接埠。


<a name="database-backups"></a>
### 資料庫備份

當您的 Homestead 虛擬機器被銷毀時，Homestead 可以自動備份您的資料庫。要使用此功能，您必須使用 Vagrant 2.1.0 或更高版本。或者，如果您使用的是較舊版本的 Vagrant，則必須安裝 `vagrant-triggers` 外掛。要啟用自動資料庫備份，請將以下行新增到您的 `Homestead.yaml` 檔案中：

```yaml
backup: true
```

設定後，當 `vagrant destroy` 指令執行時，Homestead 會將您的資料庫匯出到 `.backup/mysql_backup` 和 `.backup/postgres_backup` 目錄。這些目錄可以在您安裝 Homestead 的資料夾中找到，或者如果您使用 [依專案安裝](#per-project-installation) 方法，則可以在專案的根目錄中找到。


<a name="configuring-cron-schedules"></a>
### 設定 Cron 排程

Laravel 提供了一種便捷的方式來 [排程 cron 工作](/docs/{{version}}/scheduling)，方法是排程單一 `schedule:run` Artisan 指令每分鐘執行一次。`schedule:run` 指令會檢查您的 `routes/console.php` 檔案中定義的工作排程，以決定要執行哪些排程任務。

如果您希望 `schedule:run` 指令為 Homestead 網站執行，您可以在定義網站時將 `schedule` 選項設定為 `true`：

```yaml
sites:
    - map: homestead.test
      to: /home/vagrant/project1/public
      schedule: true
```

該網站的 cron 工作將定義在 Homestead 虛擬機器的 `/etc/cron.d` 目錄中。


<a name="configuring-mailpit"></a>
### 設定 Mailpit

[Mailpit](https://github.com/axllent/mailpit) 讓您可以攔截您的外寄電子郵件並進行檢查，而無需實際將郵件寄送給收件人。若要開始，請更新您應用程式的 `.env` 檔案以使用以下郵件設定：

```ini
MAIL_MAILER=smtp
MAIL_HOST=localhost
MAIL_PORT=1025
MAIL_USERNAME=null
MAIL_PASSWORD=null
MAIL_ENCRYPTION=null
```

一旦 Mailpit 設定完成，您可以透過 `http://localhost:8025` 存取 Mailpit 儀表板。

<a name="configuring-minio"></a>
### 設定 Minio

[Minio](https://github.com/minio/minio) 是一個開源物件儲存伺服器，提供與 Amazon S3 相容的 API。若要安裝 Minio，請在 [features](#installing-optional-features) 區塊中，使用下列設定選項來更新您的 `Homestead.yaml` 檔案：

    minio: true

Minio 預設在 9600 埠上可用。您可以透過瀏覽 `http://localhost:9600` 來存取 Minio 控制面板。預設的存取金鑰為 `homestead`，而預設的密鑰則為 `secretkey`。存取 Minio 時，您應該一律使用 `us-east-1` 區域。

若要使用 Minio，請確保您的 `.env` 檔案包含以下選項：

```ini
AWS_USE_PATH_STYLE_ENDPOINT=true
AWS_ENDPOINT=http://localhost:9600
AWS_ACCESS_KEY_ID=homestead
AWS_SECRET_ACCESS_KEY=secretkey
AWS_DEFAULT_REGION=us-east-1
```

若要配置 Minio 支援的「S3」儲存桶，請在您的 `Homestead.yaml` 檔案中新增一個 `buckets` 指示詞。定義儲存桶後，您應該在終端機中執行 `vagrant reload --provision` 命令：

```yaml
buckets:
    - name: your-bucket
      policy: public
    - name: your-private-bucket
      policy: none
```

支援的 `policy` 值包括：`none`、`download`、`upload` 和 `public`。

<a name="laravel-dusk"></a>
### Laravel Dusk

若要在 Homestead 中執行 [Laravel Dusk](/docs/{{version}}/dusk) 測試，您應該在您的 Homestead 設定中啟用 [webdriver feature](#installing-optional-features)：

```yaml
features:
    - webdriver: true
```

啟用 `webdriver` 功能後，您應該在終端機中執行 `vagrant reload --provision` 命令。

<a name="sharing-your-environment"></a>
### 分享您的環境

有時您可能會希望與同事或客戶分享您目前正在處理的工作。Vagrant 透過 `vagrant share` 命令提供了內建的支援；但是，如果您的 `Homestead.yaml` 檔案中配置了多個網站，這將不起作用。

為了解決這個問題，Homestead 包含了它自己的 `share` 命令。首先，透過 `vagrant ssh` [SSH 進入您的 Homestead 虛擬機器](#connecting-via-ssh) 並執行 `share homestead.test` 命令。此命令將分享您 `Homestead.yaml` 設定檔中的 `homestead.test` 網站。您可以將 `homestead.test` 替換為您配置的其他任何網站：

```shell
share homestead.test
```

執行此命令後，您將看到一個 Ngrok 畫面，其中包含活動日誌和共享網站的公共可訪問 URL。如果您想指定自訂的區域、子網域或其他 Ngrok 執行時選項，您可以將它們添加到您的 `share` 命令中：

```shell
share homestead.test -region=eu -subdomain=laravel
```

如果您需要透過 HTTPS 而非 HTTP 共享內容，可以使用 `sshare` 命令來代替 `share` 命令。

> [!WARNING]
> 請記住，Vagrant 本身不安全，當您執行 `share` 命令時，您正在將虛擬機器暴露在網際網路中。

<a name="debugging-and-profiling"></a>
## 偵錯與效能分析


<a name="debugging-web-requests"></a>
### 使用 Xdebug 偵錯 Web 請求

Homestead 支援使用 [Xdebug](https://xdebug.org) 進行逐步偵錯。例如，您可以在瀏覽器中存取頁面，PHP 將連接到您的 IDE，以便檢查和修改正在執行的程式碼。

預設情況下，Xdebug 已經在運行並準備好接受連線。如果您需要在 CLI 上啟用 Xdebug，請在您的 Homestead 虛擬機器中執行 `sudo phpenmod xdebug` 命令。接著，請依照您的 IDE 指示啟用偵錯。最後，設定您的瀏覽器透過擴充功能或 [書籤小程式](https://www.jetbrains.com/phpstorm/marklets/) 來觸發 Xdebug。

> [!WARNING]
> Xdebug 會導致 PHP 運行速度明顯變慢。若要停用 Xdebug，請在您的 Homestead 虛擬機器中執行 `sudo phpdismod xdebug` 並重新啟動 FPM 服務。


<a name="autostarting-xdebug"></a>
#### 自動啟動 Xdebug

在偵錯向 Web 伺服器發送請求的整合測試時，自動啟動偵錯比修改測試以透過自訂標頭或 Cookie 觸發偵錯更容易。若要強制 Xdebug 自動啟動，請修改您的 Homestead 虛擬機器中的 `/etc/php/7.x/fpm/conf.d/20-xdebug.ini` 檔案，並新增以下設定：

```ini
; If Homestead.yaml contains a different subnet for the IP address, this address may be different...
xdebug.client_host = 192.168.10.1
xdebug.mode = debug
xdebug.start_with_request = yes
```


<a name="debugging-cli-applications"></a>
### 偵錯 CLI 應用程式

若要偵錯 PHP CLI 應用程式，請在您的 Homestead 虛擬機器內使用 `xphp` shell 別名：

```shell
xphp /path/to/script
```


<a name="profiling-applications-with-blackfire"></a>
### 使用 Blackfire 分析應用程式效能

[Blackfire](https://blackfire.io/docs/introduction) 是一個用於分析 Web 請求和 CLI 應用程式的服務。它提供了一個互動式使用者介面，以呼叫圖和時間軸的形式顯示分析資料。它專為開發、預生產和生產環境而設計，對終端使用者沒有任何額外負擔。此外，Blackfire 還對程式碼和 `php.ini` 配置設定提供效能、品質和安全檢查。

[Blackfire Player](https://blackfire.io/docs/player/index) 是一個開源的網站爬蟲、網站測試和網站擷取應用程式，可與 Blackfire 協同工作以編寫效能分析腳本。

若要啟用 Blackfire，請在您的 Homestead 設定檔中使用 "features" 設定：

```yaml
features:
    - blackfire:
        server_id: "server_id"
        server_token: "server_value"
        client_id: "client_id"
        client_token: "client_value"
```

Blackfire 伺服器憑證和用戶端憑證 [需要一個 Blackfire 帳戶](https://blackfire.io/signup)。Blackfire 提供了多種選項來分析應用程式，包括 CLI 工具和瀏覽器擴充功能。請 [查閱 Blackfire 文件以獲取更多詳細資訊](https://blackfire.io/docs/php/integrations/laravel/index)。


<a name="network-interfaces"></a>
## 網路介面

`Homestead.yaml` 檔案的 `networks` 屬性用於設定您的 Homestead 虛擬機器的網路介面。您可以根據需要設定任意數量的介面：

```yaml
networks:
    - type: "private_network"
      ip: "192.168.10.20"
```

若要啟用 [橋接式](https://developer.hashicorp.com/vagrant/docs/networking/public_network) 介面，請為網路設定 `bridge` 選項，並將網路類型更改為 `public_network`：

```yaml
networks:
    - type: "public_network"
      ip: "192.168.10.20"
      bridge: "en1: Wi-Fi (AirPort)"
```

若要啟用 [DHCP](https://developer.hashicorp.com/vagrant/docs/networking/public_network#dhcp)，只需從您的設定中移除 `ip` 選項：

```yaml
networks:
    - type: "public_network"
      bridge: "en1: Wi-Fi (AirPort)"
```

若要更新網路使用的設備，您可以為網路設定新增 `dev` 選項。預設的 `dev` 值是 `eth0`：

```yaml
networks:
    - type: "public_network"
      ip: "192.168.10.20"
      bridge: "en1: Wi-Fi (AirPort)"
      dev: "enp2s0"
```


<a name="extending-homestead"></a>
## 擴展 Homestead

您可以使用 Homestead 目錄根目錄中的 `after.sh` 腳本來擴展 Homestead。在此檔案中，您可以新增任何必要的 shell 命令，以正確設定和自訂您的虛擬機器。

當自訂 Homestead 時，Ubuntu 可能會詢問您是否要保留套件的原始設定，或是以新的設定檔覆蓋它。為了避免這種情況，您在安裝套件時應使用以下命令，以避免覆寫任何先前由 Homestead 寫入的設定：

```shell
sudo apt-get -y \
    -o Dpkg::Options::="--force-confdef" \
    -o Dpkg::Options::="--force-confold" \
    install package-name
```


<a name="user-customizations"></a>
### 使用者自訂

當您與團隊一起使用 Homestead 時，您可能希望調整 Homestead 以更符合您的個人開發風格。為此，您可以在 Homestead 目錄的根目錄（與包含 `Homestead.yaml` 檔案的目錄相同）中建立一個 `user-customizations.sh` 檔案。在此檔案中，您可以進行任何您想要的自訂；但是，`user-customizations.sh` 不應被版本控制。


<a name="provider-specific-settings"></a>
## 供應商特定設定


<a name="provider-specific-virtualbox"></a>
### VirtualBox


<a name="natdnshostresolver"></a>
#### `natdnshostresolver`

預設情況下，Homestead 會將 `natdnshostresolver` 設定為 `on`。這允許 Homestead 使用您的主機作業系統的 DNS 設定。如果您希望覆寫此行為，請在您的 `Homestead.yaml` 檔案中新增以下設定選項：

```yaml
provider: virtualbox
natdnshostresolver: 'off'
```