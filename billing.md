# Laravel Cashier (Stripe)

- [簡介](#introduction)
- [升級 Cashier](#upgrading-cashier)
- [安裝](#installation)
- [設定](#configuration)
    - [可計費模型](#billable-model)
    - [API 金鑰](#api-keys)
    - [貨幣設定](#currency-configuration)
    - [稅務設定](#tax-configuration)
    - [日誌](#logging)
    - [使用自定義模型](#using-custom-models)
- [快速入門](#quickstart)
    - [販售產品](#quickstart-selling-products)
    - [販售訂閱](#quickstart-selling-subscriptions)
- [客戶](#customers)
    - [檢索客戶](#retrieving-customers)
    - [建立客戶](#creating-customers)
    - [更新客戶](#updating-customers)
    - [餘額](#balances)
    - [稅務 ID](#tax-ids)
    - [與 Stripe 同步客戶資料](#syncing-customer-data-with-stripe)
    - [帳務中心](#billing-portal)
- [付款方式](#payment-methods)
    - [儲存付款方式](#storing-payment-methods)
    - [檢索付款方式](#retrieving-payment-methods)
    - [付款方式是否存在](#payment-method-presence)
    - [更新預設付款方式](#updating-the-default-payment-method)
    - [新增付款方式](#adding-payment-methods)
    - [刪除付款方式](#deleting-payment-methods)
- [訂閱](#subscriptions)
    - [建立訂閱](#creating-subscriptions)
    - [檢查訂閱狀態](#checking-subscription-status)
    - [變更價格](#changing-prices)
    - [訂閱數量](#subscription-quantity)
    - [多重產品訂閱](#subscriptions-with-multiple-products)
    - [多重訂閱](#multiple-subscriptions)
    - [按量計費](#usage-based-billing)
    - [訂閱稅務](#subscription-taxes)
    - [訂閱錨定日期](#subscription-anchor-date)
    - [取消訂閱](#cancelling-subscriptions)
    - [恢復訂閱](#resuming-subscriptions)
- [訂閱試用](#subscription-trials)
    - [預先收集付款方式](#with-payment-method-up-front)
    - [不預先收集付款方式](#without-payment-method-up-front)
    - [延長試用期](#extending-trials)
- [處理 Stripe Webhooks](#handling-stripe-webhooks)
    - [定義 Webhook 事件處理程式](#defining-webhook-event-handlers)
    - [驗證 Webhook 簽章](#verifying-webhook-signatures)
- [單次收費](#single-charges)
    - [簡單收費](#simple-charge)
    - [含發票的收費](#charge-with-invoice)
    - [建立付款意向 (Payment Intent)](#creating-payment-intents)
    - [退款](#refunding-charges)
- [發票](#invoices)
    - [檢索發票](#retrieving-invoices)
    - [即將產生的發票](#upcoming-invoices)
    - [預覽訂閱發票](#previewing-subscription-invoices)
    - [生成發票 PDF](#generating-invoice-pdfs)
- [Checkout](#checkout)
    - [產品結帳](#product-checkouts)
    - [單次收費結帳](#single-charge-checkouts)
    - [訂閱結帳](#subscription-checkouts)
    - [收集稅務 ID](#collecting-tax-ids)
    - [訪客結帳](#guest-checkouts)
- [處理失敗的付款](#handling-failed-payments)
    - [確認付款](#confirming-payments)
- [強客戶驗證 (SCA)](#strong-customer-authentication)
    - [需要額外確認的付款](#payments-requiring-additional-confirmation)
    - [非工作階段付款通知](#off-session-payment-notifications)
- [Stripe SDK](#stripe-sdk)
- [測試](#testing)

<a name="introduction"></a>
## 簡介

[Laravel Cashier Stripe](https://github.com/laravel/cashier-stripe) 為 [Stripe](https://stripe.com) 的訂閱帳務服務提供了一個表達性強且流暢的介面。它處理了幾乎所有你感到恐懼且繁瑣的訂閱帳務程式碼。除了基本的訂閱管理外，Cashier 還可以處理優惠券 (Coupon)、切換訂閱、訂閱「數量 (Quantities)」、取消寬限期，甚至能生成發票 PDF。


<a name="upgrading-cashier"></a>
## 升級 Cashier

升級到新版本的 Cashier 時，請務必仔細閱讀 [升級指南](https://github.com/laravel/cashier-stripe/blob/16.x/UPGRADE.md)。

> [!WARNING]
> 為了防止破壞性變更，Cashier 使用了固定的 Stripe API 版本。Cashier 16 使用 Stripe API 版本 `2025-06-30.basil`。Stripe API 版本會在次要版本 (Minor Releases) 中更新，以便使用新的 Stripe 功能和改進。


<a name="installation"></a>
## 安裝

首先，使用 Composer 套件管理器安裝適用於 Stripe 的 Cashier 套件：

```shell
composer require laravel/cashier
```

安裝套件後，使用 `vendor:publish` Artisan 指令發佈 Cashier 的遷移檔 (Migrations)：

```shell
php artisan vendor:publish --tag="cashier-migrations"
```

接著，執行資料庫遷移：

```shell
php artisan migrate
```

Cashier 的遷移會為你的 `users` 資料表增加幾個欄位。它們還會建立一個新的 `subscriptions` 資料表來存放你所有客戶的訂閱，以及一個 `subscription_items` 資料表來處理具有多重價格的訂閱。

如果你願意，也可以使用 `vendor:publish` Artisan 指令發佈 Cashier 的設定檔：

```shell
php artisan vendor:publish --tag="cashier-config"
```

最後，為了確保 Cashier 能正確處理所有 Stripe 事件，請記得[設定 Cashier 的 Webhook 處理](#handling-stripe-webhooks)。

> [!WARNING]
> Stripe 建議用於儲存 Stripe 識別碼的任何欄位都應該區分大小寫。因此，當使用 MySQL 時，你應該確保 `stripe_id` 欄位的定序 (Collation) 設置為 `utf8_bin`。更多相關資訊可以在 [Stripe 文件](https://stripe.com/docs/upgrades#what-changes-does-stripe-consider-to-be-backwards-compatible)中找到。


<a name="configuration"></a>
## 設定


<a name="billable-model"></a>
### 可計費模型

在使用 Cashier 之前，請將 `Billable` Trait 加入你的可計費模型定義中。通常這會是 `App\Models\User` 模型。此 Trait 提供了多種方法，讓你可以執行常見的帳務任務，例如建立訂閱、套用優惠券以及更新付款方式資訊：

```php
use Laravel\Cashier\Billable;

class User extends Authenticatable
{
    use Billable;
}
```

Cashier 預設你的可計費模型會是 Laravel 內建的 `App\Models\User` 類別。如果你希望更改此設定，可以透過 `useCustomerModel` 方法指定一個不同的模型。此方法通常應該在 `AppServiceProvider` 類別的 `boot` 方法中呼叫：

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
> 如果你使用的模型不是 Laravel 提供的 `App\Models\User` 模型，你需要發佈並修改提供的 [Cashier 遷移檔](#installation)，以符合你自定義模型的資料表名稱。


<a name="api-keys"></a>
### API 金鑰

接下來，你應該在應用程式的 `.env` 檔案中設定你的 Stripe API 金鑰。你可以從 Stripe 控制面板取得你的 Stripe API 金鑰：

```ini
STRIPE_KEY=your-stripe-key
STRIPE_SECRET=your-stripe-secret
STRIPE_WEBHOOK_SECRET=your-stripe-webhook-secret
```

> [!WARNING]
> 你應該確保在應用程式的 `.env` 檔案中定義了 `STRIPE_WEBHOOK_SECRET` 環境變數，因為此變數用於確保傳入的 Webhook 確實來自 Stripe。


<a name="currency-configuration"></a>
### 貨幣設定

Cashier 預設貨幣為美元 (USD)。你可以透過在應用程式的 `.env` 檔案中設定 `CASHIER_CURRENCY` 環境變數來更改預設貨幣：

```ini
CASHIER_CURRENCY=eur
```

除了設定 Cashier 的貨幣之外，你還可以指定一個語系 (Locale)，用於在發票上格式化顯示金額數值。在內部，Cashier 利用 [PHP 的 `NumberFormatter` 類別](https://www.php.net/manual/en/class.numberformatter.php)來設定貨幣語系：

```ini
CASHIER_CURRENCY_LOCALE=nl_BE
```

> [!WARNING]
> 為了使用 `en` 以外的語系，請確保你的伺服器上已安裝並設定了 `ext-intl` PHP 擴充套件。


<a name="tax-configuration"></a>
### 稅務設定

感謝 [Stripe Tax](https://stripe.com/tax)，現在可以自動計算由 Stripe 生成的所有發票稅金。你可以透過在應用程式 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫 `calculateTaxes` 方法來啟用自動稅金計算：

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

一旦啟用了稅金計算，任何新訂閱以及生成的任何單次發票都將獲得自動稅金計算。

為了使此功能正常運作，你的客戶帳務詳情（例如客戶名稱、地址和稅務 ID）需要同步到 Stripe。你可以使用 Cashier 提供的[客戶資料同步](#syncing-customer-data-with-stripe)和[稅務 ID](#tax-ids) 方法來完成此操作。


<a name="logging"></a>
### 日誌

Cashier 允許你指定在記錄 Stripe 嚴重錯誤時使用的日誌頻道。你可以透過在應用程式的 `.env` 檔案中定義 `CASHIER_LOGGER` 環境變數來指定日誌頻道：

```ini
CASHIER_LOGGER=stack
```

由向 Stripe 發出的 API 呼叫產生的例外狀況將透過應用程式的預設日誌頻道記錄。


<a name="using-custom-models"></a>
### 使用自定義模型

你可以藉由定義自己的模型並擴充對應的 Cashier 模型，來擴充 Cashier 內部使用的模型：

```php
use Laravel\Cashier\Subscription as CashierSubscription;

class Subscription extends CashierSubscription
{
    // ...
}
```

定義好模型後，你可以透過 `Laravel\Cashier\Cashier` 類別指示 Cashier 使用你的自定義模型。通常，你應該在應用程式 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中通知 Cashier 你的自定義模型：

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
### 販售產品

> [!NOTE]
> 在利用 Stripe Checkout 之前，您應該先在 Stripe 控制面板中定義具有固定價格的產品 (Products)。此外，您也應該[設定 Cashier 的 Webhook 處理](#handling-stripe-webhooks)。

透過您的應用程式提供產品和訂閱計費可能令人望而生畏。然而，感謝 Cashier 和 [Stripe Checkout](https://stripe.com/payments/checkout)，您可以輕鬆構建現代、強健的付款整合方案。

為了向客戶收取非週期性、單次收費產品的費用，我們將利用 Cashier 將客戶引導至 Stripe Checkout，在那裡他們將提供付款詳情並確認購買。一旦透過 Checkout 完成付款，客戶將被重新導向到您在應用程式中選擇的成功 URL：

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

如您在上面的範例中所見，我們將利用 Cashier 提供的 `checkout` 方法，針對給定的「價格識別碼」將客戶重新導向到 Stripe Checkout。在使用 Stripe 時，「價格 (prices)」是指[為特定產品定義的價格](https://stripe.com/docs/products-prices/how-products-and-prices-work)。

如有必要，`checkout` 方法將自動在 Stripe 中建立客戶，並將該 Stripe 客戶紀錄連接到應用程式資料庫中對應的使用者。完成結帳工作階段後，客戶將被重新導向到專門的成功或取消頁面，您可以在該頁面向客戶顯示資訊訊息。

<a name="providing-meta-data-to-stripe-checkout"></a>
#### 向 Stripe Checkout 提供中繼資料 (Meta Data)

販售產品時，通常會透過您自己應用程式定義的 `Cart` 和 `Order` 模型來追蹤已完成的訂單和購買的產品。當將客戶重新導向到 Stripe Checkout 以完成購買時，您可能需要提供現有的訂單識別碼，以便在客戶被重新導向回您的應用程式時，將完成的購買與相應的訂單關聯起來。

為了實現這一點，您可以向 `checkout` 方法提供一個 `metadata` 陣列。假設當使用者開始結帳流程時，我們的應用程式內建立了一個待處理的 `Order`。請記住，此範例中的 `Cart` 與 `Order` 模型僅供說明之用，並非由 Cashier 提供。您可以根據自己應用程式的需求自由實現這些概念：

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

如您在上面的範例中所見，當使用者開始結帳流程時，我們將向 `checkout` 方法提供所有與購物車 / 訂單相關聯的 Stripe 價格識別碼。當然，您的應用程式負責在客戶新增商品時將這些項目與「購物車」或訂單關聯。我們還透過 `metadata` 陣列將訂單的 ID 提供給 Stripe Checkout 工作階段。最後，我們在 Checkout 成功路由中加入了 `CHECKOUT_SESSION_ID` 模板變數。當 Stripe 將客戶重新導向回您的應用程式時，此模板變數將自動填充為 Checkout 工作階段 ID。

接下來，讓我們構建 Checkout 成功路由。這是使用者透過 Stripe Checkout 完成購買後將被重新導向到的路由。在此路由中，我們可以檢索 Stripe Checkout 工作階段 ID 和相關的 Stripe Checkout 實例，以便訪問我們提供的中繼資料並相應地更新客戶的訂單：

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

請參閱 Stripe 的文件以獲取有關 [Checkout 工作階段物件所包含資料](https://stripe.com/docs/api/checkout/sessions/object)的更多資訊。

<a name="quickstart-selling-subscriptions"></a>
### 販售訂閱

> [!NOTE]
> 在使用 Stripe Checkout 之前，您應該在 Stripe 儀表板中定義固定價格的產品 (Products)。此外，您還應該[設定 Cashier 的 webhook 處理](#handling-stripe-webhooks)。

透過應用程式提供產品和訂閱計費可能令人望而生畏。然而，多虧了 Cashier 和 [Stripe Checkout](https://stripe.com/payments/checkout)，您可以輕鬆構建現代且強大的付款整合方案。

為了學習如何使用 Cashier 和 Stripe Checkout 販售訂閱，讓我們考慮一個簡單的情境：一個具有基礎月費 (`price_basic_monthly`) 和年費 (`price_basic_yearly`) 方案的訂閱服務。這兩個價格可以在我們的 Stripe 儀表板中歸類在一個 "Basic" 產品 (`pro_basic`) 下。此外，我們的訂閱服務可能還會提供一個專家方案 `pro_expert`。

首先，讓我們來看看客戶如何訂閱我們的服務。當然，您可以想像客戶可能會在我們應用程式的定價頁面上點擊 Basic 方案的「訂閱」按鈕。這個按鈕或連結應該將使用者引導至一個 Laravel 路由，該路由會為他們選擇的方案建立 Stripe Checkout 工作階段 (Session)：

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

如上例所示，我們將客戶重新導向至 Stripe Checkout 工作階段，讓他們訂閱我們的 Basic 方案。在成功結帳或取消後，客戶將被導回我們提供給 `checkout` 方法的 URL。為了知道他們的訂閱何時實際開始（因為某些付款方式需要幾秒鐘的時間處理），我們還需要[設定 Cashier 的 webhook 處理](#handling-stripe-webhooks)。

現在客戶可以開始訂閱了，我們需要限制應用程式的某些部分，以便只有訂閱使用者才能存取。當然，我們始終可以透過 Cashier 的 `Billable` trait 提供的 `subscribed` 方法來判斷使用者的當前訂閱狀態：

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
#### 建立 Subscribed 中介層 (Middleware)

為了方便起見，您可能希望建立一個[中介層 (Middleware)](/docs/{{version}}/middleware)，用以判斷傳入的請求是否來自已訂閱的使用者。一旦定義了此中介層，您就可以輕鬆將其分配給路由，以防止未訂閱的使用者存取該路由：

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

定義好中介層後，您可以將其分配給路由：

```php
use App\Http\Middleware\Subscribed;

Route::get('/dashboard', function () {
    // ...
})->middleware([Subscribed::class]);
```


<a name="quickstart-allowing-customers-to-manage-their-billing-plan"></a>
#### 允許客戶管理其帳務方案

當然，客戶可能想要將他們的訂閱方案更改為另一個產品或「等級」。最簡單的方法是引導客戶前往 Stripe 的 [客戶帳務門戶 (Customer Billing Portal)](https://stripe.com/docs/no-code/customer-portal)，它提供了一個託管的使用者介面，允許客戶下載發票、更新付款方式以及變更訂閱方案。

首先，在您的應用程式中定義一個連結或按鈕，將使用者引導至一個我們將用來啟動帳務門戶工作階段的 Laravel 路由：

```blade
<a href="{{ route('billing') }}">
    Billing
</a>
```

接著，讓我們定義啟動 Stripe 客戶帳務門戶工作階段並將使用者重新導向至該門戶的路由。`redirectToBillingPortal` 方法接受使用者在退出門戶時應返回的 URL：

```php
use Illuminate\Http\Request;

Route::get('/billing', function (Request $request) {
    return $request->user()->redirectToBillingPortal(route('dashboard'));
})->middleware(['auth'])->name('billing');
```

> [!NOTE]
> 只要您設定了 Cashier 的 webhook 處理，Cashier 就會透過檢查來自 Stripe 的傳入 webhook，自動讓您應用程式中與 Cashier 相關的資料庫資料表保持同步。例如，當使用者透過 Stripe 的客戶帳務門戶取消訂閱時，Cashier 將收到對應的 webhook 並在應用程式的資料庫中將訂閱標記為「已取消 (canceled)」。

<a name="customers"></a>
## 客戶


<a name="retrieving-customers"></a>
### 檢索客戶

您可以使用 `Cashier::findBillable` 方法透過 Stripe ID 檢索客戶。此方法將回傳可計費模型的執行個體：

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

一旦在 Stripe 中建立了客戶，您可以在稍後的日期開始訂閱。您可以提供一個選填的 `$options` 陣列，以傳入任何 [Stripe API 支援的額外客戶建立參數](https://stripe.com/docs/api/customers/create)：

```php
$stripeCustomer = $user->createAsStripeCustomer($options);
```

如果您想回傳可計費模型的 Stripe 客戶物件，可以使用 `asStripeCustomer` 方法：

```php
$stripeCustomer = $user->asStripeCustomer();
```

如果您想檢索給定可計費模型的 Stripe 客戶物件，但不確定該可計費模型是否已經是 Stripe 中的客戶，則可以使用 `createOrGetStripeCustomer` 方法。如果客戶尚不存在，此方法將在 Stripe 中建立一個新客戶：

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

Stripe 允許您對客戶的「餘額」進行存入或扣除。之後，此餘額將在新的發票中被折抵或增收。要檢查客戶的總餘額，您可以使用可計費模型上的 `balance` 方法。`balance` 方法將回傳客戶貨幣餘額的格式化字串表示形式：

```php
$balance = $user->balance();
```

要存入客戶餘額，您可以為 `creditBalance` 方法提供一個數值。如果您願意，也可以提供說明：

```php
$user->creditBalance(500, 'Premium customer top-up.');
```

為 `debitBalance` 方法提供一個數值將會扣除客戶的餘額：

```php
$user->debitBalance(300, 'Bad usage penalty.');
```

`applyBalance` 方法將為客戶建立新的客戶餘額交易。您可以使用 `balanceTransactions` 方法檢索這些交易記錄，這對於提供客戶查看存入與扣除的日誌非常有用：

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
### 稅務 ID

Cashier 提供了一種簡單的方法來管理客戶的稅務 ID。例如，`taxIds` 方法可用於以集合形式檢索分配給客戶的所有 [稅務 ID](https://stripe.com/docs/api/customer_tax_ids/object)：

```php
$taxIds = $user->taxIds();
```

您也可以透過其識別碼檢索客戶的特定稅務 ID：

```php
$taxId = $user->findTaxId('txi_belgium');
```

您可以透過向 `createTaxId` 方法提供有效的 [類型 (type)](https://stripe.com/docs/api/customer_tax_ids/object#tax_id_object-type) 和數值來建立新的稅務 ID：

```php
$taxId = $user->createTaxId('eu_vat', 'BE0123456789');
```

`createTaxId` 方法將立即將增值稅 (VAT) ID 新增到客戶的帳戶中。[增值稅 ID 的驗證也由 Stripe 完成](https://stripe.com/docs/invoicing/customer/tax-ids#validation)；然而，這是一個非同步過程。您可以透過訂閱 `customer.tax_id.updated` Webhook 事件並檢查 [增值稅 ID 的 `verification` 參數](https://stripe.com/docs/api/customer_tax_ids/object#tax_id_object-verification) 來獲取驗證更新的通知。有關處理 Webhook 的更多資訊，請參閱 [關於定義 Webhook 處理程式的說明文件](#handling-stripe-webhooks)。

您可以使用 `deleteTaxId` 方法刪除稅務 ID：

```php
$user->deleteTaxId('txi_belgium');
```


<a name="syncing-customer-data-with-stripe"></a>
### 與 Stripe 同步客戶資料

通常，當您的應用程式使用者更新其名稱、電子郵件地址或其他同樣儲存在 Stripe 中的資訊時，您應該將更新通知 Stripe。透過這樣做，Stripe 的資訊副本將與您的應用程式保持同步。

要自動執行此操作，您可以在可計費模型上定義一個事件監聽器，以對模型的 `updated` 事件做出反應。然後，在您的事件監聽器中，您可以呼叫模型上的 `syncStripeCustomerDetails` 方法：

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

現在，每當您的客戶模型更新時，其資訊都會與 Stripe 同步。為了方便起見，Cashier 會在最初建立客戶時自動與 Stripe 同步客戶資訊。

您可以透過覆寫 Cashier 提供的一系列方法來對應要同步到 Stripe 的客戶資訊欄位。例如，您可以覆寫 `stripeName` 方法，以自定義當 Cashier 將客戶資訊同步到 Stripe 時，應被視為客戶「名稱」的屬性：

```php
/**
 * Get the customer name that should be synced to Stripe.
 */
public function stripeName(): string|null
{
    return $this->company_name;
}
```

同樣地，您可以覆寫 `stripeEmail`、`stripePhone`（最多 20 個字元）、`stripeAddress` 和 `stripePreferredLocales` 方法。這些方法將在 [更新 Stripe 客戶物件](https://stripe.com/docs/api/customers/update) 時，將資訊同步到其對應的客戶參數。如果您希望完全控制客戶資訊同步過程，可以覆寫 `syncStripeCustomerDetails` 方法。


<a name="billing-portal"></a>
### 帳務中心

Stripe 提供 [設定帳務中心 (Billing Portal) 的簡便方法](https://stripe.com/docs/billing/subscriptions/customer-portal)，讓您的客戶可以管理其訂閱、付款方式並查看其帳務歷史記錄。您可以從控制器或路由在可計費模型上呼叫 `redirectToBillingPortal` 方法，將使用者重導向到帳務中心：

```php
use Illuminate\Http\Request;

Route::get('/billing-portal', function (Request $request) {
    return $request->user()->redirectToBillingPortal();
});
```

預設情況下，當使用者完成訂閱管理後，他們將能夠透過 Stripe 帳務中心內的連結返回到應用程式的 `home` 路由。您可以透過將 URL 作為參數傳遞給 `redirectToBillingPortal` 方法，來提供使用者應返回的自定義 URL：

```php
use Illuminate\Http\Request;

Route::get('/billing-portal', function (Request $request) {
    return $request->user()->redirectToBillingPortal(route('billing'));
});
```

如果您只想生成指向帳務中心的 URL，而不生成 HTTP 重導向回應，則可以呼叫 `billingPortalUrl` 方法：

```php
$url = $request->user()->billingPortalUrl(route('billing'));
```

<a name="payment-methods"></a>
## 付款方式


<a name="storing-payment-methods"></a>
### 儲存付款方式

為了在 Stripe 建立訂閱或進行「單次」收費，您需要儲存付款方式並從 Stripe 檢索其識別碼。實現此目的的方法因您打算將付款方式用於訂閱還是單次收費而異，因此我們將在下方分別進行探討。


<a name="payment-methods-for-subscriptions"></a>
#### 用於訂閱的付款方式

當儲存客戶的信用卡資訊以供日後訂閱使用時，必須使用 Stripe 的「Setup Intents」API 來安全地收集客戶的付款方式詳情。「Setup Intent」向 Stripe 表示有意向客戶的付款方式進行收費。Cashier 的 `Billable` trait 包含了 `createSetupIntent` 方法，可輕鬆建立新的 Setup Intent。您應該從用於渲染收集客戶付款方式詳情表單的路由或控制器中調用此方法：

```php
return view('update-payment-method', [
    'intent' => $user->createSetupIntent()
]);
```

建立 Setup Intent 並將其傳遞給視圖後，您應該將其 secret 附加到用於收集付款方式的元素上。例如，考慮這個「更新付款方式」表單：

```html
<input id="card-holder-name" type="text">

<!-- Stripe Elements Placeholder -->
<div id="card-element"></div>

<button id="card-button" data-secret="{{ $intent->client_secret }}">
    Update Payment Method
</button>
```

接下來，可以使用 Stripe.js 函式庫將 [Stripe Element](https://stripe.com/docs/stripe-js) 附加到表單中，並安全地收集客戶的付款詳情：

```html
<script src="https://js.stripe.com/v3/"></script>

<script>
    const stripe = Stripe('stripe-public-key');

    const elements = stripe.elements();
    const cardElement = elements.create('card');

    cardElement.mount('#card-element');
</script>
```

接著，可以使用 [Stripe 的 `confirmCardSetup` 方法](https://stripe.com/docs/js/setup_intents/confirm_card_setup) 來驗證卡片，並從 Stripe 檢索安全的「付款方式識別碼」：

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

在卡片通過 Stripe 驗證後，您可以將產生的 `setupIntent.payment_method` 識別碼傳遞給您的 Laravel 應用程式，在那裡可以將其附加到客戶身上。該付款方式可以[作為新的付款方式新增](#adding-payment-methods)，或者[用於更新預設付款方式](#updating-the-default-payment-method)。您也可以立即使用該付款方式識別碼來[建立新訂閱](#creating-subscriptions)。

> [!NOTE]
> 如果您想瞭解更多關於 Setup Intents 和收集客戶付款詳情的資訊，請[參閱 Stripe 提供的此概覽](https://stripe.com/docs/payments/save-and-reuse#php)。


<a name="payment-methods-for-single-charges"></a>
#### 用於單次收費的付款方式

當然，在對客戶的付款方式進行單次收費時，我們只需要使用一次付款方式識別碼。由於 Stripe 的限制，您不得將客戶儲存的預設付款方式用於單次收費。您必須允許客戶使用 Stripe.js 函式庫輸入其付款方式詳情。例如，考慮以下表單：

```html
<input id="card-holder-name" type="text">

<!-- Stripe Elements Placeholder -->
<div id="card-element"></div>

<button id="card-button">
    Process Payment
</button>
```

定義此類表單後，可以使用 Stripe.js 函式庫將 [Stripe Element](https://stripe.com/docs/stripe-js) 附加到表單中，並安全地收集客戶的付款詳情：

```html
<script src="https://js.stripe.com/v3/"></script>

<script>
    const stripe = Stripe('stripe-public-key');

    const elements = stripe.elements();
    const cardElement = elements.create('card');

    cardElement.mount('#card-element');
</script>
```

接下來，可以使用 [Stripe 的 `createPaymentMethod` 方法](https://stripe.com/docs/stripe-js/reference#stripe-create-payment-method) 來驗證卡片，並從 Stripe 檢索安全的「付款方式識別碼」：

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

如果卡片驗證成功，您可以將 `paymentMethod.id` 傳遞給您的 Laravel 應用程式並處理[單次收費](#simple-charge)。


<a name="retrieving-payment-methods"></a>
### 檢索付款方式

可計費模型實例上的 `paymentMethods` 方法會回傳一個 `Laravel\Cashier\PaymentMethod` 實例的集合：

```php
$paymentMethods = $user->paymentMethods();
```

預設情況下，此方法將回傳所有類型的付款方式。要檢索特定類型的付款方式，可以將 `type` 作為參數傳遞給該方法：

```php
$paymentMethods = $user->paymentMethods('sepa_debit');
```

要檢索客戶的預設付款方式，可以使用 `defaultPaymentMethod` 方法：

```php
$paymentMethod = $user->defaultPaymentMethod();
```

您可以使用 `findPaymentMethod` 方法檢索附加到可計費模型的特定付款方式：

```php
$paymentMethod = $user->findPaymentMethod($paymentMethodId);
```


<a name="payment-method-presence"></a>
### 付款方式是否存在

要確定可計費模型的帳戶是否附加了預設付款方式，請調用 `hasDefaultPaymentMethod` 方法：

```php
if ($user->hasDefaultPaymentMethod()) {
    // ...
}
```

您可以使用 `hasPaymentMethod` 方法來確定可計費模型的帳戶是否至少附加了一個付款方式：

```php
if ($user->hasPaymentMethod()) {
    // ...
}
```

此方法將確定可計費模型是否擁有任何付款方式。要確定該模型是否存在特定類型的付款方式，可以將 `type` 作為參數傳遞給該方法：

```php
if ($user->hasPaymentMethod('sepa_debit')) {
    // ...
}
```


<a name="updating-the-default-payment-method"></a>
### 更新預設付款方式

`updateDefaultPaymentMethod` 方法可用於更新客戶的預設付款方式資訊。此方法接受一個 Stripe 付款方式識別碼，並將新的付款方式指定為預設的帳務付款方式：

```php
$user->updateDefaultPaymentMethod($paymentMethod);
```

要將您的預設付款方式資訊與 Stripe 中客戶的預設付款方式資訊同步，您可以使用 `updateDefaultPaymentMethodFromStripe` 方法：

```php
$user->updateDefaultPaymentMethodFromStripe();
```

> [!WARNING]
> 客戶的預設付款方式僅可用於開立發票和建立新訂閱。由於 Stripe 的限制，它不能用於單次收費。

<a name="adding-payment-methods"></a>
### 新增付款方式

若要新增付款方式，您可以在可計費模型上呼叫 `addPaymentMethod` 方法，並傳入付款方式識別碼：

```php
$user->addPaymentMethod($paymentMethod);
```

> [!NOTE]
> 若要了解如何檢索付款方式識別碼，請參閱[儲存付款方式文件](#storing-payment-methods)。

<a name="deleting-payment-methods"></a>
### 刪除付款方式

若要刪除付款方式，您可以在想要刪除的 `Laravel\Cashier\PaymentMethod` 實例上呼叫 `delete` 方法：

```php
$paymentMethod->delete();
```

`deletePaymentMethod` 方法可以從可計費模型中刪除特定的付款方式：

```php
$user->deletePaymentMethod('pm_visa');
```

`deletePaymentMethods` 方法將會刪除該可計費模型的所有付款方式資訊：

```php
$user->deletePaymentMethods();
```

預設情況下，此方法將刪除所有類型的付款方式。若要刪除特定類型的付款方式，您可以將 `type` 作為參數傳遞給該方法：

```php
$user->deletePaymentMethods('sepa_debit');
```

> [!WARNING]
> 如果使用者擁有有效訂閱，您的應用程式不應允許他們刪除其預設付款方式。

<a name="subscriptions"></a>
## 訂閱

訂閱提供了一種為您的客戶設定定期付款的方法。由 Cashier 管理的 Stripe 訂閱支援多個訂閱價格、訂閱數量、試用等功能。


<a name="creating-subscriptions"></a>
### 建立訂閱

要建立訂閱，請先檢索可計費模型的執行個體，通常會是 `App\Models\User` 的執行個體。檢索到模型執行個體後，您可以使用 `newSubscription` 方法來建立該模型的訂閱：

```php
use Illuminate\Http\Request;

Route::post('/user/subscribe', function (Request $request) {
    $request->user()->newSubscription(
        'default', 'price_monthly'
    )->create($request->paymentMethodId);

    // ...
});
```

傳遞給 `newSubscription` 方法的第一個參數應該是訂閱的內部類型。如果您的應用程式僅提供單一訂閱，您可以將其命名為 `default` 或 `primary`。此訂閱類型僅供應用程式內部使用，不應顯示給使用者。此外，它不應包含空格，且在建立訂閱後不應更改。第二個參數是使用者要訂閱的特定價格。此值應對應到 Stripe 中的價格識別碼。

`create` 方法接受 [Stripe 付款方式識別碼](#storing-payment-methods) 或 Stripe `PaymentMethod` 物件，它將開始訂閱，並使用可計費模型的 Stripe 客戶 ID 和其他相關帳務資訊更新您的資料庫。

> [!WARNING]
> 將付款方式識別碼直接傳遞給 `create` 訂閱方法，也會自動將其新增至使用者已儲存的付款方式中。


<a name="collecting-recurring-payments-via-invoice-emails"></a>
#### 透過發票郵件收取定期付款

除了自動收取客戶的定期付款外，您也可以指示 Stripe 在每次定期付款到期時向客戶發送發票郵件。接著，客戶可以在收到發票後手動支付。透過發票收取定期付款時，客戶不需要預先提供付款方式：

```php
$user->newSubscription('default', 'price_monthly')->createAndSendInvoice();
```

在訂閱被取消之前，客戶支付發票的時間由 `days_until_due` 選項決定。預設為 30 天；但是，如果您願意，可以為此選項提供一個特定值：

```php
$user->newSubscription('default', 'price_monthly')->createAndSendInvoice([], [
    'days_until_due' => 30
]);
```


<a name="subscription-quantities"></a>
#### 數量

如果您想在建立訂閱時為價格設定特定的 [數量](https://stripe.com/docs/billing/subscriptions/quantities)，您應該在建立訂閱之前在訂閱構建器上呼叫 `quantity` 方法：

```php
$user->newSubscription('default', 'price_monthly')
    ->quantity(5)
    ->create($paymentMethod);
```


<a name="additional-details"></a>
#### 額外細節

如果您想指定 Stripe 支援的其他 [客戶](https://stripe.com/docs/api/customers/create) 或 [訂閱](https://stripe.com/docs/api/subscriptions/create) 選項，可以將其作為第二和第三個參數傳遞給 `create` 方法：

```php
$user->newSubscription('default', 'price_monthly')->create($paymentMethod, [
    'email' => $email,
], [
    'metadata' => ['note' => 'Some extra information.'],
]);
```


<a name="coupons"></a>
#### 優惠券

如果您想在建立訂閱時使用優惠券，可以使用 `withCoupon` 方法：

```php
$user->newSubscription('default', 'price_monthly')
    ->withCoupon('code')
    ->create($paymentMethod);
```

或者，如果您想使用 [Stripe 促銷代碼](https://stripe.com/docs/billing/subscriptions/discounts/codes)，可以使用 `withPromotionCode` 方法：

```php
$user->newSubscription('default', 'price_monthly')
    ->withPromotionCode('promo_code_id')
    ->create($paymentMethod);
```

給定的促銷代碼 ID 應該是分配給促銷代碼的 Stripe API ID，而非面向客戶的促銷代碼。如果您需要根據給定的面向客戶促銷代碼尋找促銷代碼 ID，可以使用 `findPromotionCode` 方法：

```php
// Find a promotion code ID by its customer facing code...
$promotionCode = $user->findPromotionCode('SUMMERSALE');

// Find an active promotion code ID by its customer facing code...
$promotionCode = $user->findActivePromotionCode('SUMMERSALE');
```

在上面的範例中，回傳的 `$promotionCode` 物件是 `Laravel\Cashier\PromotionCode` 的執行個體。此類別封裝了底層的 `Stripe\PromotionCode` 物件。您可以透過呼叫 `coupon` 方法來檢索與促銷代碼相關的優惠券：

```php
$coupon = $user->findPromotionCode('SUMMERSALE')->coupon();
```

優惠券執行個體允許您確定折扣金額，以及優惠券是代表固定折扣還是基於百分比的折扣：

```php
if ($coupon->isPercentage()) {
    return $coupon->percentOff().'%'; // 21.5%
} else {
    return $coupon->amountOff(); // $5.99
}
```

您還可以檢索目前套用於客戶或訂閱的折扣：

```php
$discount = $billable->discount();

$discount = $subscription->discount();
```

回傳的 `Laravel\Cashier\Discount` 執行個體封裝了底層的 `Stripe\Discount` 物件執行個體。您可以透過呼叫 `coupon` 方法來檢索與此折扣相關的優惠券：

```php
$coupon = $subscription->discount()->coupon();
```

如果您想為客戶或訂閱套用新的優惠券或促銷代碼，可以透過 `applyCoupon` 或 `applyPromotionCode` 方法來執行：

```php
$billable->applyCoupon('coupon_id');
$billable->applyPromotionCode('promotion_code_id');

$subscription->applyCoupon('coupon_id');
$subscription->applyPromotionCode('promotion_code_id');
```

請記住，您應該使用分配給促銷代碼的 Stripe API ID，而非面向客戶的促銷代碼。同一時間只能為客戶或訂閱套用一個優惠券或促銷代碼。

有關此主題的更多資訊，請參閱 Stripe 關於 [優惠券](https://stripe.com/docs/billing/subscriptions/coupons) 和 [促銷代碼](https://stripe.com/docs/billing/subscriptions/coupons/codes) 的文件。


<a name="adding-subscriptions"></a>
#### 新增訂閱

如果您想為已經有預設付款方式的客戶新增訂閱，可以在訂閱構建器上呼叫 `add` 方法：

```php
use App\Models\User;

$user = User::find(1);

$user->newSubscription('default', 'price_monthly')->add();
```


<a name="creating-subscriptions-from-the-stripe-dashboard"></a>
#### 從 Stripe Dashboard 建立訂閱

您也可以直接從 Stripe Dashboard 建立訂閱。這樣做時，Cashier 會同步新新增的訂閱，並為其分配 `default` 類型。若要自訂分配給從 Dashboard 建立之訂閱的訂閱類型，請 [定義 Webhook 事件處理程式](#defining-webhook-event-handlers)。

此外，您只能透過 Stripe Dashboard 建立一種類型的訂閱。如果您的應用程式提供多個使用不同類型的訂閱，則只能透過 Stripe Dashboard 新增一種類型的訂閱。

最後，您應該始終確保應用程式提供的每種訂閱類型只新增一個活動訂閱。如果客戶有兩個 `default` 訂閱，即使兩者都已與應用程式的資料庫同步，Cashier 也只會使用最近新增的訂閱。

<a name="checking-subscription-status"></a>
### 檢查訂閱狀態

當客戶訂閱您的應用程式後，您可以使用多種方便的方法輕鬆檢查其訂閱狀態。首先，如果客戶擁有有效訂閱，即使訂閱目前處於試用期內，`subscribed` 方法也會回傳 `true`。`subscribed` 方法接受訂閱類型作為其第一個參數：

```php
if ($user->subscribed('default')) {
    // ...
}
```

`subscribed` 方法也是[路由中介層](/docs/{{version}}/middleware)的絕佳候選對象，讓您可以根據使用者的訂閱狀態過濾對路由和控制器 (Controller) 的存取：

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

如果您想確定使用者是否仍處於試用期內，可以使用 `onTrial` 方法。此方法對於判斷是否應向使用者顯示其仍處於試用期的警告非常有用：

```php
if ($user->subscription('default')->onTrial()) {
    // ...
}
```

`subscribedToProduct` 方法可用於根據給定的 Stripe 產品識別碼來判斷使用者是否訂閱了該產品。在 Stripe 中，產品是價格的集合。在此範例中，我們將判斷使用者的 `default` 訂閱是否已主動訂閱應用程式的 "premium" 產品。給定的 Stripe 產品識別碼應對應於 Stripe 儀表板中的其中一個產品識別碼：

```php
if ($user->subscribedToProduct('prod_premium', 'default')) {
    // ...
}
```

透過向 `subscribedToProduct` 方法傳遞一個陣列，您可以判斷使用者的 `default` 訂閱是否主動訂閱了應用程式的 "basic" 或 "premium" 產品：

```php
if ($user->subscribedToProduct(['prod_basic', 'prod_premium'], 'default')) {
    // ...
}
```

`subscribedToPrice` 方法可用於判斷客戶的訂閱是否對應於給定的價格 ID：

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
> 如果使用者擁有兩個相同類型的訂閱，`subscription` 方法將始終回傳最近的訂閱。例如，使用者可能有兩條類型為 `default` 的訂閱紀錄；然而，其中一個訂閱可能是舊的、已過期的訂閱，而另一個是目前的、有效的訂閱。系統將始終回傳最近的訂閱，而較舊的訂閱則保留在資料庫中供歷史回顧。


<a name="cancelled-subscription-status"></a>
#### 取消訂閱狀態

要判斷使用者是否曾經是有效訂閱者但已取消訂閱，可以使用 `canceled` 方法：

```php
if ($user->subscription('default')->canceled()) {
    // ...
}
```

您也可以判斷使用者是否已取消訂閱，但仍處於訂閱完全過期前的「寬限期 (Grace Period)」。例如，如果使用者在 3 月 5 日取消了原定於 3 月 10 日到期的訂閱，則該使用者在 3 月 10 日之前都處於「寬限期」。請注意，在此期間 `subscribed` 方法仍然會回傳 `true`：

```php
if ($user->subscription('default')->onGracePeriod()) {
    // ...
}
```

要判斷使用者是否已取消訂閱且不再處於「寬限期」內，可以使用 `ended` 方法：

```php
if ($user->subscription('default')->ended()) {
    // ...
}
```


<a name="incomplete-and-past-due-status"></a>
#### 未完成與逾期狀態

如果訂閱在建立後需要進行二次付款操作，則該訂閱將被標記為 `incomplete`。訂閱狀態儲存在 Cashier 的 `subscriptions` 資料庫表格的 `stripe_status` 欄位中。

同樣地，如果在更換價格時需要進行二次付款操作，該訂閱將被標記為 `past_due`。當您的訂閱處於這兩種狀態中的任一種時，在客戶確認付款之前，它都不會生效。可以使用可計費模型或訂閱實例上的 `hasIncompletePayment` 方法來判斷訂閱是否存在未完成的付款：

```php
if ($user->hasIncompletePayment('default')) {
    // ...
}

if ($user->subscription('default')->hasIncompletePayment()) {
    // ...
}
```

當訂閱存在未完成的付款時，您應該將使用者引導至 Cashier 的付款確認頁面，並傳遞 `latestPayment` 識別碼。您可以使用訂閱實例上的 `latestPayment` 方法來檢索此識別碼：

```html
<a href="{{ route('cashier.payment', $subscription->latestPayment()->id) }}">
    Please confirm your payment.
</a>
```

如果您希望訂閱在處於 `past_due` 或 `incomplete` 狀態時仍被視為有效，可以使用 Cashier 提供的 `keepPastDueSubscriptionsActive` 和 `keepIncompleteSubscriptionsActive` 方法。通常，這些方法應該在 `App\Providers\AppServiceProvider` 的 `register` 方法中調用：

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
> 當訂閱處於 `incomplete` 狀態時，在付款確認之前無法進行更改。因此，當訂閱處於 `incomplete` 狀態時，`swap` 和 `updateQuantity` 方法將拋出異常。


<a name="subscription-scopes"></a>
#### 訂閱 Scope

大多數訂閱狀態也可以作為查詢 Scope 使用，以便您可以輕鬆地在資料庫中查詢處於特定狀態的訂閱：

```php
// Get all active subscriptions...
$subscriptions = Subscription::query()->active()->get();

// Get all of the canceled subscriptions for a user...
$subscriptions = $user->subscriptions()->canceled()->get();
```

可用 Scope 的完整列表如下：

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

當客戶訂閱了您的應用程式後，他們有時會想要變更為新的訂閱價格。若要將客戶更換為新價格，請將 Stripe 的價格識別碼傳遞給 `swap` 方法。在更換價格時，系統會假設如果該訂閱先前已被取消，則使用者想要重新啟用該訂閱。指定的價格識別碼應對應於 Stripe 儀表板中可用的 Stripe 價格識別碼：

```php
use App\Models\User;

$user = App\Models\User::find(1);

$user->subscription('default')->swap('price_yearly');
```

如果客戶處於試用期，則試用期將會保留。此外，如果訂閱存在「數量 (Quantity)」，該數量也將被保留。

如果您想更換價格並取消客戶目前的任何試用期，可以使用 `skipTrial` 方法：

```php
$user->subscription('default')
    ->skipTrial()
    ->swap('price_yearly');
```

如果您想更換價格並立即向客戶開立發票，而不是等待下一個計費週期，可以使用 `swapAndInvoice` 方法：

```php
$user = User::find(1);

$user->subscription('default')->swapAndInvoice('price_yearly');
```


<a name="prorations"></a>
#### 按比例分配 (Prorations)

預設情況下，Stripe 在更換價格時會按比例分配費用。`noProrate` 方法可用於更新訂閱價格，而無需按比例分配費用：

```php
$user->subscription('default')->noProrate()->swap('price_yearly');
```

有關訂閱比例分配的更多資訊，請參閱 [Stripe 文件](https://stripe.com/docs/billing/subscriptions/prorations)。

> [!WARNING]
> 在 `swapAndInvoice` 方法之前執行 `noProrate` 方法對比例分配不會產生任何影響。發票將始終被開立。


<a name="subscription-quantity"></a>
### 訂閱數量

有時訂閱會受到「數量 (Quantity)」的影響。例如，一個專案管理應用程式可能會針對每個專案每月收取 10 美元。您可以使用 `incrementQuantity` 和 `decrementQuantity` 方法輕鬆增加或減少您的訂閱數量：

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

`noProrate` 方法可用於更新訂閱數量，而無需按比例分配費用：

```php
$user->subscription('default')->noProrate()->updateQuantity(10);
```

有關訂閱數量的更多資訊，請參閱 [Stripe 文件](https://stripe.com/docs/subscriptions/quantities)。


<a name="quantities-for-subscription-with-multiple-products"></a>
#### 多重產品訂閱的數量

如果您的訂閱是[多重產品訂閱](#subscriptions-with-multiple-products)，您應該將想要增加或減少數量的價格 ID 作為第二個參數傳遞給增加/減少方法：

```php
$user->subscription('default')->incrementQuantity(1, 'price_chat');
```

<a name="subscriptions-with-multiple-products"></a>
### 多重產品訂閱

[多重產品訂閱](https://stripe.com/docs/billing/subscriptions/multiple-products) 允許您將多個計費產品分配給單個訂閱。例如，想像您正在建立一個客戶服務「Helpdesk」應用程式，其基礎訂閱價格為每月 10 美元，但提供額外的即時對話加購產品，每月 15 美元。多重產品訂閱的資訊儲存在 Cashier 的 `subscription_items` 資料庫資料表中。

您可以透過將價格陣列作為第二個參數傳遞給 `newSubscription` 方法，來為給定訂閱指定多個產品：

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

在上面的範例中，客戶將有兩個價格附加到他們的 `default` 訂閱中。這兩個價格都將依照各自的帳務週期收費。如有需要，您可以使用 `quantity` 方法為每個價格指定特定數量：

```php
$user = User::find(1);

$user->newSubscription('default', ['price_monthly', 'price_chat'])
    ->quantity(5, 'price_chat')
    ->create($paymentMethod);
```

如果您想在現有的訂閱中新增另一個價格，您可以呼叫該訂閱的 `addPrice` 方法：

```php
$user = User::find(1);

$user->subscription('default')->addPrice('price_chat');
```

上面的範例將新增一個新價格，客戶將在下一個帳務週期收到該價格的帳單。如果您想立即向客戶收取費用，可以使用 `addPriceAndInvoice` 方法：

```php
$user->subscription('default')->addPriceAndInvoice('price_chat');
```

如果您想新增具有特定數量的價格，可以將數量作為 `addPrice` 或 `addPriceAndInvoice` 方法的第二個參數傳遞：

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

您也可以變更附加到多重產品訂閱的價格。例如，想像客戶有一個包含 `price_chat` 加購產品的 `price_basic` 訂閱，而您想將客戶從 `price_basic` 升級到 `price_pro` 價格：

```php
use App\Models\User;

$user = User::find(1);

$user->subscription('default')->swap(['price_pro', 'price_chat']);
```

執行上述範例時，底層具有 `price_basic` 的訂閱項目會被刪除，而具有 `price_chat` 的項目會被保留。此外，還會為 `price_pro` 建立一個新的訂閱項目。

您還可以透過向 `swap` 方法傳遞鍵值對陣列來指定訂閱項目選項。例如，您可能需要指定訂閱價格的數量：

```php
$user = User::find(1);

$user->subscription('default')->swap([
    'price_pro' => ['quantity' => 5],
    'price_chat'
]);
```

如果您想變更訂閱中的單個價格，可以使用訂閱項目本身的 `swap` 方法。如果您想保留訂閱中其他價格的所有現有中繼資料 (Metadata)，這種方法特別有用：

```php
$user = User::find(1);

$user->subscription('default')
    ->findItemOrFail('price_basic')
    ->swap('price_pro');
```


<a name="proration"></a>
#### 按比例分配

預設情況下，當從多重產品訂閱中新增或移除價格時，Stripe 會按比例分配 (Prorate) 費用。如果您想在不按比例分配的情況下進行價格調整，您應該在價格操作中串接 `noProrate` 方法：

```php
$user->subscription('default')->noProrate()->removePrice('price_chat');
```


<a name="swapping-quantities"></a>
#### 數量

如果您想更新個別訂閱價格的數量，可以使用[現有的數量方法](#subscription-quantity)，並將價格的 ID 作為方法的額外參數傳遞：

```php
$user = User::find(1);

$user->subscription('default')->incrementQuantity(5, 'price_chat');

$user->subscription('default')->decrementQuantity(3, 'price_chat');

$user->subscription('default')->updateQuantity(10, 'price_chat');
```

> [!WARNING]
> 當一個訂閱具有多個價格時，`Subscription` 模型上的 `stripe_price` 和 `quantity` 屬性將為 `null`。要存取個別價格屬性，您應該使用 `Subscription` 模型上可用的 `items` 關聯。


<a name="subscription-items"></a>
#### 訂閱項目

當一個訂閱有多個價格時，您的資料庫 `subscription_items` 資料表中會儲存多個訂閱「項目 (Items)」。您可以透過訂閱上的 `items` 關聯來存取這些項目：

```php
use App\Models\User;

$user = User::find(1);

$subscriptionItem = $user->subscription('default')->items->first();

// Retrieve the Stripe price and quantity for a specific item...
$stripePrice = $subscriptionItem->stripe_price;
$quantity = $subscriptionItem->quantity;
```

您還可以使用 `findItemOrFail` 方法檢索特定價格：

```php
$user = User::find(1);

$subscriptionItem = $user->subscription('default')->findItemOrFail('price_chat');
```


<a name="multiple-subscriptions"></a>
### 多重訂閱

Stripe 允許您的客戶同時擁有多個訂閱。例如，您經營一家提供游泳訂閱和舉重訂閱的健身房，每種訂閱可能有不同的定價。當然，客戶應該能夠訂閱其中一種或兩種方案。

當您的應用程式建立訂閱時，您可以向 `newSubscription` 方法提供訂閱類型。該類型可以是代表使用者正在發起的訂閱類型的任何字串：

```php
use Illuminate\Http\Request;

Route::post('/swimming/subscribe', function (Request $request) {
    $request->user()->newSubscription('swimming')
        ->price('price_swimming_monthly')
        ->create($request->paymentMethodId);

    // ...
});
```

在這個範例中，我們為客戶發起了一個每月的游泳訂閱。但是，他們可能想在稍後更換為年度訂閱。調整客戶訂閱時，我們只需更換 `swimming` 訂閱上的價格：

```php
$user->subscription('swimming')->swap('price_swimming_yearly');
```

當然，您也可以完全取消訂閱：

```php
$user->subscription('swimming')->cancel();
```

<a name="usage-based-billing"></a>
### 按量計費

[按量計費 (Usage based billing)](https://stripe.com/docs/billing/subscriptions/metered-billing) 允許您根據客戶在計費週期內的產品使用量來收費。例如，您可以根據客戶每月發送的簡訊或電子郵件數量來向其收費。

要開始使用按量計費，您首先需要在 Stripe 儀表板中建立一個具有 [按量計費模型 (usage based billing model)](https://docs.stripe.com/billing/subscriptions/usage-based/implementation-guide) 與 [計量器 (meter)](https://docs.stripe.com/billing/subscriptions/usage-based/recording-usage#configure-meter) 的新產品。建立計量器後，請記錄相關的事件名稱與計量器 ID，您將需要這些資訊來回報與檢索使用量。接著，使用 `meteredPrice` 方法將按量計費的價格 ID 新增到客戶訂閱中：

```php
use Illuminate\Http\Request;

Route::post('/user/subscribe', function (Request $request) {
    $request->user()->newSubscription('default')
        ->meteredPrice('price_metered')
        ->create($request->paymentMethodId);

    // ...
});
```

您也可以透過 [Stripe Checkout](#checkout) 發起按量計費的訂閱：

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
#### 回報使用量

當客戶使用您的應用程式時，您需要向 Stripe 回報其使用量，以便能準確計費。若要回報計量事件的使用量，您可以在 `Billable` 模型上使用 `reportMeterEvent` 方法：

```php
$user = User::find(1);

$user->reportMeterEvent('emails-sent');
```

預設情況下，計費期間會增加 1 個「使用數量」。或者，您也可以傳遞特定的「使用量」數值來增加客戶在該計費期間的使用量：

```php
$user = User::find(1);

$user->reportMeterEvent('emails-sent', quantity: 15);
```

若要檢索客戶對計量器的事件摘要，您可以使用 `Billable` 實例的 `meterEventSummaries` 方法：

```php
$user = User::find(1);

$meterUsage = $user->meterEventSummaries($meterId);

$meterUsage->first()->aggregated_value // 10
```

關於計量事件摘要的更多資訊，請參閱 Stripe 的 [Meter Event Summary 物件文件](https://docs.stripe.com/api/billing/meter-event_summary/object)。

若要 [列出所有計量器](https://docs.stripe.com/api/billing/meter/list)，您可以使用 `Billable` 實例的 `meters` 方法：

```php
$user = User::find(1);

$user->meters();
```


<a name="subscription-taxes"></a>
### 訂閱稅務

> [!WARNING]
> 除了手動計算稅率，您也可以[使用 Stripe Tax 自動計算稅費](#tax-configuration)。

若要指定使用者在訂閱時支付的稅率，您應該在可計費模型中實作 `taxRates` 方法，並回傳一個包含 Stripe 稅率 ID 的陣列。您可以在 [Stripe 儀表板](https://dashboard.stripe.com/test/tax-rates) 中定義這些稅率：

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

`taxRates` 方法讓您可以根據不同客戶套用不同的稅率，這對於跨越多個國家且稅率各異的使用者群體非常有幫助。

如果您提供的是包含多重產品的訂閱，您可以透過在可計費模型中實作 `priceTaxRates` 方法，為每個價格定義不同的稅率：

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
> `taxRates` 方法僅適用於訂閱費用。如果您使用 Cashier 進行「單次」收費，則需要在那時手動指定稅率。


<a name="syncing-tax-rates"></a>
#### 同步稅率

當變更 `taxRates` 方法回傳的硬編碼稅率 ID 時，使用者任何現有訂閱的稅務設定將保持不變。如果您希望使用新的 `taxRates` 數值更新現有訂閱的稅務價值，您應該在該使用者的訂閱實例上呼電 `syncTaxRates` 方法：

```php
$user->subscription('default')->syncTaxRates();
```

這也會同步多重產品訂閱中的任何項目稅率。如果您的應用程式提供多重產品訂閱，請確保您的可計費模型實作了[上述](#subscription-taxes)提到的 `priceTaxRates` 方法。


<a name="tax-exemption"></a>
#### 免稅

Cashier 還提供了 `isNotTaxExempt`、`isTaxExempt` 以及 `reverseChargeApplies` 方法來判斷客戶是否免稅。這些方法會呼叫 Stripe API 來判斷客戶的免稅狀態：

```php
use App\Models\User;

$user = User::find(1);

$user->isTaxExempt();
$user->isNotTaxExempt();
$user->reverseChargeApplies();
```

> [!WARNING]
> 這些方法也可在任何 `Laravel\Cashier\Invoice` 物件上使用。然而，當在 `Invoice` 物件上呼叫時，這些方法將判斷發票建立時的免稅狀態。


<a name="subscription-anchor-date"></a>
### 訂閱錨定日期

預設情況下，帳單週期錨定日期是訂閱建立的日期，或者如果使用了試用期，則為試用結束的日期。如果您想修改帳單錨定日期，可以使用 `anchorBillingCycleOn` 方法：

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

關於管理訂閱計費週期的更多資訊，請參閱 [Stripe 帳單週期文件](https://stripe.com/docs/billing/subscriptions/billing-cycle)。


<a name="cancelling-subscriptions"></a>
### 取消訂閱

要取消訂閱，請在該使用者的訂閱上呼叫 `cancel` 方法：

```php
$user->subscription('default')->cancel();
```

當訂閱被取消時，Cashier 會自動設定 `subscriptions` 資料表中的 `ends_at` 欄位。此欄位用於判斷 `subscribed` 方法何時應該開始回傳 `false`。

例如，如果客戶在 3 月 1 日取消訂閱，但該訂閱原定於 3 月 5 日才結束，則 `subscribed` 方法在 3 月 5 日之前仍會回傳 `true`。這是因為通常允許使用者繼續使用應用程式，直到其計費週期結束為止。

您可以使用 `onGracePeriod` 方法來判斷使用者是否已取消訂閱但仍處於「寬限期 (grace period)」：

```php
if ($user->subscription('default')->onGracePeriod()) {
    // ...
}
```

如果您希望立即取消訂閱，請在該使用者的訂閱上呼叫 `cancelNow` 方法：

```php
$user->subscription('default')->cancelNow();
```

如果您希望立即取消訂閱並針對任何剩餘的未開票按量使用量或新建立/掛起的比例分配 (Proration) 發票項目開立發票，請在該使用者的訂閱上呼叫 `cancelNowAndInvoice` 方法：

```php
$user->subscription('default')->cancelNowAndInvoice();
```

您也可以選擇在特定時間點取消訂閱：

```php
$user->subscription('default')->cancelAt(
    now()->plus(days: 10)
);
```

最後，在刪除相關的使用者模型之前，您應該一律先取消該使用者的訂閱：

```php
$user->subscription('default')->cancelNow();

$user->delete();
```

<a name="resuming-subscriptions"></a>
### 恢復訂閱

如果客戶取消了他們的訂閱且您希望恢復它，您可以對該訂閱呼叫 `resume` 方法。客戶必須仍處於其「寬限期」內才能恢復訂閱：

```php
$user->subscription('default')->resume();
```

如果客戶取消了訂閱，並在訂閱完全過期之前恢復該訂閱，則不會立即向客戶計費。相反地，他們的訂閱將被重新啟用，並按原始帳單週期進行計費。

<a name="subscription-trials"></a>
## 訂閱試用

<a name="with-payment-method-up-front"></a>
### 預先收集付款方式

若您希望在提供客戶試用期的同時預先收集付款方式資訊，您應該在建立訂閱時使用 `trialDays` 方法：

```php
use Illuminate\Http\Request;

Route::post('/user/subscribe', function (Request $request) {
    $request->user()->newSubscription('default', 'price_monthly')
        ->trialDays(10)
        ->create($request->paymentMethodId);

    // ...
});
```

此方法會在資料庫的訂閱紀錄中設定試用結束日期，並指示 Stripe 在此日期之後才開始向客戶計費。當使用 `trialDays` 方法時，Cashier 將會覆寫 Stripe 中為該價格設定的任何預設試用期。

> [!WARNING]
> 如果客戶的訂閱未在試用結束日期前取消，則試用一到期就會立即扣款，因此請務必告知使用者其試用結束日期。

`trialUntil` 方法允許您提供一個 `DateTime` 實例，用以指定試用期應於何時結束：

```php
use Illuminate\Support\Carbon;

$user->newSubscription('default', 'price_monthly')
    ->trialUntil(Carbon::now()->plus(days: 10))
    ->create($paymentMethod);
```

您可以使用使用者實例的 `onTrial` 方法，或是訂閱實例的 `onTrial` 方法來判斷使用者是否處於試用期。以下兩個範例是等效的：

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

若要判斷現有的試用是否已過期，您可以使用 `hasExpiredTrial` 方法：

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

您可以選擇在 Stripe 控制面板中定義價格所獲得的試用天數，或是始終透過 Cashier 明確地傳遞。如果您選擇在 Stripe 中定義價格的試用天數，您應該注意，新的訂閱（包括曾有過訂閱記錄的客戶所建立的新訂閱）將始終會獲得試用期，除非您明確呼叫 `skipTrial()` 方法。

<a name="without-payment-method-up-front"></a>
### 不預先收集付款方式

若您想在不預先收集使用者付款方式資訊的情況下提供試用期，您可以將使用者紀錄中的 `trial_ends_at` 欄位設定為您想要的試用結束日期。這通常是在使用者註冊期間完成的：

```php
use App\Models\User;

$user = User::create([
    // ...
    'trial_ends_at' => now()->plus(days: 10),
]);
```

> [!WARNING]
> 請務必在您的可計費模型類別定義中為 `trial_ends_at` 屬性新增[日期轉型 (Date Casting)](/docs/{{version}}/eloquent-mutators#date-casting)。

Cashier 將這種類型的試用稱為「通用試用 (Generic Trial)」，因為它不屬於任何現有訂閱。如果當前日期未超過 `trial_ends_at` 的值，則可計費模型實例上的 `onTrial` 方法將回傳 `true`：

```php
if ($user->onTrial()) {
    // User is within their trial period...
}
```

當您準備好為使用者建立實際訂閱時，可以像平常一樣使用 `newSubscription` 方法：

```php
$user = User::find(1);

$user->newSubscription('default', 'price_monthly')->create($paymentMethod);
```

若要檢索使用者的試用結束日期，您可以使用 `trialEndsAt` 方法。如果使用者處於試用期，此方法將回傳一個 Carbon 日期實例，否則回傳 `null`。如果您想獲取預設訂閱以外的特定訂閱試用結束日期，也可以傳遞一個選用的訂閱類型參數：

```php
if ($user->onTrial()) {
    $trialEndsAt = $user->trialEndsAt('main');
}
```

如果您想特別確認使用者是否處於其「通用」試用期內且尚未建立實際訂閱，您也可以使用 `onGenericTrial` 方法：

```php
if ($user->onGenericTrial()) {
    // User is within their "generic" trial period...
}
```

<a name="extending-trials"></a>
### 延長試用期

`extendTrial` 方法允許您在建立訂閱後延長其試用期。如果試用期已經過期且客戶已經在為該訂閱付費，您仍然可以為他們提供延長試用。試用期內花費的時間將從客戶的下一張發票中扣除：

```php
use App\Models\User;

$subscription = User::find(1)->subscription('default');

// End the trial 7 days from now...
$subscription->extendTrial(
    now()->plus(days: 7)
);

// Add an additional 5 days to the trial...
$subscription->extendTrial(
    $subscription->trial_ends_at->plus(days: 5)
);
```

<a name="handling-stripe-webhooks"></a>
## 處理 Stripe Webhooks

> [!NOTE]
> 您可以使用 [Stripe CLI](https://stripe.com/docs/stripe-cli) 來協助在本地開發期間測試 webhook。

Stripe 可以透過 webhook 通知您的應用程式各種事件。預設情況下，Cashier 服務提供者會自動註冊一個指向 Cashier webhook 控制器的路由。此控制器將處理所有傳入的 webhook 請求。

根據預設，Cashier webhook 控制器會自動處理因扣款失敗次數過多而取消訂閱（如您的 Stripe 設定中所定義）、客戶更新、客戶刪除、訂閱更新以及付款方式變更；然而，正如我們很快就會發現的，您可以擴充此控制器來處理任何您喜歡的 Stripe webhook 事件。

為確保您的應用程式可以處理 Stripe webhook，請務必在 Stripe 管理面板中設定 webhook URL。預設情況下，Cashier 的 webhook 控制器會回應 `/stripe/webhook` URL 路徑。您應該在 Stripe 管理面板中啟用的所有 webhook 完整清單如下：

- `customer.subscription.created`
- `customer.subscription.updated`
- `customer.subscription.deleted`
- `customer.updated`
- `customer.deleted`
- `payment_method.automatically_updated`
- `invoice.payment_action_required`
- `invoice.payment_succeeded`

為了方便起見，Cashier 包含了一個 `cashier:webhook` Artisan 指令。此指令將在 Stripe 中建立一個 webhook，監聽 Cashier 所需的所有事件：

```shell
php artisan cashier:webhook
```

預設情況下，建立的 webhook 將指向 `APP_URL` 環境變數定義的 URL 以及 Cashier 包含的 `cashier.webhook` 路由。如果您想使用不同的 URL，可以在叫用指令時提供 `--url` 選項：

```shell
php artisan cashier:webhook --url "https://example.com/stripe/webhook"
```

建立的 webhook 將使用與您的 Cashier 版本相容的 Stripe API 版本。如果您想使用不同的 Stripe 版本，可以提供 `--api-version` 選項：

```shell
php artisan cashier:webhook --api-version="2019-12-03"
```

建立後，webhook 將立即生效。如果您希望建立 webhook 但先將其停用直到您準備好為止，可以在叫用指令時提供 `--disabled` 選項：

```shell
php artisan cashier:webhook --disabled
```

> [!WARNING]
> 請確保您使用 Cashier 包含的 [webhook 簽章驗證](#verifying-webhook-signatures)中介層來保護傳入的 Stripe webhook 請求。


<a name="webhooks-csrf-protection"></a>
#### Webhooks 與 CSRF 保護

由於 Stripe webhook 需要繞過 Laravel 的 [CSRF 保護](/docs/{{version}}/csrf)，您應該確保 Laravel 不會嘗試驗證傳入 Stripe webhook 的 CSRF 權杖 (Token)。為此，您應該在應用程式的 `bootstrap/app.php` 檔案中將 `stripe/*` 從 CSRF 保護中排除：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->validateCsrfTokens(except: [
        'stripe/*',
    ]);
})
```


<a name="defining-webhook-event-handlers"></a>
### 定義 Webhook 事件處理程式

Cashier 會自動處理因扣款失敗而導致的訂閱取消以及其他常見的 Stripe webhook 事件。然而，如果您有額外的 webhook 事件想要處理，可以透過監聽以下由 Cashier 發送的事件來達成：

- `Laravel\Cashier\Events\WebhookReceived`
- `Laravel\Cashier\Events\WebhookHandled`

這兩個事件都包含 Stripe webhook 的完整 Payload。例如，如果您想處理 `invoice.payment_succeeded` webhook，可以註冊一個用來處理該事件的[監聽器](/docs/{{version}}/events#defining-listeners)：

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

為了確保您的 webhook 安全，您可以使用 [Stripe 的 webhook 簽章](https://stripe.com/docs/webhooks/signatures)。為了方便起見，Cashier 自動包含了一個中介層，用於驗證傳入的 Stripe webhook 請求是否有效。

要啟用 webhook 驗證，請確保您的應用程式 `.env` 檔案中設定了 `STRIPE_WEBHOOK_SECRET` 環境變數。webhook 的 `secret` (金鑰) 可以從您的 Stripe 帳戶管理面板中取得。

<a name="single-charges"></a>
## 單次收費


<a name="simple-charge"></a>
### 簡單收費

若您想對客戶進行一次性收費，您可以在可計費模型實例上使用 `charge` 方法。您需要[提供一個付款方式識別碼](#payment-methods-for-single-charges)作為 `charge` 方法的第二個參數：

```php
use Illuminate\Http\Request;

Route::post('/purchase', function (Request $request) {
    $stripeCharge = $request->user()->charge(
        100, $request->paymentMethodId
    );

    // ...
});
```

`charge` 方法接受一個陣列作為其第三個參數，允許您將任何想要的選項傳遞給底層的 Stripe 收費建立程序。有關建立收費時可用選項的更多資訊，可以在 [Stripe 文件](https://stripe.com/docs/api/charges/create)中找到：

```php
$user->charge(100, $paymentMethod, [
    'custom_option' => $value,
]);
```

您也可以在沒有底層客戶或使用者的情況下使用 `charge` 方法。為此，請在應用程式的可計費模型的新實例上呼叫 `charge` 方法：

```php
use App\Models\User;

$stripeCharge = (new User)->charge(100, $paymentMethod);
```

如果收費失敗，`charge` 方法將拋出異常。如果收費成功，該方法將回傳一個 `Laravel\Cashier\Payment` 實例：

```php
try {
    $payment = $user->charge(100, $paymentMethod);
} catch (Exception $e) {
    // ...
}
```

> [!WARNING]
> `charge` 方法接受以您應用程式所使用的貨幣最低面額計算的付款金額。例如，如果客戶以美元支付，金額應以分 (pennies) 為單位。


<a name="charge-with-invoice"></a>
### 含發票的收費

有時您可能需要進行一次性收費並向客戶提供 PDF 發票。`invoicePrice` 方法可以讓您實現這一點。例如，讓我們為客戶購買的五件新襯衫開立發票：

```php
$user->invoicePrice('price_tshirt', 5);
```

發票將立即從使用者的預設付款方式中扣款。`invoicePrice` 方法也接受一個陣列作為其第三個參數。此陣列包含發票項目的計費選項。該方法接受的第四個參數也是一個陣列，應包含發票本身的計費選項：

```php
$user->invoicePrice('price_tshirt', 5, [
    'discounts' => [
        ['coupon' => 'SUMMER21SALE']
    ],
], [
    'default_tax_rates' => ['txr_id'],
]);
```

與 `invoicePrice` 類似，您可以使用 `tabPrice` 方法將多個項目（每張發票最多 250 個項目）新增到客戶的「簽帳單 (tab)」中，然後再向客戶開立發票，以此建立一次性收費。例如，我們可以為客戶開立五件襯衫和兩個馬克杯的發票：

```php
$user->tabPrice('price_tshirt', 5);
$user->tabPrice('price_mug', 2);
$user->invoice();
```

或者，您可以使用 `invoiceFor` 方法對客戶的預設付款方式進行「一次性」收費：

```php
$user->invoiceFor('One Time Fee', 500);
```

雖然 `invoiceFor` 方法可供您使用，但建議您使用預先定義價格的 `invoicePrice` 和 `tabPrice` 方法。透過這種方式，您可以在 Stripe 控制面板中獲得更好的按產品銷售分析與數據。

> [!WARNING]
> `invoice`、`invoicePrice` 和 `invoiceFor` 方法會建立一個 Stripe 發票，該發票會重試失敗的計費嘗試。如果您不希望發票重試失敗的收費，則需要在第一次收費失敗後使用 Stripe API 關閉該發票。


<a name="creating-payment-intents"></a>
### 建立付款意向 (Payment Intent)

您可以透過在可計費模型實例上呼叫 `pay` 方法來建立一個新的 Stripe 付款意向。呼叫此方法將建立一個包裹在 `Laravel\Cashier\Payment` 實例中的付款意向：

```php
use Illuminate\Http\Request;

Route::post('/pay', function (Request $request) {
    $payment = $request->user()->pay(
        $request->get('amount')
    );

    return $payment->client_secret;
});
```

建立付款意向後，您可以將用戶端金鑰回傳到應用程式的前端，以便使用者在瀏覽器中完成付款。要了解更多關於使用 Stripe 付款意向建立完整付款流程的資訊，請參考 [Stripe 文件](https://stripe.com/docs/payments/accept-a-payment?platform=web)。

使用 `pay` 方法時，在您的 Stripe 控制面板中啟用的預設付款方式將提供給客戶使用。或者，如果您只想允許使用某些特定的付款方式，可以使用 `payWith` 方法：

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
> `pay` 和 `payWith` 方法接受以您應用程式所使用的貨幣最低面額計算的付款金額。例如，如果客戶以美元支付，金額應以分 (pennies) 為單位。


<a name="refunding-charges"></a>
### 退款

如果您需要退還 Stripe 收費，可以使用 `refund` 方法。此方法接受 Stripe [付款意向 ID](#payment-methods-for-single-charges) 作為其第一個參數：

```php
$payment = $user->charge(100, $paymentMethodId);

$user->refund($payment->id);
```

<a name="invoices"></a>
## 發票


<a name="retrieving-invoices"></a>
### 檢索發票

您可以使用 `invoices` 方法輕鬆地檢索可計費模型的發票陣列。`invoices` 方法會回傳一個 `Laravel\Cashier\Invoice` 實例的集合：

```php
$invoices = $user->invoices();
```

如果您想在結果中包含處理中 (Pending) 的發票，可以使用 `invoicesIncludingPending` 方法：

```php
$invoices = $user->invoicesIncludingPending();
```

您可以使用 `findInvoice` 方法透過 ID 檢索特定發票：

```php
$invoice = $user->findInvoice($invoiceId);
```


<a name="displaying-invoice-information"></a>
#### 顯示發票資訊

在為客戶列出發票時，您可以使用發票的方法來顯示相關的發票資訊。例如，您可能希望在表格中列出每張發票，讓使用者可以輕鬆地下載其中任何一張：

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

若要檢索客戶即將產生的發票，您可以使用 `upcomingInvoice` 方法：

```php
$invoice = $user->upcomingInvoice();
```

同樣地，如果客戶有多個訂閱，您也可以檢索特定訂閱即將產生的發票：

```php
$invoice = $user->subscription('default')->upcomingInvoice();
```


<a name="previewing-subscription-invoices"></a>
### 預覽訂閱發票

使用 `previewInvoice` 方法，您可以在變更價格之前預覽發票。這將使您能夠確定當進行特定的價格變更時，客戶的發票會是什麼樣子：

```php
$invoice = $user->subscription('default')->previewInvoice('price_yearly');
```

您可以將價格陣列傳遞給 `previewInvoice` 方法，以便預覽具有多個新價格的發票：

```php
$invoice = $user->subscription('default')->previewInvoice(['price_yearly', 'price_metered']);
```


<a name="generating-invoice-pdfs"></a>
### 生成發票 PDF

在生成發票 PDF 之前，您應該使用 Composer 安裝 Dompdf 函式庫，它是 Cashier 的預設發票渲染器：

```shell
composer require dompdf/dompdf
```

在路由或控制器中，您可以使用 `downloadInvoice` 方法來生成特定發票的 PDF 下載。此方法會自動生成下載發票所需的適當 HTTP 回應：

```php
use Illuminate\Http\Request;

Route::get('/user/invoice/{invoice}', function (Request $request, string $invoiceId) {
    return $request->user()->downloadInvoice($invoiceId);
});
```

預設情況下，發票上的所有資料都源自存儲在 Stripe 中的客戶和發票資料。檔名是基於您的 `app.name` 設定值。但是，您可以透過向 `downloadInvoice` 方法提供一個陣列作為第二個參數來編修其中一些資料。此陣列允許您自定義公司和產品詳細資訊等資訊：

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

`downloadInvoice` 方法還允許透過其第三個參數自定義檔名。此檔名將自動加上 `.pdf` 後綴：

```php
return $request->user()->downloadInvoice($invoiceId, [], 'my-invoice');
```


<a name="custom-invoice-render"></a>
#### 自定義發票渲染器

Cashier 還允許使用自定義的發票渲染器。預設情況下，Cashier 使用 `DompdfInvoiceRenderer` 實作，它利用 [dompdf](https://github.com/dompdf/dompdf) PHP 函式庫來生成 Cashier 的發票。但是，您可以透過實作 `Laravel\Cashier\Contracts\InvoiceRenderer` 介面來使用任何您想要的渲染器。例如，您可能希望使用對第三方 PDF 渲染服務的 API 呼叫來渲染發票 PDF：

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

一旦您實作了發票渲染器合約 (Contract)，您應該更新應用程式 `config/cashier.php` 設定檔中的 `cashier.invoices.renderer` 設定值。此設定值應設置為您自定義渲染器實作的類別名稱。

<a name="checkout"></a>
## Checkout

Cashier Stripe 也提供對 [Stripe Checkout](https://stripe.com/payments/checkout) 的支援。Stripe Checkout 透過提供預建且代管的付款頁面，省去了實作自定義頁面來接受付款的煩惱。

以下文件包含了如何開始在 Cashier 中使用 Stripe Checkout 的資訊。若要深入瞭解 Stripe Checkout，您也應該考慮閱讀 [Stripe 官方的 Checkout 文件](https://stripe.com/docs/payments/checkout)。


<a name="product-checkouts"></a>
### 產品結帳

您可以使用可計費模型上的 `checkout` 方法，針對已在 Stripe 儀表板中建立的現有產品進行結帳。`checkout` 方法將會啟動一個新的 Stripe Checkout 工作階段。預設情況下，您必須傳入一個 Stripe 價格 ID (Price ID)：

```php
use Illuminate\Http\Request;

Route::get('/product-checkout', function (Request $request) {
    return $request->user()->checkout('price_tshirt');
});
```

如果需要，您也可以指定產品數量：

```php
use Illuminate\Http\Request;

Route::get('/product-checkout', function (Request $request) {
    return $request->user()->checkout(['price_tshirt' => 15]);
});
```

當客戶造訪此路由時，他們將被重新導向到 Stripe 的 Checkout 頁面。預設情況下，當使用者成功完成或取消購買時，他們將被重新導向到您的 `home` 路由位置，但您可以使用 `success_url` 與 `cancel_url` 選項指定自定義的回呼 URL：

```php
use Illuminate\Http\Request;

Route::get('/product-checkout', function (Request $request) {
    return $request->user()->checkout(['price_tshirt' => 1], [
        'success_url' => route('your-success-route'),
        'cancel_url' => route('your-cancel-route'),
    ]);
});
```

在定義您的 `success_url` 結帳選項時，您可以指示 Stripe 在呼叫您的 URL 時將結帳工作階段 ID 作為查詢字串參數加入。為此，請將字串字面量 `{CHECKOUT_SESSION_ID}` 加入到您的 `success_url` 查詢字串中。Stripe 會將此佔位符替換為實際的結帳工作階段 ID：

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

預設情況下，Stripe Checkout 不允許使用者兌換促銷代碼。幸運的是，有一種簡單的方法可以為您的 Checkout 頁面啟用此功能。為此，您可以呼叫 `allowPromotionCodes` 方法：

```php
use Illuminate\Http\Request;

Route::get('/product-checkout', function (Request $request) {
    return $request->user()
        ->allowPromotionCodes()
        ->checkout('price_tshirt');
});
```


<a name="single-charge-checkouts"></a>
### 單次收費結帳

您也可以針對未在 Stripe 儀表板中建立的臨時產品執行簡單的扣款。為此，您可以使用可計費模型上的 `checkoutCharge` 方法，並傳遞扣款金額、產品名稱以及選填的數量。當客戶造訪此路由時，他們將被重新導向到 Stripe 的 Checkout 頁面：

```php
use Illuminate\Http\Request;

Route::get('/charge-checkout', function (Request $request) {
    return $request->user()->checkoutCharge(1200, 'T-Shirt', 5);
});
```

> [!WARNING]
> 當使用 `checkoutCharge` 方法時，Stripe 將始終在您的 Stripe 儀表板中建立新的產品與價格。因此，我們建議您預先在 Stripe 儀表板中建立產品，並改用 `checkout` 方法。


<a name="subscription-checkouts"></a>
### 訂閱結帳

> [!WARNING]
> 在訂閱中使用 Stripe Checkout 需要您在 Stripe 儀表板中啟用 `customer.subscription.created` webhook。此 webhook 會在您的資料庫中建立訂閱紀錄，並儲存所有相關的訂閱項目。

您也可以使用 Stripe Checkout 來啟動訂閱。在使用 Cashier 的訂閱建立器方法定義好您的訂閱後，您可以呼叫 `checkout `方法。當客戶造訪此路由時，他們將被重新導向到 Stripe 的 Checkout 頁面：

```php
use Illuminate\Http\Request;

Route::get('/subscription-checkout', function (Request $request) {
    return $request->user()
        ->newSubscription('default', 'price_monthly')
        ->checkout();
});
```

就像產品結帳一樣，您可以自定義成功與取消的 URL：

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

當然，您也可以為訂閱結帳啟用促銷代碼：

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
> 遺憾的是，Stripe Checkout 在啟動訂閱時不支援所有的訂閱帳務選項。在 Stripe Checkout 工作階段期間，使用訂閱建立器上的 `anchorBillingCycleOn` 方法、設定比例分配行為 (Proration Behavior) 或設定付款行為將不會產生任何效果。請參閱 [Stripe Checkout 工作階段 API 文件](https://stripe.com/docs/api/checkout/sessions/create) 以查看有哪些可用的參數。


<a name="stripe-checkout-trial-periods"></a>
#### Stripe Checkout 與試用期

當然，在建立將使用 Stripe Checkout 完成的訂閱時，您可以定義試用期：

```php
$checkout = Auth::user()->newSubscription('default', 'price_monthly')
    ->trialDays(3)
    ->checkout();
```

然而，試用期必須至少為 48 小時，這是 Stripe Checkout 所支援的最低試用時間。


<a name="stripe-checkout-subscriptions-and-webhooks"></a>
#### 訂閱與 Webhooks

請記住，Stripe 與 Cashier 是透過 webhooks 來更新訂閱狀態，因此當客戶在輸入付款資訊後返回應用程式時，訂閱可能尚未啟用。為了處理這種情況，您可能希望顯示一條訊息，告知使用者其付款或訂閱正在處理中。


<a name="collecting-tax-ids"></a>
### 收集稅務 ID

Checkout 也支援收集客戶的稅務 ID。若要在結帳工作階段中啟用此功能，請在建立工作階段時呼叫 `collectTaxIds` 方法：

```php
$checkout = $user->collectTaxIds()->checkout('price_tshirt');
```

當呼叫此方法時，客戶將看到一個新的勾選框，允許他們指出是否以公司身份進行購買。如果是，他們將有機會提供其稅務 ID 號碼。

> [!WARNING]
> 如果您已經在應用程式的服務提供者中設定了 [自動稅務設定](#tax-configuration)，則此功能將自動啟用，不需要呼叫 `collectTaxIds` 方法。

<a name="guest-checkouts"></a>
### 訪客結帳

使用 `Checkout::guest` 方法，您可以為應用程式中沒有「帳號」的訪客啟動結帳工作階段：

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

與為現有使用者建立結帳工作階段類似，您可以利用 `Laravel\Cashier\CheckoutBuilder` 實例上提供的其他方法來對訪客結帳工作階段進行自定義：

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

在訪客結帳完成後，Stripe 會發送一個 `checkout.session.completed` 的 webhook 事件，因此請務必[設定您的 Stripe webhook](https://dashboard.stripe.com/webhooks) 以確實將此事件發送到您的應用程式。在 Stripe 控制面板中啟用 webhook 後，您可以[使用 Cashier 處理 webhook](#handling-stripe-webhooks)。Webhook 負載中包含的物件將是一個 [checkout 物件](https://stripe.com/docs/api/checkout/sessions/object)，您可以檢查該物件以完成客戶的訂單。

<a name="handling-failed-payments"></a>
## 處理失敗的付款

有時，訂閱或單次收費的付款可能會失敗。當這種情況發生時，Cashier 會拋出一個 `Laravel\Cashier\Exceptions\IncompletePayment` 例外，告知您發生了這種情況。擷取到此例外後，您有兩個處理選項。

首先，您可以將客戶重新導向到 Cashier 內建的專用付款確認頁面。此頁面已經有一個關聯的命名路由，該路由是透過 Cashier 的服務提供者註冊的。因此，您可以擷取 `IncompletePayment` 例外並將使用者重新導向到付款確認頁面：

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

在付款確認頁面上，系統將提示客戶再次輸入其信用卡資訊，並執行 Stripe 要求的任何額外操作，例如 「3D Secure」 確認。確認付款後，使用者將被重新導向到上述 `redirect` 參數提供的 URL。重新導向時，URL 將會被加上 `message` (string) 和 `success` (integer) 查詢字串變數。該付款頁面目前支援以下付款方式類型：

<div class="content-list" markdown="1">

- 信用卡
- 支付寶 (Alipay)
- Bancontact
- BECS 直接扣款
- EPS
- Giropay
- iDEAL
- SEPA 直接扣款

</div>

或者，您可以讓 Stripe 為您處理付款確認。在這種情況下，您可以在 Stripe 控制面板中[設定 Stripe 的自動帳單電子郵件](https://dashboard.stripe.com/account/billing/automatic)，而不是重新導向到付款確認頁面。但是，如果擷取到 `IncompletePayment` 例外，您仍然應該告知使用者他們將收到一封包含進一步付款確認說明的電子郵件。

使用 `Billable` trait 的模型，在呼叫以下方法時可能會拋出付款例外：`charge`、`invoiceFor` 和 `invoice`。在處理訂閱時，`SubscriptionBuilder` 上的 `create` 方法，以及 `Subscription` 和 `SubscriptionItem` 模型上的 `incrementAndInvoice` 與 `swapAndInvoice` 方法也可能會拋出未完成付款例外。

可以使用可計費模型或訂閱實例上的 `hasIncompletePayment` 方法來判斷現有訂閱是否具有未完成的付款：

```php
if ($user->hasIncompletePayment('default')) {
    // ...
}

if ($user->subscription('default')->hasIncompletePayment()) {
    // ...
}
```

您可以透過檢查例外實例上的 `payment` 屬性來取得未完成付款的具體狀態：

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
### 確認付款

某些付款方式需要額外的資料才能確認付款。例如，SEPA 付款方式在付款過程中需要額外的 「授權 (mandate)」 資料。您可以使用 `withPaymentConfirmationOptions` 方法將此資料提供給 Cashier：

```php
$subscription->withPaymentConfirmationOptions([
    'mandate_data' => '...',
])->swap('price_xxx');
```

您可以參考 [Stripe API 文件](https://stripe.com/docs/api/payment_intents/confirm)以查看確認付款時接受的所有選項。


<a name="strong-customer-authentication"></a>
## 強客戶驗證 (SCA)

如果您的企業或您的客戶位於歐洲，您將需要遵守歐盟的強客戶驗證 (SCA) 法規。這些法規由歐盟於 2019 年 9 月強制執行，旨在防止付款詐欺。幸運的是，Stripe 和 Cashier 已經為建構符合 SCA 標準的應用程式做好了準備。

> [!WARNING]
> 在開始之前，請閱讀 [Stripe 關於 PSD2 與 SCA 的指南](https://stripe.com/guides/strong-customer-authentication) 以及他們關於[新 SCA API 的文件](https://stripe.com/docs/strong-customer-authentication)。


<a name="payments-requiring-additional-confirmation"></a>
### 需要額外確認的付款

SCA 法規通常要求進行額外的驗證才能確認並處理付款。當這種情況發生時，Cashier 會拋出一個 `Laravel\Cashier\Exceptions\IncompletePayment` 例外，告知您需要額外的驗證。有關如何處理這些例外的更多資訊，可以在[處理失敗的付款](#handling-failed-payments)的文件中找到。

由 Stripe 或 Cashier 呈現的付款確認畫面可能會針對特定銀行或發卡機構的付款流程進行調整，並可能包括額外的卡片確認、臨時小額扣款、獨立裝置驗證或其他形式的驗證。


<a name="incomplete-and-past-due-state"></a>
#### 未完成與過期狀態

當付款需要額外確認時，訂閱將保持在 `incomplete` 或 `past_due` 狀態，如其 `stripe_status` 資料庫欄位所示。一旦付款確認完成，且您的應用程式透過 Webhook 收到來自 Stripe 的完成通知，Cashier 就會自動啟用客戶的訂閱。

有關 `incomplete` 和 `past_due` 狀態的更多資訊，請參閱[我們關於這些狀態的額外文件](#incomplete-and-past-due-status)。


<a name="off-session-payment-notifications"></a>
### 非工作階段付款通知

由於 SCA 法規要求客戶即使在訂閱處於活動狀態時也偶爾需要驗證其付款細節，因此當需要進行非工作階段付款確認時，Cashier 可以向客戶發送通知。例如，這可能發生在訂閱續訂時。可以透過將 `CASHIER_PAYMENT_NOTIFICATION` 環境變數設定為一個通知類別來啟用 Cashier 的付款通知。預設情況下，此通知是停用的。當然，Cashier 包含了一個您可以為此目的使用的通知類別，但如果您願意，也可以提供自定義的通知類別：

```ini
CASHIER_PAYMENT_NOTIFICATION=Laravel\Cashier\Notifications\ConfirmPayment
```

為了確保發送非工作階段付款確認通知，請確認您的應用程式已[設定 Stripe webhooks](#handling-stripe-webhooks)，並且在您的 Stripe 控制面板中啟用了 `invoice.payment_action_required` Webhook。此外，您的 `Billable` 模型也應該使用 Laravel 的 `Illuminate\Notifications\Notifiable` trait。

> [!WARNING]
> 即使客戶手動進行需要額外確認的付款，也會發送通知。遺憾的是，Stripe 無法得知付款是手動完成還是「非工作階段」完成的。但是，如果客戶在確認付款後訪問付款頁面，他們只會看到「付款成功」的訊息。系統將不允許客戶意外地對同一筆付款進行兩次確認，進而導致意外的二次收費。

<a name="stripe-sdk"></a>
## Stripe SDK

Cashier 的許多物件都是 Stripe SDK 物件的封裝 (Wrappers)。如果您想直接與 Stripe 物件互動，可以使用 `asStripe` 方法方便地檢索它們：

```php
$stripeSubscription = $subscription->asStripeSubscription();

$stripeSubscription->application_fee_percent = 5;

$stripeSubscription->save();
```

您也可以使用 `updateStripeSubscription` 方法來直接更新一個 Stripe 訂閱：

```php
$subscription->updateStripeSubscription(['application_fee_percent' => 5]);
```

如果您想直接使用 `Stripe\StripeClient` 客戶端，可以在 `Cashier` 類別上調用 `stripe` 方法。例如，您可以使用此方法存取 `StripeClient` 實例並從您的 Stripe 帳號中檢索價格列表：

```php
use Laravel\Cashier\Cashier;

$prices = Cashier::stripe()->prices->all();
```


<a name="testing"></a>
## 測試

在測試使用 Cashier 的應用程式時，您可以模擬 (Mock) 發送到 Stripe API 的實際 HTTP 請求；然而，這需要您部分地重新實作 Cashier 自身的行為。因此，我們建議允許您的測試存取實際的 Stripe API。雖然這比較慢，但它能提供更多信心，確保您的應用程式運作符合預期，且任何較慢的測試都可以放在其專屬的 Pest / PHPUnit 測試群組中。

測試時，請記住 Cashier 本身已經有一套完善的測試套件，因此您應該專注於測試應用程式自身的訂閱和付款流程，而不是測試每個底層的 Cashier 行為。

首先，將 **測試 (Testing)** 版本的 Stripe 密鑰新增到您的 `phpunit.xml` 檔案中：

```xml
<env name="STRIPE_SECRET" value="sk_test_<your-key>"/>
```

現在，在測試時每當您與 Cashier 互動，它都會發送實際的 API 請求到您的 Stripe 測試環境。為了方便起見，您應該在您的 Stripe 測試帳號中預先填入測試期間可能使用的訂閱或價格。

> [!NOTE]
> 為了測試各種帳務情境，例如信用卡拒付和失敗，您可以使用 Stripe 提供的多種 [測試卡號與代碼 (Tokens)](https://stripe.com/docs/testing)。