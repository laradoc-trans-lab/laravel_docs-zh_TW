# 部署

- [介紹](#introduction)
- [伺服器要求](#server-requirements)
- [伺服器配置](#server-configuration)
    - [Nginx](#nginx)
    - [FrankenPHP](#frankenphp)
    - [目錄權限](#directory-permissions)
- [最佳化](#optimization)
    - [快取配置](#optimizing-configuration-loading)
    - [快取事件](#caching-events)
    - [快取路由](#optimizing-route-loading)
    - [快取視圖](#optimizing-view-loading)
- [偵錯模式](#debug-mode)
- [健康檢查路由](#the-health-route)
- [使用 Laravel Cloud 或 Forge 進行部署](#deploying-with-cloud-or-forge)

<a name="introduction"></a>
## 介紹

當您準備將 Laravel 應用程式部署到生產環境時，可以採取一些重要措施，以確保您的應用程式盡可能高效地運行。在本文檔中，我們將介紹一些重要的切入點，以確保您的 Laravel 應用程式正確部署。

<a name="server-requirements"></a>
## 伺服器要求

Laravel 框架有一些系統要求。您應該確保您的網頁伺服器具有以下最低 PHP 版本和擴展：

<div class="content-list" markdown="1">

- PHP >= 8.2
- Ctype PHP Extension
- cURL PHP Extension
- DOM PHP Extension
- Fileinfo PHP Extension
- Filter PHP Extension
- Hash PHP Extension
- Mbstring PHP Extension
- OpenSSL PHP Extension
- PCRE PHP Extension
- PDO PHP Extension
- Session PHP Extension
- Tokenizer PHP Extension
- XML PHP Extension

</div>

<a name="server-configuration"></a>
## 伺服器配置

<a name="nginx"></a>
### Nginx

如果您正在將應用程式部署到運行 Nginx 的伺服器，您可以使用以下設定檔作為配置網頁伺服器的起點。此檔案很可能需要根據您的伺服器配置進行客製化。**如果您需要管理伺服器的協助，請考慮使用完全託管的 Laravel 平台，例如 [Laravel Cloud](https://cloud.laravel.com)。**

請確保您的網頁伺服器，如同以下設定，將所有請求導向您應用程式的 `public/index.php` 檔案。您絕不應該嘗試將 `index.php` 檔案移至專案的根目錄，因為從專案根目錄提供應用程式服務將會向公共網路洩露許多敏感設定檔：

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

[FrankenPHP](https://frankenphp.dev/) 也可用於服務您的 Laravel 應用程式。FrankenPHP 是一個以 Go 語言撰寫的現代 PHP 應用程式伺服器。要使用 FrankenPHP 服務 Laravel PHP 應用程式，您只需調用其 `php-server` 指令：

```shell
frankenphp php-server -r public/
```

要利用 FrankenPHP 支援的更強大功能，例如其 [Laravel Octane](/docs/{{version}}/octane) 整合、HTTP/3、現代壓縮，或將 Laravel 應用程式打包成獨立二進位檔的能力，請查閱 FrankenPHP 的 [Laravel 文件](https://frankenphp.dev/docs/laravel/)。

<a name="directory-permissions"></a>
### 目錄權限

Laravel 將需要寫入 `bootstrap/cache` 和 `storage` 目錄，因此您應該確保網頁伺服器處理程序的擁有者具有這些目錄的寫入權限。

<a name="optimization"></a>
## 最佳化

將應用程式部署到生產環境時，有多種檔案應該被快取，包括您的配置、事件、路由和視圖。Laravel 提供了一個單一且方便的 `optimize` Artisan 指令，它將快取所有這些檔案。此指令通常應該作為您應用程式部署過程的一部分來調用：

```shell
php artisan optimize
```

`optimize:clear` 方法可用於移除 `optimize` 指令產生的所有快取檔案以及預設快取驅動器中的所有鍵值：

```shell
php artisan optimize:clear
```

在以下文件，我們將討論 `optimize` 指令執行的每個細粒度最佳化指令。

<a name="optimizing-configuration-loading"></a>
### 快取配置

將應用程式部署到生產環境時，您應該確保在部署過程中執行 `config:cache` Artisan 指令：

```shell
php artisan config:cache
```

此指令將所有 Laravel 的設定檔合併成一個單一的快取檔案，這大大減少了框架在載入配置值時必須對檔案系統進行的存取次數。

> [!WARNING]
> 如果您在部署過程中執行 `config:cache` 指令，您應該確保僅從您的設定檔中呼叫 `env` 函數。一旦配置已被快取，`.env` 檔案將不會被載入，所有對 `.env` 變數的 `env` 函數呼叫都將回傳 `null`。

<a name="caching-events"></a>
### 快取事件

在部署過程中，您應該快取應用程式自動發現的事件到監聽器映射。這可以透過在部署期間調用 `event:cache` Artisan 指令來實現：

```shell
php artisan event:cache
```

<a name="optimizing-route-loading"></a>
### 快取路由

如果您正在構建一個包含許多路由的大型應用程式，您應該確保在部署過程中執行 `route:cache` Artisan 指令：

```shell
php artisan route:cache
```

此指令將所有路由註冊減少為快取檔案中的單一方法呼叫，從而提高註冊數百條路由時的路由註冊效能。

<a name="optimizing-view-loading"></a>
### 快取視圖

將應用程式部署到生產環境時，您應該確保在部署過程中執行 `view:cache` Artisan 指令：

```shell
php artisan view:cache
```

此指令會預編譯所有您的 Blade 視圖，使它們不會按需編譯，從而提高每個回傳視圖的請求效能。

<a name="debug-mode"></a>
## 偵錯模式

您 `config/app.php` 設定檔中的偵錯選項決定了實際向使用者顯示多少錯誤資訊。預設情況下，此選項設定為遵循 `APP_DEBUG` 環境變數的值，該變數儲存在您應用程式的 `.env` 檔案中。

> [!WARNING]
> **在您的生產環境中，此值應始終為 `false`。如果 `APP_DEBUG` 變數在生產環境中設定為 `true`，您可能會洩露敏感的配置值給您應用程式的終端使用者。**

<a name="the-health-route"></a>
## 健康檢查路由

Laravel 包含一個內建的健康檢查路由，可用於監控應用程式的狀態。在生產環境中，此路由可用於向運行時間監控器、負載平衡器或諸如 Kubernetes 之類的協調系統報告應用程式的狀態。

預設情況下，健康檢查路由位於 `/up`，如果應用程式沒有異常地啟動，將回傳 200 HTTP 回應。否則，將回傳 500 HTTP 回應。您可以在應用程式的 `bootstrap/app` 檔案中配置此路由的 URI：

```php
->withRouting(
    web: __DIR__.'/../routes/web.php',
    commands: __DIR__.'/../routes/console.php',
    health: '/up', // [tl! remove]
    health: '/status', // [tl! add]
)
```

當對此路由發出 HTTP 請求時，Laravel 也會分派一個 `Illuminate\Foundation\Events\DiagnosingHealth` 事件，允許您執行與應用程式相關的額外健康檢查。在此事件的 [監聽器](/docs/{{version}}/events) 中，您可以檢查應用程式的資料庫或快取狀態。如果您檢測到應用程式有問題，您可以直接從監聽器中拋出異常。

<a name="deploying-with-cloud-or-forge"></a>
## 使用 Laravel Cloud 或 Forge 進行部署


<a name="laravel-cloud"></a>
#### Laravel Cloud

如果您想要一個專為 Laravel 優化的全託管、自動擴展部署平台，請參考 [Laravel Cloud](https://cloud.laravel.com)。Laravel Cloud 是一個強大的 Laravel 部署平台，提供託管的運算、資料庫、快取和物件儲存服務。

在 Cloud 上部署您的 Laravel 應用程式，體驗其可擴展的簡潔性。Laravel Cloud 由 Laravel 的創作者精心調校，與框架無縫協作，讓您能像往常一樣撰寫 Laravel 應用程式。


<a name="laravel-forge"></a>
#### Laravel Forge

如果您偏好管理自己的伺服器，但對於配置執行強大 Laravel 應用程式所需的所有各類服務感到不自在，[Laravel Forge](https://forge.laravel.com) 是一個專為 Laravel 應用程式設計的 VPS 伺服器管理平台。

Laravel Forge 可以在 DigitalOcean、Linode、AWS 等各種基礎設施供應商上建立伺服器。此外，Forge 還會安裝並管理建立強大 Laravel 應用程式所需的所有工具，例如 Nginx、MySQL、Redis、Memcached、Beanstalk 等。