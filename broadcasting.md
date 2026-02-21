# 廣播 (Broadcasting)

- [簡介](#introduction)
- [快速入門](#quickstart)
- [伺服器端安裝](#server-side-installation)
    - [Reverb](#reverb)
    - [Pusher Channels](#pusher-channels)
    - [Ably](#ably)
- [用戶端安裝](#client-side-installation)
    - [Reverb](#client-reverb)
    - [Pusher Channels](#client-pusher-channels)
    - [Ably](#client-ably)
- [觀念總覽](#concept-overview)
    - [使用範例應用程式](#using-example-application)
- [定義廣播事件](#defining-broadcast-events)
    - [廣播名稱](#broadcast-name)
    - [廣播資料](#broadcast-data)
    - [廣播佇列](#broadcast-queue)
    - [廣播條件](#broadcast-conditions)
    - [廣播與資料庫交易](#broadcasting-and-database-transactions)
- [授權頻道](#authorizing-channels)
    - [定義授權回呼](#defining-authorization-callbacks)
    - [定義頻道類別](#defining-channel-classes)
- [廣播事件](#broadcasting-events)
    - [僅限其他人](#only-to-others)
    - [自定義連線](#customizing-the-connection)
    - [匿名事件](#anonymous-events)
    - [救援廣播](#rescuing-broadcasts)
- [接收廣播](#receiving-broadcasts)
    - [監聽事件](#listening-for-events)
    - [離開頻道](#leaving-a-channel)
    - [命名空間](#namespaces)
    - [使用 React 或 Vue](#using-react-or-vue)
- [出席頻道 (Presence Channels)](#presence-channels)
    - [授權出席頻道](#authorizing-presence-channels)
    - [加入出席頻道](#joining-presence-channels)
    - [廣播至出席頻道](#broadcasting-to-presence-channels)
- [模型廣播](#model-broadcasting)
    - [模型廣播慣例](#model-broadcasting-conventions)
    - [監聽模型廣播](#listening-for-model-broadcasts)
- [用戶端事件](#client-events)
- [通知](#notifications)

<a name="introduction"></a>
## 簡介

在許多現代 Web 應用程式中，WebSockets 被用來實作即時、動態更新的使用者介面。當伺服器上的某些資料更新時，通常會透過 WebSocket 連線發送一條訊息，並由用戶端處理。相較於持續向應用程式伺服器輪詢應反映在 UI 中的資料變更，WebSockets 提供了一種更有效率的替代方案。

例如，假設您的應用程式能夠將使用者的資料匯出為 CSV 檔案並透過電子郵件發送給他們。然而，建立此 CSV 檔案需要幾分鐘的時間，因此您選擇在 [佇列任務 (Queued Job)](/docs/{{version}}/queues) 中建立並寄送該 CSV。當 CSV 建立並郵寄給使用者後，我們可以使用事件廣播來分派一個 `App\Events\UserDataExported` 事件，並由我們應用程式的 JavaScript 接收。一旦接收到事件，我們就可以向使用者顯示一則訊息，告知他們的 CSV 已透過電子郵件發送，而無需重新整理頁面。

為了協助您構建這類功能，Laravel 讓您可以輕鬆地透過 WebSocket 連線「廣播」伺服器端的 Laravel [事件 (Events)](/docs/{{version}}/events)。廣播您的 Laravel 事件可讓您在伺服器端 Laravel 應用程式與用戶端 JavaScript 應用程式之間共用相同的事件名稱和資料。

廣播背後的核心概念很簡單：用戶端在前端連接到具名頻道 (Named Channels)，而您的 Laravel 應用程式在後端將事件廣播到這些頻道。這些事件可以包含您希望在前端使用的任何額外資料。


<a name="supported-drivers"></a>
#### 支援的驅動程式

預設情況下，Laravel 包含三個伺服器端廣播驅動程式供您選擇：[Laravel Reverb](https://reverb.laravel.com)、[Pusher Channels](https://pusher.com/channels) 以及 [Ably](https://ably.com)。

> [!NOTE]
> 在深入瞭解事件廣播之前，請確保您已閱讀 Laravel 關於 [事件與監聽器](/docs/{{version}}/events) 的文件。


<a name="quickstart"></a>
## 快速入門

預設情況下，新的 Laravel 應用程式並未啟用廣播。您可以使用 `install:broadcasting` Artisan 指令來啟用廣播：

```shell
php artisan install:broadcasting
```

`install:broadcasting` 指令會提示您選擇要使用的事件廣播服務。此外，它還會建立 `config/broadcasting.php` 設定檔和 `routes/channels.php` 檔案，您可以在此註冊應用程式的廣播授權路由和回呼 (Callbacks)。

Laravel 開箱即用支援多種廣播驅動程式：[Laravel Reverb](/docs/{{version}}/reverb)、[Pusher Channels](https://pusher.com/channels)、[Ably](https://ably.com)，以及用於本地開發和偵錯的 `log` 驅動程式。此外，還包含一個 `null` 驅動程式，可讓您在測試期間停用廣播。`config/broadcasting.php` 設定檔中包含了這些驅動程式的設定範例。

您應用程式所有的事件廣播設定都儲存在 `config/broadcasting.php` 設定檔中。如果您的應用程式中不存在此檔案，請不用擔心；它會在您執行 `install:broadcasting` Artisan 指令時建立。


<a name="quickstart-next-steps"></a>
#### 後續步驟

一旦啟用了事件廣播，您就可以開始深入瞭解如何 [定義廣播事件](#defining-broadcast-events) 以及 [監聽事件](#listening-for-events)。如果您使用的是 Laravel 的 React 或 Vue [入門套件](/docs/{{version}}/starter-kits)，您可以使用 Echo 的 [useEcho hook](#using-react-or-vue) 來監聽事件。

> [!NOTE]
> 在廣播任何事件之前，您應該先設定並執行 [佇列工作者](/docs/{{version}}/queues)。所有的事件廣播都是透過佇列任務完成的，因此應用程式的反應時間不會因為事件廣播而受到嚴重影響。

<a name="server-side-installation"></a>
## 伺服器端安裝

要開始使用 Laravel 的事件廣播，我們需要在 Laravel 應用程式中進行一些配置，並安裝一些套件。

事件廣播是由伺服器端的廣播驅動程式完成的，它會廣播您的 Laravel 事件，以便 Echo (一個 JavaScript 函式庫) 可以在瀏覽器用戶端接收它們。別擔心——我們將逐步引導您完成安裝過程的每個部分。


<a name="reverb"></a>
### Reverb

要在使用 Reverb 作為事件廣播器時快速啟用 Laravel 廣播功能的支援，請執行帶有 `--reverb` 選項的 `install:broadcasting` Artisan 指令。此 Artisan 指令將安裝 Reverb 所需的 Composer 與 NPM 套件，並使用適當的變數更新應用程式的 `.env` 檔案：

```shell
php artisan install:broadcasting --reverb
```


<a name="reverb-manual-installation"></a>
#### 手動安裝

執行 `install:broadcasting` 指令時，系統會提示您安裝 [Laravel Reverb](/docs/{{version}}/reverb)。當然，您也可以使用 Composer 套件管理器手動安裝 Reverb：

```shell
composer require laravel/reverb
```

安裝套件後，您可以執行 Reverb 的安裝指令來發布設定檔、新增 Reverb 所需的環境變數，並在您的應用程式中啟用事件廣播：

```shell
php artisan reverb:install
```

您可以在 [Reverb 說明文件](/docs/{{version}}/reverb) 中找到詳細的 Reverb 安裝與使用說明。


<a name="pusher-channels"></a>
### Pusher Channels

要在使用 Pusher 作為事件廣播器時快速啟用 Laravel 廣播功能的支援，請執行帶有 `--pusher` 選項的 `install:broadcasting` Artisan 指令。此 Artisan 指令將提示您輸入 Pusher 憑證，安裝 Pusher PHP 與 JavaScript SDK，並使用適當的變數更新應用程式的 `.env` 檔案：

```shell
php artisan install:broadcasting --pusher
```


<a name="pusher-manual-installation"></a>
#### 手動安裝

若要手動安裝 Pusher 支援，您應該使用 Composer 套件管理器安裝 Pusher Channels PHP SDK：

```shell
composer require pusher/pusher-php-server
```

接下來，您應該在 `config/broadcasting.php` 設定檔中配置您的 Pusher Channels 憑證。此檔案中已經包含了一個 Pusher Channels 配置範例，讓您可以快速指定您的 key、secret 與應用程式 ID。通常，您應該在應用程式的 `.env` 檔案中配置您的 Pusher Channels 憑證：

```ini
PUSHER_APP_ID="your-pusher-app-id"
PUSHER_APP_KEY="your-pusher-key"
PUSHER_APP_SECRET="your-pusher-secret"
PUSHER_HOST=
PUSHER_PORT=443
PUSHER_SCHEME="https"
PUSHER_APP_CLUSTER="mt1"
```

`config/broadcasting.php` 檔案的 `pusher` 配置也允許您指定 Channels 支援的其他 `options`，例如叢集 (cluster)。

接著，在應用程式的 `.env` 檔案中將 `BROADCAST_CONNECTION` 環境變數設置為 `pusher`：

```ini
BROADCAST_CONNECTION=pusher
```

最後，您已準備好安裝並配置 [Laravel Echo](#client-side-installation)，它將在用戶端接收廣播事件。


<a name="ably"></a>
### Ably

> [!NOTE]
> 下方的文件討論了如何在「Pusher 相容」模式下使用 Ably。然而，Ably 團隊推薦並維護了一個廣播器與 Echo 用戶端，能夠利用 Ably 提供的獨特功能。有關使用 Ably 維護的驅動程式的更多資訊，請[參閱 Ably 的 Laravel 廣播器說明文件](https://github.com/ably/laravel-broadcaster)。

要在使用 [Ably](https://ably.com) 作為事件廣播器時快速啟用 Laravel 廣播功能的支援，請執行帶有 `--ably` 選項的 `install:broadcasting` Artisan 指令。此 Artisan 指令將提示您輸入 Ably 憑證，安裝 Ably PHP 與 JavaScript SDK，並使用適當的變數更新應用程式的 `.env` 檔案：

```shell
php artisan install:broadcasting --ably
```

**在繼續之前，您應該在 Ably 應用程式設定中啟用 Pusher 協定支援。您可以在 Ably 應用程式設定儀表板的「Protocol Adapter Settings」部分啟用此功能。**


<a name="ably-manual-installation"></a>
#### 手動安裝

若要手動安裝 Ably 支援，您應該使用 Composer 套件管理器安裝 Ably PHP SDK：

```shell
composer require ably/ably-php
```

接下來，您應該在 `config/broadcasting.php` 設定檔中配置您的 Ably 憑證。此檔案中已經包含了一個 Ably 配置範例，讓您可以快速指定您的 key。通常，此值應透過 `ABLY_KEY` [環境變數](/docs/{{version}}/configuration#environment-configuration) 來設置：

```ini
ABLY_KEY=your-ably-key
```

接著，在應用程式的 `.env` 檔案中將 `BROADCAST_CONNECTION` 環境變數設置為 `ably`：

```ini
BROADCAST_CONNECTION=ably
```

最後，您已準備好安裝並配置 [Laravel Echo](#client-side-installation)，它將在用戶端接收廣播事件。

<a name="client-side-installation"></a>
## 用戶端安裝


<a name="client-reverb"></a>
### Reverb

[Laravel Echo](https://github.com/laravel/echo) 是一個 JavaScript 函式庫，它能讓訂閱頻道及監聽伺服器端廣播驅動程式所發送的事件變得非常輕鬆。

當透過 `install:broadcasting` Artisan 指令安裝 Laravel Reverb 時，Reverb 與 Echo 的基礎結構 (Scaffolding) 與設定將會自動注入到您的應用程式中。不過，如果您希望手動設定 Laravel Echo，可以參考下方的說明。


<a name="reverb-client-manual-installation"></a>
#### 手動安裝

若要手動為您的應用程式前端設定 Laravel Echo，首先請安裝 `pusher-js` 套件，因為 Reverb 使用 Pusher 協定來進行 WebSocket 訂閱、頻道與訊息傳輸：

```shell
npm install --save-dev laravel-echo pusher-js
```

安裝 Echo 後，您就可以在應用程式的 JavaScript 中建立一個全新的 Echo 實例。一個適合的操作位置是在 Laravel 框架隨附的 `resources/js/bootstrap.js` 檔案底部：

```js tab=JavaScript
import Echo from 'laravel-echo';

import Pusher from 'pusher-js';
window.Pusher = Pusher;

window.Echo = new Echo({
    broadcaster: 'reverb',
    key: import.meta.env.VITE_REVERB_APP_KEY,
    wsHost: import.meta.env.VITE_REVERB_HOST,
    wsPort: import.meta.env.VITE_REVERB_PORT ?? 80,
    wssPort: import.meta.env.VITE_REVERB_PORT ?? 443,
    forceTLS: (import.meta.env.VITE_REVERB_SCHEME ?? 'https') === 'https',
    enabledTransports: ['ws', 'wss'],
});
```

```js tab=React
import { configureEcho } from "@laravel/echo-react";

configureEcho({
    broadcaster: "reverb",
    // key: import.meta.env.VITE_REVERB_APP_KEY,
    // wsHost: import.meta.env.VITE_REVERB_HOST,
    // wsPort: import.meta.env.VITE_REVERB_PORT,
    // wssPort: import.meta.env.VITE_REVERB_PORT,
    // forceTLS: (import.meta.env.VITE_REVERB_SCHEME ?? 'https') === 'https',
    // enabledTransports: ['ws', 'wss'],
});
```

```js tab=Vue
import { configureEcho } from "@laravel/echo-vue";

configureEcho({
    broadcaster: "reverb",
    // key: import.meta.env.VITE_REVERB_APP_KEY,
    // wsHost: import.meta.env.VITE_REVERB_HOST,
    // wsPort: import.meta.env.VITE_REVERB_PORT,
    // wssPort: import.meta.env.VITE_REVERB_PORT,
    // forceTLS: (import.meta.env.VITE_REVERB_SCHEME ?? 'https') === 'https',
    // enabledTransports: ['ws', 'wss'],
});
```

接下來，您應該編譯應用程式的資產：

```shell
npm run build
```

> [!WARNING]
> Laravel Echo 的 `reverb` 廣播器需要 laravel-echo v1.16.0 以上版本。


<a name="client-pusher-channels"></a>
### Pusher Channels

[Laravel Echo](https://github.com/laravel/echo) 是一個 JavaScript 函式庫，它能讓訂閱頻道及監聽伺服器端廣播驅動程式所發送的事件變得非常輕鬆。

當透過 `install:broadcasting --pusher` Artisan 指令安裝廣播支援時，Pusher 與 Echo 的基礎結構與設定將會自動注入到您的應用程式中。不過，如果您希望手動設定 Laravel Echo，可以參考下方的說明。


<a name="pusher-client-manual-installation"></a>
#### 手動安裝

若要手動為您的應用程式前端設定 Laravel Echo，首先請安裝 `laravel-echo` 與 `pusher-js` 套件，這些套件利用 Pusher 協定進行 WebSocket 訂閱、頻道與訊息傳輸：

```shell
npm install --save-dev laravel-echo pusher-js
```

安裝 Echo 後，您就可以在應用程式的 `resources/js/bootstrap.js` 檔案中建立一個全新的 Echo 實例：

```js tab=JavaScript
import Echo from 'laravel-echo';

import Pusher from 'pusher-js';
window.Pusher = Pusher;

window.Echo = new Echo({
    broadcaster: 'pusher',
    key: import.meta.env.VITE_PUSHER_APP_KEY,
    cluster: import.meta.env.VITE_PUSHER_APP_CLUSTER,
    forceTLS: true
});
```

```js tab=React
import { configureEcho } from "@laravel/echo-react";

configureEcho({
    broadcaster: "pusher",
    // key: import.meta.env.VITE_PUSHER_APP_KEY,
    // cluster: import.meta.env.VITE_PUSHER_APP_CLUSTER,
    // forceTLS: true,
    // wsHost: import.meta.env.VITE_PUSHER_HOST,
    // wsPort: import.meta.env.VITE_PUSHER_PORT,
    // wssPort: import.meta.env.VITE_PUSHER_PORT,
    // enabledTransports: ["ws", "wss"],
});
```

```js tab=Vue
import { configureEcho } from "@laravel/echo-vue";

configureEcho({
    broadcaster: "pusher",
    // key: import.meta.env.VITE_PUSHER_APP_KEY,
    // cluster: import.meta.env.VITE_PUSHER_APP_CLUSTER,
    // forceTLS: true,
    // wsHost: import.meta.env.VITE_PUSHER_HOST,
    // wsPort: import.meta.env.VITE_PUSHER_PORT,
    // wssPort: import.meta.env.VITE_PUSHER_PORT,
    // enabledTransports: ["ws", "wss"],
});
```

接著，您應該在應用程式的 `.env` 檔案中為 Pusher 環境變數定義適當的值。如果這些變數尚未存在於您的 `.env` 檔案中，請新增它們：

```ini
PUSHER_APP_ID="your-pusher-app-id"
PUSHER_APP_KEY="your-pusher-key"
PUSHER_APP_SECRET="your-pusher-secret"
PUSHER_HOST=
PUSHER_PORT=443
PUSHER_SCHEME="https"
PUSHER_APP_CLUSTER="mt1"

VITE_APP_NAME="${APP_NAME}"
VITE_PUSHER_APP_KEY="${PUSHER_APP_KEY}"
VITE_PUSHER_HOST="${PUSHER_HOST}"
VITE_PUSHER_PORT="${PUSHER_PORT}"
VITE_PUSHER_SCHEME="${PUSHER_SCHEME}"
VITE_PUSHER_APP_CLUSTER="${PUSHER_APP_CLUSTER}"
```

根據您的應用程式需求調整 Echo 設定後，您就可以編譯應用程式資產：

```shell
npm run build
```

> [!NOTE]
> 若要深入了解如何編譯應用程式的 JavaScript 資產，請參閱 [Vite](/docs/{{version}}/vite) 的文件。


<a name="using-an-existing-client-instance"></a>
#### 使用現有的用戶端實例

如果您已經有一個預先設定好的 Pusher Channels 用戶端實例並希望 Echo 使用它，您可以透過 `client` 設定選項將其傳遞給 Echo：

```js
import Echo from 'laravel-echo';
import Pusher from 'pusher-js';

const options = {
    broadcaster: 'pusher',
    key: import.meta.env.VITE_PUSHER_APP_KEY
}

window.Echo = new Echo({
    ...options,
    client: new Pusher(options.key, options)
});
```

<a name="client-ably"></a>
### Ably

> [!NOTE]
> 下面的文件討論了如何以「Pusher 相容」模式使用 Ably。然而，Ably 團隊推薦並維護了一個廣播器與 Echo 用戶端，能夠利用 Ably 提供的獨特功能。有關使用 Ably 維護的驅動程式的更多資訊，請[參閱 Ably 的 Laravel 廣播器文件](https://github.com/ably/laravel-broadcaster)。

[Laravel Echo](https://github.com/laravel/echo) 是廣播一個 JavaScript 函式庫，它讓訂閱頻道與監聽由伺服器端廣播驅動程式發送的事件變得非常簡單。

透過 `install:broadcasting --ably` Artisan 指令安裝廣播支援時，Ably 與 Echo 的基礎結構與配置將會自動注入到您的應用程式中。但是，如果您希望手動配置 Laravel Echo，可以按照以下說明進行操作。

<a name="ably-client-manual-installation"></a>
#### 手動安裝

若要為應用程式的前端手動配置 Laravel Echo，首先請安裝 `laravel-echo` 與 `pusher-js` 套件，這兩個套件利用 Pusher 協定進行 WebSocket 訂閱、頻道與訊息傳遞：

```shell
npm install --save-dev laravel-echo pusher-js
```

**在繼續之前，您應該在 Ably 應用程式設定中啟用 Pusher 協定支援。您可以在 Ably 應用程式設定儀表板的「Protocol Adapter Settings」部分中啟用此功能。**

安裝 Echo 後，您就可以在應用程式的 `resources/js/bootstrap.js` 檔案中建立一個全新的 Echo 實例：

```js tab=JavaScript
import Echo from 'laravel-echo';

import Pusher from 'pusher-js';
window.Pusher = Pusher;

window.Echo = new Echo({
    broadcaster: 'pusher',
    key: import.meta.env.VITE_ABLY_PUBLIC_KEY,
    wsHost: 'realtime-pusher.ably.io',
    wsPort: 443,
    disableStats: true,
    encrypted: true,
});
```

```js tab=React
import { configureEcho } from "@laravel/echo-react";

configureEcho({
    broadcaster: "ably",
    // key: import.meta.env.VITE_ABLY_PUBLIC_KEY,
    // wsHost: "realtime-pusher.ably.io",
    // wsPort: 443,
    // disableStats: true,
    // encrypted: true,
});
```

```js tab=Vue
import { configureEcho } from "@laravel/echo-vue";

configureEcho({
    broadcaster: "ably",
    // key: import.meta.env.VITE_ABLY_PUBLIC_KEY,
    // wsHost: "realtime-pusher.ably.io",
    // wsPort: 443,
    // disableStats: true,
    // encrypted: true,
});
```

您可能已經注意到我們的 Ably Echo 配置引用了 `VITE_ABLY_PUBLIC_KEY` 環境變數。此變數的值應該是您的 Ably 公鑰 (Public Key)。您的公鑰是您的 Ably key 中出現在 `:` 字元之前的內容。

根據您的需求調整 Echo 配置後，您可以編譯應用程式的資產：

```shell
npm run dev
```

> [!NOTE]
> 若要了解有關編譯應用程式 JavaScript 資產的更多資訊，請參閱 [Vite](/docs/{{version}}/vite) 的文件。

<a name="concept-overview"></a>
## 觀念總覽

Laravel 的事件廣播允許您使用基於驅動程式的 WebSockets 方法，將伺服器端的 Laravel 事件廣播到用戶端的 JavaScript 應用程式。目前，Laravel 內建了 [Laravel Reverb](https://reverb.laravel.com)、[Pusher Channels](https://pusher.com/channels) 和 [Ably](https://ably.com) 驅動程式。可以使用 [Laravel Echo](#client-side-installation) JavaScript 套件在用戶端輕鬆地取用這些事件。

事件透過「頻道 (channels)」進行廣播，頻道可以指定為公開或私有。任何造訪您應用程式的訪客都可以在不需要任何身份驗證或授權的情況下訂閱公開頻道；然而，為了訂閱私有頻道，使用者必須經過身份驗證並獲准在該頻道上監聽。


<a name="using-example-application"></a>
### 使用範例應用程式

在深入探討事件廣播的各個元件之前，讓我們以電子商務商店為例進行高階總覽。

在我們的應用程式中，假設我們有一個頁面允許使用者查看其訂單的運送狀態。我們也假設當應用程式處理運送狀態更新時，會觸發一個 `OrderShipmentStatusUpdated` 事件：

```php
use App\Events\OrderShipmentStatusUpdated;

OrderShipmentStatusUpdated::dispatch($order);
```


<a name="the-shouldbroadcast-interface"></a>
#### The `ShouldBroadcast` 介面

當使用者正在查看其訂單之一時，我們不希望他們必須重新整理頁面才能查看狀態更新。相反地，我們希望在更新建立時將其廣播到應用程式。因此，我們需要使用 `ShouldBroadcast` 介面標記 `OrderShipmentStatusUpdated` 事件。這將指示 Laravel 在觸發該事件時對其進行廣播：

```php
<?php

namespace App\Events;

use App\Models\Order;
use Illuminate\Broadcasting\Channel;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Broadcasting\PresenceChannel;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Queue\SerializesModels;

class OrderShipmentStatusUpdated implements ShouldBroadcast
{
    /**
     * The order instance.
     *
     * @var \App\Models\Order
     */
    public $order;
}
```

`ShouldBroadcast` 介面要求我們的事件定義一個 `broadcastOn` 方法。此方法負責回傳事件應該廣播的頻道。此方法的空 stub 已在生成的事件類別中定義，因此我們只需要填寫其詳細資訊。我們只希望訂單的建立者能夠查看狀態更新，因此我們將在與訂單綁定的私有頻道上廣播事件：

```php
use Illuminate\Broadcasting\Channel;
use Illuminate\Broadcasting\PrivateChannel;

/**
 * Get the channel the event should broadcast on.
 */
public function broadcastOn(): Channel
{
    return new PrivateChannel('orders.'.$this->order->id);
}
```

如果您希望在多個頻道上廣播事件，則可以改為回傳一個 `array`：

```php
use Illuminate\Broadcasting\PrivateChannel;

/**
 * Get the channels the event should broadcast on.
 *
 * @return array<int, \Illuminate\Broadcasting\Channel>
 */
public function broadcastOn(): array
{
    return [
        new PrivateChannel('orders.'.$this->order->id),
        // ...
    ];
}
```


<a name="example-application-authorizing-channels"></a>
#### 授權頻道

請記住，使用者必須經過授權才能在私有頻道上進行監聽。我們可以在應用程式的 `routes/channels.php` 檔案中定義我們的頻道授權規則。在此範例中，我們需要驗證任何嘗試監聽私有 `orders.1` 頻道的使用者是否確實是該訂單的建立者：

```php
use App\Models\Order;
use App\Models\User;

Broadcast::channel('orders.{orderId}', function (User $user, int $orderId) {
    return $user->id === Order::findOrNew($orderId)->user_id;
});
```

`channel` 方法接受兩個參數：頻道名稱和一個回傳 `true` 或 `false` 的回呼，指示使用者是否被授權監聽該頻道。

所有授權回呼都接收目前通過驗證的使用者作為其第一個參數，並接收任何額外的萬用字元參數作為其後續參數。在此範例中，我們使用 `{orderId}` 佔位符來指出頻道名稱的 "ID" 部分是一個萬用字元。


<a name="listening-for-event-broadcasts"></a>
#### 監聽事件廣播

接下來，剩下的就是要在我們的 JavaScript 應用程式中監聽該事件。我們可以使用 [Laravel Echo](#client-side-installation) 來完成這項工作。Laravel Echo 內建的 React 和 Vue hooks 讓您可以簡單地入門，而且預設情況下，事件的所有公開屬性都將包含在廣播事件中：

```js tab=React
import { useEcho } from "@laravel/echo-react";

useEcho(
    `orders.${orderId}`,
    "OrderShipmentStatusUpdated",
    (e) => {
        console.log(e.order);
    },
);
```

```vue tab=Vue
<script setup lang="ts">
import { useEcho } from "@laravel/echo-vue";

useEcho(
    `orders.${orderId}`,
    "OrderShipmentStatusUpdated",
    (e) => {
        console.log(e.order);
    },
);
</script>
```

<a name="defining-broadcast-events"></a>
## 定義廣播事件

要告知 Laravel 給定的事件應該被廣播，您必須在事件類別實作 `Illuminate\Contracts\Broadcasting\ShouldBroadcast` 介面。此介面已匯入框架產生的所有事件類別中，因此您可以輕鬆地將其加入到您的任何事件中。

`ShouldBroadcast` 介面要求您實作一個方法：`broadcastOn`。`broadcastOn` 方法應回傳事件應廣播的頻道或頻道陣列。頻道應為 `Channel`、`PrivateChannel` 或 `PresenceChannel` 的實例。`Channel` 的實例代表任何使用者都可以訂閱的公開頻道，而 `PrivateChannels` 和 `PresenceChannels` 則代表需要[頻道授權](#authorizing-channels)的私有頻道：

```php
<?php

namespace App\Events;

use App\Models\User;
use Illuminate\Broadcasting\Channel;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Broadcasting\PresenceChannel;
use Illuminate\Broadcasting\PrivateChannel;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Queue\SerializesModels;

class ServerCreated implements ShouldBroadcast
{
    use SerializesModels;

    /**
     * Create a new event instance.
     */
    public function __construct(
        public User $user,
    ) {}

    /**
     * Get the channels the event should broadcast on.
     *
     * @return array<int, \Illuminate\Broadcasting\Channel>
     */
    public function broadcastOn(): array
    {
        return [
            new PrivateChannel('user.'.$this->user->id),
        ];
    }
}
```

在實作 `ShouldBroadcast` 介面後，您只需像往常一樣[觸發事件](/docs/{{version}}/events)。一旦事件被觸發，[佇列任務](/docs/{{version}}/queues)將自動使用您指定的廣播驅動程式廣播該事件。


<a name="broadcast-name"></a>
### 廣播名稱

預設情況下，Laravel 會使用事件的類別名稱來廣播事件。然而，您可以透過在事件中定義 `broadcastAs` 方法來過自定義廣播名稱：

```php
/**
 * The event's broadcast name.
 */
public function broadcastAs(): string
{
    return 'server.created';
}
```

如果您使用 `broadcastAs` 方法自定義廣播名稱，請確保在註冊監聽器時加上前導的 `.` 字元。這將指示 Echo 不要將應用程式的命名空間附加到事件名稱前：

```javascript
.listen('.server.created', function (e) {
    // ...
});
```


<a name="broadcast-data"></a>
### 廣播資料

當事件被廣播時，其所有 `public` 屬性都會自動被序列化並作為事件的負載 (Payload) 進行廣播，讓您可以從 JavaScript 應用程式存取任何公開資料。例如，如果您的事件具有包含 Eloquent 模型的單個公開 `$user` 屬性，則該事件的廣播負載將如下所示：

```json
{
    "user": {
        "id": 1,
        "name": "Patrick Stewart"
        ...
    }
}
```

但是，如果您希望對廣播負載進行更精細的控制，可以向事件新增 `broadcastWith` 方法。此方法應回傳您希望作為事件負載廣播的資料陣列：

```php
/**
 * Get the data to broadcast.
 *
 * @return array<string, mixed>
 */
public function broadcastWith(): array
{
    return ['id' => $this->user->id];
}
```


<a name="broadcast-queue"></a>
### 廣播佇列

預設情況下，每個廣播事件都會放置在 `queue.php` 設定檔中指定的預設佇列連線的預設佇列上。您可以透過在事件類別中定義 `connection` 和 `queue` 屬性來過自定義廣播器使用的佇列連線和名稱：

```php
/**
 * The name of the queue connection to use when broadcasting the event.
 *
 * @var string
 */
public $connection = 'redis';

/**
 * The name of the queue on which to place the broadcasting job.
 *
 * @var string
 */
public $queue = 'default';
```

或者，您可以透過在事件中定義 `broadcastQueue` 方法來過自定義佇列名稱：

```php
/**
 * The name of the queue on which to place the broadcasting job.
 */
public function broadcastQueue(): string
{
    return 'default';
}
```

如果您想使用 `sync` 佇列而不是預設佇列驅動程式來廣播事件，可以實作 `ShouldBroadcastNow` 介面來代替 `ShouldBroadcast`：

```php
<?php

namespace App\Events;

use Illuminate\Contracts\Broadcasting\ShouldBroadcastNow;

class OrderShipmentStatusUpdated implements ShouldBroadcastNow
{
    // ...
}
```


<a name="broadcast-conditions"></a>
### 廣播條件

有時您只想在給定條件為 true 時廣播事件。您可以透過在事件類別中新增 `broadcastWhen` 方法來定義這些條件：

```php
/**
 * Determine if this event should broadcast.
 */
public function broadcastWhen(): bool
{
    return $this->order->value > 100;
}
```


<a name="broadcasting-and-database-transactions"></a>
#### 廣播與資料庫交易

當廣播事件在資料庫交易中被派遣時，它們可能在資料庫交易提交之前就已被佇列處理。發生這種情況時，您在資料庫交易期間對模型或資料庫記錄所做的任何更新可能尚未反映在資料庫中。此外，在交易中建立的任何模型或資料庫記錄可能尚未存在於資料庫中。如果您的事件依賴這些模型，則在處理廣播事件的任務時可能會發生非預期的錯誤。

如果您的佇列連線 `after_commit` 設定選項設為 `false`，您仍然可以透過在事件類別實作 `ShouldDispatchAfterCommit` 介面，來指示特定的廣播事件應在所有開啟的資料庫交易提交後才派遣：

```php
<?php

namespace App\Events;

use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Contracts\Events\ShouldDispatchAfterCommit;
use Illuminate\Queue\SerializesModels;

class ServerCreated implements ShouldBroadcast, ShouldDispatchAfterCommit
{
    use SerializesModels;
}
```

> [!NOTE]
> 要了解更多關於如何解決這些問題的資訊，請參閱有關 [佇列任務與資料庫交易](/docs/{{version}}/queues#jobs-and-database-transactions) 的說明文件。

<a name="authorizing-channels"></a>
## 授權頻道

私有頻道要求您必須授權目前已通過身分驗證的使用者是否確實可以監聽該頻道。這可以透過向您的 Laravel 應用程式發送包含頻道名稱的 HTTP 請求來完成，並讓您的應用程式決定該使用者是否可以監聽該頻道。使用 [Laravel Echo](#client-side-installation) 時，會自動發送授權訂閱私有頻道的 HTTP 請求。

安裝廣播功能後，Laravel 會嘗試自動註冊 `/broadcasting/auth` 路由來處理授權請求。如果 Laravel 無法自動註冊這些路由，您可以在應用程式的 `/bootstrap/app.php` 檔案中手動註冊它們：

```php
->withRouting(
    web: __DIR__.'/../routes/web.php',
    channels: __DIR__.'/../routes/channels.php',
    health: '/up',
)
```


<a name="defining-authorization-callbacks"></a>
### 定義授權回呼

接下來，我們需要定義實際決定目前已通過身分驗證的使用者是否可以監聽給定頻道的邏輯。這是在執行 `install:broadcasting` Artisan 指令時所建立的 `routes/channels.php` 檔案中完成的。在該檔案中，您可以使用 `Broadcast::channel` 方法來註冊頻道授權回呼：

```php
use App\Models\User;

Broadcast::channel('orders.{orderId}', function (User $user, int $orderId) {
    return $user->id === Order::findOrNew($orderId)->user_id;
});
```

`channel` 方法接受兩個參數：頻道的名稱，以及一個回傳 `true` 或 `false` 以指示使用者是否有權監聽該頻道的回呼。

所有授權回呼都會接收目前已通過身分驗證的使用者作為其第一個參數，並將任何額外的萬用字元參數作為其後續參數。在此範例中，我們使用 `{orderId}` 佔位符來指示頻道名稱的「ID」部分是一個萬用字元。

您可以使用 `channel:list` Artisan 指令查看應用程式廣播授權回呼的列表：

```shell
php artisan channel:list
```


<a name="authorization-callback-model-binding"></a>
#### 授權回呼模型綁定

就像 HTTP 路由一樣，頻道路由也可以利用隱含式與顯式 [路由模型綁定](/docs/{{version}}/routing#route-model-binding)。例如，您可以請求一個實際的 `Order` 模型實例，而不是接收字串或數值的訂單 ID：

```php
use App\Models\Order;
use App\Models\User;

Broadcast::channel('orders.{order}', function (User $user, Order $order) {
    return $user->id === $order->user_id;
});
```

> [!WARNING]
> 與 HTTP 路由模型綁定不同，頻道模型綁定不支援自動 [隱含式模型綁定範圍限制](/docs/{{version}}/routing#implicit-model-binding-scoping)。然而，這很少成為問題，因為大多數頻道可以根據單一模型的唯一主鍵來限制範圍。


<a name="authorization-callback-authentication"></a>
#### 授權回呼身分驗證

私有與出席廣播頻道會透過應用程式預設的身分驗證 Guard 來驗證目前的使用者。如果使用者未通過身分驗證，則會自動拒絕頻道授權，且永遠不會執行授權回呼。但是，如果需要，您可以分配多個自定義 Guard 來驗證傳入的請求：

```php
Broadcast::channel('channel', function () {
    // ...
}, ['guards' => ['web', 'admin']]);
```


<a name="defining-channel-classes"></a>
### 定義頻道類別

如果您的應用程式使用了許多不同的頻道，您的 `routes/channels.php` 檔案可能會變得很臃腫。因此，您可以使用頻道類別來代替閉包 (Closure) 來授權頻道。要產生頻道類別，請使用 `make:channel` Artisan 指令。此指令會將新的頻道類別放置在 `App/Broadcasting` 目錄中。

```shell
php artisan make:channel OrderChannel
```

接下來，在您的 `routes/channels.php` 檔案中註冊您的頻道：

```php
use App\Broadcasting\OrderChannel;

Broadcast::channel('orders.{order}', OrderChannel::class);
```

最後，您可以將頻道的授權邏輯放在頻道類別的 `join` 方法中。這個 `join` 方法將存放您通常會放在頻道授權閉包中的相同邏輯。您也可以利用頻道模型綁定：

```php
<?php

namespace App\Broadcasting;

use App\Models\Order;
use App\Models\User;

class OrderChannel
{
    /**
     * Create a new channel instance.
     */
    public function __construct() {}

    /**
     * Authenticate the user's access to the channel.
     */
    public function join(User $user, Order $order): array|bool
    {
        return $user->id === $order->user_id;
    }
}
```

> [!NOTE]
> 與 Laravel 中的許多其他類別一樣，頻道類別將由 [服務容器](/docs/{{version}}/container) 自動解析。因此，您可以在頻道的建構函式中對所需的任何依賴項進行型別提示 (Type-hint)。

<a name="broadcasting-events"></a>
## 廣播事件

一旦你定義了一個事件並標記為實作 `ShouldBroadcast` 介面，你只需要使用該事件的 dispatch 方法來觸發事件即可。事件發送器會注意到該事件已標記為實作 `ShouldBroadcast` 介面，並將該事件排入廣播佇列：

```php
use App\Events\OrderShipmentStatusUpdated;

OrderShipmentStatusUpdated::dispatch($order);
```


<a name="only-to-others"></a>
### 僅限其他人

在建構利用事件廣播的應用程式時，你偶爾可能需要將事件廣播給特定頻道的所有訂閱者，但目前使用者除外。你可以使用 `broadcast` 輔助函式和 `toOthers` 方法來達成此目的：

```php
use App\Events\OrderShipmentStatusUpdated;

broadcast(new OrderShipmentStatusUpdated($update))->toOthers();
```

為了更了解何時可能需要使用 `toOthers` 方法，讓我們想像一個任務列表應用程式，使用者可以透過輸入任務名稱來建立新任務。為了建立任務，你的應用程式可能會向 `/task` URL 發送請求，該 URL 會廣播任務的建立並回傳新任務的 JSON 表示形式。當你的 JavaScript 應用程式從端點接收到回應時，它可能會直接將新任務插入其任務列表中，如下所示：

```js
axios.post('/task', task)
    .then((response) => {
        this.tasks.push(response.data);
    });
```

但是，請記住我們也廣播了任務的建立。如果你的 JavaScript 應用程式也在監聽此事件以將任務新增到任務列表，你的列表中將會出現重複的任務：一個來自端點，一個來自廣播。你可以透過使用 `toOthers` 方法指示廣播器不要將事件廣播給目前使用者來解決此問題。

> [!WARNING]
> 你的事件必須使用 `Illuminate\Broadcasting\InteractsWithSockets` trait 才能呼叫 `toOthers` 方法。


<a name="only-to-others-configuration"></a>
#### 設定

當你初始化 Laravel Echo 實例時，會為該連線分配一個 socket ID。如果你正在使用全域 [Axios](https://github.com/axios/axios) 實例從 JavaScript 應用程式發送 HTTP 請求，該 socket ID 將會自動附加到每個發出的請求中，作為 `X-Socket-ID` 標頭。接著，當你呼叫 `toOthers` 方法時，Laravel 會從該標頭中提取 socket ID，並指示廣播器不要廣播到具有該 socket ID 的任何連線。

如果您沒有使用全域 Axios 實例，則需要手動設定 JavaScript 應用程式，以便在所有發出的請求中發送 `X-Socket-ID` 標頭。你可以使用 `Echo.socketId` 方法取得 socket ID：

```js
var socketId = Echo.socketId();
```


<a name="customizing-the-connection"></a>
### 自定義連線

如果你的應用程式與多個廣播連線互動，並且你想使用預設以外的廣播器來廣播事件，你可以使用 `via` 方法指定要將事件推送至哪個連線：

```php
use App\Events\OrderShipmentStatusUpdated;

broadcast(new OrderShipmentStatusUpdated($update))->via('pusher');
```

或者，你也可以透過在事件的建構子中呼叫 `broadcastVia` 方法來指定事件的廣播連線。但在這樣做之前，你應確保該事件類別使用了 `InteractsWithBroadcasting` trait：

```php
<?php

namespace App\Events;

use Illuminate\Broadcasting\Channel;
use Illuminate\Broadcasting\InteractsWithBroadcasting;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Broadcasting\PresenceChannel;
use Illuminate\Broadcasting\PrivateChannel;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Queue\SerializesModels;

class OrderShipmentStatusUpdated implements ShouldBroadcast
{
    use InteractsWithBroadcasting;

    /**
     * Create a new event instance.
     */
    public function __construct()
    {
        $this->broadcastVia('pusher');
    }
}
```


<a name="anonymous-events"></a>
### 匿名事件

有時候，你可能想向應用程式的前端廣播一個簡單的事件，而不建立專用的事件類別。為了因應這種情況，`Broadcast` facade 允許你廣播「匿名事件」：

```php
Broadcast::on('orders.'.$order->id)->send();
```

上述範例將廣播以下事件：

```json
{
    "event": "AnonymousEvent",
    "data": "[]",
    "channel": "orders.1"
}
```

使用 `as` 和 `with` 方法，你可以自定義事件的名稱和資料：

```php
Broadcast::on('orders.'.$order->id)
    ->as('OrderPlaced')
    ->with($order)
    ->send();
```

上述範例將廣播如下所示的事件：

```json
{
    "event": "OrderPlaced",
    "data": "{ id: 1, total: 100 }",
    "channel": "orders.1"
}
```

如果你想在私有或出席頻道上廣播匿名事件，你可以利用 `private` 和 `presence` 方法：

```php
Broadcast::private('orders.'.$order->id)->send();
Broadcast::presence('channels.'.$channel->id)->send();
```

使用 `send` 方法廣播匿名事件會將事件發送到應用程式的 [佇列](/docs/{{version}}/queues) 進行處理。但是，如果你想立即廣播事件，可以使用 `sendNow` 方法：

```php
Broadcast::on('orders.'.$order->id)->sendNow();
```

要將事件廣播給除了當前通過身份驗證的使用者以外的所有頻道訂閱者，你可以呼叫 `toOthers` 方法：

```php
Broadcast::on('orders.'.$order->id)
    ->toOthers()
    ->send();
```


<a name="rescuing-broadcasts"></a>
### 救援廣播

當你的應用程式的佇列伺服器無法使用，或者 Laravel 在廣播事件時遇到錯誤，通常會拋出一個異常，導致最終使用者看到應用程式錯誤。由於事件廣播通常是應用程式核心功能的補充，你可以透過在事件中實作 `ShouldRescue` 介面來防止這些異常干擾使用者體驗。

實作了 `ShouldRescue` 介面的事件在嘗試廣播時會自動利用 Laravel 的 [rescue 輔助函式](/docs/{{version}}/helpers#method-rescue) 進行廣播嘗試。此輔助函式會捕獲任何異常，將其回報給應用程式的異常處理器進行記錄，並允許應用程式繼續正常執行，而不會中斷使用者的工作流程：

```php
<?php

namespace App\Events;

use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Contracts\Broadcasting\ShouldRescue;

class ServerCreated implements ShouldBroadcast, ShouldRescue
{
    // ...
}
```

<a name="receiving-broadcasts"></a>
## 接收廣播


<a name="listening-for-events"></a>
### 監聽事件

一旦你[安裝並實例化了 Laravel Echo](#client-side-installation)，你就準備好開始監聽從 Laravel 應用程式廣播出的事件了。首先，使用 `channel` 方法來取得頻道的實例，接著呼叫 `listen` 方法來監聽指定的事件：

```js
Echo.channel(`orders.${this.order.id}`)
    .listen('OrderShipmentStatusUpdated', (e) => {
        console.log(e.order.name);
    });
```

如果你想監聽私有頻道上的事件，請改用 `private` 方法。你可以繼續串接呼叫 `listen` 方法，以在單一頻道上監聽多個事件：

```js
Echo.private(`orders.${this.order.id}`)
    .listen(/* ... */)
    .listen(/* ... */)
    .listen(/* ... */);
```


<a name="stop-listening-for-events"></a>
#### 停止監聽事件

如果你想在不[離開頻道](#leaving-a-channel)的情況下停止監聽特定事件，可以使用 `stopListening` 方法：

```js
Echo.private(`orders.${this.order.id}`)
    .stopListening('OrderShipmentStatusUpdated');
```


<a name="leaving-a-channel"></a>
### 離開頻道

要離開頻道，你可以呼叫 Echo 實例上的 `leaveChannel` 方法：

```js
Echo.leaveChannel(`orders.${this.order.id}`);
```

如果你想離開頻道以及其相關的私有與出席頻道，可以呼叫 `leave` 方法：

```js
Echo.leave(`orders.${this.order.id}`);
```

<a name="namespaces"></a>
### 命名空間

你可能已經注意到，在上述範例中我們沒有為事件類別指定完整的 `App\Events` 命名空間。這是因為 Echo 會自動假設事件位於 `App\Events` 命名空間內。然而，你可以在實例化 Echo 時傳遞 `namespace` 設定選項來設定根命名空間：

```js
window.Echo = new Echo({
    broadcaster: 'pusher',
    // ...
    namespace: 'App.Other.Namespace'
});
```

或者，在使用 Echo 訂閱事件時，你可以在事件類別前加上 `.`。這將允許你始終指定完整的類別名稱：

```js
Echo.channel('orders')
    .listen('.Namespace\\Event\\Class', (e) => {
        // ...
    });
```


<a name="using-react-or-vue"></a>
### 使用 React 或 Vue

Laravel Echo 包含了 React 與 Vue 的 hook，讓監聽事件變得輕而易舉。要開始使用，請調用 `useEcho` hook，它用於監聽私有事件。當元件被卸載 (Unmounted) 時，`useEcho` hook 會自動離開頻道：

```js tab=React
import { useEcho } from "@laravel/echo-react";

useEcho(
    `orders.${orderId}`,
    "OrderShipmentStatusUpdated",
    (e) => {
        console.log(e.order);
    },
);
```

```vue tab=Vue
<script setup lang="ts">
import { useEcho } from "@laravel/echo-vue";

useEcho(
    `orders.${orderId}`,
    "OrderShipmentStatusUpdated",
    (e) => {
        console.log(e.order);
    },
);
</script>
```

你可以透過向 `useEcho` 提供一個事件陣列來監聽多個事件：

```js
useEcho(
    `orders.${orderId}`,
    ["OrderShipmentStatusUpdated", "OrderShipped"],
    (e) => {
        console.log(e.order);
    },
);
```

你也可以指定廣播事件負載 (Payload) 資料的形狀，這能提供更好的型別安全性與編輯便利性：

```ts
type OrderData = {
    order: {
        id: number;
        user: {
            id: number;
            name: string;
        };
        created_at: string;
    };
};

useEcho<OrderData>(`orders.${orderId}`, "OrderShipmentStatusUpdated", (e) => {
    console.log(e.order.id);
    console.log(e.order.user.id);
});
```

當元件被卸載時，`useEcho` hook 會自動離開頻道；不過，如有需要，你可以利用回傳的函式來手動以程式化方式停止或開始監聽頻道：

```js tab=React
import { useEcho } from "@laravel/echo-react";

const { leaveChannel, leave, stopListening, listen } = useEcho(
    `orders.${orderId}`,
    "OrderShipmentStatusUpdated",
    (e) => {
        console.log(e.order);
    },
);

// Stop listening without leaving channel...
stopListening();

// Start listening again...
listen();

// Leave channel...
leaveChannel();

// Leave a channel and also its associated private and presence channels...
leave();
```

```vue tab=Vue
<script setup lang="ts">
import { useEcho } from "@laravel/echo-vue";

const { leaveChannel, leave, stopListening, listen } = useEcho(
    `orders.${orderId}`,
    "OrderShipmentStatusUpdated",
    (e) => {
        console.log(e.order);
    },
);

// Stop listening without leaving channel...
stopListening();

// Start listening again...
listen();

// Leave channel...
leaveChannel();

// Leave a channel and also its associated private and presence channels...
leave();
</script>
```


<a name="react-vue-connecting-to-public-channels"></a>
#### 連線至公開頻道

要連線到公開頻道，你可以使用 `useEchoPublic` hook：

```js tab=React
import { useEchoPublic } from "@laravel/echo-react";

useEchoPublic("posts", "PostPublished", (e) => {
    console.log(e.post);
});
```

```vue tab=Vue
<script setup lang="ts">
import { useEchoPublic } from "@laravel/echo-vue";

useEchoPublic("posts", "PostPublished", (e) => {
    console.log(e.post);
});
</script>
```


<a name="react-vue-connecting-to-presence-channels"></a>
#### 連線至出席頻道

要連線到出席頻道，你可以使用 `useEchoPresence` hook：

```js tab=React
import { useEchoPresence } from "@laravel/echo-react";

useEchoPresence("posts", "PostPublished", (e) => {
    console.log(e.post);
});
```

```vue tab=Vue
<script setup lang="ts">
import { useEchoPresence } from "@laravel/echo-vue";

useEchoPresence("posts", "PostPublished", (e) => {
    console.log(e.post);
});
</script>
```


<a name="react-vue-connection-status"></a>
#### 連線狀態

你可以使用 `useConnectionStatus` hook 取得目前的 WebSocket 連線狀態，它提供的響應式 (Reactive) 狀態會在連線狀態改變時自動更新：

```js tab=React
import { useConnectionStatus } from "@laravel/echo-react";

function ConnectionIndicator() {
    const status = useConnectionStatus();

    return <div>Connection: {status}</div>;
}
```

```vue tab=Vue
<script setup lang="ts">
import { useConnectionStatus } from "@laravel/echo-vue";

const status = useConnectionStatus();
</script>

<template>
    <div>Connection: {{ status }}</div>
</template>
```

可能的狀態值如下：

<div class="content-list" markdown="1">

- `connected` - 已成功連線至 WebSocket 伺服器。
- `connecting` - 正在進行初始連線嘗試。
- `reconnecting` - 斷線後正在嘗試重新連線。
- `disconnected` - 未連線且未嘗試重新連線。
- `failed` - 連線失敗且不會重試。

</div>

<a name="presence-channels"></a>
## 出席頻道 (Presence Channels)

出席頻道建立在私有頻道的安全性之上，同時揭露了額外的功能：得知誰訂閱了該頻道。這使得建立強大且具協作性的應用程式功能變得容易，例如當另一個使用者正在查看同一頁面時通知使用者，或列出聊天室的參與者。


<a name="authorizing-presence-channels"></a>
### 授權出席頻道

所有出席頻道同時也是私有頻道；因此，使用者必須被[授權才能存取它們](#authorizing-channels)。然而，在為出席頻道定義授權回呼時，如果使用者被授權加入頻道，您不會回傳 `true`。相反地，您應該回傳一個包含使用者資料的陣列。

授權回呼回傳的資料將提供給您的 JavaScript 應用程式中的出席頻道事件監聽器。如果使用者未被授權加入出席頻道，則應回傳 `false` 或 `null`：

```php
use App\Models\User;

Broadcast::channel('chat.{roomId}', function (User $user, int $roomId) {
    if ($user->canJoinRoom($roomId)) {
        return ['id' => $user->id, 'name' => $user->name];
    }
});
```


<a name="joining-presence-channels"></a>
### 加入出席頻道

要加入出席頻道，您可以使用 Echo 的 `join` 方法。`join` 方法將回傳一個 `PresenceChannel` 實作，除了提供 `listen` 方法外，還允許您訂閱 `here`、`joining` 與 `leaving` 事件。

```js
Echo.join(`chat.${roomId}`)
    .here((users) => {
        // ...
    })
    .joining((user) => {
        console.log(user.name);
    })
    .leaving((user) => {
        console.log(user.name);
    })
    .error((error) => {
        console.error(error);
    });
```

一旦成功加入頻道，`here` 回呼將立即執行，並接收一個包含目前訂閱該頻道的所有其他使用者資訊的陣列。當新使用者加入頻道時，將執行 `joining` 方法；而當使用者離開頻道時，則執行 `leaving` 方法。當驗證端點回傳 HTTP 狀態碼非 200 或解析回傳的 JSON 出現問題時，將執行 `error` 方法。


<a name="broadcasting-to-presence-channels"></a>
### 廣播至出席頻道

出席頻道可以像公開或私有頻道一樣接收事件。以聊天室為例，我們可能想要廣播 `NewMessage` 事件到該房間的出席頻道。為此，我們將從事件的 `broadcastOn` 方法中回傳一個 `PresenceChannel` 實例：

```php
/**
 * Get the channels the event should broadcast on.
 *
 * @return array<int, \Illuminate\Broadcasting\Channel>
 */
public function broadcastOn(): array
{
    return [
        new PresenceChannel('chat.'.$this->message->room_id),
    ];
}
```

與其他事件一樣，您可以使用 `broadcast` 輔助函式與 `toOthers` 方法來排除目前的使用者接收該廣播：

```php
broadcast(new NewMessage($message));

broadcast(new NewMessage($message))->toOthers();
```

如同其他類型的事件，您可以使用 Echo 的 `listen` 方法來監聽發送到出席頻道的事件：

```js
Echo.join(`chat.${roomId}`)
    .here(/* ... */)
    .joining(/* ... */)
    .leaving(/* ... */)
    .listen('NewMessage', (e) => {
        // ...
    });
```

<a name="model-broadcasting"></a>
## 模型廣播

> [!WARNING]
> 在閱讀下列關於模型廣播的說明文件之前，我們建議您先熟悉 Laravel 模型廣播服務的一般概念，以及如何手動建立與監聽廣播事件。

當應用程式的 [Eloquent 模型](/docs/{{version}}/eloquent) 被建立、更新或刪除時，通常會廣播事件。當然，這可以透過手動 [為 Eloquent 模型狀態變更定義自定義事件](/docs/{{version}}/eloquent#events) 並將這些事件加上 `ShouldBroadcast` 介面來輕鬆實現。

然而，如果您在應用程式中不打算將這些事件用於其他目的，僅為了廣播而建立事件類別可能會很繁瑣。為了解決這個問題，Laravel 允許您指定 Eloquent 模型應自動廣播其狀態變更。

若要開始使用，您的 Eloquent 模型應使用 `Illuminate\Database\Eloquent\BroadcastsEvents` trait。此外，模型應定義一個 `broadcastOn` 方法，該方法將回傳模型事件應廣播到的頻道陣列：

```php
<?php

namespace App\Models;

use Illuminate\Broadcasting\Channel;
use Illuminate\Broadcasting\PrivateChannel;
use Illuminate\Database\Eloquent\BroadcastsEvents;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class Post extends Model
{
    use BroadcastsEvents, HasFactory;

    /**
     * Get the user that the post belongs to.
     */
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }

    /**
     * Get the channels that model events should broadcast on.
     *
     * @return array<int, \Illuminate\Broadcasting\Channel|\Illuminate\Database\Eloquent\Model>
     */
    public function broadcastOn(string $event): array
    {
        return [$this, $this->user];
    }
}
```

一旦您的模型包含此 trait 並定義了其廣播頻道，當模型實例被建立、更新、刪除、放進垃圾桶 (Trashed) 或還原時，它將開始自動廣播事件。

此外，您可能已經注意到 `broadcastOn` 方法接收一個字串類型的 `$event` 參數。此參數包含模型上發生的事件類型，其值為 `created`、`updated`、`deleted`、`trashed` 或 `restored`。透過檢查此變數的值，您可以決定模型在特定事件下應廣播到哪些頻道（如果有的話）：

```php
/**
 * Get the channels that model events should broadcast on.
 *
 * @return array<string, array<int, \Illuminate\Broadcasting\Channel|\Illuminate\Database\Eloquent\Model>>
 */
public function broadcastOn(string $event): array
{
    return match ($event) {
        'deleted' => [],
        default => [$this, $this->user],
    };
}
```


<a name="customizing-model-broadcasting-event-creation"></a>
#### 自定義模型廣播事件的建立

有時，您可能希望自定義 Laravel 如何建立底層的模型廣播事件。您可以透過在 Eloquent 模型上定義 `newBroadcastableEvent` 方法來達成此目的。此方法應回傳一個 `Illuminate\Database\Eloquent\BroadcastableModelEventOccurred` 實例：

```php
use Illuminate\Database\Eloquent\BroadcastableModelEventOccurred;

/**
 * Create a new broadcastable model event for the model.
 */
protected function newBroadcastableEvent(string $event): BroadcastableModelEventOccurred
{
    return (new BroadcastableModelEventOccurred(
        $this, $event
    ))->dontBroadcastToCurrentUser();
}
```


<a name="model-broadcasting-conventions"></a>
### 模型廣播慣例


<a name="model-broadcasting-channel-conventions"></a>
#### 頻道慣例

正如您可能已經注意到的，上述模型範例中的 `broadcastOn` 方法並未回傳 `Channel` 實例。相反地，它直接回傳了 Eloquent 模型。如果您的模型 `broadcastOn` 方法回傳了 Eloquent 模型實例（或包含在該方法回傳的陣列中），Laravel 將自動使用模型的類別名稱和主鍵識別碼作為頻道名稱，為該模型實例化一個私有頻道。

因此，一個 `id` 為 `1` 的 `App\Models\User` 模型將被轉換為名為 `App.Models.User.1` 的 `Illuminate\Broadcasting\PrivateChannel` 實例。當然，除了從模型的 `broadcastOn` 方法回傳 Eloquent 模型實例之外，您也可以回傳完整的 `Channel` 實例，以便完全控制模型的頻道名稱：

```php
use Illuminate\Broadcasting\PrivateChannel;

/**
 * Get the channels that model events should broadcast on.
 *
 * @return array<int, \Illuminate\Broadcasting\Channel>
 */
public function broadcastOn(string $event): array
{
    return [
        new PrivateChannel('user.'.$this->id)
    ];
}
```

如果您打算從模型的 `broadcastOn` 方法中明確回傳一個頻道實例，您可以將一個 Eloquent 模型實例傳遞給頻道的建構函式。這樣做時，Laravel 將使用上述討論的模型頻道慣例將 Eloquent 模型轉換為頻道名稱字串：

```php
return [new Channel($this->user)];
```

如果您需要確定模型的頻道名稱，可以在任何模型實例上呼叫 `broadcastChannel` 方法。例如，對於一個 `id` 為 `1` 的 `App\Models\User` 模型，此方法會回傳字串 `App.Models.User.1`：

```php
$user->broadcastChannel();
```


<a name="model-broadcasting-event-conventions"></a>
#### 事件慣例

由於模型廣播事件與應用程式 `App\Events` 目錄中的「實際」事件沒有關聯，因此會根據慣例為它們分配名稱和負載 (Payload)。Laravel 的慣例是使用模型的類別名稱（不包含命名空間）以及觸發廣播的模型事件名稱來廣播事件。

例如，對 `App\Models\Post` 模型的更新將以 `PostUpdated` 的名稱向您的用戶端應用程式廣播事件，並帶有以下負載：

```json
{
    "model": {
        "id": 1,
        "title": "My first post"
        ...
    },
    ...
    "socket": "someSocketId"
}
```

刪除 `App\Models\User` 模型將廣播一個名為 `UserDeleted` 的事件。

如果您願意，可以透過在模型中新增 `broadcastAs` 和 `broadcastWith` 方法來定義自定義廣播名稱和負載。這些方法會接收正在發生的模型事件/操作名稱，讓您可以為每個模型操作自定義事件名稱和負載。如果 `broadcastAs` 方法回傳 `null`，Laravel 在廣播事件時將使用上述討論的模型廣播事件名稱慣例：

```php
/**
 * The model event's broadcast name.
 */
public function broadcastAs(string $event): string|null
{
    return match ($event) {
        'created' => 'post.created',
        default => null,
    };
}

/**
 * Get the data to broadcast for the model.
 *
 * @return array<string, mixed>
 */
public function broadcastWith(string $event): array
{
    return match ($event) {
        'created' => ['title' => $this->title],
        default => ['model' => $this],
    };
}
```

<a name="listening-for-model-broadcasts"></a>
### 監聽模型廣播

當您將 `BroadcastsEvents` trait 加入模型並定義了模型的 `broadcastOn` 方法後，您就可以開始在用戶端應用程式中監聽廣播的模型事件了。在開始之前，您可能需要參考有關[監聽事件](#listening-for-events)的完整文件。

首先，使用 `private` 方法取得頻道的實例，接著呼叫 `listen` 方法來監聽特定事件。通常，提供給 `private` 方法的頻道名稱應對應於 Laravel 的[模型廣播慣例](#model-broadcasting-conventions)。

取得頻道實例後，您可以使用 `listen` 方法來監聽特定事件。由於模型廣播事件與應用程式 `App\Events` 目錄中的「實際」事件無關，因此[事件名稱](#model-broadcasting-event-conventions)必須以 `.` 為前綴，以表示它不屬於特定的命名空間。每個模型廣播事件都有一個 `model` 屬性，其中包含模型的所有可廣播屬性：

```js
Echo.private(`App.Models.User.${this.user.id}`)
    .listen('.UserUpdated', (e) => {
        console.log(e.model);
    });
```


<a name="model-broadcasts-with-react-or-vue"></a>
#### 使用 React 或 Vue

如果您使用的是 React 或 Vue，可以使用 Laravel Echo 內建的 `useEchoModel` hook 來輕鬆監聽模型廣播：

```js tab=React
import { useEchoModel } from "@laravel/echo-react";

useEchoModel("App.Models.User", userId, ["UserUpdated"], (e) => {
    console.log(e.model);
});
```

```vue tab=Vue
<script setup lang="ts">
import { useEchoModel } from "@laravel/echo-vue";

useEchoModel("App.Models.User", userId, ["UserUpdated"], (e) => {
    console.log(e.model);
});
</script>
```

您還可以指定模型事件負載資料的型別形狀，以提供更好的型別安全性與開發便利性：

```ts
type User = {
    id: number;
    name: string;
    email: string;
};

useEchoModel<User, "App.Models.User">("App.Models.User", userId, ["UserUpdated"], (e) => {
    console.log(e.model.id);
    console.log(e.model.name);
});
```

<a name="client-events"></a>
## 用戶端事件

> [!NOTE]
> 當使用 [Pusher Channels](https://pusher.com/channels) 時，您必須在 [應用程式儀表板](https://dashboard.pusher.com/) 的「App Settings」區塊中啟用「Client Events」選項，才能發送用戶端事件。

有時，您可能希望將事件廣播給其他已連線的用戶端，而不經過您的 Laravel 應用程式。這對於「正在輸入」通知之類的功能特別有用，您可以藉此提醒應用程式的使用者，另一位使用者正在特定畫面上輸入訊息。

要廣播用戶端事件，您可以使用 Echo 的 `whisper` 方法：

```js tab=JavaScript
Echo.private(`chat.${roomId}`)
    .whisper('typing', {
        name: this.user.name
    });
```

```js tab=React
import { useEcho } from "@laravel/echo-react";

const { channel } = useEcho(`chat.${roomId}`, ['update'], (e) => {
    console.log('Chat event received:', e);
});

channel().whisper('typing', { name: user.name });
```

```vue tab=Vue
<script setup lang="ts">
import { useEcho } from "@laravel/echo-vue";

const { channel } = useEcho(`chat.${roomId}`, ['update'], (e) => {
    console.log('Chat event received:', e);
});

channel().whisper('typing', { name: user.name });
</script>
```

要監聽用戶端事件，您可以使用 `listenForWhisper` 方法：

```js tab=JavaScript
Echo.private(`chat.${roomId}`)
    .listenForWhisper('typing', (e) => {
        console.log(e.name);
    });
```

```js tab=React
import { useEcho } from "@laravel/echo-react";

const { channel } = useEcho(`chat.${roomId}`, ['update'], (e) => {
    console.log('Chat event received:', e);
});

channel().listenForWhisper('typing', (e) => {
    console.log(e.name);
});
```

```vue tab=Vue
<script setup lang="ts">
import { useEcho } from "@laravel/echo-vue";

const { channel } = useEcho(`chat.${roomId}`, ['update'], (e) => {
    console.log('Chat event received:', e);
});

channel().listenForWhisper('typing', (e) => {
    console.log(e.name);
});
</script>
```


<a name="notifications"></a>
## 通知

透過將事件廣播與 [通知](/docs/{{version}}/notifications) 結合，您的 JavaScript 應用程式可以在新通知發生時立即接收，而無需重新整理頁面。在開始之前，請務必閱讀有關使用 [廣播通知頻道](/docs/{{version}}/notifications#broadcast-notifications) 的文件。

一旦您將通知設定為使用廣播頻道，您就可以使用 Echo 的 `notification` 方法來監聽廣播事件。請記住，頻道名稱應與接收通知的實體類別名稱相符：

```js tab=JavaScript
Echo.private(`App.Models.User.${userId}`)
    .notification((notification) => {
        console.log(notification.type);
    });
```

```js tab=React
import { useEchoModel } from "@laravel/echo-react";

const { channel } = useEchoModel('App.Models.User', userId);

channel().notification((notification) => {
    console.log(notification.type);
});
```

```vue tab=Vue
<script setup lang="ts">
import { useEchoModel } from "@laravel/echo-vue";

const { channel } = useEchoModel('App.Models.User', userId);

channel().notification((notification) => {
    console.log(notification.type);
});
</script>
```

在此範例中，所有透過 `broadcast` 頻道發送至 `App\Models\User` 實體的通知都會被該回呼接收。`App.Models.User.{id}` 頻道的頻道授權回呼已包含在您的應用程式 `routes/channels.php` 檔案中。


<a name="stop-listening-for-notifications"></a>
#### 停止監聽通知

如果您想在不 [離開頻道](#leaving-a-channel) 的情況下停止監聽通知，您可以使用 `stopListeningForNotification` 方法：

```js
const callback = (notification) => {
    console.log(notification.type);
}

// Start listening...
Echo.private(`App.Models.User.${userId}`)
    .notification(callback);

// Stop listening (callback must be the same)...
Echo.private(`App.Models.User.${userId}`)
    .stopListeningForNotification(callback);
```