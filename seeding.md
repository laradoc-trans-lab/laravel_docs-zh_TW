# 資料庫：資料填充

- [簡介](#introduction)
- [撰寫 Seeders](#writing-seeders)
    - [使用 Model Factories](#using-model-factories)
    - [呼叫額外的 Seeders](#calling-additional-seeders)
    - [靜音 Model 事件](#muting-model-events)
- [執行 Seeders](#running-seeders)

<a name="introduction"></a>
## 簡介

Laravel 包含了使用 seeder 類別來填充資料庫資料的功能。所有 seeder 類別皆儲存於 `database/seeders` 目錄中。預設情況下，系統已為您定義了一個 `DatabaseSeeder` 類別。從這個類別中，您可以使用 `call` 方法來執行其他 seeder 類別，讓您能夠控制資料填充順序。

> [!NOTE]  
> 在資料庫資料填充期間，[大量賦值保護](/docs/{{version}}/eloquent#mass-assignment) 會自動停用。

<a name="writing-seeders"></a>
## 撰寫 Seeders

要產生一個 seeder，請執行 `make:seeder` [Artisan 指令](/docs/{{version}}/artisan)。所有由框架產生的 seeder 都會放置在 `database/seeders` 目錄中：

```shell
php artisan make:seeder UserSeeder
```

一個 seeder 類別預設只包含一個方法：`run`。當 `db:seed` [Artisan 指令](/docs/{{version}}/artisan) 執行時，會呼叫此方法。在 `run` 方法中，您可以隨心所欲地將資料插入資料庫。您可以使用 [查詢產生器](/docs/{{version}}/queries) 手動插入資料，或者您也可以使用 [Eloquent model factories](/docs/{{version}}/eloquent-factories)。

舉例來說，讓我們修改預設的 `DatabaseSeeder` 類別，並在 `run` 方法中加入資料庫插入語句：

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

> [!NOTE]  
> 您可以在 `run` 方法的簽章中型別提示任何您需要的依賴。它們會透過 Laravel [服務容器](/docs/{{version}}/container) 自動解析。

<a name="using-model-factories"></a>
### 使用 Model Factories

當然，手動為每個模型指定屬性是很繁瑣的。取而代之，您可以使用 [model factories](/docs/{{version}}/eloquent-factories) 方便地產生大量的資料庫記錄。首先，請查閱 [model factory 文件](/docs/{{version}}/eloquent-factories) 以了解如何定義您的 factories。

舉例來說，讓我們建立 50 個各有一篇相關貼文的使用者：

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
### 呼叫額外的 Seeders

在 `DatabaseSeeder` 類別中，您可以使用 `call` 方法來執行額外的 seeder 類別。使用 `call` 方法能讓您將資料庫資料填充拆分成多個檔案，如此便不會讓單一 seeder 類別變得過於龐大。 `call` 方法接受一個應該執行的 seeder 類別陣列：

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

在執行資料填充時，您可能會希望阻止模型派發事件。您可以使用 `WithoutModelEvents` trait 來實現此目的。使用後，`WithoutModelEvents` trait 會確保沒有 model 事件被派發，即使是透過 `call` 方法執行額外的 seeder 類別也是如此：

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
## 執行 Seeders

您可以執行 `db:seed` Artisan 指令來填充您的資料庫。預設情況下，`db:seed` 指令會執行 `Database\Seeders\DatabaseSeeder` 類別，該類別可能會進而呼叫其他 seeder 類別。然而，您可以使用 `--class` 選項來指定一個特定的 seeder 類別單獨執行：

```shell
php artisan db:seed

php artisan db:seed --class=UserSeeder
```

您也可以結合 `--seed` 選項使用 `migrate:fresh` 指令來填充資料庫，該指令會捨棄所有資料表並重新執行所有資料遷移。此指令對於完全重建您的資料庫非常有用。 `--seeder` 選項可以用來指定一個特定的 seeder 執行：

```shell
php artisan migrate:fresh --seed

php artisan migrate:fresh --seed --seeder=UserSeeder
```

<a name="forcing-seeding-production"></a>
#### 強制 Seeders 在正式環境執行

有些資料填充操作可能會導致您更改或遺失資料。為了保護您，避免在正式環境的資料庫上執行資料填充指令，在 seeder 於 `production` 環境中執行之前，系統會提示您進行確認。若要強制 seeder 在沒有提示的情況下執行，請使用 `--force` 旗標：

```shell
php artisan db:seed --force
```