# 廣播

- [簡介](#introduction)
- [快速入門](#quickstart)
- [伺服器端安裝](#server-side-installation)
    - [Reverb](#reverb)
    - [Pusher Channels](#pusher-channels)
    - [Ably](#ably)
- [客戶端安裝](#client-side-installation)
    - [Reverb](#client-reverb)
    - [Pusher Channels](#client-pusher-channels)
    - [Ably](#client-ably)
- [概念概覽](#concept-overview)
    - [使用範例應用程式](#using-example-application)
- [定義廣播事件](#defining-broadcast-events)
    - [廣播名稱](#broadcast-name)
    - [廣播資料](#broadcast-data)
    - [廣播佇列](#broadcast-queue)
    - [廣播條件](#broadcast-conditions)
    - [廣播與資料庫交易](#broadcasting-and-database-transactions)
- [頻道授權](#authorizing-channels)
    - [定義授權回呼](#defining-authorization-callbacks)
    - [定義頻道類別](#defining-channel-classes)
- [廣播事件](#broadcasting-events)
    - [僅廣播給其他人](#only-to-others)
    - [自訂連接](#customizing-the-connection)
    - [匿名事件](#anonymous-events)
    - [廣播例外處理](#rescuing-broadcasts)
- [接收廣播](#receiving-broadcasts)
    - [監聽事件](#listening-for-events)
    - [離開頻道](#leaving-a-channel)
    - [命名空間](#namespaces)
    - [使用 React 或 Vue](#using-react-or-vue)
- [存在頻道](#presence-channels)
    - [授權存在頻道](#authorizing-presence-channels)
    - [加入存在頻道](#joining-presence-channels)
    - [廣播至存在頻道](#broadcasting-to-presence-channels)
- [模型廣播](#model-broadcasting)
    - [模型廣播慣例](#model-broadcasting-conventions)
    - [監聽模型廣播](#listening-for-model-broadcasts)
- [客戶端事件](#client-events)
- [通知](#notifications)

<a name="introduction"></a>
## 簡介

在許多現代 Web 應用程式中，WebSocket 被用於實作即時、即時更新的使用者介面。當伺服器上的資料更新時，通常會透過 WebSocket 連線傳送訊息，由客戶端處理。WebSocket 提供更有效率的替代方案，取代持續輪詢應用程式伺服器以取得應在 UI 中反映的資料變更。

例如，假設您的應用程式能夠將使用者的資料匯出為 CSV 檔案並透過電子郵件傳送給他們。然而，建立此 CSV 檔案需要數分鐘，因此您選擇在 [佇列任務](/docs/{{version}}/queues) 中建立並寄送 CSV 檔案。當 CSV 檔案建立並寄送給使用者後，我們可以使用事件廣播來分派一個 `App\Events\UserDataExported` 事件，該事件將由我們應用程式的 JavaScript 接收。一旦接收到事件，我們可以向使用者顯示訊息，告知他們 CSV 檔案已透過電子郵件傳送給他們，而他們無需重新整理頁面。

為了協助您建構這些類型的功能，Laravel 讓您可以輕鬆地透過 WebSocket 連線「廣播」您的伺服器端 Laravel [事件](/docs/{{version}}/events)。廣播您的 Laravel 事件可讓您在伺服器端 Laravel 應用程式和客戶端 JavaScript 應用程式之間共用相同的事件名稱和資料。

廣播的核心概念很簡單：客戶端在前端連線到具名頻道，而您的 Laravel 應用程式則在後端向這些頻道廣播事件。這些事件可以包含您希望提供給前端的任何額外資料。


<a name="supported-drivers"></a>
#### 支援的驅動程式

預設情況下，Laravel 包含三個伺服器端廣播驅動程式供您選擇：[Laravel Reverb](https://reverb.laravel.com)、[Pusher Channels](https://pusher.com/channels) 和 [Ably](https://ably.com)。

> [!NOTE]
> 在深入了解事件廣播之前，請務必閱讀 Laravel 關於 [事件與監聽器](/docs/{{version}}/events) 的文件。


<a name="quickstart"></a>
## 快速入門

預設情況下，新的 Laravel 應用程式並未啟用廣播功能。您可以使用 `install:broadcasting` Artisan 命令啟用廣播：

```shell
php artisan install:broadcasting
```

`install:broadcasting` 命令會提示您要使用哪個事件廣播服務。此外，它還會建立 `config/broadcasting.php` 設定檔和 `routes/channels.php` 檔案，您可以在其中註冊應用程式的廣播授權路由和回呼。

Laravel 預設支援多種廣播驅動程式：[Laravel Reverb](/docs/{{version}}/reverb)、[Pusher Channels](https://pusher.com/channels)、[Ably](https://ably.com)，以及用於本機開發和偵錯的 `log` 驅動程式。此外，還包含一個 `null` 驅動程式，可讓您在測試期間停用廣播。每個驅動程式的設定範例都包含在 `config/broadcasting.php` 設定檔中。

您應用程式的所有事件廣播設定都儲存在 `config/broadcasting.php` 設定檔中。如果您的應用程式中不存在此檔案，請不用擔心；它會在您執行 `install:broadcasting` Artisan 命令時建立。


<a name="quickstart-next-steps"></a>
#### 下一步

啟用事件廣播後，您就可以進一步了解 [定義廣播事件](#defining-broadcast-events) 和 [監聽事件](#listening-for-events)。如果您正在使用 Laravel 的 React 或 Vue [入門套件](/docs/{{version}}/starter-kits)，您可以使用 Echo 的 [useEcho hook](#using-react-or-vue) 監聽事件。

> [!NOTE]
> 在廣播任何事件之前，您應該先設定並執行一個 [佇列工作者](/docs/{{version}}/queues)。所有事件廣播都透過佇列任務完成，這樣您的應用程式的回應時間就不會因事件廣播而受到嚴重影響。

<a name="server-side-installation"></a>
## 伺服器端安裝

為了開始使用 Laravel 的事件廣播，我們需要在 Laravel 應用程式中進行一些設定，並安裝一些套件。

事件廣播是透過伺服器端的廣播驅動程式來完成的，它會廣播您的 Laravel 事件，以便 Laravel Echo (一個 JavaScript 函式庫) 可以在瀏覽器客戶端中接收它們。別擔心，我們將逐步引導您完成安裝過程的每個部分。

<a name="reverb"></a>
### Reverb

為了在使用 Reverb 作為事件廣播器時快速啟用對 Laravel 廣播功能的支援，請使用 `--reverb` 選項來執行 `install:broadcasting` Artisan 指令。此 Artisan 指令將安裝 Reverb 所需的 Composer 和 NPM 套件，並使用適當的變數更新您應用程式的 `.env` 檔案：

```shell
php artisan install:broadcasting --reverb
```

<a name="reverb-manual-installation"></a>
#### 手動安裝

執行 `install:broadcasting` 指令時，系統將提示您安裝 [Laravel Reverb](/docs/{{version}}/reverb)。當然，您也可以使用 Composer 套件管理器手動安裝 Reverb：

```shell
composer require laravel/reverb
```

套件安裝完成後，您可以執行 Reverb 的安裝指令來發布設定檔，新增 Reverb 所需的環境變數，並在您的應用程式中啟用事件廣播：

```shell
php artisan reverb:install
```

您可以在 [Reverb 文件](/docs/{{version}}/reverb)中找到詳細的 Reverb 安裝和使用說明。

<a name="pusher-channels"></a>
### Pusher Channels

為了在使用 Pusher 作為事件廣播器時快速啟用對 Laravel 廣播功能的支援，請使用 `--pusher` 選項來執行 `install:broadcasting` Artisan 指令。此 Artisan 指令將提示您輸入 Pusher 憑證，安裝 Pusher 的 PHP 和 JavaScript SDK，並使用適當的變數更新您應用程式的 `.env` 檔案：

```shell
php artisan install:broadcasting --pusher
```

<a name="pusher-manual-installation"></a>
#### 手動安裝

若要手動安裝 Pusher 支援，您應該使用 Composer 套件管理器安裝 Pusher Channels PHP SDK：

```shell
composer require pusher/pusher-php-server
```

接下來，您應該在 `config/broadcasting.php` 設定檔中配置您的 Pusher Channels 憑證。此檔案中已包含一個 Pusher Channels 設定範例，讓您可以快速指定您的 key、secret 和 application ID。通常，您應該在您應用程式的 `.env` 檔案中設定您的 Pusher Channels 憑證：

```ini
PUSHER_APP_ID="your-pusher-app-id"
PUSHER_APP_KEY="your-pusher-key"
PUSHER_APP_SECRET="your-pusher-secret"
PUSHER_HOST=
PUSHER_PORT=443
PUSHER_SCHEME="https"
PUSHER_APP_CLUSTER="mt1"
```

`config/broadcasting.php` 檔案的 `pusher` 設定也允許您指定 Channels 支援的額外 `options`，例如 cluster。

然後，在您應用程式的 `.env` 檔案中，將 `BROADCAST_CONNECTION` 環境變數設定為 `pusher`：

```ini
BROADCAST_CONNECTION=pusher
```

最後，您已準備好安裝並設定 [Laravel Echo](#client-side-installation)，它將在客戶端接收廣播事件。

<a name="ably"></a>
### Ably

> [!NOTE]
> The documentation below discusses how to use Ably in "Pusher compatibility" mode. However, the Ably team recommends and maintains a broadcaster and Echo client that is able to take advantage of the unique capabilities offered by Ably. For more information on using the Ably maintained drivers, please [consult Ably's Laravel broadcaster documentation](https://github.com/ably/laravel-broadcaster).

為了在使用 [Ably](https://ably.com) 作為事件廣播器時快速啟用對 Laravel 廣播功能的支援，請使用 `--ably` 選項來執行 `install:broadcasting` Artisan 指令。此 Artisan 指令將提示您輸入 Ably 憑證，安裝 Ably 的 PHP 和 JavaScript SDK，並使用適當的變數更新您應用程式的 `.env` 檔案：

```shell
php artisan install:broadcasting --ably
```

**在繼續之前，您應該在您的 Ably 應用程式設定中啟用 Pusher 協定支援。您可以在您的 Ably 應用程式設定儀表板的「Protocol Adapter Settings」部分啟用此功能。**

<a name="ably-manual-installation"></a>
#### 手動安裝

若要手動安裝 Ably 支援，您應該使用 Composer 套件管理器安裝 Ably PHP SDK：

```shell
composer require ably/ably-php
```

接下來，您應該在 `config/broadcasting.php` 設定檔中配置您的 Ably 憑證。此檔案中已包含一個 Ably 設定範例，讓您可以快速指定您的 key。通常，此值應透過 `ABLY_KEY` [環境變數](/docs/{{version}}/configuration#environment-configuration)設定：

```ini
ABLY_KEY=your-ably-key
```

然後，在您應用程式的 `.env` 檔案中，將 `BROADCAST_CONNECTION` 環境變數設定為 `ably`：

```ini
BROADCAST_CONNECTION=ably
```

最後，您已準備好安裝並設定 [Laravel Echo](#client-side-installation)，它將在客戶端接收廣播事件。

<a name="client-side-installation"></a>
## 客戶端安裝


<a name="client-reverb"></a>
### Reverb

[Laravel Echo](https://github.com/laravel/echo) 是一個 JavaScript 函式庫，可讓您輕鬆訂閱頻道並監聽由您伺服器端廣播驅動程式廣播的事件。

透過 `install:broadcasting` Artisan 指令安裝 Laravel Reverb 時，Reverb 和 Echo 的鷹架與設定將會自動注入您的應用程式。然而，如果您希望手動設定 Laravel Echo，您可以按照以下說明進行。


<a name="reverb-client-manual-installation"></a>
#### 手動安裝

若要手動為應用程式的前端設定 Laravel Echo，請先安裝 `pusher-js` 套件，因為 Reverb 使用 Pusher 協定來進行 WebSocket 訂閱、頻道和訊息：

```shell
npm install --save-dev laravel-echo pusher-js
```

一旦 Echo 安裝完成，您就可以在應用程式的 JavaScript 中建立一個全新的 Echo 實例。一個很好的位置是在 Laravel 框架中包含的 `resources/js/bootstrap.js` 檔案底部：

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

接著，您應該編譯您的應用程式資源：

```shell
npm run build
```

> [!WARNING]
> The Laravel Echo `reverb` broadcaster requires laravel-echo v1.16.0+.


<a name="client-pusher-channels"></a>
### Pusher Channels

[Laravel Echo](https://github.com/laravel/echo) 是一個 JavaScript 函式庫，可讓您輕鬆訂閱頻道並監聽由您伺服器端廣播驅動程式廣播的事件。

透過 `install:broadcasting --pusher` Artisan 指令安裝廣播支援時，Pusher 和 Echo 的鷹架與設定將會自動注入您的應用程式。然而，如果您希望手動設定 Laravel Echo，您可以按照以下說明進行。


<a name="pusher-client-manual-installation"></a>
#### 手動安裝

若要手動為應用程式的前端設定 Laravel Echo，請先安裝 `laravel-echo` 和 `pusher-js` 套件，這些套件使用 Pusher 協定來進行 WebSocket 訂閱、頻道和訊息：

```shell
npm install --save-dev laravel-echo pusher-js
```

一旦 Echo 安裝完成，您就可以在應用程式的 `resources/js/bootstrap.js` 檔案中建立一個全新的 Echo 實例：

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

接著，您應該在應用程式的 `.env` 檔案中定義 Pusher 環境變數的適當值。如果這些變數尚未存在於您的 `.env` 檔案中，您應該將它們加入：

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

一旦您根據應用程式的需求調整了 Echo 設定，您就可以編譯應用程式的資源了：

```shell
npm run build
```

> [!NOTE]
> 若要深入了解如何編譯您的應用程式 JavaScript 資源，請查閱 [Vite](/docs/{{version}}/vite) 的文件。


<a name="using-an-existing-client-instance"></a>
#### 使用現有的客戶端實例

如果您已經有一個預先設定的 Pusher Channels 客戶端實例，並且希望 Echo 使用它，您可以透過 `client` 設定選項將其傳遞給 Echo：

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
> 下方說明文件討論如何以「Pusher 相容模式」使用 Ably。然而，Ably 團隊推薦並維護一個廣播器與 Echo 客戶端，能夠利用 Ably 提供的獨特功能。有關使用 Ably 維護的驅動程式的更多資訊，請[查閱 Ably 的 Laravel 廣播器說明文件](https://github.com/ably/laravel-broadcaster)。

[Laravel Echo](https://github.com/laravel/echo) 是一個 JavaScript 函式庫，可讓您輕鬆訂閱頻道並監聽由伺服器端廣播驅動程式廣播的事件。

透過 `install:broadcasting --ably` Artisan 指令安裝廣播支援時，Ably 與 Echo 的基礎架構和組態將會自動注入到您的應用程式。然而，如果您希望手動設定 Laravel Echo，您可以依照以下說明操作。

<a name="ably-client-manual-installation"></a>
#### 手動安裝

若要手動設定您應用程式前端的 Laravel Echo，請先安裝 `laravel-echo` 和 `pusher-js` 套件，這些套件利用 Pusher 通訊協定來處理 WebSocket 訂閱、頻道和訊息：

```shell
npm install --save-dev laravel-echo pusher-js
```

**在繼續之前，您應該在您的 Ably 應用程式設定中啟用 Pusher 通訊協定支援。您可以在 Ably 應用程式設定儀表板的「通訊協定轉接器設定」部分啟用此功能。**

一旦 Echo 安裝完成，您就可以在您應用程式的 `resources/js/bootstrap.js` 檔案中建立一個新的 Echo 實例：

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

您可能已經注意到，我們的 Ably Echo 組態參考了一個 `VITE_ABLY_PUBLIC_KEY` 環境變數。此變數的值應該是您的 Ably 公開金鑰。您的公開金鑰是您的 Ably 金鑰中，冒號 (`:`) 字元之前的部分。

一旦您根據您的需求調整了 Echo 組態，您就可以編譯應用程式的資源了：

```shell
npm run dev
```

> [!NOTE]
> 若要了解更多關於編譯您應用程式的 JavaScript 資源的資訊，請查閱 [Vite](/docs/{{version}}/vite) 的說明文件。

<a name="concept-overview"></a>
## 概念概覽

Laravel 的事件廣播 (event broadcasting) 讓您可以透過基於驅動器的方法 (driver-based approach) 將伺服器端的 Laravel 事件廣播至客戶端的 JavaScript 應用程式，實現 WebSockets。目前，Laravel 內建了 [Laravel Reverb](https://reverb.laravel.com)、[Pusher Channels](https://pusher.com/channels) 和 [Ably](https://ably.com) 驅動器。這些事件可以使用 [Laravel Echo](#client-side-installation) JavaScript 套件在客戶端輕鬆消費。

事件透過「頻道」(channels) 進行廣播，這些頻道可以指定為公開 (public) 或私人 (private)。應用程式的任何訪客都可以訂閱公開頻道，無需任何身份驗證或授權；然而，若要訂閱私人頻道，使用者必須經過身份驗證並獲得授權才能監聽該頻道。

<a name="using-example-application"></a>
### 使用範例應用程式

在深入探討事件廣播的每個元件之前，讓我們先以一個電子商務商店為例，進行高層次的概覽。

在我們的應用程式中，假設我們有一個頁面允許使用者查看訂單的運送狀態。我們也假設當應用程式處理運送狀態更新時，會觸發 `OrderShipmentStatusUpdated` 事件：

```php
use App\Events\OrderShipmentStatusUpdated;

OrderShipmentStatusUpdated::dispatch($order);
```

<a name="the-shouldbroadcast-interface"></a>
#### `ShouldBroadcast` 介面

當使用者查看他們的訂單時，我們不希望他們必須重新整理頁面才能查看狀態更新。相反地，我們希望在狀態更新建立時，就將其廣播到應用程式。因此，我們需要用 `ShouldBroadcast` 介面標記 `OrderShipmentStatusUpdated` 事件。這會指示 Laravel 在事件觸發時廣播該事件：

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

`ShouldBroadcast` 介面要求我們的事件定義一個 `broadcastOn` 方法。此方法負責回傳事件應廣播到的頻道。此方法的一個空樣板已在生成的事件類別中定義，所以我們只需要填寫其詳細資訊。我們只希望訂單的建立者能夠查看狀態更新，因此我們將在與訂單綁定的私人頻道上廣播該事件：

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

如果您希望事件廣播到多個頻道，您可以改為回傳一個 `array`：

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

請記住，使用者必須獲得授權才能監聽私人頻道。我們可以在應用程式的 `routes/channels.php` 檔案中定義我們的頻道授權規則。在此範例中，我們需要驗證任何嘗試監聽私人 `orders.1` 頻道的用戶，確實是該訂單的建立者：

```php
use App\Models\Order;
use App\Models\User;

Broadcast::channel('orders.{orderId}', function (User $user, int $orderId) {
    return $user->id === Order::findOrNew($orderId)->user_id;
});
```

`channel` 方法接受兩個引數：頻道的名稱，以及一個回呼 (callback)，此回呼會回傳 `true` 或 `false`，指示使用者是否獲授權監聽該頻道。

所有授權回呼都會將目前已驗證的使用者作為第一個引數，並將任何額外的萬用字元參數作為其後續引數。在此範例中，我們使用 `{orderId}` 佔位符表示頻道名稱的「ID」部分是萬用字元。

<a name="listening-for-event-broadcasts"></a>
#### 監聽廣播事件

接下來，剩下的就是監聽我們的 JavaScript 應用程式中的事件了。我們可以使用 [Laravel Echo](#client-side-installation) 來做到這一點。Laravel Echo 內建的 React 和 Vue Hooks 讓入門變得輕而易舉，並且預設情況下，事件的所有公開屬性都將包含在廣播事件中：

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

要通知 Laravel 某個事件應被廣播，您必須在事件類別上實作 `Illuminate\Contracts\Broadcasting\ShouldBroadcast` 介面。此介面已匯入框架產生所有事件類別中，因此您可以輕鬆將其新增至任何事件。

`ShouldBroadcast` 介面要求您實作一個方法：`broadcastOn`。`broadcastOn` 方法應回傳一個頻道或一個頻道陣列，該事件將在其上廣播。這些頻道應為 `Channel`、`PrivateChannel` 或 `PresenceChannel` 的實例。`Channel` 的實例代表任何使用者都可以訂閱的公開頻道，而 `PrivateChannels` 和 `PresenceChannels` 則代表需要[頻道授權](#authorizing-channels)的私人頻道：

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

實作 `ShouldBroadcast` 介面後，您只需像往常一樣[觸發事件](/docs/{{version}}/events)即可。一旦事件觸發，一個[佇列作業](/docs/{{version}}/queues)將會使用您指定的廣播驅動器自動廣播該事件。

<a name="broadcast-name"></a>
### 廣播名稱

預設情況下，Laravel 會使用事件的類別名稱廣播事件。然而，您可以透過在事件上定義 `broadcastAs` 方法來客製化廣播名稱：

```php
/**
 * The event's broadcast name.
 */
public function broadcastAs(): string
{
    return 'server.created';
}
```

如果您使用 `broadcastAs` 方法客製化廣播名稱，您應該確保使用開頭帶有 `.` 字元的監聽器來註冊。這將指示 Echo 不要在事件名稱前加上應用程式的命名空間：

```javascript
.listen('.server.created', function (e) {
    // ...
});
```

<a name="broadcast-data"></a>
### 廣播資料

當事件被廣播時，其所有的 `public` 屬性會自動序列化並作為事件的負載廣播，讓您可以從 JavaScript 應用程式存取其任何公開資料。舉例來說，如果您的事件有一個包含 Eloquent 模型的單一 public `$user` 屬性，那麼事件的廣播負載將會是：

```json
{
    "user": {
        "id": 1,
        "name": "Patrick Stewart"
        ...
    }
}
```

然而，如果您希望對廣播負載有更細緻的控制，您可以為事件新增一個 `broadcastWith` 方法。此方法應回傳您希望作為事件負載廣播的資料陣列：

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

預設情況下，每個廣播事件都會被放置在 `queue.php` 設定檔中指定之預設佇列連接的預設佇列上。您可以透過在事件類別上定義 `connection` 和 `queue` 屬性來客製化廣播器使用的佇列連接和名稱：

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

或者，您可以透過在事件上定義 `broadcastQueue` 方法來客製化佇列名稱：

```php
/**
 * The name of the queue on which to place the broadcasting job.
 */
public function broadcastQueue(): string
{
    return 'default';
}
```

如果您想使用 `sync` 佇列而不是預設佇列驅動器來廣播事件，您可以實作 `ShouldBroadcastNow` 介面，而非 `ShouldBroadcast`：

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

有時，您希望僅在特定條件為真時才廣播事件。您可以透過在事件類別中新增 `broadcastWhen` 方法來定義這些條件：

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

當廣播事件在資料庫交易中分派時，它們可能會在資料庫交易提交前被佇列處理。發生這種情況時，您在資料庫交易期間對模型或資料庫記錄所做的任何更新可能尚未反映在資料庫中。此外，在交易中建立的任何模型或資料庫記錄可能不存在於資料庫中。如果您的事件依賴於這些模型，則在處理廣播事件的作業時可能會發生意外錯誤。

如果您的佇列連線的 `after_commit` 設定選項設為 `false`，您仍然可以透過在事件類別上實作 `ShouldDispatchAfterCommit` 介面，來指示特定廣播事件應在所有開啟的資料庫交易提交後分派：

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
> 若要進一步了解如何處理這些問題，請查閱有關[佇列作業與資料庫交易](/docs/{{version}}/queues#jobs-and-database-transactions)的說明文件。

<a name="authorizing-channels"></a>
## 頻道授權

私有頻道要求您授權目前已認證的使用者實際監聽該頻道。這可以透過向您的 Laravel 應用程式發出一個包含頻道名稱的 HTTP 請求來完成，讓您的應用程式判斷該使用者是否可以監聽該頻道。當使用 [Laravel Echo](#client-side-installation) 時，授權私有頻道訂閱的 HTTP 請求會自動發出。

啟用廣播時，Laravel 會自動註冊 `/broadcasting/auth` 路由來處理授權請求。此 `/broadcasting/auth` 路由會自動放置於 `web` 中介層群組中。

<a name="defining-authorization-callbacks"></a>
### 定義授權回呼

接下來，我們需要定義實際判斷目前已認證使用者是否可以監聽特定頻道的邏輯。這是在透過 `install:broadcasting` Artisan 命令建立的 `routes/channels.php` 檔案中完成的。在這個檔案中，您可以使用 `Broadcast::channel` 方法來註冊頻道授權回呼：

```php
use App\Models\User;

Broadcast::channel('orders.{orderId}', function (User $user, int $orderId) {
    return $user->id === Order::findOrNew($orderId)->user_id;
});
```

`channel` 方法接受兩個引數：頻道名稱以及一個回呼，該回呼會回傳 `true` 或 `false`，指示使用者是否被授權監聽該頻道。

所有授權回呼都會將目前已認證的使用者作為第一個引數，並將任何額外的萬用字元參數作為其後的引數。在此範例中，我們使用 `{orderId}` 預留位置來指示頻道名稱的「ID」部分是萬用字元。

您可以使用 `channel:list` Artisan 命令來檢視應用程式的廣播授權回呼清單：

```shell
php artisan channel:list
```

<a name="authorization-callback-model-binding"></a>
#### 授權回呼模型綁定

就像 HTTP 路由一樣，頻道路由也可以利用隱式與顯式 [路由模型綁定](/docs/{{version}}/routing#route-model-binding)。例如，您不是接收字串或數字的訂單 ID，而是可以請求一個實際的 `Order` 模型實例：

```php
use App\Models\Order;
use App\Models\User;

Broadcast::channel('orders.{order}', function (User $user, Order $order) {
    return $user->id === $order->user_id;
});
```

> [!WARNING]
> 與 HTTP 路由模型綁定不同，頻道模型綁定不支援自動 [隱式模型綁定作用域](/docs/{{version}}/routing#implicit-model-binding-scoping)。然而，這很少是一個問題，因為大多數頻道可以根據單一模型的唯一主鍵來進行作用域。

<a name="authorization-callback-authentication"></a>
#### 授權回呼認證

私有與存在廣播頻道會透過應用程式的預設認證守衛來認證目前的使用者。如果使用者未經認證，頻道授權會自動被拒絕，且授權回呼永遠不會執行。然而，您可以在必要時分配多個自訂守衛，用於認證傳入的請求：

```php
Broadcast::channel('channel', function () {
    // ...
}, ['guards' => ['web', 'admin']]);
```

<a name="defining-channel-classes"></a>
### 定義頻道類別

如果您的應用程式正在處理許多不同的頻道，您的 `routes/channels.php` 檔案可能會變得非常臃腫。因此，您可以使用頻道類別來取代使用閉包授權頻道。若要產生一個頻道類別，請使用 `make:channel` Artisan 命令。此命令會將一個新的頻道類別放置在 `App/Broadcasting` 目錄中。

```shell
php artisan make:channel OrderChannel
```

接下來，在您的 `routes/channels.php` 檔案中註冊您的頻道：

```php
use App\Broadcasting\OrderChannel;

Broadcast::channel('orders.{order}', OrderChannel::class);
```

最後，您可以將頻道的授權邏輯放置在頻道類別的 `join` 方法中。這個 `join` 方法會包含您通常會放置在頻道授權閉包中的相同邏輯。您也可以利用頻道模型綁定：

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
> 就像 Laravel 中的許多其他類別一樣，頻道類別會自動被 [服務容器](/docs/{{version}}/container) 解析。因此，您可以在其建構函式中型別提示頻道所需的任何依賴。

<a name="broadcasting-events"></a>
## 廣播事件

一旦您定義了一個事件並將其標記為 `ShouldBroadcast` 介面，您只需使用事件的 dispatch 方法來觸發該事件。事件分發器將會注意到該事件已標記為 `ShouldBroadcast` 介面，並將事件排入佇列以進行廣播：

```php
use App\Events\OrderShipmentStatusUpdated;

OrderShipmentStatusUpdated::dispatch($order);
```

<a name="only-to-others"></a>
### 僅廣播給其他人

在構建使用事件廣播的應用程式時，您偶爾可能需要將事件廣播給特定頻道的所有訂閱者，除了當前用戶之外。您可以使用 `broadcast` 輔助函式和 `toOthers` 方法來實現此目的：

```php
use App\Events\OrderShipmentStatusUpdated;

broadcast(new OrderShipmentStatusUpdated($update))->toOthers();
```

為了更好地理解您何時可能需要使用 `toOthers` 方法，讓我們想像一個任務列表應用程式，用戶可以通過輸入任務名稱來創建新任務。要創建任務，您的應用程式可能會向 `/task` URL 發出請求，該請求會廣播任務的創建並返回新任務的 JSON 表示。當您的 JavaScript 應用程式從端點接收到響應時，它可能會直接將新任務插入到其任務列表，如下所示：

```js
axios.post('/task', task)
    .then((response) => {
        this.tasks.push(response.data);
    });
```

然而，請記住我們也廣播了任務的創建。如果您的 JavaScript 應用程式也正在監聽此事件以將任務添加到任務列表，那麼您的列表中將會有重複的任務：一個來自端點，另一個來自廣播。您可以使用 `toOthers` 方法來指示廣播器不要將事件廣播給當前用戶，從而解決此問題。

> [!WARNING]
> 您的事件必須使用 `Illuminate\Broadcasting\InteractsWithSockets` trait 才能呼叫 `toOthers` 方法。

<a name="only-to-others-configuration"></a>
#### 配置

當您初始化 Laravel Echo 實例時，會為連接分配一個 socket ID。如果您使用全域 [Axios](https://github.com/axios/axios) 實例從您的 JavaScript 應用程式發出 HTTP 請求，該 socket ID 將會自動作為 `X-Socket-ID` 標頭附加到每個發出的請求。然後，當您呼叫 `toOthers` 方法時，Laravel 將從標頭中提取 socket ID，並指示廣播器不要向具有該 socket ID 的任何連接進行廣播。

如果您不使用全域 Axios 實例，則需要手動配置您的 JavaScript 應用程式，使其在所有發出的請求中發送 `X-Socket-ID` 標頭。您可以使用 `Echo.socketId` 方法來檢索 socket ID：

```js
var socketId = Echo.socketId();
```

<a name="customizing-the-connection"></a>
### 自訂連接

如果您的應用程式與多個廣播連接互動，並且您希望使用預設廣播器之外的廣播器來廣播事件，您可以使用 `via` 方法指定要將事件推送到哪個連接：

```php
use App\Events\OrderShipmentStatusUpdated;

broadcast(new OrderShipmentStatusUpdated($update))->via('pusher');
```

或者，您可以通過在事件的建構式中呼叫 `broadcastVia` 方法來指定事件的廣播連接。但是，在此之前，您應該確保事件類別使用了 `InteractsWithBroadcasting` trait：

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

有時，您可能希望向應用程式的前端廣播一個簡單的事件，而無需創建專用的事件類別。為此，`Broadcast` facade 允許您廣播「匿名事件」：

```php
Broadcast::on('orders.'.$order->id)->send();
```

上面的範例將廣播以下事件：

```json
{
    "event": "AnonymousEvent",
    "data": "[]",
    "channel": "orders.1"
}
```

使用 `as` 和 `with` 方法，您可以自訂事件名稱和資料：

```php
Broadcast::on('orders.'.$order->id)
    ->as('OrderPlaced')
    ->with($order)
    ->send();
```

上面的範例將廣播如下事件：

```json
{
    "event": "OrderPlaced",
    "data": "{ id: 1, total: 100 }",
    "channel": "orders.1"
}
```

如果您想在 private 或 presence 頻道上廣播匿名事件，您可以使用 `private` 和 `presence` 方法：

```php
Broadcast::private('orders.'.$order->id)->send();
Broadcast::presence('channels.'.$channel->id)->send();
```

使用 `send` 方法廣播匿名事件會將事件分發到您應用程式的 [佇列](/docs/{{version}}/queues) 進行處理。但是，如果您想立即廣播事件，您可以使用 `sendNow` 方法：

```php
Broadcast::on('orders.'.$order->id)->sendNow();
```

要將事件廣播給除了當前驗證用戶之外的所有頻道訂閱者，您可以呼叫 `toOthers` 方法：

```php
Broadcast::on('orders.'.$order->id)
    ->toOthers()
    ->send();
```

<a name="rescuing-broadcasts"></a>
### 廣播例外處理

當您的應用程式的佇列伺服器不可用或 Laravel 在廣播事件時遇到錯誤時，通常會拋出一個例外，這會導致終端用戶看到應用程式錯誤。由於事件廣播通常是應用程式核心功能的輔助性功能，您可以通過在事件上實作 `ShouldRescue` 介面來防止這些例外中斷用戶體驗。

實作 `ShouldRescue` 介面的事件會自動在廣播嘗試期間利用 Laravel 的 [rescue 輔助函式](/docs/{{version}}/helpers#method-rescue)。此輔助函式會捕獲任何例外，將它們報告給應用程式的例外處理器進行記錄，並允許應用程式正常繼續執行，而不會中斷用戶的工作流程：

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

當您已經[安裝並實例化 Laravel Echo](#client-side-installation) 後，就可以開始監聽從您的 Laravel 應用程式廣播的事件。首先，使用 `channel` 方法取得一個頻道的實例，然後呼叫 `listen` 方法來監聽指定的事件：

```js
Echo.channel(`orders.${this.order.id}`)
    .listen('OrderShipmentStatusUpdated', (e) => {
        console.log(e.order.name);
    });
```

如果您想監聽私有頻道上的事件，請改用 `private` 方法。您可以繼續鏈式呼叫 `listen` 方法，以便監聽單一頻道上的多個事件：

```js
Echo.private(`orders.${this.order.id}`)
    .listen(/* ... */)
    .listen(/* ... */)
    .listen(/* ... */);
```

<a name="stop-listening-for-events"></a>
#### 停止監聽事件

如果您想停止監聽某個事件，但不想[離開頻道](#leaving-a-channel)，可以使用 `stopListening` 方法：

```js
Echo.private(`orders.${this.order.id}`)
    .stopListening('OrderShipmentStatusUpdated');
```

<a name="leaving-a-channel"></a>
### 離開頻道

要離開頻道，您可以在您的 Echo 實例上呼叫 `leaveChannel` 方法：

```js
Echo.leaveChannel(`orders.${this.order.id}`);
```

如果您想離開一個頻道及其相關聯的私有頻道與存在頻道，您可以呼叫 `leave` 方法：

```js
Echo.leave(`orders.${this.order.id}`);
```

<a name="namespaces"></a>
### 命名空間

您可能已經注意到，在上述範例中，我們沒有指定事件類別的完整 `App\Events` 命名空間。這是因為 Echo 會自動假設事件位於 `App\Events` 命名空間中。但是，您可以在實例化 Echo 時，透過傳遞 `namespace` 設定選項來配置根命名空間：

```js
window.Echo = new Echo({
    broadcaster: 'pusher',
    // ...
    namespace: 'App.Other.Namespace'
});
```

或者，您可以在使用 Echo 訂閱事件類別時，在事件類別前加上 `.`。這將允許您始終指定完整的類別名稱：

```js
Echo.channel('orders')
    .listen('.Namespace\\Event\\Class', (e) => {
        // ...
    });
```

<a name="using-react-or-vue"></a>
### 使用 React 或 Vue

Laravel Echo 包含 React 和 Vue 鉤子，讓監聽事件變得輕鬆無痛。要開始使用，請呼叫 `useEcho` 鉤子，它用於監聽私有事件。當消費元件卸載時，`useEcho` 鉤子將自動離開頻道：

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

您可以透過向 `useEcho` 提供事件陣列來監聽多個事件：

```js
useEcho(
    `orders.${orderId}`,
    ["OrderShipmentStatusUpdated", "OrderShipped"],
    (e) => {
        console.log(e.order);
    },
);
```

您還可以指定廣播事件資料負載的形狀，提供更好的型別安全與編輯便利性：

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

當消費元件卸載時，`useEcho` 鉤子將自動離開頻道；但是，您可以在必要時利用回傳的函式，透過程式手動停止/啟動監聽頻道：

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
#### 連接到公有頻道

要連接到公有頻道，您可以使用 `useEchoPublic` 鉤子：

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
#### 連接到存在頻道

要連接到存在頻道，您可以使用 `useEchoPresence` 鉤子：

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

<a name="presence-channels"></a>
## 存在頻道

存在頻道建立在私有頻道的安全性之上，同時提供額外的功能，讓您得知誰訂閱了該頻道。這使得建立強大、協作的應用程式功能變得容易，例如當其他使用者正在查看相同頁面時通知使用者，或列出聊天室中的成員。

<a name="authorizing-presence-channels"></a>
### 授權存在頻道

所有存在頻道也都是私有頻道；因此，使用者必須被[授權存取它們](#authorizing-channels)。然而，當為存在頻道定義授權回呼時，如果使用者被授權加入頻道，您將不會回傳 `true`。取而代之的是，您應該回傳一個關於使用者的資料陣列。

授權回呼所回傳的資料將會提供給 JavaScript 應用程式中的存在頻道事件監聽器。如果使用者未被授權加入存在頻道，您應該回傳 `false` 或 `null`：

```php
use App\Models\User;

Broadcast::channel('chat.{roomId}', function (User $user, int $roomId) {
    if ($user->canJoinRoom($roomId)) {
        return ['id' => $user->id, 'name' => $user->name];
    }
});
```

<a name="joining-presence-channels"></a>
### 加入存在頻道

若要加入存在頻道，您可以使用 Echo 的 `join` 方法。 `join` 方法將會回傳一個 `PresenceChannel` 實作，它除了提供 `listen` 方法外，還允許您訂閱 `here`、`joining` 和 `leaving` 事件。

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

一旦頻道成功加入，`here` 回呼將會立即執行，並會收到一個包含目前訂閱該頻道的其他所有使用者資訊的陣列。當有新使用者加入頻道時，`joining` 方法將會執行；而當使用者離開頻道時，`leaving` 方法將會執行。當驗證端點回傳非 200 的 HTTP 狀態碼，或解析回傳的 JSON 時發生問題，`error` 方法將會執行。

<a name="broadcasting-to-presence-channels"></a>
### 廣播至存在頻道

存在頻道可以像公開頻道或私有頻道一樣接收事件。以聊天室為例，我們可能希望將 `NewMessage` 事件廣播到聊天室的存在頻道。為此，我們將從事件的 `broadcastOn` 方法回傳 `PresenceChannel` 的實例：

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

與其他事件一樣，您可以使用 `broadcast` 輔助函式和 `toOthers` 方法來排除目前使用者接收廣播：

```php
broadcast(new NewMessage($message));

broadcast(new NewMessage($message))->toOthers();
```

與其他類型的事件一樣，您可以使用 Echo 的 `listen` 方法來監聽傳送至存在頻道的事件：

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
> 在閱讀以下關於模型廣播的文件之前，我們建議您先熟悉 Laravel 模型廣播服務的通用概念，以及如何手動建立和監聽廣播事件。

當應用程式的 [Eloquent 模型](/docs/{{version}}/eloquent)被建立、更新或刪除時，廣播事件是很常見的。當然，這可以透過手動 [為 Eloquent 模型狀態變更定義自訂事件](/docs/{{version}}/eloquent#events) 並使用 `ShouldBroadcast` 介面標記這些事件來輕鬆實現。

然而，如果您的應用程式沒有將這些事件用於其他任何目的，那麼僅僅為了廣播而建立事件類別可能會很麻煩。為了解決這個問題，Laravel 允許您指定 Eloquent 模型應自動廣播其狀態變更。

要開始使用，您的 Eloquent 模型應使用 `Illuminate\Database\Eloquent\BroadcastsEvents` trait。此外，模型應定義一個 `broadcastOn` 方法，該方法將返回模型事件應廣播到的頻道陣列：

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

一旦您的模型包含此 trait 並定義了其廣播頻道，它將在模型實例被建立、更新、刪除、軟刪除或恢復時，自動開始廣播事件。

此外，您可能已經注意到 `broadcastOn` 方法接收一個字串 `$event` 引數。此引數包含模型上發生的事件類型，其值將是 `created`、`updated`、`deleted`、`trashed` 或 `restored`。透過檢查此變數的值，您可以確定模型應為特定事件廣播到哪些頻道（如果有的話）：

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
#### 自訂模型廣播事件建立

有時，您可能希望自訂 Laravel 建立底層模型廣播事件的方式。您可以透過在 Eloquent 模型上定義一個 `newBroadcastableEvent` 方法來實現此目的。此方法應返回一個 `Illuminate\Database\Eloquent\BroadcastableModelEventOccurred` 實例：

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

您可能已經注意到，上述模型範例中的 `broadcastOn` 方法沒有返回 `Channel` 實例。相反地，它直接返回了 Eloquent 模型。如果您的模型 `broadcastOn` 方法返回（或包含在該方法返回的陣列中）一個 Eloquent 模型實例，Laravel 將自動為該模型實例化一個私有頻道實例，使用模型的類別名稱和主鍵識別碼作為頻道名稱。

因此，一個 `App\Models\User` 模型，其 `id` 為 `1`，將被轉換為一個 `Illuminate\Broadcasting\PrivateChannel` 實例，名稱為 `App.Models.User.1`。當然，除了從模型的 `broadcastOn` 方法返回 Eloquent 模型實例外，您還可以返回完整的 `Channel` 實例，以便完全控制模型的頻道名稱：

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

如果您打算從模型的 `broadcastOn` 方法明確返回一個頻道實例，您可以將一個 Eloquent 模型實例傳遞給頻道的建構函式。這樣做時，Laravel 將使用上述討論的模型頻道慣例，將 Eloquent 模型轉換為頻道名稱字串：

```php
return [new Channel($this->user)];
```

如果您需要確定模型的頻道名稱，您可以呼叫任何模型實例上的 `broadcastChannel` 方法。例如，對於一個 `id` 為 `1` 的 `App\Models\User` 模型，此方法會返回字串 `App.Models.User.1`：

```php
$user->broadcastChannel();
```

<a name="model-broadcasting-event-conventions"></a>
#### 事件慣例

由於模型廣播事件與您的應用程式 `App\Events` 目錄中的「實際」事件沒有關聯，因此它們會根據慣例被賦予名稱和有效載荷。Laravel 的慣例是使用模型的類別名稱（不包含命名空間）和觸發廣播的模型事件名稱來廣播事件。

因此，例如，對 `App\Models\Post` 模型進行更新，將向您的客戶端應用程式廣播一個名為 `PostUpdated` 的事件，其有效載荷如下：

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

如果您願意，可以透過在模型中新增 `broadcastAs` 和 `broadcastWith` 方法來定義自訂廣播名稱和有效載荷。這些方法會接收正在發生的模型事件/操作的名稱，讓您可以為每個模型操作自訂事件的名稱和有效載荷。如果 `broadcastAs` 方法返回 `null`，Laravel 將在廣播事件時使用上面討論的模型廣播事件命名慣例：

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

一旦您已在模型中加入了 `BroadcastsEvents` trait 並定義了模型的 `broadcastOn` 方法，您就可以開始在客戶端應用程式中監聽廣播的模型事件了。在開始之前，您可能想查閱有關[監聽事件](#listening-for-events)的完整文件。

首先，使用 `private` 方法來獲取頻道實例，然後呼叫 `listen` 方法來監聽指定的事件。通常，傳遞給 `private` 方法的頻道名稱應與 Laravel 的[模型廣播慣例](#model-broadcasting-conventions)相符。

一旦您獲得了頻道實例，您就可以使用 `listen` 方法來監聽特定事件。由於模型廣播事件與您應用程式的 `App\Events` 目錄中的「實際」事件沒有關聯，因此[事件名稱](#model-broadcasting-event-conventions)必須以 `.` 作為字首，以表明它不屬於特定的命名空間。每個模型廣播事件都有一個 `model` 屬性，其中包含模型的所有可廣播屬性：

```js
Echo.private(`App.Models.User.${this.user.id}`)
    .listen('.UserUpdated', (e) => {
        console.log(e.model);
    });
```

<a name="model-broadcasts-with-react-or-vue"></a>
#### 使用 React 或 Vue

如果您正在使用 React 或 Vue，您可以使用 Laravel Echo 內建的 `useEchoModel` hook 來輕鬆監聽模型廣播：

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

您也可以指定模型事件酬載資料的形狀，以提供更高的型別安全性和編輯便利性：

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
## 客戶端事件

> [!NOTE]
> When using [Pusher Channels](https://pusher.com/channels), you must enable the "Client Events" option in the "App Settings" section of your [application dashboard](https://dashboard.pusher.com/) in order to send client events.

有時您可能希望在完全不經由 Laravel 應用程式的情況下，將事件廣播給其他已連接的客戶端。這對於「正在輸入」通知這類功能特別有用，例如您希望提醒應用程式使用者，有另一位使用者正在某個畫面中輸入訊息。

若要廣播客戶端事件，您可以使用 Echo 的 `whisper` 方法：

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

若要監聽客戶端事件，您可以使用 `listenForWhisper` 方法：

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

藉由將事件廣播與 [通知](/docs/{{version}}/notifications) 結合，您的 JavaScript 應用程式可以在不需重新整理頁面的情況下，即時接收新通知。在開始之前，請務必閱讀有關使用 [廣播通知頻道](/docs/{{version}}/notifications#broadcast-notifications) 的文件。

一旦您已將通知設定為使用廣播頻道，您就可以使用 Echo 的 `notification` 方法來監聽廣播事件。請記住，頻道名稱應與接收通知實體的類別名稱相符：

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

在這個範例中，所有透過 `broadcast` 頻道傳送給 `App\Models\User` 實例的通知，都將由回呼函式接收。您的應用程式 `routes/channels.php` 檔案中包含了 `App.Models.User.{id}` 頻道的頻道授權回呼。