# Laravel Pennant

- [簡介](#introduction)
- [安裝](#installation)
- [設定](#configuration)
- [定義功能](#defining-features)
    - [基於類別的功能](#class-based-features)
- [檢查功能](#checking-features)
    - [條件式執行](#conditional-execution)
    - [The `HasFeatures` 特性](#the-has-features-trait)
    - [Blade 指令](#blade-directive)
    - [中介層](#middleware)
    - [攔截功能檢查](#intercepting-feature-checks)
    - [記憶體快取](#in-memory-cache)
- [作用域](#scope)
    - [指定作用域](#specifying-the-scope)
    - [預設作用域](#default-scope)
    - [可為 Null 的作用域](#nullable-scope)
    - [識別作用域](#identifying-scope)
    - [序列化作用域](#serializing-scope)
- [豐富的功能值](#rich-feature-values)
- [擷取多個功能](#retrieving-multiple-features)
- [預載入](#eager-loading)
- [更新值](#updating-values)
    - [批量更新](#bulk-updates)
    - [清除功能](#purging-features)
- [測試](#testing)
- [新增自訂 Pennant 驅動器](#adding-custom-pennant-drivers)
    - [實作驅動器](#implementing-the-driver)
    - [註冊驅動器](#registering-the-driver)
    - [外部定義功能](#defining-features-externally)
- [事件](#events)

<a name="introduction"></a>
## 簡介

[Laravel Pennant](https://github.com/laravel/pennant) 是一個簡潔輕量的功能旗標套件 — 且不帶多餘的冗贅功能。功能旗標讓您能自信地逐步推出應用程式新功能、進行新介面設計的 A/B 測試、輔助主幹式開發策略，以及更多用途。

<a name="installation"></a>
## 安裝

首先，使用 Composer 套件管理器在您的專案中安裝 Pennant：

```shell
composer require laravel/pennant
```

接著，您應該使用 `vendor:publish` Artisan 指令來發佈 Pennant 的設定與遷移檔：

```shell
php artisan vendor:publish --provider="Laravel\Pennant\PennantServiceProvider"
```

最後，您應該執行應用程式的資料庫遷移。這會建立一個 `features` 資料表，Pennant 會用來驅動其 `database` 驅動器：

```shell
php artisan migrate
```

<a name="configuration"></a>
## 設定

發佈 Pennant 的資源後，其設定檔將位於 `config/pennant.php`。此設定檔允許您指定 Pennant 將用於儲存已解析的功能旗標值的預設儲存機制。

Pennant 支援透過 `array` 驅動器將已解析的功能旗標值儲存在記憶體陣列中。或者，Pennant 可以透過 `database` 驅動器將已解析的功能旗標值持久地儲存在關聯式資料庫中，這是 Pennant 使用的預設儲存機制。

<a name="defining-features"></a>
## 定義功能

若要定義一個功能，您可以使用 `Feature` Facade 提供的 `define` 方法。您需要為該功能提供一個名稱，以及一個將會被呼叫來解析功能初始值的閉包。

通常，功能是使用 `Feature` Facade 在服務提供者中定義的。該閉包將接收功能檢查的「作用域」。最常見的是，作用域是目前已驗證的使用者。在此範例中，我們將定義一個功能，用於逐步向應用程式的使用者推出新的 API：

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

如您所見，我們對功能設定了以下規則：

-   所有內部團隊成員都應該使用新的 API。
-   任何高流量客戶都不應該使用新的 API。
-   否則，該功能應隨機分配給使用者，有 1/100 的機率啟用。

第一次為特定使用者檢查 `new-api` 功能時，閉包的結果將由儲存驅動器儲存。下次針對同一使用者檢查該功能時，該值將從儲存中擷取，並且閉包將不會被呼叫。

為方便起見，如果功能定義只回傳一個 Lottery，您可以完全省略該閉包：

    Feature::define('site-redesign', Lottery::odds(1, 1000));

<a name="class-based-features"></a>
### 基於類別的功能

Pennant 也允許您定義基於類別的功能。與基於閉包的功能定義不同，不需要在服務提供者中註冊基於類別的功能。若要建立基於類別的功能，您可以呼叫 `pennant:feature` Artisan 指令。預設情況下，功能類別將放置在您應用程式的 `app/Features` 目錄中：

```shell
php artisan pennant:feature NewApi
```

編寫功能類別時，您只需要定義一個 `resolve` 方法，該方法將會被呼叫，以解析給定作用域的功能初始值。同樣地，作用域通常是目前已驗證的使用者：

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

如果您想手動解析基於類別功能的實例，您可以呼叫 `Feature` Facade 上的 `instance` 方法：

```php
use Illuminate\Support\Facades\Feature;

$instance = Feature::instance(NewApi::class);
```

> [!NOTE]
> 功能類別是透過 [容器](/docs/{{version}}/container) 解析的，因此您可以在需要時將依賴注入到功能類別的建構式中。

#### 自訂儲存的功能名稱

預設情況下，Pennant 會儲存功能類別的完整限定類別名稱。如果您希望將儲存的功能名稱與應用程式的內部結構解耦，您可以在功能類別上指定一個 `$name` 屬性。此屬性的值將用於替代類別名稱進行儲存：

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
## 檢查功能

要判斷某個功能是否啟用，您可以使用 `Feature` Facade 的 `active` 方法。預設情況下，功能會根據目前已認證使用者進行檢查：

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

儘管功能預設是針對目前已認證使用者進行檢查，但您可以輕鬆地針對其他使用者或[作用域](#scope)檢查功能。為此，請使用 `Feature` Facade 提供的 `for` 方法：

```php
return Feature::for($user)->active('new-api')
    ? $this->resolveNewApiResponse($request)
    : $this->resolveLegacyApiResponse($request);
```

Pennant 也提供了一些額外的便利方法，當判斷功能是否啟用時可能會很有用：

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
> 當在 HTTP 以外的上下文中使用 Pennant 時，例如在 Artisan 指令或佇列任務中，您通常應該[明確指定功能的範圍](#specifying-the-scope)。或者，您可以定義一個[預設作用域](#default-scope)，同時考慮已認證的 HTTP 上下文和未認證的上下文。

<a name="checking-class-based-features"></a>
#### 檢查基於類別的功能

對於基於類別的功能，您應該在檢查功能時提供類別名稱：

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

`when` 方法可用於流暢地執行給定的閉包（如果功能已啟用）。此外，也可以提供第二個閉包，如果功能未啟用，則會執行該閉包：

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

`unless` 方法與 `when` 方法的作用相反，如果功能未啟用，則會執行第一個閉包：

    return Feature::unless(NewApi::class,
        fn () => $this->resolveLegacyApiResponse($request),
        fn () => $this->resolveNewApiResponse($request),
    );

<a name="the-has-features-trait"></a>
### The `HasFeatures` 特性

Pennant 的 `HasFeatures` 特性可以新增到您的應用程式的 `User` 模型（或任何其他具有功能的模型）中，以提供一種流暢、方便的方式直接從模型檢查功能：

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

一旦該特性已新增到您的模型中，您就可以透過呼叫 `features` 方法輕鬆檢查功能：

```php
if ($user->features()->active('new-api')) {
    // ...
}
```

當然，`features` 方法還提供了許多其他方便的方法來與功能互動：

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

為了讓在 Blade 中檢查功能成為無縫體驗，Pennant 提供了 `@feature` 和 `@featureany` 指令：

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

Pennant 也包含一個[中介層](/docs/{{version}}/middleware)，可用於在路由被呼叫之前驗證目前已認證使用者是否能存取某個功能。您可以將此中介層指派給一個路由，並指定存取該路由所需的功能。如果指定的功能中有任何一個對於目前已認證使用者是未啟用的，則該路由將返回 `400 Bad Request` HTTP 回應。多個功能可以傳遞給靜態 `using` 方法。

```php
use Illuminate\Support\Facades\Route;
use Laravel\Pennant\Middleware\EnsureFeaturesAreActive;

Route::get('/api/servers', function () {
    // ...
})->middleware(EnsureFeaturesAreActive::using('new-api', 'servers-api'));
```

<a name="customizing-the-response"></a>
#### 自訂回應

如果您想自訂當中介層因為列表中某個功能未啟用而返回的回應，您可以使用 `EnsureFeaturesAreActive` 中介層提供的 `whenInactive` 方法。通常，此方法應在您的應用程式的其中一個服務提供者的 `boot` 方法中呼叫：

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
### 攔截功能檢查

有時，在擷取給定功能的儲存值之前，執行一些記憶體內檢查會很有用。想像您正在開發一個隱藏在功能旗標後面的新 API，並且希望能夠禁用該新 API，同時不丟失儲存中任何已解析的功能值。如果您在新 API 中發現了一個錯誤，您可以輕鬆地為除了內部團隊成員之外的所有人禁用它，修復錯誤，然後為以前有權存取該功能的用戶重新啟用新 API。

您可以使用 [基於類別的功能的](#class-based-features) `before` 方法來實現此目的。當存在時，`before` 方法總是在記憶體中執行，然後才從儲存中擷取值。如果方法傳回一個非 `null` 值，該值將在請求期間取代功能的儲存值：

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

您也可以使用此功能來排程先前隱藏在功能旗標後面的功能的全域推出：

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

檢查功能時，Pennant 將建立結果的記憶體快取。如果您正在使用 `database` 驅動器，這意味著在單一請求中重複檢查相同的功能旗標不會觸發額外的資料庫查詢。這也確保了該功能在請求期間具有一致的結果。

如果您需要手動清除記憶體快取，可以使用 `Feature` Facade 提供的 `flushCache` 方法：

    Feature::flushCache();

<a name="scope"></a>
## 作用域


<a name="specifying-the-scope"></a>
### 指定作用域

如前所述，功能通常會針對目前驗證的使用者進行檢查。然而，這可能不總是符合您的需求。因此，您可以透過 `Feature` Facade 的 `for` 方法來指定要針對哪個作用域檢查給定的功能：

```php
return Feature::for($user)->active('new-api')
    ? $this->resolveNewApiResponse($request)
    : $this->resolveLegacyApiResponse($request);
```

當然，功能作用域不限於「使用者」。想像您已建置新的帳務體驗，並計畫推行給整個團隊而非個別使用者。也許您希望較舊的團隊推行速度比新團隊慢。您的功能解析閉包可能如下所示：

```php
use App\Models\Team;
use Carbon\Carbon;
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

您會注意到，我們定義的閉包預期接收的不是 `User`，而是 `Team` 模型。若要判斷此功能是否對使用者的團隊啟用，您應該將該團隊傳入 `Feature` Facade 提供的 `for` 方法：

```php
if (Feature::for($user->team)->active('billing-v2')) {
    return redirect('/billing/v2');
}

// ...
```


<a name="default-scope"></a>
### 預設作用域

也可以自訂 Pennant 用來檢查功能的預設作用域。例如，也許您的所有功能都是針對目前驗證使用者的團隊而非使用者進行檢查。您可以將團隊指定為預設作用域，而無需每次檢查功能時都呼叫 `Feature::for($user->team)`。通常，這應該在您的應用程式服務提供者之一中完成：

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

如果沒有透過 `for` 方法明確提供作用域，功能檢查將會使用目前驗證使用者的團隊作為預設作用域：

```php
Feature::active('billing-v2');

// Is now equivalent to...

Feature::for($user->team)->active('billing-v2');
```

<a name="nullable-scope"></a>
### 可為 Null 的作用域

如果您在檢查功能時提供給作用域的值為 `null`，且該功能的定義不支援透過可為 `null` 的型別或在聯集型別中包含 `null`，Pennant 將自動回傳 `false` 作為該功能的結果值。

因此，如果您傳遞給功能的作用域可能為 `null`，且您希望功能的值解析器被呼叫，您應該在您的功能定義中考慮到這一點。如果 Artisan 指令、佇列任務或未驗證的路由中檢查功能，可能會發生 `null` 作用域的情況。由於這些情境通常沒有驗證使用者，預設作用域將為 `null`。

如果您不總是[明確指定您的功能作用域](#specifying-the-scope)，那麼您應該確保作用域的型別是「可為 null」，並在您的功能定義邏輯中處理 `null` 作用域值：

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

Pennant 內建的 `array` 與 `database` 儲存驅動器知道如何正確儲存所有 PHP 資料型別以及 Eloquent 模型的作用域識別符。然而，如果您的應用程式使用第三方 Pennant 驅動器，該驅動器可能不知道如何正確儲存 Eloquent 模型或應用程式中其他自訂型別的識別符。

因此，Pennant 允許您透過在應用程式中用作 Pennant 作用域的物件上實作 `FeatureScopeable` 契約，來格式化作用域值以供儲存。

例如，想像您在單一應用程式中使用兩種不同的功能驅動器：內建的 `database` 驅動器和第三方「Flag Rocket」驅動器。「Flag Rocket」驅動器不知道如何正確儲存 Eloquent 模型。相反地，它需要一個 `FlagRocketUser` 實例。透過實作由 `FeatureScopeable` 契約定義的 `toFeatureIdentifier`，我們可以自訂提供給應用程式所使用的每個驅動器的可儲存作用域值：

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

預設情況下，Pennant 會在儲存與 Eloquent 模型關聯的功能時，使用完整的類別名稱。如果您已經在使用 [Eloquent 多型對應](/docs/{{version}}/eloquent-relationships#custom-polymorphic-types)，您可以選擇讓 Pennant 也使用多型對應來將儲存的功能從應用程式結構中解耦。

為實現此目的，在服務提供者中定義您的 Eloquent 多型對應後，您可以呼叫 `Feature` Facade 的 `useMorphMap` 方法：

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
## 豐富的功能值

到目前為止，我們主要將功能展示為二元狀態，即「啟用」或「停用」，但 Pennant 也允許您儲存豐富的值。

例如，想像您正在測試應用程式中「立即購買」按鈕的三種新顏色。您可以返回字串，而非從功能定義中返回 `true` 或 `false`：

```php
use Illuminate\Support\Arr;
use Laravel\Pennant\Feature;

Feature::define('purchase-button', fn (User $user) => Arr::random([
    'blue-sapphire',
    'seafoam-green',
    'tart-orange',
]));
```

您可以使用 `value` 方法來擷取 `purchase-button` 功能的值：

```php
$color = Feature::value('purchase-button');
```

Pennant 內建的 Blade 指令也讓您可以輕鬆地根據功能的當前值來條件式地渲染內容：

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
> 當使用豐富的值時，請務必了解，除了 `false` 以外的任何值都將被視為「啟用」功能。

呼叫 [條件式 `when`](#conditional-execution) 方法時，功能豐富的值將會提供給第一個閉包：

    Feature::when('purchase-button',
        fn ($color) => /* ... */,
        fn () => /* ... */,
    );

同樣地，呼叫條件式 `unless` 方法時，功能豐富的值將會提供給可選的第二個閉包：

    Feature::unless('purchase-button',
        fn () => /* ... */,
        fn ($color) => /* ... */,
    );


<a name="retrieving-multiple-features"></a>
## 擷取多個功能

`values` 方法允許為給定作用域擷取多個功能：

```php
Feature::values(['billing-v2', 'purchase-button']);

// [
//     'billing-v2' => false,
//     'purchase-button' => 'blue-sapphire',
// ]
```

或者，您可以使用 `all` 方法來擷取給定作用域下所有已定義功能的值：

```php
Feature::all();

// [
//     'billing-v2' => false,
//     'purchase-button' => 'blue-sapphire',
//     'site-redesign' => true,
// ]
```

然而，基於類別的功能是動態註冊的，並且在明確檢查之前，Pennant 並不了解這些功能。這表示如果您的應用程式中基於類別的功能在當前請求期間尚未被檢查過，它們可能不會出現在 `all` 方法返回的結果中。

如果您希望在使用 `all` 方法時功能類別始終被包含在內，可以使用 Pennant 的功能發現功能。首先，在您應用程式的其中一個服務提供者中呼叫 `discover` 方法：

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

`discover` 方法將會註冊您應用程式 `app/Features` 目錄中的所有功能類別。現在，`all` 方法將在其結果中包含這些類別，無論它們是否已在當前請求期間被檢查過：

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

雖然 Pennant 會為單一請求保留所有已解析功能的記憶體快取，但仍可能遇到效能問題。為緩解此問題，Pennant 提供了預載入功能值的能力。

為了說明這一點，想像我們正在迴圈中檢查一個功能是否啟用：

```php
use Laravel\Pennant\Feature;

foreach ($users as $user) {
    if (Feature::for($user)->active('notifications-beta')) {
        $user->notify(new RegistrationSuccess);
    }
}
```

假設我們正在使用資料庫驅動器，這段程式碼將在迴圈中為每個使用者執行一次資料庫查詢——可能執行數百次查詢。然而，使用 Pennant 的 `load` 方法，我們可以透過為使用者或作用域集合預載入功能值，來消除這種潛在的效能瓶頸：

```php
Feature::for($users)->load(['notifications-beta']);

foreach ($users as $user) {
    if (Feature::for($user)->active('notifications-beta')) {
        $user->notify(new RegistrationSuccess);
    }
}
```

若只想在功能值尚未載入時才載入，您可以使用 `loadMissing` 方法：

```php
Feature::for($users)->loadMissing([
    'new-api',
    'purchase-button',
    'notifications-beta',
]);
```

您可以使用 `loadAll` 方法載入所有已定義的功能：

```php
Feature::for($users)->loadAll();
```

<a name="updating-values"></a>
## 更新值

當功能的數值首次被解析時，底層驅動器會將結果儲存起來。這通常是為了確保您的使用者在跨請求時能有一致的體驗。然而，有時您可能需要手動更新功能的儲存值。

為此，您可以使用 `activate` 和 `deactivate` 方法來切換功能的「開啟」或「關閉」狀態：

```php
use Laravel\Pennant\Feature;

// Activate the feature for the default scope...
Feature::activate('new-api');

// Deactivate the feature for the given scope...
Feature::for($user->team)->deactivate('billing-v2');
```

您也可以透過為 `activate` 方法提供第二個參數，手動為功能設定豐富值：

```php
Feature::activate('purchase-button', 'seafoam-green');
```

若要指示 Pennant 忘記功能的儲存值，您可以使用 `forget` 方法。當再次檢查該功能時，Pennant 將從其功能定義中解析功能值：

```php
Feature::forget('purchase-button');
```

<a name="bulk-updates"></a>
### 批量更新

若要批量更新儲存的功能值，您可以使用 `activateForEveryone` 和 `deactivateForEveryone` 方法。

例如，想像您現在對 `new-api` 功能的穩定性充滿信心，並已確定結帳流程的最佳 `'purchase-button'` 顏色 — 您可以相應地更新所有使用者的儲存值：

```php
use Laravel\Pennant\Feature;

Feature::activateForEveryone('new-api');

Feature::activateForEveryone('purchase-button', 'seafoam-green');
```

或者，您可以為所有使用者停用此功能：

```php
Feature::deactivateForEveryone('new-api');
```

> [!NOTE]   
> 這只會更新由 Pennant 儲存驅動器儲存的已解析功能值。您還需要更新應用程式中的功能定義。

<a name="purging-features"></a>
### 清除功能

有時，從儲存中清除整個功能可能很有用。這通常在您從應用程式中移除了該功能，或對功能定義進行了調整，並希望向所有使用者推出時是必要的。

您可以使用 `purge` 方法移除功能的所有儲存值：

```php
// Purging a single feature...
Feature::purge('new-api');

// Purging multiple features...
Feature::purge(['new-api', 'purchase-button']);
```

如果您想從儲存中清除_所有_功能，您可以不帶任何參數地呼叫 `purge` 方法：

```php
Feature::purge();
```

由於在應用程式的部署流程中清除功能可能很有用，Pennant 包含一個 `pennant:purge` Artisan 命令，它將從儲存中清除所提供的功能：

```sh
php artisan pennant:purge new-api

php artisan pennant:purge new-api purchase-button
```

您也可以清除所有功能，_除了_給定功能清單中的功能。例如，假設您想清除所有功能，但保留「new-api」和「purchase-button」功能在儲存中的值。為此，您可以將這些功能名稱傳遞給 `--except` 選項：

```sh
php artisan pennant:purge --except=new-api --except=purchase-button
```

為方便起見，`pennant:purge` 命令也支援 `--except-registered` 旗標。此旗標表示除了在服務提供者中明確註冊的功能外，所有功能都應被清除：

```sh
php artisan pennant:purge --except-registered
```

<a name="testing"></a>
## 測試

在測試與功能旗標互動的程式碼時，控制功能旗標返回值最簡單的方法是簡單地重新定義該功能。例如，假設您在應用程式的某個服務提供者中定義了以下功能：

```php
use Illuminate\Support\Arr;
use Laravel\Pennant\Feature;

Feature::define('purchase-button', fn () => Arr::random([
    'blue-sapphire',
    'seafoam-green',
    'tart-orange',
]));
```

若要在測試中修改功能的返回值，您可以在測試開始時重新定義該功能。即使 `Arr::random()` 的實作仍然存在於服務提供者中，以下測試也將總是會通過：

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

同樣的方法也可用於基於類別的功能：

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

如果您的功能返回的是 `Lottery` 實例，則有一些有用的[測試輔助函式可用](/docs/{{version}}/helpers#testing-lotteries)。

<a name="store-configuration"></a>
#### 儲存配置

您可以透過在應用程式的 `phpunit.xml` 檔案中定義 `PENNANT_STORE` 環境變數來設定 Pennant 在測試期間將使用的儲存：

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
## 新增自訂 Pennant 驅動器


<a name="implementing-the-driver"></a>
#### 實作驅動器

如果 Pennant 現有的儲存驅動器都無法滿足您應用程式的需求，您可以自行編寫儲存驅動器。您的自訂驅動器應實作 `Laravel\Pennant\Contracts\Driver` 介面：

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
> Laravel 並未提供用於存放擴充功能 (extensions) 的目錄。您可以自由選擇將它們放置在任何位置。在此範例中，我們建立了一個 `Extensions` 目錄來存放 `RedisFeatureDriver`。


<a name="registering-the-driver"></a>
#### 註冊驅動器

一旦您的驅動器實作完成，您就可以將其註冊到 Laravel。要為 Pennant 新增額外的驅動器，您可以使用 `Feature` Facade 提供的 `extend` 方法。您應該從應用程式的 [服務提供者](/docs/{{version}}/providers) 的 `boot` 方法中呼叫 `extend` 方法：

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

一旦驅動器註冊完成，您就可以在應用程式的 `config/pennant.php` 設定檔中使用 `redis` 驅動器：

    'stores' => [

        'redis' => [
            'driver' => 'redis',
            'connection' => null,
        ],

        // ...

    ],


<a name="defining-features-externally"></a>
### 外部定義功能

如果您的驅動器是第三方功能旗標平台的包裝器 (wrapper)，您很可能會直接在該平台上定義功能，而不是使用 Pennant 的 `Feature::define` 方法。在這種情況下，您的自訂驅動器也應實作 `Laravel\Pennant\Contracts\DefinesFeaturesExternally` 介面：

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

`definedFeaturesForScope` 方法應傳回為指定作用域定義的功能名稱列表。


<a name="events"></a>
## 事件

Pennant 會分派 (dispatch) 各種事件，這些事件對於追蹤應用程式中的功能旗標非常有用。


### `Laravel\Pennant\Events\FeatureRetrieved`

每當 [功能被檢查](#checking-features) 時，就會分派此事件。此事件可用於針對應用程式中功能旗標的使用情況建立和追蹤指標。


### `Laravel\Pennant\Events\FeatureResolved`

此事件在功能值首次為特定作用域解析時分派。


### `Laravel\Pennant\Events\UnknownFeatureResolved`

當未知功能首次為特定作用域解析時，就會分派此事件。如果您打算移除功能旗標，但意外地在應用程式中留下了零散的引用，監聽此事件可能會很有用：

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

當 [基於類別的功能](#class-based-features) 在請求期間首次被動態檢查時，就會分派此事件。


### `Laravel\Pennant\Events\UnexpectedNullScopeEncountered`

當 `null` 作用域傳遞給不 [支援 null](#nullable-scope) 的功能定義時，就會分派此事件。

這種情況會被優雅地處理，功能將傳回 `false`。然而，如果您想選擇不使用此功能的預設優雅行為，您可以在應用程式 `AppServiceProvider` 的 `boot` 方法中註冊此事件的監聽器：

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

當更新作用域的功能時，通常是透過呼叫 `activate` 或 `deactivate`，就會分派此事件。


### `Laravel\Pennant\Events\FeatureUpdatedForAllScopes`

當更新所有作用域的功能時，通常是透過呼叫 `activateForEveryone` 或 `deactivateForEveryone`，就會分派此事件。


### `Laravel\Pennant\Events\FeatureDeleted`

當刪除作用域的功能時，通常是透過呼叫 `forget`，就會分派此事件。


### `Laravel\Pennant\Events\FeaturesPurged`

當清除特定功能時，就會分派此事件。


### `Laravel\Pennant\Events\AllFeaturesPurged`

當清除所有功能時，就會分派此事件。