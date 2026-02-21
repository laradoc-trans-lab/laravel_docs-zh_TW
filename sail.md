# Laravel Sail

- [介紹](#introduction)
- [安裝與設定](#installation)
    - [在現有應用程式中安裝 Sail](#installing-sail-into-existing-applications)
    - [重新建置 Sail 映像檔](#rebuilding-sail-images)
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
- [預覽郵件](#previewing-emails)
- [容器 CLI](#sail-container-cli)
- [PHP 版本](#sail-php-versions)
- [Node 版本](#sail-node-versions)
- [分享您的網站](#sharing-your-site)
- [使用 Xdebug 進行偵錯](#debugging-with-xdebug)
  - [Xdebug CLI 使用方式](#xdebug-cli-usage)
  - [Xdebug 瀏覽器使用方式](#xdebug-browser-usage)
- [自定義](#sail-customization)

<a name="introduction"></a>
## 介紹

[Laravel Sail](https://github.com/laravel/sail) 是一個輕量級的命令列介面，用於與 Laravel 預設的 Docker 開發環境進行互動。Sail 為使用 PHP、MySQL 和 Redis 建置 Laravel 應用程式提供了一個絕佳的起點，且不需要事先具備 Docker 經驗。

Sail 的核心是儲存在專案根目錄中的 `compose.yaml` 檔案與 `sail` 指令稿。`sail` 指令稿提供了一個 CLI，其中包含與 `compose.yaml` 檔案定義的 Docker 容器進行互動的便捷方法。

Laravel Sail 支援在 macOS、Linux 和 Windows (透過 [WSL2](https://docs.microsoft.com/en-us/windows/wsl/about)) 上執行。


<a name="installation"></a>
## 安裝與設定

Laravel Sail 會自動安裝在所有新的 Laravel 應用程式中，因此您可以立即開始使用。


<a name="installing-sail-into-existing-applications"></a>
### 在現有應用程式中安裝 Sail

如果您有興趣在現有的 Laravel 應用程式中使用 Sail，可以簡單地使用 Composer 套件管理器來安裝 Sail。當然，這些步驟假設您現有的本地開發環境允許您安裝 Composer 依賴項目：

```shell
composer require laravel/sail --dev
```

安裝 Sail 後，您可以執行 `sail:install` Artisan 指令。此指令會將 Sail 的 `compose.yaml` 檔案發佈到應用程式的根目錄，並修改您的 `.env` 檔案以包含連接到 Docker 服務所需的環境變數：

```shell
php artisan sail:install
```

最後，您可以啟動 Sail。要繼續學習如何使用 Sail，請繼續閱讀本文件的其餘部分：

```shell
./vendor/bin/sail up
```

> [!WARNING]
> 如果您在 Linux 上使用 Docker Desktop，則應透過執行以下指令來使用 `default` Docker context：`docker context use default`。此外，如果您在容器內遇到檔案權限錯誤，則可能需要將 `SUPERVISOR_PHP_USER` 環境變數設定為 `root`。


<a name="adding-additional-services"></a>
#### 新增額外服務

如果您想在現有的 Sail 安裝中新增額外服務，可以執行 `sail:add` Artisan 指令：

```shell
php artisan sail:add
```


<a name="using-devcontainers"></a>
#### 使用 Devcontainers

如果您想在 [Devcontainer](https://code.visualstudio.com/docs/remote/containers) 中進行開發，可以為 `sail:install` 指令提供 `--devcontainer` 選項。`--devcontainer` 選項將指示 `sail:install` 指令將預設的 `.devcontainer/devcontainer.json ` 檔案發佈到應用程式的根目錄：

```shell
php artisan sail:install --devcontainer
```


<a name="rebuilding-sail-images"></a>
### 重新建置 Sail 映像檔

有時您可能想要完全重新建置您的 Sail 映像檔，以確保映像檔的所有套件和軟體都是最新的。您可以使用 `build` 指令來完成此操作：

```shell
docker compose down -v

sail build --no-cache

sail up
```


<a name="configuring-a-shell-alias"></a>
### 設定 Shell 別名

預設情況下，Sail 指令是使用隨所有新 Laravel 應用程式提供的 `vendor/bin/sail` 指令稿來呼叫的：

```shell
./vendor/bin/sail up
```

然而，您可能希望設定一個 Shell 別名，讓您可以更輕鬆地執行 Sail 的指令，而不是重複輸入 `vendor/bin/sail` 來執行 Sail 指令：

```shell
alias sail='sh $([ -f sail ] && echo sail || echo vendor/bin/sail)'
```

為了確保此設定始終可用，您可以將其新增至家目錄中的 Shell 設定檔中，例如 `~/.zshrc` 或 `~/.bashrc`，然後重新啟動您的 Shell。

設定好 Shell 別名後，您只需輸入 `sail` 即可執行 Sail 指令。本文件的其餘範例將假設您已設定此別名：

```shell
sail up
```


<a name="starting-and-stopping-sail"></a>
## 啟動與停止 Sail

Laravel Sail 的 `compose.yaml` 檔案定義了各種 Docker 容器，這些容器協同工作以幫助您建置 Laravel 應用程式。這些容器中的每一個都是 `compose.yaml` 檔案中 `services` 設定中的一個項目。`laravel.test` 容器是為您的應用程式提供服務的主要應用程式容器。

在啟動 Sail 之前，您應該確保本地電腦上沒有執行其他的網頁伺服器或資料庫。要啟動應用程式的 `compose.yaml` 檔案中定義的所有 Docker 容器，您應該執行 `up` 指令：

```shell
sail up
```

要讓所有 Docker 容器在背景執行，您可以用「分離 (detached)」模式啟動 Sail：

```shell
sail up -d
```

應用程式容器啟動後，您可以在網頁瀏覽器中透過以下網址存取專案：http://localhost。

要停止所有容器，只需按下 Control + C 即可停止容器的執行。或者，如果容器在背景執行，您可以使用 `stop` 指令：

```shell
sail stop
```


<a name="executing-sail-commands"></a>
## 執行指令

使用 Laravel Sail 時，您的應用程式是在 Docker 容器內執行，並且與您的本地電腦隔離。然而，Sail 提供了一種簡便的方法來對您的應用程式執行各種指令，例如任意 PHP 指令、Artisan 指令、Composer 指令以及 Node / NPM 指令。

**在閱讀 Laravel 文件時，您經常會看到對 Composer、Artisan 和 Node / NPM 指令的引用，但這些引用並未提到 Sail。** 這些範例假設這些工具已安裝在您的本地電腦上。如果您使用 Sail 作為本地 Laravel 開發環境，則應使用 Sail 執行這些指令：

```shell
# Running Artisan commands locally...
php artisan queue:work

# Running Artisan commands within Laravel Sail...
sail artisan queue:work
```


<a name="executing-php-commands"></a>
### 執行 PHP 指令

PHP 指令可以使用 `php` 指令來執行。當然，這些指令將使用為您的應用程式設定的 PHP 版本。要進一步了解 Laravel Sail 可用的 PHP 版本，請參閱 [PHP 版本文件](#sail-php-versions)：

```shell
sail php --version

sail php script.php
```


<a name="executing-composer-commands"></a>
### 執行 Composer 指令

Composer 指令可以使用 `composer` 指令來執行。Laravel Sail 的應用程式容器包含了 Composer 安裝：

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

如果您願意，您可以使用 Yarn 來代替 NPM：

```shell
sail yarn
```

<a name="interacting-with-sail-databases"></a>
## 與資料庫互動


<a name="mysql"></a>
### MySQL

如您所見，應用程式的 `compose.yaml` 檔案包含了一個 MySQL 容器的項目。此容器使用 [Docker 磁碟卷軸 (Docker volume)](https://docs.docker.com/storage/volumes/)，因此即使停止並重啟容器，儲存在資料庫中的資料也會持續存在。

此外，MySQL 容器第一次啟動時，它會為您建立兩個資料庫。第一個資料庫是根據您的 `DB_DATABASE` 環境變數值命名的，用於本機開發。第二個是名為 `testing` 的專用測試資料庫，以確保您的測試不會干擾開發資料。

啟動容器後，您可以透過將應用程式 `.env` 檔案中的 `DB_HOST` 環境變數設定為 `mysql`，來連線至應用程式內的 MySQL 執行個體。

若要從本機連線至應用程式的 MySQL 資料庫，您可以使用如 [TablePlus](https://tableplus.com) 之類的圖形化資料庫管理應用程式。預設情況下，MySQL 資料庫可透過 `localhost` 的 3306 埠存取，存取憑證與 `DB_USERNAME` 和 `DB_PASSWORD` 環境變數的值相對應。或者，您可以以 `root` 使用者身分連線，這同樣會使用 `DB_PASSWORD` 環境變數的值作為密碼。


<a name="mongodb"></a>
### MongoDB

如果您在安裝 Sail 時選擇安裝 [MongoDB](https://www.mongodb.com/) 服務，您的應用程式 `compose.yaml` 檔案將包含一個 [MongoDB Atlas Local](https://www.mongodb.com/docs/atlas/cli/current/atlas-cli-local-cloud/) 容器的項目，它提供了具備 Atlas 功能（如 [搜尋索引 (Search Indexes)](https://www.mongodb.com/docs/atlas/atlas-search/)）的 MongoDB 文件資料庫。此容器使用 [Docker 磁碟卷軸 (Docker volume)](https://docs.docker.com/storage/volumes/)，因此即使停止並重啟容器，儲存在資料庫中的資料也會持續存在。

啟動容器後，您可以透過將應用程式 `.env` 檔案中的 `MONGODB_URI` 環境變數設定為 `mongodb://mongodb:27017`，來連線至應用程式內的 MongoDB 執行個體。預設情況下停用驗證，但在啟動 `mongodb` 容器之前，您可以設定 `MONGODB_USERNAME` 和 `MONGODB_PASSWORD` 環境變數以啟用驗證。接著，將憑證加入連線字串：

```ini
MONGODB_USERNAME=user
MONGODB_PASSWORD=laravel
MONGODB_URI=mongodb://${MONGODB_USERNAME}:${MONGODB_PASSWORD}@mongodb:27017
```

為了將 MongoDB 與您的應用程式無縫整合，您可以安裝由 [MongoDB 官方維護的套件](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/)。

若要從本機連線至應用程式的 MongoDB 資料庫，您可以使用如 [Compass](https://www.mongodb.com/products/tools/compass) 之類的圖形化介面。預設情況下，MongoDB 資料庫可透過 `localhost` 的 `27017` 埠存取。


<a name="redis"></a>
### Redis

您的應用程式 `compose.yaml` 檔案也包含一個 [Redis](https://redis.io) 容器項目。此容器使用 [Docker 磁碟卷軸 (Docker volume)](https://docs.docker.com/storage/volumes/)，因此即使停止並重啟容器，儲存在 Redis 執行個體中的資料也會持續存在。啟動容器後，您可以透過將應用程式 `.env` 檔案中的 `REDIS_HOST` 環境變數設定為 `redis`，來連線至應用程式內的 Redis 執行個體。

若要從本機連線至應用程式的 Redis 資料庫，您可以使用如 [TablePlus](https://tableplus.com) 之類的圖形化資料庫管理應用程式。預設情況下，Redis 資料庫可透過 `localhost` 的 6379 埠存取。


<a name="valkey"></a>
### Valkey

如果您在安裝 Sail 時選擇安裝 Valkey 服務，您的應用程式 `compose.yaml` 檔案將包含一個 [Valkey](https://valkey.io/) 項目。此容器使用 [Docker 磁碟卷軸 (Docker volume)](https://docs.docker.com/storage/volumes/)，因此即使停止並重啟容器，儲存在 Valkey 執行個體中的資料也會持續存在。您可以透過將應用程式 `.env` 檔案中的 `REDIS_HOST` 環境變數設定為 `valkey`，在應用程式中連線至此容器。

若要從本機連線至應用程式的 Valkey 資料庫，您可以使用如 [TablePlus](https://tableplus.com) 之類的圖形化資料庫管理應用程式。預設情況下，Valkey 資料庫可透過 `localhost` 的 6379 埠存取。


<a name="meilisearch"></a>
### Meilisearch

如果您在安裝 Sail 時選擇安裝 [Meilisearch](https://www.meilisearch.com) 服務，您的應用程式 `compose.yaml` 檔案將包含一個與 [Laravel Scout](/docs/{{version}}/scout) 整合的強大搜尋引擎項目。啟動容器後，您可以透過將 `MEILISEARCH_HOST` 環境變數設定為 `http://meilisearch:7700`，來連線至應用程式內的 Meilisearch 執行個體。

從您的本機，您可以透過在瀏覽器中導覽至 `http://localhost:7700` 來存取 Meilisearch 網頁版管理介面。


<a name="typesense"></a>
### Typesense

如果您在安裝 Sail 時選擇安裝 [Typesense](https://typesense.org) 服務，您的應用程式 `compose.yaml` 檔案將包含一個極速、開源的搜尋引擎項目，它與 [Laravel Scout](/docs/{{version}}/scout#typesense) 原生整合。啟動容器後，您可以透過設定以下環境變數來連線至應用程式內的 Typesense 執行個體：

```ini
TYPESENSE_HOST=typesense
TYPESENSE_PORT=8108
TYPESENSE_PROTOCOL=http
TYPESENSE_API_KEY=xyz
```

從您的本機，您可以透過 `http://localhost:8108` 存取 Typesense API。


<a name="file-storage"></a>
## 檔案儲存

如果您打算在正式環境中使用 Amazon S3 儲存檔案，您可能會希望在安裝 Sail 時安裝 [RustFS](https://rustfs.com) 服務。RustFS 提供了 S3 相容的 API，讓您可以使用 Laravel 的 `s3` 檔案儲存驅動程式進行本機開發，而無需在正式環境的 S3 中建立「測試」儲存貯體 (Storage buckets)。如果您在安裝 Sail 時選擇安裝 RustFS，一個 RustFS 設定區段將會被加入到應用程式的 `compose.yaml` 檔案中。

預設情況下，應用程式的 `filesystems` 設定檔已經包含了一個 `s3` 磁碟設定。除了使用此磁碟與 Amazon S3 互動外，您還可以透過修改控制其設定的相關環境變數，將其用於與任何 S3 相容的檔案儲存服務（如 RustFS）互動。例如，使用 RustFS 時，您的檔案系統環境變數設定應定義如下：

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

Laravel 提供開箱即用的卓越測試支援，您可以使用 Sail 的 `test` 指令來執行您的應用程式[功能與單元測試](/docs/{{version}}/testing)。任何 Pest / PHPUnit 所接受的 CLI 選項也可以傳遞給 `test` 指令：

```shell
sail test

sail test --group orders
```

Sail 的 `test` 指令等同於執行 `test` Artisan 指令：

```shell
sail artisan test
```

預設情況下，Sail 會建立一個專用的 `testing` 資料庫，這樣您的測試就不會干擾到您目前的資料庫狀態。在預設的 Laravel 安裝中，Sail 也會設定您的 `phpunit.xml` 檔案，以便在執行測試時使用此資料庫：

```xml
<env name="DB_DATABASE" value="testing"/>
```


<a name="laravel-dusk"></a>
### Laravel Dusk

[Laravel Dusk](/docs/{{version}}/dusk) 提供了一個富有表現力且易於使用的瀏覽器自動化與測試 API。多虧了 Sail，您無需在本地電腦安裝 Selenium 或其他工具即可執行這些測試。要開始使用，請在應用程式的 `compose.yaml` 檔案中取消 Selenium 服務的註解：

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

接下來，確保您的應用程式 `compose.yaml` 檔案中的 `laravel.test` 服務在 `depends_on` 項目中有包含 `selenium`：

```yaml
depends_on:
    - mysql
    - redis
    - selenium
```

最後，您可以透過啟動 Sail 並執行 `dusk` 指令來執行您的 Dusk 測試套件：

```shell
sail dusk
```


<a name="selenium-on-apple-silicon"></a>
#### Selenium on Apple Silicon

如果您的本地機器使用 Apple Silicon 晶片，您的 `selenium` 服務必須使用 `selenium/standalone-chromium` 映像檔：

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

Laravel Sail 預設的 `compose.yaml` 檔案包含一個 [Mailpit](https://github.com/axllent/mailpit) 服務項目。Mailpit 會攔截應用程式在本地開發期間發送的電子郵件，並提供一個方便的網頁介面，讓您可以在瀏覽器中預覽您的郵件內容。使用 Sail 時，Mailpit 的預設主機為 `mailpit`，並可透過 1025 連接埠使用：

```ini
MAIL_HOST=mailpit
MAIL_PORT=1025
MAIL_ENCRYPTION=null
```

當 Sail 執行時，您可以透過 http://localhost:8025 存取 Mailpit 網頁介面。


<a name="sail-container-cli"></a>
## 容器 CLI

有時您可能希望在應用程式容器內開啟 Bash 工作階段。您可以使用 `shell` 指令來連接到應用程式容器，這讓您可以檢查其中的檔案與已安裝的服務，並在容器內執行任意 shell 指令：

```shell
sail shell

sail root-shell
```

要開始一個新的 [Laravel Tinker](https://github.com/laravel/tinker) 工作階段，您可以執行 `tinker` 指令：

```shell
sail tinker
```


<a name="sail-php-versions"></a>
## PHP 版本

Sail 目前支援透過 PHP 8.5, 8.4, 8.3, 8.2, 8.1 或 PHP 8.0 來運行您的應用程式。Sail 目前預設使用的 PHP 版本為 PHP 8.4。要更改用於運行應用程式的 PHP 版本，您應該更新應用程式 `compose.yaml` 檔案中 `laravel.test` 容器的 `build` 定義：

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

此外，您可能希望更新您的 `image` 名稱以反映應用程式所使用的 PHP 版本。此選項同樣是在您應用程式的 `compose.yaml` 檔案中定義：

```yaml
image: sail-8.2/app
```

更新應用程式的 `compose.yaml` 檔案後，您應該重新建置容器映像檔：

```shell
sail build --no-cache

sail up
```


<a name="sail-node-versions"></a>
## Node 版本

Sail 預設安裝 Node 22。要更改建置映像檔時安裝的 Node 版本，您可以更新應用程式 `compose.yaml` 檔案中 `laravel.test` 服務的 `build.args` 定義：

```yaml
build:
    args:
        WWWGROUP: '${WWWGROUP}'
        NODE_VERSION: '18'
```

更新應用程式的 `compose.yaml` 檔案後，您應該重新建置容器映像檔：

```shell
sail build --no-cache

sail up
```


<a name="sharing-your-site"></a>
## 分享您的網站

有時您可能需要公開分享您的網站，以便讓同事預覽或測試與應用程式整合的 Webhook。要分享您的網站，您可以使用 `share` 指令。執行此指令後，系統會發給您一個隨機的 `laravel-sail.site` 網址，您可以用它來存取您的應用程式：

```shell
sail share
```

透過 `share` 指令分享您的網站時，您應該在應用程式的 `bootstrap/app.php` 檔案中使用 `trustProxies` 中介層方法來設定應用程式的信任代理 (Trusted Proxies)。否則，像是 `url` 與 `route` 這種網址產生輔助函式將無法判斷在產生網址時應使用的正確 HTTP 主機：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->trustProxies(at: '*');
})
```

如果您想為分享的網站選擇子網域，可以在執行 `share` 指令時加上 `subdomain` 選項：

```shell
sail share --subdomain=my-sail-site
```

> [!NOTE]
> `share` 指令是由 [Expose](https://github.com/beyondcode/expose) 提供技術支援，這是一個由 [BeyondCode](https://beyondco.de) 開發的開源隧道服務。

<a name="debugging-with-xdebug"></a>
## 使用 Xdebug 進行偵錯

Laravel Sail 的 Docker 設定包含對 [Xdebug](https://xdebug.org/) 的支援，這是一個受歡迎且強大的 PHP 偵錯器。若要啟用 Xdebug，請確保您已經[發布您的 Sail 設定](#sail-customization)。接著，在您應用程式的 `.env` 檔案中新增以下變數來設定 Xdebug：

```ini
SAIL_XDEBUG_MODE=develop,debug,coverage
```

接下來，請確保您發布的 `php.ini` 檔案包含以下設定，以便在指定的模式中啟動 Xdebug：

```ini
[xdebug]
xdebug.mode=${XDEBUG_MODE}
```

修改 `php.ini` 檔案後，請記得重新建置您的 Docker 映像檔，以使對 `php.ini` 檔案的更改生效：

```shell
sail build --no-cache
```

#### Linux 主機 IP 設定

在內部，`XDEBUG_CONFIG` 環境變數被定義為 `client_host=host.docker.internal`，以便為 Mac 和 Windows (WSL2) 正確設定 Xdebug。如果您的本機電腦執行的是 Linux 且您使用的是 Docker 20.10+，則可以使用 `host.docker.internal`，且不需要手動設定。

對於早於 20.10 的 Docker 版本，Linux 不支援 `host.docker.internal`，您將需要手動定義主機 IP。為此，請透過在您的 `compose.yaml` 檔案中定義自定義網路來為您的容器設定靜態 IP：

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

設定好靜態 IP 後，請在您應用程式的 .env 檔案中定義 SAIL_XDEBUG_CONFIG 變數：

```ini
SAIL_XDEBUG_CONFIG="client_host=172.20.0.2"
```

<a name="xdebug-cli-usage"></a>
### Xdebug CLI 使用方式

在執行 Artisan 指令時，可以使用 `sail debug` 指令來啟動偵錯工作階段：

```shell
# Run an Artisan command without Xdebug...
sail artisan migrate

# Run an Artisan command with Xdebug...
sail debug migrate
```

<a name="xdebug-browser-usage"></a>
### Xdebug 瀏覽器使用方式

若要在透過網頁瀏覽器與應用程式互動時進行偵錯，請按照 [Xdebug 提供的說明](https://xdebug.org/docs/step_debug#web-application) 從網頁瀏覽器啟動 Xdebug 工作階段。

如果您使用的是 PhpStorm，請參閱 JetBrains 關於[零設定偵錯](https://www.jetbrains.com/help/phpstorm/zero-configuration-debugging.html)的說明文件。

> [!WARNING]
> Laravel Sail 依賴 `artisan serve` 來提供您的應用程式。自 Laravel 版本 8.53.0 起，`artisan serve` 指令僅接受 `XDEBUG_CONFIG` 和 `XDEBUG_MODE` 變數。較舊版本的 Laravel（8.52.0 及以下版本）不支援這些變數，且不會接受偵錯連線。

<a name="sail-customization"></a>
## 自定義

由於 Sail 只是 Docker，您可以自由地自定義幾乎所有內容。若要發布 Sail 自身的 Dockerfiles，您可以執行 `sail:publish` 指令：

```shell
sail artisan sail:publish
```

執行此指令後，Laravel Sail 使用的 Dockerfiles 和其他設定檔將放置在應用程式根目錄下的 `docker` 目錄中。自定義 Sail 安裝後，您可能希望更改應用程式 `compose.yaml` 檔案中應用程式容器的映像檔名稱。完成後，使用 `build` 指令重新建置應用程式的容器。如果您使用 Sail 在單台機器上開發多個 Laravel 應用程式，為應用程式映像檔指定唯一的名稱尤為重要：

```shell
sail build --no-cache
```