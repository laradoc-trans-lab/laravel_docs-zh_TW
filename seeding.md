# 資料庫：資料填充

- [介紹](#introduction)
- [撰寫 Seeder](#writing-seeders)
    - [使用 Model Factory](#using-model-factories)
    - [呼叫額外 Seeder](#calling-additional-seeders)
    - [靜音 Model 事件](#muting-model-events)
- [執行 Seeder](#running-seeders)

<a name="introduction"></a>
## 介紹

Laravel 包含了使用 seed class 對資料庫進行資料填充 (seeding) 的能力。所有的 seed class 都儲存在 `database/seeders` 目錄中。預設情況下，已經為您定義了一個 `DatabaseSeeder` class。從這個 class 中，您可以使用 `call` 方法來執行其他 seed class，讓您能夠控制資料填充的順序。

> [!NOTE]
> [大量指派保護](/docs/{{version}}/eloquent#mass-assignment) 在資料庫填充期間會自動停用。

<a name="writing-seeders"></a>
## 撰寫 Seeder

要產生一個 seeder，請執行 `make:seeder` [Artisan 指令](/docs/{{version}}/artisan)。所有由框架產生的 seeder 都會被放置在 `database/seeders` 目錄中：

```shell
php artisan make:seeder UserSeeder
```

Seeder class 預設只包含一個方法：`run`。當 `db:seed` [Artisan 指令](/docs/{{version}}/artisan) 被執行時，就會呼叫這個方法。在 `run` 方法中，您可以依照自己的意願將資料插入資料庫。您可以使用 [query builder](/docs/{{version}}/queries) 手動插入資料，也可以使用 [Eloquent model factories](/docs/{{version}}/eloquent-factories)。

作為一個範例，讓我們修改預設的 `DatabaseSeeder` class，並在 `run` 方法中加入一個資料庫插入語句：

```php
<?php

namespace Database\Seeders;

use Illuminate\Database\Seeder;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Hash;
use Illuminate\Support\Str;

class DatabaseSeeder extends Seeder
{
    /**
     * Run the database seeders.
     */
    public function run(): void
    {
        DB::table('users')->insert([
            'name' => Str::random(10),
            'email' => Str::random(10).'@example.com',
            'password' => Hash::make('password'),
        ]);
    }
}
```

> [!NOTE]
> 您可以在 `run` 方法的簽名中對任何所需的依賴項進行型別提示。它們將透過 Laravel [服務容器](/docs/{{version}}/container) 自動解析。

<a name="using-model-factories"></a>
### 使用 Model Factory

當然，手動為每個 model seed 指定屬性是很麻煩的。取而代之的是，您可以使用 [model factories](/docs/{{version}}/eloquent-factories) 方便地產生大量的資料庫記錄。首先，請查閱 [model factory 文件](/docs/{{version}}/eloquent-factories) 以了解如何定義您的 factory。

例如，讓我們建立 50 個 User，每個 User 都帶有一個相關的 Post：

```php
use App\Models\User;

/**
 * Run the database seeders.
 */
public function run(): void
{
    User::factory()
        ->count(50)
        ->hasPosts(1)
        ->create();
}
```

<a name="calling-additional-seeders"></a>
### 呼叫額外 Seeder

在 `DatabaseSeeder` class 中，您可以使用 `call` 方法來執行額外的 seed class。使用 `call` 方法可以讓您將資料庫填充拆分成多個檔案，這樣就不會有任何單一的 seeder class 變得過於龐大。這個 `call` 方法接受一個應該被執行的 seeder class 陣列：

```php
/**
 * Run the database seeders.
 */
public function run(): void
{
    $this->call([
        UserSeeder::class,
        PostSeeder::class,
        CommentSeeder::class,
    ]);
}
```

<a name="muting-model-events"></a>
### 靜音 Model 事件

在執行填充時，您可能希望阻止 model 派發事件。您可以使用 `WithoutModelEvents` trait 來實現這一點。使用時，`WithoutModelEvents` trait 確保不會派發任何 model 事件，即使是透過 `call` 方法執行額外的 seed class 也是如此：

```php
<?php

namespace Database\Seeders;

use Illuminate\Database\Seeder;
use Illuminate\Database\Console\Seeds\WithoutModelEvents;

class DatabaseSeeder extends Seeder
{
    use WithoutModelEvents;

    /**
     * Run the database seeders.
     */
    public function run(): void
    {
        $this->call([
            UserSeeder::class,
        ]);
    }
}
```

<a name="running-seeders"></a>
## 執行 Seeder

您可以執行 `db:seed` Artisan 指令來填充您的資料庫。預設情況下，`db:seed` 指令會執行 `Database\Seeders\DatabaseSeeder` class，而這個 class 又可能會呼叫其他的 seed class。然而，您可以使用 `--class` 選項來指定一個特定的 seeder class 單獨執行：

```shell
php artisan db:seed

php artisan db:seed --class=UserSeeder
```

您也可以使用 `migrate:fresh` 指令搭配 `--seed` 選項來填充資料庫，這會刪除所有資料表並重新執行所有的 migration。這個指令對於完全重建您的資料庫非常有用。可以使用 `--seeder` 選項來指定要執行的特定 seeder：

```shell
php artisan migrate:fresh --seed

php artisan migrate:fresh --seed --seeder=UserSeeder
```

<a name="forcing-seeding-production"></a>
#### 在生產環境中強制執行 Seeder

某些填充操作可能會導致您變更或遺失資料。為了保護您，避免對您的生產資料庫執行填充指令，在 `production` 環境中執行 seeder 之前，您會收到確認提示。若要強制 seeder 執行而不顯示提示，請使用 `--force` 旗標：

```shell
php artisan db:seed --force
```