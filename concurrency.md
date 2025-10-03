# 並行

- [簡介](#introduction)
- [執行並行任務](#running-concurrent-tasks)
- [延遲並行任務](#deferring-concurrent-tasks)

<a name="introduction"></a>
## 簡介

有時您可能需要執行多個互不依賴的緩慢任務。在許多情況下，透過並行執行這些任務可以顯著提升效能。Laravel 的 `Concurrency` facade 提供了一個簡單、便捷的 API，用於並行執行閉包。


<a name="how-it-works"></a>
#### 運作方式

Laravel 透過序列化給定的閉包，並將其分派到一個隱藏的 Artisan CLI 指令來實現並行，該指令會反序列化閉包並在其自身的 PHP 程序中呼叫它。閉包被呼叫後，其結果值會被序列化回父程序。

`Concurrency` facade 支援三種驅動器：`process` (預設)、`fork` 和 `sync`。

`fork` 驅動器相較於預設的 `process` 驅動器能提供更高的效能，但它只能在 PHP 的 CLI 環境中使用，因為 PHP 在網頁請求期間不支援 fork。在使用 `fork` 驅動器之前，您需要安裝 `spatie/fork` 套件：

```shell
composer require spatie/fork
```

`sync` 驅動器主要在測試期間有用，當您希望禁用所有並行功能並僅在父程序中依序執行給定的閉包時。


<a name="running-concurrent-tasks"></a>
## 執行並行任務

若要執行並行任務，您可以呼叫 `Concurrency` facade 的 `run` 方法。`run` 方法接受一個閉包陣列，這些閉包應在子 PHP 程序中同時執行：

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

或者，若要變更預設的並行驅動器，您應該透過 `config:publish` Artisan 指令發佈 `concurrency` 設定檔，並更新檔案中的 `default` 選項：

```shell
php artisan config:publish concurrency
```


<a name="deferring-concurrent-tasks"></a>
## 延遲並行任務

如果您想並行執行一個閉包陣列，但對這些閉包返回的結果不感興趣，您應該考慮使用 `defer` 方法。當 `defer` 方法被呼叫時，給定的閉包不會立即執行。相反地，Laravel 會在 HTTP 響應已發送給使用者之後，並行執行這些閉包：

```php
use App\Services\Metrics;
use Illuminate\Support\Facades\Concurrency;

Concurrency::defer([
    fn () => Metrics::report('users'),
    fn () => Metrics::report('orders'),
]);
```