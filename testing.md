# 測試：入門

- [介紹](#introduction)
- [環境](#environment)
- [建立測試](#creating-tests)
- [執行測試](#running-tests)
    - [平行執行測試](#running-tests-in-parallel)
    - [報告測試覆蓋率](#reporting-test-coverage)
    - [分析測試](#profiling-tests)
- [設定快取](#configuration-caching)

<a name="introduction"></a>
## 介紹

Laravel 的設計考量就包含測試。事實上，Laravel 內建了對 [Pest](https://pestphp.com) 與 [PHPUnit](https://phpunit.de) 測試的支援，並且 `phpunit.xml` 檔案已為你的應用程式設定完成。此外，框架還附帶了方便的輔助方法，讓你可以更具表達性地測試你的應用程式。

預設情況下，你的應用程式 `tests` 目錄包含兩個子目錄：`Feature` 與 `Unit`。單元測試 (Unit tests) 是一種專注於程式碼中非常小且獨立部分的測試。事實上，大多數單元測試可能都只針對單一方法。「Unit」測試目錄中的測試不會啟動你的 Laravel 應用程式，因此無法存取你的應用程式資料庫或其他框架服務。

功能測試 (Feature tests) 可以測試你的程式碼中較大部分的內容，包括多個物件如何互動，甚至對 JSON 端點的完整 HTTP 請求。**通常來說，你的大部分測試都應該是功能測試。這類型的測試能提供最大的信心，確保你的系統整體功能如預期運作。**

在 `Feature` 與 `Unit` 測試目錄中都提供了一個 `ExampleTest.php` 檔案。安裝新的 Laravel 應用程式後，執行 `vendor/bin/pest`、`vendor/bin/phpunit` 或 `php artisan test` 指令來執行你的測試。

<a name="environment"></a>
## 環境

執行測試時，由於 `phpunit.xml` 檔案中定義了環境變數，Laravel 會自動將[設定環境](/docs/{{version}}/configuration#environment-configuration)設定為 `testing`。Laravel 也會自動將 Session 和快取設定為 `array` 驅動器，這樣在測試時就不會保留任何 Session 或快取資料。

你可以根據需要自由定義其他測試環境設定值。這些 `testing` 環境變數可以在你的應用程式 `phpunit.xml` 檔案中設定，但在執行測試之前，請務必使用 `config:clear` Artisan 指令清除你的設定快取！

<a name="the-env-testing-environment-file"></a>
#### `.env.testing` 環境檔案

此外，你可以在專案的根目錄建立一個 `.env.testing` 檔案。當執行 Pest 和 PHPUnit 測試或使用 `--env=testing` 選項執行 Artisan 指令時，此檔案將取代 `.env` 檔案被使用。

<a name="creating-tests"></a>
## 建立測試

要建立新的測試案例，請使用 `make:test` Artisan 指令。預設情況下，測試會放在 `tests/Feature` 目錄中：

```shell
php artisan make:test UserTest
```

如果你想要在 `tests/Unit` 目錄中建立測試，可以在執行 `make:test` 指令時使用 `--unit` 選項：

```shell
php artisan make:test UserTest --unit
```

> [!NOTE]
> 測試 Stub 可透過 [Stub 發布](/docs/{{version}}/artisan#stub-customization)進行客製化。

測試生成後，你可以像往常一樣使用 Pest 或 PHPUnit 來定義測試。要執行你的測試，請在終端機中執行 `vendor/bin/pest`、`vendor/bin/phpunit` 或 `php artisan test` 指令：

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
> 如果你在測試類別中定義了自己的 `setUp` / `tearDown` 方法，請務必呼叫父類別中相對應的 `parent::setUp()` / `parent::tearDown()` 方法。通常，你應該在自己的 `setUp` 方法開始時呼叫 `parent::setUp()`，並在自己的 `tearDown` 方法結束時呼叫 `parent::tearDown()`。

<a name="running-tests"></a>
## 執行測試

如前所述，一旦您撰寫了測試，就可以使用 `pest` 或 `phpunit` 執行它們：

```shell tab=Pest
./vendor/bin/pest
```

```shell tab=PHPUnit
./vendor/bin/phpunit
```

除了 `pest` 或 `phpunit` 命令之外，您還可以使用 `test` Artisan 命令來執行測試。Artisan 測試執行器提供詳細的測試報告，以方便開發和偵錯：

```shell
php artisan test
```

任何可以傳遞給 `pest` 或 `phpunit` 命令的參數，也可以傳遞給 Artisan `test` 命令：

```shell
php artisan test --testsuite=Feature --stop-on-failure
```

<a name="running-tests-in-parallel"></a>
### 平行執行測試

預設情況下，Laravel 和 Pest / PHPUnit 會在單一程序中依序執行您的測試。然而，您可以透過在多個程序中同時執行測試，來大幅減少執行測試所需的時間。若要開始，您應該安裝 `brianium/paratest` Composer 套件作為「dev」依賴。然後，在執行 `test` Artisan 命令時，加入 `--parallel` 選項：

```shell
composer require brianium/paratest --dev

php artisan test --parallel
```

預設情況下，Laravel 會根據您機器上可用的 CPU 核心數量來建立相同數量的程序。然而，您可以使用 `--processes` 選項調整程序數量：

```shell
php artisan test --parallel --processes=4
```

> [!WARNING]
> 在平行執行測試時，某些 Pest / PHPUnit 選項 (例如 `--do-not-cache-result`) 可能無法使用。

<a name="parallel-testing-and-databases"></a>
#### 平行測試與資料庫

只要您配置了主要的資料庫連線，Laravel 就會自動為每個執行測試的平行程序建立並遷移一個測試資料庫。測試資料庫會附加一個程序 Token，該 Token 在每個程序中都是唯一的。例如，如果您有兩個平行測試程序，Laravel 將建立並使用 `your_db_test_1` 和 `your_db_test_2` 測試資料庫。

預設情況下，測試資料庫會在每次呼叫 `test` Artisan 命令後持續存在，以便後續的 `test` 呼叫可以再次使用它們。然而，您可以使用 `--recreate-databases` 選項重新建立它們：

```shell
php artisan test --parallel --recreate-databases
```

<a name="parallel-testing-hooks"></a>
#### 平行測試掛鉤

有時，您可能需要準備應用程式測試所使用的某些資源，以便它們可以安全地被多個測試程序使用。

使用 `ParallelTesting` facade，您可以指定在程序或測試案例的 `setUp` 和 `tearDown` 期間執行的程式碼。給定的閉包會分別接收包含程序 Token 和當前測試案例的 `$token` 和 `$testCase` 變數：

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

如果您想在應用程式測試程式碼中的任何其他位置存取當前的平行程序「Token」，可以使用 `token` 方法。此 Token 是用於單個測試程序的唯一字串識別碼，可用於在平行測試程序之間分割資源。例如，Laravel 會自動將此 Token 附加到每個平行測試程序所建立的測試資料庫末尾：

    $token = ParallelTesting::token();

<a name="reporting-test-coverage"></a>
### 報告測試覆蓋率

> [!WARNING]
> 此功能需要 [Xdebug](https://xdebug.org) 或 [PCOV](https://pecl.php.net/package/pcov)。

當執行您的應用程式測試時，您可能想確定您的測試案例是否實際覆蓋了應用程式程式碼，以及執行測試時使用了多少應用程式程式碼。為此，您可以在呼叫 `test` 命令時提供 `--coverage` 選項：

```shell
php artisan test --coverage
```

<a name="enforcing-a-minimum-coverage-threshold"></a>
#### 強制最低覆蓋率閾值

您可以使用 `--min` 選項來定義應用程式的最低測試覆蓋率閾值。如果未達到此閾值，測試套件將會失敗：

```shell
php artisan test --coverage --min=80.3
```

<a name="profiling-tests"></a>
### 分析測試

Artisan 測試執行器還包含一個方便的機制，用於列出應用程式中速度最慢的測試。呼叫 `test` 命令並附帶 `--profile` 選項，即可獲得您十個最慢測試的列表，讓您可以輕鬆調查哪些測試可以改進以加速您的測試套件：

```shell
php artisan test --profile
```

<a name="configuration-caching"></a>
## 設定快取

執行測試時，Laravel 會為每個單獨的測試方法啟動應用程式。如果沒有快取設定檔，應用程式中的每個設定檔都必須在測試開始時載入。為了只建構一次設定並在單次執行中的所有測試中重複使用它，您可以使用 `Illuminate\Foundation\Testing\WithCachedConfig` trait：

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