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
    - [Middleware](#middleware)
    - [攔截功能檢查](#intercepting-feature-checks)
    - [記憶體快取](#in-memory-cache)
- [Scope](#scope)
    - [指定 Scope](#specifying-the-scope)
    - [預設 Scope](#default-scope)
    - [可為 Null 的 Scope](#nullable-scope)
    - [識別 Scope](#identifying-scope)
    - [序列化 Scope](#serializing-scope)
- [豐富的功能值](#rich-feature-values)
- [擷取多個功能](#retrieving-multiple-features)
- [預先載入](#eager-loading)
- [更新值](#updating-values)
    - [批次更新](#bulk-updates)
    - [清除功能](#purging-features)
- [測試](#testing)
- [新增自訂 Pennant 驅動程式](#adding-custom-pennant-drivers)
    - [實作驅動程式](#implementing-the-driver)
    - [註冊驅動程式](#registering-the-driver)
    - [外部定義功能](#defining-features-externally)
- [事件](#events)

<a name="introduction"></a>
## 簡介

[Laravel Pennant](https://github.com/laravel/pennant) 是一個簡單輕量的功能旗標 (feature flag) 套件，沒有多餘的累贅。功能旗標讓您能夠自信地逐步推出新的應用程式功能、進行新介面設計的 A/B 測試、輔助主幹開發策略等等。

<a name="installation"></a>
## 安裝

首先，使用 Composer 套件管理工具將 Pennant 安裝到您的專案中：

```shell
composer require laravel/pennant
```

接下來，您應該使用 `vendor:publish` Artisan 命令發佈 Pennant 的設定檔與遷移檔：

```shell
php artisan vendor:publish --provider="Laravel\Pennant\PennantServiceProvider"
```

最後，您應該執行應用程式的資料庫遷移。這將會建立一個 `features` 資料表，Pennant 會使用它來驅動其 `database` 驅動程式：

```shell
php artisan migrate
```

<a name="configuration"></a>
## 設定

發佈 Pennant 的資源後，其設定檔將位於 `config/pennant.php`。此設定檔允許您指定 Pennant 用於儲存已解析功能旗標值的預設儲存機制。

Pennant 支援透過 `array` 驅動程式將已解析的功能旗標值儲存在記憶體陣列中。或者，Pennant 可以透過 `database` 驅動程式將已解析的功能旗標值持久儲存在關聯式資料庫中，這是 Pennant 使用的預設儲存機制。

<a name="defining-features"></a>
## 定義功能

若要定義功能，您可以使用 `Feature` Facade 提供的 `define` 方法。您需要提供功能的名稱，以及一個閉包 (closure)，該閉包將被呼叫以解析功能的初始值。

通常，功能是在服務提供者 (service provider) 中使用 `Feature` Facade 定義的。該閉包將接收功能檢查的「scope」。最常見的情況是，scope 是目前已驗證的使用者。在此範例中，我們將定義一個功能，用於逐步向應用程式的使用者推出新的 API：

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

如您所見，我們對功能有以下規則：

- 所有內部團隊成員都應該使用新的 API。
- 任何高流量客戶都不應該使用新的 API。
- 否則，該功能應隨機分配給使用者，有 1% 的機率啟用。

`new-api` 功能首次針對給定使用者進行檢查時，閉包的結果將由儲存驅動程式儲存。下次針對相同使用者檢查該功能時，該值將從儲存中擷取，並且不會呼叫閉包。

為方便起見，如果功能定義只回傳一個 Lottery，您可以完全省略閉包：

    Feature::define('site-redesign', Lottery::odds(1, 1000));

<a name="class-based-features"></a>
### 基於類別的功能

Pennant 也允許您定義基於類別的功能。與基於閉包的功能定義不同，無需在服務提供者中註冊基於類別的功能。若要建立基於類別的功能，您可以呼叫 `pennant:feature` Artisan 命令。預設情況下，功能類別將放置在應用程式的 `app/Features` 目錄中：

```shell
php artisan pennant:feature NewApi
```

編寫功能類別時，您只需要定義一個 `resolve` 方法，該方法將被呼叫以解析給定 scope 的功能初始值。同樣，scope 通常是目前已驗證的使用者：

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

如果您想手動解析基於類別的功能實例，您可以呼叫 `Feature` Facade 上的 `instance` 方法：

```php
use Illuminate\Support\Facades\Feature;

$instance = Feature::instance(NewApi::class);
```

> [!NOTE]
> 功能類別是透過 [Service Container](/docs/{{version}}/container) 解析的，因此您可以在功能類別的建構函式中注入依賴項。

#### 自訂儲存的功能名稱

預設情況下，Pennant 將儲存功能類別的完整類別名稱。如果您想將儲存的功能名稱與應用程式的內部結構解耦，您可以在功能類別上指定一個 `$name` 屬性。此屬性的值將取代類別名稱進行儲存：

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

若要判斷功能是否啟用，您可以使用 `Feature` Facade 上的 `active` 方法。預設情況下，功能會針對目前已驗證的使用者進行檢查：

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

儘管功能預設是針對目前已驗證的使用者進行檢查，但您可以輕鬆地針對其他使用者或 [scope](#scope) 檢查功能。若要實現此目的，請使用 `Feature` Facade 提供的 `for` 方法：

```php
return Feature::for($user)->active('new-api')
    ? $this->resolveNewApiResponse($request)
    : $this->resolveLegacyApiResponse($request);
```

Pennant 還提供了一些額外的便利方法，在判斷功能是否啟用時可能會很有用：

```php
// 判斷所有給定功能是否啟用...
Feature::allAreActive(['new-api', 'site-redesign']);

// 判斷任何給定功能是否啟用...
Feature::someAreActive(['new-api', 'site-redesign']);

// 判斷功能是否停用...
Feature::inactive('new-api');

// 判斷所有給定功能是否停用...
Feature::allAreInactive(['new-api', 'site-redesign']);

// 判斷任何給定功能是否停用...
Feature::someAreInactive(['new-api', 'site-redesign']);
```

> [!NOTE]
> 在 HTTP 環境之外使用 Pennant 時，例如在 Artisan 命令或佇列任務中，您通常應該 [明確指定功能的 scope](#specifying-the-scope)。或者，您可以定義一個 [預設 scope](#default-scope)，以同時考慮已驗證的 HTTP 環境和未驗證的環境。

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

`when` 方法可用於流暢地執行給定閉包，如果功能啟用。此外，可以提供第二個閉包，如果功能停用，則會執行該閉包：

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

`unless` 方法是 `when` 方法的反向，如果功能停用，則執行第一個閉包：

```php
return Feature::unless(NewApi::class,
    fn () => $this->resolveLegacyApiResponse($request),
    fn () => $this->resolveNewApiResponse($request),
);
```

<a name="the-has-features-trait"></a>
### `HasFeatures` Trait

Pennant 的 `HasFeatures` Trait 可以新增到應用程式的 `User` 模型 (或任何其他具有功能的模型) 中，以提供一種流暢、方便的方式直接從模型檢查功能：

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

一旦 Trait 已新增到您的模型中，您就可以透過呼叫 `features` 方法輕鬆檢查功能：

```php
if ($user->features()->active('new-api')) {
    // ...
}
```

當然，`features` 方法提供了許多其他方便的方法來與功能互動：

```php
// 值...
$value = $user->features()->value('purchase-button')
$values = $user->features()->values(['new-api', 'purchase-button']);

// 狀態...
$user->features()->active('new-api');
$user->features()->allAreActive(['new-api', 'server-api']);
$user->features()->someAreActive(['new-api', 'server-api']);

$user->features()->inactive('new-api');
$user->features()->allAreInactive(['new-api', 'server-api']);
$user->features()->someAreInactive(['new-api', 'server-api']);

// 條件式執行...
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
### Middleware

Pennant 還包含一個 [Middleware](/docs/{{version}}/middleware)，可用於在路由被呼叫之前驗證目前已驗證的使用者是否具有存取某個功能的權限。您可以將 Middleware 分配給路由，並指定存取路由所需的功能。如果目前已驗證的使用者有任何指定的功能停用，路由將回傳 `400 Bad Request` HTTP 回應。多個功能可以傳遞給靜態 `using` 方法。

```php
use Illuminate\Support\Facades\Route;
use Laravel\Pennant\Middleware\EnsureFeaturesAreActive;

Route::get('/api/servers', function () {
    // ...
})->middleware(EnsureFeaturesAreActive::using('new-api', 'servers-api'));
```

<a name="customizing-the-response"></a>
#### 自訂回應

如果您想自訂 Middleware 在列出的功能之一停用時回傳的回應，您可以使用 `EnsureFeaturesAreActive` Middleware 提供的 `whenInactive` 方法。通常，此方法應在應用程式服務提供者之一的 `boot` 方法中呼叫：

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

有時，在擷取給定功能的儲存值之前執行一些記憶體中檢查會很有用。想像一下，您正在開發一個功能旗標後面的新 API，並且希望能夠停用新 API，而不會丟失儲存中任何已解析的功能值。如果您在新 API 中發現錯誤，您可以輕鬆地為除內部團隊成員之外的所有人停用它，修復錯誤，然後為以前有權存取該功能的使用者重新啟用新 API。

您可以使用 [基於類別的功能](#class-based-features) 的 `before` 方法實現此目的。如果存在，`before` 方法總是在從儲存中擷取值之前在記憶體中執行。如果該方法回傳非 `null` 值，則在請求期間，該值將取代功能的儲存值：

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

您也可以使用此功能來安排以前在功能旗標後面的功能的全球推出：

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

檢查功能時，Pennant 將建立結果的記憶體中快取。如果您使用 `database` 驅動程式，這表示在單一請求中重新檢查相同的功能旗標不會觸發額外的資料庫查詢。這也確保了功能在請求期間具有一致的結果。

如果您需要手動清除記憶體中快取，您可以使用 `Feature` Facade 提供的 `flushCache` 方法：

```php
Feature::flushCache();
```

<a name="scope"></a>
## Scope

<a name="specifying-the-scope"></a>
### 指定 Scope

如前所述，功能通常是針對目前已驗證的使用者進行檢查。然而，這可能不總是符合您的需求。因此，可以透過 `Feature` Facade 的 `for` 方法指定您想要檢查給定功能的 scope：

```php
return Feature::for($user)->active('new-api')
    ? $this->resolveNewApiResponse($request)
    : $this->resolveLegacyApiResponse($request);
```

當然，功能 scope 不僅限於「使用者」。想像一下，您已經建立了一個新的帳務體驗，您正在將其推廣到整個團隊，而不是個別使用者。也許您希望最老的團隊比新的團隊推出得更慢。您的功能解析閉包可能看起來像這樣：

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

您會注意到我們定義的閉包不期望 `User`，而是期望 `Team` 模型。若要判斷此功能是否對使用者的團隊啟用，您應該將團隊傳遞給 `Feature` Facade 提供的 `for` 方法：

```php
if (Feature::for($user->team)->active('billing-v2')) {
    return redirect('/billing/v2');
}

// ...
```

<a name="default-scope"></a>
### 預設 Scope

也可以自訂 Pennant 用於檢查功能的預設 scope。例如，也許您的所有功能都是針對目前已驗證使用者的團隊進行檢查，而不是使用者。您可以將團隊指定為預設 scope，而不是每次檢查功能時都必須呼叫 `Feature::for($user->team)`。通常，這應該在應用程式的服務提供者之一中完成：

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

如果沒有透過 `for` 方法明確提供 scope，功能檢查現在將使用目前已驗證使用者的團隊作為預設 scope：

```php
Feature::active('billing-v2');

// 現在等同於...

Feature::for($user->team)->active('billing-v2');
```

<a name="nullable-scope"></a>
### 可為 Null 的 Scope

如果您在檢查功能時提供的 scope 為 `null`，並且功能的定義不支援透過可為 null 的型別或在聯集型別中包含 `null`，Pennant 將自動回傳 `false` 作為功能的結果值。

因此，如果您傳遞給功能的 scope 可能為 `null`，並且您希望呼叫功能的值解析器，您應該在功能的定義中考慮到這一點。如果您在 Artisan 命令、佇列任務或未驗證的路由中檢查功能，可能會出現 `null` scope。由於這些環境中通常沒有已驗證的使用者，因此預設 scope 將為 `null`。

如果您不總是 [明確指定您的功能 scope](#specifying-the-scope)，那麼您應該確保 scope 的型別是「可為 null 的」，並在您的功能定義邏輯中處理 `null` scope 值：

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

Pennant 內建的 `array` 和 `database` 儲存驅動程式知道如何正確儲存所有 PHP 資料型別以及 Eloquent 模型的 scope 識別碼。然而，如果您的應用程式使用第三方 Pennant 驅動程式，該驅動程式可能不知道如何正確儲存 Eloquent 模型或應用程式中其他自訂型別的識別碼。

有鑑於此，Pennant 允許您透過在應用程式中用作 Pennant scope 的物件上實作 `FeatureScopeable` 契約來格式化要儲存的 scope 值。

例如，想像一下您在單一應用程式中使用了兩個不同的功能驅動程式：內建的 `database` 驅動程式和第三方「Flag Rocket」驅動程式。「Flag Rocket」驅動程式不知道如何正確儲存 Eloquent 模型。相反，它需要一個 `FlagRocketUser` 實例。透過實作 `FeatureScopeable` 契約定義的 `toFeatureIdentifier`，我們可以自訂提供給應用程式使用的每個驅動程式的可儲存 scope 值：

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

預設情況下，Pennant 在儲存與 Eloquent 模型相關聯的功能時將使用完整的類別名稱。如果您已經使用了 [Eloquent morph map](/docs/{{version}}/eloquent-relationships#custom-polymorphic-types)，您可以選擇讓 Pennant 也使用 morph map，以將儲存的功能與您的應用程式結構解耦。

若要實現此目的，在服務提供者中定義 Eloquent morph map 後，您可以呼叫 `Feature` Facade 的 `useMorphMap` 方法：

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

到目前為止，我們主要展示了功能處於二元狀態，即它們是「啟用」或「停用」，但 Pennant 也允許您儲存豐富的值。

例如，想像一下您正在測試應用程式「立即購買」按鈕的三種新顏色。您可以回傳字串，而不是從功能定義中回傳 `true` 或 `false`：

```php
use Illuminate\Support\Arr;
use Laravel\Pennant\Feature;

Feature::define('purchase-button', fn (User $user) => Arr::random([
    'blue-sapphire',
    'seafoam-green',
    'tart-orange',
]));
```

您可以使用 `value` 方法擷取 `purchase-button` 功能的值：

```php
$color = Feature::value('purchase-button');
```

Pennant 包含的 Blade 指令也讓根據功能的目前值條件式渲染內容變得容易：

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
> 使用豐富值時，重要的是要知道當功能具有除 `false` 之外的任何值時，它被視為「啟用」。

呼叫 [條件式 `when`](#conditional-execution) 方法時，功能的豐富值將提供給第一個閉包：

```php
Feature::when('purchase-button',
    fn ($color) => /* ... */,
    fn () => /* ... */,
);
```

同樣，呼叫條件式 `unless` 方法時，功能的豐富值將提供給可選的第二個閉包：

```php
Feature::unless('purchase-button',
    fn () => /* ... */,
    fn ($color) => /* ... */,
);
```

<a name="retrieving-multiple-features"></a>
## 擷取多個功能

`values` 方法允許擷取給定 scope 的多個功能：

```php
Feature::values(['billing-v2', 'purchase-button']);

// [
//     'billing-v2' => false,
//     'purchase-button' => 'blue-sapphire',
// ]
```

或者，您可以使用 `all` 方法擷取給定 scope 的所有已定義功能的值：

```php
Feature::all();

// [
//     'billing-v2' => false,
//     'purchase-button' => 'blue-sapphire',
//     'site-redesign' => true,
// ]
```

然而，基於類別的功能是動態註冊的，直到它們被明確檢查後才被 Pennant 知道。這意味著如果您的應用程式的基於類別的功能在目前請求期間尚未被檢查，它們可能不會出現在 `all` 方法回傳的結果中。

如果您想確保在使用 `all` 方法時始終包含功能類別，您可以使用 Pennant 的功能發現功能。若要開始，請在應用程式的服務提供者之一中呼叫 `discover` 方法：

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

`discover` 方法將註冊應用程式 `app/Features` 目錄中的所有功能類別。`all` 方法現在將在其結果中包含這些類別，無論它們是否已在目前請求期間被檢查：

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

儘管 Pennant 為單一請求保留了所有已解析功能的記憶體中快取，但仍可能遇到效能問題。為緩解此問題，Pennant 提供了預先載入功能值的選項。

為了說明這一點，想像一下我們正在迴圈中檢查功能是否啟用：

```php
use Laravel\Pennant\Feature;

foreach ($users as $user) {
    if (Feature::for($user)->active('notifications-beta')) {
        $user->notify(new RegistrationSuccess);
    }
}
```

假設我們使用資料庫驅動程式，此程式碼將為迴圈中的每個使用者執行資料庫查詢，可能執行數百個查詢。然而，使用 Pennant 的 `load` 方法，我們可以透過預先載入使用者集合或 scope 的功能值來消除這種潛在的效能瓶頸：

```php
Feature::for($users)->load(['notifications-beta']);

foreach ($users as $user) {
    if (Feature::for($user)->active('notifications-beta')) {
        $user->notify(new RegistrationSuccess);
    }
}
```

若要僅在功能值尚未載入時載入它們，您可以使用 `loadMissing` 方法：

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

當功能的值首次解析時，底層驅動程式會將結果儲存在儲存中。這通常是為了確保您的使用者在不同請求之間獲得一致的體驗。然而，有時您可能希望手動更新功能的儲存值。

若要實現此目的，您可以使用 `activate` 和 `deactivate` 方法來「開啟」或「關閉」功能：

```php
use Laravel\Pennant\Feature;

// 為預設 scope 啟用功能...
Feature::activate('new-api');

// 為給定 scope 停用功能...
Feature::for($user->team)->deactivate('billing-v2');
```

也可以透過向 `activate` 方法提供第二個引數來手動設定功能的豐富值：

```php
Feature::activate('purchase-button', 'seafoam-green');
```

若要指示 Pennant 忘記功能的儲存值，您可以使用 `forget` 方法。當再次檢查功能時，Pennant 將從其功能定義中解析功能的值：

```php
Feature::forget('purchase-button');
```

<a name="bulk-updates"></a>
### 批次更新

若要批次更新儲存的功能值，您可以使用 `activateForEveryone` 和 `deactivateForEveryone` 方法。

例如，想像一下您現在對 `new-api` 功能的穩定性充滿信心，並且已經為您的結帳流程找到了最佳的 `'purchase-button'` 顏色，您可以相應地更新所有使用者的儲存值：

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
> 這只會更新 Pennant 儲存驅動程式儲存的已解析功能值。您還需要更新應用程式中的功能定義。

<a name="purging-features"></a>
### 清除功能

有時，清除儲存中的整個功能會很有用。如果您已從應用程式中移除該功能，或者您已對功能的定義進行了調整，並且希望將其推廣到所有使用者，則通常需要這樣做。

您可以使用 `purge` 方法移除功能的所有儲存值：

```php
// 清除單一功能...
Feature::purge('new-api');

// 清除多個功能...
Feature::purge(['new-api', 'purchase-button']);
```

如果您想清除儲存中的 _所有_ 功能，您可以不帶任何引數呼叫 `purge` 方法：

```php
Feature::purge();
```

由於在應用程式的部署流程中清除功能可能很有用，Pennant 包含一個 `pennant:purge` Artisan 命令，它將從儲存中清除所提供的功能：

```shell
php artisan pennant:purge new-api

php artisan pennant:purge new-api purchase-button
```

也可以清除 _除_ 給定功能列表中的功能之外的所有功能。例如，想像一下您想清除所有功能，但保留「new-api」和「purchase-button」功能的值在儲存中。若要實現此目的，您可以將這些功能名稱傳遞給 `--except` 選項：

```shell
php artisan pennant:purge --except=new-api --except=purchase-button
```

為方便起見，`pennant:purge` 命令還支援 `--except-registered` 旗標。此旗標表示應清除除在服務提供者中明確註冊的功能之外的所有功能：

```shell
php artisan pennant:purge --except-registered
```

<a name="testing"></a>
## 測試

測試與功能旗標互動的程式碼時，控制功能旗標在測試中回傳的值最簡單的方法是簡單地重新定義功能。例如，想像一下您在應用程式的服務提供者之一中定義了以下功能：

```php
use Illuminate\Support\Arr;
use Laravel\Pennant\Feature;

Feature::define('purchase-button', fn () => Arr::random([
    'blue-sapphire',
    'seafoam-green',
    'tart-orange',
]));
```

若要在測試中修改功能的回傳值，您可以在測試開始時重新定義功能。即使 `Arr::random()` 實作仍存在於服務提供者中，以下測試也將始終通過：

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

相同的方法也可用於基於類別的功能：

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

如果您的功能回傳 `Lottery` 實例，則有許多有用的 [測試輔助工具可用](/docs/{{version}}/helpers#testing-lotteries)。

<a name="store-configuration"></a>
#### 儲存設定

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
## 新增自訂 Pennant 驅動程式

<a name="implementing-the-driver"></a>
#### 實作驅動程式

如果 Pennant 現有的儲存驅動程式都不符合您的應用程式需求，您可以編寫自己的儲存驅動程式。您的自訂驅動程式應該實作 `Laravel\Pennant\Contracts\Driver` 介面：

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

現在，我們只需要使用 Redis 連線實作這些方法中的每一個。有關如何實作這些方法的範例，請參閱 [Pennant 原始碼](https://github.com/laravel/pennant/blob/1.x/src/Drivers/DatabaseDriver.php) 中的 `Laravel\Pennant\Drivers\DatabaseDriver`。

> [!NOTE]
> Laravel 不附帶包含擴充功能的目錄。您可以將它們放置在任何您喜歡的位置。在此範例中，我們建立了一個 `Extensions` 目錄來存放 `RedisFeatureDriver`。

<a name="registering-the-driver"></a>
#### 註冊驅動程式

一旦您的驅動程式已實作，您就可以將其註冊到 Laravel。若要向 Pennant 新增額外的驅動程式，您可以使用 `Feature` Facade 提供的 `extend` 方法。您應該從應用程式的 [服務提供者](/docs/{{version}}/providers) 之一的 `boot` 方法中呼叫 `extend` 方法：

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

一旦驅動程式已註冊，您就可以在應用程式的 `config/pennant.php` 設定檔中使用 `redis` 驅動程式：

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
### 外部定義功能

如果您的驅動程式是第三方功能旗標平台的包裝器，您可能會在平台上定義功能，而不是使用 Pennant 的 `Feature::define` 方法。如果是這種情況，您的自訂驅動程式也應該實作 `Laravel\Pennant\Contracts\DefinesFeaturesExternally` 介面：

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

`definedFeaturesForScope` 方法應該回傳為所提供 scope 定義的功能名稱列表。

<a name="events"></a>
## 事件

Pennant 會分派各種事件，這些事件在追蹤應用程式中的功能旗標時可能很有用。

### `Laravel\Pennant\Events\FeatureRetrieved`

每當 [檢查功能](#checking-features) 時，就會分派此事件。此事件可能對於建立和追蹤應用程式中功能旗標使用情況的指標很有用。

### `Laravel\Pennant\Events\FeatureResolved`

功能的值首次針對特定 scope 解析時，就會分派此事件。

### `Laravel\Pennant\Events\UnknownFeatureResolved`

未知功能首次針對特定 scope 解析時，就會分派此事件。如果您打算移除功能旗標，但意外地在應用程式中留下了雜亂的參考，則監聽此事件可能很有用：

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

當 [基於類別的功能](#class-based-features) 在請求期間首次動態檢查時，就會分派此事件。

### `Laravel\Pennant\Events\UnexpectedNullScopeEncountered`

當 `null` scope 傳遞給 [不支援 null](#nullable-scope) 的功能定義時，就會分派此事件。

這種情況會優雅地處理，功能將回傳 `false`。然而，如果您想選擇退出此功能的預設優雅行為，您可以在應用程式 `AppServiceProvider` 的 `boot` 方法中註冊此事件的監聽器：

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

當更新 scope 的功能時，通常透過呼叫 `activate` 或 `deactivate`，就會分派此事件。

### `Laravel\Pennant\Events\FeatureUpdatedForAllScopes`

當更新所有 scope 的功能時，通常透過呼叫 `activateForEveryone` 或 `deactivateForEveryone`，就會分派此事件。

### `Laravel\Pennant\Events\FeatureDeleted`

當刪除 scope 的功能時，通常透過呼叫 `forget`，就會分派此事件。

### `Laravel\Pennant\Events\FeaturesPurged`

當清除特定功能時，就會分派此事件。

### `Laravel\Pennant\Events\AllFeaturesPurged`

當清除所有功能時，就會分派此事件。
