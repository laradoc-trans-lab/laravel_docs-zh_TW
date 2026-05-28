# Concurrency

- [簡介](#introduction)
- [同時執行工作](#running-concurrent-tasks)
    - [命名結果](#named-results)
    - [工作逾時](#task-timeouts)
- [延遲同時執行的工作](#deferring-concurrent-tasks)

<a name="introduction"></a>
## 簡介

有時您可能需要執行數個互不相依且執行緩慢的工作。在許多情況下，透過同時執行這些工作可以顯著提升效能。Laravel 的 `Concurrency` Facade 提供了一個簡單且便利的 API，用於同時執行閉包 (Closures)。

<a name="how-it-works"></a>
#### 運作原理

Laravel 實現同時執行的方式是將指定的閉包序列化，並將其派發給一個隱藏的 Artisan CLI 指令，該指令會還原序列化閉包並在獨立的 PHP 行程(Processes)中呼叫它。閉包執行完畢後，產生的結果值會被序列化並傳回父行程。

`Concurrency` Facade 支援三種驅動：`process`（預設）、`fork` 與 `sync`。

與預設的 `process` 驅動相比，`fork` 驅動提供了更好的效能，但它只能在 PHP 的 CLI 情境中使用，因為 PHP 在 Web 請求期間不支援進行 fork（衍生子行程）。在執行 `fork` 驅動之前，您需要安裝 `spatie/fork` 套件：

```shell
composer require spatie/fork
```

`sync` 驅動主要用於測試，當您想停用所有同時執行功能，並單純地在父行程中依序執行指定的閉包時非常實用。

<a name="running-concurrent-tasks"></a>
## 同時執行工作

若要同時執行多個工作，您可以呼叫 `Concurrency` Facade 的 `run` 方法。`run` 方法接受一個閉包陣列，這些閉包將在子 PHP 行程中同時執行：

```php
use Illuminate\Support\Facades\Concurrency;
use Illuminate\Support\Facades\DB;

[$userCount, $orderCount] = Concurrency::run([
    fn () => DB::table('users')->count(),
    fn () => DB::table('orders')->count(),
]);
```

若要使用特定的驅動，您可以使用 `driver` 方法：

```php
$results = Concurrency::driver('fork')->run(...);
```

或者，若要變更預設的同時執行驅動，您應該透過 `config:publish` Artisan 指令發布 `concurrency` 設定檔，並更新該檔案中的 `default` 選項：

```shell
php artisan config:publish concurrency
```

<a name="named-results"></a>
### 命名結果

如果您想透過名稱而不是位置來取得同時執行的工作結果，可以提供一個關聯陣列的閉包。每個結果都將使用與其對應閉包相同的鍵值 (Key) 傳回：

```php
use Illuminate\Support\Facades\Concurrency;
use Illuminate\Support\Facades\DB;

$results = Concurrency::run([
    'users' => fn () => DB::table('users')->count(),
    'orders' => fn () => DB::table('orders')->count(),
]);

$userCount = $results['users'];
$orderCount = $results['orders'];
```

<a name="task-timeouts"></a>
### 工作逾時

使用 `process` 驅動（預設）時，您可以透過向 `run` 方法提供逾時時間，來指定允許同時執行工作執行的最大秒數，超過該時間工作將被終止：

```php
use Illuminate\Support\Facades\Concurrency;
use Illuminate\Support\Facades\DB;

[$userCount, $orderCount] = Concurrency::run([
    fn () => DB::table('users')->count(),
    fn () => DB::table('orders')->count(),
], timeout: 30);
```

如果您偏好更具表達力的逾時定義，也可以提供一個 `CarbonInterval` 實例：

```php
use Illuminate\Support\Facades\Concurrency;

use function Illuminate\Support\seconds;

Concurrency::run([...], timeout: seconds(30));
```

<a name="deferring-concurrent-tasks"></a>
## 延遲同時執行的工作

如果您想同時執行一組閉包，但對這些閉包傳回的結果不感興趣，您可以考慮使用 `defer` 方法。當呼叫 `defer` 方法時，指定的閉包不會立即執行。相反地，Laravel 會在 HTTP 回應傳送給使用者之後，才同時執行這些閉包：

```php
use App\Services\Metrics;
use Illuminate\Support\Facades\Concurrency;

Concurrency::defer([
    fn () => Metrics::report('users'),
    fn () => Metrics::report('orders'),
]);
```