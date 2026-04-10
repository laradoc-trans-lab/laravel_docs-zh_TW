# Laravel Cashier (Paddle)

- [簡介](#introduction)
- [升級 Cashier](#upgrading-cashier)
- [安裝](#installation)
    - [Paddle 沙盒](#paddle-sandbox)
- [設定](#configuration)
    - [可計費模型](#billable-model)
    - [API 金鑰](#api-keys)
    - [Paddle JS](#paddle-js)
    - [貨幣設定](#currency-configuration)
    - [覆蓋預設模型](#overriding-default-models)
- [快速入門](#quickstart)
    - [銷售產品](#quickstart-selling-products)
    - [銷售訂閱](#quickstart-selling-subscriptions)
- [結帳工作階段 (Checkout Sessions)](#checkout-sessions)
    - [覆蓋式結帳 (Overlay Checkout)](#overlay-checkout)
    - [內嵌式結帳 (Inline Checkout)](#inline-checkout)
    - [訪客結帳](#guest-checkouts)
- [價格預覽](#price-previews)
    - [客戶價格預覽](#customer-price-previews)
    - [折扣](#price-discounts)
- [客戶](#customers)
    - [客戶預設值](#customer-defaults)
    - [取得客戶](#retrieving-customers)
    - [建立客戶](#creating-customers)
- [訂閱](#subscriptions)
    - [建立訂閱](#creating-subscriptions)
    - [檢查訂閱狀態](#checking-subscription-status)
    - [訂閱單次收費](#subscription-single-charges)
    - [更新付款資訊](#updating-payment-information)
    - [變更方案](#changing-plans)
    - [訂閱數量](#subscription-quantity)
    - [包含多個產品的訂閱](#subscriptions-with-multiple-products)
    - [多重訂閱](#multiple-subscriptions)
    - [暫停訂閱](#pausing-subscriptions)
    - [取消訂閱](#canceling-subscriptions)
- [訂閱試用](#subscription-trials)
    - [預先收集付款方式](#with-payment-method-up-front)
    - [不預先收集付款方式](#without-payment-method-up-front)
    - [延長或啟用試用](#extend-or-activate-a-trial)
- [處理 Paddle Webhooks](#handling-paddle-webhooks)
    - [定義 Webhook 事件處理常式](#defining-webhook-event-handlers)
    - [驗證 Webhook 簽名](#verifying-webhook-signatures)
- [單次收費](#single-charges)
    - [產品收費](#charging-for-products)
    - [交易退款](#refunding-transactions)
    - [交易入帳](#crediting-transactions)
- [交易](#transactions)
    - [過去與即將到來的付款](#past-and-upcoming-payments)
- [測試](#testing)

<a name="introduction"></a>
## 簡介

> [!WARNING]
> 此文件適用於 Cashier Paddle 2.x 與 Paddle Billing 的整合。如果您仍在使用 Paddle Classic，請使用 [Cashier Paddle 1.x](https://github.com/laravel/cashier-paddle/tree/1.x)。

[Laravel Cashier Paddle](https://github.com/laravel/cashier-paddle) 為 [Paddle](https://paddle.com) 的訂閱計費服務提供了一個具表現力且流暢的介面。它處理了幾乎所有您所厭惡的重複性訂閱計費樣板程式碼。除了基本的訂閱管理外，Cashier 還能處理：切換訂閱、訂閱「數量」、暫停訂閱、取消寬限期等。

在深入研究 Cashier Paddle 之前，我們建議您也閱讀 Paddle 的 [概念指南](https://developer.paddle.com/concepts/overview) 與 [API 文件](https://developer.paddle.com/api-reference/overview)。


<a name="upgrading-cashier"></a>
## 升級 Cashier

當升級到新版本的 Cashier 時，請務必仔細閱讀 [升級指南](https://github.com/laravel/cashier-paddle/blob/master/UPGRADE.md)。


<a name="installation"></a>
## 安裝

首先，使用 Composer 套件管理工具安裝針對 Paddle 的 Cashier 套件：

```shell
composer require laravel/cashier-paddle
```

接下來，您應該使用 `vendor:publish` Artisan 指令來發布 Cashier 的遷移文件：

```shell
php artisan vendor:publish --tag="cashier-migrations"
```

然後，您應該執行應用程式的資料庫遷移。Cashier 的遷移將會建立一個新的 `customers` 資料表。此外，還會建立新的 `subscriptions` 和 `subscription_items` 資料表來儲存所有客戶的訂閱資訊。最後，會建立一個新的 `transactions` 資料表來儲存所有與您的客戶相關的 Paddle 交易：

```shell
php artisan migrate
```

> [!WARNING]
> 為了確保 Cashier 能正確處理所有 Paddle 事件，請記得 [設定 Cashier 的 Webhook 處理機制](#handling-paddle-webhooks)。


<a name="paddle-sandbox"></a>
### Paddle 沙盒

在本地端與分段 (staging) 開發期間，您應該 [註冊一個 Paddle Sandbox 帳號](https://sandbox-login.paddle.com/signup)。這個帳號將為您提供一個沙盒環境，讓您在無需支付實際款項的情況下測試並開發應用程式。您可以使用 Paddle 的 [測試卡號](https://developer.paddle.com/concepts/payment-methods/credit-debit-card#test-payment-method) 來模擬各種付款情境。

使用 Paddle Sandbox 環境時，您應該在應用程式的 `.env` 文件中將 `PADDLE_SANDBOX` 環境變數設定為 `true`：

```ini
PADDLE_SANDBOX=true
```

完成應用程式開發後，您可以 [申請 Paddle 廠商帳號](https://paddle.com)。在您的應用程式正式上線之前，Paddle 需要審核您的應用程式網域。


<a name="configuration"></a>
## 設定


<a name="billable-model"></a>
### 可計費模型

在使用 Cashier 之前，您必須在使用者模型定義中加入 `Billable` trait。這個 trait 提供了各種方法，讓您能夠執行常見的計費任務，例如建立訂閱和更新付款方式資訊：

```php
use Laravel\Paddle\Billable;

class User extends Authenticatable
{
    use Billable;
}
```

如果您有非使用者的可計費實體，您也可以將此 trait 加入到那些類別中：

```php
use Illuminate\Database\Eloquent\Model;
use Laravel\Paddle\Billable;

class Team extends Model
{
    use Billable;
}
```


<a name="api-keys"></a>
### API 金鑰

接下來，您應該在應用程式的 `.env` 文件中設定 Paddle 金鑰。您可以從 Paddle 控制面板取得您的 Paddle API 金鑰：

```ini
PADDLE_CLIENT_SIDE_TOKEN=your-paddle-client-side-token
PADDLE_API_KEY=your-paddle-api-key
PADDLE_RETAIN_KEY=your-paddle-retain-key
PADDLE_WEBHOOK_SECRET="your-paddle-webhook-secret"
PADDLE_SANDBOX=true
```

當您使用 [Paddle 的 Sandbox 環境](#paddle-sandbox) 時，`PADDLE_SANDBOX` 環境變數應設定為 `true`。如果您將應用程式部署到正式環境並使用 Paddle 的正式廠商環境，則 `PADDLE_SANDBOX` 變數應設定為 `false`。

`PADDLE_RETAIN_KEY` 是選填的，僅在您將 Paddle 與 [Retain](https://developer.paddle.com/concepts/retain/overview) 搭配使用時才需要設定。


<a name="paddle-js"></a>
### Paddle JS

Paddle 依賴其本身的 JavaScript 函式庫來啟動 Paddle 結帳元件。您可以透過將 `@paddleJS` Blade 指令放置在應用程式佈局的結束 `</head>` 標記之前來載入該 JavaScript 函式庫：

```blade
<head>
    ...

    @paddleJS
</head>
```


<a name="currency-configuration"></a>
### 貨幣設定

您可以指定一個區域設定 (locale)，用於在發票上顯示金額時的格式化。內部地，Cashier 利用 [PHP 的 `NumberFormatter` 類別](https://www.php.net/manual/en/class.numberformatter.php) 來設定貨幣的區域設定：

```ini
CASHIER_CURRENCY_LOCALE=nl_BE
```

> [!WARNING]
> 若要使用 `en` 以外的區域設定，請確保您的伺服器已安裝並設定 `ext-intl` PHP 擴展。


<a name="overriding-default-models"></a>
### 覆蓋預設模型

您可以透過定義自己的模型並繼承相對應的 Cashier 模型，來擴展 Cashier 內部使用的模型：

```php
use Laravel\Paddle\Subscription as CashierSubscription;

class Subscription extends CashierSubscription
{
    // ...
}
```

定義好模型後，您可以透過 `Laravel\Paddle\Cashier` 類別指示 Cashier 使用您的自定義模型。通常，您應該在應用程式 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中告知 Cashier 您的自定義模型：

```php
use App\Models\Cashier\Subscription;
use App\Models\Cashier\Transaction;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Cashier::useSubscriptionModel(Subscription::class);
    Cashier::useTransactionModel(Transaction::class);
}
```

<a name="quickstart"></a>
## 快速入門


<a name="quickstart-selling-products"></a>
### 銷售產品

> [!NOTE]
> 在利用 Paddle 結帳之前，您應該在 Paddle 控制面板中定義具有固定價格的產品。此外，您應該[設定 Paddle 的 Webhook 處理](#handling-paddle-webhooks)。

透過應用程式提供產品和訂閱計費可能會令人感到壓力。然而，多虧了 Cashier 和 [Paddle 的覆蓋式結帳 (Checkout Overlay)](https://developer.paddle.com/concepts/sell/overlay-checkout)，您可以輕鬆地建構現代且強健的付款整合。

要針對非週期性的單次收費產品向客戶收費，我們將利用 Cashier 透過 Paddle 的覆蓋式結帳向客戶收費，客戶將在其中提供其付款詳細資訊並確認購買。一旦透過覆蓋式結帳完成付款，客戶將被重新導向至您在應用程式中選擇的成功 URL：

```php
use Illuminate\Http\Request;

Route::get('/buy', function (Request $request) {
    $checkout = $request->user()->checkout('pri_deluxe_album')
        ->returnTo(route('dashboard'));

    return view('buy', ['checkout' => $checkout]);
})->name('checkout');
```

如上面的範例所示，我們將利用 Cashier 提供的 `checkout` 方法來建立一個結帳物件，以便針對給定的「價格識別碼」向客戶呈現 Paddle 的覆蓋式結帳。在使用 Paddle 時，「價格」是指[針對特定產品定義的價格](https://developer.paddle.com/build/products/create-products-prices)。

如果有需要，`checkout` 方法會自動在 Paddle 中建立一名客戶，並將該 Paddle 客戶記錄連結到您應用程式資料庫中對應的使用者。完成結帳工作階段後，客戶將被重新導向至專屬的成功頁面，您可以在該頁面向客戶顯示通知訊息。

在 `buy` 視圖中，我們將包含一個用來顯示覆蓋式結帳的按鈕。`paddle-button` Blade 元件已包含在 Cashier Paddle 中；不過，您也可以[手動渲染覆蓋式結帳](#manually-rendering-an-overlay-checkout)：

```html
<x-paddle-button :checkout="$checkout" class="px-8 py-4">
    Buy Product
</x-paddle-button>
```


<a name="providing-meta-data-to-paddle-checkout"></a>
#### 向 Paddle 結帳提供中繼資料

在銷售產品時，通常會透過應用程式定義的 `Cart` 與 `Order` 模型來追蹤已完成的訂單和購買的產品。當將客戶重新導向至 Paddle 的覆蓋式結帳以完成購買時，您可能需要提供現有的訂單識別碼，以便在客戶被重新導向回您的應用程式時，將完成的購買行為與對應的訂單關聯起來。

為了達成此目的，您可以向 `checkout` 方法提供一個自定義數據陣列。讓我們假設當使用者開始結帳流程時，應用程式中會建立一個待處理的 `Order`。請記得，本範例中的 `Cart` 與 `Order` 模型僅供說明，並非由 Cashier 提供。您可以根據自身應用程式的需求來實作這些概念：

```php
use App\Models\Cart;
use App\Models\Order;
use Illuminate\Http\Request;

Route::get('/cart/{cart}/checkout', function (Request $request, Cart $cart) {
    $order = Order::create([
        'cart_id' => $cart->id,
        'price_ids' => $cart->price_ids,
        'status' => 'incomplete',
    ]);

    $checkout = $request->user()->checkout($order->price_ids)
        ->customData(['order_id' => $order->id]);

    return view('billing', ['checkout' => $checkout]);
})->name('checkout');
```

如上面的範例所示，當使用者開始結帳流程時，我們會將購物車/訂單相關的所有 Paddle 價格識別碼提供給 `checkout` 方法。當然，您的應用程式負責在客戶添加項目時，將這些項目與「購物車」或訂單關聯。我們還透過 `customData` 方法將訂單 ID 提供給 Paddle 的覆蓋式結帳。

當然，您可能希望在客戶完成結帳流程後將訂單標記為「已完成」。為了達成此目的，您可以監聽由 Paddle 發送並由 Cashier 透過事件觸發的 Webhooks，以便將訂單資訊儲存在您的資料庫中。

首先，請監聽由 Cashier 發送的 `TransactionCompleted` 事件。通常，您應該在應用程式 `AppServiceProvider` 類別的 `boot` 方法中註冊事件監聽器：

```php
use App\Listeners\CompleteOrder;
use Illuminate\Support\Facades\Event;
use Laravel\Paddle\Events\TransactionCompleted;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Event::listen(TransactionCompleted::class, CompleteOrder::class);
}
```

在此範例中，`CompleteOrder` 監聽器可能如下所示：

```php
namespace App\Listeners;

use App\Models\Order;
use Laravel\Paddle\Cashier;
use Laravel\Paddle\Events\TransactionCompleted;

class CompleteOrder
{
    /**
     * Handle the incoming Cashier webhook event.
     */
    public function handle(TransactionCompleted $event): void
    {
        $orderId = $event->payload['data']['custom_data']['order_id'] ?? null;

        $order = Order::findOrFail($orderId);

        $order->update(['status' => 'completed']);
    }
}
```

有關 [ `transaction.completed` 事件包含的數據](https://developer.paddle.com/webhooks/transactions/transaction-completed) 之詳細資訊，請參閱 Paddle 的文件。

<a name="quickstart-selling-subscriptions"></a>
### 銷售訂閱

> [!NOTE]
> 在利用 Paddle Checkout 之前，您應該在 Paddle 控制面板中定義具有固定價格的產品。此外，您應該 [設定 Paddle 的 webhook 處理](#handling-paddle-webhooks)。

透過您的應用程式提供產品與訂閱計費可能會讓人感到壓力。然而，感謝 Cashier 與 [Paddle's Checkout Overlay](https://developer.paddle.com/concepts/sell/overlay-checkout)，您可以輕鬆地建構現代且強健的付款整合。

為了學習如何使用 Cashier 與 Paddle's Checkout Overlay 銷售訂閱，讓我們考慮一個簡單的場景：一個提供基礎每月 (`price_basic_monthly`) 與年度 (`price_basic_yearly`) 方案的訂閱服務。這兩個價格可以在我們的 Paddle 控制面板中被分組在一個「Basic」產品 (`pro_basic`) 之下。此外，我們的訂閱服務可能還提供一個「Expert」方案 `pro_expert`。

首先，讓我們看看客戶如何訂閱我們的服務。當然，您可以想像客戶可能會在應用程式的價格頁面上點擊 Basic 方案的「訂閱」按鈕。此按鈕將為他們選擇的方案啟動 Paddle Checkout Overlay。首先，讓我們透過 `checkout` 方法啟動一個結帳工作階段：

```php
use Illuminate\Http\Request;

Route::get('/subscribe', function (Request $request) {
    $checkout = $request->user()->checkout('price_basic_monthly')
        ->returnTo(route('dashboard'));

    return view('subscribe', ['checkout' => $checkout]);
})->name('subscribe');
```

在 `subscribe` 視圖中，我們將包含一個按鈕來顯示 Checkout Overlay。`paddle-button` Blade 元件已包含在 Cashier Paddle 中；然而，您也可以 [手動渲染覆蓋式結帳](#manually-rendering-an-overlay-checkout)：

```html
<x-paddle-button :checkout="$checkout" class="px-8 py-4">
    Subscribe
</x-paddle-button>
```

現在，當點擊 Subscribe 按鈕時，客戶將能夠輸入其付款詳細資訊並啟動訂閱。為了知道他們的訂閱何時正式開始（因為某些付款方式需要幾秒鐘處理），您還應該 [設定 Cashier 的 webhook 處理](#handling-paddle-webhooks)。

既然客戶可以開始訂閱，我們需要限制應用程式的某些部分，以便只有已訂閱的使用者才能訪問。當然，我們隨時可以透過 Cashier 的 `Billable` trait 提供的 `subscribed` 方法來判斷使用者的目前訂閱狀態：

```blade
@if ($user->subscribed())
    <p>You are subscribed.</p>
@endif
```

我們甚至可以輕鬆地判斷使用者是否訂閱了特定的產品或價格：

```blade
@if ($user->subscribedToProduct('pro_basic'))
    <p>You are subscribed to our Basic product.</p>
@endif

@if ($user->subscribedToPrice('price_basic_monthly'))
    <p>You are subscribed to our monthly Basic plan.</p>
@endif
```


<a name="quickstart-building-a-subscribed-middleware"></a>
#### 建立訂閱中介層

為了方便起見，您可能希望建立一個 [中介層](/docs/{{version}}/middleware) 來判斷傳入的請求是否來自已訂閱的使用者。一旦定義了此中介層，您就可以輕鬆地將其分配給路由，以防止未訂閱的使用者訪問該路由：

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class Subscribed
{
    /**
     * Handle an incoming request.
     */
    public function handle(Request $request, Closure $next): Response
    {
        if (! $request->user()?->subscribed()) {
            // Redirect user to billing page and ask them to subscribe...
            return redirect('/subscribe');
        }

        return $next($request);
    }
}
```

一旦定義了中介層，您就可以將其分配給路由：

```php
use App\Http\Middleware\Subscribed;

Route::get('/dashboard', function () {
    // ...
})->middleware([Subscribed::class]);
```


<a name="quickstart-allowing-customers-to-manage-their-billing-plan"></a>
#### 允許客戶管理其計費方案

當然，客戶可能想要將其訂閱方案變更為另一個產品或「層級(tier)」。在上面的範例中，我們希望允許客戶將方案從每月訂閱變更為年度訂閱。為此，您需要實作類似於指向下方路由的按鈕：

```php
use Illuminate\Http\Request;

Route::put('/subscription/{price}/swap', function (Request $request, $price) {
    $user->subscription()->swap($price); // With "$price" being "price_basic_yearly" for this example.

    return redirect()->route('dashboard');
})->name('subscription.swap');
```

除了變更方案外，您還需要允許客戶取消其訂閱。與變更方案一樣，請提供一個指向以下路由的按鈕：

```php
use Illuminate\Http\Request;

Route::put('/subscription/cancel', function (Request $request, $price) {
    $user->subscription()->cancel();

    return redirect()->route('dashboard');
})->name('subscription.cancel');
```

現在，您的訂閱將在計費週期結束時被取消。

> [!NOTE]
> 只要您設定了 Cashier 的 webhook 處理，Cashier 就會透過檢查來自 Paddle 的傳入 webhook，自動保持您應用程式中與 Cashier 相關的資料庫表格同步。例如，當您透過 Paddle 的控制面板取消客戶的訂閱時，Cashier 將收到對應的 webhook 並在您應用程式的資料庫中將該訂閱標記為「已取消」。

<a name="checkout-sessions"></a>
## 結帳工作階段 (Checkout Sessions)

大多數向客戶計費的操作都是透過 Paddle 的 [覆蓋式結帳小工具 (Checkout Overlay widget)](https://developer.paddle.com/build/checkout/build-overlay-checkout) 或利用 [內嵌式結帳 (inline checkout)](https://developer.paddle.com/build/checkout/build-branded-inline-checkout) 進行「結帳」來完成。

在使用 Paddle 處理結帳付款之前，您應該在 Paddle 的結帳設定控制面板中定義應用程式的 [預設付款連結 (default payment link)](https://developer.paddle.com/build/transactions/default-payment-link#set-default-link)。


<a name="overlay-checkout"></a>
### 覆蓋式結帳 (Overlay Checkout)

在顯示覆蓋式結帳小工具之前，您必須使用 Cashier 產生一個結帳工作階段。結帳工作階段會告知結帳小工具應執行哪項計費操作：

```php
use Illuminate\Http\Request;

Route::get('/buy', function (Request $request) {
    $checkout = $user->checkout('pri_34567')
        ->returnTo(route('dashboard'));

    return view('billing', ['checkout' => $checkout]);
});
```

Cashier 包含一個 `paddle-button` [Blade 元件](/docs/{{version}}/blade#components)。您可以將結帳工作階段作為一個 "prop" 傳遞給此元件。接著，當此按鈕被點擊時，Paddle 的結帳小工具將會顯示：

```html
<x-paddle-button :checkout="$checkout" class="px-8 py-4">
    Subscribe
</x-paddle-button>
```

預設情況下，這將使用 Paddle 的預設樣式來顯示小工具。您可以透過在元件中加入 [Paddle 支援的屬性](https://developer.paddle.com/paddlejs/html-data-attributes)，例如 `data-theme='light'` 屬性來自訂小工具：

```html
<x-paddle-button :checkout="$checkout" class="px-8 py-4" data-theme="light">
    Subscribe
</x-paddle-button>
```

Paddle 的結帳小工具是非同步的。一旦使用者在小工具中建立訂閱，Paddle 會向您的應用程式發送一個 webhook，以便您在應用程式的資料庫中正確更新訂閱狀態。因此，正確地 [設定 webhook](#handling-paddle-webhooks) 以處理來自 Paddle 的狀態變更至關重要。

> [!WARNING]
> 在訂閱狀態變更後，接收對應 webhook 的延遲通常很小，但您在應用程式中應考慮到這一點，因為使用者的訂閱在完成結帳後可能不會立即生效。


<a name="manually-rendering-an-overlay-checkout"></a>
#### 手動渲染覆蓋式結帳

您也可以在不使用 Laravel 內建 Blade 元件的情況下，手動渲染覆蓋式結帳。首先，請按照 [先前範例所示](#overlay-checkout) 產生結帳工作階段：

```php
use Illuminate\Http\Request;

Route::get('/buy', function (Request $request) {
    $checkout = $user->checkout('pri_34567')
        ->returnTo(route('dashboard'));

    return view('billing', ['checkout' => $checkout]);
});
```

接著，您可以使用 Paddle.js 來初始化結帳。在本範例中，我們將建立一個被指定 `paddle_button` 類別 (class) 的連結。當該連結被點擊時，Paddle.js 會偵測到此類別並顯示覆蓋式結帳：

```blade
<?php
$items = $checkout->getItems();
$customer = $checkout->getCustomer();
$custom = $checkout->getCustomData();
?>

<a
    href='#!'
    class='paddle_button'
    data-items='{!! json_encode($items) !!}'
    @if ($customer) data-customer-id='{{ $customer->paddle_id }}' @endif
    @if ($custom) data-custom-data='{{ json_encode($custom) }}' @endif
    @if ($returnUrl = $checkout->getReturnUrl()) data-success-url='{{ $returnUrl }}' @endif
>
    Buy Product
</a>
```


<a name="inline-checkout"></a>
### 內嵌式結帳 (Inline Checkout)

如果您不想使用 Paddle 的「覆蓋式」結帳小工具，Paddle 也提供了將小工具內嵌顯示的選項。雖然這種方法不允許您調整結帳的任何 HTML 欄位，但它允許您將小工具嵌入到應用程式中。

為了讓您輕鬆開始使用內嵌式結帳，Cashier 包含一個 `paddle-checkout` Blade 元件。首先，您應該 [產生一個結帳工作階段](#overlay-checkout)：

```php
use Illuminate\Http\Request;

Route::get('/buy', function (Request $request) {
    $checkout = $user->checkout('pri_34567')
        ->returnTo(route('dashboard'));

    return view('billing', ['checkout' => $checkout]);
});
```

然後，您可以將結帳工作階段傳遞給元件的 `checkout` 屬性：

```blade
<x-paddle-checkout :checkout="$checkout" class="w-full" />
```

若要調整內嵌式結帳元件的高度，您可以將 `height` 屬性傳遞給 Blade 元件：

```blade
<x-paddle-checkout :checkout="$checkout" class="w-full" height="500" />
```

請參閱 Paddle 的 [內嵌式結帳指南 (guide on Inline Checkout)](https://developer.paddle.com/build/checkout/build-branded-inline-checkout) 和 [可用的結帳設定 (available checkout settings)](https://developer.paddle.com/build/checkout/set-up-checkout-default-settings)，以獲取有關內嵌式結帳自訂選項的詳細資訊。


<a name="manually-rendering-an-inline-checkout"></a>
#### 手動渲染內嵌式結帳

您也可以在不使用 Laravel 內建 Blade 元件的情況下，手動渲染內嵌式結帳。首先，請按照 [先前範例所示](#inline-checkout) 產生結帳工作階段：

```php
use Illuminate\Http\Request;

Route::get('/buy', function (Request $request) {
    $checkout = $user->checkout('pri_34567')
        ->returnTo(route('dashboard'));

    return view('billing', ['checkout' => $checkout]);
});
```

接著，您可以使用 Paddle.js 來初始化結帳。在本範例中，我們將使用 [Alpine.js](https://github.com/alpinejs/alpine) 來演示；不過，您可以根據自己的前端技術棧修改此範例：

```blade
<?php
$options = $checkout->options();

$options['settings']['frameTarget'] = 'paddle-checkout';
$options['settings']['frameInitialHeight'] = 366;
?>

<div class="paddle-checkout" x-data="{}" x-init="
    Paddle.Checkout.open(@json($options));
">
</div>
```


<a name="guest-checkouts"></a>
### 訪客結帳

有時候，您可能需要為不需要在應用程式中擁有帳戶的使用者建立結帳工作階段。為此，您可以使用 `guest` 方法：

```php
use Illuminate\Http\Request;
use Laravel\Paddle\Checkout;

Route::get('/buy', function (Request $request) {
    $checkout = Checkout::guest(['pri_34567'])
        ->returnTo(route('home'));

    return view('billing', ['checkout' => $checkout]);
});
```

然後，您可以將結帳工作階段提供給 [Paddle 按鈕](#overlay-checkout) 或 [內嵌式結帳](#inline-checkout) Blade 元件。

<a name="price-previews"></a>
## 價格預覽

Paddle 允許您根據貨幣自訂價格，基本上允許您為不同國家設定不同的價格。Cashier Paddle 允許您使用 `previewPrices` 方法取得所有這些價格。此方法接收您希望取得價格的價格 ID：

```php
use Laravel\Paddle\Cashier;

$prices = Cashier::previewPrices(['pri_123', 'pri_456']);
```

貨幣將根據請求的 IP 位址決定；不過，您也可以選擇提供特定國家來取得價格：

```php
use Laravel\Paddle\Cashier;

$prices = Cashier::previewPrices(['pri_123', 'pri_456'], ['address' => [
    'country_code' => 'BE',
    'postal_code' => '1234',
]]);
```

取得價格後，您可以隨意顯示它們：

```blade
<ul>
    @foreach ($prices as $price)
        <li>{{ $price->product['name'] }} - {{ $price->total() }}</li>
    @endforeach
</ul>
```

您也可以分別顯示小計價格和稅額：

```blade
<ul>
    @foreach ($prices as $price)
        <li>{{ $price->product['name'] }} - {{ $price->subtotal() }} (+ {{ $price->tax() }} tax)</li>
    @endforeach
</ul>
```

如需更多資訊，請[參閱 Paddle 關於價格預覽的 API 文件](https://developer.paddle.com/api-reference/pricing-preview/preview-prices)。


<a name="customer-price-previews"></a>
### 客戶價格預覽

如果使用者已經是客戶，而您想要顯示適用於該客戶的價格，您可以直接從客戶實例取得價格：

```php
use App\Models\User;

$prices = User::find(1)->previewPrices(['pri_123', 'pri_456']);
```

在內部，Cashier 將使用使用者的客戶 ID 來取得其貨幣的價格。例如，居住在美國的使用者將看到以美元計價的價格，而比利時的使用者將看到以歐元計價的價格。如果找不到匹配的貨幣，將使用產品的預設貨幣。您可以在 Paddle 控制面板中自訂產品或訂閱方案的所有價格。


<a name="price-discounts"></a>
### 折扣

您也可以選擇顯示折扣後的價格。呼叫 `previewPrices` 方法時，您可以透過 `discount_id` 選項提供折扣 ID：

```php
use Laravel\Paddle\Cashier;

$prices = Cashier::previewPrices(['pri_123', 'pri_456'], [
    'discount_id' => 'dsc_123'
]);
```

然後，顯示計算後的價格：

```blade
<ul>
    @foreach ($prices as $price)
        <li>{{ $price->product['name'] }} - {{ $price->total() }}</li>
    @endforeach
</ul>
```


<a name="customers"></a>
## 客戶


<a name="customer-defaults"></a>
### 客戶預設值

Cashier 允許您在建立結帳工作階段時，為客戶定義一些有用的預設值。設定這些預設值可讓您預先填充客戶的電子郵件地址和姓名，以便他們能立即進入結帳元件的付款部分。您可以在可計費模型上覆寫以下方法來設定這些預設值：

```php
/**
 * Get the customer's name to associate with Paddle.
 */
public function paddleName(): string|null
{
    return $this->name;
}

/**
 * Get the customer's email address to associate with Paddle.
 */
public function paddleEmail(): string|null
{
    return $this->email;
}
```

這些預設值將用於 Cashier 中所有會產生 [結帳工作階段 (Checkout Sessions)](#checkout-sessions) 的動作。


<a name="retrieving-customers"></a>
### 取得客戶

您可以使用 `Cashier::findBillable` 方法透過 Paddle 客戶 ID 取得客戶。此方法將回傳可計費模型的實例：

```php
use Laravel\Paddle\Cashier;

$user = Cashier::findBillable($customerId);
```


<a name="creating-customers"></a>
### 建立客戶

有時候，您可能希望在不開始訂閱的情況下建立 Paddle 客戶。您可以使用 `createAsCustomer` 方法來實現：

```php
$customer = $user->createAsCustomer();
```

系統將回傳一個 `Laravel\Paddle\Customer` 實例。一旦在 Paddle 中建立了客戶，您可以在稍後日期開始訂閱。您可以提供一個選擇性的 `$options` 陣列，以傳遞 [Paddle API 支援的任何其他客戶建立參數](https://developer.paddle.com/api-reference/customers/create-customer)：

```php
$customer = $user->createAsCustomer($options);
```

<a name="subscriptions"></a>
## 訂閱


<a name="creating-subscriptions"></a>
### 建立訂閱

要建立訂閱，請先從資料庫中取得可計費模型的實例，這通常會是一個 `App\Models\User` 的實例。取得模型實例後，您可以使用 `subscribe` 方法來建立該模型的結帳工作階段：

```php
use Illuminate\Http\Request;

Route::get('/user/subscribe', function (Request $request) {
    $checkout = $request->user()->subscribe($premium = 'pri_123', 'default')
        ->returnTo(route('home'));

    return view('billing', ['checkout' => $checkout]);
});
```

傳遞給 `subscribe` 方法的第一個引數是使用者要訂閱的特定價格。此值應與 Paddle 中的價格識別碼對應。`returnTo` 方法接受一個 URL，使用者在成功完成結帳後將被重新導向至該處。傳遞給 `subscribe` 方法的第二個引數應為訂閱的內部「類型」。如果您的應用程式僅提供單一訂閱，您可以將其稱為 `default` 或 `primary`。此訂閱類型僅供應用程式內部使用，不應顯示給使用者。此外，它不應包含空格，且在建立訂閱後絕不能更改。

您也可以使用 `customData` 方法提供關於訂閱的自訂元數據陣列：

```php
$checkout = $request->user()->subscribe($premium = 'pri_123', 'default')
    ->customData(['key' => 'value'])
    ->returnTo(route('home'));
```

一旦建立了訂閱結帳工作階段，就可以將該結帳工作階段提供給 Cashier Paddle 隨附的 `paddle-button` [Blade 組件](#overlay-checkout)：

```blade
<x-paddle-button :checkout="$checkout" class="px-8 py-4">
    Subscribe
</x-paddle-button>
```

在使用者完成結帳後，Paddle 會發出 `subscription_created` Webhook。Cashier 將接收此 Webhook 並為您的客戶設定訂閱。為了確保所有 Webhook 都能被您的應用程式正確接收並處理，請確保您已正確 [設定 Webhook 處理](#handling-paddle-webhooks)。


<a name="checking-subscription-status"></a>
### 檢查訂閱狀態

一旦使用者訂閱了您的應用程式，您可以使用各種便捷的方法來檢查他們的訂閱狀態。首先，如果使用者擁有有效的訂閱（即使該訂閱目前處於試用期內），`subscribed` 方法將返回 `true`：

```php
if ($user->subscribed()) {
    // ...
}
```

如果您的應用程式提供多重訂閱，您可以在呼叫 `subscribed` 方法時指定訂閱類型：

```php
if ($user->subscribed('default')) {
    // ...
}
```

`subscribed` 方法也非常適合用作 [路由中介層](/docs/{{version}}/middleware)，讓您可以根據使用者的訂閱狀態來過濾對路由和控制器的存取：

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class EnsureUserIsSubscribed
{
    /**
     * Handle an incoming request.
     *
     * @param  \Closure(\Illuminate\Http\Request): (\Symfony\Component\HttpFoundation\Response)  $next
     */
    public function handle(Request $request, Closure $next): Response
    {
        if ($request->user() && ! $request->user()->subscribed()) {
            // This user is not a paying customer...
            return redirect('/billing');
        }

        return $next($request);
    }
}
```

如果您想確定使用者是否仍處於試用期內，可以使用 `onTrial` 方法。此方法可用於決定是否應向使用者顯示他們仍處於試用期的警告：

```php
if ($user->subscription()->onTrial()) {
    // ...
}
```

`subscribedToPrice` 方法可用於根據給定的 Paddle 價格 ID，確定使用者是否訂閱了特定方案。在本範例中，我們將確定使用者的 `default` 訂閱是否正活躍地訂閱每月價格：

```php
if ($user->subscribedToPrice($monthly = 'pri_123', 'default')) {
    // ...
}
```

`recurring` 方法可用於確定使用者目前是否擁有活躍的訂閱，且不再處於試用期或寬限期內：

```php
if ($user->subscription()->recurring()) {
    // ...
}
```


<a name="canceled-subscription-status"></a>
#### 已取消的訂閱狀態

要確定使用者曾經是活躍訂閱者但已取消其訂閱，您可以使用 `canceled` 方法：

```php
if ($user->subscription()->canceled()) {
    // ...
}
```

您還可以確定使用者是否已取消訂閱，但仍處於「寬限期」直到訂閱完全到期。例如，如果使用者在 3 月 5 日取消了原定於 3 月 10 日到期的訂閱，則該使用者在 3 月 10 日之前處於其「寬限期」。此外，在此期間 `subscribed` 方法仍將返回 `true`：

```php
if ($user->subscription()->onGracePeriod()) {
    // ...
}
```


<a name="past-due-status"></a>
#### 過期狀態

如果訂閱的付款失敗，它將被標記為 `past_due`。當您的訂閱處於此狀態時，直到客戶更新其付款資訊之前，它將不會是活躍的。您可以使用訂閱實例上的 `pastDue` 方法來確定訂閱是否過期：

```php
if ($user->subscription()->pastDue()) {
    // ...
}
```

當訂閱過期時，您應該指示使用者 [更新他們的付款資訊](#updating-payment-information)。

如果您希望訂閱在 `past_due` 時仍被視為有效，可以使用 Cashier 提供的 `keepPastDueSubscriptionsActive` 方法。通常，此方法應在 `AppServiceProvider` 的 `register` 方法中呼叫：

```php
use Laravel\Paddle\Cashier;

/**
 * Register any application services.
 */
public function register(): void
{
    Cashier::keepPastDueSubscriptionsActive();
}
```

> [!WARNING]
> 當訂閱處於 `past_due` 狀態時，在更新付款資訊之前無法進行更改。因此，當訂閱處於 `past_due` 狀態時，`swap` 和 `updateQuantity` 方法將會拋出異常。


<a name="subscription-scopes"></a>
#### 訂閱查詢範圍 (Subscription Scopes)

大多數訂閱狀態也可用作查詢範圍 (query scopes)，以便您可以輕鬆地在資料庫中查詢處於特定狀態的訂閱：

```php
// Get all valid subscriptions...
$subscriptions = Subscription::query()->valid()->get();

// Get all of the canceled subscriptions for a user...
$subscriptions = $user->subscriptions()->canceled()->get();
```

可用範圍的完整列表如下：

```php
Subscription::query()->valid();
Subscription::query()->onTrial();
Subscription::query()->expiredTrial();
Subscription::query()->notOnTrial();
Subscription::query()->active();
Subscription::query()->recurring();
Subscription::query()->pastDue();
Subscription::query()->paused();
Subscription::query()->notPaused();
Subscription::query()->onPausedGracePeriod();
Subscription::query()->notOnPausedGracePeriod();
Subscription::query()->canceled();
Subscription::query()->notCanceled();
Subscription::query()->onGracePeriod();
Subscription::query()->notOnGracePeriod();
```

<a name="subscription-single-charges"></a>
### 訂閱單次收費

訂閱單次收費允許您在訂閱之外，對訂閱者進行一次性的收費。在呼叫 `charge` 方法時，您必須提供一個或多個價格 ID (price ID)：

```php
// Charge a single price...
$response = $user->subscription()->charge('pri_123');

// Charge multiple prices at once...
$response = $user->subscription()->charge(['pri_123', 'pri_456']);
```

`charge` 方法不會立即向客戶收費，直到該訂閱的下一個計費週期才會執行。如果您希望立即向客戶收費，可以使用 `chargeAndInvoice` 方法：

```php
$response = $user->subscription()->chargeAndInvoice('pri_123');
```


<a name="updating-payment-information"></a>
### 更新付款資訊

Paddle 會為每個訂閱儲存一種付款方式。如果您想要更新訂閱的預設付款方式，應該使用訂閱模型上的 `redirectToUpdatePaymentMethod` 方法，將客戶重新導向至 Paddle 託管的付款方式更新頁面：

```php
use Illuminate\Http\Request;

Route::get('/update-payment-method', function (Request $request) {
    $user = $request->user();

    return $user->subscription()->redirectToUpdatePaymentMethod();
});
```

當使用者完成資訊更新後，Paddle 會發送一個 `subscription_updated` webhook，隨後您應用程式資料庫中的訂閱詳細資訊將會被更新。


<a name="changing-plans"></a>
### 變更方案

使用者在訂閱您的應用程式後，偶爾可能會想要變更為新的訂閱方案。要更新使用者的訂閱方案，您應該將 Paddle 的價格識別碼傳遞給訂閱的 `swap` 方法：

```php
use App\Models\User;

$user = User::find(1);

$user->subscription()->swap($premium = 'pri_456');
```

如果您想要變更方案並立即向使用者開立發票，而不是等待下一個計費週期，可以使用 `swapAndInvoice` 方法：

```php
$user = User::find(1);

$user->subscription()->swapAndInvoice($premium = 'pri_456');
```


<a name="prorations"></a>
#### 按比例分攤 (Prorations)

預設情況下，Paddle 在變更方案時會對費用進行按比例分攤 (prorate)。您可以使用 `noProrate` 方法來更新訂閱而不進行費用分攤：

```php
$user->subscription('default')->noProrate()->swap($premium = 'pri_456');
```

如果您想要禁用比例分攤並立即向客戶收費，可以將 `swapAndInvoice` 方法與 `noProrate` 結合使用：

```php
$user->subscription('default')->noProrate()->swapAndInvoice($premium = 'pri_456');
```

或者，如果您不希望在變更訂閱時向客戶收費，可以使用 `doNotBill` 方法：

```php
$user->subscription('default')->doNotBill()->swap($premium = 'pri_456');
```

關於 Paddle 比例分攤政策的更多資訊，請參閱 Paddle 的 [比例分攤文件](https://developer.paddle.com/concepts/subscriptions/proration)。


<a name="subscription-quantity"></a>
### 訂閱數量

有時候訂閱會受到「數量」的影響。例如，一個專案管理應用程式可能會根據每個專案每月收取 10 美元。若要輕鬆增加或減少訂閱數量，請使用 `incrementQuantity` 和 `decrementQuantity` 方法：

```php
$user = User::find(1);

$user->subscription()->incrementQuantity();

// Add five to the subscription's current quantity...
$user->subscription()->incrementQuantity(5);

$user->subscription()->decrementQuantity();

// Subtract five from the subscription's current quantity...
$user->subscription()->decrementQuantity(5);
```

或者，您可以使用 `updateQuantity` 方法設定特定的數量：

```php
$user->subscription()->updateQuantity(10);
```

您可以使用 `noProrate` 方法來更新訂閱數量而不進行費用分攤：

```php
$user->subscription()->noProrate()->updateQuantity(10);
```


<a name="quantities-for-subscription-with-multiple-products"></a>
#### 包含多個產品之訂閱的數量

如果您的訂閱是 [包含多個產品的訂閱](#subscriptions-with-multiple-products)，您應該將想要增加或減少數量的價格 ID 作為第二個引數傳遞給增加/減少方法：

```php
$user->subscription()->incrementQuantity(1, 'price_chat');
```


<a name="subscriptions-with-multiple-products"></a>
### 包含多個產品的訂閱

[包含多個產品的訂閱](https://developer.paddle.com/build/subscriptions/add-remove-products-prices-addons) 允許您將多個計費產品分配給單一訂閱。例如，想像您正在構建一個客戶服務「協助中心 (helpdesk)」應用程式，該應用程式每月有 10 美元的基礎訂閱價格，但提供一個每月額外 15 美元的即時聊天附加產品。

在建立訂閱結帳工作階段時，您可以透過將價格陣列作為 `subscribe` 方法的第一個引數，為給定的訂閱指定多個產品：

```php
use Illuminate\Http\Request;

Route::post('/user/subscribe', function (Request $request) {
    $checkout = $request->user()->subscribe([
        'price_monthly',
        'price_chat',
    ]);

    return view('billing', ['checkout' => $checkout]);
});
```

在上述範例中，客戶的 `default` 訂閱將會附加兩個價格。這兩個價格將在其各自的計費週期內收費。如有需要，您可以傳遞一個鍵值對 (key / value) 的關聯陣列，以指定每個價格的特定數量：

```php
$user = User::find(1);

$checkout = $user->subscribe('default', ['price_monthly', 'price_chat' => 5]);
```

如果您想要為現有訂閱添加另一個價格，必須使用訂閱的 `swap` 方法。在呼叫 `swap` 方法時，您也應該包含訂閱目前的價格和數量：

```php
$user = User::find(1);

$user->subscription()->swap(['price_chat', 'price_original' => 2]);
```

上述範例將添加新價格，但在下一個計費週期之前不會向客戶收費。如果您想立即向客戶收費，可以使用 `swapAndInvoice` 方法：

```php
$user->subscription()->swapAndInvoice(['price_chat', 'price_original' => 2]);
```

您可以使用 `swap` 方法並省略想要移除的價格，從訂閱中移除價格：

```php
$user->subscription()->swap(['price_original' => 2]);
```

> [!WARNING]
> 您不能移除訂閱中的最後一個價格。相反地，您應該直接取消該訂閱。


<a name="multiple-subscriptions"></a>
### 多重訂閱

Paddle 允許您的客戶同時擁有多個訂閱。例如，您可能經營一家健身房，提供游泳訂閱和重量訓練訂閱，且每個訂閱可能有不同的定價。當然，客戶應該能夠訂閱其中一個或兩個方案。

當您的應用程式建立訂閱時，您可以將訂閱類型作為第二個引數提供給 `subscribe` 方法。該類型可以是任何代表使用者啟動的訂閱類型的字串：

```php
use Illuminate\Http\Request;

Route::post('/swimming/subscribe', function (Request $request) {
    $checkout = $request->user()->subscribe($swimmingMonthly = 'pri_123', 'swimming');

    return view('billing', ['checkout' => $checkout]);
});
```

在此範例中，我們為客戶啟動了一個每月的游泳訂閱。然而，他們可能在稍後想要變更為年度訂閱。在調整客戶的訂閱時，我們只需變更 `swimming` 訂閱的價格即可：

```php
$user->subscription('swimming')->swap($swimmingYearly = 'pri_456');
```

當然，您也可以完全取消該訂閱：

```php
$user->subscription('swimming')->cancel();
```

<a name="pausing-subscriptions"></a>
### 暫停訂閱

若要暫停訂閱，請在使用者的訂閱上呼叫 `pause` 方法：

```php
$user->subscription()->pause();
```

當訂閱被暫停時，Cashier 會自動設定資料庫中的 `paused_at` 欄位。此欄位用於判定 `paused` 方法何時應開始回傳 `true`。例如，如果客戶在 3 月 1 日暫停訂閱，但訂閱原定於 3 月 5 日才再次扣款，則 `paused` 方法在 3 月 5 日之前將繼續回傳 `false`。這是因為使用者通常被允許繼續使用應用程式直到其計費週期結束。

預設情況下，暫停會在下一個計費區間發生，因此客戶可以使用其已付款期間的剩餘時間。如果您想立即暫停訂閱，可以使用 `pauseNow` 方法：

```php
$user->subscription()->pauseNow();
```

使用 `pauseUntil` 方法，您可以將訂閱暫停到特定的時間點：

```php
$user->subscription()->pauseUntil(now()->plus(months: 1));
```

或者，您可以使用 `pauseNowUntil` 方法立即暫停訂閱直到給定的時間點：

```php
$user->subscription()->pauseNowUntil(now()->plus(months: 1));
```

您可以使用 `onPausedGracePeriod` 方法來判定使用者是否已暫停訂閱但仍處於「寬限期 (grace period)」：

```php
if ($user->subscription()->onPausedGracePeriod()) {
    // ...
}
```

若要恢復已暫停的訂閱，您可以在訂閱上呼叫 `resume` 方法：

```php
$user->subscription()->resume();
```

> [!WARNING]
> 訂閱在暫停狀態下無法被修改。如果您想變更至不同的方案或更新數量，必須先恢復該訂閱。


<a name="canceling-subscriptions"></a>
### 取消訂閱

若要取消訂閱，請在使用者的訂閱上呼叫 `cancel` 方法：

```php
$user->subscription()->cancel();
```

當訂閱被取消時，Cashier 會自動設定資料庫中的 `ends_at` 欄位。此欄位用於判定 `subscribed` 方法何時應開始回傳 `false`。例如，如果客戶在 3 月 1 日取消訂閱，但訂閱原定於 3 月 5 日才結束，則 `subscribed` 方法在 3 月 5 日之前將繼續回傳 `true`。這是因為使用者通常被允許繼續使用應用程式直到其計費週期結束。

您可以使用 `onGracePeriod` 方法來判定使用者是否已取消訂閱但仍處於「寬限期 (grace period)」：

```php
if ($user->subscription()->onGracePeriod()) {
    // ...
}
```

如果您希望立即取消訂閱，可以在訂閱上呼叫 `cancelNow` 方法：

```php
$user->subscription()->cancelNow();
```

若要停止處於寬限期中的訂閱被取消，您可以呼叫 `stopCancelation` 方法：

```php
$user->subscription()->stopCancelation();
```

> [!WARNING]
> Paddle 的訂閱在取消後無法恢復。如果您的客戶希望恢復訂閱，他們必須建立一個新的訂閱。

<a name="subscription-trials"></a>
## 訂閱試用


<a name="with-payment-method-up-front"></a>
### 預先收集付款方式

如果您希望在預先收集付款方式資訊的同時為客戶提供試用期，您應該在 Paddle 控制面板中，針對客戶訂閱的價格設定試用時間。接著，照常啟動結帳工作階段：

```php
use Illuminate\Http\Request;

Route::get('/user/subscribe', function (Request $request) {
    $checkout = $request->user()
        ->subscribe('pri_monthly')
        ->returnTo(route('home'));

    return view('billing', ['checkout' => $checkout]);
});
```

當您的應用程式收到 `subscription_created` 事件時，Cashier 會將試用期結束日期設定在您應用程式資料庫中的訂閱紀錄中，並指示 Paddle 在此日期之後才開始向客戶收費。

> [!WARNING]
> 如果客戶在試用結束日期之前沒有取消訂閱，他們將在試用期結束後立即被收費，因此請務必通知您的使用者其試用結束日期。

您可以使用使用者實例的 `onTrial` 方法來判斷使用者是否處於試用期中：

```php
if ($user->onTrial()) {
    // ...
}
```

若要判斷現有的試用是否已過期，您可以使用 `hasExpiredTrial` 方法：

```php
if ($user->hasExpiredTrial()) {
    // ...
}
```

若要判斷使用者是否針對特定訂閱類型進行試用，您可以將該類型傳遞給 `onTrial` 或 `hasExpiredTrial` 方法：

```php
if ($user->onTrial('default')) {
    // ...
}

if ($user->hasExpiredTrial('default')) {
    // ...
}
```


<a name="without-payment-method-up-front"></a>
### 不預先收集付款方式

如果您希望在不預先收集使用者付款方式資訊的情況下提供試用期，您可以將使用者關聯的客戶紀錄中 `trial_ends_at` 欄位設定為您想要的試用結束日期。這通常在使用者註冊期間完成：

```php
use App\Models\User;

$user = User::create([
    // ...
]);

$user->createAsCustomer([
    'trial_ends_at' => now()->plus(days: 10)
]);
```

Cashier 將這種類型的試用稱為「通用試用 (generic trial)」，因為它沒有附加到任何現有的訂閱上。如果目前日期尚未超過 `trial_ends_at` 的值，`User` 實例上的 `onTrial` 方法將回傳 `true`：

```php
if ($user->onTrial()) {
    // User is within their trial period...
}
```

當您準備為使用者建立實際的訂閱時，可以照常使用 `subscribe` 方法：

```php
use Illuminate\Http\Request;

Route::get('/user/subscribe', function (Request $request) {
    $checkout = $request->user()
        ->subscribe('pri_monthly')
        ->returnTo(route('home'));

    return view('billing', ['checkout' => $checkout]);
});
```

若要取得使用者的試用結束日期，您可以使用 `trialEndsAt` 方法。如果使用者處於試用期，此方法將回傳一個 Carbon 日期實例，否則回傳 `null`。如果您想取得預設訂閱以外的特定訂閱之試用結束日期，也可以傳遞選填的訂閱類型參數：

```php
if ($user->onTrial('default')) {
    $trialEndsAt = $user->trialEndsAt();
}
```

如果您想明確知道使用者是否處於「通用」試用期且尚未建立實際訂閱，可以使用 `onGenericTrial` 方法：

```php
if ($user->onGenericTrial()) {
    // User is within their "generic" trial period...
}
```


<a name="extend-or-activate-a-trial"></a>
### 延長或啟用試用

您可以透過呼叫 `extendTrial` 方法並指定試用應結束的時間點，來延長訂閱中現有的試用期：

```php
$user->subscription()->extendTrial(now()->plus(days: 5));
```

或者，您也可以透過呼叫訂閱上的 `activate` 方法來結束試用，立即啟用訂閱：

```php
$user->subscription()->activate();
```

<a name="handling-paddle-webhooks"></a>
## 處理 Paddle Webhooks

Paddle 可以透過 Webhook 通知您的應用程式各種事件。預設情況下，Cashier 服務提供者會註冊一個指向 Cashier Webhook 控制器的路由，該控制器將處理所有傳入的 Webhook 請求。

預設情況下，此控制器會自動處理因扣款失敗次數過多而取消的訂閱、訂閱更新以及付款方式的變更；然而，正如我們接下來將看到的，您可以擴充此控制器以處理任何您想要的 Paddle Webhook 事件。

為了確保您的應用程式能處理 Paddle Webhooks，請務必在 [Paddle 控制面板中設定 Webhook URL](https://vendors.paddle.com/notifications-v2)。預設情況下，Cashier 的 Webhook 控制器會回應 `/paddle/webhook` URL 路徑。您應該在 Paddle 控制面板中啟用的完整 Webhook 列表如下：

- Customer Updated
- Transaction Completed
- Transaction Updated
- Subscription Created
- Subscription Updated
- Subscription Paused
- Subscription Canceled

> [!WARNING]
> 請確保您使用 Cashier 內建的 [Webhook 簽名驗證](/docs/{{version}}/cashier-paddle#verifying-webhook-signatures) 中介層來保護傳入的請求。


<a name="webhooks-csrf-protection"></a>
#### Webhooks 與 CSRF 保護

由於 Paddle Webhooks 需要繞過 Laravel 的 [CSRF 保護](/docs/{{version}}/csrf)，您應該確保 Laravel 不會嘗試驗證傳入 Paddle Webhooks 的 CSRF 令牌 (CSRF token)。為了實現這一點，您應該在應用程式的 `bootstrap/app.php` 檔案中將 `paddle/*` 從 CSRF 保護中排除：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->preventRequestForgery(except: [
        'paddle/*',
    ]);
})
```


<a name="webhooks-local-development"></a>
#### Webhooks 與本地開發

為了讓 Paddle 在本地開發期間能向您的應用程式發送 Webhooks，您需要透過網站分享服務（例如 [Ngrok](https://ngrok.com/) 或 [Expose](https://expose.dev/docs/introduction)）來公開您的應用程式。如果您是使用 [Laravel Sail](/docs/{{version}}/sail) 在本地開發應用程式，可以使用 Sail 的 [網站分享指令](/docs/{{version}}/sail#sharing-your-site)。


<a name="defining-webhook-event-handlers"></a>
### 定義 Webhook 事件處理常式

Cashier 會自動處理因扣款失敗而取消的訂閱以及其他常見的 Paddle Webhooks。但是，如果您有想要處理的其他 Webhook 事件，可以透過監聽 Cashier 派發的以下事件來實現：

- `Laravel\Paddle\Events\WebhookReceived`
- `Laravel\Paddle\Events\WebhookHandled`

這兩個事件都包含 Paddle Webhook 的完整內容 (payload)。例如，如果您想處理 `transaction.billed` Webhook，可以註冊一個 [監聽器 (listener)](/docs/{{version}}/events#defining-listeners) 來處理該事件：

```php
<?php

namespace App\Listeners;

use Laravel\Paddle\Events\WebhookReceived;

class PaddleEventListener
{
    /**
     * Handle received Paddle webhooks.
     */
    public function handle(WebhookReceived $event): void
    {
        if ($event->payload['event_type'] === 'transaction.billed') {
            // Handle the incoming event...
        }
    }
}
```

Cashier 也會針對接收到的 Webhook 類型派發專屬事件。除了 Paddle 的完整內容外，這些事件還包含用於處理該 Webhook 的相關模型，例如可計費模型、訂閱或收據：

<div class="content-list" markdown="1">

- `Laravel\Paddle\Events\CustomerUpdated`
- `Laravel\Paddle\Events\TransactionCompleted`
- `Laravel\Paddle\Events\TransactionUpdated`
- `Laravel\Paddle\Events\SubscriptionCreated`
- `Laravel\Paddle\Events\SubscriptionUpdated`
- `Laravel\Paddle\Events\SubscriptionPaused`
- `Laravel\Paddle\Events\SubscriptionCanceled`

</div>

您也可以透過在應用程式的 `.env` 檔案中定義 `CASHIER_WEBHOOK` 環境變數來覆蓋預設的內建 Webhook 路由。此值應為 Webhook 路由的完整 URL，且必須與 Paddle 控制面板中設定的 URL 一致：

```ini
CASHIER_WEBHOOK=https://example.com/my-paddle-webhook-url
```


<a name="verifying-webhook-signatures"></a>
### 驗證 Webhook 簽名

為了確保 Webhooks 的安全性，您可以使用 [Paddle 的 Webhook 簽名](https://developer.paddle.com/webhooks/signature-verification)。為了方便起見，Cashier 自動包含了一個中介層，用於驗證傳入的 Paddle Webhook 請求是否有效。

要啟用 Webhook 驗證，請確保在應用程式的 `.env` 檔案中定義了 `PADDLE_WEBHOOK_SECRET` 環境變數。Webhook 密鑰可以從您的 Paddle 帳戶儀表板中取得。

<a name="single-charges"></a>
## 單次收費


<a name="charging-for-products"></a>
### 產品收費

如果您想要為客戶啟動產品購買，可以使用可計費模型實例上的 `checkout` 方法來為該購買產生結帳工作階段。`checkout` 方法可以接受一個或多個價格 ID。如有必要，可以使用關聯陣列來提供所購買產品的數量：

```php
use Illuminate\Http\Request;

Route::get('/buy', function (Request $request) {
    $checkout = $request->user()->checkout(['pri_tshirt', 'pri_socks' => 5]);

    return view('buy', ['checkout' => $checkout]);
});
```

在產生結帳工作階段後，您可以使用 Cashier 提供的 `paddle-button` [Blade 元件](#overlay-checkout)，讓使用者查看 Paddle 結帳小工具並完成購買：

```blade
<x-paddle-button :checkout="$checkout" class="px-8 py-4">
    Buy
</x-paddle-button>
```

結帳工作階段有一個 `customData` 方法，允許您將任何想要的自訂資料傳遞給底層的交易建立過程。請參閱 [Paddle 官方文件](https://developer.paddle.com/build/transactions/custom-data) 以了解傳遞自訂資料時可用的選項：

```php
$checkout = $user->checkout('pri_tshirt')
    ->customData([
        'custom_option' => $value,
    ]);
```


<a name="refunding-transactions"></a>
### 交易退款

交易退款將把退款金額退回至客戶在購買時使用的付款方式。如果您需要退款 Paddle 購買項目，可以使用 `Cashier\Paddle\Transaction` 模型上的 `refund` 方法。此方法接受一個原因作為第一個引數，以及一個或多個要退款的價格 ID（可使用關聯陣列提供選擇性金額）。您可以使用 `transactions` 方法取得特定可計費模型的交易。

例如，假設我們想要退款價格為 `pri_123` 和 `pri_456` 的特定交易。我們想要全額退款 `pri_123`，但僅為 `pri_456` 退款兩美元：

```php
use App\Models\User;

$user = User::find(1);

$transaction = $user->transactions()->first();

$response = $transaction->refund('Accidental charge', [
    'pri_123', // Fully refund this price...
    'pri_456' => 200, // Only partially refund this price...
]);
```

上述範例退款了交易中的特定項目。如果您想要退款整個交易，只需提供原因即可：

```php
$response = $transaction->refund('Accidental charge');
```

如需更多關於退款的資訊，請參閱 [Paddle 的退款文件](https://developer.paddle.com/build/transactions/create-transaction-adjustments)。

> [!WARNING]
> 退款在完全處理前必須始終由 Paddle 核准。


<a name="crediting-transactions"></a>
### 交易入帳

與退款一樣，您也可以對交易進行入帳。交易入帳將把資金添加到客戶的餘額中，以便用於未來的購買。交易入帳僅能針對手動收集的交易，而不能針對自動收集的交易（例如訂閱），因為 Paddle 會自動處理訂閱入帳：

```php
$transaction = $user->transactions()->first();

// Credit a specific line item fully...
$response = $transaction->credit('Compensation', 'pri_123');
```

如需更多資訊，請[參閱 Paddle 關於入帳的文件](https://developer.paddle.com/build/transactions/create-transaction-adjustments)。

> [!WARNING]
> 入帳僅能應用於手動收集的交易。自動收集的交易由 Paddle 自行處理入帳。


<a name="transactions"></a>
## 交易

您可以透過 `transactions` 屬性輕鬆取得可計費模型交易的陣列：

```php
use App\Models\User;

$user = User::find(1);

$transactions = $user->transactions;
```

交易代表您產品和購買項目的付款，並附有發票。只有已完成的交易才會儲存在您應用程式的資料庫中。

在列出客戶的交易時，您可以使用交易實例的方法來顯示相關的付款資訊。例如，您可能希望在表格中列出每筆交易，讓使用者能輕鬆下載任何發票：

```html
<table>
    @foreach ($transactions as $transaction)
        <tr>
            <td>{{ $transaction->billed_at->toFormattedDateString() }}</td>
            <td>{{ $transaction->total() }}</td>
            <td>{{ $transaction->tax() }}</td>
            <td><a href="{{ route('download-invoice', $transaction->id) }}" target="_blank">Download</a></td>
        </tr>
    @endforeach
</table>
```

`download-invoice` 路由可能如下所示：

```php
use Illuminate\Http\Request;
use Laravel\Paddle\Transaction;

Route::get('/download-invoice/{transaction}', function (Request $request, Transaction $transaction) {
    return $transaction->redirectToInvoicePdf();
})->name('download-invoice');
```


<a name="past-and-upcoming-payments"></a>
### 過去與即將到來的付款

您可以使用 `lastPayment` 和 `nextPayment` 方法來取得並顯示週期性訂閱客戶過去或即將到來的付款：

```php
use App\Models\User;

$user = User::find(1);

$subscription = $user->subscription();

$lastPayment = $subscription->lastPayment();
$nextPayment = $subscription->nextPayment();
```

這兩個方法都會回傳一個 `Laravel\Paddle\Payment` 實例；然而，當交易尚未透過 Webhook 同步時，`lastPayment` 將回傳 `null`，而當計費週期結束時（例如訂閱已被取消），`nextPayment` 將回傳 `null`：

```blade
Next payment: {{ $nextPayment->amount() }} due on {{ $nextPayment->date()->format('d/m/Y') }}
```


<a name="testing"></a>
## 測試

在測試期間，您應該手動測試您的計費流程，以確保您的整合運作如預期。

對於自動化測試（包括在 CI 環境中執行的測試），您可以使用 [Laravel 的 HTTP 用戶端](/docs/{{version}}/http-client#testing) 來模擬發送到 Paddle 的 HTTP 呼叫。雖然這不會測試來自 Paddle 的實際回應，但它提供了一種在不實際呼叫 Paddle API 的情況下測試應用程式的方法。