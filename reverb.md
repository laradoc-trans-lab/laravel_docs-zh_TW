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
- [在生產環境中執行 Reverb](#production)
    - [開放檔案](#open-files)
    - [事件迴圈](#event-loop)
    - [網頁伺服器](#web-server)
    - [連接埠](#ports)
    - [處理程序管理](#process-management)
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

在幕後，`install:broadcasting` Artisan 指令會執行 `reverb:install` 指令，它會以一組合理的預設設定選項來安裝 Reverb。如果您想進行任何設定變更，可以透過更新 Reverb 的環境變數或更新 `config/reverb.php` 設定檔來實現。

<a name="application-credentials"></a>
### 應用程式憑證

為了建立與 Reverb 的連線，客戶端和伺服器之間必須交換一組 Reverb「應用程式」憑證。這些憑證在伺服器上設定，並用於驗證來自客戶端的請求。您可以使用以下環境變數定義這些憑證：

```ini
REVERB_APP_ID=my-app-id
REVERB_APP_KEY=my-app-key
REVERB_APP_SECRET=my-app-secret
```

<a name="allowed-origins"></a>
### 允許的來源

您也可以透過更新 `config/reverb.php` 設定檔中 `apps` 區塊內的 `allowed_origins` 設定值，來定義客戶端請求可以來自哪些來源。任何來自未列於您允許來源的請求將會被拒絕。您可以使用 `*` 允許所有來源：

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

通常，Reverb 會為其所安裝的應用程式提供 WebSocket 伺服器。然而，使用單一 Reverb 安裝來服務多個應用程式是可行的。

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

在大多數情況下，安全的 WebSocket 連線由上游網頁伺服器 (Nginx 等) 處理，然後才將請求代理到您的 Reverb 伺服器。

然而，有時讓 Reverb 伺服器直接處理安全連線會很有用，例如在本地開發期間。如果您正在使用 [Laravel Herd 的](https://herd.laravel.com) 安全站點功能，或者您正在使用 [Laravel Valet](/docs/{{version}}/valet) 並已對您的應用程式執行 [secure 指令](/docs/{{version}}/valet#securing-sites)，您可以使用為您的站點生成的 Herd / Valet 憑證來保護您的 Reverb 連線。為此，請將 `REVERB_HOST` 環境變數設定為您站點的主機名稱，或在啟動 Reverb 伺服器時明確傳遞主機名稱選項：

```shell
php artisan reverb:start --host="0.0.0.0" --port=8080 --hostname="laravel.test"
```

由於 Herd 和 Valet 網域解析為 `localhost`，執行上述指令將導致您的 Reverb 伺服器可透過安全的 WebSocket 協定 (`wss`) 在 `wss://laravel.test:8080` 存取。

您也可以透過在應用程式的 `config/reverb.php` 設定檔中定義 `tls` 選項來手動選擇憑證。在 `tls` 選項陣列中，您可以提供 [PHP 的 SSL 內容選項](https://www.php.net/manual/en/context.ssl.php) 支援的任何選項：

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

如果您需要指定自訂主機或連接埠，可以在啟動伺服器時透過 `--host` 和 `--port` 選項來實現：

```shell
php artisan reverb:start --host=127.0.0.1 --port=9000
```

或者，您也可以在應用程式的 `.env` 設定檔中定義 `REVERB_SERVER_HOST` 和 `REVERB_SERVER_PORT` 環境變數。

`REVERB_SERVER_HOST` 和 `REVERB_SERVER_PORT` 環境變數不應與 `REVERB_HOST` 和 `REVERB_PORT` 混淆。前者指定 Reverb 伺服器本身運行的主機和連接埠，而後者則指示 Laravel 將廣播訊息傳送到何處。例如，在生產環境中，您可能希望將來自您公開 Reverb 主機名稱在連接埠 `443` 上的請求路由到運行於 `0.0.0.0:8080` 的 Reverb 伺服器。在這種情境下，您的環境變數將定義如下：

```ini
REVERB_SERVER_HOST=0.0.0.0
REVERB_SERVER_PORT=8080

REVERB_HOST=ws.laravel.com
REVERB_PORT=443
```

<a name="debugging"></a>
### 除錯

為了提升效能，Reverb 預設不會輸出任何除錯資訊。如果您想查看流經 Reverb 伺服器的資料流，可以向 `reverb:start` 指令提供 `--debug` 選項：

```shell
php artisan reverb:start --debug
```

<a name="restarting"></a>
### 重新啟動

由於 Reverb 是一個長時間執行的處理程序，您的程式碼變更將不會反映，除非透過 `reverb:restart` Artisan 指令重新啟動伺服器。

`reverb:restart` 指令確保在停止伺服器之前，所有連線都會優雅地終止。如果您正在使用像 Supervisor 這樣的處理程序管理器來執行 Reverb，那麼在所有連線終止後，伺服器將會由處理程序管理器自動重新啟動：

```shell
php artisan reverb:restart
```

<a name="monitoring"></a>
## 監控

Reverb 可以透過與 [Laravel Pulse](/docs/{{version}}/pulse) 的整合進行監控。透過啟用 Reverb 的 Pulse 整合，您可以追蹤伺服器正在處理的連線和訊息數量。

要啟用此整合，您應首先確保您已 [安裝 Pulse](/docs/{{version}}/pulse#installation)。然後，將 Reverb 的任何記錄器新增至您應用程式的 `config/pulse.php` 設定檔：

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

接下來，將每個記錄器的 Pulse 卡片新增至您的 [Pulse 儀表板](/docs/{{version}}/pulse#dashboard-customization)：

```blade
<x-pulse>
    <livewire:reverb.connections cols="full" />
    <livewire:reverb.messages cols="full" />
    ...
</x-pulse>
```

連線活動是透過定期輪詢新更新來記錄的。為確保此資訊在 Pulse 儀表板上正確呈現，您必須在 Reverb 伺服器上執行 `pulse:check` daemon。如果您在 [水平擴展](#scaling) 設定中運行 Reverb，則應僅在其中一台伺服器上運行此 daemon。

<a name="production"></a>
## 在生產環境中執行 Reverb

由於 WebSocket 伺服器具有長時間運作的特性，您可能需要對您的伺服器和託管環境進行一些優化，以確保您的 Reverb 伺服器能夠有效地處理最佳數量的連線，以應對伺服器上可用的資源。

> [!NOTE]
> 如果您的網站由 [Laravel Forge](https://forge.laravel.com) 管理，您可以直接從「應用程式」面板自動優化您的伺服器以支援 Reverb。透過啟用 Reverb 整合，Forge 將確保您的伺服器已準備好投入生產，包括安裝任何所需的擴充功能並增加允許的連線數。

<a name="open-files"></a>
### 開放檔案

每個 WebSocket 連線都會保留在記憶體中，直到客戶端或伺服器斷開連線。在 Unix 和類似 Unix 的環境中，每個連線都以檔案形式表示。然而，作業系統和應用程式層級通常對允許的開放檔案數量設有限制。

<a name="operating-system"></a>
#### 作業系統

在基於 Unix 的作業系統上，您可以使用 `ulimit` 指令來確定允許的開放檔案數量：

```shell
ulimit -n
```

此指令將顯示不同使用者允許的開放檔案限制。您可以透過編輯 `/etc/security/limits.conf` 檔案來更新這些值。例如，將 `forge` 使用者允許的最大開放檔案數量更新為 10,000 如下所示：

```ini
# /etc/security/limits.conf
forge        soft  nofile  10000
forge        hard  nofile  10000
```

<a name="event-loop"></a>
### 事件迴圈

Reverb 在底層使用 ReactPHP 事件迴圈來管理伺服器上的 WebSocket 連線。預設情況下，此事件迴圈由 `stream_select` 提供支援，這不需要任何額外的擴充功能。然而，`stream_select` 通常僅限於 1,024 個開放檔案。因此，如果您計劃處理超過 1,000 個並行連線，您將需要使用不受相同限制的其他事件迴圈。

Reverb 在可用時會自動切換到由 `ext-uv` 驅動的迴圈。這個 PHP 擴充功能可以透過 PECL 安裝：

```shell
pecl install uv
```

<a name="web-server"></a>
### 網頁伺服器

在大多數情況下，Reverb 在您伺服器上的一個非網路面向的連接埠上運行。因此，為了將流量路由到 Reverb，您應該設定一個反向代理。假設 Reverb 在主機 `0.0.0.0` 和連接埠 `8080` 上運行，並且您的伺服器使用 Nginx 網頁伺服器，則可以使用以下 Nginx 網站設定來為您的 Reverb 伺服器定義反向代理：

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
> Reverb 監聽 `/app` 上的 WebSocket 連線，並處理 `/apps` 上的 API 請求。您應該確保處理 Reverb 請求的網頁伺服器能夠服務這兩個 URI。如果您使用 [Laravel Forge](https://forge.laravel.com) 來管理您的伺服器，您的 Reverb 伺服器將預設正確設定。

通常，網頁伺服器會配置為限制允許的連線數量，以防止伺服器過載。為了將 Nginx 網頁伺服器上允許的連線數量增加到 10,000，應該更新 `nginx.conf` 檔案中的 `worker_rlimit_nofile` 和 `worker_connections` 值：

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

上述配置將允許每個程序最多生成 10,000 個 Nginx 工作程序。此外，此配置將 Nginx 的開放檔案限制設為 10,000。

<a name="ports"></a>
### 連接埠

基於 Unix 的作業系統通常限制伺服器上可以開放的連接埠數量。您可以使用以下指令查看當前允許的範圍：

```shell
cat /proc/sys/net/ipv4/ip_local_port_range
# 32768	60999
```

上述輸出顯示伺服器最多可以處理 28,231 (60,999 - 32,768) 個連線，因為每個連線都需要一個空閒連接埠。儘管我們建議透過[水平擴展](#scaling)來增加允許的連線數量，您也可以透過更新伺服器 `/etc/sysctl.conf` 設定檔案中的允許連接埠範圍來增加可用開放連接埠的數量。

<a name="process-management"></a>
### 處理程序管理

在大多數情況下，您應該使用如 Supervisor 等處理程序管理器來確保 Reverb 伺服器持續運行。如果您使用 Supervisor 來運行 Reverb，您應該更新伺服器 `supervisor.conf` 檔案中的 `minfds` 設定，以確保 Supervisor 能夠開放處理 Reverb 伺服器連線所需的檔案：

```ini
[supervisord]
...
minfds=10000
```

<a name="scaling"></a>
### 擴展

如果您需要處理比單一伺服器所能允許的更多連線，您可以水平擴展您的 Reverb 伺服器。利用 Redis 的發佈/訂閱功能，Reverb 能夠管理多個伺服器之間的連線。當其中一個應用程式的 Reverb 伺服器收到訊息時，該伺服器將使用 Redis 將傳入的訊息發佈到所有其他伺服器。

要啟用水平擴展，您應該在應用程式的 `.env` 設定檔案中將 `REVERB_SCALING_ENABLED` 環境變數設定為 `true`：

```env
REVERB_SCALING_ENABLED=true
```

接下來，您應該有一個專用、中央的 Redis 伺服器，所有 Reverb 伺服器都將與之通訊。Reverb 將使用[為您的應用程式設定的預設 Redis 連線](/docs/{{version}}/redis#configuration)來將訊息發佈到所有 Reverb 伺服器。

一旦您啟用了 Reverb 的擴展選項並設定了 Redis 伺服器，您只需在能夠與您的 Redis 伺服器通訊的多個伺服器上呼叫 `reverb:start` 指令。這些 Reverb 伺服器應放置在負載平衡器後面，該負載平衡器會將傳入請求均勻地分佈到各個伺服器。