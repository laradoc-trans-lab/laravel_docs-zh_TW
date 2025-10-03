# Laravel Cashier (Paddle)

- [簡介](#introduction)
- [升級 Cashier](#upgrading-cashier)
- [安裝](#installation)
    - [Paddle 沙箱](#paddle-sandbox)
- [設定](#configuration)
    - [可計費模型](#billable-model)
    - [API 金鑰](#api-keys)
    - [Paddle JS](#paddle-js)
    - [貨幣設定](#currency-configuration)
    - [覆寫預設模型](#overriding-default-models)
- [快速入門](#quickstart)
    - [銷售產品](#quickstart-selling-products)
    - [銷售訂閱](#quickstart-selling-subscriptions)
- [結帳會話](#checkout-sessions)
    - [覆蓋層結帳](#overlay-checkout)
    - [內嵌結帳](#inline-checkout)
    - [訪客結帳](#guest-checkouts)
- [價格預覽](#price-previews)
    - [客戶價格預覽](#customer-price-previews)
    - [折扣](#price-discounts)
- [客戶](#customers)
    - [客戶預設值](#customer-defaults)
    - [擷取客戶](#retrieving-customers)
    - [建立客戶](#creating-customers)
- [訂閱](#subscriptions)
    - [建立訂閱](#creating-subscriptions)
    - [檢查訂閱狀態](#checking-subscription-status)
    - [訂閱單次收費](#subscription-single-charges)
    - [更新付款資訊](#updating-payment-information)
    - [變更方案](#changing-plans)
    - [訂閱數量](#subscription-quantity)
    - [多產品訂閱](#subscriptions-with-multiple-products)
    - [多個訂閱](#multiple-subscriptions)
    - [暫停訂閱](#pausing-subscriptions)
    - [取消訂閱](#canceling-subscriptions)
- [訂閱試用](#subscription-trials)
    - [預先提供付款方式](#with-payment-method-up-front)
    - [不預先提供付款方式](#without-payment-method-up-front)
    - [延長或啟用試用期](#extend-or-activate-a-trial)
- [處理 Paddle Webhooks](#handling-paddle-webhooks)
    - [定義 Webhook 事件處理器](#defining-webhook-event-handlers)
    - [驗證 Webhook 簽章](#verifying-webhook-signatures)
- [單次收費](#single-charges)
    - [產品收費](#charging-for-products)
    - [交易退款](#refunding-transactions)
    - [交易記帳](#crediting-transactions)
- [交易](#transactions)
    - [過去與即將到來的付款](#past-and-upcoming-payments)
- [測試](#testing)

<a name="introduction"></a>
## 簡介

> [!WARNING]
> 本文件是關於 Cashier Paddle 2.x 與 Paddle Billing 的整合。如果您仍在使用 Paddle Classic，應使用 [Cashier Paddle 1.x](https://github.com/laravel/cashier-paddle/tree/1.x)。

[Laravel Cashier Paddle](https://github.com/laravel/cashier-paddle) 為 [Paddle](https://paddle.com) 的訂閱計費服務提供了表達豐富、流暢的介面。它處理了您所厭惡的幾乎所有重複性訂閱計費程式碼。除了基本的訂閱管理，Cashier 還能處理：交換訂閱、訂閱「數量」、訂閱暫停、取消寬限期等。

在深入了解 Cashier Paddle 之前，我們建議您也檢閱 Paddle 的 [概念指南](https://developer.paddle.com/concepts/overview) 和 [API 文件](https://developer.paddle.com/api-reference/overview)。


<a name="upgrading-cashier"></a>
## 升級 Cashier

升級到新版本的 Cashier 時，仔細查閱[升級指南](https://github.com/laravel/cashier-paddle/blob/master/UPGRADE.md)非常重要。


<a name="installation"></a>
## 安裝

首先，使用 Composer 套件管理器為 Paddle 安裝 Cashier 套件：

```shell
composer require laravel/cashier-paddle
```

接下來，您應該使用 `vendor:publish` Artisan 指令發佈 Cashier 移轉檔案：

```shell
php artisan vendor:publish --tag="cashier-migrations"
```

然後，您應該執行應用程式的資料庫移轉。Cashier 移轉將建立一個新的 `customers` 資料表。此外，將建立新的 `subscriptions` 和 `subscription_items` 資料表，以儲存所有客戶的訂閱。最後，將建立一個新的 `transactions` 資料表，以儲存與您的客戶相關聯的所有 Paddle 交易：

```shell
php artisan migrate
```

> [!WARNING]
> 為確保 Cashier 正確處理所有 Paddle 事件，請記得[設定 Cashier 的 webhook 處理](#handling-paddle-webhooks)。


<a name="paddle-sandbox"></a>
### Paddle 沙箱

在本地和預備環境開發期間，您應該[註冊一個 Paddle 沙箱帳戶](https://sandbox-login.paddle.com/signup)。此帳戶將為您提供一個沙箱環境，以便在不進行實際付款的情況下測試和開發您的應用程式。您可以使用 Paddle 的[測試卡號](https://developer.paddle.com/concepts/payment-methods/credit-debit-card#test-payment-method)來模擬各種付款情境。

使用 Paddle 沙箱環境時，您應該在應用程式的 `.env` 檔案中將 `PADDLE_SANDBOX` 環境變數設定為 `true`：

```ini
PADDLE_SANDBOX=true
```

完成應用程式開發後，您可以[申請一個 Paddle 供應商帳戶](https://paddle.com)。在您的應用程式投入生產之前，Paddle 將需要批准您應用程式的網域。


<a name="configuration"></a>
## 設定


<a name="billable-model"></a>
### 可計費模型

在使用 Cashier 之前，您必須將 `Billable` trait 加入您的使用者模型定義中。這個 trait 提供了各種方法，讓您可以執行常見的計費任務，例如建立訂閱和更新付款方式資訊：

```php
use Laravel\Paddle\Billable;

class User extends Authenticatable
{
    use Billable;
}
```

如果您的可計費實體不是使用者，您也可以將該 trait 加入這些類別中：

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

接下來，您應該在應用程式的 `.env` 檔案中設定您的 Paddle 金鑰。您可以從 Paddle 控制面板中擷取您的 Paddle API 金鑰：

```ini
PADDLE_CLIENT_SIDE_TOKEN=your-paddle-client-side-token
PADDLE_API_KEY=your-paddle-api-key
PADDLE_RETAIN_KEY=your-paddle-retain-key
PADDLE_WEBHOOK_SECRET="your-paddle-webhook-secret"
PADDLE_SANDBOX=true
```

當您使用[Paddle 的沙箱環境](#paddle-sandbox)時，`PADDLE_SANDBOX` 環境變數應設定為 `true`。如果您將應用程式部署到生產環境並使用 Paddle 的即時供應商環境，則 `PADDLE_SANDBOX` 變數應設定為 `false`。

`PADDLE_RETAIN_KEY` 是可選的，並且僅在您搭配 [Retain](https://developer.paddle.com/concepts/retain/overview) 使用 Paddle 時才應設定。


<a name="paddle-js"></a>
### Paddle JS

Paddle 依靠其自身的 JavaScript 函式庫來啟動 Paddle 結帳小工具。您可以透過將 `@paddleJS` Blade 指令放置在應用程式版面配置的 `</head>` 標籤之前來載入 JavaScript 函式庫：

```blade
<head>
    ...

    @paddleJS
</head>
```


<a name="currency-configuration"></a>
### 貨幣設定

您可以指定用於格式化發票上顯示的貨幣值的語言環境。在內部，Cashier 利用 [PHP 的 `NumberFormatter` 類別](https://www.php.net/manual/en/class.numberformatter.php) 來設定貨幣語言環境：

```ini
CASHIER_CURRENCY_LOCALE=nl_BE
```

> [!WARNING]
> 為了使用 `en` 以外的語言環境，請確保您的伺服器上安裝並設定了 `ext-intl` PHP 擴充功能。


<a name="overriding-default-models"></a>
### 覆寫預設模型

您可以透過定義自己的模型並擴充對應的 Cashier 模型，自由地擴充 Cashier 內部使用的模型：

```php
use Laravel\Paddle\Subscription as CashierSubscription;

class Subscription extends CashierSubscription
{
    // ...
}
```

定義模型後，您可以透過 `Laravel\Paddle\Cashier` 類別指示 Cashier 使用您的自訂模型。通常，您應該在應用程式的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中告知 Cashier 您的自訂模型：

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
> 在利用 Paddle Checkout 之前，您應該在 Paddle 控制面板中定義具有固定價格的 Products。此外，您應該 [設定 Paddle 的 webhook 處理](#handling-paddle-webhooks)。

透過您的應用程式提供產品和訂閱計費可能令人望而生畏。然而，多虧了 Cashier 和 [Paddle 的結帳覆蓋層 (Checkout Overlay)](https://developer.paddle.com/concepts/sell/overlay-checkout)，您可以輕鬆地建構現代化、強大的支付整合。

為了向客戶收取非循環、單次收費的產品費用，我們將利用 Cashier 搭配 Paddle 的 Checkout Overlay 向客戶收費，客戶將在此提供付款詳情並確認購買。一旦透過 Checkout Overlay 完成付款，客戶將被重新導向到您在應用程式中選擇的成功 URL：

```php
use Illuminate\Http\Request;

Route::get('/buy', function (Request $request) {
    $checkout = $request->user()->checkout('pri_deluxe_album')
        ->returnTo(route('dashboard'));

    return view('buy', ['checkout' => $checkout]);
})->name('checkout');
```

如您在上述範例中看到的，我們將利用 Cashier 提供的 `checkout` 方法來建立一個結帳物件，以便向客戶顯示針對特定「價格識別碼」的 Paddle Checkout Overlay。使用 Paddle 時，「prices」指的是 [為特定產品定義的價格](https://developer.paddle.com/build/products/create-products-prices)。

如有必要，`checkout` 方法將會自動在 Paddle 中建立客戶，並將該 Paddle 客戶記錄與您應用程式資料庫中對應的使用者連結。完成結帳會話後，客戶將被重新導向到專屬的成功頁面，您可以在該頁面向客戶顯示資訊性訊息。

在 `buy` 視圖中，我們將包含一個按鈕來顯示 Checkout Overlay。`paddle-button` Blade 元件隨附於 Cashier Paddle；不過，您也可以 [手動呈現一個覆蓋層結帳](#manually-rendering-an-overlay-checkout)：

```html
<x-paddle-button :checkout="$checkout" class="px-8 py-4">
    Buy Product
</x-paddle-button>
```


<a name="providing-meta-data-to-paddle-checkout"></a>
#### 提供中繼資料給 Paddle 結帳

在銷售產品時，通常會透過您應用程式定義的 `Cart` 和 `Order` 模型來追蹤已完成的訂單和已購買的產品。當將客戶重新導向至 Paddle 的 Checkout Overlay 以完成購買時，您可能需要提供現有的訂單識別碼，以便在客戶重新導向回您的應用程式時，您可以將已完成的購買與對應的訂單關聯起來。

為此，您可以向 `checkout` 方法提供一個自訂資料陣列。讓我們想像一下，當使用者開始結帳流程時，我們的應用程式中建立了一個待處理的 `Order`。請記住，此範例中的 `Cart` 和 `Order` 模型僅為說明性用途，並非由 Cashier 提供。您可以根據自己應用程式的需求自由實作這些概念：

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

如您在上述範例中所見，當使用者開始結帳流程時，我們將提供所有購物車/訂單相關的 Paddle 價格識別碼給 `checkout` 方法。當然，您的應用程式有責任在客戶新增商品時，將這些項目與「購物車」或訂單關聯起來。我們也透過 `customData` 方法將訂單的 ID 提供給 Paddle Checkout Overlay。

當然，一旦客戶完成結帳流程，您可能會希望將訂單標記為「完成」。為實現此目的，您可以監聽由 Paddle 分派並透過 Cashier 的事件觸發的 webhook，以將訂單資訊儲存在您的資料庫中。

首先，監聽由 Cashier 分派的 `TransactionCompleted` 事件。通常，您應該在應用程式 `AppServiceProvider` 的 `boot` 方法中註冊事件監聽器：

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

請參閱 Paddle 的文件，以獲取有關 [`transaction.completed` 事件所包含資料](https://developer.paddle.com/webhooks/transactions/transaction-completed) 的更多資訊。

<a name="quickstart-selling-subscriptions"></a>
### 銷售訂閱

> [!NOTE]
> 在使用 Paddle Checkout 之前，您應該在 Paddle 儀表板中定義具有固定價格的產品。此外，您應該 [設定 Paddle 的 webhook 處理](#handling-paddle-webhooks)。

透過您的應用程式提供產品和訂閱計費可能令人望而卻步。然而，多虧了 Cashier 和 [Paddle 的覆蓋層結帳](https://developer.paddle.com/concepts/sell/overlay-checkout)，您可以輕鬆建立現代、強大的支付整合。

若要了解如何使用 Cashier 和 Paddle 的覆蓋層結帳來銷售訂閱，讓我們考慮一個簡單的訂閱服務情境，其中包含基本月費 (`price_basic_monthly`) 和年費 (`price_basic_yearly`) 方案。這兩種價格可以在我們的 Paddle 儀表板中歸類為「Basic」產品 (`pro_basic`)。此外，我們的訂閱服務可能還提供「Expert」方案，作為 `pro_expert`。

首先，讓我們了解客戶如何訂閱我們的服務。當然，您可以想像客戶可能會在我們應用程式的定價頁面上點擊 Basic 方案的「訂閱」按鈕。此按鈕將為其選擇的方案調用 Paddle 覆蓋層結帳。要開始使用，讓我們透過 `checkout` 方法啟動一個結帳會話：

```php
use Illuminate\Http\Request;

Route::get('/subscribe', function (Request $request) {
    $checkout = $request->user()->checkout('price_basic_monthly')
        ->returnTo(route('dashboard'));

    return view('subscribe', ['checkout' => $checkout]);
})->name('subscribe');
```

在 `subscribe` 視圖中，我們將包含一個按鈕來顯示覆蓋層結帳。`paddle-button` Blade 元件隨附於 Cashier Paddle；然而，您也可以 [手動渲染覆蓋層結帳](#manually-rendering-an-overlay-checkout)：

```html
<x-paddle-button :checkout="$checkout" class="px-8 py-4">
    Subscribe
</x-paddle-button>
```

現在，當點擊「訂閱」按鈕時，客戶將能夠輸入其付款詳細資訊並啟動其訂閱。為了解其訂閱何時真正開始（因為某些付款方式需要幾秒鐘處理），您還應該 [設定 Cashier 的 webhook 處理](#handling-paddle-webhooks)。

現在客戶可以開始訂閱了，我們需要限制應用程式的某些部分，以便只有訂閱用戶才能存取。當然，我們始終可以透過 Cashier 的 `Billable` trait 提供的 `subscribed` 方法來判斷用戶當前的訂閱狀態：

```blade
@if ($user->subscribed())
    <p>You are subscribed.</p>
@endif
```

我們甚至可以輕鬆判斷用戶是否訂閱了特定的產品或價格：

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

為方便起見，您可能希望建立一個 [中介層](/docs/{{version}}/middleware)，用於判斷傳入的請求是否來自訂閱用戶。一旦定義了此中介層，您可以輕鬆地將其分配給路由，以防止未訂閱的用戶存取該路由：

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

一旦定義了該中介層，您可以將其分配給路由：

```php
use App\Http\Middleware\Subscribed;

Route::get('/dashboard', function () {
    // ...
})->middleware([Subscribed::class]);
```


<a name="quickstart-allowing-customers-to-manage-their-billing-plan"></a>
#### 允許客戶管理其計費方案

當然，客戶可能希望將其訂閱方案更改為其他產品或「層級」。在我們上面的範例中，我們希望允許客戶將其方案從按月訂閱更改為按年訂閱。為此，您需要實作一個類似按鈕的功能，該按鈕會導向以下路由：

```php
use Illuminate\Http\Request;

Route::put('/subscription/{price}/swap', function (Request $request, $price) {
    $user->subscription()->swap($price); // With "$price" being "price_basic_yearly" for this example.

    return redirect()->route('dashboard');
})->name('subscription.swap');
```

除了更換方案之外，您還需要允許客戶取消訂閱。與更換方案類似，請提供一個按鈕，導向以下路由：

```php
use Illuminate\Http\Request;

Route::put('/subscription/cancel', function (Request $request, $price) {
    $user->subscription()->cancel();

    return redirect()->route('dashboard');
})->name('subscription.cancel');
```

現在，您的訂閱將在其計費週期結束時被取消。

> [!NOTE]
> 只要您已設定 Cashier 的 webhook 處理，Cashier 將透過檢查來自 Paddle 的傳入 webhook，自動使您應用程式中與 Cashier 相關的資料庫表格保持同步。例如，當您透過 Paddle 儀表板取消客戶的訂閱時，Cashier 將收到相應的 webhook 並將該訂閱在您應用程式的資料庫中標記為「已取消」。

<a name="checkout-sessions"></a>
## 結帳會話

大多數針對客戶的收費操作，都是透過 Paddle 的 [結帳覆蓋層元件](https://developer.paddle.com/build/checkout/build-overlay-checkout) 進行「結帳」，或是透過 [內嵌結帳](https://developer.paddle.com/build/checkout/build-branded-inline-checkout) 完成。

在處理 Paddle 結帳付款之前，您應該在 Paddle 結帳設定儀表板中，定義應用程式的 [預設付款連結](https://developer.paddle.com/build/transactions/default-payment-link#set-default-link)。

<a name="overlay-checkout"></a>
### 覆蓋層結帳

在顯示結帳覆蓋層元件之前，您必須使用 Cashier 產生一個結帳會話。結帳會話會告知結帳元件應該執行的帳務操作：

```php
use Illuminate\Http\Request;

Route::get('/buy', function (Request $request) {
    $checkout = $user->checkout('pri_34567')
        ->returnTo(route('dashboard'));

    return view('billing', ['checkout' => $checkout]);
});
```

Cashier 包含一個 `paddle-button` [Blade 元件](/docs/{{version}}/blade#components)。您可以將結帳會話作為一個「屬性 (prop)」傳遞給此元件。然後，當此按鈕被點擊時，Paddle 的結帳元件將會顯示：

```html
<x-paddle-button :checkout="$checkout" class="px-8 py-4">
    Subscribe
</x-paddle-button>
```

依預設，這將使用 Paddle 的預設樣式來顯示元件。您可以透過向元件添加 [Paddle 支援的屬性](https://developer.paddle.com/paddlejs/html-data-attributes)（例如 `data-theme='light'` 屬性）來客製化元件：

```html
<x-paddle-button :checkout="$checkout" class="px-8 py-4" data-theme="light">
    Subscribe
</x-paddle-button>
```

Paddle 結帳元件是非同步的。一旦使用者在元件中建立訂閱，Paddle 就會向您的應用程式傳送一個 webhook，以便您可以正確地更新應用程式資料庫中的訂閱狀態。因此，請務必正確地 [設定 webhooks](#handling-paddle-webhooks)，以因應來自 Paddle 的狀態變更。

> [!WARNING]
> 訂閱狀態變更後，接收到相應 webhook 的延遲通常是最小的，但您應該在應用程式中考慮這一點，因為您的使用者訂閱可能在結帳完成後不會立即可用。

<a name="manually-rendering-an-overlay-checkout"></a>
#### 手動渲染覆蓋層結帳

您也可以手動渲染覆蓋層結帳，而無需使用 Laravel 內建的 Blade 元件。要開始使用，請 [如先前範例所示](#overlay-checkout) 產生結帳會話：

```php
use Illuminate\Http\Request;

Route::get('/buy', function (Request $request) {
    $checkout = $user->checkout('pri_34567')
        ->returnTo(route('dashboard'));

    return view('billing', ['checkout' => $checkout]);
});
```

接下來，您可以使用 Paddle.js 初始化結帳。在此範例中，我們將建立一個被賦予 `paddle_button` 類別的連結。Paddle.js 將偵測到此類別，並在點擊連結時顯示覆蓋層結帳：

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
### 內嵌結帳

如果您不想使用 Paddle 的「覆蓋層」樣式結帳元件，Paddle 也提供了內嵌顯示元件的選項。雖然此方法不允許您調整結帳的任何 HTML 欄位，但它允許您將元件嵌入您的應用程式中。

為了讓您輕鬆開始使用內嵌結帳，Cashier 包含一個 `paddle-checkout` Blade 元件。要開始使用，您應該 [產生一個結帳會話](#overlay-checkout)：

```php
use Illuminate\Http\Request;

Route::get('/buy', function (Request $request) {
    $checkout = $user->checkout('pri_34567')
        ->returnTo(route('dashboard'));

    return view('billing', ['checkout' => $checkout]);
});
```

然後，您可以將結帳會話傳遞給元件的 `checkout` 屬性：

```blade
<x-paddle-checkout :checkout="$checkout" class="w-full" />
```

要調整內嵌結帳元件的高度，您可以將 `height` 屬性傳遞給 Blade 元件：

```blade
<x-paddle-checkout :checkout="$checkout" class="w-full" height="500" />
```

請參考 Paddle 的 [內嵌結帳指南](https://developer.paddle.com/build/checkout/build-branded-inline-checkout) 和 [可用結帳設定](https://developer.paddle.com/build/checkout/set-up-checkout-default-settings)，以獲取有關內嵌結帳客製化選項的更多詳細資訊。

<a name="manually-rendering-an-inline-checkout"></a>
#### 手動渲染內嵌結帳

您也可以手動渲染內嵌結帳，而無需使用 Laravel 內建的 Blade 元件。要開始使用，請 [如先前範例所示](#inline-checkout) 產生結帳會話：

```php
use Illuminate\Http\Request;

Route::get('/buy', function (Request $request) {
    $checkout = $user->checkout('pri_34567')
        ->returnTo(route('dashboard'));

    return view('billing', ['checkout' => $checkout]);
});
```

接下來，您可以使用 Paddle.js 初始化結帳。在此範例中，我們將使用 [Alpine.js](https://github.com/alpinejs/alpine) 來示範這一點；但是，您可以自由地修改此範例以適用於您自己的前端技術棧：

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

有時候，您可能需要為不需要應用程式帳戶的使用者建立結帳會話。為此，您可以使用 `guest` 方法：

```php
use Illuminate\Http\Request;
use Laravel\Paddle\Checkout;

Route::get('/buy', function (Request $request) {
    $checkout = Checkout::guest(['pri_34567'])
        ->returnTo(route('home'));

    return view('billing', ['checkout' => $checkout]);
});
```

然後，您可以將結帳會話提供給 [Paddle 按鈕](#overlay-checkout) 或 [內嵌結帳](#inline-checkout) Blade 元件。

<a name="price-previews"></a>
## 價格預覽

Paddle 允許您根據貨幣自訂價格，這基本上讓您可以為不同國家設定不同的價格。Cashier Paddle 允許您使用 `previewPrices` 方法擷取所有這些價格。此方法接受您希望擷取價格的價格 ID：

```php
use Laravel\Paddle\Cashier;

$prices = Cashier::previewPrices(['pri_123', 'pri_456']);
```

貨幣將根據請求的 IP 位址決定；但是，您可以選擇提供特定的國家以擷取其價格：

```php
use Laravel\Paddle\Cashier;

$prices = Cashier::previewPrices(['pri_123', 'pri_456'], ['address' => [
    'country_code' => 'BE',
    'postal_code' => '1234',
]]);
```

擷取價格後，您可以按照自己的意願顯示它們：

```blade
<ul>
    @foreach ($prices as $price)
        <li>{{ $price->product['name'] }} - {{ $price->total() }}</li>
    @endforeach
</ul>
```

您也可以單獨顯示小計價格和稅金金額：

```blade
<ul>
    @foreach ($prices as $price)
        <li>{{ $price->product['name'] }} - {{ $price->subtotal() }} (+ {{ $price->tax() }} tax)</li>
    @endforeach
</ul>
```

欲了解更多資訊，請查閱 [Paddle 關於價格預覽的 API 文件](https://developer.paddle.com/api-reference/pricing-preview/preview-prices)。

<a name="customer-price-previews"></a>
### 客戶價格預覽

如果使用者已是客戶，並且您希望顯示適用於該客戶的價格，您可以直接從客戶實例中擷取價格：

```php
use App\Models\User;

$prices = User::find(1)->previewPrices(['pri_123', 'pri_456']);
```

在內部，Cashier 將使用使用者的客戶 ID 來擷取其貨幣的價格。因此，舉例來說，居住在美國的使用者將看到以美元計價的價格，而比利時的使用者將看到以歐元計價的價格。如果找不到匹配的貨幣，將使用產品的預設貨幣。您可以在 Paddle 控制面板中自訂產品或訂閱方案的所有價格。

<a name="price-discounts"></a>
### 折扣

您也可以選擇顯示折扣後的價格。在呼叫 `previewPrices` 方法時，您可以透過 `discount_id` 選項提供折扣 ID：

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

Cashier 允許您在建立結帳會話時為客戶定義一些有用的預設值。設定這些預設值可讓您預填客戶的電子郵件地址和名稱，以便他們可以立即進入結帳小工具的付款部分。您可以透過覆寫您的可計費模型上的以下方法來設定這些預設值：

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

這些預設值將用於 Cashier 中生成 [結帳會話](#checkout-sessions) 的所有操作。

<a name="retrieving-customers"></a>
### 擷取客戶

您可以使用 `Cashier::findBillable` 方法透過客戶的 Paddle 客戶 ID 擷取客戶。此方法將傳回可計費模型的實例：

```php
use Laravel\Paddle\Cashier;

$user = Cashier::findBillable($customerId);
```

<a name="creating-customers"></a>
### 建立客戶

有時，您可能希望在不開始訂閱的情況下建立 Paddle 客戶。您可以使用 `createAsCustomer` 方法來完成此操作：

```php
$customer = $user->createAsCustomer();
```

將傳回 `Laravel\Paddle\Customer` 的實例。一旦客戶在 Paddle 中建立，您可以在稍後的日期開始訂閱。您可以提供一個可選的 `$options` 陣列，以傳入 [Paddle API 支援的任何額外客戶建立參數](https://developer.paddle.com/api-reference/customers/create-customer)：

```php
$customer = $user->createAsCustomer($options);
```

<a name="subscriptions"></a>
## 訂閱

<a name="creating-subscriptions"></a>
### 建立訂閱

要建立訂閱，首先從您的資料庫中擷取可計費模型實例，這通常會是 `App\Models\User` 的實例。擷取模型實例後，您可以使用 `subscribe` 方法來建立模型的結帳會話：

```php
use Illuminate\Http\Request;

Route::get('/user/subscribe', function (Request $request) {
    $checkout = $request->user()->subscribe($premium = 'pri_123', 'default')
        ->returnTo(route('home'));

    return view('billing', ['checkout' => $checkout]);
});
```

傳給 `subscribe` 方法的第一個引數是使用者訂閱的特定價格。此值應與 Paddle 中的價格識別碼相對應。 `returnTo` 方法接受一個 URL，在使用者成功完成結帳後會被重新導向到該 URL。傳給 `subscribe` 方法的第二個引數應該是訂閱的內部「類型」。如果您的應用程式只提供單一訂閱，您可以稱其為 `default` 或 `primary`。此訂閱類型僅供內部應用程式使用，不應向使用者顯示。此外，它不應包含空格，並且在建立訂閱後不應更改。

您也可以使用 `customData` 方法提供關於訂閱的自訂中繼資料陣列：

```php
$checkout = $request->user()->subscribe($premium = 'pri_123', 'default')
    ->customData(['key' => 'value'])
    ->returnTo(route('home'));
```

建立訂閱結帳會話後，該結帳會話可以提供給 Cashier Paddle 中包含的 `paddle-button` [Blade 元件](#overlay-checkout)：

```blade
<x-paddle-button :checkout="$checkout" class="px-8 py-4">
    Subscribe
</x-paddle-button>
```

使用者完成結帳後，Paddle 會發送一個 `subscription_created` webhook。Cashier 會接收此 webhook 並為您的客戶設定訂閱。為確保您的應用程式能正確接收並處理所有 webhook，請確保您已正確地[設定 webhook 處理](#handling-paddle-webhooks)。

<a name="checking-subscription-status"></a>
### 檢查訂閱狀態

一旦使用者訂閱了您的應用程式，您可以使用各種方便的方法檢查他們的訂閱狀態。首先，如果使用者擁有有效的訂閱，即使該訂閱目前仍在試用期內，`subscribed` 方法也會回傳 `true`：

```php
if ($user->subscribed()) {
    // ...
}
```

如果您的應用程式提供多個訂閱，您可以在呼叫 `subscribed` 方法時指定訂閱：

```php
if ($user->subscribed('default')) {
    // ...
}
```

 `subscribed` 方法也是一個極佳的[路由中介層](/docs/{{version}}/middleware)候選者，讓您可以根據使用者的訂閱狀態篩選路由和控制器的存取權限：

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

如果您想判斷使用者是否仍在試用期內，可以使用 `onTrial` 方法。此方法可用於判斷您是否應該向使用者顯示他們仍在試用期的警告：

```php
if ($user->subscription()->onTrial()) {
    // ...
}
```

 `subscribedToPrice` 方法可用於根據給定的 Paddle 價格 ID 來判斷使用者是否訂閱了特定方案。在此範例中，我們將判斷使用者的 `default` 訂閱是否活躍地訂閱了月費價格：

```php
if ($user->subscribedToPrice($monthly = 'pri_123', 'default')) {
    // ...
}
```

 `recurring` 方法可用於判斷使用者目前是否處於活躍訂閱狀態，且已不再試用期或寬限期內：

```php
if ($user->subscription()->recurring()) {
    // ...
}
```

<a name="canceled-subscription-status"></a>
#### 取消的訂閱狀態

要判斷使用者曾是活躍訂閱者但已取消訂閱，可以使用 `canceled` 方法：

```php
if ($user->subscription()->canceled()) {
    // ...
}
```

您還可以判斷使用者是否已取消訂閱，但仍在訂閱完全到期前的「寬限期」內。例如，如果使用者在 3 月 5 日取消了原定於 3 月 10 日到期的訂閱，那麼該使用者在 3 月 10 日之前都處於「寬限期」內。此外，在此期間 `subscribed` 方法仍會回傳 `true`：

```php
if ($user->subscription()->onGracePeriod()) {
    // ...
}
```

<a name="past-due-status"></a>
#### 逾期狀態

如果訂閱付款失敗，它將被標記為 `past_due`。當您的訂閱處於此狀態時，除非客戶更新了其付款資訊，否則該訂閱將不會是活躍的。您可以使用訂閱實例上的 `pastDue` 方法來判斷訂閱是否逾期：

```php
if ($user->subscription()->pastDue()) {
    // ...
}
```

當訂閱逾期時，您應指示使用者[更新其付款資訊](#updating-payment-information)。

如果您希望逾期（`past_due`）的訂閱仍被視為有效，可以使用 Cashier 提供的 `keepPastDueSubscriptionsActive` 方法。通常，此方法應在您的 `AppServiceProvider` 的 `register` 方法中呼叫：

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
> 當訂閱處於 `past_due` 狀態時，除非付款資訊已更新，否則無法變更。因此，當訂閱處於 `past_due` 狀態時，`swap` 和 `updateQuantity` 方法將會拋出例外。

<a name="subscription-scopes"></a>
#### 訂閱範圍

大多數訂閱狀態也可用作查詢範圍，以便您可以輕鬆地查詢資料庫中處於特定狀態的訂閱：

```php
// Get all valid subscriptions...
$subscriptions = Subscription::query()->valid()->get();

// Get all of the canceled subscriptions for a user...
$subscriptions = $user->subscriptions()->canceled()->get();
```

可用的範圍完整列表如下：

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

訂閱單次收費允許您在訂閱費用之外，向訂閱者收取一次性費用。您必須在呼叫 `charge` 方法時提供一個或多個價格 ID：

```php
// Charge a single price...
$response = $user->subscription()->charge('pri_123');

// Charge multiple prices at once...
$response = $user->subscription()->charge(['pri_123', 'pri_456']);
```

`charge` 方法在訂閱者的下一個計費週期之前不會實際向客戶收費。如果您想立即向客戶收費，可以使用 `chargeAndInvoice` 方法：

```php
$response = $user->subscription()->chargeAndInvoice('pri_123');
```

<a name="updating-payment-information"></a>
### 更新付款資訊

Paddle 始終為每個訂閱儲存一種付款方式。如果您想更新訂閱的預設付款方式，應使用訂閱模型上的 `redirectToUpdatePaymentMethod` 方法，將您的客戶重新導向到 Paddle 的託管付款方式更新頁面：

```php
use Illuminate\Http\Request;

Route::get('/update-payment-method', function (Request $request) {
    $user = $request->user();

    return $user->subscription()->redirectToUpdatePaymentMethod();
});
```

當用戶完成資訊更新後，Paddle 將會派送一個 `subscription_updated` webhook，並且訂閱詳細資訊將會在您的應用程式資料庫中更新。

<a name="changing-plans"></a>
### 變更方案

用戶訂閱您的應用程式後，有時可能會想更換新的訂閱方案。若要更新用戶的訂閱方案，您應該將 Paddle 價格的識別碼傳遞給訂閱的 `swap` 方法：

```php
use App\Models\User;

$user = User::find(1);

$user->subscription()->swap($premium = 'pri_456');
```

如果您想更換方案並立即向用戶開立發票，而不是等到他們的下一個計費週期，您可以使用 `swapAndInvoice` 方法：

```php
$user = User::find(1);

$user->subscription()->swapAndInvoice($premium = 'pri_456');
```

<a name="prorations"></a>
#### 按比例計費 (Prorations)

預設情況下，Paddle 在方案之間進行轉換時會按比例收取費用。可以使用 `noProrate` 方法來更新訂閱，而無需按比例收取費用：

```php
$user->subscription('default')->noProrate()->swap($premium = 'pri_456');
```

如果您想停用按比例計費並立即向客戶開立發票，可以將 `swapAndInvoice` 方法與 `noProrate` 結合使用：

```php
$user->subscription('default')->noProrate()->swapAndInvoice($premium = 'pri_456');
```

或者，為了避免因訂閱變更而向客戶收費，您可以使用 `doNotBill` 方法：

```php
$user->subscription('default')->doNotBill()->swap($premium = 'pri_456');
```

有關 Paddle 按比例計費政策的更多資訊，請查閱 Paddle 的[按比例計費文件](https://developer.paddle.com/concepts/subscriptions/proration)。

<a name="subscription-quantity"></a>
### 訂閱數量

有時訂閱會受「數量」影響。例如，一個專案管理應用程式可能按每個專案每月收取 $10。為了輕鬆增加或減少您訂閱的數量，請使用 `incrementQuantity` 和 `decrementQuantity` 方法：

```php
$user = User::find(1);

$user->subscription()->incrementQuantity();

// Add five to the subscription's current quantity...
$user->subscription()->incrementQuantity(5);

$user->subscription()->decrementQuantity();

// Subtract five from the subscription's current quantity...
$user->subscription()->decrementQuantity(5);
```

或者，您可以使用 `updateQuantity` 方法設定一個特定數量：

```php
$user->subscription()->updateQuantity(10);
```

`noProrate` 方法可用於更新訂閱數量，而無需按比例收取費用：

```php
$user->subscription()->noProrate()->updateQuantity(10);
```

<a name="quantities-for-subscription-with-multiple-products"></a>
#### 多產品訂閱的數量

如果您的訂閱是[多產品訂閱](#subscriptions-with-multiple-products)，您應該將要增加或減少數量的價格 ID 作為第二個參數傳遞給增加/減少方法：

```php
$user->subscription()->incrementQuantity(1, 'price_chat');
```

<a name="subscriptions-with-multiple-products"></a>
### 多產品訂閱

[多產品訂閱](https://developer.paddle.com/build/subscriptions/add-remove-products-prices-addons)允許您將多個計費產品指派給單一訂閱。例如，想像您正在建立一個客戶服務「服務台」應用程式，其基本訂閱價格為每月 $10，但提供每月額外 $15 的即時聊天附加產品。

在建立訂閱結帳會話時，您可以將一個價格陣列作為第一個參數傳遞給 `subscribe` 方法，以指定給定訂閱的多個產品：

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

在上述範例中，客戶的 `default` 訂閱將附加兩種價格。兩種價格都將按其各自的計費週期收取費用。如有必要，您可以傳遞一個鍵/值對的關聯陣列來表示每種價格的特定數量：

```php
$user = User::find(1);

$checkout = $user->subscribe('default', ['price_monthly', 'price_chat' => 5]);
```

如果您想為現有訂閱增加另一個價格，您必須使用訂閱的 `swap` 方法。在呼叫 `swap` 方法時，您也應該包含訂閱目前的價格和數量：

```php
$user = User::find(1);

$user->subscription()->swap(['price_chat', 'price_original' => 2]);
```

上述範例將新增新價格，但客戶在下一個計費週期之前不會收到帳單。如果您想立即向客戶收費，可以使用 `swapAndInvoice` 方法：

```php
$user->subscription()->swapAndInvoice(['price_chat', 'price_original' => 2]);
```

您可以使用 `swap` 方法並省略您要移除的價格，從訂閱中移除價格：

```php
$user->subscription()->swap(['price_original' => 2]);
```

> [!WARNING]
> 您不得移除訂閱上的最後一個價格。相反地，您應該直接取消訂閱。

<a name="multiple-subscriptions"></a>
### 多個訂閱

Paddle 允許您的客戶同時擁有多個訂閱。例如，您可以經營一家健身房，提供游泳訂閱和舉重訂閱，且每個訂閱可能具有不同的定價。當然，客戶應該能夠訂閱其中一個或兩個方案。

當您的應用程式建立訂閱時，您可以將訂閱類型作為第二個參數提供給 `subscribe` 方法。此類型可以是代表用戶正在啟動的訂閱類型的任何字串：

```php
use Illuminate\Http\Request;

Route::post('/swimming/subscribe', function (Request $request) {
    $checkout = $request->user()->subscribe($swimmingMonthly = 'pri_123', 'swimming');

    return view('billing', ['checkout' => $checkout]);
});
```

在此範例中，我們為客戶啟動了每月游泳訂閱。然而，他們可能稍後會想更換為年度訂閱。調整客戶訂閱時，我們可以簡單地更換 `swimming` 訂閱上的價格：

```php
$user->subscription('swimming')->swap($swimmingYearly = 'pri_456');
```

當然，您也可以完全取消訂閱：

```php
$user->subscription('swimming')->cancel();
```

<a name="pausing-subscriptions"></a>
### 暫停訂閱

若要暫停訂閱，請呼叫使用者訂閱上的 `pause` 方法：

```php
$user->subscription()->pause();
```

當訂閱暫停時，Cashier 會自動設定資料庫中的 `paused_at` 欄位。此欄位用於判斷 `paused` 方法何時開始回傳 `true`。舉例來說，如果客戶在 3 月 1 日暫停訂閱，但該訂閱原定於 3 月 5 日才會續訂，那麼 `paused` 方法將繼續回傳 `false` 直到 3 月 5 日。這是因為使用者通常被允許繼續使用應用程式直到其帳單週期結束。

預設情況下，暫停會在下一個帳單週期發生，因此客戶可以使用其已付費的剩餘期間。如果您想立即暫停訂閱，可以使用 `pauseNow` 方法：

```php
$user->subscription()->pauseNow();
```

使用 `pauseUntil` 方法，您可以將訂閱暫停至特定的時間點：

```php
$user->subscription()->pauseUntil(now()->addMonth());
```

或者，您可以使用 `pauseNowUntil` 方法來立即暫停訂閱，直到給定的時間點：

```php
$user->subscription()->pauseNowUntil(now()->addMonth());
```

您可以使用 `onPausedGracePeriod` 方法來判斷使用者是否已暫停其訂閱，但仍在其「寬限期」內：

```php
if ($user->subscription()->onPausedGracePeriod()) {
    // ...
}
```

若要恢復已暫停的訂閱，您可以呼叫訂閱上的 `resume` 方法：

```php
$user->subscription()->resume();
```

> [!WARNING]
> 訂閱在暫停期間無法修改。如果您想更換方案或更新數量，必須先恢復訂閱。

<a name="canceling-subscriptions"></a>
### 取消訂閱

若要取消訂閱，請呼叫使用者訂閱上的 `cancel` 方法：

```php
$user->subscription()->cancel();
```

當訂閱被取消時，Cashier 會自動設定資料庫中的 `ends_at` 欄位。此欄位用於判斷 `subscribed` 方法何時開始回傳 `false`。舉例來說，如果客戶在 3 月 1 日取消訂閱，但該訂閱原定於 3 月 5 日才會結束，那麼 `subscribed` 方法將繼續回傳 `true` 直到 3 月 5 日。這是因為使用者通常被允許繼續使用應用程式直到其帳單週期結束。

您可以使用 `onGracePeriod` 方法來判斷使用者是否已取消其訂閱，但仍在其「寬限期」內：

```php
if ($user->subscription()->onGracePeriod()) {
    // ...
}
```

如果您希望立即取消訂閱，可以呼叫訂閱上的 `cancelNow` 方法：

```php
$user->subscription()->cancelNow();
```

若要停止處於寬限期的訂閱取消，您可以呼叫 `stopCancelation` 方法：

```php
$user->subscription()->stopCancelation();
```

> [!WARNING]
> Paddle 的訂閱一旦取消便無法恢復。如果您的客戶希望恢復其訂閱，他們必須建立一個新的訂閱。

<a name="subscription-trials"></a>
## 訂閱試用


<a name="with-payment-method-up-front"></a>
### 預先提供付款方式

如果您想在預先收集付款方式資訊的同時向客戶提供試用期，您應該在 Paddle 儀表板中，針對客戶訂閱的價格設定試用時間。然後，照常啟動結帳會話：

```php
use Illuminate\Http\Request;

Route::get('/user/subscribe', function (Request $request) {
    $checkout = $request->user()
        ->subscribe('pri_monthly')
        ->returnTo(route('home'));

    return view('billing', ['checkout' => $checkout]);
});
```

當您的應用程式收到 `subscription_created` 事件時，Cashier 將在應用程式的資料庫中的訂閱紀錄上設定試用期結束日期，並指示 Paddle 在此日期之後才開始向客戶收費。

> [!WARNING]
> 如果客戶的訂閱在試用期結束日期前未取消，他們將在試用期一結束時立即被收費，因此您務必通知使用者他們的試用期結束日期。

您可以使用使用者實例上的 `onTrial` 方法來判斷使用者是否處於試用期內：

```php
if ($user->onTrial()) {
    // ...
}
```

若要判斷現有的試用期是否已過期，您可以使用 `hasExpiredTrial` 方法：

```php
if ($user->hasExpiredTrial()) {
    // ...
}
```

若要判斷使用者是否正處於特定訂閱類型的試用期，您可以將該類型提供給 `onTrial` 或 `hasExpiredTrial` 方法：

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

如果您想提供不預先收集使用者付款方式資訊的試用期，您可以將連結到使用者的客戶紀錄上的 `trial_ends_at` 欄位設定為您希望的試用期結束日期。這通常在使用者註冊時完成：

```php
use App\Models\User;

$user = User::create([
    // ...
]);

$user->createAsCustomer([
    'trial_ends_at' => now()->addDays(10)
]);
```

Cashier 將此類試用稱為「通用試用期」，因為它未連結到任何現有的訂閱。`User` 實例上的 `onTrial` 方法會在當前日期尚未超過 `trial_ends_at` 的值時傳回 `true`：

```php
if ($user->onTrial()) {
    // User is within their trial period...
}
```

一旦您準備好為使用者建立實際的訂閱，您可以照常使用 `subscribe` 方法：

```php
use Illuminate\Http\Request;

Route::get('/user/subscribe', function (Request $request) {
    $checkout = $request->user()
        ->subscribe('pri_monthly')
        ->returnTo(route('home'));

    return view('billing', ['checkout' => $checkout]);
});
```

若要擷取使用者的試用期結束日期，您可以使用 `trialEndsAt` 方法。如果使用者正在試用中，此方法會傳回一個 Carbon 日期實例，如果他們不在試用中，則傳回 `null`。您也可以傳遞一個選用的訂閱類型參數，以取得特定訂閱（而非預設訂閱）的試用期結束日期：

```php
if ($user->onTrial('default')) {
    $trialEndsAt = $user->trialEndsAt();
}
```

如果您想確切地知道使用者處於其「通用」試用期內且尚未建立實際訂閱，您可以使用 `onGenericTrial` 方法：

```php
if ($user->onGenericTrial()) {
    // User is within their "generic" trial period...
}
```


<a name="extend-or-activate-a-trial"></a>
### 延長或啟用試用期

您可以透過呼叫 `extendTrial` 方法並指定試用期應結束的時間點，來延長訂閱的現有試用期：

```php
$user->subscription()->extendTrial(now()->addDays(5));
```

或者，您可以透過呼叫訂閱上的 `activate` 方法來立即啟用訂閱，結束其試用期：

```php
$user->subscription()->activate();
```

<a name="handling-paddle-webhooks"></a>
## 處理 Paddle Webhooks

Paddle 可以透過 webhooks 通知您的應用程式各種事件。預設情況下，一個指向 Cashier webhook 控制器的路由已由 Cashier 服務提供者註冊。此控制器將處理所有傳入的 webhook 請求。

預設情況下，此控制器將自動處理因失敗收費過多而取消的訂閱、訂閱更新以及付款方式變更；然而，正如我們將很快發現的，您可以擴展此控制器來處理任何您喜歡的 Paddle webhook 事件。

為確保您的應用程式能夠處理 Paddle webhooks，請務必在 [Paddle 控制面板中設定 webhook URL](https://vendors.paddle.com/notifications-v2)。預設情況下，Cashier 的 webhook 控制器會回應 `/paddle/webhook` URL 路徑。您應該在 Paddle 控制面板中啟用的所有 webhooks 完整列表如下：

- Customer Updated
- Transaction Completed
- Transaction Updated
- Subscription Created
- Subscription Updated
- Subscription Paused
- Subscription Canceled

> [!WARNING]
> 確保您使用 Cashier 內建的 [webhook 簽章驗證](/docs/{{version}}/cashier-paddle#verifying-webhook-signatures) 中介層來保護傳入的請求。

<a name="webhooks-csrf-protection"></a>
#### Webhooks 與 CSRF 保護

由於 Paddle webhooks 需要繞過 Laravel 的 [CSRF 保護](/docs/{{version}}/csrf)，您應該確保 Laravel 不會嘗試驗證傳入 Paddle webhooks 的 CSRF token。為此，您應該在應用程式的 `bootstrap/app.php` 檔案中，從 CSRF 保護中排除 `paddle/*`：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->validateCsrfTokens(except: [
        'paddle/*',
    ]);
})
```

<a name="webhooks-local-development"></a>
#### Webhooks 與本地開發

為了讓 Paddle 能夠在本地開發期間向您的應用程式發送 webhooks，您需要透過網站共享服務（例如 [Ngrok](https://ngrok.com/) 或 [Expose](https://expose.dev/docs/introduction)）來公開您的應用程式。如果您正在使用 [Laravel Sail](/docs/{{version}}/sail) 進行本地應用程式開發，您可以使用 Sail 的 [網站共享命令](/docs/{{version}}/sail#sharing-your-site)。

<a name="defining-webhook-event-handlers"></a>
### 定義 Webhook 事件處理器

Cashier 會自動處理因收費失敗而取消的訂閱以及其他常見的 Paddle webhooks。然而，如果您有其他想要處理的 webhook 事件，您可以透過監聽由 Cashier 派發的以下事件來實現：

- `Laravel\Paddle\Events\WebhookReceived`
- `Laravel\Paddle\Events\WebhookHandled`

這兩個事件都包含 Paddle webhook 的完整負載。例如，如果您希望處理 `transaction.billed` webhook，您可以註冊一個將處理該事件的 [監聽器](/docs/{{version}}/events#defining-listeners)：

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

Cashier 也會發出專用於所接收 webhook 類型的事件。除了來自 Paddle 的完整負載之外，它們還包含用於處理 webhook 的相關模型，例如可計費模型、訂閱或收據：

<div class="content-list" markdown="1">

- `Laravel\Paddle\Events\CustomerUpdated`
- `Laravel\Paddle\Events\TransactionCompleted`
- `Laravel\Paddle\Events\TransactionUpdated`
- `Laravel\Paddle\Events\SubscriptionCreated`
- `Laravel\Paddle\Events\SubscriptionUpdated`
- `Laravel\Paddle\Events\SubscriptionPaused`
- `Laravel\Paddle\Events\SubscriptionCanceled`

</div>

您也可以透過在應用程式的 `.env` 檔案中定義 `CASHIER_WEBHOOK` 環境變數來覆寫預設的內建 webhook 路由。此值應該是您的 webhook 路由的完整 URL，並且需要與您在 Paddle 控制面板中設定的 URL 相符：

```ini
CASHIER_WEBHOOK=https://example.com/my-paddle-webhook-url
```

<a name="verifying-webhook-signatures"></a>
### 驗證 Webhook 簽章

為保護您的 webhooks 安全，您可以使用 [Paddle 的 webhook 簽章](https://developer.paddle.com/webhooks/signature-verification)。為了方便起見，Cashier 自動包含一個中介層，用於驗證傳入的 Paddle webhook 請求是否有效。

若要啟用 webhook 驗證，請確保在您的應用程式 `.env` 檔案中定義了 `PADDLE_WEBHOOK_SECRET` 環境變數。webhook 密鑰可以從您的 Paddle 帳戶儀表板中擷取。

<a name="single-charges"></a>
## 單次收費

<a name="charging-for-products"></a>
### 產品收費

如果您想為客戶發起產品購買，可以在可計費模型實例上使用 `checkout` 方法來產生購買的結帳會話。`checkout` 方法接受一個或多個價格 ID。如有必要，可以使用關聯陣列來提供欲購買的產品數量：

```php
use Illuminate\Http\Request;

Route::get('/buy', function (Request $request) {
    $checkout = $request->user()->checkout(['pri_tshirt', 'pri_socks' => 5]);

    return view('buy', ['checkout' => $checkout]);
});
```

產生結帳會話後，您可以使用 Cashier 提供的 `paddle-button` [Blade 元件](#overlay-checkout)來讓使用者查看 Paddle 結帳小工具並完成購買：

```blade
<x-paddle-button :checkout="$checkout" class="px-8 py-4">
    Buy
</x-paddle-button>
```

結帳會話有一個 `customData` 方法，允許您將任何自訂資料傳遞給底層的交易建立。請參閱 [Paddle 文件](https://developer.paddle.com/build/transactions/custom-data)以了解更多關於傳遞自訂資料時可用的選項：

```php
$checkout = $user->checkout('pri_tshirt')
    ->customData([
        'custom_option' => $value,
    ]);
```

<a name="refunding-transactions"></a>
### 交易退款

交易退款會將退款金額退還到客戶在購買時使用的付款方式。如果您需要退款 Paddle 購買，可以在 `Cashier\Paddle\Transaction` 模型上使用 `refund` 方法。此方法接受一個理由作為第一個引數，一個或多個價格 ID，可選擇帶有金額作為關聯陣列來進行退款。您可以使用 `transactions` 方法來擷取給定可計費模型的交易。

例如，假設我們想為價格 `pri_123` 和 `pri_456` 退款特定交易。我們希望完全退款 `pri_123`，但只退款兩美元給 `pri_456`：

```php
use App\Models\User;

$user = User::find(1);

$transaction = $user->transactions()->first();

$response = $transaction->refund('Accidental charge', [
    'pri_123', // Fully refund this price...
    'pri_456' => 200, // Only partially refund this price...
]);
```

上述範例是退款交易中的特定品項。如果您想退款整個交易，只需提供一個理由即可：

```php
$response = $transaction->refund('Accidental charge');
```

有關退款的更多資訊，請參閱 [Paddle 的退款文件](https://developer.paddle.com/build/transactions/create-transaction-adjustments)。

> [!WARNING]
> 退款必須始終經由 Paddle 批准後才能完全處理。

<a name="crediting-transactions"></a>
### 交易記帳

就像退款一樣，您也可以為交易記帳。交易記帳會將資金新增到客戶的餘額中，以便用於未來的購買。交易記帳只能用於手動收取的交易，不能用於自動收取的交易 (例如訂閱)，因為 Paddle 會自動處理訂閱記帳：

```php
$transaction = $user->transactions()->first();

// Credit a specific line item fully...
$response = $transaction->credit('Compensation', 'pri_123');
```

如需更多資訊，請參閱 [Paddle 的記帳文件](https://developer.paddle.com/build/transactions/create-transaction-adjustments)。

> [!WARNING]
> 記帳只能適用於手動收取的交易。自動收取的交易由 Paddle 本身處理記帳。

<a name="transactions"></a>
## 交易

您可以透過 `transactions` 屬性輕鬆擷取可計費模型的所有交易陣列：

```php
use App\Models\User;

$user = User::find(1);

$transactions = $user->transactions;
```

交易代表您產品和購買的付款，並附有發票。只有已完成的交易才會儲存在您應用程式的資料庫中。

列出客戶的交易時，您可以使用交易實例的方法來顯示相關的付款資訊。例如，您可能希望在表格中列出每筆交易，讓使用者可以輕鬆下載任何發票：

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

`download-invoice` 路由可能看起來像這樣：

```php
use Illuminate\Http\Request;
use Laravel\Paddle\Transaction;

Route::get('/download-invoice/{transaction}', function (Request $request, Transaction $transaction) {
    return $transaction->redirectToInvoicePdf();
})->name('download-invoice');
```

<a name="past-and-upcoming-payments"></a>
### 過去與即將到來的付款

您可以使用 `lastPayment` 和 `nextPayment` 方法來擷取並顯示客戶針對週期性訂閱的過去或即將到來的付款：

```php
use App\Models\User;

$user = User::find(1);

$subscription = $user->subscription();

$lastPayment = $subscription->lastPayment();
$nextPayment = $subscription->nextPayment();
```

這兩個方法都會返回 `Laravel\Paddle\Payment` 的實例；然而，當交易尚未由 webhooks 同步時，`lastPayment` 將返回 `null`，而當計費週期結束時（例如訂閱已取消），`nextPayment` 將返回 `null`：

```blade
Next payment: {{ $nextPayment->amount() }} due on {{ $nextPayment->date()->format('d/m/Y') }}
```

<a name="testing"></a>
## 測試

在測試時，您應該手動測試您的計費流程，以確保您的整合按預期運作。

對於自動化測試，包括在 CI 環境中執行的測試，您可以使用 [Laravel 的 HTTP Client](/docs/{{version}}/http-client#testing) 來模擬對 Paddle 發出的 HTTP 呼叫。儘管這不會測試 Paddle 的實際回應，但它提供了一種在不實際呼叫 Paddle API 的情況下測試您應用程式的方法。