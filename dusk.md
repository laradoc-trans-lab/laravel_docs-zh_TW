# Laravel Dusk

- [介紹](#introduction)
- [安裝](#installation)
    - [管理 ChromeDriver 安裝](#managing-chromedriver-installations)
    - [使用其他瀏覽器](#using-other-browsers)
- [開始使用](#getting-started)
    - [建立測試](#generating-tests)
    - [在每次測試後重設資料庫](#resetting-the-database-after-each-test)
    - [執行測試](#running-tests)
    - [環境處理](#environment-handling)
- [瀏覽器基礎操作](#browser-basics)
    - [建立瀏覽器](#creating-browsers)
    - [頁面導覽](#navigation)
    - [調整瀏覽器視窗大小](#resizing-browser-windows)
    - [瀏覽器巨集](#browser-macros)
    - [認證](#authentication)
    - [Cookies](#cookies)
    - [執行 JavaScript](#executing-javascript)
    - [擷取螢幕截圖](#taking-a-screenshot)
    - [儲存主控台輸出至磁碟](#storing-console-output-to-disk)
    - [儲存網頁原始碼至磁碟](#storing-page-source-to-disk)
- [與元素互動](#interacting-with-elements)
    - [Dusk 選擇器](#dusk-selectors)
    - [文字、值與屬性](#text-values-and-attributes)
    - [與表單互動](#interacting-with-forms)
    - [附加檔案](#attaching-files)
    - [按下按鈕](#pressing-buttons)
    - [點擊連結](#clicking-links)
    - [使用鍵盤](#using-the-keyboard)
    - [使用滑鼠](#using-the-mouse)
    - [JavaScript 對話框](#javascript-dialogs)
    - [與內嵌框架互動](#interacting-with-iframes)
    - [限定選擇器範圍](#scoping-selectors)
    - [等待元素](#waiting-for-elements)
    - [滾動元素至可視範圍](#scrolling-an-element-into-view)
- [可用的斷言](#available-assertions)
- [頁面](#pages)
    - [建立頁面](#generating-pages)
    - [設定頁面](#configuring-pages)
    - [導覽至頁面](#navigating-to-pages)
    - [簡寫選擇器](#shorthand-selectors)
    - [頁面方法](#page-methods)
- [元件](#components)
    - [建立元件](#generating-components)
    - [使用元件](#using-components)
- [持續整合](#continuous-integration)
    - [Heroku CI](#running-tests-on-heroku-ci)
    - [Travis CI](#running-tests-on-travis-ci)
    - [GitHub Actions](#running-tests-on-github-actions)
    - [Chipper CI](#running-tests-on-chipper-ci)

<a name="introduction"></a>
## 介紹

> [!WARNING]
> [Pest 4](https://pestphp.com/) 現已包含自動化瀏覽器測試，與 Laravel Dusk 相比提供了顯著的效能與易用性改善。對於新專案，我們建議使用 Pest 進行瀏覽器測試。

[Laravel Dusk](https://github.com/laravel/dusk) 提供了表達力高且易於使用的瀏覽器自動化與測試 API。預設情況下，Dusk 不需要你在本機電腦上安裝 JDK 或 Selenium。相反地，Dusk 使用獨立的 [ChromeDriver](https://sites.google.com/chromium.org/driver) 安裝。不過，你也可以自由使用任何其他相容於 Selenium 的驅動程式。


<a name="installation"></a>
## 安裝

若要開始使用，你應該安裝 [Google Chrome](https://www.google.com/chrome)，並將 `laravel/dusk` Composer 依賴套件新增至你的專案中：

```shell
composer require laravel/dusk --dev
```

> [!WARNING]
> 如果你手動註冊 Dusk 的服務提供者(Service Providers)，你**絕對不要**將其註冊在正式環境中，因為這樣可能會導致任意使用者都能通過你的應用程式認證。

安裝 Dusk 套件後，請執行 `dusk:install` Artisan 指令。`dusk:install` 指令將會建立 `tests/Browser` 目錄、一個範例 Dusk 測試，並為你的作業系統安裝 Chrome Driver 二進位執行檔：

```shell
php artisan dusk:install
```

接著，在應用程式的 `.env` 檔案中設定 `APP_URL` 環境變數。此值應與你在瀏覽器中存取應用程式時所使用的 URL 一致。

> [!NOTE]
> 如果你使用 [Laravel Sail](/docs/{{version}}/sail) 來管理本機開發環境，也請參閱 Sail 關於[設定與執行 Dusk 測試](/docs/{{version}}/sail#laravel-dusk)的文件。


<a name="managing-chromedriver-installations"></a>
### 管理 ChromeDriver 安裝

如果你想要安裝與 Laravel Dusk 透過 `dusk:install` 指令所安裝的不同版本之 ChromeDriver，你可以使用 `dusk:chrome-driver` 指令：

```shell
# Install the latest version of ChromeDriver for your OS...
php artisan dusk:chrome-driver

# Install a given version of ChromeDriver for your OS...
php artisan dusk:chrome-driver 86

# Install a given version of ChromeDriver for all supported OSs...
php artisan dusk:chrome-driver --all

# Install the version of ChromeDriver that matches the detected version of Chrome / Chromium for your OS...
php artisan dusk:chrome-driver --detect
```

> [!WARNING]
> Dusk 需要 `chromedriver` 二進位執行檔具備可執行權限。如果你在執行 Dusk 時遇到問題，應使用以下指令確保二進位執行檔具備執行權限：`chmod -R 0755 vendor/laravel/dusk/bin/`。


<a name="using-other-browsers"></a>
### 使用其他瀏覽器

預設情況下，Dusk 使用 Google Chrome 以及獨立的 [ChromeDriver](https://sites.google.com/chromium.org/driver) 安裝來執行瀏覽器測試。不過，你也可以啟動自己的 Selenium 伺服器，並針對任何你想要的瀏覽器執行測試。

首先，開啟你的 `tests/DuskTestCase.php` 檔案，這是你應用程式的基礎 Dusk 測試案例。在此檔案中，你可以移除對 `startChromeDriver` 方法的呼叫。這將停止 Dusk 自動啟動 ChromeDriver：

```php
/**
 * Prepare for Dusk test execution.
 *
 * @beforeClass
 */
public static function prepare(): void
{
    // static::startChromeDriver();
}
```

接著，你可以修改 `driver` 方法以連線至你選擇的 URL 與連接埠。此外，你也可以修改應傳遞給 WebDriver 的「desired capabilities」：

```php
use Facebook\WebDriver\Remote\RemoteWebDriver;

/**
 * Create the RemoteWebDriver instance.
 */
protected function driver(): RemoteWebDriver
{
    return RemoteWebDriver::create(
        'http://localhost:4444/wd/hub', DesiredCapabilities::phantomjs()
    );
}
```

<a name="getting-started"></a>
## 開始使用


<a name="generating-tests"></a>
### 建立測試

若要建立 Dusk 測試，請使用 `dusk:make` Artisan 指令。產生的測試將會放置在 `tests/Browser` 目錄中：

```shell
php artisan dusk:make LoginTest
```


<a name="resetting-the-database-after-each-test"></a>
### 在每次測試後重設資料庫

您撰寫的大多數測試都會與從應用程式資料庫檢索資料的頁面進行互動；然而，您的 Dusk 測試絕不應該使用 `RefreshDatabase` trait。`RefreshDatabase` trait 利用了資料庫交易機制，但這在跨 HTTP 請求時無法套用或使用。相反地，您有兩種選擇：`DatabaseMigrations` trait 和 `DatabaseTruncation` trait。


<a name="reset-migrations"></a>
#### 使用資料庫遷移

`DatabaseMigrations` trait 會在每次測試前執行您的資料庫遷移。然而，在每次測試中刪除並重新建立資料庫資料表通常比清空資料表慢：

```php tab=Pest
<?php

use Illuminate\Foundation\Testing\DatabaseMigrations;

pest()->use(DatabaseMigrations::class);

//
```

```php tab=PHPUnit
<?php

namespace Tests\Browser;

use Illuminate\Foundation\Testing\DatabaseMigrations;
use Laravel\Dusk\Browser;
use Tests\DuskTestCase;

class ExampleTest extends DuskTestCase
{
    use DatabaseMigrations;

    //
}
```

> [!WARNING]
> 執行 Dusk 測試時無法使用 SQLite 記憶體資料庫。由於瀏覽器是在自己的行程中執行，因此無法存取其他行程的記憶體資料庫。


<a name="reset-truncation"></a>
#### 使用資料庫清空

`DatabaseTruncation` trait 會在第一個測試時遷移您的資料庫，以確保您的資料庫資料表已正確建立。然而，在後續的測試中，資料庫的資料表只會被清空（truncate）——這比重新執行所有資料庫遷移大幅提升了速度：

```php tab=Pest
<?php

use Illuminate\Foundation\Testing\DatabaseTruncation;

pest()->use(DatabaseTruncation::class);

//
```

```php tab=PHPUnit
<?php

namespace Tests\Browser;

use App\Models\User;
use Illuminate\Foundation\Testing\DatabaseTruncation;
use Laravel\Dusk\Browser;
use Tests\DuskTestCase;

class ExampleTest extends DuskTestCase
{
    use DatabaseTruncation;

    //
}
```

預設情況下，此 trait 會清空除了 `migrations` 資料表之外的所有資料表。如果您想自訂應清空的資料表，可以在測試類別中定義 `$tablesToTruncate` 屬性：

> [!NOTE]
> 如果您使用的是 Pest，應在基礎 `DuskTestCase` 類別或測試檔案所繼承的任何類別上定義屬性或方法。

```php
/**
 * Indicates which tables should be truncated.
 *
 * @var array
 */
protected $tablesToTruncate = ['users'];
```

或者，您可以在測試類別上定義 `$exceptTables` 屬性，以指定哪些資料表應從清空中排除：

```php
/**
 * Indicates which tables should be excluded from truncation.
 *
 * @var array
 */
protected $exceptTables = ['users'];
```

若要指定應清空資料表的資料庫連線，您可以在測試類別上定義 `$connectionsToTruncate` 屬性：

```php
/**
 * Indicates which connections should have their tables truncated.
 *
 * @var array
 */
protected $connectionsToTruncate = ['mysql'];
```

如果您想在執行資料庫清空之前或之後執行程式碼，可以在測試類別上定義 `beforeTruncatingDatabase` 或 `afterTruncatingDatabase` 方法：

```php
/**
 * Perform any work that should take place before the database has started truncating.
 */
protected function beforeTruncatingDatabase(): void
{
    //
}

/**
 * Perform any work that should take place after the database has finished truncating.
 */
protected function afterTruncatingDatabase(): void
{
    //
}
```


<a name="running-tests"></a>
### 執行測試

若要執行瀏覽器測試，請執行 `dusk` Artisan 指令：

```shell
php artisan dusk
```

如果您上次執行 `dusk` 指令時有測試失敗，可以使用 `dusk:fails` 指令先重新執行失敗的測試來節省時間：

```shell
php artisan dusk:fails
```

`dusk` 指令接受 Pest / PHPUnit 測試執行器通常接受的任何引數，例如允許您僅執行特定[群組（group）](https://docs.phpunit.de/en/10.5/annotations.html#group)的測試：

```shell
php artisan dusk --group=foo
```

> [!NOTE]
> 如果您使用 [Laravel Sail](/docs/{{version}}/sail) 來管理本機開發環境，請參閱 Sail 說明文件中關於[設定與執行 Dusk 測試](/docs/{{version}}/sail#laravel-dusk)的章節。


<a name="manually-starting-chromedriver"></a>
#### 手動啟動 ChromeDriver

預設情況下，Dusk 會自動嘗試啟動 ChromeDriver。如果這在您的特定系統上無法運作，您可以在執行 `dusk` 指令之前手動啟動 ChromeDriver。如果您選擇手動啟動 ChromeDriver，應該註解掉 `tests/DuskTestCase.php` 檔案中的以下這行程式碼：

```php
/**
 * Prepare for Dusk test execution.
 *
 * @beforeClass
 */
public static function prepare(): void
{
    // static::startChromeDriver();
}
```

此外，如果您在 9515 以外的連接埠啟動 ChromeDriver，則應修改同一個類別的 `driver` 方法以反映正確的連接埠：

```php
use Facebook\WebDriver\Remote\RemoteWebDriver;

/**
 * Create the RemoteWebDriver instance.
 */
protected function driver(): RemoteWebDriver
{
    return RemoteWebDriver::create(
        'http://localhost:9515', DesiredCapabilities::chrome()
    );
}
```


<a name="environment-handling"></a>
### 環境處理

若要強制 Dusk 在執行測試時使用自訂的環境設定檔，請在專案根目錄中建立 `.env.dusk.{environment}` 檔案。例如，如果您將從 `local` 環境啟動 `dusk` 指令，則應建立 `.env.dusk.local` 檔案。

執行測試時，Dusk 會備份您的 `.env` 檔案並將您的 Dusk 環境設定檔重新命名為 `.env`。測試完成後，您的 `.env` 檔案將會被還原。

<a name="browser-basics"></a>
## 瀏覽器基礎操作


<a name="creating-browsers"></a>
### 建立瀏覽器

首先，讓我們編寫一個測試來驗證是否可以登入我們的應用程式。建立測試後，我們可以修改它以導覽至登入頁面、輸入憑證，然後點擊「Login」按鈕。若要建立瀏覽器實例，您可以在 Dusk 測試中呼叫 `browse` 方法：

```php tab=Pest
<?php

use App\Models\User;
use Illuminate\Foundation\Testing\DatabaseMigrations;

pest()->use(DatabaseMigrations::class);

test('basic example', function () {
    $user = User::factory()->create([
        'email' => 'taylor@laravel.com',
    ]);

    $this->browse(function (Browser $browser) use ($user) {
        $browser->visit('/login')
            ->type('email', $user->email)
            ->type('password', 'password')
            ->press('Login')
            ->assertPathIs('/home');
    });
});
```

```php tab=PHPUnit
<?php

namespace Tests\Browser;

use App\Models\User;
use Illuminate\Foundation\Testing\DatabaseMigrations;
use Laravel\Dusk\Browser;
use Tests\DuskTestCase;

class ExampleTest extends DuskTestCase
{
    use DatabaseMigrations;

    /**
     * A basic browser test example.
     */
    public function test_basic_example(): void
    {
        $user = User::factory()->create([
            'email' => 'taylor@laravel.com',
        ]);

        $this->browse(function (Browser $browser) use ($user) {
            $browser->visit('/login')
                ->type('email', $user->email)
                ->type('password', 'password')
                ->press('Login')
                ->assertPathIs('/home');
        });
    }
}
```

如您在上方範例所見，`browse` 方法接受一個閉包。Dusk 會自動將瀏覽器實例傳遞給此閉包，它是用於與應用程式互動及進行斷言的主要物件。


<a name="creating-multiple-browsers"></a>
#### 建立多個瀏覽器

有時候您可能需要多個瀏覽器才能妥善執行測試。例如，測試與 websockets 互動的聊天畫面時可能需要多個瀏覽器。若要建立多個瀏覽器，只需在傳入 `browse` 方法的閉包簽名中加入更多瀏覽器引數即可：

```php
$this->browse(function (Browser $first, Browser $second) {
    $first->loginAs(User::find(1))
        ->visit('/home')
        ->waitForText('Message');

    $second->loginAs(User::find(2))
        ->visit('/home')
        ->waitForText('Message')
        ->type('message', 'Hey Taylor')
        ->press('Send');

    $first->waitForText('Hey Taylor')
        ->assertSee('Jeffrey Way');
});
```


<a name="navigation"></a>
### 頁面導覽

`visit` 方法可用於導覽至應用程式內的指定 URI：

```php
$browser->visit('/login');
```

您可以使用 `visitRoute` 方法導覽至[具名路由](/docs/{{version}}/routing#named-routes)：

```php
$browser->visitRoute($routeName, $parameters);
```

您可以使用 `back` 與 `forward` 方法進行「返回」與「前進」導覽：

```php
$browser->back();

$browser->forward();
```

您可以使用 `refresh` 方法重新整理頁面：

```php
$browser->refresh();
```


<a name="resizing-browser-windows"></a>
### 調整瀏覽器視窗大小

您可以使用 `resize` 方法來調整瀏覽器視窗的大小：

```php
$browser->resize(1920, 1080);
```

`maximize` 方法可用於最大化瀏覽器視窗：

```php
$browser->maximize();
```

`fitContent` 方法會將瀏覽器視窗調整為符合其內容的大小：

```php
$browser->fitContent();
```

當測試失敗時，Dusk 會在擷取螢幕截圖之前自動將瀏覽器調整為符合內容的大小。您可以在測試中呼叫 `disableFitOnFailure` 方法來停用此功能：

```php
$browser->disableFitOnFailure();
```

您可以使用 `move` 方法將瀏覽器視窗移動到螢幕上的不同位置：

```php
$browser->move($x = 100, $y = 100);
```


<a name="browser-macros"></a>
### 瀏覽器巨集

如果您想要定義一個可以在各種測試中重複使用的自訂瀏覽器方法，可以使用 `Browser` 類別上的 `macro` 方法。通常，您應該在[服務提供者(Service Providers)](/docs/{{version}}/providers)的 `boot` 方法中呼叫此方法：

```php
<?php

namespace App\Providers;

use Illuminate\Support\ServiceProvider;
use Laravel\Dusk\Browser;

class DuskServiceProvider extends ServiceProvider
{
    /**
     * Register Dusk's browser macros.
     */
    public function boot(): void
    {
        Browser::macro('scrollToElement', function (string $element = null) {
            $this->script("$('html, body').animate({ scrollTop: $('$element').offset().top }, 0);");

            return $this;
        });
    }
}
```

`macro` 函式接受名稱作為其第一個引數，並接受閉包作為其第二個引數。當在 `Browser` 實例上將巨集作為方法呼叫時，將會執行該巨集的閉包：

```php
$this->browse(function (Browser $browser) use ($user) {
    $browser->visit('/pay')
        ->scrollToElement('#credit-card-details')
        ->assertSee('Enter Credit Card Details');
});
```


<a name="authentication"></a>
### 認證

通常，您會測試需要認證的頁面。您可以使用 Dusk 的 `loginAs` 方法，以避免在每次測試期間都要與應用程式的登入畫面進行互動。`loginAs` 方法接受與可認證模型相關聯的主鍵或可認證模型實例：

```php
use App\Models\User;
use Laravel\Dusk\Browser;

$this->browse(function (Browser $browser) {
    $browser->loginAs(User::find(1))
        ->visit('/home');
});
```

> [!WARNING]
> 使用 `loginAs` 方法後，該檔案內的所有測試都將維持該使用者 session。


<a name="cookies"></a>
### Cookies

您可以使用 `cookie` 方法來取得或設定加密 cookie 的值。預設情況下，Laravel 建立的所有 cookie 都是加密的：

```php
$browser->cookie('name');

$browser->cookie('name', 'Taylor');
```

您可以使用 `plainCookie` 方法來取得或設定未加密 cookie 的值：

```php
$browser->plainCookie('name');

$browser->plainCookie('name', 'Taylor');
```

您可以使用 `deleteCookie` 方法來刪除指定的 cookie：

```php
$browser->deleteCookie('name');
```


<a name="executing-javascript"></a>
### 執行 JavaScript

您可以使用 `script` 方法在瀏覽器中執行任意 JavaScript 陳述式：

```php
$browser->script('document.documentElement.scrollTop = 0');

$browser->script([
    'document.body.scrollTop = 0',
    'document.documentElement.scrollTop = 0',
]);

$output = $browser->script('return window.location.pathname');
```


<a name="taking-a-screenshot"></a>
### 擷取螢幕截圖

您可以使用 `screenshot` 方法來擷取螢幕截圖並以指定的檔案名稱儲存。所有截圖將儲存在 `tests/Browser/screenshots` 目錄中：

```php
$browser->screenshot('filename');
```

`responsiveScreenshots` 方法可用於在各種斷點擷取一系列截圖：

```php
$browser->responsiveScreenshots('filename');
```

`screenshotElement` 方法可用於擷取頁面上特定元素的截圖：

```php
$browser->screenshotElement('#selector', 'filename');
```


<a name="storing-console-output-to-disk"></a>
### 儲存主控台輸出至磁碟

您可以使用 `storeConsoleLog` 方法將當前瀏覽器的主控台輸出以指定的檔案名稱寫入磁碟。主控台輸出將儲存在 `tests/Browser/console` 目錄中：

```php
$browser->storeConsoleLog('filename');
```


<a name="storing-page-source-to-disk"></a>
### 儲存網頁原始碼至磁碟

您可以使用 `storeSource` 方法將當前頁面的原始碼以指定的檔案名稱寫入磁碟。頁面原始碼將儲存在 `tests/Browser/source` 目錄中：

```php
$browser->storeSource('filename');
```

<a name="interacting-with-elements"></a>
## 與元素互動


<a name="dusk-selectors"></a>
### Dusk 選擇器

選擇好的 CSS 選擇器來與元素互動是撰寫 Dusk 測試中最困難的部分之一。隨著時間推移，前端的變更可能會導致如下的 CSS 選擇器使您的測試中斷：

```html
// HTML...

<button>Login</button>
```

```php
// Test...

$browser->click('.login-page .container div > button');
```

Dusk 選擇器讓您能專注於撰寫有效的測試，而不是去記住 CSS 選擇器。要定義選擇器，請在 HTML 元素中加入 `dusk` 屬性。接著，在與 Dusk 瀏覽器互動時，在選擇器前面加上 `@` 前綴，即可在測試中操作該附加的元素：

```html
// HTML...

<button dusk="login-button">Login</button>
```

```php
// Test...

$browser->click('@login-button');
```

若有需要，您可以透過 `selectorHtmlAttribute` 方法自訂 Dusk 選擇器所使用的 HTML 屬性。通常，這個方法應該在應用程式的 `AppServiceProvider` 中的 `boot` 方法內呼叫：

```php
use Laravel\Dusk\Dusk;

Dusk::selectorHtmlAttribute('data-dusk');
```


<a name="text-values-and-attributes"></a>
### 文字、值與屬性


<a name="retrieving-setting-values"></a>
#### 取得與設定值

Dusk 提供了多種方法來與頁面上元素的當前值、顯示文字以及屬性進行互動。例如，若要取得符合指定 CSS 或 Dusk 選擇器的元素「值 (value)」，請使用 `value` 方法：

```php
// Retrieve the value...
$value = $browser->value('selector');

// Set the value...
$browser->value('selector', 'value');
```

您可以使用 `inputValue` 方法來取得具有指定欄位名稱的 input 元素的「值」：

```php
$value = $browser->inputValue('field');
```


<a name="retrieving-text"></a>
#### 取得文字

`text` 方法可用於取得符合指定選擇器的元素顯示文字：

```php
$text = $browser->text('selector');
```


<a name="retrieving-attributes"></a>
#### 取得屬性

最後，`attribute` 方法可用於取得符合指定選擇器的元素屬性值：

```php
$attribute = $browser->attribute('selector', 'value');
```


<a name="interacting-with-forms"></a>
### 與表單互動


<a name="typing-values"></a>
#### 輸入值

Dusk 提供了多種方法來與表單和輸入元素互動。首先，讓我們看一個在輸入欄位中鍵入文字的範例：

```php
$browser->type('email', 'taylor@laravel.com');
```

請注意，雖然該方法在必要時可以接受 CSS 選擇器，但我們並不一定要傳入 CSS 選擇器給 `type` 方法。如果未提供 CSS 選擇器，Dusk 將搜尋具有指定 `name` 屬性的 `input` 或 `textarea` 欄位。

若要在不清除欄位內容的情況下附加文字，您可以使用 `append` 方法：

```php
$browser->type('tags', 'foo')
    ->append('tags', ', bar, baz');
```

您可以使用 `clear` 方法清除輸入欄位的值：

```php
$browser->clear('email');
```

您可以使用 `typeSlowly` 方法指示 Dusk 緩慢鍵入。預設情況下，Dusk 會在每次按鍵之間暫停 100 毫秒。若要自訂按鍵之間的間隔時間，可以傳入相應的毫秒數作為該方法的第三個引數：

```php
$browser->typeSlowly('mobile', '+1 (202) 555-5555');

$browser->typeSlowly('mobile', '+1 (202) 555-5555', 300);
```

您可以使用 `appendSlowly` 方法緩慢地附加文字：

```php
$browser->type('tags', 'foo')
    ->appendSlowly('tags', ', bar, baz');
```


<a name="dropdowns"></a>
#### 下拉選單

若要選取 `select` 元素上的可用值，可以使用 `select` 方法。與 `type` 方法一樣，`select` 方法不需要完整的 CSS 選擇器。當傳遞值給 `select` 方法時，您應該傳入底層的 option 值，而非顯示文字：

```php
$browser->select('size', 'Large');
```

您可以省略第二個引數來隨機選取一個選項：

```php
$browser->select('size');
```

透過提供陣列作為 `select` 方法的第二個引數，您可以指示該方法選取多個選項：

```php
$browser->select('categories', ['Art', 'Music']);
```


<a name="checkboxes"></a>
#### 核取方塊

若要「勾選」核取方塊輸入欄位，可以使用 `check` 方法。就像許多其他與輸入相關的方法一樣，不需要完整的 CSS 選擇器。如果找不到符合的 CSS 選擇器，Dusk 將搜尋具有符合 `name` 屬性的核取方塊：

```php
$browser->check('terms');
```

`uncheck` 方法可用於「取消勾選」核取方塊輸入欄位：

```php
$browser->uncheck('terms');
```


<a name="radio-buttons"></a>
#### 單選按鈕

若要「選取」一個 `radio` 輸入選項，可以使用 `radio` 方法。就像許多其他與輸入相關的方法一樣，不需要完整的 CSS 選擇器。如果找不到符合的 CSS 選擇器，Dusk 將搜尋具有符合 `name` 和 `value` 屬性的 `radio` 輸入欄位：

```php
$browser->radio('size', 'large');
```


<a name="attaching-files"></a>
### 附加檔案

`attach` 方法可用於將檔案附加到 `file` 輸入元素中。就像許多其他與輸入相關的方法一樣，不需要完整的 CSS 選擇器。如果找不到符合的 CSS 選擇器，Dusk 將搜尋具有符合 `name` 屬性的 `file` 輸入欄位：

```php
$browser->attach('photo', __DIR__.'/photos/mountains.png');
```

> [!WARNING]
> attach 函式需要在您的伺服器上安裝並啟用 `Zip` PHP 擴充套件。


<a name="pressing-buttons"></a>
### 按下按鈕

`press` 方法可用於點擊頁面上的按鈕元素。提供給 `press` 方法的引數可以是按鈕的顯示文字，也可以是 CSS / Dusk 選擇器：

```php
$browser->press('Login');
```

在提交表單時，許多應用程式會在按下表單的提交按鈕後將其停用，然後在表單提交的 HTTP 請求完成時重新啟用該按鈕。若要按下按鈕並等待按鈕重新啟用，可以使用 `pressAndWaitFor` 方法：

```php
// Press the button and wait a maximum of 5 seconds for it to be enabled...
$browser->pressAndWaitFor('Save');

// Press the button and wait a maximum of 1 second for it to be enabled...
$browser->pressAndWaitFor('Save', 1);
```


<a name="clicking-links"></a>
### 點擊連結

若要點擊連結，您可以在瀏覽器實例上使用 `clickLink` 方法。`clickLink` 方法將點擊具有指定顯示文字的連結：

```php
$browser->clickLink($linkText);
```

您可以使用 `seeLink` 方法來確認具有指定顯示文字的連結是否在頁面上可見：

```php
if ($browser->seeLink($linkText)) {
    // ...
}
```

> [!WARNING]
> 這些方法會與 jQuery 進行互動。如果頁面上沒有 jQuery，Dusk 將自動將其注入到頁面中，以便在測試期間可以使用。

<a name="using-the-keyboard"></a>
### 使用鍵盤

`keys` 方法允許你向指定元素提供比 `type` 方法更為複雜的輸入序列。例如，你可以指示 Dusk 在輸入值的同時按住輔助按鍵。在這個範例中，當向符合給定選擇器的元素輸入 `taylor` 時會同時按住 `shift` 鍵。在輸入完 `taylor` 之後，將在不使用任何輔助按鍵的情況下輸入 `swift`：

```php
$browser->keys('selector', ['{shift}', 'taylor'], 'swift');
```

`keys` 方法的另一個實用情境是向應用程式的主要 CSS 選擇器發送「鍵盤快捷鍵」組合：

```php
$browser->keys('.app', ['{command}', 'j']);
```

> [!NOTE]
> 所有如 `{command}` 的輔助按鍵皆被包圍在 `{}` 字元中，並與 `Facebook\WebDriver\WebDriverKeys` 類別中定義的常數相對應，該類別可[在 GitHub 上找到](https://github.com/php-webdriver/php-webdriver/blob/master/lib/WebDriverKeys.php)。


<a name="fluent-keyboard-interactions"></a>
#### 流暢的鍵盤互動

Dusk 還提供了一個 `withKeyboard` 方法，讓你可以透過 `Laravel\Dusk\Keyboard` 類別流暢地執行複雜的鍵盤互動。`Keyboard` 類別提供了 `press`、`release`、`type` 及 `pause` 方法：

```php
use Laravel\Dusk\Keyboard;

$browser->withKeyboard(function (Keyboard $keyboard) {
    $keyboard->press('c')
        ->pause(1000)
        ->release('c')
        ->type(['c', 'e', 'o']);
});
```


<a name="keyboard-macros"></a>
#### 鍵盤巨集

如果你想要定義自訂的鍵盤互動，並輕鬆在整個測試套件中重複使用，你可以使用 `Keyboard` 類別提供的 `macro` 方法。通常，你應該從[服務提供者(Service Providers)](/docs/{{version}}/providers)的 `boot` 方法中呼叫此方法：

```php
<?php

namespace App\Providers;

use Facebook\WebDriver\WebDriverKeys;
use Illuminate\Support\ServiceProvider;
use Laravel\Dusk\Keyboard;
use Laravel\Dusk\OperatingSystem;

class DuskServiceProvider extends ServiceProvider
{
    /**
     * Register Dusk's browser macros.
     */
    public function boot(): void
    {
        Keyboard::macro('copy', function (string $element = null) {
            $this->type([
                OperatingSystem::onMac() ? WebDriverKeys::META : WebDriverKeys::CONTROL, 'c',
            ]);

            return $this;
        });

        Keyboard::macro('paste', function (string $element = null) {
            $this->type([
                OperatingSystem::onMac() ? WebDriverKeys::META : WebDriverKeys::CONTROL, 'v',
            ]);

            return $this;
        });
    }
}
```

`macro` 函式的第一個引數接受一個名稱，第二個引數接受一個閉包。當在 `Keyboard` 實例上以方法的形式呼叫該巨集時，巨集的閉包將會被執行：

```php
$browser->click('@textarea')
    ->withKeyboard(fn (Keyboard $keyboard) => $keyboard->copy())
    ->click('@another-textarea')
    ->withKeyboard(fn (Keyboard $keyboard) => $keyboard->paste());
```


<a name="using-the-mouse"></a>
### 使用滑鼠


<a name="clicking-on-elements"></a>
#### 點擊元素

`click` 方法可用於點擊符合給定 CSS 或 Dusk 選擇器的元素：

```php
$browser->click('.selector');
```

`clickAtXPath` 方法可用於點擊符合給定 XPath 表達式的元素：

```php
$browser->clickAtXPath('//div[@class = "selector"]');
```

`clickAtPoint` 方法可用於點擊相對於瀏覽器可視區域之指定座標的最上層元素：

```php
$browser->clickAtPoint($x = 0, $y = 0);
```

`doubleClick` 方法可用於模擬滑鼠連按兩下（雙擊）：

```php
$browser->doubleClick();

$browser->doubleClick('.selector');
```

`rightClick` 方法可用於模擬滑鼠右鍵點擊：

```php
$browser->rightClick();

$browser->rightClick('.selector');
```

`clickAndHold` 方法可用於模擬滑鼠按鍵被點擊並按住不放。隨後呼叫 `releaseMouse` 方法將復原此行為並釋放滑鼠按鍵：

```php
$browser->clickAndHold('.selector');

$browser->clickAndHold()
    ->pause(1000)
    ->releaseMouse();
```

`controlClick` 方法可用於模擬瀏覽器內的 `ctrl+click` 事件：

```php
$browser->controlClick();

$browser->controlClick('.selector');
```

`clickWhenVisible` 或 `clickWhenEnabled` 方法可用於等待元素就緒後恰好點擊一次：

```php
$browser->clickWhenVisible('@save-button');
$browser->clickWhenEnabled('@submit-button');
```


<a name="mouseover"></a>
#### 滑鼠懸停

當你需要將滑鼠移動至符合給定 CSS 或 Dusk 選擇器的元素上方時，可以使用 `mouseover` 方法：

```php
$browser->mouseover('.selector');
```


<a name="drag-drop"></a>
#### 拖放

`drag` 方法可用於將符合給定選擇器的元素拖曳至另一個元素：

```php
$browser->drag('.from-selector', '.to-selector');
```

或者，你也可以將元素向單一方向拖曳：

```php
$browser->dragLeft('.selector', $pixels = 10);
$browser->dragRight('.selector', $pixels = 10);
$browser->dragUp('.selector', $pixels = 10);
$browser->dragDown('.selector', $pixels = 10);
```

最後，你可以依照給定的偏移量拖曳元素：

```php
$browser->dragOffset('.selector', $x = 10, $y = 10);
```


<a name="javascript-dialogs"></a>
### JavaScript 對話框

Dusk 提供了多種方法來與 JavaScript 對話框互動。例如，你可以使用 `waitForDialog` 方法來等待 JavaScript 對話框出現。此方法接受一個可選引數，用以指定等待對話框出現的秒數：

```php
$browser->waitForDialog($seconds = null);
```

`assertDialogOpened` 方法可用於斷言對話框已顯示且包含給定的訊息：

```php
$browser->assertDialogOpened('Dialog message');
```

如果 JavaScript 對話框包含輸入提示（prompt），你可以使用 `typeInDialog` 方法向提示輸入值：

```php
$browser->typeInDialog('Hello World');
```

若要透過點擊「確定」按鈕來關閉開啟的 JavaScript 對話框，可以調用 `acceptDialog` 方法：

```php
$browser->acceptDialog();
```

若要透過點擊「取消」按鈕來關閉開啟的 JavaScript 對話框，可以調用 `dismissDialog` 方法：

```php
$browser->dismissDialog();
```


<a name="interacting-with-iframes"></a>
### 與內嵌框架互動

如果你需要與 iframe 內的元素互動，可以使用 `withinFrame` 方法。在傳入 `withinFrame` 方法的閉包內發生的所有元素互動，其範圍都將被限制在指定的 iframe 上下文中：

```php
$browser->withinFrame('#credit-card-details', function ($browser) {
    $browser->type('input[name="cardnumber"]', '4242424242424242')
        ->type('input[name="exp-date"]', '1224')
        ->type('input[name="cvc"]', '123')
        ->press('Pay');
});
```

<a name="scoping-selectors"></a>
### 限定選擇器範圍

有時候你可能希望執行多個操作，同時將所有操作限定在某個指定的選擇器範圍內。例如，你可能希望斷言某些文字僅存在於某個表格內，接著點擊該表格內的按鈕。你可以使用 `with` 方法來達成此目的。在傳遞給 `with` 方法的閉包中執行的所有操作，都將被限定在原始選擇器的範圍內：

```php
$browser->with('.table', function (Browser $table) {
    $table->assertSee('Hello World')
        ->clickLink('Delete');
});
```

有時你可能偶爾需要在當前範圍之外執行斷言。你可以使用 `elsewhere` 與 `elsewhereWhenAvailable` 方法來達成此目的：

```php
$browser->with('.table', function (Browser $table) {
    // Current scope is `body .table`...

    $browser->elsewhere('.page-title', function (Browser $title) {
        // Current scope is `body .page-title`...
        $title->assertSee('Hello World');
    });

    $browser->elsewhereWhenAvailable('.page-title', function (Browser $title) {
        // Current scope is `body .page-title`...
        $title->assertSee('Hello World');
    });
});
```

<a name="waiting-for-elements"></a>
### 等待元素

當測試大量使用 JavaScript 的應用程式時，通常需要「等待」某些元素或資料可用後才能繼續進行測試。Dusk 讓這件事變得非常簡單。透過各種方法，您可以等待元素在頁面上變為可見，甚至可以等待某個 JavaScript 運算式計算結果為 `true`。


<a name="waiting"></a>
#### 等待

若您只需要將測試暫停指定的毫秒數，請使用 `pause` 方法：

```php
$browser->pause(1000);
```

若您只需要在給定條件為 `true` 時才暫停測試，請使用 `pauseIf` 方法：

```php
$browser->pauseIf(App::environment('production'), 1000);
```

同樣地，若您需要除非給定條件為 `true` 否則暫停測試，可以使用 `pauseUnless` 方法：

```php
$browser->pauseUnless(App::environment('testing'), 1000);
```


<a name="waiting-for-selectors"></a>
#### 等待選擇器

`waitFor` 方法可用於暫停測試的執行，直到符合給定 CSS 或 Dusk 選擇器的元素顯示在頁面上。預設情況下，這會在拋出例外前最多暫停測試五秒鐘。如有需要，您可以傳遞自訂的逾時門檻作為該方法的第二個引數：

```php
// Wait a maximum of five seconds for the selector...
$browser->waitFor('.selector');

// Wait a maximum of one second for the selector...
$browser->waitFor('.selector', 1);
```

您也可以等待直到符合給定選擇器的元素包含指定的文字：

```php
// Wait a maximum of five seconds for the selector to contain the given text...
$browser->waitForTextIn('.selector', 'Hello World');

// Wait a maximum of one second for the selector to contain the given text...
$browser->waitForTextIn('.selector', 'Hello World', 1);
```

您也可以等待直到符合給定選擇器的元素從頁面上消失：

```php
// Wait a maximum of five seconds until the selector is missing...
$browser->waitUntilMissing('.selector');

// Wait a maximum of one second until the selector is missing...
$browser->waitUntilMissing('.selector', 1);
```

或者，您可以等待直到符合給定選擇器的元素變為啟用或停用：

```php
// Wait a maximum of five seconds until the selector is enabled...
$browser->waitUntilEnabled('.selector');

// Wait a maximum of one second until the selector is enabled...
$browser->waitUntilEnabled('.selector', 1);

// Wait a maximum of five seconds until the selector is disabled...
$browser->waitUntilDisabled('.selector');

// Wait a maximum of one second until the selector is disabled...
$browser->waitUntilDisabled('.selector', 1);
```


<a name="scoping-selectors-when-available"></a>
#### 當可用時限定選擇器範圍

有時，您可能希望等待符合給定選擇器的元素出現，然後與該元素進行互動。例如，您可能希望等待互動視窗（Modal）可用，然後按下互動視窗內的「確定」按鈕。可以使用 `whenAvailable` 方法來達成此目的。在給定閉包內執行的所有元素操作都將被限定在原始選擇器的範圍內：

```php
$browser->whenAvailable('.modal', function (Browser $modal) {
    $modal->assertSee('Hello World')
        ->press('OK');
});
```


<a name="waiting-for-text"></a>
#### 等待文字

`waitForText` 方法可用於等待直到給定的文字顯示在頁面上：

```php
// Wait a maximum of five seconds for the text...
$browser->waitForText('Hello World');

// Wait a maximum of one second for the text...
$browser->waitForText('Hello World', 1);
```

您可以使用 `waitUntilMissingText` 方法來等待直到顯示的文字從頁面上被移除：

```php
// Wait a maximum of five seconds for the text to be removed...
$browser->waitUntilMissingText('Hello World');

// Wait a maximum of one second for the text to be removed...
$browser->waitUntilMissingText('Hello World', 1);
```


<a name="waiting-for-links"></a>
#### 等待連結

`waitForLink` 方法可用於等待直到給定的連結文字顯示在頁面上：

```php
// Wait a maximum of five seconds for the link...
$browser->waitForLink('Create');

// Wait a maximum of one second for the link...
$browser->waitForLink('Create', 1);
```


<a name="waiting-for-inputs"></a>
#### 等待輸入欄位

`waitForInput` 方法可用於等待直到給定的輸入欄位在頁面上可見：

```php
// Wait a maximum of five seconds for the input...
$browser->waitForInput($field);

// Wait a maximum of one second for the input...
$browser->waitForInput($field, 1);
```


<a name="waiting-on-the-page-location"></a>
#### 等待頁面位置

當進行路徑斷言（例如 `$browser->assertPathIs('/home')`）時，若 `window.location.pathname` 是非同步更新的，斷言可能會失敗。您可以使用 `waitForLocation` 方法來等待位置變為給定的值：

```php
$browser->waitForLocation('/secret');
```

`waitForLocation` 方法也可以用來等待目前的視窗位置變為完整的 URL：

```php
$browser->waitForLocation('https://example.com/path');
```

您也可以等待[具名路由](/docs/{{version}}/routing#named-routes)的位置：

```php
$browser->waitForRoute($routeName, $parameters);
```


<a name="waiting-for-page-reloads"></a>
#### 等待頁面重新載入

若您需要在執行某個動作後等待頁面重新載入，請使用 `waitForReload` 方法：

```php
use Laravel\Dusk\Browser;

$browser->waitForReload(function (Browser $browser) {
    $browser->press('Submit');
})
->assertSee('Success!');
```

由於等待頁面重新載入的需求通常發生在點擊按鈕之後，為了方便起見，您可以使用 `clickAndWaitForReload` 方法：

```php
$browser->clickAndWaitForReload('.selector')
    ->assertSee('something');
```


<a name="waiting-on-javascript-expressions"></a>
#### 等待 JavaScript 運算式

有時您可能希望暫停測試的執行，直到給定的 JavaScript 運算式計算結果為 `true`。您可以使用 `waitUntil` 方法輕鬆達成此目的。將運算式傳遞給此方法時，您不需要包含 `return` 關鍵字或結尾的分號：

```php
// Wait a maximum of five seconds for the expression to be true...
$browser->waitUntil('App.data.servers.length > 0');

// Wait a maximum of one second for the expression to be true...
$browser->waitUntil('App.data.servers.length > 0', 1);
```


<a name="waiting-on-vue-expressions"></a>
#### 等待 Vue 運算式

`waitUntilVue` 與 `waitUntilVueIsNot` 方法可用於等待直到 [Vue 元件](https://vuejs.org) 屬性具有給定的值：

```php
// Wait until the component attribute contains the given value...
$browser->waitUntilVue('user.name', 'Taylor', '@user');

// Wait until the component attribute doesn't contain the given value...
$browser->waitUntilVueIsNot('user.name', null, '@user');
```


<a name="waiting-for-javascript-events"></a>
#### 等待 JavaScript 事件

`waitForEvent` 方法可用於暫停測試的執行，直到某個 JavaScript 事件發生：

```php
$browser->waitForEvent('load');
```

事件監聽器會附加到目前的範圍，預設為 `body` 元素。當使用限定範圍的選擇器時，事件監聽器將會附加到符合的元素上：

```php
$browser->with('iframe', function (Browser $iframe) {
    // Wait for the iframe's load event...
    $iframe->waitForEvent('load');
});
```

您也可以提供選擇器作為 `waitForEvent` 方法的第二個引數，以將事件監聽器附加到特定元素：

```php
$browser->waitForEvent('load', '.selector');
```

您也可以等待 `document` 與 `window` 物件上的事件：

```php
// Wait until the document is scrolled...
$browser->waitForEvent('scroll', 'document');

// Wait a maximum of five seconds until the window is resized...
$browser->waitForEvent('resize', 'window', 5);
```


<a name="waiting-with-a-callback"></a>
#### 使用回呼等待

Dusk 中的許多「等待」方法都依賴底層的 `waitUsing` 方法。您可以直接使用此方法來等待給定的閉包回傳 `true`。`waitUsing` 方法接受最長等待秒數、評估閉包的間隔時間、閉包本體以及可選的失敗訊息：

```php
$browser->waitUsing(10, 1, function () use ($something) {
    return $something->isReady();
}, "Something wasn't ready in time.");
```

<a name="scrolling-an-element-into-view"></a>
### 滾動元素至可視範圍

有時你可能會因為元素位於瀏覽器的可視區域之外而無法點擊它。`scrollIntoView` 方法會滾動瀏覽器視窗，直到符合指定選擇器的元素進入可視範圍內：

```php
$browser->scrollIntoView('.selector')
    ->click('.selector');
```

<a name="available-assertions"></a>
## 可用的斷言

Dusk 提供了多種可對應用程式進行驗證的斷言。所有可用的斷言都記錄在下方清單中：

<style>
    .collection-method-list > p {
        columns: 10.8em 3; -moz-columns: 10.8em 3; -webkit-columns: 10.8em 3;
    }

    .collection-method-list a {
        display: block;
        overflow: hidden;
        text-overflow: ellipsis;
        white-space: nowrap;
    }
</style>

<div class="collection-method-list" markdown="1">

[assertTitle](#assert-title)
[assertTitleContains](#assert-title-contains)
[assertUrlIs](#assert-url-is)
[assertSchemeIs](#assert-scheme-is)
[assertSchemeIsNot](#assert-scheme-is-not)
[assertHostIs](#assert-host-is)
[assertHostIsNot](#assert-host-is-not)
[assertPortIs](#assert-port-is)
[assertPortIsNot](#assert-port-is-not)
[assertPathBeginsWith](#assert-path-begins-with)
[assertPathEndsWith](#assert-path-ends-with)
[assertPathContains](#assert-path-contains)
[assertPathIs](#assert-path-is)
[assertPathIsNot](#assert-path-is-not)
[assertRouteIs](#assert-route-is)
[assertQueryStringHas](#assert-query-string-has)
[assertQueryStringMissing](#assert-query-string-missing)
[assertFragmentIs](#assert-fragment-is)
[assertFragmentBeginsWith](#assert-fragment-begins-with)
[assertFragmentIsNot](#assert-fragment-is-not)
[assertHasCookie](#assert-has-cookie)
[assertHasPlainCookie](#assert-has-plain-cookie)
[assertCookieMissing](#assert-cookie-missing)
[assertPlainCookieMissing](#assert-plain-cookie-missing)
[assertCookieValue](#assert-cookie-value)
[assertPlainCookieValue](#assert-plain-cookie-value)
[assertSee](#assert-see)
[assertDontSee](#assert-dont-see)
[assertSeeIn](#assert-see-in)
[assertDontSeeIn](#assert-dont-see-in)
[assertSeeAnythingIn](#assert-see-anything-in)
[assertSeeNothingIn](#assert-see-nothing-in)
[assertCount](#assert-count)
[assertScript](#assert-script)
[assertSourceHas](#assert-source-has)
[assertSourceMissing](#assert-source-missing)
[assertSeeLink](#assert-see-link)
[assertDontSeeLink](#assert-dont-see-link)
[assertInputValue](#assert-input-value)
[assertInputValueIsNot](#assert-input-value-is-not)
[assertChecked](#assert-checked)
[assertNotChecked](#assert-not-checked)
[assertIndeterminate](#assert-indeterminate)
[assertRadioSelected](#assert-radio-selected)
[assertRadioNotSelected](#assert-radio-not-selected)
[assertSelected](#assert-selected)
[assertNotSelected](#assert-not-selected)
[assertSelectHasOptions](#assert-select-has-options)
[assertSelectMissingOptions](#assert-select-missing-options)
[assertSelectHasOption](#assert-select-has-option)
[assertSelectMissingOption](#assert-select-missing-option)
[assertValue](#assert-value)
[assertValueIsNot](#assert-value-is-not)
[assertAttribute](#assert-attribute)
[assertAttributeMissing](#assert-attribute-missing)
[assertAttributeContains](#assert-attribute-contains)
[assertAttributeDoesntContain](#assert-attribute-doesnt-contain)
[assertAriaAttribute](#assert-aria-attribute)
[assertDataAttribute](#assert-data-attribute)
[assertVisible](#assert-visible)
[assertPresent](#assert-present)
[assertNotPresent](#assert-not-present)
[assertMissing](#assert-missing)
[assertInputPresent](#assert-input-present)
[assertInputMissing](#assert-input-missing)
[assertDialogOpened](#assert-dialog-opened)
[assertEnabled](#assert-enabled)
[assertDisabled](#assert-disabled)
[assertButtonEnabled](#assert-button-enabled)
[assertButtonDisabled](#assert-button-disabled)
[assertFocused](#assert-focused)
[assertNotFocused](#assert-not-focused)
[assertAuthenticated](#assert-authenticated)
[assertGuest](#assert-guest)
[assertAuthenticatedAs](#assert-authenticated-as)
[assertVue](#assert-vue)
[assertVueIsNot](#assert-vue-is-not)
[assertVueContains](#assert-vue-contains)
[assertVueDoesntContain](#assert-vue-doesnt-contain)

</div>


<a name="assert-title"></a>
#### assertTitle

斷言頁面標題與給定的文字相符：

```php
$browser->assertTitle($title);
```


<a name="assert-title-contains"></a>
#### assertTitleContains

斷言頁面標題包含給定的文字：

```php
$browser->assertTitleContains($title);
```


<a name="assert-url-is"></a>
#### assertUrlIs

斷言當前網址（不含查詢字串）與給定的字串相符：

```php
$browser->assertUrlIs($url);
```


<a name="assert-scheme-is"></a>
#### assertSchemeIs

斷言當前網址通訊協定（Scheme）與給定的通訊協定相符：

```php
$browser->assertSchemeIs($scheme);
```


<a name="assert-scheme-is-not"></a>
#### assertSchemeIsNot

斷言當前網址通訊協定（Scheme）與給定的通訊協定不符：

```php
$browser->assertSchemeIsNot($scheme);
```


<a name="assert-host-is"></a>
#### assertHostIs

斷言當前網址主機名稱（Host）與給定的主機名稱相符：

```php
$browser->assertHostIs($host);
```


<a name="assert-host-is-not"></a>
#### assertHostIsNot

斷言當前網址主機名稱（Host）與給定的主機名稱不符：

```php
$browser->assertHostIsNot($host);
```


<a name="assert-port-is"></a>
#### assertPortIs

斷言當前網址連接埠（Port）與給定的連接埠相符：

```php
$browser->assertPortIs($port);
```


<a name="assert-port-is-not"></a>
#### assertPortIsNot

斷言當前網址連接埠（Port）與給定的連接埠不符：

```php
$browser->assertPortIsNot($port);
```


<a name="assert-path-begins-with"></a>
#### assertPathBeginsWith

斷言當前網址路徑以給定的路徑開頭：

```php
$browser->assertPathBeginsWith('/home');
```


<a name="assert-path-ends-with"></a>
#### assertPathEndsWith

斷言當前網址路徑以給定的路徑結尾：

```php
$browser->assertPathEndsWith('/home');
```


<a name="assert-path-contains"></a>
#### assertPathContains

斷言當前網址路徑包含給定的路徑：

```php
$browser->assertPathContains('/home');
```


<a name="assert-path-is"></a>
#### assertPathIs

斷言當前路徑與給定的路徑相符：

```php
$browser->assertPathIs('/home');
```


<a name="assert-path-is-not"></a>
#### assertPathIsNot

斷言當前路徑與給定的路徑不符：

```php
$browser->assertPathIsNot('/home');
```


<a name="assert-route-is"></a>
#### assertRouteIs

斷言當前網址與給定的[命名路由](/docs/{{version}}/routing#named-routes)網址相符：

```php
$browser->assertRouteIs($name, $parameters);
```


<a name="assert-query-string-has"></a>
#### assertQueryStringHas

斷言給定的查詢字串參數存在：

```php
$browser->assertQueryStringHas($name);
```

斷言給定的查詢字串參數存在且具有給定的值：

```php
$browser->assertQueryStringHas($name, $value);
```


<a name="assert-query-string-missing"></a>
#### assertQueryStringMissing

斷言給定的查詢字串參數不存在：

```php
$browser->assertQueryStringMissing($name);
```


<a name="assert-fragment-is"></a>
#### assertFragmentIs

斷言網址當前的 Hash 錨點片段與給定的片段相符：

```php
$browser->assertFragmentIs('anchor');
```


<a name="assert-fragment-begins-with"></a>
#### assertFragmentBeginsWith

斷言網址當前的 Hash 錨點片段以給定的片段開頭：

```php
$browser->assertFragmentBeginsWith('anchor');
```


<a name="assert-fragment-is-not"></a>
#### assertFragmentIsNot

斷言網址當前的 Hash 錨點片段與給定的片段不符：

```php
$browser->assertFragmentIsNot('anchor');
```


<a name="assert-has-cookie"></a>
#### assertHasCookie

斷言給定的加密 Cookie 存在：

```php
$browser->assertHasCookie($name);
```


<a name="assert-has-plain-cookie"></a>
#### assertHasPlainCookie

斷言給定的未加密 Cookie 存在：

```php
$browser->assertHasPlainCookie($name);
```


<a name="assert-cookie-missing"></a>
#### assertCookieMissing

斷言給定的加密 Cookie 不存在：

```php
$browser->assertCookieMissing($name);
```


<a name="assert-plain-cookie-missing"></a>
#### assertPlainCookieMissing

斷言給定的未加密 Cookie 不存在：

```php
$browser->assertPlainCookieMissing($name);
```

<a name="assert-cookie-value"></a>
#### assertCookieValue

斷言加密的 cookie 具有給定的值：

```php
$browser->assertCookieValue($name, $value);
```


<a name="assert-plain-cookie-value"></a>
#### assertPlainCookieValue

斷言未加密的 cookie 具有給定的值：

```php
$browser->assertPlainCookieValue($name, $value);
```


<a name="assert-see"></a>
#### assertSee

斷言給定的文字存在於頁面上：

```php
$browser->assertSee($text);
```


<a name="assert-dont-see"></a>
#### assertDontSee

斷言給定的文字不存在於頁面上：

```php
$browser->assertDontSee($text);
```


<a name="assert-see-in"></a>
#### assertSeeIn

斷言給定的文字存在於選擇器內：

```php
$browser->assertSeeIn($selector, $text);
```


<a name="assert-dont-see-in"></a>
#### assertDontSeeIn

斷言給定的文字不存在於選擇器內：

```php
$browser->assertDontSeeIn($selector, $text);
```


<a name="assert-see-anything-in"></a>
#### assertSeeAnythingIn

斷言選擇器內存在任何文字：

```php
$browser->assertSeeAnythingIn($selector);
```


<a name="assert-see-nothing-in"></a>
#### assertSeeNothingIn

斷言選擇器內不存在任何文字：

```php
$browser->assertSeeNothingIn($selector);
```


<a name="assert-count"></a>
#### assertCount

斷言符合給定選擇器的元素出現指定次數：

```php
$browser->assertCount($selector, $count);
```


<a name="assert-script"></a>
#### assertScript

斷言給定的 JavaScript 運算式求值結果為給定的值：

```php
$browser->assertScript('window.isLoaded')
    ->assertScript('document.readyState', 'complete');
```


<a name="assert-source-has"></a>
#### assertSourceHas

斷言給定的原始碼存在於頁面上：

```php
$browser->assertSourceHas($code);
```


<a name="assert-source-missing"></a>
#### assertSourceMissing

斷言給定的原始碼不存在於頁面上：

```php
$browser->assertSourceMissing($code);
```


<a name="assert-see-link"></a>
#### assertSeeLink

斷言給定的連結存在於頁面上：

```php
$browser->assertSeeLink($linkText);
```


<a name="assert-dont-see-link"></a>
#### assertDontSeeLink

斷言給定的連結不存在於頁面上：

```php
$browser->assertDontSeeLink($linkText);
```


<a name="assert-input-value"></a>
#### assertInputValue

斷言給定的輸入欄位具有給定的值：

```php
$browser->assertInputValue($field, $value);
```


<a name="assert-input-value-is-not"></a>
#### assertInputValueIsNot

斷言給定的輸入欄位不具有給定的值：

```php
$browser->assertInputValueIsNot($field, $value);
```


<a name="assert-checked"></a>
#### assertChecked

斷言給定的核取方塊已被勾選：

```php
$browser->assertChecked($field);
```


<a name="assert-not-checked"></a>
#### assertNotChecked

斷言給定的核取方塊未被勾選：

```php
$browser->assertNotChecked($field);
```


<a name="assert-indeterminate"></a>
#### assertIndeterminate

斷言給定的核取方塊處於不確定狀態：

```php
$browser->assertIndeterminate($field);
```


<a name="assert-radio-selected"></a>
#### assertRadioSelected

斷言給定的單選欄位已被選取：

```php
$browser->assertRadioSelected($field, $value);
```


<a name="assert-radio-not-selected"></a>
#### assertRadioNotSelected

斷言給定的單選欄位未被選取：

```php
$browser->assertRadioNotSelected($field, $value);
```


<a name="assert-selected"></a>
#### assertSelected

斷言給定的下拉式選單已選取給定的值：

```php
$browser->assertSelected($field, $value);
```


<a name="assert-not-selected"></a>
#### assertNotSelected

斷言給定的下拉式選單未選取給定的值：

```php
$browser->assertNotSelected($field, $value);
```


<a name="assert-select-has-options"></a>
#### assertSelectHasOptions

斷言給定陣列的值皆可供選取：

```php
$browser->assertSelectHasOptions($field, $values);
```


<a name="assert-select-missing-options"></a>
#### assertSelectMissingOptions

斷言給定陣列的值皆不可供選取：

```php
$browser->assertSelectMissingOptions($field, $values);
```


<a name="assert-select-has-option"></a>
#### assertSelectHasOption

斷言給定的值在指定的欄位中可供選取：

```php
$browser->assertSelectHasOption($field, $value);
```


<a name="assert-select-missing-option"></a>
#### assertSelectMissingOption

斷言給定的值不可供選取：

```php
$browser->assertSelectMissingOption($field, $value);
```


<a name="assert-value"></a>
#### assertValue

斷言符合給定選擇器的元素具有給定的值：

```php
$browser->assertValue($selector, $value);
```


<a name="assert-value-is-not"></a>
#### assertValueIsNot

斷言符合給定選擇器的元素不具有給定的值：

```php
$browser->assertValueIsNot($selector, $value);
```


<a name="assert-attribute"></a>
#### assertAttribute

斷言符合給定選擇器的元素在指定的屬性中具有給定的值：

```php
$browser->assertAttribute($selector, $attribute, $value);
```


<a name="assert-attribute-missing"></a>
#### assertAttributeMissing

斷言符合給定選擇器的元素缺少指定的屬性：

```php
$browser->assertAttributeMissing($selector, $attribute);
```


<a name="assert-attribute-contains"></a>
#### assertAttributeContains

斷言符合給定選擇器的元素在指定的屬性中包含給定的值：

```php
$browser->assertAttributeContains($selector, $attribute, $value);
```


<a name="assert-attribute-doesnt-contain"></a>
#### assertAttributeDoesntContain

斷言符合給定選擇器的元素在指定的屬性中不包含給定的值：

```php
$browser->assertAttributeDoesntContain($selector, $attribute, $value);
```


<a name="assert-aria-attribute"></a>
#### assertAriaAttribute

斷言符合給定選擇器的元素在指定的 aria 屬性中具有給定的值：

```php
$browser->assertAriaAttribute($selector, $attribute, $value);
```

例如，給定標記 `<button aria-label="Add"></button>`，您可以像這樣對 `aria-label` 屬性進行斷言：

```php
$browser->assertAriaAttribute('button', 'label', 'Add')
```


<a name="assert-data-attribute"></a>
#### assertDataAttribute

斷言符合給定選擇器的元素在指定的 data 屬性中具有給定的值：

```php
$browser->assertDataAttribute($selector, $attribute, $value);
```

例如，給定標記 `<tr id="row-1" data-content="attendees"></tr>`，您可以像這樣對 `data-content` 屬性進行斷言：

```php
$browser->assertDataAttribute('#row-1', 'content', 'attendees')
```


<a name="assert-visible"></a>
#### assertVisible

斷言符合給定選擇器的元素是可見的：

```php
$browser->assertVisible($selector);
```


<a name="assert-present"></a>
#### assertPresent

斷言符合給定選擇器的元素存在於原始碼中：

```php
$browser->assertPresent($selector);
```


<a name="assert-not-present"></a>
#### assertNotPresent

斷言符合給定選擇器的元素不存在於原始碼中：

```php
$browser->assertNotPresent($selector);
```


<a name="assert-missing"></a>
#### assertMissing

斷言符合給定選擇器的元素是不可見的：

```php
$browser->assertMissing($selector);
```


<a name="assert-input-present"></a>
#### assertInputPresent

斷言具有給定名稱的輸入欄位存在：

```php
$browser->assertInputPresent($name);
```


<a name="assert-input-missing"></a>
#### assertInputMissing

斷言具有給定名稱的輸入欄位不存在於原始碼中：

```php
$browser->assertInputMissing($name);
```

<a name="assert-dialog-opened"></a>
#### assertDialogOpened

斷言已開啟帶有指定訊息的 JavaScript 對話框：

```php
$browser->assertDialogOpened($message);
```


<a name="assert-enabled"></a>
#### assertEnabled

斷言指定的欄位已啟用：

```php
$browser->assertEnabled($field);
```


<a name="assert-disabled"></a>
#### assertDisabled

斷言指定的欄位已停用：

```php
$browser->assertDisabled($field);
```


<a name="assert-button-enabled"></a>
#### assertButtonEnabled

斷言指定的按鈕已啟用：

```php
$browser->assertButtonEnabled($button);
```


<a name="assert-button-disabled"></a>
#### assertButtonDisabled

斷言指定的按鈕已停用：

```php
$browser->assertButtonDisabled($button);
```


<a name="assert-focused"></a>
#### assertFocused

斷言指定的欄位處於焦點狀態：

```php
$browser->assertFocused($field);
```


<a name="assert-not-focused"></a>
#### assertNotFocused

斷言指定的欄位未處於焦點狀態：

```php
$browser->assertNotFocused($field);
```


<a name="assert-authenticated"></a>
#### assertAuthenticated

斷言使用者已通過認證：

```php
$browser->assertAuthenticated();
```


<a name="assert-guest"></a>
#### assertGuest

斷言使用者未通過認證：

```php
$browser->assertGuest();
```


<a name="assert-authenticated-as"></a>
#### assertAuthenticatedAs

斷言使用者已作為指定的使用者通過認證：

```php
$browser->assertAuthenticatedAs($user);
```


<a name="assert-vue"></a>
#### assertVue

Dusk 甚至允許你對 [Vue 元件](https://vuejs.org)資料的狀態進行斷言。例如，假設你的應用程式包含以下 Vue 元件：

    // HTML...

    <profile dusk="profile-component"></profile>

    // Component Definition...

    Vue.component('profile', {
        template: '<div>{{ user.name }}</div>',

        data: function () {
            return {
                user: {
                    name: 'Taylor'
                }
            };
        }
    });

你可以像這樣對 Vue 元件的狀態進行斷言：

```php tab=Pest
test('vue', function () {
    $this->browse(function (Browser $browser) {
        $browser->visit('/')
            ->assertVue('user.name', 'Taylor', '@profile-component');
    });
});
```

```php tab=PHPUnit
/**
 * A basic Vue test example.
 */
public function test_vue(): void
{
    $this->browse(function (Browser $browser) {
        $browser->visit('/')
            ->assertVue('user.name', 'Taylor', '@profile-component');
    });
}
```


<a name="assert-vue-is-not"></a>
#### assertVueIsNot

斷言指定的 Vue 元件資料屬性與給定值不相符：

```php
$browser->assertVueIsNot($property, $value, $componentSelector = null);
```


<a name="assert-vue-contains"></a>
#### assertVueContains

斷言指定的 Vue 元件資料屬性為陣列且包含給定值：

```php
$browser->assertVueContains($property, $value, $componentSelector = null);
```


<a name="assert-vue-doesnt-contain"></a>
#### assertVueDoesntContain

斷言指定的 Vue 元件資料屬性為陣列且不包含給定值：

```php
$browser->assertVueDoesntContain($property, $value, $componentSelector = null);
```

<a name="pages"></a>
## 頁面

有時，測試需要依序執行多個複雜的操作。這可能會讓您的測試變得難以閱讀與理解。Dusk 頁面（Pages）允許您定義具表達力的操作，隨後可透過單一方法在指定頁面上執行這些操作。頁面還允許您為應用程式或單一頁面定義常用選擇器的快捷方式。


<a name="generating-pages"></a>
### 建立頁面

若要建立一個頁面物件，請執行 `dusk:page` Artisan 指令。所有頁面物件都將放置在應用程式的 `tests/Browser/Pages` 目錄中：

```shell
php artisan dusk:page Login
```


<a name="configuring-pages"></a>
### 設定頁面

預設情況下，頁面有三個方法：`url`、`assert` 與 `elements`。我們現在將討論 `url` 和 `assert` 方法。`elements` 方法將在[下方進行更詳細的討論](#shorthand-selectors)。


<a name="the-url-method"></a>
#### `url` 方法

`url` 方法應該回傳代表該頁面的 URL 路徑。Dusk 在瀏覽器中導覽至該頁面時將使用此 URL：

```php
/**
 * Get the URL for the page.
 */
public function url(): string
{
    return '/login';
}
```


<a name="the-assert-method"></a>
#### `assert` 方法

`assert` 方法可以進行任何必要的斷言，以驗證瀏覽器是否確實位於指定頁面上。實際上不需要在此方法中放置任何內容；但是，如果您願意，可以自由進行這些斷言。這些斷言將在導覽至該頁面時自動執行：

```php
/**
 * Assert that the browser is on the page.
 */
public function assert(Browser $browser): void
{
    $browser->assertPathIs($this->url());
}
```


<a name="navigating-to-pages"></a>
### 導覽至頁面

定義頁面後，您可以使用 `visit` 方法導覽至該頁面：

```php
use Tests\Browser\Pages\Login;

$browser->visit(new Login);
```

有時您可能已經位於某個頁面上，並需要將該頁面的選擇器和方法「載入」到目前的測試上下文中。這在按下按鈕並被重新導向到指定頁面（而非明確導覽至該頁面）時很常見。在此情況下，您可以使用 `on` 方法來載入該頁面：

```php
use Tests\Browser\Pages\CreatePlaylist;

$browser->visit('/dashboard')
    ->clickLink('Create Playlist')
    ->on(new CreatePlaylist)
    ->assertSee('@create');
```


<a name="shorthand-selectors"></a>
### 簡寫選擇器

頁面類別中的 `elements` 方法允許您為頁面上的任何 CSS 選擇器定義快速、易記的捷徑。例如，讓我們為應用程式登入頁面的「email」輸入欄位定義一個捷徑：

```php
/**
 * Get the element shortcuts for the page.
 *
 * @return array<string, string>
 */
public function elements(): array
{
    return [
        '@email' => 'input[name=email]',
    ];
}
```

定義捷徑後，您可以在通常會使用完整 CSS 選擇器的任何地方使用該簡寫選擇器：

```php
$browser->type('@email', 'taylor@laravel.com');
```


<a name="global-shorthand-selectors"></a>
#### 全域簡寫選擇器

安裝 Dusk 後，基礎 `Page` 類別將放置在您的 `tests/Browser/Pages` 目錄中。該類別包含一個 `siteElements` 方法，可用於定義應在應用程式中每個頁面上都可用的全域簡寫選擇器：

```php
/**
 * Get the global element shortcuts for the site.
 *
 * @return array<string, string>
 */
public static function siteElements(): array
{
    return [
        '@element' => '#selector',
    ];
}
```


<a name="page-methods"></a>
### 頁面方法

除了頁面上定義的預設方法之外，您還可以定義可在整個測試中使用的其他方法。例如，假設我們正在建構一個音樂管理應用程式。該應用程式的一個頁面上的常見操作可能是建立播放清單。您可以不用在每個測試中重複撰寫建立播放清單的邏輯，而是在頁面類別上定義一個 `createPlaylist` 方法：

```php
<?php

namespace Tests\Browser\Pages;

use Laravel\Dusk\Browser;
use Laravel\Dusk\Page;

class Dashboard extends Page
{
    // Other page methods...

    /**
     * Create a new playlist.
     */
    public function createPlaylist(Browser $browser, string $name): void
    {
        $browser->type('name', $name)
            ->check('share')
            ->press('Create Playlist');
    }
}
```

定義方法後，您可以在使用該頁面的任何測試中使用它。瀏覽器實例將自動作為第一個引數傳遞給自訂頁面方法：

```php
use Tests\Browser\Pages\Dashboard;

$browser->visit(new Dashboard)
    ->createPlaylist('My Playlist')
    ->assertSee('My Playlist');
```

<a name="components"></a>
## 元件

元件與 Dusk 的「頁面物件」類似，但專門用於在整個應用程式中重複使用的 UI 與功能區塊，例如導覽列或通知視窗。因此，元件不會綁定到特定的 URL。

<a name="generating-components"></a>
### 建立元件

要建立元件，請執行 `dusk:component` Artisan 指令。新元件會放置在 `tests/Browser/Components` 目錄中：

```shell
php artisan dusk:component DatePicker
```

如上所示，「日期選擇器 (date picker)」就是一個可能存在於應用程式中多個頁面的元件範例。在測試套件中的數十個測試裡，手動撰寫選擇日期的瀏覽器自動化邏輯會變得很繁瑣。相反地，我們可以定義一個 Dusk 元件來代表該日期選擇器，讓我們將該邏輯封裝在元件內：

```php
<?php

namespace Tests\Browser\Components;

use Laravel\Dusk\Browser;
use Laravel\Dusk\Component as BaseComponent;

class DatePicker extends BaseComponent
{
    /**
     * Get the root selector for the component.
     */
    public function selector(): string
    {
        return '.date-picker';
    }

    /**
     * Assert that the browser page contains the component.
     */
    public function assert(Browser $browser): void
    {
        $browser->assertVisible($this->selector());
    }

    /**
     * Get the element shortcuts for the component.
     *
     * @return array<string, string>
     */
    public function elements(): array
    {
        return [
            '@date-field' => 'input.datepicker-input',
            '@year-list' => 'div > div.datepicker-years',
            '@month-list' => 'div > div.datepicker-months',
            '@day-list' => 'div > div.datepicker-days',
        ];
    }

    /**
     * Select the given date.
     */
    public function selectDate(Browser $browser, int $year, int $month, int $day): void
    {
        $browser->click('@date-field')
            ->within('@year-list', function (Browser $browser) use ($year) {
                $browser->click($year);
            })
            ->within('@month-list', function (Browser $browser) use ($month) {
                $browser->click($month);
            })
            ->within('@day-list', function (Browser $browser) use ($day) {
                $browser->click($day);
            });
    }
}
```

<a name="using-components"></a>
### 使用元件

定義好元件後，我們就可以在任何測試中輕鬆地於日期選擇器內選擇日期。而且，如果選擇日期的邏輯發生變更，我們只需要更新該元件即可：

```php tab=Pest
<?php

use Illuminate\Foundation\Testing\DatabaseMigrations;
use Tests\Browser\Components\DatePicker;

pest()->use(DatabaseMigrations::class);

test('basic example', function () {
    $this->browse(function (Browser $browser) {
        $browser->visit('/')
            ->within(new DatePicker, function (Browser $browser) {
                $browser->selectDate(2019, 1, 30);
            })
            ->assertSee('January');
    });
});
```

```php tab=PHPUnit
<?php

namespace Tests\Browser;

use Illuminate\Foundation\Testing\DatabaseMigrations;
use Laravel\Dusk\Browser;
use Tests\Browser\Components\DatePicker;
use Tests\DuskTestCase;

class ExampleTest extends DuskTestCase
{
    /**
     * A basic component test example.
     */
    public function test_basic_example(): void
    {
        $this->browse(function (Browser $browser) {
            $browser->visit('/')
                ->within(new DatePicker, function (Browser $browser) {
                    $browser->selectDate(2019, 1, 30);
                })
                ->assertSee('January');
        });
    }
}
```

可以使用 `component` 方法來取得限定於給定元件範圍的瀏覽器實例：

```php
$datePicker = $browser->component(new DatePickerComponent);

$datePicker->selectDate(2019, 1, 30);

$datePicker->assertSee('January');
```

<a name="continuous-integration"></a>
## 持續整合

> [!WARNING]
> 大多數 Dusk 持續整合設定都預期你的 Laravel 應用程式會透過內建的 PHP 開發伺服器運行在 8000 埠 (port)。因此，在繼續之前，你應確保持續整合環境中的 `APP_URL` 環境變數值設定為 `http://127.0.0.1:8000`。

<a name="running-tests-on-heroku-ci"></a>
### Heroku CI

要在 [Heroku CI](https://www.heroku.com/continuous-integration) 上執行 Dusk 測試，請將以下 Google Chrome buildpack 和指令碼加入到你的 Heroku `app.json` 檔案中：

```json
{
  "environments": {
    "test": {
      "buildpacks": [
        { "url": "heroku/php" },
        { "url": "https://github.com/heroku/heroku-buildpack-chrome-for-testing" }
      ],
      "scripts": {
        "test-setup": "cp .env.testing .env",
        "test": "nohup bash -c './vendor/laravel/dusk/bin/chromedriver-linux --port=9515 > /dev/null 2>&1 &' && nohup bash -c 'php artisan serve --no-reload > /dev/null 2>&1 &' && php artisan dusk"
      }
    }
  }
}
```

<a name="running-tests-on-travis-ci"></a>
### Travis CI

要在 [Travis CI](https://travis-ci.org) 上執行 Dusk 測試，請使用以下 `.travis.yml` 設定檔。由於 Travis CI 不是圖形化環境，我們需要採取一些額外步驟來啟動 Chrome 瀏覽器。此外，我們將使用 `php artisan serve` 來啟動 PHP 的內建網頁伺服器：

```yaml
language: php

php:
  - 8.2

addons:
  chrome: stable

install:
  - cp .env.testing .env
  - travis_retry composer install --no-interaction --prefer-dist
  - php artisan key:generate
  - php artisan dusk:chrome-driver

before_script:
  - google-chrome-stable --headless --disable-gpu --remote-debugging-port=9222 http://localhost &
  - php artisan serve --no-reload &

script:
  - php artisan dusk
```

<a name="running-tests-on-github-actions"></a>
### GitHub Actions

如果你使用 [GitHub Actions](https://github.com/features/actions) 來執行 Dusk 測試，可以使用以下設定檔作為起點。與 Travis CI 一樣，我們將使用 `php artisan serve` 命令來啟動 PHP 的內建網頁伺服器：

```yaml
name: CI
on: [push]
jobs:

  dusk-php:
    runs-on: ubuntu-latest
    env:
      APP_URL: "http://127.0.0.1:8000"
      DB_USERNAME: root
      DB_PASSWORD: root
      MAIL_MAILER: log
    steps:
      - uses: actions/checkout@v5
      - name: Prepare The Environment
        run: cp .env.example .env
      - name: Create Database
        run: |
          sudo systemctl start mysql
          mysql --user="root" --password="root" -e "CREATE DATABASE \`my-database\` character set UTF8mb4 collate utf8mb4_bin;"
      - name: Install Composer Dependencies
        run: composer install --no-progress --prefer-dist --optimize-autoloader
      - name: Generate Application Key
        run: php artisan key:generate
      - name: Upgrade Chrome Driver
        run: php artisan dusk:chrome-driver --detect
      - name: Start Chrome Driver
        run: ./vendor/laravel/dusk/bin/chromedriver-linux --port=9515 &
      - name: Run Laravel Server
        run: php artisan serve --no-reload &
      - name: Run Dusk Tests
        run: php artisan dusk
      - name: Upload Screenshots
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: screenshots
          path: tests/Browser/screenshots
      - name: Upload Console Logs
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: console
          path: tests/Browser/console
```

<a name="running-tests-on-chipper-ci"></a>
### Chipper CI

如果你使用 [Chipper CI](https://chipperci.com) 來執行 Dusk 測試，可以使用以下設定檔作為起點。我們將使用 PHP 內建伺服器來運行 Laravel，以便監聽請求：

```yaml
# file .chipperci.yml
version: 1

environment:
  php: 8.2
  node: 16

# Include Chrome in the build environment
services:
  - dusk

# Build all commits
on:
   push:
      branches: .*

pipeline:
  - name: Setup
    cmd: |
      cp -v .env.example .env
      composer install --no-interaction --prefer-dist --optimize-autoloader
      php artisan key:generate

      # Create a dusk env file, ensuring APP_URL uses BUILD_HOST
      cp -v .env .env.dusk.ci
      sed -i "s@APP_URL=.*@APP_URL=http://$BUILD_HOST:8000@g" .env.dusk.ci

  - name: Compile Assets
    cmd: |
      npm ci --no-audit
      npm run build

  - name: Browser Tests
    cmd: |
      php -S [::0]:8000 -t public 2>server.log &
      sleep 2
      php artisan dusk:chrome-driver $CHROME_DRIVER
      php artisan dusk --env=ci
```

要深入了解如何在 Chipper CI 上執行 Dusk 測試（包含如何使用資料庫），請參考 [Chipper CI 官方文件](https://chipperci.com/docs/testing/laravel-dusk-new/)。