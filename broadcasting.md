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
- [概念總覽](#concept-overview)
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
    - [自訂連線](#customizing-the-connection)
    - [匿名事件](#anonymous-events)
    - [救援廣播](#rescuing-broadcasts)
- [接收廣播](#receiving-broadcasts)
    - [監聽事件](#listening-for-events)
    - [離開頻道](#leaving-a-channel)
    - [命名空間](#namespaces)
    - [使用 React、Vue 或 Svelte](#using-react-or-vue)
- [Presence 頻道](#presence-channels)
    - [授權 Presence 頻道](#authorizing-presence-channels)
    - [加入 Presence 頻道](#joining-presence-channels)
    - [廣播至 Presence 頻道](#broadcasting-to-presence-channels)
- [Model 廣播](#model-broadcasting)
    - [Model 廣播慣例](#model-broadcasting-conventions)
    - [監聽 Model 廣播](#listening-for-model-broadcasts)
- [客戶端事件](#client-events)
- [通知](#notifications)

<a name="introduction"></a>
## 簡介

在許多現代的 Web 應用程式中，WebSockets 被用來實現即時、動態更新的使用者介面。當伺服器上的某些資料更新時，通常會透過 WebSocket 連線發送一則訊息並交由客戶端處理。相較於持續向應用程式伺服器輪詢（Polling）資料變更以反映在 UI 上，WebSockets 提供了一個更有效率的替代方案。

舉例來說，假設您的應用程式能夠將使用者的資料匯出為 CSV 檔案並透過電子郵件寄給他們。然而，建立這個 CSV 檔案需要花費數分鐘，因此您選擇在[佇列任務](/docs/{{version}}/queues)中建立並寄出 CSV。當 CSV 建立成功並寄出給使用者後，我們可以透過事件廣播來發送一個 `App\Events\UserDataExported` 事件，並由我們應用程式的 JavaScript 接收。一旦收到該事件，我們就能在使用者完全不需要重新整理頁面的情況下，向使用者顯示其 CSV 已寄出的訊息。

為了協助您建構這類功能，Laravel 讓您可以輕鬆地透過 WebSocket 連線「廣播」伺服器端的 Laravel [事件](/docs/{{version}}/events)。廣播您的 Laravel 事件允許您在伺服器端 Laravel 應用程式與客戶端 JavaScript 應用程式之間共享相同的事件名稱與資料。

廣播背後的核心概念非常簡單：客戶端在前端連線至具名的頻道（Channel），而您的 Laravel 應用程式則在後端將事件廣播至這些頻道。這些事件可以包含任何您希望提供給前端的額外資料。

<a name="supported-drivers"></a>
#### 支援的驅動程式

預設情況下，Laravel 內建了三種伺服器端廣播驅動程式供您選擇：[Laravel Reverb](https://reverb.laravel.com)、[Pusher Channels](https://pusher.com/channels) 和 [Ably](https://ably.com)。

> [!NOTE]
> 在深入瞭解事件廣播之前，請確保您已閱讀過 Laravel 的[事件與監聽器](/docs/{{version}}/events)文件。

<a name="quickstart"></a>
## 快速入門

預設情況下，新的 Laravel 應用程式並未啟用廣播功能。您可以透過 `install:broadcasting` Artisan 指令來啟用廣播：

```shell
php artisan install:broadcasting
```

`install:broadcasting` 指令會提示您選擇要使用哪種事件廣播服務。此外，它還會建立 `config/broadcasting.php` 設定檔以及 `routes/channels.php` 檔案，您可以在該檔案中註冊應用程式的廣播授權路由與回呼。

Laravel 開箱即支援多種廣播驅動程式：[Laravel Reverb](/docs/{{version}}/reverb)、[Pusher Channels](https://pusher.com/channels)、[Ably](https://ably.com)，以及用於本機開發與偵錯的 `log` 驅動程式。此外還包含一個 `null` 驅動程式，允許您在測試期間停用廣播。在 `config/broadcasting.php` 設定檔中包含了這些驅動程式的設定範例。

您應用程式的所有事件廣播設定都儲存在 `config/broadcasting.php` 設定檔中。如果您的應用程式中沒有這個檔案，請不用擔心；當您執行 `install:broadcasting` Artisan 指令時就會建立該檔案。

<a name="quickstart-next-steps"></a>
#### 後續步驟

啟用事件廣播後，您就可以進一步瞭解如何[定義廣播事件](#defining-broadcast-events)以及[監聽事件](#listening-for-events)。如果您正在使用 Laravel 的 React、Vue 或 Svelte [入門套件](/docs/{{version}}/starter-kits)，可以使用 Echo 的 [useEcho hook](#using-react-or-vue) 來監聽事件。

> [!NOTE]
> 在廣播任何事件之前，您應該先設定並執行 [queue worker](/docs/{{version}}/queues)（佇列任務處理器）。所有的事件廣播都是透過佇列任務完成的，這樣您應用程式的回應時間才不會受到廣播事件的嚴重影響。

<a name="server-side-installation"></a>
## 伺服器端安裝

要開始使用 Laravel 的事件廣播，我們需要在 Laravel 應用程式內進行一些設定，並安裝幾個套件。

事件廣播是由伺服器端的廣播驅動程式來完成，它會廣播你的 Laravel 事件，以便 Laravel Echo（一個 JavaScript 函式庫）可以在瀏覽器客戶端中接收它們。別擔心——我們將逐步引導你完成安裝過程的每個部分。


<a name="reverb"></a>
### Reverb

若要在使用 Reverb 作為事件廣播器時快速啟用 Laravel 的廣播功能支援，請在執行 `install:broadcasting` Artisan 指令時加上 `--reverb` 選項。這個 Artisan 指令會安裝 Reverb 所需的 Composer 和 NPM 套件，並使用適當的變數更新應用程式的 `.env` 檔案：

```shell
php artisan install:broadcasting --reverb
```


<a name="reverb-manual-installation"></a>
#### 手動安裝

當執行 `install:broadcasting` 指令時，系統會提示你安裝 [Laravel Reverb](/docs/{{version}}/reverb)。當然，你也可以使用 Composer 套件管理器手動安裝 Reverb：

```shell
composer require laravel/reverb
```

套件安裝完成後，你可以執行 Reverb 的安裝指令來發布設定檔、新增 Reverb 所需的環境變數，並在你的應用程式中啟用事件廣播：

```shell
php artisan reverb:install
```

你可以在 [Reverb 文件](/docs/{{version}}/reverb) 中找到詳細的 Reverb 安裝與使用說明。


<a name="pusher-channels"></a>
### Pusher Channels

若要在使用 Pusher 作為事件廣播器時快速啟用 Laravel 的廣播功能支援，請在執行 `install:broadcasting` Artisan 指令時加上 `--pusher` 選項。這個 Artisan 指令會提示你輸入 Pusher 的憑證、安裝 Pusher 的 PHP 與 JavaScript SDK，並使用適當的變數更新應用程式的 `.env` 檔案：

```shell
php artisan install:broadcasting --pusher
```


<a name="pusher-manual-installation"></a>
#### 手動安裝

若要手動安裝 Pusher 支援，你應該使用 Composer 套件管理器安裝 Pusher Channels PHP SDK：

```shell
composer require pusher/pusher-php-server
```

接下來，你應該在 `config/broadcasting.php` 設定檔中設定你的 Pusher Channels 憑證。該檔案中已經包含一個 Pusher Channels 設定範例，讓你能快速指定你的 key、secret 與 application ID。通常，你應該在應用程式的 `.env` 檔案中設定 Pusher Channels 憑證：

```ini
PUSHER_APP_ID="your-pusher-app-id"
PUSHER_APP_KEY="your-pusher-key"
PUSHER_APP_SECRET="your-pusher-secret"
PUSHER_HOST=
PUSHER_PORT=443
PUSHER_SCHEME="https"
PUSHER_APP_CLUSTER="mt1"
```

`config/broadcasting.php` 檔案中的 `pusher` 設定還允許你指定 Channels 所支援的其他 `options`，例如叢集 (cluster)。

然後，在應用程式的 `.env` 檔案中將 `BROADCAST_CONNECTION` 環境變數設定為 `pusher`：

```ini
BROADCAST_CONNECTION=pusher
```

最後，你可以準備安裝並設定 [Laravel Echo](#client-side-installation)，它將在客戶端接收廣播事件。


<a name="ably"></a>
### Ably

> [!NOTE]
> 下方的文件討論了如何在「Pusher 相容」模式下使用 Ably。不過，Ably 團隊建議並維護了一個能夠利用 Ably 所提供之獨特功能的廣播器與 Echo 客戶端。有關使用 Ably 維護的驅動程式的更多資訊，請[參閱 Ably 的 Laravel 廣播器文件](https://github.com/ably/laravel-broadcaster)。

若要在使用 [Ably](https://ably.com) 作為事件廣播器時快速啟用 Laravel 的廣播功能支援，請在執行 `install:broadcasting` Artisan 指令時加上 `--ably` 選項。這個 Artisan 指令會提示你輸入 Ably 的憑證、安裝 Ably 的 PHP 與 JavaScript SDK，並使用適當的變數更新應用程式的 `.env` 檔案：

```shell
php artisan install:broadcasting --ably
```

**在繼續之前，你應該在 Ably 應用程式設定中啟用 Pusher 協定支援。你可以在 Ably 應用程式設定儀表板的「Protocol Adapter Settings」部分中啟用此功能。**


<a name="ably-manual-installation"></a>
#### 手動安裝

若要手動安裝 Ably 支援，你應該使用 Composer 套件管理器安裝 Ably PHP SDK：

```shell
composer require ably/ably-php
```

接下來，你應該在 `config/broadcasting.php` 設定檔中設定你的 Ably 憑證。該檔案中已經包含一個 Ably 設定範例，讓你能快速指定你的 key。通常，這個值應該透過 `ABLY_KEY` [環境變數](/docs/{{version}}/configuration#environment-configuration)來設定：

```ini
ABLY_KEY=your-ably-key
```

然後，在應用程式的 `.env` 檔案中將 `BROADCAST_CONNECTION` 環境變數設定為 `ably`：

```ini
BROADCAST_CONNECTION=ably
```

最後，你可以準備安裝並設定 [Laravel Echo](#client-side-installation)，它將在客戶端接收廣播事件。

<a name="client-side-installation"></a>
## 客戶端安裝

<a name="client-reverb"></a>
### Reverb

[Laravel Echo](https://github.com/laravel/echo) 是一個 JavaScript 函式庫，它能讓你輕鬆地訂閱頻道並監聽由伺服器端廣播驅動所廣播的事件。

當透過 `install:broadcasting` Artisan 指令安裝 Laravel Reverb 時，Reverb 與 Echo 的基底結構與設定會自動注入至你的應用程式中。然而，如果你希望手動設定 Laravel Echo，可以按照以下說明進行。

<a name="reverb-client-manual-installation"></a>
#### 手動安裝

若要為應用程式的前端手動設定 Laravel Echo，請先安裝 `pusher-js` 套件，因為 Reverb 使用 Pusher 協定來處理 WebSocket 訂閱、頻道和訊息：

```shell
npm install --save-dev laravel-echo pusher-js
```

當安裝好 Echo 後，你就可以在應用程式的 JavaScript 中建立一個全新的 Echo 實例。放置此程式碼的絕佳位置是 Laravel 框架所附帶的 `resources/js/app.js` 檔案底部：

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

接下來，你應該編譯應用程式的靜態資源：

```shell
npm run build
```

> [!WARNING]
> Laravel Echo 的 `reverb` 廣播器需要 laravel-echo v1.16.0 或更高版本。

<a name="client-pusher-channels"></a>
### Pusher Channels

[Laravel Echo](https://github.com/laravel/echo) 是一個 JavaScript 函式庫，它能讓你輕鬆地訂閱頻道並監聽由伺服器端廣播驅動所廣播的事件。

當透過 `install:broadcasting --pusher` Artisan 指令安裝廣播支援時，Pusher 與 Echo 的基底結構與設定會自動注入至你的應用程式中。然而，如果你希望手動設定 Laravel Echo，可以按照以下說明進行。

<a name="pusher-client-manual-installation"></a>
#### 手動安裝

若要為應用程式的前端手動設定 Laravel Echo，請先安裝 `laravel-echo` 與 `pusher-js` 套件，這兩個套件利用 Pusher 協定來處理 WebSocket 訂閱、頻道與訊息：

```shell
npm install --save-dev laravel-echo pusher-js
```

當安裝好 Echo 後，你就可以在應用程式的 `resources/js/app.js` 檔案中建立一個全新的 Echo 實例：

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

接下來，你應該在應用程式的 `.env` 檔案中為 Pusher 環境變數設定適當的值。如果這些變數尚未存在於你的 `.env` 檔案中，你應該新增它們：

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

當你根據應用程式的需求調整好 Echo 設定後，即可編譯應用程式的靜態資源：

```shell
npm run build
```

> [!NOTE]
> 若要深入瞭解如何編譯應用程式的 JavaScript 靜態資源，請參閱 [Vite](/docs/{{version}}/vite) 的說明文件。

<a name="using-an-existing-client-instance"></a>
#### 使用現有的客戶端實例

如果你已經有一個預先設定好的 Pusher Channels 客戶端實例，並希望 Echo 使用它，你可以透過 `client` 設定選項將其傳遞給 Echo：

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
> 以下文件將討論如何在「Pusher 相容性」模式下使用 Ably。然而，Ably 團隊建議並維護能夠利用 Ably 提供之獨特功能的廣播器與 Echo 客戶端。關於使用 Ably 維護之驅動程式的更多資訊，請[參閱 Ably 的 Laravel 廣播器文件](https://github.com/ably/laravel-broadcaster)。

[Laravel Echo](https://github.com/laravel/echo) 是一個 JavaScript 函式庫，能讓您輕鬆訂閱頻道並監聽伺服器端廣播驅動程式發布的事件。

當透過 `install:broadcasting --ably` Artisan 指令安裝廣播支援時，Ably 和 Echo 的骨架與設定將會自動注入到您的應用程式中。然而，如果您想手動設定 Laravel Echo，可以按照以下說明進行操作。

<a name="ably-client-manual-installation"></a>
#### 手動安裝

若要為應用程式的前端手動設定 Laravel Echo，首先安裝 `laravel-echo` 與 `pusher-js` 套件，這些套件利用 Pusher 協定來處理 WebSocket 的訂閱、頻道和訊息：

```shell
npm install --save-dev laravel-echo pusher-js
```

**在繼續之前，您應該在 Ably 應用程式設定中啟用 Pusher 協定支援。您可以在 Ably 應用程式設定儀表板的「Protocol Adapter Settings」部分中啟用此功能。**

安裝 Echo 後，您就可以在應用程式的 `resources/js/app.js` 檔案中建立全新的 Echo 實例：

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

您可能已經注意到我們的 Ably Echo 設定引用了 `VITE_ABLY_PUBLIC_KEY` 環境變數。此變數的值應該是您的 Ably 公開金鑰。您的公開金鑰是 Ably 金鑰中位於 `:` 字元之前的內容。

依據您的需求調整 Echo 設定後，您就可以編譯應用程式的靜態資源：

```shell
npm run dev
```

> [!NOTE]
> 若要深入了解如何編譯應用程式的 JavaScript 靜態資源，請參閱 [Vite](/docs/{{version}}/vite) 的說明文件。

<a name="concept-overview"></a>
## 概念總覽

Laravel 的事件廣播允許您透過以驅動程式為基礎的 WebSockets 做法，將伺服器端的 Laravel 事件廣播至客戶端的 JavaScript 應用程式。目前，Laravel 內建提供了 [Laravel Reverb](https://reverb.laravel.com)、[Pusher Channels](https://pusher.com/channels) 和 [Ably](https://ably.com) 驅動程式。這些事件可以在客戶端使用 [Laravel Echo](#client-side-installation) JavaScript 套件輕鬆接收。

事件是透過「頻道」來進行廣播的，頻道可以指定為公開 (public) 或私有 (private)。任何造訪您應用程式的訪客都可以在無需任何認證或授權的情況下訂閱公開頻道；然而，為了訂閱私有頻道，使用者必須通過認證並獲得授權才能監聽該頻道。


<a name="using-example-application"></a>
### 使用範例應用程式

在深入研究事件廣播的各個組件之前，讓我們以一個電子商務商店為例，來進行高階總覽。

在我們的應用程式中，假設我們有一個頁面允許使用者檢視其訂單的出貨狀態。我們也假設當應用程式處理出貨狀態更新時，會觸發一個 `OrderShipmentStatusUpdated` 事件：

```php
use App\Events\OrderShipmentStatusUpdated;

OrderShipmentStatusUpdated::dispatch($order);
```


<a name="the-shouldbroadcast-interface"></a>
#### `ShouldBroadcast` 介面

當使用者正在檢視他們的其中一個訂單時，我們不希望他們必須重新整理頁面才能檢視狀態更新。相反地，我們希望在更新建立時將其廣播給應用程式。因此，我們需要用 `ShouldBroadcast` 介面標示 `OrderShipmentStatusUpdated` 事件。這將指示 Laravel 在觸發事件時將其廣播：

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

`ShouldBroadcast` 介面要求我們的事件定義一個 `broadcastOn` 方法。此方法負責回傳該事件應該要在哪些頻道上廣播。生成的事件類別中已經定義了此方法的空白骨架，因此我們只需要填入詳細資訊即可。我們只希望訂單的建立者能夠檢視狀態更新，因此我們將在與該訂單綁定的私有頻道上廣播該事件：

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

如果您希望事件在多個頻道上廣播，您可以改為回傳一個 `array`：

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

請記住，使用者必須經過授權才能在私有頻道上監聽。我們可以在應用程式的 `routes/channels.php` 檔案中定義我們的頻道授權規則。在這個範例中，我們需要驗證嘗試在私有 `orders.1` 頻道上監聽的任何使用者是否確實是該訂單的建立者：

```php
use App\Models\Order;
use App\Models\User;

Broadcast::channel('orders.{orderId}', function (User $user, int $orderId) {
    return $user->id === Order::findOrNew($orderId)->user_id;
});
```

`channel` 方法接受兩個引數：頻道的名稱以及一個回傳 `true` 或 `false` 的回呼，用以指示使用者是否有權監聽該頻道。

所有授權回呼都會接收當前已認證的使用者作為其第一個引數，並將任何額外萬用字元參數作為其後續引數。在這個範例中，我們使用 `{orderId}` 預留位置來表示頻道名稱的 "ID" 部分是一個萬用字元。


<a name="listening-for-event-broadcasts"></a>
#### 監聽事件廣播

接下來，剩下的就是要在我們的 JavaScript 應用程式中監聽該事件。我們可以使用 [Laravel Echo](#client-side-installation) 來達成這一點。Laravel Echo 內建的 React、Vue 與 Svelte Hook 讓入門變得非常簡單，而且預設情況下，事件的所有公用屬性都將包含在廣播事件中：

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

若要告知 Laravel 特定事件應該被廣播，你必須在事件類別中實作 `Illuminate\Contracts\Broadcasting\ShouldBroadcast` 介面。框架產生的所有事件類別都已經匯入此介面，因此你可以輕鬆地將其新增至你的任何事件中。

`ShouldBroadcast` 介面需要你實作一個方法：`broadcastOn`。`broadcastOn` 方法應回傳事件應廣播於其上的頻道或頻道陣列。這些頻道應該是 `Channel`、`PrivateChannel` 或 `PresenceChannel` 的實例。`Channel` 的實例代表任何使用者皆可訂閱的公開頻道，而 `PrivateChannels` 與 `PresenceChannels` 則代表需要[頻道授權](#authorizing-channels)的私人頻道：

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

實作 `ShouldBroadcast` 介面後，你只需要像平常一樣[觸發事件](/docs/{{version}}/events)即可。事件觸發後，一個[佇列任務](/docs/{{version}}/queues)會自動使用你指定的廣播驅動程式來廣播該事件。


<a name="broadcast-name"></a>
### 廣播名稱

預設情況下，Laravel 會使用事件的類別名稱來廣播事件。不過，你可以透過在事件中定義 `broadcastAs` 方法來自訂廣播名稱：

```php
/**
 * The event's broadcast name.
 */
public function broadcastAs(): string
{
    return 'server.created';
}
```

如果你使用 `broadcastAs` 方法自訂廣播名稱，你應該確保在註冊監聽器時開頭加上 `.` 字元。這會指示 Echo 不要將應用程式的命名空間附加至事件名稱的前面：

```javascript
.listen('.server.created', function (e) {
    // ...
});
```


<a name="broadcast-data"></a>
### 廣播資料

當事件被廣播時，它的所有 `public` 屬性都會被自動序列化並作為事件的負載（Payload）廣播出去，讓你可以在 JavaScript 應用程式中存取其任何公開資料。因此，舉例來說，如果你的事件有一個包含 Eloquent Model 的公開 `$user` 屬性，該事件的廣播負載將會是：

```json
{
    "user": {
        "id": 1,
        "name": "Patrick Stewart"
        ...
    }
}
```

不過，如果你希望對廣播負載有更細粒度的控制，可以在事件中新增 `broadcastWith` 方法。該方法應回傳你希望作為事件負載廣播的資料陣列：

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

預設情況下，每個廣播事件都會放置在 `queue.php` 設定檔中指定的預設佇列連線的預設佇列上。你可以透過在事件類別上使用 `Connection` 和 `Queue` 屬性（Attribute）來自訂廣播器所使用的佇列連線與名稱：

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

另外，你也可以透過在事件中定義 `broadcastQueue` 方法來自訂佇列名稱：

```php
/**
 * The name of the queue on which to place the broadcasting job.
 */
public function broadcastQueue(): string
{
    return 'default';
}
```

如果你想使用 `sync` 佇列而非預設佇列驅動程式來廣播事件，可以實作 `ShouldBroadcastNow` 介面來替代 `ShouldBroadcast`：

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

有時你可能希望僅在給定條件為 true 時才廣播事件。你可以透過在事件類別中新增 `broadcastWhen` 方法來定義這些條件：

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

當廣播事件在資料庫交易（Transaction）內被分派時，它們可能會在資料庫交易提交（Commit）之前就被佇列處理。發生這種情況時，你在資料庫交易期間對 Model 或資料庫紀錄所做的任何更新，可能尚未反映在資料庫中。此外，在交易內建立的任何 Model 或資料庫紀錄可能還不存在於資料庫中。如果你的事件依賴這些 Model，當處理廣播該事件的任務時，可能會發生意料之外的錯誤。

如果佇列連線的 `after_commit` 設定選項設定為 `false`，你仍然可以透過在事件類別上實作 `ShouldDispatchAfterCommit` 介面，來表示特定廣播事件應該在所有未結案的資料庫交易都已提交後才被分派：

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
> 若要深入瞭解如何解決這些問題，請參考 [佇列任務與資料庫交易](/docs/{{version}}/queues#jobs-and-database-transactions) 的相關說明文件。

<a name="authorizing-channels"></a>
## 頻道授權

私有頻道需要您授權當前通過認證的使用者是否真的可以監聽該頻道。這是透過向您的 Laravel 應用程式發送包含頻道名稱的 HTTP 請求，並由您的應用程式判斷該使用者是否可以監聽該頻道來實現的。當使用 [Laravel Echo](#client-side-installation) 時，用於授權訂閱私有頻道的 HTTP 請求會自動發送。

當安裝廣播功能時，Laravel 會嘗試自動註冊 `/broadcasting/auth` 路由來處理授權請求。如果 Laravel 未能自動註冊這些路由，您可以在應用程式的 `/bootstrap/app.php` 檔案中手動註冊它們：

```php
->withRouting(
    web: __DIR__.'/../routes/web.php',
    channels: __DIR__.'/../routes/channels.php',
    health: '/up',
)
```

<a name="defining-authorization-callbacks"></a>
### 定義授權回呼

接下來，我們需要定義實際判斷當前通過認證的使用者是否可以監聽給定頻道的邏輯。這是在由 `install:broadcasting` Artisan 指令所建立的 `routes/channels.php` 檔案中完成的。在該檔案中，您可以可以使用 `Broadcast::channel` 方法來註冊頻道授權回呼：

```php
use App\Models\User;

Broadcast::channel('orders.{orderId}', function (User $user, int $orderId) {
    return $user->id === Order::findOrNew($orderId)->user_id;
});
```

`channel` 方法接受兩個引數：頻道的名稱，以及一個返回 `true` 或 `false` 以指示使用者是否有權監聽該頻道的點呼。

所有授權回呼都會接收當前通過認證的使用者作為其第一個引數，並將任何額外的萬用字元參數作為其後續引數。在這個範例中，我們使用 `{orderId}` 預留位置來表示頻道名稱的 "ID" 部分是一個萬用字元。

您可以使用 `channel:list` Artisan 指令檢視應用程式的廣播授權回呼列表：

```shell
php artisan channel:list
```

<a name="authorization-callback-model-binding"></a>
#### 授權回呼模型綁定

就像 HTTP 路由一樣，頻道路由也可以利用隱式與顯式的[路由模型綁定](/docs/{{version}}/routing#route-model-binding)。例如，您可以請求一個實際的 `Order` 模型實例，而不是接收字串或數字的訂單 ID：

```php
use App\Models\Order;
use App\Models\User;

Broadcast::channel('orders.{order}', function (User $user, Order $order) {
    return $user->id === $order->user_id;
});
```

> [!WARNING]
> 與 HTTP 路由模型綁定不同，頻道模型綁定不支援自動的[隱式模型綁定作用域](/docs/{{version}}/routing#implicit-model-binding-scoping)。然而，這很少成為問題，因為大多數頻道都可以基於單一模型的唯一主鍵來限定作用域。

<a name="authorization-callback-authentication"></a>
#### 授權回呼認證

私有和 Presence 廣播頻道會透過應用程式的預設認證 Guard 來認證當前使用者。如果使用者未通過認證，頻道授權將自動被拒絕，且授權回呼永遠不會被執行。不過，如果需要的話，您可以指定多個自訂 Guard 來認證傳入的請求：

```php
Broadcast::channel('channel', function () {
    // ...
}, ['guards' => ['web', 'admin']]);
```

<a name="defining-channel-classes"></a>
### 定義頻道類別

如果您的應用程式正在使用許多不同的頻道，您的 `routes/channels.php` 檔案可能會變得非常龐大。因此，除了使用 Closure 來授權頻道之外，您還可以使用頻道類別。要產生頻道類別，請使用 `make:channel` Artisan 指令。該指令會在 `App/Broadcasting` 目錄中放置一個新的頻道類別。

```shell
php artisan make:channel OrderChannel
```

接下來，在您的 `routes/channels.php` 檔案中註冊您的頻道：

```php
use App\Broadcasting\OrderChannel;

Broadcast::channel('orders.{order}', OrderChannel::class);
```

最後，您可以將頻道的授權邏輯放置在頻道類別的 `join` 方法中。這個 `join` method 將包含您通常會放在頻道授權 Closure 中的相同邏輯。您也可以利用頻道模型綁定的優勢：

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
> 就像 Laravel 中的許多其他類別一樣，頻道類別將自動由[服務容器](/docs/{{version}}/container)解析。因此，您可以在其建構子中對頻道所需的任何依賴進行型態提示 (Type-hint)。

<a name="broadcasting-events"></a>
## 廣播事件

當您定義了一個事件並實作 `ShouldBroadcast` 介面後，您只需要使用該事件的 dispatch（分派）方法來觸發事件即可。事件分派器會注意到該事件已實作 `ShouldBroadcast` 介面，並將該事件排入佇列以進行廣播：

```php
use App\Events\OrderShipmentStatusUpdated;

OrderShipmentStatusUpdated::dispatch($order);
```

<a name="only-to-others"></a>
### 僅廣播給其他人

在建立使用事件廣播的應用程式時，您偶爾可能需要將事件廣播給指定頻道的所有訂閱者，但排除當前使用者。您可以使用 `broadcast` 輔助函式與 `toOthers` 方法來達成此目的：

```php
use App\Events\OrderShipmentStatusUpdated;

broadcast(new OrderShipmentStatusUpdated($update))->toOthers();
```

為了更好地理解何時可能需要使用 `toOthers` 方法，讓我們想像一個工作清單（Task list）應用程式，使用者可以透過輸入工作名稱來建立新工作。為了建立工作，您的應用程式可能會向 `/task` URL 發送請求，該請求會廣播工作的建立並傳回新工作的 JSON 表示法。當您的 JavaScript 應用程式從端點收到回應時，它可能會直接將新工作插入其工作清單中，如下所示：

```js
axios.post('/task', task)
    .then((response) => {
        this.tasks.push(response.data);
    });
```

然而，請記住我們同時也廣播了工作的建立。如果您的 JavaScript 應用程式同時也在監聽此事件以將工作新增至工作清單，您的清單中將會有重複的工作：一個來自端點回應，另一個來自廣播。您可以透過使用 `toOthers` 方法來指示廣播器不要將事件廣播給當前使用者，從而解決此問題。

> [!WARNING]
> 您的事件必須使用 `Illuminate\Broadcasting\InteractsWithSockets` trait 才能呼叫 `toOthers` 方法。

<a name="only-to-others-configuration"></a>
#### 設定

當您初始化 Laravel Echo 實例時，系統會為該連線分配一個 Socket ID。如果您使用全域的 [Axios](https://github.com/axios/axios) 實例從 JavaScript 應用程式發送 HTTP 請求，該 Socket ID 將作為 `X-Socket-ID` 標頭自動附加到每個發出的請求中。然後，當您呼叫 `toOthers` 方法時，Laravel 會從標頭中擷取 Socket ID，並指示廣播器不要廣播給具有該 Socket ID 的任何連線。

如果您沒有使用全域的 Axios 實例，您需要手動設定 JavaScript 應用程式，在所有發出的請求中帶上 `X-Socket-ID` 標頭。您可以透過 `Echo.socketId` 方法取得 Socket ID：

```js
var socketId = Echo.socketId();
```

<a name="customizing-the-connection"></a>
### 自訂連線

如果您的應用程式會與多個廣播連線互動，且您希望使用預設廣播器之外的其他廣播器來廣播事件，您可以使用 `via` 方法來指定要將事件推送至哪個連線：

```php
use App\Events\OrderShipmentStatusUpdated;

broadcast(new OrderShipmentStatusUpdated($update))->via('pusher');
```

或者，您可以透過在事件的建構子中呼叫 `broadcastVia` 方法來指定事件的廣播連線。然而在這樣做之前，您應該確保事件類別使用了 `InteractsWithBroadcasting` trait：

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

有時，您可能希望向應用程式的前端廣播簡單事件，而不需要建立專用的事件類別。為滿足此需求，`Broadcast` Facade 允許您廣播「匿名事件」：

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

使用 `as` 和 `with` 方法，您可以自訂事件的名稱與資料：

```php
Broadcast::on('orders.'.$order->id)
    ->as('OrderPlaced')
    ->with($order)
    ->send();
```

上述範例將廣播類似以下的事件：

```json
{
    "event": "OrderPlaced",
    "data": "{ id: 1, total: 100 }",
    "channel": "orders.1"
}
```

如果您想要在私有頻道或 Presence 頻道上廣播匿名事件，您可以使用 `private` 和 `presence` 方法：

```php
Broadcast::private('orders.'.$order->id)->send();
Broadcast::presence('channels.'.$channel->id)->send();
```

使用 `send` 方法廣播匿名事件會將事件分派到您應用程式的[佇列](/docs/{{version}}/queues)中進行處理。然而，如果您想要立即廣播事件，可以使用 `sendNow` 方法：

```php
Broadcast::on('orders.'.$order->id)->sendNow();
```

要將事件廣播給頻道的所有訂閱者（當前已通過認證的使用者除外），您可以呼叫 `toOthers` 方法：

```php
Broadcast::on('orders.'.$order->id)
    ->toOthers()
    ->send();
```

<a name="rescuing-broadcasts"></a>
### 救援廣播

當您的應用程式佇列伺服器不可用或 Laravel 在廣播事件時遇到錯誤，系統會拋出例外，通常這會導致終端使用者看到應用程式錯誤。由於事件廣播通常是應用程式核心功能的補充，您可以透過在事件上實作 `ShouldRescue` 介面來防止這些例外影響使用者體驗。

實作 `ShouldRescue` 介面的事件在嘗試廣播期間會自動使用 Laravel 的 [rescue 輔助函式](/docs/{{version}}/helpers#method-rescue)。此輔助函式會擷取任何例外、將其回報給應用程式的例外處理器以進行記錄，並允許應用程式繼續正常執行，不會中斷使用者的工作流程：

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

當您[安裝並實例化 Laravel Echo](#client-side-installation) 後，就可以開始監聽從 Laravel 應用程式廣播的事件了。首先，使用 `channel` 方法來取得頻道的實例，然後呼叫 `listen` 方法來監聽指定的事件：

```js
Echo.channel(`orders.${this.order.id}`)
    .listen('OrderShipmentStatusUpdated', (e) => {
        console.log(e.order.name);
    });
```

如果您想在私有頻道上監聽事件，請改用 `private` 方法。您可以繼續鏈結呼叫 `listen` 方法，以在單一頻道上監聽多個事件：

```js
Echo.private(`orders.${this.order.id}`)
    .listen(/* ... */)
    .listen(/* ... */)
    .listen(/* ... */);
```

<a name="stop-listening-for-events"></a>
#### 停止監聽事件

如果您想在不[離開頻道](#leaving-a-channel)的情況下停止監聽特定事件，可以使用 `stopListening` 方法：

```js
Echo.private(`orders.${this.order.id}`)
    .stopListening('OrderShipmentStatusUpdated');
```

<a name="leaving-a-channel"></a>
### 離開頻道

若要離開頻道，可以在 Echo 實例上呼叫 `leaveChannel` 方法：

```js
Echo.leaveChannel(`orders.${this.order.id}`);
```

如果您想離開某個頻道以及其相關的私有頻道與 presence 頻道，可以呼叫 `leave` 方法：

```js
Echo.leave(`orders.${this.order.id}`);
```

<a name="namespaces"></a>
### 命名空間

您可能已經注意到在上述範例中，我們沒有為事件類別指定完整的 `App\Events` 命名空間。這是因為 Echo 會自動假設事件地位於 `App\Events` 命名空間中。然而，您可以在實例化 Echo 時傳遞 `namespace` 設定選項來指定根命名空間：

```js
window.Echo = new Echo({
    broadcaster: 'pusher',
    // ...
    namespace: 'App.Other.Namespace'
});
```

或者，在使用 Echo 訂閱事件時，可以在事件類別前面加上 `.` 前綴。這將允許您傳入完整的類別名稱：

```js
Echo.channel('orders')
    .listen('.Namespace\\Event\\Class', (e) => {
        // ...
    });
```

<a name="using-react-or-vue"></a>
### 使用 React、Vue 或 Svelte

Laravel Echo 包含了 React、Vue 及 Svelte hook，讓您可以輕鬆地監聽事件。若要開始使用，請呼叫用於監聽私有事件的 `useEcho` hook。當使用該 hook 的元件卸載時，`useEcho` hook 將會自動離開頻道：

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

您也可以透過向 `useEcho` 提供一個事件陣列來監聽多個事件：

```js
useEcho(
    `orders.${orderId}`,
    ["OrderShipmentStatusUpdated", "OrderShipped"],
    (e) => {
        console.log(e.order);
    },
);
```

您還可以指定廣播事件負載 (payload) 資料的結構形狀，從而提供更高的型別安全性與編輯便利性：

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

當使用該 hook 的元件卸載時，`useEcho` hook 會自動離開頻道；不過，如果有需要，您也可以利用回傳的函式，透過程式碼手動停止/開始監聽頻道：

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

若要連線至公開頻道，您可以使用 `useEchoPublic` hook：

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
#### 連線至 Presence 頻道

若要連線至 Presence 頻道，您可以使用 `useEchoPresence` hook：

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

您可以使用 `useConnectionStatus` hook 來取得目前的 WebSocket 連線狀態，它提供了一個響應式 (reactive) 狀態，會在連線狀態變更時自動更新：

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

可能的狀態值為：

<div class="content-list" markdown="1">

- `connected` - 已成功連線至 WebSocket 伺服器。
- `connecting` - 正在進行首次連線嘗試。
- `reconnecting` - 斷線後正在嘗試重新連線。
- `disconnected` - 未連線且未嘗試重新連線。
- `failed` - 連線失敗且不會重試。

</div>


<a name="react-vue-socket-id"></a>
#### Socket ID

您可以使用 `useSocketId` hook 來取得目前的 WebSocket socket ID，它提供了一個響應式數值，當連線以新的 socket ID 重新連線時會自動更新：

```js tab=React
import { useSocketId } from "@laravel/echo-react";

function SocketIndicator() {
    const socketId = useSocketId();

    return <div>Socket ID: {socketId}</div>;
}
```

```vue tab=Vue
<script setup lang="ts">
import { useSocketId } from "@laravel/echo-vue";

const socketId = useSocketId();
</script>

<template>
    <div>Socket ID: {{ socketId }}</div>
</template>
```

```svelte tab=Svelte
<script>
import { useSocketId } from "@laravel/echo-svelte";

const socketId = useSocketId();
</script>

<div>Socket ID: {socketId()}</div>
```

<a name="presence-channels"></a>
## Presence 頻道

Presence 頻道建立在私人頻道的安全性之上，同時額外提供了感知「誰訂閱了該頻道」的功能。這讓你可以輕鬆打造強大且具協作功能的應用程式特色，例如當其他使用者正在瀏覽相同頁面時通知使用者，或是列出聊天室裡的成員。


<a name="authorizing-presence-channels"></a>
### 授權 Presence 頻道

所有 Presence 頻道同時也是私人頻道；因此，使用者必須[獲得存取授權](#authorizing-channels)。不過，在為 Presence 頻道定義授權回呼時，若使用者獲准加入頻道，你不需要回傳 `true`，而是應該回傳一個包含該使用者資料的陣列。

由授權回呼回傳的資料，將可以在 JavaScript 應用程式中的 Presence 頻道事件監聽器中使用。如果使用者未獲准加入 Presence 頻道，則應該回傳 `false` 或 `null`：

```php
use App\Models\User;

Broadcast::channel('chat.{roomId}', function (User $user, int $roomId) {
    if ($user->canJoinRoom($roomId)) {
        return ['id' => $user->id, 'name' => $user->name];
    }
});
```


<a name="joining-presence-channels"></a>
### 加入 Presence 頻道

若要加入 Presence 頻道，你可以使用 Echo 的 `join` 方法。`join` 方法會回傳一個 `PresenceChannel` 實作，除了提供 `listen` 方法外，還允許你訂閱 `here`、`joining` 和 `leaving` 事件。

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

當成功加入頻道時，`here` 回呼會立即執行，並接收一個包含目前所有已訂閱該頻道之其他使用者資訊的陣列。當新使用者加入頻道時，會執行 `joining` 方法；而當使用者離開頻道時，則會執行 `leaving` 方法。當認證端點回傳非 200 的 HTTP 狀態碼或解析回傳的 JSON 出現問題時，將會執行 `error` 方法。


<a name="broadcasting-to-presence-channels"></a>
### 廣播至 Presence 頻道

Presence 頻道可以像公開或私人頻道一樣接收事件。以聊天室為例，我們可能希望將 `NewMessage` 事件廣播到該聊天室的 Presence 頻道。為此，我們會在事件的 `broadcastOn` 方法中回傳一個 `PresenceChannel` 實例：

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

與其他事件一樣，你可以使用 `broadcast` 輔助函式和 `toOthers` 方法來排除目前的使用者接收該廣播：

```php
broadcast(new NewMessage($message));

broadcast(new NewMessage($message))->toOthers();
```

與其他類型的事件一樣，你可以使用 Echo 的 `listen` 方法來監聽發送到 Presence 頻道的事件：

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
## Model 廣播

> [!WARNING]
> 在閱讀以下關於 Model 廣播的說明文件之前，我們建議您先熟悉 Laravel Model 廣播服務的基本概念，以及如何手動建立與監聽廣播事件。

當應用程式的 [Eloquent Model](/docs/{{version}}/eloquent) 被建立、更新或刪除時，發送廣播事件是非常常見的做法。當然，這可以透過手動[為 Eloquent Model 的狀態變更定義自訂事件](/docs/{{version}}/eloquent#events)，並讓這些事件實作 `ShouldBroadcast` 介面來輕鬆實現。

然而，如果您在應用程式中沒有將這些事件用於其他用途，單純為了廣播而建立事件類別可能會顯得繁瑣。為了補救這一點，Laravel 允許您指定 Eloquent Model 自動廣播其狀態變更。

首先，您的 Eloquent Model 應該使用 `Illuminate\Database\Eloquent\BroadcastsEvents` trait。此外，Model 應定義一個 `broadcastOn` 方法，該方法將回傳一個 Channel 陣列，指定 Model 事件應廣播至哪些 Channel：

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

一旦您的 Model 包含了這個 trait 並定義了其廣播 Channel，當 Model 實例被建立、更新、刪除、移至回收桶（trashed）或還原時，它就會開始自動廣播事件。

此外，您可能已經注意到 `broadcastOn` 方法接收一個字串型別的 `$event` 引數。這個引數包含了 Model 上發生的事件類型，其值可能是 `created`、`updated`、`deleted`、`trashed` 或 `restored`。透過檢查此變數的值，您可以決定 Model 在特定事件發生時應該廣播到哪些 Channel（如果有的話）：

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
#### 自訂 Model 廣播事件的建立

有時，您可能希望自訂 Laravel 如何建立底層的 Model 廣播事件。您可以透過在 Eloquent Model 上定義 `newBroadcastableEvent` 方法來做到這一點。該方法應回傳一個 `Illuminate\Database\Eloquent\BroadcastableModelEventOccurred` 實例：

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
### Model 廣播慣例

<a name="model-broadcasting-channel-conventions"></a>
#### Channel 慣例

您可能已經注意到，上方 Model 範例中的 `broadcastOn` 方法並未回傳 `Channel` 實例，而是直接回傳了 Eloquent Model。如果您的 Model 的 `broadcastOn` 方法回傳了 Eloquent Model 實例（或包含在該方法回傳的陣列中），Laravel 將會使用該 Model 的類別名稱與主鍵識別碼作為 Channel 名稱，自動為該 Model 實例化一個私有 Channel 實例。

因此，`id` 為 `1` 的 `App\Models\User` Model 將會被轉換為名稱為 `App.Models.User.1` 的 `Illuminate\Broadcasting\PrivateChannel` 實例。當然，除了從 Model 的 `broadcastOn` 方法回傳 Eloquent Model 實例之外，您也可以回傳完整的 `Channel` 實例，以便完全掌控 Model 的 Channel 名稱：

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

如果您打算從 Model 的 `broadcastOn` 方法顯式回傳一個 Channel 實例，您可以將 Eloquent Model 實例傳遞給 Channel 的建構函式。當這樣做時，Laravel 將使用上述討論的 Model Channel 慣例將 Eloquent Model 轉換為 Channel 名稱字串：

```php
return [new Channel($this->user)];
```

如果您需要取得某個 Model 的 Channel 名稱，可以對任何 Model 實例呼叫 `broadcastChannel` 方法。例如，對於 `id` 為 `1` 的 `App\Models\User` Model，此方法會回傳字串 `App.Models.User.1`：

```php
$user->broadcastChannel();
```

<a name="model-broadcasting-event-conventions"></a>
#### 事件慣例

由於 Model 廣播事件並未與應用程式 `App\Events` 目錄下的「實際」事件相關聯，因此會根據慣例指派名稱與有效載荷（payload）。Laravel 的慣例是使用 Model 的類別名稱（不包含命名空間）加上觸發廣播的 Model 事件名稱來廣播該事件。

因此，舉例來說，更新 `App\Models\Post` Model 將會向您的客戶端應用程式廣播一個名為 `PostUpdated` 的事件，其有效載荷如下：

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

刪除 `App\Models\User` Model 則會廣播一個名為 `UserDeleted` 的事件。

如果您願意，可以透過在 Model 中新增 `broadcastAs` 和 `broadcastWith` 方法來定義自訂的廣播名稱與有效載荷。這些方法會接收正在發生的 Model 事件 / 操作名稱，讓您可以為每個 Model 操作自訂事件名稱與有效載荷。如果從 `broadcastAs` 方法回傳 `null`，Laravel 將在廣播事件時使用上述討論的 Model 廣播事件名稱慣例：

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
### 監聽 Model 廣播

當你將 `BroadcastsEvents` trait 加入至你的 Model 並定義了 Model 的 `broadcastOn` 方法後，就準備好可以開始在客戶端應用程式中監聽廣播的 Model 事件了。在開始之前，你可能需要先參閱關於[監聽事件](#listening-for-events)的完整說明文件。

首先，使用 `private` 方法來取得頻道的實例，接著呼叫 `listen` 方法來監聽指定的事件。通常傳給 `private` 方法的頻道名稱應該要符合 Laravel 的 [Model 廣播慣例](#model-broadcasting-conventions)。

一旦取得頻道實例，你就可以使用 `listen` 方法來監聽特定事件。由於 Model 廣播事件並未與應用程式 `App\Events` 目錄中的「實際」事件產生關聯，因此[事件名稱](#model-broadcasting-event-conventions)前面必須加上前綴 `.`，以表示它不屬於特定命名空間。每個 Model 廣播事件都有一個 `model` 屬性，其中包含該 Model 所有可廣播的屬性：

```js
Echo.private(`App.Models.User.${this.user.id}`)
    .listen('.UserUpdated', (e) => {
        console.log(e.model);
    });
```

<a name="model-broadcasts-with-react-or-vue"></a>
#### 使用 React、Vue 或 Svelte

如果你使用的是 React、Vue 或 Svelte，可以使用 Laravel Echo 內建的 `useEchoModel` hook 來輕鬆監聽 Model 廣播：

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

你也可以指定 Model 事件載荷（payload）資料的型態結構，以提供更高的型別安全性與編輯便利性：

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
> 當使用 [Pusher Channels](https://pusher.com/channels) 時，您必須在 [應用程式儀表板](https://dashboard.pusher.com/) 的「App Settings」區塊中啟用「Client Events」選項，才能發送客戶端事件。

有時候，您可能希望將事件廣播給其他已連線的客戶端，完全不經過您的 Laravel 應用程式。這對於「正在輸入」通知這類功能特別有用，例如當您想要提醒應用程式的使用者，另一位使用者正在特定畫面上輸入訊息時。

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

```svelte tab=Svelte
<script>
import { useEcho } from "@laravel/echo-svelte";

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

透過將事件廣播與[通知](/docs/{{version}}/notifications)結合，您的 JavaScript 應用程式可以在新通知發生時立即接收，而不需要重新整理頁面。在開始之前，請務必先閱讀使用[廣播通知頻道](/docs/{{version}}/notifications#broadcast-notifications)的相關文件。

一旦您將通知設定為使用廣播頻道後，您就可以使用 Echo 的 `notification` 方法來監聽廣播事件。請記住，頻道名稱應與接收通知實體的類別名稱相符合：

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

在此範例中，所有透過 `broadcast` 頻道發送到 `App\Models\User` 實體的通知都將由該回呼接收。您的應用程式 `routes/channels.php` 檔案中已經包含了一個針對 `App.Models.User.{id}` 頻道的頻道授權回呼。


<a name="stop-listening-for-notifications"></a>
#### 停止監聽通知

若您想要停止監聽通知但不想[離開頻道](#leaving-a-channel)，您可以使用 `stopListeningForNotification` 方法：

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