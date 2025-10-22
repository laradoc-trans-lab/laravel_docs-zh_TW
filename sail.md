# Laravel Sail

- [簡介](#introduction)
- [安裝與設定](#installation)
    - [將 Sail 安裝到既有應用程式](#installing-sail-into-existing-applications)
    - [重建 Sail 映像檔](#rebuilding-sail-images)
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
- [使用 Xdebug 進行偵錯](#debugging-with-xdebug)
  - [Xdebug CLI 使用方式](#xdebug-cli-usage)
  - [Xdebug 瀏覽器使用方式](#xdebug-browser-usage)
- [客製化](#sail-customization)

<a name="introduction"></a>
## 簡介

[Laravel Sail](https://github.com/laravel/sail) 是一個輕量級命令列介面，用於與 Laravel 預設的 Docker 開發環境互動。Sail 為使用 PHP、MySQL 和 Redis 建立 Laravel 應用程式提供了絕佳的起點，且無須事先具備 Docker 經驗。

核心而言，Sail 是您專案根目錄中的 `compose.yaml` 檔案和 `sail` 腳本。`sail` 腳本提供了一個 CLI，其中包含用於與 `compose.yaml` 檔案定義的 Docker 容器互動的便利方法。

Laravel Sail 支援 macOS、Linux 和 Windows (透過 [WSL2](https://docs.microsoft.com/en-us/windows/wsl/about))。

<a name="installation"></a>
## 安裝與設定

Laravel Sail 會自動安裝到所有新的 Laravel 應用程式中，因此您可以立即開始使用。

<a name="installing-sail-into-existing-applications"></a>
### 將 Sail 安裝到既有應用程式

如果您有興趣將 Sail 與現有的 Laravel 應用程式一起使用，您可以直接使用 Composer 套件管理器安裝 Sail。當然，這些步驟假設您現有的本機開發環境允許您安裝 Composer 依賴套件：

```shell
composer require laravel/sail --dev
```

Sail 安裝完成後，您可以執行 `sail:install` Artisan 指令。此指令會將 Sail 的 `compose.yaml` 檔案發佈到應用程式的根目錄，並修改您的 `.env` 檔案以包含連接 Docker 服務所需的環境變數：

```shell
php artisan sail:install
```

最後，您可以啟動 Sail。若要繼續學習如何使用 Sail，請繼續閱讀本文件的其餘部分：

```shell
./vendor/bin/sail up
```

> [!WARNING]
> 如果您正在使用適用於 Linux 的 Docker Desktop，您應該執行以下指令來使用 `default` Docker context：`docker context use default`。

<a name="adding-additional-services"></a>
#### 新增額外服務

如果您想為現有的 Sail 安裝新增一項額外服務，您可以執行 `sail:add` Artisan 指令：

```shell
php artisan sail:add
```

<a name="using-devcontainers"></a>
#### 使用 Devcontainers

如果您想在 [Devcontainer](https://code.visualstudio.com/docs/remote/containers) 中進行開發，您可以為 `sail:install` 指令提供 `--devcontainer` 選項。`--devcontainer` 選項將指示 `sail:install` 指令將預設的 `.devcontainer/devcontainer.json ` 檔案發佈到應用程式的根目錄：

```shell
php artisan sail:install --devcontainer
```

<a name="rebuilding-sail-images"></a>
### 重建 Sail 映像檔

有時您可能需要完全重建 Sail 映像檔，以確保映像檔中的所有套件和軟體都是最新的。您可以使用 `build` 指令來完成此操作：

```shell
docker compose down -v

sail build --no-cache

sail up
```

<a name="configuring-a-shell-alias"></a>
### 設定 Shell 別名

預設情況下，Sail 指令是透過所有新的 Laravel 應用程式中包含的 `vendor/bin/sail` 腳本來呼叫的：

```shell
./vendor/bin/sail up
```

然而，為了避免重複輸入 `vendor/bin/sail` 來執行 Sail 指令，您可能希望設定一個 Shell 別名，讓您可以更輕鬆地執行 Sail 的指令：

```shell
alias sail='sh $([ -f sail ] && echo sail || echo vendor/bin/sail)'
```

為了確保此別名始終可用，您可以將其新增到您家目錄下的 Shell 設定檔中，例如 `~/.zshrc` 或 `~/.bashrc`，然後重新啟動您的 Shell。

一旦設定了 Shell 別名，您只需輸入 `sail` 即可執行 Sail 指令。本文件其餘的範例將假設您已經設定了此別名：

```shell
sail up
```

<a name="starting-and-stopping-sail"></a>
## 啟動與停止 Sail

Laravel Sail 的 `compose.yaml` 檔案定義了多種 Docker 容器，它們協同工作以幫助您建立 Laravel 應用程式。這些容器中的每一個都是您 `compose.yaml` 檔案的 `services` 設定中的一個項目。`laravel.test` 容器是主要應用程式容器，它將為您的應用程式提供服務。

在啟動 Sail 之前，您應該確保您的本機電腦上沒有其他網頁伺服器或資料庫正在執行。若要啟動應用程式的 `compose.yaml` 檔案中定義的所有 Docker 容器，您應該執行 `up` 指令：

```shell
sail up
```

若要在背景中啟動所有 Docker 容器，您可以以「分離」模式啟動 Sail：

```shell
sail up -d
```

一旦應用程式的容器啟動後，您可以在網路瀏覽器中透過 http://localhost 存取您的專案。

若要停止所有容器，您可以直接按下 Control + C 來停止容器的執行。或者，如果容器在背景中執行，您可以使用 `stop` 指令：

```shell
sail stop
```

<a name="executing-sail-commands"></a>
## 執行指令

當使用 Laravel Sail 時，您的應用程式會在 Docker 容器中執行，並與您的本機電腦隔離。然而，Sail 提供了一種方便的方式來對您的應用程式執行各種指令，例如任意的 PHP 指令、Artisan 指令、Composer 指令，以及 Node / NPM 指令。

**當閱讀 Laravel 文件時，您經常會看到對 Composer、Artisan 和 Node / NPM 指令的參考，而這些指令並未提及 Sail。** 這些範例假設這些工具已安裝在您的本機電腦上。如果您使用 Sail 作為本機 Laravel 開發環境，則應使用 Sail 執行這些指令：

```shell
# Running Artisan commands locally...
php artisan queue:work

# Running Artisan commands within Laravel Sail...
sail artisan queue:work
```

<a name="executing-php-commands"></a>
### 執行 PHP 指令

PHP 指令可以使用 `php` 指令執行。當然，這些指令將使用為您的應用程式配置的 PHP 版本來執行。若要了解更多關於 Laravel Sail 可用的 PHP 版本，請查閱 [PHP 版本文件](#sail-php-versions)：

```shell
sail php --version

sail php script.php
```

<a name="executing-composer-commands"></a>
### 執行 Composer 指令

Composer 指令可以使用 `composer` 指令來執行。Laravel Sail 的應用程式容器包含了 Composer 的安裝：

```shell
sail composer require laravel/sanctum
```

<a name="executing-artisan-commands"></a>
### 執行 Artisan 指令

Laravel Artisan 指令可以使用 `artisan` 指令來執行：

```shell
sail artisan queue:work
```

<a name="executing-node-npm-commands"></a>
### 執行 Node / NPM 指令

Node 指令可以使用 `node` 指令執行，而 NPM 指令可以使用 `npm` 指令執行：

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

您可能已經注意到，您的應用程式的 `compose.yaml` 檔案包含一個 MySQL 容器的項目。此容器使用 [Docker volume](https://docs.docker.com/storage/volumes/)，因此即使在停止並重新啟動容器後，資料庫中儲存的資料也會持續存在。

此外，MySQL 容器首次啟動時，會為您建立兩個資料庫。第一個資料庫是使用您 `DB_DATABASE` 環境變數的值命名的，用於您的本地開發。第二個是一個專用的測試資料庫，命名為 `testing`，它將確保您的測試不會干擾您的開發資料。

啟動容器後，您可以透過將應用程式 `.env` 檔案中的 `DB_HOST` 環境變數設定為 `mysql` 來連接到應用程式內的 MySQL 實例。

要從您的本地機器連接到應用程式的 MySQL 資料庫，您可以使用圖形化資料庫管理應用程式，例如 [TablePlus](https://tableplus.com)。預設情況下，MySQL 資料庫可在 `localhost` 的 3306 連接埠存取，且存取憑證與您的 `DB_USERNAME` 和 `DB_PASSWORD` 環境變數的值相對應。或者，您可以以 `root` 使用者身分連接，該使用者也使用您的 `DB_PASSWORD` 環境變數的值作為其密碼。


<a name="mongodb"></a>
### MongoDB

如果您在安裝 Sail 時選擇安裝 [MongoDB](https://www.mongodb.com/) 服務，您的應用程式的 `compose.yaml` 檔案將包含一個 [MongoDB Atlas Local](https://www.mongodb.com/docs/atlas/cli/current/atlas-cli-local-cloud/) 容器的項目，該容器提供具有 Atlas 功能 (例如 [Search Indexes](https://www.mongodb.com/docs/atlas/atlas-search/)) 的 MongoDB 文件資料庫。此容器使用 [Docker volume](https://docs.docker.com/storage/volumes/)，因此即使在停止並重新啟動容器後，資料庫中儲存的資料也會持續存在。

啟動容器後，您可以透過將應用程式 `.env` 檔案中的 `MONGODB_URI` 環境變數設定為 `mongodb://mongodb:27017` 來連接到應用程式內的 MongoDB 實例。預設情況下，驗證功能是停用的，但您可以在啟動 `mongodb` 容器之前設定 `MONGODB_USERNAME` 和 `MONGODB_PASSWORD` 環境變數來啟用驗證。然後，將憑證新增至連接字串：

```ini
MONGODB_USERNAME=user
MONGODB_PASSWORD=laravel
MONGODB_URI=mongodb://${MONGODB_USERNAME}:${MONGODB_PASSWORD}@mongodb:27017
```

為了將 MongoDB 與您的應用程式無縫整合，您可以安裝 [MongoDB 維護的官方套件](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/)。

要從您的本地機器連接到應用程式的 MongoDB 資料庫，您可以使用圖形化介面，例如 [Compass](https://www.mongodb.com/products/tools/compass)。預設情況下，MongoDB 資料庫可在 `localhost` 的 `27017` 連接埠存取。


<a name="redis"></a>
### Redis

您的應用程式的 `compose.yaml` 檔案還包含一個 [Redis](https://redis.io) 容器的項目。此容器使用 [Docker volume](https://docs.docker.com/storage/volumes/)，因此即使在停止並重新啟動容器後，Redis 實例中儲存的資料也會持續存在。一旦啟動容器，您可以透過將應用程式 `.env` 檔案中的 `REDIS_HOST` 環境變數設定為 `redis` 來連接到應用程式內的 Redis 實例。

要從您的本地機器連接到應用程式的 Redis 資料庫，您可以使用圖形化資料庫管理應用程式，例如 [TablePlus](https://tableplus.com)。預設情況下，Redis 資料庫可在 `localhost` 的 6379 連接埠存取。


<a name="valkey"></a>
### Valkey

如果您在安裝 Sail 時選擇安裝 Valkey 服務，您的應用程式的 `compose.yaml` 檔案將包含一個 [Valkey](https://valkey.io/) 的項目。此容器使用 [Docker volume](https://docs.docker.com/storage/volumes/)，因此即使在停止並重新啟動容器後，您的 Valkey 實例中儲存的資料也會持續存在。您可以透過將應用程式 `.env` 檔案中的 `REDIS_HOST` 環境變數設定為 `valkey` 來連接到應用程式中的此容器。

要從您的本地機器連接到應用程式的 Valkey 資料庫，您可以使用圖形化資料庫管理應用程式，例如 [TablePlus](https://tableplus.com)。預設情況下，Valkey 資料庫可在 `localhost` 的 6379 連接埠存取。


<a name="meilisearch"></a>
### Meilisearch

如果您在安裝 Sail 時選擇安裝 [Meilisearch](https://www.meilisearch.com) 服務，您的應用程式的 `compose.yaml` 檔案將包含此強大搜尋引擎的項目，該引擎與 [Laravel Scout](/docs/{{version}}/scout) 整合。一旦啟動容器，您可以透過將 `MEILISEARCH_HOST` 環境變數設定為 `http://meilisearch:7700` 來連接到應用程式內的 Meilisearch 實例。

從您的本地機器，您可以透過在網路瀏覽器中導航到 `http://localhost:7700` 來存取 Meilisearch 的基於網頁的管理面板。


<a name="typesense"></a>
### Typesense

如果您在安裝 Sail 時選擇安裝 [Typesense](https://typesense.org) 服務，您的應用程式的 `compose.yaml` 檔案將包含此閃電般快速、開源搜尋引擎的項目，該引擎與 [Laravel Scout](/docs/{{version}}/scout#typesense) 原生整合。一旦啟動容器，您可以透過設定以下環境變數來連接到應用程式內的 Typesense 實例：

```ini
TYPESENSE_HOST=typesense
TYPESENSE_PORT=8108
TYPESENSE_PROTOCOL=http
TYPESENSE_API_KEY=xyz
```

從您的本地機器，您可以透過 `http://localhost:8108` 存取 Typesense 的 API。


<a name="file-storage"></a>
## 檔案儲存

如果您計畫在應用程式的生產環境中執行時使用 Amazon S3 儲存檔案，您可能希望在安裝 Sail 時安裝 [MinIO](https://min.io) 服務。MinIO 提供了一個與 S3 相容的 API，您可以使用它來使用 Laravel 的 `s3` 檔案儲存驅動程式進行本地開發，而無需在您的生產 S3 環境中建立「測試」儲存儲存桶。如果您選擇在安裝 Sail 時安裝 MinIO，MinIO 設定區塊將新增到您的應用程式的 `compose.yaml` 檔案中。

預設情況下，您的應用程式的 `filesystems` 設定檔案已包含 `s3` 磁碟的磁碟設定。除了使用此磁碟與 Amazon S3 互動外，您還可以透過簡單地修改控制其設定的相關環境變數，使用它與任何 S3 相容的檔案儲存服務（例如 MinIO）互動。例如，當使用 MinIO 時，您的檔案系統環境變數設定應定義如下：

```ini
FILESYSTEM_DISK=s3
AWS_ACCESS_KEY_ID=sail
AWS_SECRET_ACCESS_KEY=password
AWS_DEFAULT_REGION=us-east-1
AWS_BUCKET=local
AWS_ENDPOINT=http://minio:9000
AWS_USE_PATH_STYLE_ENDPOINT=true
```

為了讓 Laravel 的 Flysystem 整合在使用 MinIO 時產生正確的 URL，您應該定義 `AWS_URL` 環境變數，使其與您的應用程式的本地 URL 匹配，並在 URL 路徑中包含儲存桶名稱：

```ini
AWS_URL=http://localhost:9000/local
```

您可以透過 MinIO 控制台建立儲存桶，該控制台可在 `http://localhost:8900` 存取。MinIO 控制台的預設使用者名稱為 `sail`，而預設密碼為 `password`。

> [!WARNING]
> 在使用 MinIO 時，不支援透過 `temporaryUrl` 方法產生臨時儲存 URL。

<a name="running-tests"></a>
## 執行測試

Laravel 開箱即用提供了強大的測試支援，您可以使用 Sail 的 `test` 指令來執行應用程式的[功能測試與單元測試](/docs/{{version}}/testing)。任何 Pest / PHPUnit 接受的 CLI 選項也可以傳遞給 `test` 指令：

```shell
sail test

sail test --group orders
```

Sail 的 `test` 指令等同於執行 `test` Artisan 指令：

```shell
sail artisan test
```

預設情況下，Sail 會建立一個專用的 `testing` 資料庫，這樣您的測試就不會干擾資料庫目前的狀態。在預設的 Laravel 安裝中，Sail 也會設定您的 `phpunit.xml` 檔案，以便在執行測試時使用這個資料庫：

```xml
<env name="DB_DATABASE" value="testing"/>
```


<a name="laravel-dusk"></a>
### Laravel Dusk

[Laravel Dusk](/docs/{{version}}/dusk) 提供了富有表現力且易於使用的瀏覽器自動化測試 API。由於 Sail 的幫助，您可以執行這些測試，而無需在本地電腦上安裝 Selenium 或其他工具。首先，請取消應用程式 `compose.yaml` 檔案中 Selenium 服務的註解：

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

接下來，請確保應用程式 `compose.yaml` 檔案中的 `laravel.test` 服務包含一個 `selenium` 的 `depends_on` 項目：

```yaml
depends_on:
    - mysql
    - redis
    - selenium
```

最後，您可以透過啟動 Sail 並執行 `dusk` 指令來運行您的 Dusk 測試套件：

```shell
sail dusk
```


<a name="selenium-on-apple-silicon"></a>
#### Selenium on Apple Silicon

如果您的本地機器包含 Apple Silicon 晶片，您的 `selenium` 服務必須使用 `selenium/standalone-chromium` 映像檔：

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

Laravel Sail 的預設 `compose.yaml` 檔案包含 [Mailpit](https://github.com/axllent/mailpit) 服務項目。Mailpit 會在本地開發期間攔截應用程式發送的電子郵件，並提供一個方便的網頁介面，讓您可以在瀏覽器中預覽電子郵件訊息。使用 Sail 時，Mailpit 的預設主機是 `mailpit`，可透過連接埠 1025 取得：

```ini
MAIL_HOST=mailpit
MAIL_PORT=1025
MAIL_ENCRYPTION=null
```

當 Sail 運行時，您可以透過 http://localhost:8025 存取 Mailpit 網頁介面。


<a name="sail-container-cli"></a>
## 容器 CLI

有時您可能希望在應用程式的容器中啟動 Bash 會話。您可以使用 `shell` 指令連接到應用程式的容器，讓您可以檢查其檔案和已安裝的服務，並在容器內執行任意的 Shell 指令：

```shell
sail shell

sail root-shell
```

若要啟動新的 [Laravel Tinker](https://github.com/laravel/tinker) 會話，您可以執行 `tinker` 指令：

```shell
sail tinker
```


<a name="sail-php-versions"></a>
## PHP 版本

Sail 目前支援透過 PHP 8.4、8.3、8.2、8.1 或 PHP 8.0 來提供您的應用程式服務。Sail 目前使用的預設 PHP 版本是 PHP 8.4。若要變更用於提供應用程式服務的 PHP 版本，您應該更新應用程式 `compose.yaml` 檔案中 `laravel.test` 容器的 `build` 定義：

```yaml
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

此外，您可能希望更新您的 `image` 名稱以反映應用程式所使用的 PHP 版本。這個選項也定義在應用程式的 `compose.yaml` 檔案中：

```yaml
image: sail-8.2/app
```

更新應用程式的 `compose.yaml` 檔案後，您應該重建您的容器映像檔：

```shell
sail build --no-cache

sail up
```


<a name="sail-node-versions"></a>
## Node 版本

Sail 預設安裝 Node 22。若要變更在建立映像檔時安裝的 Node 版本，您可以更新應用程式 `compose.yaml` 檔案中 `laravel.test` 服務的 `build.args` 定義：

```yaml
build:
    args:
        WWWGROUP: '${WWWGROUP}'
        NODE_VERSION: '18'
```

更新應用程式的 `compose.yaml` 檔案後，您應該重建您的容器映像檔：

```shell
sail build --no-cache

sail up
```


<a name="sharing-your-site"></a>
## 分享您的網站

有時您可能需要公開分享您的網站，以便供同事預覽您的網站或測試應用程式的 webhook 整合。若要分享您的網站，您可以使用 `share` 指令。執行此指令後，您將會收到一個隨機的 `laravel-sail.site` URL，您可以用它來存取您的應用程式：

```shell
sail share
```

透過 `share` 指令分享您的網站時，您應該在應用程式的 `bootstrap/app.php` 檔案中使用 `trustProxies` 中介層方法來設定應用程式的受信任代理。否則，諸如 `url` 和 `route` 之類的 URL 產生輔助函式將無法確定在 URL 產生期間應使用的正確 HTTP 主機：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->trustProxies(at: '*');
})
```

如果您想為您分享的網站選擇子網域，您可以在執行 `share` 指令時提供 `subdomain` 選項：

```shell
sail share --subdomain=my-sail-site
```

> [!NOTE]
> `share` 指令由 [BeyondCode](https://beyondco.de) 的開源隧道服務 [Expose](https://github.com/beyondcode/expose) 提供支援。

<a name="debugging-with-xdebug"></a>
## 使用 Xdebug 進行偵錯

Laravel Sail 的 Docker 設定包含對 [Xdebug](https://xdebug.org/) 的支援，Xdebug 是一個流行且功能強大的 PHP 偵錯工具。要啟用 Xdebug，請確保您已 [發布 Sail 設定](#sail-customization)。然後，將以下變數新增到應用程式的 `.env` 檔案中以設定 Xdebug：

```ini
SAIL_XDEBUG_MODE=develop,debug,coverage
```

接下來，請確保您發布的 `php.ini` 檔案包含以下設定，以便 Xdebug 以指定的模式啟用：

```ini
[xdebug]
xdebug.mode=${XDEBUG_MODE}
```

修改 `php.ini` 檔案後，請記得重建您的 Docker 映像檔，以使對 `php.ini` 檔案的變更生效：

```shell
sail build --no-cache
```


#### Linux 主機 IP 設定

在內部，`XDEBUG_CONFIG` 環境變數被定義為 `client_host=host.docker.internal`，以便 Xdebug 能正確地為 Mac 和 Windows (WSL2) 設定。如果您的本機是 Linux 且您正在使用 Docker 20.10+，則 `host.docker.internal` 可用，不需要手動設定。

對於 20.10 之前的 Docker 版本，Linux 不支援 `host.docker.internal`，您將需要手動定義主機 IP。為此，請透過在 `compose.yaml` 檔案中定義自訂網路來為容器設定靜態 IP：

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
### Xdebug CLI 使用方式

執行 Artisan 指令時，可以使用 `sail debug` 指令來啟動偵錯會話：

```shell
# Run an Artisan command without Xdebug...
sail artisan migrate

# Run an Artisan command with Xdebug...
sail debug migrate
```


<a name="xdebug-browser-usage"></a>
### Xdebug 瀏覽器使用方式

要在透過網頁瀏覽器與應用程式互動時偵錯您的應用程式，請按照 [Xdebug 提供的說明](https://xdebug.org/docs/step_debug#web-application) 從網頁瀏覽器啟動 Xdebug 會話。

如果您使用 PhpStorm，請查閱 JetBrains 關於 [零設定偵錯](https://www.jetbrains.com/help/phpstorm/zero-configuration-debugging.html) 的文件。

> [!WARNING]
> Laravel Sail 依賴 `artisan serve` 來服務您的應用程式。`artisan serve` 指令僅在 Laravel 8.53.0 版或更高版本中接受 `XDEBUG_CONFIG` 和 `XDEBUG_MODE` 變數。較舊的 Laravel 版本 (8.52.0 及以下) 不支援這些變數，也不會接受偵錯連線。


<a name="sail-customization"></a>
## 客製化

由於 Sail 只是 Docker，您可以自由地客製化它的幾乎所有內容。要發布 Sail 自己的 Dockerfiles，您可以執行 `sail:publish` 指令：

```shell
sail artisan sail:publish
```

執行此指令後，Laravel Sail 使用的 Dockerfiles 和其他設定檔將會放置在應用程式根目錄中的 `docker` 目錄內。客製化 Sail 安裝後，您可能希望變更應用程式容器在 `compose.yaml` 檔案中的映像檔名稱。完成此操作後，請使用 `build` 指令重建應用程式的容器。如果您在單一機器上使用 Sail 開發多個 Laravel 應用程式，為應用程式映像檔指定唯一名稱尤其重要：

```shell
sail build --no-cache
```