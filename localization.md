# 本地化

- [簡介](#introduction)
    - [發佈語言檔案](#publishing-the-language-files)
    - [設定語系](#configuring-the-locale)
    - [複數形式語言](#pluralization-language)
- [定義翻譯字串](#defining-translation-strings)
    - [使用簡短鍵](#using-short-keys)
    - [使用翻譯字串作為鍵](#using-translation-strings-as-keys)
- [擷取翻譯字串](#retrieving-translation-strings)
    - [取代翻譯字串中的參數](#replacing-parameters-in-translation-strings)
    - [複數形式](#pluralization)
- [覆寫套件語言檔案](#overriding-package-language-files)

<a name="introduction"></a>
## 簡介

> [!NOTE]  
> 預設情況下，Laravel 應用程式骨架不包含 `lang` 目錄。如果您想自訂 Laravel 的語言檔案，您可以透過 `lang:publish` Artisan 指令來發佈它們。

Laravel 的本地化功能提供了一種方便的方式來擷取各種語言的字串，讓您能夠在應用程式中輕鬆支援多國語言。

Laravel 提供了兩種管理翻譯字串的方式。首先，語言字串可以儲存在應用程式 `lang` 目錄中的檔案內。在這個目錄中，可以為應用程式支援的每種語言建立子目錄。這是 Laravel 用於管理內建 Laravel 功能（例如驗證錯誤訊息）的翻譯字串的方法：

    /lang
        /en
            messages.php
        /es
            messages.php

或者，翻譯字串可以定義在放置於 `lang` 目錄中的 JSON 檔案內。採用這種方法時，您的應用程式支援的每種語言都將在這個目錄中擁有一個對應的 JSON 檔案。這種方法推薦用於擁有大量可翻譯字串的應用程式：

    /lang
        en.json
        es.json

我們將在這份文件裡討論每種管理翻譯字串的方法。

<a name="publishing-the-language-files"></a>
### 發佈語言檔案

預設情況下，Laravel 應用程式骨架不包含 `lang` 目錄。如果您想自訂 Laravel 的語言檔案或建立自己的語言檔案，您應該透過 `lang:publish` Artisan 指令來建構 `lang` 目錄。`lang:publish` 指令將在您的應用程式中建立 `lang` 目錄並發佈 Laravel 使用的預設語言檔案集：

```shell
php artisan lang:publish
```

<a name="configuring-the-locale"></a>
### 設定語系

應用程式的預設語言儲存在 `config/app.php` 設定檔的 `locale` 設定選項中，通常透過 `APP_LOCALE` 環境變數設定。您可以自由修改此值以適應您的應用程式需求。

您也可以設定「備用語言」，當預設語言不包含給定的翻譯字串時，將使用備用語言。與預設語言一樣，備用語言也設定在 `config/app.php` 設定檔中，其值通常透過 `APP_FALLBACK_LOCALE` 環境變數設定。

您可以使用 `App` Facade 提供的 `setLocale` 方法，在執行時期為單一 HTTP 請求修改預設語言：

    use Illuminate\Support\Facades\App;

    Route::get('/greeting/{locale}', function (string $locale) {
        if (! in_array($locale, ['en', 'es', 'fr'])) {
            abort(400);
        }

        App::setLocale($locale);

        // ...
    });

<a name="determining-the-current-locale"></a>
#### 判斷當前語系

您可以使用 `App` Facade 上的 `currentLocale` 和 `isLocale` 方法來判斷當前語系或檢查語系是否為給定值：

    use Illuminate\Support\Facades\App;

    $locale = App::currentLocale();

    if (App::isLocale('en')) {
        // ...
    }

<a name="pluralization-language"></a>
### 複數形式語言

您可以指示 Laravel 的「複數器」（用於 Eloquent 和框架其他部分將單數形式字串轉換為複數形式字串）使用英語以外的語言。這可以透過在應用程式的其中一個服務提供者的 `boot` 方法中呼叫 `useLanguage` 方法來實現。目前複數器支援的語言有：`french`、`norwegian-bokmal`、`portuguese`、`spanish` 和 `turkish`：

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
> 如果您自訂了複數器的語言，您應該明確定義 Eloquent 模型中的[資料表名稱](/docs/{{version}}/eloquent#table-names)。

<a name="defining-translation-strings"></a>
## 定義翻譯字串

<a name="using-short-keys"></a>
### 使用簡短鍵

通常，翻譯字串儲存在 `lang` 目錄中的檔案內。在這個目錄中，應該為應用程式支援的每種語言建立一個子目錄。這是 Laravel 用於管理內建 Laravel 功能（例如驗證錯誤訊息）的翻譯字串的方法：

    /lang
        /en
            messages.php
        /es
            messages.php

所有語言檔案都回傳一個鍵值字串陣列。例如：

    <?php

    // lang/en/messages.php

    return [
        'welcome' => 'Welcome to our application!',
    ];

> [!WARNING]  
> 對於依地區而異的語言，您應該根據 ISO 15897 命名語言目錄。例如，英國英語應該使用「en_GB」而不是「en-gb」。

<a name="using-translation-strings-as-keys"></a>
### 使用翻譯字串作為鍵

對於擁有大量可翻譯字串的應用程式，在視圖中引用每個「簡短鍵」定義的字串可能會變得混亂，而且持續為應用程式支援的每個翻譯字串發明鍵名會很麻煩。

因此，Laravel 也支援使用字串的「預設」翻譯作為鍵來定義翻譯字串。使用翻譯字串作為鍵的語言檔案儲存在 `lang` 目錄中的 JSON 檔案。例如，如果您的應用程式有西班牙語翻譯，您應該建立一個 `lang/es.json` 檔案：

```json
{
    "I love programming.": "Me encanta programar."
}
```

#### 鍵與檔案的衝突

您不應該定義與其他翻譯檔案名稱衝突的翻譯字串鍵。例如，如果 `nl/action.php` 檔案存在但 `nl.json` 檔案不存在，則為「NL」語系翻譯 `__('Action')` 將導致翻譯器回傳 `nl/action.php` 的全部內容。

<a name="retrieving-translation-strings"></a>
## 擷取翻譯字串

您可以使用 `__` 輔助函式從語言檔案中擷取翻譯字串。如果您使用「簡短鍵」來定義翻譯字串，您應該使用「點式語法」將包含該鍵的檔案和鍵本身傳遞給 `__` 函式。例如，讓我們從 `lang/en/messages.php` 語言檔案中擷取 `welcome` 翻譯字串：

    echo __('messages.welcome');

如果指定的翻譯字串不存在，`__` 函式將會回傳翻譯字串的鍵。因此，以上述範例來說，如果翻譯字串不存在，`__` 函式將回傳 `messages.welcome`。

如果您使用 [預設翻譯字串作為翻譯鍵](#using-translation-strings-as-keys)，您應該將字串的預設翻譯傳遞給 `__` 函式：

    echo __('I love programming.');

同樣地，如果翻譯字串不存在，`__` 函式將回傳它所獲得的翻譯字串鍵。

如果您使用 [Blade 樣板引擎](/docs/{{version}}/blade)，您可以使用 `{{ }}` echo 語法來顯示翻譯字串：

    {{ __('messages.welcome') }}


<a name="replacing-parameters-in-translation-strings"></a>
### 取代翻譯字串中的參數

如果您願意，可以在翻譯字串中定義預留位置。所有預留位置都以 `:` 為前綴。例如，您可以定義一個帶有名稱預留位置的歡迎訊息：

    'welcome' => 'Welcome, :name',

要在擷取翻譯字串時取代預留位置，您可以將一個替換陣列作為第二個參數傳遞給 `__` 函式：

    echo __('messages.welcome', ['name' => 'dayle']);

如果您的預留位置包含所有大寫字母，或只有第一個字母大寫，則翻譯值將會相應地大寫：

    'welcome' => 'Welcome, :NAME', // Welcome, DAYLE
    'goodbye' => 'Goodbye, :Name', // Goodbye, Dayle


<a name="object-replacement-formatting"></a>
#### 物件替換格式化

如果您嘗試提供一個物件作為翻譯預留位置，該物件的 `__toString` 方法將會被呼叫。 [`__toString`](https://www.php.net/manual/en/language.oop5.magic.php#object.tostring) 方法是 PHP 內建的「魔法方法」之一。然而，有時您可能無法控制給定類別的 `__toString` 方法，例如當您正在互動的類別屬於第三方函式庫時。

在這些情況下，Laravel 允許您為該特定型別的物件註冊一個自訂格式化處理器。為了實現這一點，您應該呼叫翻譯器 (translator) 的 `stringable` 方法。`stringable` 方法接受一個閉包，該閉包應該對其負責格式化的物件型別進行型別提示。通常，`stringable` 方法應該在您應用程式的 `AppServiceProvider` 類別的 `boot` 方法中被呼叫：

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
### 複數形式

複數形式是一個複雜的問題，因為不同的語言有各種複雜的複數規則；然而，Laravel 可以幫助您根據您定義的複數規則來區分翻譯字串。使用 `|` 字元，您可以區分字串的單數和複數形式：

    'apples' => 'There is one apple|There are many apples',

當然，當使用 [翻譯字串作為鍵](#using-translation-strings-as-keys) 時，也支援複數形式：

```json
{
    "There is one apple|There are many apples": "Hay una manzana|Hay muchas manzanas"
}
```

您甚至可以建立更複雜的複數規則，為多個值範圍指定翻譯字串：

    'apples' => '{0} There are none|[1,19] There are some|[20,*] There are many',

在定義了具有複數選項的翻譯字串後，您可以使用 `trans_choice` 函式來擷取指定「計數」的行。在此範例中，由於計數大於一，因此回傳翻譯字串的複數形式：

    echo trans_choice('messages.apples', 10);

您也可以在複數形式字串中定義預留位置屬性。這些預留位置可以透過將陣列作為第三個參數傳遞給 `trans_choice` 函式來替換：

    'minutes_ago' => '{1} :value minute ago|[2,*] :value minutes ago',

    echo trans_choice('time.minutes_ago', 5, ['value' => 5]);

如果您想顯示傳遞給 `trans_choice` 函式的整數值，可以使用內建的 `:count` 預留位置：

    'apples' => '{0} There are none|{1} There is one|[2,*] There are :count',


<a name="overriding-package-language-files"></a>
## 覆寫套件語言檔案

有些套件可能附帶自己的語言檔案。您不必更改套件的核心檔案來調整這些行，您可以透過將檔案放置在 `lang/vendor/{package}/{locale}` 目錄中來覆寫它們。

因此，舉例來說，如果您需要覆寫名為 `skyrim/hearthfire` 套件中 `messages.php` 的英文翻譯字串，您應該在 `lang/vendor/hearthfire/en/messages.php` 放置一個語言檔案。在此檔案中，您應該只定義您希望覆寫的翻譯字串。任何您沒有覆寫的翻譯字串仍將從套件的原始語言檔案中載入。