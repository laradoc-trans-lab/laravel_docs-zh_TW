# 服務容器

- [簡介](#introduction)
    - [零設定解析](#zero-configuration-resolution)
    - [何時使用容器](#when-to-use-the-container)
- [綁定](#binding)
    - [綁定基礎](#binding-basics)
    - [將介面綁定至實作](#binding-interfaces-to-implementations)
    - [情境綁定](#contextual-binding)
    - [情境屬性](#contextual-attributes)
    - [綁定基本型別](#binding-primitives)
    - [綁定型別可變參數](#binding-typed-variadics)
    - [標記](#tagging)
    - [擴充綁定](#extending-bindings)
- [解析](#resolving)
    - [`make` 方法](#the-make-method)
    - [自動注入](#automatic-injection)
- [方法呼叫與注入](#method-invocation-and-injection)
- [容器事件](#container-events)
    - [重新綁定](#rebinding)
- [PSR-11](#psr-11)

<a name="introduction"></a>
## 簡介

Laravel 服務容器是一個強大的工具，用於管理類別依賴關係並執行依賴注入。依賴注入是一個高深的詞彙，本質上的意思是：類別的依賴項是透過建構子，或者在某些情況下透過「setter」方法被「注入」到類別中。

讓我們來看一個簡單的範例：

```php
<?php

namespace App\Http\Controllers;

use App\Services\AppleMusic;
use Illuminate\View\View;

class PodcastController extends Controller
{
    /**
     * Create a new controller instance.
     */
    public function __construct(
        protected AppleMusic $apple,
    ) {}

    /**
     * Show information about the given podcast.
     */
    public function show(string $id): View
    {
        return view('podcasts.show', [
            'podcast' => $this->apple->findPodcast($id)
        ]);
    }
}
```

在這個範例中，`PodcastController` 需要從諸如 Apple Music 之類的資料源取得 Podcast。因此，我們將**注入**一個能夠取得 Podcast 的服務。由於該服務是被注入的，當我們在測試應用程式時，能夠輕鬆地「mock」或建立 `AppleMusic` 服務的虛擬實作。

深入瞭解 Laravel 服務容器對於建構強大且大型的應用程式至關重要，同時也是為 Laravel 核心本身做出貢獻的必備知識。

<a name="zero-configuration-resolution"></a>
### 零設定解析

如果一個類別沒有任何依賴，或者僅依賴於其他具體類別（而非介面），則容器不需要被指示如何解析該類別。例如，您可以在 `routes/web.php` 檔案中放置以下程式碼：

```php
<?php

class Service
{
    // ...
}

Route::get('/', function (Service $service) {
    dd($service::class);
});
```

在這個範例中，造訪應用程式的 `/` 路由將會自動解析 `Service` 類別，並將其注入到您的路由處理常式中。這是一項顛覆性的特色。這意味著您可以開發應用程式並善用依賴注入的好處，而無需擔心龐大的設定檔。

幸運的是，在建構 Laravel 應用程式時，您撰寫的許多類別都會自動透過容器接收其依賴，包括[控制器](/docs/{{version}}/controllers)、[事件監聽器](/docs/{{version}}/events)、[中介層](/docs/{{version}}/middleware)等等。此外，您還可以在[佇列任務](/docs/{{version}}/queues)的 `handle` 方法中對依賴進行型別提示。一旦您體驗過自動化與零設定依賴注入的威力，就會發現沒有它簡直無法進行開發。

<a name="when-to-use-the-container"></a>
### 何時使用容器

得益於零設定解析，您通常會在路由、控制器、事件監聽器及其他地方對依賴項進行型別提示，而無需手動與容器進行互動。例如，您可以在路由定義中對 `Illuminate\Http\Request` 物件進行型別提示，以便輕鬆存取當前請求。儘管我們在撰寫此程式碼時從不需要手動操作容器，但它會在幕後管理這些依賴項的注入：

```php
use Illuminate\Http\Request;

Route::get('/', function (Request $request) {
    // ...
});
```

在許多情況下，得益於自動依賴注入和 [facades](/docs/{{version}}/facades)，您可以建構 Laravel 應用程式，而**完全不需要**手動從容器綁定或解析任何東西。**那麼，究竟何時才會手動與容器互動呢？**讓我們來看看兩種情況。

首先，如果您撰寫了一個實作介面的類別，並且希望在路由或類別建構子中對該介面進行型別提示，您必須[告訴容器如何解析該介面](#binding-interfaces-to-implementations)。其次，如果您正在[撰寫 Laravel 套件](/docs/{{version}}/packages)並打算與其他 Laravel 開發者分享，您可能需要將套件的服務綁定到容器中。

<a name="binding"></a>
## 綁定


<a name="binding-basics"></a>
### 綁定基礎


<a name="simple-bindings"></a>
#### 簡單綁定

幾乎您所有的服務容器綁定都會在[服務提供者(Service Providers)](/docs/{{version}}/providers)中註冊，因此大多數範例都會展示在該情境下如何使用容器。

在服務提供者中，您隨時可以透過 `$this->app` 屬性存取容器。我們可以使用 `bind` 方法來註冊綁定，傳入我們希望註冊的類別或介面名稱，以及一個傳回該類別實例的閉包：

```php
use App\Services\Transistor;
use App\Services\PodcastParser;
use Illuminate\Contracts\Foundation\Application;

$this->app->bind(Transistor::class, function (Application $app) {
    return new Transistor($app->make(PodcastParser::class));
});
```

請注意，我們接收到容器本身作為解析器的引數。接著，我們便能使用容器來解析我們正在建構的物件的子依賴項。

如前所述，您通常會在服務提供者中與容器進行互動；然而，如果您想在服務提供者之外與容器互動，可以透過 `App` [Facade](/docs/{{version}}/facades) 來完成：

```php
use App\Services\Transistor;
use Illuminate\Contracts\Foundation\Application;
use Illuminate\Support\Facades\App;

App::bind(Transistor::class, function (Application $app) {
    // ...
});
```

您可以使用 `bindIf` 方法，僅在指定的型別尚未註冊任何綁定時，才註冊容器綁定：

```php
$this->app->bindIf(Transistor::class, function (Application $app) {
    return new Transistor($app->make(PodcastParser::class));
});
```

為了方便起見，您可以省略將要註冊的類別或介面名稱作為獨立引數傳入，改為讓 Laravel 從您提供給 `bind` 方法的閉包傳回型別中推導出該型別：

```php
App::bind(function (Application $app): Transistor {
    return new Transistor($app->make(PodcastParser::class));
});
```

> [!NOTE]
> 如果類別不依賴於任何介面，就不需要將這些類別綁定到容器中。容器不需要被指示如何建構這些物件，因為它可以透過反射機制自動解析這些物件。


<a name="binding-a-singleton"></a>
#### 綁定單例

`singleton` 方法將類別或介面綁定至容器中，且只會被解析一次。一旦單例綁定被解析，後續呼叫容器時都會傳回相同的物件實例：

```php
use App\Services\Transistor;
use App\Services\PodcastParser;
use Illuminate\Contracts\Foundation\Application;

$this->app->singleton(Transistor::class, function (Application $app) {
    return new Transistor($app->make(PodcastParser::class));
});
```

您可以使用 `singletonIf` 方法，僅在指定的型別尚未註冊任何綁定時，才註冊單例容器綁定：

```php
$this->app->singletonIf(Transistor::class, function (Application $app) {
    return new Transistor($app->make(PodcastParser::class));
});
```


<a name="singleton-attribute"></a>
#### Singleton 屬性

或者，您可以為介面或類別加上 `#[Singleton]` 屬性，以指示容器該物件應僅被解析一次：

```php
<?php

namespace App\Services;

use Illuminate\Container\Attributes\Singleton;

#[Singleton]
class Transistor
{
    // ...
}
```


<a name="binding-scoped"></a>
#### 綁定作用域單例

`scoped` 方法將類別或介面綁定到容器中，並使其在給定的 Laravel 請求 / 任務生命週期內僅被解析一次。雖然此方法與 `singleton` 方法類似，但使用 `scoped` 方法註冊的實例會在 Laravel 應用程式開始新的「生命週期」時被清除，例如當 [Laravel Octane](/docs/{{version}}/octane) worker 處理新請求或 Laravel [佇列 worker](/docs/{{version}}/queues) 處理新任務時：

```php
use App\Services\Transistor;
use App\Services\PodcastParser;
use Illuminate\Contracts\Foundation\Application;

$this->app->scoped(Transistor::class, function (Application $app) {
    return new Transistor($app->make(PodcastParser::class));
});
```

您可以使用 `scopedIf` 方法，僅在指定的型別尚未註冊任何綁定時，才註冊作用域容器綁定：

```php
$this->app->scopedIf(Transistor::class, function (Application $app) {
    return new Transistor($app->make(PodcastParser::class));
});
```


<a name="scoped-attribute"></a>
#### Scoped 屬性

或者，您可以為介面或類別加上 `#[Scoped]` 屬性，以指示容器該物件在給定的 Laravel 請求 / 任務生命週期內應僅被解析一次：

```php
<?php

namespace App\Services;

use Illuminate\Container\Attributes\Scoped;

#[Scoped]
class Transistor
{
    // ...
}
```


<a name="binding-instances"></a>
#### 綁定實例

您也可以使用 `instance` 方法將現有的物件實例綁定到容器中。後續呼叫容器時將總是傳回該給定的實例：

```php
use App\Services\Transistor;
use App\Services\PodcastParser;

$service = new Transistor(new PodcastParser);

$this->app->instance(Transistor::class, $service);
```

<a name="binding-interfaces-to-implementations"></a>
### 將介面綁定至實作

服務容器一個非常強大的功能，就是能夠將介面綁定至給定的實作。例如，假設我們有一個 `EventPusher` 介面和一個 `RedisEventPusher` 實作。一旦編寫好該介面的 `RedisEventPusher` 實作後，我們就可以像這樣將其註冊至服務容器：

```php
use App\Contracts\EventPusher;
use App\Services\RedisEventPusher;

$this->app->bind(EventPusher::class, RedisEventPusher::class);
```

這個敘述告訴容器，當某個類別需要 `EventPusher` 的實作時，應該注入 `RedisEventPusher`。現在，我們可以在由容器解析的類別建構子中，對 `EventPusher` 介面進行型別提示 (Type-hint)。請記住，Laravel 應用程式中的控制器、事件監聽器、中介層以及其他各種型態的類別，都是透過容器來解析的：

```php
use App\Contracts\EventPusher;

/**
 * Create a new class instance.
 */
public function __construct(
    protected EventPusher $pusher,
) {}
```


<a name="bind-attribute"></a>
#### Bind 屬性

為了更加方便，Laravel 還提供了 `Bind` 屬性。您可以將此屬性套用至任何介面，以告訴 Laravel 每當請求該介面時，應該自動注入哪一個實作。使用 `Bind` 屬性時，不需要在應用程式的服務提供者(Service Providers)中進行任何額外的服務註冊。

此外，可以在介面上放置多個 `Bind` 屬性，以便針對特定環境設定注入不同的實作：

```php
<?php

namespace App\Contracts;

use App\Services\FakeEventPusher;
use App\Services\RedisEventPusher;
use Illuminate\Container\Attributes\Bind;

#[Bind(RedisEventPusher::class)]
#[Bind(FakeEventPusher::class, environments: ['local', 'testing'])]
interface EventPusher
{
    // ...
}
```

此外，還可以套用 [Singleton](#singleton-attribute) 與 [Scoped](#scoped-attribute) 屬性，以表示容器綁定應該僅解析一次，還是在每個請求 / 任務(Job)生命週期中解析一次：

```php
use App\Services\RedisEventPusher;
use Illuminate\Container\Attributes\Bind;
use Illuminate\Container\Attributes\Singleton;

#[Bind(RedisEventPusher::class)]
#[Singleton]
interface EventPusher
{
    // ...
}
```

對於取決於任意條件的綁定，您可以使用 `BindWhen` 屬性。該 Closure 可以接收容器，並應在需要套用綁定時回傳 `true`。`Bind` 和 `BindWhen` 屬性會按照它們被宣告的順序進行評估：

```php
use App\Services\BetaEventPusher;
use Illuminate\Container\Attributes\BindWhen;
use Laravel\Pennant\Feature;

#[BindWhen(BetaEventPusher::class, static fn () => Feature::active('beta-events'))]
interface EventPusher
{
    // ...
}
```

> [!NOTE]
> `BindWhen` 屬性需要 PHP 8.5 或更高版本。


<a name="contextual-binding"></a>
### 情境綁定

有時候，您可能有兩個使用相同介面的類別，但您希望向每個類別注入不同的實作。例如，兩個控制器可能依賴於 `Illuminate\Contracts\Filesystem\Filesystem` [契約(Contracts)](/docs/{{version}}/contracts) 的不同實作。Laravel 提供了一個簡單且流暢的介面來定義此行為：

```php
use App\Http\Controllers\PhotoController;
use App\Http\Controllers\UploadController;
use App\Http\Controllers\VideoController;
use Illuminate\Contracts\Filesystem\Filesystem;
use Illuminate\Support\Facades\Storage;

$this->app->when(PhotoController::class)
    ->needs(Filesystem::class)
    ->give(function () {
        return Storage::disk('local');
    });

$this->app->when([VideoController::class, UploadController::class])
    ->needs(Filesystem::class)
    ->give(function () {
        return Storage::disk('s3');
    });
```

<a name="contextual-attributes"></a>
### 情境屬性

由於情境綁定通常用於注入驅動程式實作或設定值，Laravel 提供了一系列情境綁定屬性 (Attributes)，讓你無須在服務提供者中手動定義情境綁定，就能直接注入這些類型的數值。

例如，`Storage` 屬性可用於注入特定的[儲存磁碟](/docs/{{version}}/filesystem)：

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Container\Attributes\Storage;
use Illuminate\Contracts\Filesystem\Filesystem;

class PhotoController extends Controller
{
    public function __construct(
        #[Storage('local')] protected Filesystem $filesystem
    ) {
        // ...
    }
}
```

除了 `Storage` 屬性外，Laravel 還提供了 `Auth`、`Cache`、`Config`、`Context`、`DB`、`Give`、`Log`、`RequestAttribute`、`RouteParameter` 以及 [Tag](#tagging) 屬性：

```php
<?php

namespace App\Http\Controllers;

use App\Contracts\UserRepository;
use App\Models\Organization;
use App\Models\Photo;
use App\Repositories\DatabaseRepository;
use Illuminate\Container\Attributes\Auth;
use Illuminate\Container\Attributes\Cache;
use Illuminate\Container\Attributes\Config;
use Illuminate\Container\Attributes\Context;
use Illuminate\Container\Attributes\DB;
use Illuminate\Container\Attributes\Give;
use Illuminate\Container\Attributes\Log;
use Illuminate\Container\Attributes\RequestAttribute;
use Illuminate\Container\Attributes\RouteParameter;
use Illuminate\Container\Attributes\Tag;
use Illuminate\Contracts\Auth\Guard;
use Illuminate\Contracts\Cache\Repository;
use Illuminate\Database\Connection;
use Psr\Log\LoggerInterface;

class PhotoController extends Controller
{
    public function __construct(
        #[Auth('web')] protected Guard $auth,
        #[Cache('redis')] protected Repository $cache,
        #[Config('app.timezone')] protected string $timezone,
        #[Context('uuid')] protected string $uuid,
        #[Context('ulid', hidden: true)] protected string $ulid,
        #[DB('mysql')] protected Connection $connection,
        #[Give(DatabaseRepository::class)] protected UserRepository $users,
        #[Log('daily')] protected LoggerInterface $log,
        #[RequestAttribute('organization')] protected Organization $organization,
        #[RouteParameter] protected Photo $photo,
        #[Tag('reports')] protected iterable $reports,
    ) {
        // ...
    }
}
```

`RouteParameter` 屬性會解析與變數名稱比對相符的路由參數。如果需要，你也可以明確指定路由參數名稱：`#[RouteParameter('photo')]`。

`RequestAttribute` 屬性會解析當前請求的 [屬性包 (attribute bag)](https://symfony.com/doc/current/components/http_foundation.html#accessing-request-data) 中儲存在指定鍵名下的值：`#[RequestAttribute('organization')]`。

此外，Laravel 還提供了 `CurrentUser` 屬性，用於將當前通過認證的使用者注入到指定的路由或類別中：

```php
use App\Models\User;
use Illuminate\Container\Attributes\CurrentUser;

Route::get('/user', function (#[CurrentUser] User $user) {
    return $user;
})->middleware('auth');
```


<a name="defining-custom-attributes"></a>
#### 定義自訂屬性

你可以透過實作 `Illuminate\Contracts\Container\ContextualAttribute` 契約(Contracts)來建立自訂的情境屬性。容器會呼叫屬性的 `resolve` 方法，該方法應該要解析出並回傳注入到使用該屬性之類別中的值。在以下範例中，我們將重新實作 Laravel 內建的 `Config` 屬性：

```php
<?php

namespace App\Attributes;

use Attribute;
use Illuminate\Contracts\Container\Container;
use Illuminate\Contracts\Container\ContextualAttribute;
use ReflectionParameter;

#[Attribute(Attribute::TARGET_PARAMETER)]
class Config implements ContextualAttribute
{
    /**
     * Create a new attribute instance.
     */
    public function __construct(public string $key, public mixed $default = null)
    {
    }

    /**
     * Resolve the configuration value.
     *
     * @param  self  $attribute
     * @param  \Illuminate\Contracts\Container\Container  $container
     * @param  \ReflectionParameter  $parameter
     * @return mixed
     */
    public static function resolve(self $attribute, Container $container, ReflectionParameter $parameter)
    {
        return $container->make('config')->get($attribute->key, $attribute->default);
    }
}
```


<a name="binding-primitives"></a>
### 綁定基本型別

有時候，你可能有一個類別除了接收注入的類別外，還需要注入一個基本型別的值（例如整數）。你可以輕鬆地使用情境綁定來注入該類別所需的任何數值：

```php
use App\Http\Controllers\UserController;

$this->app->when(UserController::class)
    ->needs('$variableName')
    ->give($value);
```

有時候，某個類別可能會依賴一個包含[已標記 (tagged)](#tagging)實例的陣列。使用 `giveTagged` 方法，你可以輕鬆地注入帶有該標籤的所有容器綁定：

```php
$this->app->when(ReportAggregator::class)
    ->needs('$reports')
    ->giveTagged('reports');
```

如果你需要從應用程式的設定檔中注入一個值，你可以使用 `giveConfig` 方法：

```php
$this->app->when(ReportAggregator::class)
    ->needs('$timezone')
    ->giveConfig('app.timezone');
```


<a name="binding-typed-variadics"></a>
### 綁定型別可變參數

偶爾，你可能會有一個類別，透過可變建構子引數 (variadic constructor argument) 來接收指定型別的物件陣列：

```php
<?php

use App\Models\Filter;
use App\Services\Logger;

class Firewall
{
    /**
     * The filter instances.
     *
     * @var array
     */
    protected $filters;

    /**
     * Create a new class instance.
     */
    public function __construct(
        protected Logger $logger,
        Filter ...$filters,
    ) {
        $this->filters = $filters;
    }
}
```

使用情境綁定，你可以傳遞一個閉包給 `give` 方法來解析此依賴關係，該閉包需回傳由解析後的 `Filter` 實例組成的陣列：

```php
$this->app->when(Firewall::class)
    ->needs(Filter::class)
    ->give(function (Application $app) {
          return [
              $app->make(NullFilter::class),
              $app->make(ProfanityFilter::class),
              $app->make(TooLongFilter::class),
          ];
    });
```

為了方便起見，當 `Firewall` 需要 `Filter` 實例時，你也可以直接提供一個類別名稱陣列，讓容器自動解析：

```php
$this->app->when(Firewall::class)
    ->needs(Filter::class)
    ->give([
        NullFilter::class,
        ProfanityFilter::class,
        TooLongFilter::class,
    ]);
```


<a name="variadic-tag-dependencies"></a>
#### 可變參數標籤依賴

有時候類別會有一個型別提示為指定類別的可變參數依賴（`Report ...$reports`）。使用 `needs` 與 `giveTagged` 方法，你可以輕鬆地為該依賴注入帶有該[標籤](#tagging)的所有容器綁定：

```php
$this->app->when(ReportAggregator::class)
    ->needs(Report::class)
    ->giveTagged('reports');
```


<a name="tagging"></a>
### 標記

偶爾，你可能需要解析某個「類別種類」的所有綁定。例如，你可能正在建構一個報表分析器 (report analyzer)，它會接收包含許多不同 `Report` 介面實作的陣列。在註冊了 `Report` 的實作後，你可以使用 `tag` 方法為它們分配一個標籤：

```php
$this->app->bind(CpuReport::class, function () {
    // ...
});

$this->app->bind(MemoryReport::class, function () {
    // ...
});

$this->app->tag([CpuReport::class, MemoryReport::class], 'reports');
```

一旦服務被標記後，你就可以透過容器的 `tagged` 方法輕鬆解析它們全部：

```php
$this->app->bind(ReportAnalyzer::class, function (Application $app) {
    return new ReportAnalyzer($app->tagged('reports'));
});
```

<a name="extending-bindings"></a>
### 擴充綁定

`extend` 方法允許對已解析的服務進行修改。例如，當一個服務被解析時，您可以執行額外的程式碼來裝飾（Decorate）或設定該服務。`extend` 方法接受兩個引數：您要擴充的服務類別，以及一個應回傳修改後服務的閉包。該閉包會接收正在被解析的服務與容器實例：

```php
$this->app->extend(Service::class, function (Service $service, Application $app) {
    return new DecoratedService($service);
});
```

<a name="resolving"></a>
## 解析


<a name="the-make-method"></a>
### `make` 方法

你可以使用 `make` 方法從容器中解析出類別實例。`make` 方法接受你希望解析的類別或介面名稱：

```php
use App\Services\Transistor;

$transistor = $this->app->make(Transistor::class);
```

若類別的部分依賴無法透過容器解析，你可以將這些依賴以關聯陣列的方式傳入 `makeWith` 方法來進行注入。例如，我們可以手動傳入 `Transistor` 服務所需的建構函式引數 `$id`：

```php
use App\Services\Transistor;

$transistor = $this->app->makeWith(Transistor::class, ['id' => 1]);
```

可以使用 `bound` 方法來確認某個類別或介面是否已在容器中明確綁定：

```php
if ($this->app->bound(Transistor::class)) {
    // ...
}
```

如果你處於服務提供者之外，且該程式碼位置無法存取 `$app` 變數，則可以使用 `App` [Facade](/docs/{{version}}/facades) 或 `app` [輔助函式](/docs/{{version}}/helpers#method-app)來從容器中解析類別實例：

```php
use App\Services\Transistor;
use Illuminate\Support\Facades\App;

$transistor = App::make(Transistor::class);

$transistor = app(Transistor::class);
```

如果你希望將 Laravel 容器實例本身注入到由容器解析的類別中，可以在該類別的建構函式中型別提示 `Illuminate\Container\Container` 類別：

```php
use Illuminate\Container\Container;

/**
 * Create a new class instance.
 */
public function __construct(
    protected Container $container,
) {}
```


<a name="automatic-injection"></a>
### 自動注入

另外且非常重要的一點是，你可以在由容器解析的類別建構函式中型別提示其依賴，這些類別包含[控制器](/docs/{{version}}/controllers)、[事件監聽器](/docs/{{version}}/events)、[中介層](/docs/{{version}}/middleware)等等。此外，你也可以在[佇列任務](/docs/{{version}}/queues)的 `handle` 方法中型別提示依賴。在實務上，這正是絕大多數物件透過容器解析的方式。

例如，你可以在控制器的建構函式中型別提示應用程式所定義的服務。該服務將會自動被解析並注入至該類別中：

```php
<?php

namespace App\Http\Controllers;

use App\Services\AppleMusic;

class PodcastController extends Controller
{
    /**
     * Create a new controller instance.
     */
    public function __construct(
        protected AppleMusic $apple,
    ) {}

    /**
     * Show information about the given podcast.
     */
    public function show(string $id): Podcast
    {
        return $this->apple->findPodcast($id);
    }
}
```


<a name="method-invocation-and-injection"></a>
## 方法呼叫與注入

有時候，你可能希望呼叫物件實例上的某個方法，同時讓容器自動注入該方法的依賴。例如，假設有以下類別：

```php
<?php

namespace App;

use App\Services\AppleMusic;

class PodcastStats
{
    /**
     * Generate a new podcast stats report.
     */
    public function generate(AppleMusic $apple): array
    {
        return [
            // ...
        ];
    }
}
```

你可以像這樣透過容器呼叫 `generate` 方法：

```php
use App\PodcastStats;
use Illuminate\Support\Facades\App;

$stats = App::call([new PodcastStats, 'generate']);
```

`call` 方法接受任何 PHP callable。容器的 `call` 方法甚至可用於呼叫 Closure，並自動注入其依賴：

```php
use App\Services\AppleMusic;
use Illuminate\Support\Facades\App;

$result = App::call(function (AppleMusic $apple) {
    // ...
});
```


<a name="container-events"></a>
## 容器事件

服務容器每次解析物件時都會觸發一個事件。你可以使用 `resolving` 方法來監聽此事件：

```php
use App\Services\Transistor;
use Illuminate\Contracts\Foundation\Application;

$this->app->resolving(Transistor::class, function (Transistor $transistor, Application $app) {
    // Called when container resolves objects of type "Transistor"...
});

$this->app->resolving(function (mixed $object, Application $app) {
    // Called when container resolves object of any type...
});
```

如你所見，正在被解析的物件將會被傳遞給回呼函式，讓你在該物件提供給使用者之前，先為該物件設定任何額外屬性。


<a name="rebinding"></a>
### 重新綁定

`rebinding` 方法讓你能夠監聽服務何時被重新綁定至容器，也就是在初始綁定之後再次註冊或覆蓋。當你需要每次更新特定綁定時一併更新依賴或修改行為，這會非常有用：

```php
use App\Contracts\PodcastPublisher;
use App\Services\SpotifyPublisher;
use App\Services\TransistorPublisher;
use Illuminate\Contracts\Foundation\Application;

$this->app->bind(PodcastPublisher::class, SpotifyPublisher::class);

$this->app->rebinding(
    PodcastPublisher::class,
    function (Application $app, PodcastPublisher $newInstance) {
        //
    },
);

// New binding will trigger rebinding closure...
$this->app->bind(PodcastPublisher::class, TransistorPublisher::class);
```


<a name="psr-11"></a>
## PSR-11

Laravel 的服務容器實作了 [PSR-11](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-11-container.md) 介面。因此，你可以型別提示 PSR-11 容器介面來取得 Laravel 容器實例：

```php
use App\Services\Transistor;
use Psr\Container\ContainerInterface;

Route::get('/', function (ContainerInterface $container) {
    $service = $container->get(Transistor::class);

    // ...
});
```

如果給定的識別碼無法解析，則會拋出例外。若識別碼從未綁定過，該例外將會是 `Psr\Container\NotFoundExceptionInterface` 的實例。若識別碼已綁定但無法解析，則會拋出 `Psr\Container\ContainerExceptionInterface` 的實例。