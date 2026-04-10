# 資料庫測試

- [簡介](#introduction)
    - [在每次測試後重設資料庫](#resetting-the-database-after-each-test)
- [模型工廠](#model-factories)
- [執行資料填充](#running-seeders)
- [可用的斷言](#available-assertions)

<a name="introduction"></a>
## 簡介

Laravel 提供了各種實用的工具與斷言，讓您能更輕鬆地測試由資料庫驅動的應用程式。此外，Laravel 的模型工廠與填充器讓您能毫不費力地使用應用程式的 Eloquent 模型與關聯來建立測試資料庫紀錄。我們將在接下來的文件中討論所有這些強大的功能。


<a name="resetting-the-database-after-each-test"></a>
### 在每次測試後重設資料庫

在深入研究之前，讓我們討論如何在每次測試後重設資料庫，以確保前一次測試的資料不會干擾後續的測試。Laravel 內建的 `Illuminate\Foundation\Testing\RefreshDatabase` trait 會為您處理這件事。只需在您的測試類別中使用該 trait：

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

如果您的結構描述 (schema) 已是最新，`Illuminate\Foundation\Testing\RefreshDatabase` trait 將不會對資料庫執行遷移。相反地，它僅會在資料庫交易 (database transaction) 中執行測試。因此，由未使用此 trait 的測試案例所新增到資料庫中的任何紀錄，可能仍然存在於資料庫中。

如果您想要完全重設資料庫，可以使用 `Illuminate\Foundation\Testing\DatabaseMigrations` 或 `Illuminate\Foundation\Testing\DatabaseTruncation` trait。然而，這兩個選項都比 `RefreshDatabase` trait 慢得多。


<a name="model-factories"></a>
## 模型工廠

在測試時，您可能需要在執行測試前向資料庫插入一些紀錄。Laravel 允許您使用 [模型工廠](/docs/{{version}}/eloquent-factories) 為您的每個 [Eloquent 模型](/docs/{{version}}/eloquent) 定義一組預設屬性，而不需要在建立測試資料時手動指定每個欄位的值。

若要了解更多關於建立與利用模型工廠來產生模型的資訊，請參閱完整的 [模型工廠文件](/docs/{{version}}/eloquent-factories)。一旦定義了模型工廠，您就可以在測試中利用該工廠來建立模型：

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
## 執行資料填充

如果您想在功能測試期間使用 [資料庫填充器](/docs/{{version}}/seeding) 來填充資料庫，可以呼叫 `seed` 方法。預設情況下，`seed` 方法將執行 `DatabaseSeeder`，而它應該會執行所有其他填充器。或者，您可以將特定的填充器類別名稱傳遞給 `seed` 方法：

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

或者，您可以指示 Laravel 在每個使用 `RefreshDatabase` trait 的測試之前自動填充資料庫。您可以透過將 `Seed` 屬性添加到您的基礎測試類別來達成此目的：

```php
<?php

namespace Tests;

use Illuminate\Foundation\Testing\Attributes\Seed;
use Illuminate\Foundation\Testing\TestCase as BaseTestCase;

#[Seed]
abstract class TestCase extends BaseTestCase
{
}
```

當 `Seed` 屬性存在時，測試將在每個使用 `RefreshDatabase` trait 的測試之前執行 `Database\Seeders\DatabaseSeeder` 類別。不過，您可以使用 `Seeder` 屬性在您的測試類別上指定應該執行的特定填充器：

```php
<?php

namespace Tests\Feature;

use Database\Seeders\OrderStatusSeeder;
use Illuminate\Foundation\Testing\Attributes\Seeder;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

#[Seeder(OrderStatusSeeder::class)]
class OrderTest extends TestCase
{
    use RefreshDatabase;

    // ...
}
```


<a name="available-assertions"></a>
## 可用的斷言

Laravel 為您的 [Pest](https://pestphp.com) 或 [PHPUnit](https://phpunit.de) 功能測試提供了數個資料庫斷言。我們將在下方討論每一個斷言。


<a name="assert-database-count"></a>
#### assertDatabaseCount

斷言資料庫中的表包含指定數量的紀錄：

```php
$this->assertDatabaseCount('users', 5);
```


<a name="assert-database-empty"></a>
#### assertDatabaseEmpty

斷言資料庫中的表不包含任何紀錄：

```php
$this->assertDatabaseEmpty('users');
```


<a name="assert-database-has"></a>
#### assertDatabaseHas

斷言資料庫中的表包含符合給定鍵/值查詢限制的紀錄：

```php
$this->assertDatabaseHas('users', [
    'email' => 'sally@example.com',
]);
```


<a name="assert-database-missing"></a>
#### assertDatabaseMissing

斷言資料庫中的表不包含符合給定鍵/值查詢限制的紀錄：

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

`assertNotSoftDeleted` 方法可用於斷言給定的 Eloquent 模型未被「軟刪除」：

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

可以在測試開始時呼叫 `expectsDatabaseQueryCount` 方法，以指定您預期在測試期間執行的資料庫查詢總數。如果實際執行的查詢數量與此預期不完全一致，測試將會失敗：

```php
$this->expectsDatabaseQueryCount(5);

// Test...
```