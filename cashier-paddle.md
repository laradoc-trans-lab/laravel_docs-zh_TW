# Laravel Cashier (Paddle)

- [簡介](#introduction)
- [升級 Cashier](#upgrading-cashier)
- [安裝](#installation)
    - [Paddle 沙盒](#paddle-sandbox)
- [設定](#configuration)
    - [可計費模型 (Billable Model)](#billable-model)
    - [API 金鑰](#api-keys)
    - [Paddle JS](#paddle-js)
    - [貨幣設定](#currency-configuration)
    - [覆寫預設模型](#overriding-default-models)
- [快速入門](#quickstart)
    - [販售產品](#quickstart-selling-products)
    - [販售訂閱](#quickstart-selling-subscriptions)
- [結帳 Session](#checkout-sessions)
    - [覆蓋式結帳 (Overlay Checkout)](#overlay-checkout)
    - [內嵌式結帳 (Inline Checkout)](#inline-checkout)
    - [訪客結帳](#guest-checkouts)
- [價格預覽](#price-previews)
    - [顧客價格預覽](#customer-price-previews)
    - [折扣](#price-discounts)
- [顧客管理](#customers)
    - [顧客預設值](#customer-defaults)
    - [取得顧客資料](#retrieving-customers)
    - [建立顧客](#creating-customers)
- [訂閱](#subscriptions)
    - [建立訂閱](#creating-subscriptions)
    - [檢查訂閱狀態](#checking-subscription-status)
    - [訂閱單次收費](#subscription-single-charges)
    - [更新付款資訊](#updating-payment-information)
    - [變更方案](#changing-plans)
    - [訂閱數量](#subscription-quantity)
    - [包含多項產品的訂閱](#subscriptions-with-multiple-products)
    - [多重訂閱](#multiple-subscriptions)
    - [暫停訂閱](#pausing-subscriptions)
    - [取消訂閱](#canceling-subscriptions)
- [訂閱試用](#subscription-trials)
    - [預先提供付款方式](#with-payment-method-up-front)
    - [不預先提供付款方式](#without-payment-method-up-front)
    - [延長或啟用試用](#extend-or-activate-a-trial)
- [處理 Paddle Webhook](#handling-paddle-webhooks)
    - [定義 Webhook 事件處理器](#defining-webhook-event-handlers)
    - [驗證 Webhook 簽章](#verifying-webhook-signatures)
- [單次扣款](#single-charges)
    - [對產品進行扣款](#charging-for-products)
    - [交易退款](#refunding-transactions)
    - [交易折抵 (Crediting)](#crediting-transactions)
- [交易紀錄](#transactions)
    - [歷史與未來付款](#past-and-upcoming-payments)
- [測試](#testing)

<a name="introduction"></a>
## 簡介

> [!WARNING]
> 本文件適用於 Cashier Paddle 2.x 與 Paddle Billing 的整合。如果您仍在使用 Paddle Classic，應使用 [Cashier Paddle 1.x](https://github.com/laravel/cashier-paddle/tree/1.x)。

[Laravel Cashier Paddle](https://github.com/laravel/cashier-paddle) 為 [Paddle](https://paddle.com) 的訂閱計費服務提供了一個極具表達力且流暢的介面。它幾乎處理了所有您令人頭疼的訂閱計費樣板程式碼。除了基本的訂閱管理之外，Cashier 還可以處理：切換訂閱方案、訂閱「數量」、暫停訂閱、取消訂閱的寬限期等等。

在深入瞭解 Cashier Paddle 之前，建議您也先閱讀 Paddle 的[概念指南](https://developer.paddle.com/concepts/overview)與 [API 文件](https://developer.paddle.com/api-reference/overview)。


<a name="upgrading-cashier"></a>
## 升級 Cashier

當升級至新版本的 Cashier 時，請務必仔細閱讀[升級指南](https://github.com/laravel/cashier-paddle/blob/master/UPGRADE.md)。


<a name="installation"></a>
## 安裝

首先，使用 Composer 套件管理器安裝 Paddle 版的 Cashier 套件：

```shell
composer require laravel/cashier-paddle
```

接下來，您應該使用 `vendor:publish` Artisan 指令發布 Cashier 遷移檔：

```shell
php artisan vendor:publish --tag="cashier-migrations"
```

然後，您應該執行應用程式的資料庫遷移。Cashier 遷移將會建立一個新的 `customers` 資料表。此外，還會建立新的 `subscriptions` 與 `subscription_items` 資料表來儲存您顧客所有的訂閱資料。最後，會建立一個新的 `transactions` 資料表來儲存與您顧客相關的所有 Paddle 交易紀錄：

```shell
php artisan migrate
```

> [!WARNING]
> 為確保 Cashier 能正常處理所有 Paddle 事件，請記得[設定 Cashier 的 Webhook 處理](#handling-paddle-webhooks)。


<a name="paddle-sandbox"></a>
### Paddle 沙盒

在本機與測試 (Staging) 環境開發期間，您應該[註冊一個 Paddle 沙盒帳號](https://sandbox-login.paddle.com/signup)。此帳號提供沙盒環境，供您在無需實際付款的情況下測試與開發應用程式。您可以使用 Paddle 的[測試卡號](https://developer.paddle.com/concepts/payment-methods/credit-debit-card#test-payment-method)來模擬各種付款情境。

使用 Paddle 沙盒環境時，您應該在應用程式的 `.env` 檔案中將 `PADDLE_SANDBOX` 環境變數設定為 `true`：

```ini
PADDLE_SANDBOX=true
```

完成應用程式開發後，您可以[申請 Paddle 廠商 (Vendor) 帳號](https://paddle.com)。在您的應用程式上線至正式環境前，Paddle 需要審核並核准您的應用程式網域。


<a name="configuration"></a>
## 設定


<a name="billable-model"></a>
### 可計費模型 (Billable Model)

在使用 Cashier 之前，您必須將 `Billable` trait 新增至 User 模型定義中。此 trait 提供各種方法，讓您能執行常見的計費任務，例如建立訂閱以及更新付款方式資訊：

```php
use Laravel\Paddle\Billable;

class User extends Authenticatable
{
    use Billable;
}
```

如果您有非使用者但需要計費的實體，也可以將此 trait 新增至這些類別中：

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

接下來，您應該在應用程式的 `.env` 檔案中設定您的 Paddle 金鑰。您可以從 Paddle 管理控制台取得您的 Paddle API 金鑰：

```ini
PADDLE_CLIENT_SIDE_TOKEN=your-paddle-client-side-token
PADDLE_API_KEY=your-paddle-api-key
PADDLE_RETAIN_KEY=your-paddle-retain-key
PADDLE_WEBHOOK_SECRET="your-paddle-webhook-secret"
PADDLE_SANDBOX=true
```

當您使用 [Paddle 的沙盒環境](#paddle-sandbox) 時，`PADDLE_SANDBOX` 環境變數應設定為 `true`。若是將應用程式部署至正式環境並使用 Paddle 的線上廠商環境，則 `PADDLE_SANDBOX` 變數應設定為 `false`。

`PADDLE_RETAIN_KEY` 是選填的，僅在您結合 Paddle 使用 [Retain](https://developer.paddle.com/concepts/retain/overview) 時才需要設定。


<a name="paddle-js"></a>
### Paddle JS

Paddle 依賴其本身的 JavaScript 函式庫來啟動 Paddle 結帳元件 (Checkout Widget)。您可以將 `@paddleJS` Blade 指令放在應用程式版面配置的 `</head>` 結束標籤正前方，以載入該 JavaScript 函式庫：

```blade
<head>
    ...

    @paddleJS
</head>
```


<a name="currency-configuration"></a>
### 貨幣設定

您可以指定在發票上顯示金額格式時所使用的語系 (Locale)。在內部，Cashier 會利用 [PHP 的 `NumberFormatter` 類別](https://www.php.net/manual/en/class.numberformatter.php)來設定貨幣語系：

```ini
CASHIER_CURRENCY_LOCALE=nl_BE
```

> [!WARNING]
> 若要使用 `en` 以外的語系，請確保伺服器上已安裝並設定 `ext-intl` PHP 擴充功能。


<a name="overriding-default-models"></a>
### 覆寫預設模型

您可以隨意擴充 Cashier 內部使用的模型，方法是定義自己的模型並繼承對應的 Cashier 模型：

```php
use Laravel\Paddle\Subscription as CashierSubscription;

class Subscription extends CashierSubscription
{
    // ...
}
```

定義好模型後，您可以透過 `Laravel\Paddle\Cashier` 類別指示 Cashier 使用您的自訂模型。通常，您應該在應用程式的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中告知 Cashier 關於您的自訂模型：

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
### 販售產品

> [!NOTE]
> 在使用 Paddle 結帳之前，您應該先在 Paddle 控制面板中定義具有固定價格的產品。此外，您還應該[設定 Paddle 的 Webhook 處理](#handling-paddle-webhooks)。

透過您的應用程式提供產品與訂閱計費可能會令人感到繁瑣。然而，多虧了 Cashier 與 [Paddle 的覆蓋式結帳 (Checkout Overlay)](https://developer.paddle.com/concepts/sell/overlay-checkout)，您可以輕鬆建立現代且強大的付款整合。

要向顧客收取非循環、單次扣款的產品費用，我們將利用 Cashier 透過 Paddle 的覆蓋式結帳來向顧客扣款，顧客將在其中提供其付款詳細資訊並確認購買。一旦透過覆蓋式結帳完成付款，顧客將會被重定向至您在應用程式中指定的成功網址 (URL)：

```php
use Illuminate\Http\Request;

Route::get('/buy', function (Request $request) {
    $checkout = $request->user()->checkout('pri_deluxe_album')
        ->returnTo(route('dashboard'));

    return view('buy', ['checkout' => $checkout]);
})->name('checkout');
```

如上方的範例所示，我們將利用 Cashier 提供的 `checkout` 方法建立一個結帳物件，以針對給定的「價格標識符 (Price Identifier)」向顧客呈現 Paddle 覆蓋式結帳。使用 Paddle 時，「價格 (Prices)」是指[特定產品所定義的價格](https://developer.paddle.com/build/products/create-products-prices)。

如有需要，`checkout` 方法會自動在 Paddle 中建立一位顧客，並將該 Paddle 顧客紀錄連結至您應用程式資料庫中的對應使用者。完成結帳 Session 後，顧客將被重定向至專門的成功頁面，您可以在該頁面上向顧客顯示提示訊息。

在 `buy` 視圖中，我們將包含一個用於顯示覆蓋式結帳的按鈕。Cashier Paddle 隨附了 `paddle-button` Blade 元件；不過，您也可以[手動渲染覆蓋式結帳](#manually-rendering-an-overlay-checkout)：

```html
<x-paddle-button :checkout="$checkout" class="px-8 py-4">
    Buy Product
</x-paddle-button>
```


<a name="providing-meta-data-to-paddle-checkout"></a>
#### 提供元資料至 Paddle 結帳

販售產品時，通常會透過您自己應用程式所定義的 `Cart` 與 `Order` 模型來追蹤已完成的訂單與購買的產品。當重定向顧客至 Paddle 的覆蓋式結帳以完成購買時，您可能需要提供現有的訂單標識符，以便在顧客被重定向回您的應用程式時，將已完成的購買與對應的訂單關聯起來。

為實現此目的，您可以向 `checkout` 方法提供自訂資料的陣列。假設當使用者開始結帳流程時，我們的應用程式內會建立一個待處理的 `Order`。請記住，此範例中的 `Cart` 與 `Order` 模型僅作為說明示範，並非由 Cashier 提供。您可以根據自己應用程式的需求自由實現這些概念：

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

如上方的範例所示，當使用者開始結帳流程時，我們將提供所有與購物車/訂單相關聯的 Paddle 價格標識符給 `checkout` 方法。當然，您的應用程式負責在顧客新增項目時將這些項目與「購物車」或訂單進行關聯。我們還透過 `customData` 方法將訂單 ID 提供給 Paddle 覆蓋式結帳。

當然，顧客完成結帳流程後，您可能希望將訂單標示為「完成」。為達成此目的，您可以監聽由 Paddle 發出並透過 Cashier 以事件形式觸發的 Webhook，從而將訂單資訊儲存在資料庫中。

首先，請監聽由 Cashier 發出的 `TransactionCompleted` 事件。通常，您應該在應用程式 `AppServiceProvider` 的 `boot` 方法中註冊該事件監聽器：

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

請參考 Paddle 的官方文件，以了解更多關於 [`transaction.completed` 事件包含的資料](https://developer.paddle.com/webhooks/transactions/transaction-completed) 的詳細資訊。

<a name="quickstart-selling-subscriptions"></a>
### 販售訂閱

> [!NOTE]
> 在使用 Paddle Checkout 之前，您應該先在 Paddle 控制面板中定義具有固定價格的產品。此外，您還應該[設定 Paddle 的 Webhook 處理](#handling-paddle-webhooks)。

透過應用程式提供產品和訂閱計費功能可能會讓人望而生畏。然而，多虧了 Cashier 和 [Paddle 的 Checkout Overlay](https://developer.paddle.com/concepts/sell/overlay-checkout)，您可以輕鬆建立現代且強大的付款整合。

要瞭解如何使用 Cashier 與 Paddle 的 Checkout Overlay 販售訂閱，讓我們考慮一個簡單的情境：一個具有基礎月繳 (`price_basic_monthly`) 與年繳 (`price_basic_yearly`) 方案的訂閱服務。這兩種價格可以在我們的 Paddle 控制面板中歸類在 "Basic" 產品 (`pro_basic`) 下。此外，我們的訂閱服務可能還會提供一個名為 `pro_expert` 的 "Expert" 方案。

首先，讓我們瞭解顧客如何訂閱我們的服務。當然，您可以想像顧客可能會在我們應用程式的價格頁面上點擊 Basic 方案的「訂閱」按鈕。此按鈕將為他們選擇的方案調用 Paddle Checkout Overlay。首先，讓我們透過 `checkout` 方法來發起一個結帳 Session：

```php
use Illuminate\Http\Request;

Route::get('/subscribe', function (Request $request) {
    $checkout = $request->user()->checkout('price_basic_monthly')
        ->returnTo(route('dashboard'));

    return view('subscribe', ['checkout' => $checkout]);
})->name('subscribe');
```

在 `subscribe` 視圖中，我們將包含一個用於顯示 Checkout Overlay 的按鈕。`paddle-button` Blade 元件已內建於 Cashier Paddle 中；不過，您也可以[手動渲染覆蓋式結帳](#manually-rendering-an-overlay-checkout)：

```html
<x-paddle-button :checkout="$checkout" class="px-8 py-4">
    Subscribe
</x-paddle-button>
```

現在，當點擊訂閱按鈕時，顧客將能夠輸入其付款詳細資訊並發起訂閱。為了得知訂閱何時真正開始（因為某些付款方式需要幾秒鐘的處理時間），您還應該[設定 Cashier 的 Webhook 處理](#handling-paddle-webhooks)。

現在顧客可以開始訂閱了，我們需要限制應用程式的特定部分，讓只有已訂閱的使用者能夠存取。當然，我們隨時可以透過 Cashier 的 `Billable` trait 所提供的 `subscribed` 方法來確認使用者目前的訂閱狀態：

```blade
@if ($user->subscribed())
    <p>You are subscribed.</p>
@endif
```

我們甚至可以輕鬆判定使用者是否訂閱了特定產品或價格：

```blade
@if ($user->subscribedToProduct('pro_basic'))
    <p>You are subscribed to our Basic product.</p>
@endif

@if ($user->subscribedToPrice('price_basic_monthly'))
    <p>You are subscribed to our monthly Basic plan.</p>
@endif
```

<a name="quickstart-building-a-subscribed-middleware"></a>
#### 建立已訂閱中介層

為了方便起見，您可能希望建立一個[中介層](/docs/{{version}}/middleware)，用來檢查傳入的請求是否來自已訂閱的使用者。一旦定義了這個中介層，您就可以輕鬆地將其指派給路由，以防止未訂閱的使用者存取該路由：

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

定義好中介層後，您可以將其指派給路由：

```php
use App\Http\Middleware\Subscribed;

Route::get('/dashboard', function () {
    // ...
})->middleware([Subscribed::class]);
```

<a name="quickstart-allowing-customers-to-manage-their-billing-plan"></a>
#### 允許顧客管理他們的計費方案

當然，顧客可能會想將他們的訂閱方案變更為另一個產品或「層級 (tier)」。在我們上面的範例中，我們會希望允許顧客將他們的方案從月繳訂閱變更為年繳訂閱。為此，您需要實作類似導向至下方路由的按鈕：

```php
use Illuminate\Http\Request;

Route::put('/subscription/{price}/swap', function (Request $request, $price) {
    $user->subscription()->swap($price); // With "$price" being "price_basic_yearly" for this example.

    return redirect()->route('dashboard');
})->name('subscription.swap');
```

除了切換方案之外，您還需要允許顧客取消他們的訂閱。就像切換方案一樣，請提供一個導向至以下路由的按鈕：

```php
use Illuminate\Http\Request;

Route::put('/subscription/cancel', function (Request $request, $price) {
    $user->subscription()->cancel();

    return redirect()->route('dashboard');
})->name('subscription.cancel');
```

現在，您的訂閱將會在計費週期結束時被取消。

> [!NOTE]
> 只要您設定好了 Cashier 的 Webhook 處理，Cashier 就會透過檢查來自 Paddle 的傳入 Webhook，自動讓您應用程式中與 Cashier 相關的資料庫資料表保持同步。因此，舉例來說，當您透過 Paddle 的控制面板取消顧客的訂閱時，Cashier 會收到對應的 Webhook，並在您的應用程式資料庫中將該訂閱標記為「已取消」。

<a name="checkout-sessions"></a>
## 結帳 Session

向顧客計費的大多數操作，都是透過 Paddle 的 [Checkout Overlay 元件](https://developer.paddle.com/build/checkout/build-overlay-checkout) 或使用 [內嵌式結帳 (Inline Checkout)](https://developer.paddle.com/build/checkout/build-branded-inline-checkout) 來完成「結帳」。

在使用 Paddle 處理結帳付款之前，您應該在 Paddle 的結帳設定儀表板中定義應用程式的 [預設付款連結](https://developer.paddle.com/build/transactions/default-payment-link#set-default-link)。


<a name="overlay-checkout"></a>
### 覆蓋式結帳 (Overlay Checkout)

在顯示 Checkout Overlay 小工具之前，您必須使用 Cashier 產生一個結帳 Session。結帳 Session 會告知結帳小工具應該執行哪種計費操作：

```php
use Illuminate\Http\Request;

Route::get('/buy', function (Request $request) {
    $checkout = $user->checkout('pri_34567')
        ->returnTo(route('dashboard'));

    return view('billing', ['checkout' => $checkout]);
});
```

Cashier 包含了一個 `paddle-button` [Blade 元件](/docs/{{version}}/blade#components)。您可以將結帳 Session 作為 "prop" 傳遞給此元件。然後，當按下此按鈕時，就會顯示 Paddle 的結帳小工具：

```html
<x-paddle-button :checkout="$checkout" class="px-8 py-4">
    Subscribe
</x-paddle-button>
```

預設情況下，這會使用 Paddle 的預設樣式來顯示小工具。您可以透過為元件新增 [Paddle 支援的屬性](https://developer.paddle.com/paddlejs/html-data-attributes)（例如 `data-theme='light'` 屬性）來自訂小工具：

```html
<x-paddle-button :checkout="$checkout" class="px-8 py-4" data-theme="light">
    Subscribe
</x-paddle-button>
```

Paddle 結帳小工具是非同步執行的。一旦使用者在小工具內建立訂閱，Paddle 就會向您的應用程式傳送 Webhook，以便您可以在應用程式的資料庫中正確更新訂閱狀態。因此，請務必正確[設定 Webhook](#handling-paddle-webhooks) 以因應來自 Paddle 的狀態變更。

> [!WARNING]
> 在訂閱狀態變更後，收到對應 Webhook 的延遲時間通常極短，但您仍應在應用程式中考慮這一點，意即使用者的訂閱在完成結帳後可能無法立即可用。


<a name="manually-rendering-an-overlay-checkout"></a>
#### 手動渲染覆蓋式結帳

您也可以手動渲染覆蓋式結帳，而不使用 Laravel 內建的 Blade 元件。首先，請[如先前範例所示](#overlay-checkout)產生結帳 Session：

```php
use Illuminate\Http\Request;

Route::get('/buy', function (Request $request) {
    $checkout = $user->checkout('pri_34567')
        ->returnTo(route('dashboard'));

    return view('billing', ['checkout' => $checkout]);
});
```

接著，您可以使用 Paddle.js 來初始化結帳。在這個範例中，我們將建立一個被指派 `paddle_button` 類別 (class) 的連結。Paddle.js 會偵測到此類別，並在點擊該連結時顯示覆蓋式結帳：

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

如果您不想使用 Paddle 的「覆蓋 (overlay)」風格結帳小工具，Paddle 也提供了以內嵌 (inline) 方式顯示小工具的選項。雖然這種方法不允許您調整任何結帳的 HTML 欄位，但它允許您將小工具嵌入到您的應用程式中。

為了方便您快速上手內嵌式結帳，Cashier 包含了一個 `paddle-checkout` Blade 元件。首先，您應該[產生一個結帳 Session](#overlay-checkout)：

```php
use Illuminate\Http\Request;

Route::get('/buy', function (Request $request) {
    $checkout = $user->checkout('pri_34567')
        ->returnTo(route('dashboard'));

    return view('billing', ['checkout' => $checkout]);
});
```

然後，您可以將結帳 Session 傳遞給該元件的 `checkout` 屬性：

```blade
<x-paddle-checkout :checkout="$checkout" class="w-full" />
```

若要調整內嵌式結帳元件的高度，您可以將 `height` 屬性傳遞給該 Blade 元件：

```blade
<x-paddle-checkout :checkout="$checkout" class="w-full" height="500" />
```

請參閱 Paddle 的 [內嵌式結帳指南](https://developer.paddle.com/build/checkout/build-branded-inline-checkout) 以及 [可用的結帳設定](https://developer.paddle.com/build/checkout/set-up-checkout-default-settings)，瞭解更多關於內嵌式結帳自訂選項的詳細資訊。


<a name="manually-rendering-an-inline-checkout"></a>
#### 手動渲染內嵌式結帳

您也可以手動渲染內嵌式結帳，而不使用 Laravel 內建的 Blade 元件。首先，請[如先前範例所示](#inline-checkout)產生結帳 Session：

```php
use Illuminate\Http\Request;

Route::get('/buy', function (Request $request) {
    $checkout = $user->checkout('pri_34567')
        ->returnTo(route('dashboard'));

    return view('billing', ['checkout' => $checkout]);
});
```

接著，您可以使用 Paddle.js 來初始化結帳。在底下範例中，我們將使用 [Alpine.js](https://github.com/alpinejs/alpine) 進行示範；不過，您可以根據自己使用的前端技術棧自由修改此範例：

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

有時候，您可能需要為不需要在您的應用程式中建立帳號的使用者建立結帳 Session。為此，您可以使用 `guest` 方法：

```php
use Illuminate\Http\Request;
use Laravel\Paddle\Checkout;

Route::get('/buy', function (Request $request) {
    $checkout = Checkout::guest(['pri_34567'])
        ->returnTo(route('home'));

    return view('billing', ['checkout' => $checkout]);
});
```

然後，您可以將結帳 Session 提供給 [Paddle 按鈕](#overlay-checkout) 或 [內嵌式結帳](#inline-checkout) 的 Blade 元件。

<a name="price-previews"></a>
## 價格預覽

Paddle 允許您根據不同的貨幣自訂價格，這讓您可以為不同的國家設定不同的價格。Cashier Paddle 允許您使用 `previewPrices` 方法取得所有這些價格。此方法接受您想要取得價格的價格 ID：

```php
use Laravel\Paddle\Cashier;

$prices = Cashier::previewPrices(['pri_123', 'pri_456']);
```

幣別將根據請求的 IP 位址決定；不過，您也可以選擇性地提供特定國家來取得該國家的價格：

```php
use Laravel\Paddle\Cashier;

$prices = Cashier::previewPrices(['pri_123', 'pri_456'], ['address' => [
    'country_code' => 'BE',
    'postal_code' => '1234',
]]);
```

取得價格後，您可以依據需求任意顯示它們：

```blade
<ul>
    @foreach ($prices as $price)
        <li>{{ $price->product['name'] }} - {{ $price->total() }}</li>
    @endforeach
</ul>
```

您也可以分別顯示未稅小計價格與稅額：

```blade
<ul>
    @foreach ($prices as $price)
        <li>{{ $price->product['name'] }} - {{ $price->subtotal() }} (+ {{ $price->tax() }} tax)</li>
    @endforeach
</ul>
```

如需更多資訊，請[參閱 Paddle 關於價格預覽的 API 文件](https://developer.paddle.com/api-reference/pricing-preview/preview-prices)。


<a name="customer-price-previews"></a>
### 顧客價格預覽

如果使用者已經是顧客，且您希望顯示適用於該顧客的價格，您可以直接從顧客實例取得價格：

```php
use App\Models\User;

$prices = User::find(1)->previewPrices(['pri_123', 'pri_456']);
```

在內部，Cashier 會使用使用者的顧客 ID 來取得對應其貨幣的價格。因此，舉例來說，居住在美國的使用者將看到以美元計價的價格，而比利時的使用者則會看到歐元價格。如果找不到相符的貨幣，則會使用產品的預設貨幣。您可以在 Paddle 控制台自訂產品或訂閱方案的所有價格。


<a name="price-discounts"></a>
### 折扣

您也可以選擇顯示套用折扣後的價格。呼叫 `previewPrices` 方法時，您可以透過 `discount_id` 選項提供折扣 ID：

```php
use Laravel\Paddle\Cashier;

$prices = Cashier::previewPrices(['pri_123', 'pri_456'], [
    'discount_id' => 'dsc_123'
]);
```

接著，顯示計算後的價格：

```blade
<ul>
    @foreach ($prices as $price)
        <li>{{ $price->product['name'] }} - {{ $price->total() }}</li>
    @endforeach
</ul>
```


<a name="customers"></a>
## 顧客管理


<a name="customer-defaults"></a>
### 顧客預設值

Cashier 允許您在建立結帳 Session 時為顧客定義一些實用的預設值。設定這些預設值可讓您預先填寫顧客的 Email 與姓名，讓他們能立即進入結帳元件的付款環節。您可以在可計費模型 (Billable Model) 上透過覆寫以下方法來設定這些預設值：

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

這些預設值將用於 Cashier 中每個會產生 [結帳 Session](#checkout-sessions) 的動作。


<a name="retrieving-customers"></a>
### 取得顧客資料

您可以使用 `Cashier::findBillable` 方法透過顧客的 Paddle Customer ID 取得該顧客。此方法將回傳可計費模型的實例：

```php
use Laravel\Paddle\Cashier;

$user = Cashier::findBillable($customerId);
```


<a name="creating-customers"></a>
### 建立顧客

有時候，您可能希望在尚未開始訂閱的情況下建立 Paddle 顧客。您可以使用 `createAsCustomer` 方法來達成：

```php
$customer = $user->createAsCustomer();
```

此方法會回傳一個 `Laravel\Paddle\Customer` 實例。在 Paddle 中建立顧客後，您可以在日後再開始建立訂閱。您可以傳入可選的 `$options` 陣列，以帶入任何 [Paddle API 支援的顧客建立參數](https://developer.paddle.com/api-reference/customers/create-customer)：

```php
$customer = $user->createAsCustomer($options);
```

<a name="subscriptions"></a>
## 訂閱


<a name="creating-subscriptions"></a>
### 建立訂閱

要建立訂閱，首先需從資料庫中取得可計費模型的實例，這通常會是 `App\Models\User` 的實例。取得模型實例後，您可以使用 `subscribe` 方法來建立該模型的結帳 Session：

```php
use Illuminate\Http\Request;

Route::get('/user/subscribe', function (Request $request) {
    $checkout = $request->user()->subscribe($premium = 'pri_123', 'default')
        ->returnTo(route('home'));

    return view('billing', ['checkout' => $checkout]);
});
```

傳給 `subscribe` 方法的第一個引數是使用者要訂閱的特定價格。此數值應對應至 Paddle 中的價格識別碼。`returnTo` 方法接收一個 URL，使用者在成功完成結帳後會被重導向至該位置。傳給 `subscribe` 方法的第二個引數應該是訂閱的內部「類型 (type)」。如果您的應用程式只提供單一訂閱，您可以將其稱為 `default` 或 `primary`。此訂閱類型僅供內部應用程式使用，不適合顯示給使用者。此外，它不應包含空格，且在建立訂閱後絕不應該更改。

您也可以使用 `customData` 方法傳入與訂閱相關的自訂元資料陣列：

```php
$checkout = $request->user()->subscribe($premium = 'pri_123', 'default')
    ->customData(['key' => 'value'])
    ->returnTo(route('home'));
```

建立訂閱結帳 Session 後，可以將該結帳 Session 傳給 Cashier Paddle 隨附的 `paddle-button` [Blade 元件](#overlay-checkout)：

```blade
<x-paddle-button :checkout="$checkout" class="px-8 py-4">
    Subscribe
</x-paddle-button>
```

在使用者完成結帳後，Paddle 會發送 `subscription_created` Webhook。Cashier 會接收此 Webhook 並為您的顧客設定訂閱。為了確保所有 Webhook 都能被您的應用程式正確接收與處理，請務必正確[設定 Webhook 處理](#handling-paddle-webhooks)。


<a name="checking-subscription-status"></a>
### 檢查訂閱狀態

使用者訂閱您的應用程式後，您可以使用各種便利的方法來檢查他們的訂閱狀態。首先，如果使用者擁有有效的訂閱，即使訂閱目前仍在試用期內，`subscribed` 方法也會回傳 `true`：

```php
if ($user->subscribed()) {
    // ...
}
```

如果您的應用程式提供多種訂閱，可以在呼叫 `subscribed` 方法時指定訂閱：

```php
if ($user->subscribed('default')) {
    // ...
}
```

`subscribed` 方法也非常適合用於[路由中介層](/docs/{{version}}/middleware)，讓您能根據使用者的訂閱狀態來限制對路由與控制器的存取：

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

如果您想確定使用者是否仍處於試用期內，可以使用 `onTrial` 方法。此方法對於決定是否向使用者顯示提示警告（說明他們仍處於試用期）非常有幫助：

```php
if ($user->subscription()->onTrial()) {
    // ...
}
```

`subscribedToPrice` 方法可用於根據給定的 Paddle 價格 ID 確定使用者是否已訂閱給定的方案。在此範例中，我們將判斷使用者的 `default` 訂閱是否已主動訂閱月費價格：

```php
if ($user->subscribedToPrice($monthly = 'pri_123', 'default')) {
    // ...
}
```

`recurring` 方法可以用於確定使用者目前是否為有效訂閱中，且不再處於試用期或寬限期：

```php
if ($user->subscription()->recurring()) {
    // ...
}
```


<a name="canceled-subscription-status"></a>
#### 已取消的訂閱狀態

要確定使用者是否曾是活躍訂閱者但已取消其訂閱，可以使用 `canceled` 方法：

```php
if ($user->subscription()->canceled()) {
    // ...
}
```

您也可以確定使用者是否已取消訂閱，但目前仍處於「寬限期」直至訂閱完全過期。例如，若使用者在 3 月 5 日取消原本預計 3 月 10 日過期的訂閱，則使用者在 3 月 10 日前均處於「寬限期」。此外，在此期間，`subscribed` 方法依然會回傳 `true`：

```php
if ($user->subscription()->onGracePeriod()) {
    // ...
}
```


<a name="past-due-status"></a>
#### 逾期狀態

如果訂閱付款失敗，將被標示為 `past_due`。當您的訂閱處於此狀態時，在顧客更新其付款資訊之前，訂閱將不會生效。您可以使用訂閱實例上的 `pastDue` 方法來確定訂閱是否逾期：

```php
if ($user->subscription()->pastDue()) {
    // ...
}
```

當訂閱逾期時，您應該指示使用者[更新其付款資訊](#updating-payment-information)。

如果您希望訂閱在 `past_due` 狀態下仍被視為有效，您可以使用 Cashier 提供的 `keepPastDueSubscriptionsActive` 方法。通常，此方法應在 `AppServiceProvider` 的 `register` 方法中呼叫：

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
> 當訂閱處於 `past_due` 狀態時，在付款資訊更新之前無法進行任何變更。因此，當訂閱處於 `past_due` 狀態時，`swap` 和 `updateQuantity` 方法將會拋出例外。


<a name="subscription-scopes"></a>
#### 訂閱查詢範圍

大多數訂閱狀態也可以作為查詢範圍 (Query Scopes) 使用，以便您輕鬆在資料庫中查詢處於特定狀態的訂閱：

```php
// Get all valid subscriptions...
$subscriptions = Subscription::query()->valid()->get();

// Get all of the canceled subscriptions for a user...
$subscriptions = $user->subscriptions()->canceled()->get();
```

所有可用查詢範圍的完整列表如下：

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

訂閱單次收費允許您在訂閱費用之外，額外向訂閱者收取一次性的費用。當呼叫 `charge` 方法時，您必須提供一個或多個價格 ID：

```php
// Charge a single price...
$response = $user->subscription()->charge('pri_123');

// Charge multiple prices at once...
$response = $user->subscription()->charge(['pri_123', 'pri_456']);
```

`charge` 方法不會立即向顧客扣款，而是會等到顧客訂閱的下一個計費週期才進行扣款。如果您想立即向顧客開立帳單並扣款，可以使用 `chargeAndInvoice` 方法替代：

```php
$response = $user->subscription()->chargeAndInvoice('pri_123');
```


<a name="updating-payment-information"></a>
### 更新付款資訊

Paddle 總是會為每個訂閱儲存一種付款方式。如果您想要更新訂閱的預設付款方式，應使用訂閱模型上的 `redirectToUpdatePaymentMethod` 方法，將您的顧客重新導向至 Paddle 代管的付款方式更新頁面：

```php
use Illuminate\Http\Request;

Route::get('/update-payment-method', function (Request $request) {
    $user = $request->user();

    return $user->subscription()->redirectToUpdatePaymentMethod();
});
```

當使用者完成更新資訊後，Paddle 會發送一個 `subscription_updated` Webhook，且訂閱詳細資料將會更新於您應用程式的資料庫中。


<a name="changing-plans"></a>
### 變更方案

當使用者訂閱了您的應用程式後，有時可能會想更換為新的訂閱方案。若要為使用者更新訂閱方案，您應該將 Paddle 價格識別碼傳入訂閱的 `swap` 方法：

```php
use App\Models\User;

$user = User::find(1);

$user->subscription()->swap($premium = 'pri_456');
```

如果您想更換方案並立即向使用者開立帳單，而不是等待下一個計費週期，您可以使用 `swapAndInvoice` 方法：

```php
$user = User::find(1);

$user->subscription()->swapAndInvoice($premium = 'pri_456');
```


<a name="prorations"></a>
#### 按比例計費

預設情況下，Paddle 在更換方案時會按比例計算費用。可以使用 `noProrate` 方法來更新訂閱，而不按比例計算費用：

```php
$user->subscription('default')->noProrate()->swap($premium = 'pri_456');
```

如果您想停用按比例計費並立即向顧客開立帳單，可以將 `swapAndInvoice` 方法與 `noProrate` 搭配使用：

```php
$user->subscription('default')->noProrate()->swapAndInvoice($premium = 'pri_456');
```

或者，如果不希望為訂閱變更向您的顧客收取費用，可以使用 `doNotBill` 方法：

```php
$user->subscription('default')->doNotBill()->swap($premium = 'pri_456');
```

關於 Paddle 按比例計費政策的更多資訊，請參閱 Paddle 的 [按比例計費文件](https://developer.paddle.com/concepts/subscriptions/proration)。


<a name="subscription-quantity"></a>
### 訂閱數量

有時訂閱會受到「數量」影響。例如，專案管理應用程式可能會對每個專案每月收取 10 美元。若要輕鬆增加或減少訂閱數量，請使用 `incrementQuantity` 與 `decrementQuantity` 方法：

```php
$user = User::find(1);

$user->subscription()->incrementQuantity();

// Add five to the subscription's current quantity...
$user->subscription()->incrementQuantity(5);

$user->subscription()->decrementQuantity();

// Subtract five from the subscription's current quantity...
$user->subscription()->decrementQuantity(5);
```

或者，您可以使用 `updateQuantity` 方法設定特定數量：

```php
$user->subscription()->updateQuantity(10);
```

可以使用 `noProrate` 方法更新訂閱數量，而不按比例計算費用：

```php
$user->subscription()->noProrate()->updateQuantity(10);
```


<a name="quantities-for-subscription-with-multiple-products"></a>
#### 包含多項產品之訂閱的數量

如果您的訂閱是[包含多項產品的訂閱](#subscriptions-with-multiple-products)，您應該將想要增加或減少數量的價格 ID，作為第二個引數傳給遞增／遞減方法：

```php
$user->subscription()->incrementQuantity(1, 'price_chat');
```


<a name="subscriptions-with-multiple-products"></a>
### 包含多項產品的訂閱

[包含多項產品的訂閱](https://developer.paddle.com/build/subscriptions/add-remove-products-prices-addons)允許您將多個計費產品分配給單個訂閱。例如，想像您正在建立一個客戶服務「線上客服 (helpdesk)」應用程式，其基本訂閱價格為每月 10 美元，但額外提供每月 15 美元的真人即時通訊 (live chat) 加購產品。

建立訂閱結帳 Session 時，您可以透過傳入價格陣列作為 `subscribe` 方法的第一個引數，來為指定的訂閱指定多個產品：

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

在上面的範例中，顧客的 `default` 訂閱將附加兩個價格。這兩個價格都會在其各自的計費週期進行扣款。如有需要，您可以傳入一個鍵／值對的關聯陣列，來為每個價格指定特定數量：

```php
$user = User::find(1);

$checkout = $user->subscribe('default', ['price_monthly', 'price_chat' => 5]);
```

如果您想向現有訂閱新增另一個價格，則必須使用訂閱的 `swap` 方法。呼叫 `swap` 方法時，您也應該包含訂閱目前的價格與數量：

```php
$user = User::find(1);

$user->subscription()->swap(['price_chat', 'price_original' => 2]);
```

上面的範例會新增新價格，但在下一個計費週期之前不會向顧客收取該費用。如果您想立即向顧客開立帳單，可以使用 `swapAndInvoice` 方法：

```php
$user->subscription()->swapAndInvoice(['price_chat', 'price_original' => 2]);
```

您可以透過使用 `swap` 方法並忽略您想要移除的價格，來從訂閱中移除價格：

```php
$user->subscription()->swap(['price_original' => 2]);
```

> [!WARNING]
> 您不能移除訂閱中的最後一個價格。相反地，您應該直接取消訂閱。


<a name="multiple-subscriptions"></a>
### 多重訂閱

Paddle 允許您的顧客同時擁有重疊的多個訂閱。例如，您可能營運一家健身房，同時提供游泳訂閱與舉重訂閱，且每個訂閱都有不同的定價。當然，顧客應該能夠訂閱其中一種或兩種方案。

當您的應用程式建立訂閱時，您可以將訂閱類型作為第二個引數傳給 `subscribe` 方法。類型可以是代表使用者正在發起的訂閱類型的任何字串：

```php
use Illuminate\Http\Request;

Route::post('/swimming/subscribe', function (Request $request) {
    $checkout = $request->user()->subscribe($swimmingMonthly = 'pri_123', 'swimming');

    return view('billing', ['checkout' => $checkout]);
});
```

在這個範例中，我們為顧客發起了按月計費的游泳訂閱。然而，他們之後可能會想更換為按年計費的訂閱。當調整顧客的訂閱時，我們可以簡單地更換 `swimming` 訂閱上的價格：

```php
$user->subscription('swimming')->swap($swimmingYearly = 'pri_456');
```

當然，您也可以完全取消該訂閱：

```php
$user->subscription('swimming')->cancel();
```

<a name="pausing-subscriptions"></a>
### 暫停訂閱

若要暫停訂閱，可以在使用者的訂閱實例上呼叫 `pause` 方法：

```php
$user->subscription()->pause();
```

當訂閱被暫停時，Cashier 會自動設定資料庫中的 `paused_at` 欄位。該欄位用來決定 `paused` 方法何時開始回傳 `true`。例如，如果顧客在 3 月 1 日暫停訂閱，但該訂閱預計要到 3 月 5 日才會續扣，則 `paused` 方法在 3 月 5 日之前仍會繼續回傳 `false`。這是因為通常會允許使用者繼續使用應用程式直到其計費週期結束。

預設情況下，暫停會在下一個計費週期生效，因此顧客可以繼續使用他們已付費的剩餘期間。如果您希望立即暫停訂閱，可以使用 `pauseNow` 方法：

```php
$user->subscription()->pauseNow();
```

使用 `pauseUntil` 方法，您可以將訂閱暫停至特定的時間點：

```php
$user->subscription()->pauseUntil(now()->plus(months: 1));
```

或者，您可以使用 `pauseNowUntil` 方法立即暫停訂閱，直到給定的時間點為止：

```php
$user->subscription()->pauseNowUntil(now()->plus(months: 1));
```

您可以透過 `onPausedGracePeriod` 方法判斷使用者是否已暫停訂閱但仍處於「寬限期」：

```php
if ($user->subscription()->onPausedGracePeriod()) {
    // ...
}
```

若要恢復已暫停的訂閱，您可以在訂閱實例上呼叫 `resume` 方法：

```php
$user->subscription()->resume();
```

> [!WARNING]
> 訂閱在暫停狀態下無法進行修改。如果您想切換到不同的方案或更新數量，必須先恢復該訂閱。

<a name="canceling-subscriptions"></a>
### 取消訂閱

若要取消訂閱，請呼叫使用者訂閱實例上的 `cancel` 方法：

```php
$user->subscription()->cancel();
```

當訂閱被取消時，Cashier 會自動設定資料庫中的 `ends_at` 欄位。該欄位用來決定 `subscribed` 方法何時開始回傳 `false`。例如，如果顧客在 3 月 1 日取消訂閱，但該訂閱預計要到 3 月 5 日才結束，則 `subscribed` 方法在 3 月 5 日之前仍會繼續回傳 `true`。這樣做是因為通常會允許使用者繼續使用應用程式直到其計費週期結束。

您可以透過 `onGracePeriod` 方法判斷使用者是否已取消訂閱，但仍處於「寬限期」：

```php
if ($user->subscription()->onGracePeriod()) {
    // ...
}
```

如果您希望立即取消訂閱，可以在訂閱實例上呼叫 `cancelNow` 方法：

```php
$user->subscription()->cancelNow();
```

若要停止處於寬限期內的訂閱被取消，您可以呼叫 `stopCancelation` 方法：

```php
$user->subscription()->stopCancelation();
```

> [!WARNING]
> Paddle 的訂閱在取消後無法恢復。如果您的顧客希望重新訂閱，他們必須建立一個新的訂閱。

<a name="subscription-trials"></a>
## 訂閱試用


<a name="with-payment-method-up-front"></a>
### 預先提供付款方式

如果你希望在預先收集付款方式資訊的同時向顧客提供試用期，你應該在 Paddle 控制台中顧客所訂閱的價格上設定試用時間。接著，照常發起結帳 Session：

```php
use Illuminate\Http\Request;

Route::get('/user/subscribe', function (Request $request) {
    $checkout = $request->user()
        ->subscribe('pri_monthly')
        ->returnTo(route('home'));

    return view('billing', ['checkout' => $checkout]);
});
```

當你的應用程式收到 `subscription_created` 事件時，Cashier 會在應用程式資料庫的訂閱紀錄中設定試用期結束日期，並指示 Paddle 在該日期之後才開始向顧客扣款。

> [!WARNING]
> 如果顧客的訂閱未在試用期結束前取消，他們將會在試用期滿時被扣款，因此請務必提醒使用者其試用期結束的日期。

你可以使用使用者實例上的 `onTrial` 方法來判斷使用者是否處於試用期內：

```php
if ($user->onTrial()) {
    // ...
}
```

若要判斷現有的試用期是否已過期，你可以使用 `hasExpiredTrial` 方法：

```php
if ($user->hasExpiredTrial()) {
    // ...
}
```

若要判斷使用者是否處於特定訂閱類型的試用期，你可以傳入該類型至 `onTrial` 或 `hasExpiredTrial` 方法：

```php
if ($user->onTrial('default')) {
    // ...
}

if ($user->hasExpiredTrial('default')) {
    // ...
}
```


<a name="without-payment-method-up-front"></a>
### 不預先提供付款方式

如果你希望在不預先收集使用者付款方式資訊的情況下提供試用期，可以將附加在使用者上的顧客紀錄中的 `trial_ends_at` 欄位設定為你希望的試用結束日期。這通常是在使用者註冊時進行：

```php
use App\Models\User;

$user = User::create([
    // ...
]);

$user->createAsCustomer([
    'trial_ends_at' => now()->plus(days: 10)
]);
```

Cashier 將這種類型的試用稱為「通用試用 (Generic Trial)」，因為它不屬於任何現有的訂閱。如果當前日期未超過 `trial_ends_at` 的值，`User` 實例上的 `onTrial` 方法將會回傳 `true`：

```php
if ($user->onTrial()) {
    // User is within their trial period...
}
```

當你準備好為使用者建立實際的訂閱時，可以照常使用 `subscribe` 方法：

```php
use Illuminate\Http\Request;

Route::get('/user/subscribe', function (Request $request) {
    $checkout = $request->user()
        ->subscribe('pri_monthly')
        ->returnTo(route('home'));

    return view('billing', ['checkout' => $checkout]);
});
```

若要取得使用者的試用結束日期，可以使用 `trialEndsAt` 方法。如果使用者正處於試用期，此方法將回傳一個 Carbon 日期實例，否則回傳 `null`。如果你想取得預設訂閱以外之特定訂閱的試用結束日期，也可以傳入選擇性的訂閱類型參數：

```php
if ($user->onTrial('default')) {
    $trialEndsAt = $user->trialEndsAt();
}
```

如果你想特別確認使用者是否正處於其「通用」試用期內且尚未建立實際的訂閱，可以使用 `onGenericTrial` 方法：

```php
if ($user->onGenericTrial()) {
    // User is within their "generic" trial period...
}
```


<a name="extend-or-activate-a-trial"></a>
### 延長或啟用試用

你可以透過呼叫 `extendTrial` 方法並指定試用應該結束的時間點，來延長訂閱上的現有試用期：

```php
$user->subscription()->extendTrial(now()->plus(days: 5));
```

或者，你可以透過呼叫訂閱上的 `activate` 方法來結束其試用，進而立即啟用該訂閱：

```php
$user->subscription()->activate();
```

<a name="handling-paddle-webhooks"></a>
## 處理 Paddle Webhook

Paddle 可以透過 Webhook 將各種事件通知您的應用程式。預設情況下，Cashier 服務提供者(Service Providers)已註冊指向 Cashier 的 Webhook 控制器的路由。該控制器會處理所有傳入的 Webhook 請求。

預設情況下，此控制器會自動處理因扣款失敗次數過多而取消訂閱、訂閱更新以及付款方式的變更；然而，如我們接下來會看到的，您可以擴充此控制器來處理任何您想要的 Paddle Webhook 事件。

為確保您的應用程式可以處理 Paddle Webhook，請務必在 [Paddle 控制面板中設定 Webhook URL](https://vendors.paddle.com/notifications-v2)。預設情況下，Cashier 的 Webhook 控制器回應的 URL 路徑為 `/paddle/webhook`。您應該在 Paddle 控制面板中啟用的完整 Webhook 列表如下：

- Customer Updated
- Transaction Completed
- Transaction Updated
- Subscription Created
- Subscription Updated
- Subscription Paused
- Subscription Canceled

> [!WARNING]
> 請確保使用 Cashier 隨附的 [Webhook 簽章驗證](/docs/{{version}}/cashier-paddle#verifying-webhook-signatures)中介層來保護傳入的請求。


<a name="webhooks-csrf-protection"></a>
#### Webhook 與 CSRF 防護

由於 Paddle Webhook 需要繞過 Laravel 的 [CSRF 防護](/docs/{{version}}/csrf)，您應確保 Laravel 不會嘗試驗證傳入 Paddle Webhook 的 CSRF token。要達到這個目的，您應該在應用程式的 `bootstrap/app.php` 檔案中將 `paddle/*` 排除在 CSRF 防護之外：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->preventRequestForgery(except: [
        'paddle/*',
    ]);
})
```


<a name="webhooks-local-development"></a>
#### Webhook 與本機開發

為了讓 Paddle 能夠在本機開發期間向您的應用程式發送 Webhook，您需要透過網站共享服務（例如 [Ngrok](https://ngrok.com/) 或 [Expose](https://expose.dev/docs/introduction)）將您的應用程式暴露至外網。若您使用 [Laravel Sail](/docs/{{version}}/sail) 進行本機開發，您可以使用 Sail 的[網站共享命令](/docs/{{version}}/sail#sharing-your-site)。


<a name="defining-webhook-event-handlers"></a>
### 定義 Webhook 事件處理器

Cashier 會自動處理因扣款失敗導致的取消訂閱與其他常見的 Paddle Webhook。不過，若您有其他想要處理的 Webhook 事件，可以透過監聽 Cashier 所派發的以下事件來達成：

- `Laravel\Paddle\Events\WebhookReceived`
- `Laravel\Paddle\Events\WebhookHandled`

這兩個事件都包含 Paddle Webhook 的完整有效負載 (Payload)。例如，如果您想要處理 `transaction.billed` Webhook，可以註冊一個處理該事件的[監聽器](/docs/{{version}}/events#defining-listeners)：

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

Cashier 還發布針對收到 Webhook 類別的專屬事件。除了來自 Paddle 的完整 Payload 外，它們還包含用於處理 Webhook 的相關模型，例如可計費模型 (Billable Model)、訂閱或收據：

<div class="content-list" markdown="1">

- `Laravel\Paddle\Events\CustomerUpdated`
- `Laravel\Paddle\Events\TransactionCompleted`
- `Laravel\Paddle\Events\TransactionUpdated`
- `Laravel\Paddle\Events\SubscriptionCreated`
- `Laravel\Paddle\Events\SubscriptionUpdated`
- `Laravel\Paddle\Events\SubscriptionPaused`
- `Laravel\Paddle\Events\SubscriptionCanceled`

</div>

您也可以透過在應用程式的 `.env` 檔案中定義 `CASHIER_WEBHOOK` 環境變數來覆寫預設內建的 Webhook 路由。此值應為您 Webhook 路由的完整 URL，且必須與 Paddle 控制面板中設定的 URL 相符：

```ini
CASHIER_WEBHOOK=https://example.com/my-paddle-webhook-url
```


<a name="verifying-webhook-signatures"></a>
### 驗證 Webhook 簽章

為了保護您的 Webhook 安全，您可以使用 [Paddle 的 Webhook 簽章](https://developer.paddle.com/webhooks/signature-verification)。為了方便起見，Cashier 自動包含一個中介層，用於驗證傳入的 Paddle Webhook 請求是否有效。

若要啟用 Webhook 驗證，請確保應用程式的 `.env` 檔案中已設定 `PADDLE_WEBHOOK_SECRET` 環境變數。該 Webhook 密鑰可從您的 Paddle 帳號主控台取得。

<a name="single-charges"></a>
## 單次扣款


<a name="charging-for-products"></a>
### 對產品進行扣款

如果您想為顧客發起一次產品購買，可以在可計費模型 (Billable Model) 執行個體上使用 `checkout` 方法來為該次購買建立結帳 Session。`checkout` 方法接收一個或多個價格 ID。如果有需要，可以使用關聯陣列來提供購買該產品的數量：

```php
use Illuminate\Http\Request;

Route::get('/buy', function (Request $request) {
    $checkout = $request->user()->checkout(['pri_tshirt', 'pri_socks' => 5]);

    return view('buy', ['checkout' => $checkout]);
});
```

建立結帳 Session 後，您可以使用 Cashier 提供的 `paddle-button` [Blade 元件](#overlay-checkout)，讓使用者開啟 Paddle 結帳小工具並完成購買：

```blade
<x-paddle-button :checkout="$checkout" class="px-8 py-4">
    Buy
</x-paddle-button>
```

結帳 Session 包含一個 `customData` 方法，允許您將任何自訂資料傳遞給底層建立的交易紀錄。請參閱 [Paddle 的文件](https://developer.paddle.com/build/transactions/custom-data)以了解更多關於傳遞自訂資料時可用的選項：

```php
$checkout = $user->checkout('pri_tshirt')
    ->customData([
        'custom_option' => $value,
    ]);
```


<a name="refunding-transactions"></a>
### 交易退款

交易退款會將退款金額退回到顧客購買時所使用的付款方式。如果您需要對 Paddle 的購買進行退款，可以在 `Cashier\Paddle\Transaction` 模型上使用 `refund` 方法。此方法的第一個引數接受退款原因，第二個引數則接受要退款的一個或多個價格 ID，以及作為選用金額的關聯陣列。您可以使用 `transactions` 方法取得特定可計費模型的所有交易紀錄。

例如，假設我們想對價格 `pri_123` 和 `pri_456` 的特定交易進行退款。我們想要全額退款 `pri_123`，但對 `pri_456` 僅退款兩美元：

```php
use App\Models\User;

$user = User::find(1);

$transaction = $user->transactions()->first();

$response = $transaction->refund('Accidental charge', [
    'pri_123', // Fully refund this price...
    'pri_456' => 200, // Only partially refund this price...
]);
```

上述範例退款了交易中的特定項目。如果您想要對整筆交易進行退款，只需要提供退款原因即可：

```php
$response = $transaction->refund('Accidental charge');
```

關於退款的更多資訊，請參閱 [Paddle 的退款文件](https://developer.paddle.com/build/transactions/create-transaction-adjustments)。

> [!WARNING]
> 退款在完全處理前，必須始終經過 Paddle 的核准。


<a name="crediting-transactions"></a>
### 交易折抵 (Crediting)

就像退款一樣，您也可以對交易進行折抵 (Credit)。交易折抵會將資金新增到顧客的餘額中，以便用於未來的購買。交易折抵只能針對手動收集 (Manually-collected) 的交易進行，不能針對自動收集 (Automatically-collected) 的交易（例如訂閱）進行，因為 Paddle 會自動處理訂閱的折抵：

```php
$transaction = $user->transactions()->first();

// Credit a specific line item fully...
$response = $transaction->credit('Compensation', 'pri_123');
```

如需更多資訊，請參閱 [Paddle 關於交易折抵的文件](https://developer.paddle.com/build/transactions/create-transaction-adjustments)。

> [!WARNING]
> 交易折抵僅適用於手動收集的交易。自動收集的交易將由 Paddle 本身進行折抵。


<a name="transactions"></a>
## 交易紀錄

您可以透過 `transactions` 屬性輕鬆取得可計費模型的所有交易紀錄陣列：

```php
use App\Models\User;

$user = User::find(1);

$transactions = $user->transactions;
```

交易紀錄代表您產品與購買的付款紀錄，並附有發票。只有已完成的交易才會儲存在應用程式的資料庫中。

當列出顧客的交易紀錄時，您可以使用交易執行個體的方法來顯示相關的付款資訊。例如，您可能希望在表格中列出每筆交易，讓使用者可以輕鬆下載任何發票：

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
### 歷史與未來付款

您可以使用 `lastPayment` 與 `nextPayment` 方法來取得並顯示顧客過去或未來定期訂閱的付款資訊：

```php
use App\Models\User;

$user = User::find(1);

$subscription = $user->subscription();

$lastPayment = $subscription->lastPayment();
$nextPayment = $subscription->nextPayment();
```

這兩個方法都會回傳 `Laravel\Paddle\Payment` 的執行個體；然而，當交易尚未透過 Webhook 同步時，`lastPayment` 將回傳 `null`；而當計費週期已結束時（例如訂閱已取消），`nextPayment` 則會回傳 `null`：

```blade
Next payment: {{ $nextPayment->amount() }} due on {{ $nextPayment->date()->format('d/m/Y') }}
```


<a name="testing"></a>
## 測試

在測試時，您應該手動測試付款流程，以確保您的整合如預期般運作。

對於自動化測試（包含在 CI 環境中執行的測試），您可以使用 [Laravel 的 HTTP Client](/docs/{{version}}/http-client#testing) 來模擬 (Fake) 向 Paddle 發出的 HTTP 請求。雖然這不會測試來自 Paddle 的實際回應，但它提供了一種在不實際呼叫 Paddle API 的情況下測試應用程式的方法。