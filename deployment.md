# 部署

- [介紹](#introduction)
- [伺服器需求](#server-requirements)
- [伺服器設定](#server-configuration)
    - [Nginx](#nginx)
    - [FrankenPHP](#frankenphp)
    - [目錄權限](#directory-permissions)
- [優化](#optimization)
    - [快取設定](#optimizing-configuration-loading)
    - [快取事件](#caching-events)
    - [快取路由](#optimizing-route-loading)
    - [快取視圖](#optimizing-view-loading)
- [重新載入服務](#reloading-services)
- [除錯模式](#debug-mode)
- [健康檢查路由](#the-health-route)
- [使用 Laravel Cloud 或 Forge 部署](#deploying-with-cloud-or-forge)

<a name="introduction"></a>
## 介紹

當您準備將 Laravel 應用程式部署到正式環境時，有一些重要的事情可以做，以確保您的應用程式盡可能高效地運行。在本文件中，我們將介紹一些確保您的 Laravel 應用程式正確部署的重要起點。

<a name="server-requirements"></a>
## 伺服器需求

Laravel 框架有一些系統需求。您應確保您的網路伺服器具備以下最低 PHP 版本和擴充功能：

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
## 伺服器設定

<a name="nginx"></a>
### Nginx

如果您將應用程式部署到運行 Nginx 的伺服器上，可以使用以下設定檔作為配置網路伺服器的起點。此檔案很可能需要根據您的伺服器配置進行客製化。**如果您需要管理伺服器的協助，請考慮使用像 [Laravel Cloud](https://cloud.laravel.com) 這樣完全託管的 Laravel 平台。**

請確保您的網路伺服器（如下方配置所示）將所有請求導向您應用程式的 `public/index.php` 檔案。您絕不應嘗試將 `index.php` 檔案移動到專案的根目錄，因為從專案根目錄提供應用程式服務將會把許多敏感的設定檔暴露給公共網路：

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

[FrankenPHP](https://frankenphp.dev/) 也可用於提供您的 Laravel 應用程式。FrankenPHP 是一個用 Go 編寫的現代 PHP 應用程式伺服器。要使用 FrankenPHP 提供 Laravel PHP 應用程式服務，您只需調用其 `php-server` 指令：

```shell
frankenphp php-server -r public/
```

要利用 FrankenPHP 支援的更強大功能，例如其 [Laravel Octane](/docs/{{version}}/octane) 整合、HTTP/3、現代壓縮，或將 Laravel 應用程式打包為獨立二進位檔的能力，請查閱 FrankenPHP 的 [Laravel 文件](https://frankenphp.dev/docs/laravel/)。

<a name="directory-permissions"></a>
### 目錄權限

Laravel 需要寫入 `bootstrap/cache` 和 `storage` 目錄，因此您應確保網路伺服器處理程序擁有者具有寫入這些目錄的權限。

<a name="optimization"></a>
## 優化

當您將應用程式部署到正式環境時，有許多檔案應進行快取，包括您的設定、事件、路由和視圖。Laravel 提供了一個方便的 `optimize` Artisan 指令，它會快取所有這些檔案。此指令通常應作為您應用程式部署過程的一部分來調用：

```shell
php artisan optimize
```

`optimize:clear` 方法可用於移除由 `optimize` 指令產生的所有快取檔案，以及預設快取驅動程式中的所有鍵值：

```shell
php artisan optimize:clear
```

在以下文件中，我們將討論 `optimize` 指令所執行的每個細粒度優化指令。

<a name="optimizing-configuration-loading"></a>
### 快取設定

當您將應用程式部署到正式環境時，您應該確保在部署過程中執行 `config:cache` Artisan 指令：

```shell
php artisan config:cache
```

此指令會將所有 Laravel 的設定檔合併成一個單一的快取檔案，這大大減少了框架在載入設定值時必須對檔案系統進行的存取次數。

> [!WARNING]
> 如果您在部署過程中執行 `config:cache` 指令，應確保只在您的設定檔中呼叫 `env` 函數。一旦設定被快取，`.env` 檔案將不會被載入，並且所有對 `.env` 變數的 `env` 函數呼叫都將返回 `null`。

<a name="caching-events"></a>
### 快取事件

您應在部署過程中快取應用程式的自動發現事件到監聽器的映射。這可以透過在部署期間調用 `event:cache` Artisan 指令來完成：

```shell
php artisan event:cache
```

<a name="optimizing-route-loading"></a>
### 快取路由

如果您正在建構一個擁有許多路由的大型應用程式，您應該確保在部署過程中執行 `route:cache` Artisan 指令：

```shell
php artisan route:cache
```

此指令會將您所有的路由註冊縮減為快取檔案中的單一方法呼叫，從而提高了註冊數百個路由時的效能。

<a name="optimizing-view-loading"></a>
### 快取視圖

當您將應用程式部署到正式環境時，您應該確保在部署過程中執行 `view:cache` Artisan 指令：

```shell
php artisan view:cache
```

此指令會預編譯所有 Blade 視圖，使它們無需按需編譯，從而提高每個返回視圖的請求效能。

<a name="reloading-services"></a>
## 重新載入服務

> [!NOTE]
> 部署到 [Laravel Cloud](https://cloud.laravel.com) 時，不需要使用 `reload` 指令，因為所有服務的優雅重新載入都是自動處理的。

部署新版應用程式後，任何長時間運行的服務，例如佇列工作者、Laravel Reverb 或 Laravel Octane，都應該重新載入/重新啟動以使用新程式碼。Laravel 提供了一個單一的 `reload` Artisan 指令，它將終止這些服務：

```shell
php artisan reload
```

如果您沒有使用 [Laravel Cloud](https://cloud.laravel.com)，您應該手動配置一個程序監控器，它可以在您的可重新載入程序退出時檢測到並自動重新啟動它們。

<a name="debug-mode"></a>
## 除錯模式

您的 `config/app.php` 設定檔中的除錯選項，決定了實際向使用者顯示多少錯誤資訊。預設情況下，此選項設定為遵循 `APP_DEBUG` 環境變數的值，該變數儲存在您應用程式的 `.env` 檔案中。

> [!WARNING]
> **在您的正式環境中，此值應始終為 `false`。如果 `APP_DEBUG` 變數在正式環境中設定為 `true`，您將面臨將敏感設定值暴露給應用程式終端使用者的風險。**

<a name="the-health-route"></a>
## 健康檢查路由

Laravel 內建健康檢查路由，可用於監控應用程式的狀態。在正式環境中，此路由可用於向稼動時間監控器、負載平衡器或如 Kubernetes 等協調系統回報應用程式狀態。

預設情況下，健康檢查路由會在 `/up` 處提供服務，若應用程式啟動時沒有例外狀況，則會回傳 200 HTTP 回應。否則，將回傳 500 HTTP 回應。您可以在應用程式的 `bootstrap/app` 檔案中設定此路由的 URI：

```php
->withRouting(
    web: __DIR__.'/../routes/web.php',
    commands: __DIR__.'/../routes/console.php',
    health: '/up', // [tl! remove]
    health: '/status', // [tl! add]
)
```

當對此路由發出 HTTP 請求時，Laravel 也會發送一個 `Illuminate\Foundation\Events\DiagnosingHealth` 事件，讓您可以執行與應用程式相關的額外健康檢查。在此事件的 [監聽器](/docs/{{version}}/events) 中，您可以檢查應用程式的資料庫或快取狀態。如果您發現應用程式有問題，可以直接從監聽器中拋出一個例外。

<a name="deploying-with-cloud-or-forge"></a>
## 使用 Laravel Cloud 或 Forge 部署

<a name="laravel-cloud"></a>
#### Laravel Cloud

如果您想要一個專為 Laravel 調校、完全託管且自動擴展的部署平台，請查看 [Laravel Cloud](https://cloud.laravel.com)。Laravel Cloud 是一個強大的 Laravel 部署平台，提供託管的運算、資料庫、快取和物件儲存。

在 Cloud 上啟動您的 Laravel 應用程式，並愛上其可擴展的簡潔性。Laravel Cloud 由 Laravel 的創造者精心調校，能與框架無縫協同運作，讓您可以像往常一樣撰寫您的 Laravel 應用程式。

<a name="laravel-forge"></a>
#### Laravel Forge

如果您偏好管理自己的伺服器，但不熟悉設定執行強大 Laravel 應用程式所需的各種服務，[Laravel Forge](https://forge.laravel.com) 是一個用於 Laravel 應用程式的 VPS 伺服器管理平台。

Laravel Forge 可以在各種基礎設施供應商上建立伺服器，例如 DigitalOcean、Linode、AWS 等。此外，Forge 會安裝並管理建立強大 Laravel 應用程式所需的所有工具，例如 Nginx、MySQL、Redis、Memcached、Beanstalk 等。