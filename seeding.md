# 資料庫：填充 (Seeding)

- [簡介](#introduction)
- [撰寫填充器 (Seeders)](#writing-seeders)
    - [使用 Model Factory](#using-model-factories)
    - [呼叫額外的填充器](#calling-additional-seeders)
    - [靜音 Model 事件](#muting-model-events)
- [執行填充器](#running-seeders)

<a name="introduction"></a>
## 簡介

Laravel 提供了使用填充類別 (seed classes) 來填充資料庫的功能。所有的填充類別都儲存在 `database/seeders` 目錄中。預設情況下，系統會為您定義一個 `DatabaseSeeder` 類別。您可以從這個類別中使用 `call` 方法來執行其他的填充類別，讓您能夠控制填充的順序。

> [!NOTE]  
> 在資料庫填充期間，[大量賦值保護](/docs/{{version}}/eloquent#mass-assignment)會自動停用。

<a name="writing-seeders"></a>
## 撰寫填充器

若要產生一個填充器，請執行 `make:seeder` [Artisan 命令](/docs/{{version}}/artisan)。所有由框架產生的填充器都會被放置在 `database/seeders` 目錄中：

```shell
php artisan make:seeder UserSeeder
```

填充器類別預設只包含一個方法：`run`。當執行 `db:seed` [Artisan 命令](/docs/{{version}}/artisan)時，就會呼叫此方法。在 `run` 方法中，您可以依照自己的需求將資料插入資料庫。您可以使用 [查詢產生器](/docs/{{version}}/queries) 手動插入資料，也可以使用 [Eloquent Model Factory](/docs/{{version}}/eloquent-factories)。

舉例來說，讓我們修改預設的 `DatabaseSeeder` 類別，並在 `run` 方法中加入一個資料庫插入語句：

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
                'email' => Str::random(10).' @example.com',
                'password' => Hash::make('password'),
            ]);
        }
    }

> [!NOTE]  
> 您可以在 `run` 方法的簽章中型別提示 (type-hint) 任何您需要的依賴。它們將透過 Laravel [Service Container](/docs/{{version}}/container) 自動解析。

<a name="using-model-factories"></a>
### 使用 Model Factory

當然，手動為每個 Model 填充指定屬性是很麻煩的。您可以改用 [Model Factory](/docs/{{version}}/eloquent-factories) 來方便地產生大量的資料庫記錄。首先，請查閱 [Model Factory 說明文件](/docs/{{version}}/eloquent-factories) 以了解如何定義您的 Factory。

例如，讓我們建立 50 個使用者，每個使用者都有一篇相關的貼文：

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

<a name="calling-additional-seeders"></a>
### 呼叫額外的填充器

在 `DatabaseSeeder` 類別中，您可以使用 `call` 方法來執行額外的填充類別。使用 `call` 方法可以讓您將資料庫填充拆分成多個檔案，這樣就不會讓單一填充器類別變得過於龐大。`call` 方法接受一個應執行的填充類別陣列：

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

<a name="muting-model-events"></a>
### 靜音 Model 事件

在執行填充時，您可能希望阻止 Model 觸發事件。您可以使用 `WithoutModelEvents` Trait 來實現這一點。使用 `WithoutModelEvents` Trait 時，即使透過 `call` 方法執行額外的填充類別，也能確保不會觸發任何 Model 事件：

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

<a name="running-seeders"></a>
## 執行填充器

您可以執行 `db:seed` Artisan 命令來填充您的資料庫。預設情況下，`db:seed` 命令會執行 `Database\Seeders\DatabaseSeeder` 類別，該類別又可以呼叫其他的填充類別。但是，您可以使用 `--class` 選項來指定要單獨執行的特定填充器類別：

```shell
php artisan db:seed

php artisan db:seed --class=UserSeeder
```

您也可以使用 `migrate:fresh` 命令結合 `--seed` 選項來填充您的資料庫，這將會刪除所有資料表並重新執行所有的 Migration。此命令對於完全重建資料庫非常有用。`--seeder` 選項可用於指定要執行的特定填充器：

```shell
php artisan migrate:fresh --seed

php artisan migrate:fresh --seed --seeder=UserSeeder
```

<a name="forcing-seeding-production"></a>
#### 強制在正式環境中執行填充器

某些填充操作可能會導致您更改或遺失資料。為了保護您免於在正式環境資料庫上執行填充命令，在 `production` 環境中執行填充器之前，系統會提示您進行確認。若要強制填充器在沒有提示的情況下執行，請使用 `--force` 旗標：

```shell
php artisan db:seed --force
```
