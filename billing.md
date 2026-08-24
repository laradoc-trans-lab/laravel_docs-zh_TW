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
    - [稅號](#tax-ids)
    - [與 Stripe 同步客戶資料](#syncing-customer-data-with-stripe)
    - [帳務門戶](#billing-portal)
- [付款方式](#payment-methods)
    - [儲存付款方式](#storing-payment-methods)
    - [取得付款方式](#retrieving-payment-methods)
    - [檢查付款方式是否存在](#payment-method-presence)
    - [更新預設付款方式](#updating-the-default-payment-method)
    - [新增付款方式](#adding-payment-methods)
    - [刪除付款方式](#deleting-payment-methods)
- [訂閱](#subscriptions)
    - [建立訂閱](#creating-subscriptions)
    - [檢查訂閱狀態](#checking-subscription-status)
    - [變更價格](#changing-prices)
    - [訂閱數量](#subscription-quantity)
    - [包含多種產品的訂閱](#subscriptions-with-multiple-products)
    - [多重訂閱](#multiple-subscriptions)
    - [按用量計費](#usage-based-billing)
    - [訂閱稅金](#subscription-taxes)
    - [訂閱基準日期](#subscription-anchor-date)
    - [取消訂閱](#cancelling-subscriptions)
    - [恢復訂閱](#resuming-subscriptions)
- [訂閱試用](#subscription-trials)
    - [預先提供付款方式](#with-payment-method-up-front)
    - [不預先提供付款方式](#without-payment-method-up-front)
    - [延長試用期](#extending-trials)
- [處理 Stripe Webhook](#handling-stripe-webhooks)
    - [定義 Webhook 事件處理常式](#defining-webhook-event-handlers)
    - [驗證 Webhook 簽章](#verifying-webhook-signatures)
- [單次扣款](#single-charges)
    - [簡單扣款](#simple-charge)
    - [附帶發票扣款](#charge-with-invoice)
    - [建立付款意圖 (Payment Intents)](#creating-payment-intents)
    - [扣款退款](#refunding-charges)
- [發票](#invoices)
    - [取得發票](#retrieving-invoices)
    - [待出帳發票](#upcoming-invoices)
    - [預覽訂閱發票](#previewing-subscription-invoices)
    - [產生發票 PDF](#generating-invoice-pdfs)
- [結帳 (Checkout)](#checkout)
    - [產品結帳](#product-checkouts)
    - [單次扣款結帳](#single-charge-checkouts)
    - [訂閱結帳](#subscription-checkouts)
    - [收集稅號](#collecting-tax-ids)
    - [訪客結帳](#guest-checkouts)
- [處理失敗的付款](#handling-failed-payments)
    - [確認付款](#confirming-payments)
- [強效客戶認證 (SCA)](#strong-customer-authentication)
    - [需要額外確認的付款](#payments-requiring-additional-confirmation)
    - [非即時會話付款通知 (Off-session)](#off-session-payment-notifications)
- [Stripe SDK](#stripe-sdk)
- [測試](#testing)

<a name="introduction"></a>
## 簡介

[Laravel Cashier Stripe](https://github.com/laravel/cashier-stripe) 為 [Stripe](https://stripe.com) 的訂閱帳務服務提供了一個表達力強且順暢的介面。它處理了幾乎所有您害怕撰寫的訂閱帳務樣板程式碼。除了基本的訂閱管理外，Cashier 還可以處理優惠券、更換訂閱、訂閱「數量」、取消寬限期，甚至能產生發票 PDF。


<a name="upgrading-cashier"></a>
## 升級 Cashier

當升級到新版本的 Cashier 時，請務必仔細審閱[升級指南](https://github.com/laravel/cashier-stripe/blob/16.x/UPGRADE.md)。

> [!WARNING]
> 為防止重大變更（breaking changes），Cashier 使用固定的 Stripe API 版本。Cashier 16 利用 Stripe API 版本 `2025-06-30.basil`。Stripe API 版本會在次要版本更新時更新，以便利用新的 Stripe 功能和改進。


<a name="installation"></a>
## 安裝

首先，使用 Composer 套件管理器安裝適用於 Stripe 的 Cashier 套件：

```shell
composer require laravel/cashier
```

安裝套件後，使用 `vendor:publish` Artisan 指令發布 Cashier 的資料庫遷移檔（migrations）：

```shell
php artisan vendor:publish --tag="cashier-migrations"
```

接著，執行資料庫遷移：

```shell
php artisan migrate
```

Cashier 的遷移檔將會為您的 `users` 資料表新增數個欄位。它們還會建立一個新的 `subscriptions` 資料表來儲存您客戶的所有訂閱，以及一個用於多價格訂閱的 `subscription_items` 資料表。

如果您希望，也可以使用 `vendor:publish` Artisan 指令發布 Cashier 的設定檔：

```shell
php artisan vendor:publish --tag="cashier-config"
```

最後，為確保 Cashier 能正確處理所有 Stripe 事件，請記得[設定 Cashier 的 Webhook 處理](#handling-stripe-webhooks)。

> [!WARNING]
> Stripe 建議任何用於儲存 Stripe 識別碼的欄位都應該區分大小寫。因此，在使用 MySQL 時，您應該確保 `stripe_id` 欄位的定序（collation）設定為 `utf8_bin`。更多相關資訊可以在 [Stripe 文件](https://stripe.com/docs/upgrades#what-changes-does-stripe-consider-to-be-backwards-compatible)中找到。


<a name="configuration"></a>
## 設定


<a name="billable-model"></a>
### 可計費模型

在使用 Cashier 之前，請先將 `Billable` trait 新增到您的可計費模型定義中。通常這會是 `App\Models\User` 模型。此 trait 提供了各種方法，讓您執行常見的帳務任務，例如建立訂閱、套用優惠券以及更新付款方式資訊：

```php
use Laravel\Cashier\Billable;

class User extends Authenticatable
{
    use Billable;
}
```

Cashier 假設您的可計費模型是 Laravel 隨附的 `App\Models\User` 類別。如果您希望變更此模型，可以透過 `useCustomerModel` 方法指定不同的模型。此方法通常應在 `AppServiceProvider` 類別的 `boot` 方法中呼叫：

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
> 如果您使用的模型不是 Laravel 提供的 `App\Models\User` 模型，您需要發布並修改所提供的 [Cashier 遷移檔](#installation)，以符合您替代模型的資料表名稱。


<a name="api-keys"></a>
### API 金鑰

接下來，您應該在應用程式的 `.env` 檔案中設定 Stripe API 金鑰。您可以從 Stripe 控制台取得您的 Stripe API 金鑰：

```ini
STRIPE_KEY=your-stripe-key
STRIPE_SECRET=your-stripe-secret
STRIPE_WEBHOOK_SECRET=your-stripe-webhook-secret
```

> [!WARNING]
> 您應該確保在應用程式的 `.env` 檔案中定義了 `STRIPE_WEBHOOK_SECRET` 環境變數，因為此變數用於確保傳入的 Webhook 確實來自 Stripe。


<a name="currency-configuration"></a>
### 貨幣設定

Cashier 的預設貨幣為美元 (USD)。您可以透過在應用程式的 `.env` 檔案中設定 `CASHIER_CURRENCY` 環境變數來變更預設貨幣：

```ini
CASHIER_CURRENCY=eur
```

除了設定 Cashier 的貨幣外，您還可以指定在發票上顯示金額數值時所使用的語系（locale）。在內部，Cashier 利用 [PHP 的 `NumberFormatter` 類別](https://www.php.net/manual/en/class.numberformatter.php)來設定貨幣語系：

```ini
CASHIER_CURRENCY_LOCALE=nl_BE
```

> [!WARNING]
> 若要使用 `en` 以外的語系，請確保您的伺服器上已安裝並設定 `ext-intl` PHP 擴充套件。


<a name="tax-configuration"></a>
### 稅務設定

感謝 [Stripe Tax](https://stripe.com/tax)，現在可以自動為 Stripe 產生的所有發票計算稅金。您可以在應用程式的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫 `calculateTaxes` 方法來啟用自動稅金計算：

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

一旦啟用了稅金計算，任何新的訂閱和產生的單次發票都將獲得自動稅金計算。

為了使此功能正常運作，您客戶的帳務詳細資訊（例如客戶的姓名、地址和稅號）需要同步到 Stripe。您可以使用 Cashier 提供的[客戶資料同步](#syncing-customer-data-with-stripe)和[稅號](#tax-ids)方法來達成此目的。


<a name="logging"></a>
### 日誌記錄

Cashier 允許您指定在記錄 Stripe 嚴重錯誤（fatal errors）時所使用的日誌管道（log channel）。您可以透過在應用程式的 `.env` 檔案中定義 `CASHIER_LOGGER` 環境變數來指定日誌管道：

```ini
CASHIER_LOGGER=stack
```

由對 Stripe API 呼叫所產生的例外狀況（exceptions）將透過您應用程式的預設日誌管道進行記錄。


<a name="using-custom-models"></a>
### 使用自訂模型

您可以透過定義您自己的模型並繼承對應的 Cashier 模型，自由地擴充 Cashier 內部使用的模型：

```php
use Laravel\Cashier\Subscription as CashierSubscription;

class Subscription extends CashierSubscription
{
    // ...
}
```

定義模型後，您可以透過 `Laravel\Cashier\Cashier` 類別指示 Cashier 使用您的自訂模型。通常，您應該在應用程式的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中告知 Cashier 關於您的自訂模型：

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
> 在使用 Stripe Checkout 之前，您應該先在 Stripe 儀表板中定義具有固定價格的產品。此外，您還應該[設定 Cashier 的 Webhook 處理](#handling-stripe-webhooks)。

透過您的應用程式提供產品與訂閱帳務服務可能會讓人望而生畏。然而，多虧了 Cashier 與 [Stripe Checkout](https://stripe.com/payments/checkout)，您可以輕鬆建置現代且強大的付款整合。

若要針對非週期性、單次扣款的產品向客戶收款，我們將利用 Cashier 引導客戶至 Stripe Checkout，讓他們在那裡提供付款詳細資訊並確認購買。經由 Checkout 完成付款後，客戶將被重定向到您在應用程式中選擇的成功 URL：

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

如上方的範例所示，我們將利用 Cashier 提供的 `checkout` 方法，將客戶重定向至指定「價格識別碼」的 Stripe Checkout。使用 Stripe 時，「價格」指的是[特定產品所定義的價格](https://stripe.com/docs/products-prices/how-products-and-prices-work)。

如有必要，`checkout` 方法將會自動在 Stripe 中建立客戶，並將該 Stripe 客戶紀錄連結至您應用程式資料庫中的對應使用者。完成結帳會話 (Checkout session) 後，客戶將被重定向到專屬的成功或取消頁面，您可以在該頁面中向客戶顯示訊息說明。

<a name="providing-meta-data-to-stripe-checkout"></a>
#### 向 Stripe Checkout 提供 Metadata

銷售產品時，通常會透過您應用程式自訂的 `Cart` 與 `Order` 模型來追蹤已完成的訂單和購買的產品。當重定向客戶至 Stripe Checkout 以完成購買時，您可能需要提供現有的訂單標識符，以便在客戶被重定向回您的應用程式時，能夠將已完成的購買與對應的訂單關聯起來。

為了達成此目的，您可以將 `metadata` 陣列傳遞給 `checkout` 方法。讓我們假設當使用者開始結帳流程時，會在我們的應用程式中建立一個待處理的 `Order`。請記住，此範例中的 `Cart` 和 `Order` 模型僅供說明使用，並非由 Cashier 所提供。您可以根據自身應用程式的需求自由實現這些概念：

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

如上方的範例所示，當使用者開始結帳流程時，我們將提供所有與購物車 / 訂單關聯的 Stripe 價格標識符給 `checkout` 方法。當然，當客戶新增這些項目時，您的應用程式有責任將它們與「購物車」或訂單進行關聯。我們還透過 `metadata` 陣列將訂單的 ID 提供給 Stripe Checkout 會話。最後，我們將 `CHECKOUT_SESSION_ID` 範本變數新增至 Checkout 成功路由。當 Stripe 將客戶重定向回您的應用程式時，該範本變數將自動填入 Checkout 會話 ID。

接下來，讓我們建立 Checkout 成功路由。使用者透過 Stripe Checkout 完成購買後，將會被重定向至此路由。在此路由中，我們可以取得 Stripe Checkout 會話 ID 以及相關聯的 Stripe Checkout 執行個體，以存取我們提供的 Metadata 並據此更新客戶的訂單：

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

如需了解更多關於 [Checkout 會話物件所包含的資料](https://stripe.com/docs/api/checkout/sessions/object) 的詳細資訊，請參閱 Stripe 的官方文件。

<a name="quickstart-selling-subscriptions"></a>
### 銷售訂閱

> [!NOTE]
> 在使用 Stripe Checkout 之前，您應該先在 Stripe 儀表板中定義具有固定價格的產品。此外，您還應該[設定 Cashier 的 Webhook 處理](#handling-stripe-webhooks)。

在應用程式中提供產品和訂閱計費功能可能會令人心生畏懼。然而，多虧了 Cashier 與 [Stripe Checkout](https://stripe.com/payments/checkout)，您可以輕鬆建立現代且強健的付款整合。

為了了解如何使用 Cashier 和 Stripe Checkout 來銷售訂閱，讓我們考慮一個簡單的場景：一個擁有基礎月繳 (`price_basic_monthly`) 與年繳 (`price_basic_yearly`) 方案的訂閱服務。這兩個價格可以歸類在 Stripe 儀表板中的 "Basic" 產品 (`pro_basic`) 之下。此外，我們的訂閱服務可能還會提供名為 `pro_expert` 的專家方案。

首先，讓我們了解客戶如何訂閱我們的服務。當然，您可以想像客戶可能會在我們應用程式的價格頁面上點擊 Basic 方案的「訂閱」按鈕。該按鈕或連結應該將使用者引導至一個 Laravel 路由，該路由會為他們選擇的方案建立 Stripe Checkout 會話：

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

如上方的範例所示，我們將客戶重導向至 Stripe Checkout 會話，這將允許他們訂閱我們的 Basic 方案。在成功結帳或取消後，客戶將被重導向回我們提供給 `checkout` 方法的 URL。為了得知他們的訂閱何時真正開始（因為某些付款方式需要幾秒鐘來處理），我們還需要[設定 Cashier 的 Webhook 處理](#handling-stripe-webhooks)。

現在客戶可以開始訂閱了，我們需要限制應用程式的某些部分，以便只有已訂閱的使用者才能存取。當然，我們隨時可以透過 Cashier 的 `Billable` trait 所提供的 `subscribed` 方法來判斷使用者的當前訂閱狀態：

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
#### 建立檢查訂閱的中介層

為了方便起見，您可能希望建立一個[中介層](/docs/{{version}}/middleware)，用來判斷傳入的請求是否來自已訂閱的使用者。一旦定義了這個中介層，您可以輕鬆地將其指派給某個路由，以防止未訂閱的使用者存取該路由：

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

定義好中介層之後，您可以將其指派給路由：

```php
use App\Http\Middleware\Subscribed;

Route::get('/dashboard', function () {
    // ...
})->middleware([Subscribed::class]);
```


<a name="quickstart-allowing-customers-to-manage-their-billing-plan"></a>
#### 允許客戶管理其帳務方案

當然，客戶可能會想將其訂閱方案變更為其他產品或「層級 (tier)」。實現此需求最簡單的方式是將客戶引導至 Stripe 的[客戶帳務門戶 (Customer Billing Portal)](https://stripe.com/docs/no-code/customer-portal)，它提供了一個託管的使用者介面，允許客戶下載發票、更新付款方式以及變更訂閱方案。

首先，在應用程式中定義一個連結或按鈕，將使用者引導至我們用來發起帳務門戶會話的 Laravel 路由：

```blade
<a href="{{ route('billing') }}">
    Billing
</a>
```

接下來，讓我們定義一個發起 Stripe 客戶帳務門戶會話並將使用者重導向至門戶的路由。`redirectToBillingPortal` 方法接受使用者在退出門戶時應該返回的 URL：

```php
use Illuminate\Http\Request;

Route::get('/billing', function (Request $request) {
    return $request->user()->redirectToBillingPortal(route('dashboard'));
})->middleware(['auth'])->name('billing');
```

> [!NOTE]
> 只要您設定了 Cashier 的 Webhook 處理，Cashier 就會透過檢查來自 Stripe 的傳入 Webhook，自動保持應用程式中與 Cashier 相關的資料庫資料表同步。因此，舉例來說，當使用者透過 Stripe 的客戶帳務門戶取消訂閱時，Cashier 將接收到相應的 Webhook，並在您的應用程式資料庫中將該訂閱標示為「已取消」。

<a name="customers"></a>
## 客戶

<a name="retrieving-customers"></a>
### 取得客戶

您可以使用 `Cashier::findBillable` 方法，透過 Stripe ID 取得客戶。該方法將會傳回可計費模型的實例：

```php
use Laravel\Cashier\Cashier;

$user = Cashier::findBillable($stripeId);
```

<a name="creating-customers"></a>
### 建立客戶

有時候，您可能希望在不開始訂閱的情況下建立 Stripe 客戶。您可以使用 `createAsStripeCustomer` 方法來達成此目的：

```php
$stripeCustomer = $user->createAsStripeCustomer();
```

客戶在 Stripe 中建立完成後，您可以在日後隨時開始訂閱。您可以傳入一個可選的 `$options` 陣列，以帶入任何 [Stripe API 所支援額外的客戶建立參數](https://stripe.com/docs/api/customers/create)：

```php
$stripeCustomer = $user->createAsStripeCustomer($options);
```

如果您想取得可計費模型對應的 Stripe 客戶物件，可以使用 `asStripeCustomer` 方法：

```php
$stripeCustomer = $user->asStripeCustomer();
```

若您想取得特定可計費模型的 Stripe 客戶物件，但不確定該可計費模型是否已是 Stripe 中的客戶，可以使用 `createOrGetStripeCustomer` 方法。如果客戶尚不存在，此方法會在 Stripe 中建立一個新客戶：

```php
$stripeCustomer = $user->createOrGetStripeCustomer();
```

<a name="updating-customers"></a>
### 更新客戶

有時候，您可能希望直接使用額外資訊來更新 Stripe 客戶。您可以使用 `updateStripeCustomer` 方法來達成此目的。該方法接受一個 [Stripe API 所支援的客戶更新選項](https://stripe.com/docs/api/customers/update) 陣列：

```php
$stripeCustomer = $user->updateStripeCustomer($options);
```

<a name="balances"></a>
### 餘額

Stripe 允許您對客戶的「餘額」進行儲值（Credit）或扣款（Debit）。日後，此餘額將會在產生新發票時進行折抵或加收。若要檢查客戶的總餘額，您可以使用可計費模型上的 `balance` 方法。`balance` 方法會傳回格式化後的字串，代表以客戶貨幣顯示的餘額：

```php
$balance = $user->balance();
```

若要為客戶的餘額儲值，您可以向 `creditBalance` 方法提供一個數值。如果需要，您也可以提供說明：

```php
$user->creditBalance(500, 'Premium customer top-up.');
```

向 `debitBalance` 方法提供數值則會扣減客戶的餘額：

```php
$user->debitBalance(300, 'Bad usage penalty.');
```

`applyBalance` 方法會為客戶建立新的客戶餘額交易。您可以使用 `balanceTransactions` 方法來取得這些交易紀錄，這對於提供儲值與扣款紀錄供客戶查閱非常有用：

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

Cashier 提供了一種管理客戶稅號（Tax ID）的簡單方法。例如，可以使用 `taxIds` 方法以集合（Collection）形式取得指定給客戶的所有 [稅號](https://stripe.com/docs/api/customer_tax_ids/object)：

```php
$taxIds = $user->taxIds();
```

您也可以透過識別碼取得客戶的特定稅號：

```php
$taxId = $user->findTaxId('txi_belgium');
```

您可以透過向 `createTaxId` 方法提供有效的 [類型 (Type)](https://stripe.com/docs/api/customer_tax_ids/object#tax_id_object-type) 與數值來建立新的稅號：

```php
$taxId = $user->createTaxId('eu_vat', 'BE0123456789');
```

`createTaxId` 方法會立即將加值稅號（VAT ID）新增到客戶的帳號中。[VAT ID 的驗證也是由 Stripe 進行](https://stripe.com/docs/invoicing/customer/tax-ids#validation)；然而，這是一個非同步的過程。您可以透過訂閱 `customer.tax_id.updated` webhook 事件並檢視 [VAT ID 的 `verification` 參數](https://stripe.com/docs/api/customer_tax_ids/object#tax_id_object-verification) 來接收驗證更新通知。關於處理 webhook 的更多資訊，請參考 [定義 Webhook 處理常式的文件](#handling-stripe-webhooks)。

您可以使用 `deleteTaxId` 方法刪除稅號：

```php
$user->deleteTaxId('txi_belgium');
```

<a name="syncing-customer-data-with-stripe"></a>
### 與 Stripe 同步客戶資料

通常，當您應用程式的使用者更新其姓名、電子郵件地址或其他同樣儲存在 Stripe 中的資訊時，您應該通知 Stripe 這些更新。這樣一來，Stripe 中的資訊副本就能與您應用程式中的資訊保持同步。

為了使此流程自動化，您可以在可計費模型上定義一個事件監聽器（Event listener），回應模型的 `updated` 事件。然後在事件監聽器內，對模型呼叫 `syncStripeCustomerDetails` 方法：

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

現在，每當您的客戶模型被更新時，其資訊都會同步至 Stripe。為了便利起見，Cashier 會在首次建立客戶時自動將客戶資訊與 Stripe 同步。

您可以透過覆寫 Cashier 提供的一系列方法，來自訂用於同步客戶資訊至 Stripe 的欄位。例如，您可以覆寫 `stripeName` 方法，以自訂當 Cashier 同步客戶資訊至 Stripe 時，哪一個屬性應該被視為客戶的「姓名」：

```php
/**
 * Get the customer name that should be synced to Stripe.
 */
public function stripeName(): string|null
{
    return $this->company_name;
}
```

同樣地，您也可以覆寫 `stripeEmail`、`stripePhone`（最多 20 個字元）、`stripeAddress` 與 `stripePreferredLocales` 方法。在 [更新 Stripe 客戶物件](https://stripe.com/docs/api/customers/update) 時，這些方法會將資訊同步至對應的客戶參數。如果您希望全面掌控客戶資訊的同步流程，可以覆寫 `syncStripeCustomerDetails` 方法。

<a name="billing-portal"></a>
### 帳務門戶

Stripe 提供了 [設定帳務門戶 (Billing portal) 的簡單方法](https://stripe.com/docs/billing/subscriptions/customer-portal)，讓您的客戶可以管理其訂閱、付款方式並檢視帳務歷史紀錄。您可以從控制器或路由中呼叫可計費模型上的 `redirectToBillingPortal` 方法，將使用者重導向至帳務門戶：

```php
use Illuminate\Http\Request;

Route::get('/billing-portal', function (Request $request) {
    return $request->user()->redirectToBillingPortal();
});
```

預設情況下，當使用者完成訂閱管理後，他們可以透過 Stripe 帳務門戶內的連結返回應用程式的 `home` 路由。您可以將自訂的返回 URL 作為引數傳給 `redirectToBillingPortal` 方法：

```php
use Illuminate\Http\Request;

Route::get('/billing-portal', function (Request $request) {
    return $request->user()->redirectToBillingPortal(route('billing'));
});
```

如果您想產生帳務門戶的 URL 而不產生 HTTP 重導向回應，可以呼叫 `billingPortalUrl` 方法：

```php
$url = $request->user()->billingPortalUrl(route('billing'));
```

<a name="payment-methods"></a>
## 付款方式


<a name="storing-payment-methods"></a>
### 儲存付款方式

為了建立訂閱或使用 Stripe 進行「單次」扣款，您的應用程式需要安全地收集客戶的付款詳細資料。完成此操作的方法會根據您計劃儲存付款方式以供未來訂閱使用，還是立即處理單次扣款而有所不同，因此我們將在下方探討這兩種情況。

Stripe 的 [Payment Element](https://stripe.com/docs/payments/payment-element) 可用於支援多種付款方式，例如信用卡/金融卡、Apple Pay、Google Pay 和 iDEAL。


<a name="payment-element-for-subscriptions"></a>
#### 用於訂閱的 Payment Element

首先，建立一個 Setup Intent 並將其傳遞給您的視圖 (View)：

```php
return view('subscribe', [
    'intent' => $user->createSetupIntent()
]);
```

使用 Setup Intent 的 `client_secret` 掛載 Payment Element：

```html
<div id="payment-element"></div>
<button id="submit">Subscribe</button>

<script src="https://js.stripe.com/v3/"></script>
<script>
    const stripe = Stripe('stripe-public-key');

    const elements = stripe.elements({
        clientSecret: '{{ $intent->client_secret }}'
    });

    const paymentElement = elements.create('payment');

    paymentElement.mount('#payment-element');

    document.getElementById('submit').addEventListener('click', async () => {
        const { error } = await stripe.confirmSetup({
            elements,
            confirmParams: {
                return_url: '{{ route("subscription.complete") }}',
            },
        });

        if (error) {
            // Display "error.message" to the user...
        }
    });
</script>
```

在 Stripe 重導向至您的 `return_url` 後，`setup_intent` ID 將會作為網址查詢參數 (Query String Parameter) 提供。您可以使用此值來取得付款方式並建立訂閱：

```php
use Illuminate\Http\Request;

Route::get('/subscription/complete', function (Request $request) {
    $setupIntent = $request->user()->findSetupIntent(
        $request->setup_intent
    );

    $paymentMethod = $setupIntent->payment_method;

    $request->user()
        ->newSubscription('default', 'price_xxx')
        ->create($paymentMethod);

    return redirect('/dashboard');
})->name('subscription.complete');
```

如果您使用 Payment Element 來更新客戶的預設付款方式而非建立訂閱，您可以將付款方式識別碼傳遞給 [`updateDefaultPaymentMethod`](#updating-the-default-payment-method) 方法。


<a name="payment-element-for-single-charges"></a>
#### 用於單次扣款的 Payment Element

對於單次付款，請使用 Cashier 的 `pay` 方法建立 Payment Intent。通常，您應該將 Payment Intent ID 儲存在應用程式對應的訂單中，以便在 Stripe 將客戶重導向回您的應用程式後可以檢索該訂單。以下範例假設您的應用程式有一個包含 `user_id`、`amount`、`status` 和 `stripe_payment_intent_id` 欄位的 `Order` 模型：

```php
use App\Models\Order;
use Illuminate\Http\Request;

Route::post('/pay', function (Request $request) {
    $amount = 1000;

    $payment = $request->user()->pay($amount);

    $order = Order::create([
        'user_id' => $request->user()->id,
        'amount' => $amount,
        'status' => 'pending',
        'stripe_payment_intent_id' => $payment->id,
    ]);

    return view('checkout', [
        'clientSecret' => $payment->client_secret,
        'order' => $order,
    ]);
});
```

然後，掛載 Payment Element 並確認付款：

```html
<div id="payment-element"></div>
<button id="submit">Pay Now</button>

<script src="https://js.stripe.com/v3/"></script>
<script>
    const stripe = Stripe('stripe-public-key');

    const elements = stripe.elements({
        clientSecret: '{{ $clientSecret }}'
    });

    const paymentElement = elements.create('payment');

    paymentElement.mount('#payment-element');

    document.getElementById('submit').addEventListener('click', async () => {
        const { error } = await stripe.confirmPayment({
            elements,
            confirmParams: {
                return_url: '{{ route("payment.complete") }}',
            },
        });

        if (error) {
            // Display "error.message" to the user...
        }
    });
</script>
```

重導向後，您可以使用 `payment_intent` 網址查詢參數來取得對應的訂單和 Payment Intent。在履行訂單之前，您應該驗證該訂單屬於已驗證的客戶，並且 Payment Intent 屬於已驗證的客戶且已成功付款：

```php
use App\Models\Order;
use Illuminate\Http\Request;

Route::get('/payment/complete', function (Request $request) {
    $order = Order::where('user_id', $request->user()->id)
        ->where('stripe_payment_intent_id', $request->payment_intent)
        ->firstOrFail();

    $paymentIntent = $request->user()
        ->stripe()
        ->paymentIntents
        ->retrieve($request->payment_intent);

    if ($paymentIntent->customer === $request->user()->stripe_id &&
        $paymentIntent->status === 'succeeded') {
        $order->update(['status' => 'paid']);

        // Fulfill the order...
    }

    return redirect('/dashboard');
})->name('payment.complete');
```


<a name="retrieving-payment-methods"></a>
### 取得付款方式

可計費模型實例上的 `paymentMethods` 方法會回傳一個 `Laravel\Cashier\PaymentMethod` 實例的集合：

```php
$paymentMethods = $user->paymentMethods();
```

預設情況下，此方法將回傳所有類型的付款方式。若要取得特定類型的付款方式，您可以將 `type` 作為引數傳遞給該方法：

```php
$paymentMethods = $user->paymentMethods('sepa_debit');
```

若要取得客戶的預設付款方式，可以使用 `defaultPaymentMethod` 方法：

```php
$paymentMethod = $user->defaultPaymentMethod();
```

您可以使用 `findPaymentMethod` 方法取得附加至可計費模型的特定付款方式：

```php
$paymentMethod = $user->findPaymentMethod($paymentMethodId);
```


<a name="payment-method-presence"></a>
### 檢查付款方式是否存在

若要確定可計費模型的帳號中是否已附加預設付款方式，可呼叫 `hasDefaultPaymentMethod` 方法：

```php
if ($user->hasDefaultPaymentMethod()) {
    // ...
}
```

您可以使用 `hasPaymentMethod` 方法來確定可計費模型的帳號中是否至少附加了一種付款方式：

```php
if ($user->hasPaymentMethod()) {
    // ...
}
```

此方法將確定可計費模型是否擁有任何付款方式。若要確定該模型是否存在特定類型的付款方式，您可以將 `type` 作為引數傳遞給該方法：

```php
if ($user->hasPaymentMethod('sepa_debit')) {
    // ...
}
```


<a name="updating-the-default-payment-method"></a>
### 更新預設付款方式

`updateDefaultPaymentMethod` 方法可以用於更新客戶的預設付款方式資訊。此方法接受一個 Stripe 付款方式識別碼，並將新的付款方式設定為預設的帳務付款方式：

```php
$user->updateDefaultPaymentMethod($paymentMethod);
```

若要將您的預設付款方式資訊與客戶在 Stripe 中的預設付款方式資訊進行同步，可以使用 `updateDefaultPaymentMethodFromStripe` 方法：

```php
$user->updateDefaultPaymentMethodFromStripe();
```

> [!WARNING]
> 客戶的預設付款方式僅能用於開立發票和建立新訂閱。由於 Stripe 的限制，它不能用於單次扣款。

<a name="adding-payment-methods"></a>
### 新增付款方式

若要新增付款方式，您可以在可計費模型上呼叫 `addPaymentMethod` 方法，並傳入付款方式識別碼：

```php
$user->addPaymentMethod($paymentMethod);
```

> [!NOTE]
> 若要瞭解如何取得付款方式識別碼，請參閱[儲存付款方式文件](#storing-payment-methods)。

<a name="deleting-payment-methods"></a>
### 刪除付款方式

若要刪除付款方式，您可以對想要刪除的 `Laravel\Cashier\PaymentMethod` 執行個體呼叫 `delete` 方法：

```php
$paymentMethod->delete();
```

`deletePaymentMethod` 方法會從可計費模型中刪除特定的付款方式：

```php
$user->deletePaymentMethod('pm_visa');
```

`deletePaymentMethods` 方法會刪除該可計費模型的所有付款方式資訊：

```php
$user->deletePaymentMethods();
```

預設情況下，此方法會刪除所有類型的付款方式。若要刪除特定類型的付款方式，您可以將 `type` 作為引數傳入該方法：

```php
$user->deletePaymentMethods('sepa_debit');
```

> [!WARNING]
> 如果使用者有正在進行中的有效訂閱，您的應用程式不應允許他們刪除其預設付款方式。

<a name="subscriptions"></a>
## 訂閱

訂閱功能為您提供了一種為客戶設定定期付款的方法。由 Cashier 管理的 Stripe 訂閱支援多種訂閱價格、訂閱數量、試用期等功能。


<a name="creating-subscriptions"></a>
### 建立訂閱

若要建立訂閱，首先請取得可計費模型的實例，這通常會是 `App\Models\User` 的實例。取得模型實例後，您可以使用 `newSubscription` 方法來建立該模型的訂閱：

```php
use Illuminate\Http\Request;

Route::post('/user/subscribe', function (Request $request) {
    $request->user()->newSubscription(
        'default', 'price_monthly'
    )->create($request->paymentMethodId);

    // ...
});
```

傳遞給 `newSubscription` 方法的第一個引數應為訂閱的內部類型。如果您的應用程式僅提供單一訂閱，您可以將其命名為 `default` 或 `primary`。此訂閱類型僅供應用程式內部使用，並非用於展示給使用者。此外，它不應包含空格，且在建立訂閱後絕不應該更改。第二個引數是使用者所訂閱的特定價格，此值應對應至 Stripe 中的價格識別碼。

`create` 方法接受 [Stripe 付款方式識別碼](#storing-payment-methods) 或 Stripe `PaymentMethod` 物件，它將開始該訂閱，並使用可計費模型的 Stripe 客戶 ID 及其他相關帳務資訊更新您的資料庫。

> [!WARNING]
> 直接將付款方式識別碼傳遞給 `create` 訂閱方法，也會自動將其新增至使用者已儲存的付款方式中。


<a name="collecting-recurring-payments-via-invoice-emails"></a>
#### 透過發票 Email 收取定期款項

除了自動收取客戶的定期款項外，您也可以指示 Stripe 在每次定期付款到期時，透過 Email 發送發票給客戶。然後，客戶可以在收到發票後手動付款。透過發票收取定期款項時，客戶不需要預先提供付款方式：

```php
$user->newSubscription('default', 'price_monthly')->createAndSendInvoice();
```

客戶在訂閱被取消前支付發票的時間長短由 `days_until_due` 選項決定。預設為 30 天；但是，如果您希望的話，可以為此選項提供特定的數值：

```php
$user->newSubscription('default', 'price_monthly')->createAndSendInvoice([], [
    'days_until_due' => 30
]);
```


<a name="subscription-quantities"></a>
#### 數量

如果您想在建立訂閱時為價格設定特定的[數量](https://stripe.com/docs/billing/subscriptions/quantities)，可以在建立訂閱之前，在訂閱建構器上呼叫 `quantity` 方法：

```php
$user->newSubscription('default', 'price_monthly')
    ->quantity(5)
    ->create($paymentMethod);
```


<a name="additional-details"></a>
#### 額外詳細資訊

如果您想指定 Stripe 所支援的其他[客戶](https://stripe.com/docs/api/customers/create)或[訂閱](https://stripe.com/docs/api/subscriptions/create)選項，可以將它們作為第二個和第三個引數傳遞給 `create` 方法：

```php
$user->newSubscription('default', 'price_monthly')->create($paymentMethod, [
    'email' => $email,
], [
    'metadata' => ['note' => 'Some extra information.'],
]);
```


<a name="coupons"></a>
#### 優惠券

如果您想在建立訂閱時套用優惠券，可以使用 `withCoupon` 方法：

```php
$user->newSubscription('default', 'price_monthly')
    ->withCoupon('code')
    ->create($paymentMethod);
```

或者，如果您想套用 [Stripe 促銷代碼](https://stripe.com/docs/billing/subscriptions/discounts/codes)，可以使用 `withPromotionCode` 方法：

```php
$user->newSubscription('default', 'price_monthly')
    ->withPromotionCode('promo_code_id')
    ->create($paymentMethod);
```

給定的促銷代碼 ID 應該是分配給促銷代碼的 Stripe API ID，而非面向客戶的促銷代碼。如果您需要根據給定的面向客戶促銷代碼查找促銷代碼 ID，可以使用 `findPromotionCode` 方法：

```php
// Find a promotion code ID by its customer facing code...
$promotionCode = $user->findPromotionCode('SUMMERSALE');

// Find an active promotion code ID by its customer facing code...
$promotionCode = $user->findActivePromotionCode('SUMMERSALE');
```

在上述範例中，傳回的 `$promotionCode` 物件是 `Laravel\Cashier\PromotionCode` 的實例。此類別封裝了底層的 `Stripe\PromotionCode` 物件。您可以透過呼叫 `coupon` 方法來取得與促銷代碼相關的優惠券：

```php
$coupon = $user->findPromotionCode('SUMMERSALE')->coupon();
```

優惠券實例允許您確定折扣金額，以及優惠券代表的是固定金額折扣還是基於百分比的折扣：

```php
if ($coupon->isPercentage()) {
    return $coupon->percentOff().'%'; // 21.5%
} else {
    return $coupon->amountOff(); // $5.99
}
```

您還可以取得目前套用到客戶或訂閱的折扣：

```php
$discount = $billable->discount();

$discount = $subscription->discount();
```

傳回的 `Laravel\Cashier\Discount` 實例封裝了底層的 `Stripe\Discount` 物件實例。您可以透過呼叫 `coupon` 方法來取得與此折扣相關的優惠券：

```php
$coupon = $subscription->discount()->coupon();
```

如果您想向客戶或訂閱套用新的優惠券或促銷代碼，可以透過 `applyCoupon` 或 `applyPromotionCode` 方法來實現：

```php
$billable->applyCoupon('coupon_id');
$billable->applyPromotionCode('promotion_code_id');

$subscription->applyCoupon('coupon_id');
$subscription->applyPromotionCode('promotion_code_id');
```

請記住，您應該使用分配給促銷代碼的 Stripe API ID，而非面向客戶的促銷代碼。同一時間內只能將一張優惠券或一個促銷代碼套用到客戶或訂閱上。

有關此主題的更多資訊，請參閱 Stripe 關於[優惠券](https://stripe.com/docs/billing/subscriptions/coupons)和[促銷代碼](https://stripe.com/docs/billing/subscriptions/coupons/codes)的官方文件。


<a name="adding-subscriptions"></a>
#### 新增訂閱

如果您想為已經有預設付款方式的客戶新增訂閱，可以在訂閱建構器上呼叫 `add` 方法：

```php
use App\Models\User;

$user = User::find(1);

$user->newSubscription('default', 'price_monthly')->add();
```


<a name="creating-subscriptions-from-the-stripe-dashboard"></a>
#### 從 Stripe 控制台建立訂閱

您也可以直接從 Stripe 控制台建立訂閱。這樣做時，Cashier 將同步新新增的訂閱，並將其類型分配為 `default`。若要自訂分配給從控制台建立的訂閱類型，請[定義 Webhook 事件處理常式](#defining-webhook-event-handlers)。

此外，您只能透過 Stripe 控制台建立一種類型的訂閱。如果您的應用程式提供使用不同類型的多個訂閱，則只能透過 Stripe 控制台新增一種類型的訂閱。

最後，您應該始終確保您應用程式提供的每種訂閱類型僅新增一個活躍訂閱。如果客戶擁有兩個 `default` 訂閱，即使兩者都會與您應用程式的資料庫同步，Cashier 也只會使用最新新增的訂閱。

<a name="checking-subscription-status"></a>
### 檢查訂閱狀態

當客戶訂閱了您的應用程式後，您可以使用各種方便的方法輕鬆檢查其訂閱狀態。首先，如果客戶擁有有效的訂閱（即使訂閱目前正處於試用期內），`subscribed` 方法會回傳 `true`。`subscribed` 方法的第一個引數接受訂閱的類型：

```php
if ($user->subscribed('default')) {
    // ...
}
```

`subscribed` 方法也非常適合用於 [路由中介層](/docs/{{version}}/middleware)，讓您能夠根據使用者的訂閱狀態來過濾路由與控制器 (Controller) 的存取權限：

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

如果您想確認使用者是否仍處於試用期內，可以使用 `onTrial` 方法。這個方法有助於決定是否要向使用者顯示其仍處於試用期的警告訊息：

```php
if ($user->subscription('default')->onTrial()) {
    // ...
}
```

`subscribedToProduct` 方法可用於根據指定的 Stripe 產品識別碼，來判斷使用者是否訂閱了該產品。在 Stripe 中，產品是價格的集合。在這個範例中，我們將判斷使用者的 `default` 訂閱是否正有效訂閱了應用程式的「premium」產品。傳入的 Stripe 產品識別碼應該要對應到您 Stripe 控制面板中的其中一個產品識別碼：

```php
if ($user->subscribedToProduct('prod_premium', 'default')) {
    // ...
}
```

透過傳送陣列給 `subscribedToProduct` 方法，您可以判斷使用者的 `default` 訂閱是否正有效訂閱了應用程式的「basic」或「premium」產品：

```php
if ($user->subscribedToProduct(['prod_basic', 'prod_premium'], 'default')) {
    // ...
}
```

`subscribedToPrice` 方法可用於判斷客戶的訂閱是否對應到指定的價格 ID：

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
> 如果使用者擁有兩個相同類型的訂閱，`subscription` 方法將永遠回傳最新建立的訂閱。例如，使用者可能擁有兩個類型為 `default` 的訂閱紀錄；然而，其中一個訂閱可能是舊的且已過期的訂閱，而另一個則是目前有效的訂閱。系統將永遠回傳最新建立的訂閱，而較舊的訂閱則會保留在資料庫中供歷史查閱。

<a name="cancelled-subscription-status"></a>
#### 取消的訂閱狀態

若要判斷使用者是否曾經是有效訂閱者但已取消訂閱，您可以使用 `canceled` 方法：

```php
if ($user->subscription('default')->canceled()) {
    // ...
}
```

您也可以判斷使用者是否已取消訂閱，但在訂閱完全到期之前仍處於「寬限期 (Grace period)」。例如，如果使用者在 3 月 5 日取消了原定於 3 月 10 日到期的訂閱，則該使用者在 3 月 10 日之前都處於「寬限期」。請注意，在這段期間內 `subscribed` 方法仍會回傳 `true`：

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

如果訂閱在建立後需要二次付款驗證動作，該訂閱將會被標記為 `incomplete`。訂閱狀態會儲存在 Cashier 的 `subscriptions` 資料庫表格的 `stripe_status` 欄位中。

同樣地，如果在變更價格時需要二次付款動作，該訂閱將會被標記為 `past_due`。當您的訂閱處於這兩種狀態之一時，在客戶確認付款之前，該訂閱都不會處於啟用狀態。可以使用可計費模型或訂閱實例上的 `hasIncompletePayment` 方法來判斷訂閱是否有未完成的付款：

```php
if ($user->hasIncompletePayment('default')) {
    // ...
}

if ($user->subscription('default')->hasIncompletePayment()) {
    // ...
}
```

當訂閱有未完成的付款時，您應該引導使用者前往 Cashier 的付款確認頁面，並傳入 `latestPayment` 識別碼。您可以使用訂閱實例上的 `latestPayment` 方法來取得此識別碼：

```html
<a href="{{ route('cashier.payment', $subscription->latestPayment()->id) }}">
    Please confirm your payment.
</a>
```

如果您希望訂閱在處於 `past_due` 或 `incomplete` 狀態時仍被視為啟用狀態，您可以使用 Cashier 提供的 `keepPastDueSubscriptionsActive` 和 `keepIncompleteSubscriptionsActive` 方法。通常，這些方法應該在 `App\Providers\AppServiceProvider` 的 `register` 方法中呼叫：

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
> 當訂閱處於 `incomplete` 狀態時，在確認付款之前無法進行修改。因此，當訂閱處於 `incomplete` 狀態時，`swap` 和 `updateQuantity` 方法將會拋出例外 (Exception)。

<a name="subscription-scopes"></a>
#### 訂閱查詢範圍 (Scopes)

大多數訂閱狀態也提供作為查詢範圍 (Query scopes)，以便您輕鬆地查詢資料庫中處於特定狀態的訂閱：

```php
// Get all active subscriptions...
$subscriptions = Subscription::query()->active()->get();

// Get all of the canceled subscriptions for a user...
$subscriptions = $user->subscriptions()->canceled()->get();
```

以下是所有可用查詢範圍的完整清單：

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

當客戶訂閱了您的應用程式後，他們偶爾可能會想變更為新的訂閱價格。要將客戶切換至新價格，請將 Stripe 的價格識別碼傳入 `swap` 方法。切換價格時，若該訂閱先前已被取消，系統會預設使用者想要重新啟用他們的訂閱。傳入的價格識別碼應與 Stripe 主控台中可用的 Stripe 價格識別碼一致：

```php
use App\Models\User;

$user = App\Models\User::find(1);

$user->subscription('default')->swap('price_yearly');
```

若客戶正處於試用期，試用期將會予以保留。此外，若該訂閱存在「數量 (quantity)」，該數量也會繼續保持。

若您想在變更價格的同時取消客戶當前所有的試用期，可以呼叫 `skipTrial` 方法：

```php
$user->subscription('default')
    ->skipTrial()
    ->swap('price_yearly');
```

若您想變更價格並立即開立發票給客戶，而不是等待下一個計費週期，可以使用 `swapAndInvoice` 方法：

```php
$user = User::find(1);

$user->subscription('default')->swapAndInvoice('price_yearly');
```

<a name="prorations"></a>
#### 按比例計費

預設情況下，Stripe 在不同價格之間切換時會按比例計算費用。`noProrate` 方法可用於更新訂閱價格而不按比例計算費用：

```php
$user->subscription('default')->noProrate()->swap('price_yearly');
```

如需更多有關訂閱按比例計費的資訊，請參閱 [Stripe 文件](https://stripe.com/docs/billing/subscriptions/prorations)。

> [!WARNING]
> 在 `swapAndInvoice` 方法之前執行 `noProrate` 方法對按比例計費不會產生任何效果，系統仍會一律開立發票。

<a name="subscription-quantity"></a>
### 訂閱數量

有時候訂閱會受到「數量」的影響。例如，專案管理應用程式可能會針對每個專案每月收取 10 美元。您可以使用 `incrementQuantity` 與 `decrementQuantity` 方法輕鬆增加或減少您的訂閱數量：

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

`noProrate` 方法可用於更新訂閱數量而不按比例計算費用：

```php
$user->subscription('default')->noProrate()->updateQuantity(10);
```

如需更多有關訂閱數量的資訊，請參閱 [Stripe 文件](https://stripe.com/docs/subscriptions/quantities)。

<a name="quantities-for-subscription-with-multiple-products"></a>
#### 包含多種產品的訂閱數量

如果您的訂閱是[包含多種產品的訂閱](#subscriptions-with-multiple-products)，您應該將欲增加或減少數量的價格 ID 作為第二個引數傳給增加 / 減少方法：

```php
$user->subscription('default')->incrementQuantity(1, 'price_chat');
```

<a name="subscriptions-with-multiple-products"></a>
### 包含多種產品的訂閱

[包含多種產品的訂閱 (Subscription with multiple products)](https://stripe.com/docs/billing/subscriptions/multiple-products) 允許您將多個計費產品分配給單一訂閱。例如，想像您正在建立一個客服支援 (helpdesk) 應用程式，其基礎訂閱價格為每月 $10 美元，但提供每月額外 $15 美元的即時對談 (live chat) 擴充產品。多種產品訂閱的資訊儲存在 Cashier 的 `subscription_items` 資料庫表中。

您可以透過將價格陣列作為第二個引數傳遞給 `newSubscription` 方法，來為給定的訂閱指定多個產品：

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

在上方的範例中，客戶的 `default` 訂閱將附加兩個價格。這兩個價格都會在其各自的計費週期進行扣款。如有需要，您可以使用 `quantity` 方法來指定每個價格的特定數量：

```php
$user = User::find(1);

$user->newSubscription('default', ['price_monthly', 'price_chat'])
    ->quantity(5, 'price_chat')
    ->create($paymentMethod);
```

如果您想為現有的訂閱新增另一個價格，可以呼叫訂閱的 `addPrice` 方法：

```php
$user = User::find(1);

$user->subscription('default')->addPrice('price_chat');
```

上方的範例將會新增新的價格，並在客戶的下一個計費週期進行扣款。如果您想立即向客戶開立帳單扣款，可以使用 `addPriceAndInvoice` 方法：

```php
$user->subscription('default')->addPriceAndInvoice('price_chat');
```

如果您想新增具有特定數量的價格，可以將數量作為第二個引數傳遞給 `addPrice` 或 `addPriceAndInvoice` 方法：

```php
$user = User::find(1);

$user->subscription('default')->addPrice('price_chat', 5);
```

您可以透過 `removePrice` 方法從訂閱中移除價格：

```php
$user->subscription('default')->removePrice('price_chat');
```

> [!WARNING]
> 您不能移除訂閱中的最後一個價格。相反地，您應該直接取消該訂閱。


<a name="swapping-prices"></a>
#### 變更價格

您也可以變更附加到包含多種產品訂閱的價格。例如，想像客戶擁有附帶 `price_chat` 擴充產品的 `price_basic` 訂閱，而您想要將該客戶從 `price_basic` 價格升級到 `price_pro` 價格：

```php
use App\Models\User;

$user = User::find(1);

$user->subscription('default')->swap(['price_pro', 'price_chat']);
```

執行上方的範例時，帶有 `price_basic` 的底層訂閱項目會被刪除，而帶有 `price_chat` 的訂閱項目則會保留。此外，還會為 `price_pro` 建立一個全新的訂閱項目。

您也可以透過傳遞鍵 / 值對的陣列給 `swap` 方法來指定訂閱項目選項。例如，您可能需要指定訂閱價格的數量：

```php
$user = User::find(1);

$user->subscription('default')->swap([
    'price_pro' => ['quantity' => 5],
    'price_chat'
]);
```

如果您想更換訂閱上的單一價格，可以使用訂閱項目本身的 `swap` 方法。如果您想要保留訂閱其他價格上的所有現有中繼資料 (Metadata)，這個方法特別有用：

```php
$user = User::find(1);

$user->subscription('default')
    ->findItemOrFail('price_basic')
    ->swap('price_pro');
```


<a name="proration"></a>
#### 按比例計算 (Proration)

預設情況下，從包含多種產品的訂閱中新增或移除價格時，Stripe 會按比例計算費用。如果您想在不進行按比例計算的情況下調整價格，應該將 `noProrate` 方法鏈結到您的價格操作上：

```php
$user->subscription('default')->noProrate()->removePrice('price_chat');
```


<a name="swapping-quantities"></a>
#### 數量

如果您想更新個別訂閱價格的數量，可以使用[現有的數量方法](#subscription-quantity)，並將價格 ID 作為額外的引數傳遞給該方法：

```php
$user = User::find(1);

$user->subscription('default')->incrementQuantity(5, 'price_chat');

$user->subscription('default')->decrementQuantity(3, 'price_chat');

$user->subscription('default')->updateQuantity(10, 'price_chat');
```

> [!WARNING]
> 當訂閱擁有多個價格時，`Subscription` 模型上的 `stripe_price` 和 `quantity` 屬性將會為 `null`。若要存取個別價格的屬性，您應該使用 `Subscription` 模型上提供的 `items` 關聯。


<a name="subscription-items"></a>
#### 訂閱項目

當訂閱擁有多個價格時，會在資料庫的 `subscription_items` 表格中儲存多個訂閱「項目 (items)」。您可以透過訂閱上的 `items` 關聯來存取這些項目：

```php
use App\Models\User;

$user = User::find(1);

$subscriptionItem = $user->subscription('default')->items->first();

// Retrieve the Stripe price and quantity for a specific item...
$stripePrice = $subscriptionItem->stripe_price;
$quantity = $subscriptionItem->quantity;
```

您也可以使用 `findItemOrFail` 方法取得特定的價格：

```php
$user = User::find(1);

$subscriptionItem = $user->subscription('default')->findItemOrFail('price_chat');
```


<a name="multiple-subscriptions"></a>
### 多重訂閱

Stripe 允許您的客戶同時擁有額外的多個訂閱。例如，您可能經營一家健身房，提供游泳訂閱和重訓訂閱，並且每個訂閱可能會有不同的定價。當然，客戶應該能夠訂閱其中一種方案或同時訂閱這兩種方案。

當您的應用程式建立訂閱時，您可以將訂閱類型提供給 `newSubscription` 方法。類型可以是代表使用者發起的訂閱類型的任何字串：

```php
use Illuminate\Http\Request;

Route::post('/swimming/subscribe', function (Request $request) {
    $request->user()->newSubscription('swimming')
        ->price('price_swimming_monthly')
        ->create($request->paymentMethodId);

    // ...
});
```

在這個範例中，我們為客戶發起了按月計費的游泳訂閱。然而，他們之後可能會想更換為按年計費的訂閱。調整客戶的訂閱時，我們可以簡單地更換 `swimming` 訂閱上的價格：

```php
$user->subscription('swimming')->swap('price_swimming_yearly');
```

當然，您也可以完全取消該訂閱：

```php
$user->subscription('swimming')->cancel();
```

<a name="usage-based-billing"></a>
### 按用量計費

[按用量計費](https://stripe.com/docs/billing/subscriptions/metered-billing)允許您根據客戶在計費週期內對產品的使用量來向其收取費用。例如，您可以根據客戶每月發送的簡訊或電子郵件數量來向他們收費。

要開始使用按用量計費，您首先需要在 Stripe 控制面板中建立一個帶有[按用量計費模型](https://docs.stripe.com/billing/subscriptions/usage-based/implementation-guide)和[計量器 (Meter)](https://docs.stripe.com/billing/subscriptions/usage-based/recording-usage#configure-meter)的新產品。建立計量器後，請儲存相關聯的事件名稱與計量器 ID，回報和取得使用量時會需要它們。接著，使用 `meteredPrice` 方法將計量價格 ID 新增至客戶的訂閱中：

```php
use Illuminate\Http\Request;

Route::post('/user/subscribe', function (Request $request) {
    $request->user()->newSubscription('default')
        ->meteredPrice('price_metered')
        ->create($request->paymentMethodId);

    // ...
});
```

您也可以透過 [Stripe Checkout](#checkout) 來建立按用量計費的訂閱：

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

當您的客戶使用您的應用程式時，您需要將他們的使用量回報給 Stripe，以便精準計費。若要回報計量事件的使用量，您可以對 `Billable` 模型使用 `reportMeterEvent` 方法：

```php
$user = User::find(1);

$user->reportMeterEvent('emails-sent');
```

預設情況下，計費週期內會增加 1 個「使用數量」。或者，您可以傳入特定的「使用量」數值，將其新增至客戶在該計費週期的使用量中：

```php
$user = User::find(1);

$user->reportMeterEvent('emails-sent', quantity: 15);
```

若要取得客戶針對某個計量器的事件摘要，您可以使用 `Billable` 實例的 `meterEventSummaries` 方法：

```php
$user = User::find(1);

$meterUsage = $user->meterEventSummaries($meterId);

$meterUsage->first()->aggregated_value // 10
```

關於計量器事件摘要的更多資訊，請參考 Stripe 的 [Meter Event Summary 物件文件](https://docs.stripe.com/api/billing/meter-event_summary/object)。

若要[列出所有計量器](https://docs.stripe.com/api/billing/meter/list)，您可以呼叫 `Billable` 實例的 `meters` 方法：

```php
$user = User::find(1);

$user->meters();
```

<a name="subscription-taxes"></a>
### 訂閱稅金

> [!WARNING]
> 除了手動計算稅率，您也可以[使用 Stripe Tax 自動計算稅金](#tax-configuration)。

若要指定使用者在訂閱時支付的稅率，您應該在可計費模型上實作 `taxRates` 方法，並回傳一個包含 Stripe 稅率 ID 的陣列。您可以在 [Stripe 控制面板](https://dashboard.stripe.com/test/tax-rates) 中定義這些稅率：

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

`taxRates` 方法允許您針對不同客戶套用不同的稅率，這對於跨越多個國家和不同稅率的使用者群體非常有用。

如果您提供包含多種產品的訂閱，您可以透過在可計費模型上實作 `priceTaxRates` 方法，為每個價格定義不同的稅率：

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
> `taxRates` 方法僅適用於訂閱費用。如果您使用 Cashier 進行「單次」扣款，您將需要當時手動指定稅率。

<a name="syncing-tax-rates"></a>
#### 同步稅率

當變更由 `taxRates` 方法回傳的硬編碼稅率 ID 時，使用者現有訂閱上的稅金設定將保持不變。如果您希望使用新的 `taxRates` 數值更新現有訂閱的稅率，您應該在使用者的訂閱實例上呼叫 `syncTaxRates` 方法：

```php
$user->subscription('default')->syncTaxRates();
```

這也會同步包含多種產品訂閱的任何項目稅率。如果您的應用程式提供包含多種產品的訂閱，您應確保可計費模型實作了[上文討論的](#subscription-taxes) `priceTaxRates` 方法。

<a name="tax-exemption"></a>
#### 免稅

Cashier 還提供了 `isNotTaxExempt`、`isTaxExempt` 和 `reverseChargeApplies` 方法來確認客戶是否免稅。這些方法會呼叫 Stripe API 來確定客戶的免稅狀態：

```php
use App\Models\User;

$user = User::find(1);

$user->isTaxExempt();
$user->isNotTaxExempt();
$user->reverseChargeApplies();
```

> [!WARNING]
> 這些方法也可用於任何 `Laravel\Cashier\Invoice` 物件。然而，當對 `Invoice` 物件呼叫時，這些方法將判定發票建立當時的免稅狀態。

<a name="subscription-anchor-date"></a>
### 訂閱基準日期

預設情況下，計費週期基準是訂閱建立的日期，或者如果使用了試用期，則是試用期結束的日期。如果您想要修改計費基準日期，可以使用 `anchorBillingCycleOn` 方法：

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

關於管理訂閱計費週期的更多資訊，請參考 [Stripe 計費週期文件](https://stripe.com/docs/billing/subscriptions/billing-cycle)。

<a name="cancelling-subscriptions"></a>
### 取消訂閱

若要取消訂閱，請在使用者的訂閱上呼叫 `cancel` 方法：

```php
$user->subscription('default')->cancel();
```

當訂閱被取消時，Cashier 會自動設定資料庫中 `subscriptions` 資料表的 `ends_at` 欄位。此欄位用於了解 `subscribed` 方法何時該開始回傳 `false`。

例如，如果客戶在 3 月 1 日取消訂閱，但該訂閱原本預計到 3 月 5 日才結束，則 `subscribed` 方法會繼續回傳 `true` 直到 3 月 5 日為止。這樣做是因為使用者通常被允許繼續使用應用程式，直到其計費週期結束。

您可以使用 `onGracePeriod` 方法來確認使用者是否已取消訂閱但仍處於「寬限期」內：

```php
if ($user->subscription('default')->onGracePeriod()) {
    // ...
}
```

如果您希望立即取消訂閱，可以在使用者的訂閱上呼叫 `cancelNow` 方法：

```php
$user->subscription('default')->cancelNow();
```

如果您希望立即取消訂閱並針對任何剩餘尚未出帳的按用量計費使用量或新增/待處理的按比例計算發票項目進行開立發票，可以在使用者的訂閱上呼叫 `cancelNowAndInvoice` 方法：

```php
$user->subscription('default')->cancelNowAndInvoice();
```

您也可以選擇在特定時間點取消訂閱：

```php
$user->subscription('default')->cancelAt(
    now()->plus(days: 10)
);
```

最後，在刪除相關聯的使用者模型之前，您應該總是先取消使用者的訂閱：

```php
$user->subscription('default')->cancelNow();

$user->delete();
```

<a name="resuming-subscriptions"></a>
### 恢復訂閱

如果客戶取消了訂閱，而您希望恢復該訂閱，可以在訂閱實例上呼叫 `resume` 方法。客戶必須仍處於其「寬限期」內才能恢復訂閱：

```php
$user->subscription('default')->resume();
```

如果客戶取消了訂閱，並在訂閱完全過期前恢復該訂閱，客戶不會被立即扣款。相反地，他們的訂閱將會被重新啟用，並依照原本的計費週期進行扣款。

<a name="subscription-trials"></a>
## 訂閱試用

<a name="with-payment-method-up-front"></a>
### 預先提供付款方式

若您希望在預先收集付款方式資訊的同時向客戶提供試用期，您應該在建立訂閱時使用 `trialDays` 方法：

```php
use Illuminate\Http\Request;

Route::post('/user/subscribe', function (Request $request) {
    $request->user()->newSubscription('default', 'price_monthly')
        ->trialDays(10)
        ->create($request->paymentMethodId);

    // ...
});
```

此方法會在資料庫的訂閱紀錄中設定試用期結束日期，並指示 Stripe 在該日期之前不要開始向客戶請款。使用 `trialDays` 方法時，Cashier 會覆蓋在 Stripe 中為該價格設定的任何預設試用期。

> [!WARNING]
> 如果客戶的訂閱未在試用結束日期之前取消，他們將在試用期滿時立即被扣款，因此您務必通知使用者其試用結束日期。

`trialUntil` 方法允許您提供一個 `DateTime` 實例，用來指定試用期何時結束：

```php
use Illuminate\Support\Carbon;

$user->newSubscription('default', 'price_monthly')
    ->trialUntil(Carbon::now()->plus(days: 10))
    ->create($paymentMethod);
```

您可以使用使用者實例的 `onTrial` 方法，或是訂閱實例的 `onTrial` 方法來判斷使用者是否處於試用期內。以下兩個範例效果相同：

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

若要檢查現有的試用是否已到期，您可以使用 `hasExpiredTrial` 方法：

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

您可以選擇在 Stripe 控制面板中定義價格所獲得的試用天數，或是始終使用 Cashier 明確傳遞。如果您選擇在 Stripe 中定義價格的試用天數，您應該注意到新訂閱（包括過去曾有過訂閱記錄的客戶的新訂閱）將總是獲得試用期，除非您明確呼叫 `skipTrial()` 方法。

<a name="without-payment-method-up-front"></a>
### 不預先提供付款方式

若您希望在不預先收集使用者付款方式資訊的情況下提供試用期，您可以將使用者紀錄中的 `trial_ends_at` 欄位設定為您希望的試用結束日期。這通常會在使用者註冊時進行：

```php
use App\Models\User;

$user = User::create([
    // ...
    'trial_ends_at' => now()->plus(days: 10),
]);
```

> [!WARNING]
> 請務必在您的可計費模型類別定義中，為 `trial_ends_at` 屬性加上[日期型別轉換 (Date Cast)](/docs/{{version}}/eloquent-mutators#date-casting)。

Cashier 將這種類型的試用稱為「通用試用 (Generic trial)」，因為它不屬於任何現有的訂閱。若目前日期尚未超過 `trial_ends_at` 的值，則可計費模型實例上的 `onTrial` 方法將傳回 `true`：

```php
if ($user->onTrial()) {
    // User is within their trial period...
}
```

當您準備好為使用者建立實際訂閱時，您可以像往常一樣使用 `newSubscription` 方法：

```php
$user = User::find(1);

$user->newSubscription('default', 'price_monthly')->create($paymentMethod);
```

若要取得使用者的試用結束日期，您可以使用 `trialEndsAt` 方法。如果使用者處於試用期，此方法將傳回 Carbon 日期實例，否則傳回 `null`。若您想取得預設訂閱之外的其他特定訂閱試用結束日期，也可以傳入可選的訂閱類型參數：

```php
if ($user->onTrial()) {
    $trialEndsAt = $user->trialEndsAt('main');
}
```

若您想特別確認使用者是否處於其「通用」試用期內且尚未建立實際訂閱，您可以使用 `onGenericTrial` 方法：

```php
if ($user->onGenericTrial()) {
    // User is within their "generic" trial period...
}
```

<a name="extending-trials"></a>
### 延長試用期

`extendTrial` 方法允許您在建立訂閱後延長訂閱的試用期。如果試用期已經到期，且已經開始向客戶收取該訂閱的費用，您仍然可以為他們提供延長的試用期。在試用期內所消耗的時間將會從客戶的下一張發票中扣除：

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
## 處理 Stripe Webhook

> [!NOTE]
> 您可以使用 [Stripe CLI](https://stripe.com/docs/stripe-cli) 來協助在本機開發期間測試 Webhook。

Stripe 可以透過 Webhook 來將各種事件通知您的應用程式。預設情況下，Cashier 的服務提供者(Service Providers)會自動註冊指向 Cashier 的 Webhook 控制器的路由。該控制器將會處理所有傳入的 Webhook 請求。

預設情況下，Cashier 的 Webhook 控制器會自動處理因扣款失敗次數過多（根據您的 Stripe 設定所定義）而取消訂閱的情況、客戶資料更新、客戶刪除、訂閱更新以及付款方式變更；然而，正如我們即將看到的，您可以擴充此控制器來處理任何您想處理的 Stripe Webhook 事件。

為了確保您的應用程式可以處理 Stripe Webhook，請務必在 Stripe 控制面板中設定 Webhook URL。預設情況下，Cashier 的 Webhook 控制器會回應 `/stripe/webhook` URL 路徑。您應該在 Stripe 控制面板中啟用的所有 Webhook 完整列表如下：

- `customer.subscription.created`
- `customer.subscription.updated`
- `customer.subscription.deleted`
- `customer.updated`
- `customer.deleted`
- `payment_method.automatically_updated`
- `invoice.payment_action_required`
- `invoice.payment_succeeded`

為了方便起見，Cashier 包含了一個 `cashier:webhook` Artisan 指令。該指令會在 Stripe 中建立一個收聽 Cashier 所需的所有事件的 Webhook：

```shell
php artisan cashier:webhook
```

預設情況下，建立的 Webhook 會指向由 `APP_URL` 環境變數所定義的 URL 以及 Cashier 內建的 `cashier.webhook` 路由。若您想使用不同的 URL，可在執行指令時提供 `--url` 選項：

```shell
php artisan cashier:webhook --url "https://example.com/stripe/webhook"
```

建立的 Webhook 將會使用與您的 Cashier 版本相容的 Stripe API 版本。若您想使用不同的 Stripe 版本，可提供 `--api-version` 選項：

```shell
php artisan cashier:webhook --api-version="2019-12-03"
```

建立後，Webhook 將會立即生效。若您希望建立 Webhook 但在其準備好前先保持停用狀態，可以在執行指令時提供 `--disabled` 選項：

```shell
php artisan cashier:webhook --disabled
```

> [!WARNING]
> 請務必使用 Cashier 內建的 [Webhook 簽章驗證](#verifying-webhook-signatures)中介層來保護傳入的 Stripe Webhook 請求。


<a name="webhooks-csrf-protection"></a>
#### Webhook 與 CSRF 保護

由於 Stripe Webhook 需要繞過 Laravel 的 [CSRF 保護](/docs/{{version}}/csrf)，您應該確保 Laravel 不會嘗試針對傳入的 Stripe Webhook 驗證 CSRF Token。為達成此目的，您應該在應用程式的 `bootstrap/app.php` 檔案中將 `stripe/*` 排除在 CSRF 保護之外：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->preventRequestForgery(except: [
        'stripe/*',
    ]);
})
```


<a name="defining-webhook-event-handlers"></a>
### 定義 Webhook 事件處理常式

Cashier 會自動處理因扣款失敗導致的訂閱取消以及其他常見的 Stripe Webhook 事件。但是，若您有其他想要處理的 Webhook 事件，可以透過監聽由 Cashier 發送的以下事件來處理：

- `Laravel\Cashier\Events\WebhookReceived`
- `Laravel\Cashier\Events\WebhookHandled`

這兩個事件都包含 Stripe Webhook 的完整有效載荷 (Payload)。例如，若您希望處理 `invoice.payment_succeeded` Webhook，可以註冊一個會處理該事件的[監聽器](/docs/{{version}}/events#defining-listeners)：

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

為了確保您的 Webhook 安全，您可以使用 [Stripe 的 Webhook 簽章](https://stripe.com/docs/webhooks/signatures)。為方便起見，Cashier 自動包含了一個中介層，用於驗證傳入的 Stripe Webhook 請求是否有效。

若要啟用 Webhook 驗證，請確保您的應用程式 `.env` 檔案中設定了 `STRIPE_WEBHOOK_SECRET` 環境變數。Webhook 的 `secret` 可以從您的 Stripe 帳號控制面板中取得。

<a name="single-charges"></a>
## 單次扣款

<a name="simple-charge"></a>
### 簡單扣款

如果您想使用付款方式標識符 (identifier) 對客戶進行單次扣款，可以使用可計費模型實例上的 `charge` 方法。若您需要在處理單次扣款前向客戶收集付款詳細資訊，請參閱[單次扣款的 Payment Element](#payment-element-for-single-charges) 文件：

```php
use Illuminate\Http\Request;

Route::post('/purchase', function (Request $request) {
    $payment = $request->user()->charge(
        100, $request->paymentMethodId
    );

    // ...
});
```

`charge` 方法接受一個陣列作為其第三個引數，允許您傳遞任何想用於底層 Stripe Payment Intent 建立的選項。關於建立 Payment Intents 時可用的選項詳細資訊，可以在 [Stripe 文件](https://stripe.com/docs/api/payment_intents/create)中找到：

```php
$user->charge(100, $paymentMethod, [
    'custom_option' => $value,
]);
```

您也可以在沒有底層客戶或使用者的情況下使用 `charge` 方法。若要達成此目的，請在應用程式的可計費模型新實例上呼叫 `charge` 方法：

```php
use App\Models\User;

$payment = (new User)->charge(100, $paymentMethod);
```

如果扣款失敗，`charge` 方法將會拋出例外。如果扣款成功，該方法將會回傳一個 `Laravel\Cashier\Payment` 實例：

```php
try {
    $payment = $user->charge(100, $paymentMethod);
} catch (Exception $e) {
    // ...
}
```

> [!WARNING]
> `charge` 方法接受的付款金額為您應用程式所使用貨幣的最小單位。例如，如果客戶使用的是美金付款，金額應以美分（pennies）為單位來指定。

<a name="charge-with-invoice"></a>
### 附帶發票扣款

有時您可能需要進行單次扣款並向客戶提供 PDF 發票。`invoicePrice` 方法就能幫您做到這一點。例如，讓我們為客戶開立五件新襯衫的發票：

```php
$user->invoicePrice('price_tshirt', 5);
```

發票將立即從使用者的預設付款方式中扣款。`invoicePrice` 方法也接受一個陣列作為其第三個引數，此陣列包含了該發票項目的計費選項。該方法接受的第四個引數也是一個陣列，其中應包含發票本身的計費選項：

```php
$user->invoicePrice('price_tshirt', 5, [
    'discounts' => [
        ['coupon' => 'SUMMER21SALE']
    ],
], [
    'default_tax_rates' => ['txr_id'],
]);
```

與 `invoicePrice` 類似，您可以使用 `tabPrice` 方法將多個項目新增至客戶的「帳單 (tab)」中，接著再開立發票給客戶，以針對多個項目進行單次扣款（每張發票最多可達 250 個項目）。例如，我們可以為客戶開立五件襯衫和兩個馬克杯的發票：

```php
$user->tabPrice('price_tshirt', 5);
$user->tabPrice('price_mug', 2);
$user->invoice();
```

或者，您可以使用 `invoiceFor` 方法對使用者的預設付款方式進行「單次」扣款：

```php
$user->invoiceFor('One Time Fee', 500);
```

雖然可以使用 `invoiceFor` 方法，但建議您搭配預先定義的價格使用 `invoicePrice` 和 `tabPrice` 方法。這樣做能讓您在 Stripe 控制面板中獲得更好的分析與資料，進而瞭解個別產品的銷售狀況。

> [!WARNING]
> `invoice`、`invoicePrice` 和 `invoiceFor` 方法將會建立一個 Stripe 發票，並對失敗的扣款進行重試。如果您不希望發票重試失敗的扣款，需要在首次扣款失敗後使用 Stripe API 將其關閉。

<a name="creating-payment-intents"></a>
### 建立付款意圖 (Payment Intents)

您可以在可計費模型實例上呼叫 `pay` 方法來建立新的 Stripe 付款意圖 (payment intent)。呼叫此方法將會建立一個包裹在 `Laravel\Cashier\Payment` 實例中的付款意圖：

```php
use Illuminate\Http\Request;

Route::post('/pay', function (Request $request) {
    $payment = $request->user()->pay(
        $request->get('amount')
    );

    return $payment->client_secret;
});
```

建立付款意圖後，您可以將用戶端金鑰 (client secret) 回傳至您應用程式的前端，以便使用者可在其瀏覽器中完成付款。若要深入瞭解如何使用 Stripe 付款意圖建置完整的付款流程，請參閱 [Stripe 文件](https://stripe.com/docs/payments/accept-a-payment?platform=web)。

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
> `pay` 和 `payWith` 方法接受的付款金額為您應用程式所使用貨幣的最小單位。例如，如果客戶使用的是美金付款，金額應以美分（pennies）為單位來指定。

<a name="refunding-charges"></a>
### 扣款退款

如果您需要退款 Stripe 付款，可以使用 `refund` 方法。該方法接受 Stripe Payment Intent ID 作為其第一個引數：

```php
$payment = $user->charge(100, $paymentMethodId);

$user->refund($payment->id);
```

<a name="invoices"></a>
## 發票


<a name="retrieving-invoices"></a>
### 取得發票

您可以使用 `invoices` 方法輕鬆地取得可計費模型的發票陣列。`invoices` 方法會傳回一個 `Laravel\Cashier\Invoice` 實例的集合：

```php
$invoices = $user->invoices();
```

如果您希望在結果中包含待處理的發票，可以使用 `invoicesIncludingPending` 方法：

```php
$invoices = $user->invoicesIncludingPending();
```

您可以使用 `findInvoice` 方法透過發票 ID 取得特定的發票：

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
### 待出帳發票

若要取得客戶的下一張待出帳發票，您可以使用 `upcomingInvoice` 方法：

```php
$invoice = $user->upcomingInvoice();
```

同樣地，如果客戶有多個訂閱，您也可以取得特定訂閱的待出帳發票：

```php
$invoice = $user->subscription('default')->upcomingInvoice();
```


<a name="previewing-subscription-invoices"></a>
### 預覽訂閱發票

使用 `previewInvoice` 方法，您可以在進行價格變更前預覽發票。這能讓您瞭解當指定價格變更時，客戶的發票看起來會是什麼樣子：

```php
$invoice = $user->subscription('default')->previewInvoice('price_yearly');
```

您可以傳送一個價格陣列給 `previewInvoice` 方法，以預覽包含多個新價格的發票：

```php
$invoice = $user->subscription('default')->previewInvoice(['price_yearly', 'price_metered']);
```


<a name="generating-invoice-pdfs"></a>
### 產生發票 PDF

在產生發票 PDF 之前，您應該使用 Composer 安裝 Dompdf 函式庫，這是 Cashier 的預設發票渲染器：

```shell
composer require dompdf/dompdf
```

在路由或控制器中，您可以使用 `downloadInvoice` 方法來產生特定發票的 PDF 下載。此方法會自動產生下載發票所需的適當 HTTP 回應：

```php
use Illuminate\Http\Request;

Route::get('/user/invoice/{invoice}', function (Request $request, string $invoiceId) {
    return $request->user()->downloadInvoice($invoiceId);
});
```

預設情況下，發票上的所有資料都源自於儲存在 Stripe 中的客戶與發票資料。檔名則是基於您的 `app.name` 設定值。不過，您可以傳遞一個陣列作為 `downloadInvoice` 方法的第二個引數來自訂其中部分資料。這個陣列允許您自訂像是公司名稱與產品細節等資訊：

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

`downloadInvoice` 方法也允許透過第三個引數自訂檔名。此檔名會自動附加 `.pdf` 副檔名：

```php
return $request->user()->downloadInvoice($invoiceId, [], 'my-invoice');
```


<a name="custom-invoice-render"></a>
#### 自訂發票渲染器

Cashier 也允許使用自訂的發票渲染器。預設情況下，Cashier 使用 `DompdfInvoiceRenderer` 實作，它利用 [dompdf](https://github.com/dompdf/dompdf) PHP 函式庫來產生 Cashier 的發票。然而，您可以透過實作 `Laravel\Cashier\Contracts\InvoiceRenderer` 契約 (Contracts) 來使用任何您想要的渲染器。例如，您可能希望透過 API 呼叫第三方 PDF 渲染服務來渲染發票 PDF：

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

當您實作了發票渲染器契約後，您應該更新應用程式的 `config/cashier.php` 設定檔中的 `cashier.invoices.renderer` 設定值。這個設定值應該設定為您自訂渲染器實作類別的名稱。

<a name="checkout"></a>
## 結帳 (Checkout)

Cashier Stripe 也支援 [Stripe Checkout](https://stripe.com/payments/checkout)。Stripe Checkout 提供預先建置好的託管付款頁面，讓你省去自行實作客製化付款頁面的麻煩。

以下說明文件包含如何在 Cashier 中開始使用 Stripe Checkout 的相關資訊。若要瞭解更多有關 Stripe Checkout 的詳細資訊，建議你也可以參閱 [Stripe 官方的 Checkout 說明文件](https://stripe.com/docs/payments/checkout)。


<a name="product-checkouts"></a>
### 產品結帳

你可以在可計費模型上使用 `checkout` 方法，為已在 Stripe 控制面板中建立的現有產品進行結帳。`checkout` 方法會啟動一個新的 Stripe Checkout 會話 (Session)。預設情況下，你必須傳入一個 Stripe 的價格 ID：

```php
use Illuminate\Http\Request;

Route::get('/product-checkout', function (Request $request) {
    return $request->user()->checkout('price_tshirt');
});
```

如果有需要，你也可以指定產品數量：

```php
use Illuminate\Http\Request;

Route::get('/product-checkout', function (Request $request) {
    return $request->user()->checkout(['price_tshirt' => 15]);
});
```

當客戶造訪此路由時，會被重新導向至 Stripe 的 Checkout 頁面。預設情況下，當使用者成功完成購買或取消購買時，他們會被重新導向至你的 `home` 路由位置，但你可以使用 `success_url` 與 `cancel_url` 選項來指定自訂的回呼 URL：

```php
use Illuminate\Http\Request;

Route::get('/product-checkout', function (Request $request) {
    return $request->user()->checkout(['price_tshirt' => 1], [
        'success_url' => route('your-success-route'),
        'cancel_url' => route('your-cancel-route'),
    ]);
});
```

當定義 `success_url` 結帳選項時，你可以指示 Stripe 在呼叫你的 URL 時將結帳會話 ID 作為查詢字串參數加入。為此，請將字面值字串 `{CHECKOUT_SESSION_ID}` 新增至 your `success_url` 查詢字串中。Stripe 會將此預留位置替換為實際的結帳會話 ID：

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

預設情況下，Stripe Checkout 不允許[使用者兌換促銷代碼](https://stripe.com/docs/billing/subscriptions/discounts/codes)。幸運的是，有一種簡單的方法可以在你的 Checkout 頁面上啟用此功能。為此，你可以呼叫 `allowPromotionCodes` 方法：

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

你也可以對尚未在 Stripe 控制面板中建立的臨時產品進行簡單扣款。為此，你可以在可計費模型上使用 `checkoutCharge` 方法，並傳入扣款金額、產品名稱以及選擇性的數量。當客戶造訪此路由時，會被重新導向至 Stripe 的 Checkout 頁面：

```php
use Illuminate\Http\Request;

Route::get('/charge-checkout', function (Request $request) {
    return $request->user()->checkoutCharge(1200, 'T-Shirt', 5);
});
```

> [!WARNING]
> 使用 `checkoutCharge` 方法時，Stripe 總是會在你的 Stripe 控制面板中建立新的產品與價格。因此，我們建議你預先在 Stripe 控制面板中建立產品，並改為使用 `checkout` 方法。


<a name="subscription-checkouts"></a>
### 訂閱結帳

> [!WARNING]
> 使用 Stripe Checkout 進行訂閱時，需要在你的 Stripe 控制面板中啟用 `customer.subscription.created` Webhook。此 Webhook 會在你的資料庫中建立訂閱紀錄，並儲存所有相關的訂閱項目。

你也可以使用 Stripe Checkout 來建立訂閱。在透過 Cashier 的訂閱建構器方法定義訂閱後，即可呼叫 `checkout `方法。當客戶造訪此路由時，會被重新導向至 Stripe 的 Checkout 頁面：

```php
use Illuminate\Http\Request;

Route::get('/subscription-checkout', function (Request $request) {
    return $request->user()
        ->newSubscription('default', 'price_monthly')
        ->checkout();
});
```

就如同產品結帳一樣，你可以自訂成功與取消的 URL：

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

當然，你也可以為訂閱結帳啟用促銷代碼：

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
> 不幸的是，Stripe Checkout 在建立訂閱時不支援所有的訂閱計費選項。在訂閱建構器上使用 `anchorBillingCycleOn` 方法、設定按比例分配 (Proration) 行為或設定付款行為，在 Stripe Checkout 會話期間都不會產生任何效果。請參閱 [Stripe Checkout Session API 說明文件](https://stripe.com/docs/api/checkout/sessions/create)以瞭解有哪些可用的參數。


<a name="stripe-checkout-trial-periods"></a>
#### Stripe Checkout 與試用期

當然，在建立將透過 Stripe Checkout 完成的訂閱時，你可以定義試用期：

```php
$checkout = Auth::user()->newSubscription('default', 'price_monthly')
    ->trialDays(3)
    ->checkout();
```

然而，試用期必須至少為 48 小時，這是 Stripe Checkout 所支援的最低試用時間。


<a name="stripe-checkout-subscriptions-and-webhooks"></a>
#### 訂閱與 Webhook

請記住，Stripe 和 Cashier 是透過 Webhook 更新訂閱狀態的，因此當客戶輸入付款資訊並返回應用程式時，訂閱可能尚未處於作用中 (Active) 狀態。為了處理這種情況，你可能希望顯示一則訊息，告知使用者其付款或訂閱正在處理中。


<a name="collecting-tax-ids"></a>
### 收集稅號

Checkout 也支援收集客戶的稅號 (Tax ID)。若要在結帳會話中啟用此功能，請在建立會話時呼叫 `collectTaxIds` 方法：

```php
$checkout = $user->collectTaxIds()->checkout('price_tshirt');
```

呼叫此方法時，客戶將會看到一個新的核取方塊，讓他們可以標示是否以公司身分購買。如果是，他們將可以提供其稅號。

> [!WARNING]
> 如果你已經在應用程式的服務提供者中設定了[自動計稅](#tax-configuration)，則此功能將會自動啟用，無需呼叫 `collectTaxIds` 方法。

<a name="guest-checkouts"></a>
### 訪客結帳

使用 `Checkout::guest` 方法，你可以為應用程式中沒有「帳號」的訪客發起結帳 Session：

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

與為現有使用者建立結帳 Session 時類似，你可以利用 `Laravel\Cashier\CheckoutBuilder` 實例上提供的其他方法來自訂訪客結帳 Session：

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

當訪客完成結帳後，Stripe 會發送 `checkout.session.completed` Webhook 事件，因此請確保已[設定你的 Stripe Webhook](https://dashboard.stripe.com/webhooks)，以便將此事件發送到你的應用程式。在 Stripe 主控台中啟用 Webhook 後，你可以[使用 Cashier 處理 Webhook](#handling-stripe-webhooks)。Webhook 負載中包含的物件將會是一個[結帳物件](https://stripe.com/docs/api/checkout/sessions/object)，你可以檢視該物件以完成客戶的訂單履約。

<a name="handling-failed-payments"></a>
## 處理失敗的付款

有時候，訂閱或單次扣款的付款可能會失敗。當這種情況發生時，Cashier 會拋出一個 `Laravel\Cashier\Exceptions\IncompletePayment` 例外來通知您。在捕捉到這個例外後，您有兩種選擇來決定如何處理。

首先，您可以將客戶重導向至 Cashier 內建的專用付款確認頁面。該頁面已經包含透過 Cashier 的服務提供者(Service Providers)所註冊的具名路由。因此，您可以捕捉 `IncompletePayment` 例外，並將使用者重導向至付款確認頁面：

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

在付款確認頁面上，系統會提示客戶再次輸入其信用卡資訊，並執行 Stripe 所需的任何額外動作，例如「3D 驗證 (3D Secure)」確認。確認付款後，使用者將被重導向至上述 `redirect` 參數所指定的 URL。重導向時，`message`（字串）與 `success`（整數）查詢字串變數將會被附加至 URL 中。該付款頁面目前支援以下付款方式類型：

<div class="content-list" markdown="1">

- 信用卡 (Credit Cards)
- 支付寶 (Alipay)
- Bancontact
- BECS 直接扣款 (BECS Direct Debit)
- EPS
- Giropay
- iDEAL
- SEPA 直接扣款 (SEPA Direct Debit)

</div>

或者，您可以讓 Stripe 替您處理付款確認。在這種情況下，您可以在 Stripe 控制面板中[設定 Stripe 的自動發票電子郵件](https://dashboard.stripe.com/account/billing/automatic)，而不是重導向至付款確認頁面。然而，如果捕捉到 `IncompletePayment` 例外，您仍應通知使用者他們將收到一封包含進一步付款確認指示的電子郵件。

使用 `Billable` trait 的模型上的以下方法可能會拋出付款例外：`charge`、`invoiceFor` 以及 `invoice`。在處理訂閱時，`SubscriptionBuilder` 上的 `create` 方法，以及 `Subscription` 與 `SubscriptionItem` 模型上的 `incrementAndInvoice` 和 `swapAndInvoice` 方法也都可能會拋出未完成付款例外。

要判斷現有訂閱是否有未完成的付款，可以使用可計費模型或訂閱實例上的 `hasIncompletePayment` 方法：

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

某些付款方式需要額外資料才能確認付款。例如，SEPA 付款方式在付款過程中需要額外的「授權 (mandate)」資料。您可以透過 `withPaymentConfirmationOptions` 方法將此資料提供給 Cashier：

```php
$subscription->withPaymentConfirmationOptions([
    'mandate_data' => '...',
])->swap('price_xxx');
```

您可以查閱 [Stripe API 文件](https://stripe.com/docs/api/payment_intents/confirm)以瞭解確認付款時接受的所有選項。


<a name="strong-customer-authentication"></a>
## 強效客戶認證 (SCA)

如果您的業務或您的客戶之一位於歐洲，您需要遵守歐盟的強效客戶認證 (Strong Customer Authentication, SCA) 法規。這些法規是歐盟於 2019 年 9 月實施的，旨在防止付款詐欺。幸運的是，Stripe 和 Cashier 已為建置符合 SCA 規範的應用程式做好準備。

> [!WARNING]
> 在開始之前，請先閱讀 [Stripe 關於 PSD2 與 SCA 的指南](https://stripe.com/guides/strong-customer-authentication)以及他們[關於新 SCA API 的文件](https://stripe.com/docs/strong-customer-authentication)。


<a name="payments-requiring-additional-confirmation"></a>
### 需要額外確認的付款

SCA 法規通常需要額外的驗證才能確認並處理付款。當這種情況發生時，Cashier 會拋出 `Laravel\Cashier\Exceptions\IncompletePayment` 例外，告知您需要額外的驗證。有關如何處理這些例外的更多資訊，可以在[處理失敗的付款](#handling-failed-payments)的文件中找到。

由 Stripe 或 Cashier 顯示的付款確認畫面可能會針對特定銀行或發卡機構的付款流程進行客製化，並且可能包含額外的卡片確認、臨時的小額扣款、獨立的裝置認證或其他形式的驗證。


<a name="incomplete-and-past-due-state"></a>
#### 未完成與逾期狀態

當付款需要額外確認時，如資料庫欄位 `stripe_status` 所示，該訂閱將保持在 `incomplete` 或 `past_due` 狀態。一旦付款確認完成，且您的應用程式透過 Webhook 收到 Stripe 的完成通知，Cashier 就會自動啟動客戶的訂閱。

有關 `incomplete` 與 `past_due` 狀態的更多資訊，請參考[關於這些狀態的補充文件](#incomplete-and-past-due-status)。


<a name="off-session-payment-notifications"></a>
### 非即時會話付款通知 (Off-session)

由於 SCA 法規要求客戶即使在訂閱處於活動狀態時，有時也需要驗證其付款詳細資料，因此當需要進行非即時會話 (Off-session) 付款確認時，Cashier 可以向客戶傳送通知。例如，這可能會在訂閱續訂時發生。只要將 `CASHIER_PAYMENT_NOTIFICATION` 環境變數設定為通知類別，即可啟用 Cashier 的付款通知。預設情況下，此通知是停用的。當然，Cashier 包含一個可用於此目的的通知類別，但如果您需要，也可以自由提供您自己的通知類別：

```ini
CASHIER_PAYMENT_NOTIFICATION=Laravel\Cashier\Notifications\ConfirmPayment
```

為確保能送達非即時會話付款確認通知，請確認您的應用程式已[設定 Stripe Webhook](#handling-stripe-webhooks)，並且已在 Stripe 控制面板中啟用了 `invoice.payment_action_required` Webhook。此外，您的 `Billable` 模型也應該使用 Laravel 的 `Illuminate\Notifications\Notifiable` trait。

> [!WARNING]
> 即便客戶是手動進行需要額外確認的付款，系統仍會傳送通知。遺憾的是，Stripe 無法得知該付款是手動完成還是「非即時會話 (Off-session)」完成的。不過，如果客戶在確認付款後造訪該付款頁面，他們只會看到「付款成功」的訊息。系統不會允許客戶意外確認同筆付款兩次並產生第二次扣款。

<a name="stripe-sdk"></a>
## Stripe SDK

Cashier 的許多物件都是對 Stripe SDK 物件進行的封裝 (wrapper)。如果您想直接與 Stripe 物件進行互動，可以使用 `asStripe` 方法方便地取得它們：

```php
$stripeSubscription = $subscription->asStripeSubscription();

$stripeSubscription->application_fee_percent = 5;

$stripeSubscription->save();
```

您也可以使用 `updateStripeSubscription` 方法來直接更新 Stripe 訂閱：

```php
$subscription->updateStripeSubscription(['application_fee_percent' => 5]);
```

如果您想直接使用 `Stripe\StripeClient` 客戶端，可以呼叫 `Cashier` 類別上的 `stripe` 方法。例如，您可以使用此方法來存取 `StripeClient` 實例，並從您的 Stripe 帳號中取得價格列表：

```php
use Laravel\Cashier\Cashier;

$prices = Cashier::stripe()->prices->all();
```

<a name="testing"></a>
## 測試

在測試使用 Cashier 的應用程式時，您可以模擬 (mock) 對 Stripe API 的實際 HTTP 請求；然而，這會需要您重新實作一部分 Cashier 本身行為。因此，我們建議讓您的測試實際存取 Stripe API。雖然這樣比較慢，但能更確定您的應用程式運作符合預期，且任何較慢的測試都可以放在自己的 Pest / PHPUnit 測試群組中。

進行測試時，請記住 Cashier 本身就已經有一套完善的測試套件，因此您應該只專注於測試您自己應用程式的訂閱與付款流程，而不是測試 Cashier 底層的每個行為。

若要開始，請將 Stripe 密鑰的**測試 (testing)** 版本新增至您的 `phpunit.xml` 檔案中：

```xml
<env name="STRIPE_SECRET" value="sk_test_<your-key>"/>
```

現在，每當您在測試期間與 Cashier 互動時，它都會發送實際的 API 請求至您的 Stripe 測試環境。為了方便起見，您應該預先在 Stripe 測試帳號中建立好測試期間可能用到的訂閱 / 價格。

> [!NOTE]
> 為了測試各種扣款情境（例如信用卡拒刷或失敗），您可以使用 Stripe 所提供豐富的[測試卡號與 token](https://stripe.com/docs/testing)。