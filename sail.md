# Laravel Sail

- [簡介](#introduction)
- [安裝與設定](#installation)
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
    - [額外的 PHP 擴充功能](#sail-php-extensions)
- [Node 版本](#sail-node-versions)
- [分享您的網站](#sharing-your-site)
- [使用 Xdebug 進行偵錯](#debugging-with-xdebug)
    - [Xdebug CLI 用法](#xdebug-cli-usage)
    - [Xdebug 瀏覽器用法](#xdebug-browser-usage)
- [客製化](#sail-customization)

<a name="introduction"></a>
## 簡介

[Laravel Sail](https://github.com/laravel/sail) 是一個輕量級的命令列介面，用於與 Laravel 預設的 Docker 開發環境進行互動。Sail 提供了一個絕佳的起點，讓您可以使用 PHP、MySQL 及 Redis 建置 Laravel 應用程式，且不需要具備事先的 Docker 經驗。

Sail 的核心是儲存於您專案根目錄中的 `compose.yaml` 檔案與 `sail` 腳本。`sail` 腳本提供了一個 CLI，包含各種便利的方法，可用於與 `compose.yaml` 檔案所定義的 Docker 容器進行互動。

Laravel Sail 在 macOS、Linux 與 Windows（透過 [WSL2](https://docs.microsoft.com/en-us/windows/wsl/about)）上皆受支援。


<a name="installation"></a>
## 安裝與設定

您可以使用 Composer 套件管理工具來安裝 Sail：

```shell
composer require laravel/sail --dev
```

安裝 Sail 之後，您可以執行 `sail:install` Artisan 指令。此指令會將 Sail 的 `compose.yaml` 檔案發布到您應用程式的根目錄，並修改您的 `.env` 檔案以填入連接 Docker 服務所需的環境變數：

```shell
php artisan sail:install
```

最後，您可以啟動 Sail。若要繼續學習如何使用 Sail，請繼續閱讀本文件剩餘的內容：

```shell
./vendor/bin/sail up
```

> [!WARNING]
> 若您正在使用 Linux 版的 Docker Desktop，您應該透過執行以下指令來使用 `default` Docker context：`docker context use default`。此外，若您在容器內遇到檔案權限錯誤，您可能需要將 `SUPERVISOR_PHP_USER` 環境變數設定為 `root`。


<a name="adding-additional-services"></a>
#### 新增額外的服務

如果您想在現有的 Sail 安裝中新增額外的服務，可以執行 `sail:add` Artisan 指令：

```shell
php artisan sail:add
```


<a name="using-devcontainers"></a>
#### 使用 Devcontainer

如果您想在 [Devcontainer](https://code.visualstudio.com/docs/remote/containers) 環境中進行開發，可以在執行 `sail:install` 指令時提供 `--devcontainer` 選項。`--devcontainer` 選項會指示 `sail:install` 指令將預設的 `.devcontainer/devcontainer.json ` 檔案發布到應用程式的根目錄：

```shell
php artisan sail:install --devcontainer
```


<a name="rebuilding-sail-images"></a>
### 重建 Sail 映像檔

有時候您可能想要完全重建 Sail 映像檔，以確保映像檔中的所有套件與軟體都是最新的。您可以透過使用 `build` 指令來達成此目的：

```shell
docker compose down -v

sail build --no-cache

sail up
```


<a name="configuring-a-shell-alias"></a>
### 設定 Shell 別名

預設情況下，Sail 指令是透過所有新 Laravel 應用程式中附帶的 `vendor/bin/sail` 腳本來呼叫的：

```shell
./vendor/bin/sail up
```

然而，與其重複輸入 `vendor/bin/sail` 來執行 Sail 指令，您可能希望設定一個 shell 別名，讓您能更輕鬆地執行 Sail 的指令：

```shell
alias sail='sh $([ -f sail ] && echo sail || echo vendor/bin/sail)'
```

為了確保此別名始終可用，您可以將其新增至家目錄中的 shell 設定檔（例如 `~/.zshrc` 或 `~/.bashrc`），然後重啟您的 shell。

設定好 shell 別名後，您只需輸入 `sail` 即可執行 Sail 指令。本文件其餘範例皆假設您已經設定了此別名：

```shell
sail up
```


<a name="starting-and-stopping-sail"></a>
## 啟動與停止 Sail

Laravel Sail 的 `compose.yaml` 檔案定義了各種不同的 Docker 容器，它們協同工作以協助您建置 Laravel 應用程式。這些容器中的每一個都是 `compose.yaml` 檔案中 `services` 設定裡的一個項目。`laravel.test` 容器是為您的應用程式提供服務的主要應用程式容器。

在啟動 Sail 之前，您應該確保本機電腦上沒有其他 Web 伺服器或資料庫正在執行。若要啟動您應用程式 `compose.yaml` 檔案中定義的所有 Docker 容器，您應該執行 `up` 指令：

```shell
sail up
```

若要在背景啟動所有 Docker 容器，您可以在「分離 (detached)」模式下啟動 Sail：

```shell
sail up -d
```

當應用程式的容器啟動後，您可以在 Web 瀏覽器中造訪專案：http://localhost。

若要停止所有容器，您可以按下 Control + C 來停止容器的執行。或者，若容器是在背景執行，您可以使用 `stop` 指令：

```shell
sail stop
```


<a name="executing-sail-commands"></a>
## 執行指令

使用 Laravel Sail 時，您的應用程式是在 Docker 容器內執行，並與本機電腦隔離開來。不過，Sail 提供了一種便利的方式來對您的應用程式執行各種指令，例如任意的 PHP 指令、Artisan 指令、Composer 指令以及 Node / NPM 指令。

**在閱讀 Laravel 官方文件時，您經常會看到提及 Composer、Artisan 以及 Node / NPM 指令，但這些參考範例並未提及 Sail。** 那些範例假設這些工具已經安裝在您的本機電腦上。如果您正在使用 Sail 作為本機的 Laravel 開發環境，則應使用 Sail 來執行這些指令：

```shell
# Running Artisan commands locally...
php artisan queue:work

# Running Artisan commands within Laravel Sail...
sail artisan queue:work
```


<a name="executing-php-commands"></a>
### 執行 PHP 指令

PHP 指令可以使用 `php` 指令來執行。當然，這些指令將使用為您應用程式所設定的 PHP 版本來執行。若要深入瞭解 Laravel Sail 可用的 PHP 版本，請參閱 [PHP 版本文件](#sail-php-versions)：

```shell
sail php --version

sail php script.php
```


<a name="executing-composer-commands"></a>
### 執行 Composer 指令

Composer 指令可以使用 `composer` 指令來執行。Laravel Sail 的應用程式容器內內建了 Composer：

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

Node 指令可以使用 `node` 指令來執行，而 NPM 指令則可以使用 `npm` 指令來執行：

```shell
sail node --version

sail npm run dev
```

如果您願意，也可以使用 Yarn 來替代 NPM：

```shell
sail yarn
```

<a name="interacting-with-sail-databases"></a>
## 與資料庫互動


<a name="mysql"></a>
### MySQL

您可能已經注意到，應用程式的 `compose.yaml` 檔案中包含一個 MySQL 容器的設定項目。這個容器使用了 [Docker volume](https://docs.docker.com/storage/volumes/)，因此即使停止並重新啟動容器，儲存在資料庫中的資料也會持續保留。

此外，當 MySQL 容器首次啟動時，它會為您建立兩個資料庫。第一個資料庫會使用 `DB_DATABASE` 環境變數的值來命名，用於您的本機開發。第二個則是名為 `testing` 的專用測試資料庫，以確保您的測試不會干擾到開發資料。

啟動容器後，您可以透過將應用程式 `.env` 檔案中的 `DB_HOST` 環境變數設定為 `mysql`，來從應用程式內部連接到 MySQL 執行個體。

若要從本機電腦連接至應用程式的 MySQL 資料庫，您可以使用圖形化資料庫管理應用程式，例如 [TablePlus](https://tableplus.com)。預設情況下，MySQL 資料庫可透過 `localhost` 的連接埠 3306 存取，存取憑證則對應至 `DB_USERNAME` 和 `DB_PASSWORD` 環境變數的值。或者，您也可以使用 `root` 使用者進行連接，該使用者同樣使用 `DB_PASSWORD` 環境變數的值作為密碼。


<a name="mongodb"></a>
### MongoDB

若您在安裝 Sail 時選擇安裝 [MongoDB](https://www.mongodb.com/) 服務，您的應用程式 `compose.yaml` 檔案將包含一個 [MongoDB Atlas Local](https://www.mongodb.com/docs/atlas/cli/current/atlas-cli-local-cloud/) 容器的設定項目，該容器提供具有 Atlas 功能（如 [搜尋索引 (Search Indexes)](https://www.mongodb.com/docs/atlas/atlas-search/)）的 MongoDB 文件資料庫。這個容器使用了 [Docker volume](https://docs.docker.com/storage/volumes/)，因此即使停止並重新啟動容器，儲存在資料庫中的資料也會持續保留。

啟動容器後，您可以透過將應用程式 `.env` 檔案中的 `MONGODB_URI` 環境變數設定為 `mongodb://mongodb:27017`，來從應用程式內部連接到 MongoDB 執行個體。預設情況下，身份驗證功能是停用的，但您可以在啟動 `mongodb` 容器之前，設定 `MONGODB_USERNAME` 和 `MONGODB_PASSWORD` 環境變數來啟用身份驗證。接著，將憑證新增至連接字串中：

```ini
MONGODB_USERNAME=user
MONGODB_PASSWORD=laravel
MONGODB_URI=mongodb://${MONGODB_USERNAME}:${MONGODB_PASSWORD}@mongodb:27017
```

為了將 MongoDB 完美整合至您的應用程式中，您可以安裝由 [MongoDB 官方維護的套件](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/)。

若要從本機電腦連接至應用程式的 MongoDB 資料庫，您可以使用圖形化介面，例如 [Compass](https://www.mongodb.com/products/tools/compass)。預設情況下，MongoDB 資料庫可透過 `localhost` 的連接埠 `27017` 存取。


<a name="redis"></a>
### Redis

應用程式的 `compose.yaml` 檔案中也包含一個 [Redis](https://redis.io) 容器的設定項目。這個容器使用了 [Docker volume](https://docs.docker.com/storage/volumes/)，因此即使停止並重新啟動容器，儲存在 Redis 執行個體中的資料也會持續保留。啟動容器後，您可以透過將應用程式 `.env` 檔案中的 `REDIS_HOST` 環境變數設定為 `redis`，來從應用程式內部連接到 Redis 執行個體。

若要從本機電腦連接至應用程式的 Redis 資料庫，您可以使用圖形化資料庫管理應用程式，例如 [TablePlus](https://tableplus.com)。預設情況下，Redis 資料庫可透過 `localhost` 的連接埠 6379 存取。


<a name="valkey"></a>
### Valkey

若您在安裝 Sail 時選擇安裝 Valkey 服務，您的應用程式 `compose.yaml` 檔案將包含 [Valkey](https://valkey.io/) 的設定項目。這個容器使用了 [Docker volume](https://docs.docker.com/storage/volumes/)，因此即使停止並重新啟動容器，儲存在 Valkey 執行個體中的資料也會持續保留。您可以透過將應用程式 `.env` 檔案中的 `REDIS_HOST` 環境變數設定為 `valkey`，來在應用程式中連接到這個容器。

若要從本機電腦連接至應用程式的 Valkey 資料庫，您可以使用圖形化資料庫管理應用程式，例如 [TablePlus](https://tableplus.com)。預設情況下，Valkey 資料庫可透過 `localhost` 的連接埠 6379 存取。


<a name="meilisearch"></a>
### Meilisearch

若您在安裝 Sail 時選擇安裝 [Meilisearch](https://www.meilisearch.com) 服務，您的應用程式 `compose.yaml` 檔案將包含這個與 [Laravel Scout](/docs/{{version}}/scout) 整合的強大搜尋引擎的設定項目。啟動容器後，您可以透過將 `MEILISEARCH_HOST` 環境變數設定為 `http://meilisearch:7700`，來從應用程式內部連接到 Meilisearch 執行個體。

在本機電腦上，您可以透過在網頁瀏覽器中瀏覽 `http://localhost:7700` 來存取 Meilisearch 的網頁管理面板。


<a name="typesense"></a>
### Typesense

若您在安裝 Sail 時選擇安裝 [Typesense](https://typesense.org) 服務，您的應用程式 `compose.yaml` 檔案將包含這個原生整合至 [Laravel Scout](/docs/{{version}}/scout#typesense) 的極速開源搜尋引擎的設定項目。啟動容器後，您可以透過設定以下環境變數，來從應用程式內部連接到 Typesense 執行個體：

```ini
TYPESENSE_HOST=typesense
TYPESENSE_PORT=8108
TYPESENSE_PROTOCOL=http
TYPESENSE_API_KEY=xyz
```

在本機電腦上，您可以透過 `http://localhost:8108` 存取 Typesense 的 API。


<a name="file-storage"></a>
## 檔案儲存

如果您打算在正式環境中執行應用程式時使用 Amazon S3 來儲存檔案，您可能會希望在安裝 Sail 時安裝 [RustFS](https://rustfs.com) 服務。RustFS 提供了相容於 S3 的 API，您可以利用它搭配 Laravel 的 `s3` 檔案儲存驅動進行本機開發，而不需要在正式環境的 S3 中建立「測試用」的儲存桶 (Bucket)。若您在安裝 Sail 時選擇安裝 RustFS，RustFS 的設定段落將會被新增至您應用程式的 `compose.yaml` 檔案中。

預設情況下，您應用程式的 `filesystems` 設定檔已經包含 `s3` 磁碟的設定。除了使用此磁碟與 Amazon S3 互動外，您還可以透過修改控制其設定的相關環境變數，將其用於與任何相容於 S3 的檔案儲存服務（例如 RustFS）進行互動。例如，當使用 RustFS 時，您的檔案系統環境變數設定應定義如下：

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

Laravel 提供開箱即用的強大測試支援，您可以透過 Sail 的 `test` 指令來執行應用程式的[功能與單元測試](/docs/{{version}}/testing)。Pest / PHPUnit 所接受的任何 CLI 選項都可以傳遞給 `test` 指令：

```shell
sail test

sail test --group orders
```

Sail 的 `test` 指令等同於執行 `test` Artisan 指令：

```shell
sail artisan test
```

預設情況下，Sail 會建立一個專用的 `testing` 資料庫，確保您的測試不會干擾當前的資料庫狀態。在預設的 Laravel 安裝中，Sail 也會設定您的 `phpunit.xml` 檔案，以便在執行測試時使用此資料庫：

```xml
<env name="DB_DATABASE" value="testing"/>
```


<a name="laravel-dusk"></a>
### Laravel Dusk

[Laravel Dusk](/docs/{{version}}/dusk) 提供流暢且易於使用的瀏覽器自動化與測試 API。多虧了 Sail，您完全不需要在本地電腦上安裝 Selenium 或其他工具即可執行這些測試。若要開始使用，請取消註解您應用程式 `compose.yaml` 檔案中的 Selenium 服務：

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

接下來，請確保您應用程式 `compose.yaml` 檔案中的 `laravel.test` 服務擁有指向 `selenium` 的 `depends_on` 項目：

```yaml
depends_on:
    - mysql
    - redis
    - selenium
```

最後，您可以透過啟動 Sail 並執行 `dusk` 指令來執行 Dusk 測試套件：

```shell
sail dusk
```


<a name="selenium-on-apple-silicon"></a>
#### Apple Silicon 上的 Selenium

如果您的本地電腦使用 Apple Silicon 晶片，則您的 `selenium` 服務必須使用 `selenium/standalone-chromium` 映像檔：

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

Laravel Sail 預設的 `compose.yaml` 檔案包含 [Mailpit](https://github.com/axllent/mailpit) 的服務項目。Mailpit 會在本地開發期間攔截由您的應用程式發送的電子郵件，並提供便捷的 Web 介面，以便您可以在瀏覽器中預覽電子郵件訊息。使用 Sail 時，Mailpit 的預設主機名稱為 `mailpit`，並透過通訊埠 1025 提供服務：

```ini
MAIL_HOST=mailpit
MAIL_PORT=1025
MAIL_ENCRYPTION=null
```

當 Sail 正在運行時，您可以造訪 http://localhost:8025 來開啟 Mailpit 的 Web 介面。


<a name="sail-container-cli"></a>
## 容器 CLI

有時您可能想要在應用程式的容器內啟動 Bash 工作階段 (Session)。您可以使用 `shell` 指令來連線至應用程式容器，以檢視其檔案與已安裝的服務，並在容器內執行任意 Shell 指令：

```shell
sail shell

sail root-shell
```

若要啟動新的 [Laravel Tinker](https://github.com/laravel/tinker) 工作階段，您可以執行 `tinker` 指令：

```shell
sail tinker
```


<a name="sail-php-versions"></a>
## PHP 版本

Sail 目前支援透過 PHP 8.5、8.4、8.3、8.2、8.1 或 PHP 8.0 來提供應用程式服務。目前 Sail 預設使用的 PHP 版本為 PHP 8.5。若要變更用於提供應用程式服務的 PHP 版本，您應該更新應用程式 `compose.yaml` 檔案中 `laravel.test` 容器的 `build` 定義：

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

此外，您可能也希望更新 `image` 名稱以反映應用程式所使用的 PHP 版本。這個選項也是在應用程式的 `compose.yaml` 檔案中定義：

```yaml
image: sail-8.2/app
```

在更新應用程式的 `compose.yaml` 檔案後，您應該重建容器映像檔：

```shell
sail build --no-cache

sail up
```


<a name="sail-php-extensions"></a>
### 額外的 PHP 擴充功能

Sail 的執行階段 (Runtime) 映像檔包含了常用的 PHP 擴充功能集合。如果您的應用程式需要額外的擴充功能，可以在建立映像檔時，透過在應用程式的 `compose.yaml` 檔案中的 `laravel.test` 服務加入以空白分隔的 `PHP_EXTENSIONS` 建置引數來安裝它們：

```yaml
build:
    args:
        WWWGROUP: '${WWWGROUP}'
        PHP_EXTENSIONS: 'gmp imagick'
```

更新應用程式的 `compose.yaml` 檔案後，您應該重建容器映像檔。


<a name="sail-node-versions"></a>
## Node 版本

Sail 預設會安裝 Node 24。若要變更建置映像檔時所安裝的 Node 版本，您可以更新應用程式 `compose.yaml` 檔案中 `laravel.test` 服務的 `build.args` 定義：

```yaml
build:
    args:
        WWWGROUP: '${WWWGROUP}'
        NODE_VERSION: '18'
```

在更新應用程式的 `compose.yaml` 檔案後，您應該重建容器映像檔：

```shell
sail build --no-cache

sail up
```


<a name="sharing-your-site"></a>
## 分享您的網站

有時您可能需要公開分享您的網站，以便向同事預覽您的網站或測試應用程式的 Webhook 整合。若要分享您的網站，您可以使用 `share` 指令。執行此指令後，您將獲得一個隨機的 `laravel-sail.site` URL，可用來造訪您的應用程式：

```shell
sail share
```

當透過 `share` 指令分享您的網站時，您應該在應用程式的 `bootstrap/app.php` 檔案中使用 `trustProxies` 中介層方法來設定應用程式受信任的 Proxy。否則，例如 `url` 與 `route` 等 URL 產生輔助函式將無法確定產生 URL 時應使用的正確 HTTP 主機：

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
> `share` 指令由 [BeyondCode](https://beyondco.de) 開發的開放原始碼通道服務 [Expose](https://github.com/beyondcode/expose) 提供支援。

<a name="debugging-with-xdebug"></a>
## 使用 Xdebug 進行偵錯

Laravel Sail 的 Docker 設定包含對 [Xdebug](https://xdebug.org/) 的支援，這是一個熱門且強大的 PHP 偵錯工具。若要啟用 Xdebug，請確保您已[發布您的 Sail 設定](#sail-customization)。然後，將以下變數新增至應用程式的 `.env` 檔案中以設定 Xdebug：

```ini
SAIL_XDEBUG_MODE=develop,debug,coverage
```

接下來，確保發布的 `php.ini` 檔案中包含以下設定，以便在指定的模式下啟用 Xdebug：

```ini
[xdebug]
xdebug.mode=${XDEBUG_MODE}
```

修改 `php.ini` 檔案後，請記得重建 Docker 映像檔，讓對 `php.ini` 檔案的變更生效：

```shell
sail build --no-cache
```

#### Linux 主機 IP 設定

在內部，`XDEBUG_CONFIG` 環境變數被定義為 `client_host=host.docker.internal`，如此一來，Xdebug 就能為 Mac 與 Windows (WSL2) 做好正確設定。如果您的本機電腦執行的是 Linux 且使用 Docker 20.10+，即可使用 `host.docker.internal`，無須進行手動設定。

對於早於 20.10 的 Docker 版本，Linux 上並不支援 `host.docker.internal`，因此您需要手動定義主機 IP。若要執行此操作，請透過在 `compose.yaml` 檔案中定義自訂網路，為您的容器設定靜態 IP：

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

設定好靜態 IP 後，請在應用程式的 .env 檔案中定義 SAIL_XDEBUG_CONFIG 變數：

```ini
SAIL_XDEBUG_CONFIG="client_host=172.20.0.2"
```

<a name="xdebug-cli-usage"></a>
### Xdebug CLI 用法

可以在執行 Artisan 指令時使用 `sail debug` 指令來啟動偵錯工作階段：

```shell
# Run an Artisan command without Xdebug...
sail artisan migrate

# Run an Artisan command with Xdebug...
sail debug migrate
```

<a name="xdebug-browser-usage"></a>
### Xdebug 瀏覽器用法

若要在透過 Web 瀏覽器與應用程式互動時對其進行偵錯，請遵循 [Xdebug 提供的說明](https://xdebug.org/docs/step_debug#web-application)，從 Web 瀏覽器發起 Xdebug 工作階段。

如果您使用的是 PhpStorm，請參閱 JetBrains 關於[零設定偵錯 (zero-configuration debugging)](https://www.jetbrains.com/help/phpstorm/zero-configuration-debugging.html) 的說明文件。

> [!WARNING]
> Laravel Sail 依賴 `artisan serve` 來提供應用程式服務。`artisan serve` 指令自 Laravel 版本 8.53.0 起才開始接受 `XDEBUG_CONFIG` 與 `XDEBUG_MODE` 變數。較舊版本的 Laravel（8.52.0 及以下）不支援這些變數，且不會接受偵錯連線。

<a name="sail-customization"></a>
## 客製化

由於 Sail 本質上就是 Docker，您可以自由地客製化幾乎所有內容。若要發布 Sail 自己的 Dockerfile，您可以執行 `sail:publish` 指令：

```shell
sail artisan sail:publish
```

執行此指令後，Laravel Sail 所使用的 Dockerfile 和其他設定檔將會置於應用程式根目錄下的 `docker` 目錄中。在客製化您的 Sail 安裝之後，您可能希望在應用程式的 `compose.yaml` 檔案中修改應用程式容器的映像檔名稱。完成之後，請使用 `build` 指令重建您應用程式的容器。如果您在單一機器上使用 Sail 開發多個 Laravel 應用程式，為應用程式映像檔指定唯一的名稱特別重要：

```shell
sail build --no-cache
```