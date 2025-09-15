# Laravel Octane

- [簡介](#introduction)
- [安裝](#installation)
- [伺服器先決條件](#server-prerequisites)
    - [FrankenPHP](#frankenphp)
    - [RoadRunner](#roadrunner)
    - [Swoole](#swoole)
- [啟動應用程式](#serving-your-application)
    - [透過 HTTPS 啟動應用程式](#serving-your-application-via-https)
    - [透過 Nginx 啟動應用程式](#serving-your-application-via-nginx)
    - [監控檔案變更](#watching-for-file-changes)
    - [指定 Worker 數量](#specifying-the-worker-count)
    - [指定最大請求數量](#specifying-the-max-request-count)
    - [重新載入 Worker](#reloading-the-workers)
    - [停止伺服器](#stopping-the-server)
- [依賴注入與 Octane](#dependency-injection-and-octane)
    - [Container 注入](#container-injection)
    - [Request 注入](#request-injection)
    - [Configuration Repository 注入](#configuration-repository-injection)
- [管理記憶體洩漏](#managing-memory-leaks)
- [並行任務](#concurrent-tasks)
- [Tick 與 Interval](#ticks-and-intervals)
- [Octane 快取](#the-octane-cache)
- [資料表](#tables)

<a name="introduction"></a>
## 簡介

[Laravel Octane](https://github.com/laravel/octane) 透過使用高效能的應用程式伺服器，包括 [FrankenPHP](https://frankenphp.dev/)、[Open Swoole](https://openswoole.com/)、[Swoole](https://github.com/swoole/swoole-src) 和 [RoadRunner](https://roadrunner.dev)，大幅提升應用程式的效能。Octane 會啟動應用程式一次，將其保留在記憶體中，然後以超音速處理請求。

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

> [!WARNING]  
> Laravel Octane 需要 [PHP 8.1+](https://php.net/releases/)。

<a name="frankenphp"></a>
### FrankenPHP

[FrankenPHP](https://frankenphp.dev) 是一個用 Go 語言編寫的 PHP 應用程式伺服器，支援現代網路功能，例如 Early Hints、Brotli 和 Zstandard 壓縮。當您安裝 Octane 並選擇 FrankenPHP 作為伺服器時，Octane 會自動為您下載並安裝 FrankenPHP 二進位檔。

<a name="frankenphp-via-laravel-sail"></a>
#### 透過 Laravel Sail 使用 FrankenPHP

如果您打算使用 [Laravel Sail](/docs/{{version}}/sail) 開發應用程式，您應該執行以下命令來安裝 Octane 和 FrankenPHP：

```shell
./vendor/bin/sail up

./vendor/bin/sail composer require laravel/octane
```

接下來，您應該使用 `octane:install` Artisan 命令來安裝 FrankenPHP 二進位檔：

```shell
./vendor/bin/sail artisan octane:install --server=frankenphp
```

最後，在應用程式的 `docker-compose.yml` 檔案中，為 `laravel.test` 服務定義新增一個 `SUPERVISOR_PHP_COMMAND` 環境變數。此環境變數將包含 Sail 用來透過 Octane 而非 PHP 開發伺服器啟動應用程式的命令：

```yaml
services:
  laravel.test:
    environment:
      SUPERVISOR_PHP_COMMAND: "/usr/bin/php -d variables_order=EGPCS /var/www/html/artisan octane:start --server=frankenphp --host=0.0.0.0 --admin-port=2019 --port='${APP_PORT:-80}'" # [tl! add]
      XDG_CONFIG_HOME:  /var/www/html/config # [tl! add]
      XDG_DATA_HOME:  /var/www/html/data # [tl! add]
```

若要啟用 HTTPS、HTTP/2 和 HTTP/3，請改為套用這些修改：

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

通常，您應該透過 `https://localhost` 存取您的 FrankenPHP Sail 應用程式，因為使用 `https://127.0.0.1` 需要額外的設定並且[不建議](https://frankenphp.dev/docs/known-issues/#using-https127001-with-docker)。

<a name="frankenphp-via-docker"></a>
#### 透過 Docker 使用 FrankenPHP

使用 FrankenPHP 的官方 Docker 映像檔可以提供更高的效能，並使用 FrankenPHP 靜態安裝中未包含的其他擴充功能。此外，官方 Docker 映像檔支援在 FrankenPHP 不原生支援的平台上執行，例如 Windows。FrankenPHP 的官方 Docker 映像檔適用於本地開發和生產環境。

您可以將以下 Dockerfile 作為容器化您的 FrankenPHP 驅動的 Laravel 應用程式的起點：

```dockerfile
FROM dunglas/frankenphp

RUN install-php-extensions \
    pcntl
    # Add other PHP extensions here...

COPY . /app

ENTRYPOINT ["php", "artisan", "octane:frankenphp"]
```

然後，在開發過程中，您可以使用以下 Docker Compose 檔案來執行您的應用程式：

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

如果 `--log-level` 選項明確傳遞給 `php artisan octane:start` 命令，Octane 將使用 FrankenPHP 的原生記錄器，並且除非另行設定，否則將產生結構化的 JSON 記錄。

您可以查閱 [FrankenPHP 官方文件](https://frankenphp.dev/docs/docker/)以獲取更多關於透過 Docker 執行 FrankenPHP 的資訊。

<a name="roadrunner"></a>
### RoadRunner

[RoadRunner](https://roadrunner.dev) 由使用 Go 語言建構的 RoadRunner 二進位檔提供支援。當您第一次啟動基於 RoadRunner 的 Octane 伺服器時，Octane 會提供為您下載並安裝 RoadRunner 二進位檔。

<a name="roadrunner-via-laravel-sail"></a>
#### 透過 Laravel Sail 使用 RoadRunner

如果您打算使用 [Laravel Sail](/docs/{{version}}/sail) 開發應用程式，您應該執行以下命令來安裝 Octane 和 RoadRunner：

```shell
./vendor/bin/sail up

./vendor/bin/sail composer require laravel/octane spiral/roadrunner-cli spiral/roadrunner-http
```

接下來，您應該啟動一個 Sail shell 並使用 `rr` 可執行檔來取得最新基於 Linux 的 RoadRunner 二進位檔：

```shell
./vendor/bin/sail shell

# Within the Sail shell...
./vendor/bin/rr get-binary
```

然後，在應用程式的 `docker-compose.yml` 檔案中，為 `laravel.test` 服務定義新增一個 `SUPERVISOR_PHP_COMMAND` 環境變數。此環境變數將包含 Sail 用來透過 Octane 而非 PHP 開發伺服器啟動應用程式的命令：

```yaml
services:
  laravel.test:
    environment:
      SUPERVISOR_PHP_COMMAND: "/usr/bin/php -d variables_order=EGPCS /var/www/html/artisan octane:start --server=roadrunner --host=0.0.0.0 --rpc-port=6001 --port='${APP_PORT:-80}'" # [tl! add]
```

最後，確保 `rr` 二進位檔是可執行的，並建構您的 Sail 映像檔：

```shell
chmod +x ./rr

./vendor/bin/sail build --no-cache
```

<a name="swoole"></a>
### Swoole

如果您打算使用 Swoole 應用程式伺服器來啟動您的 Laravel Octane 應用程式，您必須安裝 Swoole PHP 擴充功能。通常，這可以透過 PECL 完成：

```shell
pecl install swoole
```

<a name="openswoole"></a>
#### Open Swoole

如果您想使用 Open Swoole 應用程式伺服器來啟動您的 Laravel Octane 應用程式，您必須安裝 Open Swoole PHP 擴充功能。通常，這可以透過 PECL 完成：

```shell
pecl install openswoole
```

將 Laravel Octane 與 Open Swoole 搭配使用，可提供與 Swoole 相同的功能，例如並行任務、Tick 和 Interval。

<a name="swoole-via-laravel-sail"></a>
#### 透過 Laravel Sail 使用 Swoole

> [!WARNING]  
> 在透過 Sail 啟動 Octane 應用程式之前，請確保您擁有最新版本的 Laravel Sail，並在應用程式的根目錄中執行 `./vendor/bin/sail build --no-cache`。

或者，您可以使用 [Laravel Sail](/docs/{{version}}/sail) (Laravel 官方基於 Docker 的開發環境) 來開發您的基於 Swoole 的 Octane 應用程式。Laravel Sail 預設包含 Swoole 擴充功能。但是，您仍然需要調整 Sail 使用的 `docker-compose.yml` 檔案。

首先，在應用程式的 `docker-compose.yml` 檔案中，為 `laravel.test` 服務定義新增一個 `SUPERVISOR_PHP_COMMAND` 環境變數。此環境變數將包含 Sail 用來透過 Octane 而非 PHP 開發伺服器啟動應用程式的命令：

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
#### Swoole 設定

Swoole 支援一些額外的設定選項，如果需要，您可以將它們新增到您的 `octane` 設定檔中。由於它們很少需要修改，因此這些選項不包含在預設設定檔中：

```php
'swoole' => [
    'options' => [
        'log_file' => storage_path('logs/swoole_http.log'),
        'package_max_length' => 10 * 1024 * 1024,
    ],
],
```

<a name="serving-your-application"></a>
## 啟動應用程式

Octane 伺服器可以透過 `octane:start` Artisan 命令啟動。預設情況下，此命令將使用應用程式 `octane` 設定檔的 `server` 設定選項中指定的伺服器：

```shell
php artisan octane:start
```

預設情況下，Octane 將在 8000 埠啟動伺服器，因此您可以透過 `http://localhost:8000` 在網頁瀏覽器中存取您的應用程式。

<a name="serving-your-application-via-https"></a>
### 透過 HTTPS 啟動應用程式

預設情況下，透過 Octane 執行的應用程式會產生以 `http://` 為前綴的連結。當透過 HTTPS 啟動應用程式時，可以在應用程式的 `config/octane.php` 設定檔中將 `OCTANE_HTTPS` 環境變數設定為 `true`。當此設定值設定為 `true` 時，Octane 將指示 Laravel 將所有產生的連結加上 `https://` 前綴：

```php
'https' => env('OCTANE_HTTPS', false),
```

<a name="serving-your-application-via-nginx"></a>
### 透過 Nginx 啟動應用程式

> [!NOTE]  
> 如果您還沒有準備好管理自己的伺服器設定，或者不熟悉設定執行穩健的 Laravel Octane 應用程式所需的所有各種服務，請查看 [Laravel Forge](https://forge.laravel.com)。

在生產環境中，您應該將 Octane 應用程式部署在傳統的網頁伺服器（例如 Nginx 或 Apache）之後。這樣做將允許網頁伺服器提供您的靜態資產（例如圖片和樣式表），並管理您的 SSL 憑證終止。

在下面的 Nginx 設定範例中，Nginx 將提供網站的靜態資產，並將請求代理到在 8000 埠執行的 Octane 伺服器：

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

由於您的應用程式在 Octane 伺服器啟動時會載入到記憶體中一次，因此對應用程式檔案的任何變更都不會在您重新整理瀏覽器時反映出來。例如，新增到 `routes/web.php` 檔案中的路由定義在伺服器重新啟動之前不會反映出來。為了方便起見，您可以使用 `--watch` 旗標來指示 Octane 在應用程式中的任何檔案變更時自動重新啟動伺服器：

```shell
php artisan octane:start --watch
```

在使用此功能之前，您應該確保您的本地開發環境中已安裝 [Node](https://nodejs.org)。此外，您應該在您的專案中安裝 [Chokidar](https://github.com/paulmillr/chokidar) 檔案監控函式庫：

```shell
npm install --save-dev chokidar
```

您可以使用應用程式 `config/octane.php` 設定檔中的 `watch` 設定選項來設定應監控的目錄和檔案。

<a name="specifying-the-worker-count"></a>
### 指定 Worker 數量

預設情況下，Octane 會為您的機器提供的每個 CPU 核心啟動一個應用程式請求 Worker。這些 Worker 將用於處理進入應用程式的傳入 HTTP 請求。您可以在呼叫 `octane:start` 命令時使用 `--workers` 選項手動指定要啟動的 Worker 數量：

```shell
php artisan octane:start --workers=4
```

如果您正在使用 Swoole 應用程式伺服器，您還可以指定要啟動的 ["task workers"](#concurrent-tasks) 數量：

```shell
php artisan octane:start --workers=4 --task-workers=6
```

<a name="specifying-the-max-request-count"></a>
### 指定最大請求數量

為了幫助防止記憶體洩漏，Octane 會在每個 Worker 處理 500 個請求後優雅地重新啟動。若要調整此數字，您可以使用 `--max-requests` 選項：

```shell
php artisan octane:start --max-requests=250
```

<a name="reloading-the-workers"></a>
### 重新載入 Worker

您可以使用 `octane:reload` 命令優雅地重新啟動 Octane 伺服器的應用程式 Worker。通常，這應該在部署後完成，以便您的新部署程式碼載入到記憶體中並用於處理後續請求：

```shell
php artisan octane:reload
```

<a name="stopping-the-server"></a>
### 停止伺服器

您可以使用 `octane:stop` Artisan 命令停止 Octane 伺服器：

```shell
php artisan octane:stop
```

<a name="checking-the-server-status"></a>
#### 檢查伺服器狀態

您可以使用 `octane:status` Artisan 命令檢查 Octane 伺服器的目前狀態：

```shell
php artisan octane:status
```

<a name="dependency-injection-and-octane"></a>
## 依賴注入與 Octane

由於 Octane 會啟動您的應用程式一次，並在處理請求時將其保留在記憶體中，因此在建構應用程式時應考慮一些注意事項。例如，應用程式服務提供者的 `register` 和 `boot` 方法只會在請求 Worker 初次啟動時執行一次。在後續請求中，將重複使用相同的應用程式實例。

有鑑於此，在將應用程式服務 Container 或 Request 注入任何物件的建構函式時，您應該特別小心。這樣做可能會導致該物件在後續請求中持有過時版本的 Container 或 Request。

Octane 會自動處理重置請求之間任何第一方框架狀態。然而，Octane 並非總是知道如何重置應用程式建立的全域狀態。因此，您應該了解如何以 Octane 友好的方式建構應用程式。下面，我們將討論在使用 Octane 時可能導致問題的最常見情況。

<a name="container-injection"></a>
### Container 注入

一般來說，您應該避免將應用程式服務 Container 或 HTTP Request 實例注入其他物件的建構函式中。例如，以下綁定將整個應用程式服務 Container 注入到綁定為單例的物件中：

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

在此範例中，如果在應用程式啟動過程中解析 `Service` 實例，則 Container 將被注入到服務中，並且該相同的 Container 將在後續請求中由 `Service` 實例持有。這對於您的特定應用程式**可能**不是問題；但是，它可能導致 Container 意外地缺少在啟動週期後期或由後續請求新增的綁定。

作為解決方法，您可以停止將綁定註冊為單例，或者您可以將一個 Container 解析器閉包注入到服務中，該閉包始終解析當前的 Container 實例：

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

全域 `app` 輔助函式和 `Container::getInstance()` 方法將始終返回應用程式 Container 的最新版本。

<a name="request-injection"></a>
### Request 注入

一般來說，您應該避免將應用程式服務 Container 或 HTTP Request 實例注入其他物件的建構函式中。例如，以下綁定將整個 Request 實例注入到綁定為單例的物件中：

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

在此範例中，如果在應用程式啟動過程中解析 `Service` 實例，則 HTTP Request 將被注入到服務中，並且該相同的 Request 將在後續請求中由 `Service` 實例持有。因此，所有標頭、輸入和查詢字串資料都將不正確，以及所有其他請求資料。

作為解決方法，您可以停止將綁定註冊為單例，或者您可以將一個 Request 解析器閉包注入到服務中，該閉包始終解析當前的 Request 實例。或者，最推薦的方法是簡單地在執行時將物件所需的特定請求資訊傳遞給物件的方法之一：

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

全域 `request` 輔助函式將始終返回應用程式目前正在處理的請求，因此在您的應用程式中可以安全使用。

> [!WARNING]  
> 在您的控制器方法和路由閉包上型別提示 `Illuminate\Http\Request` 實例是可以接受的。

<a name="configuration-repository-injection"></a>
### Configuration Repository 注入

一般來說，您應該避免將 Configuration Repository 實例注入其他物件的建構函式中。例如，以下綁定將 Configuration Repository 注入到綁定為單例的物件中：

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

在此範例中，如果設定值在請求之間發生變化，該服務將無法存取新值，因為它依賴於原始的 Repository 實例。

作為解決方法，您可以停止將綁定註冊為單例，或者您可以將一個 Configuration Repository 解析器閉包注入到類別中：

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

全域 `config` 將始終返回 Configuration Repository 的最新版本，因此在您的應用程式中可以安全使用。

<a name="managing-memory-leaks"></a>
### 管理記憶體洩漏

請記住，Octane 會在請求之間將您的應用程式保留在記憶體中；因此，將資料新增到靜態維護的陣列將導致記憶體洩漏。例如，以下控制器存在記憶體洩漏，因為對應用程式的每個請求都會繼續將資料新增到靜態 `$data` 陣列中：

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

在建構應用程式時，您應該特別小心，避免產生這些類型的記憶體洩漏。建議您在本地開發期間監控應用程式的記憶體使用情況，以確保您沒有在應用程式中引入新的記憶體洩漏。

<a name="concurrent-tasks"></a>
## 並行任務

> [!WARNING]  
> 此功能需要 [Swoole](#swoole)。

使用 Swoole 時，您可以透過輕量級的背景任務並行執行操作。您可以使用 Octane 的 `concurrently` 方法來實現此目的。您可以將此方法與 PHP 陣列解構結合使用，以檢索每個操作的結果：

```php
use App\Models\User;
use App\Models\Server;
use Laravel\Octane\Facades\Octane;

[$users, $servers] = Octane::concurrently([
    fn () => User::all(),
    fn () => Server::all(),
]);
```

由 Octane 處理的並行任務利用 Swoole 的「task workers」，並在與傳入請求完全不同的程序中執行。可用於處理並行任務的 Worker 數量由 `octane:start` 命令上的 `--task-workers` 指令決定：

```shell
php artisan octane:start --workers=4 --task-workers=6
```

在呼叫 `concurrently` 方法時，由於 Swoole 任務系統的限制，您不應提供超過 1024 個任務。

<a name="ticks-and-intervals"></a>
## Tick 與 Interval

> [!WARNING]  
> 此功能需要 [Swoole](#swoole)。

使用 Swoole 時，您可以註冊「Tick」操作，這些操作將每隔指定的秒數執行一次。您可以透過 `tick` 方法註冊「Tick」回呼。提供給 `tick` 方法的第一個參數應該是一個字串，表示 Tick 的名稱。第二個參數應該是一個可呼叫的函式，它將以指定的間隔被呼叫。

在此範例中，我們將註冊一個閉包，每 10 秒呼叫一次。通常，`tick` 方法應該在應用程式服務提供者的 `boot` 方法中呼叫：

```php
Octane::tick('simple-ticker', fn () => ray('Ticking...'))
    ->seconds(10);
```

使用 `immediate` 方法，您可以指示 Octane 在 Octane 伺服器初次啟動時立即呼叫 Tick 回呼，然後每 N 秒呼叫一次：

```php
Octane::tick('simple-ticker', fn () => ray('Ticking...'))
    ->seconds(10)
    ->immediate();
```

<a name="the-octane-cache"></a>
## Octane 快取

> [!WARNING]  
> 此功能需要 [Swoole](#swoole)。

使用 Swoole 時，您可以利用 Octane 快取驅動程式，它提供每秒高達 200 萬次操作的讀寫速度。因此，此快取驅動程式對於需要從快取層獲得極致讀寫速度的應用程式來說是一個絕佳的選擇。

此快取驅動程式由 [Swoole tables](https://www.swoole.co.uk/docs/modules/swoole-table) 提供支援。儲存在快取中的所有資料都可供伺服器上的所有 Worker 存取。但是，當伺服器重新啟動時，快取資料將被清除：

```php
Cache::store('octane')->put('framework', 'Laravel', 30);
```

> [!NOTE]  
> Octane 快取中允許的最大條目數可以在應用程式的 `octane` 設定檔中定義。

<a name="cache-intervals"></a>
### 快取 Interval

除了 Laravel 快取系統提供的典型方法之外，Octane 快取驅動程式還具有基於 Interval 的快取。這些快取會以指定的 Interval 自動重新整理，並且應該在應用程式服務提供者的 `boot` 方法中註冊。例如，以下快取將每五秒重新整理一次：

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

使用 Swoole 時，您可以定義並與自己的任意 [Swoole tables](https://www.swoole.co.uk/docs/modules/swoole-table) 互動。Swoole tables 提供極致的效能吞吐量，並且這些資料表中的資料可以由伺服器上的所有 Worker 存取。但是，當伺服器重新啟動時，其中的資料將會遺失。

資料表應該在應用程式 `octane` 設定檔的 `tables` 設定陣列中定義。一個允許最多 1000 行的範例資料表已經為您設定好。字串欄位的最大大小可以透過在欄位型別之後指定欄位大小來設定，如下所示：

```php
'tables' => [
    'example:1000' => [
        'name' => 'string:1000',
        'votes' => 'int',
    ],
],
```

若要存取資料表，您可以使用 `Octane::table` 方法：

```php
use Laravel\Octane\Facades\Octane;

Octane::table('example')->set('uuid', [
    'name' => 'Nuno Maduro',
    'votes' => 1000,
]);

return Octane::table('example')->get('uuid');
```

> [!WARNING]  
> Swoole tables 支援的欄位型別有：`string`、`int` 和 `float`。
