# 本地化

- [簡介](#introduction)
    - [發布語系檔](#publishing-the-language-files)
    - [設定地區](#configuring-the-locale)
    - [複數用語系](#pluralization-language)
- [定義翻譯字串](#defining-translation-strings)
    - [使用簡短索引鍵](#using-short-keys)
    - [使用翻譯字串作為索引鍵](#using-translation-strings-as-keys)
- [取得翻譯字串](#retrieving-translation-strings)
    - [替換翻譯字串中的參數](#replacing-parameters-in-translation-strings)
    - [複數化](#pluralization)
- [覆寫套件的語系檔](#overriding-package-language-files)

<a name="introduction"></a>
## 簡介

> [!NOTE]
> 預設情況下，Laravel 應用程式骨架未包含 `lang` 目錄。若想自訂 Laravel 的語系檔，可透過 `lang:publish` Artisan 指令來發布這些檔案。

Laravel 的本地化功能提供了一個方便的方式來取得各種語系的字串，讓你可以輕鬆地在應用程式中支援多國語系。

Laravel 提供了兩種管理翻譯字串的方式。第一種，語系字串可以儲存在應用程式的 `lang` 目錄下的檔案中。在這個目錄下，可以為應用程式支援的每種語系建立子目錄。這就是 Laravel 用來管理內建功能 (如驗證錯誤訊息) 的翻譯字串的方法：

```text
/lang
    /en
        messages.php
    /es
        messages.php
```

或者，翻譯字串也可以定義在 `lang` 目錄下的 JSON 檔案中。採用這種方法時，應用程式支援的每種語系都會在此目錄下有一個對應的 JSON 檔案。對於有大量可翻譯字串的應用程式，建議使用這種方法：

```text
/lang
    en.json
    es.json
```

我們將在本文件中討論這兩種管理翻譯字串的方法。

<a name="publishing-the-language-files"></a>
### 發布語系檔

預設情況下，Laravel 應用程式骨架未包含 `lang` 目錄。若想自訂 Laravel 的語系檔或建立自己的語系檔，應透過 `lang:publish` Artisan 指令來搭建 `lang` 目錄。`lang:publish` 指令會在你的應用程式中建立 `lang` 目錄，並發布 Laravel 使用的預設語系檔：

```shell
php artisan lang:publish
```

<a name="configuring-the-locale"></a>
### 設定地區

應用程式的預設語系儲存在 `config/app.php` 設定檔的 `locale` 設定選項中，通常是透過 `APP_LOCALE` 環境變數來設定。你可以自由修改此值以滿足應用程式的需求。

你也可以設定一個「備用語系 (fallback language)」，當預設語系不包含給定的翻譯字串時，就會使用這個備用語系。與預設語系一樣，備用語系也在 `config/app.php` 設定檔中設定，其值通常是透過 `APP_FALLBACK_LOCALE` 環境變數來設定。

你可以使用 `App` Facade 提供的 `setLocale` 方法，在執行階段為單一 HTTP 請求修改預設語系：

```php
use Illuminate\Support\Facades\App;

Route::get('/greeting/{locale}', function (string $locale) {
    if (! in_array($locale, ['en', 'es', 'fr'])) {
        abort(400);
    }

    App::setLocale($locale);

    // ...
});
```

<a name="determining-the-current-locale"></a>
#### 判斷當前地區

你可以使用 `App` Facade 上的 `currentLocale` 和 `isLocale` 方法來判斷當前地區或檢查地區是否為給定值：

```php
use Illuminate\Support\Facades\App;

$locale = App::currentLocale();

if (App::isLocale('en')) {
    // ...
}
```

<a name="pluralization-language"></a>
### 複數用語系

<style>
.code-list-no-flex-break code {
    display: contents !important;
}
</style>

<div class="code-list-no-flex-break">

你可以指示 Laravel 的「pluralizer」(複數處理器)，也就是 Eloquent 和框架其他部分用來將單數字串轉為複數字串的工具，使用英語以外的語系。這可以透過在應用程式的其中一個 Service Provider 的 `boot` 方法中叫用 `useLanguage` 方法來達成。pluralizer 目前支援的語系有：`french`、`norwegian-bokmal`、`portuguese`、`spanish` 和 `turkish`：

</div>

```php
use Illuminate\Support\Pluralizer;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Pluralizer::useLanguage('spanish');

    // ...
}
```

> [!WARNING]
> 若你自訂了 pluralizer 的語系，應明確地定義你的 Eloquent Model 的[資料表名稱](/docs/{{version}}/eloquent#table-names)。

<a name="defining-translation-strings"></a>
## 定義翻譯字串

<a name="using-short-keys"></a>
### 使用簡短索引鍵

一般來說，翻譯字串儲存在 `lang` 目錄下的檔案中。在這個目錄下，應該為應用程式支援的每種語系建立一個子目錄。這就是 Laravel 用來管理內建功能 (如驗證錯誤訊息) 的翻譯字串的方法：

```text
/lang
    /en
        messages.php
    /es
        messages.php
```

所有的語系檔都會回傳一個帶有索引鍵的字串陣列。例如：

```php
<?php

// lang/en/messages.php

return [
    'welcome' => 'Welcome to our application!',
];
```

> [!WARNING]
> 對於因地區而異的語系，應根據 ISO 15897 來命名語系目錄。例如，英式英語應使用「en_GB」而不是「en-gb」。

<a name="using-translation-strings-as-keys"></a>
### 使用翻譯字串作為索引鍵

對於有大量可翻譯字串的應用程式，用「簡短索引鍵」來定義每個字串可能會在 View 中參考這些索引鍵時變得很混亂，而且要不斷為應用程式支援的每個翻譯字串發明新的索引鍵是很麻煩的。

因此，Laravel 也支援使用字串的「預設」翻譯作為索引鍵來定義翻譯字串。使用翻譯字串作為索引鍵的語系檔會以 JSON 檔案的形式儲存在 `lang` 目錄中。例如，如果你的應用程式有西班牙語翻譯，你應該建立一個 `lang/es.json` 檔案：

```json
{
    "I love programming.": "Me encanta programar."
}
```

#### 索引鍵 / 檔案衝突

不應定義會與其他翻譯檔檔名衝突的翻譯字串索引鍵。例如，在 `nl/action.php` 檔案存在但 `nl.json` 檔案不存在的情況下，為「NL」地區翻譯 `__('Action')`，會導致翻譯器回傳 `nl/action.php` 的完整內容。

<a name="retrieving-translation-strings"></a>
## 取得翻譯字串

可使用 `__` 輔助函式來從語系檔中取得翻譯字串。若使用「簡短索引鍵」來定義翻譯字串，則應使用「點」語法將包含該索引鍵的檔案與該索引鍵本身傳給 `__` 函式。舉例來說，讓我們來從 `lang/en/messages.php` 語系檔中取得 `welcome` 翻譯字串：

```php
echo __('messages.welcome');
```

若指定的翻譯字串不存在，則 `__` 函式會回傳該翻譯字串的索引鍵。所以，以上述範例來說，若翻譯字串不存在，`__` 函式就會回傳 `messages.welcome`。

若[使用預設的翻譯字串作為翻譯索引鍵](#using-translation-strings-as-keys)，則應將字串的預設翻譯傳給 `__` 函式：

```php
echo __('I love programming.');
```

同樣地，若翻譯字串不存在，`__` 函式會回傳其收到的翻譯字串索引鍵。

若正在使用 [Blade 樣板引擎](/docs/{{version}}/blade)，則可使用 `{{ }}` Echo 語法來顯示翻譯字串：

```blade
{{ __('messages.welcome') }}
```


<a name="replacing-parameters-in-translation-strings"></a>
### 替換翻譯字串中的參數

若有需要，也可以在翻譯字串中定義預留位置。所有的預留位置都以 `:` 作為前綴。舉例來說，可以像這樣定義一個帶有名稱預留位置的歡迎訊息：

```php
'welcome' => 'Welcome, :name',
```

若要在取得翻譯字串時替換掉預留位置，可傳遞一個用來替換的陣列作為 `__` 函式的第二個引數：

```php
echo __('messages.welcome', ['name' => 'dayle']);
```

若預留位置全為大寫字母，或只有首字母為大寫，則翻譯出來的值也會有相對應的大小寫：

```php
'welcome' => 'Welcome, :NAME', // Welcome, DAYLE
'goodbye' => 'Goodbye, :Name', // Goodbye, Dayle
```


<a name="object-replacement-formatting"></a>
#### 物件替換格式化

若嘗試提供一個物件作為翻譯的預留位置，則會叫用該物件的 `__toString` 方法。[__toString](https://www.php.net/manual/en/language.oop5.magic.php#object.tostring) 方法是 PHP 的內建「魔術方法」之一。不過，有時候我們可能無法控制某個給定類別的 `__toString` 方法，例如當我們要互動的類別屬於第三方函式庫時。

在這些情況下，Laravel 允許為特定類型的物件註冊一個自訂的格式化處理常式。為此，應叫用譯者 (translator) 的 `stringable` 方法。`stringable` 方法可接受一個閉包，該閉包應對其負責格式化的物件類型進行型別提示。一般來說，`stringable` 方法應在應用程式 `AppServiceProvider` 類別的 `boot` 方法中叫用：

```php
use Illuminate\Support\Facades\Lang;
use Money\Money;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Lang::stringable(function (Money $money) {
        return $money->formatTo('en_GB');
    });
}
```


<a name="pluralization"></a>
### 複數化

複數化是個複雜的問題，因為不同的語系對複數有各種複雜的規則。不過，Laravel 可以幫助我們根據自訂的複數規則來以不同方式翻譯字串。使用 `|` 字元，即可區分字串的單數與複數形式：

```php
'apples' => 'There is one apple|There are many apples',
```

當然，在[使用翻譯字串作為索引鍵](#using-translation-strings-as-keys)時也支援複數化：

```json
{
    "There is one apple|There are many apples": "Hay una manzana|Hay muchas manzanas"
}
```

我們甚至可以建立更複雜的複數規則，為多個數值範圍指定不同的翻譯字串：

```php
'apples' => '{0} There are none|[1,19] There are some|[20,*] There are many',
```

在定義了有複數選項的翻譯字串後，可使用 `trans_choice` 函式來為給定的「數量」取得該行。在這個範例中，由於數量大於 1，因此會回傳複數形式的翻譯字串：

```php
echo trans_choice('messages.apples', 10);
```

也可以在複數字串中定義預留位置屬性。只要傳遞一個陣列作為 `trans_choice` 函式的第三個引數即可替換掉這些預留位置：

```php
'minutes_ago' => '{1} :value minute ago|[2,*] :value minutes ago',

echo trans_choice('time.minutes_ago', 5, ['value' => 5]);
```

若想顯示傳給 `trans_choice` 函式的整數值，可使用內建的 `:count` 預留位置：

```php
'apples' => '{0} There are none|{1} There is one|[2,*] There are :count',
```


<a name="overriding-package-language-files"></a>
## 覆寫套件的語系檔

有些套件可能會附帶自己的語系檔。與其為了調整這些行而去修改套件的核心檔，我們可以在 `lang/vendor/{package}/{locale}` 目錄下放置檔案來覆寫它們。

舉例來說，若想為 `skyrim/hearthfire` 套件覆寫 `messages.php` 中的英文翻譯字串，就應該將語系檔放置在 `lang/vendor/hearthfire/en/messages.php`。在這個檔案中，只需定義想覆寫的翻譯字串即可。任何未覆寫的翻譯字串仍會從套件原本的語系檔中載入。