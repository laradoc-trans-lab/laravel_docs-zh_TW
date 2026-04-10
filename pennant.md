# Laravel Pennant

- [簡介](#introduction)
- [安裝](#installation)
- [設定](#configuration)
- [定義功能](#defining-features)
    - [基於類別的功能](#class-based-features)
- [檢查功能](#checking-features)
    - [條件式執行](#conditional-execution)
    - [`HasFeatures` Trait](#the-has-features-trait)
    - [Blade 指令](#blade-directive)
    - [中介層](#middleware)
    - [攔截功能檢查](#intercepting-feature-checks)
    - [記憶體內快取](#in-memory-cache)
- [範圍](#scope)
    - [指定範圍](#specifying-the-scope)
    - [預設範圍](#default-scope)
    - [可為空的範圍](#nullable-scope)
    - [識別範圍](#identifying-scope)
    - [序列化範圍](#serializing-scope)
- [豐富的功能值](#rich-feature-values)
- [取得多個功能](#retrieving-multiple-features)
- [預先載入](#eager-loading)
- [更新數值](#updating-values)
    - [批次更新](#bulk-updates)
    - [清除功能](#purging-features)
- [測試](#testing)
- [新增自定義 Pennant 驅動](#adding-custom-pennant-drivers)
    - [實作驅動](#implementing-the-driver)
    - [註冊驅動](#registering-the-driver)
    - [在外部定義功能](#defining-features-externally)
- [事件](#events)

<a name="introduction"></a>
## 簡介

[Laravel Pennant](https://github.com/laravel/pennant) 是一款簡單且輕量級的功能旗標 (Feature Flag) 套件——沒有多餘的雜質。功能旗標讓您能更有自信地逐步推出新的應用程式功能、對新的介面設計進行 A/B 測試、輔助主幹開發 (Trunk-based development) 策略等。


<a name="installation"></a>
## 安裝

首先，使用 Composer 套件管理員將 Pennant 安裝到您的專案中：

```shell
composer require laravel/pennant
```

接著，您應該使用 `vendor:publish` Artisan 指令發布 Pennant 的設定檔與遷移 (Migration) 檔案：

```shell
php artisan vendor:publish --provider="Laravel\Pennant\PennantServiceProvider"
```

最後，您應該執行應用程式的資料庫遷移。這將建立一個 `features` 資料表，Pennant 會使用它來支援其 `database` 驅動：

```shell
php artisan migrate
```


<a name="configuration"></a>
## 設定

發布 Pennant 的資源後，其設定檔將位於 `config/pennant.php`。此設定檔允許您指定 Pennant 用於儲存已解析功能旗標值的預設儲存機制。

Pennant 支援透過 `array` 驅動將已解析的功能旗標值儲存在記憶體陣列中。或者，Pennant 也可以透過 `database` 驅動將已解析的功能旗標值持久化儲存在關聯式資料庫中，這也是 Pennant 使用的預設儲存機制。


<a name="defining-features"></a>
## 定義功能

若要定義一個功能，您可以使用 `Feature` Facade 提供的 `define` 方法。您需要為功能提供一個名稱，以及一個將被呼叫以解析該功能初始值的閉包 (Closure)。

通常，功能是在服務提供者 (Service Provider) 中使用 `Feature` Facade 定義的。該閉包將接收功能檢查的「範圍 (Scope)」。最常見的情況下，範圍是目前通過驗證的使用者。在此範例中，我們將定義一個功能，以便逐步向應用程式的使用者推出新的 API：

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

如您所見，我們為該功能設定了以下規則：

- 所有內部團隊成員都應該使用新的 API。
- 任何高流量客戶都不應該使用新的 API。
- 否則，該功能應以百分之一的機率隨機分配給使用者啟用。

第一次為特定使用者檢查 `new-api` 功能時，閉包的結果將由儲存驅動儲存。下一次針對同一位使用者檢查該功能時，將從儲存空間中取得數值，且不會呼叫閉包。

為了方便起見，如果功能定義僅回傳一個樂透 (Lottery)，您可以完全省略閉包：

    Feature::define('site-redesign', Lottery::odds(1, 1000));


<a name="class-based-features"></a>
### 基於類別的功能

Pennant 還允許您定義基於類別的功能。與基於閉包的功能定義不同，基於類別的功能不需要在服務提供者中註冊。若要建立基於類別的功能，您可以呼叫 `pennant:feature` Artisan 指令。預設情況下，功能類別將放置在應用程式的 `app/Features` 目錄中：

```shell
php artisan pennant:feature NewApi
```

編寫功能類別時，您只需要定義一個 `resolve` 方法，該方法將被呼叫以解析給定範圍的功能初始值。同樣地，範圍通常會是目前通過驗證的使用者：

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

如果您想手動解析基於類別之功能的實例，可以呼叫 `Feature` Facade 上的 `instance` 方法：

```php
use Illuminate\Support\Facades\Feature;

$instance = Feature::instance(NewApi::class);
```

> [!NOTE]
> 功能類別是透過 [服務容器 (Container)](/docs/{{version}}/container) 解析的，因此您可以根據需要在功能類別的建構函式中注入依賴項目。


#### 自定義儲存的功能名稱

預設情況下，Pennant 會儲存功能類別的完整類別名稱 (Fully Qualified Class Name)。如果您希望將儲存的功能名稱與應用程式的內部結構解耦，可以在功能類別上添加 `Name` 屬性。此屬性的值將取代類別名稱被儲存：

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
## 檢查功能

要判斷一個功能是否啟動，您可以使用 `Feature` Facade 上的 `active` 方法。預設情況下，系統會針對當前通過身分驗證的使用者檢查功能：

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

雖然預設情況下會針對當前通過身分驗證的使用者檢查功能，但您也可以輕鬆地針對另一個使用者或 [scope](#scope) 檢查功能。若要實現此目的，請使用 `Feature` Facade 提供的 `for` 方法：

```php
return Feature::for($user)->active('new-api')
    ? $this->resolveNewApiResponse($request)
    : $this->resolveLegacyApiResponse($request);
```

Pennant 還提供了一些額外的便捷方法，在判斷功能是否啟動時可能會非常有用：

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
> 當在 HTTP 請求環境以外的地方使用 Pennant 時（例如在 Artisan 命令或隊列任務中），您通常應該[明確指定功能的範圍](#specifying-the-scope)。或者，您可以定義一個同時兼顧已驗證 HTTP 環境與未驗證環境的[預設範圍](#default-scope)。


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

`when` 方法可用於在功能啟動時流暢地執行給定的 Closure。此外，還可以提供第二個 Closure，該 Closure 將在功能停用時執行：

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

`unless` 方法的作用與 `when` 方法相反，如果功能處於停用狀態，則執行第一個 Closure：

```php
return Feature::unless(NewApi::class,
    fn () => $this->resolveLegacyApiResponse($request),
    fn () => $this->resolveNewApiResponse($request),
);
```


<a name="the-has-features-trait"></a>
### `HasFeatures` Trait

Pennant 的 `HasFeatures` trait 可以新增到您應用程式的 `User` 模型（或任何其他具有功能的模型）中，以提供一種流暢、便捷的方式直接從模型檢查功能：

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

一旦將 trait 新增到您的模型中，您就可以透過呼叫 `features` 方法輕鬆檢查功能：

```php
if ($user->features()->active('new-api')) {
    // ...
}
```

當然，`features` 方法提供了許多其他與功能互動的便捷方法：

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

為了讓在 Blade 中檢查功能成為無縫的體驗，Pennant 提供了 `@feature` 和 `@featureany` 指令：

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

Pennant 還包含了一個[中介層](/docs/{{version}}/middleware)，可用於在調用路由之前驗證當前通過身分驗證的使用者是否具有功能的存取權限。您可以將該中介層分配給路由，並指定存取該路由所需的功能。如果指定的任何功能對於當前通過身分驗證的使用者處於停用狀態，該路由將回傳 `400 Bad Request` HTTP 回應。可以將多個功能傳遞給靜態的 `using` 方法。

```php
use Illuminate\Support\Facades\Route;
use Laravel\Pennant\Middleware\EnsureFeaturesAreActive;

Route::get('/api/servers', function () {
    // ...
})->middleware(EnsureFeaturesAreActive::using('new-api', 'servers-api'));
```


<a name="customizing-the-response"></a>
#### 自定義回應

如果您想自定義當中介層列出的功能之一處於停用狀態時所回傳的回應，可以使用 `EnsureFeaturesAreActive` 中介層提供的 `whenInactive` 方法。通常，此方法應在應用程式其中一個服務提供者的 `boot` 方法中呼叫：

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

有時，在取得特定功能的儲存值之前，先執行一些記憶體內 (in-memory) 檢查會很有用。想像一下，你正在一個功能旗標 (feature flag) 後方開發一個新的 API，並希望能夠在不遺失儲存空間中任何已解析功能值的情況下停用該新 API。如果你發現新 API 中有錯誤，你可以輕鬆地對除內部團隊成員以外的所有人停用它，修復錯誤，然後再為先前有權存取該功能的使用者重新啟用新 API。

你可以透過[基於類別的功能](#class-based-features)的 `before` 方法來實現這一點。當 `before` 方法存在時，它總是在從儲存空間取得數值之前於記憶體中執行。如果該方法回傳了非 `null` 的數值，則在該請求期間內，它將取代該功能的儲存值：

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

你也可以使用此功能來為原本位於功能旗標後方的功能排程全球發布 (rollout)：

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

檢查功能時，Pennant 會為結果建立記憶體內快取。如果你使用的是 `database` 驅動，這意味著在單個請求中重新檢查相同的功能旗標將不會觸發額外的資料庫查詢。這也確保了功能在請求期間具有一致的結果。

如果你需要手動清除記憶體內快取，可以使用 `Feature` Facade 提供的 `flushCache` 方法：

```php
Feature::flushCache();
```

<a name="scope"></a>
## 範圍


<a name="specifying-the-scope"></a>
### 指定範圍

如前所述，功能通常是針對當前通過身份驗證的使用者進行檢查的。然而，這並不總是符合您的需求。因此，可以透過 `Feature` facade 的 `for` 方法指定您想要檢查特定功能的範圍：

```php
return Feature::for($user)->active('new-api')
    ? $this->resolveNewApiResponse($request)
    : $this->resolveLegacyApiResponse($request);
```

當然，功能範圍並不限於「使用者」。假設您建立了一個新的帳單體驗，並正向整個團隊而不是個別使用者推出。或許您希望較舊的團隊比新團隊有更慢的推出速度。您的功能解析閉包可能如下所示：

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

您會注意到，我們定義的閉包並不期待一個 `User`，而是期待一個 `Team` 模型。要確定此功能對於使用者的團隊是否處於活動狀態，您應該將團隊傳遞給 `Feature` facade 提供的 `for` 方法：

```php
if (Feature::for($user->team)->active('billing-v2')) {
    return redirect('/billing/v2');
}

// ...
```


<a name="default-scope"></a>
### 預設範圍

也可以自定義 Pennant 用來檢查功能的預設範圍。例如，也許您所有的功能都是針對當前通過身份驗證的使用者的團隊而不是使用者進行檢查。與其每次檢查功能時都必須呼叫 `Feature::for($user->team)`，不如將團隊指定為預設範圍。通常，這應該在應用程式的一個服務提供者中完成：

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

如果沒有透過 `for` 方法明確提供範圍，功能檢查現在將使用當前通過身份驗證的使用者的團隊作為預設範圍：

```php
Feature::active('billing-v2');

// Is now equivalent to...

Feature::for($user->team)->active('billing-v2');
```


<a name="nullable-scope"></a>
### 可為空的範圍

如果您在檢查功能時提供的範圍是 `null`，且功能的定義不透過可為空型別或在聯合型別中包含 `null` 來支援 `null`，Pennant 將自動回傳 `false` 作為功能的結果值。

因此，如果您傳遞給功能的範圍可能是 `null`，並且您希望觸發功能的值解析器，則應在功能定義中考慮到這一點。如果是在 Artisan 指令、佇列任務或未通過身份驗證的路由中檢查功能，則可能會出現 `null` 範圍。因為在這些上下文中通常沒有通過身份驗證的使用者，所以預設範圍將為 `null`。

如果您不總是 [明確指定功能範圍](#specifying-the-scope)，那麼您應該確保範圍的型別是「可為空 (nullable)」，並在功能定義邏輯中處理 `null` 範圍值：

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
### 識別範圍

Pennant 內建的 `array` 和 `database` 儲存驅動程式知道如何妥善儲存所有 PHP 資料型別以及 Eloquent 模型的範圍識別碼。但是，如果您的應用程式使用第三方 Pennant 驅動程式，該驅動程式可能不知道如何妥善儲存 Eloquent 模型或應用程式中其他自定義型別的識別碼。

鑒於此，Pennant 允許您透過在應用程式中用作 Pennant 範圍的物件上實作 `FeatureScopeable` 合約，來格式化用於儲存的範圍值。

例如，假設您在單個應用程式中使用兩個不同的功能驅動程式：內建的 `database` 驅動程式和第三方的「Flag Rocket」驅動程式。「Flag Rocket」驅動程式不知道如何妥善儲存 Eloquent 模型。相反，它需要一個 `FlagRocketUser` 實例。透過實作由 `FeatureScopeable` 合約定義的 `toFeatureIdentifier`，我們可以自定義提供給應用程式所使用的每個驅動程式的可儲存範圍值：

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
### 序列化範圍

預設情況下，Pennant 在儲存與 Eloquent 模型相關聯的功能時會使用完整的類別名稱。如果您已經在使用 [Eloquent morph map](/docs/{{version}}/eloquent-relationships#custom-polymorphic-types)，您可以選擇讓 Pennant 也使用該 morph map，以將儲存的功能與應用程式結構解耦。

為了實現這一點，在服務提供者中定義 Eloquent morph map 之後，您可以呼叫 `Feature` facade 的 `useMorphMap` 方法：

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

到目前為止，我們主要展示了功能處於二進位 (binary) 狀態，這意味著它們不是「啟動 (active)」就是「非啟動 (inactive)」，但 Pennant 也允許您儲存豐富的數值。

例如，想像您正在為應用程式的「立即購買」按鈕測試三種新顏色。您可以從功能定義中回傳字串，而不是回傳 `true` 或 `false`：

```php
use Illuminate\Support\Arr;
use Laravel\Pennant\Feature;

Feature::define('purchase-button', fn (User $user) => Arr::random([
    'blue-sapphire',
    'seafoam-green',
    'tart-orange',
]));
```

您可以使用 `value` 方法來取得 `purchase-button` 功能的值：

```php
$color = Feature::value('purchase-button');
```

Pennant 內建的 Blade 指令也讓根據功能的目前值來有條件地渲染內容變得非常容易：

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
> 使用豐富的值時，請務必了解，當功能擁有任何非 `false` 的值時，該功能即被視為「啟動」狀態。

當呼叫 [條件式 `when`](#conditional-execution) 方法時，功能豐富的值將會被提供給第一個閉包 (closure)：

```php
Feature::when('purchase-button',
    fn ($color) => /* ... */,
    fn () => /* ... */,
);
```

同樣地，當呼叫條件式 `unless` 方法時，功能豐富的值將會被提供給選配的第二個閉包：

```php
Feature::unless('purchase-button',
    fn () => /* ... */,
    fn ($color) => /* ... */,
);
```


<a name="retrieving-multiple-features"></a>
## 取得多個功能

`values` 方法允許為指定的範圍取得多個功能：

```php
Feature::values(['billing-v2', 'purchase-button']);

// [
//     'billing-v2' => false,
//     'purchase-button' => 'blue-sapphire',
// ]
```

或者，您可以使用 `all` 方法來取得指定範圍內所有已定義功能的值：

```php
Feature::all();

// [
//     'billing-v2' => false,
//     'purchase-button' => 'blue-sapphire',
//     'site-redesign' => true,
// ]
```

然而，基於類別的功能是動態註冊的，在被明確檢查之前，Pennant 並不知道它們的存在。這意味著如果您的應用程式中基於類別的功能在目前的請求期間尚未被檢查過，則可能不會出現在 `all` 方法回傳的結果中。

如果您想確保在使用 `all` 方法時始終包含功能類別，可以使用 Pennant 的功能探索 (feature discovery) 能力。首先，請在應用程式的其中一個服務提供者中呼叫 `discover` 方法：

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

`discover` 方法將會註冊應用程式 `app/Features` 目錄中的所有功能類別。現在，無論這些類別是否在目前的請求中被檢查過，`all` 方法回傳的結果都將包含它們：

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
## 預先載入

雖然 Pennant 會為單次請求保留所有已解析功能的記憶體內快取，但仍有可能遇到效能問題。為了減輕這種情況，Pennant 提供了預先載入 (eager load) 功能值的能力。

為了說明這一點，想像我們正在迴圈中檢查某個功能是否啟動：

```php
use Laravel\Pennant\Feature;

foreach ($users as $user) {
    if (Feature::for($user)->active('notifications-beta')) {
        $user->notify(new RegistrationSuccess);
    }
}
```

假設我們使用的是資料庫驅動，這段程式碼會為迴圈中的每個使用者執行一次資料庫查詢，可能會執行數百次查詢。然而，使用 Pennant 的 `load` 方法，我們可以透過為一組使用者或範圍預先載入功能值，來消除這個潛在的效能瓶頸：

```php
Feature::for($users)->load(['notifications-beta']);

foreach ($users as $user) {
    if (Feature::for($user)->active('notifications-beta')) {
        $user->notify(new RegistrationSuccess);
    }
}
```

若要僅在尚未載入功能值時才進行載入，您可以使用 `loadMissing` 方法：

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
## 更新數值

當功能的值第一次被解析時，底層的驅動會將結果存儲在儲存空間中。這通常是為了確保使用者在不同請求之間有一致的體驗。然而，有時您可能想要手動更新功能的儲存值。

若要達成此目的，您可以使用 `activate` 和 `deactivate` 方法來切換功能的「開啟」或「關閉」：

```php
use Laravel\Pennant\Feature;

// Activate the feature for the default scope...
Feature::activate('new-api');

// Deactivate the feature for the given scope...
Feature::for($user->team)->deactivate('billing-v2');
```

也可以透過向 `activate` 方法提供第二個參數來手動為功能設定豐富的數值：

```php
Feature::activate('purchase-button', 'seafoam-green');
```

若要指示 Pennant 捨棄功能的儲存值，可以使用 `forget` 方法。當再次檢查該功能時，Pennant 將從其功能定義中解析該功能的值：

```php
Feature::forget('purchase-button');
```

<a name="bulk-updates"></a>
### 批次更新

若要批次更新儲存的功能值，可以使用 `activateForEveryone` 和 `deactivateForEveryone` 方法。

例如，假設您現在對 `new-api` 功能的穩定性充滿信心，並且已經為結帳流程確定了最佳的 `'purchase-button'` 顏色，您可以相應地更新所有使用者的儲存值：

```php
use Laravel\Pennant\Feature;

Feature::activateForEveryone('new-api');

Feature::activateForEveryone('purchase-button', 'seafoam-green');
```

或者，您可以為所有使用者停用該功能：

```php
Feature::deactivateForEveryone('new-api');
```

> [!NOTE]
> 這只會更新由 Pennant 儲存驅動所儲存的已解析功能值。您還需要更新應用程式中的功能定義。

<a name="purging-features"></a>
### 清除功能

有時，從儲存空間中清除整個功能會很有用。如果您從應用程式中移除某個功能，或者您對功能的定義進行了調整並希望向所有使用者推出，這通常是必要的。

您可以使用 `purge` 方法移除功能的所有儲存值：

```php
// Purging a single feature...
Feature::purge('new-api');

// Purging multiple features...
Feature::purge(['new-api', 'purchase-button']);
```

如果您想從儲存空間中清除「所有」功能，可以在不帶任何參數的情況下調用 `purge` 方法：

```php
Feature::purge();
```

由於在應用程式部署管線中清除功能很有用，Pennant 包含了一個 `pennant:purge` Artisan 指令，該指令將從儲存空間中清除指定的功能：

```shell
php artisan pennant:purge new-api

php artisan pennant:purge new-api purchase-button
```

也可以清除除了指定功能清單「以外」的所有功能。例如，假設您想清除所有功能，但在儲存空間中保留 "new-api" 和 "purchase-button" 功能的值。為此，您可以將這些功能名稱傳遞給 `--except` 選項：

```shell
php artisan pennant:purge --except=new-api --except=purchase-button
```

為了方便起見，`pennant:purge` 指令還支援 `--except-registered` 旗標。此旗標表示除了在服務提供者中顯式註冊的功能之外，應清除所有功能：

```shell
php artisan pennant:purge --except-registered
```

<a name="testing"></a>
## 測試

在測試與功能旗標互動的程式碼時，在測試中控制功能旗標回傳值的最簡單方法就是重新定義該功能。例如，假設您在應用程式的一個服務提供者中定義了以下功能：

```php
use Illuminate\Support\Arr;
use Laravel\Pennant\Feature;

Feature::define('purchase-button', fn () => Arr::random([
    'blue-sapphire',
    'seafoam-green',
    'tart-orange',
]));
```

若要在測試中修改功能的傳回值，您可以在測試開始時重新定義該功能。即使服務提供者中仍然存在 `Arr::random()` 實作，以下測試也將始終通過：

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

如果您的功能回傳一個 `Lottery` 實例，則有一些實用的 [測試輔助函式可用](/docs/{{version}}/helpers#testing-lotteries)。

<a name="store-configuration"></a>
#### Store 設定

您可以透過在應用程式的 `phpunit.xml` 檔案中定義 `PENNANT_STORE` 環境變數，來設定 Pennant 在測試期間使用的 Store：

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
## 新增自定義 Pennant 驅動


<a name="implementing-the-driver"></a>
#### 實作驅動

如果 Pennant 現有的儲存驅動都不符合您應用程式的需求，您可以撰寫自己的儲存驅動。您的自定義驅動程式應該實作 `Laravel\Pennant\Contracts\Driver` 介面：

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

現在，我們只需要使用 Redis 連線來實作這些方法。關於如何實作這些方法的範例，請查看 [Pennant 原始碼](https://github.com/laravel/pennant/blob/1.x/src/Drivers/DatabaseDriver.php) 中的 `Laravel\Pennant\Drivers\DatabaseDriver`。

> [!NOTE]
> Laravel 並未提供存放擴充功能 (Extension) 的目錄。您可以自由地將它們放在任何您喜歡的地方。在此範例中，我們建立了一個 `Extensions` 目錄來存放 `RedisFeatureDriver`。


<a name="registering-the-driver"></a>
#### 註冊驅動

實作驅動程式後，您就準備好將其註冊到 Laravel。要為 Pennant 新增額外的驅動程式，您可以使用 `Feature` Facade 提供的 `extend` 方法。您應該在應用程式的其中一個 [服務提供者 (Service Provider)](/docs/{{version}}/providers) 的 `boot` 方法中呼叫 `extend` 方法：

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

驅動程式註冊完成後，您就可以在應用程式的 `config/pennant.php` 設定檔中使用 `redis` 驅動程式：

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
### 在外部定義功能

如果您的驅動程式是封裝了第三方功能旗標 (Feature Flag) 平台，您可能會在該平台上定義功能，而不是使用 Pennant 的 `Feature::define` 方法。在這種情況下，您的自定義驅動程式還應該實作 `Laravel\Pennant\Contracts\DefinesFeaturesExternally` 介面：

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

`definedFeaturesForScope` 方法應該回傳為所提供範圍定義的功能名稱列表。


<a name="events"></a>
## 事件

Pennant 會發送多種事件，這在追蹤整個應用程式中的功能旗標時非常有用。


### `Laravel\Pennant\Events\FeatureRetrieved`

每當 [檢查功能](#checking-features) 時，都會發送此事件。此事件對於針對整個應用程式中的功能旗標使用情況建立並追蹤指標非常有用。


### `Laravel\Pennant\Events\FeatureResolved`

當功能的數值首次針對特定範圍被解析時，會發送此事件。


### `Laravel\Pennant\Events\UnknownFeatureResolved`

當首次針對特定範圍解析未知功能時，會發送此事件。如果您打算移除某個功能旗標，但意外地在應用程式中遺留了零星的參照，那麼監聽此事件可能會很有用：

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

當在請求期間首次動態檢查 [基於類別的功能](#class-based-features) 時，會發送此事件。


### `Laravel\Pennant\Events\UnexpectedNullScopeEncountered`

當將 `null` 範圍傳遞給 [不支援 null](#nullable-scope) 的功能定義時，會發送此事件。

這種情況會被優雅地處理，且功能將回傳 `false`。但是，如果您想選擇不使用此功能的預設優雅行為，您可以在應用程式 `AppServiceProvider` 的 `boot` 方法中註冊此事件的監聽器：

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

當為範圍更新功能時（通常透過呼叫 `activate` 或 `deactivate`），會發送此事件。


### `Laravel\Pennant\Events\FeatureUpdatedForAllScopes`

當為所有範圍更新功能時（通常透過呼叫 `activateForEveryone` 或 `deactivateForEveryone`），會發送此事件。


### `Laravel\Pennant\Events\FeatureDeleted`

當為範圍刪除功能時（通常透過呼叫 `forget`），會發送此事件。


### `Laravel\Pennant\Events\FeaturesPurged`

當清除特定功能時，會發送此事件。


### `Laravel\Pennant\Events\AllFeaturesPurged`

當清除所有功能時，會發送此事件。