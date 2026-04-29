# Laravel Sail

- [介紹](#introduction)
- [安裝與設定](#installation)
    - [重新建構 Sail 映像檔](#rebuilding-sail-images)
    - [設定 Shell 別名](#configuring-a-shell-alias)
- [啟動與停止 Sail](#starting-and-stopping-sail)
- [執行命令](#executing-sail-commands)
    - [執行 PHP 命令](#executing-php-commands)
    - [執行 Composer 命令](#executing-composer-commands)
    - [執行 Artisan 命令](#executing-artisan-commands)
    - [執行 Node / NPM 命令](#executing-node-npm-commands)
- [與資料庫互動](#interacting-with-sail-databases)
    - [MySQL](#mysql)
    - [MongoDB](#mongodb)
    - [Redis](#redis)
    - [Valkey](#valkey)
    - [Meilisearch](#meilisearch)
    - [Typesense](#typesense)
- [檔案儲存](#file-storage)
- [執行測試](#running-tests)
    - [Laravel Dusk](#laravel-dusk)
- [預覽郵件](#previewing-emails)
- [容器 CLI](#sail-container-cli)
- [PHP 版本](#sail-php-versions)
- [Node 版本](#sail-node-versions)
- [分享您的網站](#sharing-your-site)
- [使用 Xdebug 偵錯](#debugging-with-xdebug)
  - [Xdebug CLI 用法](#xdebug-cli-usage)
  - [Xdebug 瀏覽器用法](#xdebug-browser-usage)
- [自訂化](#sail-customization)

<a name="introduction"></a>
## 介紹

[Laravel Sail](https://github.com/laravel/sail) 是一個輕量級的命令列介面，用於與 Laravel 預設的 Docker 開發環境進行互動。Sail 提供了一個絕佳的起點，讓您可以使用 PHP、MySQL 與 Redis 來建構 Laravel 應用程式，且不需要具備 Docker 的事先經驗。

Sail 的核心是儲存在專案根目錄下的 `compose.yaml` 檔案與 `sail` 腳本。`sail` 腳本提供了一個 CLI，包含各種便利的方法來與 `compose.yaml` 檔案所定義的 Docker 容器進行互動。

Laravel Sail 支援 macOS、Linux 以及 Windows (透過 [WSL2](https://docs.microsoft.com/en-us/windows/wsl/about))。


<a name="installation"></a>
## 安裝與設定

您可以使用 Composer 套件管理員安裝 Sail：

```shell
composer require laravel/sail --dev
```

在安裝 Sail 之後，您可以執行 `sail:install` Artisan 命令。此命令會將 Sail 的 `compose.yaml` 檔案發布到應用程式的根目錄，並修改您的 `.env` 檔案，加入連線至 Docker 服務所需的環境變數：

```shell
php artisan sail:install
```

最後，您可以啟動 Sail。若要繼續學習如何使用 Sail，請閱讀本文件的其餘部分：

```shell
./vendor/bin/sail up
```

> [!WARNING]
> 如果您正在 Linux 上使用 Docker Desktop，應執行以下命令來使用 `default` Docker context：`docker context use default`。此外，如果您在容器內遇到檔案權限錯誤，您可能需要將 `SUPERVISOR_PHP_USER` 環境變數設定為 `root`。


<a name="adding-additional-services"></a>
#### 新增額外服務

如果您想為現有的 Sail 安裝新增額外服務，可以執行 `sail:add` Artisan 命令：

```shell
php artisan sail:add
```


<a name="using-devcontainers"></a>
#### 使用 Devcontainers

如果您想在 [Devcontainer](https://code.visualstudio.com/docs/remote/containers) 中進行開發，可以在執行 `sail:install` 命令時提供 `--devcontainer` 選項。`--devcontainer` 選項會指示 `sail:install` 命令將預設的 `.devcontainer/devcontainer.json ` 檔案發布到應用程式的根目錄：

```shell
php artisan sail:install --devcontainer
```


<a name="rebuilding-sail-images"></a>
### 重新建構 Sail 映像檔

有時您可能想要完全重新建構 Sail 映像檔，以確保映像檔中所有的套件與軟體都是最新的。您可以使用 `build` 命令來達成此目的：

```shell
docker compose down -v

sail build --no-cache

sail up
```


<a name="configuring-a-shell-alias"></a>
### 設定 Shell 別名

預設情況下，Sail 命令是透過所有新 Laravel 應用程式中包含的 `vendor/bin/sail` 腳本來呼叫的：

```shell
./vendor/bin/sail up
```

然而，為了避免重複輸入 `vendor/bin/sail` 來執行 Sail 命令，您可能希望設定一個 Shell 別名，讓您可以更輕鬆地執行 Sail 的命令：

```shell
alias sail='sh $([ -f sail ] && echo sail || echo vendor/bin/sail)'
```

為了確保此設定始終有效，您可以將其加入到家目錄中的 Shell 設定檔，例如 `~/.zshrc` 或 `~/.bashrc`，然後重啟您的 Shell。

設定好 Shell 別名後，您只需輸入 `sail` 即可執行 Sail 命令。本文件其餘部分的範例都將假設您已設定此別名：

```shell
sail up
```


<a name="starting-and-stopping-sail"></a>
## 啟動與停止 Sail

Laravel Sail 的 `compose.yaml` 檔案定義了各種 Docker 容器，這些容器協同工作以幫助您建構 Laravel 應用程式。這些容器中的每一個都是 `compose.yaml` 檔案中 `services` 設定裡的一個項目。`laravel.test` 容器是為您的應用程式提供服務的主要應用程式容器。

在啟動 Sail 之前，應確保您的本機電腦上沒有其他網頁伺服器或資料庫正在執行。若要啟動應用程式 `compose.yaml` 檔案中定義的所有 Docker 容器，應執行 `up` 命令：

```shell
sail up
```

若要在背景啟動所有 Docker 容器，可以以「分離 (Detached)」模式啟動 Sail：

```shell
sail up -d
```

應用程式容器啟動後，您可以在網頁瀏覽器中透過以下網址存取專案：http://localhost。

若要停止所有容器，只需按 Control + C 即可停止容器的執行。或者，如果容器是在背景執行，您可以使用 `stop` 命令：

```shell
sail stop
```


<a name="executing-sail-commands"></a>
## 執行命令

使用 Laravel Sail 時，您的應用程式是在 Docker 容器中執行，並且與您的本機電腦隔離。不過，Sail 提供了一種便利的方式來對您的應用程式執行各種命令，例如任意的 PHP 命令、Artisan 命令、Composer 命令以及 Node / NPM 命令。

**閱讀 Laravel 文件時，您經常會看到指向 Composer、Artisan 和 Node / NPM 命令的說明，但這些說明並未提及 Sail。** 那些範例假設這些工具已安裝在您的本機電腦上。如果您在本地 Laravel 開發環境中使用 Sail，則應使用 Sail 來執行這些命令：

```shell
# Running Artisan commands locally...
php artisan queue:work

# Running Artisan commands within Laravel Sail...
sail artisan queue:work
```


<a name="executing-php-commands"></a>
### 執行 PHP 命令

PHP 命令可以使用 `php` 命令來執行。當然，這些命令將使用為您的應用程式設定的 PHP 版本執行。若要瞭解更多關於 Laravel Sail 可用 PHP 版本的資訊，請參閱 [PHP 版本文件](#sail-php-versions)：

```shell
sail php --version

sail php script.php
```


<a name="executing-composer-commands"></a>
### 執行 Composer 命令

Composer 命令可以使用 `composer` 命令來執行。Laravel Sail 的應用程式容器包含了一個 Composer 安裝：

```shell
sail composer require laravel/sanctum
```


<a name="executing-artisan-commands"></a>
### 執行 Artisan 命令

Laravel Artisan 命令可以使用 `artisan` 命令來執行：

```shell
sail artisan queue:work
```


<a name="executing-node-npm-commands"></a>
### 執行 Node / NPM 命令

Node 命令可以使用 `node` 命令執行，而 NPM 命令可以使用 `npm` 命令執行：

```shell
sail node --version

sail npm run dev
```

如果您願意，也可以使用 Yarn 代替 NPM：

```shell
sail yarn
```

<a name="interacting-with-sail-databases"></a>
## 與資料庫互動

<a name="mysql"></a>
### MySQL

如同您可能已經注意到的，您應用程式的 `compose.yaml` 檔案包含了一個 MySQL 容器的項目。此容器使用 [Docker 磁碟區 (Docker volume)](https://docs.docker.com/storage/volumes/)，因此即使停止並重新啟動容器，儲存在資料庫中的資料也會被保留。

此外，在 MySQL 容器第一次啟動時，它會為您建立兩座資料庫。第一座資料庫的名稱是使用 `DB_DATABASE` 環境變數的值，用於您的本地開發。第二座是專用的測試資料庫，名為 `testing`，這能確保您的測試不會干擾到開發環境的資料。

啟動容器後，您可以透過將應用程式 `.env` 檔案中的 `DB_HOST` 環境變數設定為 `mysql`，以此連接到應用程式內的 MySQL 實例。

若要從您的本地機器連接到應用程式的 MySQL 資料庫，您可以使用圖形化資料庫管理應用程式，例如 [TablePlus](https://tableplus.com)。預設情況下，MySQL 資料庫可透過 `localhost` 的 3306 埠存取，存取憑證對應於 `DB_USERNAME` 和 `DB_PASSWORD` 環境變數的值。或者，您也可以使用 `root` 使用者進行連接，其密碼同樣使用 `DB_PASSWORD` 環境變數的值。

<a name="mongodb"></a>
### MongoDB

如果您在安裝 Sail 時選擇安裝 [MongoDB](https://www.mongodb.com/) 服務，您應用程式的 `compose.yaml` 檔案會包含一個 [MongoDB Atlas Local](https://www.mongodb.com/docs/atlas/cli/current/atlas-cli-local-cloud/) 容器項目，它提供了具備 Atlas 功能（如[搜尋索引](https://www.mongodb.com/docs/atlas/atlas-search/)）的 MongoDB 文件資料庫。此容器使用 [Docker 磁碟區 (Docker volume)](https://docs.docker.com/storage/volumes/)，因此即使停止並重新啟動容器，儲存在資料庫中的資料也會被保留。

啟動容器後，您可以透過將應用程式 `.env` 檔案中的 `MONGODB_URI` 環境變數設定為 `mongodb://mongodb:27017`，以此連接到應用程式內的 MongoDB 實例。驗證功能預設是停用的，但您可以在啟動 `mongodb` 容器前，透過設定 `MONGODB_USERNAME` 和 `MONGODB_PASSWORD` 環境變數來啟用驗證。然後，將憑證加入連線字串中：

```ini
MONGODB_USERNAME=user
MONGODB_PASSWORD=laravel
MONGODB_URI=mongodb://${MONGODB_USERNAME}:${MONGODB_PASSWORD}@mongodb:27017
```

為了讓 MongoDB 與您的應用程式無縫整合，您可以安裝由 [MongoDB 官方維護的套件](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/)。

若要從您的本地機器連接到應用程式的 MongoDB 資料庫，您可以使用圖形化介面，例如 [Compass](https://www.mongodb.com/products/tools/compass)。預設情況下，MongoDB 資料庫可透過 `localhost` 的 `27017` 埠存取。

<a name="redis"></a>
### Redis

您應用程式的 `compose.yaml` 檔案也包含了一個 [Redis](https://redis.io) 容器項目。此容器使用 [Docker 磁碟區 (Docker volume)](https://docs.docker.com/storage/volumes/)，因此即使停止並重新啟動容器，儲存在 Redis 實例中的資料也會被保留。啟動容器後，您可以透過將應用程式 `.env` 檔案中的 `REDIS_HOST` 環境變數設定為 `redis`，以此連接到應用程式內的 Redis 實例。

若要從您的本地機器連接到應用程式的 Redis 資料庫，您可以使用圖形化資料庫管理應用程式，例如 [TablePlus](https://tableplus.com)。預設情況下，Redis 資料庫可透過 `localhost` 的 6379 埠存取。

<a name="valkey"></a>
### Valkey

如果您在安裝 Sail 時選擇安裝 Valkey 服務，您應用程式的 `compose.yaml` 檔案將包含一個 [Valkey](https://valkey.io/) 項目。此容器使用 [Docker 磁碟區 (Docker volume)](https://docs.docker.com/storage/volumes/)，因此即使停止並重新啟動容器，儲存在 Valkey 實例中的資料也會被保留。您可以透過將應用程式 `.env` 檔案中的 `REDIS_HOST` 環境變數設定為 `valkey`，以此在應用程式中連接到此容器。

若要從您的本地機器連接到應用程式的 Valkey 資料庫，您可以使用圖形化資料庫管理應用程式，例如 [TablePlus](https://tableplus.com)。預設情況下，Valkey 資料庫可透過 `localhost` 的 6379 埠存取。

<a name="meilisearch"></a>
### Meilisearch

如果您在安裝 Sail 時選擇安裝 [Meilisearch](https://www.meilisearch.com) 服務，您應用程式的 `compose.yaml` 檔案將包含一個強大搜尋引擎的項目，該引擎已與 [Laravel Scout](/docs/{{version}}/scout) 整合。啟動容器後，您可以透過將 `MEILISEARCH_HOST` 環境變數設定為 `http://meilisearch:7700`，以此連接到應用程式內的 Meilisearch 實例。

您可以從本地機器透過瀏覽器瀏覽 `http://localhost:7700` 來存取 Meilisearch 的網頁版管理介面。

<a name="typesense"></a>
### Typesense

如果您在安裝 Sail 時選擇安裝 [Typesense](https://typesense.org) 服務，您應用程式的 `compose.yaml` 檔案將包含一個極速開源搜尋引擎的項目，該引擎已原生整合於 [Laravel Scout](/docs/{{version}}/scout#typesense)。啟動容器後，您可以透過設定以下環境變數來連接到應用程式內的 Typesense 實例：

```ini
TYPESENSE_HOST=typesense
TYPESENSE_PORT=8108
TYPESENSE_PROTOCOL=http
TYPESENSE_API_KEY=xyz
```

您可以從本地機器透過 `http://localhost:8108` 存取 Typesense 的 API。

<a name="file-storage"></a>
## 檔案儲存

如果您打算在應用程式的正式環境中使用 Amazon S3 來儲存檔案，您可能希望在安裝 Sail 時安裝 [RustFS](https://rustfs.com) 服務。RustFS 提供了與 S3 相容的 API，讓您可以使用 Laravel 的 `s3` 檔案儲存驅動進行本地開發，而不需要在正式的 S3 環境中建立「測試用」儲存貯體。如果您在安裝 Sail 時選擇安裝 RustFS，一個 RustFS 設定區塊將會被加入到應用程式的 `compose.yaml` 檔案中。

預設情況下，您應用程式的 `filesystems` 設定檔已經包含了一個 `s3` 磁碟的設定。除了使用此磁碟與 Amazon S3 互動外，您還可以透過修改控制其設定的相關環境變數，將其用於與任何 S3 相容的檔案儲存服務（如 RustFS）互動。例如，在使用 RustFS 時，您的檔案系統環境變數設定應定義如下：

```ini
FILESYSTEM_DISK=s3
AWS_ACCESS_KEY_ID=sail
AWS_SECRET_ACCESS_KEY=password
AWS_DEFAULT_REGION=us-east-1
AWS_BUCKET=local
AWS_ENDPOINT=http://rustfs:9000
AWS_USE_PATH_STYLE_ENDPOINT=true
```

<a name="running-tests"></a>
## 執行測試

Laravel 內建提供了優異的測試支援，您可以透過 Sail 的 `test` 命令來執行應用程式的 [功能與單元測試 (feature and unit tests)](/docs/{{version}}/testing)。任何 Pest / PHPUnit 支援的 CLI 選項也都可以傳遞給 `test` 命令：

```shell
sail test

sail test --group orders
```

Sail 的 `test` 命令等同於執行 `test` Artisan 命令：

```shell
sail artisan test
```

預設情況下，Sail 會建立一個專用的 `testing` 資料庫，以免測試干擾資料庫目前的狀態。在預設的 Laravel 安裝中，Sail 也會設定您的 `phpunit.xml` 檔案，使其在執行測試時使用該資料庫：

```xml
<env name="DB_DATABASE" value="testing"/>
```


<a name="laravel-dusk"></a>
### Laravel Dusk

[Laravel Dusk](/docs/{{version}}/dusk) 提供了一個富有表現力且易於使用的瀏覽器自動化與測試 API。感謝 Sail，您無需在本地電腦安裝 Selenium 或其他工具即可執行這些測試。要開始使用，請取消註解應用程式 `compose.yaml` 檔案中的 Selenium 服務：

```yaml
selenium:
    image: 'selenium/standalone-chrome'
    extra_hosts:
      - 'host.docker.internal:host-gateway'
    volumes:
        - '/dev/shm:/dev/shm'
    networks:
        - sail
```

接著，請確保應用程式 `compose.yaml` 檔案中的 `laravel.test` 服務在 `depends_on` 項目中包含 `selenium`：

```yaml
depends_on:
    - mysql
    - redis
    - selenium
```

最後，您可以透過啟動 Sail 並執行 `dusk` 命令來執行您的 Dusk 測試套件：

```shell
sail dusk
```


<a name="selenium-on-apple-silicon"></a>
#### Selenium 於 Apple Silicon

如果您的本地機器使用的是 Apple Silicon 晶片，您的 `selenium` 服務必須使用 `selenium/standalone-chromium` 映像檔：

```yaml
selenium:
    image: 'selenium/standalone-chromium'
    extra_hosts:
        - 'host.docker.internal:host-gateway'
    volumes:
        - '/dev/shm:/dev/shm'
    networks:
        - sail
```


<a name="previewing-emails"></a>
## 預覽郵件

Laravel Sail 預設的 `compose.yaml` 檔案包含了一個 [Mailpit](https://github.com/axllent/mailpit) 的服務項目。Mailpit 會攔截應用程式在本地開發期間發送的電子郵件，並提供一個便利的網頁介面，讓您可以在瀏覽器中預覽郵件內容。使用 Sail 時，Mailpit 的預設主機名稱為 `mailpit`，並透過連接埠 1025 提供服務：

```ini
MAIL_HOST=mailpit
MAIL_PORT=1025
MAIL_ENCRYPTION=null
```

當 Sail 啟動時，您可以透過此網址存取 Mailpit 網頁介面：http://localhost:8025


<a name="sail-container-cli"></a>
## 容器 CLI

有時您可能想在應用程式容器中開啟一個 Bash 工作階段。您可以使用 `shell` 命令連接到應用程式容器，以便檢查其檔案、已安裝的服務，並在容器內執行任意的 Shell 命令：

```shell
sail shell

sail root-shell
```

要啟動一個新的 [Laravel Tinker](https://github.com/laravel/tinker) 工作階段，您可以執行 `tinker` 命令：

```shell
sail tinker
```


<a name="sail-php-versions"></a>
## PHP 版本

Sail 目前支援透過 PHP 8.5, 8.4, 8.3, 8.2, 8.1, 或 PHP 8.0 來運行您的應用程式。Sail 目前預設使用的 PHP 版本為 PHP 8.5。要更改運行應用程式的 PHP 版本，您應該更新應用程式 `compose.yaml` 檔案中 `laravel.test` 容器的 `build` 定義：

```yaml
# PHP 8.5
context: ./vendor/laravel/sail/runtimes/8.5

# PHP 8.4
context: ./vendor/laravel/sail/runtimes/8.4

# PHP 8.3
context: ./vendor/laravel/sail/runtimes/8.3

# PHP 8.2
context: ./vendor/laravel/sail/runtimes/8.2

# PHP 8.1
context: ./vendor/laravel/sail/runtimes/8.1

# PHP 8.0
context: ./vendor/laravel/sail/runtimes/8.0
```

此外，您可能也想更新 `image` 名稱以反映應用程式所使用的 PHP 版本。此選項同樣定義在應用程式的 `compose.yaml` 檔案中：

```yaml
image: sail-8.2/app
```

更新完應用程式的 `compose.yaml` 檔案後，您應該重新建構您的容器映像檔：

```shell
sail build --no-cache

sail up
```


<a name="sail-node-versions"></a>
## Node 版本

Sail 預設安裝 Node 24。要更改建構映像檔時安裝的 Node 版本，您可以更新應用程式 `compose.yaml` 檔案中 `laravel.test` 服務的 `build.args` 定義：

```yaml
build:
    args:
        WWWGROUP: '${WWWGROUP}'
        NODE_VERSION: '18'
```

更新完應用程式的 `compose.yaml` 檔案後，您應該重新建構您的容器映像檔：

```shell
sail build --no-cache

sail up
```


<a name="sharing-your-site"></a>
## 分享您的網站

有時您可能需要公開分享您的網站，以便讓同事預覽或測試與應用程式整合的 Webhook。要分享您的網站，您可以使用 `share` 命令。執行此命令後，系統會核發一個隨機的 `laravel-sail.site` 網址給您，您可以使用該網址存取您的應用程式：

```shell
sail share
```

透過 `share` 命令分享網站時，您應該在應用程式 `bootstrap/app.php` 檔案中，使用 `trustProxies` 中介層方法來設定應用程式的信任代理伺服器。否則，諸如 `url` 和 `route` 之類的網址產生輔助函式將無法判斷產生網址時應使用的正確 HTTP 主機：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->trustProxies(at: '*');
})
```

如果您想為分享的網站自訂子網域，可以在執行 `share` 命令時提供 `subdomain` 選項：

```shell
sail share --subdomain=my-sail-site
```

> [!NOTE]
> `share` 命令是由 [Expose](https://github.com/beyondcode/expose) 提供技術支援，這是一個由 [BeyondCode](https://beyondco.de) 開發的開源隧道服務 (tunneling service)。

<a name="debugging-with-xdebug"></a>
## 使用 Xdebug 偵錯

Laravel Sail 的 Docker 設定包含對 [Xdebug](https://xdebug.org/) 的支援，這是一個熱門且功能強大的 PHP 偵錯工具。若要啟用 Xdebug，請確保您已經[發布了 Sail 的設定](#sail-customization)。接著，將以下變數新增到應用程式的 `.env` 檔案中以設定 Xdebug：

```ini
SAIL_XDEBUG_MODE=develop,debug,coverage
```

接下來，確保您發布的 `php.ini` 檔案包含以下設定，以便在指定的模式中啟動 Xdebug：

```ini
[xdebug]
xdebug.mode=${XDEBUG_MODE}
```

修改 `php.ini` 檔案後，請記得重新建構您的 Docker 映像檔，以使 `php.ini` 檔案的變更生效：

```shell
sail build --no-cache
```

#### Linux 主機 IP 設定

在內部，`XDEBUG_CONFIG` 環境變數被定義為 `client_host=host.docker.internal`，以便為 Mac 和 Windows (WSL2) 正確設定 Xdebug。如果您的本機電腦執行的是 Linux，且您使用的是 Docker 20.10+，則 `host.docker.internal` 是可用的，且不需要手動設定。

對於早於 20.10 的 Docker 版本，Linux 不支援 `host.docker.internal`，您需要手動定義主機 IP。為此，請透過在 `compose.yaml` 檔案中定義自訂網路來為您的容器設定靜態 IP：

```yaml
networks:
  custom_network:
    ipam:
      config:
        - subnet: 172.20.0.0/16

services:
  laravel.test:
    networks:
      custom_network:
        ipv4_address: 172.20.0.2
```

設定靜態 IP 後，請在應用程式的 .env 檔案中定義 SAIL_XDEBUG_CONFIG 變數：

```ini
SAIL_XDEBUG_CONFIG="client_host=172.20.0.2"
```

<a name="xdebug-cli-usage"></a>
### Xdebug CLI 用法

在執行 Artisan 命令時，可以使用 `sail debug` 命令來啟動偵錯工作階段：

```shell
# Run an Artisan command without Xdebug...
sail artisan migrate

# Run an Artisan command with Xdebug...
sail debug migrate
```

<a name="xdebug-browser-usage"></a>
### Xdebug 瀏覽器用法

若要在透過網頁瀏覽器與應用程式互動時進行偵錯，請按照 [Xdebug 提供的說明](https://xdebug.org/docs/step_debug#web-application)從網頁瀏覽器啟動 Xdebug 工作階段。

如果您使用的是 PhpStorm，請參閱 JetBrains 關於[零設定偵錯](https://www.jetbrains.com/help/phpstorm/zero-configuration-debugging.html)的說明文件。

> [!WARNING]
> Laravel Sail 依賴 `artisan serve` 來提供您的應用程式服務。從 Laravel 版本 8.53.0 開始，`artisan serve` 命令才支援 `XDEBUG_CONFIG` 和 `XDEBUG_MODE` 變數。較舊版本的 Laravel (8.52.0 及以下版本) 不支援這些變數，且不會接受偵錯連線。

<a name="sail-customization"></a>
## 自訂化

由於 Sail 就是 Docker，您可以隨意自訂幾乎所有內容。若要發布 Sail 自己的 Dockerfiles，您可以執行 `sail:publish` 命令：

```shell
sail artisan sail:publish
```

執行此命令後，Laravel Sail 使用的 Dockerfiles 和其他設定檔案將被放置在應用程式根目錄下的 `docker` 目錄中。自訂 Sail 安裝後，您可能希望更改應用程式 `compose.yaml` 檔案中應用程式容器的映像檔名稱。完成後，使用 `build` 命令重新建構應用程式的容器。如果您使用 Sail 在單台機器上開發多個 Laravel 應用程式，則為應用程式映像檔指派一個唯一的名稱尤為重要：

```shell
sail build --no-cache
```