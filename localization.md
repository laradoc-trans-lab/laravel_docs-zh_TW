# 本地化

- [簡介](#introduction)
    - [發佈語系檔案](#publishing-the-language-files)
    - [設定語系](#configuring-the-locale)
    - [複數形語系](#pluralization-language)
- [定義翻譯字串](#defining-translation-strings)
    - [使用短鍵](#using-short-keys)
    - [使用翻譯字串作為鍵](#using-translation-strings-as-keys)
- [取得翻譯字串](#retrieving-translation-strings)
    - [替換翻譯字串中的參數](#replacing-parameters-in-translation-strings)
    - [複數形](#pluralization)
- [覆寫套件語系檔案](#overriding-package-language-files)

<a name="introduction"></a>
## 簡介

> [!NOTE]  
> 預設情況下，Laravel 應用程式骨架不包含 `lang` 目錄。如果您想自訂 Laravel 的語系檔案，可以透過 `lang:publish` Artisan 命令來發佈它們。

Laravel 的本地化功能提供了一種方便的方式來取得各種語言的字串，讓您能夠輕鬆地在應用程式中支援多種語言。

Laravel 提供了兩種管理翻譯字串的方式。首先，語系字串可以儲存在應用程式 `lang` 目錄中的檔案裡。在這個目錄中，可以為應用程式支援的每種語言建立子目錄。這是 Laravel 用來管理內建 Laravel 功能（例如驗證錯誤訊息）翻譯字串的方法：

    /lang
        /en
            messages.php
        /es
            messages.php

或者，翻譯字串可以定義在放置於 `lang` 目錄中的 JSON 檔案裡。採用這種方法時，應用程式支援的每種語言都將在此目錄中擁有一個對應的 JSON 檔案。對於擁有大量可翻譯字串的應用程式，建議採用這種方法：

    /lang
        en.json
        es.json

我們將在本文件中討論管理翻譯字串的這兩種方法。

<a name="publishing-the-language-files"></a>
### 發佈語系檔案

預設情況下，Laravel 應用程式骨架不包含 `lang` 目錄。如果您想自訂 Laravel 的語系檔案或建立自己的語系檔案，您應該透過 `lang:publish` Artisan 命令來建立 `lang` 目錄。`lang:publish` 命令將在您的應用程式中建立 `lang` 目錄並發佈 Laravel 使用的預設語系檔案集：

```shell
php artisan lang:publish
```

<a name="configuring-the-locale"></a>
### 設定語系

應用程式的預設語言儲存在 `config/app.php` 設定檔的 `locale` 設定選項中，該選項通常使用 `APP_LOCALE` 環境變數來設定。您可以自由修改此值以符合應用程式的需求。

您還可以設定一個「備用語系 (fallback language)」，當預設語系不包含給定的翻譯字串時，將使用該語系。與預設語系一樣，備用語系也配置在 `config/app.php` 設定檔中，其值通常使用 `APP_FALLBACK_LOCALE` 環境變數來設定。

您可以使用 `App` Facade 提供的 `setLocale` 方法，在執行時為單一 HTTP 請求修改預設語系：

    use Illuminate\Support\Facades\App;

    Route::get('/greeting/{locale}', function (string $locale) {
        if (! in_array($locale, ['en', 'es', 'fr'])) {
            abort(400);
        }

        App::setLocale($locale);

        // ...
    });

<a name="determining-the-current-locale"></a>
#### 判斷目前的語系

您可以使用 `App` Facade 上的 `currentLocale` 和 `isLocale` 方法來判斷目前的語系或檢查語系是否為給定值：

    use Illuminate\Support\Facades\App;

    $locale = App::currentLocale();

    if (App::isLocale('en')) {
        // ...
    }

<a name="pluralization-language"></a>
### 複數形語系

您可以指示 Laravel 的「複數形轉換器 (pluralizer)」（Eloquent 和框架其他部分用來將單數形字串轉換為複數形字串）使用英語以外的語言。這可以透過在應用程式其中一個 Service Provider 的 `boot` 方法中呼叫 `useLanguage` 方法來實現。複數形轉換器目前支援的語言有：`french`、`norwegian-bokmal`、`portuguese`、`spanish` 和 `turkish`：

    use Illuminate\Support\Pluralizer;

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        Pluralizer::useLanguage('spanish');

        // ...
    }

> [!WARNING]  
> 如果您自訂了複數形轉換器的語言，您應該明確定義您的 Eloquent Model 的[資料表名稱](/docs/{{version}}/eloquent#table-names)。

<a name="defining-translation-strings"></a>
## 定義翻譯字串

<a name="using-short-keys"></a>
### 使用短鍵

通常，翻譯字串儲存在 `lang` 目錄中的檔案裡。在這個目錄中，應該為應用程式支援的每種語言建立一個子目錄。這是 Laravel 用來管理內建 Laravel 功能（例如驗證錯誤訊息）翻譯字串的方法：

    /lang
        /en
            messages.php
        /es
            messages.php

所有語系檔案都回傳一個鍵值字串陣列。例如：

    <?php

    // lang/en/messages.php

    return [
        'welcome' => 'Welcome to our application!',
    ];

> [!WARNING]  
> 對於因地區而異的語言，您應該根據 ISO 15897 命名語言目錄。例如，英式英語應使用「en_GB」而不是「en-gb」。

<a name="using-translation-strings-as-keys"></a>
### 使用翻譯字串作為鍵

對於擁有大量可翻譯字串的應用程式，在 View 中使用「短鍵」來定義每個字串可能會變得混亂，而且為應用程式支援的每個翻譯字串不斷發明鍵值也很麻煩。

因此，Laravel 也支援使用字串的「預設」翻譯作為鍵來定義翻譯字串。使用翻譯字串作為鍵的語系檔案以 JSON 檔案的形式儲存在 `lang` 目錄中。例如，如果您的應用程式有西班牙語翻譯，您應該建立一個 `lang/es.json` 檔案：

```json
{
    "I love programming.": "Me encanta programar."
}
```

#### 鍵 / 檔案衝突

您不應該定義與其他翻譯檔案名稱衝突的翻譯字串鍵。例如，如果 `nl/action.php` 檔案存在但 `nl.json` 檔案不存在，則為「NL」語系翻譯 `__('Action')` 將導致翻譯器回傳 `nl/action.php` 的全部內容。

<a name="retrieving-translation-strings"></a>
## 取得翻譯字串

您可以使用 `__` 輔助函式從語系檔案中取得翻譯字串。如果您使用「短鍵」來定義翻譯字串，您應該使用「點」語法將包含鍵的檔案和鍵本身傳遞給 `__` 函式。例如，讓我們從 `lang/en/messages.php` 語系檔案中取得 `welcome` 翻譯字串：

    echo __('messages.welcome');

如果指定的翻譯字串不存在，`__` 函式將回傳翻譯字串鍵。因此，使用上面的範例，如果翻譯字串不存在，`__` 函式將回傳 `messages.welcome`。

如果您使用[預設翻譯字串作為翻譯鍵](#using-translation-strings-as-keys)，您應該將字串的預設翻譯傳遞給 `__` 函式；

    echo __('I love programming.');

同樣，如果翻譯字串不存在，`__` 函式將回傳給定的翻譯字串鍵。

如果您使用 [Blade 模板引擎](/docs/{{version}}/blade)，您可以使用 `{{ }}` 輸出語法來顯示翻譯字串：

    {{ __('messages.welcome') }}

<a name="replacing-parameters-in-translation-strings"></a>
### 替換翻譯字串中的參數

如果您願意，可以在翻譯字串中定義佔位符。所有佔位符都以 `:` 為前綴。例如，您可以定義一個帶有佔位符名稱的歡迎訊息：

    'welcome' => 'Welcome, :name',

要在取得翻譯字串時替換佔位符，您可以將替換陣列作為第二個參數傳遞給 `__` 函式：

    echo __('messages.welcome', ['name' => 'dayle']);

如果您的佔位符包含所有大寫字母，或者只有第一個字母大寫，則翻譯後的值將相應地大寫：

    'welcome' => 'Welcome, :NAME', // Welcome, DAYLE
    'goodbye' => 'Goodbye, :Name', // Goodbye, Dayle

<a name="object-replacement-formatting"></a>
#### 物件替換格式化

如果您嘗試提供一個物件作為翻譯佔位符，將會呼叫該物件的 `__toString` 方法。[`__toString`](https://www.php.net/manual/en/language.oop5.magic.php#object.tostring) 方法是 PHP 內建的「魔術方法」之一。然而，有時您可能無法控制給定類別的 `__toString` 方法，例如當您正在互動的類別屬於第三方函式庫時。

在這些情況下，Laravel 允許您為該特定類型的物件註冊一個自訂格式處理器。為此，您應該呼叫翻譯器的 `stringable` 方法。`stringable` 方法接受一個閉包，該閉包應該型別提示它負責格式化的物件類型。通常，`stringable` 方法應該在應用程式 `AppServiceProvider` 類別的 `boot` 方法中呼叫：

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

<a name="pluralization"></a>
### 複數形

複數形是一個複雜的問題，因為不同的語言有各種複雜的複數形規則；然而，Laravel 可以幫助您根據您定義的複數形規則來翻譯不同的字串。使用 `|` 字元，您可以區分字串的單數形和複數形：

    'apples' => 'There is one apple|There are many apples',

當然，當使用[翻譯字串作為鍵](#using-translation-strings-as-keys)時，也支援複數形：

```json
{
    "There is one apple|There are many apples": "Hay una manzana|Hay muchas manzanas"
}
```

您甚至可以建立更複雜的複數形規則，為多個值範圍指定翻譯字串：

    'apples' => '{0} There are none|[1,19] There are some|[20,*] There are many',

定義了具有複數形選項的翻譯字串後，您可以使用 `trans_choice` 函式來取得給定「計數」的行。在此範例中，由於計數大於一，因此回傳翻譯字串的複數形：

    echo trans_choice('messages.apples', 10);

您也可以在複數形字串中定義佔位符屬性。這些佔位符可以透過將陣列作為第三個參數傳遞給 `trans_choice` 函式來替換：

    'minutes_ago' => '{1} :value minute ago|[2,*] :value minutes ago',

    echo trans_choice('time.minutes_ago', 5, ['value' => 5]);

如果您想顯示傳遞給 `trans_choice` 函式的整數值，您可以使用內建的 `:count` 佔位符：

    'apples' => '{0} There are none|{1} There is one|[2,*] There are :count',

<a name="overriding-package-language-files"></a>
## 覆寫套件語系檔案

有些套件可能附帶自己的語系檔案。您無需更改套件的核心檔案來調整這些行，而是可以透過將檔案放置在 `lang/vendor/{package}/{locale}` 目錄中來覆寫它們。

因此，例如，如果您需要覆寫名為 `skyrim/hearthfire` 的套件中 `messages.php` 的英文翻譯字串，您應該將語系檔案放置在：`lang/vendor/hearthfire/en/messages.php`。在此檔案中，您應該只定義您希望覆寫的翻譯字串。任何您未覆寫的翻譯字串仍將從套件的原始語系檔案中載入。

