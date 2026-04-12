# 測試：入門

- [簡介](#introduction)
- [環境](#environment)
- [建立測試](#creating-tests)
- [執行測試](#running-tests)
    - [平行執行測試](#running-tests-in-parallel)
    - [報告測試覆蓋率](#reporting-test-coverage)
    - [測試分析](#profiling-tests)
- [設定快取](#configuration-caching)

<a name="introduction"></a>
## 簡介

Laravel 在設計之初就將測試納入考量。事實上，原生就內建了對 [Pest](https://pestphp.com) 與 [PHPUnit](https://phpunit.de) 測試的支持，且 `phpunit.xml` 檔案已為您的應用程式設定就緒。此框架還提供了方便的輔助方法，讓您可以以具表達力的方式測試您的應用程式。

預設情況下，您的應用程式 `tests` 目錄包含兩個目錄：`Feature` 與 `Unit`。單元測試（Unit tests）是專注於程式碼中極小且獨立部分的測試。事實上，大多數的單元測試可能僅專注於單一方法。位於 "Unit" 測試目錄中的測試不會啟動您的 Laravel 應用程式，因此無法存取應用程式的資料庫或其他框架服務。

功能測試（Feature tests）可以測試較大範圍的程式碼，包括多個物件如何相互作用，甚至是對 JSON 端點發送完整的 HTTP 請求。**通常情況下，您的大部分測試應該是功能測試。這類測試能讓您最確信整個系統正按預期運作。**

`Feature` 與 `Unit` 測試目錄中都提供了一個 `ExampleTest.php` 檔案。安裝完新的 Laravel 應用程式後，請執行 `vendor/bin/pest`、`vendor/bin/phpunit` 或 `php artisan test` 指令來執行您的測試。


<a name="environment"></a>
## 環境

執行測試時，由於 `phpunit.xml` 檔案中定義了環境變數，Laravel 會自動將 [設定環境](/docs/{{version}}/configuration#environment-configuration) 設定為 `testing`。Laravel 也會自動將 session 與快取設定為 `array` 驅動程式，以便在測試期間不會持久化任何 session 或快取資料。

您可以根據需要自由定義其他測試環境設定值。`testing` 環境變數可以在應用程式的 `phpunit.xml` 檔案中設定，但請確保在執行測試之前，使用 `config:clear` Artisan 指令清除您的設定快取！


<a name="the-env-testing-environment-file"></a>
#### `.env.testing` 環境檔案

此外，您可以在專案根目錄建立 `.env.testing` 檔案。當執行 Pest 與 PHPUnit 測試，或使用 `--env=testing` 選項執行 Artisan 指令時，將使用此檔案取代 `.env` 檔案。


<a name="creating-tests"></a>
## 建立測試

要建立新的測試案例，請使用 `make:test` Artisan 指令。預設情況下，測試將被放置在 `tests/Feature` 目錄中：

```shell
php artisan make:test UserTest
```

如果您想在 `tests/Unit` 目錄中建立測試，可以在執行 `make:test` 指令時使用 `--unit` 選項：

```shell
php artisan make:test UserTest --unit
```

如果您有一個大部分依賴 Laravel 測試功能的測試類別，但某個特定的測試方法不需要啟動框架，您可以將 `#[UnitTest]` 屬性套用到該方法，以便僅針對該測試跳過啟動應用程式。

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use Illuminate\Foundation\Testing\Attributes\UnitTest;
use Tests\TestCase;

class LocationServiceTest extends TestCase
{
    public function test_get_coordinates_resolves_address(): void
    {
        // This test uses Laravel's testing features...
    }

    #[UnitTest]
    public function test_get_state_returns_state_from_abbreviation(): void
    {
        // This test runs without booting the application...
    }
}
```

> [!NOTE]
> 測試存根（stubs）可以使用 [存根發布](/docs/{{version}}/artisan#stub-customization) 進行自訂。

測試產生後，您可以使用 Pest 或 PHPUnit 像往常一樣定義測試。要執行測試，請從終端機執行 `vendor/bin/pest`、`vendor/bin/phpunit` 或 `php artisan test` 指令：

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
> 如果您在測試類別中定義了自己的 `setUp` / `tearDown` 方法，請務必呼叫父類別中對應的 `parent::setUp()` / `parent::tearDown()` 方法。通常，您應該在自己的 `setUp` 方法開始時呼叫 `parent::setUp()`，並在 `tearDown` 方法結束時呼叫 `parent::tearDown()`。

<a name="running-tests"></a>
## 執行測試

如前所述，一旦您寫好測試，可以使用 `pest` 或 `phpunit` 來執行它們：

```shell tab=Pest
./vendor/bin/pest
```

```shell tab=PHPUnit
./vendor/bin/phpunit
```

除了 `pest` 或 `phpunit` 命令外，您也可以使用 `test` Artisan 命令來執行測試。Artisan 測試執行器會提供詳細的測試報告，以方便開發與除錯：

```shell
php artisan test
```

任何可以傳遞給 `pest` 或 `phpunit` 命令的引數，也可以傳遞給 Artisan `test` 命令：

```shell
php artisan test --testsuite=Feature --stop-on-failure
```


<a name="running-tests-in-parallel"></a>
### 平行執行測試

預設情況下，Laravel 和 Pest / PHPUnit 會在單一行程中依序執行您的測試。然而，透過在多個行程中同時執行測試，您可以大幅縮短執行測試所需的時間。要開始使用，您應該安裝 `brianium/paratest` Composer 封裝作為「開發 (dev)」依賴項。接著，在執行 `test` Artisan 命令時加上 `--parallel` 選項：

```shell
composer require brianium/paratest --dev

php artisan test --parallel
```

預設情況下，Laravel 會根據您機器上可用的 CPU 核心數來建立對應數量的行程。不過，您可以使用 `--processes` 選項來調整行程數量：

```shell
php artisan test --parallel --processes=4
```

> [!WARNING]
> 在平行執行測試時，某些 Pest / PHPUnit 選項（例如 `--do-not-cache-result`）可能無法使用。


<a name="parallel-testing-and-databases"></a>
#### 平行測試與資料庫

只要您配置了主要資料庫連線，Laravel 就會自動為每個執行測試的平行行程建立並遷移測試資料庫。測試資料庫將會加上一個對每個行程唯一的行程令牌 (process token) 作為後綴。例如，如果您有兩個平行測試行程，Laravel 將會建立並使用 `your_db_test_1` 和 `your_db_test_2` 測試資料庫。

預設情況下，測試資料庫在多次呼叫 `test` Artisan 命令之間會持續存在，以便後續的 `test` 呼叫可以重複使用。不過，您可以使用 `--recreate-databases` 選項來重新建立它們：

```shell
php artisan test --parallel --recreate-databases
```


<a name="parallel-testing-hooks"></a>
#### 平行測試鉤子 (Hooks)

有時，您可能需要準備應用程式測試中使用的某些資源，以便它們能被多個測試行程安全地使用。

使用 `ParallelTesting` Facade，您可以指定在行程或測試案例的 `setUp` 和 `tearDown` 時要執行的程式碼。提供的閉包會接收 `$token` 和 `$testCase` 變數，分別包含行程令牌和目前的測試案例：

```php
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
```


<a name="accessing-the-parallel-testing-token"></a>
#### 存取平行測試令牌

如果您想從應用程式測試程式碼中的任何其他位置存取目前的平行行程「令牌 (token)」，可以使用 `token` 方法。此令牌是單一測試行程的唯一字串識別碼，可用於在平行測試行程之間區分資源。例如，Laravel 會自動將此令牌附加到每個平行測試行程建立的測試資料庫末尾：

    $token = ParallelTesting::token();


<a name="reporting-test-coverage"></a>
### 報告測試覆蓋率

> [!WARNING]
> 此功能需要 [Xdebug](https://xdebug.org) 或 [PCOV](https://pecl.php.net/package/pcov)。

在執行應用程式測試時，您可能想要確定您的測試案例是否確實覆蓋了應用程式程式碼，以及執行測試時使用了多少應用程式程式碼。為了實現這一目標，您可以在呼叫 `test` 命令時提供 `--coverage` 選項：

```shell
php artisan test --coverage
```


<a name="enforcing-a-minimum-coverage-threshold"></a>
#### 強制執行最低覆蓋率門檻

您可以使用 `--min` 選項為您的應用程式定義最低測試覆蓋率門檻。如果未達到此門檻，測試套件將會失敗：

```shell
php artisan test --coverage --min=80.3
```


<a name="profiling-tests"></a>
### 測試分析

Artisan 測試執行器還包含一個方便的機制，可用於列出應用程式中最慢的測試。使用 `--profile` 選項呼叫 `test` 命令，即可看到前十個最慢測試的列表，讓您可以輕鬆調查哪些測試可以被優化以加速測試套件：

```shell
php artisan test --profile
```


<a name="configuration-caching"></a>
## 設定快取

在執行測試時，Laravel 會為每個獨立的測試方法啟動應用程式。如果沒有快取的設定檔，應用程式中的每個設定檔在測試開始時都必須被載入。為了在單次執行中僅構建一次設定並在所有測試中重複使用，您可以使用 `Illuminate\Foundation\Testing\WithCachedConfig` trait：

```php tab=Pest
<?php

use Illuminate\Foundation\Testing\WithCachedConfig;

pest()->use(WithCachedConfig::class);

// ...
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use Illuminate\Foundation\Testing\WithCachedConfig;
use Tests\TestCase;

class ConfigTest extends TestCase
{
    use WithCachedConfig;

    // ...
}
```