# 服務容器

- [簡介](#introduction)
    - [零配置解析](#zero-configuration-resolution)
    - [何時使用容器](#when-to-use-the-container)
- [綁定](#binding)
    - [綁定基礎](#binding-basics)
    - [將介面綁定至實作](#binding-interfaces-to-implementations)
    - [情境綁定](#contextual-binding)
    - [情境屬性](#contextual-attributes)
    - [綁定原始值](#binding-primitives)
    - [綁定具型態的可變引數](#binding-typed-variadics)
    - [標籤](#tagging)
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

Laravel 服務容器是一個強大的工具，用於管理類別依賴並執行依賴注入。依賴注入是一個花俏的說法，其本質是：類別依賴透過建構式，或在某些情況下透過「setter」方法，「注入」到類別中。

讓我們看一個簡單的範例：

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

在這個範例中，`PodcastController` 需要從諸如 Apple Music 之類型的資料來源擷取 podcast。因此，我們將會**注入**一個能夠擷取 podcast 的服務。由於服務是被注入的，在測試應用程式時，我們能夠輕鬆地「模擬」，或建立 `AppleMusic` 服務的虛擬實作。

深入理解 Laravel 服務容器對於建立強大、大型的應用程式以及為 Laravel 核心做出貢獻至關重要。

<a name="zero-configuration-resolution"></a>
### 零配置解析

如果一個類別沒有依賴，或者只依賴於其他具體類別（而非介面），則容器無需被告知如何解析該類別。例如，您可以在 `routes/web.php` 檔案中放置以下程式碼：

    <?php

    class Service
    {
        // ...
    }

    Route::get('/', function (Service $service) {
        die($service::class);
    });

在這個範例中，存取您應用程式的 `/` 路由將自動解析 `Service` 類別並將其注入到您的路由處理器中。這是顛覆性的。這表示您可以開發應用程式並利用依賴注入，而無需擔心臃腫的設定檔。

幸運的是，許多您在建立 Laravel 應用程式時編寫的類別都會透過容器自動接收其依賴，包括[控制器](/docs/{{version}}/controllers)、[事件監聽器](/docs/{{version}}/events)、[中介層](/docs/{{version}}/middleware)等等。此外，您可以在[佇列任務](/docs/{{version}}/queues)的 `handle` 方法中型別提示依賴。一旦您體驗過自動化和零配置依賴注入的力量，就會覺得沒有它就無法開發。

<a name="when-to-use-the-container"></a>
### 何時使用容器

多虧了零配置解析，您將經常在路由、控制器、事件監聽器和其他地方型別提示依賴，而無需手動與容器互動。例如，您可以在路由定義中型別提示 `Illuminate\Http\Request` 物件，以便輕鬆存取當前請求。儘管我們從不需要與容器互動來編寫此程式碼，但它在幕後管理這些依賴的注入：

    use Illuminate\Http\Request;

    Route::get('/', function (Request $request) {
        // ...
    });

在許多情況下，多虧了自動依賴注入和 [Facades](/docs/{{version}}/facades)，您無需**手動**從容器中綁定或解析任何內容，就能建立 Laravel 應用程式。**那麼，您何時會手動與容器互動呢？** 讓我們探討兩種情況。

首先，如果您編寫的類別實作了一個介面，並且您希望在路由或類別建構式上型別提示該介面，您必須[告訴容器如何解析該介面](#binding-interfaces-to-implementations)。其次，如果您正在[編寫一個 Laravel 擴充套件](/docs/{{version}}/packages)，並且您計劃與其他 Laravel 開發人員分享，您可能需要將擴充套件的服務綁定到容器中。

<a name="binding"></a>
## 綁定

<a name="binding-basics"></a>
### 綁定基礎

<a name="simple-bindings"></a>
#### 簡單綁定

幾乎所有的服務容器綁定都會在 [服務提供者](/docs/{{version}}/providers) 中註冊，因此這些範例大部分都會示範如何在此情境下使用容器。

在服務提供者中，您總是能透過 `$this->app` 屬性來存取容器。我們可以使用 `bind` 方法來註冊綁定，傳入我們希望註冊的類別或介面名稱，以及一個會回傳類別實例的閉包：

    use App\Services\Transistor;
    use App\Services\PodcastParser;
    use Illuminate\Contracts\Foundation\Application;

    $this->app->bind(Transistor::class, function (Application $app) {
        return new Transistor($app->make(PodcastParser::class));
    });

請注意，我們將容器本身作為引數傳遞給解析器。然後，我們可以使用容器來解析我們正在建構的物件的子依賴。

如前所述，您通常會在服務提供者內部與容器互動；但是，如果您想在服務提供者外部與容器互動，可以透過 `App` [Facade](/docs/{{version}}/facades) 來完成：

    use App\Services\Transistor;
    use Illuminate\Contracts\Foundation\Application;
    use Illuminate\Support\Facades\App;

    App::bind(Transistor::class, function (Application $app) {
        // ...
    });

您可以使用 `bindIf` 方法來註冊容器綁定，但前提是該型別尚未註冊綁定：

```php
$this->app->bindIf(Transistor::class, function (Application $app) {
    return new Transistor($app->make(PodcastParser::class));
});
```

> [!NOTE]  
> 如果類別不依賴任何介面，則無需將其綁定到容器中。容器不需要被告知如何建構這些物件，因為它可以使用反射 (reflection) 自動解析這些物件。

<a name="binding-a-singleton"></a>
#### 綁定單例 (Singleton)

`singleton` 方法將類別或介面綁定到容器中，該類別或介面只應解析一次。一旦單例綁定被解析，後續對容器的呼叫將回傳相同的物件實例：

    use App\Services\Transistor;
    use App\Services\PodcastParser;
    use Illuminate\Contracts\Foundation\Application;

    $this->app->singleton(Transistor::class, function (Application $app) {
        return new Transistor($app->make(PodcastParser::class));
    });

您可以使用 `singletonIf` 方法來註冊單例容器綁定，但前提是該型別尚未註冊綁定：

```php
$this->app->singletonIf(Transistor::class, function (Application $app) {
    return new Transistor($app->make(PodcastParser::class));
});
```

<a name="binding-scoped"></a>
#### 綁定 Scoped 單例

`scoped` 方法將類別或介面綁定到容器中，該類別或介面在給定的 Laravel 請求 / Job 生命周期內只應解析一次。雖然此方法與 `singleton` 方法類似，但使用 `scoped` 方法註冊的實例將在 Laravel 應用程式啟動新的「生命週期」時被清除，例如當 [Laravel Octane](/docs/{{version}}/octane) worker 處理新請求時，或當 Laravel [佇列 worker](/docs/{{version}}/queues) 處理新 Job 時：

    use App\Services\Transistor;
    use App\Services\PodcastParser;
    use Illuminate\Contracts\Foundation\Application;

    $this->app->scoped(Transistor::class, function (Application $app) {
        return new Transistor($app->make(PodcastParser::class));
    });

您可以使用 `scopedIf` 方法來註冊 Scoped 容器綁定，但前提是該型別尚未註冊綁定：

    $this->app->scopedIf(Transistor::class, function (Application $app) {
        return new Transistor($app->make(PodcastParser::class));
    });

<a name="binding-instances"></a>
#### 綁定實例

您也可以使用 `instance` 方法將現有的物件實例綁定到容器中。給定實例將在後續對容器的呼叫中始終被回傳：

    use App\Services\Transistor;
    use App\Services\PodcastParser;

    $service = new Transistor(new PodcastParser);

    $this->app->instance(Transistor::class, $service);

<a name="binding-interfaces-to-implementations"></a>
### 將介面綁定至實作

服務容器的一個非常強大的功能是它能夠將介面綁定到給定的實作。例如，假設我們有一個 `EventPusher` 介面和一個 `RedisEventPusher` 實作。一旦我們編寫了這個介面的 `RedisEventPusher` 實作，我們就可以這樣將它註冊到服務容器中：

    use App\Contracts\EventPusher;
    use App\Services\RedisEventPusher;

    $this->app->bind(EventPusher::class, RedisEventPusher::class);

這條語句告訴容器，當一個類別需要 `EventPusher` 的實作時，它應該注入 `RedisEventPusher`。現在我們可以在由容器解析的類別建構子中型別提示 `EventPusher` 介面。請記住，Laravel 應用程式中的控制器、事件監聽器、中介層以及各種其他型別的類別總是使用容器解析：

    use App\Contracts\EventPusher;

    /**
     * Create a new class instance.
     */
    public function __construct(
        protected EventPusher $pusher,
    ) {}

<a name="contextual-binding"></a>
### 情境綁定

有時您可能有兩個類別使用相同的介面，但您希望向每個類別注入不同的實作。例如，兩個控制器可能依賴 `Illuminate\Contracts\Filesystem\Filesystem` [契約](/docs/{{version}}/contracts) 的不同實作。Laravel 提供了一個簡單、流暢的介面來定義此行為：

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

<a name="contextual-attributes"></a>
### 情境屬性

由於情境綁定經常被用於注入驅動器或設定值實作，Laravel 提供多種情境綁定屬性，讓您無需手動在服務提供者中定義情境綁定即可注入這些類型的值。

舉例來說，`Storage` 屬性可用於注入特定的 [儲存磁碟](/docs/{{version}}/filesystem)：

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Container\Attributes\Storage;
use Illuminate\Contracts\Filesystem\Filesystem;

class PhotoController extends Controller
{
    public function __construct(
        #[Storage('local')] protected Filesystem $filesystem
    )
    {
        // ...
    }
}
```

除了 `Storage` 屬性之外，Laravel 還提供 `Auth`、`Cache`、`Config`、`DB`、`Log`、`RouteParameter` 和 [`Tag`](#tagging) 屬性：

```php
<?php

namespace App\Http\Controllers;

use App\Models\Photo;
use Illuminate\Container\Attributes\Auth;
use Illuminate\Container\Attributes\Cache;
use Illuminate\Container\Attributes\Config;
use Illuminate\Container\Attributes\DB;
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
        #[DB('mysql')] protected Connection $connection,
        #[Log('daily')] protected LoggerInterface $log,
        #[RouteParameter('photo')] protected Photo $photo,
        #[Tag('reports')] protected iterable $reports,
    )
    {
        // ...
    }
}
```

此外，Laravel 提供 `CurrentUser` 屬性，用於將目前已驗證的使用者注入到指定的路由或類別中：

```php
use App\Models\User;
use Illuminate\Container\Attributes\CurrentUser;

Route::get('/user', function (#[CurrentUser] User $user) {
    return $user;
})->middleware('auth');
```


<a name="defining-custom-attributes"></a>
#### 定義客製化屬性

您可以透過實作 `Illuminate\Contracts\Container\ContextualAttribute` 契約來建立自己的情境屬性。容器將呼叫您屬性的 `resolve` 方法，該方法應解析要注入到使用該屬性的類別中的值。在下面的範例中，我們將重新實作 Laravel 內建的 `Config` 屬性：

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
### 綁定原始值

有時您可能會有一個類別接收一些注入的類別，但也需要注入原始值，例如整數。您可以輕鬆地使用情境綁定來注入類別所需的任何值：

    use App\Http\Controllers\UserController;

    $this->app->when(UserController::class)
        ->needs('$variableName')
        ->give($value);

有時類別可能依賴於一個 [帶有標籤](#tagging) 的實例陣列。使用 `giveTagged` 方法，您可以輕鬆注入所有帶有該標籤的容器綁定：

    $this->app->when(ReportAggregator::class)
        ->needs('$reports')
        ->giveTagged('reports');

如果您需要從應用程式的設定檔中注入值，可以使用 `giveConfig` 方法：

    $this->app->when(ReportAggregator::class)
        ->needs('$timezone')
        ->giveConfig('app.timezone');


<a name="binding-typed-variadics"></a>
### 綁定具型態的可變引數

有時您可能會有一個類別，它使用可變引數的建構函式引數來接收一個具型態物件的陣列：

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

使用情境綁定，您可以透過提供 `give` 方法一個回傳已解析的 `Filter` 實例陣列的閉包來解析此依賴：

    $this->app->when(Firewall::class)
        ->needs(Filter::class)
        ->give(function (Application $app) {
              return [
                  $app->make(NullFilter::class),
                  $app->make(ProfanityFilter::class),
                  $app->make(TooLongFilter::class),
              ];
        });

為方便起見，您也可以直接提供一個類別名稱陣列，供容器在 `Firewall` 需要 `Filter` 實例時進行解析：

    $this->app->when(Firewall::class)
        ->needs(Filter::class)
        ->give([
            NullFilter::class,
            ProfanityFilter::class,
            TooLongFilter::class,
        ]);


<a name="variadic-tag-dependencies"></a>
#### 具型態可變引數的標籤依賴

有時類別可能具有一個具型態可變引數的依賴（`Report ...$reports`）。使用 `needs` 和 `giveTagged` 方法，您可以輕鬆地為給定的依賴注入所有帶有該 [標籤](#tagging) 的容器綁定：

    $this->app->when(ReportAggregator::class)
        ->needs(Report::class)
        ->giveTagged('reports');


<a name="tagging"></a>
### 標籤

有時，您可能需要解析所有特定「類別」的綁定。例如，您可能正在建立一個報告分析器，它接收許多不同 `Report` 介面實作的陣列。在註冊 `Report` 實作後，您可以使用 `tag` 方法為它們分配標籤：

    $this->app->bind(CpuReport::class, function () {
        // ...
    });

    $this->app->bind(MemoryReport::class, function () {
        // ...
    });

    $this->app->tag([CpuReport::class, MemoryReport::class], 'reports');

一旦服務被標記，您就可以透過容器的 `tagged` 方法輕鬆解析所有這些服務：

    $this->app->bind(ReportAnalyzer::class, function (Application $app) {
        return new ReportAnalyzer($app->tagged('reports'));
    });


<a name="extending-bindings"></a>
### 擴展綁定

`extend` 方法允許修改已解析的服務。例如，當服務被解析時，您可以執行額外的程式碼來裝飾或設定服務。`extend` 方法接受兩個參數：您要擴展的服務類別，以及一個應該回傳修改後服務的閉包。該閉包會接收正在解析的服務和容器實例：

    $this->app->extend(Service::class, function (Service $service, Application $app) {
        return new DecoratedService($service);
    });

<a name="resolving"></a>
## 解析

<a name="the-make-method"></a>
### `make` 方法

您可以使用 `make` 方法從容器中解析出一個類別實例。`make` 方法接受您希望解析的類別或介面名稱：

    use App\Services\Transistor;

    $transistor = $this->app->make(Transistor::class);

如果您類別的某些依賴無法透過容器解析，您可以將它們作為關聯陣列傳遞給 `makeWith` 方法來注入。例如，我們可能會手動傳遞 `Transistor` 服務所需的 `$id` 建構函式引數：

    use App\Services\Transistor;

    $transistor = $this->app->makeWith(Transistor::class, ['id' => 1]);

`bound` 方法可以用來判斷某個類別或介面是否已在容器中明確綁定：

    if ($this->app->bound(Transistor::class)) {
        // ...
    }

如果您在服務提供者 (service provider) 之外，且在您的程式碼中無法存取 `$app` 變數的位置，您可以使用 `App` [Facade](/docs/{{version}}/facades) 或 `app` [輔助函式](/docs/{{version}}/helpers#method-app) 從容器中解析類別實例：

    use App\Services\Transistor;
    use Illuminate\Support\Facades\App;

    $transistor = App::make(Transistor::class);

    $transistor = app(Transistor::class);

如果您想將 Laravel 容器實例本身注入到一個由容器解析的類別中，您可以在該類別的建構函式上型別提示 `Illuminate\Container\Container` 類別：

    use Illuminate\Container\Container;

    /**
     * Create a new class instance.
     */
    public function __construct(
        protected Container $container,
    ) {}

<a name="automatic-injection"></a>
### 自動注入

另外，更重要的是，您可以在由容器解析的類別的建構函式中型別提示其依賴，這些類別包括 [控制器](/docs/{{version}}/controllers)、[事件監聽器](/docs/{{version}}/events)、[中介層](/docs/{{version}}/middleware) 等。此外，您也可以在 [佇列任務](/docs/{{version}}/queues) 的 `handle` 方法中型別提示依賴。實際上，這是您的絕大多數物件應如何透過容器解析的方式。

例如，您可以在控制器的建構函式中型別提示應用程式定義的服務。該服務將自動被解析並注入到類別中：

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

<a name="method-invocation-and-injection"></a>
## 方法呼叫與注入

有時您可能希望在呼叫物件實例上的方法時，允許容器自動注入該方法的依賴。例如，給定以下類別：

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

您可以透過容器呼叫 `generate` 方法，如下所示：

    use App\PodcastStats;
    use Illuminate\Support\Facades\App;

    $stats = App::call([new PodcastStats, 'generate']);

`call` 方法接受任何 PHP 可呼叫 (callable)。容器的 `call` 方法甚至可以用來呼叫一個閉包 (closure)，同時自動注入其依賴：

    use App\Services\AppleMusic;
    use Illuminate\Support\Facades\App;

    $result = App::call(function (AppleMusic $apple) {
        // ...
    });

<a name="container-events"></a>
## 容器事件

服務容器在每次解析物件時都會觸發一個事件。您可以使用 `resolving` 方法監聽此事件：

    use App\Services\Transistor;
    use Illuminate\Contracts\Foundation\Application;

    $this->app->resolving(Transistor::class, function (Transistor $transistor, Application $app) {
        // Called when container resolves objects of type "Transistor"...
    });

    $this->app->resolving(function (mixed $object, Application $app) {
        // Called when container resolves object of any type...
    });

如您所見，正在解析的物件將傳遞給回呼 (callback)，允許您在將其提供給消費者之前，設定物件上的任何額外屬性。

<a name="rebinding"></a>
### 重新綁定

`rebinding` 方法允許您監聽服務重新綁定到容器的時機，這表示它在初始綁定之後再次註冊或被覆寫。當您需要更新依賴或每次特定綁定更新時修改行為時，這會很有用：

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

<a name="psr-11"></a>
## PSR-11

Laravel 的服務容器實作了 [PSR-11](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-11-container.md) 介面。因此，您可以型別提示 PSR-11 容器介面以取得 Laravel 容器的實例：

    use App\Services\Transistor;
    use Psr\Container\ContainerInterface;

    Route::get('/', function (ContainerInterface $container) {
        $service = $container->get(Transistor::class);

        // ...
    });

如果給定的識別碼無法解析，則會拋出例外。如果識別碼從未綁定，該例外將是 `Psr\Container\NotFoundExceptionInterface` 的一個實例。如果識別碼已綁定但無法解析，將拋出 `Psr\Container\ContainerExceptionInterface` 的一個實例。