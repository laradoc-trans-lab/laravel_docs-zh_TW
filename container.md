# Service Container

- [簡介](#introduction)
    - [零配置解析](#zero-configuration-resolution)
    - [何時使用 Container](#when-to-use-the-container)
- [綁定](#binding)
    - [綁定基礎](#binding-basics)
    - [將介面綁定到實作](#binding-interfaces-to-implementations)
    - [情境綁定](#contextual-binding)
    - [情境屬性](#contextual-attributes)
    - [綁定基本型別](#binding-primitives)
    - [綁定具型別的可變參數](#binding-typed-variadics)
    - [標籤](#tagging)
    - [擴展綁定](#extending-bindings)
- [解析](#resolving)
    - [Make 方法](#the-make-method)
    - [自動注入](#automatic-injection)
- [方法呼叫與注入](#method-invocation-and-injection)
- [Container 事件](#container-events)
    - [重新綁定](#rebinding)
- [PSR-11](#psr-11)

<a name="introduction"></a>
## 簡介

Laravel Service Container 是一個強大的工具，用於管理類別依賴並執行依賴注入。依賴注入是一個花俏的詞彙，其本質上意味著：類別依賴是透過建構子，或在某些情況下，透過「setter」方法「注入」到類別中。

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

在這個範例中，`PodcastController` 需要從 Apple Music 等資料來源擷取 Podcast。因此，我們將**注入**一個能夠擷取 Podcast 的服務。由於服務是注入的，我們在測試應用程式時，可以輕鬆地「mock」或建立 `AppleMusic` 服務的虛擬實作。

深入理解 Laravel Service Container 對於建構強大、大型的應用程式，以及為 Laravel 核心本身做出貢獻至關重要。

<a name="zero-configuration-resolution"></a>
### 零配置解析

如果一個類別沒有依賴，或者只依賴於其他具體類別（而非介面），則無需指示 Container 如何解析該類別。例如，您可以將以下程式碼放在 `routes/web.php` 檔案中：

    <?php

    class Service
    {
        // ...
    }

    Route::get('/', function (Service $service) {
        die($service::class);
    });

在這個範例中，造訪應用程式的 `/` 路由將自動解析 `Service` 類別並將其注入到路由的處理器中。這是一個改變遊戲規則的功能。這意味著您可以開發應用程式並利用依賴注入，而無需擔心臃腫的配置檔案。

幸運的是，您在建構 Laravel 應用程式時將編寫的許多類別會自動透過 Container 接收其依賴，包括 [controllers](/docs/{{version}}/controllers)、[event listeners](/docs/{{version}}/events)、[middleware](/docs/{{version}}/middleware) 等。此外，您可以在 [queued jobs](/docs/{{version}}/queues) 的 `handle` 方法中型別提示依賴。一旦您體驗到自動化和零配置依賴注入的強大功能，就感覺沒有它就無法開發。

<a name="when-to-use-the-container"></a>
### 何時使用 Container

由於零配置解析，您通常會在路由、controllers、event listeners 和其他地方型別提示依賴，而無需手動與 Container 互動。例如，您可能會在路由定義上型別提示 `Illuminate\Http\Request` 物件，以便輕鬆存取目前的請求。儘管我們從未手動與 Container 互動來編寫此程式碼，但它在幕後管理著這些依賴的注入：

    use Illuminate\Http\Request;

    Route::get('/', function (Request $request) {
        // ...
    });

在許多情況下，由於自動依賴注入和 [facades](/docs/{{version}}/facades)，您可以建構 Laravel 應用程式，而**無需**手動綁定或解析 Container 中的任何內容。**那麼，您何時會手動與 Container 互動呢？** 讓我們探討兩種情況。

首先，如果您編寫的類別實作了一個介面，並且您希望在路由或類別建構子上型別提示該介面，則必須[告訴 Container 如何解析該介面](#binding-interfaces-to-implementations)。其次，如果您正在[編寫一個 Laravel 套件](/docs/{{version}}/packages)，並且您打算與其他 Laravel 開發者分享，您可能需要將套件的服務綁定到 Container 中。

<a name="binding"></a>
## 綁定

<a name="binding-basics"></a>
### 綁定基礎

<a name="simple-bindings"></a>
#### 簡單綁定

幾乎所有 Service Container 綁定都將在 [service providers](/docs/{{version}}/providers) 中註冊，因此這些範例大多會示範在該情境下使用 Container。

在 Service Provider 中，您始終可以透過 `$this->app` 屬性存取 Container。我們可以使用 `bind` 方法註冊綁定，傳入我們希望註冊的類別或介面名稱以及一個回傳類別實例的閉包：

    use App\Services\Transistor;
    use App\Services\PodcastParser;
    use Illuminate\Contracts\Foundation\Application;

    $this->app->bind(Transistor::class, function (Application $app) {
        return new Transistor($app->make(PodcastParser::class));
    });

請注意，我們將 Container 本身作為參數接收到解析器中。然後，我們可以使用 Container 來解析我們正在建構的物件的子依賴。

如前所述，您通常會在 Service Providers 中與 Container 互動；但是，如果您想在 Service Provider 之外與 Container 互動，您可以透過 `App` [facade](/docs/{{version}}/facades) 進行：

    use App\Services\Transistor;
    use Illuminate\Contracts\Foundation\Application;
    use Illuminate\Support\Facades\App;

    App::bind(Transistor::class, function (Application $app) {
        // ...
    });

您可以使用 `bindIf` 方法僅在尚未為給定型別註冊綁定的情況下註冊 Container 綁定：

```php
$this->app->bindIf(Transistor::class, function (Application $app) {
    return new Transistor($app->make(PodcastParser::class));
});
```

> [!NOTE]
> 如果類別不依賴任何介面，則無需將其綁定到 Container 中。Container 無需指示如何建構這些物件，因為它可以使用反射自動解析這些物件。

<a name="binding-a-singleton"></a>
#### 綁定 Singleton

`singleton` 方法將類別或介面綁定到 Container 中，該類別或介面應僅解析一次。一旦解析了 Singleton 綁定，後續對 Container 的呼叫將回傳相同的物件實例：

    use App\Services\Transistor;
    use App\Services\PodcastParser;
    use Illuminate\Contracts\Foundation\Application;

    $this->app->singleton(Transistor::class, function (Application $app) {
        return new Transistor($app->make(PodcastParser::class));
    });

您可以使用 `singletonIf` 方法僅在尚未為給定型別註冊綁定的情況下註冊 Singleton Container 綁定：

```php
$this->app->singletonIf(Transistor::class, function (Application $app) {
    return new Transistor($app->make(PodcastParser::class));
});
```

<a name="binding-scoped"></a>
#### 綁定 Scoped Singletons

`scoped` 方法將類別或介面綁定到 Container 中，該類別或介面應在給定的 Laravel 請求/任務生命週期內僅解析一次。雖然此方法類似於 `singleton` 方法，但使用 `scoped` 方法註冊的實例將在 Laravel 應用程式啟動新的「生命週期」時被清除，例如當 [Laravel Octane](/docs/{{version}}/octane) worker 處理新請求時，或當 Laravel [queue worker](/docs/{{version}}/queues) 處理新任務時：

    use App\Services\Transistor;
    use App\Services\PodcastParser;
    use Illuminate\Contracts\Foundation\Application;

    $this->app->scoped(Transistor::class, function (Application $app) {
        return new Transistor($app->make(PodcastParser::class));
    });

您可以使用 `scopedIf` 方法僅在尚未為給定型別註冊綁定的情況下註冊 Scoped Container 綁定：

    $this->app->scopedIf(Transistor::class, function (Application $app) {
        return new Transistor($app->make(PodcastParser::class));
    });

<a name="binding-instances"></a>
#### 綁定實例

您也可以使用 `instance` 方法將現有物件實例綁定到 Container 中。給定的實例將始終在後續對 Container 的呼叫中回傳：

    use App\Services\Transistor;
    use App\Services\PodcastParser;

    $service = new Transistor(new PodcastParser);

    $this->app->instance(Transistor::class, $service);

<a name="binding-interfaces-to-implementations"></a>
### 將介面綁定到實作

Service Container 一個非常強大的功能是它能夠將介面綁定到給定的實作。例如，假設我們有一個 `EventPusher` 介面和一個 `RedisEventPusher` 實作。一旦我們編寫了此介面的 `RedisEventPusher` 實作，我們就可以將其註冊到 Service Container 中，如下所示：

    use App\Contracts\EventPusher;
    use App\Services\RedisEventPusher;

    $this->app->bind(EventPusher::class, RedisEventPusher::class);

此語句告訴 Container，當類別需要 `EventPusher` 的實作時，它應該注入 `RedisEventPusher`。現在我們可以在由 Container 解析的類別的建構子中型別提示 `EventPusher` 介面。請記住，Laravel 應用程式中的 controllers、event listeners、middleware 和各種其他型別的類別始終使用 Container 解析：

    use App\Contracts\EventPusher;

    /**
     * Create a new class instance.
     */
    public function __construct(
        protected EventPusher $pusher,
    ) {}

<a name="contextual-binding"></a>
### 情境綁定

有時您可能有兩個類別使用相同的介面，但您希望將不同的實作注入到每個類別中。例如，兩個 controllers 可能依賴於 `Illuminate\Contracts\Filesystem\Filesystem` [contract](/docs/{{version}}/contracts) 的不同實作。Laravel 提供了一個簡單、流暢的介面來定義此行為：

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

由於情境綁定通常用於注入驅動程式的實作或配置值，Laravel 提供了各種情境綁定屬性，允許在不手動定義 Service Providers 中的情境綁定的情況下注入這些型別的值。

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
    )
    {
        // ...
    }
}
```

除了 `Storage` 屬性之外，Laravel 還提供了 `Auth`、`Cache`、`Config`、`DB`、`Log`、`RouteParameter` 和 [`Tag`](#tagging) 屬性：

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

此外，Laravel 提供了一個 `CurrentUser` 屬性，用於將目前已驗證的使用者注入到給定的路由或類別中：

```php
use App\Models\User;
use Illuminate\Container\Attributes\CurrentUser;

Route::get('/user', function (#[CurrentUser] User $user) {
    return $user;
})->middleware('auth');
```

<a name="defining-custom-attributes"></a>
#### 定義自訂屬性

您可以透過實作 `Illuminate\Contracts\Container\ContextualAttribute` contract 來建立自己的情境屬性。Container 將呼叫您屬性的 `resolve` 方法，該方法應解析應注入到使用該屬性的類別中的值。在下面的範例中，我們將重新實作 Laravel 內建的 `Config` 屬性：

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
### 綁定基本型別

有時您可能有一個類別接收一些注入的類別，但也需要注入一個基本型別值，例如整數。您可以輕鬆使用情境綁定來注入類別可能需要的任何值：

    use App\Http\Controllers\UserController;

    $this->app->when(UserController::class)
        ->needs('$variableName')
        ->give($value);

有時類別可能依賴於[標籤化](#tagging)實例的陣列。使用 `giveTagged` 方法，您可以輕鬆注入所有具有該標籤的 Container 綁定：

    $this->app->when(ReportAggregator::class)
        ->needs('$reports')
        ->giveTagged('reports');

如果您需要從應用程式的其中一個配置檔案中注入值，您可以使用 `giveConfig` 方法：

    $this->app->when(ReportAggregator::class)
        ->needs('$timezone')
        ->giveConfig('app.timezone');

<a name="binding-typed-variadics"></a>
### 綁定具型別的可變參數

有時，您可能有一個類別使用可變參數建構子引數接收一個具型別物件陣列：

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

使用情境綁定，您可以透過向 `give` 方法提供一個閉包來解析此依賴，該閉包回傳已解析的 `Filter` 實例陣列：

    $this->app->when(Firewall::class)
        ->needs(Filter::class)
        ->give(function (Application $app) {
              return [
                  $app->make(NullFilter::class),
                  $app->make(ProfanityFilter::class),
                  $app->make(TooLongFilter::class),
              ];
        });

為方便起見，您也可以只提供一個類別名稱陣列，以便在 `Firewall` 需要 `Filter` 實例時由 Container 解析：

    $this->app->when(Firewall::class)
        ->needs(Filter::class)
        ->give([
            NullFilter::class,
            ProfanityFilter::class,
            TooLongFilter::class,
        ]);

<a name="variadic-tag-dependencies"></a>
#### 可變參數標籤依賴

有時類別可能具有可變參數依賴，其型別提示為給定類別（`Report ...$reports`）。使用 `needs` 和 `giveTagged` 方法，您可以輕鬆地為給定依賴注入所有具有該[標籤](#tagging)的 Container 綁定：

    $this->app->when(ReportAggregator::class)
        ->needs(Report::class)
        ->giveTagged('reports');

<a name="tagging"></a>
### 標籤

有時，您可能需要解析某個「類別」的所有綁定。例如，您可能正在建構一個報告分析器，它接收一個包含許多不同 `Report` 介面實作的陣列。註冊 `Report` 實作後，您可以使用 `tag` 方法為它們分配一個標籤：

    $this->app->bind(CpuReport::class, function () {
        // ...
    });

    $this->app->bind(MemoryReport::class, function () {
        // ...
    });

    $this->app->tag([CpuReport::class, MemoryReport::class], 'reports');

服務被標籤後，您可以透過 Container 的 `tagged` 方法輕鬆解析所有服務：

    $this->app->bind(ReportAnalyzer::class, function (Application $app) {
        return new ReportAnalyzer($app->tagged('reports'));
    });

<a name="extending-bindings"></a>
### 擴展綁定

`extend` 方法允許修改已解析的服務。例如，當服務被解析時，您可以執行額外的程式碼來裝飾或配置服務。`extend` 方法接受兩個引數，您要擴展的服務類別和一個應回傳修改後服務的閉包。該閉包接收正在解析的服務和 Container 實例：

    $this->app->extend(Service::class, function (Service $service, Application $app) {
        return new DecoratedService($service);
    });

<a name="resolving"></a>
## 解析

<a name="the-make-method"></a>
### Make 方法

您可以使用 `make` 方法從 Container 解析類別實例。`make` 方法接受您希望解析的類別或介面名稱：

    use App\Services\Transistor;

    $transistor = $this->app->make(Transistor::class);

如果您的某些類別依賴無法透過 Container 解析，您可以透過將它們作為關聯陣列傳遞給 `makeWith` 方法來注入它們。例如，我們可以手動傳遞 `Transistor` 服務所需的 `$id` 建構子引數：

    use App\Services\Transistor;

    $transistor = $this->app->makeWith(Transistor::class, ['id' => 1]);

`bound` 方法可用於判斷類別或介面是否已在 Container 中明確綁定：

    if ($this->app->bound(Transistor::class)) {
        // ...
    }

如果您在 Service Provider 之外的程式碼位置，無法存取 `$app` 變數，您可以使用 `App` [facade](/docs/{{version}}/facades) 或 `app` [helper](/docs/{{version}}/helpers#method-app) 從 Container 解析類別實例：

    use App\Services\Transistor;
    use Illuminate\Support\Facades\App;

    $transistor = App::make(Transistor::class);

    $transistor = app(Transistor::class);

如果您希望將 Laravel Container 實例本身注入到由 Container 解析的類別中，您可以在類別的建構子上型別提示 `Illuminate\Container\Container` 類別：

    use Illuminate\Container\Container;

    /**
     * Create a new class instance.
     */
    public function __construct(
        protected Container $container,
    ) {}

<a name="automatic-injection"></a>
### 自動注入

或者，更重要的是，您可以在由 Container 解析的類別的建構子中型別提示依賴，包括 [controllers](/docs/{{version}}/controllers)、[event listeners](/docs/{{version}}/events)、[middleware](/docs/{{version}}/middleware) 等。此外，您可以在 [queued jobs](/docs/{{version}}/queues) 的 `handle` 方法中型別提示依賴。實際上，這就是大多數物件應該由 Container 解析的方式。

例如，您可以在 controller 的建構子中型別提示應用程式定義的服務。該服務將自動解析並注入到類別中：

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

有時您可能希望在物件實例上呼叫方法，同時允許 Container 自動注入該方法的依賴。例如，給定以下類別：

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

您可以透過 Container 呼叫 `generate` 方法，如下所示：

    use App\PodcastStats;
    use Illuminate\Support\Facades\App;

    $stats = App::call([new PodcastStats, 'generate']);

`call` 方法接受任何 PHP callable。Container 的 `call` 方法甚至可以用於呼叫閉包，同時自動注入其依賴：

    use App\Services\AppleMusic;
    use Illuminate\Support\Facades\App;

    $result = App::call(function (AppleMusic $apple) {
        // ...
    });

<a name="container-events"></a>
## Container 事件

Service Container 每次解析物件時都會觸發一個事件。您可以使用 `resolving` 方法監聽此事件：

    use App\Services\Transistor;
    use Illuminate\Contracts\Foundation\Application;

    $this->app->resolving(Transistor::class, function (Transistor $transistor, Application $app) {
        // Called when container resolves objects of type "Transistor"...
    });

    $this->app->resolving(function (mixed $object, Application $app) {
        // Called when container resolves object of any type...
    });

如您所見，正在解析的物件將傳遞給回呼，允許您在將物件提供給其消費者之前設定物件上的任何其他屬性。

<a name="rebinding"></a>
### 重新綁定

`rebinding` 方法允許您監聽服務何時重新綁定到 Container，這表示它在初始綁定之後再次註冊或被覆蓋。當您需要每次更新特定綁定時更新依賴或修改行為時，這會很有用：

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

Laravel 的 Service Container 實作了 [PSR-11](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-11-container.md) 介面。因此，您可以型別提示 PSR-11 Container 介面以取得 Laravel Container 的實例：

    use App\Services\Transistor;
    use Psr\Container\ContainerInterface;

    Route::get('/', function (ContainerInterface $container) {
        $service = $container->get(Transistor::class);

        // ...
    });

如果給定的識別碼無法解析，則會拋出異常。如果識別碼從未綁定，則異常將是 `Psr\Container\NotFoundExceptionInterface` 的實例。如果識別碼已綁定但無法解析，則將拋出 `Psr\Container\ContainerExceptionInterface` 的實例。
