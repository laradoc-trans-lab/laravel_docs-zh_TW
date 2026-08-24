# 部署

- [介紹](#introduction)
- [伺服器需求](#server-requirements)
- [伺服器設定](#server-configuration)
    - [Nginx](#nginx)
    - [FrankenPHP](#frankenphp)
    - [目錄權限](#directory-permissions)
- [最佳化](#optimization)
    - [快取設定](#optimizing-configuration-loading)
    - [快取事件](#caching-events)
    - [快取路由](#optimizing-route-loading)
    - [快取視圖](#optimizing-view-loading)
- [重新載入服務](#reloading-services)
- [除錯模式](#debug-mode)
- [健康檢查路由](#the-health-route)
- [使用 Laravel Cloud 或 Forge 進行部署](#deploying-with-cloud-or-forge)

<a name="introduction"></a>
## 介紹

當你準備好將 Laravel 應用程式部署到正式環境時，有一些重要事項可以確保你的應用程式盡可能高效地運作。在本文件中，我們將介紹一些絕佳的起點，以確保你的 Laravel 應用程式能被正確部署。


<a name="server-requirements"></a>
## 伺服器需求

Laravel 框架有少許系統需求。你應確保你的網頁伺服器具備以下最低 PHP 版本與擴充套件：

<div class="content-list" markdown="1">

- PHP >= 8.3
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
## 伺服器設定


<a name="nginx"></a>
### Nginx

如果你要將應用程式部署到執行 Nginx 的伺服器，可以使用以下設定檔作為設定網頁伺服器的起點。很可能需要根據伺服器的設定自訂此檔案。**如果你希望在管理伺服器方面獲得協助，請考慮使用像 [Laravel Cloud](https://cloud.laravel.com) 這樣的全代管 Laravel 平台。**

請確保像下方的設定一樣，讓你的網頁伺服器將所有請求導向至應用程式的 `public/index.php` 檔案。切勿嘗試將 `index.php` 檔案移至專案根目錄，因為從專案根目錄提供應用程式服務會將許多敏感的設定檔暴露給公開的網際網路：

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
        fastcgi_buffer_size 32k;
        fastcgi_buffers 8 32k;
        fastcgi_busy_buffers_size 64k;
        fastcgi_hide_header X-Powered-By;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```


<a name="frankenphp"></a>
### FrankenPHP

[FrankenPHP](https://frankenphp.dev/) 也可以用來執行你的 Laravel 應用程式。FrankenPHP 是一個使用 Go 語言編寫的現代 PHP 應用程式伺服器。要使用 FrankenPHP 提供 Laravel PHP 應用程式服務，只需執行其 `php-server` 指令：

```shell
frankenphp php-server -r public/
```

若要運用 FrankenPHP 支援的更強大功能，例如其與 [Laravel Octane](/docs/{{version}}/octane) 的整合、HTTP/3、現代壓縮技術，或是將 Laravel 應用程式封裝為獨立二進位檔的功能，請參閱 FrankenPHP 的 [Laravel 文件](https://frankenphp.dev/docs/laravel/)。


<a name="directory-permissions"></a>
### 目錄權限

Laravel 需要寫入 `bootstrap/cache` 和 `storage` 目錄，因此你應確保網頁伺服器行程的擁有者具備寫入這些目錄的權限。


<a name="optimization"></a>
## 最佳化

當將應用程式部署到正式環境時，有許多檔案應該被快取，包括設定、事件、路由以及視圖。Laravel 提供了一個單一且便利的 `optimize` Artisan 指令，可以快取所有這些檔案。此指令通常應該作為應用程式部署流程的一部分來執行：

```shell
php artisan optimize
```

`optimize:clear` 方法可用於移除由 `optimize` 指令產生的所有快取檔案，以及預設快取驅動中的所有鍵值：

```shell
php artisan optimize:clear
```

在接下來的文件中，我們將討論由 `optimize` 指令所執行的每個細項最佳化指令。


<a name="optimizing-configuration-loading"></a>
### 快取設定

當將應用程式部署到正式環境時，你應該確保在部署流程中執行 `config:cache` Artisan 指令：

```shell
php artisan config:cache
```

此指令會將 Laravel 的所有設定檔合併為單一快取檔案，這能大幅減少框架在載入設定值時對檔案系統的讀取次數。

> [!WARNING]
> 如果你在部署流程中執行 `config:cache` 指令，請務必確認你只在設定檔內呼叫 `env` 函式。一旦設定被快取後，`.env` 檔案將不會被載入，所有針對 `.env` 變數呼叫 `env` 函式的回傳值都將是 `null`。


<a name="caching-events"></a>
### 快取事件

你應該在部署流程中快取應用程式自動發現的事件與監聽器對應關係。這可以透過在部署期間執行 `event:cache` Artisan 指令來完成：

```shell
php artisan event:cache
```


<a name="optimizing-route-loading"></a>
### 快取路由

如果你正在建構一個包含許多路由的大型應用程式，你應該確保在部署流程中執行 `route:cache` Artisan 指令：

```shell
php artisan route:cache
```

此指令會將所有的路由註冊縮減為快取檔案中的單一方法呼叫，從而在註冊數百個路由時提升路由註冊的效能。


<a name="optimizing-view-loading"></a>
### 快取視圖

當將應用程式部署到正式環境時，你應該確保在部署流程中執行 `view:cache` Artisan 指令：

```shell
php artisan view:cache
```

此指令會預先編譯所有 Blade 視圖，使其不會在請求時才即時編譯，從而提升每個回傳視圖的請求效能。


<a name="reloading-services"></a>
## 重新載入服務

> [!NOTE]
> 部署到 [Laravel Cloud](https://cloud.laravel.com) 時，不需要使用 `reload` 指令，因為系統會自動處理所有服務的平滑重新載入。

在部署應用程式的新版本後，任何長時間執行的服務（例如佇列 Worker、Laravel Reverb 或 Laravel Octane）都應該重新載入 / 重新啟動以使用新程式碼。Laravel 提供了一個單一的 `reload` Artisan 指令來終止這些服務：

```shell
php artisan reload
```

如果你沒有使用 [Laravel Cloud](https://cloud.laravel.com)，則應手動設定行程監控程式，以便在可重新載入的行程結束時偵測到並自動重新啟動它們。


<a name="debug-mode"></a>
## 除錯模式

在 `config/app.php` 設定檔中的 debug 選項決定了實際上向使用者顯示多少關於錯誤的資訊。預設情況下，此選項設定為遵循儲存在應用程式 `.env` 檔案中的 `APP_DEBUG` 環境變數值。

> [!WARNING]
> **在正式環境中，此值應始終為 `false`。如果在正式環境中將 `APP_DEBUG` 變數設定為 `true`，你將面臨將敏感設定值暴露給應用程式終端使用者的風險。**

<a name="the-health-route"></a>
## 健康檢查路由

Laravel 包含一個內建的健康檢查路由，可用於監控應用程式的狀態。在正式環境中，此路由可用於向正常運行時間監控工具（Uptime Monitor）、負載平衡器（Load Balancer）或 Kubernetes 等編排系統回報應用程式的狀態。

預設情況下，健康檢查路由設定於 `/up`，如果應用程式順利啟動且沒有拋出例外，將會回傳 200 HTTP 回應。否則，將會回傳 500 HTTP 回應。您可以在應用程式的 `bootstrap/app` 檔案中設定此路由的 URI：

```php
->withRouting(
    web: __DIR__.'/../routes/web.php',
    commands: __DIR__.'/../routes/console.php',
    health: '/up', // [tl! remove]
    health: '/status', // [tl! add]
)
```

當有 HTTP 請求發送到此路由時，Laravel 還會發送 `Illuminate\Foundation\Events\DiagnosingHealth` 事件，讓您可以執行與應用程式相關的其他健康檢查。在此事件的[監聽器](/docs/{{version}}/events)中，您可以檢查應用程式的資料庫或快取狀態。如果您偵測到應用程式有問題，只需直接從監聽器拋出例外即可。

<a name="deploying-with-cloud-or-forge"></a>
## 使用 Laravel Cloud 或 Forge 進行部署

<a name="laravel-cloud"></a>
#### Laravel Cloud

如果您想要一個為 Laravel 量身打造、全託管且具備自動擴展功能的部署平台，請參考 [Laravel Cloud](https://cloud.laravel.com)。Laravel Cloud 是一個強大的 Laravel 部署平台，提供託管運算、資料庫、快取以及物件儲存服務。

在 Cloud 上啟動您的 Laravel 應用程式，體驗兼具擴展性與簡潔的優勢。Laravel Cloud 由 Laravel 核心團隊精心調校，能與框架無縫整合，讓您能夠完全按照習慣的方式繼續開發 Laravel 應用程式。

<a name="laravel-forge"></a>
#### Laravel Forge

如果您偏好管理自己的伺服器，但對於設定執行強大 Laravel 應用程式所需的各種服務感到繁瑣，[Laravel Forge](https://forge.laravel.com) 是一個專為 Laravel 應用程式打造的 VPS 伺服器管理平台。

Laravel Forge 可以在各種基礎架構提供商（例如 DigitalOcean、Linode、AWS 等）上建立伺服器。此外，Forge 還會安裝並管理建構健全 Laravel 應用程式所需的所有工具，例如 Nginx、MySQL、Redis、Memcached、Beanstalk 等。