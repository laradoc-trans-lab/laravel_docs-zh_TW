# Laravel Cashier (Stripe)

- [簡介](#introduction)
- [升級 Cashier](#upgrading-cashier)
- [安裝](#installation)
- [設定](#configuration)
    - [可計費模型](#billable-model)
    - [API 密鑰](#api-keys)
    - [貨幣設定](#currency-configuration)
    - [稅務設定](#tax-configuration)
    - [日誌](#logging)
    - [使用自訂模型](#using-custom-models)
- [快速入門](#quickstart)
    - [銷售產品](#quickstart-selling-products)
    - [銷售訂閱](#quickstart-selling-subscriptions)
- [客戶](#customers)
    - [擷取客戶](#retrieving-customers)
    - [建立客戶](#creating-customers)
    - [更新客戶](#updating-customers)
    - [餘額](#balances)
    - [稅號](#tax-ids)
    - [與 Stripe 同步客戶資料](#syncing-customer-data-with-stripe)
    - [帳務入口](#billing-portal)
- [支付方式](#payment-methods)
    - [儲存支付方式](#storing-payment-methods)
    - [擷取支付方式](#retrieving-payment-methods)
    - [支付方式存在性](#payment-method-presence)
    - [更新預設支付方式](#updating-the-default-payment-method)
    - [新增支付方式](#adding-payment-methods)
    - [刪除支付方式](#deleting-payment-methods)
- [訂閱](#subscriptions)
    - [建立訂閱](#creating-subscriptions)
    - [檢查訂閱狀態](#checking-subscription-status)
    - [變更價格](#changing-prices)
    - [訂閱數量](#subscription-quantity)
    - [多產品訂閱](#subscriptions-with-multiple-products)
    - [多重訂閱](#multiple-subscriptions)
    - [按用量計費](#usage-based-billing)
    - [訂閱稅金](#subscription-taxes)
    - [訂閱錨定日期](#subscription-anchor-date)
    - [取消訂閱](#cancelling-subscriptions)
    - [恢復訂閱](#resuming-subscriptions)
- [訂閱試用](#subscription-trials)
    - [預先提供支付方式](#with-payment-method-up-front)
    - [無需預先提供支付方式](#without-payment-method-up-front)
    - [延長試用](#extending-trials)
- [處理 Stripe Webhooks](#handling-stripe-webhooks)
    - [定義 Webhook 事件處理器](#defining-webhook-event-handlers)
    - [驗證 Webhook 簽章](#verifying-webhook-signatures)
- [單次扣款](#single-charges)
    - [簡單扣款](#simple-charge)
    - [帶有發票的扣款](#charge-with-invoice)
    - [建立支付意圖](#creating-payment-intents)
    - [退款](#refunding-charges)
- [結帳](#checkout)
    - [產品結帳](#product-checkouts)
    - [單次扣款結帳](#single-charge-checkouts)
    - [訂閱結帳](#subscription-checkouts)
    - [收集稅號](#collecting-tax-ids)
    - [訪客結帳](#guest-checkouts)
- [發票](#invoices)
    - [擷取發票](#retrieving-invoices)
    - [即將產生的發票](#upcoming-invoices)
    - [預覽訂閱發票](#previewing-subscription-invoices)
    - [產生發票 PDF](#generating-invoice-pdfs)
- [處理失敗的支付](#handling-failed-payments)
    - [確認支付](#confirming-payments)
- [強客戶驗證 (SCA)](#strong-customer-authentication)
    - [需要額外確認的支付](#payments-requiring-additional-confirmation)
    - [離線支付通知](#off-session-payment-notifications)
- [Stripe SDK](#stripe-sdk)
- [測試](#testing)

<a name="introduction"></a>
## 簡介

[Laravel Cashier Stripe](https://github.com/laravel/cashier-stripe) 提供了一個富表達力、流暢的介面，用於 [Stripe 的](https://stripe.com) 訂閱計費服務。它處理了您不願編寫的幾乎所有繁瑣的訂閱計費樣板程式碼。除了基本的訂閱管理，Cashier 還能處理優惠券、交換訂閱、訂閱「數量」、取消寬限期，甚至能產生發票 PDF。

<a name="upgrading-cashier"></a>
## 升級 Cashier

當升級到新版 Cashier 時，仔細查閱[升級指南](https://github.com/laravel/cashier-stripe/blob/master/UPGRADE.md)非常重要。

> [!WARNING]
> 為了防止破壞性變更，Cashier 使用固定的 Stripe API 版本。Cashier 16 使用 Stripe API 版本 `2025-07-30.basil`。Stripe API 版本將會在次要發布版本中更新，以利用 Stripe 的新功能與改進。

<a name="installation"></a>
## 安裝

首先，使用 Composer 套件管理器安裝 Cashier 的 Stripe 套件：

```shell
composer require laravel/cashier
```

安裝套件後，使用 `vendor:publish` Artisan 指令發佈 Cashier 的遷移檔：

```shell
php artisan vendor:publish --tag="cashier-migrations"
```

接著，遷移您的資料庫：

```shell
php artisan migrate
```

Cashier 的遷移檔將會為您的 `users` 資料表新增多個欄位。它們還會建立一個新的 `subscriptions` 資料表，以存放所有客戶的訂閱，以及一個 `subscription_items` 資料表，用於存放包含多個價格的訂閱。

如果您願意，您也可以使用 `vendor:publish` Artisan 指令發佈 Cashier 的設定檔：

```shell
php artisan vendor:publish --tag="cashier-config"
```

最後，為了確保 Cashier 正確處理所有 Stripe 事件，請記得[設定 Cashier 的 webhook 處理](#handling-stripe-webhooks)。

> [!WARNING]
> Stripe 建議任何用於存放 Stripe 識別碼的欄位都應區分大小寫。因此，當使用 MySQL 時，您應確保 `stripe_id` 欄位的排序規則 (collation) 設定為 `utf8_bin`。有關此項的更多資訊，請參閱 [Stripe 文件](https://stripe.com/docs/upgrades#what-changes-does-stripe-consider-to-be-backwards-compatible)。

<a name="configuration"></a>
## 設定

<a name="billable-model"></a>
### 可計費模型

在使用 Cashier 之前，請將 `Billable` Trait 新增到您的可計費模型定義中。通常，這會是 `App\Models\User` 模型。此 Trait 提供了多種方法，讓您能執行常見的計費任務，例如建立訂閱、應用優惠券和更新支付方式資訊：

```php
use Laravel\Cashier\Billable;

class User extends Authenticatable
{
    use Billable;
}
```

Cashier 預設您的可計費模型將是 Laravel 提供的 `App\Models\User` 類別。如果您希望變更此設定，可以透過 `useCustomerModel` 方法指定不同的模型。此方法通常應在應用程式 `AppServiceProvider` 類別的 `boot` 方法中呼叫：

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
> 如果您使用的模型不是 Laravel 提供的 `App\Models\User` 模型，您將需要發佈並修改提供的 [Cashier 遷移檔](#installation) 以符合您替代模型的資料表名稱。

<a name="api-keys"></a>
### API 密鑰

接著，您應該在應用程式的 `.env` 檔案中設定您的 Stripe API 密鑰。您可以從 Stripe 控制面板中擷取您的 Stripe API 密鑰：

```ini
STRIPE_KEY=your-stripe-key
STRIPE_SECRET=your-stripe-secret
STRIPE_WEBHOOK_SECRET=your-stripe-webhook-secret
```

> [!WARNING]
> 您應該確保 `STRIPE_WEBHOOK_SECRET` 環境變數已定義在應用程式的 `.env` 檔案中，因為此變數用於確保傳入的 webhooks 確實來自 Stripe。

<a name="currency-configuration"></a>
### 貨幣設定

Cashier 預設貨幣為美元 (USD)。您可以透過設定應用程式 `.env` 檔案中的 `CASHIER_CURRENCY` 環境變數來變更預設貨幣：

```ini
CASHIER_CURRENCY=eur
```

除了設定 Cashier 的貨幣，您還可以指定一個語系 (locale) 用於格式化顯示在發票上的貨幣值。在內部，Cashier 利用 [PHP 的 `NumberFormatter` 類別](https://www.php.net/manual/en/class.numberformatter.php) 來設定貨幣語系：

```ini
CASHIER_CURRENCY_LOCALE=nl_BE
```

> [!WARNING]
> 為了使用 `en` 以外的語系，請確保您的伺服器上已安裝並設定 `ext-intl` PHP 擴充功能。

<a name="tax-configuration"></a>
### 稅務設定

感謝 [Stripe Tax](https://stripe.com/tax)，它能為所有由 Stripe 產生的發票自動計算稅金。您可以透過在應用程式 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫 `calculateTaxes` 方法來啟用自動稅金計算：

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

一旦稅金計算功能啟用，任何新訂閱和任何一次性發票都將會自動計算稅金。

為了使此功能正常運作，您的客戶帳單詳細資料，例如客戶姓名、地址和稅號，需要同步到 Stripe。您可以使用 Cashier 提供的[客戶資料同步](#syncing-customer-data-with-stripe)和[稅號](#tax-ids)方法來達成此目的。

<a name="logging"></a>
### 日誌

Cashier 允許您指定在記錄嚴重的 Stripe 錯誤時要使用的日誌頻道。您可以透過在應用程式的 `.env` 檔案中定義 `CASHIER_LOGGER` 環境變數來指定日誌頻道：

```ini
CASHIER_LOGGER=stack
```

透過 API 呼叫 Stripe 所產生的例外將會透過應用程式的預設日誌頻道記錄。

<a name="using-custom-models"></a>
### 使用自訂模型

您可以自由擴展 Cashier 內部使用的模型，方法是定義您自己的模型並擴展對應的 Cashier 模型：

```php
use Laravel\Cashier\Subscription as CashierSubscription;

class Subscription extends CashierSubscription
{
    // ...
}
```

定義模型後，您可以透過 `Laravel\Cashier\Cashier` 類別指示 Cashier 使用您的自訂模型。通常，您應該在應用程式 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中告知 Cashier 您的自訂模型：

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
> 在使用 Stripe Checkout 之前，您應該在 Stripe Dashboard 中定義具有固定價格的產品。此外，您也應該[設定 Cashier 的 Webhook 處理](#handling-stripe-webhooks)。

透過您的應用程式提供產品和訂閱計費可能令人望而生畏。然而，由於 Cashier 和 [Stripe Checkout](https://stripe.com/payments/checkout)，您可以輕鬆建立現代、強大的支付整合。

為了向客戶收取非經常性的單次費用產品，我們將利用 Cashier 將客戶導向 Stripe Checkout，在那裡他們將提供支付詳細資訊並確認購買。一旦透過 Checkout 完成支付，客戶將會被重新導向到您應用程式中選擇的成功 URL：

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

如上例所示，我們將利用 Cashier 提供的 `checkout` 方法將客戶導向 Stripe Checkout 以取得給定的「價格識別碼」。使用 Stripe 時，「prices」指的是[針對特定產品定義的價格](https://stripe.com/docs/products-prices/how-products-and-prices-work)。

如有必要，`checkout` 方法將會自動在 Stripe 中建立客戶，並將該 Stripe 客戶記錄連接到您應用程式資料庫中相對應的使用者。完成結帳會話後，客戶將會被重新導向到專門的成功或取消頁面，您可以在其中向客戶顯示資訊性訊息。

<a name="providing-meta-data-to-stripe-checkout"></a>
#### 提供中繼資料給 Stripe Checkout

銷售產品時，通常會透過您應用程式中定義的 `Cart` 和 `Order` 模型來追蹤已完成的訂單和已購買的產品。當將客戶導向 Stripe Checkout 以完成購買時，您可能需要提供現有的訂單識別碼，以便在客戶重新導向回您的應用程式時，將已完成的購買與相對應的訂單關聯起來。

為了實現這一點，您可以向 `checkout` 方法提供一個 `metadata` 陣列。假設當使用者開始結帳流程時，我們的應用程式中建立了一個待處理的 `Order`。請記住，此範例中的 `Cart` 和 `Order` 模型僅為說明性，並非由 Cashier 提供。您可以根據自己應用程式的需求自由實作這些概念：

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

如上例所示，當使用者開始結帳流程時，我們將提供所有與購物車/訂單相關聯的 Stripe 價格識別碼給 `checkout` 方法。當然，您的應用程式負責將這些項目與客戶添加時的「購物車」或訂單相關聯。我們也透過 `metadata` 陣列將訂單的 ID 提供給 Stripe Checkout 會話。最後，我們已將 `CHECKOUT_SESSION_ID` 模板變數新增到 Checkout 成功路由。當 Stripe 將客戶重新導向回您的應用程式時，此模板變數將會自動填入 Checkout 會話 ID。

接下來，讓我們建立 Checkout 成功路由。這是使用者在透過 Stripe Checkout 完成購買後將被重新導向的路由。在此路由中，我們可以擷取 Stripe Checkout 會話 ID 和相關聯的 Stripe Checkout 實例，以便存取我們提供的中繼資料並相應地更新客戶的訂單：

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

請參閱 Stripe 文件，以獲取有關 [Checkout 會話物件所包含資料](https://stripe.com/docs/api/checkout/sessions/object)的更多資訊。

<a name="quickstart-selling-subscriptions"></a>
### 銷售訂閱

> [!NOTE]
> 在使用 Stripe Checkout 之前，您應該在您的 Stripe 控制面板中定義具有固定價格的 Products。此外，您應該 [設定 Cashier 的 webhook 處理](#handling-stripe-webhooks)。

透過您的應用程式提供產品和訂閱計費可能令人望而生畏。然而，多虧了 Cashier 和 [Stripe Checkout](https://stripe.com/payments/checkout)，您可以輕鬆地建構現代、強大的支付整合。

要了解如何使用 Cashier 和 Stripe Checkout 銷售訂閱，讓我們考慮一個簡單的情境：訂閱服務包含基本月付 (`price_basic_monthly`) 和年付 (`price_basic_yearly`) 方案。這兩個價格可以在我們的 Stripe 控制面板中歸類為一個「Basic」產品 (`pro_basic`)。此外，我們的訂閱服務可能提供一個 Expert 方案作為 `pro_expert`。

首先，讓我們了解客戶如何訂閱我們的服務。當然，您可以想像客戶可能會在我們應用程式的定價頁面上點擊 Basic 方案的「訂閱」按鈕。此按鈕或連結應將使用者導向一個 Laravel 路由，該路由會為他們選擇的方案建立 Stripe Checkout 工作階段：

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

如您在上述範例中看到的，我們將客戶重新導向到 Stripe Checkout 工作階段，這將允許他們訂閱我們的 Basic 方案。在成功結帳或取消後，客戶將被重新導向回我們提供給 `checkout` 方法的 URL。為了知道他們的訂閱何時實際開始（因為某些支付方式需要幾秒鐘才能處理），我們還需要 [設定 Cashier 的 webhook 處理](#handling-stripe-webhooks)。

現在客戶可以開始訂閱了，我們需要限制應用程式的某些部分，以便只有訂閱使用者才能存取它們。當然，我們始終可以透過 Cashier 的 `Billable` Trait 提供的 `subscribed` 方法來判斷使用者的當前訂閱狀態：

```blade
@if ($user->subscribed())
    <p>You are subscribed.</p>
@endif
```

我們甚至可以輕鬆判斷使用者是否訂閱了特定的產品或價格：

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

為了方便起見，您可能希望建立一個 [中介層](/docs/{{version}}/middleware)，它會判斷傳入的請求是否來自訂閱使用者。一旦定義了這個中介層，您就可以輕鬆地將其分配給路由，以防止未訂閱的使用者存取該路由：

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
            return redirect('/billing');
        }

        return $next($request);
    }
}
```

一旦定義了中介層，您可以將其分配給路由：

```php
use App\Http\Middleware\Subscribed;

Route::get('/dashboard', function () {
    // ...
})->middleware([Subscribed::class]);
```


<a name="quickstart-allowing-customers-to-manage-their-billing-plan"></a>
#### 允許客戶管理他們的帳務方案

當然，客戶可能希望將他們的訂閱方案變更為其他產品或「級別」。最簡單的方法是引導客戶到 Stripe 的 [Customer Billing Portal](https://stripe.com/docs/no-code/customer-portal)，它提供了一個託管的使用者介面，允許客戶下載發票、更新他們的支付方式以及變更訂閱方案。

首先，在您的應用程式中定義一個連結或按鈕，將使用者引導至一個 Laravel 路由，我們將利用該路由來啟動一個 Billing Portal 工作階段：

```blade
<a href="{{ route('billing') }}">
    Billing
</a>
```

接下來，讓我們定義一個路由，它將啟動一個 Stripe Customer Billing Portal 工作階段並將使用者重新導向到 Portal。`redirectToBillingPortal` 方法接受使用者在退出 Portal 時應返回的 URL：

```php
use Illuminate\Http\Request;

Route::get('/billing', function (Request $request) {
    return $request->user()->redirectToBillingPortal(route('dashboard'));
})->middleware(['auth'])->name('billing');
```

> [!NOTE]
> 只要您已設定 Cashier 的 webhook 處理，Cashier 將透過檢查來自 Stripe 的傳入 webhooks，自動保持您應用程式中與 Cashier 相關的資料庫表格同步。因此，舉例來說，當使用者透過 Stripe 的 Customer Billing Portal 取消他們的訂閱時，Cashier 將會收到相應的 webhook 並在您應用程式的資料庫中將訂閱標記為「已取消」。

<a name="customers"></a>
## 客戶

<a name="retrieving-customers"></a>
### 擷取客戶

您可以使用 `Cashier::findBillable` 方法透過客戶的 Stripe ID 擷取客戶。此方法將會回傳一個可計費模型 (billable model) 的實例：

```php
use Laravel\Cashier\Cashier;

$user = Cashier::findBillable($stripeId);
```

<a name="creating-customers"></a>
### 建立客戶

有時，您可能希望在不開始訂閱的情況下建立一個 Stripe 客戶。您可以使用 `createAsStripeCustomer` 方法來完成此操作：

```php
$stripeCustomer = $user->createAsStripeCustomer();
```

一旦客戶在 Stripe 中建立完成，您就可以在之後開始訂閱。您可以提供一個選用的 `$options` 陣列，以傳遞任何額外 [Stripe API 支援的客戶建立參數](https://stripe.com/docs/api/customers/create)：

```php
$stripeCustomer = $user->createAsStripeCustomer($options);
```

如果您想回傳可計費模型 (billable model) 的 Stripe 客戶物件，可以使用 `asStripeCustomer` 方法：

```php
$stripeCustomer = $user->asStripeCustomer();
```

如果您想擷取給定可計費模型 (billable model) 的 Stripe 客戶物件，但不確定該可計費模型是否已是 Stripe 中的客戶，則可以使用 `createOrGetStripeCustomer` 方法。如果 Stripe 中尚不存在客戶，此方法將會建立一個新客戶：

```php
$stripeCustomer = $user->createOrGetStripeCustomer();
```

<a name="updating-customers"></a>
### 更新客戶

有時，您可能希望直接使用額外資訊更新 Stripe 客戶。您可以使用 `updateStripeCustomer` 方法來完成此操作。此方法接受一個陣列，其中包含 [Stripe API 支援的客戶更新選項](https://stripe.com/docs/api/customers/update)：

```php
$stripeCustomer = $user->updateStripeCustomer($options);
```

<a name="balances"></a>
### 餘額

Stripe 允許您為客戶的「餘額」進行入帳或扣款。之後，此餘額將會用於新發票的入帳或扣款。要檢查客戶的總餘額，您可以使用可計費模型 (billable model) 上可用的 `balance` 方法。`balance` 方法將會回傳客戶貨幣中格式化後的餘額字串：

```php
$balance = $user->balance();
```

要為客戶餘額入帳，您可以向 `creditBalance` 方法提供一個值。如果需要，您也可以提供描述：

```php
$user->creditBalance(500, 'Premium customer top-up.');
```

向 `debitBalance` 方法提供一個值將會從客戶餘額中扣款：

```php
$user->debitBalance(300, 'Bad usage penalty.');
```

`applyBalance` 方法將會為客戶建立新的客戶餘額交易。您可以使用 `balanceTransactions` 方法來擷取這些交易記錄，這對於提供客戶查看的入帳和扣款日誌可能很有用：

```php
// Retrieve all transactions...
$transactions = $user->balanceTransactions();

foreach ($transactions as $transaction) {
    // Transaction amount...
    $amount = $transaction->amount(); // $2.31

    // Retrieve the related invoice when available...
    $invoice = $transaction->invoice();
}
```

<a name="tax-ids"></a>
### 稅號

Cashier 提供了一種簡單的方式來管理客戶的稅號。例如，`taxIds` 方法可用於擷取分配給客戶的所有 [稅號](https://stripe.com/docs/api/customer_tax_ids/object) 作為一個集合：

```php
$taxIds = $user->taxIds();
```

您也可以透過其識別碼擷取客戶的特定稅號：

```php
$taxId = $user->findTaxId('txi_belgium');
```

您可以透過向 `createTaxId` 方法提供有效的 [類型](https://stripe.com/docs/api/customer_tax_ids/object#tax_id_object-type) 和值來建立新的稅號：

```php
$taxId = $user->createTaxId('eu_vat', 'BE0123456789');
```

`createTaxId` 方法將立即將 VAT ID 新增到客戶帳戶中。[VAT ID 的驗證也由 Stripe 完成](https://stripe.com/docs/invoicing/customer/tax-ids#validation)；但是，這是一個非同步過程。您可以透過訂閱 `customer.tax_id.updated` webhook 事件並檢查 [VAT ID 的 `verification` 參數](https://stripe.com/docs/api/customer_tax_ids/object#tax_id_object-verification) 來接收驗證更新通知。有關處理 webhook 的更多資訊，請參閱 [定義 webhook 處理器](#handling-stripe-webhooks) 的文件。

您可以使用 `deleteTaxId` 方法刪除稅號：

```php
$user->deleteTaxId('txi_belgium');
```

<a name="syncing-customer-data-with-stripe"></a>
### 與 Stripe 同步客戶資料

通常，當您的應用程式使用者更新其姓名、電子郵件地址或 Stripe 也儲存的其他資訊時，您應該通知 Stripe 這些更新。透過這樣做，Stripe 上的資訊副本將會與您的應用程式保持同步。

為了實現自動化，您可以在可計費模型 (billable model) 上定義一個事件監聽器，以回應模型的 `updated` 事件。然後，在您的事件監聽器中，您可以呼叫模型上的 `syncStripeCustomerDetails` 方法：

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

現在，每次您的客戶模型更新時，其資訊都將與 Stripe 同步。為方便起見，Cashier 將在客戶初次建立時自動將客戶資訊與 Stripe 同步。

您可以透過覆寫 Cashier 提供的各種方法，來自訂用於將客戶資訊同步到 Stripe 的欄位。例如，您可以覆寫 `stripeName` 方法，來自訂當 Cashier 將客戶資訊同步到 Stripe 時應被視為客戶「姓名」的屬性：

```php
/**
 * Get the customer name that should be synced to Stripe.
 */
public function stripeName(): string|null
{
    return $this->company_name;
}
```

同樣地，您可以覆寫 `stripeEmail`、`stripePhone`、`stripeAddress` 和 `stripePreferredLocales` 方法。這些方法在 [更新 Stripe 客戶物件](https://stripe.com/docs/api/customers/update) 時，會將資訊同步到其相對應的客戶參數。如果您希望完全控制客戶資訊同步過程，您可以覆寫 `syncStripeCustomerDetails` 方法。

<a name="billing-portal"></a>
### 帳務入口

Stripe 提供 [一種簡單的帳務入口設定方式](https://stripe.com/docs/billing/subscriptions/customer-portal)，讓您的客戶可以管理他們的訂閱、支付方式和查看他們的帳單歷史記錄。您可以透過在控制器或路由中呼叫可計費模型 (billable model) 上的 `redirectToBillingPortal` 方法，將您的使用者重新導向至帳務入口：

```php
use Illuminate\Http\Request;

Route::get('/billing-portal', function (Request $request) {
    return $request->user()->redirectToBillingPortal();
});
```

預設情況下，當使用者完成訂閱管理時，他們將能夠透過 Stripe 帳務入口中的連結返回您的應用程式的 `home` 路由位置。您可以透過將 URL 作為引數傳遞給 `redirectToBillingPortal` 方法，來提供使用者應返回的自訂 URL：

```php
use Illuminate\Http\Request;

Route::get('/billing-portal', function (Request $request) {
    return $request->user()->redirectToBillingPortal(route('billing'));
});
```

如果您想生成帳務入口的 URL，而不需要生成 HTTP 重新導向回應，您可以呼叫 `billingPortalUrl` 方法：

```php
$url = $request->user()->billingPortalUrl(route('billing'));
```

<a name="payment-methods"></a>
## 支付方式


<a name="storing-payment-methods"></a>
### 儲存支付方式

為了建立訂閱或執行對 Stripe 的「一次性」扣款，您需要儲存支付方式並從 Stripe 擷取其識別碼。完成此操作的方法會根據您打算將支付方式用於訂閱還是單次扣款而有所不同，因此我們將在下方探討這兩種情況。


<a name="payment-methods-for-subscriptions"></a>
#### 訂閱的支付方式

當儲存客戶的信用卡資訊以供訂閱將來使用時，必須使用 Stripe 的「Setup Intents」API 來安全地收集客戶的支付方式詳細資訊。「Setup Intent」向 Stripe 表明了向客戶支付方式扣款的意圖。Cashier 的 `Billable` trait 包含了 `createSetupIntent` 方法，可輕鬆建立新的 Setup Intent。您應該從將呈現收集客戶支付方式詳細資訊表單的路由或控制器中呼叫此方法：

```php
return view('update-payment-method', [
    'intent' => $user->createSetupIntent()
]);
```

建立 Setup Intent 並將其傳遞給視圖後，您應該將其密鑰附加到將收集支付方式的元素上。例如，考慮這個「更新支付方式」表單：

```html
<input id="card-holder-name" type="text">

<!-- Stripe Elements Placeholder -->
<div id="card-element"></div>

<button id="card-button" data-secret="{{ $intent->client_secret }}">
    Update Payment Method
</button>
```

接著，可以使用 Stripe.js 函式庫將一個 [Stripe Element](https://stripe.com/docs/stripe-js) 附加到表單，並安全地收集客戶的支付詳細資訊：

```html
<script src="https://js.stripe.com/v3/"></script>

<script>
    const stripe = Stripe('stripe-public-key');

    const elements = stripe.elements();
    const cardElement = elements.create('card');

    cardElement.mount('#card-element');
</script>
```

接著，可以使用 [Stripe 的 `confirmCardSetup` 方法](https://stripe.com/docs/js/setup_intents/confirm_card_setup) 來驗證卡片並從 Stripe 擷取安全的「支付方式識別碼」：

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
        // Display "error.message" to the user...
    } else {
        // The card has been verified successfully...
    }
});
```

卡片經 Stripe 驗證成功後，您可以將產生的 `setupIntent.payment_method` 識別碼傳遞給您的 Laravel 應用程式，並將其附加到客戶。該支付方式可以 [新增為新的支付方式](#adding-payment-methods)，或 [用於更新預設支付方式](#updating-the-default-payment-method)。您也可以立即使用該支付方式識別碼來 [建立新訂閱](#creating-subscriptions)。

> [!NOTE]
> 如果您想了解更多關於 Setup Intents 和收集客戶支付詳細資訊的資訊，請 [參閱 Stripe 提供的此總覽](https://stripe.com/docs/payments/save-and-reuse#php)。


<a name="payment-methods-for-single-charges"></a>
#### 單次扣款的支付方式

當然，當對客戶的支付方式進行單次扣款時，我們只需使用支付方式識別碼一次。由於 Stripe 的限制，您不能使用客戶儲存的預設支付方式進行單次扣款。您必須允許客戶使用 Stripe.js 函式庫輸入其支付方式詳細資訊。例如，考慮以下表單：

```html
<input id="card-holder-name" type="text">

<!-- Stripe Elements Placeholder -->
<div id="card-element"></div>

<button id="card-button">
    Process Payment
</button>
```

定義此類表單後，可以使用 Stripe.js 函式庫將一個 [Stripe Element](https://stripe.com/docs/stripe-js) 附加到表單，並安全地收集客戶的支付詳細資訊：

```html
<script src="https://js.stripe.com/v3/"></script>

<script>
    const stripe = Stripe('stripe-public-key');

    const elements = stripe.elements();
    const cardElement = elements.create('card');

    cardElement.mount('#card-element');
</script>
```

接著，可以使用 [Stripe 的 `createPaymentMethod` 方法](https://stripe.com/docs/stripe-js/reference#stripe-create-payment-method) 來驗證卡片並從 Stripe 擷取安全的「支付方式識別碼」：

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
        // Display "error.message" to the user...
    } else {
        // The card has been verified successfully...
    }
});
```

如果卡片驗證成功，您可以將 `paymentMethod.id` 傳遞給您的 Laravel 應用程式並處理 [單次扣款](#simple-charge)。


<a name="retrieving-payment-methods"></a>
### 擷取支付方式

可計費模型實例上的 `paymentMethods` 方法會回傳一個 `Laravel\Cashier\PaymentMethod` 實例的集合：

```php
$paymentMethods = $user->paymentMethods();
```

預設情況下，此方法將回傳所有類型的支付方式。要擷取特定類型的支付方式，您可以將 `type` 作為引數傳遞給該方法：

```php
$paymentMethods = $user->paymentMethods('sepa_debit');
```

要擷取客戶的預設支付方式，可以使用 `defaultPaymentMethod` 方法：

```php
$paymentMethod = $user->defaultPaymentMethod();
```

您可以使用 `findPaymentMethod` 方法擷取附加到可計費模型的特定支付方式：

```php
$paymentMethod = $user->findPaymentMethod($paymentMethodId);
```


<a name="payment-method-presence"></a>
### 支付方式存在性

要判斷可計費模型是否在其帳戶中附加了預設支付方式，請呼叫 `hasDefaultPaymentMethod` 方法：

```php
if ($user->hasDefaultPaymentMethod()) {
    // ...
}
```

您可以使用 `hasPaymentMethod` 方法來判斷可計費模型是否在其帳戶中至少附加了一種支付方式：

```php
if ($user->hasPaymentMethod()) {
    // ...
}
```

此方法將判斷可計費模型是否擁有任何支付方式。要判斷模型是否存在特定類型的支付方式，您可以將 `type` 作為引數傳遞給該方法：

```php
if ($user->hasPaymentMethod('sepa_debit')) {
    // ...
}
```


<a name="updating-the-default-payment-method"></a>
### 更新預設支付方式

`updateDefaultPaymentMethod` 方法可用於更新客戶的預設支付方式資訊。此方法接受 Stripe 支付方式識別碼，並將新的支付方式指定為預設帳單支付方式：

```php
$user->updateDefaultPaymentMethod($paymentMethod);
```

要將您的預設支付方式資訊與客戶在 Stripe 中的預設支付方式資訊同步，您可以使用 `updateDefaultPaymentMethodFromStripe` 方法：

```php
$user->updateDefaultPaymentMethodFromStripe();
```

> [!WARNING]
> 客戶的預設支付方式僅能用於開立發票和建立新訂閱。由於 Stripe 的限制，它不能用於單次扣款。

<a name="adding-payment-methods"></a>
### 新增支付方式

若要新增支付方式，您可以在可計費模型上呼叫 `addPaymentMethod` 方法，並傳入支付方式識別碼：

```php
$user->addPaymentMethod($paymentMethod);
```

> [!NOTE]
> 若要了解如何擷取支付方式識別碼，請參閱[支付方式儲存文件](#storing-payment-methods)。


<a name="deleting-payment-methods"></a>
### 刪除支付方式

若要刪除支付方式，您可以在想要刪除的 `Laravel\Cashier\PaymentMethod` 實例上呼叫 `delete` 方法：

```php
$paymentMethod->delete();
```

`deletePaymentMethod` 方法將從可計費模型中刪除指定的支付方式：

```php
$user->deletePaymentMethod('pm_visa');
```

`deletePaymentMethods` 方法將刪除可計費模型的所有支付方式資訊：

```php
$user->deletePaymentMethods();
```

依預設，此方法將刪除所有類型的支付方式。若要刪除特定類型的支付方式，您可以將 `type` 作為參數傳遞給該方法：

```php
$user->deletePaymentMethods('sepa_debit');
```

> [!WARNING]
> 如果使用者有活躍訂閱，您的應用程式不應允許他們刪除其預設支付方式。

<a name="subscriptions"></a>
## 訂閱

訂閱提供了一種為客戶設定定期付款的方式。由 Cashier 管理的 Stripe 訂閱支援多種訂閱價格、訂閱數量、試用期等功能。


<a name="creating-subscriptions"></a>
### 建立訂閱

要建立訂閱，首先擷取您的可計費模型實例，通常會是 `App\Models\User` 的實例。一旦您擷取了模型實例，您可以使用 `newSubscription` 方法來建立模型的訂閱：

```php
use Illuminate\Http\Request;

Route::post('/user/subscribe', function (Request $request) {
    $request->user()->newSubscription(
        'default', 'price_monthly'
    )->create($request->paymentMethodId);

    // ...
});
```

傳遞給 `newSubscription` 方法的第一個引數應該是訂閱的內部類型。如果您的應用程式只提供單一訂閱，您可以將其稱為 `default` 或 `primary`。此訂閱類型僅用於應用程式內部使用，不應向使用者顯示。此外，它不應包含空格，並且在建立訂閱後不應更改。第二個引數是使用者訂閱的特定價格。此值應與 Stripe 中的價格識別符相對應。

`create` 方法接受 [Stripe 支付方式識別符](#storing-payment-methods)或 Stripe `PaymentMethod` 物件，它將啟動訂閱並更新您的資料庫，其中包含可計費模型的 Stripe 客戶 ID 和其他相關帳務資訊。

> [!WARNING]
> 直接將支付方式識別符傳遞給 `create` 訂閱方法，也會自動將其新增到使用者的已儲存支付方式中。


<a name="collecting-recurring-payments-via-invoice-emails"></a>
#### 透過發票電子郵件收取定期付款

除了自動收取客戶的定期付款外，您還可以指示 Stripe 在每次定期付款到期時，透過電子郵件向客戶寄送發票。然後，客戶可以在收到發票後手動支付。在透過發票收取定期付款時，客戶無需預先提供支付方式：

```php
$user->newSubscription('default', 'price_monthly')->createAndSendInvoice();
```

客戶在訂閱被取消之前支付發票的時間量由 `days_until_due` 選項決定。預設為 30 天；但是，如果您願意，可以為此選項提供一個特定的值：

```php
$user->newSubscription('default', 'price_monthly')->createAndSendInvoice([], [
    'days_until_due' => 30
]);
```


<a name="subscription-quantities"></a>
#### 數量

如果您想在建立訂閱時為價格設定特定 [數量](https://stripe.com/docs/billing/subscriptions/quantities)，您應該在建立訂閱之前，在訂閱建構器上呼叫 `quantity` 方法：

```php
$user->newSubscription('default', 'price_monthly')
    ->quantity(5)
    ->create($paymentMethod);
```


<a name="additional-details"></a>
#### 額外細節

如果您想指定 Stripe 支援的額外 [客戶](https://stripe.com/docs/api/customers/create)或 [訂閱](https://stripe.com/docs/api/subscriptions/create) 選項，您可以將它們作為 `create` 方法的第二個和第三個引數傳遞：

```php
$user->newSubscription('default', 'price_monthly')->create($paymentMethod, [
    'email' => $email,
], [
    'metadata' => ['note' => 'Some extra information.'],
]);
```


<a name="coupons"></a>
#### 優惠券

如果您想在建立訂閱時套用優惠券，您可以使用 `withCoupon` 方法：

```php
$user->newSubscription('default', 'price_monthly')
    ->withCoupon('code')
    ->create($paymentMethod);
```

或者，如果您想套用 [Stripe 促銷碼](https://stripe.com/docs/billing/subscriptions/discounts/codes)，您可以使用 `withPromotionCode` 方法：

```php
$user->newSubscription('default', 'price_monthly')
    ->withPromotionCode('promo_code_id')
    ->create($paymentMethod);
```

提供的促銷碼 ID 應該是分配給該促銷碼的 Stripe API ID，而不是客戶可見的促銷碼。如果您需要根據客戶可見的促銷碼來尋找促銷碼 ID，您可以使用 `findPromotionCode` 方法：

```php
// Find a promotion code ID by its customer facing code...
$promotionCode = $user->findPromotionCode('SUMMERSALE');

// Find an active promotion code ID by its customer facing code...
$promotionCode = $user->findActivePromotionCode('SUMMERSALE');
```

在上面的範例中，返回的 `$promotionCode` 物件是 `Laravel\Cashier\PromotionCode` 的實例。此類別封裝了一個底層的 `Stripe\PromotionCode` 物件。您可以透過呼叫 `coupon` 方法來擷取與該促銷碼相關的優惠券：

```php
$coupon = $user->findPromotionCode('SUMMERSALE')->coupon();
```

優惠券實例允許您確定折扣金額，以及優惠券是固定折扣還是百分比折扣：

```php
if ($coupon->isPercentage()) {
    return $coupon->percentOff().'%'; // 21.5%
} else {
    return $coupon->amountOff(); // $5.99
}
```

您還可以擷取當前套用至客戶或訂閱的折扣：

```php
$discount = $billable->discount();

$discount = $subscription->discount();
```

返回的 `Laravel\Cashier\Discount` 實例會封裝一個底層的 `Stripe\Discount` 物件實例。您可以透過呼叫 `coupon` 方法來擷取與此折扣相關的優惠券：

```php
$coupon = $subscription->discount()->coupon();
```

如果您想向客戶或訂閱套用新的優惠券或促銷碼，您可以透過 `applyCoupon` 或 `applyPromotionCode` 方法執行此操作：

```php
$billable->applyCoupon('coupon_id');
$billable->applyPromotionCode('promotion_code_id');

$subscription->applyCoupon('coupon_id');
$subscription->applyPromotionCode('promotion_code_id');
```

請記住，您應該使用分配給促銷碼的 Stripe API ID，而不是客戶可見的促銷碼。在任何給定時間，只能將一個優惠券或促銷碼套用至一個客戶或訂閱。

有關此主題的更多資訊，請參閱 Stripe 關於 [優惠券](https://stripe.com/docs/billing/subscriptions/coupons) 和 [促銷碼](https://stripe.com/docs/billing/subscriptions/coupons/codes) 的文件。


<a name="adding-subscriptions"></a>
#### 新增訂閱

如果您想為已擁有預設支付方式的客戶新增訂閱，您可以在訂閱建構器上呼叫 `add` 方法：

```php
use App\Models\User;

$user = User::find(1);

$user->newSubscription('default', 'price_monthly')->add();
```


<a name="creating-subscriptions-from-the-stripe-dashboard"></a>
#### 從 Stripe 儀表板建立訂閱

您也可以直接從 Stripe 儀表板建立訂閱。這樣做時，Cashier 會同步新新增的訂閱，並將其指派為 `default` 類型。要自訂從儀表板建立的訂閱所指派的訂閱類型，請 [定義 webhook 事件處理器](#defining-webhook-event-handlers)。

此外，您只能透過 Stripe 儀表板建立一種訂閱類型。如果您的應用程式提供多種類型的訂閱，則只能透過 Stripe 儀表板新增一種訂閱類型。

最後，您應該始終確保應用程式為每種類型的訂閱只新增一個活躍訂閱。如果客戶有兩個 `default` 訂閱，即使兩者都將與您的應用程式資料庫同步，Cashier 也只會使用最近新增的訂閱。

<a name="checking-subscription-status"></a>
### 檢查訂閱狀態

一旦客戶訂閱了您的應用程式，您可以使用各種便捷方法輕鬆檢查其訂閱狀態。首先，`subscribed` 方法會回傳 `true`，如果客戶有活躍的訂閱，即使該訂閱目前仍在試用期內。`subscribed` 方法接受訂閱類型作為其第一個參數：

```php
if ($user->subscribed('default')) {
    // ...
}
```

`subscribed` 方法也是 [路由中介層](/docs/{{version}}/middleware) 的絕佳候選，它允許您根據使用者的訂閱狀態篩選路由和控制器存取。

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
            // This user is not a paying customer...
            return redirect('/billing');
        }

        return $next($request);
    }
}
```

如果您想判斷使用者是否仍在試用期內，可以使用 `onTrial` 方法。此方法對於判斷您是否應該向使用者顯示仍在試用期內的警告很有用：

```php
if ($user->subscription('default')->onTrial()) {
    // ...
}
```

`subscribedToProduct` 方法可用於判斷使用者是否訂閱了某個產品，該判斷基於指定的 Stripe 產品識別碼。在 Stripe 中，產品是價格的集合。在此範例中，我們將判斷使用者的 `default` 訂閱是否活躍地訂閱了應用程式的「premium」產品。指定的 Stripe 產品識別碼應與您在 Stripe 控制面板中的產品識別碼之一相對應：

```php
if ($user->subscribedToProduct('prod_premium', 'default')) {
    // ...
}
```

透過向 `subscribedToProduct` 方法傳遞陣列，您可以判斷使用者的 `default` 訂閱是否活躍地訂閱了應用程式的「basic」或「premium」產品：

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
> If a user has two subscriptions with the same type, the most recent subscription will always be returned by the `subscription` method. For example, a user might have two subscription records with the type of `default`; however, one of the subscriptions may be an old, expired subscription, while the other is the current, active subscription. The most recent subscription will always be returned while older subscriptions are kept in the database for historical review.

<a name="cancelled-subscription-status"></a>
#### 已取消訂閱狀態

為了判斷使用者是否曾經是活躍訂閱者但已取消訂閱，您可以使用 `canceled` 方法：

```php
if ($user->subscription('default')->canceled()) {
    // ...
}
```

您也可以判斷使用者是否已取消訂閱，但仍在「寬限期」內，直到訂閱完全到期。例如，如果使用者在 3 月 5 日取消了原定於 3 月 10 日到期的訂閱，那麼使用者將在 3 月 10 日之前處於「寬限期」。請注意，在此期間，`subscribed` 方法仍會回傳 `true`：

```php
if ($user->subscription('default')->onGracePeriod()) {
    // ...
}
```

若要判斷使用者是否已取消訂閱且不再處於「寬限期」內，您可以使用 `ended` 方法：

```php
if ($user->subscription('default')->ended()) {
    // ...
}
```

<a name="incomplete-and-past-due-status"></a>
#### 未完成與逾期狀態

如果訂閱在建立後需要二次支付動作，該訂閱將被標記為 `incomplete` (未完成)。訂閱狀態儲存在 Cashier 的 `subscriptions` 資料庫表格的 `stripe_status` 欄位中。

同樣地，如果在更換價格時需要二次支付動作，訂閱將被標記為 `past_due` (逾期)。當您的訂閱處於這兩種狀態時，在客戶確認支付之前，它將不會是活躍的。判斷訂閱是否有未完成的支付，可以使用可計費模型或訂閱實例上的 `hasIncompletePayment` 方法：

```php
if ($user->hasIncompletePayment('default')) {
    // ...
}

if ($user->subscription('default')->hasIncompletePayment()) {
    // ...
}
```

當訂閱有未完成的支付時，您應該引導使用者前往 Cashier 的支付確認頁面，並傳遞 `latestPayment` 識別碼。您可以使用訂閱實例上可用的 `latestPayment` 方法來擷取此識別碼：

```html
<a href="{{ route('cashier.payment', $subscription->latestPayment()->id) }}">
    Please confirm your payment.
</a>
```

如果您希望訂閱在處於 `past_due` 或 `incomplete` 狀態時仍被視為活躍，您可以使用 Cashier 提供的 `keepPastDueSubscriptionsActive` 和 `keepIncompleteSubscriptionsActive` 方法。通常，這些方法應該在您的 `App\Providers\AppServiceProvider` 的 `register` 方法中呼叫：

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
> When a subscription is in an `incomplete` state it cannot be changed until the payment is confirmed. Therefore, the `swap` and `updateQuantity` methods will throw an exception when the subscription is in an `incomplete` state.

<a name="subscription-scopes"></a>
#### 訂閱範圍

大多數訂閱狀態也可用作查詢範圍 (query scopes)，以便您可以輕鬆查詢資料庫中處於特定狀態的訂閱：

```php
// Get all active subscriptions...
$subscriptions = Subscription::query()->active()->get();

// Get all of the canceled subscriptions for a user...
$subscriptions = $user->subscriptions()->canceled()->get();
```

以下是所有可用查詢範圍的完整列表：

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

客戶訂閱您的應用程式後，他們可能偶爾會想變更為新的訂閱價格。要將客戶更換到新的價格，請將 Stripe 價格的識別符傳遞給 `swap` 方法。更換價格時，預設使用者希望重新啟用其之前已取消的訂閱。給定的價格識別符應與 Stripe 控制面板中可用的 Stripe 價格識別符相對應：

```php
use App\Models\User;

$user = App\Models\User::find(1);

$user->subscription('default')->swap('price_yearly');
```

如果客戶處於試用期，試用期將會保留。此外，如果訂閱存在「數量」，該數量也將會保留。

如果您想更換價格並取消客戶目前正在進行的任何試用期，可以呼叫 `skipTrial` 方法：

```php
$user->subscription('default')
    ->skipTrial()
    ->swap('price_yearly');
```

如果您想更換價格並立即向客戶開立發票，而不是等待他們下一個計費週期，可以使用 `swapAndInvoice` 方法：

```php
$user = User::find(1);

$user->subscription('default')->swapAndInvoice('price_yearly');
```

<a name="prorations"></a>
#### 按比例收費

依預設，Stripe 會在價格更換時按比例計算費用。`noProrate` 方法可用於更新訂閱價格，而無需按比例計算費用：

```php
$user->subscription('default')->noProrate()->swap('price_yearly');
```

有關訂閱按比例收費的更多資訊，請查閱 [Stripe 文件](https://stripe.com/docs/billing/subscriptions/prorations)。

> [!WARNING]
> 執行 `noProrate` 方法於 `swapAndInvoice` 方法之前對按比例收費將沒有任何影響。發票將會總是開立。

<a name="subscription-quantity"></a>
### 訂閱數量

有時候訂閱會受到「數量」的影響。例如，一個專案管理應用程式可能會按每個專案每月收取 $10。您可以使用 `incrementQuantity` 和 `decrementQuantity` 方法來輕鬆增加或減少您的訂閱數量：

```php
use App\Models\User;

$user = User::find(1);

$user->subscription('default')->incrementQuantity();

// Add five to the subscription's current quantity...
$user->subscription('default')->incrementQuantity(5);

$user->subscription('default')->decrementQuantity();

// Subtract five from the subscription's current quantity...
$user->subscription('default')->decrementQuantity(5);
```

或者，您可以使用 `updateQuantity` 方法設定特定數量：

```php
$user->subscription('default')->updateQuantity(10);
```

`noProrate` 方法可用於更新訂閱數量，而無需按比例計算費用：

```php
$user->subscription('default')->noProrate()->updateQuantity(10);
```

有關訂閱數量的更多資訊，請查閱 [Stripe 文件](https://stripe.com/docs/subscriptions/quantities)。

<a name="quantities-for-subscription-with-multiple-products"></a>
#### 多產品訂閱的數量

如果您的訂閱是 [多產品訂閱](#subscriptions-with-multiple-products)，您應該將要增加或減少數量的價格 ID 作為第二個參數傳遞給增 / 減量方法：

```php
$user->subscription('default')->incrementQuantity(1, 'price_chat');
```

<a name="subscriptions-with-multiple-products"></a>
### 多產品訂閱

[多產品訂閱](https://stripe.com/docs/billing/subscriptions/multiple-products) 允許您將多個計費產品分配給單一訂閱。例如，想像您正在建立一個客戶服務的「客服中心」應用程式，其基本訂閱價格為每月 $10，但額外提供每月 $15 的即時聊天附加產品。多產品訂閱的資訊儲存在 Cashier 的 `subscription_items` 資料庫表格中。

您可以透過將價格陣列作為 `newSubscription` 方法的第二個參數傳遞，為給定的訂閱指定多個產品：

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

在上述範例中，客戶將有兩個價格附加到他們的 `default` 訂閱中。這兩個價格將根據其各自的計費週期進行扣款。如有必要，您可以使用 `quantity` 方法來為每個價格指定特定數量：

```php
$user = User::find(1);

$user->newSubscription('default', ['price_monthly', 'price_chat'])
    ->quantity(5, 'price_chat')
    ->create($paymentMethod);
```

如果您想將另一個價格添加到現有訂閱中，您可以呼叫訂閱的 `addPrice` 方法：

```php
$user = User::find(1);

$user->subscription('default')->addPrice('price_chat');
```

上述範例將添加新價格，客戶將在下一個計費週期內支付費用。如果您想立即向客戶收費，可以使用 `addPriceAndInvoice` 方法：

```php
$user->subscription('default')->addPriceAndInvoice('price_chat');
```

如果您想以特定數量添加價格，您可以將數量作為 `addPrice` 或 `addPriceAndInvoice` 方法的第二個參數傳遞：

```php
$user = User::find(1);

$user->subscription('default')->addPrice('price_chat', 5);
```

您可以使用 `removePrice` 方法從訂閱中移除價格：

```php
$user->subscription('default')->removePrice('price_chat');
```

> [!WARNING]
> 您不能移除訂閱中的最後一個價格。相反地，您應該直接取消訂閱。

<a name="swapping-prices"></a>
#### 變更價格

您也可以更改多產品訂閱中附加的價格。例如，想像一個客戶有一個帶有 `price_chat` 附加產品的 `price_basic` 訂閱，並且您想將客戶從 `price_basic` 升級到 `price_pro` 價格：

```php
use App\Models\User;

$user = User::find(1);

$user->subscription('default')->swap(['price_pro', 'price_chat']);
```

執行上述範例時，底層的 `price_basic` 訂閱項目將被刪除，而 `price_chat` 訂閱項目將被保留。此外，將為 `price_pro` 建立一個新的訂閱項目。

您也可以透過將鍵/值對陣列傳遞給 `swap` 方法來指定訂閱項目選項。例如，您可能需要指定訂閱價格的數量：

```php
$user = User::find(1);

$user->subscription('default')->swap([
    'price_pro' => ['quantity' => 5],
    'price_chat'
]);
```

如果您想變更訂閱中的單一價格，您可以使用訂閱項目本身的 `swap` 方法來進行。如果您想保留訂閱其他價格的所有現有中繼資料，這種方法特別有用：

```php
$user = User::find(1);

$user->subscription('default')
    ->findItemOrFail('price_basic')
    ->swap('price_pro');
```

<a name="proration"></a>
#### 按比例計費

預設情況下，Stripe 在從多產品訂閱中添加或移除價格時會按比例計費。如果您想在不按比例計費的情況下調整價格，您應該在您的價格操作上鏈接 `noProrate` 方法：

```php
$user->subscription('default')->noProrate()->removePrice('price_chat');
```

<a name="swapping-quantities"></a>
#### 數量

如果您想更新個別訂閱價格的數量，您可以使用[現有的數量方法](#subscription-quantity)，並將價格 ID 作為額外參數傳遞給該方法：

```php
$user = User::find(1);

$user->subscription('default')->incrementQuantity(5, 'price_chat');

$user->subscription('default')->decrementQuantity(3, 'price_chat');

$user->subscription('default')->updateQuantity(10, 'price_chat');
```

> [!WARNING]
> 當訂閱具有多個價格時，`Subscription` 模型上的 `stripe_price` 和 `quantity` 屬性將為 `null`。要存取個別價格屬性，您應該使用 `Subscription` 模型上的 `items` 關係。

<a name="subscription-items"></a>
#### 訂閱項目

當訂閱有多個價格時，它將在您資料庫的 `subscription_items` 表中儲存多個訂閱「項目」。您可以透過訂閱上的 `items` 關係來存取這些項目：

```php
use App\Models\User;

$user = User::find(1);

$subscriptionItem = $user->subscription('default')->items->first();

// Retrieve the Stripe price and quantity for a specific item...
$stripePrice = $subscriptionItem->stripe_price;
$quantity = $subscriptionItem->quantity;
```

您也可以使用 `findItemOrFail` 方法擷取特定價格：

```php
$user = User::find(1);

$subscriptionItem = $user->subscription('default')->findItemOrFail('price_chat');
```

<a name="multiple-subscriptions"></a>
### 多重訂閱

Stripe 允許您的客戶同時擁有多個訂閱。例如，您可能經營一家健身房，提供游泳訂閱和舉重訂閱，且每個訂閱可能具有不同的定價。當然，客戶應該能夠訂閱其中一項或兩項方案。

當您的應用程式建立訂閱時，您可以將訂閱類型提供給 `newSubscription` 方法。此類型可以是代表使用者正在啟動的訂閱類型的任何字串：

```php
use Illuminate\Http\Request;

Route::post('/swimming/subscribe', function (Request $request) {
    $request->user()->newSubscription('swimming')
        ->price('price_swimming_monthly')
        ->create($request->paymentMethodId);

    // ...
});
```

在此範例中，我們為客戶啟動了每月游泳訂閱。但是，他們可能希望在稍後時間變更為年度訂閱。當調整客戶的訂閱時，我們只需變更 `swimming` 訂閱上的價格：

```php
$user->subscription('swimming')->swap('price_swimming_yearly');
```

當然，您也可以完全取消訂閱：

```php
$user->subscription('swimming')->cancel();
```

<a name="usage-based-billing"></a>
### 按用量計費

[按用量計費](https://stripe.com/docs/billing/subscriptions/metered-billing) 允許您根據客戶在計費週期內的產品使用量向其收費。例如，您可以根據客戶每月發送的簡訊或電子郵件數量來收費。

要開始使用按用量計費，您首先需要在 Stripe 控制台建立一個新的產品，其中包含一個[按用量計費模型](https://docs.stripe.com/billing/subscriptions/usage-based/implementation-guide)和一個[計量器](https://docs.stripe.com/billing/subscriptions/usage-based/recording-usage#configure-meter)。建立計量器後，請儲存相關聯的事件名稱和計量器 ID，您將需要這些資訊來報告和擷取使用量。然後，使用 `meteredPrice` 方法將按量計費的價格 ID 加入客戶訂閱中：

```php
use Illuminate\Http\Request;

Route::post('/user/subscribe', function (Request $request) {
    $request->user()->newSubscription('default')
        ->meteredPrice('price_metered')
        ->create($request->paymentMethodId);

    // ...
});
```

您也可以透過 [Stripe Checkout](#checkout) 啟用按量計費訂閱：

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
#### 報告使用量

當客戶使用您的應用程式時，您需要向 Stripe 報告他們的使用量，以便他們能準確計費。要報告按量計費事件的使用量，您可以使用 `Billable` 模型上的 `reportMeterEvent` 方法：

```php
$user = User::find(1);

$user->reportMeterEvent('emails-sent');
```

預設情況下，計費週期會新增 1 的「使用量數量」。另外，您可以傳遞特定「用量」以新增至客戶該計費週期的使用量中：

```php
$user = User::find(1);

$user->reportMeterEvent('emails-sent', quantity: 15);
```

要擷取客戶某計量器的事件摘要，您可以使用 `Billable` 實例的 `meterEventSummaries` 方法：

```php
$user = User::find(1);

$meterUsage = $user->meterEventSummaries($meterId);

$meterUsage->first()->aggregated_value // 10
```

有關計量器事件摘要的更多資訊，請參閱 Stripe 的[計量器事件摘要物件文件](https://docs.stripe.com/api/billing/meter-event_summary/object)。

要[列出所有計量器](https://docs.stripe.com/api/billing/meter/list)，您可以使用 `Billable` 實例的 `meters` 方法：

```php
$user = User::find(1);

$user->meters();
```


<a name="subscription-taxes"></a>
### 訂閱稅金

> [!WARNING]
> 您可以使用 [Stripe Tax](#tax-configuration) 自動計算稅金

若要指定用戶在訂閱時支付的稅率，您應該在可計費模型上實作 `taxRates` 方法，並回傳一個包含 Stripe 稅率 ID 的陣列。您可以在 [Stripe 控制台](https://dashboard.stripe.com/test/tax-rates)中定義這些稅率：

```php
/**
 * The tax rates that should apply to the customer's subscriptions.
 *
 * @return array<int, string>
 */
public function taxRates(): array
{
    return ['txr_id'];
}
```

`taxRates` 方法允許您以客戶為基礎應用稅率，這對於跨越多個國家和稅率的用戶群可能會很有幫助。

如果您提供多產品訂閱，您可以透過在可計費模型上實作 `priceTaxRates` 方法，為每個價格定義不同的稅率：

```php
/**
 * The tax rates that should apply to the customer's subscriptions.
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
> `taxRates` 方法僅適用於訂閱費用。如果您使用 Cashier 進行「單次扣款」，您將需要在該次扣款時手動指定稅率。


<a name="syncing-tax-rates"></a>
#### 同步稅率

當您變更 `taxRates` 方法回傳的硬編碼稅率 ID 時，用戶任何現有訂閱的稅務設定將保持不變。如果您希望使用新的 `taxRates` 值更新現有訂閱的稅金，您應該在用戶的訂閱實例上呼叫 `syncTaxRates` 方法：

```php
$user->subscription('default')->syncTaxRates();
```

這也會同步多產品訂閱的任何項目稅率。如果您的應用程式提供多產品訂閱，您應該確保您的可計費模型實作[上方討論過](#subscription-taxes)的 `priceTaxRates` 方法。


<a name="tax-exemption"></a>
#### 免稅

Cashier 也提供 `isNotTaxExempt`、`isTaxExempt` 和 `reverseChargeApplies` 方法來判斷客戶是否免稅。這些方法會呼叫 Stripe API 來判斷客戶的免稅狀態：

```php
use App\Models\User;

$user = User::find(1);

$user->isTaxExempt();
$user->isNotTaxExempt();
$user->reverseChargeApplies();
```

> [!WARNING]
> 這些方法也適用於任何 `Laravel\Cashier\Invoice` 物件。然而，當在 `Invoice` 物件上呼叫時，這些方法將會判斷發票建立時的免稅狀態。


<a name="subscription-anchor-date"></a>
### 訂閱錨定日期

預設情況下，計費週期錨點是訂閱建立日期，如果使用試用期，則是試用期結束日期。如果您想修改計費錨定日期，可以使用 `anchorBillingCycleOn` 方法：

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

有關管理訂閱計費週期的更多資訊，請參閱 [Stripe 計費週期文件](https://stripe.com/docs/billing/subscriptions/billing-cycle)


<a name="cancelling-subscriptions"></a>
### 取消訂閱

要取消訂閱，請在用戶的訂閱上呼叫 `cancel` 方法：

```php
$user->subscription('default')->cancel();
```

當訂閱被取消時，Cashier 會自動設定您 `subscriptions` 資料庫表格中的 `ends_at` 欄位。此欄位用於判斷 `subscribed` 方法何時應該開始回傳 `false`。

例如，如果客戶在 3 月 1 日取消訂閱，但該訂閱原本預定於 3 月 5 日才結束，那麼 `subscribed` 方法會持續回傳 `true` 直到 3 月 5 日。這是因為用戶通常被允許繼續使用應用程式直到其計費週期結束。

您可以使用 `onGracePeriod` 方法來判斷用戶是否已取消訂閱但仍處於「寬限期」：

```php
if ($user->subscription('default')->onGracePeriod()) {
    // ...
}
```

如果您希望立即取消訂閱，請在用戶的訂閱上呼叫 `cancelNow` 方法：

```php
$user->subscription('default')->cancelNow();
```

如果您希望立即取消訂閱並開立任何剩餘未開票的按量計費使用量或新的/待處理的按比例計費發票項目，請在用戶的訂閱上呼叫 `cancelNowAndInvoice` 方法：

```php
$user->subscription('default')->cancelNowAndInvoice();
```

您也可以選擇在特定時間取消訂閱：

```php
$user->subscription('default')->cancelAt(
    now()->addDays(10)
);
```

最後，在刪除相關聯的用戶模型之前，應始終取消用戶訂閱：

```php
$user->subscription('default')->cancelNow();

$user->delete();
```

<a name="resuming-subscriptions"></a>
### 恢復訂閱

如果客戶已取消他們的訂閱，且您希望恢復訂閱，您可以在訂閱上呼叫 `resume` 方法。客戶必須仍在他們的「寬限期」內，才能恢復訂閱：

```php
$user->subscription('default')->resume();
```

如果客戶取消訂閱，然後在訂閱完全到期之前恢復該訂閱，客戶將不會立即被計費。相反地，他們的訂閱將被重新啟用，並且他們將按照原始計費週期被計費。

<a name="subscription-trials"></a>
## 訂閱試用


<a name="with-payment-method-up-front"></a>
### 預先提供支付方式

如果您希望在仍預先收取支付方式資訊的情況下為客戶提供試用期，您應該在建立訂閱時使用 `trialDays` 方法：

```php
use Illuminate\Http\Request;

Route::post('/user/subscribe', function (Request $request) {
    $request->user()->newSubscription('default', 'price_monthly')
        ->trialDays(10)
        ->create($request->paymentMethodId);

    // ...
});
```

此方法會設定資料庫中訂閱記錄的試用期結束日期，並指示 Stripe 在此日期之前不要向客戶收費。使用 `trialDays` 方法時，Cashier 將覆寫在 Stripe 中為價格配置的任何預設試用期。

> [!WARNING]
> 如果客戶的訂閱在試用期結束日期前未取消，他們將在試用期一到期就被收費，因此您務必通知使用者其試用期結束日期。

`trialUntil` 方法允許您提供一個 `DateTime` 實例，以指定試用期應何時結束：

```php
use Illuminate\Support\Carbon;

$user->newSubscription('default', 'price_monthly')
    ->trialUntil(Carbon::now()->addDays(10))
    ->create($paymentMethod);
```

您可以使用使用者實例的 `onTrial` 方法或訂閱實例的 `onTrial` 方法來判斷使用者是否在試用期內。以下兩個範例是等效的：

```php
if ($user->onTrial('default')) {
    // ...
}

if ($user->subscription('default')->onTrial()) {
    // ...
}
```

您可以使用 `endTrial` 方法立即結束訂閱試用：

```php
$user->subscription('default')->endTrial();
```

若要判斷現有試用是否已過期，您可以使用 `hasExpiredTrial` 方法：

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

您可以選擇在 Stripe dashboard 中定義您的價格應獲得多少試用天數，或者始終透過 Cashier 明確地傳遞它們。如果您選擇在 Stripe 中定義您的價格試用天數，您應該注意，新的訂閱，包括客戶過去曾有訂閱的新訂閱，將始終獲得試用期，除非您明確呼叫 `skipTrial()` 方法。


<a name="without-payment-method-up-front"></a>
### 無需預先提供支付方式

如果您希望在不預先收集使用者支付方式資訊的情況下提供試用期，您可以將使用者記錄上的 `trial_ends_at` 欄位設定為您想要的試用期結束日期。這通常在使用者註冊時完成：

```php
use App\Models\User;

$user = User::create([
    // ...
    'trial_ends_at' => now()->addDays(10),
]);
```

> [!WARNING]
> 請務必在您的可計費模型類別定義中為 `trial_ends_at` 屬性新增 [日期轉換](/docs/{{version}}/eloquent-mutators#date-casting)。

Cashier 將此類型的試用稱為「通用試用」，因為它未附加到任何現有訂閱。如果當前日期未超過 `trial_ends_at` 的值，則可計費模型實例上的 `onTrial` 方法將返回 `true`：

```php
if ($user->onTrial()) {
    // User is within their trial period...
}
```

一旦您準備好為使用者建立實際訂閱，您可以像往常一樣使用 `newSubscription` 方法：

```php
$user = User::find(1);

$user->newSubscription('default', 'price_monthly')->create($paymentMethod);
```

若要擷取使用者的試用期結束日期，您可以使用 `trialEndsAt` 方法。如果使用者處於試用期，此方法將返回一個 Carbon 日期實例；如果沒有，則返回 `null`。如果您想獲取特定訂閱（而非預設訂閱）的試用期結束日期，您也可以傳遞一個可選的訂閱類型參數：

```php
if ($user->onTrial()) {
    $trialEndsAt = $user->trialEndsAt('main');
}
```

如果您想明確知道使用者是否處於其「通用」試用期且尚未建立實際訂閱，您也可以使用 `onGenericTrial` 方法：

```php
if ($user->onGenericTrial()) {
    // User is within their "generic" trial period...
}
```


<a name="extending-trials"></a>
### 延長試用

`extendTrial` 方法允許您在訂閱建立後延長訂閱的試用期。如果試用期已經過期且客戶已在為訂閱付費，您仍然可以為他們提供延長試用。試用期內的時間將從客戶的下一個發票中扣除：

```php
use App\Models\User;

$subscription = User::find(1)->subscription('default');

// End the trial 7 days from now...
$subscription->extendTrial(
    now()->addDays(7)
);

// Add an additional 5 days to the trial...
$subscription->extendTrial(
    $subscription->trial_ends_at->addDays(5)
);
```

<a name="handling-stripe-webhooks"></a>
## 處理 Stripe Webhooks

> [!NOTE]
> 您可以使用 [Stripe CLI](https://stripe.com/docs/stripe-cli) 在本地開發期間協助測試 webhooks。

Stripe 可以透過 webhooks 將各種事件通知您的應用程式。預設情況下，一個指向 Cashier webhook 控制器的路由會被 Cashier 服務提供者自動註冊。這個控制器將處理所有傳入的 webhook 請求。

預設情況下，Cashier webhook 控制器會自動處理因過多失敗扣款（依據您的 Stripe 設定）、客戶更新、客戶刪除、訂閱更新和支付方式變更而取消的訂閱；然而，正如我們即將發現的，您可以擴充此控制器來處理您想要的任何 Stripe webhook 事件。

為確保您的應用程式能夠處理 Stripe webhooks，請務必在 Stripe 控制面板中設定 webhook URL。預設情況下，Cashier 的 webhook 控制器回應 `/stripe/webhook` URL 路徑。您應該在 Stripe 控制面板中啟用的所有 webhooks 完整列表如下：

- `customer.subscription.created`
- `customer.subscription.updated`
- `customer.subscription.deleted`
- `customer.updated`
- `customer.deleted`
- `payment_method.automatically_updated`
- `invoice.payment_action_required`
- `invoice.payment_succeeded`

為方便起見，Cashier 包含了 `cashier:webhook` Artisan 指令。此指令將在 Stripe 中建立一個 webhook，監聽所有 Cashier 所需的事件：

```shell
php artisan cashier:webhook
```

預設情況下，建立的 webhook 將指向 `APP_URL` 環境變數定義的 URL，以及 Cashier 中包含的 `cashier.webhook` 路由。如果您想使用不同的 URL，可以在呼叫指令時提供 `--url` 選項：

```shell
php artisan cashier:webhook --url "https://example.com/stripe/webhook"
```

建立的 webhook 將使用您的 Cashier 版本相容的 Stripe API 版本。如果您想使用不同的 Stripe 版本，可以提供 `--api-version` 選項：

```shell
php artisan cashier:webhook --api-version="2019-12-03"
```

建立後，webhook 將立即啟用。如果您希望建立 webhook 但先禁用它直到您準備好，可以在呼叫指令時提供 `--disabled` 選項：

```shell
php artisan cashier:webhook --disabled
```

> [!WARNING]
> 請確保您使用 Cashier 包含的 [webhook 簽章驗證](#verifying-webhook-signatures) 中介層來保護傳入的 Stripe webhook 請求。

<a name="webhooks-csrf-protection"></a>
#### Webhooks 和 CSRF 保護

由於 Stripe webhooks 需要繞過 Laravel 的 [CSRF 保護](/docs/{{version}}/csrf)，您應該確保 Laravel 不會嘗試驗證傳入 Stripe webhooks 的 CSRF 令牌。為此，您應該在應用程式的 `bootstrap/app.php` 檔案中將 `stripe/*` 從 CSRF 保護中排除：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->validateCsrfTokens(except: [
        'stripe/*',
    ]);
})
```

<a name="defining-webhook-event-handlers"></a>
### 定義 Webhook 事件處理器

Cashier 會自動處理因失敗扣款和其他常見 Stripe webhook 事件而取消的訂閱。然而，如果您有其他想處理的 webhook 事件，可以透過監聽 Cashier 分派的以下事件來實現：

- `Laravel\Cashier\Events\WebhookReceived`
- `Laravel\Cashier\Events\WebhookHandled`

這兩個事件都包含 Stripe webhook 的完整負載。例如，如果您希望處理 `invoice.payment_succeeded` webhook，您可以註冊一個 [監聽器](/docs/{{version}}/events#defining-listeners) 來處理該事件：

```php
<?php

namespace App\Listeners;

use Laravel\Cashier\Events\WebhookReceived;

class StripeEventListener
{
    /**
     * Handle received Stripe webhooks.
     */
    public function handle(WebhookReceived $event): void
    {
        if ($event->payload['type'] === 'invoice.payment_succeeded') {
            // Handle the incoming event...
        }
    }
}
```

<a name="verifying-webhook-signatures"></a>
### 驗證 Webhook 簽章

為保護您的 webhooks，您可以使用 [Stripe 的 webhook 簽章](https://stripe.com/docs/webhooks/signatures)。為方便起見，Cashier 自動包含一個中介層，用於驗證傳入的 Stripe webhook 請求是否有效。

要啟用 webhook 驗證，請確保應用程式的 `.env` 檔案中設定了 `STRIPE_WEBHOOK_SECRET` 環境變數。webhook `secret` 可以從您的 Stripe 帳戶控制面板中擷取。

<a name="single-charges"></a>
## 單次扣款


<a name="simple-charge"></a>
### 簡單扣款

如果您想對客戶進行一次性扣款，可以使用可計費模型實例上的 `charge` 方法。您需要將 [支付方式識別碼](#payment-methods-for-single-charges) 作為 `charge` 方法的第二個參數提供：

```php
use Illuminate\Http\Request;

Route::post('/purchase', function (Request $request) {
    $stripeCharge = $request->user()->charge(
        100, $request->paymentMethodId
    );

    // ...
});
```

`charge` 方法接受一個陣列作為其第三個參數，允許您將任何所需的選項傳遞到底層的 Stripe 扣款建立中。有關建立扣款可用選項的更多資訊，請參閱 [Stripe 文件](https://stripe.com/docs/api/charges/create)：

```php
$user->charge(100, $paymentMethod, [
    'custom_option' => $value,
]);
```

您也可以在沒有底層客戶或使用者的情況下使用 `charge` 方法。為此，請在您的應用程式可計費模型的新實例上呼叫 `charge` 方法：

```php
use App\Models\User;

$stripeCharge = (new User)->charge(100, $paymentMethod);
```

如果扣款失敗，`charge` 方法將會拋出例外。如果扣款成功，該方法將返回一個 `Laravel\Cashier\Payment` 實例：

```php
try {
    $payment = $user->charge(100, $paymentMethod);
} catch (Exception $e) {
    // ...
}
```

> [!WARNING]
> `charge` 方法接受的支付金額是您應用程式所使用的貨幣的最小單位。例如，如果客戶以美元支付，金額應以美分指定。


<a name="charge-with-invoice"></a>
### 帶有發票的扣款

有時您可能需要進行一次性扣款並向客戶提供 PDF 發票。`invoicePrice` 方法讓您能夠做到這一點。例如，讓我們向客戶開立五件新襯衫的發票：

```php
$user->invoicePrice('price_tshirt', 5);
```

發票將立即從使用者的預設支付方式中扣款。`invoicePrice` 方法也接受一個陣列作為其第三個參數。這個陣列包含發票項目的計費選項。該方法接受的第四個參數也是一個陣列，其中應包含發票本身的計費選項：

```php
$user->invoicePrice('price_tshirt', 5, [
    'discounts' => [
        ['coupon' => 'SUMMER21SALE']
    ],
], [
    'default_tax_rates' => ['txr_id'],
]);
```

與 `invoicePrice` 類似，您可以使用 `tabPrice` 方法，透過將多個項目（每張發票最多 250 個項目）加入客戶的「帳單」中，然後向客戶開立發票，來建立一次性扣款。例如，我們可以向客戶開立五件襯衫和兩個馬克杯的發票：

```php
$user->tabPrice('price_tshirt', 5);
$user->tabPrice('price_mug', 2);
$user->invoice();
```

或者，您可以使用 `invoiceFor` 方法對客戶的預設支付方式進行「一次性」扣款：

```php
$user->invoiceFor('One Time Fee', 500);
```

儘管 `invoiceFor` 方法可用，但建議您使用 `invoicePrice` 和 `tabPrice` 方法與預先定義的價格。這樣做，您將可以在 Stripe 控制面板中獲得關於您每產品銷售的更好分析和數據。

> [!WARNING]
> `invoice`、`invoicePrice` 和 `invoiceFor` 方法將會建立一個 Stripe 發票，該發票將會重試失敗的計費嘗試。如果您不希望發票重試失敗的扣款，您將需要在第一次失敗扣款後使用 Stripe API 關閉它們。


<a name="creating-payment-intents"></a>
### 建立支付意圖

您可以透過在可計費模型實例上呼叫 `pay` 方法來建立一個新的 Stripe 支付意圖。呼叫此方法將建立一個包裝在 `Laravel\Cashier\Payment` 實例中的支付意圖：

```php
use Illuminate\Http\Request;

Route::post('/pay', function (Request $request) {
    $payment = $request->user()->pay(
        $request->get('amount')
    );

    return $payment->client_secret;
});
```

建立支付意圖後，您可以將客戶端密鑰返回給應用程式的前端，以便使用者在瀏覽器中完成支付。要了解更多關於使用 Stripe 支付意圖建立完整支付流程的資訊，請查閱 [Stripe 文件](https://stripe.com/docs/payments/accept-a-payment?platform=web)。

當使用 `pay` 方法時，Stripe 控制面板中啟用的預設支付方式將可供客戶使用。另外，如果您只想允許使用某些特定的支付方式，可以使用 `payWith` 方法：

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
> `pay` 和 `payWith` 方法接受的支付金額是您應用程式所使用的貨幣的最小單位。例如，如果客戶以美元支付，金額應以美分指定。


<a name="refunding-charges"></a>
### 退款

如果您需要退還 Stripe 扣款，可以使用 `refund` 方法。此方法將 Stripe [支付意圖 ID](#payment-methods-for-single-charges) 作為第一個參數：

```php
$payment = $user->charge(100, $paymentMethodId);

$user->refund($payment->id);
```

<a name="invoices"></a>
## 發票

<a name="retrieving-invoices"></a>
### 擷取發票

您可以透過 `invoices` 方法輕鬆擷取可計費模型 (billable model) 的發票陣列。`invoices` 方法會回傳 `Laravel\Cashier\Invoice` 實例的集合：

```php
$invoices = $user->invoices();
```

如果您想在結果中包含待處理的發票，可以使用 `invoicesIncludingPending` 方法：

```php
$invoices = $user->invoicesIncludingPending();
```

您可以使用 `findInvoice` 方法透過發票 ID 擷取特定發票：

```php
$invoice = $user->findInvoice($invoiceId);
```

<a name="displaying-invoice-information"></a>
#### 顯示發票資訊

列出客戶的發票時，您可以使用發票的方法來顯示相關的發票資訊。例如，您可能希望在表格中列出所有發票，讓使用者可以輕鬆下載任何一份發票：

```blade
<table>
    @foreach ($invoices as $invoice)
        <tr>
            <td>{{ $invoice->date()->toFormattedDateString() }}</td>
            <td>{{ $invoice->total() }}</td>
            <td><a href="/user/invoice/{{ $invoice->id }}">Download</a></td>
        </tr>
    @endforeach
</table>
```

<a name="upcoming-invoices"></a>
### 即將產生的發票

要擷取客戶即將產生的發票，您可以使用 `upcomingInvoice` 方法：

```php
$invoice = $user->upcomingInvoice();
```

同樣地，如果客戶有多重訂閱，您也可以擷取特定訂閱即將產生的發票：

```php
$invoice = $user->subscription('default')->upcomingInvoice();
```

<a name="previewing-subscription-invoices"></a>
### 預覽訂閱發票

使用 `previewInvoice` 方法，您可以在進行價格變更之前預覽發票。這將讓您能夠確定當進行給定的價格變更時，客戶的發票會是什麼樣子：

```php
$invoice = $user->subscription('default')->previewInvoice('price_yearly');
```

您可以傳遞一個價格陣列給 `previewInvoice` 方法，以預覽包含多個新價格的發票：

```php
$invoice = $user->subscription('default')->previewInvoice(['price_yearly', 'price_metered']);
```

<a name="generating-invoice-pdfs"></a>
### 產生發票 PDF

在產生發票 PDF 之前，您應該使用 Composer 安裝 Dompdf 函式庫，它是 Cashier 預設的發票渲染器：

```shell
composer require dompdf/dompdf
```

在路由或控制器中，您可以使用 `downloadInvoice` 方法為給定的發票產生 PDF 下載。此方法會自動產生下載發票所需的適當 HTTP 回應：

```php
use Illuminate\Http\Request;

Route::get('/user/invoice/{invoice}', function (Request $request, string $invoiceId) {
    return $request->user()->downloadInvoice($invoiceId);
});
```

預設情況下，發票上的所有資料都來自儲存在 Stripe 中的客戶和發票資料。檔名基於您的 `app.name` 配置值。但是，您可以透過將一個陣列作為 `downloadInvoice` 方法的第二個參數來客製化部分資料。這個陣列允許您客製化您的公司和產品詳細資訊等資訊：

```php
return $request->user()->downloadInvoice($invoiceId, [
    'vendor' => 'Your Company',
    'product' => 'Your Product',
    'street' => 'Main Str. 1',
    'location' => '2000 Antwerp, Belgium',
    'phone' => '+32 499 00 00 00',
    'email' => 'info@example.com',
    'url' => 'https://example.com',
    'vendorVat' => 'BE123456789',
]);
```

`downloadInvoice` 方法還允許透過其第三個參數指定自訂檔名。此檔名將自動加上 `.pdf` 後綴：

```php
return $request->user()->downloadInvoice($invoiceId, [], 'my-invoice');
```

<a name="custom-invoice-render"></a>
#### 自訂發票渲染器

Cashier 也支援使用自訂發票渲染器。預設情況下，Cashier 使用 `DompdfInvoiceRenderer` 實作，它利用 [dompdf](https://github.com/dompdf/dompdf) PHP 函式庫來產生 Cashier 的發票。但是，您可以透過實作 `Laravel\Cashier\Contracts\InvoiceRenderer` 介面來使用任何您想要的渲染器。例如，您可能希望透過 API 呼叫第三方 PDF 渲染服務來渲染發票 PDF：

```php
use Illuminate\Support\Facades\Http;
use Laravel\Cashier\Contracts\InvoiceRenderer;
use Laravel\Cashier\Invoice;

class ApiInvoiceRenderer implements InvoiceRenderer
{
    /**
     * Render the given invoice and return the raw PDF bytes.
     */
    public function render(Invoice $invoice, array $data = [], array $options = []): string
    {
        $html = $invoice->view($data)->render();

        return Http::get('https://example.com/html-to-pdf', ['html' => $html])->get()->body();
    }
}
```

一旦您實作了發票渲染器合約，您就應該更新應用程式 `config/cashier.php` 配置檔中的 `cashier.invoices.renderer` 配置值。此配置值應該設定為您的自訂渲染器實作的類別名稱。

<a name="checkout"></a>
## 結帳

Cashier Stripe 也提供對 [Stripe Checkout](https://stripe.com/payments/checkout) 的支援。Stripe Checkout 提供了一個預建的託管支付頁面，免去了實作自訂頁面來接受支付的麻煩。

以下文件包含有關如何開始使用 Cashier 搭配 Stripe Checkout 的資訊。要了解更多關於 Stripe Checkout 的資訊，您也應該考慮查閱 [Stripe 關於 Checkout 的文件](https://stripe.com/docs/payments/checkout)。

<a name="product-checkouts"></a>
### 產品結帳

您可以使用可計費模型上的 `checkout` 方法對在 Stripe dashboard 中已建立的現有產品執行結帳。`checkout` 方法將啟動一個新的 Stripe Checkout session。預設情況下，您需要傳遞一個 Stripe Price ID：

```php
use Illuminate\Http\Request;

Route::get('/product-checkout', function (Request $request) {
    return $request->user()->checkout('price_tshirt');
});
```

如有需要，您也可以指定產品數量：

```php
use Illuminate\Http\Request;

Route::get('/product-checkout', function (Request $request) {
    return $request->user()->checkout(['price_tshirt' => 15]);
});
```

當客戶訪問此路由時，他們將被重新導向到 Stripe 的 Checkout 頁面。預設情況下，當使用者成功完成或取消購買時，他們將被重新導向到您應用程式的 `home` 路由位置，但您可以使用 `success_url` 和 `cancel_url` 選項指定自訂回呼 URL：

```php
use Illuminate\Http\Request;

Route::get('/product-checkout', function (Request $request) {
    return $request->user()->checkout(['price_tshirt' => 1], [
        'success_url' => route('your-success-route'),
        'cancel_url' => route('your-cancel-route'),
    ]);
});
```

在定義您的 `success_url` 結帳選項時，您可以指示 Stripe 在呼叫您的 URL 時將結帳 session ID 作為查詢字串參數添加。為此，請將字串 `{CHECKOUT_SESSION_ID}` 添加到您的 `success_url` 查詢字串中。Stripe 將用實際的結帳 session ID 替換此佔位符：

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
#### 促銷碼

預設情況下，Stripe Checkout 不允許 [使用者兌換的促銷碼](https://stripe.com/docs/billing/subscriptions/discounts/codes)。幸運的是，有一種簡單的方法可以為您的 Checkout 頁面啟用這些功能。為此，您可以呼叫 `allowPromotionCodes` 方法：

```php
use Illuminate\Http\Request;

Route::get('/product-checkout', function (Request $request) {
    return $request->user()
        ->allowPromotionCodes()
        ->checkout('price_tshirt');
});
```

<a name="single-charge-checkouts"></a>
### 單次扣款結帳

您也可以對尚未在 Stripe dashboard 中建立的臨時產品執行簡單扣款。為此，您可以使用可計費模型上的 `checkoutCharge` 方法，並傳遞一個可扣款金額、產品名稱和可選數量。當客戶訪問此路由時，他們將被重新導向到 Stripe 的 Checkout 頁面：

```php
use Illuminate\Http\Request;

Route::get('/charge-checkout', function (Request $request) {
    return $request->user()->checkoutCharge(1200, 'T-Shirt', 5);
});
```

> [!WARNING]
> 使用 `checkoutCharge` 方法時，Stripe 將始終在您的 Stripe dashboard 中建立一個新的產品和價格。因此，我們建議您預先在 Stripe dashboard 中建立產品並改用 `checkout` 方法。

<a name="subscription-checkouts"></a>
### 訂閱結帳

> [!WARNING]
> 使用 Stripe Checkout 進行訂閱需要您在 Stripe dashboard 中啟用 `customer.subscription.created` webhook。此 webhook 將在您的資料庫中建立訂閱記錄並儲存所有相關的訂閱項目。

您也可以使用 Stripe Checkout 來啟動訂閱。在使用 Cashier 的訂閱建構器方法定義訂閱後，您可以呼叫 `checkout ` 方法。當客戶訪問此路由時，他們將被重新導向到 Stripe 的 Checkout 頁面：

```php
use Illuminate\Http\Request;

Route::get('/subscription-checkout', function (Request $request) {
    return $request->user()
        ->newSubscription('default', 'price_monthly')
        ->checkout();
});
```

與產品結帳一樣，您可以自訂成功和取消 URL：

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

當然，您也可以為訂閱結帳啟用促銷碼：

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
> 不幸的是，Stripe Checkout 在啟動訂閱時不支援所有訂閱計費選項。在訂閱建構器上使用 `anchorBillingCycleOn` 方法、設定按比例計費行為或設定支付行為在 Stripe Checkout session 期間將無效。請查閱 [Stripe Checkout Session API 文件](https://stripe.com/docs/api/checkout/sessions/create) 以檢閱哪些參數可用。

<a name="stripe-checkout-trial-periods"></a>
#### Stripe Checkout 和試用期

當然，您可以在建構將使用 Stripe Checkout 完成的訂閱時定義一個試用期：

```php
$checkout = Auth::user()->newSubscription('default', 'price_monthly')
    ->trialDays(3)
    ->checkout();
```

但是，試用期必須至少為 48 小時，這是 Stripe Checkout 支援的最短試用時間。

<a name="stripe-checkout-subscriptions-and-webhooks"></a>
#### 訂閱與 Webhooks

請記住，Stripe 和 Cashier 透過 webhooks 更新訂閱狀態，因此客戶在輸入支付資訊後返回應用程式時，訂閱可能尚未啟用。為了處理這種情況，您可能希望顯示一條訊息，告知使用者其支付或訂閱正在處理中。

<a name="collecting-tax-ids"></a>
### 收集稅號

Checkout 也支援收集客戶的稅號。要在結帳 session 上啟用此功能，請在建立 session 時呼叫 `collectTaxIds` 方法：

```php
$checkout = $user->collectTaxIds()->checkout('price_tshirt');
```

呼叫此方法時，客戶將會看到一個新的核取方塊，允許他們表明自己是否以公司身份購買。如果是，他們將有機會提供其稅號。

> [!WARNING]
> 如果您已在應用程式的服務提供者中設定了 [自動稅務收集](#tax-configuration)，則此功能將自動啟用，無需呼叫 `collectTaxIds` 方法。

<a name="guest-checkouts"></a>
### 訪客結帳

使用 `Checkout::guest` 方法，您可以為沒有「帳戶」的應用程式訪客啟動結帳 session：

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

與為現有使用者建立結帳 session 類似，您可以利用 `Laravel\Cashier\CheckoutBuilder` 實例上可用的其他方法來自訂訪客結帳 session：

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

訪客結帳完成後，Stripe 可以派發 `checkout.session.completed` webhook 事件，因此請務必 [設定您的 Stripe webhook](https://dashboard.stripe.com/webhooks) 以實際將此事件傳送到您的應用程式。在 Stripe dashboard 中啟用 webhook 後，您可以 [使用 Cashier 處理 webhook](#handling-stripe-webhooks)。webhook 酬載中包含的物件將是一個 [結帳物件](https://stripe.com/docs/api/checkout/sessions/object)，您可以檢查該物件以完成客戶的訂單。

<a name="guest-checkouts"></a>
### 訪客結帳

使用 `Checkout::guest` 方法，您可以為應用程式中沒有「帳號」的訪客啟動結帳會話：

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

與為現有使用者建立結帳會話類似，您可以利用 `Laravel\Cashier\CheckoutBuilder` 實例上提供的額外方法來客製化訪客結帳會話：

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

訪客結帳完成後，Stripe 可以派送 `checkout.session.completed` webhook 事件，因此請務必[設定您的 Stripe webhook](https://dashboard.stripe.com/webhooks) 以實際將此事件發送到您的應用程式。一旦在 Stripe 管理後台啟用該 webhook，您就可以[使用 Cashier 處理該 webhook](#handling-stripe-webhooks)。webhook 酬載中包含的物件將是一個[結帳物件](https://stripe.com/docs/api/checkout/sessions/object)，您可以檢查該物件以履行客戶的訂單。

<a name="handling-failed-payments"></a>
## 處理失敗的支付

有時，訂閱或單次扣款可能會失敗。當這種情況發生時，Cashier 會拋出一個 `Laravel\Cashier\Exceptions\IncompletePayment` 例外，通知您發生了此情況。捕獲此例外後，您有兩種處理方式。

首先，您可以將客戶重定向到 Cashier 包含的專用支付確認頁面。此頁面已經有一個相關的具名路由，透過 Cashier 的服務提供者自動註冊。因此，您可以捕獲 `IncompletePayment` 例外並將使用者重定向到支付確認頁面：

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

在支付確認頁面，客戶將被提示再次輸入其信用卡資訊並執行 Stripe 要求的任何額外操作，例如「3D Secure」確認。確認支付後，使用者將被重定向到上面指定的 `redirect` 參數提供的 URL。重定向時，URL 將新增 `message` (字串) 和 `success` (整數) 查詢字串變數。支付頁面目前支援以下支付方式類型：

<div class="content-list" markdown="1">

-   信用卡
-   支付寶
-   Bancontact
-   BECS Direct Debit
-   EPS
-   Giropay
-   iDEAL
-   SEPA Direct Debit

</div>

或者，您可以讓 Stripe 為您處理支付確認。在這種情況下，您無需重定向到支付確認頁面，而可以在您的 Stripe 儀表板中[設定 Stripe 的自動帳單電子郵件](https://dashboard.stripe.com/account/billing/automatic)。但是，如果捕獲到 `IncompletePayment` 例外，您仍應通知使用者他們將收到一封包含進一步支付確認說明的電子郵件。

支付例外可能會在使用了 `Billable` trait 的模型上的以下方法中拋出：`charge`、`invoiceFor` 和 `invoice`。在與訂閱互動時，`SubscriptionBuilder` 上的 `create` 方法以及 `Subscription` 和 `SubscriptionItem` 模型上的 `incrementAndInvoice` 和 `swapAndInvoice` 方法可能會拋出未完成支付例外。

判斷現有訂閱是否具有未完成支付可以透過在可計費模型或訂閱實例上使用 `hasIncompletePayment` 方法來完成：

```php
if ($user->hasIncompletePayment('default')) {
    // ...
}

if ($user->subscription('default')->hasIncompletePayment()) {
    // ...
}
```

您可以透過檢查例外實例上的 `payment` 屬性來推導未完成支付的特定狀態：

```php
use Laravel\Cashier\Exceptions\IncompletePayment;

try {
    $user->charge(1000, 'pm_card_threeDSecure2Required');
} catch (IncompletePayment $exception) {
    // Get the payment intent status...
    $exception->payment->status;

    // Check specific conditions...
    if ($exception->payment->requiresPaymentMethod()) {
        // ...
    } elseif ($exception->payment->requiresConfirmation()) {
        // ...
    }
}
```


<a name="confirming-payments"></a>
### 確認支付

某些支付方式需要額外資料才能確認支付。例如，SEPA 支付方式在支付過程中需要額外的「授權」資料。您可以透過 `withPaymentConfirmationOptions` 方法將此資料提供給 Cashier：

```php
$subscription->withPaymentConfirmationOptions([
    'mandate_data' => '...',
])->swap('price_xxx');
```

您可以查閱 [Stripe API 文件](https://stripe.com/docs/api/payment_intents/confirm)以審查確認支付時接受的所有選項。


<a name="strong-customer-authentication"></a>
## 強客戶驗證 (SCA)

如果您的業務或您的客戶之一位於歐洲，您將需要遵守歐盟的強客戶驗證 (SCA) 法規。這些法規於 2019 年 9 月由歐盟實施，旨在防止支付詐欺。幸運的是，Stripe 和 Cashier 已準備好建構符合 SCA 的應用程式。

> [!WARNING]
> 在開始之前，請查閱 [Stripe 關於 PSD2 和 SCA 的指南](https://stripe.com/guides/strong-customer-authentication)以及他們關於[新 SCA API 的文件](https://stripe.com/docs/strong-customer-authentication)。


<a name="payments-requiring-additional-confirmation"></a>
### 需要額外確認的支付

SCA 法規通常要求額外的驗證才能確認和處理支付。當這種情況發生時，Cashier 會拋出一個 `Laravel\Cashier\Exceptions\IncompletePayment` 例外，通知您需要額外驗證。有關如何處理這些例外的更多資訊可以在[處理失敗支付](#handling-failed-payments)的文件中找到。

Stripe 或 Cashier 呈現的支付確認畫面可能會根據特定銀行或發卡機構的支付流程進行調整，並且可能包括額外的卡片確認、臨時小額扣款、單獨的設備驗證或其他形式的驗證。


<a name="incomplete-and-past-due-state"></a>
#### 未完成與逾期狀態

當支付需要額外確認時，訂閱將保持在 `incomplete` 或 `past_due` 狀態，如其 `stripe_status` 資料庫欄位所示。一旦支付確認完成，並且您的應用程式透過 Stripe 的 webhook 收到完成通知，Cashier 將自動啟用客戶的訂閱。

有關 `incomplete` 和 `past_due` 狀態的更多資訊，請參閱[我們關於這些狀態的額外文件](#incomplete-and-past-due-status)。


<a name="off-session-payment-notifications"></a>
### 離線支付通知

由於 SCA 法規要求客戶即使在訂閱處於活動狀態時也偶爾驗證其支付詳細資訊，因此當需要離線支付確認時，Cashier 可以向客戶發送通知。例如，這可能發生在訂閱續訂時。Cashier 的支付通知可以透過將 `CASHIER_PAYMENT_NOTIFICATION` 環境變數設定為通知類別來啟用。預設情況下，此通知是禁用的。當然，Cashier 包含一個可用於此目的的通知類別，但如果您願意，也可以提供自己的通知類別：

```ini
CASHIER_PAYMENT_NOTIFICATION=Laravel\Cashier\Notifications\ConfirmPayment
```

為確保離線支付確認通知的傳遞，請驗證您的應用程式是否[設定了 Stripe webhooks](#handling-stripe-webhooks)，並且在您的 Stripe 儀表板中啟用了 `invoice.payment_action_required` webhook。此外，您的 `Billable` 模型也應該使用 Laravel 的 `Illuminate\Notifications\Notifiable` trait。

> [!WARNING]
> 即使客戶手動進行需要額外確認的支付，也會發送通知。不幸的是，Stripe 無法知道支付是手動完成還是「離線」完成。但是，如果客戶在確認支付後再次訪問支付頁面，他們只會看到「支付成功」訊息。客戶將不允許意外地兩次確認同一筆支付並產生意外的第二次扣款。

<a name="stripe-sdk"></a>
## Stripe SDK

Cashier 中的許多物件都是 Stripe SDK 物件的封裝。如果您想直接與 Stripe 物件互動，可以使用 `asStripe` 方法方便地擷取它們：

```php
$stripeSubscription = $subscription->asStripeSubscription();

$stripeSubscription->application_fee_percent = 5;

$stripeSubscription->save();
```

您也可以使用 `updateStripeSubscription` 方法直接更新 Stripe subscription：

```php
$subscription->updateStripeSubscription(['application_fee_percent' => 5]);
```

如果您想直接使用 `Stripe\StripeClient` 客戶端，可以呼叫 `Cashier` 類別上的 `stripe` 方法。例如，您可以使用此方法存取 `StripeClient` 實例，並從您的 Stripe 帳號擷取價格列表：

```php
use Laravel\Cashier\Cashier;

$prices = Cashier::stripe()->prices->all();
```

<a name="testing"></a>
## 測試

在測試使用 Cashier 的應用程式時，您可以模擬對 Stripe API 的實際 HTTP 請求；然而，這要求您部分重新實作 Cashier 自身的行為。因此，我們建議讓您的測試實際呼叫 Stripe API。雖然這會比較慢，但能提供更多信心，確保您的應用程式如預期般運作，並且任何緩慢的測試都可以放置在獨立的 Pest / PHPUnit 測試群組中。

進行測試時，請記住 Cashier 本身已經有一個完善的測試套件，因此您應該只專注於測試您應用程式的 subscription 和 payment flow，而不是 Cashier 的所有底層行為。

首先，將您的 Stripe secret **測試**版本新增到 `phpunit.xml` 檔案中：

```xml
<env name="STRIPE_SECRET" value="sk_test_<your-key>"/>
```

現在，每當您在測試時與 Cashier 互動，它將會向您的 Stripe 測試環境發送實際的 API 請求。為方便起見，您應該預先在您的 Stripe 測試帳號中填入您在測試期間可能使用的 subscriptions / prices。

> [!NOTE]
> 為了測試各種計費情境，例如信用卡拒絕和失敗，您可以使用 Stripe 提供的各種[測試信用卡號碼和權杖](https://stripe.com/docs/testing)。