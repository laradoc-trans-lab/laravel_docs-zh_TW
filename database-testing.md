# 資料庫測試

- [簡介](#introduction)
    - [在每次測試後重設資料庫](#resetting-the-database-after-each-test)
- [Model Factories](#model-factories)
- [執行 Seeders](#running-seeders)
- [可用的斷言](#available-assertions)

<a name="introduction"></a>
## 簡介

Laravel 提供了各種實用的工具與斷言，讓測試資料庫驅動的應用程式變得更加容易。此外，Laravel 的 Model Factories 與 Seeders 讓您能夠輕鬆地使用應用程式的 Eloquent Model 與關聯來建立測試資料庫記錄。我們將在接下來的文件中討論所有這些強大的功能。

<a name="resetting-the-database-after-each-test"></a>
### 在每次測試後重設資料庫

在進一步深入之前，讓我們先討論如何在每次測試後重設資料庫，以避免前一個測試的資料干擾後續的測試。Laravel 內建的 `Illuminate\Foundation\Testing\RefreshDatabase` Trait 將會為您處理這件事。只需在您的測試類別上使用此 Trait 即可：

```php tab=Pest
<?php

use Illuminate\Foundation\Testing\RefreshDatabase;

uses(RefreshDatabase::class);

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

如果您的資料庫 Schema 是最新的，`Illuminate\Foundation\Testing\RefreshDatabase` Trait 並不會執行資料庫遷移。相反地，它只會在資料庫交易中執行測試。因此，由未使用此 Trait 的測試案例新增到資料庫的任何記錄，可能仍然存在於資料庫中。

如果您想完全重設資料庫，可以使用 `Illuminate\Foundation\Testing\DatabaseMigrations` 或 `Illuminate\Foundation\Testing\DatabaseTruncation` Trait。然而，這兩種選項都比 `RefreshDatabase` Trait 慢得多。

<a name="model-factories"></a>
## Model Factories

在測試時，您可能需要在執行測試之前向資料庫插入一些記錄。Laravel 允許您為每個 [Eloquent Model](/docs/{{version}}/eloquent) 定義一組預設屬性，而不是在建立這些測試資料時手動指定每個欄位的值，這可以透過使用 [Model Factories](/docs/{{version}}/eloquent-factories) 來實現。

要了解更多關於建立和使用 Model Factories 來建立 Model 的資訊，請查閱完整的 [Model Factory 說明文件](/docs/{{version}}/eloquent-factories)。一旦您定義了 Model Factory，您就可以在測試中使用該 Factory 來建立 Model：

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
## 執行 Seeders

如果您想在功能測試期間使用 [Database Seeders](/docs/{{version}}/seeding) 來填充資料庫，您可以呼叫 `seed` 方法。預設情況下，`seed` 方法將會執行 `DatabaseSeeder`，它應該會執行所有其他的 Seeder。或者，您可以將特定的 Seeder 類別名稱傳遞給 `seed` 方法：

```php tab=Pest
<?php

use Database\Seeders\OrderStatusSeeder;
use Database\Seeders\TransactionStatusSeeder;
use Illuminate\Foundation\Testing\RefreshDatabase;

uses(RefreshDatabase::class);

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

或者，您可以指示 Laravel 在每個使用 `RefreshDatabase` Trait 的測試之前自動填充資料庫。您可以透過在您的基礎測試類別上定義 `$seed` 屬性來實現這一點：

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

當 `$seed` 屬性為 `true` 時，測試將在每個使用 `RefreshDatabase` Trait 的測試之前執行 `Database\Seeders\DatabaseSeeder` 類別。但是，您可以透過在測試類別上定義 `$seeder` 屬性來指定應執行的特定 Seeder：

    use Database\Seeders\OrderStatusSeeder;

    /**
     * Run a specific seeder before each test.
     *
     * @var string
     */
    protected $seeder = OrderStatusSeeder::class;

<a name="available-assertions"></a>
## 可用的斷言

Laravel 為您的 [Pest](https://pestphp.com) 或 [PHPUnit](https://phpunit.de) 功能測試提供了多個資料庫斷言。我們將在下面討論每個斷言。

<a name="assert-database-count"></a>
#### assertDatabaseCount

斷言資料庫中的某個資料表包含給定數量的記錄：

    $this->assertDatabaseCount('users', 5);

<a name="assert-database-empty"></a>
#### assertDatabaseEmpty

斷言資料庫中的某個資料表不包含任何記錄：

    $this->assertDatabaseEmpty('users');
    
<a name="assert-database-has"></a>
#### assertDatabaseHas

斷言資料庫中的某個資料表包含符合給定鍵/值查詢條件的記錄：

    $this->assertDatabaseHas('users', [
        'email' => 'sally @example.com',
    ]);

<a name="assert-database-missing"></a>
#### assertDatabaseMissing

斷言資料庫中的某個資料表不包含符合給定鍵/值查詢條件的記錄：

    $this->assertDatabaseMissing('users', [
        'email' => 'sally @example.com',
    ]);

<a name="assert-deleted"></a>
#### assertSoftDeleted

`assertSoftDeleted` 方法可用於斷言給定的 Eloquent Model 已被「軟刪除」：

    $this->assertSoftDeleted($user);

<a name="assert-not-deleted"></a>
#### assertNotSoftDeleted

`assertNotSoftDeleted` 方法可用於斷言給定的 Eloquent Model 尚未被「軟刪除」：

    $this->assertNotSoftDeleted($user);

<a name="assert-model-exists"></a>
#### assertModelExists

斷言給定的 Model 存在於資料庫中：

    use App\Models\User;

    $user = User::factory()->create();

    $this->assertModelExists($user);

<a name="assert-model-missing"></a>
#### assertModelMissing

斷言給定的 Model 不存在於資料庫中：

    use App\Models\User;

    $user = User::factory()->create();

    $user->delete();

    $this->assertModelMissing($user);

<a name="expects-database-query-count"></a>
#### expectsDatabaseQueryCount

`expectsDatabaseQueryCount` 方法可以在測試開始時呼叫，以指定您預期在測試期間執行的資料庫查詢總數。如果實際執行的查詢數量與此預期不完全匹配，測試將會失敗：

    $this->expectsDatabaseQueryCount(5);

    // Test...
