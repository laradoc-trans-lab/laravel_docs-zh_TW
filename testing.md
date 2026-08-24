# 測試：入門指南

- [介紹](#introduction)
- [環境](#environment)
- [建立測試](#creating-tests)
- [執行測試](#running-tests)
    - [平行執行測試](#running-tests-in-parallel)
    - [報告測試覆蓋率](#reporting-test-coverage)
    - [分析測試效能](#profiling-tests)
- [設定快取](#configuration-caching)

<a name="introduction"></a>
## 介紹

Laravel 在設計時就已經將測試考量在內。事實上，內建就支援使用 [Pest](https://pestphp.com) 與 [PHPUnit](https://phpunit.de) 進行測試，並且已經為您的應用程式設定好了 `phpunit.xml` 檔案。框架還附帶了便利的輔助方法，讓您能以極具表達力的方式測試應用程式。

預設情況下，應用程式的 `tests` 目錄包含兩個目錄：`Feature` 和 `Unit`。單元測試 (Unit tests) 專注於程式碼中非常小且孤立的一部分。事實上，大多數單元測試可能只專注於單一方法。在 "Unit" 測試目錄內的測試不會啟動您的 Laravel 應用程式，因此無法存取應用程式的資料庫或其他框架服務。

功能測試 (Feature tests) 可以測試更大範圍的程式碼，包含多個物件之間如何相互作用，甚至是對 JSON 端點的完整 HTTP 請求。**通常，您的大多數測試應該都是功能測試。這類測試能為您的整個系統是否如預期運作提供最大的信心。**

`Feature` 和 `Unit` 測試目錄中皆提供了 `ExampleTest.php` 檔案。安裝新的 Laravel 應用程式後，執行 `vendor/bin/pest`、`vendor/bin/phpunit` 或 `php artisan test` 命令即可執行您的測試。


<a name="environment"></a>
## 環境

執行測試時，由於 `phpunit.xml` 檔案中定義的環境變數，Laravel 會自動將 [設定環境](/docs/{{version}}/configuration#environment-configuration) 設定為 `testing`。Laravel 還會自動將 Session 和快取設定為 `array` 驅動器，如此一來在測試時就不會持久化任何 Session 或快取資料。

如有需要，您可以自由定義其他測試環境設定值。`testing` 環境變數可以在應用程式的 `phpunit.xml` 檔案中進行設定，但在執行測試前，請務必使用 `config:clear` Artisan 命令清除設定快取！


<a name="the-env-testing-environment-file"></a>
#### `.env.testing` 環境檔案

此外，您可以在專案根目錄中建立 `.env.testing` 檔案。當執行 Pest 和 PHPUnit 測試或使用 `--env=testing` 選項執行 Artisan 命令時，將會優先使用此檔案來取代 `.env` 檔案。


<a name="creating-tests"></a>
## 建立測試

若要建立新的測試案例，請使用 `make:test` Artisan 命令。預設情況下，測試會放在 `tests/Feature` 目錄中：

```shell
php artisan make:test UserTest
```

如果您想在 `tests/Unit` 目錄中建立測試，可以在執行 `make:test` 命令時使用 `--unit` 選項：

```shell
php artisan make:test UserTest --unit
```

如果您有一個測試類別主要依賴 Laravel 的測試功能，但某個特定的測試方法不需要啟動框架，您可以對該方法套用 `#[UnitTest]` 屬性，以僅為該測試跳過啟動應用程式。

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
> 測試 Stub 可以透過 [Stub 發布](/docs/{{version}}/artisan#stub-customization) 進行客製化。

測試產生後，您可以像平常使用 Pest 或 PHPUnit 一樣來定義測試。若要執行您的測試，請在終端機中執行 `vendor/bin/pest`、`vendor/bin/phpunit` 或 `php artisan test` 命令：

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
> 若您在測試類別中自訂了 `setUp` / `tearDown` 方法，請務必呼叫父類別對應的 `parent::setUp()` / `parent::tearDown()` 方法。通常，您應該在自己的 `setUp` 方法開頭呼叫 `parent::setUp()`，並在 `tearDown` 方法的結尾呼叫 `parent::tearDown()`。

<a name="running-tests"></a>
## 執行測試

如前所述，撰寫好測試後，您可以使用 `pest` 或 `phpunit` 來執行它們：

```shell tab=Pest
./vendor/bin/pest
```

```shell tab=PHPUnit
./vendor/bin/phpunit
```

除了 `pest` 或 `phpunit` 指令外，您也可以使用 `test` Artisan 指令來執行測試。Artisan 測試執行器提供了詳細的測試報告，以利開發與除錯：

```shell
php artisan test
```

任何可以傳遞給 `pest` 或 `phpunit` 指令的引數，也都可以傳遞給 Artisan 的 `test` 指令：

```shell
php artisan test --testsuite=Feature --stop-on-failure
```


<a name="running-tests-in-parallel"></a>
### 平行執行測試

預設情況下，Laravel 與 Pest / PHPUnit 會在單一行程中依序執行您的測試。然而，透過跨多個行程同時執行測試，您可以大幅減少執行測試所需的時間。若要開始使用，您應該安裝 `brianium/paratest` Composer 套件作為「dev」依賴項目。接著，在執行 `test` Artisan 指令時加上 `--parallel` 選項：

```shell
composer require brianium/paratest --dev

php artisan test --parallel
```

預設情況下，Laravel 會根據您電腦上可用的 CPU 核心數量建立相同數量的行程。不過，您可以使用 `--processes` 選項來調整行程數量：

```shell
php artisan test --parallel --processes=4
```

> [!WARNING]
> 當平行執行測試時，某些 Pest / PHPUnit 選項（例如 `--do-not-cache-result`）可能無法使用。


<a name="parallel-testing-and-databases"></a>
#### 平行測試與資料庫

只要您設定了主要資料庫連線，Laravel 就會自動為每一個執行測試的平行行程建立並遷移測試資料庫。測試資料庫的名稱後方會加上每個行程獨有的行程令牌 (Process Token)。例如，如果您有兩個平行測試行程，Laravel 將會建立並使用 `your_db_test_1` 與 `your_db_test_2` 測試資料庫。

預設情況下，測試資料庫會在多次呼叫 `test` Artisan 指令之間保留，以便後續執行 `test` 時可以重複使用。不過，您可以使用 `--recreate-databases` 選項來重新建立它們：

```shell
php artisan test --parallel --recreate-databases
```


<a name="parallel-testing-hooks"></a>
#### 平行測試 Hook

有時，您可能需要準備應用程式測試所需的特定資源，以便多個測試行程能夠安全地使用它們。

使用 `ParallelTesting` Facade，您可以指定要在行程或測試用例 (Test Case) 的 `setUp` 與 `tearDown` 時執行的程式碼。傳入的閉包 (Closure) 會分別接收包含行程令牌與當前測試用例的 `$token` 與 `$testCase` 變數：

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

如果您想在應用程式測試程式碼的任何其他位置存取當前平行行程的「token」，可以使用 `token` 方法。此令牌是一個獨特的字串識別碼，用於代表單一測試行程，可用於跨平行測試行程切割資源。例如，Laravel 會自動將此令牌附加到每個平行測試行程所建立的測試資料庫名稱末尾：

    $token = ParallelTesting::token();


<a name="reporting-test-coverage"></a>
### 報告測試覆蓋率

> [!WARNING]
> 此功能需要 [Xdebug](https://xdebug.org) 或 [PCOV](https://pecl.php.net/package/pcov)。

在執行應用程式測試時，您可能想要確認測試用例是否確實覆蓋了應用程式的程式碼，以及執行測試時使用了多少應用程式程式碼。為此，您可以在呼叫 `test` 指令時提供 `--coverage` 選項：

```shell
php artisan test --coverage
```


<a name="enforcing-a-minimum-coverage-threshold"></a>
#### 強制實施最低覆蓋率門檻

您可以使用 `--min` 選項來為應用程式定義最低測試覆蓋率門檻。如果未達此門檻，測試套件將會判定為失敗：

```shell
php artisan test --coverage --min=80.3
```


<a name="profiling-tests"></a>
### 分析測試效能

Artisan 測試執行器還包含了一個方便的機制，用於列出應用程式中最慢的測試。使用 `--profile` 選項呼叫 `test` 指令，將會顯示前十個最慢測試的清單，讓您能夠輕鬆調查哪些測試可以進行改善以加快測試套件的執行速度：

```shell
php artisan test --profile
```


<a name="configuration-caching"></a>
## 設定快取

執行測試時，Laravel 會為每個獨立的測試方法啟動應用程式。在沒有快取設定檔的情況下，應用程式中的每個設定檔都必須在測試開始時載入。若要建置一次設定並在單次執行中的所有測試重複使用，您可以使用 `Illuminate\Foundation\Testing\WithCachedConfig` Trait：

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