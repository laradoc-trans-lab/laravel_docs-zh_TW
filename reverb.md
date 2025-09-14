# Laravel Reverb

- [簡介](#introduction)
- [安裝](#installation)
- [設定](#configuration)
    - [應用程式憑證](#application-credentials)
    - [允許的來源](#allowed-origins)
    - [額外應用程式](#additional-applications)
    - [SSL](#ssl)
- [執行伺服器](#running-server)
    - [偵錯](#debugging)
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

[Laravel Reverb](https://github.com/laravel/reverb) 將極速且可擴展的即時 WebSocket 通訊直接帶入您的 Laravel 應用程式，並與 Laravel 現有的 [事件廣播工具](/docs/{{version}}/broadcasting) 套件無縫整合。

<a name="installation"></a>
## 安裝

您可以使用 `install:broadcasting` Artisan 命令來安裝 Reverb：

```shell
php artisan install:broadcasting
```

<a name="configuration"></a>
## 設定

在幕後，`install:broadcasting` Artisan 命令會執行 `reverb:install` 命令，這將以一組合理的預設設定選項來安裝 Reverb。如果您想進行任何設定變更，可以透過更新 Reverb 的環境變數或更新 `config/reverb.php` 設定檔來完成。

<a name="application-credentials"></a>
### 應用程式憑證

為了建立與 Reverb 的連線，客戶端與伺服器之間必須交換一組 Reverb「應用程式」憑證。這些憑證在伺服器上設定，用於驗證來自客戶端的請求。您可以使用以下環境變數來定義這些憑證：

```ini
REVERB_APP_ID=my-app-id
REVERB_APP_KEY=my-app-key
REVERB_APP_SECRET=my-app-secret
```

<a name="allowed-origins"></a>
### 允許的來源

您也可以透過更新 `config/reverb.php` 設定檔中 `apps` 區塊內的 `allowed_origins` 設定值，來定義客戶端請求可以來自的來源。任何來自未列在您允許來源中的請求都將被拒絕。您可以使用 `*` 來允許所有來源：

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

通常，Reverb 為其安裝的應用程式提供 WebSocket 伺服器。然而，單一 Reverb 安裝也可以服務多個應用程式。

例如，您可能希望維護一個單一的 Laravel 應用程式，透過 Reverb 為多個應用程式提供 WebSocket 連線。這可以透過在應用程式的 `config/reverb.php` 設定檔中定義多個 `apps` 來實現：

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

在大多數情況下，安全的 WebSocket 連線是由上游的網頁伺服器 (Nginx 等) 處理，然後請求才被代理到您的 Reverb 伺服器。

然而，有時讓 Reverb 伺服器直接處理安全連線會很有用，例如在本地開發期間。如果您正在使用 [Laravel Herd](https://herd.laravel.com) 的安全網站功能，或者您正在使用 [Laravel Valet](/docs/{{version}}/valet) 並已對您的應用程式執行 [secure 命令](/docs/{{version}}/valet#securing-sites)，您可以使用為您的網站生成的 Herd / Valet 憑證來保護您的 Reverb 連線。為此，請將 `REVERB_HOST` 環境變數設定為您網站的主機名稱，或在啟動 Reverb 伺服器時明確傳遞主機名稱選項：

```shell
php artisan reverb:start --host="0.0.0.0" --port=8080 --hostname="laravel.test"
```

由於 Herd 和 Valet 網域解析為 `localhost`，執行上述命令將使您的 Reverb 伺服器可透過安全 WebSocket 協定 (`wss`) 在 `wss://laravel.test:8080` 存取。

您也可以透過在應用程式的 `config/reverb.php` 設定檔中定義 `tls` 選項來手動選擇憑證。在 `tls` 選項陣列中，您可以提供 [PHP SSL context options](https://www.php.net/manual/en/context.ssl.php) 支援的任何選項：

```php
'options' => [
    'tls' => [
        'local_cert' => '/path/to/cert.pem'
    ],
],
```

<a name="running-server"></a>
## 執行伺服器

Reverb 伺服器可以使用 `reverb:start` Artisan 命令啟動：

```shell
php artisan reverb:start
```

預設情況下，Reverb 伺服器將在 `0.0.0.0:8080` 啟動，使其可從所有網路介面存取。

如果您需要指定自訂的主機或連接埠，可以在啟動伺服器時透過 `--host` 和 `--port` 選項來完成：

```shell
php artisan reverb:start --host=127.0.0.1 --port=9000
```

或者，您可以在應用程式的 `.env` 設定檔中定義 `REVERB_SERVER_HOST` 和 `REVERB_SERVER_PORT` 環境變數。

`REVERB_SERVER_HOST` 和 `REVERB_SERVER_PORT` 環境變數不應與 `REVERB_HOST` 和 `REVERB_PORT` 混淆。前者指定 Reverb 伺服器本身運行的主機和連接埠，而後者則指示 Laravel 將廣播訊息發送到何處。例如，在正式環境中，您可能將來自公開 Reverb 主機名稱在連接埠 `443` 上的請求路由到在 `0.0.0.0:8080` 運行的 Reverb 伺服器。在這種情況下，您的環境變數將定義如下：

```ini
REVERB_SERVER_HOST=0.0.0.0
REVERB_SERVER_PORT=8080

REVERB_HOST=ws.laravel.com
REVERB_PORT=443
```

<a name="debugging"></a>
### 偵錯

為了提高效能，Reverb 預設不會輸出任何偵錯資訊。如果您想查看通過 Reverb 伺服器的資料流，可以向 `reverb:start` 命令提供 `--debug` 選項：

```shell
php artisan reverb:start --debug
```

<a name="restarting"></a>
### 重新啟動

由於 Reverb 是一個長時間運行的程序，程式碼的變更不會反映出來，除非透過 `reverb:restart` Artisan 命令重新啟動伺服器。

`reverb:restart` 命令確保在停止伺服器之前，所有連線都會優雅地終止。如果您正在使用像 Supervisor 這樣的程序管理器來運行 Reverb，那麼在所有連線終止後，伺服器將由程序管理器自動重新啟動：

```shell
php artisan reverb:restart
```

<a name="monitoring"></a>
## 監控

Reverb 可以透過與 [Laravel Pulse](/docs/{{version}}/pulse) 的整合來進行監控。透過啟用 Reverb 的 Pulse 整合，您可以追蹤伺服器處理的連線數和訊息數。

要啟用此整合，您應該首先確保已 [安裝 Pulse](/docs/{{version}}/pulse#installation)。然後，將 Reverb 的任何記錄器新增到應用程式的 `config/pulse.php` 設定檔中：

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

接下來，將每個記錄器的 Pulse 卡片新增到您的 [Pulse 儀表板](/docs/{{version}}/pulse#dashboard-customization)：

```blade
<x-pulse>
    <livewire:reverb.connections cols="full" />
    <livewire:reverb.messages cols="full" />
    ...
</x-pulse>
```

連線活動是透過定期輪詢新更新來記錄的。為確保此資訊在 Pulse 儀表板上正確呈現，您必須在 Reverb 伺服器上運行 `pulse:check` 守護程序。如果您在 [水平擴展](#scaling) 配置中運行 Reverb，則應僅在其中一台伺服器上運行此守護程序。

<a name="production"></a>
## 在正式環境中執行 Reverb

由於 WebSocket 伺服器的長時間運行特性，您可能需要對伺服器和託管環境進行一些優化，以確保您的 Reverb 伺服器能夠有效處理伺服器上可用資源的最佳連線數。

> [!NOTE]
> 如果您的網站由 [Laravel Forge](https://forge.laravel.com) 管理，您可以直接從「應用程式」面板自動優化您的伺服器以用於 Reverb。透過啟用 Reverb 整合，Forge 將確保您的伺服器已準備好用於正式環境，包括安裝任何所需的擴充功能並增加允許的連線數。

<a name="open-files"></a>
### 開啟檔案

每個 WebSocket 連線都保留在記憶體中，直到客戶端或伺服器斷開連線。在 Unix 和類 Unix 環境中，每個連線都由一個檔案表示。然而，作業系統和應用程式層級通常對允許的開啟檔案數有限制。

<a name="operating-system"></a>
#### 作業系統

在基於 Unix 的作業系統上，您可以使用 `ulimit` 命令來確定允許的開啟檔案數：

```shell
ulimit -n
```

此命令將顯示不同使用者允許的開啟檔案限制。您可以透過編輯 `/etc/security/limits.conf` 檔案來更新這些值。例如，將 `forge` 使用者的最大開啟檔案數更新為 10,000 將如下所示：

```ini
# /etc/security/limits.conf
forge        soft  nofile  10000
forge        hard  nofile  10000
```

<a name="event-loop"></a>
### 事件迴圈

在底層，Reverb 使用 ReactPHP 事件迴圈來管理伺服器上的 WebSocket 連線。預設情況下，此事件迴圈由 `stream_select` 提供支援，它不需要任何額外的擴充功能。然而，`stream_select` 通常限制為 1,024 個開啟檔案。因此，如果您計劃處理超過 1,000 個並發連線，您將需要使用不受相同限制約束的替代事件迴圈。

Reverb 將在可用時自動切換到由 `ext-uv` 提供支援的迴圈。此 PHP 擴充功能可透過 PECL 安裝：

```shell
pecl install uv
```

<a name="web-server"></a>
### 網頁伺服器

在大多數情況下，Reverb 在伺服器上的一個非面向網路的連接埠上運行。因此，為了將流量路由到 Reverb，您應該配置一個反向代理。假設 Reverb 在主機 `0.0.0.0` 和連接埠 `8080` 上運行，並且您的伺服器使用 Nginx 網頁伺服器，則可以使用以下 Nginx 網站設定為您的 Reverb 伺服器定義反向代理：

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
> Reverb 監聽 `/app` 上的 WebSocket 連線並處理 `/apps` 上的 API 請求。您應該確保處理 Reverb 請求的網頁伺服器可以服務這兩個 URI。如果您正在使用 [Laravel Forge](https://forge.laravel.com) 來管理您的伺服器，您的 Reverb 伺服器將預設正確配置。

通常，網頁伺服器會配置為限制允許的連線數，以防止伺服器過載。要將 Nginx 網頁伺服器上允許的連線數增加到 10,000，應更新 `nginx.conf` 檔案的 `worker_rlimit_nofile` 和 `worker_connections` 值：

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

上述配置將允許每個程序最多產生 10,000 個 Nginx worker。此外，此配置將 Nginx 的開啟檔案限制設定為 10,000。

<a name="ports"></a>
### 連接埠

基於 Unix 的作業系統通常會限制伺服器上可以開啟的連接埠數量。您可以透過以下命令查看當前允許的範圍：

```shell
cat /proc/sys/net/ipv4/ip_local_port_range
# 32768	60999
```

上述輸出顯示伺服器最多可以處理 28,231 (60,999 - 32,768) 個連線，因為每個連線都需要一個空閒連接埠。儘管我們建議 [水平擴展](#scaling) 以增加允許的連線數，但您可以透過更新伺服器 `/etc/sysctl.conf` 設定檔中的允許連接埠範圍來增加可用開啟連接埠的數量。

<a name="process-management"></a>
### 程序管理

在大多數情況下，您應該使用像 Supervisor 這樣的程序管理器來確保 Reverb 伺服器持續運行。如果您正在使用 Supervisor 來運行 Reverb，您應該更新伺服器 `supervisor.conf` 檔案的 `minfds` 設定，以確保 Supervisor 能夠開啟處理與 Reverb 伺服器連線所需的檔案：

```ini
[supervisord]
...
minfds=10000
```

<a name="scaling"></a>
### 擴展

如果您需要處理比單一伺服器允許的更多連線，您可以水平擴展您的 Reverb 伺服器。利用 Redis 的發布/訂閱功能，Reverb 能夠管理跨多個伺服器的連線。當您的應用程式的其中一個 Reverb 伺服器收到訊息時，該伺服器將使用 Redis 將傳入訊息發布到所有其他伺服器。

要啟用水平擴展，您應該在應用程式的 `.env` 設定檔中將 `REVERB_SCALING_ENABLED` 環境變數設定為 `true`：

```env
REVERB_SCALING_ENABLED=true
```

接下來，您應該有一個專用的中央 Redis 伺服器，所有 Reverb 伺服器都將與之通訊。Reverb 將使用為您的應用程式配置的 [預設 Redis 連線](/docs/{{version}}/redis#configuration) 來將訊息發布到所有 Reverb 伺服器。

一旦您啟用了 Reverb 的擴展選項並配置了 Redis 伺服器，您只需在能夠與您的 Redis 伺服器通訊的多個伺服器上調用 `reverb:start` 命令即可。這些 Reverb 伺服器應該放置在負載平衡器後面，該負載平衡器將傳入請求均勻地分發到各個伺服器。

