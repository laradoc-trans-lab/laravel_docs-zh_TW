# Laravel Sail

- [簡介](#introduction)
- [安裝與設定](#installation)
    - [重新建立 Sail 映像檔](#rebuilding-sail-images)
    - [設定 Shell 別名](#configuring-a-shell-alias)
- [啟動與停止 Sail](#starting-and-stopping-sail)
- [執行指令](#executing-sail-commands)
    - [執行 PHP 指令](#executing-php-commands)
    - [執行 Composer 指令](#executing-composer-commands)
    - [執行 Artisan 指令](#executing-artisan-commands)
    - [執行 Node / NPM 指令](#executing-node-npm-commands)
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
- [預覽電子郵件](#previewing-emails)
- [容器 CLI](#sail-container-cli)
- [PHP 版本](#sail-php-versions)
- [Node 版本](#sail-node-versions)
- [分享您的網站](#sharing-your-site)
- [使用 Xdebug 進行除錯](#debugging-with-xdebug)
  - [Xdebug CLI 用法](#xdebug-cli-usage)
  - [Xdebug 瀏覽器用法](#xdebug-browser-usage)
- [自定義](#sail-customization)

<a name="introduction"></a>
## 簡介

[Laravel Sail](https://github.com/laravel/sail) 是一個輕量級的命令列介面，用於與 Laravel 預設的 Docker 開發環境進行互動。Sail 為使用 PHP、MySQL 和 Redis 構建 Laravel 應用程式提供了一個絕佳的起點，且不需要先前的 Docker 經驗。

本質上，Sail 就是儲存在專案根目錄下的 `compose.yaml` 檔案和 `sail` 腳本。`sail` 腳本提供了一個 CLI，具有便捷的方法可用於與 `compose.yaml` 檔案中定義的 Docker 容器進行互動。

Laravel Sail 支援 macOS、Linux 和 Windows (透過 [WSL2](https://docs.microsoft.com/en-us/windows/wsl/about))。


<a name="installation"></a>
## 安裝與設定

您可以使用 Composer 套件管理工具來安裝 Sail：

```shell
composer require laravel/sail --dev
```

安裝 Sail 後，您可以執行 `sail:install` Artisan 指令。此指令會將 Sail 的 `compose.yaml` 檔案發布到應用程式的根目錄，並修改您的 `.env` 檔案，加入連接 Docker 服務所需的環境變數：

```shell
php artisan sail:install
```

最後，您可以啟動 Sail。若要繼續學習如何使用 Sail，請繼續閱讀本文件的其餘部分：

```shell
./vendor/bin/sail up
```

> [!WARNING]
> 如果您使用的是 Linux 版的 Docker Desktop，您應該透過執行以下指令來使用 `default` Docker 上下文：`docker context use default`。此外，如果您在容器中遇到檔案權限錯誤，您可能需要將 `SUPERVISOR_PHP_USER` 環境變數設定為 `root`。


<a name="adding-additional-services"></a>
#### 新增額外服務

如果您想在現有的 Sail 安裝中新增額外服務，可以執行 `sail:add` Artisan 指令：

```shell
php artisan sail:add
```


<a name="using-devcontainers"></a>
#### 使用 Devcontainers

如果您想在 [Devcontainer](https://code.visualstudio.com/docs/remote/containers) 中開發，可以在 `sail:install` 指令中提供 `--devcontainer` 選項。`--devcontainer` 選項會指示 `sail:install` 指令將預設的 `.devcontainer/devcontainer.json ` 檔案發布到應用程式的根目錄：

```shell
php artisan sail:install --devcontainer
```


<a name="rebuilding-sail-images"></a>
### 重新建立 Sail 映像檔

有時候您可能想要完全重新建立您的 Sail 映像檔，以確保映像檔中的所有套件和軟體都是最新版本。您可以使用 `build` 指令來達成此目的：

```shell
docker compose down -v

sail build --no-cache

sail up
```


<a name="configuring-a-shell-alias"></a>
### 設定 Shell 別名

預設情況下，Sail 指令是使用所有新 Laravel 應用程式中包含的 `vendor/bin/sail` 腳本來啟動的：

```shell
./vendor/bin/sail up
```

然而，為了避免重複輸入 `vendor/bin/sail` 來執行 Sail 指令，您可能希望設定一個 Shell 別名，以便更輕鬆地執行 Sail 的指令：

```shell
alias sail='sh $([ -f sail ] && echo sail || echo vendor/bin/sail)'
```

為了確保此別名始終可用，您可以將其添加到家目錄中的 Shell 設定檔（例如 `~/.zshrc` 或 `~/.bashrc`），然後重新啟動您的 Shell。

一旦設定好 Shell 別名，您只需輸入 `sail` 即可執行 Sail 指令。本文件隨後的範例將假設您已設定此別名：

```shell
sail up
```


<a name="starting-and-stopping-sail"></a>
## 啟動與停止 Sail

Laravel Sail 的 `compose.yaml` 檔案定義了多種協同工作的 Docker 容器，以幫助您構建 Laravel 應用程式。這些容器中的每一個都是 `compose.yaml` 檔案中 `services` 設定的一個項目。`laravel.test` 容器是提供您應用程式服務的主要應用程式容器。

在啟動 Sail 之前，您應該確保本地電腦上沒有其他網頁伺服器或資料庫正在執行。若要啟動應用程式 `compose.yaml` 檔案中定義的所有 Docker 容器，您應該執行 `up` 指令：

```shell
sail up
```

若要將所有 Docker 容器在背景啟動，您可以讓 Sail 以「分離 (detached)」模式啟動：

```shell
sail up -d
```

一旦應用程式的容器啟動，您就可以在網頁瀏覽器中透過 http://localhost 存取該專案。

若要停止所有容器，您只需按下 Control + C 即可停止容器的執行。或者，如果容器是在背景執行，您可以使用 `stop` 指令：

```shell
sail stop
```


<a name="executing-sail-commands"></a>
## 執行指令

使用 Laravel Sail 時，您的應用程式是在 Docker 容器中執行，並與您的本地電腦隔離。然而，Sail 提供了一種便捷的方法，可以用於對您的應用程式執行各種指令，例如任意的 PHP 指令、Artisan 指令、Composer 指令以及 Node / NPM 指令。

**在閱讀 Laravel 文件時，您經常會看到提到 Composer、Artisan 和 Node / NPM 指令但未提及 Sail 的部分。** 那些範例假設這些工具已安裝在您的本地電腦上。如果您使用 Sail 作為本地 Laravel 開發環境，您應該使用 Sail 來執行這些指令：

```shell
# Running Artisan commands locally...
php artisan queue:work

# Running Artisan commands within Laravel Sail...
sail artisan queue:work
```


<a name="executing-php-commands"></a>
### 執行 PHP 指令

可以使用 `php` 指令來執行 PHP 指令。當然，這些指令將使用為您的應用程式所設定的 PHP 版本來執行。若要了解更多關於 Laravel Sail 可用的 PHP 版本，請參閱 [PHP 版本文件](#sail-php-versions)：

```shell
sail php --version

sail php script.php
```


<a name="executing-composer-commands"></a>
### 執行 Composer 指令

可以使用 `composer` 指令來執行 Composer 指令。Laravel Sail 的應用程式容器中包含 Composer 安裝：

```shell
sail composer require laravel/sanctum
```


<a name="executing-artisan-commands"></a>
### 執行 Artisan 指令

可以使用 `artisan` 指令來執行 Laravel Artisan 指令：

```shell
sail artisan queue:work
```


<a name="executing-node-npm-commands"></a>
### 執行 Node / NPM 指令

可以使用 `node` 指令執行 Node 指令，而 `npm` 指令則用於執行 NPM 指令：

```shell
sail node --version

sail npm run dev
```

如果您願意，可以使用 Yarn 代替 NPM：

```shell
sail yarn
```

<a name="interacting-with-sail-databases"></a>
## 與資料庫互動


<a name="mysql"></a>
### MySQL

您可能已經注意到，您的應用程式 `compose.yaml` 檔案中包含一個 MySQL 容器的項目。此容器使用 [Docker volume](https://docs.docker.com/storage/volumes/)，因此即使在停止和重新啟動容器時，儲存在資料庫中的資料也會被保留。

此外，MySQL 容器在第一次啟動時會為您建立兩個資料庫。第一個資料庫的名稱使用 `DB_DATABASE` 環境變數的值，用於本地開發。第二個是專用的測試資料庫，名稱為 `testing`，可確保您的測試不會干擾到開發資料。

一旦啟動容器，您可以將應用程式 `.env` 檔案中的 `DB_HOST` 環境變數設定為 `mysql`，即可連線到應用程式內的 MySQL 執行個體。

若要從本地機器連線到應用程式的 MySQL 資料庫，您可以使用圖形化資料庫管理應用程式，例如 [TablePlus](https://tableplus.com)。預設情況下，MySQL 資料庫可透過 `localhost` 的 3306 埠 (port) 存取，存取憑據對應於 `DB_USERNAME` 和 `DB_PASSWORD` 環境變數的值。或者，您也可以以 `root` 使用者身分連線，同樣使用 `DB_PASSWORD` 環境變數的值作為密碼。


<a name="mongodb"></a>
### MongoDB

如果您在安裝 Sail 時選擇安裝 [MongoDB](https://www.mongodb.com/) 服務，您的應用程式 `compose.yaml` 檔案將包含一個 [MongoDB Atlas Local](https://www.mongodb.com/docs/atlas/cli/current/atlas-cli-local-cloud/) 容器的項目，該容器提供具有 Atlas 功能（如 [Search Indexes](https://www.mongodb.com/docs/atlas/atlas-search/)）的 MongoDB 文件資料庫。此容器使用 [Docker volume](https://docs.docker.com/storage/volumes/)，因此即使在停止和重新啟動容器時，儲存在資料庫中的資料也會被保留。

一旦啟動容器，您可以將應用程式 `.env` 檔案中的 `MONGODB_URI` 環境變數設定為 `mongodb://mongodb:27017`，即可連線到應用程式內的 MongoDB 執行個體。預設情況下會停用認證，但您可以在啟動 `mongodb` 容器之前設定 `MONGODB_USERNAME` 和 `MONGODB_PASSWORD` 環境變數以啟用認證。然後，將憑據添加到連線字串中：

```ini
MONGODB_USERNAME=user
MONGODB_PASSWORD=laravel
MONGODB_URI=mongodb://${MONGODB_USERNAME}:${MONGODB_PASSWORD}@mongodb:27017
```

為了讓 MongoDB 與您的應用程式無縫整合，您可以安裝 [MongoDB 維護的官方套件](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/)。

若要從本地機器連線到應用程式的 MongoDB 資料庫，您可以使用圖形化介面，例如 [Compass](https://www.mongodb.com/products/tools/compass)。預設情況下，MongoDB 資料庫可透過 `localhost` 的 `27017` 埠 (port) 存取。


<a name="redis"></a>
### Redis

您的應用程式 `compose.yaml` 檔案中還包含一個 [Redis](https://redis.io) 容器的項目。此容器使用 [Docker volume](https://docs.docker.com/storage/volumes/)，因此即使在停止和重新啟動容器時，儲存在 Redis 執行個體中的資料也會被保留。一旦啟動容器，您可以將應用程式 `.env` 檔案中的 `REDIS_HOST` 環境變數設定為 `redis`，即可連線到應用程式內的 Redis 執行個體。

若要從本地機器連線到應用程式的 Redis 資料庫，您可以使用圖形化資料庫管理應用程式，例如 [TablePlus](https://tableplus.com)。預設情況下，Redis 資料庫可透過 `localhost` 的 6379 埠 (port) 存取。


<a name="valkey"></a>
### Valkey

如果您在安裝 Sail 時選擇安裝 Valkey 服務，您的應用程式 `compose.yaml` 檔案將包含一個 [Valkey](https://valkey.io/) 的項目。此容器使用 [Docker volume](https://docs.docker.com/storage/volumes/)，因此即使在停止和重新啟動容器時，儲存在 Valkey 執行個體中的資料也會被保留。您可以將應用程式 `.env` 檔案中的 `REDIS_HOST` 環境變數設定為 `valkey` 以連線到此容器。

若要從本地機器連線到應用程式的 Valkey 資料庫，您可以使用圖形化資料庫管理應用程式，例如 [TablePlus](https://tableplus.com)。預設情況下，Valkey 資料庫可透過 `localhost` 的 6379 埠 (port) 存取。


<a name="meilisearch"></a>
### Meilisearch

如果您在安裝 Sail 時選擇安裝 [Meilisearch](https://www.meilisearch.com) 服務，您的應用程式 `compose.yaml` 檔案將包含一個此強大搜尋引擎的項目，該引擎與 [Laravel Scout](/docs/{{version}}/scout) 整合。一旦啟動容器，您可以將 `MEILISEARCH_HOST` 環境變數設定為 `http://meilisearch:7700`，即可連線到應用程式內的 Meilisearch 執行個體。

從您的本地機器，您可以透過在瀏覽器中造訪 `http://localhost:7700` 來存取 Meilisearch 的網頁管理面板。


<a name="typesense"></a>
### Typesense

如果您在安裝 Sail 時選擇安裝 [Typesense](https://typesense.org) 服務，您的應用程式 `compose.yaml` 檔案將包含一個此極速、開源搜尋引擎的項目，它與 [Laravel Scout](/docs/{{version}}/scout#typesense) 原生整合。一旦啟動容器，您可以設定以下環境變數來連線到應用程式內的 Typesense 執行個體：

```ini
TYPESENSE_HOST=typesense
TYPESENSE_PORT=8108
TYPESENSE_PROTOCOL=http
TYPESENSE_API_KEY=xyz
```

從您的本地機器，您可透過 `http://localhost:8108` 存取 Typesense 的 API。


<a name="file-storage"></a>
## 檔案儲存

如果您計劃在生產環境執行應用程式時使用 Amazon S3 來儲存檔案，您可能希望在安裝 Sail 時安裝 [RustFS](https://rustfs.com) 服務。RustFS 提供與 S3 相容的 API，您可以使用 Laravel 的 `s3` 檔案儲存驅動程式在本地進行開發，而無需在生產 S3 環境中建立「測試」儲存儲存桶 (buckets)。如果您在安裝 Sail 時選擇安裝 RustFS，您的應用程式 `compose.yaml` 檔案將會增加一個 RustFS 設定區塊。

預設情況下，您的應用程式 `filesystems` 設定檔中已經包含 `s3` 磁碟的設定。除了使用此磁碟與 Amazon S3 互動外，您也可以透過修改控制其設定的相關環境變數，來使用它與任何與 S3 相容的檔案儲存服務（如 RustFS）互動。例如，在使用 RustFS 時，您的檔案系統環境變數設定應定義如下：

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

Laravel 原生提供了強大的測試支援，您可以使用 Sail 的 `test` 指令來執行應用程式的 [功能測試與單元測試](/docs/{{version}}/testing)。任何 Pest / PHPUnit 接受的 CLI 選項也可以傳遞給 `test` 指令：

```shell
sail test

sail test --group orders
```

Sail 的 `test` 指令等同於執行 `test` Artisan 指令：

```shell
sail artisan test
```

預設情況下，Sail 會建立一個專用的 `testing` 資料庫，以便您的測試不會干擾資料庫的目前狀態。在預設的 Laravel 安裝中，Sail 還會設定您的 `phpunit.xml` 檔案，以便在執行測試時使用此資料庫：

```xml
<env name="DB_DATABASE" value="testing"/>
```


<a name="laravel-dusk"></a>
### Laravel Dusk

[Laravel Dusk](/docs/{{version}}/dusk) 提供了一個強大且易於使用的瀏覽器自動化與測試 API。感謝 Sail，您無需在本地電腦上安裝 Selenium 或其他工具即可執行這些測試。要開始使用，請取消應用程式 `compose.yaml` 檔案中 Selenium 服務的註解：

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

接下來，確保應用程式 `compose.yaml` 檔案中的 `laravel.test` 服務具有 `selenium` 的 `depends_on` 項目：

```yaml
depends_on:
    - mysql
    - redis
    - selenium
```

最後，您可以啟動 Sail 並執行 `dusk` 指令來執行您的 Dusk 測試套件：

```shell
sail dusk
```


<a name="selenium-on-apple-silicon"></a>
#### Selenium on Apple Silicon

如果您的本地電腦使用 Apple Silicon 晶片，您的 `selenium` 服務必須使用 `selenium/standalone-chromium` 映像檔：

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
## 預覽電子郵件

Laravel Sail 的預設 `compose.yaml` 檔案包含一個 [Mailpit](https://github.com/axllent/mailpit) 服務項目。Mailpit 會攔截本地開發期間由應用程式發出的電子郵件，並提供一個方便的網頁介面，讓您可以在瀏覽器中預覽電子郵件訊息。使用 Sail 時，Mailpit 的預設主機為 `mailpit`，並透過連接埠 1025 提供服務：

```ini
MAIL_HOST=mailpit
MAIL_PORT=1025
MAIL_ENCRYPTION=null
```

當 Sail 執行時，您可以透過以下網址存取 Mailpit 網頁介面： http://localhost:8025


<a name="sail-container-cli"></a>
## 容器 CLI

有時您可能希望在應用程式容器中啟動一個 Bash 工作階段。您可以使用 `shell` 指令連接到應用程式容器，讓您可以檢查其檔案與安裝的服務，以及在容器內執行任意的 shell 指令：

```shell
sail shell

sail root-shell
```

要啟動新的 [Laravel Tinker](https://github.com/laravel/tinker) 工作階段，您可以執行 `tinker` 指令：

```shell
sail tinker
```


<a name="sail-php-versions"></a>
## PHP 版本

Sail 目前支援透過 PHP 8.5, 8.4, 8.3, 8.2, 8.1 或 PHP 8.0 提供應用程式服務。Sail 目前使用的預設 PHP 版本是 PHP 8.5。要更改用於提供應用程式服務的 PHP 版本，您應該更新應用程式 `compose.yaml` 檔案中 `laravel.test` 容器的 `build` 定義：

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

此外，您可能希望更新 `image` 名稱以反映應用程式所使用的 PHP 版本。此選項同樣定義在應用程式的 `compose.yaml` 檔案中：

```yaml
image: sail-8.2/app
```

更新應用程式的 `compose.yaml` 檔案後，您應該重新建立容器映像檔：

```shell
sail build --no-cache

sail up
```


<a name="sail-node-versions"></a>
## Node 版本

Sail 預設安裝 Node 22。要更改建立映像檔時安裝的 Node 版本，您可以更新應用程式 `compose.yaml` 檔案中 `laravel.test` 服務的 `build.args` 定義：

```yaml
build:
    args:
        WWWGROUP: '${WWWGROUP}'
        NODE_VERSION: '18'
```

更新應用程式的 `compose.yaml` 檔案後，您應該重新建立容器映像檔：

```shell
sail build --no-cache

sail up
```


<a name="sharing-your-site"></a>
## 分享您的網站

有時您可能需要公開分享您的網站，以便讓同事預覽或測試應用程式的 Webhook 整合。要分享您的網站，您可以使用 `share` 指令。執行此指令後，您將獲得一個隨機的 `laravel-sail.site` URL，可用於存取您的應用程式：

```shell
sail share
```

當您使用 `share` 指令分享網站時，您應該使用應用程式 `bootstrap/app.php` 檔案中的 `trustProxies` 中介層方法來設定應用程式的信任代理。否則，如 `url` 和 `route` 等 URL 產生輔助函數將無法確定在產生 URL 時應使用的正確 HTTP 主機：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->trustProxies(at: '*');
})
```

如果您想為分享的網站選擇子網域，可以在執行 `share` 指令時提供 `subdomain` 選項：

```shell
sail share --subdomain=my-sail-site
```

> [!NOTE]
> `share` 指令由 [Expose](https://github.com/beyondcode/expose) 提供支援，這是一個由 [BeyondCode](https://beyondco.de) 開發的開源隧道服務。

<a name="debugging-with-xdebug"></a>
## 使用 Xdebug 進行除錯

Laravel Sail 的 Docker 設定包含對 [Xdebug](https://xdebug.org/) 的支援，這是一款針對 PHP 的強大且流行的除錯工具。要啟用 Xdebug，請確保您已[發布您的 Sail 設定](#sail-customization)。接著，將以下變數新增至您應用程式的 `.env` 檔案中以設定 Xdebug：

```ini
SAIL_XDEBUG_MODE=develop,debug,coverage
```

接著，確保您發布的 `php.ini` 檔案包含以下設定，以便 Xdebug 在指定模式下啟動：

```ini
[xdebug]
xdebug.mode=${XDEBUG_MODE}
```

在修改 `php.ini` 檔案後，請記得重新建立您的 Docker 映像檔，以使對 `php.ini` 檔案的變更生效：

```shell
sail build --no-cache
```


#### Linux 主機 IP 設定

在內部，`XDEBUG_CONFIG` 環境變數被定義為 `client_host=host.docker.internal`，以便 Xdebug 能在 Mac 和 Windows (WSL2) 上正確設定。如果您的本地機器執行的是 Linux 且使用 Docker 20.10+，則 `host.docker.internal` 是可用的，不需要進行手動設定。

對於 20.10 之前的 Docker 版本，Linux 不支援 `host.docker.internal`，您將需要手動定義主機 IP。若要執行此操作，請在 `compose.yaml` 檔案中定義自訂網路，為您的容器設定靜態 IP：

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

一旦您設定好靜態 IP，請在應用程式的 .env 檔案中定義 SAIL_XDEBUG_CONFIG 變數：

```ini
SAIL_XDEBUG_CONFIG="client_host=172.20.0.2"
```


<a name="xdebug-cli-usage"></a>
### Xdebug CLI 用法

可以使用 `sail debug` 指令在執行 Artisan 指令時啟動除錯工作階段：

```shell
# Run an Artisan command without Xdebug...
sail artisan migrate

# Run an Artisan command with Xdebug...
sail debug migrate
```


<a name="xdebug-browser-usage"></a>
### Xdebug 瀏覽器用法

若要在透過網頁瀏覽器與應用程式互動時對其進行除錯，請參考 [Xdebug 提供的說明](https://xdebug.org/docs/step_debug#web-application) 以從網頁瀏覽器啟動 Xdebug 工作階段。

如果您使用 PhpStorm，請參閱 JetBrains 關於 [零設定除錯](https://www.jetbrains.com/help/phpstorm/zero-configuration-debugging.html) 的文件。

> [!WARNING]
> Laravel Sail 依賴 `artisan serve` 來提供您的應用程式服務。從 Laravel 8.53.0 版本開始，`artisan serve` 指令僅接受 `XDEBUG_CONFIG` 與 `XDEBUG_MODE` 變數。較舊的 Laravel 版本 (8.52.0 及以下) 不支援這些變數，且不會接受除錯連線。


<a name="sail-customization"></a>
## 自定義

由於 Sail 僅僅是 Docker，您可以隨意自定義其幾乎所有內容。若要發布 Sail 自己的 Dockerfiles，您可以執行 `sail:publish` 指令：

```shell
sail artisan sail:publish
```

執行此指令後，Laravel Sail 使用的 Dockerfiles 與其他設定檔將被放置在應用程式根目錄的 `docker` 目錄中。在自定義 Sail 安裝後，您可能希望更改 `compose.yaml` 檔案中應用程式容器的映像檔名稱。完成後，請使用 `build` 指令重新建立應用程式容器。如果您在單台機器上使用 Sail 開發多個 Laravel 應用程式，為應用程式映像檔指定唯一名稱尤為重要：

```shell
sail build --no-cache
```