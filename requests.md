# HTTP Requests

- [介紹](#introduction)
- [與請求互動](#interacting-with-the-request)
    - [存取請求](#accessing-the-request)
    - [請求路徑、主機與方法](#request-path-and-method)
    - [請求標頭](#request-headers)
    - [請求 IP 位址](#request-ip-address)
    - [內容協商](#content-negotiation)
    - [PSR-7 請求](#psr7-requests)
- [輸入](#input)
    - [取得輸入](#retrieving-input)
    - [輸入存在性](#input-presence)
    - [合併額外輸入](#merging-additional-input)
    - [舊有輸入](#old-input)
    - [Cookies](#cookies)
    - [輸入修剪與正規化](#input-trimming-and-normalization)
- [檔案](#files)
    - [取得上傳檔案](#retrieving-uploaded-files)
    - [儲存上傳檔案](#storing-uploaded-files)
- [設定信任的 Proxy](#configuring-trusted-proxies)
- [設定信任的主機](#configuring-trusted-hosts)

<a name="introduction"></a>
## 介紹

Laravel 的 `Illuminate\Http\Request` 類別提供了一種物件導向的方式，用於與應用程式正在處理的當前 HTTP 請求進行互動，以及取得隨該請求提交的輸入、Cookies 和檔案。

<a name="interacting-with-the-request"></a>
## 與請求互動


<a name="accessing-the-request"></a>
### 存取請求

要透過依賴注入取得目前的 HTTP 請求實例，您應在路由閉包或控制器方法上型別提示 `Illuminate\Http\Request` 類別。傳入的請求實例將由 Laravel [服務容器](/docs/{{version}}/container)自動注入：

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;

class UserController extends Controller
{
    /**
     * Store a new user.
     */
    public function store(Request $request): RedirectResponse
    {
        $name = $request->input('name');

        // Store the user...

        return redirect('/users');
    }
}
```

如前所述，您也可以在路由閉包上型別提示 `Illuminate\Http\Request` 類別。服務容器將在閉包執行時自動注入傳入的請求：

```php
use Illuminate\Http\Request;

Route::get('/', function (Request $request) {
    // ...
});
```


<a name="dependency-injection-route-parameters"></a>
#### 依賴注入與路由參數

如果您的控制器方法也預期從路由參數取得輸入，您應該在其他依賴之後列出您的路由參數。例如，如果您的路由定義如下：

```php
use App\Http\Controllers\UserController;

Route::put('/user/{id}', [UserController::class, 'update']);
```

您仍然可以型別提示 `Illuminate\Http\Request` 並透過以下方式定義您的控制器方法來存取您的 `id` 路由參數：

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;

class UserController extends Controller
{
    /**
     * Update the specified user.
     */
    public function update(Request $request, string $id): RedirectResponse
    {
        // Update the user...

        return redirect('/users');
    }
}
```


<a name="request-path-and-method"></a>
### 請求路徑、主機與方法

`Illuminate\Http\Request` 實例提供了多種方法來檢查傳入的 HTTP 請求，並繼承了 `Symfony\Component\HttpFoundation\Request` 類別。我們將在下面討論一些最重要的方法。


<a name="retrieving-the-request-path"></a>
#### 取得請求路徑

`path` 方法回傳請求的路徑資訊。因此，如果傳入請求的目標為 `http://example.com/foo/bar`，`path` 方法將回傳 `foo/bar`：

```php
$uri = $request->path();
```


<a name="inspecting-the-request-path"></a>
#### 檢查請求路徑 / 路由

`is` 方法允許您驗證傳入請求路徑是否符合指定模式。使用此方法時，您可以使用 `*` 字元作為萬用字元：

```php
if ($request->is('admin/*')) {
    // ...
}
```

使用 `routeIs` 方法，您可以判斷傳入請求是否符合 [具名路由](/docs/{{version}}/routing#named-routes)：

```php
if ($request->routeIs('admin.*')) {
    // ...
}
```


<a name="retrieving-the-request-url"></a>
#### 取得請求 URL

要取得傳入請求的完整 URL，您可以使用 `url` 或 `fullUrl` 方法。`url` 方法將回傳不包含查詢字串的 URL，而 `fullUrl` 方法則包含查詢字串：

```php
$url = $request->url();

$urlWithQueryString = $request->fullUrl();
```

如果您想將查詢字串資料附加到目前的 URL，您可以呼叫 `fullUrlWithQuery` 方法。此方法會將給定的查詢字串變數陣列與目前的查詢字串合併：

```php
$request->fullUrlWithQuery(['type' => 'phone']);
```

如果您想取得不含指定查詢字串參數的目前 URL，您可以使用 `fullUrlWithoutQuery` 方法：

```php
$request->fullUrlWithoutQuery(['type']);
```


<a name="retrieving-the-request-host"></a>
#### 取得請求主機

您可以透過 `host`、`httpHost` 和 `schemeAndHttpHost` 方法取得傳入請求的「主機」：

```php
$request->host();
$request->httpHost();
$request->schemeAndHttpHost();
```


<a name="retrieving-the-request-method"></a>
#### 取得請求方法

`method` 方法將回傳請求的 HTTP 動詞。您可以使用 `isMethod` 方法來驗證 HTTP 動詞是否符合給定的字串：

```php
$method = $request->method();

if ($request->isMethod('post')) {
    // ...
}
```


<a name="request-headers"></a>
### 請求標頭

您可以使用 `header` 方法從 `Illuminate\Http\Request` 實例中取得請求標頭。如果請求中不存在該標頭，將回傳 `null`。然而，`header` 方法接受一個可選的第二個參數，如果請求中不存在該標頭，則會回傳該參數值：

```php
$value = $request->header('X-Header-Name');

$value = $request->header('X-Header-Name', 'default');
```

`hasHeader` 方法可用於判斷請求是否包含指定的標頭：

```php
if ($request->hasHeader('X-Header-Name')) {
    // ...
}
```

為了方便，`bearerToken` 方法可用於從 `Authorization` 標頭中取得 bearer token。如果不存在此標頭，將回傳一個空字串：

```php
$token = $request->bearerToken();
```


<a name="request-ip-address"></a>
### 請求 IP 位址

`ip` 方法可用於取得向您的應用程式發出請求的客戶端 IP 位址：

```php
$ipAddress = $request->ip();
```

如果您想取得 IP 位址陣列，包含所有由代理伺服器轉發的客戶端 IP 位址，您可以使用 `ips` 方法。「原始」客戶端 IP 位址將位於陣列的末尾：

```php
$ipAddresses = $request->ips();
```

一般而言，IP 位址應被視為不可信、使用者控制的輸入，僅用於資訊目的。


<a name="content-negotiation"></a>
### 內容協商

Laravel 提供了多種方法，透過 `Accept` 標頭檢查傳入請求所要求的內容類型。首先，`getAcceptableContentTypes` 方法將回傳一個包含所有由請求接受的內容類型的陣列：

```php
$contentTypes = $request->getAcceptableContentTypes();
```

`accepts` 方法接受一個內容類型陣列，如果請求接受任何內容類型，則回傳 `true`。否則，將回傳 `false`：

```php
if ($request->accepts(['text/html', 'application/json'])) {
    // ...
}
```

您可以使用 `prefers` 方法來判斷在給定的內容類型陣列中，哪個內容類型是請求最偏好的。如果請求不接受任何提供的內容類型，則回傳 `null`：

```php
$preferred = $request->prefers(['text/html', 'application/json']);
```

由於許多應用程式僅提供 HTML 或 JSON，您可以使用 `expectsJson` 方法快速判斷傳入請求是否預期一個 JSON 回應：

```php
if ($request->expectsJson()) {
    // ...
}
```

<a name="psr7-requests"></a>
### PSR-7 請求

[PSR-7 標準](https://www.php-fig.org/psr/psr-7/)定義了 HTTP 訊息的介面，包括請求與回應。如果您想取得 PSR-7 請求的實例，而非 Laravel 請求的實例，您會需要先安裝一些函式庫。Laravel 使用 *Symfony HTTP Message Bridge* 元件將典型的 Laravel 請求與回應轉換為 PSR-7 相容的實作：

```shell
composer require symfony/psr-http-message-bridge
composer require nyholm/psr7
```

一旦您安裝了這些函式庫，您就可以透過在您的路由閉包或控制器方法上對請求介面進行型別提示來取得 PSR-7 請求：

```php
use Psr\Http\Message\ServerRequestInterface;

Route::get('/', function (ServerRequestInterface $request) {
    // ...
});
```

> [!NOTE]
> 如果您從路由或控制器回傳一個 PSR-7 回應實例，它將會自動轉換回 Laravel 回應實例，並由框架顯示。

<a name="input"></a>
## 輸入

<a name="retrieving-input"></a>
### 取得輸入

<a name="retrieving-all-input-data"></a>
#### 取得所有輸入資料

您可以使用 `all` 方法，將傳入請求的所有輸入資料擷取為 `array`。無論傳入請求是來自 HTML 表單還是 XHR 請求，都可以使用此方法：

```php
$input = $request->all();
```

您可以使用 `collect` 方法，將傳入請求的所有輸入資料擷取為 [collection](/docs/{{version}}/collections)：

```php
$input = $request->collect();
```

`collect` 方法也允許您將傳入請求的輸入子集擷取為 collection：

```php
$request->collect('users')->each(function (string $user) {
    // ...
});
```

<a name="retrieving-an-input-value"></a>
#### 取得單一輸入值

您可以使用一些簡單的方法，從 `Illuminate\Http\Request` 實例存取所有使用者輸入，而無需擔心請求使用了哪個 HTTP 動詞。無論 HTTP 動詞為何，`input` 方法都可以用來取得使用者輸入：

```php
$name = $request->input('name');
```

您可以將預設值作為第二個引數傳遞給 `input` 方法。如果請求中不存在所需的輸入值，則會回傳此值：

```php
$name = $request->input('name', 'Sally');
```

當處理包含陣列輸入的表單時，請使用「點標記法」來存取陣列：

```php
$name = $request->input('products.0.name');

$names = $request->input('products.*.name');
```

您可以不帶任何引數呼叫 `input` 方法，以取得所有輸入值作為關聯陣列：

```php
$input = $request->input();
```

<a name="retrieving-input-from-the-query-string"></a>
#### 從查詢字串取得輸入

雖然 `input` 方法會從整個請求酬載 (包括查詢字串) 擷取值，但 `query` 方法只會從查詢字串擷取值：

```php
$name = $request->query('name');
```

如果請求的查詢字串值資料不存在，則會回傳此方法的第二個引數：

```php
$name = $request->query('name', 'Helen');
```

您可以不帶任何引數呼叫 `query` 方法，以取得所有查詢字串值作為關聯陣列：

```php
$query = $request->query();
```

<a name="retrieving-json-input-values"></a>
#### 取得 JSON 輸入值

當向應用程式發送 JSON 請求時，只要請求的 `Content-Type` 標頭正確設定為 `application/json`，您就可以透過 `input` 方法存取 JSON 資料。您甚至可以使用「點標記法」來取得巢狀在 JSON 陣列/物件中的值：

```php
$name = $request->input('user.name');
```

<a name="retrieving-stringable-input-values"></a>
#### 取得可字串化的輸入值

您可以使用 `string` 方法將請求資料擷取為 [Illuminate\Support\Stringable](/docs/{{version}}/strings) 實例，而不是將其擷取為原始 `string`：

```php
$name = $request->string('name')->trim();
```

<a name="retrieving-integer-input-values"></a>
#### 取得整數輸入值

若要將輸入值擷取為整數，您可以使用 `integer` 方法。此方法會嘗試將輸入值轉換為整數。如果輸入不存在或轉換失敗，它將回傳您指定的預設值。這對於分頁或其他數字輸入特別有用：

```php
$perPage = $request->integer('per_page');
```

<a name="retrieving-boolean-input-values"></a>
#### 取得布林輸入值

當處理像核取方塊這樣的 HTML 元素時，您的應用程式可能會收到實際為字串的「真值」(truthy) 值。例如，「true」或「on」。為了方便起見，您可以使用 `boolean` 方法將這些值擷取為布林值。`boolean` 方法對於 1、「1」、true、「true」、「on」和「yes」回傳 `true`。所有其他值都會回傳 `false`：

```php
$archived = $request->boolean('archived');
```

<a name="retrieving-array-input-values"></a>
#### 取得陣列輸入值

包含陣列的輸入值可以使用 `array` 方法擷取。此方法會始終將輸入值轉換為陣列。如果請求不包含具有指定名稱的輸入值，則會回傳一個空陣列：

```php
$versions = $request->array('versions');
```

<a name="retrieving-date-input-values"></a>
#### 取得日期輸入值

為了方便起見，包含日期/時間的輸入值可以使用 `date` 方法擷取為 Carbon 實例。如果請求不包含具有指定名稱的輸入值，則會回傳 `null`：

```php
$birthday = $request->date('birthday');
```

`date` 方法接受的第二個和第三個引數可以用來分別指定日期的格式和時區：

```php
$elapsed = $request->date('elapsed', '!H:i', 'Europe/Madrid');
```

如果輸入值存在但格式無效，將會拋出 `InvalidArgumentException`；因此，建議您在呼叫 `date` 方法之前驗證輸入。

<a name="retrieving-enum-input-values"></a>
#### 取得 Enum 輸入值

屬於 [PHP enums](https://www.php.net/manual/en/language.types.enumerations.php) 的輸入值也可以從請求中擷取。如果請求不包含具有指定名稱的輸入值，或 enum 沒有與輸入值相符的基礎值，則會回傳 `null`。`enum` 方法接受輸入值的名稱和 enum 類別作為其第一個和第二個引數：

```php
use App\Enums\Status;

$status = $request->enum('status', Status::class);
```

您也可以提供預設值，如果該值遺失或無效，則會回傳此預設值：

```php
$status = $request->enum('status', Status::class, Status::Pending);
```

如果輸入值是與 PHP enum 對應的值陣列，您可以使用 `enums` 方法將該值陣列擷取為 enum 實例：

```php
use App\Enums\Product;

$products = $request->enums('products', Product::class);
```

<a name="retrieving-input-via-dynamic-properties"></a>
#### 透過動態屬性取得輸入

您也可以使用 `Illuminate\Http\Request` 實例上的動態屬性來存取使用者輸入。例如，如果應用程式的其中一個表單包含 `name` 欄位，您可以像這樣存取該欄位的值：

```php
$name = $request->name;
```

當使用動態屬性時，Laravel 會首先在請求酬載中尋找參數的值。如果不存在，Laravel 會在匹配的路由參數中尋找該欄位。

<a name="retrieving-a-portion-of-the-input-data"></a>
#### 取得部分輸入資料

如果您需要擷取輸入資料的子集，可以使用 `only` 和 `except` 方法。這兩種方法都接受單個 `array` 或動態引數列表：

```php
$input = $request->only(['username', 'password']);

$input = $request->only('username', 'password');

$input = $request->except(['credit_card']);

$input = $request->except('credit_card');
```

> [!WARNING]
> `only` 方法會回傳您請求的所有鍵/值對；但是，它不會回傳請求中不存在的鍵/值對。

<a name="input-presence"></a>
### 輸入存在性

您可以使用 `has` 方法來判斷請求中是否存在某個值。如果該值存在於請求中，`has` 方法將返回 `true`：

```php
if ($request->has('name')) {
    // ...
}
```

當傳入一個陣列時，`has` 方法將判斷所有指定的值是否存在：

```php
if ($request->has(['name', 'email'])) {
    // ...
}
```

如果任何指定的值存在，`hasAny` 方法會返回 `true`：

```php
if ($request->hasAny(['name', 'email'])) {
    // ...
}
```

如果請求中存在某個值，`whenHas` 方法將執行給定的閉包：

```php
$request->whenHas('name', function (string $input) {
    // ...
});
```

您可以傳遞第二個閉包給 `whenHas` 方法，如果指定的值不存在於請求中，該閉包將會被執行：

```php
$request->whenHas('name', function (string $input) {
    // The "name" value is present...
}, function () {
    // The "name" value is not present...
});
```

如果您想判斷請求中是否存在某個值且不為空字串，可以使用 `filled` 方法：

```php
if ($request->filled('name')) {
    // ...
}
```

如果您想判斷請求中是否缺少某個值或該值為空字串，可以使用 `isNotFilled` 方法：

```php
if ($request->isNotFilled('name')) {
    // ...
}
```

當傳入一個陣列時，`isNotFilled` 方法將判斷所有指定的值是否缺少或為空：

```php
if ($request->isNotFilled(['name', 'email'])) {
    // ...
}
```

如果任何指定的值不為空字串，`anyFilled` 方法會返回 `true`：

```php
if ($request->anyFilled(['name', 'email'])) {
    // ...
}
```

如果請求中存在某個值且不為空字串，`whenFilled` 方法將執行給定的閉包：

```php
$request->whenFilled('name', function (string $input) {
    // ...
});
```

您可以傳遞第二個閉包給 `whenFilled` 方法，如果指定的值不是「已填寫 (filled)」，該閉包將會被執行：

```php
$request->whenFilled('name', function (string $input) {
    // The "name" value is filled...
}, function () {
    // The "name" value is not filled...
});
```

要判斷請求中是否缺少給定的鍵，可以使用 `missing` 和 `whenMissing` 方法：

```php
if ($request->missing('name')) {
    // ...
}

$request->whenMissing('name', function () {
    // The "name" value is missing...
}, function () {
    // The "name" value is present...
});
```

<a name="merging-additional-input"></a>
### 合併額外輸入

有時您可能需要手動將額外輸入合併到請求現有的輸入資料中。為此，您可以使用 `merge` 方法。如果請求中已存在給定的輸入鍵，它將被提供給 `merge` 方法的資料所覆寫：

```php
$request->merge(['votes' => 0]);
```

如果請求的輸入資料中尚不存在對應的鍵，則可以使用 `mergeIfMissing` 方法將輸入合併到請求中：

```php
$request->mergeIfMissing(['votes' => 0]);
```

<a name="old-input"></a>
### 舊有輸入

Laravel 允許您在下一個請求期間保留來自前一個請求的輸入。此功能對於在偵測到驗證錯誤後重新填充表單特別有用。然而，如果您正在使用 Laravel 內建的 [驗證功能](/docs/{{version}}/validation)，您可能不需要直接手動使用這些 session 輸入快閃方法，因為 Laravel 的某些內建驗證設施會自動呼叫它們。

<a name="flashing-input-to-the-session"></a>
#### 快閃輸入至 Session

`Illuminate\Http\Request` 類別上的 `flash` 方法會將當前輸入快閃 (flash) 到 [session](/docs/{{version}}/session) 中，以便在使用者對應用程式進行下一個請求時可用：

```php
$request->flash();
```

您也可以使用 `flashOnly` 和 `flashExcept` 方法將請求資料的子集快閃到 session 中。這些方法對於將敏感資訊 (例如密碼) 保留不存入 session 中非常有用：

```php
$request->flashOnly(['username', 'email']);

$request->flashExcept('password');
```

<a name="flashing-input-then-redirecting"></a>
#### 快閃輸入然後重新導向

由於您通常會希望將輸入快閃到 session 中，然後重新導向到上一頁，因此您可以使用 `withInput` 方法輕鬆地將輸入快閃功能串接到重新導向：

```php
return redirect('/form')->withInput();

return redirect()->route('user.create')->withInput();

return redirect('/form')->withInput(
    $request->except('password')
);
```

<a name="retrieving-old-input"></a>
#### 取得舊有輸入

要從上一個請求中取得快閃輸入，請在 `Illuminate\Http\Request` 實例上呼叫 `old` 方法。`old` 方法將從 [session](/docs/{{version}}/session) 中提取之前快閃的輸入資料：

```php
$username = $request->old('username');
```

Laravel 也提供了一個全域的 `old` 輔助函式。如果您在 [Blade 模板](/docs/{{version}}/blade) 中顯示舊有輸入，使用 `old` 輔助函式來重新填充表單會更方便。如果指定欄位沒有舊有輸入，則會返回 `null`：

```blade
<input type="text" name="username" value="{{ old('username') }}">
```

<a name="cookies"></a>
### Cookies

<a name="retrieving-cookies-from-requests"></a>
#### 從請求中取得 Cookies

所有由 Laravel 框架建立的 cookie 都經過加密並使用驗證碼簽署，這表示如果它們被客戶端更改，將被視為無效。要從請求中取得 cookie 值，請在 `Illuminate\Http\Request` 實例上使用 `cookie` 方法：

```php
$value = $request->cookie('name');
```

<a name="input-trimming-and-normalization"></a>
## 輸入修剪與正規化

預設情況下，Laravel 在你的應用程式的全域中介層堆疊中包含了 `Illuminate\Foundation\Http\Middleware\TrimStrings` 和 `Illuminate\Foundation\Http\Middleware\ConvertEmptyStringsToNull` 中介層。這些中介層會自動修剪請求中所有傳入的字串欄位，並將任何空白字串欄位轉換為 `null`。這讓你無需在路由和控制器中擔心這些正規化問題。

#### 停用輸入正規化

如果你想為所有請求停用此行為，可以透過在應用程式的 `bootstrap/app.php` 檔案中呼叫 `$middleware->remove` 方法，從應用程式的中介層堆疊中移除這兩個中介層：

```php
use Illuminate\Foundation\Http\Middleware\ConvertEmptyStringsToNull;
use Illuminate\Foundation\Http\Middleware\TrimStrings;

->withMiddleware(function (Middleware $middleware): void {
    $middleware->remove([
        ConvertEmptyStringsToNull::class,
        TrimStrings::class,
    ]);
})
```

如果你想為應用程式的部分請求停用字串修剪和空白字串轉換，可以在應用程式的 `bootstrap/app.php` 檔案中使用 `trimStrings` 和 `convertEmptyStringsToNull` 中介層方法。這兩個方法都接受一個閉包陣列，這些閉包應回傳 `true` 或 `false` 以指示是否應跳過輸入正規化：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->convertEmptyStringsToNull(except: [
        fn (Request $request) => $request->is('admin/*'),
    ]);

    $middleware->trimStrings(except: [
        fn (Request $request) => $request->is('admin/*'),
    ]);
})
```

<a name="files"></a>
## 檔案

<a name="retrieving-uploaded-files"></a>
### 取得上傳檔案

你可以使用 `file` 方法或動態屬性從 `Illuminate\Http\Request` 實例中取得上傳的檔案。`file` 方法會回傳 `Illuminate\Http\UploadedFile` 類別的實例，該實例繼承了 PHP 的 `SplFileInfo` 類別，並提供多種與檔案互動的方法：

```php
$file = $request->file('photo');

$file = $request->photo;
```

你可以使用 `hasFile` 方法來判斷請求中是否存在檔案：

```php
if ($request->hasFile('photo')) {
    // ...
}
```

<a name="validating-successful-uploads"></a>
#### 驗證成功上傳

除了檢查檔案是否存在，你還可以透過 `isValid` 方法驗證檔案上傳過程中沒有發生問題：

```php
if ($request->file('photo')->isValid()) {
    // ...
}
```

<a name="file-paths-extensions"></a>
#### 檔案路徑與副檔名

`UploadedFile` 類別還包含用於存取檔案的完整路徑及其副檔名的方法。`extension` 方法會嘗試根據檔案內容猜測其副檔名。此副檔名可能與用戶端提供的副檔名不同：

```php
$path = $request->photo->path();

$extension = $request->photo->extension();
```

<a name="other-file-methods"></a>
#### 其他檔案方法

`UploadedFile` 實例上還有多種其他方法可用。請查閱[此類別的 API 文件](https://github.com/symfony/symfony/blob/6.0/src/Symfony/Component/HttpFoundation/File/UploadedFile.php)以獲取有關這些方法的更多資訊。

<a name="storing-uploaded-files"></a>
### 儲存上傳檔案

若要儲存上傳的檔案，你通常會使用已設定的[檔案系統](/docs/{{version}}/filesystem)之一。`UploadedFile` 類別具有一個 `store` 方法，它會將上傳的檔案移動到你的其中一個磁碟，該磁碟可以是本地檔案系統上的位置，或是像 Amazon S3 這樣的雲端儲存位置。

`store` 方法接受檔案應儲存的路徑，此路徑應相對於檔案系統已設定的根目錄。此路徑不應包含檔案名稱，因為會自動生成一個唯一 ID 作為檔案名稱。

`store` 方法還接受一個可選的第二個引數，用於指定應儲存檔案的磁碟名稱。該方法將回傳檔案相對於磁碟根目錄的路徑：

```php
$path = $request->photo->store('images');

$path = $request->photo->store('images', 's3');
```

如果你不希望自動生成檔案名稱，可以使用 `storeAs` 方法，該方法接受路徑、檔案名稱和磁碟名稱作為其引數：

```php
$path = $request->photo->storeAs('images', 'filename.jpg');

$path = $request->photo->storeAs('images', 'filename.jpg', 's3');
```

> [!NOTE]
> 有關 Laravel 中檔案儲存的更多資訊，請查閱完整的[檔案儲存文件](/docs/{{version}}/filesystem)。

<a name="configuring-trusted-proxies"></a>
## 設定信任的 Proxy

當你的應用程式在終止 TLS / SSL 憑證的負載平衡器後執行時，你可能會注意到你的應用程式有時在使用 `url` 輔助函式時不會生成 HTTPS 連結。這通常是因為你的應用程式正在從負載平衡器接收 80 埠的流量，且不知道它應該生成安全連結。

為了解決這個問題，你可以啟用 Laravel 應用程式中包含的 `Illuminate\Http\Middleware\TrustProxies` 中介層，這讓你能夠快速自訂你的應用程式應信任的負載平衡器或 Proxy。你的信任 Proxy 應透過應用程式的 `bootstrap/app.php` 檔案中的 `trustProxies` 中介層方法來指定：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->trustProxies(at: [
        '192.168.1.1',
        '10.0.0.0/8',
    ]);
})
```

除了設定信任的 Proxy，你還可以設定應信任的 Proxy 標頭：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->trustProxies(headers: Request::HEADER_X_FORWARDED_FOR |
        Request::HEADER_X_FORWARDED_HOST |
        Request::HEADER_X_FORWARDED_PORT |
        Request::HEADER_X_FORWARDED_PROTO |
        Request::HEADER_X_FORWARDED_AWS_ELB
    );
})
```

> [!NOTE]
> 如果你正在使用 AWS Elastic Load Balancing，`headers` 值應為 `Request::HEADER_X_FORWARDED_AWS_ELB`。如果你的負載平衡器使用 [RFC 7239](https://www.rfc-editor.org/rfc/rfc7239#section-4) 中的標準 `Forwarded` 標頭，`headers` 值應為 `Request::HEADER_FORWARDED`。有關 `headers` 值中可能使用的常數的更多資訊，請查閱 Symfony 關於[信任 Proxy](https://symfony.com/doc/current/deployment/proxies.html) 的文件。

<a name="trusting-all-proxies"></a>
#### 信任所有 Proxy

如果你正在使用 Amazon AWS 或其他「雲端」負載平衡器供應商，你可能不知道實際負載平衡器的 IP 位址。在這種情況下，你可以使用 `*` 來信任所有 Proxy：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->trustProxies(at: '*');
})
```

<a name="configuring-trusted-hosts"></a>
## 設定信任的主機

預設情況下，Laravel 將會回應所有收到的請求，無論 HTTP 請求的 `Host` 標頭內容為何。此外，在網頁請求期間，`Host` 標頭的值將會用於為你的應用程式產生絕對 URL。

通常，你應該設定你的網頁伺服器 (例如 Nginx 或 Apache)，使其只將符合指定主機名稱的請求傳送給你的應用程式。然而，如果你無法直接自訂你的網頁伺服器，並且需要指示 Laravel 只回應特定主機名稱，你可以透過為你的應用程式啟用 `Illuminate\Http\Middleware\TrustHosts` 中介層來達成。

若要啟用 `TrustHosts` 中介層，你應該在應用程式的 `bootstrap/app.php` 檔案中呼叫 `trustHosts` 中介層方法。使用此方法的 `at` 參數，你可以指定你的應用程式應該回應的主機名稱。傳入的帶有其他 `Host` 標頭的請求將會被拒絕：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->trustHosts(at: ['laravel.test']);
})
```

預設情況下，來自應用程式 URL 子網域的請求也會自動被信任。如果你想要停用此行為，你可以使用 `subdomains` 參數：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->trustHosts(at: ['laravel.test'], subdomains: false);
})
```

如果你需要存取應用程式的設定檔或資料庫以決定你的信任主機，你可以提供一個閉包給 `at` 參數：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->trustHosts(at: fn () => config('app.trusted_hosts'));
})
```