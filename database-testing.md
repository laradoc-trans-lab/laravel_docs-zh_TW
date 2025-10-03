# 資料庫測試

- [簡介](#introduction)
    - [在每次測試後重設資料庫](#resetting-the-database-after-each-test)
- [模型工廠](#model-factories)
- [執行 Seeder](#running-seeders)
- [可用的斷言](#available-assertions)

<a name="introduction"></a>
## 簡介

Laravel 提供了各種有用的工具和斷言，讓測試您的資料庫驅動應用程式變得更容易。此外，Laravel 模型工廠和 Seeder 讓使用您應用程式的 Eloquent models 和關聯來建立測試資料庫紀錄變得輕而易舉。我們將在以下文件中討論所有這些強大的功能。


<a name="resetting-the-database-after-each-test"></a>
### 在每次測試後重設資料庫

在進一步說明之前，讓我們先討論如何在每次測試後重設資料庫，以避免先前測試的資料干擾後續測試。Laravel 內建的 `Illuminate\Foundation\Testing\RefreshDatabase` trait 會為您處理此問題。只需在您的測試類別上使用此 trait 即可：

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

`Illuminate\Foundation\Testing\RefreshDatabase` trait 不會在您的資料庫 Schema 是最新的情況下遷移您的資料庫。相反地，它只會在資料庫交易中執行測試。因此，未曾使用此 trait 的測試案例所新增的任何紀錄可能仍會存在於資料庫中。

如果您想完全重設資料庫，可以改用 `Illuminate\Foundation\Testing\DatabaseMigrations` 或 `Illuminate\Foundation\Testing\DatabaseTruncation` traits。然而，這兩種選項都比 `RefreshDatabase` trait 慢得多。


<a name="model-factories"></a>
## 模型工廠

進行測試時，您可能需要在執行測試前於資料庫中插入一些紀錄。Laravel 允許您為每個 [Eloquent models](/docs/{{version}}/eloquent) 使用 [模型工廠](/docs/{{version}}/eloquent-factories) 來定義一組預設屬性，而非在建立這些測試資料時手動指定每個欄位的值。

要了解更多關於建立和利用模型工廠來建立模型，請參閱完整的 [模型工廠文件](/docs/{{version}}/eloquent-factories)。一旦您定義了模型工廠，您就可以在測試中利用該工廠來建立模型：

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
## 執行 Seeder

如果您想在功能測試期間使用 [資料庫 Seeder](/docs/{{version}}/seeding) 來填充您的資料庫，您可以呼叫 `seed` 方法。預設情況下，`seed` 方法將執行 `DatabaseSeeder`，它應該會執行所有其他 Seeder。或者，您可以將特定的 Seeder 類別名稱傳遞給 `seed` 方法：

```php tab=Pest
<?php

use Database\Seeders\OrderStatusSeeder;
use Database\Seeders\TransactionStatusSeeder;
use Illuminate\Foundation\Testing\RefreshDatabase;

pest()->use(RefreshDatabase::class);

test('orders can be created', function () {
    // Run the DatabaseSeeder...
    $this->seed();

    // Run a specific seeder...
    $this->seed(OrderStatusSeeder::class);

    // ...

    // Run an array of specific seeders...
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
        // Run the DatabaseSeeder...
        $this->seed();

        // Run a specific seeder...
        $this->seed(OrderStatusSeeder::class);

        // ...

        // Run an array of specific seeders...
        $this->seed([
            OrderStatusSeeder::class,
            TransactionStatusSeeder::class,
            // ...
        ]);
    }
}
```

此外，您可以指示 Laravel 在每個使用 `RefreshDatabase` trait 的測試之前自動為資料庫填充 Seeder。您可以透過在您的基礎測試類別上定義一個 `$seed` 屬性來實現這一點：

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

當 `$seed` 屬性為 `true` 時，測試將在每個使用 `RefreshDatabase` trait 的測試之前執行 `Database\Seeders\DatabaseSeeder` 類別。然而，您可以透過在您的測試類別上定義 `$seeder` 屬性來指定一個應執行的特定 Seeder：

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

Laravel 為您的 [Pest](https://pestphp.com) 或 [PHPUnit](https://phpunit.de) 功能測試提供了多種資料庫斷言。我們將在下方討論每個這些斷言。


<a name="assert-database-count"></a>
#### assertDatabaseCount

斷言資料庫中的資料表包含給定數量的紀錄：

```php
$this->assertDatabaseCount('users', 5);
```


<a name="assert-database-empty"></a>
#### assertDatabaseEmpty

斷言資料庫中的資料表不包含任何紀錄：

```php
$this->assertDatabaseEmpty('users');
```


<a name="assert-database-has"></a>
#### assertDatabaseHas

斷言資料庫中的資料表包含符合給定鍵/值查詢限制的紀錄：

```php
$this->assertDatabaseHas('users', [
    'email' => 'sally@example.com',
]);
```


<a name="assert-database-missing"></a>
#### assertDatabaseMissing

斷言資料庫中的資料表不包含符合給定鍵/值查詢限制的紀錄：

```php
$this->assertDatabaseMissing('users', [
    'email' => 'sally@example.com',
]);
```


<a name="assert-deleted"></a>
#### assertSoftDeleted

`assertSoftDeleted` 方法可用於斷言給定的 Eloquent model 已被「軟刪除」：

```php
$this->assertSoftDeleted($user);
```


<a name="assert-not-deleted"></a>
#### assertNotSoftDeleted

`assertNotSoftDeleted` 方法可用於斷言給定的 Eloquent model 尚未被「軟刪除」：

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

`expectsDatabaseQueryCount` 方法可以在測試開始時呼叫，以指定您期望在測試期間執行的資料庫查詢總數。如果實際執行的查詢數量與此預期不完全符合，測試將會失敗：

```php
$this->expectsDatabaseQueryCount(5);

// Test...
```