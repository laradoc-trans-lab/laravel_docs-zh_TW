# Laravel Pennant

- [簡介](#introduction)
- [安裝](#installation)
- [設定](#configuration)
- [定義 Feature](#defining-features)
    - [基於類別的 Feature](#class-based-features)
- [檢查 Feature](#checking-features)
    - [條件式執行](#conditional-execution)
    - [`HasFeatures` Trait](#the-has-features-trait)
    - [Blade 指令](#blade-directive)
    - [中介層](#middleware)
    - [攔截 Feature 檢查](#intercepting-feature-checks)
    - [記憶體內快取](#in-memory-cache)
- [作用域](#scope)
    - [指定作用域](#specifying-the-scope)
    - [全域作用域](#global-scope)
    - [預設作用域](#default-scope)
    - [可為 Null 的作用域](#nullable-scope)
    - [識別作用域](#identifying-scope)
    - [序列化作用域](#serializing-scope)
- [豐富的 Feature 值](#rich-feature-values)
- [取得多個 Feature](#retrieving-multiple-features)
- [預載入](#eager-loading)
- [更新數值](#updating-values)
    - [整批更新](#bulk-updates)
    - [清除 Feature](#purging-features)
- [測試](#testing)
- [新增自訂 Pennant 驅動程式](#adding-custom-pennant-drivers)
    - [實作驅動程式](#implementing-the-driver)
    - [註冊驅動程式](#registering-the-driver)
    - [於外部定義 Feature](#defining-features-externally)
- [事件](#events)

<a name="introduction"></a>
## 簡介

[Laravel Pennant](https://github.com/laravel/pennant) 是一款簡潔輕量的功能旗標 (Feature flag) 套件——沒有多餘的雜質。功能旗標能讓您充滿自信地漸進式釋出新的應用程式功能、針對新的介面設計進行 A/B 測試、輔助主幹開發 (Trunk-based development) 策略等等。


<a name="installation"></a>
## 安裝

首先，使用 Composer 套件包管理器將 Pennant 安裝至您的專案中：

```shell
composer require laravel/pennant
```

接下來，您應該使用 `vendor:publish` Artisan 命令發布 Pennant 的設定檔與遷移 (Migration) 檔案：

```shell
php artisan vendor:publish --provider="Laravel\Pennant\PennantServiceProvider"
```

最後，您應該執行應用程式的資料庫遷移。這會建立一個 `features` 資料表，供 Pennant 的 `database` 驅動程式使用：

```shell
php artisan migrate
```


<a name="configuration"></a>
## 設定

發布 Pennant 的靜態資源後，其設定檔將位於 `config/pennant.php`。該設定檔允許您指定 Pennant 用於儲存解析後的 Feature Flag 數值之預設儲存機制。

Pennant 支援透過 `array` 驅動程式將解析後的 Feature Flag 數值儲存在記憶體陣列中；或者，Pennant 也可以透過 `database` 驅動程式將解析後的 Feature Flag 數值持久化儲存在關聯式資料庫中，這也是 Pennant 預設使用的儲存機制。


<a name="defining-features"></a>
## 定義 Feature

要定義 Feature，您可以使用 `Feature` Facade 所提供的 `define` 方法。您需要提供 Feature 的名稱，以及一個用於解析該 Feature 初始值的閉包 (Closure)。

通常，Feature 會在服務提供者(Service Providers)中使用 `Feature` Facade 來定義。該閉包將接收用於 Feature 檢查的「作用域 (Scope)」。最常見的作用域是目前通過認證的使用者。在此範例中，我們將定義一個 Feature，用來向應用程式的使用者漸進式推出新的 API：

```php
<?php

namespace App\Providers;

use App\Models\User;
use Illuminate\Support\Lottery;
use Illuminate\Support\ServiceProvider;
use Laravel\Pennant\Feature;

class AppServiceProvider extends ServiceProvider
{
    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        Feature::define('new-api', fn (User $user) => match (true) {
            $user->isInternalTeamMember() => true,
            $user->isHighTrafficCustomer() => false,
            default => Lottery::odds(1 / 100),
        });
    }
}
```

如您所見，我們為這個 Feature 設定了以下規則：

- 所有內部團隊成員都應該使用新的 API。
- 任何高流量客戶都不應該使用新的 API。
- 否則，該 Feature 將隨機指派給使用者，啟用的機率為 100 分之 1。

當第一次針對特定使用者檢查 `new-api` Feature 時，閉包的執行結果將由儲存驅動程式儲存起來。下一次針對同一個使用者檢查該 Feature 時，數值將直接從儲存空間中取得，且不會再次執行閉包。

為求方便，如果 Feature 的定義僅傳回抽獎 (Lottery) 結果，您可以完全省略閉包：

    Feature::define('site-redesign', Lottery::odds(1, 1000));


<a name="class-based-features"></a>
### 基於類別的 Feature

Pennant 也允許您定義基於類別的 Feature。與基於閉包的 Feature 定義不同，基於類別的 Feature 無需在服務提供者中註冊。要建立基於類別的 Feature，您可以執行 `pennant:feature` Artisan 命令。預設情況下，Feature 類別將放置在應用程式的 `app/Features` 目錄中：

```shell
php artisan pennant:feature NewApi
```

撰寫 Feature 類別時，您只需要定義一個 `resolve` 方法，該方法將被呼叫以解析給定作用域的 Feature 初始值。同樣地，作用域通常是目前通過認證的使用者：

```php
<?php

namespace App\Features;

use App\Models\User;
use Illuminate\Support\Lottery;

class NewApi
{
    /**
     * Resolve the feature's initial value.
     */
    public function resolve(User $user): mixed
    {
        return match (true) {
            $user->isInternalTeamMember() => true,
            $user->isHighTrafficCustomer() => false,
            default => Lottery::odds(1 / 100),
        };
    }
}
```

如果您想手動解析基於類別的 Feature 實例，可以呼叫 `Feature` Facade 上的 `instance` 方法：

```php
use Illuminate\Support\Facades\Feature;

$instance = Feature::instance(NewApi::class);
```

> [!NOTE]
> Feature 類別是透過 [服務容器](/docs/{{version}}/container) 解析的，因此您可以根據需要將依賴注入到 Feature 類別的建構子中。


#### 自訂儲存的 Feature 名稱

預設情況下，Pennant 會儲存 Feature 類別的完整類別名稱 (Fully Qualified Class Name)。如果您希望將儲存的 Feature 名稱與應用程式的內部結構解耦，可以在 Feature 類別上新增 `Name` 屬性 (Attribute)。該屬性的值將取代類別名稱進行儲存：

```php
<?php

namespace App\Features;

use Laravel\Pennant\Attributes\Name;

#[Name('new-api')]
class NewApi
{
    // ...
}
```

<a name="checking-features"></a>
## 檢查 Feature

若要判斷某個 Feature 是否啟用，您可以使用 `Feature` Facade 的 `active` 方法。預設情況下，Feature 是針對當前已認證的使用者進行檢查：

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Http\Response;
use Laravel\Pennant\Feature;

class PodcastController
{
    /**
     * Display a listing of the resource.
     */
    public function index(Request $request): Response
    {
        return Feature::active('new-api')
            ? $this->resolveNewApiResponse($request)
            : $this->resolveLegacyApiResponse($request);
    }

    // ...
}
```

雖然預設情況下是針對當前已認證的使用者來檢查 Feature，但您也可以輕鬆地針對其他使用者或[作用域](#scope)來檢查 Feature。若要做到這一點，請使用 `Feature` Facade 所提供的 `for` 方法：

```php
return Feature::for($user)->active('new-api')
    ? $this->resolveNewApiResponse($request)
    : $this->resolveLegacyApiResponse($request);
```

Pennant 還提供了一些額外的便利方法，在判斷 Feature 是否啟用時非常實用：

```php
// Determine if all of the given features are active...
Feature::allAreActive(['new-api', 'site-redesign']);

// Determine if any of the given features are active...
Feature::someAreActive(['new-api', 'site-redesign']);

// Determine if a feature is inactive...
Feature::inactive('new-api');

// Determine if all of the given features are inactive...
Feature::allAreInactive(['new-api', 'site-redesign']);

// Determine if any of the given features are inactive...
Feature::someAreInactive(['new-api', 'site-redesign']);
```

> [!NOTE]
> 當在 HTTP 上下文（Context）之外使用 Pennant 時（例如在 Artisan 指令或佇列任務中），您通常應該[明確指定 Feature 的作用域](#specifying-the-scope)。或者，您也可以定義一個兼顧已認證 HTTP 上下文與未認證上下文的[預設作用域](#default-scope)。

<a name="checking-class-based-features"></a>
#### 檢查基於類別的 Feature

對於基於類別的 Feature，您在檢查 Feature 時應提供類別名稱：

```php
<?php

namespace App\Http\Controllers;

use App\Features\NewApi;
use Illuminate\Http\Request;
use Illuminate\Http\Response;
use Laravel\Pennant\Feature;

class PodcastController
{
    /**
     * Display a listing of the resource.
     */
    public function index(Request $request): Response
    {
        return Feature::active(NewApi::class)
            ? $this->resolveNewApiResponse($request)
            : $this->resolveLegacyApiResponse($request);
    }

    // ...
}
```

<a name="conditional-execution"></a>
### 條件式執行

`when` 方法可以用於在 Feature 為啟用狀態時流暢地執行給定的 Closure。此外，還可以提供第二個 Closure，會在 Feature 為未啟用狀態時執行：

```php
<?php

namespace App\Http\Controllers;

use App\Features\NewApi;
use Illuminate\Http\Request;
use Illuminate\Http\Response;
use Laravel\Pennant\Feature;

class PodcastController
{
    /**
     * Display a listing of the resource.
     */
    public function index(Request $request): Response
    {
        return Feature::when(NewApi::class,
            fn () => $this->resolveNewApiResponse($request),
            fn () => $this->resolveLegacyApiResponse($request),
        );
    }

    // ...
}
```

`unless` 方法的作用與 `when` 方法相反，當 Feature 為未啟用狀態時，會執行第一個 Closure：

```php
return Feature::unless(NewApi::class,
    fn () => $this->resolveLegacyApiResponse($request),
    fn () => $this->resolveNewApiResponse($request),
);
```

<a name="the-has-features-trait"></a>
### `HasFeatures` Trait

可以將 Pennant 的 `HasFeatures` Trait 新增到您應用程式的 `User` Model（或任何其他具有 Feature 的 Model）中，以提供直接從 Model 檢查 Feature 的流暢且便利的方法：

```php
<?php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Laravel\Pennant\Concerns\HasFeatures;

class User extends Authenticatable
{
    use HasFeatures;

    // ...
}
```

將該 Trait 新增到您的 Model 後，您可以透過呼叫 `features` 方法輕鬆地檢查 Feature：

```php
if ($user->features()->active('new-api')) {
    // ...
}
```

當然，`features` 方法還提供了許多其他方便的方法來與 Feature 進行互動：

```php
// Values...
$value = $user->features()->value('purchase-button')
$values = $user->features()->values(['new-api', 'purchase-button']);

// State...
$user->features()->active('new-api');
$user->features()->allAreActive(['new-api', 'server-api']);
$user->features()->someAreActive(['new-api', 'server-api']);

$user->features()->inactive('new-api');
$user->features()->allAreInactive(['new-api', 'server-api']);
$user->features()->someAreInactive(['new-api', 'server-api']);

// Conditional execution...
$user->features()->when('new-api',
    fn () => /* ... */,
    fn () => /* ... */,
);

$user->features()->unless('new-api',
    fn () => /* ... */,
    fn () => /* ... */,
);
```

<a name="blade-directive"></a>
### Blade 指令

為了讓在 Blade 中檢查 Feature 獲得無縫體驗，Pennant 提供了 `@feature` 與 `@featureany` 指令：

```blade
@feature('site-redesign')
    <!-- 'site-redesign' is active -->
@else
    <!-- 'site-redesign' is inactive -->
@endfeature

@featureany(['site-redesign', 'beta'])
    <!-- 'site-redesign' or `beta` is active -->
@endfeatureany
```

<a name="middleware"></a>
### 中介層

Pennant 還包含一個[中介層](/docs/{{version}}/middleware)，可以在執行路由之前驗證當前已認證的使用者是否擁有存取某個 Feature 的權限。您可以將該中介層指派給路由，並指定存取該路由所需的 Feature。如果指定的任何 Feature 對當前已認證的使用者來說為未啟用狀態，該路由將會回傳 `400 Bad Request` 的 HTTP 回應。您可以將多個 Feature 傳遞給靜態的 `using` 方法。

```php
use Illuminate\Support\Facades\Route;
use Laravel\Pennant\Middleware\EnsureFeaturesAreActive;

Route::get('/api/servers', function () {
    // ...
})->middleware(EnsureFeaturesAreActive::using('new-api', 'servers-api'));
```

<a name="customizing-the-response"></a>
#### 自訂回應

如果您想要自訂當列出的其中一個 Feature 為未啟用狀態時，由中介層回傳的回應，您可以使用 `EnsureFeaturesAreActive` 中介層提供的 `whenInactive` 方法。通常，此方法應該在應用程式某個服務提供者的 `boot` 方法內呼叫：

```php
use Illuminate\Http\Request;
use Illuminate\Http\Response;
use Laravel\Pennant\Middleware\EnsureFeaturesAreActive;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    EnsureFeaturesAreActive::whenInactive(
        function (Request $request, array $features) {
            return new Response(status: 403);
        }
    );

    // ...
}
```

<a name="intercepting-feature-checks"></a>
### 攔截 Feature 檢查

有時在取得特定 Feature 的儲存值之前，先進行一些記憶體內檢查會很有幫助。想像一下，您正在開發一個隱藏在 Feature Flag 後面的新 API，並希望能夠停用該新 API，但又不會失去儲存空間中任何已解析的 Feature 值。若您發現新 API 中有 Bug，您可以輕鬆地為除了內部團隊成員以外的所有人停用它、修復該 Bug，然後為先前能夠存取該 Feature 的使用者重新啟用該新 API。

您可以透過 [基於類別的 Feature](#class-based-features) 的 `before` 方法來實現此目的。當存在 `before` 方法時，它總是在從儲存空間讀取值之前於記憶體中執行。若該方法回傳了非 `null` 的值，則在該次請求的持續期間內，該值將被用來替代 Feature 的儲存值：

```php
<?php

namespace App\Features;

use App\Models\User;
use Illuminate\Support\Facades\Config;
use Illuminate\Support\Lottery;

class NewApi
{
    /**
     * Run an always-in-memory check before the stored value is retrieved.
     */
    public function before(User $user): mixed
    {
        if (Config::get('features.new-api.disabled')) {
            return $user->isInternalTeamMember();
        }
    }

    /**
     * Resolve the feature's initial value.
     */
    public function resolve(User $user): mixed
    {
        return match (true) {
            $user->isInternalTeamMember() => true,
            $user->isHighTrafficCustomer() => false,
            default => Lottery::odds(1 / 100),
        };
    }
}
```

您也可以使用此功能來排定先前隱藏在 Feature Flag 後面的 Feature 進行全域推出的時程：

```php
<?php

namespace App\Features;

use Illuminate\Support\Carbon;
use Illuminate\Support\Facades\Config;

class NewApi
{
    /**
     * Run an always-in-memory check before the stored value is retrieved.
     */
    public function before(User $user): mixed
    {
        if (Config::get('features.new-api.disabled')) {
            return $user->isInternalTeamMember();
        }

        if (Carbon::parse(Config::get('features.new-api.rollout-date'))->isPast()) {
            return true;
        }
    }

    // ...
}
```

<a name="in-memory-cache"></a>
### 記憶體內快取

當檢查 Feature 時，Pennant 會為結果建立記憶體內快取。若您使用的是 `database` 驅動程式，這意味著在單一請求內重複檢查相同的 Feature Flag 將不會觸發額外的資料庫查詢。這也能確保該 Feature 在該次請求期間保持一致的結果。

若您需要手動清除記憶體內快取，可以使用 `Feature` Facade 提供的 `flushCache` 方法：

```php
Feature::flushCache();
```

<a name="scope"></a>
## 作用域


<a name="specifying-the-scope"></a>
### 指定作用域

如前所述，Feature 通常是針對目前已認證的使用者進行檢查。然而，這可能並不總是符合您的需求。因此，您可以透過 `Feature` Facade 的 `for` 方法，指定您想要針對哪個作用域來檢查給定的 Feature：

```php
return Feature::for($user)->active('new-api')
    ? $this->resolveNewApiResponse($request)
    : $this->resolveLegacyApiResponse($request);
```

當然，Feature 作用域並不局限於「使用者」。想像一下，您建立了一個新的計費功能體驗，並且希望向整個團隊而非單一使用者逐步推出。或許您希望最老舊的團隊比較新的團隊更慢獲得更新。您的 Feature 解析 Closure 可能會像這樣：

```php
use App\Models\Team;
use Illuminate\Support\Carbon;
use Illuminate\Support\Lottery;
use Laravel\Pennant\Feature;

Feature::define('billing-v2', function (Team $team) {
    if ($team->created_at->isAfter(new Carbon('1st Jan, 2023'))) {
        return true;
    }

    if ($team->created_at->isAfter(new Carbon('1st Jan, 2019'))) {
        return Lottery::odds(1 / 100);
    }

    return Lottery::odds(1 / 1000);
});
```

您會注意到我們定義的 Closure 並不接收 `User`，而是接收 `Team` Model。若要判斷此 Feature 對於使用者的團隊是否為啟用狀態，您應該將該團隊傳遞給 `Feature` Facade 所提供的 `for` 方法：

```php
if (Feature::for($user->team)->active('billing-v2')) {
    return redirect('/billing/v2');
}

// ...
```


<a name="global-scope"></a>
### 全域作用域

若要使用全域作用域來檢查 Feature 或與其進行互動，而不受設定的預設作用域解析器影響，可以使用 `globally` 方法。這對於全應用程式範圍的 Feature Flag 非常實用，例如暫時啟用維護行為或向所有使用者推出某個 Feature：

```php
Feature::globally()->active('new-api');

Feature::globally()->activate('new-api');
```


<a name="default-scope"></a>
### 預設作用域

您也可以自訂 Pennant 用於檢查 Feature 的預設作用域。例如，可能您所有的 Feature 都是針對目前已認證使用者的團隊而非使用者本人進行檢查。與其每次檢查 Feature 時都要呼叫 `Feature::for($user->team)`，不如直接將團隊指定為預設作用域。通常，這應該在您應用程式的服務提供者(Service Providers)其中之一進行：

```php
<?php

namespace App\Providers;

use Illuminate\Support\Facades\Auth;
use Illuminate\Support\ServiceProvider;
use Laravel\Pennant\Feature;

class AppServiceProvider extends ServiceProvider
{
    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        Feature::resolveScopeUsing(fn ($driver) => Auth::user()?->team);

        // ...
    }
}
```

如果沒有透過 `for` 方法明確提供作用域，Feature 檢查現在將會使用目前已認證使用者的團隊作為預設作用域：

```php
Feature::active('billing-v2');

// Is now equivalent to...

Feature::for($user->team)->active('billing-v2');
```


<a name="nullable-scope"></a>
### 可為 Null 的作用域

如果在檢查 Feature 時您傳入的作用域為 `null`，且該 Feature 的定義並未透過可為 Null 型別（Nullable Type）或在聯合型別（Union Type）中包含 `null` 來支援 `null`，Pennant 將會自動回傳 `false` 作為該 Feature 的結果值。

因此，如果您傳遞給 Feature 的作用域有可能為 `null`，且您希望執行該 Feature 的數值解析器，您應該在 Feature 的定義中考量到這一點。如果您在 Artisan 指令、佇列任務（Queued Job）或未認證的路由中檢查 Feature，就可能會出現 `null` 作用域。因為在這些情境下通常沒有已認證的使用者，所以預設作用域將會是 `null`。

如果您並非總是[明確指定 Feature 的作用域](#specifying-the-scope)，那麼您應該確保作用域的型別是「可為 Null」，並在您的 Feature 定義邏輯中處理 `null` 作用域數值：

```php
use App\Models\User;
use Illuminate\Support\Lottery;
use Laravel\Pennant\Feature;

Feature::define('new-api', fn (User $user) => match (true) {// [tl! remove]
Feature::define('new-api', fn (User|null $user) => match (true) {// [tl! add]
    $user === null => true,// [tl! add]
    $user->isInternalTeamMember() => true,
    $user->isHighTrafficCustomer() => false,
    default => Lottery::odds(1 / 100),
});
```


<a name="identifying-scope"></a>
### 識別作用域

Pennant 內建的 `array` 和 `database` 儲存驅動程式知道如何為所有 PHP 資料型別以及 Eloquent Model 正確儲存作用域識別碼。然而，如果您的應用程式使用了第三方 Pennant 驅動程式，該驅動程式可能不知道如何正確地儲存 Eloquent Model 或您應用程式中其他自訂型別的識別碼。

鑑於此，Pennant 允許您透過在應用程式中用作 Pennant 作用域的物件上實作 `FeatureScopeable` 契約(Contracts)，來格式化用於儲存的作用域數值。

例如，想像您在單一應用程式中使用兩種不同的 Feature 驅動程式：內建的 `database` 驅動程式和第三方的 "Flag Rocket" 驅動程式。"Flag Rocket" 驅動程式不知道如何正確儲存 Eloquent Model。相反地，它需要一個 `FlagRocketUser` 實例。透過實作 `FeatureScopeable` 契約(Contracts)所定義的 `toFeatureIdentifier`，我們可以自訂提供給應用程式所使用的每個驅動程式的可儲存作用域數值：

```php
<?php

namespace App\Models;

use FlagRocket\FlagRocketUser;
use Illuminate\Database\Eloquent\Model;
use Laravel\Pennant\Contracts\FeatureScopeable;

class User extends Model implements FeatureScopeable
{
    /**
     * Cast the object to a feature scope identifier for the given driver.
     */
    public function toFeatureIdentifier(string $driver): mixed
    {
        return match($driver) {
            'database' => $this,
            'flag-rocket' => FlagRocketUser::fromId($this->flag_rocket_id),
        };
    }
}
```


<a name="serializing-scope"></a>
### 序列化作用域

預設情況下，當儲存與 Eloquent Model 相關聯的 Feature 時，Pennant 會使用完全合格的類別名稱（Fully Qualified Class Name）。如果您已經在使用 [Eloquent 多型對照表 (Morph Map)](/docs/{{version}}/eloquent-relationships#custom-polymorphic-types)，您可以選擇讓 Pennant 也使用 Morph Map，以解耦儲存的 Feature 與您的應用程式架構。

若要達成此目的，在服務提供者(Service Providers)中定義您的 Eloquent Morph Map 之後，您可以呼叫 `Feature` Facade 的 `useMorphMap` 方法：

```php
use Illuminate\Database\Eloquent\Relations\Relation;
use Laravel\Pennant\Feature;

Relation::enforceMorphMap([
    'post' => 'App\Models\Post',
    'video' => 'App\Models\Video',
]);

Feature::useMorphMap();
```

<a name="rich-feature-values"></a>
## 豐富的 Feature 值

到目前為止，我們主要展示了 Feature 處於二元狀態（即「active」或「inactive」），但 Pennant 也允許您儲存更豐富的值。

例如，假設您正在為應用程式中的「Buy now」按鈕測試三種新顏色。您可以從 Feature 定義中傳回一個字串，而不是傳回 `true` 或 `false`：

```php
use Illuminate\Support\Arr;
use Laravel\Pennant\Feature;

Feature::define('purchase-button', fn (User $user) => Arr::random([
    'blue-sapphire',
    'seafoam-green',
    'tart-orange',
]));
```

您可以使用 `value` 方法來取得 `purchase-button` Feature 的值：

```php
$color = Feature::value('purchase-button');
```

Pennant 內建的 Blade 指令也讓您能輕鬆根據 Feature 的當前值來條件式渲染內容：

```blade
@feature('purchase-button', 'blue-sapphire')
    <!-- 'blue-sapphire' is active -->
@elsefeature('purchase-button', 'seafoam-green')
    <!-- 'seafoam-green' is active -->
@elsefeature('purchase-button', 'tart-orange')
    <!-- 'tart-orange' is active -->
@endfeature
```

> [!NOTE]
> 使用豐富的值時，請特別注意：只要 Feature 的值不為 `false`，就被視為「active」。

當呼叫[條件式 `when`](#conditional-execution) 方法時，Feature 的豐富數值將會傳遞給第一個閉包：

```php
Feature::when('purchase-button',
    fn ($color) => /* ... */,
    fn () => /* ... */,
);
```

同樣地，當呼叫條件式 `unless` 方法時，Feature 的豐富數值將會傳遞給選填的第二個閉包：

```php
Feature::unless('purchase-button',
    fn () => /* ... */,
    fn ($color) => /* ... */,
);
```

<a name="retrieving-multiple-features"></a>
## 取得多個 Feature

`values` 方法允許您取得指定作用域下的多個 Feature：

```php
Feature::values(['billing-v2', 'purchase-button']);

// [
//     'billing-v2' => false,
//     'purchase-button' => 'blue-sapphire',
// ]
```

或者，您可以使用 `all` 方法來取得指定作用域下所有已定義 Feature 的值：

```php
Feature::all();

// [
//     'billing-v2' => false,
//     'purchase-button' => 'blue-sapphire',
//     'site-redesign' => true,
// ]
```

然而，基於類別的 Feature 是動態註冊的，在被明確檢查之前，Pennant 並不知道它們的存在。這意味著如果您的基於類別的 Feature 在當前請求期間尚未被檢查過，它們可能不會出現在 `all` 方法傳回的結果中。

若您想確保在使用 `all` 方法時總是包含 Feature 類別，您可以使用 Pennant 的 Feature 自動探索（Feature discovery）功能。首先，請在您應用程式的其中一個服務提供者中呼叫 `discover` 方法：

```php
<?php

namespace App\Providers;

use Illuminate\Support\ServiceProvider;
use Laravel\Pennant\Feature;

class AppServiceProvider extends ServiceProvider
{
    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        Feature::discover();

        // ...
    }
}
```

`discover` 方法將會註冊您應用程式 `app/Features` 目錄中的所有 Feature 類別。現在，無論這些類別是否已在當前請求中被檢查過，`all` 方法傳回的結果都會包含它們：

```php
Feature::all();

// [
//     'App\Features\NewApi' => true,
//     'billing-v2' => false,
//     'purchase-button' => 'blue-sapphire',
//     'site-redesign' => true,
// ]
```

<a name="eager-loading"></a>
## 預載入

雖然 Pennant 會在單一請求中將所有解析出的 Feature 保存在記憶體內快取中，但仍有可能遇到效能問題。為了減輕這種狀況，Pennant 提供了預載入 Feature 值的機能。

為了說明這一點，假設我們在迴圈中檢查某個 Feature 是否處於啟用狀態（active）：

```php
use Laravel\Pennant\Feature;

foreach ($users as $user) {
    if (Feature::for($user)->active('notifications-beta')) {
        $user->notify(new RegistrationSuccess);
    }
}
```

假設我們使用的是資料庫驅動程式，這段程式碼會為迴圈中的每個使用者執行一次資料庫查詢，可能導致執行數百次查詢。不過，使用 Pennant 的 `load` 方法，我們可以透過預載入一整組使用者或作用域的 Feature 值，來消除這個潛在的效能瓶頸：

```php
Feature::for($users)->load(['notifications-beta']);

foreach ($users as $user) {
    if (Feature::for($user)->active('notifications-beta')) {
        $user->notify(new RegistrationSuccess);
    }
}
```

若要僅在 Feature 值尚未被載入時才進行載入，您可以使用 `loadMissing` 方法：

```php
Feature::for($users)->loadMissing([
    'new-api',
    'purchase-button',
    'notifications-beta',
]);
```

您可以使用 `loadAll` 方法來載入所有已定義的 Feature：

```php
Feature::for($users)->loadAll();
```

<a name="updating-values"></a>
## 更新數值

當 feature 的數值第一次被解析時，底層的驅動程式會將結果儲存至儲存空間中。這通常是為了確保使用者跨請求的一致體驗。然而，有時您可能想要手動更新 feature 儲存的數值。

若要達成此目的，您可以使用 `activate` 和 `deactivate` 方法來切換 feature 的「開啟」或「關閉」：

```php
use Laravel\Pennant\Feature;

// Activate the feature for the default scope...
Feature::activate('new-api');

// Deactivate the feature for the given scope...
Feature::for($user->team)->deactivate('billing-v2');
```

您也可以透過為 `activate` 方法提供第二個引數，手動為 feature 設定豐富的數值：

```php
Feature::activate('purchase-button', 'seafoam-green');
```

若要指示 Pennant 忘記 feature 的儲存值，您可以使用 `forget` 方法。當再次檢查該 feature 時，Pennant 會從其 feature 定義中重新解析 feature 的數值：

```php
Feature::forget('purchase-button');
```


<a name="bulk-updates"></a>
### 整批更新

若要整批更新已儲存的 feature 數值，您可以使用 `activateForEveryone` 和 `deactivateForEveryone` 方法。

例如，想像您現在對 `new-api` feature 的穩定性非常有信心，並且已決定了結帳流程中最佳的 `'purchase-button'` 顏色——您可以相應地更新所有使用者的儲存值：

```php
use Laravel\Pennant\Feature;

Feature::activateForEveryone('new-api');

Feature::activateForEveryone('purchase-button', 'seafoam-green');
```

或者，您可以為所有使用者停用該 feature：

```php
Feature::deactivateForEveryone('new-api');
```

> [!NOTE]
> 這只會更新由 Pennant 儲存驅動程式所儲存的解析後 feature 數值。您還需要更新應用程式中的 feature 定義。


<a name="purging-features"></a>
### 清除 Feature

有時，清除儲存空間中的整個 feature 是很有用的。如果您已從應用程式中移除該 feature，或者對 feature 的定義進行了調整且希望向所有使用者發布，這通常是必要的。

您可以使用 `purge` 方法移除 feature 的所有已儲存值：

```php
// Purging a single feature...
Feature::purge('new-api');

// Purging multiple features...
Feature::purge(['new-api', 'purchase-button']);
```

如果您想從儲存空間中清除 _所有_ feature，可以在呼叫 `purge` 方法時不帶任何引數：

```php
Feature::purge();
```

由於清除 feature 作為應用程式部署流程的一部分非常有用，Pennant 包含了一個 `pennant:purge` Artisan 指令，它可以清除儲存空間中指定的 feature：

```shell
php artisan pennant:purge new-api

php artisan pennant:purge new-api purchase-button
```

還可以清除 _除了_ 給定 feature 列表之外的所有 feature。例如，假設您想清除所有 feature，但保留儲存空間中 "new-api" 和 "purchase-button" feature 的數值。若要做到這一點，您可以將這些 feature 名稱傳遞給 `--except` 選項：

```shell
php artisan pennant:purge --except=new-api --except=purchase-button
```

為了方便起見，`pennant:purge` 指令還支援 `--except-registered` 標誌。此標誌表示除了在服務提供者中明確註冊的 feature 之外，應清除所有 feature：

```shell
php artisan pennant:purge --except-registered
```


<a name="testing"></a>
## 測試

當測試與 Feature Flag 互動的程式碼時，在測試中控制 Feature Flag 回傳值最簡單的方法就是重新定義該 feature。例如，假設您在應用程式的某個服務提供者中定義了以下 feature：

```php
use Illuminate\Support\Arr;
use Laravel\Pennant\Feature;

Feature::define('purchase-button', fn () => Arr::random([
    'blue-sapphire',
    'seafoam-green',
    'tart-orange',
]));
```

若要在測試中修改 feature 的回傳值，您可以在測試開始時重新定義該 feature。即使服務提供者中仍然存在 `Arr::random()` 的實作，以下測試也總是會通過：

```php tab=Pest
use Laravel\Pennant\Feature;

test('it can control feature values', function () {
    Feature::define('purchase-button', 'seafoam-green');

    expect(Feature::value('purchase-button'))->toBe('seafoam-green');
});
```

```php tab=PHPUnit
use Laravel\Pennant\Feature;

public function test_it_can_control_feature_values()
{
    Feature::define('purchase-button', 'seafoam-green');

    $this->assertSame('seafoam-green', Feature::value('purchase-button'));
}
```

基於類別的 feature 也可以使用相同的方法：

```php tab=Pest
use Laravel\Pennant\Feature;

test('it can control feature values', function () {
    Feature::define(NewApi::class, true);

    expect(Feature::value(NewApi::class))->toBeTrue();
});
```

```php tab=PHPUnit
use App\Features\NewApi;
use Laravel\Pennant\Feature;

public function test_it_can_control_feature_values()
{
    Feature::define(NewApi::class, true);

    $this->assertTrue(Feature::value(NewApi::class));
}
```

如果您的 feature 回傳的是 `Lottery` 實例，這裡有一些有用的[測試輔助函式可以使用](/docs/{{version}}/helpers#testing-lotteries)。


<a name="store-configuration"></a>
#### 儲存設定

您可以在應用程式的 `phpunit.xml` 檔案中定義 `PENNANT_STORE` 環境變數，以設定 Pennant 在測試期間將使用的儲存區：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<phpunit colors="true">
    <!-- ... -->
    <php>
        <env name="PENNANT_STORE" value="array"/>
        <!-- ... -->
    </php>
</phpunit>
```

<a name="adding-custom-pennant-drivers"></a>
## 新增自訂 Pennant 驅動程式


<a name="implementing-the-driver"></a>
#### 實作驅動程式

如果 Pennant 現有的儲存驅動程式都不符合您應用程式的需求，您可以撰寫自己的儲存驅動程式。您的自訂驅動程式應實作 `Laravel\Pennant\Contracts\Driver` 介面：

```php
<?php

namespace App\Extensions;

use Laravel\Pennant\Contracts\Driver;

class RedisFeatureDriver implements Driver
{
    public function define(string $feature, callable $resolver): void {}
    public function defined(): array {}
    public function getAll(array $features): array {}
    public function get(string $feature, mixed $scope): mixed {}
    public function set(string $feature, mixed $scope, mixed $value): void {}
    public function setForAllScopes(string $feature, mixed $value): void {}
    public function delete(string $feature, mixed $scope): void {}
    public function purge(array|null $features): void {}
}
```

現在，我們只需要使用 Redis 連線來實作這些方法。關於如何實作每個方法的範例，請參考 [Pennant 原始碼](https://github.com/laravel/pennant/blob/1.x/src/Drivers/DatabaseDriver.php) 中的 `Laravel\Pennant\Drivers\DatabaseDriver`。

> [!NOTE]
> Laravel 預設並未提供放置擴充套件的目錄。您可以自由將它們放在任何您喜歡的地方。在這個範例中，我們建立了一個 `Extensions` 目錄來放置 `RedisFeatureDriver`。


<a name="registering-the-driver"></a>
#### 註冊驅動程式

當您的驅動程式實作完成後，就可以將其註冊到 Laravel 中。若要向 Pennant 新增額外驅動程式，您可以使用 `Feature` Facade 所提供的 `extend` 方法。您應該在應用程式的某個[服務提供者(Service Providers)](/docs/{{version}}/providers)的 `boot` 方法中呼叫 `extend` 方法：

```php
<?php

namespace App\Providers;

use App\Extensions\RedisFeatureDriver;
use Illuminate\Contracts\Foundation\Application;
use Illuminate\Support\ServiceProvider;
use Laravel\Pennant\Feature;

class AppServiceProvider extends ServiceProvider
{
    /**
     * Register any application services.
     */
    public function register(): void
    {
        // ...
    }

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        Feature::extend('redis', function (Application $app) {
            return new RedisFeatureDriver($app->make('redis'), $app->make('events'), []);
        });
    }
}
```

當驅動程式註冊完成後，您就可以在應用程式的 `config/pennant.php` 設定檔中使用 `redis` 驅動程式：

```php
'stores' => [

    'redis' => [
        'driver' => 'redis',
        'connection' => null,
    ],

    // ...

],
```


<a name="defining-features-externally"></a>
### 於外部定義 Feature

如果您的驅動程式是包裝第三方 Feature Flag 平台，您可能會在該平台上定義 Feature，而不是使用 Pennant 的 `Feature::define` 方法。若是這種情況，您的自訂驅動程式還應該實作 `Laravel\Pennant\Contracts\DefinesFeaturesExternally` 介面：

```php
<?php

namespace App\Extensions;

use Laravel\Pennant\Contracts\Driver;
use Laravel\Pennant\Contracts\DefinesFeaturesExternally;

class FeatureFlagServiceDriver implements Driver, DefinesFeaturesExternally
{
    /**
     * Get the features defined for the given scope.
     */
    public function definedFeaturesForScope(mixed $scope): array {}

    /* ... */
}
```

`definedFeaturesForScope` 方法應傳回為給定作用域定義的 Feature 名稱清單。


<a name="events"></a>
## 事件

Pennant 會分派各種事件，這在追蹤整個應用程式中的 Feature Flag 時非常有用。


### `Laravel\Pennant\Events\FeatureRetrieved`

每當 [Feature 被檢查](#checking-features)時就會分派此事件。此事件可用於建立並追蹤整個應用程式中 Feature Flag 的使用指標。


### `Laravel\Pennant\Events\FeatureResolved`

當 Feature 的數值首次針對特定作用域被解析時，就會分派此事件。


### `Laravel\Pennant\Events\UnknownFeatureResolved`

當未知的 Feature 首次針對特定作用域被解析時，就會分派此事件。如果您本意是要移除某個 Feature Flag，但意外地在應用程式各處殘留了對它的引用，監聽此事件可能會很有幫助：

```php
<?php

namespace App\Providers;

use Illuminate\Support\ServiceProvider;
use Illuminate\Support\Facades\Event;
use Illuminate\Support\Facades\Log;
use Laravel\Pennant\Events\UnknownFeatureResolved;

class AppServiceProvider extends ServiceProvider
{
    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        Event::listen(function (UnknownFeatureResolved $event) {
            Log::error("Resolving unknown feature [{$event->feature}].");
        });
    }
}
```


### `Laravel\Pennant\Events\DynamicallyRegisteringFeatureClass`

當[基於類別的 Feature](#class-based-features) 在請求期間首次被動態檢查時，就會分派此事件。


### `Laravel\Pennant\Events\UnexpectedNullScopeEncountered`

當 `null` 作用域傳遞給[不支援 null](#nullable-scope) 的 Feature 定義時，就會分派此事件。

這種情況會被優雅地處理，並且 Feature 將傳回 `false`。但是，如果您希望停用此 Feature 的預設優雅行為，可以在應用程式的 `AppServiceProvider` 之 `boot` 方法中註冊此事件的監聽器：

```php
use Illuminate\Support\Facades\Log;
use Laravel\Pennant\Events\UnexpectedNullScopeEncountered;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Event::listen(UnexpectedNullScopeEncountered::class, fn () => abort(500));
}
```


### `Laravel\Pennant\Events\FeatureUpdated`

當為某個作用域更新 Feature 時，通常是透過呼叫 `activate` 或 `deactivate`，就會分派此事件。


### `Laravel\Pennant\Events\FeatureUpdatedForAllScopes`

當為所有作用域更新 Feature 時，通常是透過呼叫 `activateForEveryone` 或 `deactivateForEveryone`，就會分派此事件。


### `Laravel\Pennant\Events\FeatureDeleted`

當刪除某個作用域的 Feature 時，通常是透過呼叫 `forget`，就會分派此事件。


### `Laravel\Pennant\Events\FeaturesPurged`

當清除特定的 Feature 時，就會分派此事件。


### `Laravel\Pennant\Events\AllFeaturesPurged`

當清除所有 Feature 時，就會分派此事件。