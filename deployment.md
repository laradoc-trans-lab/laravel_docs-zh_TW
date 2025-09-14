# 部署

- [簡介](#introduction)
- [伺服器要求](#server-requirements)
- [伺服器設定](#server-configuration)
    - [Nginx](#nginx)
    - [FrankenPHP](#frankenphp)
    - [目錄權限](#directory-permissions)
- [最佳化](#optimization)
    - [快取設定](#optimizing-configuration-loading)
    - [快取事件](#caching-events)
    - [快取路由](#optimizing-route-loading)
    - [快取視圖](#optimizing-view-loading)
- [除錯模式](#debug-mode)
- [健康檢查路由](#the-health-route)
- [使用 Laravel Cloud 或 Forge 部署](#deploying-with-cloud-or-forge)

<a name="introduction"></a>
## 簡介

當您準備將 Laravel 應用程式部署到生產環境時，可以採取一些重要措施來確保您的應用程式盡可能高效地運行。本文檔將介紹一些確保 Laravel 應用程式正確部署的絕佳起點。

<a name="server-requirements"></a>
## 伺服器要求

Laravel 框架有一些系統要求。您應該確保您的網頁伺服器具有以下最低 PHP 版本和擴充功能：

<div class="content-list" markdown="1">

- PHP >= 8.2
- Ctype PHP 擴充功能
- cURL PHP 擴充功能
- DOM PHP 擴充功能
- Fileinfo PHP 擴充功能
- Filter PHP 擴充功能
- Hash PHP 擴充功能
- Mbstring PHP 擴充功能
- OpenSSL PHP 擴充功能
- PCRE PHP 擴充功能
- PDO PHP 擴充功能
- Session PHP 擴充功能
- Tokenizer PHP 擴充功能
- XML PHP 擴充功能

</div>

<a name="server-configuration"></a>
## 伺服器設定

<a name="nginx"></a>
### Nginx

如果您要將應用程式部署到運行 Nginx 的伺服器，您可以使用以下設定檔作為設定網頁伺服器的起點。此檔案很可能需要根據您的伺服器設定進行自訂。**如果您需要伺服器管理方面的協助，請考慮使用像 [Laravel Cloud](https://cloud.laravel.com) 這樣完全託管的 Laravel 平台。**

請確保，如同以下設定，您的網頁伺服器將所有請求導向應用程式的 `public/index.php` 檔案。您絕不應嘗試將 `index.php` 檔案移動到專案的根目錄，因為從專案根目錄提供應用程式將會將許多敏感的設定檔暴露給公共網路：

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

[FrankenPHP](https://frankenphp.dev/) 也可以用來提供您的 Laravel 應用程式。FrankenPHP 是一個用 Go 編寫的現代 PHP 應用程式伺服器。要使用 FrankenPHP 提供 Laravel PHP 應用程式，您只需調用其 `php-server` 命令：

```shell
frankenphp php-server -r public/
```

要利用 FrankenPHP 支援的更強大功能，例如其 [Laravel Octane](/docs/{{version}}/octane) 整合、HTTP/3、現代壓縮，或將 Laravel 應用程式打包為獨立二進位檔的能力，請查閱 FrankenPHP 的 [Laravel 文件](https://frankenphp.dev/docs/laravel/)。

<a name="directory-permissions"></a>
### 目錄權限

Laravel 需要寫入 `bootstrap/cache` 和 `storage` 目錄，因此您應該確保網頁伺服器處理程序擁有寫入這些目錄的權限。

<a name="optimization"></a>
## 最佳化

將應用程式部署到生產環境時，有各種檔案應該被快取，包括您的設定、事件、路由和視圖。Laravel 提供了一個單一、方便的 `optimize` Artisan 命令，它將快取所有這些檔案。此命令通常應作為應用程式部署過程的一部分調用：

```shell
php artisan optimize
```

`optimize:clear` 方法可用於移除 `optimize` 命令生成的所有快取檔案以及預設快取驅動程式中的所有鍵：

```shell
php artisan optimize:clear
```

在以下文件中，我們將討論 `optimize` 命令執行的每個細粒度最佳化命令。

<a name="optimizing-configuration-loading"></a>
### 快取設定

將應用程式部署到生產環境時，您應該確保在部署過程中運行 `config:cache` Artisan 命令：

```shell
php artisan config:cache
```

此命令會將所有 Laravel 的設定檔合併到一個單一的快取檔案中，這大大減少了框架在載入設定值時必須對檔案系統進行的存取次數。

> [!WARNING]
> 如果您在部署過程中執行 `config:cache` 命令，您應該確保只從設定檔中呼叫 `env` 函數。一旦設定被快取，`.env` 檔案將不會被載入，並且所有對 `.env` 變數的 `env` 函數呼叫都將返回 `null`。

<a name="caching-events"></a>
### 快取事件

您應該在部署過程中快取應用程式自動發現的事件到監聽器映射。這可以透過在部署期間調用 `event:cache` Artisan 命令來完成：

```shell
php artisan event:cache
```

<a name="optimizing-route-loading"></a>
### 快取路由

如果您正在構建一個具有許多路由的大型應用程式，您應該確保在部署過程中運行 `route:cache` Artisan 命令：

```shell
php artisan route:cache
```

此命令將所有路由註冊減少為快取檔案中的單一方法呼叫，從而提高了註冊數百個路由時的路由註冊效能。

<a name="optimizing-view-loading"></a>
### 快取視圖

將應用程式部署到生產環境時，您應該確保在部署過程中運行 `view:cache` Artisan 命令：

```shell
php artisan view:cache
```

此命令會預編譯所有 Blade 視圖，因此它們不會按需編譯，從而提高了每個返回視圖的請求的效能。

<a name="debug-mode"></a>
## 除錯模式

`config/app.php` 設定檔中的除錯選項決定了實際向使用者顯示多少錯誤資訊。預設情況下，此選項設定為遵循 `APP_DEBUG` 環境變數的值，該變數儲存在應用程式的 `.env` 檔案中。

> [!WARNING]
> **在您的生產環境中，此值應始終為 `false`。如果 `APP_DEBUG` 變數在生產環境中設定為 `true`，您將面臨將敏感設定值暴露給應用程式終端使用者的風險。**

<a name="the-health-route"></a>
## 健康檢查路由

Laravel 包含一個內建的健康檢查路由，可用於監控應用程式的狀態。在生產環境中，此路由可用於向正常運行時間監控器、負載平衡器或 Kubernetes 等協調系統報告應用程式的狀態。

預設情況下，健康檢查路由在 `/up` 提供服務，如果應用程式在沒有例外的情況下啟動，則將返回 200 HTTP 回應。否則，將返回 500 HTTP 回應。您可以在應用程式的 `bootstrap/app` 檔案中設定此路由的 URI：

```php
->withRouting(
    web: __DIR__.'/../routes/web.php',
    commands: __DIR__.'/../routes/console.php',
    health: '/up', // [tl! remove]
    health: '/status', // [tl! add]
)
```

當對此路由發出 HTTP 請求時，Laravel 還將分派 `Illuminate\Foundation\Events\DiagnosingHealth` 事件，允許您執行與應用程式相關的其他健康檢查。在此事件的 [監聽器](/docs/{{version}}/events) 中，您可以檢查應用程式的資料庫或快取狀態。如果您發現應用程式存在問題，您可以簡單地從監聽器中拋出例外。

<a name="deploying-with-cloud-or-forge"></a>
## 使用 Laravel Cloud 或 Forge 部署

<a name="laravel-cloud"></a>
#### Laravel Cloud

如果您需要一個完全託管、自動擴展且針對 Laravel 進行調整的部署平台，請查看 [Laravel Cloud](https://cloud.laravel.com)。Laravel Cloud 是一個強大的 Laravel 部署平台，提供託管計算、資料庫、快取和物件儲存。

在 Cloud 上啟動您的 Laravel 應用程式，並愛上其可擴展的簡潔性。Laravel Cloud 由 Laravel 的創建者精心調整，可與框架無縫協作，因此您可以像往常一樣繼續編寫您的 Laravel 應用程式。

<a name="laravel-forge"></a>
#### Laravel Forge

如果您更喜歡管理自己的伺服器，但又不習慣設定運行強大 Laravel 應用程式所需的所有各種服務，[Laravel Forge](https://forge.laravel.com) 是一個用於 Laravel 應用程式的 VPS 伺服器管理平台。

Laravel Forge 可以在各種基礎設施提供商（例如 DigitalOcean、Linode、AWS 等）上創建伺服器。此外，Forge 還安裝和管理構建強大 Laravel 應用程式所需的所有工具，例如 Nginx、MySQL、Redis、Memcached、Beanstalk 等。

