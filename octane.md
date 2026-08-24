# Laravel Octane

- [介紹](#introduction)
- [安裝](#installation)
- [伺服器先決條件](#server-prerequisites)
    - [FrankenPHP](#frankenphp)
    - [RoadRunner](#roadrunner)
    - [Swoole](#swoole)
- [運行您的應用程式](#serving-your-application)
    - [透過 HTTPS 運行您的應用程式](#serving-your-application-via-https)
    - [透過 Nginx 運行您的應用程式](#serving-your-application-via-nginx)
    - [監聽檔案變更](#watching-for-file-changes)
    - [指定 Worker 數量](#specifying-the-worker-count)
    - [指定最大請求數](#specifying-the-max-request-count)
    - [指定最大執行時間](#specifying-the-max-execution-time)
    - [重新載入 Worker](#reloading-the-workers)
    - [停止伺服器](#stopping-the-server)
- [依賴注入與 Octane](#dependency-injection-and-octane)
    - [容器注入](#container-injection)
    - [請求注入](#request-injection)
    - [設定儲存庫注入](#configuration-repository-injection)
- [管理記憶體洩漏](#managing-memory-leaks)
- [並行任務](#concurrent-tasks)
- [Ticks 與間隔](#ticks-and-intervals)
- [Octane 快取](#the-octane-cache)
- [Table](#tables)

<a name="introduction"></a>
## 介紹

[Laravel Octane](https://github.com/laravel/octane) 透過使用高效能的應用程式伺服器來提供應用程式服務，大幅提升應用程式的效能，支援的伺服器包含 [FrankenPHP](https://frankenphp.dev/)、[Open Swoole](https://openswoole.com/)、[Swoole](https://github.com/swoole/swoole-src) 與 [RoadRunner](https://roadrunner.dev)。Octane 只會啟動一次您的應用程式，並將其常駐在記憶體中，接著以極快的速度處理傳入的請求。


<a name="installation"></a>
## 安裝

Octane 可以透過 Composer 套件管理器進行安裝：

```shell
composer require laravel/octane
```

安裝 Octane 之後，您可以執行 `octane:install` Artisan 命令，這會將 Octane 的設定檔安裝至您的應用程式中：

```shell
php artisan octane:install
```

<a name="server-prerequisites"></a>
## 伺服器先決條件

<a name="frankenphp"></a>
### FrankenPHP

[FrankenPHP](https://frankenphp.dev) 是一個用 Go 編寫的 PHP 應用程式伺服器，支援現代 Web 功能，如 Early Hints、Brotli 和 Zstandard 壓縮。當您安裝 Octane 並選擇 FrankenPHP 作為您的伺服器時，Octane 將自動為您下載並安裝 FrankenPHP 二進位檔案。

<a name="frankenphp-via-laravel-sail"></a>
#### 透過 Laravel Sail 使用 FrankenPHP

如果您打算使用 [Laravel Sail](/docs/{{version}}/sail) 開發應用程式，您應該執行以下指令來安裝 Octane 和 FrankenPHP：

```shell
./vendor/bin/sail up

./vendor/bin/sail composer require laravel/octane
```

接著，您應該使用 `octane:install` Artisan 指令來安裝 FrankenPHP 二進位檔案：

```shell
./vendor/bin/sail artisan octane:install --server=frankenphp
```

最後，在應用程式的 `docker-compose.yml` 檔案中的 `laravel.test` 服務定義裡新增 `SUPERVISOR_PHP_COMMAND` 環境變數。此環境變數將包含 Sail 用於透過 Octane（而非 PHP 開發伺服器）運行應用程式的指令：

```yaml
services:
  laravel.test:
    environment:
      SUPERVISOR_PHP_COMMAND: "/usr/bin/php -d variables_order=EGPCS /var/www/html/artisan octane:start --server=frankenphp --host=0.0.0.0 --admin-port=2019 --port='${APP_PORT:-80}'" # [tl! add]
      XDG_CONFIG_HOME:  /var/www/html/config # [tl! add]
      XDG_DATA_HOME:  /var/www/html/data # [tl! add]
```

若要啟用 HTTPS、HTTP/2 和 HTTP/3，請改為套用以下修改：

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

通常，您應該透過 `https://localhost` 存取您的 FrankenPHP Sail 應用程式，因為使用 `https://127.0.0.1` 需要額外設定且[不建議使用](https://frankenphp.dev/docs/known-issues/#using-https127001-with-docker)。

<a name="frankenphp-via-docker"></a>
#### 透過 Docker 使用 FrankenPHP

使用 FrankenPHP 的官方 Docker 映像檔可以提供更好的效能，並能使用靜態安裝 FrankenPHP 時未包含的額外擴充功能。此外，官方 Docker 映像檔還支援在 FrankenPHP 原生不支援的平台（如 Windows）上運行。FrankenPHP 的官方 Docker 映像檔適用於本地端開發和正式環境。

您可以使用以下 Dockerfile 作為將 FrankenPHP 驅動的 Laravel 應用程式容器化的起點：

```dockerfile
FROM dunglas/frankenphp

RUN install-php-extensions \
    pcntl
    # Add other PHP extensions here...

COPY . /app

ENTRYPOINT ["php", "artisan", "octane:frankenphp"]
```

然後，在開發過程中，您可以使用以下 Docker Compose 檔案來運行您的應用程式：

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

如果明確將 `--log-level` 選項傳遞給 `php artisan octane:start` 指令，Octane 將使用 FrankenPHP 的原生記錄器，並且除非進行了不同設定，否則將產生結構化的 JSON 記錄。

您可以參考 [FrankenPHP 官方文件](https://frankenphp.dev/docs/docker/)以取得更多關於在 Docker 中運行 FrankenPHP 的資訊。

<a name="frankenphp-caddyfile"></a>
#### 自訂 Caddyfile 設定

使用 FrankenPHP 時，您可以在啟動 Octane 時使用 `--caddyfile` 選項指定自訂的 Caddyfile：

```shell
php artisan octane:start --server=frankenphp --caddyfile=/path/to/your/Caddyfile
```

這允許您在預設設定之外自訂 FrankenPHP 的設定，例如新增自訂中介層、設定進階路由或設定自訂指令 (Directives)。您可以參考 [Caddy 官方文件](https://caddyserver.com/docs/caddyfile)以取得更多關於 Caddyfile 語法和設定選項的資訊。

<a name="roadrunner"></a>
### RoadRunner

[RoadRunner](https://roadrunner.dev) 由使用 Go 構建的 RoadRunner 二進位檔案提供支援。當您第一次啟動基於 RoadRunner 的 Octane 伺服器時，Octane 將會提示為您下載並安裝 RoadRunner 二進位檔案。

<a name="roadrunner-via-laravel-sail"></a>
#### 透過 Laravel Sail 使用 RoadRunner

如果您打算使用 [Laravel Sail](/docs/{{version}}/sail) 開發應用程式，您應該執行以下指令來安裝 Octane 和 RoadRunner：

```shell
./vendor/bin/sail up

./vendor/bin/sail composer require laravel/octane spiral/roadrunner-cli spiral/roadrunner-http
```

接著，您應該啟動 Sail shell 並使用 `rr` 執行檔來取得最新的 Linux 版 RoadRunner 二進位建置檔案：

```shell
./vendor/bin/sail shell

# Within the Sail shell...
./vendor/bin/rr get-binary
```

然後，在應用程式的 `docker-compose.yml` 檔案中的 `laravel.test` 服務定義裡新增 `SUPERVISOR_PHP_COMMAND` 環境變數。此環境變數將包含 Sail 用於透過 Octane（而非 PHP 開發伺服器）運行應用程式的指令：

```yaml
services:
  laravel.test:
    environment:
      SUPERVISOR_PHP_COMMAND: "/usr/bin/php -d variables_order=EGPCS /var/www/html/artisan octane:start --server=roadrunner --host=0.0.0.0 --rpc-port=6001 --port='${APP_PORT:-80}'" # [tl! add]
```

最後，確保 `rr` 二進位檔案具有執行權限並重新建置您的 Sail 映像檔：

```shell
chmod +x ./rr

./vendor/bin/sail build --no-cache
```

<a name="swoole"></a>
### Swoole

如果您打算使用 Swoole 應用程式伺服器來運行您的 Laravel Octane 應用程式，則必須安裝 Swoole PHP 擴充功能。通常可以透過 PECL 來安裝：

```shell
pecl install swoole
```


<a name="openswoole"></a>
#### Open Swoole

如果您想使用 Open Swoole 應用程式伺服器來運行您的 Laravel Octane 應用程式，則必須安裝 Open Swoole PHP 擴充功能。通常可以透過 PECL 來安裝：

```shell
pecl install openswoole
```

在 Laravel Octane 中使用 Open Swoole 可享有與 Swoole 相同的功能，例如並行任務、ticks 以及間隔。


<a name="swoole-via-laravel-sail"></a>
#### 透過 Laravel Sail 使用 Swoole

> [!WARNING]
> 透過 Sail 運行 Octane 應用程式之前，請確保您擁有最新版本的 Laravel Sail，並在應用程式的根目錄中執行 `./vendor/bin/sail build --no-cache`。

或者，您也可以使用 [Laravel Sail](/docs/{{version}}/sail) 來開發基於 Swoole 的 Octane 應用程式，這是 Laravel 官方提供的 Docker 開發環境。Laravel Sail 預設包含 Swoole 擴充功能。不過，您仍然需要調整 Sail 所使用的 `docker-compose.yml` 檔案。

首先，在應用程式的 `docker-compose.yml` 檔案中的 `laravel.test` 服務定義裡新增 `SUPERVISOR_PHP_COMMAND` 環境變數。此環境變數將包含 Sail 用來透過 Octane（而非 PHP 開發伺服器）運行應用程式的指令：

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

Swoole 支援一些額外的設定選項，如有需要，您可以將其新增至 `octane` 設定檔中。由於這些選項很少需要修改，因此它們並未包含在預設的設定檔中：

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

Octane 伺服器可以透過 `octane:start` Artisan 命令啟動。預設情況下，此命令將使用應用程式的 `octane` 設定檔中 `server` 設定選項所指定的伺服器：

```shell
php artisan octane:start
```

預設情況下，Octane 會在連接埠 8000 上啟動伺服器，因此您可以透過 `http://localhost:8000` 在網頁瀏覽器中存取您的應用程式。


<a name="keeping-octane-running-in-production"></a>
#### 在正式環境中保持 Octane 運行

如果您要將 Octane 應用程式部署到正式環境，您應該使用如 Supervisor 等行程監控工具，以確保 Octane 伺服器保持運行狀態。Octane 的 Supervisor 設定檔範例可能如下所示：

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

預設情況下，透過 Octane 運行的應用程式產生的連結會帶有 `http://` 前綴。當透過 HTTPS 運行應用程式時，可以在應用程式的 `config/octane.php` 設定檔中將 `OCTANE_HTTPS` 環境變數設定為 `true`。當此設定值設為 `true` 時，Octane 將指示 Laravel 為所有產生的連結加上 `https://` 前綴：

```php
'https' => env('OCTANE_HTTPS', false),
```


<a name="serving-your-application-via-nginx"></a>
### 透過 Nginx 運行您的應用程式

> [!NOTE]
> 如果您還沒準備好自行管理伺服器設定，或者不熟悉設定運行穩健的 Laravel Octane 應用程式所需的各種服務，請參考提供全代管 Laravel Octane 支援的 [Laravel Cloud](https://cloud.laravel.com)。

在正式環境中，您應該將 Octane 應用程式運行在傳統網頁伺服器（如 Nginx 或 Apache）之後。這樣做可以讓網頁伺服器處理靜態資源（例如圖片和樣式表），並管理 SSL 憑證終止。

在下方的 Nginx 設定範例中，Nginx 將處理網站的靜態資源，並將請求反向代理至運行在連接埠 8000 上的 Octane 伺服器：

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

由於在 Octane 伺服器啟動時，您的應用程式只會載入到記憶體中一次，因此重新整理瀏覽器時將不會反映對應用程式檔案所做的任何變更。例如，新增到 `routes/web.php` 檔案中的路由定義在伺服器重新啟動之前都不會生效。為了方便起見，您可以使用 `--watch` 旗標來指示 Octane 在應用程式內的任何檔案發生變更時自動重新啟動伺服器：

```shell
php artisan octane:start --watch
```

在使用此功能之前，您應該確保本地開發環境中已安裝 [Node](https://nodejs.org)。此外，您應該在專案中安裝 [Chokidar](https://github.com/paulmillr/chokidar) 檔案監聽程式庫：

```shell
npm install --save-dev chokidar
```

您可以在應用程式的 `config/octane.php` 設定檔中使用 `watch` 設定選項來設定應該監聽的目錄與檔案。


<a name="specifying-the-worker-count"></a>
### 指定 Worker 數量

預設情況下，Octane 會為您的機器所提供的每個 CPU 核心啟動一個應用程式請求 Worker。這些 Worker 將在連入的 HTTP 請求進入您的應用程式時用來處理它們。您可以在執行 `octane:start` 命令時使用 `--workers` 選項手動指定要啟動的 Worker 數量：

```shell
php artisan octane:start --workers=4
```

如果您使用的是 Swoole 應用程式伺服器，您還可以指定要啟動的 [「任務 Worker」](#concurrent-tasks) 數量：

```shell
php artisan octane:start --workers=4 --task-workers=6
```


<a name="specifying-the-max-request-count"></a>
### 指定最大請求數

為了防止無意間產生的記憶體洩漏，Octane 會在任何 Worker 處理完 500 個請求後優雅地重新啟動它。若要調整此數量，您可以使用 `--max-requests` 選項：

```shell
php artisan octane:start --max-requests=250
```


<a name="specifying-the-max-execution-time"></a>
### 指定最大執行時間

預設情況下，Laravel Octane 透過應用程式 `config/octane.php` 設定檔中的 `max_execution_time` 選項，將連入請求的最大執行時間設定為 30 秒：

```php
'max_execution_time' => 30,
```

此設定定義了連入請求在被終止之前允許執行的最長秒數。將此值設為 `0` 將完全停用執行時間限制。此設定選項對於處理耗時請求的應用程式特別有用，例如檔案上傳、資料處理或對外部服務的 API 呼叫。

> [!WARNING]
> 當您修改 `max_execution_time` 設定時，必須重新啟動 Octane 伺服器才能使變更生效。


<a name="reloading-the-workers"></a>
### 重新載入 Worker

您可以使用 `octane:reload` 命令優雅地重新啟動 Octane 伺服器的應用程式 Worker。通常，這應該在部署後執行，以便將新部署的程式碼載入到記憶體中，並用於處理後續的請求：

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

由於 Octane 只會啟動您的應用程式一次，並在處理請求時將其保留在記憶體中，因此在建構應用程式時有一些注意事項需要考量。例如，您應用程式的服務提供者中的 `register` 和 `boot` 方法只會在請求 Worker 初次啟動時執行一次。在後續的請求中，將會重複使用相同的應用程式實例。

有鑑於此，當將應用程式服務容器或請求注入到任何物件的建構函式中時，應特別小心。這樣做可能會導致該物件在後續請求中持有過期版本的容器或請求。

Octane 會自動處理在請求之間重設任何第一方框架的狀態。然而，Octane 不一定知道如何重設由您的應用程式所建立的全域狀態。因此，您應該了解如何以對 Octane 友善的方式來建構您的應用程式。以下我們將討論在使用 Octane 時可能導致問題的最常見情況。


<a name="container-injection"></a>
### 容器注入

一般來說，您應該避免將應用程式服務容器或 HTTP 請求實例注入到其他物件的建構函式中。例如，以下綁定將整個應用程式服務容器注入到一個綁定為單例 (Singleton) 的物件中：

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

在這個範例中，如果在應用程式啟動過程中解析了 `Service` 實例，該容器將被注入到服務中，並且在後續的請求中，該 `Service` 實例將繼續持有同一個容器。這對您的特定應用程式**可能**不是問題；但是，這可能會導致容器意外遺漏在啟動週期後期或後續請求中新增的綁定。

作為替代方案，您可以停止將該綁定註冊為單例，或者您可以將一個容器解析器閉包注入到服務中，該閉包始終解析目前的容器實例：

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

全域的 `app` 輔助函式與 `Container::getInstance()` 方法將始終回傳最新版本的應用程式容器。


<a name="request-injection"></a>
### 請求注入

一般來說，您應該避免將應用程式服務容器或 HTTP 請求實例注入到其他物件的建構函式中。例如，以下綁定將整個請求實例注入到一個綁定為單例的物件中：

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

在這個範例中，如果在應用程式啟動過程中解析了 `Service` 實例，該 HTTP 請求將被注入到服務中，並且在後續請求中，該 `Service` 實例將繼續持有相同的請求。因此，所有標頭 (Headers)、輸入和查詢字串資料以及所有其他請求資料都將是不正確的。

作為替代方案，您可以停止將該綁定註冊為單例，或者您可以將一個請求解析器閉包注入到服務中，該閉包始終解析當前的請求實例。或者，最推薦的做法是在執行階段直接將物件所需的特定請求資訊傳遞給該物件的方法：

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

全域的 `request` 輔助函式將始終回傳應用程式目前正在處理的請求，因此在您的應用程式中使用它是安全的。

> [!WARNING]
> 在控制器方法和路由閉包中對 `Illuminate\Http\Request` 實例進行型別提示是完全可以接受的。


<a name="configuration-repository-injection"></a>
### 設定儲存庫注入

一般來說，您應該避免將設定儲存庫實例注入到其他物件的建構函式中。例如，以下綁定將設定儲存庫注入到一個綁定為單例的物件中：

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

在此範例中，如果設定值在請求之間發生變更，該服務將無法存取新值，因為它依賴於原始的儲存庫實例。

作為替代方案，您可以停止將該綁定註冊為單例，或者您可以將設定儲存庫解析器閉包注入到該類別中：

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

全域的 `config` 將始終回傳最新版本的設定儲存庫，因此在您的應用程式中使用它是安全的。


<a name="managing-memory-leaks"></a>
### 管理記憶體洩漏

請記住，Octane 會在請求之間將您的應用程式保留在記憶體中；因此，將資料新增至靜態維護的陣列中將會導致記憶體洩漏。例如，以下控制器存在記憶體洩漏，因為對應用程式的每個請求都會持續將資料新增至靜態 `$data` 陣列中：

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

在建構應用程式時，應特別小心避免產生這種類型的記憶體洩漏。建議您在本地開發期間監控應用程式的記憶體使用情況，以確保沒有在應用程式中引入新的記憶體洩漏。


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

由 Octane 處理的並行任務利用了 Swoole 的「任務 Worker」，並在與傳入請求完全不同的行程中執行。可用於處理並行任務的 Worker 數量由 `octane:start` 指令上的 `--task-workers` 指令引數決定：

```shell
php artisan octane:start --workers=4 --task-workers=6
```

呼叫 `concurrently` 方法時，由於 Swoole 任務系統施加的限制，您不應提供超過 1024 個任務。

<a name="ticks-and-intervals"></a>
## Ticks 與間隔

> [!WARNING]
> 此功能需要 [Swoole](#swoole)。

使用 Swoole 時，您可以註冊每隔指定秒數執行的「tick」操作。您可以透過 `tick` 方法註冊「tick」回呼。提供給 `tick` 方法的第一個引數應該是代表計時器名稱的字串。第二個引數應該是在指定間隔時被呼叫的可呼叫物件 (Callable)。

在此範例中，我們將註冊一個每 10 秒被呼叫一次的閉包。通常，`tick` 方法應該在應用程式的某個服務提供者(Service Providers)的 `boot` 方法中被呼叫：

```php
Octane::tick('simple-ticker', fn () => ray('Ticking...'))
    ->seconds(10);
```

使用 `immediate` 方法，您可以指示 Octane 在 Octane 伺服器初始啟動時立即呼叫 tick 回呼，並在此後每隔 N 秒執行一次：

```php
Octane::tick('simple-ticker', fn () => ray('Ticking...'))
    ->seconds(10)
    ->immediate();
```


<a name="the-octane-cache"></a>
## Octane 快取

> [!WARNING]
> 此功能需要 [Swoole](#swoole)。

使用 Swoole 時，您可以利用 Octane 快取驅動，其提供每秒高達 200 萬次操作的讀取和寫入速度。因此，對於需要快取層具備極致讀寫速度的應用程式來說，此快取驅動是極佳的選擇。

此快取驅動由 [Swoole tables](https://www.swoole.co.uk/docs/modules/swoole-table) 提供支援。儲存在快取中的所有資料都可供伺服器上的所有 worker 存取。但是，當伺服器重新啟動時，快取的資料將被清空：

```php
Cache::store('octane')->put('framework', 'Laravel', 30);
```

> [!NOTE]
> Octane 快取中允許的最大項目數可以在應用程式的 `octane` 設定檔中定義。


<a name="cache-intervals"></a>
### 快取間隔

除了 Laravel 快取系統提供的常見方法外，Octane 快取驅動還具備基於間隔的快取功能。這些快取會在指定的間隔自動更新，且應該在應用程式的某個服務提供者(Service Providers)的 `boot` 方法中進行註冊。例如，以下快取將每五秒更新一次：

```php
use Illuminate\Support\Str;

Cache::store('octane')->interval('random', function () {
    return Str::random(10);
}, seconds: 5);
```


<a name="tables"></a>
## Table

> [!WARNING]
> 此功能需要 [Swoole](#swoole)。

使用 Swoole 時，您可以定義自己的任意 [Swoole tables](https://www.swoole.co.uk/docs/modules/swoole-table) 並與之互動。Swoole table 提供極高的效能吞吐量，且這些資料表中的資料可以被伺服器上的所有 worker 存取。但是，當伺服器重新啟動時，其中的資料將會遺失。

Table 應該在應用程式的 `octane` 設定檔中的 `tables` 設定陣列內定義。預設已為您設定了一個最多允許 1000 列的範例資料表。字串欄位的最大大小可以透過在欄位型別之後指定欄位大小來進行設定，如下所示：

```php
'tables' => [
    'example:1000' => [
        'name' => 'string:1000',
        'votes' => 'int',
    ],
],
```

要存取 table，您可以使用 `Octane::table` 方法：

```php
use Laravel\Octane\Facades\Octane;

Octane::table('example')->set('uuid', [
    'name' => 'Nuno Maduro',
    'votes' => 1000,
]);

return Octane::table('example')->get('uuid');
```

> [!WARNING]
> Swoole table 支援的欄位型別為：`string`、`int` 和 `float`。