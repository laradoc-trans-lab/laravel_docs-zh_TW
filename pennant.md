# Laravel Pennant

- [簡介](#introduction)
- [安裝](#installation)
- [設定](#configuration)
- [定義 Feature](#defining-features)
    - [基於 Class 的 Feature](#class-based-features)
- [檢查 Feature](#checking-features)
    - [條件式執行](#conditional-execution)
    - [`HasFeatures` Trait](#the-has-features-trait)
    - [Blade 指令](#blade-directive)
    - [中介層](#middleware)
    - [攔截 Feature 檢查](#intercepting-feature-checks)
    - [記憶體快取](#in-memory-cache)
- [Scope](#scope)
    - [指定 Scope](#specifying-the-scope)
    - [預設 Scope](#default-scope)
    - [可為空的 Scope](#nullable-scope)
    - [識別 Scope](#identifying-scope)
    - [序列化 Scope](#serializing-scope)
- [豐富的 Feature 值](#rich-feature-values)
- [取回多個 Feature](#retrieving-multiple-features)
- [預載入](#eager-loading)
- [更新值](#updating-values)
    - [批次更新](#bulk-updates)
    - [清除 Feature](#purging-features)
- [測試](#testing)
- [新增自訂 Pennant 驅動](#adding-custom-pennant-drivers)
    - [實作驅動](#implementing-the-driver)
    - [註冊驅動](#registering-the-driver)
    - [在外部定義 Feature](#defining-features-externally)
- [事件](#events)

<a name="introduction"></a>
## 簡介

[Laravel Pennant](https://github.com/laravel/pennant) 是一個簡單輕巧的功能旗標套件——沒有多餘的累贅。功能旗標讓您能夠自信地逐步推出新的應用程式功能、A/B 測試新的介面設計、補充主幹式開發策略等等。


<a name="installation"></a>
## 安裝

首先，請使用 Composer 套件管理器在您的專案中安裝 Pennant：

```shell
composer require laravel/pennant
```

接著，您應該使用 `vendor:publish` Artisan 指令發佈 Pennant 的設定檔和遷移檔：

```shell
php artisan vendor:publish --provider="Laravel\Pennant\PennantServiceProvider"
```

最後，您應該執行應用程式的資料庫遷移。這將會建立一個 `features` 資料表，Pennant 將其用於支援 `database` 驅動：

```shell
php artisan migrate
```


<a name="configuration"></a>
## 設定

發佈 Pennant 資源後，其設定檔將位於 `config/pennant.php`。此設定檔允許您指定 Pennant 用來儲存已解析的功能旗標值的預設儲存機制。

Pennant 支援透過 `array` 驅動將已解析的功能旗標值儲存在記憶體陣列中。或者，Pennant 可以透過 `database` 驅動將已解析的功能旗標值永久儲存在關聯式資料庫中，此為 Pennant 使用的預設儲存機制。


<a name="defining-features"></a>
## 定義 Feature

要定義 Feature，您可以使用 `Feature` facade 提供的 `define` 方法。您需要提供 Feature 的名稱，以及一個將被呼叫來解析 Feature 初始值的閉包。

通常，Feature 會使用 `Feature` facade 在 service provider 中定義。閉包將會接收 Feature 檢查的「Scope」。最常見的是，Scope 為目前通過身分驗證的 user。在此範例中，我們將定義一個 Feature，以逐步向應用程式的 user 推出新的 API：

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

如您所見，我們的 Feature 有以下規則：

- 所有內部團隊成員都應該使用新的 API。
- 任何高流量的客戶都不應該使用新的 API。
- 否則，該 Feature 應隨機分配給 user，有 1% 的機會啟用。

首次為給定 user 檢查 `new-api` Feature 時，閉包的結果將由儲存驅動儲存。下次為同一 user 檢查 Feature 時，該值將從儲存中取回，且閉包將不會被呼叫。

為方便起見，如果 Feature 定義僅回傳一個 lottery，您可以完全省略閉包：

    Feature::define('site-redesign', Lottery::odds(1, 1000));


<a name="class-based-features"></a>
### 基於 Class 的 Feature

Pennant 也允許您定義基於 Class 的 Feature。與基於閉包的 Feature 定義不同，無需在 service provider 中註冊基於 Class 的 Feature。要建立基於 Class 的 Feature，您可以呼叫 `pennant:feature` Artisan 指令。預設情況下，Feature Class 將放置在您應用程式的 `app/Features` 目錄中：

```shell
php artisan pennant:feature NewApi
```

編寫 Feature Class 時，您只需要定義一個 `resolve` 方法，該方法將被呼叫以解析給定 Scope 的 Feature 初始值。同樣地，Scope 通常會是目前通過身分驗證的 user：

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

如果您想手動解析基於 Class 的 Feature 實例，您可以呼叫 `Feature` facade 上的 `instance` 方法：

```php
use Illuminate\Support\Facades\Feature;

$instance = Feature::instance(NewApi::class);
```

> [!NOTE]
> Feature Class 是透過 [container](/docs/{{version}}/container) 解析的，因此您可以根據需要將依賴注入 Feature Class 的建構函式中。


#### 自訂儲存的 Feature 名稱

預設情況下，Pennant 將儲存 Feature Class 的完整限定類別名稱。如果您想將儲存的 Feature 名稱與應用程式的內部結構解耦，您可以在 Feature Class 上指定一個 `$name` 屬性。此屬性的值將取代 Class 名稱進行儲存：

```php
<?php

namespace App\Features;

class NewApi
{
    /**
     * The stored name of the feature.
     *
     * @var string
     */
    public $name = 'new-api';

    // ...
}
```

<a name="checking-features"></a>
## 檢查 Feature

若要判斷 Feature 是否啟用中，您可以使用 `Feature` facade 提供的 `active` 方法。預設情況下，Feature 會根據目前已驗證的使用者進行檢查：

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

儘管預設情況下 Feature 會根據目前已驗證的使用者進行檢查，您也可以輕鬆地針對其他使用者或 [scope](#scope) 檢查 Feature。若要完成此操作，請使用 `Feature` facade 提供的 `for` 方法：

```php
return Feature::for($user)->active('new-api')
    ? $this->resolveNewApiResponse($request)
    : $this->resolveLegacyApiResponse($request);
```

Pennant 也提供了一些額外的便利方法，在判斷 Feature 是否啟用中時可能會很有用：

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
> 當您在 HTTP 語境之外使用 Pennant 時，例如在 Artisan command 或佇列工作中，您通常應該 [明確指定 Feature 的 scope](#specifying-the-scope)。或者，您可以定義一個 [預設 scope](#default-scope)，它同時考慮已驗證的 HTTP 語境和未驗證的語境。

<a name="checking-class-based-features"></a>
#### 檢查基於 Class 的 Feature

對於基於 Class 的 Feature，您應該在檢查 Feature 時提供 Class 名稱：

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

`when` 方法可用於流暢地執行一個給定的閉包，如果 Feature 啟用中。此外，可以提供第二個閉包，如果 Feature 未啟用，則會執行該閉包：

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

`unless` 方法與 `when` 方法的作用相反，如果 Feature 未啟用，則執行第一個閉包：

```php
return Feature::unless(NewApi::class,
    fn () => $this->resolveLegacyApiResponse($request),
    fn () => $this->resolveNewApiResponse($request),
);
```

<a name="the-has-features-trait"></a>
### `HasFeatures` Trait

Pennant 的 `HasFeatures` trait 可新增到您應用程式的 `User` model (或任何其他擁有 Feature 的 model)，以提供一種流暢、便捷的方式，直接從 model 檢查 Feature：

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

一旦 trait 被新增到您的 model，您就可以透過呼叫 `features` 方法輕鬆檢查 Feature：

```php
if ($user->features()->active('new-api')) {
    // ...
}
```

當然，`features` 方法提供了許多其他便捷的方法，用於與 Feature 互動：

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

為了讓在 Blade 中檢查 Feature 成為無縫的體驗，Pennant 提供了 `@feature` 和 `@featureany` 指令：

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

Pennant 也包含一個 [中介層](/docs/{{version}}/middleware)，可用於在路由被呼叫之前驗證目前已驗證的使用者是否具有 Feature 的存取權限。您可以將中介層指派給路由，並指定存取路由所需的 Feature。如果任何指定的 Feature 對於目前已驗證的使用者是未啟用中，路由將會回傳 `400 Bad Request` HTTP 回應。多個 Feature 可以傳遞給靜態的 `using` 方法。

```php
use Illuminate\Support\Facades\Route;
use Laravel\Pennant\Middleware\EnsureFeaturesAreActive;

Route::get('/api/servers', function () {
    // ...
})->middleware(EnsureFeaturesAreActive::using('new-api', 'servers-api'));
```

<a name="customizing-the-response"></a>
#### 自訂回應

如果您想要自訂當所列出的 Feature 之一未啟用時，由中介層回傳的回應，您可以使用 `EnsureFeaturesAreActive` 中介層提供的 `whenInactive` 方法。通常，此方法應在您應用程式的其中一個 service provider 的 `boot` 方法中呼叫：

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

有時，在取回給定 Feature 的儲存值之前，在記憶體中執行一些檢查會很有用。想像您正在開發一個新的 API，它在 Feature flag 後面，並且您希望能夠停用這個新 API，同時不遺失儲存中任何已解析的 Feature 值。如果您在新 API 中發現錯誤，您可以輕鬆地為除了內部團隊成員之外的所有人停用它，修正錯誤，然後為先前有權限存取該 Feature 的使用者重新啟用新 API。

您可以使用 [基於 Class 的 Feature](#class-based-features) 的 `before` 方法來達成此目的。當存在時，`before` 方法總是在從儲存中取回值之前於記憶體中執行。如果從該方法傳回非 `null` 值，它將在請求的持續時間內取代 Feature 的儲存值：

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

您也可以使用此 Feature 來安排先前在 Feature flag 後面的 Feature 進行全球推出：

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
### 記憶體快取

檢查 Feature 時，Pennant 會建立結果的記憶體快取。如果您使用 `database` 驅動，這表示在單一請求中重新檢查相同的 Feature flag 不會觸發額外的資料庫查詢。這也確保了 Feature 在請求的持續時間內具有一致的結果。

如果您需要手動清除記憶體快取，可以使用 `Feature` Facade 提供的 `flushCache` 方法：

```php
Feature::flushCache();
```

<a name="scope"></a>
## Scope


<a name="specifying-the-scope"></a>
### 指定 Scope

如前所述，Feature 通常會針對當前已驗證的使用者進行檢查。然而，這可能不總是符合您的需求。因此，您可以透過 `Feature` Facade 的 `for` 方法來指定您想要檢查給定 Feature 所針對的 Scope：

```php
return Feature::for($user)->active('new-api')
    ? $this->resolveNewApiResponse($request)
    : $this->resolveLegacyApiResponse($request);
```

當然，Feature 的 Scope 不限於「使用者」。想像您建立了一個新的計費體驗，並且您想將其推廣給整個團隊而非個別使用者。也許您希望最舊的團隊比新的團隊擁有更慢的推廣速度。您的 Feature 解析閉包可能看起來像這樣：

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

您會注意到我們定義的閉包預期並非一個 `User`，而是預期一個 `Team` 模型。要判斷此 Feature 對於使用者的團隊是否為啟用狀態，您應該將團隊傳遞給 `Feature` Facade 所提供的 `for` 方法：

```php
if (Feature::for($user->team)->active('billing-v2')) {
    return redirect('/billing/v2');
}

// ...
```


<a name="default-scope"></a>
### 預設 Scope

也可以自訂 Pennant 用於檢查 Feature 的預設 Scope。例如，也許您所有的 Feature 都針對當前已驗證使用者的團隊而非使用者進行檢查。您不必每次檢查 Feature 都呼叫 `Feature::for($user->team)`，而是可以直接指定團隊作為預設 Scope。通常，這應該在您應用程式的其中一個 Service Provider 中完成：

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

如果沒有透過 `for` 方法明確提供 Scope，Feature 檢查現在將使用當前已驗證使用者的團隊作為預設 Scope：

```php
Feature::active('billing-v2');

// Is now equivalent to...

Feature::for($user->team)->active('billing-v2');
```


<a name="nullable-scope"></a>
### 可為空的 Scope

如果您在檢查 Feature 時提供的 Scope 為 `null`，並且該 Feature 的定義不支援透過`null`或在聯集型別中包含 `null`，則 Pennant 將自動傳回 `false` 作為 Feature 的結果值。

因此，如果您傳遞給 Feature 的 Scope 可能為 `null`，並且您希望觸發 Feature 的值解析器，您應該在 Feature 的定義中考量這一點。如果在 Artisan 命令、佇列作業或未驗證的路由中檢查 Feature，可能會發生 `null` Scope。由於這些情境下通常沒有已驗證的使用者，預設 Scope 將為 `null`。

如果您不總是[明確指定 Feature Scope](#specifying-the-scope)，那麼您應該確保 Scope 的型別為「nullable」，並在您的 Feature 定義邏輯中處理 `null` Scope 值：

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
### 識別 Scope

Pennant 內建的 `array` 和 `database` 儲存驅動器知道如何正確儲存所有 PHP 資料型別以及 Eloquent 模型的 Scope 識別符。然而，如果您的應用程式使用了第三方 Pennant 驅動器，該驅動器可能不知道如何正確儲存 Eloquent 模型 或應用程式中其他自訂型別的識別符。

鑒於此，Pennant 允許您透過在應用程式中用作 Pennant Scope 的物件上實作 `FeatureScopeable` 契約，來格式化用於儲存的 Scope 值。

例如，想像您在單一應用程式中使用兩個不同的 Feature 驅動器：內建的 `database` 驅動器和第三方「Flag Rocket」驅動器。「Flag Rocket」驅動器不知道如何正確儲存 Eloquent 模型。相反地，它需要一個 `FlagRocketUser` 實例。透過實作 `FeatureScopeable` 契約中定義的 `toFeatureIdentifier` 方法，我們可以自訂提供給應用程式所使用每個驅動器的可儲存 Scope 值：

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
### 序列化 Scope

預設情況下，Pennant 在儲存與 Eloquent 模型關聯的 Feature 時，會使用完整的類別名稱。如果您已經在使用 [Eloquent morph map](/docs/{{version}}/eloquent-relationships#custom-polymorphic-types)，您可以選擇讓 Pennant 也使用 morph map，以便將儲存的 Feature 與您的應用程式結構解耦。

為此，在 Service Provider 中定義您的 Eloquent morph map 後，您可以呼叫 `Feature` Facade 的 `useMorphMap` 方法：

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

迄今為止，我們主要展示的是處於二元狀態的 Feature，也就是它們「啟用」或「停用」兩種狀態。但 Pennant 也允許您儲存豐富的值。

舉例來說，假設您正在為應用程式中的「立即購買」按鈕測試三種新顏色。您可以從 Feature 定義中回傳字串，而不是回傳 `true` 或 `false`：

```php
use Illuminate\Support\Arr;
use Laravel\Pennant\Feature;

Feature::define('purchase-button', fn (User $user) => Arr::random([
    'blue-sapphire',
    'seafoam-green',
    'tart-orange',
]));
```

您可以使用 `value` 方法取回 `purchase-button` Feature 的值：

```php
$color = Feature::value('purchase-button');
```

Pennant 內建的 Blade 指令也能讓您輕鬆地根據 Feature 的當前值條件式地渲染內容：

```blade
@feature('purchase-button', 'blue-sapphire')
    <!-- 'blue-sapphire' is active -->
@elsefeature('purchase-button', 'seafoam-green')
    <!-- 'seafoam-green' is active -->
@elsefeature('purchase-button', 'tart-orange')
    <!-- 'tart-orange' is active -->
@endfeature
```

> [!NOTE] 備註
> 當使用豐富的值時，請務必記住，如果 Feature 有除了 `false` 之外的任何值，則它被視為「啟用」。

當呼叫 [條件式 `when`](#conditional-execution) 方法時，Feature 的豐富值將提供給第一個閉包：

```php
Feature::when('purchase-button',
    fn ($color) => /* ... */,
    fn () => /* ... */,
);
```

同樣地，當呼叫條件式 `unless` 方法時，Feature 的豐富值將提供給選用的第二個閉包：

```php
Feature::unless('purchase-button',
    fn () => /* ... */,
    fn ($color) => /* ... */,
);
```

<a name="retrieving-multiple-features"></a>
## 取回多個 Feature

`values` 方法允許針對給定的 Scope 取回多個 Feature：

```php
Feature::values(['billing-v2', 'purchase-button']);

// [
//     'billing-v2' => false,
//     'purchase-button' => 'blue-sapphire',
// ]
```

或者，您可以使用 `all` 方法取回針對給定 Scope 的所有已定義 Feature 的值：

```php
Feature::all();

// [
//     'billing-v2' => false,
//     'purchase-button' => 'blue-sapphire',
//     'site-redesign' => true,
// ]
```

然而，基於 Class 的 Feature 是動態註冊的，並且在它們被明確檢查之前，Pennant 是不知道的。這表示如果您應用程式中基於 Class 的 Feature 在當前請求期間尚未被檢查過，它們可能不會出現在 `all` 方法回傳的結果中。

如果您想確保在使用 `all` 方法時 Feature Class 總是包含在內，您可以使用 Pennant 的 Feature 探索功能。要開始使用，請在您應用程式的其中一個 Service Provider 中呼叫 `discover` 方法：

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

`discover` 方法將會註冊您應用程式 `app/Features` 目錄中的所有 Feature Class。現在，`all` 方法將在結果中包含這些 Class，無論它們是否已在當前請求期間被檢查：

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

儘管 Pennant 會為單一請求保留所有已解析 Feature 的記憶體快取，但仍然可能遇到效能問題。為緩解這個問題，Pennant 提供了預載入 Feature 值的功能。

為了說明這一點，想像我們正在迴圈中檢查 Feature 是否啟用：

```php
use Laravel\Pennant\Feature;

foreach ($users as $user) {
    if (Feature::for($user)->active('notifications-beta')) {
        $user->notify(new RegistrationSuccess);
    }
}
```

假設我們正在使用 database 驅動，這段程式碼將針對迴圈中的每個使用者執行一次資料庫查詢 —— 可能執行數百次查詢。然而，使用 Pennant 的 `load` 方法，我們可以透過為使用者或 Scope 的集合預載入 Feature 值來消除這個潛在的效能瓶頸：

```php
Feature::for($users)->load(['notifications-beta']);

foreach ($users as $user) {
    if (Feature::for($user)->active('notifications-beta')) {
        $user->notify(new RegistrationSuccess);
    }
}
```

僅在 Feature 值尚未被載入時才載入，您可以使用 `loadMissing` 方法：

```php
Feature::for($users)->loadMissing([
    'new-api',
    'purchase-button',
    'notifications-beta',
]);
```

您可以使用 `loadAll` 方法載入所有已定義 Feature：

```php
Feature::for($users)->loadAll();
```

<a name="updating-values"></a>
## 更新值

當 Feature 的值首次被解析時，底層驅動會將結果儲存在儲存空間中。這通常是為了確保使用者在不同請求中有一致的體驗。然而，有時您可能希望手動更新 Feature 的儲存值。

為此，您可以使用 `activate` 和 `deactivate` 方法來開啟或關閉 Feature：

```php
use Laravel\Pennant\Feature;

// Activate the feature for the default scope...
Feature::activate('new-api');

// Deactivate the feature for the given scope...
Feature::for($user->team)->deactivate('billing-v2');
```

您也可以透過為 `activate` 方法提供第二個引數來手動設定 Feature 的豐富值：

```php
Feature::activate('purchase-button', 'seafoam-green');
```

若要指示 Pennant 忘記 Feature 的儲存值，您可以使用 `forget` 方法。當 Feature 再次被檢查時，Pennant 將從其 Feature 定義中解析該 Feature 的值：

```php
Feature::forget('purchase-button');
```

<a name="bulk-updates"></a>
### 批次更新

若要批次更新 Feature 儲存值，您可以使用 `activateForEveryone` 和 `deactivateForEveryone` 方法。

例如，假設您現在對 `new-api` Feature 的穩定性充滿信心，並已確定了結帳流程中最佳的 `'purchase-button'` 顏色 — 您可以相應地為所有使用者更新儲存值：

```php
use Laravel\Pennant\Feature;

Feature::activateForEveryone('new-api');

Feature::activateForEveryone('purchase-button', 'seafoam-green');
```

或者，您可以為所有使用者停用該 Feature：

```php
Feature::deactivateForEveryone('new-api');
```

> [!NOTE]
> 這只會更新由 Pennant 儲存驅動儲存的已解析 Feature 值。您還需要在應用程式中更新 Feature 定義。

<a name="purging-features"></a>
### 清除 Feature

有時，從儲存空間中清除整個 Feature 會很有用。如果您已從應用程式中移除了 Feature，或者您對 Feature 的定義進行了調整，並希望將其推廣給所有使用者，則通常需要這樣做。

您可以使用 `purge` 方法移除 Feature 的所有儲存值：

```php
// Purging a single feature...
Feature::purge('new-api');

// Purging multiple features...
Feature::purge(['new-api', 'purchase-button']);
```

如果您想從儲存空間中清除_所有_ Feature，您可以不帶任何引數地呼叫 `purge` 方法：

```php
Feature::purge();
```

由於清除 Feature 在應用程式的部署流程中可能很有用，Pennant 包含一個 `pennant:purge` Artisan 指令，它將從儲存空間中清除提供的 Feature：

```shell
php artisan pennant:purge new-api

php artisan pennant:purge new-api purchase-button
```

也可以清除 _除了_ 指定 Feature 列表之外的所有 Feature。例如，假設您想清除所有 Feature，但保留「new-api」和「purchase-button」Feature 的值在儲存空間中。為此，您可以將這些 Feature 名稱傳遞給 `--except` 選項：

```shell
php artisan pennant:purge --except=new-api --except=purchase-button
```

為方便起見，`pennant:purge` 指令還支援 `--except-registered` 旗標。此旗標表示除了那些在服務提供者中明確註冊的 Feature 之外，所有 Feature 都應該被清除：

```shell
php artisan pennant:purge --except-registered
```

<a name="testing"></a>
## 測試

當測試與 Feature 旗標互動的程式碼時，控制 Feature 旗標傳回值最簡單的方法就是重新定義該 Feature。例如，假設您在應用程式的其中一個服務提供者中定義了以下 Feature：

```php
use Illuminate\Support\Arr;
use Laravel\Pennant\Feature;

Feature::define('purchase-button', fn () => Arr::random([
    'blue-sapphire',
    'seafoam-green',
    'tart-orange',
]));
```

若要在測試中修改 Feature 的傳回值，您可以在測試開始時重新定義該 Feature。即使 `Arr::random()` 實作仍然存在於服務提供者中，以下測試也將始終通過：

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

同樣的方法也可用於基於 Class 的 Feature：

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

如果您的 Feature 傳回 `Lottery` 實例，則有許多有用的[測試輔助函式可用](/docs/{{version}}/helpers#testing-lotteries)。

<a name="store-configuration"></a>
#### 儲存設定

您可以透過在應用程式的 `phpunit.xml` 檔案中定義 `PENNANT_STORE` 環境變數，來設定 Pennant 在測試期間將使用的儲存區：

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
## 新增自訂 Pennant 驅動

<a name="implementing-the-driver"></a>
#### 實作驅動

如果 Pennant 現有的儲存驅動都不符合您應用程式的需求，您可以自行編寫儲存驅動。您的自訂驅動應實作 `Laravel\Pennant\Contracts\Driver` 介面：

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

現在，我們只需使用 Redis 連線來實作這些方法。有關如何實作這些方法的範例，請參考 [Pennant 原始碼](https://github.com/laravel/pennant/blob/1.x/src/Drivers/DatabaseDriver.php) 中的 `Laravel\Pennant\Drivers\DatabaseDriver`。

> [!NOTE]
> Laravel 並未附帶用於存放擴充套件的目錄。您可以將它們放置在任何您喜歡的位置。在此範例中，我們建立了一個 `Extensions` 目錄來存放 `RedisFeatureDriver`。

<a name="registering-the-driver"></a>
#### 註冊驅動

一旦您的驅動實作完成，您就可以將其註冊到 Laravel。要為 Pennant 新增額外的驅動，您可以使用 `Feature` Facade 提供的 `extend` 方法。您應該在應用程式的其中一個 [服務提供者](/docs/{{version}}/providers) 的 `boot` 方法中呼叫 `extend` 方法：

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

一旦驅動註冊完成，您就可以在應用程式的 `config/pennant.php` 設定檔中使用 `redis` 驅動：

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
### 在外部定義 Feature

如果您的驅動是第三方 Feature Flag 平台的包裝器，您可能會在該平台上定義 Feature，而不是使用 Pennant 的 `Feature::define` 方法。在這種情況下，您的自訂驅動也應該實作 `Laravel\Pennant\Contracts\DefinesFeaturesExternally` 介面：

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

`definedFeaturesForScope` 方法應該回傳為指定 scope 定義的 Feature 名稱列表。

<a name="events"></a>
## 事件

Pennant 觸發各種事件，這些事件在追蹤應用程式中的 Feature Flag 時非常有用。

### `Laravel\Pennant\Events\FeatureRetrieved`

每當 [檢查 Feature](#checking-features) 時，就會觸發此事件。此事件對於建立和追蹤應用程式中 Feature Flag 的使用指標可能很有用。

### `Laravel\Pennant\Events\FeatureResolved`

當 Feature 的值首次為特定 scope 解析時，就會觸發此事件。

### `Laravel\Pennant\Events\UnknownFeatureResolved`

當未知的 Feature 首次為特定 scope 解析時，就會觸發此事件。如果您有意移除了 Feature Flag，但在應用程式中意外留下了對其的引用，則監聽此事件可能會很有用：

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

當 [基於 Class 的 Feature](#class-based-features) 在請求期間首次動態檢查時，就會觸發此事件。

### `Laravel\Pennant\Events\UnexpectedNullScopeEncountered`

當 `null` scope 傳遞給 [不支援 null](#nullable-scope) 的 Feature 定義時，就會觸發此事件。

這種情況會被優雅地處理，並且 Feature 將回傳 `false`。但是，如果您想選擇退出此 Feature 的預設優雅行為，您可以將此事件的監聽器註冊到應用程式 `AppServiceProvider` 的 `boot` 方法中：

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

當更新特定 scope 的 Feature 時，通常透過呼叫 `activate` 或 `deactivate`，就會觸發此事件。

### `Laravel\Pennant\Events\FeatureUpdatedForAllScopes`

當更新所有 scope 的 Feature 時，通常透過呼叫 `activateForEveryone` 或 `deactivateForEveryone`，就會觸發此事件。

### `Laravel\Pennant\Events\FeatureDeleted`

當刪除特定 scope 的 Feature 時，通常透過呼叫 `forget`，就會觸發此事件。

### `Laravel\Pennant\Events\FeaturesPurged`

當清除特定 Feature 時，就會觸發此事件。

### `Laravel\Pennant\Events\AllFeaturesPurged`

當清除所有 Feature 時，就會觸發此事件。