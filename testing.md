# 測試：入門指南

- [介紹](#introduction)
- [環境](#environment)
- [建立測試](#creating-tests)
- [執行測試](#running-tests)
    - [平行執行測試](#running-tests-in-parallel)
    - [報告測試覆蓋率](#reporting-test-coverage)
    - [測試效能分析](#profiling-tests)

<a name="introduction"></a>
## 介紹

Laravel 的設計考量就包含了測試功能。事實上，Laravel 開箱即用就支援 [Pest](https://pestphp.com) 和 [PHPUnit](https://phpunit.de) 測試，且 `phpunit.xml` 檔案已為您的應用程式設定好。此框架也提供了便利的輔助方法，讓您能夠更清晰地測試您的應用程式。

預設情況下，您的應用程式的 `tests` 目錄包含兩個子目錄：`Feature` 和 `Unit`。Unit tests (單元測試) 是一種著重於您程式碼中非常小、獨立部分的測試。事實上，大多數的單元測試可能都專注於一個單一方法。位於「Unit」測試目錄中的測試不會啟動您的 Laravel 應用程式，因此無法存取您的應用程式資料庫或其他框架服務。

Feature tests (功能測試) 可以測試您程式碼中較大的部分，包括多個物件如何互相作用，甚至是一個完整的 HTTP 請求到 JSON 端點。**一般來說，您的大多數測試都應該是功能測試。這類型的測試能讓您對整個系統如預期般運作抱有最大的信心。**

`Feature` 和 `Unit` 兩個測試目錄中都提供了一個 `ExampleTest.php` 檔案。安裝新的 Laravel 應用程式後，執行 `vendor/bin/pest`、`vendor/bin/phpunit` 或 `php artisan test` 指令來執行您的測試。

<a name="environment"></a>
## 環境

執行測試時，由於 `phpunit.xml` 檔案中定義的環境變數，Laravel 會自動將[設定環境](/docs/{{version}}/configuration#environment-configuration)設定為 `testing`。Laravel 也會自動將 session 和 cache 設定為 `array` 驅動器，這樣在測試時就不會保留任何 session 或 cache 資料。

您可以根據需要自由定義其他的測試環境設定值。`testing` 環境變數可以在您的應用程式的 `phpunit.xml` 檔案中進行設定，但在執行測試之前，請務必使用 `config:clear` Artisan 指令清除您的設定快取！

<a name="the-env-testing-environment-file"></a>
#### `.env.testing` 環境檔案

此外，您可以在專案的根目錄中建立一個 `.env.testing` 檔案。當執行 Pest 和 PHPUnit 測試，或執行帶有 `--env=testing` 選項的 Artisan 指令時，此檔案將會取代 `.env` 檔案被使用。

<a name="creating-tests"></a>
## 建立測試

要建立新的測試案例，請使用 `make:test` Artisan 指令。預設情況下，測試會被放置在 `tests/Feature` 目錄中：

```shell
php artisan make:test UserTest
```

如果您想在 `tests/Unit` 目錄中建立測試，可以在執行 `make:test` 指令時使用 `--unit` 選項：

```shell
php artisan make:test UserTest --unit
```

> [!NOTE]
> 測試骨架 (stub) 可以透過[發佈骨架](/docs/{{version}}/artisan#stub-customization)進行客製化。

產生測試後，您可以像往常一樣使用 Pest 或 PHPUnit 來定義測試。要執行您的測試，請從終端機執行 `vendor/bin/pest`、`vendor/bin/phpunit` 或 `php artisan test` 指令：

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
> 如果您在測試類別中定義了自己的 `setUp` / `tearDown` 方法，請務必呼叫父類別中相對應的 `parent::setUp()` / `parent::tearDown()` 方法。通常，您應該在自己的 `setUp` 方法開始時呼叫 `parent::setUp()`，並在 `tearDown` 方法結束時呼叫 `parent::tearDown()`。

<a name="running-tests"></a>
## 執行測試

如前所述，一旦您編寫完測試，便可以使用 `pest` 或 `phpunit` 來執行它們：

```shell tab=Pest
./vendor/bin/pest
```

```shell tab=PHPUnit
./vendor/bin/phpunit
```

除了 `pest` 或 `phpunit` 指令之外，您也可以使用 `test` Artisan 指令來執行測試。Artisan 測試執行器提供詳細的測試報告，以方便開發與除錯：

```shell
php artisan test
```

任何可以傳遞給 `pest` 或 `phpunit` 指令的參數，也都可以傳遞給 Artisan `test` 指令：

```shell
php artisan test --testsuite=Feature --stop-on-failure
```

<a name="running-tests-in-parallel"></a>
### 平行執行測試

預設情況下，Laravel 和 Pest / PHPUnit 會在單一程序中依序執行您的測試。然而，您可以透過在多個程序中同時執行測試來大幅減少測試所需的時間。首先，您應該安裝 `brianium/paratest` Composer 套件作為「dev」依賴。然後，在執行 `test` Artisan 指令時包含 `--parallel` 選項：

```shell
composer require brianium/paratest --dev

php artisan test --parallel
```

預設情況下，Laravel 會建立與您機器上可用的 CPU 核心數量相同的程序。不過，您可以使用 `--processes` 選項調整程序數量：

```shell
php artisan test --parallel --processes=4
```

> [!WARNING]
> 當平行執行測試時，某些 Pest / PHPUnit 選項（例如 `--do-not-cache-result`）可能無法使用。

<a name="parallel-testing-and-databases"></a>
#### 平行測試與資料庫

只要您配置了主要資料庫連線，Laravel 就會自動處理為每個執行測試的平行程序建立和遷移測試資料庫。測試資料庫會以每個程序獨有的程序 Token 為後綴。例如，如果您有兩個平行測試程序，Laravel 將會建立並使用 `your_db_test_1` 和 `your_db_test_2` 測試資料庫。

預設情況下，測試資料庫會在 `test` Artisan 指令的呼叫之間保留，以便後續的 `test` 呼叫可以再次使用它們。不過，您可以使用 `--recreate-databases` 選項重新建立它們：

```shell
php artisan test --parallel --recreate-databases
```

<a name="parallel-testing-hooks"></a>
#### 平行測試掛鉤

有時，您可能需要準備應用程式測試所使用的某些資源，以便多個測試程序可以安全地使用它們。

使用 `ParallelTesting` Facade，您可以指定要在程序或測試案例的 `setUp` 和 `tearDown` 時執行的程式碼。給定的閉包會收到包含程序 Token 和當前測試案例的 `$token` 和 `$testCase` 變數，分別為：

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
#### 存取平行測試 Token

如果您想從應用程式測試程式碼中的任何其他位置存取當前平行程序的「Token」，您可以使用 `token` 方法。此 Token 是單一測試程序的唯一字串識別符，可用於在平行測試程序之間劃分資源。例如，Laravel 會自動將此 Token 附加到每個平行測試程序建立的測試資料庫名稱末尾：

    $token = ParallelTesting::token();

<a name="reporting-test-coverage"></a>
### 報告測試覆蓋率

> [!WARNING]
> 此功能需要 [Xdebug](https://xdebug.org) 或 [PCOV](https://pecl.php.net/package/pcov)。

當執行應用程式測試時，您可能想確定測試案例是否實際涵蓋了應用程式程式碼，以及在執行測試時使用了多少應用程式程式碼。為此，您可以在呼叫 `test` 指令時提供 `--coverage` 選項：

```shell
php artisan test --coverage
```

<a name="enforcing-a-minimum-coverage-threshold"></a>
#### 強制最低覆蓋率閾值

您可以使用 `--min` 選項為您的應用程式定義一個最低測試覆蓋率閾值。如果未達到此閾值，測試套件將會失敗：

```shell
php artisan test --coverage --min=80.3
```

<a name="profiling-tests"></a>
### 測試效能分析

Artisan 測試執行器還包含一個方便的機制，用於列出應用程式中最慢的測試。使用 `--profile` 選項呼叫 `test` 指令，將會顯示您十個最慢的測試列表，讓您輕鬆調查哪些測試可以改進以加速您的測試套件：

```shell
php artisan test --profile
```