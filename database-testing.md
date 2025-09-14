# 資料庫測試

- [簡介](#introduction)
    - [每次測試後重設資料庫](#resetting-the-database-after-each-test)
- [模型工廠](#model-factories)
- [執行資料填充器](#running-seeders)
- [可用的斷言](#available-assertions)

<a name="introduction"></a>
## 簡介

Laravel 提供了各種有用的工具和斷言，讓測試資料庫驅動的應用程式變得更容易。此外，Laravel 模型工廠和資料填充器讓您可以使用應用程式的 Eloquent 模型和關聯輕鬆建立測試資料庫記錄。我們將在以下文件中討論所有這些強大的功能。

<a name="resetting-the-database-after-each-test"></a>
### 每次測試後重設資料庫

在進一步深入之前，讓我們先討論如何在每次測試後重設資料庫，以便前一個測試的資料不會干擾後續測試。Laravel 內建的 `Illuminate\Foundation\Testing\RefreshDatabase` trait 將為您處理此問題。只需在您的測試類別上使用該 trait：

```php tab=Pest
<?php

use Illuminate\Foundation\Testing\RefreshDatabase;

pest()->use(RefreshDatabase::class);

test('basic example', function () {
    $response = $this->get('/');

    // ...
});
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class ExampleTest extends TestCase
{
    use RefreshDatabase;

    /**
     * A basic functional test example.
     */
    public function test_basic_example(): void
    {
        $response = $this->get('/');

        // ...
    }
}
```

`Illuminate\Foundation\Testing\RefreshDatabase` trait 不會在您的資料庫結構是最新的情況下遷移資料庫。相反，它只會在資料庫交易中執行測試。因此，未經此 trait 處理的測試案例新增到資料庫的任何記錄可能仍存在於資料庫中。

如果您想完全重設資料庫，可以使用 `Illuminate\Foundation\Testing\DatabaseMigrations` 或 `Illuminate\Foundation\Testing\DatabaseTruncation` trait。然而，這兩種選項都比 `RefreshDatabase` trait 慢得多。

<a name="model-factories"></a>
## 模型工廠

測試時，您可能需要在執行測試之前向資料庫插入一些記錄。Laravel 允許您為每個 [Eloquent 模型](/docs/{{version}}/eloquent) 使用 [模型工廠](/docs/{{version}}/eloquent-factories) 定義一組預設屬性，而不是在建立此測試資料時手動指定每個欄位的值。

要了解有關建立和利用模型工廠來建立模型的更多資訊，請查閱完整的 [模型工廠文件](/docs/{{version}}/eloquent-factories)。定義模型工廠後，您可以在測試中使用該工廠來建立模型：

```php tab=Pest
use App\Models\User;

test('models can be instantiated', function () {
    $user = User::factory()->create();

    // ...
});
```

```php tab=PHPUnit
use App\Models\User;

public function test_models_can_be_instantiated(): void
{
    $user = User::factory()->create();

    // ...
}
```

<a name="running-seeders"></a>
## 執行資料填充器

如果您想在功能測試期間使用 [資料庫填充器](/docs/{{version}}/seeding) 來填充資料庫，可以呼叫 `seed` 方法。預設情況下，`seed` 方法將執行 `DatabaseSeeder`，它應該執行所有其他資料填充器。或者，您可以將特定的資料填充器類別名稱傳遞給 `seed` 方法：

```php tab=Pest
<?php

use Database\Seeders\OrderStatusSeeder;
use Database\Seeders\TransactionStatusSeeder;
use Illuminate\Foundation\Testing\RefreshDatabase;

pest()->use(RefreshDatabase::class);

test('orders can be created', function () {
    // 執行 DatabaseSeeder...
    $this->seed();

    // 執行特定的資料填充器...
    $this->seed(OrderStatusSeeder::class);

    // ...

    // 執行特定的資料填充器陣列...
    $this->seed([
        OrderStatusSeeder::class,
        TransactionStatusSeeder::class,
        // ...
    ]);
});
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use Database\Seeders\OrderStatusSeeder;
use Database\Seeders\TransactionStatusSeeder;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class ExampleTest extends TestCase
{
    use RefreshDatabase;

    /**
     * Test creating a new order.
     */
    public function test_orders_can_be_created(): void
    {
        // 執行 DatabaseSeeder...
        $this->seed();

        // 執行特定的資料填充器...
        $this->seed(OrderStatusSeeder::class);

        // ...

        // 執行特定的資料填充器陣列...
        $this->seed([
            OrderStatusSeeder::class,
            TransactionStatusSeeder::class,
            // ...
        ]);
    }
}
```

或者，您可以指示 Laravel 在每個使用 `RefreshDatabase` trait 的測試之前自動填充資料庫。您可以透過在基礎測試類別上定義 `$seed` 屬性來實現此目的：

```php
<?php

namespace Tests;

use Illuminate\Foundation\Testing\TestCase as BaseTestCase;

abstract class TestCase extends BaseTestCase
{
    /**
     * Indicates whether the default seeder should run before each test.
     *
     * @var bool
     */
    protected $seed = true;
}
```

當 `$seed` 屬性為 `true` 時，測試將在每個使用 `RefreshDatabase` trait 的測試之前執行 `Database\Seeders\DatabaseSeeder` 類別。但是，您可以透過在測試類別上定義 `$seeder` 屬性來指定應執行的特定資料填充器：

```php
use Database\Seeders\OrderStatusSeeder;

/**
 * Run a specific seeder before each test.
 *
 * @var string
 */
protected $seeder = OrderStatusSeeder::class;
```

<a name="available-assertions"></a>
## 可用的斷言

Laravel 為您的 [Pest](https://pestphp.com) 或 [PHPUnit](https://phpunit.de) 功能測試提供了多個資料庫斷言。我們將在下面討論每個斷言。

<a name="assert-database-count"></a>
#### assertDatabaseCount

斷言資料庫中的資料表包含給定數量的記錄：

```php
$this->assertDatabaseCount('users', 5);
```

<a name="assert-database-empty"></a>
#### assertDatabaseEmpty

斷言資料庫中的資料表不包含任何記錄：

```php
$this->assertDatabaseEmpty('users');
```

<a name="assert-database-has"></a>
#### assertDatabaseHas

斷言資料庫中的資料表包含符合給定鍵/值查詢約束的記錄：

```php
$this->assertDatabaseHas('users', [
    'email' => 'sally@example.com',
]);
```

<a name="assert-database-missing"></a>
#### assertDatabaseMissing

斷言資料庫中的資料表不包含符合給定鍵/值查詢約束的記錄：

```php
$this->assertDatabaseMissing('users', [
    'email' => 'sally@example.com',
]);
```

<a name="assert-deleted"></a>
#### assertSoftDeleted

`assertSoftDeleted` 方法可用於斷言給定的 Eloquent 模型已被「軟刪除」：

```php
$this->assertSoftDeleted($user);
```

<a name="assert-not-deleted"></a>
#### assertNotSoftDeleted

`assertNotSoftDeleted` 方法可用於斷言給定的 Eloquent 模型尚未被「軟刪除」：

```php
$this->assertNotSoftDeleted($user);
```

<a name="assert-model-exists"></a>
#### assertModelExists

斷言給定的模型或模型集合存在於資料庫中：

```php
use App\Models\User;

$user = User::factory()->create();

$this->assertModelExists($user);
```

<a name="assert-model-missing"></a>
#### assertModelMissing

斷言給定的模型或模型集合不存在於資料庫中：

```php
use App\Models\User;

$user = User::factory()->create();

$user->delete();

$this->assertModelMissing($user);
```

<a name="expects-database-query-count"></a>
#### expectsDatabaseQueryCount

`expectsDatabaseQueryCount` 方法可以在測試開始時呼叫，以指定測試期間預期執行的資料庫查詢總數。如果實際執行的查詢數量與此預期不完全匹配，則測試將失敗：

```php
$this->expectsDatabaseQueryCount(5);

// Test...
```

