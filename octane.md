# Laravel Octane

- [簡介](#introduction)
- [安裝](#installation)
- [伺服器先決條件](#server-prerequisites)
    - [FrankenPHP](#frankenphp)
    - [RoadRunner](#roadrunner)
    - [Swoole](#swoole)
- [提供應用程式](#serving-your-application)
    - [透過 HTTPS 提供應用程式](#serving-your-application-via-https)
    - [透過 Nginx 提供應用程式](#serving-your-application-via-nginx)
    - [監控檔案變更](#watching-for-file-changes)
    - [指定 Worker 數量](#specifying-the-worker-count)
    - [指定最大請求數量](#specifying-the-max-request-count)
    - [指定最大執行時間](#specifying-the-max-execution-time)
    - [重載 Worker](#reloading-the-workers)
    - [停止伺服器](#stopping-the-server)
- [依賴注入與 Octane](#dependency-injection-and-octane)
    - [容器注入](#container-injection)
    - [請求注入](#request-injection)
    - [設定儲存庫注入](#configuration-repository-injection)
- [管理記憶體洩漏](#managing-memory-leaks)
- [並行任務](#concurrent-tasks)
- [計時與間隔](#ticks-and-intervals)
- [Octane 快取](#the-octane-cache)
- [資料表](#tables)

<a name="introduction"></a>
## 簡介

[Laravel Octane](https://github.com/laravel/octane) 透過使用高效能應用程式伺服器（包含 [FrankenPHP](https://frankenphp.dev/)、[Open Swoole](https://openswoole.com/)、[Swoole](https://github.com/swoole/swoole-src) 和 [RoadRunner](https://roadrunner.dev)）來提供您的應用程式，大幅提升其效能。Octane 會將您的應用程式啟動一次，將其保留在記憶體中，然後以超音速向其傳送請求。

<a name="installation"></a>
## 安裝

Octane 可以透過 Composer 套件管理器安裝：

```shell
composer require laravel/octane
```

安裝 Octane 後，您可以執行 `octane:install` Artisan 命令，這會將 Octane 的設定檔安裝到您的應用程式中：

```shell
php artisan octane:install
```

<a name="server-prerequisites"></a>
## 伺服器先決條件


<a name="frankenphp"></a>
### FrankenPHP

[FrankenPHP](https://frankenphp.dev) 是一個用 Go 語言編寫的 PHP 應用程式伺服器，它支援現代 Web 功能，例如 early hints、Brotli 和 Zstandard 壓縮。當您安裝 Octane 並選擇 FrankenPHP 作為您的伺服器時，Octane 會自動為您下載並安裝 FrankenPHP 二進位檔。


<a name="frankenphp-via-laravel-sail"></a>
#### 透過 Laravel Sail 使用 FrankenPHP

如果您計畫使用 [Laravel Sail](/docs/{{version}}/sail) 來開發您的應用程式，您應該執行以下命令來安裝 Octane 和 FrankenPHP：

```shell
./vendor/bin/sail up

./vendor/bin/sail composer require laravel/octane
```

接著，您應該使用 `octane:install` Artisan 命令來安裝 FrankenPHP 二進位檔：

```shell
./vendor/bin/sail artisan octane:install --server=frankenphp
```

最後，在您應用程式的 `docker-compose.yml` 檔案中，為 `laravel.test` 服務定義新增一個 `SUPERVISOR_PHP_COMMAND` 環境變數。此環境變數將包含 Sail 用來透過 Octane 服務您的應用程式的命令，而不是 PHP 開發伺服器：

```yaml
services:
  laravel.test:
    environment:
      SUPERVISOR_PHP_COMMAND: "/usr/bin/php -d variables_order=EGPCS /var/www/html/artisan octane:start --server=frankenphp --host=0.0.0.0 --admin-port=2019 --port='${APP_PORT:-80}'" # [tl! add]
      XDG_CONFIG_HOME:  /var/www/html/config # [tl! add]
      XDG_DATA_HOME:  /var/www/html/data # [tl! add]
```

若要啟用 HTTPS、HTTP/2 和 HTTP/3，請改為應用以下修改：

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

通常，您應該透過 `https://localhost` 存取您的 FrankenPHP Sail 應用程式，因為使用 `https://127.0.0.1` 需要額外的設定且 [不建議](https://frankenphp.dev/docs/known-issues/#using-https127001-with-docker)。


<a name="frankenphp-via-docker"></a>
#### 透過 Docker 使用 FrankenPHP

使用 FrankenPHP 的官方 Docker 映像檔可以提供更高的效能，並使用 FrankenPHP 靜態安裝中不包含的其他擴充功能。此外，官方 Docker 映像檔支援在 FrankenPHP 原生不支援的平台（例如 Windows）上執行 FrankenPHP。FrankenPHP 的官方 Docker 映像檔適用於本機開發和生產環境。

您可以將以下 Dockerfile 作為起點，用於將 FrankenPHP 驅動的 Laravel 應用程式容器化：

```dockerfile
FROM dunglas/frankenphp

RUN install-php-extensions \
    pcntl
    # Add other PHP extensions here...

COPY . /app

ENTRYPOINT ["php", "artisan", "octane:frankenphp"]
```

然後，在開發期間，您可以使用以下 Docker Compose 檔案來執行您的應用程式：

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

如果將 `--log-level` 選項明確傳遞給 `php artisan octane:start` 命令，Octane 將使用 FrankenPHP 的原生記錄器，並且（除非另行設定）將產生結構化的 JSON 記錄。

您可以查閱 [FrankenPHP 官方文件](https://frankenphp.dev/docs/docker/) 以獲取有關在 Docker 中執行 FrankenPHP 的更多資訊。


<a name="frankenphp-caddyfile"></a>
#### 自訂 Caddyfile 設定

使用 FrankenPHP 時，您可以在啟動 Octane 時使用 `--caddyfile` 選項來指定自訂的 Caddyfile：

```shell
php artisan octane:start --server=frankenphp --caddyfile=/path/to/your/Caddyfile
```

這允許您自訂 FrankenPHP 的設定，超越預設值，例如新增自訂中介層 (middleware)、設定進階路由或建立自訂指令。您可以查閱 [Caddy 官方文件](https://caddyserver.com/docs/caddyfile) 以獲取有關 Caddyfile 語法和設定選項的更多資訊。


<a name="roadrunner"></a>
### RoadRunner

[RoadRunner](https://roadrunner.dev) 由 RoadRunner 二進位檔驅動，該二進位檔使用 Go 編寫。當您首次啟動基於 RoadRunner 的 Octane 伺服器時，Octane 將會提供下載並為您安裝 RoadRunner 二進位檔。


<a name="roadrunner-via-laravel-sail"></a>
#### 透過 Laravel Sail 使用 RoadRunner

如果您計畫使用 [Laravel Sail](/docs/{{version}}/sail) 來開發您的應用程式，您應該執行以下命令來安裝 Octane 和 RoadRunner：

```shell
./vendor/bin/sail up

./vendor/bin/sail composer require laravel/octane spiral/roadrunner-cli spiral/roadrunner-http
```

接著，您應該啟動一個 Sail shell，並使用 `rr` 可執行檔來獲取 RoadRunner 二進位檔的最新 Linux 版本：

```shell
./vendor/bin/sail shell

# Within the Sail shell...
./vendor/bin/rr get-binary
```

然後，在您應用程式的 `docker-compose.yml` 檔案中，為 `laravel.test` 服務定義新增一個 `SUPERVISOR_PHP_COMMAND` 環境變數。此環境變數將包含 Sail 用來透過 Octane 服務您的應用程式的命令，而不是 PHP 開發伺服器：

```yaml
services:
  laravel.test:
    environment:
      SUPERVISOR_PHP_COMMAND: "/usr/bin/php -d variables_order=EGPCS /var/www/html/artisan octane:start --server=roadrunner --host=0.0.0.0 --rpc-port=6001 --port='${APP_PORT:-80}'" # [tl! add]
```

最後，確保 `rr` 二進位檔是可執行檔，並建構您的 Sail 映像檔：

```shell
chmod +x ./rr

./vendor/bin/sail build --no-cache
```

<a name="swoole"></a>
### Swoole

如果您打算使用 Swoole 應用程式伺服器來提供您的 Laravel Octane 應用程式，您必須安裝 Swoole PHP 擴充功能。通常，這可以透過 PECL 完成：

```shell
pecl install swoole
```

<a name="openswoole"></a>
#### Open Swoole

如果您想使用 Open Swoole 應用程式伺服器來提供您的 Laravel Octane 應用程式，您必須安裝 Open Swoole PHP 擴充功能。通常，這可以透過 PECL 完成：

```shell
pecl install openswoole
```

將 Laravel Octane 與 Open Swoole 結合使用，可獲得 Swoole 提供的相同功能，例如並行任務、計時和間隔功能。

<a name="swoole-via-laravel-sail"></a>
#### Swoole via Laravel Sail

> [!WARNING]
> 在透過 Sail 提供 Octane 應用程式之前，請確保您已擁有最新版本的 Laravel Sail，並在您的應用程式根目錄中執行 `./vendor/bin/sail build --no-cache`。

或者，您可以使用 [Laravel Sail](/docs/{{version}}/sail) 開發基於 Swoole 的 Octane 應用程式，這是 Laravel 官方基於 Docker 的開發環境。Laravel Sail 預設包含 Swoole 擴充功能。然而，您仍然需要調整 Sail 所使用的 `docker-compose.yml` 檔案。

首先，請在您的應用程式 `docker-compose.yml` 檔案中的 `laravel.test` 服務定義中新增 `SUPERVISOR_PHP_COMMAND` 環境變數。此環境變數將包含 Sail 用來透過 Octane 提供您的應用程式的指令，而不是 PHP 開發伺服器：

```yaml
services:
  laravel.test:
    environment:
      SUPERVISOR_PHP_COMMAND: "/usr/bin/php -d variables_order=EGPCS /var/www/html/artisan octane:start --server=swoole --host=0.0.0.0 --port='${APP_PORT:-80}'" # [tl! add]
```

最後，建構您的 Sail 映像檔：

```shell
./vendor/bin/sail build --no-cache
```

<a name="swoole-configuration"></a>
#### Swoole Configuration

Swoole 支援一些額外的配置選項，您可以在必要時將其新增到 `octane` 配置檔案中。由於這些選項很少需要修改，因此它們沒有包含在預設的配置檔案中：

```php
'swoole' => [
    'options' => [
        'log_file' => storage_path('logs/swoole_http.log'),
        'package_max_length' => 10 * 1024 * 1024,
    ],
],
```

<a name="serving-your-application"></a>
## 提供應用程式

Octane 伺服器可透過 `octane:start` Artisan 指令啟動。預設情況下，此指令將使用應用程式 `octane` 設定檔中 `server` 設定選項所指定的伺服器：

```shell
php artisan octane:start
```

預設情況下，Octane 會在 8000 埠啟動伺服器，因此您可以透過網頁瀏覽器經由 `http://localhost:8000` 存取您的應用程式。

<a name="keeping-octane-running-in-production"></a>
#### 在生產環境中保持 Octane 運行

如果您正在將 Octane 應用程式部署到生產環境，您應該使用諸如 Supervisor 等程序監控器來確保 Octane 伺服器持續運行。一個 Octane 的 Supervisor 範例設定檔可能如下所示：

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
### 透過 HTTPS 提供應用程式

預設情況下，透過 Octane 運行的應用程式會產生以 `http://` 為字首的連結。當您透過 HTTPS 提供應用程式時，可在應用程式的 `config/octane.php` 設定檔中，將 `OCTANE_HTTPS` 環境變數設為 `true`。當此設定值設為 `true` 時，Octane 將會指示 Laravel 以 `https://` 為所有產生的連結加上字首：

```php
'https' => env('OCTANE_HTTPS', false),
```

<a name="serving-your-application-via-nginx"></a>
### 透過 Nginx 提供應用程式

> [!NOTE]
> 如果您尚未準備好自行管理伺服器設定，或者不熟悉設定運行強固的 Laravel Octane 應用程式所需的各種服務，請參考 [Laravel Cloud](https://cloud.laravel.com)，它提供全面託管的 Laravel Octane 支援。

在生產環境中，您應該將 Octane 應用程式部署在傳統的網頁伺服器（例如 Nginx 或 Apache）之後。這樣做將允許網頁伺服器提供靜態資產（例如圖片和樣式表），並管理您的 SSL 憑證終止。

在下面的 Nginx 設定範例中，Nginx 將提供網站的靜態資產，並將請求代理到運行在 8000 埠的 Octane 伺服器：

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
### 監控檔案變更

由於您的應用程式在 Octane 伺服器啟動時僅載入到記憶體中一次，因此當您重新整理瀏覽器時，應用程式檔案的任何變更都不會反映出來。例如，新增到 `routes/web.php` 檔案中的路由定義在伺服器重新啟動前將不會生效。為方便起見，您可以使用 `--watch` 旗標指示 Octane 在應用程式中的任何檔案變更時自動重新啟動伺服器：

```shell
php artisan octane:start --watch
```

在使用此功能之前，您應確保在本地開發環境中已安裝 [Node](https://nodejs.org)。此外，您應該在專案中安裝 [Chokidar](https://github.com/paulmillr/chokidar) 檔案監控函式庫：

```shell
npm install --save-dev chokidar
```

您可以在應用程式的 `config/octane.php` 設定檔中使用 `watch` 設定選項來設定應監控的目錄與檔案。

<a name="specifying-the-worker-count"></a>
### 指定 Worker 數量

預設情況下，Octane 會為您機器提供的每個 CPU 核心啟動一個應用程式請求 Worker。這些 Worker 隨後將用於處理進入應用程式的 HTTP 請求。您可以在呼叫 `octane:start` 指令時，使用 `--workers` 選項手動指定要啟動的 Worker 數量：

```shell
php artisan octane:start --workers=4
```

如果您正在使用 Swoole 應用程式伺服器，您也可以指定要啟動的 ["任務 Worker"](#concurrent-tasks) 數量：

```shell
php artisan octane:start --workers=4 --task-workers=6
```

<a name="specifying-the-max-request-count"></a>
### 指定最大請求數量

為防止記憶體洩漏，Octane 會在任何 Worker 處理 500 個請求後優雅地重新啟動該 Worker。要調整此數量，您可以使用 `--max-requests` 選項：

```shell
php artisan octane:start --max-requests=250
```

<a name="specifying-the-max-execution-time"></a>
### 指定最大執行時間

預設情況下，Laravel Octane 透過應用程式 `config/octane.php` 設定檔中的 `max_execution_time` 選項，為傳入請求設定了 30 秒的最大執行時間：

```php
'max_execution_time' => 30,
```

此設定定義了傳入請求在終止前允許執行的最大秒數。將此值設為 `0` 將完全禁用執行時間限制。此設定選項對於處理長時間運行的請求的應用程式特別有用，例如檔案上傳、資料處理或對外部服務的 API 呼叫。

> [!WARNING]
> 當您修改 `max_execution_time` 設定時，必須重新啟動 Octane 伺服器才能使變更生效。

<a name="reloading-the-workers"></a>
### 重載 Worker

您可以使用 `octane:reload` 指令優雅地重新啟動 Octane 伺服器的應用程式 Worker。通常，這應在部署後執行，以便將您新部署的程式碼載入到記憶體中，並用於處理後續請求：

```shell
php artisan octane:reload
```

<a name="stopping-the-server"></a>
### 停止伺服器

您可以使用 `octane:stop` Artisan 指令來停止 Octane 伺服器：

```shell
php artisan octane:stop
```

<a name="checking-the-server-status"></a>
#### 檢查伺服器狀態

您可以使用 `octane:status` Artisan 指令來檢查 Octane 伺服器的目前狀態：

```shell
php artisan octane:status
```

<a name="dependency-injection-and-octane"></a>
## 依賴注入與 Octane

由於 Octane 在啟動後會將您的應用程式載入一次並保留在記憶體中以處理後續請求，因此在建構應用程式時需要考慮一些注意事項。例如，應用程式服務提供者的 `register` 和 `boot` 方法只會在請求 worker 首次啟動時執行一次。在後續的請求中，將重複使用相同的應用程式實例。

因此，在將應用程式服務容器或請求注入任何物件的建構子時，應特別小心。這樣做可能導致該物件在後續請求中持有過時的容器或請求版本。

Octane 會自動處理重置請求之間的所有第一方框架狀態。然而，Octane 並非總是知道如何重置您的應用程式所建立的全域狀態。因此，您應該了解如何以 Octane 友好的方式建構應用程式。下面，我們將討論使用 Octane 時可能導致問題的最常見情況。

<a name="container-injection"></a>
### 容器注入

一般而言，您應避免將應用程式服務容器或 HTTP 請求實例注入其他物件的建構子中。例如，以下綁定將整個應用程式服務容器注入到一個被綁定為 singleton 的物件中：

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

在此範例中，如果在應用程式啟動過程中解析 `Service` 實例，容器將被注入到服務中，並且相同的容器將在後續請求中由 `Service` 實例持有。這對您的特定應用程式來說**可能**不是問題；然而，這可能導致容器意外地缺少在啟動週期後期或由後續請求添加的綁定。

作為解決方法，您可以停止將綁定註冊為 singleton，或者您可以將一個容器解析閉包注入到服務中，該閉包始終解析當前的容器實例：

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

全域 `app` 輔助函式和 `Container::getInstance()` 方法將始終返回應用程式容器的最新版本。

<a name="request-injection"></a>
### 請求注入

一般而言，您應避免將應用程式服務容器或 HTTP 請求實例注入其他物件的建構子中。例如，以下綁定將整個請求實例注入到一個被綁定為 singleton 的物件中：

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

在此範例中，如果在應用程式啟動過程中解析 `Service` 實例，HTTP 請求將被注入到服務中，並且相同的請求將在後續請求中由 `Service` 實例持有。因此，所有標頭、輸入和查詢字串資料都將不正確，以及所有其他請求資料。

作為解決方法，您可以停止將綁定註冊為 singleton，或者您可以將一個請求解析閉包注入到服務中，該閉包始終解析當前的請求實例。或者，最推薦的方法是在執行時簡單地將物件所需的特定請求資訊傳遞給物件的其中一個方法：

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

全域 `request` 輔助函式將始終返回應用程式當前處理的請求，因此在您的應用程式中可以安全使用。

> [!WARNING]
> 在您的控制器方法和路由閉包中對 `Illuminate\Http\Request` 實例進行型別提示是可以接受的。

<a name="configuration-repository-injection"></a>
### 設定儲存庫注入

一般而言，您應避免將設定儲存庫實例注入其他物件的建構子中。例如，以下綁定將設定儲存庫注入到一個被綁定為 singleton 的物件中：

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

在此範例中，如果設定值在請求之間發生變化，該服務將無法存取新值，因為它依賴於原始的儲存庫實例。

作為解決方法，您可以停止將綁定註冊為 singleton，或者您可以將一個設定儲存庫解析閉包注入到該類別中：

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

全域 `config` 輔助函式將始終返回設定儲存庫的最新版本，因此在您的應用程式中可以安全使用。

<a name="managing-memory-leaks"></a>
### 管理記憶體洩漏

請記住，Octane 在請求之間會將您的應用程式保留在記憶體中；因此，將資料添加到靜態維護的陣列中將導致記憶體洩漏。例如，以下控制器存在記憶體洩漏，因為每次應用程式請求都會繼續將資料添加到靜態 `$data` 陣列中：

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

在建構您的應用程式時，您應該特別注意避免產生這些類型的記憶體洩漏。建議您在本地開發期間監控應用程式的記憶體使用情況，以確保您沒有在應用程式中引入新的記憶體洩漏。

<a name="concurrent-tasks"></a>
## 並行任務

> [!WARNING]
> 此功能需要 [Swoole](#swoole)。

使用 Swoole 時，您可以透過輕量級的背景任務並行執行操作。您可以使用 Octane 的 `concurrently` 方法來實現這一點。您可以將此方法與 PHP 陣列解構結合使用，以取得每個操作的結果：

```php
use App\Models\User;
use App\Models\Server;
use Laravel\Octane\Facades\Octane;

[$users, $servers] = Octane::concurrently([
    fn () => User::all(),
    fn () => Server::all(),
]);
```

由 Octane 處理的並行任務利用 Swoole 的「任務 worker」，並在與傳入請求完全不同的程序中執行。可用於處理並行任務的 worker 數量由 `octane:start` 命令上的 `--task-workers` 指令決定：

```shell
php artisan octane:start --workers=4 --task-workers=6
```

調用 `concurrently` 方法時，由於 Swoole 任務系統的限制，您不應提供超過 1024 個任務。

<a name="ticks-and-intervals"></a>
## 計時與間隔

> [!WARNING]
> 此功能需要 [Swoole](#swoole)。

當使用 Swoole 時，您可以註冊「計時」操作，這些操作將每隔指定的秒數執行一次。您可以透過 `tick` 方法註冊「計時」回呼。提供給 `tick` 方法的第一個參數應為一個字串，代表計時器的名稱。第二個參數應為一個可呼叫的函式，它將在指定的間隔內被呼叫。

在此範例中，我們將註冊一個閉包，每 10 秒執行一次。通常，`tick` 方法應該在應用程式的某個服務提供者的 `boot` 方法中被呼叫：

```php
Octane::tick('simple-ticker', fn () => ray('Ticking...'))
    ->seconds(10);
```

使用 `immediate` 方法，您可以指示 Octane 在 Octane 伺服器首次啟動時立即呼叫計時回呼，並在之後每 N 秒呼叫一次：

```php
Octane::tick('simple-ticker', fn () => ray('Ticking...'))
    ->seconds(10)
    ->immediate();
```


<a name="the-octane-cache"></a>
## Octane 快取

> [!WARNING]
> 此功能需要 [Swoole](#swoole)。

當使用 Swoole 時，您可以利用 Octane 快取驅動，它提供高達每秒 2 百萬次操作的讀寫速度。因此，對於需要極致讀寫速度的快取層應用程式來說，這個快取驅動是一個絕佳的選擇。

此快取驅動由 [Swoole tables](https://www.swoole.co.uk/docs/modules/swoole-table) 提供支援。儲存在快取中的所有資料都可供伺服器上的所有 Worker 使用。然而，當伺服器重新啟動時，快取的資料將會被清除：

```php
Cache::store('octane')->put('framework', 'Laravel', 30);
```

> [!NOTE]
> Octane 快取中允許的最大項目數量可以在應用程式的 `octane` 設定檔中定義。


<a name="cache-intervals"></a>
### 快取間隔

除了 Laravel 快取系統提供的典型方法外，Octane 快取驅動還具有基於間隔的快取。這些快取會以指定的間隔自動刷新，並且應該在應用程式的某個服務提供者的 `boot` 方法中註冊。例如，以下快取將每五秒刷新一次：

```php
use Illuminate\Support\Str;

Cache::store('octane')->interval('random', function () {
    return Str::random(10);
}, seconds: 5);
```


<a name="tables"></a>
## 資料表

> [!WARNING]
> 此功能需要 [Swoole](#swoole)。

當使用 Swoole 時，您可以定義和操作自己的任意 [Swoole tables](https://www.swoole.co.uk/docs/modules/swoole-table)。Swoole tables 提供極致的效能吞吐量，並且這些資料表中的資料可以被伺服器上的所有 Worker 存取。然而，當伺服器重新啟動時，其中的資料將會遺失。

資料表應在應用程式 `octane` 設定檔的 `tables` 設定陣列中定義。一個允許最多 1000 行的範例資料表已經為您配置好了。字串欄位的最大大小可以透過在欄位類型後指定欄位大小來配置，如下所示：

```php
'tables' => [
    'example:1000' => [
        'name' => 'string:1000',
        'votes' => 'int',
    ],
],
```

要存取資料表，您可以使用 `Octane::table` 方法：

```php
use Laravel\Octane\Facades\Octane;

Octane::table('example')->set('uuid', [
    'name' => 'Nuno Maduro',
    'votes' => 1000,
]);

return Octane::table('example')->get('uuid');
```

> [!WARNING]
> Swoole tables 支援的欄位類型有：`string`、`int` 和 `float`。