# 部署

- [簡介](#introduction)
- [伺服器需求](#server-requirements)
- [伺服器配置](#server-configuration)
    - [Nginx](#nginx)
    - [FrankenPHP](#frankenphp)
    - [目錄權限](#directory-permissions)
- [優化](#optimization)
    - [快取配置](#optimizing-configuration-loading)
    - [快取事件](#caching-events)
    - [快取路由](#optimizing-route-loading)
    - [快取視圖](#optimizing-view-loading)
- [重新載入服務](#reloading-services)
- [除錯模式](#debug-mode)
- [健康檢查路由](#the-health-route)
- [使用 Laravel Cloud 或 Forge 部署](#deploying-with-cloud-or-forge)

<a name="introduction"></a>
## 簡介

當您準備將 Laravel 應用程式部署到正式環境時，有一些重要事項可以確保您的應用程式能以最高效率執行。在本文件中，我們將介紹一些確保 Laravel 應用程式正確部署的良好起點。


<a name="server-requirements"></a>
## 伺服器需求

Laravel 框架有一些系統需求。您應確保您的網頁伺服器具有以下最低 PHP 版本與擴展：

<div class="content-list" markdown="1">

- PHP >= 8.3
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

如果您將應用程式部署到運行 Nginx 的伺服器，可以使用以下設定檔作為配置網頁伺服器的起點。這份檔案很可能需要根據您的伺服器配置進行自訂。**如果您需要伺服器管理方面的協助，請考慮使用如 [Laravel Cloud](https://cloud.laravel.com) 這樣的全託管 Laravel 平台。**

請確保您的網頁伺服器如同下方設定一樣，將所有請求導向應用程式的 `public/index.php` 檔案。您絕不應該嘗試將 `index.php` 檔案移至專案根目錄，因為從專案根目錄提供應用程式服務會將許多敏感的設定檔暴露在公開的網際網路上：

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
        fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;
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

[FrankenPHP](https://frankenphp.dev/) 也可以用來提供 Laravel 應用程式的服務。FrankenPHP 是一個使用 Go 語言編寫的現代 PHP 應用程式伺服器。若要使用 FrankenPHP 提供 Laravel PHP 應用程式服務，您只需調用其 `php-server` 指令：

```shell
frankenphp php-server -r public/
```

若要利用 FrankenPHP 支援的更強大功能，例如其 [Laravel Octane](/docs/{{version}}/octane) 整合、HTTP/3、現代壓縮技術，或是將 Laravel 應用程式打包為獨立二進位檔的能力，請參閱 FrankenPHP 的 [Laravel 文件](https://frankenphp.dev/docs/laravel/)。


<a name="directory-permissions"></a>
### 目錄權限

Laravel 需要寫入 `bootstrap/cache` 與 `storage` 目錄，因此您應確保網頁伺服器的行程擁有者具有寫入這些目錄的權限。


<a name="optimization"></a>
## 優化

將應用程式部署到正式環境時，有許多檔案應該被快取，包括設定、事件、路由與視圖。Laravel 提供了一個便捷的 `optimize` Artisan 指令，可以快取所有這些檔案。此指令通常應作為應用程式部署流程的一部分來執行：

```shell
php artisan optimize
```

`optimize:clear` 方法可用於移除由 `optimize` 指令產生的所有快取檔案，以及預設快取驅動程式中的所有鍵值：

```shell
php artisan optimize:clear
```

在接下來的文件中，我們將討論 `optimize` 指令所執行的各個細粒度優化指令。


<a name="optimizing-configuration-loading"></a>
### 快取配置

將應用程式部署到正式環境時，您應確保在部署流程中執行 `config:cache` Artisan 指令：

```shell
php artisan config:cache
```

此指令會將所有 Laravel 的設定檔合併成單一的快取檔案，這大大減少了框架在載入設定值時存取檔案系統的次數。

> [!WARNING]
> 如果您在部署流程中執行 `config:cache` 指令，請務必確保您僅在設定檔中調用 `env` 函式。一旦設定被快取，`.env` 檔案將不會被載入，所有對 `.env` 變數的 `env` 函式調用都將回傳 `null`。


<a name="caching-events"></a>
### 快取事件

您應該在部署流程中快取應用程式自動發現的事件對監聽器對照表。這可以透過在部署時調用 `event:cache` Artisan 指令來完成：

```shell
php artisan event:cache
```


<a name="optimizing-route-loading"></a>
### 快取路由

如果您正在構建一個具有許多路由的大型應用程式，應確保在部署流程中執行 `route:cache` Artisan 指令：

```shell
php artisan route:cache
```

此指令將所有路由註冊簡化為快取檔案中的單一方法調用，在註冊數百個路由時能提高路由註冊的效能。


<a name="optimizing-view-loading"></a>
### 快取視圖

將應用程式部署到正式環境時，您應確保在部署流程中執行 `view:cache` Artisan 指令：

```shell
php artisan view:cache
```

此指令會預編譯所有 Blade 視圖，使其不再按需編譯，從而提高每個回傳視圖之請求的效能。


<a name="reloading-services"></a>
## 重新載入服務

> [!NOTE]
> 當部署到 [Laravel Cloud](https://cloud.laravel.com) 時，不需要使用 `reload` 指令，因為所有服務的平滑重新載入都會自動處理。

部署新版本的應用程式後，任何長時間運行的服務（例如佇列工作者、Laravel Reverb 或 Laravel Octane）都應該重新載入 / 重啟以使用新程式碼。Laravel 提供了一個單一的 `reload` Artisan 指令來終止這些服務：

```shell
php artisan reload
```

如果您沒有使用 [Laravel Cloud](https://cloud.laravel.com)，您應該手動配置一個行程監控工具，以偵測可重新載入的行程何時結束並自動重啟它們。


<a name="debug-mode"></a>
## 除錯模式

`config/app.php` 設定檔中的除錯選項決定了實際上會向使用者顯示多少關於錯誤的資訊。預設情況下，此選項設定為遵循 `APP_DEBUG` 環境變數的值，該變數儲存在應用程式的 `.env` 檔案中。

> [!WARNING]
> **在您的正式環境中，此值應始終為 `false`。如果在正式環境中將 `APP_DEBUG` 變數設定為 `true`，您將面臨向應用程式終端使用者洩漏敏感設定值的風險。**

<a name="the-health-route"></a>
## 健康檢查路由

Laravel 包含一個內建的健康檢查路由，可用於監控應用程式的狀態。在正式環境中，此路由可用於向運行時間監控工具 (uptime monitor)、負載平衡器或如 Kubernetes 等編排系統報告應用程式的狀態。

預設情況下，健康檢查路由位於 `/up`，如果應用程式在沒有例外狀況的情況下啟動，將返回 200 HTTP 回應。否則，將返回 500 HTTP 回應。您可以在應用程式的 `bootstrap/app` 檔案中設定此路由的 URI：

```php
->withRouting(
    web: __DIR__.'/../routes/web.php',
    commands: __DIR__.'/../routes/console.php',
    health: '/up', // [tl! remove]
    health: '/status', // [tl! add]
)
```

當對此路由發出 HTTP 請求時，Laravel 還會發送一個 `Illuminate\Foundation\Events\DiagnosingHealth` 事件，讓您可以針對應用程式執行額外的健康檢查。在此事件的 [監聽器 (listener)](/docs/{{version}}/events) 中，您可以檢查應用程式的資料庫或快取狀態。如果您發現應用程式有問題，只需在監聽器中拋出例外狀況即可。


<a name="deploying-with-cloud-or-forge"></a>
## 使用 Laravel Cloud 或 Forge 部署


<a name="laravel-cloud"></a>
#### Laravel Cloud

如果您想要一個專為 Laravel 調校、完全託管且支援自動擴展的部署平台，請參考 [Laravel Cloud](https://cloud.laravel.com)。Laravel Cloud 是一個強大的 Laravel 部署平台，提供託管的運算、資料庫、快取和物件儲存。

在 Cloud 上啟動您的 Laravel 應用程式，體驗可擴展的簡潔之美。Laravel Cloud 由 Laravel 的創作者精心調校，能與框架無縫協作，讓您可以像往常一樣繼續開發您的 Laravel 應用程式。


<a name="laravel-forge"></a>
#### Laravel Forge

如果您偏好自行管理伺服器，但不擅長配置運行強大 Laravel 應用程式所需的所有各種服務，[Laravel Forge](https://forge.laravel.com) 是一個專為 Laravel 應用程式設計的 VPS 伺服器管理平台。

Laravel Forge 可以在各種基礎設施提供商上建立伺服器，例如 DigitalOcean、Linode、AWS 等。此外，Forge 會安裝並管理建構強大 Laravel 應用程式所需的所有工具，例如 Nginx、MySQL、Redis、Memcached、Beanstalk 等。