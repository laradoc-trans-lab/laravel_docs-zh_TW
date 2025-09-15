# Broadcasting

- [簡介](#introduction)
- [伺服器端安裝](#server-side-installation)
    - [設定](#configuration)
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
- [授權 Channel](#authorizing-channels)
    - [定義授權回呼](#defining-authorization-callbacks)
    - [定義 Channel 類別](#defining-channel-classes)
- [廣播事件](#broadcasting-events)
    - [僅限其他人](#only-to-others)
    - [自訂連線](#customizing-the-connection)
    - [匿名事件](#anonymous-events)
- [接收廣播](#receiving-broadcasts)
    - [監聽事件](#listening-for-events)
    - [離開 Channel](#leaving-a-channel)
    - [命名空間](#namespaces)
- [Presence Channel](#presence-channels)
    - [授權 Presence Channel](#authorizing-presence-channels)
    - [加入 Presence Channel](#joining-presence-channels)
    - [廣播至 Presence Channel](#broadcasting-to-presence-channels)
- [Model 廣播](#model-broadcasting)
    - [Model 廣播慣例](#model-broadcasting-conventions)
    - [監聽 Model 廣播](#listening-for-model-broadcasts)
- [客戶端事件](#client-events)
- [通知](#notifications)

<a name="introduction"></a>
## 簡介

在許多現代 Web 應用程式中，WebSockets 被用於實作即時、動態更新的使用者介面。當伺服器上的某些資料更新時，通常會透過 WebSocket 連線發送訊息，由客戶端處理。WebSockets 提供了一種更有效率的替代方案，以取代持續輪詢應用程式伺服器以獲取應反映在 UI 中的資料變更。

舉例來說，假設您的應用程式能夠將使用者的資料匯出為 CSV 檔案並透過電子郵件發送給他們。然而，建立此 CSV 檔案需要數分鐘，因此您選擇在一個 [佇列任務](/docs/{{version}}/queues) 中建立並郵寄 CSV。當 CSV 建立並郵寄給使用者後，我們可以使用事件廣播來分派一個 `App\Events\UserDataExported` 事件，該事件由我們應用程式的 JavaScript 接收。一旦事件被接收，我們就可以向使用者顯示一條訊息，告知他們 CSV 已透過電子郵件發送給他們，而無需他們重新整理頁面。

為了協助您建構這類功能，Laravel 讓您能夠輕鬆地透過 WebSocket 連線「廣播」您的伺服器端 Laravel [事件](/docs/{{version}}/events)。廣播您的 Laravel 事件允許您在伺服器端 Laravel 應用程式和客戶端 JavaScript 應用程式之間共用相同的事件名稱和資料。

廣播的核心概念很簡單：客戶端在前端連線到具名 Channel，而您的 Laravel 應用程式則在後端將事件廣播到這些 Channel。這些事件可以包含您希望提供給前端的任何額外資料。

<a name="supported-drivers"></a>
#### 支援的 Driver

預設情況下，Laravel 包含三個伺服器端廣播 Driver 供您選擇：[Laravel Reverb](https://reverb.laravel.com)、[Pusher Channels](https://pusher.com/channels) 和 [Ably](https://ably.com)。

> [!NOTE]
> 在深入了解事件廣播之前，請務必閱讀 Laravel 關於 [事件與監聽器](/docs/{{version}}/events) 的說明文件。

<a name="server-side-installation"></a>
## 伺服器端安裝

要開始使用 Laravel 的事件廣播，我們需要在 Laravel 應用程式中進行一些設定，並安裝一些套件。

事件廣播是透過伺服器端廣播 Driver 完成的，該 Driver 會廣播您的 Laravel 事件，以便 Laravel Echo (一個 JavaScript 函式庫) 可以在瀏覽器客戶端中接收它們。別擔心 — 我們將逐步引導您完成安裝過程的每個部分。

<a name="configuration"></a>
### 設定

您應用程式的所有事件廣播設定都儲存在 `config/broadcasting.php` 設定檔中。如果此目錄在您的應用程式中不存在，請不要擔心；它將在您執行 `install:broadcasting` Artisan 命令時建立。

Laravel 預設支援多種廣播 Driver：[Laravel Reverb](/docs/{{version}}/reverb)、[Pusher Channels](https://pusher.com/channels)、[Ably](https://ably.com)，以及用於本地開發和偵錯的 `log` Driver。此外，還包含一個 `null` Driver，允許您在測試期間停用廣播。`config/broadcasting.php` 設定檔中包含了每個 Driver 的設定範例，讓您可以快速指定您的金鑰、密鑰和應用程式 ID。

<a name="installation"></a>
#### 安裝

預設情況下，新 Laravel 應用程式中未啟用廣播。您可以使用 `install:broadcasting` Artisan 命令啟用廣播：

```shell
php artisan install:broadcasting
```

`install:broadcasting` 命令將建立 `config/broadcasting.php` 設定檔。此外，該命令還將建立 `routes/channels.php` 檔案，您可以在其中註冊應用程式的廣播授權路由和回呼。

<a name="queue-configuration"></a>
#### 佇列設定

在廣播任何事件之前，您應該先設定並執行一個 [佇列 Worker](/docs/{{version}}/queues)。所有事件廣播都是透過佇列任務完成的，這樣您的應用程式回應時間不會因事件廣播而受到嚴重影響。

<a name="reverb"></a>
### Reverb

執行 `install:broadcasting` 命令時，系統會提示您安裝 [Laravel Reverb](/docs/{{version}}/reverb)。當然，您也可以使用 Composer 套件管理器手動安裝 Reverb。

```sh
composer require laravel/reverb
```

套件安裝完成後，您可以執行 Reverb 的安裝命令來發布設定、新增 Reverb 所需的環境變數，並在您的應用程式中啟用事件廣播：

```sh
php artisan reverb:install
```

您可以在 [Reverb 說明文件](/docs/{{version}}/reverb) 中找到詳細的 Reverb 安裝和使用說明。

<a name="pusher-channels"></a>
### Pusher Channels

如果您打算使用 [Pusher Channels](https://pusher.com/channels) 廣播事件，您應該使用 Composer 套件管理器安裝 Pusher Channels PHP SDK：

```shell
composer require pusher/pusher-php-server
```

接下來，您應該在 `config/broadcasting.php` 設定檔中設定您的 Pusher Channels 憑證。此檔案中已包含一個 Pusher Channels 設定範例，讓您可以快速指定您的金鑰、密鑰和應用程式 ID。通常，您應該在應用程式的 `.env` 檔案中設定您的 Pusher Channels 憑證：

```ini
PUSHER_APP_ID="your-pusher-app-id"
PUSHER_APP_KEY="your-pusher-key"
PUSHER_APP_SECRET="your-pusher-secret"
PUSHER_HOST=
PUSHER_PORT=443
PUSHER_SCHEME="https"
PUSHER_APP_CLUSTER="mt1"
```

`config/broadcasting.php` 檔案的 `pusher` 設定也允許您指定 Channels 支援的其他 `options`，例如 Cluster。

然後，在您應用程式的 `.env` 檔案中將 `BROADCAST_CONNECTION` 環境變數設定為 `pusher`：

```ini
BROADCAST_CONNECTION=pusher
```

最後，您已準備好安裝和設定 [Laravel Echo](#client-side-installation)，它將在客戶端接收廣播事件。

<a name="ably"></a>
### Ably

> [!NOTE]
> 以下說明文件討論如何在「Pusher 相容模式」下使用 Ably。然而，Ably 團隊建議並維護一個廣播器和 Echo 客戶端，能夠利用 Ably 提供的獨特功能。有關使用 Ably 維護的 Driver 的更多資訊，請 [查閱 Ably 的 Laravel 廣播器說明文件](https://github.com/ably/laravel-broadcaster)。

如果您打算使用 [Ably](https://ably.com) 廣播事件，您應該使用 Composer 套件管理器安裝 Ably PHP SDK：

```shell
composer require ably/ably-php
```

接下來，您應該在 `config/broadcasting.php` 設定檔中設定您的 Ably 憑證。此檔案中已包含一個 Ably 設定範例，讓您可以快速指定您的金鑰。通常，此值應透過 `ABLY_KEY` [環境變數](/docs/{{version}}/configuration#environment-configuration) 設定：

```ini
ABLY_KEY=your-ably-key
```

然後，在您應用程式的 `.env` 檔案中將 `BROADCAST_CONNECTION` 環境變數設定為 `ably`：

```ini
BROADCAST_CONNECTION=ably
```

最後，您已準備好安裝和設定 [Laravel Echo](#client-side-installation)，它將在客戶端接收廣播事件。

<a name="client-side-installation"></a>
## 客戶端安裝

<a name="client-reverb"></a>
### Reverb

[Laravel Echo](https://github.com/laravel/echo) 是一個 JavaScript 函式庫，讓訂閱 Channel 和監聽伺服器端廣播 Driver 廣播的事件變得輕而易舉。您可以透過 NPM 套件管理器安裝 Echo。在此範例中，我們還將安裝 `pusher-js` 套件，因為 Reverb 利用 Pusher 協定進行 WebSocket 訂閱、Channel 和訊息：

```shell
npm install --save-dev laravel-echo pusher-js
```

Echo 安裝完成後，您就可以在應用程式的 JavaScript 中建立一個新的 Echo 實例。一個很好的位置是在 Laravel 框架隨附的 `resources/js/bootstrap.js` 檔案底部。預設情況下，此檔案中已包含一個 Echo 設定範例 — 您只需取消註解並將 `broadcaster` 設定選項更新為 `reverb`：

```js
import Echo from 'laravel-echo';

import Pusher from 'pusher-js';
window.Pusher = Pusher;

window.Echo = new Echo({
    broadcaster: 'reverb',
    key: import.meta.env.VITE_REVERB_APP_KEY,
    wsHost: import.meta.env.VITE_REVERB_HOST,
    wsPort: import.meta.env.VITE_REVERB_PORT,
    wssPort: import.meta.env.VITE_REVERB_PORT,
    forceTLS: (import.meta.env.VITE_REVERB_SCHEME ?? 'https') === 'https',
    enabledTransports: ['ws', 'wss'],
});
```

接下來，您應該編譯應用程式的資產：

```shell
npm run build
```

> [!WARNING]
> Laravel Echo `reverb` 廣播器需要 laravel-echo v1.16.0+。

<a name="client-pusher-channels"></a>
### Pusher Channels

[Laravel Echo](https://github.com/laravel/echo) 是一個 JavaScript 函式庫，讓訂閱 Channel 和監聽伺服器端廣播 Driver 廣播的事件變得輕而易舉。Echo 也利用 `pusher-js` NPM 套件來實作 Pusher 協定，用於 WebSocket 訂閱、Channel 和訊息。

`install:broadcasting` Artisan 命令會自動為您安裝 `laravel-echo` 和 `pusher-js` 套件；但是，您也可以透過 NPM 手動安裝這些套件：

```shell
npm install --save-dev laravel-echo pusher-js
```

Echo 安裝完成後，您就可以在應用程式的 JavaScript 中建立一個新的 Echo 實例。`install:broadcasting` 命令會在 `resources/js/echo.js` 建立一個 Echo 設定檔；但是，此檔案中的預設設定是針對 Laravel Reverb。您可以複製以下設定以將您的設定轉換為 Pusher：

```js
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

接下來，您應該在應用程式的 `.env` 檔案中定義 Pusher 環境變數的適當值。如果這些變數尚未存在於您的 `.env` 檔案中，您應該新增它們：

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

一旦您根據應用程式的需求調整了 Echo 設定，您就可以編譯應用程式的資產：

```shell
npm run build
```

> [!NOTE]
> 要了解有關編譯應用程式 JavaScript 資產的更多資訊，請查閱 [Vite](/docs/{{version}}/vite) 上的說明文件。

<a name="using-an-existing-client-instance"></a>
#### 使用現有的客戶端實例

如果您已經有一個預先設定好的 Pusher Channels 客戶端實例，並且希望 Echo 使用它，您可以透過 `client` 設定選項將其傳遞給 Echo：

```js
import Echo from 'laravel-echo';
import Pusher from 'pusher-js';

const options = {
    broadcaster: 'pusher',
    key: 'your-pusher-channels-key'
}

window.Echo = new Echo({
    ...options,
    client: new Pusher(options.key, options)
});
```

<a name="client-ably"></a>
### Ably

> [!NOTE]
> 以下說明文件討論如何在「Pusher 相容模式」下使用 Ably。然而，Ably 團隊建議並維護一個廣播器和 Echo 客戶端，能夠利用 Ably 提供的獨特功能。有關使用 Ably 維護的 Driver 的更多資訊，請 [查閱 Ably 的 Laravel 廣播器說明文件](https://github.com/ably/laravel-broadcaster)。

[Laravel Echo](https://github.com/laravel/echo) 是一個 JavaScript 函式庫，讓訂閱 Channel 和監聽伺服器端廣播 Driver 廣播的事件變得輕而易舉。Echo 也利用 `pusher-js` NPM 套件來實作 Pusher 協定，用於 WebSocket 訂閱、Channel 和訊息。

`install:broadcasting` Artisan 命令會自動為您安裝 `laravel-echo` 和 `pusher-js` 套件；但是，您也可以透過 NPM 手動安裝這些套件：

```shell
npm install --save-dev laravel-echo pusher-js
```

**在繼續之前，您應該在 Ably 應用程式設定中啟用 Pusher 協定支援。您可以在 Ably 應用程式設定儀表板的「Protocol Adapter Settings」部分中啟用此功能。**

Echo 安裝完成後，您就可以在應用程式的 JavaScript 中建立一個新的 Echo 實例。`install:broadcasting` 命令會在 `resources/js/echo.js` 建立一個 Echo 設定檔；但是，此檔案中的預設設定是針對 Laravel Reverb。您可以複製以下設定以將您的設定轉換為 Ably：

```js
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

您可能已經注意到我們的 Ably Echo 設定引用了一個 `VITE_ABLY_PUBLIC_KEY` 環境變數。此變數的值應該是您的 Ably 公開金鑰。您的公開金鑰是您的 Ably 金鑰中 `:` 字元之前的部分。

一旦您根據需求調整了 Echo 設定，您就可以編譯應用程式的資產：

```shell
npm run dev
```

> [!NOTE]
> 要了解有關編譯應用程式 JavaScript 資產的更多資訊，請查閱 [Vite](/docs/{{version}}/vite) 上的說明文件。

<a name="concept-overview"></a>
## 概念概覽

Laravel 的事件廣播允許您使用基於 Driver 的 WebSocket 方法，將伺服器端 Laravel 事件廣播到客戶端 JavaScript 應用程式。目前，Laravel 隨附 [Pusher Channels](https://pusher.com/channels) 和 [Ably](https://ably.com) Driver。這些事件可以使用 [Laravel Echo](#client-side-installation) JavaScript 套件在客戶端輕鬆消費。

事件透過「Channel」廣播，這些 Channel 可以指定為公開或私有。應用程式的任何訪客都可以訂閱公開 Channel，無需任何身份驗證或授權；但是，為了訂閱私有 Channel，使用者必須經過身份驗證並被授權監聽該 Channel。

<a name="using-example-application"></a>
### 使用範例應用程式

在深入了解事件廣播的每個元件之前，讓我們先以一個電子商務商店為例，進行高層次的概覽。

在我們的應用程式中，假設我們有一個頁面允許使用者查看其訂單的運送狀態。我們也假設當應用程式處理運送狀態更新時，會觸發一個 `OrderShipmentStatusUpdated` 事件：

    use App\Events\OrderShipmentStatusUpdated;

    OrderShipmentStatusUpdated::dispatch($order);

<a name="the-shouldbroadcast-interface"></a>
#### `ShouldBroadcast` 介面

當使用者正在查看他們的訂單時，我們不希望他們必須重新整理頁面才能查看狀態更新。相反，我們希望在更新建立時將其廣播到應用程式。因此，我們需要使用 `ShouldBroadcast` 介面標記 `OrderShipmentStatusUpdated` 事件。這將指示 Laravel 在事件觸發時廣播該事件：

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

`ShouldBroadcast` 介面要求我們的事件定義一個 `broadcastOn` 方法。此方法負責回傳事件應廣播到的 Channel。此方法的空存根已在生成的事件類別上定義，因此我們只需填寫其詳細資訊。我們只希望訂單的建立者能夠查看狀態更新，因此我們將在與訂單綁定的私有 Channel 上廣播事件：

    use Illuminate\Broadcasting\Channel;
    use Illuminate\Broadcasting\PrivateChannel;

    /**
     * Get the channel the event should broadcast on.
     */
    public function broadcastOn(): Channel
    {
        return new PrivateChannel('orders.'.$this->order->id);
    }

如果您希望事件廣播到多個 Channel，您可以回傳一個 `array`：

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

<a name="example-application-authorizing-channels"></a>
#### 授權 Channel

請記住，使用者必須被授權才能監聽私有 Channel。我們可以在應用程式的 `routes/channels.php` 檔案中定義 Channel 授權規則。在此範例中，我們需要驗證任何嘗試監聽私有 `orders.1` Channel 的使用者確實是該訂單的建立者：

    use App\Models\Order;
    use App\Models\User;

    Broadcast::channel('orders.{orderId}', function (User $user, int $orderId) {
        return $user->id === Order::findOrNew($orderId)->user_id;
    });

`channel` 方法接受兩個引數：Channel 的名稱和一個回呼，該回呼回傳 `true` 或 `false`，表示使用者是否被授權監聽該 Channel。

所有授權回呼都會將目前已驗證的使用者作為第一個引數，並將任何額外的萬用字元參數作為後續引數。在此範例中，我們使用 `{orderId}` 佔位符來表示 Channel 名稱的「ID」部分是萬用字元。

<a name="listening-for-event-broadcasts"></a>
#### 監聽事件廣播

接下來，剩下的就是在我們的 JavaScript 應用程式中監聽事件。我們可以使用 [Laravel Echo](#client-side-installation) 來完成此操作。首先，我們將使用 `private` 方法訂閱私有 Channel。然後，我們可以使用 `listen` 方法監聽 `OrderShipmentStatusUpdated` 事件。預設情況下，事件的所有公開屬性都將包含在廣播事件中：

```js
Echo.private(`orders.${orderId}`)
    .listen('OrderShipmentStatusUpdated', (e) => {
        console.log(e.order);
    });
```

<a name="defining-broadcast-events"></a>
## 定義廣播事件

為了通知 Laravel 某個事件應該被廣播，您必須在事件類別上實作 `Illuminate\Contracts\Broadcasting\ShouldBroadcast` 介面。此介面已匯入框架生成的所有事件類別中，因此您可以輕鬆地將其新增到任何事件中。

`ShouldBroadcast` 介面要求您實作一個方法：`broadcastOn`。`broadcastOn` 方法應該回傳事件應廣播到的 Channel 或 Channel 陣列。這些 Channel 應該是 `Channel`、`PrivateChannel` 或 `PresenceChannel` 的實例。`Channel` 的實例代表任何使用者都可以訂閱的公開 Channel，而 `PrivateChannels` 和 `PresenceChannels` 則代表需要 [Channel 授權](#authorizing-channels) 的私有 Channel：

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

實作 `ShouldBroadcast` 介面後，您只需像往常一樣 [觸發事件](/docs/{{version}}/events)。事件觸發後，[佇列任務](/docs/{{version}}/queues) 將自動使用您指定的廣播 Driver 廣播事件。

<a name="broadcast-name"></a>
### 廣播名稱

預設情況下，Laravel 將使用事件的類別名稱廣播事件。但是，您可以透過在事件上定義 `broadcastAs` 方法來自訂廣播名稱：

    /**
     * The event's broadcast name.
     */
    public function broadcastAs(): string
    {
        return 'server.created';
    }

如果您使用 `broadcastAs` 方法自訂廣播名稱，您應該確保使用開頭的 `.` 字元註冊您的監聽器。這將指示 Echo 不要將應用程式的命名空間前置到事件中：

    .listen('.server.created', function (e) {
        ....
    });

<a name="broadcast-data"></a>
### 廣播資料

當事件被廣播時，其所有 `public` 屬性都會自動序列化並作為事件的 Payload 廣播，讓您可以從 JavaScript 應用程式存取其任何公開資料。因此，舉例來說，如果您的事件有一個包含 Eloquent Model 的單一公開 `$user` 屬性，則事件的廣播 Payload 將會是：

```json
{
    "user": {
        "id": 1,
        "name": "Patrick Stewart"
        ...
    }
}
```

但是，如果您希望對廣播 Payload 進行更精細的控制，您可以為事件新增一個 `broadcastWith` 方法。此方法應回傳您希望作為事件 Payload 廣播的資料陣列：

    /**
     * Get the data to broadcast.
     *
     * @return array<string, mixed>
     */
    public function broadcastWith(): array
    {
        return ['id' => $this->user->id];
    }

<a name="broadcast-queue"></a>
### 廣播佇列

預設情況下，每個廣播事件都會放置在 `queue.php` 設定檔中指定的預設佇列連線的預設佇列上。您可以透過在事件類別上定義 `connection` 和 `queue` 屬性來自訂廣播器使用的佇列連線和名稱：

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

或者，您可以透過在事件上定義 `broadcastQueue` 方法來自訂佇列名稱：

    /**
     * The name of the queue on which to place the broadcasting job.
     */
    public function broadcastQueue(): string
    {
        return 'default';
    }

如果您希望使用 `sync` 佇列而不是預設佇列 Driver 廣播事件，您可以實作 `ShouldBroadcastNow` 介面而不是 `ShouldBroadcast`：

    <?php

    use Illuminate\Contracts\Broadcasting\ShouldBroadcastNow;

    class OrderShipmentStatusUpdated implements ShouldBroadcastNow
    {
        // ...
    }

<a name="broadcast-conditions"></a>
### 廣播條件

有時您希望僅在給定條件為真時才廣播事件。您可以透過在事件類別中新增 `broadcastWhen` 方法來定義這些條件：

    /**
     * Determine if this event should broadcast.
     */
    public function broadcastWhen(): bool
    {
        return $this->order->value > 100;
    }

<a name="broadcasting-and-database-transactions"></a>
#### 廣播與資料庫交易

當廣播事件在資料庫交易中分派時，它們可能會在資料庫交易提交之前由佇列處理。發生這種情況時，您在資料庫交易期間對 Model 或資料庫記錄所做的任何更新可能尚未反映在資料庫中。此外，在交易中建立的任何 Model 或資料庫記錄可能不存在於資料庫中。如果您的事件依賴於這些 Model，則在處理廣播事件的任務時可能會發生意外錯誤。

如果您的佇列連線的 `after_commit` 設定選項設定為 `false`，您仍然可以透過在事件類別上實作 `ShouldDispatchAfterCommit` 介面來指示特定廣播事件應在所有開啟的資料庫交易提交後分派：

    <?php

    namespace App\Events;

    use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
    use Illuminate\Contracts\Events\ShouldDispatchAfterCommit;
    use Illuminate\Queue\SerializesModels;

    class ServerCreated implements ShouldBroadcast, ShouldDispatchAfterCommit
    {
        use SerializesModels;
    }

> [!NOTE]
> 要了解有關解決這些問題的更多資訊，請查閱有關 [佇列任務和資料庫交易](/docs/{{version}}/queues#jobs-and-database-transactions) 的說明文件。

<a name="authorizing-channels"></a>
## 授權 Channel

私有 Channel 要求您授權目前已驗證的使用者確實可以監聽該 Channel。這是透過向您的 Laravel 應用程式發出 HTTP 請求，並附帶 Channel 名稱，並允許您的應用程式判斷使用者是否可以監聽該 Channel 來完成的。使用 [Laravel Echo](#client-side-installation) 時，授權訂閱私有 Channel 的 HTTP 請求將自動發出。

啟用廣播後，Laravel 會自動註冊 `/broadcasting/auth` 路由來處理授權請求。`/broadcasting/auth` 路由會自動放置在 `web` Middleware 群組中。

<a name="defining-authorization-callbacks"></a>
### 定義授權回呼

接下來，我們需要定義實際判斷目前已驗證使用者是否可以監聽給定 Channel 的邏輯。這是在 `install:broadcasting` Artisan 命令建立的 `routes/channels.php` 檔案中完成的。在此檔案中，您可以使用 `Broadcast::channel` 方法註冊 Channel 授權回呼：

    use App\Models\User;

    Broadcast::channel('orders.{orderId}', function (User $user, int $orderId) {
        return $user->id === Order::findOrNew($orderId)->user_id;
    });

`channel` 方法接受兩個引數：Channel 的名稱和一個回呼，該回呼回傳 `true` 或 `false`，表示使用者是否被授權監聽該 Channel。

所有授權回呼都會將目前已驗證的使用者作為第一個引數，並將任何額外的萬用字元參數作為後續引數。在此範例中，我們使用 `{orderId}` 佔位符來表示 Channel 名稱的「ID」部分是萬用字元。

您可以使用 `channel:list` Artisan 命令查看應用程式的廣播授權回呼列表：

```shell
php artisan channel:list
```

<a name="authorization-callback-model-binding"></a>
#### 授權回呼 Model Binding

就像 HTTP 路由一樣，Channel 路由也可以利用隱式和顯式 [路由 Model Binding](/docs/{{version}}/routing#route-model-binding)。例如，您可以請求實際的 `Order` Model 實例，而不是接收字串或數字訂單 ID：

    use App\Models\Order;
    use App\Models\User;

    Broadcast::channel('orders.{order}', function (User $user, Order $order) {
        return $user->id === $order->user_id;
    });

> [!WARNING]
> 與 HTTP 路由 Model Binding 不同，Channel Model Binding 不支援自動 [隱式 Model Binding 範圍](/docs/{{version}}/routing#implicit-model-binding-scoping)。然而，這很少是問題，因為大多數 Channel 可以根據單一 Model 的唯一主鍵進行範圍設定。

<a name="authorization-callback-authentication"></a>
#### 授權回呼驗證

私有和 Presence 廣播 Channel 透過您應用程式的預設驗證 Guard 驗證目前使用者。如果使用者未經驗證，Channel 授權會自動拒絕，並且授權回呼永遠不會執行。但是，您可以指派多個自訂 Guard，以便在必要時驗證傳入的請求：

    Broadcast::channel('channel', function () {
        // ...
    }, ['guards' => ['web', 'admin']]);

<a name="defining-channel-classes"></a>
### 定義 Channel 類別

如果您的應用程式正在消費許多不同的 Channel，您的 `routes/channels.php` 檔案可能會變得臃腫。因此，您可以改用 Channel 類別來授權 Channel，而不是使用閉包。要生成 Channel 類別，請使用 `make:channel` Artisan 命令。此命令會將一個新的 Channel 類別放置在 `App/Broadcasting` 目錄中。

```shell
php artisan make:channel OrderChannel
```

接下來，在您的 `routes/channels.php` 檔案中註冊您的 Channel：

    use App\Broadcasting\OrderChannel;

    Broadcast::channel('orders.{order}', OrderChannel::class);

最後，您可以將 Channel 的授權邏輯放置在 Channel 類別的 `join` 方法中。此 `join` 方法將包含您通常會放置在 Channel 授權閉包中的相同邏輯。您也可以利用 Channel Model Binding：

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

> [!NOTE]
> 像 Laravel 中的許多其他類別一樣，Channel 類別將由 [Service Container](/docs/{{version}}/container) 自動解析。因此，您可以在其建構函式中型別提示 Channel 所需的任何依賴項。

<a name="broadcasting-events"></a>
## 廣播事件

一旦您定義了一個事件並使用 `ShouldBroadcast` 介面標記它，您只需使用事件的分派方法觸發該事件。事件分派器將注意到該事件已使用 `ShouldBroadcast` 介面標記，並將事件排入佇列以進行廣播：

    use App\Events\OrderShipmentStatusUpdated;

    OrderShipmentStatusUpdated::dispatch($order);

<a name="only-to-others"></a>
### 僅限其他人

在建構使用事件廣播的應用程式時，您可能偶爾需要將事件廣播給給定 Channel 的所有訂閱者，除了目前使用者。您可以使用 `broadcast` 輔助函式和 `toOthers` 方法來完成此操作：

    use App\Events\OrderShipmentStatusUpdated;

    broadcast(new OrderShipmentStatusUpdated($update))->toOthers();

為了更好地理解何時可能需要使用 `toOthers` 方法，讓我們想像一個任務列表應用程式，使用者可以透過輸入任務名稱來建立新任務。為了建立任務，您的應用程式可能會向 `/task` URL 發出請求，該 URL 會廣播任務的建立並回傳新任務的 JSON 表示。當您的 JavaScript 應用程式從端點接收到回應時，它可能會直接將新任務插入其任務列表，如下所示：

```js
axios.post('/task', task)
    .then((response) => {
        this.tasks.push(response.data);
    });
```

但是，請記住我們也廣播了任務的建立。如果您的 JavaScript 應用程式也正在監聽此事件以將任務新增到任務列表，您的列表中將會有重複的任務：一個來自端點，一個來自廣播。您可以透過使用 `toOthers` 方法來指示廣播器不要將事件廣播給目前使用者來解決此問題。

> [!WARNING]
> 您的事件必須使用 `Illuminate\Broadcasting\InteractsWithSockets` Trait 才能呼叫 `toOthers` 方法。

<a name="only-to-others-configuration"></a>
#### 設定

當您初始化 Laravel Echo 實例時，會為連線指派一個 Socket ID。如果您使用全域 [Axios](https://github.com/axios/axios) 實例從 JavaScript 應用程式發出 HTTP 請求，Socket ID 將自動作為 `X-Socket-ID` 標頭附加到每個傳出請求。然後，當您呼叫 `toOthers` 方法時，Laravel 將從標頭中提取 Socket ID，並指示廣播器不要廣播到具有該 Socket ID 的任何連線。

如果您沒有使用全域 Axios 實例，您將需要手動設定您的 JavaScript 應用程式以將 `X-Socket-ID` 標頭與所有傳出請求一起發送。您可以使用 `Echo.socketId` 方法檢索 Socket ID：

```js
var socketId = Echo.socketId();
```

<a name="customizing-the-connection"></a>
### 自訂連線

如果您的應用程式與多個廣播連線互動，並且您希望使用非預設的廣播器廣播事件，您可以使用 `via` 方法指定要將事件推送到哪個連線：

    use App\Events\OrderShipmentStatusUpdated;

    broadcast(new OrderShipmentStatusUpdated($update))->via('pusher');

或者，您可以在事件的建構函式中呼叫 `broadcastVia` 方法來指定事件的廣播連線。但是，在這樣做之前，您應該確保事件類別使用 `InteractsWithBroadcasting` Trait：

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

<a name="anonymous-events"></a>
### 匿名事件

有時，您可能希望將一個簡單的事件廣播到應用程式的前端，而無需建立專用的事件類別。為了實現這一點，`Broadcast` Facade 允許您廣播「匿名事件」：

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

使用 `as` 和 `with` 方法，您可以自訂事件的名稱和資料：

```php
Broadcast::on('orders.'.$order->id)
    ->as('OrderPlaced')
    ->with($order)
    ->send();
```

上面的範例將廣播一個類似以下的事件：

```json
{
    "event": "OrderPlaced",
    "data": "{ id: 1, total: 100 }",
    "channel": "orders.1"
}
```

如果您希望在私有或 Presence Channel 上廣播匿名事件，您可以使用 `private` 和 `presence` 方法：

```php
Broadcast::private('orders.'.$order->id)->send();
Broadcast::presence('channels.'.$channel->id)->send();
```

使用 `send` 方法廣播匿名事件會將事件分派到應用程式的 [佇列](/docs/{{version}}/queues) 進行處理。但是，如果您希望立即廣播事件，您可以使用 `sendNow` 方法：

```php
Broadcast::on('orders.'.$order->id)->sendNow();
```

要將事件廣播給所有 Channel 訂閱者，除了目前已驗證的使用者之外，您可以呼叫 `toOthers` 方法：

```php
Broadcast::on('orders.'.$order->id)
    ->toOthers()
    ->send();
```

<a name="receiving-broadcasts"></a>
## 接收廣播

<a name="listening-for-events"></a>
### 監聽事件

一旦您 [安裝並實例化 Laravel Echo](#client-side-installation)，您就可以開始監聽從 Laravel 應用程式廣播的事件。首先，使用 `channel` 方法檢索 Channel 實例，然後呼叫 `listen` 方法監聽指定的事件：

```js
Echo.channel(`orders.${this.order.id}`)
    .listen('OrderShipmentStatusUpdated', (e) => {
        console.log(e.order.name);
    });
```

如果您想監聽私有 Channel 上的事件，請改用 `private` 方法。您可以繼續鏈式呼叫 `listen` 方法，以在單一 Channel 上監聽多個事件：

```js
Echo.private(`orders.${this.order.id}`)
    .listen(/* ... */)
    .listen(/* ... */)
    .listen(/* ... */);
```

<a name="stop-listening-for-events"></a>
#### 停止監聽事件

如果您想停止監聽給定事件而無需 [離開 Channel](#leaving-a-channel)，您可以使用 `stopListening` 方法：

```js
Echo.private(`orders.${this.order.id}`)
    .stopListening('OrderShipmentStatusUpdated')
```

<a name="leaving-a-channel"></a>
### 離開 Channel

要離開 Channel，您可以在 Echo 實例上呼叫 `leaveChannel` 方法：

```js
Echo.leaveChannel(`orders.${this.order.id}`);
```

如果您想離開 Channel 及其相關的私有和 Presence Channel，您可以呼叫 `leave` 方法：

```js
Echo.leave(`orders.${this.order.id}`);
```
<a name="namespaces"></a>
### 命名空間

您可能已經注意到，在上面的範例中，我們沒有為事件類別指定完整的 `App\Events` 命名空間。這是因為 Echo 會自動假設事件位於 `App\Events` 命名空間中。但是，您可以在實例化 Echo 時透過傳遞 `namespace` 設定選項來設定根命名空間：

```js
window.Echo = new Echo({
    broadcaster: 'pusher',
    // ...
    namespace: 'App.Other.Namespace'
});
```

或者，您可以在使用 Echo 訂閱事件類別時，在事件類別前加上 `.`。這將允許您始終指定完全限定的類別名稱：

```js
Echo.channel('orders')
    .listen('.Namespace\\Event\\Class', (e) => {
        // ...
    });
```

<a name="presence-channels"></a>
## Presence Channel

Presence Channel 建立在私有 Channel 的安全性基礎上，同時公開了訂閱 Channel 的使用者感知功能。這使得建構強大、協作的應用程式功能變得容易，例如在另一個使用者正在查看同一頁面時通知使用者，或列出聊天室中的成員。

<a name="authorizing-presence-channels"></a>
### 授權 Presence Channel

所有 Presence Channel 也是私有 Channel；因此，使用者必須被 [授權才能存取它們](#authorizing-channels)。但是，在為 Presence Channel 定義授權回呼時，如果使用者被授權加入 Channel，您將不會回傳 `true`。相反，您應該回傳一個關於使用者資料的陣列。

授權回呼回傳的資料將提供給您的 JavaScript 應用程式中的 Presence Channel 事件監聽器。如果使用者未被授權加入 Presence Channel，您應該回傳 `false` 或 `null`：

    use App\Models\User;

    Broadcast::channel('chat.{roomId}', function (User $user, int $roomId) {
        if ($user->canJoinRoom($roomId)) {
            return ['id' => $user->id, 'name' => $user->name];
        }
    });

<a name="joining-presence-channels"></a>
### 加入 Presence Channel

要加入 Presence Channel，您可以使用 Echo 的 `join` 方法。`join` 方法將回傳一個 `PresenceChannel` 實作，除了公開 `listen` 方法外，還允許您訂閱 `here`、`joining` 和 `leaving` 事件。

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

`here` 回呼將在 Channel 成功加入後立即執行，並將接收一個陣列，其中包含目前訂閱該 Channel 的所有其他使用者的使用者資訊。`joining` 方法將在新使用者加入 Channel 時執行，而 `leaving` 方法將在使用者離開 Channel 時執行。`error` 方法將在驗證端點回傳非 200 的 HTTP 狀態碼或解析回傳的 JSON 時出現問題時執行。

<a name="broadcasting-to-presence-channels"></a>
### 廣播至 Presence Channel

Presence Channel 可以像公開或私有 Channel 一樣接收事件。以聊天室為例，我們可能希望將 `NewMessage` 事件廣播到聊天室的 Presence Channel。為此，我們將從事件的 `broadcastOn` 方法回傳 `PresenceChannel` 的實例：

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

與其他事件一樣，您可以使用 `broadcast` 輔助函式和 `toOthers` 方法來排除目前使用者接收廣播：

    broadcast(new NewMessage($message));

    broadcast(new NewMessage($message))->toOthers();

與其他類型的事件一樣，您可以使用 Echo 的 `listen` 方法監聽發送到 Presence Channel 的事件：

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
> 在閱讀以下有關 Model 廣播的說明文件之前，我們建議您熟悉 Laravel Model 廣播服務的一般概念，以及如何手動建立和監聽廣播事件。

當應用程式的 [Eloquent Model](/docs/{{version}}/eloquent) 被建立、更新或刪除時，廣播事件是很常見的。當然，這可以透過手動 [為 Eloquent Model 狀態變更定義自訂事件](/docs/{{version}}/eloquent#events) 並使用 `ShouldBroadcast` 介面標記這些事件來輕鬆完成。

但是，如果您的應用程式沒有將這些事件用於任何其他目的，那麼僅為了廣播而建立事件類別可能會很麻煩。為了解決這個問題，Laravel 允許您指示 Eloquent Model 自動廣播其狀態變更。

首先，您的 Eloquent Model 應該使用 `Illuminate\Database\Eloquent\BroadcastsEvents` Trait。此外，Model 應該定義一個 `broadcastOn` 方法，該方法將回傳 Model 事件應廣播到的 Channel 陣列：

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

一旦您的 Model 包含此 Trait 並定義其廣播 Channel，它將在 Model 實例被建立、更新、刪除、軟刪除或還原時自動廣播事件。

此外，您可能已經注意到 `broadcastOn` 方法接收一個字串 `$event` 引數。此引數包含 Model 上發生的事件類型，其值將為 `created`、`updated`、`deleted`、`trashed` 或 `restored`。透過檢查此變數的值，您可以確定 Model 應針對特定事件廣播到哪些 Channel (如果有的話)：

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
#### 自訂 Model 廣播事件建立

有時，您可能希望自訂 Laravel 建立底層 Model 廣播事件的方式。您可以透過在 Eloquent Model 上定義 `newBroadcastableEvent` 方法來完成此操作。此方法應回傳一個 `Illuminate\Database\Eloquent\BroadcastableModelEventOccurred` 實例：

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

您可能已經注意到，上面 Model 範例中的 `broadcastOn` 方法沒有回傳 `Channel` 實例。相反，直接回傳了 Eloquent Model。如果 Model 的 `broadcastOn` 方法回傳 Eloquent Model 實例 (或包含在該方法回傳的陣列中)，Laravel 將自動為該 Model 實例化一個私有 Channel 實例，並使用 Model 的類別名稱和主鍵識別碼作為 Channel 名稱。

因此，一個 `id` 為 `1` 的 `App\Models\User` Model 將被轉換為一個名稱為 `App.Models.User.1` 的 `Illuminate\Broadcasting\PrivateChannel` 實例。當然，除了從 Model 的 `broadcastOn` 方法回傳 Eloquent Model 實例外，您還可以回傳完整的 `Channel` 實例，以便完全控制 Model 的 Channel 名稱：

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

如果您打算從 Model 的 `broadcastOn` 方法中明確回傳 Channel 實例，您可以將 Eloquent Model 實例傳遞給 Channel 的建構函式。這樣做時，Laravel 將使用上面討論的 Model Channel 慣例將 Eloquent Model 轉換為 Channel 名稱字串：

```php
return [new Channel($this->user)];
```

如果您需要確定 Model 的 Channel 名稱，您可以在任何 Model 實例上呼叫 `broadcastChannel` 方法。例如，對於 `id` 為 `1` 的 `App\Models\User` Model，此方法回傳字串 `App.Models.User.1`：

```php
$user->broadcastChannel()
```

<a name="model-broadcasting-event-conventions"></a>
#### 事件慣例

由於 Model 廣播事件與應用程式 `App\Events` 目錄中的「實際」事件無關，因此它們會根據慣例被指派名稱和 Payload。Laravel 的慣例是使用 Model 的類別名稱 (不包括命名空間) 和觸發廣播的 Model 事件名稱來廣播事件。

因此，舉例來說，對 `App\Models\Post` Model 的更新將向客戶端應用程式廣播一個名為 `PostUpdated` 的事件，其 Payload 如下：

```json
{
    "model": {
        "id": 1,
        "title": "My first post"
        ...
    },
    ...
    "socket": "someSocketId",
}
```

`App\Models\User` Model 的刪除將廣播一個名為 `UserDeleted` 的事件。

如果您願意，您可以透過在 Model 中新增 `broadcastAs` 和 `broadcastWith` 方法來定義自訂廣播名稱和 Payload。這些方法接收正在發生的 Model 事件/操作的名稱，允許您為每個 Model 操作自訂事件的名稱和 Payload。如果 `broadcastAs` 方法回傳 `null`，Laravel 將在廣播事件時使用上面討論的 Model 廣播事件名稱慣例：

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

一旦您將 `BroadcastsEvents` Trait 新增到您的 Model 並定義了 Model 的 `broadcastOn` 方法，您就可以開始在客戶端應用程式中監聽廣播的 Model 事件。在開始之前，您可能希望查閱有關 [監聽事件](#listening-for-events) 的完整說明文件。

首先，使用 `private` 方法檢索 Channel 實例，然後呼叫 `listen` 方法監聽指定的事件。通常，傳遞給 `private` 方法的 Channel 名稱應與 Laravel 的 [Model 廣播慣例](#model-broadcasting-conventions) 相符。

一旦您獲得了 Channel 實例，您就可以使用 `listen` 方法監聽特定事件。由於 Model 廣播事件與應用程式 `App\Events` 目錄中的「實際」事件無關，因此 [事件名稱](#model-broadcasting-event-conventions) 必須以 `.` 為前綴，以表示它不屬於特定命名空間。每個 Model 廣播事件都有一個 `model` 屬性，其中包含 Model 的所有可廣播屬性：

```js
Echo.private(`App.Models.User.${this.user.id}`)
    .listen('.PostUpdated', (e) => {
        console.log(e.model);
    });
```

<a name="client-events"></a>
## 客戶端事件

> [!NOTE]
> 使用 [Pusher Channels](https://pusher.com/channels) 時，您必須在 [應用程式儀表板](https://dashboard.pusher.com/) 的「App Settings」部分中啟用「Client Events」選項，才能發送客戶端事件。

有時您可能希望將事件廣播給其他已連線的客戶端，而無需觸及您的 Laravel 應用程式。這對於「正在輸入」通知等功能特別有用，您希望提醒應用程式使用者，另一個使用者正在給定螢幕上輸入訊息。

要廣播客戶端事件，您可以使用 Echo 的 `whisper` 方法：

```js
Echo.private(`chat.${roomId}`)
    .whisper('typing', {
        name: this.user.name
    });
```

要監聽客戶端事件，您可以使用 `listenForWhisper` 方法：

```js
Echo.private(`chat.${roomId}`)
    .listenForWhisper('typing', (e) => {
        console.log(e.name);
    });
```

<a name="notifications"></a>
## 通知

透過將事件廣播與 [通知](/docs/{{version}}/notifications) 配對，您的 JavaScript 應用程式可以在新通知發生時接收它們，而無需重新整理頁面。在開始之前，請務必閱讀有關使用 [廣播通知 Channel](/docs/{{version}}/notifications#broadcast-notifications) 的說明文件。

一旦您設定了通知以使用廣播 Channel，您就可以使用 Echo 的 `notification` 方法監聽廣播事件。請記住，Channel 名稱應與接收通知的實體的類別名稱相符：

```js
Echo.private(`App.Models.User.${userId}`)
    .notification((notification) => {
        console.log(notification.type);
    });
```

在此範例中，透過 `broadcast` Channel 發送到 `App\Models\User` 實例的所有通知都將由回呼接收。您的應用程式的 `routes/channels.php` 檔案中包含 `App.Models.User.{id}` Channel 的 Channel 授權回呼。
