# Laravel Cashier (Stripe)

- [簡介](#introduction)
- [升級 Cashier](#upgrading-cashier)
- [安裝](#installation)
- [設定](#configuration)
    - [可計費模型](#billable-model)
    - [API 金鑰](#api-keys)
    - [貨幣設定](#currency-configuration)
    - [稅務設定](#tax-configuration)
    - [日誌記錄](#logging)
    - [使用自訂模型](#using-custom-models)
- [快速入門](#quickstart)
    - [銷售產品](#quickstart-selling-products)
    - [銷售訂閱](#quickstart-selling-subscriptions)
- [客戶](#customers)
    - [取得客戶](#retrieving-customers)
    - [建立客戶](#creating-customers)
    - [更新客戶](#updating-customers)
    - [餘額](#balances)
    - [稅籍編號](#tax-ids)
    - [將客戶資料與 Stripe 同步](#syncing-customer-data-with-stripe)
    - [帳務入口網站](#billing-portal)
- [付款方式](#payment-methods)
    - [儲存付款方式](#storing-payment-methods)
    - [取得付款方式](#retrieving-payment-methods)
    - [付款方式存在性](#payment-method-presence)
    - [更新預設付款方式](#updating-the-default-payment-method)
    - [新增付款方式](#adding-payment-methods)
    - [刪除付款方式](#deleting-payment-methods)
- [訂閱](#subscriptions)
    - [建立訂閱](#creating-subscriptions)
    - [檢查訂閱狀態](#checking-subscription-status)
    - [變更價格](#changing-prices)
    - [訂閱數量](#subscription-quantity)
    - [多產品訂閱](#subscriptions-with-multiple-products)
    - [多個訂閱](#multiple-subscriptions)
    - [依用量計費](#usage-based-billing)
    - [訂閱稅金](#subscription-taxes)
    - [訂閱錨定日期](#subscription-anchor-date)
    - [取消訂閱](#cancelling-subscriptions)
    - [恢復訂閱](#resuming-subscriptions)
- [訂閱試用期](#subscription-trials)
    - [預先提供付款方式](#with-payment-method-up-front)
    - [不預先提供付款方式](#without-payment-method-up-front)
    - [延長試用期](#extending-trials)
- [處理 Stripe Webhook](#handling-stripe-webhooks)
    - [定義 Webhook 事件處理器](#defining-webhook-event-handlers)
    - [驗證 Webhook 簽章](#verifying-webhook-signatures)
- [單次收費](#single-charges)
    - [簡單收費](#simple-charge)
    - [含發票收費](#charge-with-invoice)
    - [建立 Payment Intent](#creating-payment-intents)
    - [退款](#refunding-charges)
- [Checkout](#checkout)
    - [產品 Checkout](#product-checkouts)
    - [單次收費 Checkout](#single-charge-checkouts)
    - [訂閱 Checkout](#subscription-checkouts)
    - [收集稅籍編號](#collecting-tax-ids)
    - [訪客 Checkout](#guest-checkouts)
- [發票](#invoices)
    - [取得發票](#retrieving-invoices)
    - [即將到來的發票](#upcoming-invoices)
    - [預覽訂閱發票](#previewing-subscription-invoices)
    - [產生發票 PDF](#generating-invoice-pdfs)
- [處理付款失敗](#handling-failed-payments)
    - [確認付款](#confirming-payments)
- [強客戶認證 (SCA)](#strong-customer-authentication)
    - [需要額外確認的付款](#payments-requiring-additional-confirmation)
    - [離線付款通知](#off-session-payment-notifications)
- [Stripe SDK](#stripe-sdk)
- [測試](#testing)

<a name="introduction"></a>
## 簡介

[Laravel Cashier Stripe](https://github.com/laravel/cashier-stripe) 為 [Stripe](https://stripe.com) 的訂閱計費服務提供了富有表現力且流暢的介面。它處理了幾乎所有您不願編寫的重複性訂閱計費程式碼。除了基本的訂閱管理之外，Cashier 還可以處理優惠券、更換訂閱、訂閱「數量」、取消寬限期，甚至產生發票 PDF。

<a name="upgrading-cashier"></a>
## 升級 Cashier

升級到新版 Cashier 時，請務必仔細閱讀[升級指南](https://github.com/laravel/cashier-stripe/blob/master/UPGRADE.md)。

> [!WARNING]
> 為防止破壞性變更，Cashier 使用固定的 Stripe API 版本。Cashier 15 使用 Stripe API 版本 `2023-10-16`。Stripe API 版本將在次要版本中更新，以利用新的 Stripe 功能和改進。

<a name="installation"></a>
## 安裝

首先，使用 Composer 套件管理器安裝適用於 Stripe 的 Cashier 套件：

```shell
composer require laravel/cashier
```

安裝套件後，使用 `vendor:publish` Artisan 命令發布 Cashier 的遷移檔：

```shell
php artisan vendor:publish --tag="cashier-migrations"
```

然後，遷移您的資料庫：

```shell
php artisan migrate
```

Cashier 的遷移檔將為您的 `users` 資料表新增多個欄位。它們還將建立一個新的 `subscriptions` 資料表來儲存所有客戶的訂閱，以及一個 `subscription_items` 資料表用於多價格訂閱。

如果您願意，也可以使用 `vendor:publish` Artisan 命令發布 Cashier 的設定檔：

```shell
php artisan vendor:publish --tag="cashier-config"
```

最後，為確保 Cashier 正確處理所有 Stripe 事件，請記得[設定 Cashier 的 Webhook 處理](#handling-stripe-webhooks)。

> [!WARNING]
> Stripe 建議用於儲存 Stripe 識別碼的任何欄位都應區分大小寫。因此，在使用 MySQL 時，您應確保 `stripe_id` 欄位的定序設定為 `utf8_bin`。有關此內容的更多資訊，請參閱 [Stripe 說明文件](https://stripe.com/docs/upgrades#what-changes-does-stripe-consider-to-be-backwards-compatible)。

<a name="configuration"></a>
## 設定

<a name="billable-model"></a>
### 可計費模型

在使用 Cashier 之前，請將 `Billable` trait 新增到您的可計費模型定義中。通常，這將是 `App\Models\User` 模型。此 trait 提供了各種方法，讓您可以執行常見的計費任務，例如建立訂閱、應用優惠券和更新付款方式資訊：

```php
use Laravel\Cashier\Billable;

class User extends Authenticatable
{
    use Billable;
}
```

Cashier 假設您的可計費模型將是 Laravel 隨附的 `App\Models\User` 類別。如果您希望更改此設定，可以透過 `useCustomerModel` 方法指定不同的模型。此方法通常應在您的 `AppServiceProvider` 類別的 `boot` 方法中呼叫：

```php
use App\Models\Cashier\User;
use Laravel\Cashier\Cashier;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Cashier::useCustomerModel(User::class);
}
```

> [!WARNING]
> 如果您使用的模型不是 Laravel 提供的 `App\Models\User` 模型，您將需要發布並修改提供的 [Cashier 遷移檔](#installation)以符合您替代模型的資料表名稱。

<a name="api-keys"></a>
### API 金鑰

接下來，您應該在應用程式的 `.env` 檔案中設定您的 Stripe API 金鑰。您可以從 Stripe 控制面板中取得您的 Stripe API 金鑰：

```ini
STRIPE_KEY=your-stripe-key
STRIPE_SECRET=your-stripe-secret
STRIPE_WEBHOOK_SECRET=your-stripe-webhook-secret
```

> [!WARNING]
> 您應該確保在應用程式的 `.env` 檔案中定義了 `STRIPE_WEBHOOK_SECRET` 環境變數，因為此變數用於確保傳入的 Webhook 確實來自 Stripe。

<a name="currency-configuration"></a>
### 貨幣設定

Cashier 的預設貨幣是美元 (USD)。您可以透過在應用程式的 `.env` 檔案中設定 `CASHIER_CURRENCY` 環境變數來更改預設貨幣：

```ini
CASHIER_CURRENCY=eur
```

除了設定 Cashier 的貨幣之外，您還可以指定一個語系，用於格式化發票上顯示的金額值。在內部，Cashier 利用 [PHP 的 `NumberFormatter` 類別](https://www.php.net/manual/en/class.numberformatter.php)來設定貨幣語系：

```ini
CASHIER_CURRENCY_LOCALE=nl_BE
```

> [!WARNING]
> 為了使用 `en` 以外的語系，請確保您的伺服器上已安裝並設定 `ext-intl` PHP 擴充功能。

<a name="tax-configuration"></a>
### 稅務設定

多虧了 [Stripe Tax](https://stripe.com/tax)，可以自動計算 Stripe 產生的所有發票的稅金。您可以透過在應用程式的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫 `calculateTaxes` 方法來啟用自動稅金計算：

```php
use Laravel\Cashier\Cashier;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Cashier::calculateTaxes();
}
```

啟用稅金計算後，任何新的訂閱和任何產生的一次性發票都將收到自動稅金計算。

為了使此功能正常運作，您的客戶的帳單詳細資訊，例如客戶的姓名、地址和稅籍編號，需要同步到 Stripe。您可以使用 Cashier 提供的[客戶資料同步](#syncing-customer-data-with-stripe)和[稅籍編號](#tax-ids)方法來完成此操作。

<a name="logging"></a>
### 日誌記錄

Cashier 允許您指定在記錄嚴重 Stripe 錯誤時要使用的日誌通道。您可以透過在應用程式的 `.env` 檔案中定義 `CASHIER_LOGGER` 環境變數來指定日誌通道：

```ini
CASHIER_LOGGER=stack
```

對 Stripe 進行 API 呼叫時產生的例外將透過應用程式的預設日誌通道記錄。

<a name="using-custom-models"></a>
### 使用自訂模型

您可以透過定義自己的模型並擴展相應的 Cashier 模型來自由擴展 Cashier 內部使用的模型：

```php
use Laravel\Cashier\Subscription as CashierSubscription;

class Subscription extends CashierSubscription
{
    // ...
}
```

定義模型後，您可以透過 `Laravel\Cashier\Cashier` 類別指示 Cashier 使用您的自訂模型。通常，您應該在應用程式的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中通知 Cashier 您的自訂模型：

```php
use App\Models\Cashier\Subscription;
use App\Models\Cashier\SubscriptionItem;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Cashier::useSubscriptionModel(Subscription::class);
    Cashier::useSubscriptionItemModel(SubscriptionItem::class);
}
```

<a name="quickstart"></a>
## 快速入門

<a name="quickstart-selling-products"></a>
### 銷售產品

> [!NOTE]
> 在使用 Stripe Checkout 之前，您應該在 Stripe 控制面板中定義具有固定價格的產品。此外，您應該[設定 Cashier 的 Webhook 處理](#handling-stripe-webhooks)。

透過您的應用程式提供產品和訂閱計費可能令人望而生畏。然而，多虧了 Cashier 和 [Stripe Checkout](https://stripe.com/payments/checkout)，您可以輕鬆建立現代、強大的支付整合。

為了向客戶收取非經常性、單次收費產品的費用，我們將利用 Cashier 將客戶引導至 Stripe Checkout，他們將在那裡提供付款詳細資訊並確認購買。一旦透過 Checkout 完成付款，客戶將被重新導向到您在應用程式中選擇的成功 URL：

```php
use Illuminate\Http\Request;

Route::get('/checkout', function (Request $request) {
    $stripePriceId = 'price_deluxe_album';

    $quantity = 1;

    return $request->user()->checkout([$stripePriceId => $quantity], [
        'success_url' => route('checkout-success'),
        'cancel_url' => route('checkout-cancel'),
    ]);
})->name('checkout');

Route::view('/checkout/success', 'checkout.success')->name('checkout-success');
Route::view('/checkout/cancel', 'checkout.cancel')->name('checkout-cancel');
```

如您在上面的範例中看到的，我們將利用 Cashier 提供的 `checkout` 方法將客戶重新導向到 Stripe Checkout 以獲取給定的「價格識別碼」。在使用 Stripe 時，「價格」是指[為特定產品定義的價格](https://stripe.com/docs/products-prices/how-products-and-prices-work)。

如有必要，`checkout` 方法將自動在 Stripe 中建立客戶，並將該 Stripe 客戶記錄連接到應用程式資料庫中對應的使用者。完成 Checkout 會話後，客戶將被重新導向到專用的成功或取消頁面，您可以在其中向客戶顯示資訊性訊息。

<a name="providing-meta-data-to-stripe-checkout"></a>
#### 向 Stripe Checkout 提供中繼資料

銷售產品時，通常會透過您自己的應用程式定義的 `Cart` 和 `Order` 模型來追蹤已完成的訂單和已購買的產品。當將客戶重新導向到 Stripe Checkout 以完成購買時，您可能需要提供現有的訂單識別碼，以便在客戶重新導向回您的應用程式時，您可以將已完成的購買與相應的訂單關聯起來。

為此，您可以向 `checkout` 方法提供一個 `metadata` 陣列。讓我們想像一下，當使用者開始結帳流程時，我們的應用程式中會建立一個待處理的 `Order`。請記住，此範例中的 `Cart` 和 `Order` 模型是說明性的，並非由 Cashier 提供。您可以根據自己應用程式的需求自由實作這些概念：

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

    return $request->user()->checkout($order->price_ids, [
        'success_url' => route('checkout-success').'?session_id={CHECKOUT_SESSION_ID}',
        'cancel_url' => route('checkout-cancel'),
        'metadata' => ['order_id' => $order->id],
    ]);
})->name('checkout');
```

如您在上面的範例中看到的，當使用者開始結帳流程時，我們將向 `checkout` 方法提供所有購物車/訂單相關的 Stripe 價格識別碼。當然，您的應用程式負責在客戶新增這些項目時將它們與「購物車」或訂單關聯起來。我們還透過 `metadata` 陣列將訂單的 ID 提供給 Stripe Checkout 會話。最後，我們已將 `CHECKOUT_SESSION_ID` 範本變數新增到 Checkout 成功路由。當 Stripe 將客戶重新導向回您的應用程式時，此範本變數將自動填入 Checkout 會話 ID。

接下來，讓我們建立 Checkout 成功路由。這是使用者透過 Stripe Checkout 完成購買後將被重新導向的路由。在此路由中，我們可以取得 Stripe Checkout 會話 ID 和相關的 Stripe Checkout 實例，以便存取我們提供的中繼資料並相應地更新客戶的訂單：

```php
use App\Models\Order;
use Illuminate\Http\Request;
use Laravel\Cashier\Cashier;

Route::get('/checkout/success', function (Request $request) {
    $sessionId = $request->get('session_id');

    if ($sessionId === null) {
        return;
    }

    $session = Cashier::stripe()->checkout->sessions->retrieve($sessionId);

    if ($session->payment_status !== 'paid') {
        return;
    }

    $orderId = $session['metadata']['order_id'] ?? null;

    $order = Order::findOrFail($orderId);

    $order->update(['status' => 'completed']);

    return view('checkout-success', ['order' => $order]);
})->name('checkout-success');
```

請參閱 Stripe 的說明文件，以獲取有關 [Checkout 會話物件所包含資料](https://stripe.com/docs/api/checkout/sessions/object)的更多資訊。

<a name="quickstart-selling-subscriptions"></a>
### 銷售訂閱

> [!NOTE]
> 在使用 Stripe Checkout 之前，您應該在 Stripe 控制面板中定義具有固定價格的產品。此外，您應該[設定 Cashier 的 Webhook 處理](#handling-stripe-webhooks)。

透過您的應用程式提供產品和訂閱計費可能令人望而生畏。然而，多虧了 Cashier 和 [Stripe Checkout](https://stripe.com/payments/checkout)，您可以輕鬆建立現代、強大的支付整合。

為了學習如何使用 Cashier 和 Stripe Checkout 銷售訂閱，讓我們考慮一個簡單的訂閱服務場景，其中包含基本月度 (`price_basic_monthly`) 和年度 (`price_basic_yearly`) 方案。這兩個價格可以在我們的 Stripe 控制面板中歸類到一個「基本」產品 (`pro_basic`) 下。此外，我們的訂閱服務可能提供一個 Expert 方案作為 `pro_expert`。

首先，讓我們了解客戶如何訂閱我們的服務。當然，您可以想像客戶可能會在我們應用程式的定價頁面上點擊基本方案的「訂閱」按鈕。此按鈕或連結應將使用者導向一個 Laravel 路由，該路由將為他們選擇的方案建立 Stripe Checkout 會話：

```php
use Illuminate\Http\Request;

Route::get('/subscription-checkout', function (Request $request) {
    return $request->user()
        ->newSubscription('default', 'price_basic_monthly')
        ->trialDays(5)
        ->allowPromotionCodes()
        ->checkout([
            'success_url' => route('your-success-route'),
            'cancel_url' => route('your-cancel-route'),
        ]);
});
```

如您在上面的範例中看到的，我們將客戶重新導向到一個 Stripe Checkout 會話，該會話將允許他們訂閱我們的基本方案。成功結帳或取消後，客戶將被重新導向回我們提供給 `checkout` 方法的 URL。為了知道他們的訂閱何時實際開始（因為某些付款方式需要幾秒鐘才能處理），我們還需要[設定 Cashier 的 Webhook 處理](#handling-stripe-webhooks)。

現在客戶可以開始訂閱了，我們需要限制應用程式的某些部分，以便只有訂閱使用者才能存取它們。當然，我們始終可以透過 Cashier 的 `Billable` trait 提供的 `subscribed` 方法來確定使用者的當前訂閱狀態：

```blade
 @if ($user->subscribed())
    <p>您已訂閱。</p>
 @endif
```

我們甚至可以輕鬆確定使用者是否訂閱了特定產品或價格：

```blade
 @if ($user->subscribedToProduct('pro_basic'))
    <p>您已訂閱我們的基本產品。</p>
 @endif@if ($user->subscribedToPrice('price_basic_monthly'))
    <p>您已訂閱我們的月度基本方案。</p>
 @endif
```

<a name="quickstart-building-a-subscribed-middleware"></a>
#### 建立訂閱中介軟體

為了方便起見，您可能希望建立一個[中介軟體](/docs/{{version}}/middleware)，該中介軟體用於判斷傳入的請求是否來自訂閱使用者。一旦定義了此中介軟體，您可以輕鬆地將其分配給路由，以防止未訂閱的使用者存取該路由：

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
            // 將使用者重新導向到帳單頁面並要求他們訂閱...
            return redirect('/billing');
        }

        return $next($request);
    }
}
```

一旦定義了中介軟體，您可以將其分配給路由：

```php
use App\Http\Middleware\Subscribed;

Route::get('/dashboard', function () {
    // ...
})->middleware([Subscribed::class]);
```

<a name="quickstart-allowing-customers-to-manage-their-billing-plan"></a>
#### 允許客戶管理其帳單方案

當然，客戶可能希望將其訂閱方案更改為其他產品或「層級」。最簡單的方法是將客戶導向 Stripe 的 [Customer Billing Portal](https://stripe.com/docs/no-code/customer-portal)，該入口網站提供了一個託管的使用者介面，允許客戶下載發票、更新其付款方式和更改訂閱方案。

首先，在您的應用程式中定義一個連結或按鈕，將使用者導向一個 Laravel 路由，我們將利用該路由來啟動 Billing Portal 會話：

```blade
<a href="{{ route('billing') }}">
    帳單
</a>
```

接下來，讓我們定義一個路由，該路由啟動 Stripe Customer Billing Portal 會話並將使用者重新導向到 Portal。`redirectToBillingPortal` 方法接受使用者在退出 Portal 時應返回的 URL：

```php
use Illuminate\Http\Request;

Route::get('/billing', function (Request $request) {
    return $request->user()->redirectToBillingPortal(route('dashboard'));
})->middleware(['auth'])->name('billing');
```

> [!NOTE]
> 只要您已設定 Cashier 的 Webhook 處理，Cashier 將透過檢查來自 Stripe 的傳入 Webhook 自動保持您應用程式的 Cashier 相關資料庫資料表同步。因此，例如，當使用者透過 Stripe 的 Customer Billing Portal 取消訂閱時，Cashier 將收到相應的 Webhook 並在您應用程式的資料庫中將訂閱標記為「已取消」。

<a name="customers"></a>
## 客戶

<a name="retrieving-customers"></a>
### 取得客戶

您可以使用 `Cashier::findBillable` 方法透過客戶的 Stripe ID 取得客戶。此方法將返回可計費模型的實例：

```php
use Laravel\Cashier\Cashier;

$user = Cashier::findBillable($stripeId);
```

<a name="creating-customers"></a>
### 建立客戶

有時，您可能希望在不開始訂閱的情況下建立 Stripe 客戶。您可以使用 `createAsStripeCustomer` 方法來完成此操作：

```php
$stripeCustomer = $user->createAsStripeCustomer();
```

一旦在 Stripe 中建立了客戶，您可以在以後的日期開始訂閱。您可以提供一個可選的 `$options` 陣列，以傳遞 [Stripe API 支援的任何額外客戶建立參數](https://stripe.com/docs/api/customers/create)：

```php
$stripeCustomer = $user->createAsStripeCustomer($options);
```

如果您想返回可計費模型的 Stripe 客戶物件，可以使用 `asStripeCustomer` 方法：

```php
$stripeCustomer = $user->asStripeCustomer();
```

如果您想為給定的可計費模型取得 Stripe 客戶物件，但不確定該可計費模型是否已是 Stripe 中的客戶，則可以使用 `createOrGetStripeCustomer` 方法。如果 Stripe 中不存在客戶，此方法將建立一個新客戶：

```php
$stripeCustomer = $user->createOrGetStripeCustomer();
```

<a name="updating-customers"></a>
### 更新客戶

有時，您可能希望直接使用額外資訊更新 Stripe 客戶。您可以使用 `updateStripeCustomer` 方法來完成此操作。此方法接受一個 [Stripe API 支援的客戶更新選項](https://stripe.com/docs/api/customers/update)陣列：

```php
$stripeCustomer = $user->updateStripeCustomer($options);
```

<a name="balances"></a>
### 餘額

Stripe 允許您貸記或借記客戶的「餘額」。稍後，此餘額將在新的發票上貸記或借記。要檢查客戶的總餘額，您可以使用可計費模型上可用的 `balance` 方法。`balance` 方法將以客戶貨幣返回餘額的格式化字串表示：

```php
$balance = $user->balance();
```

要貸記客戶的餘額，您可以向 `creditBalance` 方法提供一個值。如果您願意，您還可以提供一個描述：

```php
$user->creditBalance(500, 'Premium customer top-up.');
```

向 `debitBalance` 方法提供一個值將借記客戶的餘額：

```php
$user->debitBalance(300, 'Bad usage penalty.');
```

`applyBalance` 方法將為客戶建立新的客戶餘額交易。您可以使用 `balanceTransactions` 方法取得這些交易記錄，這對於為客戶提供貸記和借記日誌以供審查可能很有用：

```php
// 取得所有交易...
$transactions = $user->balanceTransactions();

foreach ($transactions as $transaction) {
    // 交易金額...
    $amount = $transaction->amount(); // $2.31

    // 取得相關發票 (如果可用)...
    $invoice = $transaction->invoice();
}
```

<a name="tax-ids"></a>
### 稅籍編號

Cashier 提供了一種管理客戶稅籍編號的簡單方法。例如，`taxIds` 方法可用於以集合的形式取得分配給客戶的所有[稅籍編號](https://stripe.com/docs/api/customer_tax_ids/object)：

```php
$taxIds = $user->taxIds();
```

您還可以透過其識別碼取得客戶的特定稅籍編號：

```php
$taxId = $user->findTaxId('txi_belgium');
```

您可以透過向 `createTaxId` 方法提供有效的[類型](https://stripe.com/docs/api/customer_tax_ids/object#tax_id_object-type)和值來建立新的稅籍編號：

```php
$taxId = $user->createTaxId('eu_vat', 'BE0123456789');
```

`createTaxId` 方法將立即將 VAT ID 新增到客戶的帳戶中。[VAT ID 的驗證也由 Stripe 完成](https://stripe.com/docs/invoicing/customer/tax-ids#validation)；但是，這是一個非同步過程。您可以透過訂閱 `customer.tax_id.updated` Webhook 事件並檢查 [VAT ID 的 `verification` 參數](https://stripe.com/docs/api/customer_tax_ids/object#tax_id_object-verification)來接收驗證更新通知。有關處理 Webhook 的更多資訊，請參閱[定義 Webhook 處理器](handling-stripe-webhooks)的說明文件。

您可以使用 `deleteTaxId` 方法刪除稅籍編號：

```php
$user->deleteTaxId('txi_belgium');
```

<a name="syncing-customer-data-with-stripe"></a>
### 將客戶資料與 Stripe 同步

通常，當您應用程式的使用者更新其姓名、電子郵件地址或其他也由 Stripe 儲存的資訊時，您應該通知 Stripe 這些更新。這樣做，Stripe 的資訊副本將與您應用程式的資訊同步。

為了自動化此過程，您可以在可計費模型上定義一個事件監聽器，該監聽器響應模型的 `updated` 事件。然後，在您的事件監聽器中，您可以呼叫模型上的 `syncStripeCustomerDetails` 方法：

```php
use App\Models\User;
use function Illuminate\Events\queueable;

/**
 * The "booted" method of the model.
 */
protected static function booted(): void
{
    static::updated(queueable(function (User $customer) {
        if ($customer->hasStripeId()) {
            $customer->syncStripeCustomerDetails();
        }
    }));
}
```

現在，每次更新您的客戶模型時，其資訊都將與 Stripe 同步。為了方便起見，Cashier 將在客戶首次建立時自動將您的客戶資訊與 Stripe 同步。

您可以透過覆寫 Cashier 提供的各種方法來自訂用於將客戶資訊同步到 Stripe 的欄位。例如，您可以覆寫 `stripeName` 方法來自訂當 Cashier 將客戶資訊同步到 Stripe 時應被視為客戶「姓名」的屬性：

```php
/**
 * 取得應同步到 Stripe 的客戶姓名。
 */
public function stripeName(): string|null
{
    return $this->company_name;
}
```

同樣，您可以覆寫 `stripeEmail`、`stripePhone`、`stripeAddress` 和 `stripePreferredLocales` 方法。這些方法將在[更新 Stripe 客戶物件](https://stripe.com/docs/api/customers/update)時將資訊同步到其對應的客戶參數。如果您希望完全控制客戶資訊同步過程，您可以覆寫 `syncStripeCustomerDetails` 方法。

<a name="billing-portal"></a>
### 帳務入口網站

Stripe 提供[一種簡單的方法來設定帳務入口網站](https://stripe.com/docs/billing/subscriptions/customer-portal)，以便您的客戶可以管理其訂閱、付款方式和查看其帳單歷史記錄。您可以透過從控制器或路由呼叫可計費模型上的 `redirectToBillingPortal` 方法，將使用者重新導向到帳務入口網站：

```php
use Illuminate\Http\Request;

Route::get('/billing-portal', function (Request $request) {
    return $request->user()->redirectToBillingPortal();
});
```

預設情況下，當使用者完成訂閱管理後，他們將能夠透過 Stripe 帳務入口網站中的連結返回您應用程式的 `home` 路由。您可以透過將 URL 作為參數傳遞給 `redirectToBillingPortal` 方法來提供使用者應返回的自訂 URL：

```php
use Illuminate\Http\Request;

Route::get('/billing-portal', function (Request $request) {
    return $request->user()->redirectToBillingPortal(route('billing'));
});
```

如果您想在不產生 HTTP 重新導向響應的情況下產生帳務入口網站的 URL，您可以呼叫 `billingPortalUrl` 方法：

```php
$url = $request->user()->billingPortalUrl(route('billing'));
```

<a name="payment-methods"></a>
## 付款方式

<a name="storing-payment-methods"></a>
### 儲存付款方式

為了建立訂閱或使用 Stripe 執行「一次性」收費，您需要儲存付款方式並從 Stripe 取得其識別碼。完成此操作的方法因您是打算將付款方式用於訂閱還是單次收費而異，因此我們將在下面分別探討這兩種情況。

<a name="payment-methods-for-subscriptions"></a>
#### 訂閱的付款方式

當儲存客戶的信用卡資訊以供訂閱未來使用時，必須使用 Stripe「Setup Intents」API 安全地收集客戶的付款方式詳細資訊。「Setup Intent」向 Stripe 指示了向客戶付款方式收費的意圖。Cashier 的 `Billable` trait 包含 `createSetupIntent` 方法，可輕鬆建立新的 Setup Intent。您應該從將呈現收集客戶付款方式詳細資訊的表單的路由或控制器中呼叫此方法：

```php
return view('update-payment-method', [
    'intent' => $user->createSetupIntent()
]);
```

建立 Setup Intent 並將其傳遞給視圖後，您應該將其密鑰附加到將收集付款方式的元素。例如，考慮這個「更新付款方式」表單：

```html
<input id="card-holder-name" type="text">

<!-- Stripe Elements Placeholder -->
<div id="card-element"></div>

<button id="card-button" data-secret="{{ $intent->client_secret }}">
    更新付款方式
</button>
```

接下來，可以使用 Stripe.js 函式庫將 [Stripe Element](https://stripe.com/docs/stripe-js) 附加到表單並安全地收集客戶的付款詳細資訊：

```html
<script src="https://js.stripe.com/v3/"></script>

<script>
    const stripe = Stripe('stripe-public-key');

    const elements = stripe.elements();
    const cardElement = elements.create('card');

    cardElement.mount('#card-element');
</script>
```

接下來，可以使用 [Stripe 的 `confirmCardSetup` 方法](https://stripe.com/docs/js/setup_intents/confirm_card_setup)驗證卡片並從 Stripe 取得安全的「付款方式識別碼」：

```js
const cardHolderName = document.getElementById('card-holder-name');
const cardButton = document.getElementById('card-button');
const clientSecret = cardButton.dataset.secret;

cardButton.addEventListener('click', async (e) => {
    const { setupIntent, error } = await stripe.confirmCardSetup(
        clientSecret, {
            payment_method: {
                card: cardElement,
                billing_details: { name: cardHolderName.value }
            }
        }
    );

    if (error) {
        // 向使用者顯示 "error.message"...
    } else {
        // 卡片已成功驗證...
    }
});
```

卡片經 Stripe 驗證後，您可以將產生的 `setupIntent.payment_method` 識別碼傳遞給您的 Laravel 應用程式，在那裡它可以附加到客戶。付款方式可以[新增為新的付款方式](#adding-payment-methods)或[用於更新預設付款方式](#updating-the-default-payment-method)。您也可以立即使用付款方式識別碼來[建立新的訂閱](#creating-subscriptions)。

> [!NOTE]
> 如果您想了解有關 Setup Intents 和收集客戶付款詳細資訊的更多資訊，請[查閱 Stripe 提供的此概述](https://stripe.com/docs/payments/save-and-reuse#php)。

<a name="payment-methods-for-single-charges"></a>
#### 單次收費的付款方式

當然，當對客戶的付款方式進行單次收費時，我們只需要使用付款方式識別碼一次。由於 Stripe 的限制，您不能將客戶儲存的預設付款方式用於單次收費。您必須允許客戶使用 Stripe.js 函式庫輸入其付款方式詳細資訊。例如，考慮以下表單：

```html
<input id="card-holder-name" type="text">

<!-- Stripe Elements Placeholder -->
<div id="card-element"></div>

<button id="card-button">
    處理付款
</button>
```

定義此類表單後，可以使用 Stripe.js 函式庫將 [Stripe Element](https://stripe.com/docs/stripe-js) 附加到表單並安全地收集客戶的付款詳細資訊：

```html
<script src="https://js.stripe.com/v3/"></script>

<script>
    const stripe = Stripe('stripe-public-key');

    const elements = stripe.elements();
    const cardElement = elements.create('card');

    cardElement.mount('#card-element');
</script>
```

接下來，可以使用 [Stripe 的 `createPaymentMethod` 方法](https://stripe.com/docs/stripe-js/reference#stripe-create-payment-method)驗證卡片並從 Stripe 取得安全的「付款方式識別碼」：

```js
const cardHolderName = document.getElementById('card-holder-name');
const cardButton = document.getElementById('card-button');

cardButton.addEventListener('click', async (e) => {
    const { paymentMethod, error } = await stripe.createPaymentMethod(
        'card', cardElement, {
            billing_details: { name: cardHolderName.value }
        }
    );

    if (error) {
        // 向使用者顯示 "error.message"...
    } else {
        // 卡片已成功驗證...
    }
});
```

如果卡片驗證成功，您可以將 `paymentMethod.id` 傳遞給您的 Laravel 應用程式並處理[單次收費](#simple-charge)。

<a name="retrieving-payment-methods"></a>
### 取得付款方式

可計費模型實例上的 `paymentMethods` 方法返回 `Laravel\Cashier\PaymentMethod` 實例的集合：

```php
$paymentMethods = $user->paymentMethods();
```

預設情況下，此方法將返回所有類型的付款方式。要取得特定類型的付款方式，您可以將 `type` 作為參數傳遞給該方法：

```php
$paymentMethods = $user->paymentMethods('sepa_debit');
```

要取得客戶的預設付款方式，可以使用 `defaultPaymentMethod` 方法：

```php
$paymentMethod = $user->defaultPaymentMethod();
```

您可以使用 `findPaymentMethod` 方法取得附加到可計費模型的特定付款方式：

```php
$paymentMethod = $user->findPaymentMethod($paymentMethodId);
```

<a name="payment-method-presence"></a>
### 付款方式存在性

要判斷可計費模型是否已將預設付款方式附加到其帳戶，請呼叫 `hasDefaultPaymentMethod` 方法：

```php
if ($user->hasDefaultPaymentMethod()) {
    // ...
}
```

您可以使用 `hasPaymentMethod` 方法來判斷可計費模型是否至少有一個付款方式附加到其帳戶：

```php
if ($user->hasPaymentMethod()) {
    // ...
}
```

此方法將判斷可計費模型是否具有任何付款方式。要判斷模型是否存在特定類型的付款方式，您可以將 `type` 作為參數傳遞給該方法：

```php
if ($user->hasPaymentMethod('sepa_debit')) {
    // ...
}
```

<a name="updating-the-default-payment-method"></a>
### 更新預設付款方式

`updateDefaultPaymentMethod` 方法可用於更新客戶的預設付款方式資訊。此方法接受一個 Stripe 付款方式識別碼，並將新的付款方式指定為預設帳單付款方式：

```php
$user->updateDefaultPaymentMethod($paymentMethod);
```

要將您的預設付款方式資訊與 Stripe 中客戶的預設付款方式資訊同步，您可以使用 `updateDefaultPaymentMethodFromStripe` 方法：

```php
$user->updateDefaultPaymentMethodFromStripe();
```

> [!WARNING]
> 客戶的預設付款方式只能用於開立發票和建立新訂閱。由於 Stripe 施加的限制，它不能用於單次收費。

<a name="adding-payment-methods"></a>
### 新增付款方式

要新增付款方式，您可以呼叫可計費模型上的 `addPaymentMethod` 方法，並傳遞付款方式識別碼：

```php
$user->addPaymentMethod($paymentMethod);
```

> [!NOTE]
> 要了解如何取得付款方式識別碼，請查閱[付款方式儲存說明文件](#storing-payment-methods)。

<a name="deleting-payment-methods"></a>
### 刪除付款方式

要刪除付款方式，您可以呼叫您希望刪除的 `Laravel\Cashier\PaymentMethod` 實例上的 `delete` 方法：

```php
$paymentMethod->delete();
```

`deletePaymentMethod` 方法將從可計費模型中刪除特定的付款方式：

```php
$user->deletePaymentMethod('pm_visa');
```

`deletePaymentMethods` 方法將刪除可計費模型的所有付款方式資訊：

```php
$user->deletePaymentMethods();
```

預設情況下，此方法將刪除所有類型的付款方式。要刪除特定類型的付款方式，您可以將 `type` 作為參數傳遞給該方法：

```php
$user->deletePaymentMethods('sepa_debit');
```

> [!WARNING]
> 如果使用者有有效的訂閱，您的應用程式不應允許他們刪除其預設付款方式。

<a name="subscriptions"></a>
## 訂閱

訂閱提供了一種為客戶設定定期付款的方式。由 Cashier 管理的 Stripe 訂閱支援多種訂閱價格、訂閱數量、試用期等。

<a name="creating-subscriptions"></a>
### 建立訂閱

要建立訂閱，首先取得您的可計費模型的實例，這通常是 `App\Models\User` 的實例。取得模型實例後，您可以使用 `newSubscription` 方法建立模型的訂閱：

```php
use Illuminate\Http\Request;

Route::post('/user/subscribe', function (Request $request) {
    $request->user()->newSubscription(
        'default', 'price_monthly'
    )->create($request->paymentMethodId);

    // ...
});
```

傳遞給 `newSubscription` 方法的第一個參數應該是訂閱的內部類型。如果您的應用程式只提供單一訂閱，您可以將其稱為 `default` 或 `primary`。此訂閱類型僅供內部應用程式使用，不應向使用者顯示。此外，它不應包含空格，並且在建立訂閱後永遠不應更改。第二個參數是使用者訂閱的特定價格。此值應與 Stripe 中的價格識別碼相對應。

`create` 方法接受[一個 Stripe 付款方式識別碼](#storing-payment-methods)或 Stripe `PaymentMethod` 物件，它將開始訂閱並使用可計費模型的 Stripe 客戶 ID 和其他相關帳單資訊更新您的資料庫。

> [!WARNING]
> 將付款方式識別碼直接傳遞給 `create` 訂閱方法也將自動將其新增到使用者的儲存付款方式中。

<a name="collecting-recurring-payments-via-invoice-emails"></a>
#### 透過發票電子郵件收取定期付款

您可以指示 Stripe 在每次定期付款到期時向客戶發送電子郵件發票，而不是自動收取客戶的定期付款。然後，客戶可以在收到發票後手動支付。透過發票收取定期付款時，客戶無需預先提供付款方式：

```php
$user->newSubscription('default', 'price_monthly')->createAndSendInvoice();
```

客戶在訂閱被取消之前支付發票的時間長度由 `days_until_due` 選項決定。預設情況下，這是 30 天；但是，如果您願意，您可以為此選項提供一個特定值：

```php
$user->newSubscription('default', 'price_monthly')->createAndSendInvoice([], [
    'days_until_due' => 30
]);
```

<a name="subscription-quantities"></a>
#### 數量

如果您想在建立訂閱時為價格設定特定的[數量](https://stripe.com/docs/billing/subscriptions/quantities)，您應該在建立訂閱之前呼叫訂閱建構器上的 `quantity` 方法：

```php
$user->newSubscription('default', 'price_monthly')
    ->quantity(5)
    ->create($paymentMethod);
```

<a name="additional-details"></a>
#### 額外詳細資訊

如果您想指定 Stripe 支援的額外[客戶](https://stripe.com/docs/api/customers/create)或[訂閱](https://stripe.com/docs/api/subscriptions/create)選項，您可以將它們作為第二個和第三個參數傳遞給 `create` 方法：

```php
$user->newSubscription('default', 'price_monthly')->create($paymentMethod, [
    'email' => $email,
], [
    'metadata' => ['note' => 'Some extra information.'],
]);
```

<a name="coupons"></a>
#### 優惠券

如果您想在建立訂閱時應用優惠券，您可以使用 `withCoupon` 方法：

```php
$user->newSubscription('default', 'price_monthly')
    ->withCoupon('code')
    ->create($paymentMethod);
```

或者，如果您想應用 [Stripe 促銷代碼](https://stripe.com/docs/billing/subscriptions/discounts/codes)，您可以使用 `withPromotionCode` 方法：

```php
$user->newSubscription('default', 'price_monthly')
    ->withPromotionCode('promo_code_id')
    ->create($paymentMethod);
```

給定的促銷代碼 ID 應該是分配給促銷代碼的 Stripe API ID，而不是面向客戶的促銷代碼。如果您需要根據給定的面向客戶的促銷代碼查找促銷代碼 ID，您可以使用 `findPromotionCode` 方法：

```php
// 透過其面向客戶的代碼查找促銷代碼 ID...
$promotionCode = $user->findPromotionCode('SUMMERSALE');

// 透過其面向客戶的代碼查找有效的促銷代碼 ID...
$promotionCode = $user->findActivePromotionCode('SUMMERSALE');
```

在上面的範例中，返回的 `$promotionCode` 物件是 `Laravel\Cashier\PromotionCode` 的實例。此類別裝飾了底層的 `Stripe\PromotionCode` 物件。您可以透過呼叫 `coupon` 方法來取得與促銷代碼相關的優惠券：

```php
$coupon = $user->findPromotionCode('SUMMERSALE')->coupon();
```

優惠券實例允許您確定折扣金額以及優惠券是固定折扣還是基於百分比的折扣：

```php
if ($coupon->isPercentage()) {
    return $coupon->percentOff().'%'; // 21.5%
} else {
    return $coupon->amountOff(); // $5.99
}
```

您還可以取得目前應用於客戶或訂閱的折扣：

```php
$discount = $billable->discount();

$discount = $subscription->discount();
```

返回的 `Laravel\Cashier\Discount` 實例裝飾了底層的 `Stripe\Discount` 物件實例。您可以透過呼叫 `coupon` 方法來取得與此折扣相關的優惠券：

```php
$coupon = $subscription->discount()->coupon();
```

如果您想向客戶或訂閱應用新的優惠券或促銷代碼，您可以透過 `applyCoupon` 或 `applyPromotionCode` 方法來完成：

```php
$billable->applyCoupon('coupon_id');
$billable->applyPromotionCode('promotion_code_id');

$subscription->applyCoupon('coupon_id');
$subscription->applyPromotionCode('promotion_code_id');
```

請記住，您應該使用分配給促銷代碼的 Stripe API ID，而不是面向客戶的促銷代碼。一次只能將一個優惠券或促銷代碼應用於客戶或訂閱。

有關此主題的更多資訊，請參閱 Stripe 關於[優惠券](https://stripe.com/docs/billing/subscriptions/coupons)和[促銷代碼](https://stripe.com/docs/billing/subscriptions/coupons/codes)的說明文件。

<a name="adding-subscriptions"></a>
#### 新增訂閱

如果您想為已經有預設付款方式的客戶新增訂閱，您可以呼叫訂閱建構器上的 `add` 方法：

```php
use App\Models\User;

$user = User::find(1);

$user->newSubscription('default', 'price_monthly')->add();
```

<a name="creating-subscriptions-from-the-stripe-dashboard"></a>
#### 從 Stripe 控制面板建立訂閱

您也可以從 Stripe 控制面板本身建立訂閱。這樣做時，Cashier 將同步新新增的訂閱並將其指定為 `default` 類型。要自訂分配給從控制面板建立的訂閱的訂閱類型，請[定義 Webhook 事件處理器](#defining-webhook-event-handlers)。

此外，您只能透過 Stripe 控制面板建立一種訂閱類型。如果您的應用程式提供使用不同類型的多個訂閱，則只能透過 Stripe 控制面板新增一種訂閱類型。

最後，您應該始終確保每個應用程式提供的訂閱類型只新增一個有效訂閱。如果客戶有兩個 `default` 訂閱，則 Cashier 將只使用最近新增的訂閱，即使兩者都將與您的應用程式的資料庫同步。

<a name="checking-subscription-status"></a>
### 檢查訂閱狀態

一旦客戶訂閱了您的應用程式，您可以使用各種方便的方法輕鬆檢查其訂閱狀態。首先，`subscribed` 方法返回 `true`，如果客戶有有效的訂閱，即使訂閱目前處於試用期內。`subscribed` 方法將訂閱類型作為其第一個參數：

```php
if ($user->subscribed('default')) {
    // ...
}
```

`subscribed` 方法也是[路由中介軟體](/docs/{{version}}/middleware)的絕佳候選者，允許您根據使用者的訂閱狀態篩選對路由和控制器的存取：

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
        if ($request->user() && ! $request->user()->subscribed('default')) {
            // 此使用者不是付費客戶...
            return redirect('/billing');
        }

        return $next($request);
    }
}
```

如果您想判斷使用者是否仍在試用期內，您可以使用 `onTrial` 方法。此方法對於判斷您是否應該向使用者顯示他們仍在試用期內的警告很有用：

```php
if ($user->subscription('default')->onTrial()) {
    // ...
}
```

`subscribedToProduct` 方法可用於根據給定的 Stripe 產品識別碼判斷使用者是否訂閱了給定產品。在 Stripe 中，產品是價格的集合。在此範例中，我們將判斷使用者的 `default` 訂閱是否已主動訂閱應用程式的「premium」產品。給定的 Stripe 產品識別碼應與您在 Stripe 控制面板中的產品識別碼之一相對應：

```php
if ($user->subscribedToProduct('prod_premium', 'default')) {
    // ...
}
```

透過將陣列傳遞給 `subscribedToProduct` 方法，您可以判斷使用者的 `default` 訂閱是否已主動訂閱應用程式的「basic」或「premium」產品：

```php
if ($user->subscribedToProduct(['prod_basic', 'prod_premium'], 'default')) {
    // ...
}
```

`subscribedToPrice` 方法可用於判斷客戶的訂閱是否與給定的價格 ID 相符：

```php
if ($user->subscribedToPrice('price_basic_monthly', 'default')) {
    // ...
}
```

`recurring` 方法可用於判斷使用者目前是否已訂閱且不再處於試用期內：

```php
if ($user->subscription('default')->recurring()) {
    // ...
}
```

> [!WARNING]
> 如果使用者有兩個相同類型的訂閱，`subscription` 方法將始終返回最新的訂閱。例如，使用者可能有兩個類型為 `default` 的訂閱記錄；但是，其中一個訂閱可能是舊的、已過期的訂閱，而另一個是當前有效的訂閱。最新的訂閱將始終返回，而較舊的訂閱則保留在資料庫中以供歷史審查。

<a name="cancelled-subscription-status"></a>
#### 已取消訂閱狀態

要判斷使用者是否曾經是活躍訂閱者但已取消訂閱，您可以使用 `canceled` 方法：

```php
if ($user->subscription('default')->canceled()) {
    // ...
}
```

您還可以判斷使用者是否已取消訂閱但仍在「寬限期」內，直到訂閱完全到期。例如，如果使用者在 3 月 5 日取消了原定於 3 月 10 日到期的訂閱，則使用者在 3 月 10 日之前都處於「寬限期」內。請注意，在此期間 `subscribed` 方法仍返回 `true`：

```php
if ($user->subscription('default')->onGracePeriod()) {
    // ...
}
```

要判斷使用者是否已取消訂閱且不再處於「寬限期」內，您可以使用 `ended` 方法：

```php
if ($user->subscription('default')->ended()) {
    // ...
}
```

<a name="incomplete-and-past-due-status"></a>
#### 不完整和逾期狀態

如果訂閱在建立後需要二次付款操作，則訂閱將被標記為 `incomplete`。訂閱狀態儲存在 Cashier 的 `subscriptions` 資料庫資料表的 `stripe_status` 欄位中。

同樣，如果在更換價格時需要二次付款操作，則訂閱將被標記為 `past_due`。當您的訂閱處於這些狀態之一時，它將在客戶確認付款之前不會處於活動狀態。判斷訂閱是否具有不完整付款可以使用可計費模型或訂閱實例上的 `hasIncompletePayment` 方法來完成：

```php
if ($user->hasIncompletePayment('default')) {
    // ...
}

if ($user->subscription('default')->hasIncompletePayment()) {
    // ...
}
```

當訂閱有不完整付款時，您應該將使用者導向 Cashier 的付款確認頁面，並傳遞 `latestPayment` 識別碼。您可以使用訂閱實例上可用的 `latestPayment` 方法來取得此識別碼：

```html
<a href="{{ route('cashier.payment', $subscription->latestPayment()->id) }}">
    請確認您的付款。
</a>
```

如果您希望訂閱在處於 `past_due` 或 `incomplete` 狀態時仍被視為有效，您可以使用 Cashier 提供的 `keepPastDueSubscriptionsActive` 和 `keepIncompleteSubscriptionsActive` 方法。通常，這些方法應在您的 `App\Providers\AppServiceProvider` 的 `register` 方法中呼叫：

```php
use Laravel\Cashier\Cashier;

/**
 * Register any application services.
 */
public function register(): void
{
    Cashier::keepPastDueSubscriptionsActive();
    Cashier::keepIncompleteSubscriptionsActive();
}
```

> [!WARNING]
> 當訂閱處於 `incomplete` 狀態時，在確認付款之前無法更改。因此，當訂閱處於 `incomplete` 狀態時，`swap` 和 `updateQuantity` 方法將拋出例外。

<a name="subscription-scopes"></a>
#### 訂閱範圍

大多數訂閱狀態也可用作查詢範圍，以便您可以輕鬆地查詢資料庫中處於給定狀態的訂閱：

```php
// 取得所有有效訂閱...
$subscriptions = Subscription::query()->active()->get();

// 取得使用者所有已取消的訂閱...
$subscriptions = $user->subscriptions()->canceled()->get();
```

以下是可用範圍的完整列表：

```php
Subscription::query()->active();
Subscription::query()->canceled();
Subscription::query()->ended();
Subscription::query()->incomplete();
Subscription::query()->notCanceled();
Subscription::query()->notOnGracePeriod();
Subscription::query()->notOnTrial();
Subscription::query()->onGracePeriod();
Subscription::query()->onTrial();
Subscription::query()->pastDue();
Subscription::query()->recurring();
```

<a name="changing-prices"></a>
### 變更價格

客戶訂閱您的應用程式後，他們可能偶爾會想更改為新的訂閱價格。要將客戶更換為新價格，請將 Stripe 價格的識別碼傳遞給 `swap` 方法。更換價格時，假設使用者希望重新啟用其訂閱（如果之前已取消）。給定的價格識別碼應與 Stripe 控制面板中可用的 Stripe 價格識別碼相對應：

```php
use App\Models\User;

$user = App\Models\User::find(1);

$user->subscription('default')->swap('price_yearly');
```

如果客戶處於試用期，試用期將會保留。此外，如果訂閱存在「數量」，該數量也將會保留。

如果您想更換價格並取消客戶目前正在進行的任何試用期，您可以呼叫 `skipTrial` 方法：

```php
$user->subscription('default')
    ->skipTrial()
    ->swap('price_yearly');
```

如果您想更換價格並立即向客戶開立發票，而不是等待下一個帳單週期，您可以使用 `swapAndInvoice` 方法：

```php
$user = User::find(1);

$user->subscription('default')->swapAndInvoice('price_yearly');
```

<a name="prorations"></a>
#### 按比例計費

預設情況下，Stripe 在更換價格時會按比例計費。`noProrate` 方法可用於更新訂閱價格而不按比例計費：

```php
$user->subscription('default')->noProrate()->swap('price_yearly');
```

有關訂閱按比例計費的更多資訊，請參閱 [Stripe 說明文件](https://stripe.com/docs/billing/subscriptions/prorations)。

> [!WARNING]
> 在 `swapAndInvoice` 方法之前執行 `noProrate` 方法對按比例計費沒有影響。發票將始終開立。

<a name="subscription-quantity"></a>
### 訂閱數量

有時訂閱會受到「數量」的影響。例如，專案管理應用程式可能會按每個專案每月收取 10 美元。您可以使用 `incrementQuantity` 和 `decrementQuantity` 方法輕鬆增加或減少您的訂閱數量：

```php
use App\Models\User;

$user = User::find(1);

$user->subscription('default')->incrementQuantity();

// 為訂閱的當前數量增加五個...
$user->subscription('default')->incrementQuantity(5);

$user->subscription('default')->decrementQuantity();

// 從訂閱的當前數量中減去五個...
$user->subscription('default')->decrementQuantity(5);
```

或者，您可以使用 `updateQuantity` 方法設定特定數量：

```php
$user->subscription('default')->updateQuantity(10);
```

`noProrate` 方法可用於更新訂閱數量而不按比例計費：

```php
$user->subscription('default')->noProrate()->updateQuantity(10);
```

有關訂閱數量的更多資訊，請參閱 [Stripe 說明文件](https://stripe.com/docs/subscriptions/quantities)。

<a name="quantities-for-subscription-with-multiple-products"></a>
#### 多產品訂閱的數量

如果您的訂閱是[多產品訂閱](#subscriptions-with-multiple-products)，您應該將您希望增加或減少數量的價格 ID 作為第二個參數傳遞給增加/減少方法：

```php
$user->subscription('default')->incrementQuantity(1, 'price_chat');
```

<a name="subscriptions-with-multiple-products"></a>
### 多產品訂閱

[多產品訂閱](https://stripe.com/docs/billing/subscriptions/multiple-products)允許您將多個計費產品分配給單一訂閱。例如，想像您正在建立一個客戶服務「服務台」應用程式，其基本訂閱價格為每月 10 美元，但提供每月額外 15 美元的即時聊天附加產品。多產品訂閱的資訊儲存在 Cashier 的 `subscription_items` 資料庫資料表中。

您可以透過將價格陣列作為第二個參數傳遞給 `newSubscription` 方法來為給定訂閱指定多個產品：

```php
use Illuminate\Http\Request;

Route::post('/user/subscribe', function (Request $request) {
    $request->user()->newSubscription('default', [
        'price_monthly',
        'price_chat',
    ])->create($request->paymentMethodId);

    // ...
});
```

在上面的範例中，客戶的 `default` 訂閱將附加兩個價格。這兩個價格將在各自的帳單間隔內收取。如有必要，您可以使用 `quantity` 方法來指示每個價格的特定數量：

```php
$user = User::find(1);

$user->newSubscription('default', ['price_monthly', 'price_chat'])
    ->quantity(5, 'price_chat')
    ->create($paymentMethod);
```

如果您想向現有訂閱新增另一個價格，您可以呼叫訂閱的 `addPrice` 方法：

```php
$user = User::find(1);

$user->subscription('default')->addPrice('price_chat');
```

上面的範例將新增新價格，客戶將在下一個帳單週期收到帳單。如果您想立即向客戶開立帳單，您可以使用 `addPriceAndInvoice` 方法：

```php
$user->subscription('default')->addPriceAndInvoice('price_chat');
```

如果您想新增具有特定數量的價格，您可以將數量作為 `addPrice` 或 `addPriceAndInvoice` 方法的第二個參數傳遞：

```php
$user = User::find(1);

$user->subscription('default')->addPrice('price_chat', 5);
```

您可以使用 `removePrice` 方法從訂閱中移除價格：

```php
$user->subscription('default')->removePrice('price_chat');
```

> [!WARNING]
> 您不能移除訂閱上的最後一個價格。相反，您應該直接取消訂閱。

<a name="swapping-prices"></a>
#### 更換價格

您也可以更改附加到多產品訂閱的價格。例如，想像客戶有一個帶有 `price_chat` 附加產品的 `price_basic` 訂閱，您想將客戶從 `price_basic` 升級到 `price_pro` 價格：

```php
use App\Models\User;

$user = User::find(1);

$user->subscription('default')->swap(['price_pro', 'price_chat']);
```

執行上面的範例時，帶有 `price_basic` 的底層訂閱項目將被刪除，而帶有 `price_chat` 的訂閱項目將被保留。此外，將為 `price_pro` 建立一個新的訂閱項目。

您還可以透過將鍵/值對陣列傳遞給 `swap` 方法來指定訂閱項目選項。例如，您可能需要指定訂閱價格數量：

```php
$user = User::find(1);

$user->subscription('default')->swap([
    'price_pro' => ['quantity' => 5],
    'price_chat'
]);
```

如果您想更換訂閱上的單一價格，您可以使用訂閱項目本身上的 `swap` 方法。如果希望保留訂閱其他價格上的所有現有中繼資料，此方法特別有用：

```php
$user = User::find(1);

$user->subscription('default')
    ->findItemOrFail('price_basic')
    ->swap('price_pro');
```

<a name="proration"></a>
#### 按比例計費

預設情況下，Stripe 在從多產品訂閱中新增或移除價格時會按比例計費。如果您想在不按比例計費的情況下調整價格，您應該將 `noProrate` 方法鏈接到您的價格操作上：

```php
$user->subscription('default')->noProrate()->removePrice('price_chat');
```

<a name="swapping-quantities"></a>
#### 數量

如果您想更新個別訂閱價格的數量，您可以使用[現有的數量方法](#subscription-quantity)，將價格 ID 作為額外參數傳遞給該方法：

```php
$user = User::find(1);

$user->subscription('default')->incrementQuantity(5, 'price_chat');

$user->subscription('default')->decrementQuantity(3, 'price_chat');

$user->subscription('default')->updateQuantity(10, 'price_chat');
```

> [!WARNING]
> 當訂閱有多個價格時，`Subscription` 模型上的 `stripe_price` 和 `quantity` 屬性將為 `null`。要存取個別價格屬性，您應該使用 `Subscription` 模型上可用的 `items` 關聯。

<a name="subscription-items"></a>
#### 訂閱項目

當訂閱有多個價格時，它將在您的資料庫的 `subscription_items` 資料表中儲存多個訂閱「項目」。您可以透過訂閱上的 `items` 關聯來存取這些項目：

```php
use App\Models\User;

$user = User::find(1);

$subscriptionItem = $user->subscription('default')->items->first();

// 取得特定項目的 Stripe 價格和數量...
$stripePrice = $subscriptionItem->stripe_price;
$quantity = $subscriptionItem->quantity;
```

您還可以使用 `findItemOrFail` 方法取得特定價格：

```php
$user = User::find(1);

$subscriptionItem = $user->subscription('default')->findItemOrFail('price_chat');
```

<a name="multiple-subscriptions"></a>
### 多個訂閱

Stripe 允許您的客戶同時擁有多個訂閱。例如，您可能經營一家健身房，提供游泳訂閱和舉重訂閱，並且每個訂閱可能具有不同的定價。當然，客戶應該能夠訂閱其中一個或兩個方案。

當您的應用程式建立訂閱時，您可以向 `newSubscription` 方法提供訂閱的類型。該類型可以是表示使用者正在啟動的訂閱類型的任何字串：

```php
use Illuminate\Http\Request;

Route::post('/swimming/subscribe', function (Request $request) {
    $request->user()->newSubscription('swimming')
        ->price('price_swimming_monthly')
        ->create($request->paymentMethodId);

    // ...
});
```

在此範例中，我們為客戶啟動了每月游泳訂閱。但是，他們可能希望在以後更換為年度訂閱。調整客戶訂閱時，我們只需更換 `swimming` 訂閱上的價格：

```php
$user->subscription('swimming')->swap('price_swimming_yearly');
```

當然，您也可以完全取消訂閱：

```php
$user->subscription('swimming')->cancel();
```

<a name="usage-based-billing"></a>
### 依用量計費

[依用量計費](https://stripe.com/docs/billing/subscriptions/metered-billing)允許您根據客戶在帳單週期內的產品使用量向其收費。例如，您可以根據客戶每月發送的簡訊或電子郵件數量向其收費。

要開始使用依用量計費，您首先需要在 Stripe 控制面板中建立一個具有[依用量計費模型](https://docs.stripe.com/billing/subscriptions/usage-based/implementation-guide)和[計量器](https://docs.stripe.com/billing/subscriptions/usage-based/recording-usage#configure-meter)的新產品。建立計量器後，儲存相關的事件名稱和計量器 ID，您將需要這些資訊來報告和取得使用量。然後，使用 `meteredPrice` 方法將計量價格 ID 新增到客戶訂閱中：

```php
use Illuminate\Http\Request;

Route::post('/user/subscribe', function (Request $request) {
    $request->user()->newSubscription('default')
        ->meteredPrice('price_metered')
        ->create($request->paymentMethodId);

    // ...
});
```

您也可以透過 [Stripe Checkout](#checkout) 開始計量訂閱：

```php
$checkout = Auth::user()
    ->newSubscription('default', [])
    ->meteredPrice('price_metered')
    ->checkout();

return view('your-checkout-view', [
    'checkout' => $checkout,
]);
```

<a name="reporting-usage"></a>
#### 報告用量

當您的客戶使用您的應用程式時，您將向 Stripe 報告其使用量，以便他們可以準確計費。要報告計量事件的使用量，您可以使用 `Billable` 模型上的 `reportMeterEvent` 方法：

```php
$user = User::find(1);

$user->reportMeterEvent('emails-sent');
```

預設情況下，帳單週期會新增 1 個「使用量」。或者，您可以傳遞特定數量的「使用量」以新增到客戶在帳單週期內的使用量：

```php
$user = User::find(1);

$user->reportMeterEvent('emails-sent', quantity: 15);
```

要取得客戶計量器的事件摘要，您可以使用 `Billable` 實例的 `meterEventSummaries` 方法：

```php
$user = User::find(1);

$meterUsage = $user->meterEventSummaries($meterId);

$meterUsage->first()->aggregated_value // 10
```

請參閱 Stripe 的 [Meter Event Summary 物件說明文件](https://docs.stripe.com/api/billing/meter-event_summary/object)，以獲取有關計量器事件摘要的更多資訊。

要[列出所有計量器](https://docs.stripe.com/api/billing/meter/list)，您可以使用 `Billable` 實例的 `meters` 方法：

```php
$user = User::find(1);

$user->meters();
```

<a name="subscription-taxes"></a>
### 訂閱稅金

> [!WARNING]
> 您可以[使用 Stripe Tax 自動計算稅金](#tax-configuration)，而不是手動計算稅率。

要指定使用者在訂閱上支付的稅率，您應該在您的可計費模型上實作 `taxRates` 方法，並返回一個包含 Stripe 稅率 ID 的陣列。您可以在[您的 Stripe 控制面板](https://dashboard.stripe.com/test/tax-rates)中定義這些稅率：

```php
/**
 * 應適用於客戶訂閱的稅率。
 *
 * @return array<int, string>
 */
public function taxRates(): array
{
    return ['txr_id'];
}
```

`taxRates` 方法使您能夠根據客戶逐一應用稅率，這對於跨多個國家和稅率的使用者群可能很有幫助。

如果您提供多產品訂閱，您可以透過在您的可計費模型上實作 `priceTaxRates` 方法來為每個價格定義不同的稅率：

```php
/**
 * 應適用於客戶訂閱的稅率。
 *
 * @return array<string, array<int, string>>
 */
public function priceTaxRates(): array
{
    return [
        'price_monthly' => ['txr_id'],
    ];
}
```

> [!WARNING]
> `taxRates` 方法僅適用於訂閱費用。如果您使用 Cashier 進行「一次性」收費，您將需要在該時間手動指定稅率。

<a name="syncing-tax-rates"></a>
#### 同步稅率

當更改 `taxRates` 方法返回的硬編碼稅率 ID 時，使用者任何現有訂閱的稅務設定將保持不變。如果您希望使用新的 `taxRates` 值更新現有訂閱的稅務值，您應該在使用者訂閱實例上呼叫 `syncTaxRates` 方法：

```php
$user->subscription('default')->syncTaxRates();
```

這也將同步多產品訂閱的任何項目稅率。如果您的應用程式提供多產品訂閱，您應該確保您的可計費模型實作了[上面討論的](#subscription-taxes) `priceTaxRates` 方法。

<a name="tax-exemption"></a>
#### 免稅

Cashier 還提供 `isNotTaxExempt`、`isTaxExempt` 和 `reverseChargeApplies` 方法來判斷客戶是否免稅。這些方法將呼叫 Stripe API 來判斷客戶的免稅狀態：

```php
use App\Models\User;

$user = User::find(1);

$user->isTaxExempt();
$user->isNotTaxExempt();
$user->reverseChargeApplies();
```

> [!WARNING]
> 這些方法也適用於任何 `Laravel\Cashier\Invoice` 物件。但是，當在 `Invoice` 物件上呼叫時，這些方法將判斷發票建立時的免稅狀態。

<a name="subscription-anchor-date"></a>
### 訂閱錨定日期

預設情況下，帳單週期錨定日期是訂閱建立日期，如果使用試用期，則是試用期結束日期。如果您想修改帳單錨定日期，您可以使用 `anchorBillingCycleOn` 方法：

```php
use Illuminate\Http\Request;

Route::post('/user/subscribe', function (Request $request) {
    $anchor = Carbon::parse('first day of next month');

    $request->user()->newSubscription('default', 'price_monthly')
        ->anchorBillingCycleOn($anchor->startOfDay())
        ->create($request->paymentMethodId);

    // ...
});
```

有關管理訂閱帳單週期的更多資訊，請參閱 [Stripe 帳單週期說明文件](https://stripe.com/docs/billing/subscriptions/billing-cycle)。

<a name="cancelling-subscriptions"></a>
### 取消訂閱

要取消訂閱，請呼叫使用者訂閱上的 `cancel` 方法：

```php
$user->subscription('default')->cancel();
```

當訂閱被取消時，Cashier 將自動設定您的 `subscriptions` 資料庫資料表中的 `ends_at` 欄位。此欄位用於判斷 `subscribed` 方法何時應開始返回 `false`。

例如，如果客戶在 3 月 1 日取消訂閱，但訂閱原定於 3 月 5 日結束，則 `subscribed` 方法將繼續返回 `true` 直到 3 月 5 日。這樣做是因為使用者通常被允許繼續使用應用程式直到其帳單週期結束。

您可以使用 `onGracePeriod` 方法判斷使用者是否已取消訂閱但仍在「寬限期」內：

```php
if ($user->subscription('default')->onGracePeriod()) {
    // ...
}
```

如果您希望立即取消訂閱，請呼叫使用者訂閱上的 `cancelNow` 方法：

```php
$user->subscription('default')->cancelNow();
```

如果您希望立即取消訂閱並開立任何剩餘未開立發票的計量使用量或新的/待處理的按比例計費發票項目，請呼叫使用者訂閱上的 `cancelNowAndInvoice` 方法：

```php
$user->subscription('default')->cancelNowAndInvoice();
```

您也可以選擇在特定時間點取消訂閱：

```php
$user->subscription('default')->cancelAt(
    now()->addDays(10)
);
```

最後，您應該始終在刪除相關使用者模型之前取消使用者訂閱：

```php
$user->subscription('default')->cancelNow();

$user->delete();
```

<a name="resuming-subscriptions"></a>
### 恢復訂閱

如果客戶已取消訂閱並且您希望恢復它，您可以呼叫訂閱上的 `resume` 方法。客戶必須仍在「寬限期」內才能恢復訂閱：

```php
$user->subscription('default')->resume();
```

如果客戶取消訂閱然後在訂閱完全到期之前恢復該訂閱，則客戶將不會立即收到帳單。相反，他們的訂閱將被重新啟用，並將在原始帳單週期內收到帳單。

<a name="subscription-trials"></a>
## 訂閱試用期

<a name="with-payment-method-up-front"></a>
### 預先提供付款方式

如果您想向客戶提供試用期，同時仍預先收集付款方式資訊，您應該在建立訂閱時使用 `trialDays` 方法：

```php
use Illuminate\Http\Request;

Route::post('/user/subscribe', function (Request $request) {
    $request->user()->newSubscription('default', 'price_monthly')
        ->trialDays(10)
        ->create($request->paymentMethodId);

    // ...
});
```

此方法將設定資料庫中訂閱記錄的試用期結束日期，並指示 Stripe 在此日期之後才開始向客戶收費。使用 `trialDays` 方法時，Cashier 將覆寫 Stripe 中為價格設定的任何預設試用期。

> [!WARNING]
> 如果客戶的訂閱在試用期結束日期之前未取消，他們將在試用期到期後立即收到帳單，因此您應該務必通知您的使用者其試用期結束日期。

`trialUntil` 方法允許您提供一個 `DateTime` 實例，該實例指定試用期應何時結束：

```php
use Illuminate\Support\Carbon;

$user->newSubscription('default', 'price_monthly')
    ->trialUntil(Carbon::now()->addDays(10))
    ->create($paymentMethod);
```

您可以使用使用者實例的 `onTrial` 方法或訂閱實例的 `onTrial` 方法來判斷使用者是否處於試用期內。以下兩個範例是等效的：

```php
if ($user->onTrial('default')) {
    // ...
}

if ($user->subscription('default')->onTrial()) {
    // ...
}
```

您可以使用 `endTrial` 方法立即結束訂閱試用期：

```php
$user->subscription('default')->endTrial();
```

要判斷現有試用期是否已過期，您可以使用 `hasExpiredTrial` 方法：

```php
if ($user->hasExpiredTrial('default')) {
    // ...
}

if ($user->subscription('default')->hasExpiredTrial()) {
    // ...
}
```

<a name="defining-trial-days-in-stripe-cashier"></a>
#### 在 Stripe / Cashier 中定義試用天數

您可以選擇在 Stripe 控制面板中定義您的價格的試用天數，或者始終使用 Cashier 明確傳遞它們。如果您選擇在 Stripe 中定義您的價格的試用天數，您應該注意，新的訂閱，包括過去有訂閱的客戶的新訂閱，將始終收到試用期，除非您明確呼叫 `skipTrial()` 方法。

<a name="without-payment-method-up-front"></a>
### 不預先提供付款方式

如果您想在不預先收集使用者付款方式資訊的情況下提供試用期，您可以將使用者記錄上的 `trial_ends_at` 欄位設定為您想要的試用期結束日期。這通常在使用者註冊期間完成：

```php
use App\Models\User;

$user = User::create([
    // ...
    'trial_ends_at' => now()->addDays(10),
]);
```

> [!WARNING]
> 請務必在您的可計費模型的類別定義中為 `trial_ends_at` 屬性新增[日期轉換](/docs/{{version}}/eloquent-mutators#date-casting)。

Cashier 將此類型的試用期稱為「通用試用期」，因為它不附加到任何現有訂閱。可計費模型實例上的 `onTrial` 方法將返回 `true`，如果當前日期未超過 `trial_ends_at` 的值：

```php
if ($user->onTrial()) {
    // 使用者處於試用期內...
}
```

一旦您準備好為使用者建立實際訂閱，您可以像往常一樣使用 `newSubscription` 方法：

```php
$user = User::find(1);

$user->newSubscription('default', 'price_monthly')->create($paymentMethod);
```

要取得使用者的試用期結束日期，您可以使用 `trialEndsAt` 方法。如果使用者處於試用期內，此方法將返回 Carbon 日期實例，如果沒有，則返回 `null`。如果您想取得特定訂閱（而非預設訂閱）的試用期結束日期，您還可以傳遞一個可選的訂閱類型參數：

```php
if ($user->onTrial()) {
    $trialEndsAt = $user->trialEndsAt('main');
}
```

如果您希望明確知道使用者處於其「通用」試用期內且尚未建立實際訂閱，您也可以使用 `onGenericTrial` 方法：

```php
if ($user->onGenericTrial()) {
    // 使用者處於其「通用」試用期內...
}
```

<a name="extending-trials"></a>
### 延長試用期

`extendTrial` 方法允許您在訂閱建立後延長訂閱的試用期。如果試用期已過期且客戶已收到訂閱帳單，您仍然可以為他們提供延長試用期。試用期內花費的時間將從客戶的下一張發票中扣除：

```php
use App\Models\User;

$subscription = User::find(1)->subscription('default');

// 從現在起 7 天後結束試用期...
$subscription->extendTrial(
    now()->addDays(7)
);

// 為試用期額外增加 5 天...
$subscription->extendTrial(
    $subscription->trial_ends_at->addDays(5)
);
```

<a name="handling-stripe-webhooks"></a>
## 處理 Stripe Webhook

> [!NOTE]
> 您可以使用 [Stripe CLI](https://stripe.com/docs/stripe-cli) 在本地開發期間協助測試 Webhook。

Stripe 可以透過 Webhook 通知您的應用程式各種事件。預設情況下，指向 Cashier Webhook 控制器的路由會由 Cashier 服務提供者自動註冊。此控制器將處理所有傳入的 Webhook 請求。

預設情況下，Cashier Webhook 控制器將自動處理因太多失敗收費（由您的 Stripe 設定定義）而取消的訂閱、客戶更新、客戶刪除、訂閱更新和付款方式變更；但是，正如我們很快將發現的，您可以擴展此控制器以處理您喜歡的任何 Stripe Webhook 事件。

為確保您的應用程式可以處理 Stripe Webhook，請務必在 Stripe 控制面板中設定 Webhook URL。預設情況下，Cashier 的 Webhook 控制器響應 `/stripe/webhook` URL 路徑。您應該在 Stripe 控制面板中啟用的所有 Webhook 的完整列表是：

- `customer.subscription.created`
- `customer.subscription.updated`
- `customer.subscription.deleted`
- `customer.updated`
- `customer.deleted`
- `payment_method.automatically_updated`
- `invoice.payment_action_required`
- `invoice.payment_succeeded`

為了方便起見，Cashier 包含一個 `cashier:webhook` Artisan 命令。此命令將在 Stripe 中建立一個 Webhook，該 Webhook 監聽 Cashier 所需的所有事件：

```shell
php artisan cashier:webhook
```

預設情況下，建立的 Webhook 將指向由 `APP_URL` 環境變數定義的 URL 和 Cashier 中包含的 `cashier.webhook` 路由。如果您想使用不同的 URL，可以在呼叫命令時提供 `--url` 選項：

```shell
php artisan cashier:webhook --url "https://example.com/stripe/webhook"
```

建立的 Webhook 將使用您的 Cashier 版本相容的 Stripe API 版本。如果您想使用不同的 Stripe 版本，您可以提供 `--api-version` 選項：

```shell
php artisan cashier:webhook --api-version="2019-12-03"
```

建立後，Webhook 將立即啟用。如果您希望建立 Webhook 但在準備好之前將其停用，您可以在呼叫命令時提供 `--disabled` 選項：

```shell
php artisan cashier:webhook --disabled
```

> [!WARNING]
> 請務必使用 Cashier 包含的 [Webhook 簽章驗證](#verifying-webhook-signatures)中介軟體保護傳入的 Stripe Webhook 請求。

<a name="webhooks-csrf-protection"></a>
#### Webhook 與 CSRF 保護

由於 Stripe Webhook 需要繞過 Laravel 的 [CSRF 保護](/docs/{{version}}/csrf)，您應該確保 Laravel 不會嘗試驗證傳入 Stripe Webhook 的 CSRF 權杖。為此，您應該在應用程式的 `bootstrap/app.php` 檔案中將 `stripe/*` 從 CSRF 保護中排除：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->validateCsrfTokens(except: [
        'stripe/*',
    ]);
})
```

<a name="defining-webhook-event-handlers"></a>
### 定義 Webhook 事件處理器

Cashier 會自動處理因收費失敗而取消的訂閱以及其他常見的 Stripe Webhook 事件。但是，如果您有其他要處理的 Webhook 事件，您可以透過監聽 Cashier 分派的以下事件來完成：

- `Laravel\Cashier\Events\WebhookReceived`
- `Laravel\Cashier\Events\WebhookHandled`

這兩個事件都包含 Stripe Webhook 的完整 payload。例如，如果您希望處理 `invoice.payment_succeeded` Webhook，您可以註冊一個[監聽器](/docs/{{version}}/events#defining-listeners)來處理該事件：

```php
<?php

namespace App\Listeners;

use Laravel\Cashier\Events\WebhookReceived;

class StripeEventListener
{
    /**
     * 處理收到的 Stripe Webhook。
     */
    public function handle(WebhookReceived $event): void
    {
        if ($event->payload['type'] === 'invoice.payment_succeeded') {
            // 處理傳入的事件...
        }
    }
}
```

<a name="verifying-webhook-signatures"></a>
### 驗證 Webhook 簽章

為了保護您的 Webhook，您可以使用 [Stripe 的 Webhook 簽章](https://stripe.com/docs/webhooks/signatures)。為了方便起見，Cashier 自動包含一個中介軟體，用於驗證傳入的 Stripe Webhook 請求是否有效。

要啟用 Webhook 驗證，請確保在應用程式的 `.env` 檔案中設定了 `STRIPE_WEBHOOK_SECRET` 環境變數。Webhook `secret` 可以從您的 Stripe 帳戶控制面板中取得。

<a name="single-charges"></a>
## 單次收費

<a name="simple-charge"></a>
### 簡單收費

如果您想對客戶進行一次性收費，您可以使用可計費模型實例上的 `charge` 方法。您需要將[付款方式識別碼](#payment-methods-for-single-charges)作為第二個參數傳遞給 `charge` 方法：

```php
use Illuminate\Http\Request;

Route::post('/purchase', function (Request $request) {
    $stripeCharge = $request->user()->charge(
        100, $request->paymentMethodId
    );

    // ...
});
```

`charge` 方法接受一個陣列作為其第三個參數，允許您將任何選項傳遞給底層的 Stripe 收費建立。有關建立收費時可用的選項的更多資訊，請參閱 [Stripe 說明文件](https://stripe.com/docs/api/charges/create)：

```php
$user->charge(100, $paymentMethod, [
    'custom_option' => $value,
]);
```

您也可以在沒有底層客戶或使用者的情況下使用 `charge` 方法。為此，請在應用程式可計費模型的新實例上呼叫 `charge` 方法：

```php
use App\Models\User;

$stripeCharge = (new User)->charge(100, $paymentMethod);
```

如果收費失敗，`charge` 方法將拋出例外。如果收費成功，該方法將返回 `Laravel\Cashier\Payment` 的實例：

```php
try {
    $payment = $user->charge(100, $paymentMethod);
} catch (Exception $e) {
    // ...
}
```

> [!WARNING]
> `charge` 方法接受應用程式使用的貨幣的最低面額的付款金額。例如，如果客戶以美元支付，則金額應以美分指定。

<a name="charge-with-invoice"></a>
### 含發票收費

有時您可能需要進行一次性收費並向客戶提供 PDF 發票。`invoicePrice` 方法可以做到這一點。例如，讓我們為客戶開立五件新襯衫的發票：

```php
$user->invoicePrice('price_tshirt', 5);
```

發票將立即從使用者的預設付款方式中收取。`invoicePrice` 方法也接受一個陣列作為其第三個參數。此陣列包含發票項目的帳單選項。該方法接受的第四個參數也是一個陣列，其中應包含發票本身的帳單選項：

```php
$user->invoicePrice('price_tshirt', 5, [
    'discounts' => [
        ['coupon' => 'SUMMER21SALE']
    ],
], [
    'default_tax_rates' => ['txr_id'],
]);
```

類似於 `invoicePrice`，您可以使用 `tabPrice` 方法透過將多個項目（每張發票最多 250 個項目）新增到客戶的「帳單」中，然後向客戶開立發票來建立一次性收費。例如，我們可以為客戶開立五件襯衫和兩個馬克杯的發票：

```php
$user->tabPrice('price_tshirt', 5);
$user->tabPrice('price_mug', 2);
$user->invoice();
```

或者，您可以使用 `invoiceFor` 方法對客戶的預設付款方式進行「一次性」收費：

```php
$user->invoiceFor('One Time Fee', 500);
```

儘管 `invoiceFor` 方法可供您使用，但建議您使用 `invoicePrice` 和 `tabPrice` 方法以及預定義的價格。這樣做，您將能夠在您的 Stripe 控制面板中獲得更好的分析和資料，以了解您按產品銷售的情況。

> [!WARNING]
> `invoice`、`invoicePrice` 和 `invoiceFor` 方法將建立一個 Stripe 發票，該發票將重試失敗的帳單嘗試。如果您不希望發票重試失敗的收費，您將需要在第一次收費失敗後使用 Stripe API 關閉它們。

<a name="creating-payment-intents"></a>
### 建立 Payment Intent

您可以透過在可計費模型實例上呼叫 `pay` 方法來建立新的 Stripe Payment Intent。呼叫此方法將建立一個包裝在 `Laravel\Cashier\Payment` 實例中的 Payment Intent：

```php
use Illuminate\Http\Request;

Route::post('/pay', function (Request $request) {
    $payment = $request->user()->pay(
        $request->get('amount')
    );

    return $payment->client_secret;
});
```

建立 Payment Intent 後，您可以將客戶端密鑰返回給應用程式的前端，以便使用者可以在其瀏覽器中完成付款。要了解有關使用 Stripe Payment Intent 建立整個支付流程的更多資訊，請參閱 [Stripe 說明文件](https://stripe.com/docs/payments/accept-a-payment?platform=web)。

使用 `pay` 方法時，您的 Stripe 控制面板中啟用的預設付款方式將可供客戶使用。或者，如果您只想允許使用某些特定的付款方式，您可以使用 `payWith` 方法：

```php
use Illuminate\Http\Request;

Route::post('/pay', function (Request $request) {
    $payment = $request->user()->payWith(
        $request->get('amount'), ['card', 'bancontact']
    );

    return $payment->client_secret;
});
```

> [!WARNING]
> `pay` 和 `payWith` 方法接受應用程式使用的貨幣的最低面額的付款金額。例如，如果客戶以美元支付，則金額應以美分指定。

<a name="refunding-charges"></a>
### 退款

如果您需要退款 Stripe 收費，您可以使用 `refund` 方法。此方法將 Stripe [Payment Intent ID](#payment-methods-for-single-charges) 作為其第一個參數：

```php
$payment = $user->charge(100, $paymentMethodId);

$user->refund($payment->id);
```

<a name="checkout"></a>
## Checkout

Cashier Stripe 也支援 [Stripe Checkout](https://stripe.com/payments/checkout)。Stripe Checkout 透過提供預先建置的託管支付頁面，消除了實作自訂頁面以接受付款的麻煩。

以下說明文件包含有關如何開始使用 Stripe Checkout 與 Cashier 的資訊。要了解有關 Stripe Checkout 的更多資訊，您還應該考慮查閱 [Stripe 關於 Checkout 的說明文件](https://stripe.com/docs/payments/checkout)。

<a name="product-checkouts"></a>
### 產品 Checkout

您可以使用可計費模型上的 `checkout` 方法對已在您的 Stripe 控制面板中建立的現有產品執行 Checkout。`checkout` 方法將啟動新的 Stripe Checkout 會話。預設情況下，您需要傳遞一個 Stripe Price ID：

```php
use Illuminate\Http\Request;

Route::get('/product-checkout', function (Request $request) {
    return $request->user()->checkout('price_tshirt');
});
```

如有必要，您還可以指定產品數量：

```php
use Illuminate\Http\Request;

Route::get('/product-checkout', function (Request $request) {
    return $request->user()->checkout(['price_tshirt' => 15]);
});
```

當客戶訪問此路由時，他們將被重新導向到 Stripe 的 Checkout 頁面。預設情況下，當使用者成功完成或取消購買時，他們將被重新導向到您的 `home` 路由位置，但您可以使用 `success_url` 和 `cancel_url` 選項指定自訂回呼 URL：

```php
use Illuminate\Http\Request;

Route::get('/product-checkout', function (Request $request) {
    return $request->user()->checkout(['price_tshirt' => 1], [
        'success_url' => route('your-success-route'),
        'cancel_url' => route('your-cancel-route'),
    ]);
});
```

在定義您的 `success_url` Checkout 選項時，您可以指示 Stripe 在呼叫您的 URL 時將 Checkout 會話 ID 作為查詢字串參數新增。為此，請將字串 `{CHECKOUT_SESSION_ID}` 新增到您的 `success_url` 查詢字串中。Stripe 將用實際的 Checkout 會話 ID 替換此佔位符：

```php
use Illuminate\Http\Request;
use Stripe\Checkout\Session;
use Stripe\Customer;

Route::get('/product-checkout', function (Request $request) {
    return $request->user()->checkout(['price_tshirt' => 1], [
        'success_url' => route('checkout-success').'?session_id={CHECKOUT_SESSION_ID}',
        'cancel_url' => route('checkout-cancel'),
    ]);
});

Route::get('/checkout-success', function (Request $request) {
    $checkoutSession = $request->user()->stripe()->checkout->sessions->retrieve($request->get('session_id'));

    return view('checkout.success', ['checkoutSession' => $checkoutSession]);
})->name('checkout-success');
```

<a name="checkout-promotion-codes"></a>
#### 促銷代碼

預設情況下，Stripe Checkout 不允許[使用者可兌換的促銷代碼](https://stripe.com/docs/billing/subscriptions/discounts/codes)。幸運的是，有一種簡單的方法可以為您的 Checkout 頁面啟用這些功能。為此，您可以呼叫 `allowPromotionCodes` 方法：

```php
use Illuminate\Http\Request;

Route::get('/product-checkout', function (Request $request) {
    return $request->user()
        ->allowPromotionCodes()
        ->checkout('price_tshirt');
});
```

<a name="single-charge-checkouts"></a>
### 單次收費 Checkout

您還可以為未在您的 Stripe 控制面板中建立的臨時產品執行簡單收費。為此，您可以使用可計費模型上的 `checkoutCharge` 方法，並傳遞可收費金額、產品名稱和可選數量。當客戶訪問此路由時，他們將被重新導向到 Stripe 的 Checkout 頁面：

```php
use Illuminate\Http\Request;

Route::get('/charge-checkout', function (Request $request) {
    return $request->user()->checkoutCharge(1200, 'T-Shirt', 5);
});
```

> [!WARNING]
> 使用 `checkoutCharge` 方法時，Stripe 將始終在您的 Stripe 控制面板中建立新的產品和價格。因此，我們建議您預先在您的 Stripe 控制面板中建立產品並改用 `checkout` 方法。

<a name="subscription-checkouts"></a>
### 訂閱 Checkout

> [!WARNING]
> 使用 Stripe Checkout 進行訂閱需要您在 Stripe 控制面板中啟用 `customer.subscription.created` Webhook。此 Webhook 將在您的資料庫中建立訂閱記錄並儲存所有相關的訂閱項目。

您也可以使用 Stripe Checkout 來啟動訂閱。使用 Cashier 的訂閱建構器方法定義訂閱後，您可以呼叫 `checkout` 方法。當客戶訪問此路由時，他們將被重新導向到 Stripe 的 Checkout 頁面：

```php
use Illuminate\Http\Request;

Route::get('/subscription-checkout', function (Request $request) {
    return $request->user()
        ->newSubscription('default', 'price_monthly')
        ->checkout();
});
```

就像產品 Checkout 一樣，您可以自訂成功和取消 URL：

```php
use Illuminate\Http\Request;

Route::get('/subscription-checkout', function (Request $request) {
    return $request->user()
        ->newSubscription('default', 'price_monthly')
        ->checkout([
            'success_url' => route('your-success-route'),
            'cancel_url' => route('your-cancel-route'),
        ]);
});
```

當然，您也可以為訂閱 Checkout 啟用促銷代碼：

```php
use Illuminate\Http\Request;

Route::get('/subscription-checkout', function (Request $request) {
    return $request->user()
        ->newSubscription('default', 'price_monthly')
        ->allowPromotionCodes()
        ->checkout();
});
```

> [!WARNING]
> 不幸的是，Stripe Checkout 在啟動訂閱時不支援所有訂閱帳單選項。在訂閱建構器上使用 `anchorBillingCycleOn` 方法、設定按比例計費行為或設定付款行為在 Stripe Checkout 會話期間不會產生任何影響。請查閱 [Stripe Checkout Session API 說明文件](https://stripe.com/docs/api/checkout/sessions/create)以查看哪些參數可用。

<a name="stripe-checkout-trial-periods"></a>
#### Stripe Checkout 與試用期

當然，您可以在建立將使用 Stripe Checkout 完成的訂閱時定義試用期：

```php
$checkout = Auth::user()->newSubscription('default', 'price_monthly')
    ->trialDays(3)
    ->checkout();
```

但是，試用期必須至少為 48 小時，這是 Stripe Checkout 支援的最短試用時間。

<a name="stripe-checkout-subscriptions-and-webhooks"></a>
#### 訂閱與 Webhook

請記住，Stripe 和 Cashier 透過 Webhook 更新訂閱狀態，因此客戶在輸入付款資訊後返回應用程式時，訂閱可能尚未啟用。為了處理這種情況，您可能希望顯示一條訊息，通知使用者其付款或訂閱正在等待處理。

<a name="collecting-tax-ids"></a>
### 收集稅籍編號

Checkout 也支援收集客戶的稅籍編號。要在 Checkout 會話中啟用此功能，請在建立會話時呼叫 `collectTaxIds` 方法：

```php
$checkout = $user->collectTaxIds()->checkout('price_tshirt');
```

呼叫此方法後，客戶將可以使用一個新的核取方塊，允許他們表明他們是否以公司身份購買。如果是，他們將有機會提供其稅籍編號。

> [!WARNING]
> 如果您已在應用程式的服務提供者中設定了[自動稅金收集](#tax-configuration)，則此功能將自動啟用，無需呼叫 `collectTaxIds` 方法。

<a name="guest-checkouts"></a>
### 訪客 Checkout

使用 `Checkout::guest` 方法，您可以為應用程式中沒有「帳戶」的訪客啟動 Checkout 會話：

```php
use Illuminate\Http\Request;
use Laravel\Cashier\Checkout;

Route::get('/product-checkout', function (Request $request) {
    return Checkout::guest()->create('price_tshirt', [
        'success_url' => route('your-success-route'),
        'cancel_url' => route('your-cancel-route'),
    ]);
});
```

類似於為現有使用者建立 Checkout 會話時，您可以使用 `Laravel\Cashier\CheckoutBuilder` 實例上可用的其他方法來自訂訪客 Checkout 會話：

```php
use Illuminate\Http\Request;
use Laravel\Cashier\Checkout;

Route::get('/product-checkout', function (Request $request) {
    return Checkout::guest()
        ->withPromotionCode('promo-code')
        ->create('price_tshirt', [
            'success_url' => route('your-success-route'),
            'cancel_url' => route('your-cancel-route'),
        ]);
});
```

訪客 Checkout 完成後，Stripe 可以分派 `checkout.session.completed` Webhook 事件，因此請務必[設定您的 Stripe Webhook](https://dashboard.stripe.com/webhooks) 以實際將此事件發送到您的應用程式。一旦在 Stripe 控制面板中啟用了 Webhook，您就可以[使用 Cashier 處理 Webhook](#handling-stripe-webhooks)。Webhook payload 中包含的物件將是一個[Checkout 物件](https://stripe.com/docs/api/checkout/sessions/object)，您可以檢查該物件以完成客戶的訂單。

<a name="handling-failed-payments"></a>
## 處理付款失敗

有時，訂閱或單次收費的付款可能會失敗。發生這種情況時，Cashier 將拋出 `Laravel\Cashier\Exceptions\IncompletePayment` 例外，通知您發生了這種情況。捕獲此例外後，您有兩種處理方式。

首先，您可以將客戶重新導向到 Cashier 中包含的專用付款確認頁面。此頁面已透過 Cashier 的服務提供者註冊了一個相關的命名路由。因此，您可以捕獲 `IncompletePayment` 例外並將使用者重新導向到付款確認頁面：

```php
use Laravel\Cashier\Exceptions\IncompletePayment;

try {
    $subscription = $user->newSubscription('default', 'price_monthly')
        ->create($paymentMethod);
} catch (IncompletePayment $exception) {
    return redirect()->route(
        'cashier.payment',
        [$exception->payment->id, 'redirect' => route('home')]
    );
}
```

在付款確認頁面上，客戶將被提示再次輸入其信用卡資訊並執行 Stripe 要求的任何額外操作，例如「3D Secure」確認。確認付款後，使用者將被重新導向到上面指定的 `redirect` 參數提供的 URL。重新導向後，`message` (字串) 和 `success` (整數) 查詢字串變數將新增到 URL。付款頁面目前支援以下付款方式類型：

<div class="content-list" markdown="1">

- 信用卡
- 支付寶
- Bancontact
- BECS 直接扣款
- EPS
- Giropay
- iDEAL
- SEPA 直接扣款

</div>

或者，您可以讓 Stripe 為您處理付款確認。在這種情況下，您可以[在您的 Stripe 控制面板中設定 Stripe 的自動帳單電子郵件](https://dashboard.stripe.com/account/billing/automatic)，而不是重新導向到付款確認頁面。但是，如果捕獲到 `IncompletePayment` 例外，您仍應通知使用者他們將收到一封包含進一步付款確認說明的電子郵件。

付款例外可能會針對以下方法拋出：使用 `Billable` trait 的模型上的 `charge`、`invoiceFor` 和 `invoice`。與訂閱互動時，`SubscriptionBuilder` 上的 `create` 方法，以及 `Subscription` 和 `SubscriptionItem` 模型上的 `incrementAndInvoice` 和 `swapAndInvoice` 方法可能會拋出不完整付款例外。

判斷現有訂閱是否具有不完整付款可以使用可計費模型或訂閱實例上的 `hasIncompletePayment` 方法來完成：

```php
if ($user->hasIncompletePayment('default')) {
    // ...
}

if ($user->subscription('default')->hasIncompletePayment()) {
    // ...
}
```

您可以透過檢查例外實例上的 `payment` 屬性來取得不完整付款的特定狀態：

```php
use Laravel\Cashier\Exceptions\IncompletePayment;

try {
    $user->charge(1000, 'pm_card_threeDSecure2Required');
} catch (IncompletePayment $exception) {
    // 取得 Payment Intent 狀態...
    $exception->payment->status;

    // 檢查特定條件...
    if ($exception->payment->requiresPaymentMethod()) {
        // ...
    } elseif ($exception->payment->requiresConfirmation()) {
        // ...
    }
}
```

<a name="confirming-payments"></a>
### 確認付款

某些付款方式需要額外資料才能確認付款。例如，SEPA 付款方式在付款過程中需要額外的「授權」資料。您可以使用 `withPaymentConfirmationOptions` 方法向 Cashier 提供此資料：

```php
$subscription->withPaymentConfirmationOptions([
    'mandate_data' => '...',
])->swap('price_xxx');
```

您可以查閱 [Stripe API 說明文件](https://stripe.com/docs/api/payment_intents/confirm)以查看確認付款時接受的所有選項。

<a name="strong-customer-authentication"></a>
## 強客戶認證 (SCA)

如果您的企業或您的客戶之一位於歐洲，您將需要遵守歐盟的強客戶認證 (SCA) 法規。這些法規於 2019 年 9 月由歐盟實施，旨在防止支付詐欺。幸運的是，Stripe 和 Cashier 已準備好建立符合 SCA 的應用程式。

> [!WARNING]
> 在開始之前，請查閱 [Stripe 關於 PSD2 和 SCA 的指南](https://stripe.com/guides/strong-customer-authentication)以及他們[關於新 SCA API 的說明文件](https://stripe.com/docs/strong-customer-authentication)。

<a name="payments-requiring-additional-confirmation"></a>
### 需要額外確認的付款

SCA 法規通常需要額外驗證才能確認和處理付款。發生這種情況時，Cashier 將拋出 `Laravel\Cashier\Exceptions\IncompletePayment` 例外，通知您需要額外驗證。有關如何處理這些例外的更多資訊，請參閱[處理付款失敗](#handling-failed-payments)的說明文件。

Stripe 或 Cashier 呈現的付款確認畫面可能會根據特定銀行或發卡機構的付款流程進行調整，並且可能包括額外的卡片確認、臨時小額收費、單獨的裝置驗證或其他形式的驗證。

<a name="incomplete-and-past-due-state"></a>
#### 不完整和逾期狀態

當付款需要額外確認時，訂閱將保持 `incomplete` 或 `past_due` 狀態，如其 `stripe_status` 資料庫欄位所示。一旦付款確認完成並且您的應用程式透過 Webhook 收到 Stripe 的通知，Cashier 將自動啟用客戶的訂閱。

有關 `incomplete` 和 `past_due` 狀態的更多資訊，請參閱[我們關於這些狀態的額外說明文件](#incomplete-and-past-due-status)。

<a name="off-session-payment-notifications"></a>
### 離線付款通知

由於 SCA 法規要求客戶即使在訂閱有效時也偶爾需要驗證其付款詳細資訊，因此當需要離線付款確認時，Cashier 可以向客戶發送通知。例如，這可能發生在訂閱續訂時。Cashier 的付款通知可以透過將 `CASHIER_PAYMENT_NOTIFICATION` 環境變數設定為通知類別來啟用。預設情況下，此通知是停用的。當然，Cashier 包含一個您可以為此目的使用的通知類別，但如果您願意，您可以自由提供自己的通知類別：

```ini
CASHIER_PAYMENT_NOTIFICATION=Laravel\Cashier\Notifications\ConfirmPayment
```

為確保離線付款確認通知送達，請驗證您的應用程式是否[設定了 Stripe Webhook](#handling-stripe-webhooks)，並且在您的 Stripe 控制面板中啟用了 `invoice.payment_action_required` Webhook。此外，您的 `Billable` 模型也應該使用 Laravel 的 `Illuminate\Notifications\Notifiable` trait。

> [!WARNING]
> 即使客戶手動進行需要額外確認的付款，也會發送通知。不幸的是，Stripe 無法知道付款是手動完成還是「離線」完成。但是，如果客戶在已經確認付款後訪問付款頁面，他們將只會看到「付款成功」訊息。客戶將不允許意外地兩次確認相同的付款並產生意外的第二次收費。

<a name="stripe-sdk"></a>
## Stripe SDK

Cashier 的許多物件都是 Stripe SDK 物件的包裝器。如果您想直接與 Stripe 物件互動，您可以方便地使用 `asStripe` 方法取得它們：

```php
$stripeSubscription = $subscription->asStripeSubscription();

$stripeSubscription->application_fee_percent = 5;

$stripeSubscription->save();
```

您也可以使用 `updateStripeSubscription` 方法直接更新 Stripe 訂閱：

```php
$subscription->updateStripeSubscription(['application_fee_percent' => 5]);
```

如果您想直接使用 `Stripe\StripeClient` 客戶端，您可以呼叫 `Cashier` 類別上的 `stripe` 方法。例如，您可以使用此方法存取 `StripeClient` 實例並從您的 Stripe 帳戶中取得價格列表：

```php
use Laravel\Cashier\Cashier;

$prices = Cashier::stripe()->prices->all();
```

<a name="testing"></a>
## 測試

在測試使用 Cashier 的應用程式時，您可以模擬對 Stripe API 的實際 HTTP 請求；但是，這需要您部分重新實作 Cashier 自己的行為。因此，我們建議允許您的測試實際命中 Stripe API。雖然這會較慢，但它提供了更大的信心，確保您的應用程式按預期工作，並且任何慢速測試都可以放在自己的 Pest / PHPUnit 測試組中。

測試時，請記住 Cashier 本身已經有一個很棒的測試套件，因此您應該只專注於測試您自己應用程式的訂閱和支付流程，而不是每個底層 Cashier 行為。

首先，將您的 Stripe 密鑰的**測試**版本新增到您的 `phpunit.xml` 檔案中：

```xml
<env name="STRIPE_SECRET" value="sk_test_<your-key>"/>
```

現在，每當您在測試時與 Cashier 互動時，它都會向您的 Stripe 測試環境發送實際的 API 請求。為了方便起見，您應該預先填寫您的 Stripe 測試帳戶，其中包含您在測試期間可能使用的訂閱/價格。

> [!NOTE]
> 為了測試各種帳單場景，例如信用卡拒絕和失敗，您可以使用 Stripe 提供的各種[測試卡號和權杖](https://stripe.com/docs/testing)。

