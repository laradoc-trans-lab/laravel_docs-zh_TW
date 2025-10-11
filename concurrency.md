# 並行

- [簡介](#introduction)
- [執行並行任務](#running-concurrent-tasks)
- [延遲並行任務](#deferring-concurrent-tasks)

<a name="introduction"></a>
## 簡介

> [!WARNING]
> Laravel 的 `Concurrency` facade 目前為測試版，我們正在收集社群回饋。

有時您可能需要執行數個彼此沒有依賴關係的緩慢任務。在許多情況下，透過並行執行這些任務，可以實現顯著的性能提升。Laravel 的 `Concurrency` facade 提供一個簡單、方便的 API，用於並行執行閉包。

<a name="concurrency-compatibility"></a>
#### Concurrency 相容性

如果您是從 Laravel 10.x 應用程式升級到 Laravel 11.x，您可能需要將 `ConcurrencyServiceProvider` 加入到您應用程式的 `config/app.php` 設定檔中的 `providers` 陣列：

```php
'providers' => ServiceProvider::defaultProviders()->merge([
    /*
     * Package Service Providers...
     */
    Illuminate\Concurrency\ConcurrencyServiceProvider::class, // [tl! add]

    /*
     * Application Service Providers...
     */
    App\Providers\AppServiceProvider::class,
    App\Providers\AuthServiceProvider::class,
    // App\Providers\BroadcastServiceProvider::class,
    App\Providers\EventServiceProvider::class,
    App\Providers\RouteServiceProvider::class,
])->toArray(),
```

<a name="how-it-works"></a>
#### 運作方式

Laravel 透過序列化給定的閉包，並將它們分派到一個隱藏的 Artisan CLI 指令來實現並行，該指令會反序列化這些閉包並在自己的 PHP 處理程序中呼叫它們。閉包被呼叫後，其結果值會被序列化回父處理程序。

`Concurrency` facade 支援三種驅動器：`process` (預設)、`fork` 和 `sync`。

相較於預設的 `process` 驅動器，`fork` 驅動器提供改進的性能，但它只能在 PHP 的 CLI 環境中使用，因為 PHP 在網頁請求期間不支援 forking。在使用 `fork` 驅動器之前，您需要安裝 `spatie/fork` 套件：

```bash
composer require spatie/fork
```

`sync` 驅動器主要在測試期間很有用，當您想要禁用所有並行功能，並只是在父處理程序中依序執行給定的閉包時。

<a name="running-concurrent-tasks"></a>
## 執行並行任務

為了執行並行任務，您可以呼叫 `Concurrency` facade 的 `run` 方法。`run` 方法接受一個閉包陣列，這些閉包應該在子 PHP 處理程序中同時執行：

```php
use Illuminate\Support\Facades\Concurrency;
use Illuminate\Support\Facades\DB;

[$userCount, $orderCount] = Concurrency::run([
    fn () => DB::table('users')->count(),
    fn () => DB::table('orders')->count(),
]);
```

若要使用特定的驅動器，您可以使用 `driver` 方法：

```php
$results = Concurrency::driver('fork')->run(...);
```

或者，要更改預設的並行驅動器，您應該透過 `config:publish` Artisan 指令發布 `concurrency` 設定檔，並更新檔案中的 `default` 選項：

```bash
php artisan config:publish concurrency
```

<a name="deferring-concurrent-tasks"></a>
## 延遲並行任務

如果您想並行執行一個閉包陣列，但不關心這些閉包返回的結果，您應該考慮使用 `defer` 方法。當 `defer` 方法被呼叫時，給定的閉包不會立即執行。相反地，Laravel 會在 HTTP 回應發送給使用者之後，並行執行這些閉包：

```php
use App\Services\Metrics;
use Illuminate\Support\Facades\Concurrency;

Concurrency::defer([
    fn () => Metrics::report('users'),
    fn () => Metrics::report('orders'),
]);
```