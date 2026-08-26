# Laravel Octane

- [簡介](#introduction)
- [安裝](#installation)
- [伺服器前置需求](#server-prerequisites)
    - [FrankenPHP](#frankenphp)
    - [RoadRunner](#roadrunner)
    - [Swoole](#swoole)
- [運行您的應用程式](#serving-your-application)
    - [透過 HTTPS 運行您的應用程式](#serving-your-application-via-https)
    - [透過 Nginx 運行您的應用程式](#serving-your-application-via-nginx)
    - [監聽檔案變更](#watching-for-file-changes)
    - [指定 Worker 數量](#specifying-the-worker-count)
    - [指定最大請求數](#specifying-the-max-request-count)
    - [指定最長執行時間](#specifying-the-max-execution-time)
    - [重新載入 Worker](#reloading-the-workers)
    - [停止伺服器](#stopping-the-server)
- [依賴注入與 Octane](#dependency-injection-and-octane)
    - [容器注入](#container-injection)
    - [請求注入](#request-injection)
    - [設定儲存庫注入](#configuration-repository-injection)
- [管理記憶體洩漏](#managing-memory-leaks)
- [同時執行的任務](#concurrent-tasks)
- [Ticks 與 Intervals](#ticks-and-intervals)
- [Octane 快取](#the-octane-cache)
    - [快取時間間隔](#cache-intervals)
- [表格](#tables)

<a name="introduction"></a>
## 簡介

[Laravel Octane](https://github.com/laravel/octane) 透過使用高效能的應用程式伺服器（包括 [FrankenPHP](https://frankenphp.dev/)、[Open Swoole](https://openswoole.com/)、[Swoole](https://github.com/swoole/swoole-src) 與 [RoadRunner](https://roadrunner.dev)）來運行您的應用程式，極大地提升了應用程式的效能。Octane 只需要啟動您的應用程式一次，並將其保留在記憶體中，接著便能以超音速般的速度處理傳入的請求。

<a name="installation"></a>
## 安裝

Octane 可以透過 Composer 套件包管理器來安裝：

```shell
composer require laravel/octane
```

安裝 Octane 後，您可以執行 `octane:install` Artisan 指令，這會將 Octane 的設定檔安裝到您的應用程式中：

```shell
php artisan octane:install
```

<a name="server-prerequisites"></a>
## 伺服器前置需求

<a name="frankenphp"></a>
### FrankenPHP

[FrankenPHP](https://frankenphp.dev) 是一個由 Go 語言編寫的 PHP 應用程式伺服器，支援 Early Hints、Brotli 與 Zstandard 壓縮等現代 Web 功能。當您安裝 Octane 並選擇 FrankenPHP 作為您的伺服器時，Octane 會自動為您下載並安裝 FrankenPHP 二進位檔。

<a name="frankenphp-via-laravel-sail"></a>
#### 透過 Laravel Sail 使用 FrankenPHP

若您打算使用 [Laravel Sail](/docs/{{version}}/sail) 開發您的應用程式，您應該執行以下命令來安裝 Octane 與 FrankenPHP：

```shell
./vendor/bin/sail up

./vendor/bin/sail composer require laravel/octane
```

接下來，您應該使用 `octane:install` Artisan 命令來安裝 FrankenPHP 二進位檔：

```shell
./vendor/bin/sail artisan octane:install --server=frankenphp
```

最後，在應用程式的 `docker-compose.yml` 檔案中的 `laravel.test` 服務定義內新增 `SUPERVISOR_PHP_COMMAND` 環境變數。這個環境變數將包含 Sail 用來使用 Octane 運行應用程式的命令，而不是使用 PHP 開發伺服器：

```yaml
services:
  laravel.test:
    environment:
      SUPERVISOR_PHP_COMMAND: "/usr/bin/php -d variables_order=EGPCS /var/www/html/artisan octane:start --server=frankenphp --host=0.0.0.0 --admin-port=2019 --port='${APP_PORT:-80}'" # [tl! add]
      XDG_CONFIG_HOME:  /var/www/html/config # [tl! add]
      XDG_DATA_HOME:  /var/www/html/data # [tl! add]
```

若要啟用 HTTPS、HTTP/2 與 HTTP/3，請改用以下修改：

```yaml
services:
  laravel.test:
    ports:
        - '${APP_PORT:-80}:80'
        - '${VITE_PORT:-5173}:${VITE_PORT:-5173}'
        - '443:443' # [tl! add]
        - '443:443/udp' # [tl! add]
    environment:
      SUPERVISOR_PHP_COMMAND: "/usr/bin/php -d variables_order=EGPCS /var/www/html/artisan octane:start --host=localhost --port=443 --admin-port=2019 --https" # [tl! add]
      XDG_CONFIG_HOME:  /var/www/html/config # [tl! add]
      XDG_DATA_HOME:  /var/www/html/data # [tl! add]
```

通常，您應該透過 `https://localhost` 來存取您的 FrankenPHP Sail 應用程式，因為使用 `https://127.0.0.1` 需要額外的設定，且[不被建議](https://frankenphp.dev/docs/known-issues/#using-https127001-with-docker)。

<a name="frankenphp-via-docker"></a>
#### 透過 Docker 使用 FrankenPHP

使用 FrankenPHP 的官方 Docker 映像檔可以提供更好的效能，並支援 FrankenPHP 靜態安裝檔中未包含的額外擴充功能。此外，官方 Docker 映像檔還支援在原生不支援的平台（例如 Windows）上執行 FrankenPHP。FrankenPHP 的官方 Docker 映像檔同時適用於本機開發與正式環境。

您可以使用以下的 Dockerfile 作為起點，將基於 FrankenPHP 運作的 Laravel 應用程式進行容器化：

```dockerfile
FROM dunglas/frankenphp

RUN install-php-extensions \
    pcntl
    # Add other PHP extensions here...

COPY . /app

ENTRYPOINT ["php", "artisan", "octane:frankenphp"]
```

接著，在開發過程中，您可以使用以下的 Docker Compose 檔案來執行您的應用程式：

```yaml
# compose.yaml
services:
  frankenphp:
    build:
      context: .
    entrypoint: php artisan octane:frankenphp --workers=1 --max-requests=1
    ports:
      - "8000:8000"
    volumes:
      - .:/app
```

若向 `php artisan octane:start` 命令明確傳遞了 `--log-level` 選項，Octane 將會使用 FrankenPHP 的原生日誌記錄器，並且除非有其他設定，否則會產出結構化的 JSON 日誌。

您可以參閱 [FrankenPHP 官方文件](https://frankenphp.dev/docs/docker/)以取得更多關於透過 Docker 執行 FrankenPHP 的詳細資訊。

<a name="frankenphp-caddyfile"></a>
#### 自訂 Caddyfile 設定

在使用 FrankenPHP 時，您可以在啟動 Octane 時使用 `--caddyfile` 選項來指定自訂的 Caddyfile：

```shell
php artisan octane:start --server=frankenphp --caddyfile=/path/to/your/Caddyfile
```

這讓您可以超越預設設定來自訂 FrankenPHP 的配置，例如加入自訂中介層、設定進階路由或設置自訂指令。您可以參閱 [Caddy 官方文件](https://caddyserver.com/docs/caddyfile)以取得更多關於 Caddyfile 語法與設定選項的詳細資訊。

<a name="roadrunner"></a>
### RoadRunner

[RoadRunner](https://roadrunner.dev) 由 Go 語言所建置的 RoadRunner 二進位檔提供動力。當您第一次啟動基於 RoadRunner 的 Octane 伺服器時，Octane 會提示是否為您下載並安裝 RoadRunner 二進位檔。

<a name="roadrunner-via-laravel-sail"></a>
#### 透過 Laravel Sail 使用 RoadRunner

若您打算使用 [Laravel Sail](/docs/{{version}}/sail) 開發您的應用程式，您應該執行以下命令來安裝 Octane 與 RoadRunner：

```shell
./vendor/bin/sail up

./vendor/bin/sail composer require laravel/octane spiral/roadrunner-cli spiral/roadrunner-http
```

接下來，您應該啟動 Sail shell 並使用 `rr` 可執行檔來取得最新的 Linux 版 RoadRunner 二進位檔：

```shell
./vendor/bin/sail shell

# Within the Sail shell...
./vendor/bin/rr get-binary
```

然後，在應用程式的 `docker-compose.yml` 檔案中的 `laravel.test` 服務定義內新增 `SUPERVISOR_PHP_COMMAND` 環境變數。這個環境變數將包含 Sail 用來使用 Octane 運行應用程式的命令，而不是使用 PHP 開發伺服器：

```yaml
services:
  laravel.test:
    environment:
      SUPERVISOR_PHP_COMMAND: "/usr/bin/php -d variables_order=EGPCS /var/www/html/artisan octane:start --server=roadrunner --host=0.0.0.0 --rpc-port=6001 --port='${APP_PORT:-80}'" # [tl! add]
```

最後，請確保 `rr` 二進位檔為可執行狀態並建置您的 Sail 映像檔：

```shell
chmod +x ./rr

./vendor/bin/sail build --no-cache
```

<a name="swoole"></a>
### Swoole

如果您打算使用 Swoole 應用程式伺服器來運行您的 Laravel Octane 應用程式，則必須安裝 Swoole PHP 擴充套件。通常可以透過 PECL 來安裝：

```shell
pecl install swoole
```

<a name="openswoole"></a>
#### Open Swoole

如果您想使用 Open Swoole 應用程式伺服器來運行您的 Laravel Octane 應用程式，您必須安裝 Open Swoole PHP 擴充套件。通常可以透過 PECL 來安裝：

```shell
pecl install openswoole
```

搭配 Open Swoole 使用 Laravel Octane 可以獲得與 Swoole 相同的完整功能，例如同時執行的任務 (concurrent tasks)、ticks 以及 intervals。

<a name="swoole-via-laravel-sail"></a>
#### 透過 Laravel Sail 使用 Swoole

> [!WARNING]
> 透過 Sail 運行 Octane 應用程式前，請確保您擁有最新版本的 Laravel Sail，並在您應用程式的根目錄中執行 `./vendor/bin/sail build --no-cache`。

另外，您也可以使用 [Laravel Sail](/docs/{{version}}/sail) 來開發基於 Swoole 的 Octane 應用程式，這是 Laravel 官方基於 Docker 的開發環境。Laravel Sail 預設已包含 Swoole 擴充套件。不過，您仍需要調整 Sail 所使用的 `docker-compose.yml` 檔案。

首先，請在您應用程式的 `docker-compose.yml` 檔案中，為 `laravel.test` 服務定義新增一個 `SUPERVISOR_PHP_COMMAND` 環境變數。這個環境變數包含 Sail 用來使用 Octane (而非 PHP 開發伺服器) 來運行應用程式的命令：

```yaml
services:
  laravel.test:
    environment:
      SUPERVISOR_PHP_COMMAND: "/usr/bin/php -d variables_order=EGPCS /var/www/html/artisan octane:start --server=swoole --host=0.0.0.0 --port='${APP_PORT:-80}'" # [tl! add]
```

最後，建置您的 Sail 映像檔：

```shell
./vendor/bin/sail build --no-cache
```

<a name="swoole-configuration"></a>
#### Swoole 設定

Swoole 支援一些額外的設定選項，如果有需要，您可以將這些選項新增至 `octane` 設定檔中。因為極少需要修改它們，所以這些選項並未包含在預設的設定檔中：

```php
'swoole' => [
    'options' => [
        'log_file' => storage_path('logs/swoole_http.log'),
        'package_max_length' => 10 * 1024 * 1024,
    ],
],
```

<a name="serving-your-application"></a>
## 運行您的應用程式

Octane 伺服器可以透過 `octane:start` Artisan 指令啟動。預設情況下，此指令會使用您應用程式的 `octane` 設定檔中 `server` 設定選項所指定的伺服器：

```shell
php artisan octane:start
```

預設情況下，Octane 會在 Port 8000 啟動伺服器，因此您可以透過瀏覽器存取 `http://localhost:8000` 來存取您的應用程式。

<a name="keeping-octane-running-in-production"></a>
#### 在正式環境中保持 Octane 持續運行

如果您要將 Octane 應用程式部署到正式環境 (Production)，您應該使用像是 Supervisor 的行程(Processes)監控工具來確保 Octane 伺服器保持運行。Octane 的 Supervisor 設定檔範例如下：

```ini
[program:octane]
process_name=%(program_name)s_%(process_num)02d
command=php /home/forge/example.com/artisan octane:start --server=frankenphp --host=127.0.0.1 --port=8000
autostart=true
autorestart=true
user=forge
redirect_stderr=true
stdout_logfile=/home/forge/example.com/storage/logs/octane.log
stopwaitsecs=3600
```

<a name="serving-your-application-via-https"></a>
### 透過 HTTPS 運行您的應用程式

預設情況下，透過 Octane 運行的應用程式產生的連結前綴為 `http://`。當透過 HTTPS 運行您的應用程式時，可以在應用程式的 `config/octane.php` 設定檔中將 `OCTANE_HTTPS` 環境變數設定為 `true`。當此設定值設為 `true` 時，Octane 會指示 Laravel 為所有產生的連結加上 `https://` 前綴：

```php
'https' => env('OCTANE_HTTPS', false),
```

<a name="serving-your-application-via-nginx"></a>
### 透過 Nginx 運行您的應用程式

> [!NOTE]
> 如果您尚未準備好自行管理伺服器設定，或是對設定運行穩健的 Laravel Octane 應用程式所需的所有各類服務不夠熟悉，可以參考 [Laravel Cloud](https://cloud.laravel.com)，它提供完全託管的 Laravel Octane 支援。

在正式環境中，您應該將 Octane 應用程式放在傳統 Web 伺服器（如 Nginx 或 Apache）後方運作。這樣做可以讓 Web 伺服器負責提供圖片和樣式表等靜態資源，並處理 SSL 憑證終止 (SSL Certificate Termination)。

在下方的 Nginx 設定範例中，Nginx 會提供網站的靜態資源，並將請求代理轉發給運行在 Port 8000 的 Octane 伺服器：

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

server {
    listen 80;
    listen [::]:80;
    server_name domain.com;
    server_tokens off;
    root /home/forge/domain.com/public;

    index index.php;

    charset utf-8;

    location /index.php {
        try_files /not_exists @octane;
    }

    location / {
        try_files $uri $uri/ @octane;
    }

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    access_log off;
    error_log  /var/log/nginx/domain.com-error.log error;

    error_page 404 /index.php;

    location @octane {
        set $suffix "";

        if ($uri = /index.php) {
            set $suffix ?$query_string;
        }

        proxy_http_version 1.1;
        proxy_set_header Host $http_host;
        proxy_set_header Scheme $scheme;
        proxy_set_header SERVER_PORT $server_port;
        proxy_set_header REMOTE_ADDR $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;

        proxy_pass http://127.0.0.1:8000$suffix;
    }
}
```

<a name="watching-for-file-changes"></a>
### 監聽檔案變更

由於您的應用程式在 Octane 伺服器啟動時會載入記憶體中一次，因此當您重新整理瀏覽器時，任何對應用程式檔案進行的變更都不會立即反映出來。例如，新增到 `routes/web.php` 檔案中的路由定義要到重啟伺服器後才會生效。為了方便開發，您可以傳入 `--watch` 旗標來指示 Octane 在您應用程式內的任何檔案發生變更時自動重啟伺服器：

```shell
php artisan octane:start --watch
```

在使用此功能前，您應確保本機開發環境中已安裝 [Node](https://nodejs.org)。此外，您需要在專案中安裝 [Chokidar](https://github.com/paulmillr/chokidar) 檔案監聽套件：

```shell
npm install --save-dev chokidar
```

您可以透過應用程式的 `config/octane.php` 設定檔中的 `watch` 設定選項來指定需要監聽的目錄與檔案。

<a name="specifying-the-worker-count"></a>
### 指定 Worker 數量

預設情況下，Octane 會為您的機器所提供的每一個 CPU 核心啟動一個應用程式請求 Worker。這些 Worker 將用於處理進入您應用程式的 HTTP 請求。在執行 `octane:start` 指令時，您可以透過 `--workers` 選項手動指定要啟動多少個 Worker：

```shell
php artisan octane:start --workers=4
```

如果您使用的是 Swoole 應用程式伺服器，您還可以指定要啟動多少個 [「Task Worker」](#concurrent-tasks)：

```shell
php artisan octane:start --workers=4 --task-workers=6
```

<a name="specifying-the-max-request-count"></a>
### 指定最大請求數

為了防止記憶體洩漏 (Memory Leak)，Octane 會在 Worker 處理完 500 個請求後平滑重啟 (Gracefully restart) 該 Worker。若要調整此數字，可以使用 `--max-requests` 選項：

```shell
php artisan octane:start --max-requests=250
```

<a name="specifying-the-max-execution-time"></a>
### 指定最長執行時間

預設情況下，Laravel Octane 透過應用程式 `config/octane.php` 設定檔中的 `max_execution_time` 選項，為傳入的請求設定了 30 秒的最長執行時間：

```php
'max_execution_time' => 30,
```

此設定定義了傳入的請求在被終止前允許執行的最長秒數。將此值設為 `0` 將會完全停用執行時間限制。此設定選項對於處理耗時較長請求（例如檔案上傳、資料處理或對外部服務的 API 呼叫）的應用程式特別有用。

> [!WARNING]
> 當您修改 `max_execution_time` 設定時，必須重啟 Octane 伺服器才能使變更生效。

<a name="reloading-the-workers"></a>
### 重新載入 Worker

您可以透過 `octane:reload` 指令來平滑重啟 Octane 伺服器的應用程式 Worker。通常，這應該在部署後執行，以便將您最新部署的程式碼載入記憶體中並用於處理後續的請求：

```shell
php artisan octane:reload
```

<a name="stopping-the-server"></a>
### 停止伺服器

您可以透過 `octane:stop` Artisan 指令來停止 Octane 伺服器：

```shell
php artisan octane:stop
```

<a name="checking-the-server-status"></a>
#### 檢查伺服器狀態

您可以透過 `octane:status` Artisan 指令來檢查 Octane 伺服器的目前狀態：

```shell
php artisan octane:status
```

<a name="dependency-injection-and-octane"></a>
## 依賴注入與 Octane

由於 Octane 只會啟動您的應用程式一次並將其保持在記憶體中來處理請求，因此在開發應用程式時，有幾個注意事項需要考量。例如，應用程式的服務提供者(Service Providers)中的 `register` 與 `boot` 方法只會在 request worker 初次啟動時執行一次。在後續的請求中，將會重複使用相同的應用程式實例。

鑑於這一點，在將應用程式服務容器或請求注入到任何物件的建構子 (Constructor) 時應特別小心。這樣做可能會導致該物件在後續請求中持有過期版本的容器或請求。

Octane 會自動處理在請求之間重置任何第一方框架的狀態。然而，Octane 並不總是知道如何重置由您的應用程式所建立的全域狀態。因此，您應該瞭解如何以對 Octane 友善的方式來構建應用程式。下面我們將討論在使用 Octane 時最常造成問題的幾種情況。

<a name="container-injection"></a>
### 容器注入

一般來說，您應該避免將應用程式服務容器或 HTTP 請求實例注入到其他物件的建構子中。例如，以下綁定將整個應用程式服務容器注入到一個被綁定為單例 (Singleton) 的物件中：

```php
use App\Service;
use Illuminate\Contracts\Foundation\Application;

/**
 * Register any application services.
 */
public function register(): void
{
    $this->app->singleton(Service::class, function (Application $app) {
        return new Service($app);
    });
}
```

在此範例中，若 `Service` 實例是在應用程式啟動流程期間解析的，容器將會被注入到該服務中，且該 `Service` 實例會在後續請求中繼續持有同一個容器。這對您的特定應用程式來說**可能**不是問題；然而，這可能導致容器意外遺失了在啟動週期後期或後續請求中所新增的綁定。

作為替代方案，您可以停止將該綁定註冊為單例，或者您可以向服務注入一個容器解析器閉包 (Closure)，讓它總是解析出目前的容器實例：

```php
use App\Service;
use Illuminate\Container\Container;
use Illuminate\Contracts\Foundation\Application;

$this->app->bind(Service::class, function (Application $app) {
    return new Service($app);
});

$this->app->singleton(Service::class, function () {
    return new Service(fn () => Container::getInstance());
});
```

全域的 `app` 輔助函式與 `Container::getInstance()` 方法將總是傳回最新版本的應用程式容器。

<a name="request-injection"></a>
### 請求注入

一般來說，您應該避免將應用程式服務容器或 HTTP 請求實例注入到其他物件的建構子中。例如，以下綁定將整個請求實例注入到一個被綁定為單例的物件中：

```php
use App\Service;
use Illuminate\Contracts\Foundation\Application;

/**
 * Register any application services.
 */
public function register(): void
{
    $this->app->singleton(Service::class, function (Application $app) {
        return new Service($app['request']);
    });
}
```

在此範例中，若 `Service` 實例是在應用程式啟動流程期間解析的，HTTP 請求將會被注入到該服務中，且該 `Service` 實例會在後續請求中繼續持有同一個請求。因此，所有的標頭 (Header)、輸入與查詢字串資料，以及所有其他的請求資料都會是錯誤的。

作為替代方案，您可以停止將該綁定註冊為單例，或者您可以向服務注入一個請求解析器閉包，讓它總是解析出目前的請求實例。或是最推薦的做法，是在執行階段單純地將您物件所需的特定請求資訊傳遞給該物件的其中一個方法：

```php
use App\Service;
use Illuminate\Contracts\Foundation\Application;

$this->app->bind(Service::class, function (Application $app) {
    return new Service($app['request']);
});

$this->app->singleton(Service::class, function (Application $app) {
    return new Service(fn () => $app['request']);
});

// Or...

$service->method($request->input('name'));
```

全域的 `request` 輔助函式將總是傳回應用程式目前正在處理的請求，因此在您的應用程式中使用它是安全的。

> [!WARNING]
> 在您的 Controller 方法與路由閉包上型別提示 (Type-hint) `Illuminate\Http\Request` 實例是可以接受的。

<a name="configuration-repository-injection"></a>
### 設定儲存庫注入

一般來說，您應該避免將設定儲存庫 (Configuration Repository) 實例注入到其他物件的建構子中。例如，以下綁定將設定儲存庫注入到一個被綁定為單例的物件中：

```php
use App\Service;
use Illuminate\Contracts\Foundation\Application;

/**
 * Register any application services.
 */
public function register(): void
{
    $this->app->singleton(Service::class, function (Application $app) {
        return new Service($app->make('config'));
    });
}
```

在此範例中，若設定值在請求之間發生變更，該服務將無法存取新的數值，因為它依賴的是原本的儲存庫實例。

作為替代方案，您可以停止將該綁定註冊為單例，或者您可以向該類別注入一個設定儲存庫解析器閉包：

```php
use App\Service;
use Illuminate\Container\Container;
use Illuminate\Contracts\Foundation\Application;

$this->app->bind(Service::class, function (Application $app) {
    return new Service($app->make('config'));
});

$this->app->singleton(Service::class, function () {
    return new Service(fn () => Container::getInstance()->make('config'));
});
```

全域的 `config` 將總是傳回最新版本的設定儲存庫，因此在您的應用程式中使用它是安全的。

<a name="managing-memory-leaks"></a>
### 管理記憶體洩漏

請記住，Octane 會在請求之間將您的應用程式保留在記憶體中；因此，將資料新增至靜態維護的陣列中將導致記憶體洩漏。例如，以下 Controller 存在記憶體洩漏，因為對應用程式的每次請求都會持續將資料新增至靜態的 `$data` 陣列中：

```php
use App\Service;
use Illuminate\Http\Request;
use Illuminate\Support\Str;

/**
 * Handle an incoming request.
 */
public function index(Request $request): array
{
    Service::$data[] = Str::random(10);

    return [
        // ...
    ];
}
```

在構建應用程式時，您應該特別注意避免建立此類型的記憶體洩漏。建議您在本地開發期間監控應用程式的記憶體使用情況，以確保沒有為應用程式引入新的記憶體洩漏。

<a name="concurrent-tasks"></a>
## 同時執行的任務

> [!WARNING]
> 此功能需要 [Swoole](#swoole)。

使用 Swoole 時，您可以透過輕量級的背景任務同時執行多個操作。您可以利用 Octane 的 `concurrently` 方法來實現。您可以將此方法與 PHP 陣列解構 (Array Destructuring) 結合使用，以取得各個操作的結果：

```php
use App\Models\User;
use App\Models\Server;
use Laravel\Octane\Facades\Octane;

[$users, $servers] = Octane::concurrently([
    fn () => User::all(),
    fn () => Server::all(),
]);
```

由 Octane 處理的同時執行任務利用了 Swoole 的「task worker」，並在與傳入請求完全不同的行程 (Process) 中執行。可用於處理同時執行任務的 worker 數量由 `octane:start` 命令上的 `--task-workers` 指令決定：

```shell
php artisan octane:start --workers=4 --task-workers=6
```

呼叫 `concurrently` 方法時，受限於 Swoole 任務系統的限制，您所提供的任務數量不應超過 1024 個。

<a name="ticks-and-intervals"></a>
## Ticks 與 Intervals

> [!WARNING]
> 此功能需要 [Swoole](#swoole)。

當使用 Swoole 時，您可以註冊每隔指定秒數執行的「tick」操作。您可以透過 `tick` 方法註冊「tick」回呼函式。提供給 `tick` 方法的第一個引數應該是代表該 ticker 名稱的字串。第二個引數則應該是在指定時間間隔時被呼叫的可呼叫物件 (callable)。

在此範例中，我們將註冊一個每 10 秒被呼叫一次的閉包。通常，`tick` 方法應該在您的應用程式服務提供者(Service Providers)中的 `boot` 方法內呼叫：

```php
Octane::tick('simple-ticker', fn () => ray('Ticking...'))
    ->seconds(10);
```

使用 `immediate` 方法，您可以指示 Octane 在 Octane 伺服器初次啟動時立即執行 tick 回呼，之後則每隔 N 秒執行一次：

```php
Octane::tick('simple-ticker', fn () => ray('Ticking...'))
    ->seconds(10)
    ->immediate();
```


<a name="the-octane-cache"></a>
## Octane 快取

> [!WARNING]
> 此功能需要 [Swoole](#swoole)。

使用 Swoole 時，您可以利用 Octane 快取驅動程式，它每秒提供高達 200 萬次操作的讀寫速度。因此，對於快取層需要極致讀寫速度的應用程式來說，此快取驅動程式是絕佳選擇。

此快取驅動程式由 [Swoole tables](https://www.swoole.co.uk/docs/modules/swoole-table) 提供支援。儲存在快取中的所有資料對伺服器上的所有 worker 都是可用的。然而，當伺服器重新啟動時，快取資料將會被清空：

```php
Cache::store('octane')->put('framework', 'Laravel', 30);
```

> [!NOTE]
> Octane 快取中允許的最大項目數量可以在您應用程式的 `octane` 設定檔中定義。


<a name="cache-intervals"></a>
### 快取時間間隔

除了 Laravel 快取系統提供的典型方法之外，Octane 快取驅動程式還具備基於時間間隔的快取功能。這些快取會在指定的時間間隔自動重新整理，並應在您應用程式的服務提供者(Service Providers)中的 `boot` 方法內進行註冊。例如，以下快取將每五秒重新整理一次：

```php
use Illuminate\Support\Str;

Cache::store('octane')->interval('random', function () {
    return Str::random(10);
}, seconds: 5);
```


<a name="tables"></a>
## 表格

> [!WARNING]
> 此功能需要 [Swoole](#swoole)。

當使用 Swoole 時，您可以定義並操作自訂的任意 [Swoole tables](https://www.swoole.co.uk/docs/modules/swoole-table)。Swoole tables 提供極高的效能吞吐量，且這些表格中的資料可以被伺服器上的所有 worker 存取。不過，當伺服器重新啟動時，其中的資料將會遺失。

表格應該定義在您應用程式的 `octane` 設定檔中的 `tables` 設定陣列內。系統已為您預設設定了一個最多允許 1000 列的範例表格。字串欄位的最大尺寸可以透過在型態後方指定欄位大小來設定，如下所示：

```php
'tables' => [
    'example:1000' => [
        'name' => 'string:1000',
        'votes' => 'int',
    ],
],
```

要存取表格，您可以使用 `Octane::table` 方法：

```php
use Laravel\Octane\Facades\Octane;

Octane::table('example')->set('uuid', [
    'name' => 'Nuno Maduro',
    'votes' => 1000,
]);

return Octane::table('example')->get('uuid');
```

> [!WARNING]
> Swoole tables 支援的欄位型態有：`string`、`int` 與 `float`。