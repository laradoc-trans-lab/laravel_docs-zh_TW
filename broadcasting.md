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
- [概念總覽](#concept-overview)
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
    - [僅廣播給其他人](#only-to-others)
    - [自訂連線](#customizing-the-connection)
    - [匿名事件](#anonymous-events)
    - [救援廣播](#rescuing-broadcasts)
- [接收廣播](#receiving-broadcasts)
    - [監聽事件](#listening-for-events)
    - [離開頻道](#leaving-a-channel)
    - [命名空間](#namespaces)
    - [使用 React、Vue 或 Svelte](#using-react-or-vue)
- [狀態頻道 (Presence Channels)](#presence-channels)
    - [授權狀態頻道](#authorizing-presence-channels)
    - [加入狀態頻道](#joining-presence-channels)
    - [廣播至狀態頻道](#broadcasting-to-presence-channels)
- [模型廣播 (Model Broadcasting)](#model-broadcasting)
    - [模型廣播慣例](#model-broadcasting-conventions)
    - [監聽模型廣播](#listening-for-model-broadcasts)
- [用戶端事件](#client-events)
- [通知](#notifications)

<a name="introduction"></a>
## 簡介

在許多現代網頁應用程式中，WebSockets 被用來實現即時、動態更新的使用者介面。當伺服器端更新了某些資料，通常會透過 WebSocket 連線發送一條訊息，並由用戶端進行處理。相較於持續向應用程式伺服器輪詢（Polling）應反映在 UI 上的資料變更，WebSockets 提供了一個更有效率的替代方案。

例如，假設您的應用程式可以將使用者資料匯出為 CSV 檔案並透過電子郵件寄送給他們。然而，建立這個 CSV 檔案需要花費幾分鐘的時間，因此您選擇在 [佇列工作](/docs/{{version}}/queues) 中建立並寄送該 CSV。當 CSV 建立完成並郵寄給使用者後，我們可以使用事件廣播來分派一個 `App\Events\UserDataExported` 事件，並由我們應用程式的 JavaScript 接收。一旦接收到該事件，我們就可以向使用者顯示一條訊息，告知他們 CSV 已經寄到他們的電子郵件信箱，而不需要他們重新整理網頁。

為了協助您構建此類功能，Laravel 讓您可以輕鬆地透過 WebSocket 連線「廣播」伺服器端的 Laravel [事件](/docs/{{version}}/events)。廣播您的 Laravel 事件可以讓您在伺服器端 Laravel 應用程式與用戶端 JavaScript 應用程式之間共享相同的事件名稱和資料。

廣播背後的核心概念非常簡單：用戶端在前端連接到具名的頻道，而您的 Laravel 應用程式則在後端向這些頻道廣播事件。這些事件可以包含您希望提供給前端的任何額外資料。

<a name="supported-drivers"></a>
#### 支援的驅動程式

預設情況下，Laravel 提供了三個伺服器端廣播驅動程式供您選擇：[Laravel Reverb](https://reverb.laravel.com)、[Pusher Channels](https://pusher.com/channels) 和 [Ably](https://ably.com)。

> [!NOTE]
> 在深入探討事件廣播之前，請確保您已閱讀 Laravel 關於 [事件與監聽器](/docs/{{version}}/events) 的文件。

<a name="quickstart"></a>
## 快速入門

預設情況下，新的 Laravel 應用程式中並未啟用廣播。您可以使用 `install:broadcasting` Artisan 指令來啟用廣播：

```shell
php artisan install:broadcasting
```

`install:broadcasting` 指令會詢問您想要使用哪種事件廣播服務。此外，它還會建立 `config/broadcasting.php` 設定檔和 `routes/channels.php` 檔案，您可以在其中註冊應用程式的廣播授權路由和回呼（Callbacks）。

Laravel 開箱即支援多種廣播驅動程式：[Laravel Reverb](/docs/{{version}}/reverb)、[Pusher Channels](https://pusher.com/channels)、[Ably](https://ably.com)，以及一個用於本地端開發和偵錯的 `log` 驅動程式。此外，還包含一個 `null` 驅動程式，讓您可以在測試期間停用廣播。在 `config/broadcasting.php` 設定檔中包含了這些驅動程式的設定範例。

您應用程式的所有事件廣播設定都儲存在 `config/broadcasting.php` 設定檔中。如果您的應用程式中目前沒有此檔案，請不要擔心；當您執行 `install:broadcasting` Artisan 指令時，它就會被建立。

<a name="quickstart-next-steps"></a>
#### 後續步驟

啟用事件廣播後，您就可以準備了解更多關於 [定義廣播事件](#defining-broadcast-events) 和 [監聽事件](#listening-for-events) 的內容。如果您使用的是 Laravel 的 React、Vue 或 Svelte [入門套件](/docs/{{version}}/starter-kits)，您可以使用 Echo 的 [useEcho hook](#using-react-or-vue) 來監聽事件。

> [!NOTE]
> 在廣播任何事件之前，您應該先設定並執行一個 [佇列工作者 (Queue Worker)](/docs/{{version}}/queues)。所有的事件廣播都是透過佇列工作（Queued jobs）來完成的，這樣您應用程式的回應時間就不會因為正在廣播的事件而受到嚴重影響。

<a name="server-side-installation"></a>
## 伺服器端安裝

要開始使用 Laravel 的事件廣播，我們需要在 Laravel 應用程式中進行一些設定，並安裝幾個套件。

事件廣播是透過伺服器端的廣播驅動程式來完成的，它會廣播您的 Laravel 事件，以便 Laravel Echo（一個 JavaScript 函式庫）可以在瀏覽器用戶端接收它們。不用擔心——我們將逐步引導您完成每個安裝步驟。

<a name="reverb"></a>
### Reverb

要在使用 Reverb 作為事件廣播器時快速啟用 Laravel 廣播功能的支援，請執行帶有 `--reverb` 選項的 `install:broadcasting` Artisan 指令。此 Artisan 指令將安裝 Reverb 所需的 Composer 和 NPM 套件，並使用適當的變數更新您的應用程式 `.env` 檔案：

```shell
php artisan install:broadcasting --reverb
```

<a name="reverb-manual-installation"></a>
#### 手動安裝

執行 `install:broadcasting` 指令時，系統會提示您安裝 [Laravel Reverb](/docs/{{version}}/reverb)。當然，您也可以使用 Composer 套件管理工具手動安裝 Reverb：

```shell
composer require laravel/reverb
```

安裝套件後，您可以執行 Reverb 的安裝指令來發布設定檔、新增 Reverb 所需的環境變數，並在應用程式中啟用事件廣播：

```shell
php artisan reverb:install
```

您可以在 [Reverb 說明文件](/docs/{{version}}/reverb)中找到詳細的 Reverb 安裝與使用說明。

<a name="pusher-channels"></a>
### Pusher Channels

要在使用 Pusher 作為事件廣播器時快速啟用 Laravel 廣播功能的支援，請執行帶有 `--pusher` 選項的 `install:broadcasting` Artisan 指令。此 Artisan 指令會提示您輸入 Pusher 憑證，安裝 Pusher PHP 與 JavaScript SDK，並使用適當的變數更新您的應用程式 `.env` 檔案：

```shell
php artisan install:broadcasting --pusher
```

<a name="pusher-manual-installation"></a>
#### 手動安裝

要手動安裝 Pusher 支援，您應該使用 Composer 套件管理工具安裝 Pusher Channels PHP SDK：

```shell
composer require pusher/pusher-php-server
```

接下來，您應該在 `config/broadcasting.php` 設定檔中設定您的 Pusher Channels 憑證。此檔案中已經包含了一個 Pusher Channels 設定範例，讓您可以快速指定您的 key、secret 和應用程式 ID。通常，您應該在應用程式的 `.env` 檔案中設定您的 Pusher Channels 憑證：

```ini
PUSHER_APP_ID="your-pusher-app-id"
PUSHER_APP_KEY="your-pusher-key"
PUSHER_APP_SECRET="your-pusher-secret"
PUSHER_HOST=
PUSHER_PORT=443
PUSHER_SCHEME="https"
PUSHER_APP_CLUSTER="mt1"
```

`config/broadcasting.php` 檔案的 `pusher` 設定也允許您指定 Channels 支援的其他 `options`（選項），例如叢集 (cluster)。

然後，在應用程式的 `.env` 檔案中將 `BROADCAST_CONNECTION` 環境變數設定為 `pusher`：

```ini
BROADCAST_CONNECTION=pusher
```

最後，您就可以準備安裝與設定 [Laravel Echo](#client-side-installation)，它將在用戶端接收廣播事件。

<a name="ably"></a>
### Ably

> [!NOTE]
> 下面的說明文件討論了如何以「Pusher 相容」模式使用 Ably。不過，Ably 團隊建議並維護了一個能夠利用 Ably 所提供之獨特功能的廣播器與 Echo 用戶端。有關使用 Ably 維護之驅動程式的更多資訊，請[參閱 Ably 的 Laravel 廣播器說明文件](https://github.com/ably/laravel-broadcaster)。

要在使用 [Ably](https://ably.com) 作為事件廣播器時快速啟用 Laravel 廣播功能的支援，請執行帶有 `--ably` 選項的 `install:broadcasting` Artisan 指令。此 Artisan 指令會提示您輸入 Ably 憑證，安裝 Ably PHP 與 JavaScript SDK，並使用適當的變數更新您的應用程式 `.env` 檔案：

```shell
php artisan install:broadcasting --ably
```

**在繼續之前，您應該在 Ably 應用程式設定中啟用 Pusher 協定支援。您可以在 Ably 應用程式設定儀表板的「Protocol Adapter Settings」部分啟用此功能。**

<a name="ably-manual-installation"></a>
#### 手動安裝

要手動安裝 Ably 支援，您應該使用 Composer 套件管理工具安裝 Ably PHP SDK：

```shell
composer require ably/ably-php
```

接下來，您 should 在 `config/broadcasting.php` 設定檔中設定您的 Ably 憑證。此檔案中已經包含了一個 Ably 設定範例，讓您可以快速指定您的 key。通常，此值應該透過 `ABLY_KEY` [環境變數](/docs/{{version}}/configuration#environment-configuration)來設定：

```ini
ABLY_KEY=your-ably-key
```

然後，在應用程式的 `.env` 檔案中將 `BROADCAST_CONNECTION` 環境變數設定為 `ably`：

```ini
BROADCAST_CONNECTION=ably
```

最後，您就可以準備安裝與設定 [Laravel Echo](#client-side-installation)，它將在用戶端接收廣播事件。

<a name="client-side-installation"></a>
## 用戶端安裝


<a name="client-reverb"></a>
### Reverb

[Laravel Echo](https://github.com/laravel/echo) 是一個 JavaScript 函式庫，它能讓您輕鬆訂閱頻道並監聽由伺服器端廣播驅動器所廣播的事件。

當透過 `install:broadcasting` Artisan 指令安裝 Laravel Reverb 時，Reverb 與 Echo 的鷹架 (Scaffolding) 和設定將會自動注入到您的應用程式中。然而，如果您希望手動設定 Laravel Echo，可以按照下方的說明進行。


<a name="reverb-client-manual-installation"></a>
#### 手動安裝

若要為應用程式的前端手動設定 Laravel Echo，首先請安裝 `pusher-js` 套件，因為 Reverb 使用 Pusher 協定來進行 WebSocket 訂閱、頻道和訊息傳遞：

```shell
npm install --save-dev laravel-echo pusher-js
```

安裝 Echo 後，您就可以在應用程式的 JavaScript 中建立一個全新的 Echo 執行個體。最適合執行此操作的地方是 Laravel 框架隨附的 `resources/js/app.js` 檔案底部：

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

接下來，您應該編譯您的應用程式靜態資源：

```shell
npm run build
```

> [!WARNING]
> Laravel Echo `reverb` 廣播器需要 laravel-echo v1.16.0+。


<a name="client-pusher-channels"></a>
### Pusher Channels

[Laravel Echo](https://github.com/laravel/echo) 是一個 JavaScript 函式庫，它能讓您輕鬆訂閱頻道並監聽由伺服器端廣播驅動器所廣播的事件。

當透過 `install:broadcasting --pusher` Artisan 指令安裝廣播支援時，Pusher 與 Echo 的鷹架和設定將會自動注入到您的應用程式中。然而，如果您希望手動設定 Laravel Echo，可以按照下方的說明進行。


<a name="pusher-client-manual-installation"></a>
#### 手動安裝

若要為應用程式的前端手動設定 Laravel Echo，首先請安裝 `laravel-echo` 與 `pusher-js` 套件，這兩個套件使用 Pusher 協定來進行 WebSocket 訂閱、頻道和訊息傳遞：

```shell
npm install --save-dev laravel-echo pusher-js
```

安裝 Echo 後，您就可以在應用程式的 `resources/js/app.js` 檔案中建立一個全新的 Echo 執行個體：

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

接下來，您應該在應用程式的 `.env` 檔案中定義對應的 Pusher 環境變數。如果這些變數尚未存在於您的 `.env` 檔案中，請新增它們：

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

根據您的應用程式需求調整完 Echo 設定後，您便可以編譯應用程式的靜態資源：

```shell
npm run build
```

> [!NOTE]
> 若要深入瞭解如何編譯您的應用程式 JavaScript 靜態資源，請參閱 [Vite](/docs/{{version}}/vite) 的說明文件。


<a name="using-an-existing-client-instance"></a>
#### 使用現有的用戶端執行個體

如果您已經有一個預先設定好的 Pusher Channels 用戶端執行個體，並且希望 Echo 使用它，您可以透過 `client` 設定選項將其傳遞給 Echo：

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
> The documentation below discusses how to use Ably in "Pusher compatibility" mode. However, the Ably team recommends and maintains a broadcaster and Echo client that is able to take advantage of the unique capabilities offered by Ably. For more information on using the Ably maintained drivers, please [consult Ably's Laravel broadcaster documentation](https://github.com/ably/laravel-broadcaster).

[Laravel Echo](https://github.com/laravel/echo) 是一個 JavaScript 函式庫，它能讓您輕鬆訂閱頻道並監聽由伺服器端廣播驅動程式所廣播的事件。

透過 `install:broadcasting --ably` Artisan 指令安裝廣播支援時，Ably 和 Echo 的基礎結構與設定將會自動注入到您的應用程式中。然而，如果您希望手動設定 Laravel Echo，可以按照以下說明進行。


<a name="ably-client-manual-installation"></a>
#### 手動安裝

若要為應用程式的前端手動設定 Laravel Echo，首先請安裝 `laravel-echo` 和 `pusher-js` 套件，這些套件利用 Pusher 協定進行 WebSocket 訂閱、頻道和訊息傳遞：

```shell
npm install --save-dev laravel-echo pusher-js
```

**在繼續之前，您應該在 Ably 應用程式設定中啟用 Pusher 協定支援。您可以在 Ably 應用程式設定主控面板的「Protocol Adapter Settings」部分啟用此功能。**

安裝 Echo 後，您就可以準備在應用程式的 `resources/js/app.js` 檔案中建立一個全新的 Echo 執行個體：

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

您可能已經注意到，我們的 Ably Echo 設定引用了一個 `VITE_ABLY_PUBLIC_KEY` 環境變數。這個變數的值應該是您的 Ably 公開金鑰。您的公開金鑰是 Ably 金鑰中出現在 `:` 字元之前的部分。

根據您的需求調整 Echo 設定後，即可編譯應用程式的靜態資源：

```shell
npm run dev
```

> [!NOTE]
> 若要深入了解如何編譯應用程式的 JavaScript 靜態資源，請參閱 [Vite](/docs/{{version}}/vite) 的說明文件。

<a name="concept-overview"></a>
## 概念總覽

Laravel 的事件廣播功能讓你可以使用基於驅動程式的 WebSockets 技術，將伺服器端的 Laravel 事件廣播到用戶端的 JavaScript 應用程式中。目前，Laravel 內建提供了 [Laravel Reverb](https://reverb.laravel.com)、[Pusher Channels](https://pusher.com/channels) 和 [Ably](https://ably.com) 驅動程式。在用戶端，你可以使用 [Laravel Echo](#client-side-installation) JavaScript 套件輕鬆接收並處理這些事件。

事件是透過「頻道 (Channels)」來廣播的，頻道可以被指定為公開（Public）或私有（Private）。任何造訪你應用程式的訪客都可以在沒有任何認證與授權的情況下訂閱公開頻道；然而，若要訂閱私有頻道，使用者必須先通過認證，且獲得在該頻道上監聽的授權。


<a name="using-example-application"></a>
### 使用範例應用程式

在深入探討事件廣播的各個元件之前，讓我們以電子商務商店為例，來進行一次高階的總覽。

在我們的應用程式中，假設有一個頁面可以讓使用者查看其訂單的運送狀態。同時也假設當應用程式處理運送狀態更新時，會觸發一個 `OrderShipmentStatusUpdated` 事件：

```php
use App\Events\OrderShipmentStatusUpdated;

OrderShipmentStatusUpdated::dispatch($order);
```


<a name="the-shouldbroadcast-interface"></a>
#### `ShouldBroadcast` 介面

當使用者正在查看他們的其中一張訂單時，我們不希望他們必須重新整理網頁才能看到狀態更新。相反地，我們希望在更新產生時，能即時將其廣播到應用程式中。因此，我們需要讓 `OrderShipmentStatusUpdated` 事件實作 `ShouldBroadcast` 介面。這將會指示 Laravel 在該事件被觸發時進行廣播：

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

`ShouldBroadcast` 介面要求我們的事件必須定義一個 `broadcastOn` 方法。此方法負責返回事件應該廣播在哪些頻道上。在產生的事件類別中，已經預先定義了此方法的空 stub，因此我們只需要填入其實作細節。由於我們只希望訂單的建立者能夠看到狀態更新，所以我們會將事件廣播在與該訂單綁定的私有頻道上：

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

如果你希望事件廣播在多個頻道上，你可以改為返回一個 `array`：

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

請記住，使用者必須獲得授權才能監聽私有頻道。我們可以在應用程式的 `routes/channels.php` 檔案中定義頻道的授權規則。在這個範例中，我們需要驗證任何嘗試監聽私有 `orders.1` 頻道的使用者是否確實是該訂單的建立者：

```php
use App\Models\Order;
use App\Models\User;

Broadcast::channel('orders.{orderId}', function (User $user, int $orderId) {
    return $user->id === Order::findOrNew($orderId)->user_id;
});
```

`channel` 方法接受兩個引數：頻道的名稱，以及一個返回 `true` 或 `false` 的回呼（Callback），用以指出使用者是否有權限監聽該頻道。

所有授權回呼都會接收目前已認證的使用者作為其第一個引數，並將任何額外的萬用字元參數作為其後的引數。在此範例中，我們使用 `{orderId}` 預留位置來表示頻道名稱中的「ID」部分是一個萬用字元。


<a name="listening-for-event-broadcasts"></a>
#### 監聽事件廣播

接下來，剩下的工作就只有在我們的 JavaScript 應用程式中監聽該事件。我們可以使用 [Laravel Echo](#client-side-installation) 來達成這一點。Laravel Echo 內建的 React、Vue 和 Svelte hook 讓你可以輕鬆上手，而且在預設情況下，該事件的所有公開屬性都會包含在廣播事件中：

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

<!-- SECTION_TO_TRANSLATE_START -->
<a name="defining-broadcast-events"></a>
## 定義廣播事件

若要通知 Laravel 某個指定的事件應該被廣播，您必須在該事件類別上實作 `Illuminate\Contracts\Broadcasting\ShouldBroadcast` 契約(Contracts)。框架所產生的所有事件類別都已經匯入了這個契約，因此您可以輕鬆地將其加入到您的任何事件中。

`ShouldBroadcast` 契約要求您實作一個方法：`broadcastOn`。`broadcastOn` 方法應傳回該事件要廣播的一個頻道或頻道陣列。這些頻道必須是 `Channel`、`PrivateChannel` 或 `PresenceChannel` 的實例。`Channel` 的實例代表任何使用者都可以訂閱的公開頻道，而 `PrivateChannels` 與 `PresenceChannels` 則代表需要[頻道授權](#authorizing-channels)的私密頻道：

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

實作 `ShouldBroadcast` 契約後，您只需要像往常一樣[觸發事件](/docs/{{version}}/events)。一旦事件被觸發，[佇列工作](/docs/{{version}}/queues)將會自動使用您指定的廣播驅動程式來廣播該事件。


<a name="broadcast-name"></a>
### 廣播名稱

預設情況下，Laravel 會使用事件的類別名稱來廣播該事件。不過，您可以透過在事件上定義 `broadcastAs` 方法來客製化廣播名稱：

```php
/**
 * The event's broadcast name.
 */
public function broadcastAs(): string
{
    return 'server.created';
}
```

如果您使用 `broadcastAs` 方法客製化了廣播名稱，您應該確保在註冊接聽器時加上前導的 `.` 字元。這會指示 Echo 不要將應用程式的命名空間加到該事件名稱的前面：

```javascript
.listen('.server.created', function (e) {
    // ...
});
```


<a name="broadcast-data"></a>
### 廣播資料

當事件被廣播時，它的所有 `public` 屬性都會自動被序列化，並作為該事件的負載 (Payload) 進行廣播，讓您可以從 JavaScript 應用程式中存取其任何公開資料。因此，舉例來說，如果您的事件有一個包含 Eloquent 模型的單一公開 `$user` 屬性，該事件的廣播負載將會是：

```json
{
    "user": {
        "id": 1,
        "name": "Patrick Stewart"
        ...
    }
}
```

然而，如果您希望對廣播負載有更細緻的控制，您可以在事件中加入 `broadcastWith` 方法。此方法應傳回您希望作為事件負載進行廣播的資料陣列：

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

預設情況下，每個廣播事件都會被置於您的 `queue.php` 設定檔中指定的預設佇列連接的預設佇列上。您可以透過在事件類別上使用 `Connection` 和 `Queue` 屬性 (Attributes) 來客製化廣播器所使用的佇列連接與名稱：

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

或者，您也可以透過在事件上定義 `broadcastQueue` 方法來客製化佇列名稱：

```php
/**
 * The name of the queue on which to place the broadcasting job.
 */
public function broadcastQueue(): string
{
    return 'default';
}
```

如果您想使用 `sync` 佇列而不是預設的佇列驅動程式來廣播事件，您可以實作 `ShouldBroadcastNow` 契約以代替 `ShouldBroadcast`：

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

有時您只想在特定條件為真時才廣播事件。您可以透過在事件類別中加入 `broadcastWhen` 方法來定義這些條件：

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

當廣播事件在資料庫交易中被分派時，它們可能會在資料庫交易提交之前就已經被佇列處理了。當這種情況發生時，您在資料庫交易期間對模型或資料庫紀錄所做的任何更新，可能尚未反映到資料庫中。此外，在交易中建立的任何模型或資料庫紀錄也可能尚未存在於資料庫中。如果您的事件依賴這些模型，則在處理廣播事件的工作時可能會發生非預期的錯誤。

如果您的佇列連接的 `after_commit` 設定選項設為 `false`，您仍然可以透過在事件類別上實作 `ShouldDispatchAfterCommit` 契約，來指出某個特定的廣播事件應該在所有開啟的資料庫交易都提交之後再進行分派：

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
> 若要深入瞭解如何解決這些問題，請參閱關於[佇列工作與資料庫交易](/docs/{{version}}/queues#jobs-and-database-transactions)的說明文件。
<!-- SECTION_TO_TRANSLATE_END -->

<a name="authorizing-channels"></a>
## 授權頻道

私有頻道需要您授權目前已認證的使用者是否確實可以監聽該頻道。這是透過向您的 Laravel 應用程式發送包含頻道名稱的 HTTP 請求來完成的，並允許您的應用程式決定該使用者是否可以監聽該頻道。當使用 [Laravel Echo](#client-side-installation) 時，授權私有頻道訂閱的 HTTP 請求將會自動發送。

安裝廣播功能時，Laravel 會嘗試自動註冊 `/broadcasting/auth` 路由來處理授權請求。如果 Laravel 無法自動註冊這些路由，您可以在應用程式的 `/bootstrap/app.php` 檔案中手動註冊它們：

```php
->withRouting(
    web: __DIR__.'/../routes/web.php',
    channels: __DIR__.'/../routes/channels.php',
    health: '/up',
)
```

<a name="defining-authorization-callbacks"></a>
### 定義授權回呼

接下來，我們需要定義實際判斷目前已認證的使用者是否可以監聽給定頻道的邏輯。這是在由 `install:broadcasting` Artisan 指令所建立的 `routes/channels.php` 檔案中完成的。在此檔案中，您可以使用 `Broadcast::channel` 方法來註冊頻道授權回呼：

```php
use App\Models\User;

Broadcast::channel('orders.{orderId}', function (User $user, int $orderId) {
    return $user->id === Order::findOrNew($orderId)->user_id;
});
```

`channel` 方法接受兩個引數：頻道名稱，以及一個回傳 `true` 或 `false` 以指示使用者是否被授權監聽該頻道的回呼。

所有的授權回呼都會接收目前已認證的使用者作為其第一個引數，並將任何額外的萬用字元參數作為其後的引數。在此範例中，我們使用 `{orderId}` 預留位置來表示頻道名稱的 "ID" 部分是一個萬用字元。

您可以使用 `channel:list` Artisan 指令來檢視應用程式的廣播授權回呼列表：

```shell
php artisan channel:list
```

<a name="authorization-callback-model-binding"></a>
#### 授權回呼的模型綁定

就像 HTTP 路由一樣，頻道路由也可以利用隱式與顯式的[路由模型綁定](/docs/{{version}}/routing#route-model-binding)。例如，您可以要求一個實際的 `Order` 模型實例，而不是接收字串或數值的訂單 ID：

```php
use App\Models\Order;
use App\Models\User;

Broadcast::channel('orders.{order}', function (User $user, Order $order) {
    return $user->id === $order->user_id;
});
```

> [!WARNING]
> 與 HTTP 路由模型綁定不同，頻道模型綁定不支援自動的[隱式模型綁定範圍限制 (Scoping)](/docs/{{version}}/routing#implicit-model-binding-scoping)。然而，這很少會是一個問題，因為大多數頻道都可以根據單一模型的唯一主鍵來限制範圍。

<a name="authorization-callback-authentication"></a>
#### 授權回呼認證

私有與狀態廣播頻道會透過應用程式的預設認證保護器 (Guard) 來認證目前的使用者。如果使用者未通過認證，頻道授權將會自動被拒絕，且授權回呼永遠不會被執行。然而，如有需要，您可以指定多個自訂保護器來認證傳入的請求：

```php
Broadcast::channel('channel', function () {
    // ...
}, ['guards' => ['web', 'admin']]);
```

<a name="defining-channel-classes"></a>
### 定義頻道類別

如果您的應用程式正在使用許多不同的頻道，您的 `routes/channels.php` 檔案可能會變得非常龐大。因此，您可以改用頻道類別，而不是使用閉包來授權頻道。若要產生頻道類別，請使用 `make:channel` Artisan 指令。此指令會將新的頻道類別放置在 `App/Broadcasting` 目錄中。

```shell
php artisan make:channel OrderChannel
```

接下來，在您的 `routes/channels.php` 檔案中註冊您的頻道：

```php
use App\Broadcasting\OrderChannel;

Broadcast::channel('orders.{order}', OrderChannel::class);
```

最後，您可以將頻道的授權邏輯放置在頻道類別的 `join` 方法中。這個 `join` 方法將容納您通常會放置在頻道授權閉包中的相同邏輯。您也可以利用頻道模型綁定的優勢：

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
> 與 Laravel 中的許多其他類別一樣，頻道類別將由[服務容器 (Service Container)](/docs/{{version}}/container)自動解析。因此，您可以在頻道的建構函式中對其所需的任何依賴項進行型別提示。

<a name="broadcasting-events"></a>
## 廣播事件

一旦你定義了事件並將其標記了 `ShouldBroadcast` 介面，你只需要使用該事件的 dispatch 方法來觸發事件。事件分派器會注意到該事件已標記 `ShouldBroadcast` 介面，並會將該事件排入廣播佇列中：

```php
use App\Events\OrderShipmentStatusUpdated;

OrderShipmentStatusUpdated::dispatch($order);
```


<a name="only-to-others"></a>
### 僅廣播給其他人

在建構使用事件廣播的應用程式時，你偶爾會需要將事件廣播給給定頻道的所有訂閱者，但目前使用者除外。你可以使用 `broadcast` 輔助函式和 `toOthers` 方法來達成此目的：

```php
use App\Events\OrderShipmentStatusUpdated;

broadcast(new OrderShipmentStatusUpdated($update))->toOthers();
```

為了解釋什麼時候你可能會想使用 `toOthers` 方法，讓我們想像一個工作清單應用程式，使用者可以透過輸入工作名稱來建立新工作。為了建立工作，你的應用程式可能會向 `/task` URL 發送請求，該 URL 會廣播工作的建立，並回傳新工作的 JSON 表示形式。當你的 JavaScript 應用程式收到來自端點 (endpoint) 的回應時，它可能會直接將新工作插入到其工作清單中，如下所示：

```js
axios.post('/task', task)
    .then((response) => {
        this.tasks.push(response.data);
    });
```

然而，請記住我們也廣播了工作的建立。如果你的 JavaScript 應用程式也在監聽此事件以將工作新增到工作清單中，你的清單中將會出現重複的工作：一個來自端點，一個來自廣播。你可以使用 `toOthers` 方法來解決此問題，以指示廣播器不要將事件廣播給目前使用者。

> [!WARNING]
> 你的事件必須使用 `Illuminate\Broadcasting\InteractsWithSockets` trait 才能呼叫 `toOthers` 方法。


<a name="only-to-others-configuration"></a>
#### 設定

當你初始化 Laravel Echo 實例時，連線會被分配一個 socket ID。如果你使用全域的 [Axios](https://github.com/axios/axios) 實例從 JavaScript 應用程式發送 HTTP 請求，該 socket ID 將會自動作為 `X-Socket-ID` 標頭附加到每個發出的請求中。然後，當你呼叫 `toOthers` 方法時，Laravel 會從標頭中提取 socket ID，並指示廣播器不要廣播到具有該 socket ID 的任何連線。

如果你沒有使用全域的 Axios 實例，你需要手動設定 JavaScript 應用程式在所有發出的請求中發送 `X-Socket-ID` 標頭。你可以使用 `Echo.socketId` 方法來取得 socket ID：

```js
var socketId = Echo.socketId();
```


<a name="customizing-the-connection"></a>
### 自訂連線

如果你的應用程式與多個廣播連線進行互動，並且你希望使用預設以外的廣播器來廣播事件，你可以使用 `via` 方法指定要將事件推送到哪個連線：

```php
use App\Events\OrderShipmentStatusUpdated;

broadcast(new OrderShipmentStatusUpdated($update))->via('pusher');
```

或者，你可以透過在事件的建構子中呼叫 `broadcastVia` 方法來指定事件的廣播連線。然而，在這樣做之前，你應該確保事件類別使用了 `InteractsWithBroadcasting` trait：

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

有時候，你可能想在不建立專用事件類別的情況下，向應用程式的前端廣播一個簡單的事件。為了滿足這種需求，`Broadcast` facade 允許你廣播「匿名事件」：

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

使用 `as` 和 `with` 方法，你可以自訂事件的名稱和資料：

```php
Broadcast::on('orders.'.$order->id)
    ->as('OrderPlaced')
    ->with($order)
    ->send();
```

上面的範例將廣播類似以下的事件：

```json
{
    "event": "OrderPlaced",
    "data": "{ id: 1, total: 100 }",
    "channel": "orders.1"
}
```

如果你想在私有或狀態頻道上廣播匿名事件，可以使用 `private` 和 `presence` 方法：

```php
Broadcast::private('orders.'.$order->id)->send();
Broadcast::presence('channels.'.$channel->id)->send();
```

使用 `send` 方法廣播匿名事件會將事件分派到應用程式的[佇列](/docs/{{version}}/queues)中進行處理。然而，如果你想立即廣播事件，可以使用 `sendNow` 方法：

```php
Broadcast::on('orders.'.$order->id)->sendNow();
```

若要將事件廣播給除了目前已認證使用者之外的所有頻道訂閱者，你可以呼叫 `toOthers` 方法：

```php
Broadcast::on('orders.'.$order->id)
    ->toOthers()
    ->send();
```


<a name="rescuing-broadcasts"></a>
### 救援廣播

當你的應用程式佇列伺服器無法使用，或者 Laravel 在廣播事件時遇到錯誤，通常會拋出異常，導致終端使用者看到應用程式錯誤。由於事件廣播通常是應用程式核心功能的補充，你可以透過在事件上實作 `ShouldRescue` 介面，來防止這些異常干擾使用者體驗。

實作 `ShouldRescue` 介面的事件在嘗試廣播時，會自動利用 Laravel 的 [rescue 輔助函式](/docs/{{version}}/helpers#method-rescue)。此輔助函式會捕獲任何異常，將其報告給應用程式的異常處理器進行記錄，並允許應用程式繼續正常執行，而不會中斷使用者的工作流程：

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

一旦您[安裝並實例化了 Laravel Echo](#client-side-installation)，就可以開始監聽從 Laravel 應用程式廣播的事件。首先，使用 `channel` 方法來取得頻道的實例，然後呼叫 `listen` 方法來監聽指定的事件：

```js
Echo.channel(`orders.${this.order.id}`)
    .listen('OrderShipmentStatusUpdated', (e) => {
        console.log(e.order.name);
    });
```

如果您想監聽私密頻道上的事件，請改用 `private` 方法。您可以繼續串接呼叫 `listen` 方法，以在單一頻道上監聽多個事件：

```js
Echo.private(`orders.${this.order.id}`)
    .listen(/* ... */)
    .listen(/* ... */)
    .listen(/* ... */);
```


<a name="stop-listening-for-events"></a>
#### 停止監聽事件

如果您想停止監聽特定的事件而不[離開頻道](#leaving-a-channel)，可以使用 `stopListening` 方法：

```js
Echo.private(`orders.${this.order.id}`)
    .stopListening('OrderShipmentStatusUpdated');
```


<a name="leaving-a-channel"></a>
### 離開頻道

要離開頻道，您可以在 Echo 實例上呼叫 `leaveChannel` 方法：

```js
Echo.leaveChannel(`orders.${this.order.id}`);
```

如果您想離開頻道及其關聯的私密與狀態頻道，可以呼叫 `leave` 方法：

```js
Echo.leave(`orders.${this.order.id}`);
```

<a name="namespaces"></a>
### 命名空間

您可能已經注意到在上述範例中，我們並未指定事件類別的完整 `App\Events` 命名空間。這是因為 Echo 會自動假設事件位於 `App\Events` 命名空間中。然而，您可以在實例化 Echo 時，透過傳遞 `namespace` 設定選項來設定根命名空間：

```js
window.Echo = new Echo({
    broadcaster: 'pusher',
    // ...
    namespace: 'App.Other.Namespace'
});
```

或者，在使用 Echo 訂閱事件時，您可以在事件類別前面加上 `.` 作為前綴。這將允許您總是指定完整符合的類別名稱：

```js
Echo.channel('orders')
    .listen('.Namespace\\Event\\Class', (e) => {
        // ...
    });
```

<a name="using-react-or-vue"></a>
### 使用 React、Vue 或 Svelte

Laravel Echo 包含了 React、Vue 和 Svelte 的 Hook，讓您可以輕鬆無痛地監聽事件。若要開始使用，請調用 `useEcho` Hook，它用於監聽私有事件。當使用該 Hook 的元件被卸載 (Unmounted) 時，`useEcho` Hook 會自動離開頻道：

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

您可以藉由向 `useEcho` 傳遞一個事件陣列來監聽多個事件：

```js
useEcho(
    `orders.${orderId}`,
    ["OrderShipmentStatusUpdated", "OrderShipped"],
    (e) => {
        console.log(e.order);
    },
);
```

您還可以指定廣播事件 Payload 資料的結構 (Shape)，提供更好的型別安全與開發便利性：

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

雖然 `useEcho` Hook 會在元件卸載時自動離開頻道，但如果需要，您也可以利用其回傳的函式，以程式化的方式手動停止或開始監聽頻道：

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
#### 連線至公開頻道

要連線到公開頻道，您可以使用 `useEchoPublic` Hook：

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
#### 連線至狀態頻道

要連線到狀態頻道，您可以使用 `useEchoPresence` Hook：

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

您可以使用 `useConnectionStatus` Hook 來取得目前的 WebSocket 連線狀態，該 Hook 提供了響應式 (Reactive) 的狀態，並會在連線狀態改變時自動更新：

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

可能的狀態值如下：

<div class="content-list" markdown="1">

- `connected` - 成功連線至 WebSocket 伺服器。
- `connecting` - 正在進行首次連線嘗試。
- `reconnecting` - 在斷開連線後嘗試重新連線。
- `disconnected` - 未連線，且不嘗試重新連線。
- `failed` - 連線失敗，且不會再重試。

</div>

<a name="presence-channels"></a>
## 狀態頻道 (Presence Channels)

狀態頻道 (Presence Channels) 建立在私密頻道的安全基礎之上，同時提供了額外的功能：能夠得知目前有哪些使用者訂閱了該頻道。這使得開發強大的協同作業功能變得非常容易，例如在其他使用者瀏覽相同頁面時通知使用者，或是列出聊天室中的在線成員。


<a name="authorizing-presence-channels"></a>
### 授權狀態頻道

所有的狀態頻道同時也是私密頻道；因此，使用者必須獲得[授權才能存取它們](#authorizing-channels)。然而，在為狀態頻道定義授權回呼時，若使用者被授權加入頻道，您不應該只回傳 `true`。相反地，您應該回傳一個包含該使用者資訊的陣列。

授權回呼所回傳的資料，將會在您的 JavaScript 應用程式中的狀態頻道事件監聽器中取得。如果使用者未被授權加入該狀態頻道，您應該回傳 `false` 或 `null`：

```php
use App\Models\User;

Broadcast::channel('chat.{roomId}', function (User $user, int $roomId) {
    if ($user->canJoinRoom($roomId)) {
        return ['id' => $user->id, 'name' => $user->name];
    }
});
```


<a name="joining-presence-channels"></a>
### 加入狀態頻道

要加入狀態頻道，您可以使用 Echo 的 `join` 方法。`join` 方法會回傳一個 `PresenceChannel` 的實作，除了提供 `listen` 方法外，還能讓您訂閱 `here`、`joining` 與 `leaving` 事件。

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

一旦成功加入頻道，`here` 回呼將會立即執行，並接收一個包含目前已訂閱該頻道之所有其他使用者資訊的陣列。當新使用者加入頻道時，會執行 `joining` 方法；而當使用者離開頻道時，則會執行 `leaving` 方法。當驗證端點回傳非 200 的 HTTP 狀態碼，或是解析回傳的 JSON 出現問題時，則會執行 `error` 方法。


<a name="broadcasting-to-presence-channels"></a>
### 廣播至狀態頻道

狀態頻道可以像公開或私密頻道一樣接收事件。以聊天室為例，我們可能想要廣播 `NewMessage` 事件到該聊天室的狀態頻道。為此，我們要在事件的 `broadcastOn` 方法中回傳 `PresenceChannel` 的實例：

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

與其他事件相同，您可以使用 `broadcast` 輔助函式和 `toOthers` 方法，來排除目前的使用者接收該廣播：

```php
broadcast(new NewMessage($message));

broadcast(new NewMessage($message))->toOthers();
```

如同其他類型的事件，您可以使用 Echo 的 `listen` 方法來監聽發送到狀態頻道的事件：

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
## 模型廣播 (Model Broadcasting)

> [!WARNING]
> 在閱讀以下關於模型廣播的說明文件之前，我們建議您先熟悉 Laravel 廣播服務的一般概念，以及如何手動建立與監聽廣播事件。

當您應用程式的 [Eloquent models](/docs/{{version}}/eloquent) 被建立、更新或刪除時，通常會廣播事件。當然，這可以透過手動[為 Eloquent 模型狀態變更定義自訂事件](/docs/{{version}}/eloquent#events)並讓這些事件實作 `ShouldBroadcast` 介面來輕鬆達成。

然而，如果您在應用程式中不打算將這些事件用於其他用途，僅為了廣播而建立事件類別可能會顯得有些繁瑣。為了解決這個問題，Laravel 允許您指定 Eloquent 模型在狀態變更時自動進行廣播。

要開始使用，您的 Eloquent 模型應該使用 `Illuminate\Database\Eloquent\BroadcastsEvents` trait。此外，該模型應定義一個 `broadcastOn` 方法，該方法將返回模型事件應廣播於其上的頻道陣列：

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

一旦您的模型引入了此 trait 並定義了其廣播頻道，當模型實例被建立、更新、刪除、移至垃圾桶 (trashed) 或還原 (restored) 時，它就會開始自動廣播事件。

此外，您可能已經注意到 `broadcastOn` 方法接收了一個字串型別的 `$event` 參數。此參數包含模型上發生的事件類型，其值會是 `created`、`updated`、`deleted`、`trashed` 或 `restored`。透過檢查此變數的值，您可以決定模型在特定事件發生時應該廣播到哪些頻道（如果有的話）：

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

有時，您可能希望自訂 Laravel 建立底層模型廣播事件的方式。您可以透過在 Eloquent 模型中定義 `newBroadcastableEvent` 方法來實現。此方法應返回一個 `Illuminate\Database\Eloquent\BroadcastableModelEventOccurred` 實例：

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

如您所見，在上述的模型範例中，`broadcastOn` 方法並沒有返回 `Channel` 實例。相反地，它直接返回了 Eloquent 模型。如果您的模型 `broadcastOn` 方法返回了 Eloquent 模型實例（或包含在該方法返回的陣列中），Laravel 將會自動使用該模型的類別名稱與主鍵識別碼作為頻道名稱，為該模型實例化一個私有頻道。

因此，一個 `id` 為 `1` 的 `App\Models\User` 模型將會被轉換為一個名稱為 `App.Models.User.1` 的 `Illuminate\Broadcasting\PrivateChannel` 實例。當然，除了從模型的 `broadcastOn` 方法返回 Eloquent 模型實例之外，您也可以返回完整的 `Channel` 實例，以便完全控制模型的頻道名稱：

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

如果您打算從模型的 `broadcastOn` 方法中明確返回一個頻道實例，您可以將 Eloquent 模型實例傳遞給該頻道的建構函式。這樣做時，Laravel 將會使用上述討論的模型頻道慣例，將 Eloquent 模型轉換為頻道名稱字串：

```php
return [new Channel($this->user)];
```

如果您需要取得某個模型的頻道名稱，您可以在任何模型實例上呼叫 `broadcastChannel` 方法。例如，對於一個 `id` 為 `1` 的 `App\Models\User` 模型，此方法會返回字串 `App.Models.User.1`：

```php
$user->broadcastChannel();
```

<a name="model-broadcasting-event-conventions"></a>
#### 事件慣例

由於模型廣播事件並非與您應用程式中 `App\Events` 目錄下的「實際」事件相關聯，因此它們會根據慣例被賦予名稱與承載資料 (Payload)。Laravel 的慣例是使用模型的類別名稱（不包含命名空間）加上觸發廣播的模型事件名稱來廣播該事件。

因此，舉例來說，更新 `App\Models\Post` 模型將會向您的用戶端應用程式廣播一個名為 `PostUpdated` 的事件，其承載資料如下：

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

而刪除 `App\Models\User` 模型則會廣播一個名為 `UserDeleted` 的事件。

如果您希望的話，也可以透過在模型中新增 `broadcastAs` 與 `broadcastWith` 方法來定義自訂的廣播名稱與承載資料。這些方法會接收正在發生的模型事件/操作名稱，讓您可以針對每種模型操作自訂事件的名稱與承載資料。如果 `broadcastAs` 方法返回 `null`，Laravel 在廣播事件時將會採用上述討論的模型廣播事件名稱慣例：

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

當您將 `BroadcastsEvents` Trait 新增至您的 Model，並定義了 Model 的 `broadcastOn` 方法後，您就準備好開始在用戶端應用程式中監聽廣播的模型事件了。在開始之前，您可能需要參考關於[監聽事件](#listening-for-events)的完整說明文件。

首先，使用 `private` 方法來取得頻道的實例，接著呼叫 `listen` 方法來監聽指定的事件。通常，提供給 `private` 方法的頻道名稱應符合 Laravel 的[模型廣播慣例](#model-broadcasting-conventions)。

一旦取得頻道實例後，您就可以使用 `listen` 方法來監聽特定的事件。由於模型廣播事件與您應用程式中 `App\Events` 目錄下的「實際」事件無關，因此[事件名稱](#model-broadcasting-event-conventions)前面必須加上 `.` 前綴，以表示它不屬於特定的命名空間。每個模型廣播事件都包含一個 `model` 屬性，其中含有該模型所有可廣播的屬性：

```js
Echo.private(`App.Models.User.${this.user.id}`)
    .listen('.UserUpdated', (e) => {
        console.log(e.model);
    });
```


<a name="model-broadcasts-with-react-or-vue"></a>
#### 使用 React、Vue 或 Svelte

如果您使用的是 React、Vue 或 Svelte，您可以使用 Laravel Echo 內建的 `useEchoModel` Hook 來輕鬆監聽模型廣播：

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

您也可以指定模型事件 Payload 資料的結構，提供更好的型別安全與開發編輯時的便利性：

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
> 使用 [Pusher Channels](https://pusher.com/channels) 時，您必須在 [應用程式主控面板 (application dashboard)](https://dashboard.pusher.com/) 的「App Settings (應用程式設定)」區塊中啟用「Client Events」選項，才能發送用戶端事件。

有時您可能希望在完全不觸及 Laravel 應用程式的情況下，直接向其他已連線的用戶端廣播事件。這在像是「正在輸入 (typing)」的通知特別有用，例如您想要提示應用程式的使用者，另一個使用者正在特定畫面上輸入訊息。

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

```svelte tab=Svelte
<script>
import { useEcho } from "@laravel/echo-svelte";

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

藉由將事件廣播與 [通知](/docs/{{version}}/notifications) 結合，您的 JavaScript 應用程式可以在通知發生時即時接收，而無需重新整理頁面。在開始之前，請務必先閱讀有關使用 [廣播通知頻道](/docs/{{version}}/notifications#broadcast-notifications) 的文件。

一旦您將通知設定為使用廣播頻道，就可以使用 Echo 的 `notification` 方法來監聽廣播事件。請記住，頻道名稱應與接收通知之實體的類別名稱相符合：

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

在此範例中，所有透過 `broadcast` 頻道發送給 `App\Models\User` 實例的通知都會被該回呼函式接收。而 `App.Models.User.{id}` 頻道的授權回呼則已內建於應用程式的 `routes/channels.php` 檔案中。


<a name="stop-listening-for-notifications"></a>
#### 停止監聽通知

如果您希望在不 [離開頻道](#leaving-a-channel) 的情況下停止監聽通知，可以使用 `stopListeningForNotification` 方法：

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