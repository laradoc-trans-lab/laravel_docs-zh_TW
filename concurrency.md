# 並行處理

- [簡介](#introduction)
- [執行並行任務](#running-concurrent-tasks)
- [延遲並行任務](#deferring-concurrent-tasks)

<a name="introduction"></a>
## 簡介

有時候您可能需要執行一些彼此不依賴的耗時任務。在許多情況下，透過並行執行這些任務可以顯著提升效能。Laravel 的 `Concurrency` 外觀 (Facade) 提供了一個簡單、方便的 API，用於並行執行閉包 (closure)。

<a name="how-it-works"></a>
#### 運作方式

Laravel 透過序列化給定的閉包，並將它們分派到一個隱藏的 Artisan CLI 命令來實現並行處理，該命令會反序列化閉包並在其自己的 PHP 處理程序中呼叫它。閉包被呼叫後，結果值會被序列化回父處理程序。

`Concurrency` 外觀支援三種驅動程式：`process` (預設)、`fork` 和 `sync`。

`fork` 驅動程式比預設的 `process` 驅動程式提供更好的效能，但它只能在 PHP 的 CLI 環境中使用，因為 PHP 在網頁請求期間不支援分叉 (forking)。在使用 `fork` 驅動程式之前，您需要安裝 `spatie/fork` 套件：

```shell
composer require spatie/fork
```

`sync` 驅動程式主要在測試期間有用，當您想要禁用所有並行處理並僅在父處理程序中依序執行給定的閉包時。

<a name="running-concurrent-tasks"></a>
## 執行並行任務

要執行並行任務，您可以呼叫 `Concurrency` 外觀的 `run` 方法。`run` 方法接受一個閉包陣列，這些閉包應該在子 PHP 處理程序中同時執行：

```php
use Illuminate\Support\Facades\Concurrency;
use Illuminate\Support\Facades\DB;

[$userCount, $orderCount] = Concurrency::run([
    fn () => DB::table('users')->count(),
    fn () => DB::table('orders')->count(),
]);
```

要使用特定的驅動程式，您可以使用 `driver` 方法：

```php
$results = Concurrency::driver('fork')->run(...);
```

或者，要更改預設的並行處理驅動程式，您應該透過 `config:publish` Artisan 命令發布 `concurrency` 設定檔，並更新檔案中的 `default` 選項：

```shell
php artisan config:publish concurrency
```

<a name="deferring-concurrent-tasks"></a>
## 延遲並行任務

如果您想並行執行一個閉包陣列，但對這些閉包返回的結果不感興趣，您應該考慮使用 `defer` 方法。當呼叫 `defer` 方法時，給定的閉包不會立即執行。相反，Laravel 會在 HTTP 回應發送給使用者後並行執行這些閉包：

```php
use App\Services\Metrics;
use Illuminate\Support\Facades\Concurrency;

Concurrency::defer([
    fn () => Metrics::report('users'),
    fn () => Metrics::report('orders'),
]);
```

