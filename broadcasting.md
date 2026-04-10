# 廣播

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
- [概念概覽](#concept-overview)
    - [使用範例應用程式](#using-example-application)
- [定義廣播事件](#defining-broadcast-events)
    - [廣播名稱](#broadcast-name)
    - [廣播資料](#broadcast-data)
    - [廣播佇列](#broadcast-queue)
    - [廣播條件](#broadcast-conditions)
    - [廣播與資料庫交易](#broadcasting-and-database-transactions)
- [授權通道](#authorizing-channels)
    - [定義授權回呼](#defining-authorization-callbacks)
    - [定義通道類別](#defining-channel-classes)
- [廣播事件](#broadcasting-events)
    - [僅發送給他人](#only-to-others)
    - [自訂連線](#customizing-the-connection)
    - [匿名事件](#anonymous-events)
    - [救援廣播](#rescuing-broadcasts)
- [接收廣播](#receiving-broadcasts)
    - [監聽事件](#listening-for-events)
    - [離開通道](#leaving-a-channel)
    - [命名空間](#namespaces)
    - [使用 React, Vue 或 Svelte](#using-react-or-vue)
- [存在通道](#presence-channels)
    - [授權存在通道](#authorizing-presence-channels)
    - [加入存在通道](#joining-presence-channels)
    - [向存在通道廣播](#broadcasting-to-presence-channels)
- [模型廣播](#model-broadcasting)
    - [模型廣播慣例](#model-broadcasting-conventions)
    - [監聽模型廣播](#listening-for-model-broadcasts)
- [用戶端事件](#client-events)
- [通知](#notifications)

<a name="introduction"></a>
## 簡介

在許多現代網頁應用程式中，WebSockets 被用來實作即時更新的使用者介面。當伺服器上的某些資料被更新時，通常會透過 WebSocket 連線發送一則訊息由用戶端處理。相較於為了將資料變更反映在 UI 上而持續輪詢應用程式伺服器，WebSockets 提供了一個更高效的替代方案。

例如，想像您的應用程式能夠將使用者的資料匯出為 CSV 檔案並透過電子郵件發送給他們。然而，建立這個 CSV 檔案需要幾分鐘的時間，因此您選擇在 [佇列作業](/docs/{{version}}/queues) 中建立並發送該 CSV。當 CSV 建立完成並郵寄給使用者後，我們可以使用事件廣播來發送一個 `App\Events\UserDataExported` 事件，並由應用程式的 JavaScript 接收。一旦接收到該事件，我們就可以向使用者顯示一則訊息，告知其 CSV 已發送至電子郵件，而使用者完全不需要重新整理頁面。

為了協助您建構這類功能，Laravel 讓您能輕鬆地透過 WebSocket 連線來「廣播」伺服器端的 Laravel [events](/docs/{{version}}/events)。廣播您的 Laravel 事件可以讓您在伺服器端的 Laravel 應用程式與用戶端的 JavaScript 應用程式之間共享相同的事件名稱和資料。

廣播的核心概念很簡單：用戶端在前端連接到具名的通道 (channels)，而您的 Laravel 應用程式在後端將事件廣播到這些通道。這些事件可以包含任何您希望提供給前端的額外資料。

<a name="supported-drivers"></a>
#### 支援的驅動程式

預設情況下，Laravel 包含三個伺服器端廣播驅動程式供您選擇：[Laravel Reverb](https://reverb.laravel.com)、[Pusher Channels](https://pusher.com/channels) 以及 [Ably](https://ably.com)。

> [!NOTE]
> 在深入研究事件廣播之前，請確保您已閱讀 Laravel 關於 [events and listeners](/docs/{{version}}/events) 的文件。

<a name="quickstart"></a>
## 快速入門

預設情況下，新的 Laravel 應用程式未啟用廣播功能。您可以使用 `install:broadcasting` Artisan 指令來啟用廣播：

```shell
php artisan install:broadcasting
```

`install:broadcasting` 指令會提示您想要使用哪種事件廣播服務。此外，它會建立 `config/broadcasting.php` 設定檔以及 `routes/channels.php` 檔案，您可以在其中註冊應用程式的廣播授權路由與回呼。

Laravel 開箱即用地支援數種廣播驅動程式：[Laravel Reverb](/docs/{{version}}/reverb)、[Pusher Channels](https://pusher.com/channels)、[Ably](https://ably.com)，以及用於本地開發與除錯的 `log` 驅動程式。此外，還包含一個 `null` 驅動程式，允許您在測試期間禁用廣播。`config/broadcasting.php` 設定檔中為這些驅動程式各提供了一個設定範例。

您應用程式的所有事件廣播設定都儲存在 `config/broadcasting.php` 設定檔中。如果您的應用程式中不存在此檔案，請不用擔心；當您執行 `install:broadcasting` Artisan 指令時，它會被自動建立。

<a name="quickstart-next-steps"></a>
#### 後續步驟

啟用事件廣播後，您就可以進一步學習關於 [定義廣播事件](#defining-broadcast-events) 與 [監聽事件](#listening-for-events) 的內容。如果您使用 Laravel 的 React、Vue 或 Svelte [入門套件](/docs/{{version}}/starter-kits)，您可以使用 Echo 的 [useEcho hook](#using-react-or-vue) 來監聽事件。

> [!NOTE]
> 在廣播任何事件之前，您應該先設定並執行 [佇列工作者](/docs/{{version}}/queues)。所有事件廣播都是透過佇列作業完成的，這樣您的應用程式回應時間才不會受到事件廣播的嚴重影響。

<a name="server-side-installation"></a>
## 伺服器端安裝

要開始使用 Laravel 的事件廣播，我們需要在 Laravel 應用程式中進行一些設定，並安裝幾個套件。

事件廣播是由伺服器端的廣播驅動程式完成的，它會廣播您的 Laravel 事件，以便 Laravel Echo（一個 JavaScript 函式庫）可以在瀏覽器用戶端接收。別擔心 — 我們將逐步引導您完成安裝過程的每個部分。


<a name="reverb"></a>
### Reverb

若要快速啟用 Laravel 廣播功能的支援並使用 Reverb 作為事件廣播者，請執行帶有 `--reverb` 選項的 `install:broadcasting` Artisan 指令。此 Artisan 指令將安裝 Reverb 所需的 Composer 和 NPM 套件，並使用適當的變數更新應用程式的 `.env` 檔案：

```shell
php artisan install:broadcasting --reverb
```


<a name="reverb-manual-installation"></a>
#### 手動安裝

執行 `install:broadcasting` 指令時，系統會提示您安裝 [Laravel Reverb](/docs/{{version}}/reverb)。當然，您也可以使用 Composer 套件管理工具手動安裝 Reverb：

```shell
composer require laravel/reverb
```

套件安裝完成後，您可以執行 Reverb 的安裝指令來發布設定、新增 Reverb 所需的環境變數，並在應用程式中啟用事件廣播：

```shell
php artisan reverb:install
```

您可以在 [Reverb 文件](/docs/{{version}}/reverb) 中找到詳細的 Reverb 安裝與使用說明。


<a name="pusher-channels"></a>
### Pusher Channels

若要快速啟用 Laravel 廣播功能的支援並使用 Pusher 作為事件廣播者，請執行帶有 `--pusher` 選項的 `install:broadcasting` Artisan 指令。此 Artisan 指令會提示您輸入 Pusher 的憑據，安裝 Pusher 的 PHP 與 JavaScript SDK，並使用適當的變數更新應用程式的 `.env` 檔案：

```shell
php artisan install:broadcasting --pusher
```


<a name="pusher-manual-installation"></a>
#### 手動安裝

若要手動安裝 Pusher 支援，您應該使用 Composer 套件管理工具安裝 Pusher Channels PHP SDK：

```shell
composer require pusher/pusher-php-server
```

接下來，您應該在 `config/broadcasting.php` 設定檔案中設定您的 Pusher Channels 憑據。此檔案中已包含一個 Pusher Channels 的設定範例，讓您可以快速指定 key、secret 以及 application ID。通常，您應該在應用程式的 `.env` 檔案中設定 Pusher Channels 的憑據：

```ini
PUSHER_APP_ID="your-pusher-app-id"
PUSHER_APP_KEY="your-pusher-key"
PUSHER_APP_SECRET="your-pusher-secret"
PUSHER_HOST=
PUSHER_PORT=443
PUSHER_SCHEME="https"
PUSHER_APP_CLUSTER="mt1"
```

`config/broadcasting.php` 檔案中的 `pusher` 設定還允許您指定 Channels 支援的額外 `options`，例如 cluster。

然後，在應用程式的 `.env` 檔案中將 `BROADCAST_CONNECTION` 環境變數設定為 `pusher`：

```ini
BROADCAST_CONNECTION=pusher
```

最後，您就可以安裝並設定 [Laravel Echo](#client-side-installation)，它將在用戶端接收廣播事件。


<a name="ably"></a>
### Ably

> [!NOTE]
> 下方的文件將討論如何在「Pusher 相容」模式下使用 Ably。然而，Ably 團隊建議並維護了一個能夠利用 Ably 獨特功能的廣播者和 Echo 用戶端。有關使用 Ably 維護之驅動程式的更多資訊，請[參閱 Ably 的 Laravel 廣播者文件](https://github.com/ably/laravel-broadcaster)。

若要快速啟用 Laravel 廣播功能的支援並使用 [Ably](https://ably.com) 作為事件廣播者，請執行帶有 `--ably` 選項的 `install:broadcasting` Artisan 指令。此 Artisan 指令會提示您輸入 Ably 的憑據，安裝 Ably 的 PHP 與 JavaScript SDK，並使用適當的變數更新應用程式的 `.env` 檔案：

```shell
php artisan install:broadcasting --ably
```

**在繼續之前，您應該在 Ably 應用程式設定中啟用 Pusher 協定支援。您可以在 Ably 應用程式設定控制面板的「Protocol Adapter Settings」部分中啟用此功能。**


<a name="ably-manual-installation"></a>
#### 手動安裝

若要手動安裝 Ably 支援，您應該使用 Composer 套件管理工具安裝 Ably PHP SDK：

```shell
composer require ably/ably-php
```

接下來，您應該在 `config/broadcasting.php` 設定檔案中設定您的 Ably 憑據。此檔案中已包含一個 Ably 的設定範例，讓您可以快速指定 key。通常，此值應該透過 `ABLY_KEY` [環境變數](/docs/{{version}}/configuration#environment-configuration) 來設定：

```ini
ABLY_KEY=your-ably-key
```

然後，在應用程式的 `.env` 檔案中將 `BROADCAST_CONNECTION` 環境變數設定為 `ably`：

```ini
BROADCAST_CONNECTION=ably
```

最後，您就可以安裝並設定 [Laravel Echo](#client-side-installation)，它將在用戶端接收廣播事件。

<a name="client-side-installation"></a>
## 用戶端安裝


<a name="client-reverb"></a>
### Reverb

[Laravel Echo](https://github.com/laravel/echo) 是一個 JavaScript 函式庫，讓您能輕鬆地訂閱通道並監聽由伺服器端廣播驅動程式所發送的事件。

當您透過 `install:broadcasting` Artisan 指令安裝 Laravel Reverb 時，Reverb 與 Echo 的基礎結構與設定將會自動注入到您的應用程式中。然而，如果您希望手動設定 Laravel Echo，可以參考以下說明。


<a name="reverb-client-manual-installation"></a>
#### 手動安裝

若要為應用程式的前端手動設定 Laravel Echo，請先安裝 `pusher-js` 套件，因為 Reverb 使用 Pusher 協定來處理 WebSocket 訂閱、通道與訊息：

```shell
npm install --save-dev laravel-echo pusher-js
```

安裝好 Echo 後，您就可以在應用程式的 JavaScript 中建立一個新的 Echo 實例。建議將其放置在 Laravel 框架內附的 `resources/js/bootstrap.js` 檔案底部：

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

```js tab=Svelte
import { configureEcho } from "@laravel/echo-svelte";

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

接下來，您應該編譯應用程式的靜態資源：

```shell
npm run build
```

> [!WARNING]
> Laravel Echo 的 `reverb` 廣播驅動程式需要 laravel-echo v1.16.0 以上版本。


<a name="client-pusher-channels"></a>
### Pusher Channels

[Laravel Echo](https://github.com/laravel/echo) 是一個 JavaScript 函式庫，讓您能輕鬆地訂閱通道並監聽由伺服器端廣播驅動程式所發送的事件。

當您透過 `install:broadcasting --pusher` Artisan 指令安裝廣播支援時，Pusher 與 Echo 的基礎結構與設定將會自動注入到您的應用程式中。然而，如果您希望手動設定 Laravel Echo，可以參考以下說明。


<a name="pusher-client-manual-installation"></a>
#### 手動安裝

若要為應用程式的前端手動設定 Laravel Echo，請先安裝 `laravel-echo` 與 `pusher-js` 套件，這兩個套件使用 Pusher 協定來處理 WebSocket 訂閱、通道與訊息：

```shell
npm install --save-dev laravel-echo pusher-js
```

安裝好 Echo 後，您就可以在應用程式的 `resources/js/bootstrap.js` 檔案中建立一個新的 Echo 實例：

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

```js tab=Svelte
import { configureEcho } from "@laravel/echo-svelte";

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

接下來，您應該在應用程式的 `.env` 檔案中定義 Pusher 環境變數的適當值。如果這些變數尚未存在於您的 `.env` 檔案中，請將其新增：

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

在根據應用程式需求調整好 Echo 設定後，您即可編譯應用程式的靜態資源：

```shell
npm run build
```

> [!NOTE]
> 若要進一步了解如何編譯應用程式的 JavaScript 靜態資源，請參閱 [Vite](/docs/{{version}}/vite) 文件。


<a name="using-an-existing-client-instance"></a>
#### 使用現有的用戶端實例

如果您已經有一個預先設定好的 Pusher Channels 用戶端實例且希望 Echo 使用它，您可以透過 `client` 設定選項將其傳遞給 Echo：

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
> 以下文件討論如何以「Pusher 相容」模式使用 Ably。然而，Ably 團隊建議並維護了一套能夠利用 Ably 獨特功能的廣播器與 Echo 用戶端。如需更多關於使用 Ably 維護之驅動程式的資訊，請[參閱 Ably 的 Laravel 廣播器文件](https://github.com/ably/laravel-broadcaster)。

[Laravel Echo](https://github.com/laravel/echo) 是一個 JavaScript 函式庫，讓您可以輕鬆地訂閱通道並監聽由伺服器端廣播驅動程式所廣播的事件。

當您透過 `install:broadcasting --ably` Artisan 指令安裝廣播支援時，Ably 與 Echo 的開發支架 (scaffolding) 與配置將會自動注入到您的應用程式中。然而，如果您希望手動配置 Laravel Echo，可以參考以下說明。


<a name="ably-client-manual-installation"></a>
#### 手動安裝

若要為應用程式的前端手動配置 Laravel Echo，請先安裝 `laravel-echo` 與 `pusher-js` 套件，這些套件利用 Pusher 協定來處理 WebSocket 訂閱、通道與訊息：

```shell
npm install --save-dev laravel-echo pusher-js
```

**在繼續之前，您應該在 Ably 應用程式設定中啟用 Pusher 協定支援。您可以在 Ably 應用程式設定儀表板的「Protocol Adapter Settings」部分啟用此功能。**

安裝完 Echo 後，您就可以在應用程式的 `resources/js/bootstrap.js` 檔案中建立一個新的 Echo 實例：

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

```js tab=Svelte
import { configureEcho } from "@laravel/echo-svelte";

configureEcho({
    broadcaster: "ably",
    // key: import.meta.env.VITE_ABLY_PUBLIC_KEY,
    // wsHost: "realtime-pusher.ably.io",
    // wsPort: 443,
    // disableStats: true,
    // encrypted: true,
});
```

您可能注意到我們的 Ably Echo 配置引用了 `VITE_ABLY_PUBLIC_KEY` 環境變數。此變數的值應該是您的 Ably 公鑰。您的公鑰是 Ably 金鑰中 `:` 字元之前的部分。

一旦您根據需求調整好 Echo 配置，即可編譯應用程式的靜態資源：

```shell
npm run dev
```

> [!NOTE]
> 若要了解更多關於編譯應用程式 JavaScript 靜態資源的資訊，請參閱 [Vite](/docs/{{version}}/vite) 的文件。

<a name="concept-overview"></a>
## 概念概覽

Laravel 的事件廣播允許您使用基於驅動程式的方法透過 WebSocket 將伺服器端的 Laravel 事件廣播到用戶端的 JavaScript 應用程式。目前，Laravel 內建提供 [Laravel Reverb](https://reverb.laravel.com)、[Pusher Channels](https://pusher.com/channels) 和 [Ably](https://ably.com) 驅動程式。這些事件可以使用 [Laravel Echo](#client-side-installation) JavaScript 套件在用戶端輕鬆地被接收與處理。

事件透過「通道 (channels)」進行廣播，通道可以指定為公開 (public) 或私有 (private)。任何造訪您應用程式的訪客都可以在不需要任何認證或授權的情況下訂閱公開通道；然而，若要訂閱私有通道，使用者必須經過認證且獲准在該通道上監聽。


<a name="using-example-application"></a>
### 使用範例應用程式

在深入探討事件廣播的每個元件之前，讓我們以一個電子商務商店為例，進行高層次的概覽。

在我們的應用程式中，假設我們有一個頁面允許使用者查看其訂單的運送狀態。同時假設當應用程式處理運送狀態更新時，會觸發一個 `OrderShipmentStatusUpdated` 事件：

```php
use App\Events\OrderShipmentStatusUpdated;

OrderShipmentStatusUpdated::dispatch($order);
```


<a name="the-shouldbroadcast-interface"></a>
#### `ShouldBroadcast` 介面

當使用者在查看其中一筆訂單時，我們不希望他們必須重新整理頁面才能查看狀態更新。相反地，我們希望在更新建立時立即將其廣播到應用程式中。因此，我們需要為 `OrderShipmentStatusUpdated` 事件標記 `ShouldBroadcast` 介面。這將指示 Laravel 在事件被觸發時進行廣播：

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

`ShouldBroadcast` 介面要求我們的事件必須定義一個 `broadcastOn` 方法。此方法負責回傳事件應該廣播的通道。在生成的事件類別中已經定義了此方法的空殼 (stub)，因此我們只需要填入詳細內容。我們只希望訂單的建立者能夠查看狀態更新，因此我們將在與訂單綁定的私有通道上廣播該事件：

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

如果您希望事件在多個通道上廣播，則可以回傳一個 `array`：

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
#### 授權通道

請記住，使用者必須獲得授權才能在私有通道上監聽。我們可以在應用程式的 `routes/channels.php` 檔案中定義通道授權規則。在這個範例中，我們需要驗證任何嘗試在私有 `orders.1` 通道上監聽的使用者是否確實是該訂單的建立者：

```php
use App\Models\Order;
use App\Models\User;

Broadcast::channel('orders.{orderId}', function (User $user, int $orderId) {
    return $user->id === Order::findOrNew($orderId)->user_id;
});
```

`channel` 方法接受兩個引數：通道名稱以及一個回傳 `true` 或 `false` 的回呼函數，用以指示使用者是否獲准在該通道上監聽。

所有授權回呼都會接收目前已認證的使用者作為第一個引數，以及任何額外的萬用字元參數作為後續引數。在這個範例中，我們使用 `{orderId}` 佔位符來表示通道名稱中的「ID」部分是一個萬用字元。


<a name="listening-for-event-broadcasts"></a>
#### 監聽事件廣播

接下來，剩下的就是在我們的 JavaScript 應用程式中監聽該事件。我們可以使用 [Laravel Echo](#client-side-installation) 來達成。Laravel Echo 內建的 React、Vue 和 Svelte hooks 讓入門變得簡單，而且預設情況下，事件的所有公用屬性都將包含在廣播事件中：

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

```svelte tab=Svelte
<script>
import { useEcho } from "@laravel/echo-svelte";

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

若要告知 Laravel 某個特定的事件應該被廣播，您必須在事件類別上實作 `Illuminate\Contracts\Broadcasting\ShouldBroadcast` 介面。這個介面已經被匯入到所有由框架產生的事件類別中，因此您可以輕鬆地將其添加到任何事件中。

`ShouldBroadcast` 介面要求您實作一個方法：`broadcastOn`。`broadcastOn` 方法應該回傳該事件應該廣播的通道或通道陣列。通道應該是 `Channel`、`PrivateChannel` 或 `PresenceChannel` 的實例。`Channel` 實例代表任何使用者都可以訂閱的公開通道，而 `PrivateChannels` 和 `PresenceChannels` 則代表需要 [通道授權](#authorizing-channels) 的私有通道：

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

在實作 `ShouldBroadcast` 介面後，您只需要像平常一樣 [觸發事件](/docs/{{version}}/events) 即可。一旦事件被觸發，一個 [佇列工作](/docs/{{version}}/queues) 將會自動使用您指定的廣播驅動程式來廣播該事件。


<a name="broadcast-name"></a>
### 廣播名稱

預設情況下，Laravel 會使用事件的類別名稱來廣播事件。不過，您可以透過在事件上定義 `broadcastAs` 方法來自訂廣播名稱：

```php
/**
 * The event's broadcast name.
 */
public function broadcastAs(): string
{
    return 'server.created';
}
```

如果您使用 `broadcastAs` 方法自訂廣播名稱，請務必在註冊監聽器時在前面加上 `.` 字元。這將告知 Echo 不要將應用程式的命名空間前綴添加到事件中：

```javascript
.listen('.server.created', function (e) {
    // ...
});
```


<a name="broadcast-data"></a>
### 廣播資料

當事件被廣播時，其所有的 `public` 屬性會自動被序列化並作為事件的有效載荷 (payload) 進行廣播，讓您可以在 JavaScript 應用程式中存取其任何公開資料。例如，如果您的事件有一個包含 Eloquent 模型的單一公開 `$user` 屬性，則事件的廣播有效載荷將會是：

```json
{
    "user": {
        "id": 1,
        "name": "Patrick Stewart"
        ...
    }
}
```

然而，如果您希望對廣播有效載荷有更精細的控制，可以在事件中添加 `broadcastWith` 方法。此方法應該回傳您希望作為事件有效載荷廣播的資料陣列：

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

預設情況下，每個廣播事件都會被放置在 `queue.php` 設定檔中指定的預設佇列連線的預設佇列中。您可以使用事件類別上的 `Connection` 和 `Queue` 屬性來自訂廣播程式所使用的佇列連線和名稱：

```php
use Illuminate\Queue\Attributes\Connection;
use Illuminate\Queue\Attributes\Queue;

#[Connection('redis')]
#[Queue('default')]
class ServerCreated implements ShouldBroadcast
{
    // ...
}
```

或者，您也可以透過在事件上定義 `broadcastQueue` 方法來自訂佇列名稱：

```php
/**
 * The name of the queue on which to place the broadcasting job.
 */
public function broadcastQueue(): string
{
    return 'default';
}
```

如果您希望使用 `sync` 佇列而非預設的佇列驅動程式來廣播事件，您可以實作 `ShouldBroadcastNow` 介面而非 `ShouldBroadcast`：

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

有時候您可能只希望在特定條件為真時才廣播事件。您可以透過在事件類別中添加 `broadcastWhen` 方法來定義這些條件：

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

當廣播事件在資料庫交易中被分發時，它們可能會在資料庫交易提交之前就由佇列處理。當這種情況發生時，您在資料庫交易期間對模型或資料庫紀錄所做的任何更新可能尚未反映在資料庫中。此外，在交易中創建的任何模型或資料庫紀錄可能還不存在於資料庫中。如果您的事件依賴於這些模型，則在處理廣播事件的工作時可能會發生意外錯誤。

如果您的佇列連線 `after_commit` 設定選項被設定為 `false`，您仍然可以在事件類別上實作 `ShouldDispatchAfterCommit` 介面，以表示特定的廣播事件應該在所有開啟的資料庫交易提交後才分發：

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
> 若要了解更多關於解決這些問題的方法，請參閱有關 [佇列工作與資料庫交易](/docs/{{version}}/queues#jobs-and-database-transactions) 的文件。

<a name="authorizing-channels"></a>
## 授權通道

私人通道需要您授權目前已認證的使用者是否可以監聽該通道。這是透過向您的 Laravel 應用程式發送一個包含通道名稱的 HTTP 請求，並由應用程式決定該使用者是否能監聽該通道來實現。當使用 [Laravel Echo](#client-side-installation) 時，授權訂閱私人通道的 HTTP 請求將會自動發出。

當安裝廣播功能時，Laravel 會嘗試自動註冊 `/broadcasting/auth` 路由來處理授權請求。如果 Laravel 無法自動註冊這些路由，您可以在應用程式的 `/bootstrap/app.php` 檔案中手動註冊它們：

```php
->withRouting(
    web: __DIR__.'/../routes/web.php',
    channels: __DIR__.'/../routes/channels.php',
    health: '/up',
)
```


<a name="defining-authorization-callbacks"></a>
### 定義授權回呼

接下來，我們需要定義用來決定目前已認證的使用者是否能監聽特定通道的邏輯。這是在由 `install:broadcasting` Artisan 命令建立的 `routes/channels.php` 檔案中完成的。在此檔案中，您可以使用 `Broadcast::channel` 方法來註冊通道授權回呼：

```php
use App\Models\User;

Broadcast::channel('orders.{orderId}', function (User $user, int $orderId) {
    return $user->id === Order::findOrNew($orderId)->user_id;
});
```

`channel` 方法接受兩個引數：通道名稱以及一個回傳 `true` 或 `false` 以表示使用者是否獲權監聽該通道的回呼函數。

所有授權回呼都會接收目前已認證的使用者作為第一個引數，並將任何額外的萬用字元參數作為後續引數。在此範例中，我們使用 `{orderId}` 佔位符來表示通道名稱中的「ID」部分是萬用字元。

您可以使用 `channel:list` Artisan 命令來查看應用程式的廣播授權回呼列表：

```shell
php artisan channel:list
```


<a name="authorization-callback-model-binding"></a>
#### 授權回呼模型綁定

與 HTTP 路由一樣，通道路由也可以利用隱式和顯式的 [路由模型綁定](/docs/{{version}}/routing#route-model-binding)。例如，您可以請求一個實際的 `Order` 模型實例，而不是接收字串或數字形式的訂單 ID：

```php
use App\Models\Order;
use App\Models\User;

Broadcast::channel('orders.{order}', function (User $user, Order $order) {
    return $user->id === $order->user_id;
});
```

> [!WARNING]
> 與 HTTP 路由模型綁定不同，通道模型綁定不支援自動的 [隱式模型綁定範圍限制](/docs/{{version}}/routing#implicit-model-binding-scoping)。不過，這很少成為問題，因為大多數通道都可以根據單一模型的唯一主鍵來限定範圍。


<a name="authorization-callback-authentication"></a>
#### 授權回呼認證

私人與存在廣播通道透過應用程式的預設認證守衛 (guard) 來認證目前的使用者。如果使用者未經過認證，通道授權將自動被拒絕，且授權回呼永遠不會被執行。不過，如有需要，您可以指定多個自訂守衛來認證傳入的請求：

```php
Broadcast::channel('channel', function () {
    // ...
}, ['guards' => ['web', 'admin']]);
```


<a name="defining-channel-classes"></a>
### 定義通道類別

如果您的應用程式使用了許多不同的通道，您的 `routes/channels.php` 檔案可能會變得臃腫。因此，您可以改用通道類別而非閉包來授權通道。要產生通道類別，請使用 `make:channel` Artisan 命令。此命令將在 `App/Broadcasting` 目錄中建立一個新的通道類別。

```shell
php artisan make:channel OrderChannel
```

接下來，在您的 `routes/channels.php` 檔案中註冊您的通道：

```php
use App\Broadcasting\OrderChannel;

Broadcast::channel('orders.{order}', OrderChannel::class);
```

最後，您可以將通道的授權邏輯放置在通道類別的 `join` 方法中。此 `join` 方法將包含您通常會放置在通道授權閉包中的相同邏輯。您也可以利用通道模型綁定：

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
> 與 Laravel 中的許多其他類別一樣，通道類別將會由 [服務容器](/docs/{{version}}/container) 自動解析。因此，您可以在建構子中對通道所需的任何依賴項進行型別提示 (type-hint)。

<a name="broadcasting-events"></a>
## 廣播事件

一旦您定義了事件並將其標記為 `ShouldBroadcast` 介面，您只需要使用事件的分發方法來觸發該事件。事件分發器會注意到該事件被標記為 `ShouldBroadcast` 介面，並將該事件放入佇列中進行廣播：

```php
use App\Events\OrderShipmentStatusUpdated;

OrderShipmentStatusUpdated::dispatch($order);
```


<a name="only-to-others"></a>
### 僅發送給他人

在建立使用事件廣播的應用程式時，您可能偶爾需要將事件廣播給某個通道的所有訂閱者，但排除目前的使用者。您可以使用 `broadcast` 輔助函式和 `toOthers` 方法來達成此目的：

```php
use App\Events\OrderShipmentStatusUpdated;

broadcast(new OrderShipmentStatusUpdated($update))->toOthers();
```

為了更清楚何時需要使用 `toOthers` 方法，讓我們想像一個工作清單應用程式，使用者可以透過輸入工作名稱來建立新工作。為了建立工作，您的應用程式可能會向 `/task` URL 發出請求，該請求會廣播工作的建立並返回新工作的 JSON 表示形式。當您的 JavaScript 應用程式收到來自端點的響應時，它可能會直接將新工作插入其工作清單中，如下所示：

```js
axios.post('/task', task)
    .then((response) => {
        this.tasks.push(response.data);
    });
```

然而，請記得我們也廣播了工作的建立。如果您的 JavaScript 應用程式也在監聽此事件以將工作添加到工作清單中，您的清單中將出現重複的工作：一個來自端點，另一個來自廣播。您可以使用 `toOthers` 方法來解決這個問題，指示廣播器不要將事件廣播給目前的使用者。

> [!WARNING]
> 您的事件必須使用 `Illuminate\Broadcasting\InteractsWithSockets` trait 才能呼叫 `toOthers` 方法。


<a name="only-to-others-configuration"></a>
#### 配置

當您初始化 Laravel Echo 實例時，連線會被分配一個 socket ID。如果您使用全域的 [Axios](https://github.com/axios/axios) 實例從 JavaScript 應用程式發出 HTTP 請求，socket ID 將自動作為 `X-Socket-ID` 標頭附加到每個發出的請求中。接著，當您呼叫 `toOthers` 方法時，Laravel 會從標頭中提取 socket ID，並指示廣播器不要廣播給任何具有該 socket ID 的連線。

如果您沒有使用全域 Axios 實例，您將需要手動配置您的 JavaScript 應用程式，以便在所有發出的請求中發送 `X-Socket-ID` 標頭。您可以使用 `Echo.socketId` 方法來獲取 socket ID：

```js
var socketId = Echo.socketId();
```


<a name="customizing-the-connection"></a>
### 自訂連線

如果您的應用程式與多個廣播連線互動，且您想要使用預設以外的廣播器來廣播事件，您可以使用 `via` 方法指定將事件推送至哪個連線：

```php
use App\Events\OrderShipmentStatusUpdated;

broadcast(new OrderShipmentStatusUpdated($update))->via('pusher');
```

或者，您可以在事件的建構子中呼叫 `broadcastVia` 方法來指定事件的廣播連線。但在這樣做之前，您應確保該事件類別使用了 `InteractsWithBroadcasting` trait：

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

有時，您可能想要向應用程式的前端廣播一個簡單的事件，而無需建立專用的事件類別。為了實現這一點，`Broadcast` Facade 允許您廣播「匿名事件」：

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

使用 `as` 和 `with` 方法，您可以自訂事件的名稱和資料：

```php
Broadcast::on('orders.'.$order->id)
    ->as('OrderPlaced')
    ->with($order)
    ->send();
```

上述範例將廣播如下的事件：

```json
{
    "event": "OrderPlaced",
    "data": "{ id: 1, total: 100 }",
    "channel": "orders.1"
}
```

如果您想在私有或存在通道上廣播匿名事件，可以使用 `private` 和 `presence` 方法：

```php
Broadcast::private('orders.'.$order->id)->send();
Broadcast::presence('channels.'.$channel->id)->send();
```

使用 `send` 方法廣播匿名事件會將事件分發到應用程式的 [佇列](/docs/{{version}}/queues) 中進行處理。然而，如果您想立即廣播事件，可以使用 `sendNow` 方法：

```php
Broadcast::on('orders.'.$order->id)->sendNow();
```

若要將事件廣播給除目前已認證使用者之外的所有通道訂閱者，您可以呼叫 `toOthers` 方法：

```php
Broadcast::on('orders.'.$order->id)
    ->toOthers()
    ->send();
```


<a name="rescuing-broadcasts"></a>
### 救援廣播

當您的應用程式佇列伺服器不可用，或者 Laravel 在廣播事件時遇到錯誤，會拋出一個異常，這通常會導致終端使用者看到應用程式錯誤。由於事件廣播通常是您應用程式核心功能的補充，您可以在事件上實作 `ShouldRescue` 介面，以防止這些異常干擾使用者體驗。

實作 `ShouldRescue` 介面的事件在嘗試廣播期間會自動使用 Laravel 的 [rescue 輔助函式](/docs/{{version}}/helpers#method-rescue)。此輔助函式會捕捉任何異常，將其回報給應用程式的異常處理程式以記錄日誌，並允許應用程式在不中斷使用者工作流程的情況下正常繼續執行：

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

一旦您[安裝並實例化 Laravel Echo](#client-side-installation) 後，就可以開始監聽從您的 Laravel 應用程式廣播的事件了。首先，使用 `channel` 方法來取得通道的實例，接著呼叫 `listen` 方法來監聽特定的事件：

```js
Echo.channel(`orders.${this.order.id}`)
    .listen('OrderShipmentStatusUpdated', (e) => {
        console.log(e.order.name);
    });
```

如果您想監聽私有通道上的事件，請改用 `private` 方法。您可以繼續鏈結呼叫 `listen` 方法，以便在單一通道上監聽多個事件：

```js
Echo.private(`orders.${this.order.id}`)
    .listen(/* ... */)
    .listen(/* ... */)
    .listen(/* ... */);
```


<a name="stop-listening-for-events"></a>
#### 停止監聽事件

如果您想在不[離開通道](#leaving-a-channel) 的情況下停止監聽特定事件，可以使用 `stopListening` 方法：

```js
Echo.private(`orders.${this.order.id}`)
    .stopListening('OrderShipmentStatusUpdated');
```


<a name="leaving-a-channel"></a>
### 離開通道

若要離開通道，您可以呼叫 Echo 實例上的 `leaveChannel` 方法：

```js
Echo.leaveChannel(`orders.${this.order.id}`);
```

如果您想離開某個通道及其相關的私有和存在通道，可以呼叫 `leave` 方法：

```js
Echo.leave(`orders.${this.order.id}`);
```

<a name="namespaces"></a>
### 命名空間

您可能注意到在上面的範例中，我們沒有為事件類別指定完整的 `App\Events` 命名空間。這是因為 Echo 會自動假設事件位於 `App\Events` 命名空間中。然而，您可以在實例化 Echo 時，透過傳入 `namespace` 設定選項來設定根命名空間：

```js
window.Echo = new Echo({
    broadcaster: 'pusher',
    // ...
    namespace: 'App.Other.Namespace'
});
```

或者，您在使用 Echo 訂閱事件時，可以在事件類別前加上 `.` 前綴。這讓您能夠始終指定完整的類別名稱：

```js
Echo.channel('orders')
    .listen('.Namespace\\Event\\Class', (e) => {
        // ...
    });
```

<a name="using-react-or-vue"></a>
### 使用 React, Vue 或 Svelte

Laravel Echo 包含了 React, Vue 和 Svelte 的 hooks，讓監聽事件變得非常簡單。要開始使用，請呼叫 `useEcho` hook，它用於監聽私有事件。`useEcho` hook 會在消耗該 hook 的元件被卸載 (unmounted) 時自動離開通道：

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

```svelte tab=Svelte
<script>
import { useEcho } from "@laravel/echo-svelte";

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

您也可以指定廣播事件酬載 (payload) 資料的結構，以提供更高的型別安全性與編輯便利性：

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

`useEcho` hook 會在消耗該 hook 的元件被卸載時自動離開通道；然而，在必要時，您可以使用回傳的函式來以程式方式手動停止或開始監聽通道：

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

```svelte tab=Svelte
<script>
import { useEcho } from "@laravel/echo-svelte";

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
#### 連接至公開通道

要連接至公開通道，您可以使用 `useEchoPublic` hook：

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

```svelte tab=Svelte
<script>
import { useEchoPublic } from "@laravel/echo-svelte";

useEchoPublic("posts", "PostPublished", (e) => {
    console.log(e.post);
});
</script>
```


<a name="react-vue-connecting-to-presence-channels"></a>
#### 連接至存在通道

要連接至存在通道，您可以使用 `useEchoPresence` hook：

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

```svelte tab=Svelte
<script>
import { useEchoPresence } from "@laravel/echo-svelte";

useEchoPresence("posts", "PostPublished", (e) => {
    console.log(e.post);
});
</script>
```


<a name="react-vue-connection-status"></a>
#### 連線狀態

您可以使用 `useConnectionStatus` hook 取得目前的 WebSocket 連線狀態，它提供了會隨著連線狀態改變而自動更新的反應式 (reactive) 狀態：

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

```svelte tab=Svelte
<script>
import { useConnectionStatus } from "@laravel/echo-svelte";

const status = useConnectionStatus();
</script>

<div>Connection: {status()}</div>
```

可能的狀態值包括：

<div class="content-list" markdown="1">

- `connected` - 已成功連接至 WebSocket 伺服器。
- `connecting` - 初始連線嘗試中。
- `reconnecting` - 中斷連線後嘗試重新連線中。
- `disconnected` - 未連接且未嘗試重新連線。
- `failed` - 連線失敗且不會重試。

</div>

<a name="presence-channels"></a>
## 存在通道

存在通道基於私有通道的安全性，同時提供了額外的功能，讓你能感知誰訂閱了該通道。這讓開發強大的協作應用功能變得非常簡單，例如在另一個使用者瀏覽同一頁面時通知使用者，或是列出聊天室中的成員。


<a name="authorizing-presence-channels"></a>
### 授權存在通道

所有存在通道同時也是私有通道；因此，使用者必須被[授權才能存取它們](#authorizing-channels)。然而，在定義存在通道的授權回呼時，如果使用者獲准加入通道，你不需要回傳 `true`，而是應該回傳一個關於該使用者的資料陣列。

授權回呼回傳的資料將提供給 JavaScript 應用程式中存在通道的事件監聽器使用。如果使用者未獲准加入存在通道，你應該回傳 `false` 或 `null`：

```php
use App\Models\User;

Broadcast::channel('chat.{roomId}', function (User $user, int $roomId) {
    if ($user->canJoinRoom($roomId)) {
        return ['id' => $user->id, 'name' => $user->name];
    }
});
```


<a name="joining-presence-channels"></a>
### 加入存在通道

要加入存在通道，你可以使用 Echo 的 `join` 方法。`join` 方法會回傳一個 `PresenceChannel` 實作，除了提供 `listen` 方法外，還允許你訂閱 `here`、`joining` 和 `leaving` 事件。

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

`here` 回呼會在成功加入通道後立即執行，並接收一個陣列，其中包含目前訂閱該通道的所有其他使用者的資訊。`joining` 方法會在有新使用者加入通道時執行，而 `leaving` 方法則會在使用者離開通道時執行。`error` 方法會在認證端點回傳非 200 的 HTTP 狀態碼，或者在解析回傳的 JSON 時發生問題時執行。


<a name="broadcasting-to-presence-channels"></a>
### 向存在通道廣播

存在通道可以像公共或私有通道一樣接收事件。以聊天室為例，我們可能想向房間的存在通道廣播 `NewMessage` 事件。若要實現此功能，我們將在事件的 `broadcastOn` 方法中回傳 `PresenceChannel` 實例：

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

與其他事件一樣，你可以使用 `broadcast` 輔助函式和 `toOthers` 方法來排除目前使用者接收此廣播：

```php
broadcast(new NewMessage($message));

broadcast(new NewMessage($message))->toOthers();
```

如同其他類型的事件，你可以使用 Echo 的 `listen` 方法來監聽發送到存在通道的事件：

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
> 在閱讀以下關於模型廣播的說明之前，我們建議您先熟悉 Laravel 模型廣播服務的一般概念，以及如何手動建立和監聽廣播事件。

在應用程式的 [Eloquent 模型](/docs/{{version}}/eloquent) 被建立、更新或刪除時進行事件廣播是非常常見的。當然，這可以透過手動 [為 Eloquent 模型狀態變更定義自訂事件](/docs/{{version}}/eloquent#events) 並將這些事件標記為 `ShouldBroadcast` 介面來輕鬆達成。

然而，如果您在應用程式中沒有將這些事件用於其他用途，單純為了廣播而建立事件類別可能會很繁瑣。為了改善這個問題，Laravel 允許您指定 Eloquent 模型應自動廣播其狀態變更。

要開始使用，您的 Eloquent 模型應使用 `Illuminate\Database\Eloquent\BroadcastsEvents` trait。此外，模型應定義一個 `broadcastOn` 方法，該方法將回傳模型事件應廣播的通道陣列：

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

一旦您的模型包含了這個 trait 並定義了其廣播通道，當模型實例被建立、更新、刪除、移至回收桶 (trashed) 或還原 (restored) 時，它將開始自動廣播事件。

此外，您可能已經注意到 `broadcastOn` 方法接收一個字串 `$event` 引數。這個引數包含模型上發生的事件類型，其值將為 `created`、`updated`、`deleted`、`trashed` 或 `restored`。透過檢查這個變數的值，您可以決定模型針對特定事件應廣播到哪些通道（如果有）：

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
#### 自訂模型廣播事件的建立

有時候，您可能希望自訂 Laravel 建立底層模型廣播事件的方式。您可以透過在 Eloquent 模型上定義 `newBroadcastableEvent` 方法來達成。此方法應回傳一個 `Illuminate\Database\Eloquent\BroadcastableModelEventOccurred` 實例：

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
#### 通道慣例

您可能已經注意到，上述模型範例中的 `broadcastOn` 方法並沒有回傳 `Channel` 實例，而是直接回傳 Eloquent 模型。如果您的模型 `broadcastOn` 方法回傳了一個 Eloquent 模型實例（或包含在回傳的陣列中），Laravel 將自動使用該模型的類別名稱和主鍵識別碼作為通道名稱，為該模型實例化一個私有通道實例。

因此，一個 `id` 為 `1` 的 `App\Models\User` 模型將被轉換為一個名稱為 `App.Models.User.1` 的 `Illuminate\Broadcasting\PrivateChannel` 實例。當然，除了從模型的 `broadcastOn` 方法回傳 Eloquent 模型實例之外，您也可以回傳完整的 `Channel` 實例，以便完全控制模型的通道名稱：

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

如果您計畫從模型的 `broadcastOn` 方法明確回傳通道實例，您可以將 Eloquent 模型實例傳遞給通道的建構子。這樣做時，Laravel 將使用上述的模型通道慣例將 Eloquent 模型轉換為通道名稱字串：

```php
return [new Channel($this->user)];
```

如果您需要確定模型的通道名稱，可以在任何模型實例上呼叫 `broadcastChannel` 方法。例如，對於一個 `id` 為 `1` 的 `App\Models\User` 模型，此方法將回傳字串 `App.Models.User.1`：

```php
$user->broadcastChannel();
```


<a name="model-broadcasting-event-conventions"></a>
#### 事件慣例

由於模型廣播事件不與應用程式 `App\Events` 目錄中的「實際」事件相關聯，因此它們會根據慣例被分配名稱和酬載 (payload)。Laravel 的慣例是使用模型的類別名稱（不包含命名空間）以及觸發廣播的模型事件名稱來廣播該事件。

因此，例如，對 `App\Models\Post` 模型的更新將向您的用戶端應用程式廣播一個名為 `PostUpdated` 的事件，其酬載如下：

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

刪除 `App\Models\User` 模型則會廣播一個名為 `UserDeleted` 的事件。

如果您願意，可以透過在模型中加入 `broadcastAs` 和 `broadcastWith` 方法來定義自訂的廣播名稱和酬載。這些方法會接收目前正在發生的模型事件/操作名稱，讓您可以針對每個模型操作自訂事件名稱和酬載。如果 `broadcastAs` 方法回傳 `null`，Laravel 在廣播事件時將使用上述的模型廣播事件名稱慣例：

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

一旦您將 `BroadcastsEvents` trait 新增至您的模型並定義了模型的 `broadcastOn` 方法，您就可以開始在您的用戶端應用程式中監聽廣播的模型事件。在開始之前，您可能想要參考關於 [監聽事件](#listening-for-events) 的完整文件。

首先，使用 `private` 方法來取得通道的實例，然後呼叫 `listen` 方法來監聽指定的事件。通常，傳遞給 `private` 方法的通道名稱應符合 Laravel 的 [模型廣播慣例](#model-broadcasting-conventions)。

一旦您取得了通道實例，您可以使用 `listen` 方法來監聽特定事件。由於模型廣播事件並未與應用程式 `App\Events` 目錄中的「實際」事件相關聯，因此 [事件名稱](#model-broadcasting-event-conventions) 必須加上 `.` 前綴，以表示它不屬於特定的命名空間。每個模型廣播事件都有一個 `model` 屬性，其中包含模型所有可廣播的屬性：

```js
Echo.private(`App.Models.User.${this.user.id}`)
    .listen('.UserUpdated', (e) => {
        console.log(e.model);
    });
```


<a name="model-broadcasts-with-react-or-vue"></a>
#### 使用 React, Vue 或 Svelte

如果您使用的是 React、Vue 或 Svelte，您可以使用 Laravel Echo 內建的 `useEchoModel` hook 來輕鬆監聽模型廣播：

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

```svelte tab=Svelte
<script>
import { useEchoModel } from "@laravel/echo-svelte";

useEchoModel("App.Models.User", userId, ["UserUpdated"], (e) => {
    console.log(e.model);
});
</script>
```

您也可以指定模型事件有效負載 (payload) 資料的形狀，以提供更高的型別安全性與編輯便利性：

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
> 當使用 [Pusher Channels](https://pusher.com/channels) 時，您必須在 [應用程式控制面板](https://dashboard.pusher.com/) 的 "App Settings" 區塊中啟用 "Client Events" 選項，才能發送用戶端事件。

有時候您可能希望在完全不經過 Laravel 應用程式的情況下，將事件廣播給其他已連線的用戶端。這在實作像是「正在輸入」的通知時非常有用，讓您可以提醒應用程式的使用者，有另一位使用者正在該畫面輸入訊息。

若要廣播用戶端事件，您可以使用 Echo 的 `whisper` 方法：

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

```svelte tab=Svelte
<script>
import { useEcho } from "@laravel/echo-svelte";

const { channel } = useEcho(`chat.${roomId}`, ['update'], (e) => {
    console.log('Chat event received:', e);
});

channel().whisper('typing', { name: user.name });
</script>
```

若要監聽用戶端事件，您可以使用 `listenForWhisper` 方法：

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

```svelte tab=Svelte
<script>
import { useEcho } from "@laravel/echo-svelte";

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

透過將事件廣播與 [通知](/docs/{{version}}/notifications) 結合，您的 JavaScript 應用程式可以在通知發生時立即接收，而無需重新整理頁面。在開始之前，請務必閱讀關於使用 [廣播通知通道](/docs/{{version}}/notifications#broadcast-notifications) 的文件。

一旦您設定好通知使用廣播通道後，就可以使用 Echo 的 `notification` 方法來監聽廣播事件。請記得，通道名稱應與接收通知的實體類別名稱一致：

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

```svelte tab=Svelte
<script>
import { useEchoModel } from "@laravel/echo-svelte";

const { channel } = useEchoModel('App.Models.User', userId);

channel().notification((notification) => {
    console.log(notification.type);
});
</script>
```

在此範例中，所有透過 `broadcast` 通道發送到 `App\Models\User` 實例的通知都將由回呼函式接收。您的應用程式 `routes/channels.php` 檔案中已包含 `App.Models.User.{id}` 通道的通道授權回呼。


<a name="stop-listening-for-notifications"></a>
#### 停止監聽通知

如果您想在不 [離開通道](#leaving-a-channel) 的情況下停止監聽通知，可以使用 `stopListeningForNotification` 方法：

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