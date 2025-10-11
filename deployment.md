# 部署

- [簡介](#introduction)
- [伺服器需求](#server-requirements)
- [伺服器配置](#server-configuration)
    - [Nginx](#nginx)
    - [FrankenPHP](#frankenphp)
    - [目錄權限](#directory-permissions)
- [最佳化](#optimization)
    - [快取配置](#optimizing-configuration-loading)
    - [快取事件](#caching-events)
    - [快取路由](#optimizing-route-loading)
    - [快取視圖](#optimizing-view-loading)
- [除錯模式](#debug-mode)
- [健康檢查路由](#the-health-route)
- [使用 Forge / Vapor 輕鬆部署](#deploying-with-forge-or-vapor)

<a name="introduction"></a>
## 簡介

當您準備將 Laravel 應用程式部署到生產環境時，有一些重要的事情可以做，以確保您的應用程式盡可能高效地運行。在本文檔中，我們將介紹一些重要的起點，以確保您的 Laravel 應用程式正確部署。

<a name="server-requirements"></a>
## 伺服器需求

Laravel 框架有一些系統需求。您應該確保您的 Web 伺服器具有以下最低 PHP 版本和擴充套件：

<div class="content-list" markdown="1">

- PHP >= 8.2
- Ctype PHP 擴充套件
- cURL PHP 擴充套件
- DOM PHP 擴充套件
- Fileinfo PHP 擴充套件
- Filter PHP 擴充套件
- Hash PHP 擴充套件
- Mbstring PHP 擴充套件
- OpenSSL PHP 擴充套件
- PCRE PHP 擴充套件
- PDO PHP 擴充套件
- Session PHP 擴充套件
- Tokenizer PHP 擴充套件
- XML PHP 擴充套件

</div>

<a name="server-configuration"></a>
## 伺服器配置

<a name="nginx"></a>
### Nginx

如果您正在將應用程式部署到運行 Nginx 的伺服器，您可以使用以下配置檔案作為配置 Web 伺服器的起點。此檔案很可能需要根據您的伺服器配置進行客製化。**如果您需要伺服器管理協助，可以考慮使用 Laravel 官方伺服器管理與部署服務，例如 [Laravel Forge](https://forge.laravel.com)。**

請確保，如同以下配置所示，您的 Web 伺服器將所有請求導向您應用程式的 `public/index.php` 檔案。您絕不應嘗試將 `index.php` 檔案移至專案的根目錄，因為從專案根目錄提供應用程式服務將會將許多敏感配置檔案暴露給公共網路：

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name example.com;
    root /srv/example.com/public;

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";

    index index.php;

    charset utf-8;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    error_page 404 /index.php;

    location ~ ^/index\.php(/|$) {
        fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
        fastcgi_hide_header X-Powered-By;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

<a name="frankenphp"></a>
### FrankenPHP

[FrankenPHP](https://frankenphp.dev/) 也可用於服務您的 Laravel 應用程式。FrankenPHP 是一個使用 Go 編寫的現代 PHP 應用程式伺服器。要使用 FrankenPHP 服務 Laravel PHP 應用程式，您只需呼叫其 `php-server` 指令：

```shell
frankenphp php-server -r public/
```

要利用 FrankenPHP 支援的更強大功能，例如其 [Laravel Octane](/docs/{{version}}/octane) 整合、HTTP/3、現代壓縮，或將 Laravel 應用程式打包成獨立二進位檔的能力，請查閱 FrankenPHP 的 [Laravel 文件](https://frankenphp.dev/docs/laravel/)。

<a name="directory-permissions"></a>
### 目錄權限

Laravel 需要寫入 `bootstrap/cache` 和 `storage` 目錄，因此您應該確保 Web 伺服器行程擁有者對這些目錄具有寫入權限。

<a name="optimization"></a>
## 最佳化

將應用程式部署到生產環境時，有各種應當被快取的檔案，包括您的配置、事件、路由和視圖。Laravel 提供了一個單一且方便的 `optimize` Artisan 指令，它將快取所有這些檔案。此指令通常應作為您應用程式部署過程的一部分來呼叫：

```shell
php artisan optimize
```

`optimize:clear` 方法可用於移除所有由 `optimize` 指令產生的快取檔案，以及預設快取驅動器中的所有鍵：

```shell
php artisan optimize:clear
```

在以下文件中，我們將討論由 `optimize` 指令執行的每個細粒度的最佳化指令。

<a name="optimizing-configuration-loading"></a>
### 快取配置

將應用程式部署到生產環境時，您應該確保在部署過程期間運行 `config:cache` Artisan 指令：

```shell
php artisan config:cache
```

此指令將所有 Laravel 的配置檔案組合到一個單一的快取檔案中，這大大減少了框架在載入配置值時對檔案系統的訪問次數。

> [!WARNING]  
> 如果您在部署過程期間執行 `config:cache` 指令，您應該確保只從配置檔案中呼叫 `env` 函式。一旦配置已被快取，`.env` 檔案將不會被載入，並且所有針對 `.env` 變數的 `env` 函式呼叫都將返回 `null`。

<a name="caching-events"></a>
### 快取事件

在部署過程期間，您應該快取應用程式自動探索的事件到監聽器映射。這可以透過在部署期間呼叫 `event:cache` Artisan 指令來完成：

```shell
php artisan event:cache
```

<a name="optimizing-route-loading"></a>
### 快取路由

如果您正在構建一個包含許多路由的大型應用程式，您應該確保在部署過程期間運行 `route:cache` Artisan 指令：

```shell
php artisan route:cache
```

此指令將所有路由註冊精簡為快取檔案中的單一方法呼叫，從而提高註冊數百條路由時的效能。

<a name="optimizing-view-loading"></a>
### 快取視圖

將應用程式部署到生產環境時，您應該確保在部署過程期間運行 `view:cache` Artisan 指令：

```shell
php artisan view:cache
```

此指令會預編譯所有您的 Blade 視圖，使其不再按需編譯，從而提高每個返回視圖的請求的效能。

<a name="debug-mode"></a>
## 除錯模式

您的 `config/app.php` 配置檔案中的除錯選項決定了實際上向使用者顯示多少錯誤資訊。預設情況下，此選項設定為遵循 `APP_DEBUG` 環境變數的值，該變數儲存在您應用程式的 `.env` 檔案中。

> [!WARNING]  
> **在您的生產環境中，此值應始終為 `false`。如果在生產環境中將 `APP_DEBUG` 變數設定為 `true`，您有將敏感配置值暴露給應用程式終端使用者的風險。**

<a name="the-health-route"></a>
## 健康檢查路由

Laravel 包含一個內建的健康檢查路由，可用於監控應用程式的狀態。在生產環境中，此路由可用於向上線時間監控器、負載平衡器或編排系統（例如 Kubernetes）報告應用程式的狀態。

預設情況下，健康檢查路由在 `/up` 提供服務，並且如果應用程式在沒有例外的情況下啟動，將返回 200 HTTP 回應。否則，將返回 500 HTTP 回應。您可以在應用程式的 `bootstrap/app` 檔案中配置此路由的 URI：

    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        commands: __DIR__.'/../routes/console.php',
        health: '/up', // [tl! remove]
        health: '/status', // [tl! add]
    )

當對此路由發出 HTTP 請求時，Laravel 也將分派一個 `Illuminate\Foundation\Events\DiagnosingHealth` 事件，讓您能夠執行與應用程式相關的額外健康檢查。在此事件的 [監聽器](/docs/{{version}}/events) 中，您可以檢查應用程式的資料庫或快取狀態。如果您檢測到應用程式有問題，您可以直接從監聽器中拋出一個例外。

<a name="deploying-with-forge-or-vapor"></a>
## 使用 Forge / Vapor 輕鬆部署

<a name="laravel-forge"></a>
#### Laravel Forge

如果您還沒有準備好自行管理伺服器配置，或者不習慣配置運行一個健壯的 Laravel 應用程式所需的所有服務，那麼 [Laravel Forge](https://forge.laravel.com) 是一個絕佳的替代方案。

Laravel Forge 可以在多種基礎設施提供商（例如 DigitalOcean、Linode、AWS 等）上建立伺服器。此外，Forge 還會安裝和管理構建健壯 Laravel 應用程式所需的所有工具，例如 Nginx、MySQL、Redis、Memcached、Beanstalk 等。

> [!NOTE]  
> 想要一份關於使用 Laravel Forge 部署的完整指南嗎？請查看 [Laravel Bootcamp](https://bootcamp.laravel.com/deploying) 以及 Forge 在 [Laracasts 上提供的影片系列](https://laracasts.com/series/learn-laravel-forge-2022-edition)。

<a name="laravel-vapor"></a>
#### Laravel Vapor

如果您想要一個專為 Laravel 調優的完全無伺服器、自動擴展部署平台，請查看 [Laravel Vapor](https://vapor.laravel.com)。Laravel Vapor 是一個由 AWS 驅動，專為 Laravel 設計的無伺服器部署平台。在 Vapor 上啟動您的 Laravel 基礎設施，並愛上無伺服器可擴展的簡潔性。Laravel Vapor 經過 Laravel 創作者的精心調校，能與框架無縫協作，讓您能夠像往常一樣繼續編寫您的 Laravel 應用程式。