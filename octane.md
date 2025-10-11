# Laravel Octane

- [簡介](#introduction)
- [安裝](#installation)
- [伺服器先決條件](#server-prerequisites)
    - [FrankenPHP](#frankenphp)
    - [RoadRunner](#roadrunner)
    - [Swoole](#swoole)
- [執行您的應用程式](#serving-your-application)
    - [透過 HTTPS 執行您的應用程式](#serving-your-application-via-https)
    - [透過 Nginx 執行您的應用程式](#serving-your-application-via-nginx)
    - [監控檔案變更](#watching-for-file-changes)
    - [指定 Worker 數量](#specifying-the-worker-count)
    - [指定最大請求數量](#specifying-the-max-request-count)
    - [重新載入 Worker](#reloading-the-workers)
    - [停止伺服器](#stopping-the-server)
- [依賴注入與 Octane](#dependency-injection-and-octane)
    - [容器注入](#container-injection)
    - [請求注入](#request-injection)
    - [設定儲存庫注入](#configuration-repository-injection)
- [管理記憶體洩漏](#managing-memory-leaks)
- [並行任務](#concurrent-tasks)
- [定時器與間隔](#ticks-and-intervals)
- [Octane 快取](#the-octane-cache)
- [資料表](#tables)

<a name="introduction"></a>
## 簡介

[Laravel Octane](https://github.com/laravel/octane) 透過使用高效能應用程式伺服器來執行您的應用程式，大幅提升您應用程式的效能，這些伺服器包含 [FrankenPHP](https://frankenphp.dev/)、[Open Swoole](https://openswoole.com/)、[Swoole](https://github.com/swoole/swoole-src) 與 [RoadRunner](https://roadrunner.dev)。Octane 僅會啟動您的應用程式一次，將其保留在記憶體中，然後以超音速處理請求。


<a name="installation"></a>
## 安裝

Octane 可以透過 Composer 套件管理器來安裝：

```shell
composer require laravel/octane
```

安裝 Octane 之後，您可以執行 `octane:install` Artisan 指令，這將會把 Octane 的設定檔安裝到您的應用程式中：

```shell
php artisan octane:install
```

<a name="server-prerequisites"></a>
## 伺服器先決條件

> [!WARNING]  
> Laravel Octane 需要 [PHP 8.1+](https://php.net/releases/)。


<a name="frankenphp"></a>
### FrankenPHP

[FrankenPHP](https://frankenphp.dev) 是一個以 Go 語言撰寫的 PHP 應用程式伺服器，支援 early hints、Brotli 與 Zstandard 壓縮等現代網頁功能。當您安裝 Octane 並選擇 FrankenPHP 作為伺服器時，Octane 將會自動為您下載並安裝 FrankenPHP 二進位檔。


<a name="frankenphp-via-laravel-sail"></a>
#### FrankenPHP via Laravel Sail

如果您打算使用 [Laravel Sail](/docs/{{version}}/sail) 開發應用程式，您應該執行以下指令來安裝 Octane 和 FrankenPHP：

```shell
./vendor/bin/sail up

./vendor/bin/sail composer require laravel/octane
```

接著，您應該使用 `octane:install` Artisan 指令來安裝 FrankenPHP 二進位檔：

```shell
./vendor/bin/sail artisan octane:install --server=frankenphp
```

最後，在應用程式的 `docker-compose.yml` 檔案中，為 `laravel.test` 服務定義新增一個 `SUPERVISOR_PHP_COMMAND` 環境變數。此環境變數將包含 Sail 用來透過 Octane 執行應用程式的指令，而不是透過 PHP 開發伺服器執行：

```yaml
services:
  laravel.test:
    environment:
      SUPERVISOR_PHP_COMMAND: "/usr/bin/php -d variables_order=EGPCS /var/www/html/artisan octane:start --server=frankenphp --host=0.0.0.0 --admin-port=2019 --port='${APP_PORT:-80}'" # [tl! add]
      XDG_CONFIG_HOME:  /var/www/html/config # [tl! add]
      XDG_DATA_HOME:  /var/www/html/data # [tl! add]
```

若要啟用 HTTPS、HTTP/2 與 HTTP/3，請改用這些修改：

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

通常，您應該透過 `https://localhost` 存取您的 FrankenPHP Sail 應用程式，因為使用 `https://127.0.0.1` 需要額外的設定，並且 [不建議](https://frankenphp.dev/docs/known-issues/#using-https127001-with-docker)。


<a name="frankenphp-via-docker"></a>
#### FrankenPHP via Docker

使用 FrankenPHP 的官方 Docker 映像檔可以提供更好的效能，並支援靜態安裝的 FrankenPHP 不包含的額外擴充功能。此外，官方 Docker 映像檔也支援在其不原生支援的平台上執行 FrankenPHP，例如 Windows。FrankenPHP 的官方 Docker 映像檔適用於本地開發和正式環境使用。

您可以將以下 Dockerfile 作為起點，來容器化您的 FrankenPHP 驅動的 Laravel 應用程式：

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

如果 `--log-level` 選項明確傳遞給 `php artisan octane:start` 指令，Octane 將會使用 FrankenPHP 的原生日誌記錄器，並且，除非另行配置，將會產生結構化的 JSON 日誌。

您可以參考 [官方 FrankenPHP 文件](https://frankenphp.dev/docs/docker/) 以取得更多關於使用 Docker 執行 FrankenPHP 的資訊。


<a name="roadrunner"></a>
### RoadRunner

[RoadRunner](https://roadrunner.dev) 由以 Go 語言建置的 RoadRunner 二進位檔驅動。當您第一次啟動基於 RoadRunner 的 Octane 伺服器時，Octane 將會詢問您是否要下載並安裝 RoadRunner 二進位檔。


<a name="roadrunner-via-laravel-sail"></a>
#### RoadRunner via Laravel Sail

如果您打算使用 [Laravel Sail](/docs/{{version}}/sail) 開發您的應用程式，您應該執行以下指令來安裝 Octane 和 RoadRunner：

```shell
./vendor/bin/sail up

./vendor/bin/sail composer require laravel/octane spiral/roadrunner-cli spiral/roadrunner-http
```

接著，您應該啟動一個 Sail shell，並使用 `rr` 可執行檔來取得最新基於 Linux 的 RoadRunner 二進位檔：

```shell
./vendor/bin/sail shell

# Within the Sail shell...
./vendor/bin/rr get-binary
```

然後，在應用程式的 `docker-compose.yml` 檔案中，為 `laravel.test` 服務定義新增一個 `SUPERVISOR_PHP_COMMAND` 環境變數。此環境變數將包含 Sail 用來透過 Octane 執行應用程式的指令，而不是透過 PHP 開發伺服器執行：

```yaml
services:
  laravel.test:
    environment:
      SUPERVISOR_PHP_COMMAND: "/usr/bin/php -d variables_order=EGPCS /var/www/html/artisan octane:start --server=roadrunner --host=0.0.0.0 --rpc-port=6001 --port='${APP_PORT:-80}'" # [tl! add]
```

最後，確保 `rr` 二進位檔是可執行檔，並建置您的 Sail 映像檔：

```shell
chmod +x ./rr

./vendor/bin/sail build --no-cache
```


<a name="swoole"></a>
### Swoole

如果您打算使用 Swoole 應用程式伺服器來執行您的 Laravel Octane 應用程式，您必須安裝 Swoole PHP 擴充功能。通常，這可以透過 PECL 完成：

```shell
pecl install swoole
```


<a name="openswoole"></a>
#### Open Swoole

如果您想使用 Open Swoole 應用程式伺服器來執行您的 Laravel Octane 應用程式，您必須安裝 Open Swoole PHP 擴充功能。通常，這可以透過 PECL 完成：

```shell
pecl install openswoole
```

將 Laravel Octane 與 Open Swoole 搭配使用，可提供與 Swoole 相同的功能，例如並行任務、定時器和間隔。


<a name="swoole-via-laravel-sail"></a>
#### Swoole via Laravel Sail

> [!WARNING]  
> 在透過 Sail 執行 Octane 應用程式之前，請確保您擁有最新版本的 Laravel Sail，並在應用程式的根目錄中執行 `./vendor/bin/sail build --no-cache`。

或者，您可以使用 [Laravel Sail](/docs/{{version}}/sail) 開發基於 Swoole 的 Octane 應用程式，它是 Laravel 官方的基於 Docker 的開發環境。Laravel Sail 預設包含 Swoole 擴充功能。但是，您仍然需要調整 Sail 使用的 `docker-compose.yml` 檔案。

首先，在應用程式的 `docker-compose.yml` 檔案中，為 `laravel.test` 服務定義新增一個 `SUPERVISOR_PHP_COMMAND` 環境變數。此環境變數將包含 Sail 用來透過 Octane 執行應用程式的指令，而不是透過 PHP 開發伺服器執行：

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
#### Swoole Configuration

Swoole 支援一些額外的配置選項，您可以根據需要新增到 `octane` 配置檔案中。由於這些選項很少需要修改，因此預設配置檔案中不包含這些選項：

```php
'swoole' => [
    'options' => [
        'log_file' => storage_path('logs/swoole_http.log'),
        'package_max_length' => 10 * 1024 * 1024,
    ],
],
```

<a name="serving-your-application"></a>
## 執行您的應用程式

Octane 伺服器可以透過 `octane:start` Artisan 指令啟動。預設情況下，此指令將使用您的應用程式 `octane` 設定檔中 `server` 設定選項所指定的伺服器：

```shell
php artisan octane:start
```

預設情況下，Octane 將在 8000 埠啟動伺服器，因此您可以透過 `http://localhost:8000` 在網頁瀏覽器中存取您的應用程式。

<a name="serving-your-application-via-https"></a>
### 透過 HTTPS 執行您的應用程式

預設情況下，透過 Octane 執行的應用程式會產生以 `http://` 為前綴的連結。當透過 HTTPS 執行您的應用程式時，可以將應用程式 `config/octane.php` 設定檔中使用的 `OCTANE_HTTPS` 環境變數設定為 `true`。當此設定值設為 `true` 時，Octane 將會指示 Laravel 為所有產生的連結加上 `https://` 的前綴：

```php
'https' => env('OCTANE_HTTPS', false),
```

<a name="serving-your-application-via-nginx"></a>
### 透過 Nginx 執行您的應用程式

> [!NOTE]  
> 如果您還沒有準備好管理自己的伺服器設定，或者不熟悉設定運行一個強健 (robust) 的 Laravel Octane 應用程式所需的所有服務，請查看 [Laravel Forge](https://forge.laravel.com)。

在正式環境中，您應該將您的 Octane 應用程式運行在傳統的網頁伺服器（例如 Nginx 或 Apache）之後。這樣做將允許網頁伺服器提供您的靜態資源，例如圖片和樣式表，並管理您的 SSL 憑證終止。

在下面的 Nginx 設定範例中，Nginx 將提供網站的靜態資源，並將請求代理到運行在 8000 埠的 Octane 伺服器：

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

由於您的應用程式在 Octane 伺服器啟動時會一次性載入到記憶體中，因此重新整理瀏覽器時，您應用程式檔案的任何變更將不會反映出來。例如，新增到 `routes/web.php` 檔案中的路由定義，在伺服器重新啟動之前不會反映出來。為了方便起見，您可以使用 `--watch` 旗標來指示 Octane 在應用程式中的任何檔案變更時自動重新啟動伺服器：

```shell
php artisan octane:start --watch
```

在使用此功能之前，您應該確保 [Node](https://nodejs.org) 已安裝在您的本地開發環境中。此外，您應該在您的專案中安裝 [Chokidar](https://github.com/paulmillr/chokidar) 檔案監控程式庫：

```shell
npm install --save-dev chokidar
```

您可以使用應用程式 `config/octane.php` 設定檔中的 `watch` 設定選項來配置應該被監控的目錄和檔案。

<a name="specifying-the-worker-count"></a>
### 指定 Worker 數量

預設情況下，Octane 將為您的機器每個 CPU 核心啟動一個應用程式請求 Worker。這些 Worker 將用於處理傳入的 HTTP 請求。您可以在調用 `octane:start` 指令時使用 `--workers` 選項來手動指定您希望啟動多少個 Worker：

```shell
php artisan octane:start --workers=4
```

如果您正在使用 Swoole 應用程式伺服器，您也可以指定您希望啟動多少個「[任務 Worker](#concurrent-tasks)」：

```shell
php artisan octane:start --workers=4 --task-workers=6
```

<a name="specifying-the-max-request-count"></a>
### 指定最大請求數量

為了幫助防止零星的記憶體洩漏，Octane 會在任何 Worker 處理 500 個請求後優雅地重新啟動。要調整此數量，您可以使用 `--max-requests` 選項：

```shell
php artisan octane:start --max-requests=250
```

<a name="reloading-the-workers"></a>
### 重新載入 Worker

您可以使用 `octane:reload` 指令優雅地重新啟動 Octane 伺服器的應用程式 Worker。通常，這應該在部署後進行，以便您新部署的程式碼會載入到記憶體中，並用於處理後續的請求：

```shell
php artisan octane:reload
```

<a name="stopping-the-server"></a>
### 停止伺服器

您可以使用 `octane:stop` Artisan 指令停止 Octane 伺服器：

```shell
php artisan octane:stop
```

<a name="checking-the-server-status"></a>
#### 檢查伺服器狀態

您可以使用 `octane:status` Artisan 指令檢查 Octane 伺服器的當前狀態：

```shell
php artisan octane:status
```

<a name="dependency-injection-and-octane"></a>
## 依賴注入與 Octane

由於 Octane 會在啟動應用程式後將其保留在記憶體中以處理請求，因此在建構應用程式時，有一些注意事項需要考慮。例如，您的應用程式服務提供者的 `register` 和 `boot` 方法只會在請求 worker 初次啟動時執行一次。在後續請求中，會重複使用相同的應用程式實例。

有鑑於此，當將應用程式服務容器或請求注入到任何物件的建構子時，您應特別小心。這樣做的話，該物件在後續請求中可能會持有容器或請求的過時版本。

Octane 會自動處理重置請求之間的所有框架內部狀態。然而，Octane 並非總是知道如何重置應用程式所建立的全域狀態。因此，您應該了解如何以對 Octane 友好的方式建構應用程式。下面，我們將討論在使用 Octane 時可能導致問題的最常見情況。

<a name="container-injection"></a>
### 容器注入

通常，您應該避免將應用程式服務容器或 HTTP 請求實例注入到其他物件的建構子中。例如，以下綁定會將整個應用程式服務容器注入到一個綁定為單例的物件中：

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

在這個範例中，如果在應用程式啟動流程中解析了 `Service` 實例，容器會被注入到服務中，並且在後續請求中，該 `Service` 實例將持有同一個容器。這對於您的特定應用程式來說**可能**不是問題；然而，這可能導致容器意外遺失在啟動週期後期或由後續請求添加的綁定。

作為解決方法，您可以停止將綁定註冊為單例，或者您可以將一個容器解析器閉包注入到服務中，該閉包總是解析當前容器實例：

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

通常，您應該避免將應用程式服務容器或 HTTP 請求實例注入到其他物件的建構子中。例如，以下綁定會將整個請求實例注入到一個綁定為單例的物件中：

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

在這個範例中，如果在應用程式啟動流程中解析了 `Service` 實例，HTTP 請求會被注入到服務中，並且在後續請求中，該 `Service` 實例將持有同一個請求。因此，所有標頭、輸入和查詢字串資料以及所有其他請求資料都將不正確。

作為解決方法，您可以停止將綁定註冊為單例，或者您可以將一個請求解析器閉包注入到服務中，該閉包總是解析當前請求實例。或者，最推薦的方法是簡單地在執行時將物件所需特定請求資訊傳遞給其中一個方法：

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
> 在控制器方法和路由閉包中對 `Illuminate\Http\Request` 實例進行型別提示是可接受的。

<a name="configuration-repository-injection"></a>
### 設定儲存庫注入

通常，您應該避免將設定儲存庫實例注入到其他物件的建構子中。例如，以下綁定會將設定儲存庫注入到一個綁定為單例的物件中：

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

在這個範例中，如果設定值在請求之間發生變更，該服務將無法存取新值，因為它依賴於原始儲存庫實例。

作為解決方法，您可以停止將綁定註冊為單例，或者您可以將一個設定儲存庫解析器閉包注入到該類別中：

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

全域 `config` 將始終返回設定儲存庫的最新版本，因此在您的應用程式中可以安全使用。

<a name="managing-memory-leaks"></a>
### 管理記憶體洩漏

請記住，Octane 會在請求之間將您的應用程式保持在記憶體中；因此，將資料添加到靜態維護的陣列中將導致記憶體洩漏。例如，以下控制器存在記憶體洩漏，因為對應用程式的每個請求都會不斷向靜態 `$data` 陣列中添加資料：

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

在建構應用程式時，您應特別注意避免造成這類記憶體洩漏。建議您在本地開發環境中監控應用程式的記憶體使用情況，以確保您沒有引入新的記憶體洩漏到您的應用程式中。

<a name="concurrent-tasks"></a>
## 並行任務

> [!WARNING]
> 此功能需要 [Swoole](#swoole)。

使用 Swoole 時，您可以透過輕量背景任務並行執行操作。您可以使用 Octane 的 `concurrently` 方法來實現此目的。您可以將此方法與 PHP 陣列解構結合使用，以獲取每個操作的結果：

```php
use App\Models\User;
use App\Models\Server;
use Laravel\Octane\Facades\Octane;

[$users, $servers] = Octane::concurrently([
    fn () => User::all(),
    fn () => Server::all(),
]);
```

由 Octane 處理的並行任務會利用 Swoole 的「任務 worker」，並在與傳入請求完全不同的程序中執行。用於處理並行任務的可用 worker 數量由 `octane:start` 命令上的 `--task-workers` 指令決定：

```shell
php artisan octane:start --workers=4 --task-workers=6
```

呼叫 `concurrently` 方法時，由於 Swoole 任務系統所施加的限制，您提供的任務不應超過 1024 個。

<a name="ticks-and-intervals"></a>
## 定時器與間隔

> [!WARNING]  
> 此功能需要 [Swoole](#swoole)。

當使用 Swoole 時，您可以註冊「定時器 (tick)」操作，這些操作將每隔指定秒數執行一次。您可以使用 `tick` 方法來註冊「定時器 (tick)」回呼。提供給 `tick` 方法的第一個參數應該是一個字串，代表定時器的名稱。第二個參數應該是一個可呼叫 (callable) 物件，它將會以指定的間隔被呼叫。

在此範例中，我們將註冊一個閉包，每 10 秒被呼叫一次。通常，`tick` 方法應該在您應用程式的服務提供者 (service provider) 中的 `boot` 方法內呼叫：

```php
Octane::tick('simple-ticker', fn () => ray('Ticking...'))
    ->seconds(10);
```

使用 `immediate` 方法，您可以指示 Octane 在 Octane 伺服器初次啟動時立即呼叫定時器回呼，然後每 N 秒再呼叫一次：

```php
Octane::tick('simple-ticker', fn () => ray('Ticking...'))
    ->seconds(10)
    ->immediate();
```

<a name="the-octane-cache"></a>
## Octane 快取

> [!WARNING]  
> 此功能需要 [Swoole](#swoole)。

當使用 Swoole 時，您可以利用 Octane 快取驅動程式，它提供高達每秒 2 百萬次操作的讀寫速度。因此，對於需要極致讀寫速度的快取層應用程式來說，此快取驅動程式是絕佳的選擇。

此快取驅動程式由 [Swoole 資料表](https://www.swoole.co.uk/docs/modules/swoole-table)提供支援。所有儲存在快取中的資料都可供伺服器上的所有 worker 存取。然而，當伺服器重新啟動時，快取資料將會被清除：

```php
Cache::store('octane')->put('framework', 'Laravel', 30);
```

> [!NOTE]  
> Octane 快取允許的最大條目數可以在您應用程式的 `octane` 設定檔中定義。

<a name="cache-intervals"></a>
### 快取間隔

除了 Laravel 快取系統提供的典型方法之外，Octane 快取驅動程式還具備基於間隔的快取。這些快取會以指定的間隔自動刷新，並且應該在您應用程式的服務提供者 (service provider) 中的 `boot` 方法內註冊。例如，以下快取將每五秒刷新一次：

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

當使用 Swoole 時，您可以定義並與您自己的任意 [Swoole 資料表](https://www.swoole.co.uk/docs/modules/swoole-table)互動。Swoole 資料表提供極致的效能吞吐量，並且這些資料表中的資料可供伺服器上的所有 worker 存取。然而，當伺服器重新啟動時，其中的資料將會遺失。

資料表應定義在您應用程式的 `octane` 設定檔的 `tables` 設定陣列中。一個允許最多 1000 行的範例資料表已經為您設定好了。字串欄位的最大大小可以透過在欄位類型後指定欄位大小來設定，如下所示：

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
> Swoole 資料表支援的欄位類型為：`string`、`int` 和 `float`。