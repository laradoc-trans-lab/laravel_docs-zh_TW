# 服務容器

- [簡介](#introduction)
    - [零配置解析](#zero-configuration-resolution)
    - [何時使用容器](#when-to-use-the-container)
- [綁定](#binding)
    - [綁定基礎](#binding-basics)
    - [介面綁定至實作](#binding-interfaces-to-implementations)
    - [情境綁定](#contextual-binding)
    - [情境屬性](#contextual-attributes)
    - [綁定基礎型別](#binding-primitives)
    - [綁定型別可變參數](#binding-typed-variadics)
    - [標記](#tagging)
    - [擴展綁定](#extending-bindings)
- [解析](#resolving)
    - [`make` 方法](#the-make-method)
    - [自動注入](#automatic-injection)
- [方法呼叫與注入](#method-invocation-and-injection)
- [容器事件](#container-events)
    - [重新綁定](#rebinding)
- [PSR-11](#psr-11)

<a name="introduction"></a>
## 簡介

Laravel 服務容器是一個強大的工具，用於管理類別依賴並執行依賴注入。依賴注入是一個花俏的術語，但其本質意義是：類別依賴會透過建構函式，或在某些情況下透過「setter」方法，「注入」到類別中。

讓我們看一個簡單的範例：

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

在這個範例中，`PodcastController` 需要從諸如 Apple Music 之類的資料來源中取得 podcast。因此，我們將 **注入** 一個能夠取得 podcast 的服務。由於服務是被注入的，所以在測試應用程式時，我們可以輕鬆地「mock」（模擬）或建立一個 `AppleMusic` 服務的虛擬實作。

深入理解 Laravel 服務容器對於建立強大且大型的應用程式，以及對於貢獻 Laravel 核心本身，都至關重要。

<a name="zero-configuration-resolution"></a>
### 零配置解析

如果一個類別沒有任何依賴，或者僅依賴於其他具體類別（而非介面），則容器無需被告知如何解析該類別。例如，您可以在 `routes/web.php` 檔案中放入以下程式碼：

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

在這個範例中，訪問您應用程式的 `/` 路由將會自動解析 `Service` 類別並將其注入到您的路由處理器中。這是一個顛覆性的改變。這表示您可以開發應用程式並利用依賴注入的優勢，而無需擔心臃腫的設定檔。

幸運的是，在建構 Laravel 應用程式時，許多您將編寫的類別會自動透過容器接收其依賴，其中包括 [控制器](/docs/{{version}}/controllers)、[事件監聽器](/docs/{{version}}/events)、[中介層](/docs/{{version}}/middleware) 等。此外，您可以在 [佇列任務](/docs/{{version}}/queues) 的 `handle` 方法中型別提示依賴。一旦您體驗到自動化和零配置依賴注入的強大功能，就感覺沒有它便無法進行開發。

<a name="when-to-use-the-container"></a>
### 何時使用容器

由於零配置解析的緣故，您將經常在路由、控制器、事件監聽器以及其他地方型別提示依賴，而無需手動與容器互動。例如，您可以在路由定義上型別提示 `Illuminate\Http\Request` 物件，以便輕鬆存取目前的請求。即使我們從未手動與容器互動來編寫這段程式碼，容器仍在幕後管理這些依賴的注入：

```php
use Illuminate\Http\Request;

Route::get('/', function (Request $request) {
    // ...
});
```

在許多情況下，由於自動依賴注入和 [Facades](/docs/{{version}}/facades)，您可以建構 Laravel 應用程式，而無需**從未**手動綁定或解析容器中的任何內容。**那麼，您何時會手動與容器互動呢？** 讓我們探討兩種情況。

首先，如果您編寫一個實作介面的類別，並且希望在路由或類別建構函式中型別提示該介面，您必須 [告知容器如何解析該介面](#binding-interfaces-to-implementations)。其次，如果您正在 [編寫 Laravel 套件](/docs/{{version}}/packages) 並計畫與其他 Laravel 開發者分享，您可能需要將您的套件服務綁定到容器中。

<a name="binding"></a>
## 綁定

<a name="binding-basics"></a>
### 綁定基礎

<a name="simple-bindings"></a>
#### 簡單綁定

幾乎所有服務容器綁定都將在 [服務提供者](/docs/{{version}}/providers) 中註冊，因此大多數範例將示範在該情境下使用容器。

在服務提供者中，您總是能透過 `$this->app` 屬性來存取容器。我們可以透過 `bind` 方法註冊綁定，傳入我們希望註冊的類別或介面名稱，以及一個回傳該類別實例的閉包：

```php
use App\Services\Transistor;
use App\Services\PodcastParser;
use Illuminate\Contracts\Foundation\Application;

$this->app->bind(Transistor::class, function (Application $app) {
    return new Transistor($app->make(PodcastParser::class));
});
```

請注意，我們收到容器本身作為解析器的引數。然後，我們可以使用容器來解析我們正在建構之物件的子依賴。

如前所述，您通常會在服務提供者內部與容器互動；然而，如果您想在服務提供者外部與容器互動，您可以透過 `App` [Facade](/docs/{{version}}/facades) 來實現：

```php
use App\Services\Transistor;
use Illuminate\Contracts\Foundation\Application;
use Illuminate\Support\Facades\App;

App::bind(Transistor::class, function (Application $app) {
    // ...
});
```

您可以使用 `bindIf` 方法來註冊容器綁定，但僅當該型別的綁定尚未註冊時。

```php
$this->app->bindIf(Transistor::class, function (Application $app) {
    return new Transistor($app->make(PodcastParser::class));
});
```

為了方便，您可以省略提供希望註冊的類別或介面名稱作為單獨的引數，改為讓 Laravel 從您提供給 `bind` 方法的閉包回傳型別中推斷出型別：

```php
App::bind(function (Application $app): Transistor {
    return new Transistor($app->make(PodcastParser::class));
});
```

> [!NOTE]
> 如果類別不依賴任何介面，則無需將其綁定到容器中。容器無需被告知如何建構這些物件，因為它可以使用反射 (reflection) 自動解析這些物件。

<a name="binding-a-singleton"></a>
#### 綁定單例

`singleton` 方法會將一個類別或介面綁定到容器中，該類別或介面應該只被解析一次。一旦單例綁定被解析，相同的物件實例將在後續對容器的呼叫中回傳：

```php
use App\Services\Transistor;
use App\Services\PodcastParser;
use Illuminate\Contracts\Foundation\Application;

$this->app->singleton(Transistor::class, function (Application $app) {
    return new Transistor($app->make(PodcastParser::class));
});
```

您可以使用 `singletonIf` 方法來註冊單例容器綁定，但僅當該型別的綁定尚未註冊時。

```php
$this->app->singletonIf(Transistor::class, function (Application $app) {
    return new Transistor($app->make(PodcastParser::class));
});
```

<a name="singleton-attribute"></a>
#### 單例屬性

或者，您可以將介面或類別標記為 `#[Singleton]` 屬性，以指示容器它應該只被解析一次：

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

`scoped` 方法會將一個類別或介面綁定到容器中，該類別或介面應該在給定的 Laravel 請求/任務生命週期內只被解析一次。儘管此方法類似於 `singleton` 方法，但使用 `scoped` 方法註冊的實例將在 Laravel 應用程式開始一個新的「生命週期」時被清除，例如當 [Laravel Octane](/docs/{{version}}/octane) 工作者處理一個新請求，或者當 Laravel [佇列工作者](/docs/{{version}}/queues) 處理一個新任務時：

```php
use App\Services\Transistor;
use App\Services\PodcastParser;
use Illuminate\Contracts\Foundation\Application;

$this->app->scoped(Transistor::class, function (Application $app) {
    return new Transistor($app->make(PodcastParser::class));
});
```

您可以使用 `scopedIf` 方法來註冊作用域容器綁定，但僅當該型別的綁定尚未註冊時。

```php
$this->app->scopedIf(Transistor::class, function (Application $app) {
    return new Transistor($app->make(PodcastParser::class));
});
```

<a name="scoped-attribute"></a>
#### 作用域屬性

或者，您可以將介面或類別標記為 `#[Scoped]` 屬性，以指示容器它應該在給定的 Laravel 請求/任務生命週期內只被解析一次：

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

您也可以使用 `instance` 方法將現有物件實例綁定到容器中。給定的實例將始終在後續對容器的呼叫中回傳：

```php
use App\Services\Transistor;
use App\Services\PodcastParser;

$service = new Transistor(new PodcastParser);

$this->app->instance(Transistor::class, $service);
```

<a name="binding-interfaces-to-implementations"></a>
### 介面綁定至實作

服務容器一個非常強大的功能是它能將一個介面綁定到給定的實作。例如，我們假設有一個 `EventPusher` 介面和一個 `RedisEventPusher` 實作。一旦我們編寫了這個介面的 `RedisEventPusher` 實作，我們就可以像這樣將它註冊到服務容器中：

```php
use App\Contracts\EventPusher;
use App\Services\RedisEventPusher;

$this->app->bind(EventPusher::class, RedisEventPusher::class);
```

這條語句告訴容器，當一個類別需要 `EventPusher` 的實作時，它應該注入 `RedisEventPusher`。現在我們可以在由容器解析的類別的建構函式中，對 `EventPusher` 介面進行型別提示。請記住，控制器 (controllers)、事件監聽器 (event listeners)、中介層 (middleware) 以及 Laravel 應用程式中各種其他型別的類別，總是透過容器來解析：

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
#### 綁定屬性

Laravel 還提供 `Bind` 屬性以增加便利性。您可以將此屬性應用於任何介面，以告知 Laravel 每當請求該介面時，應該自動注入哪個實作。使用 `Bind` 屬性時，無需在應用程式的服務提供者中執行任何額外的服務註冊。

此外，可以在介面上放置多個 `Bind` 屬性，以配置針對給定的一組環境應注入的不同實作：

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

此外，[單例](#singleton-attribute) 和 [作用域](#scoped-attribute) 屬性可以被應用，以指示容器綁定是應該解析一次，還是在每個請求/任務生命週期中解析一次：

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

<a name="contextual-binding"></a>
### 情境綁定

有時候，您可能有兩個類別使用相同的介面，但您希望為每個類別注入不同的實作。舉例來說，兩個控制器可能依賴 `Illuminate\Contracts\Filesystem\Filesystem` [契約](/docs/{{version}}/contracts)的不同實作。Laravel 提供了一個簡單且流暢的介面來定義此行為：

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

由於情境綁定常被用於注入驅動程式實作或設定值，Laravel 提供多種情境綁定屬性，讓您無需手動在服務提供者中定義情境綁定即可注入這些型別的值。

例如，`Storage` 屬性可用於注入特定的 [儲存磁碟](/docs/{{version}}/filesystem)：

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

除了 `Storage` 屬性外，Laravel 還提供 `Auth`、`Cache`、`Config`、`Context`、`DB`、`Give`、`Log`、`RouteParameter` 和 [Tag](#tagging) 屬性：

```php
<?php

namespace App\Http\Controllers;

use App\Contracts\UserRepository;
use App\Models\Photo;
use App\Repositories\DatabaseRepository;
use Illuminate\Container\Attributes\Auth;
use Illuminate\Container\Attributes\Cache;
use Illuminate\Container\Attributes\Config;
use Illuminate\Container\Attributes\Context;
use Illuminate\Container\Attributes\DB;
use Illuminate\Container\Attributes\Give;
use Illuminate\Container\Attributes\Log;
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
        #[RouteParameter('photo')] protected Photo $photo,
        #[Tag('reports')] protected iterable $reports,
    ) {
        // ...
    }
}
```

此外，Laravel 提供了 `CurrentUser` 屬性，用於將目前已驗證的使用者注入到給定的路由或類別中：

```php
use App\Models\User;
use Illuminate\Container\Attributes\CurrentUser;

Route::get('/user', function (#[CurrentUser] User $user) {
    return $user;
})->middleware('auth');
```


<a name="defining-custom-attributes"></a>
#### 定義自訂屬性

您可以透過實作 `Illuminate\Contracts\Container\ContextualAttribute` 契約來建立自己的情境屬性。容器將呼叫您屬性的 `resolve` 方法，該方法應解析將被注入到使用該屬性的類別中的值。在下面的範例中，我們將重新實作 Laravel 內建的 `Config` 屬性：

```php
<?php

namespace App\Attributes;

use Attribute;
use Illuminate\Contracts\Container\Container;
use Illuminate\Contracts\Container\ContextualAttribute;

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
     * @return mixed
     */
    public static function resolve(self $attribute, Container $container)
    {
        return $container->make('config')->get($attribute->key, $attribute->default);
    }
}
```


<a name="binding-primitives"></a>
### 綁定基礎型別

有時候，您可能有一個類別接收一些被注入的類別，但也需要注入基礎型別值，例如整數。您可以輕鬆地使用情境綁定來注入類別可能需要的任何值：

```php
use App\Http\Controllers\UserController;

$this->app->when(UserController::class)
    ->needs('$variableName')
    ->give($value);
```

有時候，類別可能依賴於一個 [已標記](#tagging) 實例的陣列。使用 `giveTagged` 方法，您可以輕鬆地注入所有帶有該標記的容器綁定：

```php
$this->app->when(ReportAggregator::class)
    ->needs('$reports')
    ->giveTagged('reports');
```

如果您需要從應用程式的設定檔中注入值，可以使用 `giveConfig` 方法：

```php
$this->app->when(ReportAggregator::class)
    ->needs('$timezone')
    ->giveConfig('app.timezone');
```


<a name="binding-typed-variadics"></a>
### 綁定型別可變參數

有時候，您可能有一個類別使用可變參數建構子引數來接收型別化物件陣列：

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

使用情境綁定，您可以透過提供 `give` 方法一個閉包來解析此依賴，該閉包會回傳一個已解析的 `Filter` 實例陣列：

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

為方便起見，您也可以直接提供一個類別名稱陣列，供容器在 `Firewall` 需要 `Filter` 實例時解析：

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
#### 可變參數標記依賴

有時候，類別可能具有型別提示為特定類別 (`Report ...$reports`) 的可變參數依賴。使用 `needs` 和 `giveTagged` 方法，您可以輕鬆地為給定的依賴注入所有帶有該 [標記](#tagging) 的容器綁定：

```php
$this->app->when(ReportAggregator::class)
    ->needs(Report::class)
    ->giveTagged('reports');
```

<a name="tagging"></a>
### 標記

偶爾，您可能需要解析某個特定「類別」的所有綁定。例如，您可能正在建構一個報告分析器，它接收多個不同的 `Report` 介面實作陣列。在註冊 `Report` 實作後，您可以使用 `tag` 方法為它們分配一個標籤：

```php
$this->app->bind(CpuReport::class, function () {
    // ...
});

$this->app->bind(MemoryReport::class, function () {
    // ...
});

$this->app->tag([CpuReport::class, MemoryReport::class], 'reports');
```

一旦服務被標記後，您即可透過容器的 `tagged` 方法輕鬆解析所有這些服務：

```php
$this->app->bind(ReportAnalyzer::class, function (Application $app) {
    return new ReportAnalyzer($app->tagged('reports'));
});
```

<a name="extending-bindings"></a>
### 擴展綁定

`extend` 方法允許修改已解析的服務。例如，當服務被解析時，您可以執行額外的程式碼來裝飾或配置該服務。`extend` 方法接受兩個參數：您要擴展的服務類別，以及一個應返回修改後服務的閉包。該閉包會接收正在解析的服務和容器實例：

```php
$this->app->extend(Service::class, function (Service $service, Application $app) {
    return new DecoratedService($service);
});
```

<a name="resolving"></a>
## 解析


<a name="the-make-method"></a>
### `make` 方法

您可以使用 `make` 方法從容器中解析類別實例。`make` 方法接受您希望解析的類別或介面名稱：

```php
use App\Services\Transistor;

$transistor = $this->app->make(Transistor::class);
```

如果您的類別有些依賴項無法透過容器解析，您可以將它們作為關聯陣列傳遞給 `makeWith` 方法來注入。例如，我們可以手動傳遞 `Transistor` 服務所需的建構子引數 `$id`：

```php
use App\Services\Transistor;

$transistor = $this->app->makeWith(Transistor::class, ['id' => 1]);
```

`bound` 方法可用於判斷類別或介面是否已明確綁定到容器中：

```php
if ($this->app->bound(Transistor::class)) {
    // ...
}
```

如果您在服務提供者之外的程式碼位置無法存取 `$app` 變數，您可以使用 `App` [Facade](/docs/{{version}}/facades) 或 `app` [輔助函數](/docs/{{version}}/helpers#method-app) 來從容器中解析類別實例：

```php
use App\Services\Transistor;
use Illuminate\Support\Facades\App;

$transistor = App::make(Transistor::class);

$transistor = app(Transistor::class);
```

如果您希望將 Laravel 容器實例本身注入到正在被容器解析的類別中，您可以在該類別的建構子中型別提示 `Illuminate\Container\Container` 類別：

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

另外，更重要的是，您可以將依賴項型別提示在由容器解析的類別之建構子中，包括 [控制器](/docs/{{version}}/controllers)、[事件監聽器](/docs/{{version}}/events)、[中介層](/docs/{{version}}/middleware) 等。此外，您還可以在 [佇列任務](/docs/{{version}}/queues) 的 `handle` 方法中型別提示依賴項。實際上，這就是大多數物件應該由容器解析的方式。

例如，您可以將應用程式中定義的服務型別提示在控制器的建構子中。該服務將自動被解析並注入到該類別中：

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

有時您可能希望在物件實例上呼叫一個方法，同時允許容器自動注入該方法的依賴項。例如，假設有以下類別：

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

您可以透過容器呼叫 `generate` 方法，如下所示：

```php
use App\PodcastStats;
use Illuminate\Support\Facades\App;

$stats = App::call([new PodcastStats, 'generate']);
```

`call` 方法接受任何 PHP 可呼叫物件 (callable)。容器的 `call` 方法甚至可以用於呼叫閉包，同時自動注入其依賴項：

```php
use App\Services\AppleMusic;
use Illuminate\Support\Facades\App;

$result = App::call(function (AppleMusic $apple) {
    // ...
});
```


<a name="container-events"></a>
## 容器事件

服務容器每次解析物件時都會觸發一個事件。您可以使用 `resolving` 方法監聽此事件：

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

如您所見，正在解析的物件將傳遞給回呼 (callback)，讓您可以在物件提供給其消費者之前，設定任何額外的屬性。


<a name="rebinding"></a>
### 重新綁定

`rebinding` 方法允許您監聽服務何時被重新綁定到容器，這表示它在初始綁定之後再次被註冊或覆寫。當您需要更新依賴項或每次特定綁定更新時修改行為時，這會很有用：

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

Laravel 的服務容器實作了 [PSR-11](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-11-container.md) 介面。因此，您可以型別提示 PSR-11 容器介面以取得 Laravel 容器的實例：

```php
use App\Services\Transistor;
use Psr\Container\ContainerInterface;

Route::get('/', function (ContainerInterface $container) {
    $service = $container->get(Transistor::class);

    // ...
});
```

如果給定的識別碼無法解析，則會拋出例外。如果該識別碼從未綁定，則例外將是 `Psr\Container\NotFoundExceptionInterface` 的實例。如果該識別碼已綁定但無法解析，則會拋出 `Psr\Container\ContainerExceptionInterface` 的實例。