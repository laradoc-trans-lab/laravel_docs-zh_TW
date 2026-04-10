# Laravel Reverb

- [簡介](#introduction)
- [安裝](#installation)
- [設定](#configuration)
    - [應用程式憑證](#application-credentials)
    - [允許的來源](#allowed-origins)
    - [額外的應用程式](#additional-applications)
    - [SSL](#ssl)
- [執行伺服器](#running-server)
    - [偵錯](#debugging)
    - [重新啟動](#restarting)
- [監控](#monitoring)
- [在正式環境執行 Reverb](#production)
    - [開啟檔案數](#open-files)
    - [事件迴圈](#event-loop)
    - [網頁伺服器](#web-server)
    - [連接埠](#ports)
    - [程序管理](#process-management)
    - [擴充](#scaling)
- [事件](#events)

<a name="introduction"></a>
## 簡介

[Laravel Reverb](https://github.com/laravel/reverb) 為您的 Laravel 應用程式直接帶來極速且可擴充的即時 WebSocket 通訊，並提供與 Laravel 現有的 [事件廣播工具 (Event Broadcasting Tools)](/docs/{{version}}/broadcasting) 套件的無縫整合。


<a name="installation"></a>
## 安裝

您可以使用 `install:broadcasting` Artisan 指令來安裝 Reverb：

```shell
php artisan install:broadcasting
```


<a name="configuration"></a>
## 設定

在幕後，`install:broadcasting` Artisan 指令會執行 `reverb:install` 指令，這將會以一組合理的預設設定選項來安裝 Reverb。如果您想進行任何設定更改，可以透過更新 Reverb 的環境變數或更新 `config/reverb.php` 設定檔來完成。


<a name="application-credentials"></a>
### 應用程式憑證

為了建立與 Reverb 的連線，必須在用戶端與伺服器之間交換一組 Reverb「應用程式」憑證。這些憑證是在伺服器上設定的，用於驗證來自用戶端的請求。您可以使用以下環境變數來定義這些憑證：

```ini
REVERB_APP_ID=my-app-id
REVERB_APP_KEY=my-app-key
REVERB_APP_SECRET=my-app-secret
```


<a name="allowed-origins"></a>
### 允許的來源

您也可以透過更新 `config/reverb.php` 設定檔中 `apps` 區段內的 `allowed_origins` 設定值，來定義允許發送用戶端請求的來源 (Origin)。任何來自未列在允許來源清單中的請求都將被拒絕。您可以使用 `*` 允許所有來源：

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
### 額外的應用程式

通常，Reverb 會為其安裝所在的應用程式提供 WebSocket 伺服器。然而，使用單個 Reverb 安裝來為多個應用程式提供服務是可行的。

例如，您可能希望維護一個單一的 Laravel 應用程式，並透過 Reverb 為多個應用程式提供 WebSocket 連線。這可以透過在應用程式的 `config/reverb.php` 設定檔中定義多個 `apps` 來達成：

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

在大多數情況下，安全的 WebSocket 連線在請求被代理到您的 Reverb 伺服器之前，會先由上游網頁伺服器 (Nginx 等) 處理。

然而，有時（例如在本地開發期間）讓 Reverb 伺服器直接處理安全連線會很有用。如果您正在使用 [Laravel Herd](https://herd.laravel.com) 的安全網站功能，或者您正在使用 [Laravel Valet](/docs/{{version}}/valet) 並已對您的應用程式執行了 [secure 指令](/docs/{{version}}/valet#securing-sites)，您可以使用為您的網站產生的 Herd / Valet 憑證來保護您的 Reverb 連線。為此，請將 `REVERB_HOST` 環境變數設定為您的網站主機名稱，或者在啟動 Reverb 伺服器時明確傳遞主機名稱選項：

```shell
php artisan reverb:start --host="0.0.0.0" --port=8080 --hostname="laravel.test"
```

由於 Herd 和 Valet 網域會解析為 `localhost`，執行上述指令將使您的 Reverb 伺服器可以透過安全 WebSocket 協定 (`wss`) 在 `wss://laravel.test:8080` 進行存取。

您也可以透過在應用程式的 `config/reverb.php` 設定檔中定義 `tls` 選項來手動選擇憑證。在 `tls` 選項陣列中，您可以提供 [PHP SSL 上下文選項 (SSL Context Options)](https://www.php.net/manual/en/context.ssl.php) 支援的任何選項：

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

預設情況下，Reverb 伺服器將在 `0.0.0.0:8080` 啟動，使其可以從所有網路介面存取。

如果您需要指定自定義主機或連接埠，可以在啟動伺服器時透過 `--host` 與 `--port` 選項來完成：

```shell
php artisan reverb:start --host=127.0.0.1 --port=9000
```

或者，您可以在應用程式的 `.env` 設定檔中定義 `REVERB_SERVER_HOST` 與 `REVERB_SERVER_PORT` 環境變數。

`REVERB_SERVER_HOST` 與 `REVERB_SERVER_PORT` 環境變數不應與 `REVERB_HOST` 及 `REVERB_PORT` 混淆。前者指定執行 Reverb 伺服器本身的主機和連接埠，而後者則指示 Laravel 將廣播訊息發送到何處。例如，在正式環境中，您可能會將來自連接埠 `443` 的公開 Reverb 主機名稱請求路由到運作於 `0.0.0.0:8080` 的 Reverb 伺服器。在這種情況下，您的環境變數將定義如下：

```ini
REVERB_SERVER_HOST=0.0.0.0
REVERB_SERVER_PORT=8080

REVERB_HOST=ws.laravel.com
REVERB_PORT=443
```


<a name="debugging"></a>
### 偵錯

為了提高效能，Reverb 預設不會輸出任何偵錯資訊。如果您想查看通過 Reverb 伺服器的數據流，可以在執行 `reverb:start` 指令時加上 `--debug` 選項：

```shell
php artisan reverb:start --debug
```


<a name="restarting"></a>
### 重新啟動

由於 Reverb 是一個長期執行的程序，如果不透過 `reverb:restart` Artisan 指令重新啟動伺服器，您的程式碼變更將不會反映出來。

`reverb:restart` 指令可確保在停止伺服器之前優雅地終止所有連線。如果您使用 Supervisor 等程序管理器來執行 Reverb，則在所有連線終止後，程序管理器將自動重新啟動伺服器：

```shell
php artisan reverb:restart
```


<a name="monitoring"></a>
## 監控

可以透過與 [Laravel Pulse](/docs/{{version}}/pulse) 的整合來監控 Reverb。藉由啟用 Reverb 的 Pulse 整合，您可以追蹤伺服器正在處理的連線數和訊息數。

要啟用整合，您應該先確保已[安裝 Pulse](/docs/{{version}}/pulse#installation)。然後，將任何 Reverb 的記錄器 (Recorder) 新增到應用程式的 `config/pulse.php` 設定檔中：

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

連線活動是透過定期輪詢新更新來記錄的。為了確保這些資訊在 Pulse 儀表板上正確渲染，您必須在 Reverb 伺服器上執行 `pulse:check` 守護程序 (Daemon)。如果您在[水平擴充](#scaling)的配置中執行 Reverb，則只需在其中一台伺服器上執行此守護程序。

<a name="production"></a>
## 在正式環境執行 Reverb

由於 WebSocket 伺服器長時間執行的特性，你可能需要針對伺服器和代管環境進行一些優化，以確保你的 Reverb 伺服器能夠在伺服器現有的資源下有效地處理最佳的連線數量。

> [!NOTE]
> [Laravel Cloud](https://cloud.laravel.com) 提供了由 Laravel Reverb 叢集驅動的全代管 WebSocket 基礎設施，讓你無需管理基礎設施即可擴充並發佈支援 Reverb 的應用程式。


<a name="open-files"></a>
### 開啟檔案數

每個 WebSocket 連線都會保留在記憶體中，直到客戶端或伺服器中斷連線。在 Unix 和類 Unix 環境中，每個連線都由一個檔案代表。然而，在作業系統和應用程式層級，通常都會限制允許開啟的檔案數量。


<a name="operating-system"></a>
#### 作業系統

在基於 Unix 的作業系統上，你可以使用 `ulimit` 指令來確定允許開啟的檔案數量：

```shell
ulimit -n
```

此指令將顯示不同使用者允許的開啟檔案限制。你可以透過編輯 `/etc/security/limits.conf` 檔案來更新這些值。例如，將 `forge` 使用者的最大開啟檔案數更新為 10,000，如下所示：

```ini
# /etc/security/limits.conf
forge        soft  nofile  10000
forge        hard  nofile  10000
```


<a name="event-loop"></a>
### 事件迴圈

在底層，Reverb 使用 ReactPHP 事件迴圈來管理伺服器上的 WebSocket 連線。預設情況下，此事件迴圈由 `stream_select` 驅動，它不需要任何額外的擴充功能。然而，`stream_select` 通常被限制在 1,024 個開啟檔案。因此，如果你計劃處理超過 1,000 個並發連線，則需要使用不受相同限制約束的替代事件迴圈。

當 `ext-uv` 可用時，Reverb 會自動切換到由其驅動的迴圈。此 PHP 擴充功能可以透過 PECL 安裝：

```shell
pecl install uv
```


<a name="web-server"></a>
### 網頁伺服器

在大多數情況下，Reverb 執行在伺服器上非對外公開的連接埠。因此，為了將流量導向 Reverb，你應該設定反向代理。假設 Reverb 執行在主機 `0.0.0.0` 和連接埠 `8080`，且你的伺服器使用 Nginx 網頁伺服器，可以使用以下 Nginx 站台設定為你的 Reverb 伺服器定義反向代理：

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
> Reverb 在 `/app` 監聽 WebSocket 連線，並在 `/apps` 處理 API 請求。你應該確保處理 Reverb 請求的網頁伺服器可以提供這兩個 URI 的服務。如果你使用 [Laravel Forge](https://forge.laravel.com) 來管理你的伺服器，你的 Reverb 伺服器預設將會被正確設定。

通常，網頁伺服器會設定限制允許的連線數量，以防止伺服器過載。若要將 Nginx 網頁伺服器上允許的連線數增加到 10,000，應更新 `nginx.conf` 檔案的 `worker_rlimit_nofile` 和 `worker_connections` 值：

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

基於 Unix 的作業系統通常會限制伺服器上可以開啟的連接埠數量。你可以透過以下指令查看目前允許的範圍：

```shell
cat /proc/sys/net/ipv4/ip_local_port_range
# 32768	60999
```

上面的輸出顯示伺服器最多可以處理 28,231 (60,999 - 32,768) 個連線，因為每個連線都需要一個閒置的連接埠。雖然我們建議使用[水平擴充](#scaling)來增加允許的連線數量，但你也可以透過更新伺服器 `/etc/sysctl.conf` 設定檔中允許的連接埠範圍來增加可用的開啟連接埠數量。


<a name="process-management"></a>
### 程序管理

在大多數情況下，你應該使用 Supervisor 等程序管理器來確保 Reverb 伺服器持續執行。如果你使用 Supervisor 來執行 Reverb，你應該更新伺服器 `supervisor.conf` 檔案的 `minfds` 設定，以確保 Supervisor 能夠開啟處理 Reverb 伺服器連線所需的檔案：

```ini
[supervisord]
...
minfds=10000
```


<a name="scaling"></a>
### 擴充

如果你需要處理超過單一伺服器所能負擔的連線，你可以水平擴充你的 Reverb 伺服器。利用 Redis 的發佈 / 訂閱功能，Reverb 能夠管理跨多個伺服器的連線。當應用程式的其中一個 Reverb 伺服器收到訊息時，該伺服器將使用 Redis 將傳入的訊息發佈到所有其他伺服器。

若要啟用水平擴充，你應該在應用程式的 `.env` 設定檔中將 `REVERB_SCALING_ENABLED` 環境變數設定為 `true`：

```env
REVERB_SCALING_ENABLED=true
```

接下來，你應該有一個專用的、中央的 Redis 伺服器，所有的 Reverb 伺服器都將與其通訊。Reverb 將使用[為你的應用程式設定的預設 Redis 連線](/docs/{{version}}/redis#configuration)將訊息發佈到所有的 Reverb 伺服器。

一旦你啟用了 Reverb 的擴充選項並設定了 Redis 伺服器，你只需在多台能夠與你的 Redis 伺服器通訊的伺服器上呼叫 `reverb:start` 指令即可。這些 Reverb 伺服器應該放置在負載平衡器之後，該負載平衡器將傳入的請求平均分配給各個伺服器。


<a name="events"></a>
## 事件

Reverb 在連線生命週期和訊息處理期間會發送內部事件。你可以[監聽這些事件](/docs/{{version}}/events)，以便在管理連線或交換訊息時執行動作。

Reverb 會發送以下事件：


#### `Laravel\Reverb\Events\ChannelCreated`

當頻道被建立時發送。這通常發生在第一個連線訂閱特定頻道時。該事件接收 `Laravel\Reverb\Protocols\Pusher\Channel` 實例。


#### `Laravel\Reverb\Events\ChannelRemoved`

當頻道被移除時發送。這通常發生在最後一個連線取消訂閱頻道時。該事件接收 `Laravel\Reverb\Protocols\Pusher\Channel` 實例。


#### `Laravel\Reverb\Events\ConnectionPruned`

當逾期連線被伺服器清除時發送。該事件接收 `Laravel\Reverb\Contracts\Connection` 實例。


#### `Laravel\Reverb\Events\MessageReceived`

當從客戶端連線收到訊息時發送。該事件接收 `Laravel\Reverb\Contracts\Connection` 實例和原始字串 `$message`。


#### `Laravel\Reverb\Events\MessageSent`

當訊息傳送到客戶端連線時發送。該事件接收 `Laravel\Reverb\Contracts\Connection` 實例和原始字串 `$message`。