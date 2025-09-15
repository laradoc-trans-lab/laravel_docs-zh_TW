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
    - [多重訂閱](#multiple-subscriptions)
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
- [結帳](#checkout)
    - [產品結帳](#product-checkouts)
    - [單次收費結帳](#single-charge-checkouts)
    - [訂閱結帳](#subscription-checkouts)
    - [收集稅籍編號](#collecting-tax-ids)
    - [訪客結帳](#guest-checkouts)
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

[Laravel Cashier Stripe](https://github.com/laravel/cashier-stripe) 為 [Stripe](https://stripe.com) 的訂閱計費服務提供了表達性強、流暢的介面。它處理了幾乎所有您不願編寫的樣板訂閱計費程式碼。除了基本的訂閱管理之外，Cashier 還可以處理優惠券、交換訂閱、訂閱「數量」、取消寬限期，甚至產生發票 PDF。

<a name="upgrading-cashier"></a>
## 升級 Cashier

升級到新版 Cashier 時，請務必仔細閱讀[升級指南](https://github.com/laravel/cashier-stripe/blob/master/UPGRADE.md)。

> [!WARNING]
> 為防止破壞性變更，Cashier 使用固定的 Stripe API 版本。Cashier 15 使用 Stripe API 版本 `2023-10-16`。Stripe API 版本將在次要版本中更新，以便利用新的 Stripe 功能和改進。

<a name="installation"></a>
## 安裝

首先，使用 Composer 套件管理器安裝適用於 Stripe 的 Cashier 套件：

```shell
composer require laravel/cashier
```

安裝套件後，使用 `vendor:publish` Artisan 命令發佈 Cashier 的遷移檔：

```shell
php artisan vendor:publish --tag="cashier-migrations"
```

然後，遷移您的資料庫：

```shell
php artisan migrate
```

Cashier 的遷移檔將為您的 `users` 資料表新增多個欄位。它們還將建立一個新的 `subscriptions` 資料表來儲存所有客戶的訂閱，以及一個 `subscription_items` 資料表用於多價格訂閱。

如果您願意，也可以使用 `vendor:publish` Artisan 命令發佈 Cashier 的設定檔：

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

    use Laravel\Cashier\Billable;

    class User extends Authenticatable
    {
        use Billable;
    }

Cashier 假設您的可計費模型將是 Laravel 隨附的 `App\Models\User` 類別。如果您希望更改此設定，可以透過 `useCustomerModel` 方法指定不同的模型。此方法通常應在您的 `AppServiceProvider` 類別的 `boot` 方法中呼叫：

    use App\Models\Cashier\User;
    use Laravel\Cashier\Cashier;

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        Cashier::useCustomerModel(User::class);
    }

> [!WARNING]
> 如果您使用的模型不是 Laravel 提供的 `App\Models\User` 模型，您將需要發佈並修改提供的 [Cashier 遷移檔](#installation)以符合您替代模型的資料表名稱。

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

除了設定 Cashier 的貨幣之外，您還可以指定一個語系，用於在發票上格式化貨幣值以供顯示。在內部，Cashier 利用 [PHP 的 `NumberFormatter` 類別](https://www.php.net/manual/en/class.numberformatter.php)來設定貨幣語系：

```ini
CASHIER_CURRENCY_LOCALE=nl_BE
```

> [!WARNING]
> 為了使用 `en` 以外的語系，請確保您的伺服器上已安裝並設定 `ext-intl` PHP 擴充功能。

<a name="tax-configuration"></a>
### 稅務設定

多虧了 [Stripe Tax](https://stripe.com/tax)，可以自動計算 Stripe 產生的所有發票的稅金。您可以透過在應用程式的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫 `calculateTaxes` 方法來啟用自動稅金計算：

    use Laravel\Cashier\Cashier;

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        Cashier::calculateTaxes();
    }

啟用稅金計算後，任何新的訂閱和任何產生的一次性發票都將收到自動稅金計算。

為了使此功能正常運作，客戶的帳單詳細資訊，例如客戶的姓名、地址和稅籍編號，需要同步到 Stripe。您可以使用 Cashier 提供的[客戶資料同步](#syncing-customer-data-with-stripe)和[稅籍編號](#tax-ids)方法來完成此操作。

<a name="logging"></a>
### 日誌

Cashier 允許您指定在記錄嚴重 Stripe 錯誤時要使用的日誌通道。您可以透過在應用程式的 `.env` 檔案中定義 `CASHIER_LOGGER` 環境變數來指定日誌通道：

```ini
CASHIER_LOGGER=stack
```

由對 Stripe 的 API 呼叫產生的例外將透過應用程式的預設日誌通道記錄。

<a name="using-custom-models"></a>
### 使用自訂模型

您可以自由地透過定義自己的模型並擴展相應的 Cashier 模型來擴展 Cashier 內部使用的模型：

    use Laravel\Cashier\Subscription as CashierSubscription;

    class Subscription extends CashierSubscription
    {
        // ...
    }

定義模型後，您可以透過 `Laravel\Cashier\Cashier` 類別指示 Cashier 使用您的自訂模型。通常，您應該在應用程式的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中通知 Cashier 您的自訂模型：

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

<a name="quickstart"></a>
## 快速入門

<a name="quickstart-selling-products"></a>
### 銷售產品

> [!NOTE]
> 在使用 Stripe Checkout 之前，您應該在 Stripe 控制面板中定義具有固定價格的產品。此外，您應該[設定 Cashier 的 Webhook 處理](#handling-stripe-webhooks)。

透過您的應用程式提供產品和訂閱計費可能令人望而生畏。然而，多虧了 Cashier 和 [Stripe Checkout](https://stripe.com/payments/checkout)，您可以輕鬆建立現代、強大的支付整合。

為了向客戶收取非經常性、單次收費產品的費用，我們將利用 Cashier 將客戶引導至 Stripe Checkout，他們將在那裡提供付款詳細資訊並確認購買。一旦透過 Checkout 完成付款，客戶將被重新導向到您在應用程式中選擇的成功 URL：

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

如您在上述範例中看到的，我們將利用 Cashier 提供的 `checkout` 方法將客戶重新導向到 Stripe Checkout 以獲取給定的「價格識別碼」。在使用 Stripe 時，「價格」是指[針對特定產品定義的價格](https://stripe.com/docs/products-prices/how-products-and-prices-work)。

如有必要，`checkout` 方法將自動在 Stripe 中建立客戶，並將該 Stripe 客戶記錄連接到應用程式資料庫中對應的使用者。完成結帳會話後，客戶將被重新導向到專用的成功或取消頁面，您可以在其中向客戶顯示資訊性訊息。

<a name="providing-meta-data-to-stripe-checkout"></a>
#### 向 Stripe Checkout 提供中繼資料

銷售產品時，通常會透過您自己的應用程式定義的 `Cart` 和 `Order` 模型來追蹤已完成的訂單和已購買的產品。當將客戶重新導向到 Stripe Checkout 以完成購買時，您可能需要提供現有的訂單識別碼，以便在客戶重新導向回您的應用程式時，您可以將已完成的購買與相應的訂單關聯起來。

為此，您可以向 `checkout` 方法提供一個 `metadata` 陣列。讓我們想像一下，當使用者開始結帳流程時，我們的應用程式中會建立一個待處理的 `Order`。請記住，此範例中的 `Cart` 和 `Order` 模型是說明性的，並非由 Cashier 提供。您可以根據自己應用程式的需求自由實作這些概念：

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

如您在上述範例中看到的，當使用者開始結帳流程時，我們將向 `checkout` 方法提供所有購物車/訂單相關的 Stripe 價格識別碼。當然，您的應用程式負責在客戶新增這些項目時將它們與「購物車」或訂單關聯起來。我們還透過 `metadata` 陣列將訂單的 ID 提供給 Stripe Checkout 會話。最後，我們已將 `CHECKOUT_SESSION_ID` 範本變數新增到 Checkout 成功路由。當 Stripe 將客戶重新導向回您的應用程式時，此範本變數將自動填入 Checkout 會話 ID。

接下來，讓我們建立 Checkout 成功路由。這是使用者在透過 Stripe Checkout 完成購買後將被重新導向的路由。在此路由中，我們可以取得 Stripe Checkout 會話 ID 和相關的 Stripe Checkout 實例，以便存取我們提供的中繼資料並相應地更新客戶的訂單：

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

請參閱 Stripe 的說明文件，以獲取有關 [Checkout 會話物件所包含資料](https://stripe.com/docs/api/checkout/sessions/object)的更多資訊。

<a name="quickstart-selling-subscriptions"></a>
### 銷售訂閱

> [!NOTE]
> 在使用 Stripe Checkout 之前，您應該在 Stripe 控制面板中定義具有固定價格的產品。此外，您應該[設定 Cashier 的 Webhook 處理](#handling-stripe-webhooks)。

透過您的應用程式提供產品和訂閱計費可能令人望而生畏。然而，多虧了 Cashier 和 [Stripe Checkout](https://stripe.com/payments/checkout)，您可以輕鬆建立現代、強大的支付整合。

為了學習如何使用 Cashier 和 Stripe Checkout 銷售訂閱，讓我們考慮一個簡單的訂閱服務場景，其中包含基本月度 (`price_basic_monthly`) 和年度 (`price_basic_yearly`) 方案。這兩個價格可以在我們的 Stripe 控制面板中歸類到一個「基本」產品 (`pro_basic`) 下。此外，我們的訂閱服務可能提供一個 Expert 方案作為 `pro_expert`。

首先，讓我們了解客戶如何訂閱我們的服務。當然，您可以想像客戶可能會在我們應用程式的定價頁面上點擊基本方案的「訂閱」按鈕。此按鈕或連結應將使用者引導到一個 Laravel 路由，該路由將為他們選擇的方案建立 Stripe Checkout 會話：

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

如您在上述範例中看到的，我們將客戶重新導向到一個 Stripe Checkout 會話，該會話將允許他們訂閱我們的基本方案。成功結帳或取消後，客戶將被重新導向回我們提供給 `checkout` 方法的 URL。為了知道他們的訂閱何時實際開始（因為某些付款方式需要幾秒鐘才能處理），我們還需要[設定 Cashier 的 Webhook 處理](#handling-stripe-webhooks)。

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
#### 建立訂閱中介層

為了方便起見，您可能希望建立一個[中介層](/docs/{{version}}/middleware)，該中介層判斷傳入的請求是否來自訂閱使用者。一旦定義了此中介層，您可以輕鬆地將其分配給路由，以防止未訂閱的使用者存取該路由：

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

一旦定義了中介層，您可以將其分配給路由：

    use App\Http\Middleware\Subscribed;

    Route::get('/dashboard', function () {
        // ...
    })->middleware([Subscribed::class]);

<a name="quickstart-allowing-customers-to-manage-their-billing-plan"></a>
#### 允許客戶管理其帳單方案

當然，客戶可能希望將其訂閱方案更改為其他產品或「層級」。最簡單的方法是將客戶引導至 Stripe 的[客戶帳單入口網站](https://stripe.com/docs/no-code/customer-portal)，該入口網站提供了一個託管的使用者介面，允許客戶下載發票、更新其付款方式和更改訂閱方案。

首先，在您的應用程式中定義一個連結或按鈕，將使用者引導到一個 Laravel 路由，我們將利用該路由來啟動帳單入口網站會話：

```blade
<a href="{{ route('billing') }}">
    帳單
</a>
```

接下來，讓我們定義一個路由，該路由啟動 Stripe 客戶帳單入口網站會話並將使用者重新導向到入口網站。`redirectToBillingPortal` 方法接受使用者在退出入口網站時應返回的 URL：

    use Illuminate\Http\Request;

    Route::get('/billing', function (Request $request) {
        return $request->user()->redirectToBillingPortal(route('dashboard'));
    })->middleware(['auth'])->name('billing');

> [!NOTE]
> 只要您已設定 Cashier 的 Webhook 處理，Cashier 將透過檢查來自 Stripe 的傳入 Webhook 自動保持應用程式的 Cashier 相關資料庫表格同步。因此，例如，當使用者透過 Stripe 的客戶帳單入口網站取消訂閱時，Cashier 將收到相應的 Webhook 並在應用程式的資料庫中將訂閱標記為「已取消」。

<a name="customers"></a>
## 客戶

<a name="retrieving-customers"></a>
### 取得客戶

您可以使用 `Cashier::findBillable` 方法透過客戶的 Stripe ID 取得客戶。此方法將返回可計費模型的實例：

    use Laravel\Cashier\Cashier;

    $user = Cashier::findBillable($stripeId);

<a name="creating-customers"></a>
### 建立客戶

有時，您可能希望在不開始訂閱的情況下建立 Stripe 客戶。您可以使用 `createAsStripeCustomer` 方法來完成此操作：

    $stripeCustomer = $user->createAsStripeCustomer();

一旦客戶在 Stripe 中建立，您可以在以後開始訂閱。您可以提供一個可選的 `$options` 陣列，以傳遞 [Stripe API 支援的任何額外客戶建立參數](https://stripe.com/docs/api/customers/create)：

    $stripeCustomer = $user->createAsStripeCustomer($options);

如果您想返回可計費模型的 Stripe 客戶物件，可以使用 `asStripeCustomer` 方法：

    $stripeCustomer = $user->asStripeCustomer();

如果您想取得給定可計費模型的 Stripe 客戶物件，但不確定該可計費模型是否已是 Stripe 中的客戶，則可以使用 `createOrGetStripeCustomer` 方法。如果 Stripe 中不存在客戶，此方法將建立一個新客戶：

    $stripeCustomer = $user->createOrGetStripeCustomer();

<a name="updating-customers"></a>
### 更新客戶

有時，您可能希望直接使用額外資訊更新 Stripe 客戶。您可以使用 `updateStripeCustomer` 方法來完成此操作。此方法接受一個 [Stripe API 支援的客戶更新選項](https://stripe.com/docs/api/customers/update)陣列：

    $stripeCustomer = $user->updateStripeCustomer($options);

<a name="balances"></a>
### 餘額

Stripe 允許您貸記或借記客戶的「餘額」。稍後，此餘額將在新的發票上貸記或借記。要檢查客戶的總餘額，您可以使用可計費模型上可用的 `balance` 方法。`balance` 方法將以客戶貨幣返回餘額的格式化字串表示：

    $balance = $user->balance();

要貸記客戶的餘額，您可以向 `creditBalance` 方法提供一個值。如果您願意，您還可以提供一個描述：

    $user->creditBalance(500, 'Premium customer top-up.');

向 `debitBalance` 方法提供一個值將借記客戶的餘額：

    $user->debitBalance(300, 'Bad usage penalty.');

`applyBalance` 方法將為客戶建立新的客戶餘額交易。您可以使用 `balanceTransactions` 方法取得這些交易記錄，這對於為客戶提供貸記和借記日誌以供審查可能很有用：

    // 取得所有交易...
    $transactions = $user->balanceTransactions();

    foreach ($transactions as $transaction) {
        // 交易金額...
        $amount = $transaction->amount(); // $2.31

        // 取得相關發票 (如果可用)...
        $invoice = $transaction->invoice();
    }

<a name="tax-ids"></a>
### 稅籍編號

Cashier 提供了一種管理客戶稅籍編號的簡單方法。例如，`taxIds` 方法可用於以集合形式取得分配給客戶的所有[稅籍編號](https://stripe.com/docs/api/customer_tax_ids/object)：

    $taxIds = $user->taxIds();

您還可以透過其識別碼取得客戶的特定稅籍編號：

    $taxId = $user->findTaxId('txi_belgium');

您可以透過向 `createTaxId` 方法提供有效的[類型](https://stripe.com/docs/api/customer_tax_ids/object#tax_id_object-type)和值來建立新的稅籍編號：

    $taxId = $user->createTaxId('eu_vat', 'BE0123456789');

`createTaxId` 方法將立即將 VAT ID 新增到客戶的帳戶中。[VAT ID 的驗證也由 Stripe 完成](https://stripe.com/docs/invoicing/customer/tax-ids#validation)；但是，這是一個非同步過程。您可以透過訂閱 `customer.tax_id.updated` Webhook 事件並檢查 [VAT ID 的 `verification` 參數](https://stripe.com/docs/api/customer_tax_ids/object#tax_id_object-verification)來接收驗證更新通知。有關處理 Webhook 的更多資訊，請參閱[定義 Webhook 處理器](##defining-webhook-event-handlers)的說明文件。

您可以使用 `deleteTaxId` 方法刪除稅籍編號：

    $user->deleteTaxId('txi_belgium');

<a name="syncing-customer-data-with-stripe"></a>
### 將客戶資料與 Stripe 同步

通常，當您的應用程式使用者更新其姓名、電子郵件地址或其他也由 Stripe 儲存的資訊時，您應該通知 Stripe 這些更新。這樣做，Stripe 的資訊副本將與您的應用程式同步。

為了自動化此過程，您可以在可計費模型上定義一個事件監聽器，該監聽器響應模型的 `updated` 事件。然後，在您的事件監聽器中，您可以呼叫模型上的 `syncStripeCustomerDetails` 方法：

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

現在，每次更新您的客戶模型時，其資訊都將與 Stripe 同步。為了方便起見，Cashier 將在客戶首次建立時自動將您的客戶資訊與 Stripe 同步。

您可以透過覆寫 Cashier 提供的各種方法來自訂用於將客戶資訊同步到 Stripe 的欄位。例如，您可以覆寫 `stripeName` 方法來自訂當 Cashier 將客戶資訊同步到 Stripe 時應被視為客戶「姓名」的屬性：

    /**
     * Get the customer name that should be synced to Stripe.
     */
    public function stripeName(): string|null
    {
        return $this->company_name;
    }

同樣，您可以覆寫 `stripeEmail`、`stripePhone`、`stripeAddress` 和 `stripePreferredLocales` 方法。這些方法將在[更新 Stripe 客戶物件](https://stripe.com/docs/api/customers/update)時將資訊同步到其對應的客戶參數。如果您希望完全控制客戶資訊同步過程，您可以覆寫 `syncStripeCustomerDetails` 方法。

<a name="billing-portal"></a>
### 帳務入口網站

Stripe 提供[一種簡單的方法來設定帳單入口網站](https://stripe.com/docs/billing/subscriptions/customer-portal)，以便您的客戶可以管理其訂閱、付款方式和查看其帳單歷史記錄。您可以透過從控制器或路由中呼叫可計費模型上的 `redirectToBillingPortal` 方法，將使用者重新導向到帳單入口網站：

    use Illuminate\Http\Request;

    Route::get('/billing-portal', function (Request $request) {
        return $request->user()->redirectToBillingPortal();
    });

預設情況下，當使用者完成訂閱管理後，他們將能夠透過 Stripe 帳單入口網站中的連結返回應用程式的 `home` 路由。您可以透過將 URL 作為參數傳遞給 `redirectToBillingPortal` 方法來提供使用者應返回的自訂 URL：

    use Illuminate\Http\Request;

    Route::get('/billing-portal', function (Request $request) {
        return $request->user()->redirectToBillingPortal(route('billing'));
    });

如果您想在不產生 HTTP 重新導向響應的情況下產生帳單入口網站的 URL，您可以呼叫 `billingPortalUrl` 方法：

    $url = $request->user()->billingPortalUrl(route('billing'));

<a name="payment-methods"></a>
## 付款方式

<a name="storing-payment-methods"></a>
### 儲存付款方式

為了建立訂閱或使用 Stripe 執行「一次性」收費，您需要儲存付款方式並從 Stripe 取得其識別碼。完成此操作的方法因您是打算將付款方式用於訂閱還是單次收費而異，因此我們將在下面分別探討這兩種情況。

<a name="payment-methods-for-subscriptions"></a>
#### 訂閱的付款方式

當儲存客戶的信用卡資訊以供訂閱未來使用時，必須使用 Stripe「Setup Intents」API 安全地收集客戶的付款方式詳細資訊。「Setup Intent」向 Stripe 表明打算向客戶的付款方式收費。Cashier 的 `Billable` trait 包含 `createSetupIntent` 方法，可輕鬆建立新的 Setup Intent。您應該從將呈現收集客戶付款方式詳細資訊的表單的路由或控制器中呼叫此方法：

    return view('update-payment-method', [
        'intent' => $user->createSetupIntent()
    ]);

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

當然，當對客戶的付款方式進行單次收費時，我們只需要使用一次付款方式識別碼。由於 Stripe 的限制，您不能將客戶儲存的預設付款方式用於單次收費。您必須允許客戶使用 Stripe.js 函式庫輸入其付款方式詳細資訊。例如，考慮以下表單：

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

    $paymentMethods = $user->paymentMethods();

預設情況下，此方法將返回所有類型的付款方式。要取得特定類型的付款方式，您可以將 `type` 作為參數傳遞給該方法：

    $paymentMethods = $user->paymentMethods('sepa_debit');

要取得客戶的預設付款方式，可以使用 `defaultPaymentMethod` 方法：

    $paymentMethod = $user->defaultPaymentMethod();

您可以使用 `findPaymentMethod` 方法取得附加到可計費模型的特定付款方式：

    $paymentMethod = $user->findPaymentMethod($paymentMethodId);

<a name="payment-method-presence"></a>
### 付款方式存在性

要判斷可計費模型是否已將預設付款方式附加到其帳戶，請呼叫 `hasDefaultPaymentMethod` 方法：

    if ($user->hasDefaultPaymentMethod()) {
        // ...
    }

您可以使用 `hasPaymentMethod` 方法判斷可計費模型是否至少有一個付款方式附加到其帳戶：

    if ($user->hasPaymentMethod()) {
        // ...
    }

此方法將判斷可計費模型是否具有任何付款方式。要判斷模型是否存在特定類型的付款方式，您可以將 `type` 作為參數傳遞給該方法：

    if ($user->hasPaymentMethod('sepa_debit')) {
        // ...
    }

<a name="updating-the-default-payment-method"></a>
### 更新預設付款方式

`updateDefaultPaymentMethod` 方法可用於更新客戶的預設付款方式資訊。此方法接受一個 Stripe 付款方式識別碼，並將新的付款方式指定為預設帳單付款方式：

    $user->updateDefaultPaymentMethod($paymentMethod);

要將您的預設付款方式資訊與 Stripe 中客戶的預設付款方式資訊同步，您可以使用 `updateDefaultPaymentMethodFromStripe` 方法：

    $user->updateDefaultPaymentMethodFromStripe();

> [!WARNING]
> 客戶的預設付款方式只能用於開立發票和建立新訂閱。由於 Stripe 施加的限制，它不能用於單次收費。

<a name="adding-payment-methods"></a>
### 新增付款方式

要新增付款方式，您可以呼叫可計費模型上的 `addPaymentMethod` 方法，並傳遞付款方式識別碼：

    $user->addPaymentMethod($paymentMethod);

> [!NOTE]
> 要了解如何取得付款方式識別碼，請查閱[付款方式儲存說明文件](#storing-payment-methods)。

<a name="deleting-payment-methods"></a>
### 刪除付款方式

要刪除付款方式，您可以呼叫您希望刪除的 `Laravel\Cashier\PaymentMethod` 實例上的 `delete` 方法：

    $paymentMethod->delete();

`deletePaymentMethod` 方法將從可計費模型中刪除特定的付款方式：

    $user->deletePaymentMethod('pm_visa');

`deletePaymentMethods` 方法將刪除可計費模型的所有付款方式資訊：

    $user->deletePaymentMethods();

預設情況下，此方法將刪除所有類型的付款方式。要刪除特定類型的付款方式，您可以將 `type` 作為參數傳遞給該方法：

    $user->deletePaymentMethods('sepa_debit');

> [!WARNING]
> 如果使用者有有效的訂閱，您的應用程式不應允許他們刪除其預設付款方式。

<a name="subscriptions"></a>
## 訂閱

訂閱提供了一種為客戶設定定期付款的方式。由 Cashier 管理的 Stripe 訂閱支援多種訂閱價格、訂閱數量、試用期等。

<a name="creating-subscriptions"></a>
### 建立訂閱

要建立訂閱，首先取得您的可計費模型實例，這通常是 `App\Models\User` 的實例。取得模型實例後，您可以使用 `newSubscription` 方法建立模型的訂閱：

    use Illuminate\Http\Request;

    Route::post('/user/subscribe', function (Request $request) {
        $request->user()->newSubscription(
            'default', 'price_monthly'
        )->create($request->paymentMethodId);

        // ...
    });

傳遞給 `newSubscription` 方法的第一個參數應該是訂閱的內部類型。如果您的應用程式只提供單一訂閱，您可以將其稱為 `default` 或 `primary`。此訂閱類型僅供內部應用程式使用，不應向使用者顯示。此外，它不應包含空格，並且在建立訂閱後絕不應更改。第二個參數是使用者訂閱的特定價格。此值應與 Stripe 中的價格識別碼相對應。

`create` 方法接受[一個 Stripe 付款方式識別碼](#storing-payment-methods)或 Stripe `PaymentMethod` 物件，它將開始訂閱並使用可計費模型的 Stripe 客戶 ID 和其他相關帳單資訊更新您的資料庫。

> [!WARNING]
> 將付款方式識別碼直接傳遞給 `create` 訂閱方法也將自動將其新增到使用者的儲存付款方式中。

<a name="collecting-recurring-payments-via-invoice-emails"></a>
#### 透過發票電子郵件收取定期付款

您可以指示 Stripe 在每次定期付款到期時向客戶發送電子郵件發票，而不是自動收取客戶的定期付款。然後，客戶可以在收到發票後手動支付。透過發票收取定期付款時，客戶無需預先提供付款方式：

    $user->newSubscription('default', 'price_monthly')->createAndSendInvoice();

客戶在訂閱被取消之前支付發票的時間長度由 `days_until_due` 選項決定。預設情況下，這是 30 天；但是，如果您願意，您可以為此選項提供一個特定值：

    $user->newSubscription('default', 'price_monthly')->createAndSendInvoice([], [
        'days_until_due' => 30
    ]);

<a name="subscription-quantities"></a>
#### 數量

如果您想在建立訂閱時為價格設定特定的[數量](https://stripe.com/docs/billing/subscriptions/quantities)，您應該在建立訂閱之前呼叫訂閱建構器上的 `quantity` 方法：

    $user->newSubscription('default', 'price_monthly')
        ->quantity(5)
        ->create($paymentMethod);

<a name="additional-details"></a>
#### 額外詳細資訊

如果您想指定 Stripe 支援的額外[客戶](https://stripe.com/docs/api/customers/create)或[訂閱](https://stripe.com/docs/api/subscriptions/create)選項，您可以將它們作為第二個和第三個參數傳遞給 `create` 方法：

    $user->newSubscription('default', 'price_monthly')->create($paymentMethod, [
        'email' => $email,
    ], [
        'metadata' => ['note' => 'Some extra information.'],
    ]);

<a name="coupons"></a>
#### 優惠券

如果您想在建立訂閱時應用優惠券，您可以使用 `withCoupon` 方法：

    $user->newSubscription('default', 'price_monthly')
        ->withCoupon('code')
        ->create($paymentMethod);

或者，如果您想應用 [Stripe 促銷代碼](https://stripe.com/docs/billing/subscriptions/discounts/codes)，您可以使用 `withPromotionCode` 方法：

    $user->newSubscription('default', 'price_monthly')
        ->withPromotionCode('promo_code_id')
        ->create($paymentMethod);

給定的促銷代碼 ID 應該是分配給促銷代碼的 Stripe API ID，而不是面向客戶的促銷代碼。如果您需要根據給定的面向客戶的促銷代碼找到促銷代碼 ID，您可以使用 `findPromotionCode` 方法：

    // 透過其面向客戶的代碼找到促銷代碼 ID...
    $promotionCode = $user->findPromotionCode('SUMMERSALE');

    // 透過其面向客戶的代碼找到有效的促銷代碼 ID...
    $promotionCode = $user->findActivePromotionCode('SUMMERSALE');

在上面的範例中，返回的 `$promotionCode` 物件是 `Laravel\Cashier\PromotionCode` 的實例。此類別裝飾了一個底層的 `Stripe\PromotionCode` 物件。您可以透過呼叫 `coupon` 方法來取得與促銷代碼相關的優惠券：

    $coupon = $user->findPromotionCode('SUMMERSALE')->coupon();

優惠券實例允許您確定折扣金額以及優惠券是固定折扣還是基於百分比的折扣：

    if ($coupon->isPercentage()) {
        return $coupon->percentOff().'%'; // 21.5%
    } else {
        return $coupon->amountOff(); // $5.99
    }

您還可以取得目前應用於客戶或訂閱的折扣：

    $discount = $billable->discount();

    $discount = $subscription->discount();

返回的 `Laravel\Cashier\Discount` 實例裝飾了一個底層的 `Stripe\Discount` 物件實例。您可以透過呼叫 `coupon` 方法來取得與此折扣相關的優惠券：

    $coupon = $subscription->discount()->coupon();

如果您想向客戶或訂閱應用新的優惠券或促銷代碼，您可以透過 `applyCoupon` 或 `applyPromotionCode` 方法來完成：

    $billable->applyCoupon('coupon_id');
    $billable->applyPromotionCode('promotion_code_id');

    $subscription->applyCoupon('coupon_id');
    $subscription->applyPromotionCode('promotion_code_id');

請記住，您應該使用分配給促銷代碼的 Stripe API ID，而不是面向客戶的促銷代碼。在給定時間內，只能將一個優惠券或促銷代碼應用於客戶或訂閱。

有關此主題的更多資訊，請參閱 Stripe 關於[優惠券](https://stripe.com/docs/billing/subscriptions/coupons)和[促銷代碼](https://stripe.com/docs/billing/subscriptions/coupons/codes)的說明文件。

<a name="adding-subscriptions"></a>
#### 新增訂閱

如果您想為已經有預設付款方式的客戶新增訂閱，您可以呼叫訂閱建構器上的 `add` 方法：

    use App\Models\User;

    $user = User::find(1);

    $user->newSubscription('default', 'price_monthly')->add();

<a name="creating-subscriptions-from-the-stripe-dashboard"></a>
#### 從 Stripe 控制面板建立訂閱

您也可以從 Stripe 控制面板本身建立訂閱。這樣做時，Cashier 將同步新新增的訂閱並將其指定為 `default` 類型。要自訂分配給控制面板建立的訂閱的訂閱類型，請[定義 Webhook 事件處理器](#defining-webhook-event-handlers)。

此外，您只能透過 Stripe 控制面板建立一種訂閱類型。如果您的應用程式提供使用不同類型的多個訂閱，則只能透過 Stripe 控制面板新增一種訂閱類型。

最後，您應該始終確保每個應用程式提供的訂閱類型只新增一個有效訂閱。如果客戶有兩個 `default` 訂閱，則 Cashier 將只使用最近新增的訂閱，即使這兩個訂閱都將與您的應用程式的資料庫同步。

<a name="checking-subscription-status"></a>
### 檢查訂閱狀態

一旦客戶訂閱了您的應用程式，您可以使用各種方便的方法輕鬆檢查其訂閱狀態。首先，如果客戶有有效的訂閱，即使訂閱目前處於試用期內，`subscribed` 方法也會返回 `true`。`subscribed` 方法將訂閱類型作為其第一個參數：

    if ($user->subscribed('default')) {
        // ...
    }

`subscribed` 方法也是[路由中介層](/docs/{{version}}/middleware)的絕佳候選者，允許您根據使用者的訂閱狀態篩選對路由和控制器的存取：

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

如果您想判斷使用者是否仍在試用期內，您可以使用 `onTrial` 方法。此方法對於判斷您是否應該向使用者顯示他們仍在試用期內的警告很有用：

    if ($user->subscription('default')->onTrial()) {
        // ...
    }

`subscribedToProduct` 方法可用於根據給定的 Stripe 產品識別碼判斷使用者是否訂閱了給定產品。在 Stripe 中，產品是價格的集合。在此範例中，我們將判斷使用者的 `default` 訂閱是否已主動訂閱應用程式的「premium」產品。給定的 Stripe 產品識別碼應與您在 Stripe 控制面板中的產品識別碼之一相對應：

    if ($user->subscribedToProduct('prod_premium', 'default')) {
        // ...
    }

透過向 `subscribedToProduct` 方法傳遞一個陣列，您可以判斷使用者的 `default` 訂閱是否已主動訂閱應用程式的「basic」或「premium」產品：

    if ($user->subscribedToProduct(['prod_basic', 'prod_premium'], 'default')) {
        // ...
    }

`subscribedToPrice` 方法可用於判斷客戶的訂閱是否與給定的價格 ID 相符：

    if ($user->subscribedToPrice('price_basic_monthly', 'default')) {
        // ...
    }

`recurring` 方法可用於判斷使用者目前是否已訂閱且不再處於試用期內：

    if ($user->subscription('default')->recurring()) {
        // ...
    }

> [!WARNING]
> 如果使用者有兩個相同類型的訂閱，`subscription` 方法將始終返回最新的訂閱。例如，使用者可能有兩個類型為 `default` 的訂閱記錄；但是，其中一個訂閱可能是舊的、已過期的訂閱，而另一個是當前有效的訂閱。最新的訂閱將始終返回，而舊的訂閱則保留在資料庫中以供歷史審查。

<a name="cancelled-subscription-status"></a>
#### 已取消訂閱狀態

要判斷使用者是否曾經是活躍訂閱者但已取消訂閱，您可以使用 `canceled` 方法：

    if ($user->subscription('default')->canceled()) {
        // ...
    }

您還可以判斷使用者是否已取消訂閱但仍在「寬限期」內，直到訂閱完全到期。例如，如果使用者在 3 月 5 日取消訂閱，而該訂閱原定於 3 月 10 日到期，則使用者將在 3 月 10 日之前處於「寬限期」內。請注意，在此期間 `subscribed` 方法仍返回 `true`：

    if ($user->subscription('default')->onGracePeriod()) {
        // ...
    }

要判斷使用者是否已取消訂閱且不再處於「寬限期」內，您可以使用 `ended` 方法：

    if ($user->subscription('default')->ended()) {
        // ...
    }

<a name="incomplete-and-past-due-status"></a>
#### 不完整和逾期狀態

如果訂閱在建立後需要二次付款操作，則訂閱將被標記
