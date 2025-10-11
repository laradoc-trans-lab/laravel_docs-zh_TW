# 測試：入門

- [簡介](#introduction)
- [環境](#environment)
- [建立測試](#creating-tests)
- [執行測試](#running-tests)
    - [平行執行測試](#running-tests-in-parallel)
    - [報告測試覆蓋率](#reporting-test-coverage)
    - [測試分析](#profiling-tests)

<a name="introduction"></a>
## 簡介

Laravel 的設計旨在考慮測試。事實上，Laravel 預設就包含了對 [Pest](https://pestphp.com) 和 [PHPUnit](https://phpunit.de) 的測試支援，並且為你的應用程式設定好了 `phpunit.xml` 檔案。此框架還提供了方便的輔助方法，讓你可以清晰地測試你的應用程式。

預設情況下，你的應用程式 `tests` 目錄中包含兩個目錄：`Feature` 和 `Unit`。單元測試 (Unit tests) 是一種專注於程式碼中非常小且獨立部分的測試。事實上，大多數單元測試可能都只關注單一方法。在「Unit」測試目錄中的測試不會啟動你的 Laravel 應用程式，因此無法存取你的應用程式資料庫或其他框架服務。

功能測試 (Feature tests) 可以測試程式碼中較大部分，包括多個物件如何相互作用，甚至是對 JSON 端點的完整 HTTP 請求。**通常，你的大多數測試都應該是功能測試。這些類型的測試為你的系統整體是否按預期運作提供了最大的信心。**

在 `Feature` 和 `Unit` 測試目錄中都提供了一個 `ExampleTest.php` 檔案。安裝新的 Laravel 應用程式後，執行 `vendor/bin/pest`、`vendor/bin/phpunit` 或 `php artisan test` 命令來執行你的測試。

<a name="environment"></a>
## 環境

執行測試時，由於在 `phpunit.xml` 檔案中定義了環境變數，Laravel 會自動將 [設定環境](/docs/{{version}}/configuration#environment-configuration) 設定為 `testing`。Laravel 還會自動將 session 和 cache 設定為 `array` 驅動器，這樣在測試時不會有任何 session 或 cache 資料被持久化。

你可以根據需要自由定義其他測試環境的設定值。`testing` 環境變數可以在你的應用程式 `phpunit.xml` 檔案中進行設定，但在執行測試之前，請務必使用 `config:clear` Artisan 命令清除你的設定快取！

<a name="the-env-testing-environment-file"></a>
#### `.env.testing` 環境檔案

此外，你可以在專案的根目錄中建立一個 `.env.testing` 檔案。當執行 Pest 和 PHPUnit 測試或執行帶有 `--env=testing` 選項的 Artisan 命令時，此檔案將取代 `.env` 檔案被使用。

<a name="creating-tests"></a>
## 建立測試

要建立新的測試案例，請使用 `make:test` Artisan 命令。預設情況下，測試會被放置在 `tests/Feature` 目錄中：

```shell
php artisan make:test UserTest
```

如果你想在 `tests/Unit` 目錄中建立測試，可以在執行 `make:test` 命令時使用 `--unit` 選項：

```shell
php artisan make:test UserTest --unit
```

> [!NOTE]  
> 測試 Stub 可以透過 [Stub 發佈](/docs/{{version}}/artisan#stub-customization) 進行自訂。

測試生成後，你就可以像往常一樣使用 Pest 或 PHPUnit 定義測試。要執行測試，請從你的終端機執行 `vendor/bin/pest`、`vendor/bin/phpunit` 或 `php artisan test` 命令：

```php tab=Pest
<?php

test('basic', function () {
    expect(true)->toBeTrue();
});
```

```php tab=PHPUnit
<?php

namespace Tests\Unit;

use PHPUnit\Framework\TestCase;

class ExampleTest extends TestCase
{
    /**
     * A basic test example.
     */
    public function test_basic_test(): void
    {
        $this->assertTrue(true);
    }
}
```

> [!WARNING]  
> 如果你在測試類別中定義了自己的 `setUp` / `tearDown` 方法，請務必呼叫父類別中相對應的 `parent::setUp()` / `parent::tearDown()` 方法。通常，你應該在自己的 `setUp` 方法開始時呼叫 `parent::setUp()`，並在 `tearDown` 方法結束時呼叫 `parent::tearDown()`。

<a name="running-tests"></a>
## 執行測試

如前所述，一旦您寫好測試，您可以使用 `pest` 或 `phpunit` 來執行它們：

```shell tab=Pest
./vendor/bin/pest
```

```shell tab=PHPUnit
./vendor/bin/phpunit
```

除了 `pest` 或 `phpunit` 命令之外，您還可以使用 `test` Artisan 命令來執行測試。Artisan 測試執行器提供詳細的測試報告，以便於開發和除錯：

```shell
php artisan test
```

任何可以傳遞給 `pest` 或 `phpunit` 命令的引數，也可以傳遞給 Artisan 的 `test` 命令：

```shell
php artisan test --testsuite=Feature --stop-on-failure
```

<a name="running-tests-in-parallel"></a>
### 平行執行測試

預設情況下，Laravel 和 Pest / PHPUnit 在單一程序中依序執行您的測試。然而，您可以透過在多個程序中同時執行測試，大幅減少執行測試所需的時間。首先，您應該安裝 `brianium/paratest` Composer 套件作為「開發」依賴。然後，在執行 `test` Artisan 命令時，加入 `--parallel` 選項：

```shell
composer require brianium/paratest --dev

php artisan test --parallel
```

預設情況下，Laravel 將建立與您機器上可用 CPU 核心數一樣多的程序。然而，您可以使用 `--processes` 選項來調整程序數量：

```shell
php artisan test --parallel --processes=4
```

> [!WARNING]  
> 當平行執行測試時，某些 Pest / PHPUnit 選項 (例如 `--do-not-cache-result`) 可能不可用。

<a name="parallel-testing-and-databases"></a>
#### 平行測試與資料庫

只要您設定了主要的資料庫連線，Laravel 會自動處理為每個執行您測試的平行程序建立和遷移測試資料庫。測試資料庫將會附加一個程序 token，該 token 在每個程序中都是唯一的。例如，如果您有兩個平行測試程序，Laravel 將會建立並使用 `your_db_test_1` 和 `your_db_test_2` 測試資料庫。

預設情況下，測試資料庫會在 `test` Artisan 命令呼叫之間持續存在，以便後續的 `test` 呼叫可以再次使用它們。然而，您可以使用 `--recreate-databases` 選項來重新建立它們：

```shell
php artisan test --parallel --recreate-databases
```

<a name="parallel-testing-hooks"></a>
#### 平行測試掛鉤

有時，您可能需要準備某些資源供您的應用程式測試使用，以便多個測試程序可以安全地使用它們。

使用 `ParallelTesting` Facade，您可以指定在程序或測試案例的 `setUp` 和 `tearDown` 階段執行的程式碼。這些閉包會收到 `$token` 和 `$testCase` 變數，分別包含程序 token 和當前的測試案例：

    <?php

    namespace App\Providers;

    use Illuminate\Support\Facades\Artisan;
    use Illuminate\Support\Facades\ParallelTesting;
    use Illuminate\Support\ServiceProvider;
    use PHPUnit\Framework\TestCase;

    class AppServiceProvider extends ServiceProvider
    {
        /**
         * Bootstrap any application services.
         */
        public function boot(): void
        {
            ParallelTesting::setUpProcess(function (int $token) {
                // ...
            });

            ParallelTesting::setUpTestCase(function (int $token, TestCase $testCase) {
                // ...
            });

            // Executed when a test database is created...
            ParallelTesting::setUpTestDatabase(function (string $database, int $token) {
                Artisan::call('db:seed');
            });

            ParallelTesting::tearDownTestCase(function (int $token, TestCase $testCase) {
                // ...
            });

            ParallelTesting::tearDownProcess(function (int $token) {
                // ...
            });
        }
    }

<a name="accessing-the-parallel-testing-token"></a>
#### 存取平行測試 Token

如果您想從應用程式測試程式碼中的任何其他位置存取當前的平行程序「token」，您可以使用 `token` 方法。這個 token 是一個唯一的字串識別碼，用於識別單一的測試程序，並可用於在平行測試程序之間劃分資源。例如，Laravel 會自動將此 token 附加到每個平行測試程序建立的測試資料庫名稱末尾：

    $token = ParallelTesting::token();

<a name="reporting-test-coverage"></a>
### 報告測試覆蓋率

> [!WARNING]  
> 此功能需要 [Xdebug](https://xdebug.org) 或 [PCOV](https://pecl.php.net/package/pcov)。

當執行您的應用程式測試時，您可能想確定您的測試案例是否真的覆蓋了應用程式程式碼，以及執行測試時使用了多少應用程式程式碼。為了實現這一點，您可以在呼叫 `test` 命令時提供 `--coverage` 選項：

```shell
php artisan test --coverage
```

<a name="enforcing-a-minimum-coverage-threshold"></a>
#### 強制最低覆蓋率門檻

您可以使用 `--min` 選項為您的應用程式定義一個最低測試覆蓋率門檻。如果未達到此門檻，測試套件將會失敗：

```shell
php artisan test --coverage --min=80.3
```

<a name="profiling-tests"></a>
### 測試分析

Artisan 測試執行器還包含一種方便的機制，用於列出您的應用程式中最慢的測試。呼叫帶有 `--profile` 選項的 `test` 命令，將會顯示您十個最慢的測試列表，讓您可以輕鬆地調查哪些測試可以改進以加速您的測試套件：

```shell
php artisan test --profile
```