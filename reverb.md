# Laravel Reverb

- [簡介](#introduction)
- [安裝](#installation)
- [設定](#configuration)
    - [應用程式憑證](#application-credentials)
    - [允許的來源](#allowed-origins)
    - [額外應用程式](#additional-applications)
    - [SSL](#ssl)
- [執行伺服器](#running-server)
    - [除錯](#debugging)
    - [重新啟動](#restarting)
- [監控](#monitoring)
- [在正式環境中執行 Reverb](#production)
    - [開啟檔案](#open-files)
    - [事件迴圈](#event-loop)
    - [網頁伺服器](#web-server)
    - [連接埠](#ports)
    - [程序管理](#process-management)
    - [擴展](#scaling)

<a name="introduction"></a>
## 簡介

[Laravel Reverb](https://github.com/laravel/reverb) 為您的 Laravel 應用程式帶來極速且可擴展的即時 WebSocket 通訊，並與 Laravel 現有的 [事件廣播工具](/docs/{{version}}/broadcasting) 套件提供無縫整合。


<a name="installation"></a>
## 安裝

您可以使用 `install:broadcasting` Artisan 指令來安裝 Reverb：

```shell
php artisan install:broadcasting
```


<a name="configuration"></a>
## 設定

在幕後，`install:broadcasting` Artisan 指令將會執行 `reverb:install` 指令，它會以一套合理的預設設定選項來安裝 Reverb。如果您想進行任何設定變更，可以透過更新 Reverb 的環境變數或更新 `config/reverb.php` 設定檔來完成。


<a name="application-credentials"></a>
### 應用程式憑證

為了建立與 Reverb 的連線，客戶端與伺服器之間必須交換一組 Reverb「應用程式」憑證。這些憑證在伺服器上進行設定，並用於驗證來自客戶端的請求。您可以使用以下環境變數來定義這些憑證：

```ini
REVERB_APP_ID=my-app-id
REVERB_APP_KEY=my-app-key
REVERB_APP_SECRET=my-app-secret
```


<a name="allowed-origins"></a>
### 允許的來源

您還可以透過更新 `config/reverb.php` 設定檔的 `apps` 區段中的 `allowed_origins` 設定值，來定義客戶端請求可能來自的來源。任何來自未列在您允許來源中的請求將會被拒絕。您可以使用 `*` 來允許所有來源：

```php
'apps' => [
    [
        'app_id' => 'my-app-id',
        'allowed_origins' => ['laravel.com'],
        // ...
    ]
]
```


<a name="additional-applications"></a>
### 額外應用程式

通常，Reverb 會為其安裝的應用程式提供 WebSocket 伺服器。然而，使用單一 Reverb 安裝來服務多個應用程式也是可能的。

例如，您可能希望維護一個單一的 Laravel 應用程式，它透過 Reverb 為多個應用程式提供 WebSocket 連線能力。這可以透過在您應用程式的 `config/reverb.php` 設定檔中定義多個 `apps` 來實現：

```php
'apps' => [
    [
        'app_id' => 'my-app-one',
        // ...
    ],
    [
        'app_id' => 'my-app-two',
        // ...
    ],
],
```


<a name="ssl"></a>
### SSL

在大多數情況下，安全的 WebSocket 連線由上游的網頁伺服器（Nginx 等）處理，然後請求才會被代理到您的 Reverb 伺服器。

然而，有時在本地開發期間，由 Reverb 伺服器直接處理安全連線會很有用。如果您正在使用 [Laravel Herd 的](https://herd.laravel.com) 安全網站功能，或者您正在使用 [Laravel Valet](/docs/{{version}}/valet) 並已對您的應用程式執行 [secure 指令](/docs/{{version}}/valet#securing-sites)，您可以使用為您的網站生成的 Herd / Valet 憑證來保護您的 Reverb 連線。為此，請將 `REVERB_HOST` 環境變數設定為您網站的主機名稱，或在啟動 Reverb 伺服器時明確傳遞 hostname 選項：

```shell
php artisan reverb:start --host="0.0.0.0" --port=8080 --hostname="laravel.test"
```

由於 Herd 和 Valet 網域解析為 `localhost`，執行上述指令將使您的 Reverb 伺服器可透過安全的 WebSocket 協定 (`wss`) 於 `wss://laravel.test:8080` 存取。

您也可以透過在應用程式的 `config/reverb.php` 設定檔中定義 `tls` 選項來手動選擇憑證。在 `tls` 選項陣列中，您可以提供 [PHP 的 SSL 上下文選項](https://www.php.net/manual/en/context.ssl.php) 所支援的任何選項：

```php
'options' => [
    'tls' => [
        'local_cert' => '/path/to/cert.pem'
    ],
],
```


<a name="running-server"></a>
## 執行伺服器

Reverb 伺服器可以使用 `reverb:start` Artisan 指令啟動：

```shell
php artisan reverb:start
```

預設情況下，Reverb 伺服器將在 `0.0.0.0:8080` 啟動，使其可從所有網路介面存取。

如果您需要指定自訂的主機或連接埠，可以在啟動伺服器時透過 `--host` 和 `--port` 選項來完成：

```shell
php artisan reverb:start --host=127.0.0.1 --port=9000
```

或者，您可以在應用程式的 `.env` 設定檔中定義 `REVERB_SERVER_HOST` 和 `REVERB_SERVER_PORT` 環境變數。

`REVERB_SERVER_HOST` 和 `REVERB_SERVER_PORT` 環境變數不應與 `REVERB_HOST` 和 `REVERB_PORT` 混淆。前者指定執行 Reverb 伺服器本身的主機和連接埠，而後者則指示 Laravel 將廣播訊息發送到何處。例如，在正式環境中，您可以將來自公開 Reverb 主機名稱上連接埠 `443` 的請求路由到在 `0.0.0.0:8080` 運作的 Reverb 伺服器。在這種情況下，您的環境變數將定義如下：

```ini
REVERB_SERVER_HOST=0.0.0.0
REVERB_SERVER_PORT=8080

REVERB_HOST=ws.laravel.com
REVERB_PORT=443
```


<a name="debugging"></a>
### 除錯

為了提高效能，Reverb 預設不會輸出任何除錯資訊。如果您想查看通過 Reverb 伺服器的資料流，您可以為 `reverb:start` 指令提供 `--debug` 選項：

```shell
php artisan reverb:start --debug
```


<a name="restarting"></a>
### 重新啟動

由於 Reverb 是一個長時間執行的程序，您的程式碼變更若未透過 `reverb:restart` Artisan 指令重新啟動伺服器，將不會反映出來。

`reverb:restart` 指令會確保所有連線在停止伺服器之前，都能夠優雅地終止。如果您使用 Supervisor 等程序管理器來執行 Reverb，則在所有連線終止後，伺服器將會由程序管理器自動重新啟動：

```shell
php artisan reverb:restart
```


<a name="monitoring"></a>
## 監控

Reverb 可以透過與 [Laravel Pulse](/docs/{{version}}/pulse) 的整合來監控。透過啟用 Reverb 的 Pulse 整合，您可以追蹤伺服器處理的連線數和訊息數。

若要啟用此整合，您應首先確保已 [安裝 Pulse](/docs/{{version}}/pulse#installation)。然後，將 Reverb 的任何記錄器新增至您應用程式的 `config/pulse.php` 設定檔中：

```php
use Laravel\Reverb\Pulse\Recorders\ReverbConnections;
use Laravel\Reverb\Pulse\Recorders\ReverbMessages;

'recorders' => [
    ReverbConnections::class => [
        'sample_rate' => 1,
    ],

    ReverbMessages::class => [
        'sample_rate' => 1,
    ],

    // ...
],
```

接著，將每個記錄器的 Pulse 卡片新增至您的 [Pulse 儀表板](/docs/{{version}}/pulse#dashboard-customization)：

```blade
<x-pulse>
    <livewire:reverb.connections cols="full" />
    <livewire:reverb.messages cols="full" />
    ...
</x-pulse>
```

連線活動是透過定期輪詢新更新來記錄的。為確保此資訊在 Pulse 儀表板上正確呈現，您必須在 Reverb 伺服器上執行 `pulse:check` 背景程式。如果您在 [水平擴展](#scaling) 的設定中執行 Reverb，則應僅在其中一台伺服器上執行此背景程式。

<a name="production"></a>
## 在正式環境中執行 Reverb

由於 WebSocket 伺服器是長時間執行的特性，您可能需要對伺服器和託管環境進行一些優化，以確保您的 Reverb 伺服器能夠有效處理伺服器可用資源的最佳連線數。

> [!NOTE]
> 如果您的網站由 [Laravel Forge](https://forge.laravel.com) 管理，您可以直接從「應用程式」面板中自動優化您的伺服器以運行 Reverb。透過啟用 Reverb 整合，Forge 將確保您的伺服器已準備好上線，包括安裝任何所需的擴充功能並增加允許的連線數。

<a name="open-files"></a>
### 開啟檔案

每個 WebSocket 連線都會保存在記憶體中，直到用戶端或伺服器斷開連線。在 Unix 和類 Unix 環境中，每個連線都由一個檔案表示。然而，作業系統和應用程式層級通常對允許開啟的檔案數量有限制。

<a name="operating-system"></a>
#### 作業系統

在基於 Unix 的作業系統上，您可以使用 `ulimit` 命令來確定允許開啟的檔案數量：

```shell
ulimit -n
```

此命令會顯示允許不同使用者開啟檔案的限制。您可以透過編輯 `/etc/security/limits.conf` 檔案來更新這些值。例如，將 `forge` 使用者的最大開啟檔案數量更新為 10,000 將會如下所示：

```ini
# /etc/security/limits.conf
forge        soft  nofile  10000
forge        hard  nofile  10000
```

<a name="event-loop"></a>
### 事件迴圈

在內部，Reverb 使用 ReactPHP 事件迴圈來管理伺服器上的 WebSocket 連線。預設情況下，此事件迴圈由 `stream_select` 提供支援，它不需要任何額外的擴充功能。然而，`stream_select` 通常僅限於 1,024 個開啟檔案。因此，如果您計畫處理超過 1,000 個並發連線，您將需要使用不受相同限制約束的替代事件迴圈。

當可用時，Reverb 將自動切換到由 `ext-uv` 支援的迴圈。這個 PHP 擴充功能可以透過 PECL 安裝：

```shell
pecl install uv
```


<a name="web-server"></a>
### 網頁伺服器

在大多數情況下，Reverb 在您的伺服器上執行於一個非網路對外開放的連接埠。因此，為了將流量路由到 Reverb，您應該配置一個反向代理。假設 Reverb 在主機 `0.0.0.0` 和連接埠 `8080` 上運行，並且您的伺服器使用 Nginx 網頁伺服器，則可以使用以下 Nginx 網站設定來為您的 Reverb 伺服器定義反向代理：

```nginx
server {
    ...

    location / {
        proxy_http_version 1.1;
        proxy_set_header Host $http_host;
        proxy_set_header Scheme $scheme;
        proxy_set_header SERVER_PORT $server_port;
        proxy_set_header REMOTE_ADDR $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";

        proxy_pass http://0.0.0.0:8080;
    }

    ...
}
```

> [!WARNING]
> Reverb 監聽 `/app` 上的 WebSocket 連線，並處理 `/apps` 上的 API 請求。您應確保處理 Reverb 請求的網頁伺服器可以服務這兩個 URI。如果您使用 [Laravel Forge](https://forge.laravel.com) 來管理您的伺服器，您的 Reverb 伺服器預設將被正確配置。

通常，網頁伺服器會配置為限制允許的連線數量，以防止伺服器過載。若要將 Nginx 網頁伺服器上允許的連線數量增加到 10,000，應更新 `nginx.conf` 檔案中的 `worker_rlimit_nofile` 和 `worker_connections` 值：

```nginx
user forge;
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;
worker_rlimit_nofile 10000;

events {
  worker_connections 10000;
  multi_accept on;
}
```

上述設定將允許每個程序產生最多 10,000 個 Nginx worker。此外，此設定將 Nginx 的開啟檔案限制設為 10,000。


<a name="ports"></a>
### 連接埠

基於 Unix 的作業系統通常會限制伺服器上可以開啟的連接埠數量。您可以透過以下指令查看目前允許的範圍：

```shell
cat /proc/sys/net/ipv4/ip_local_port_range

# 32768	60999
```

上述輸出顯示伺服器最多可以處理 28,231 (60,999 - 32,768) 個連線，因為每個連線都需要一個空閒連接埠。雖然我們建議透過[水平擴展](#scaling)來增加允許的連線數量，但您可以透過更新伺服器 `/etc/sysctl.conf` 設定檔中允許的連接埠範圍來增加可用開啟連接埠的數量。


<a name="process-management"></a>
### 程序管理

在大多數情況下，您應該使用像 Supervisor 這樣的程序管理器來確保 Reverb 伺服器持續運行。如果您使用 Supervisor 來執行 Reverb，您應該更新伺服器 `supervisor.conf` 檔案的 `minfds` 設定，以確保 Supervisor 能夠開啟處理 Reverb 伺服器連線所需的檔案：

```ini
[supervisord]
...
minfds=10000
```


<a name="scaling"></a>
### 擴展

如果您需要處理的連線數超過單一伺服器所允許的範圍，您可以水平擴展您的 Reverb 伺服器。利用 Redis 的發布/訂閱功能，Reverb 能夠跨多個伺服器管理連線。當您應用程式的其中一個 Reverb 伺服器收到訊息時，該伺服器將使用 Redis 將傳入訊息發布到所有其他伺服器。

要啟用水平擴展，您應該在應用程式的 `.env` 設定檔中將 `REVERB_SCALING_ENABLED` 環境變數設定為 `true`：

```env
REVERB_SCALING_ENABLED=true
```

接著，您應該有一個專用的中央 Redis 伺服器，所有 Reverb 伺服器都將與其通訊。Reverb 將使用[為您的應用程式配置的預設 Redis 連線](/docs/{{version}}/redis#configuration)來向所有 Reverb 伺服器發布訊息。

一旦您啟用了 Reverb 的擴展選項並配置了 Redis 伺服器，您只需在能夠與您的 Redis 伺服器通訊的多個伺服器上執行 `reverb:start` 命令。這些 Reverb 伺服器應該放置在負載平衡器後面，由其將傳入請求平均分配到各個伺服器。