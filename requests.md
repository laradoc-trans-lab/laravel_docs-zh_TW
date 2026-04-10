# HTTP 請求

- [簡介](#introduction)
- [與請求互動](#interacting-with-the-request)
    - [存取請求](#accessing-the-request)
    - [請求路徑、主機與方法](#request-path-and-method)
    - [請求標頭](#request-headers)
    - [請求 IP 位址](#request-ip-address)
    - [內容協商](#content-negotiation)
    - [PSR-7 請求](#psr7-requests)
- [輸入](#input)
    - [取得輸入](#retrieving-input)
    - [輸入存在判定](#input-presence)
    - [合併額外輸入](#merging-additional-input)
    - [舊輸入](#old-input)
    - [Cookies](#cookies)
    - [輸入修剪與標準化](#input-trimming-and-normalization)
- [檔案](#files)
    - [取得上傳檔案](#retrieving-uploaded-files)
    - [儲存上傳檔案](#storing-uploaded-files)
- [設定受信任的代理伺服器](#configuring-trusted-proxies)
- [設定受信任的主機](#configuring-trusted-hosts)

<a name="introduction"></a>
## 簡介

Laravel 的 `Illuminate\Http\Request` 類別提供了一種物件導向的方式，讓您可以與目前由應用程式處理的 HTTP 請求進行互動，並取得隨請求提交的輸入、cookies 與檔案。

<a name="interacting-with-the-request"></a>
## 與請求互動


<a name="accessing-the-request"></a>
### 存取請求

若要透過依賴注入 (dependency injection) 取得目前 HTTP 請求的實例，您應該在您的路由閉包或控制器方法中對 `Illuminate\Http\Request` 類別進行型別提示 (type-hint)。傳入的請求實例將會由 Laravel [服務容器](/docs/{{version}}/container) 自動注入：

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

如前所述，您也可以在路由閉包中對 `Illuminate\Http\Request` 類別進行型別提示。服務容器會在閉包執行時，自動將傳入的請求注入到閉包中：

```php
use Illuminate\Http\Request;

Route::get('/', function (Request $request) {
    // ...
});
```


<a name="dependency-injection-route-parameters"></a>
#### 依賴注入與路由參數

如果您的控制器方法也預期從路由參數接收輸入，您應該將路由參數放在其他依賴項之後。例如，如果您的路由定義如下：

```php
use App\Http\Controllers\UserController;

Route::put('/user/{id}', [UserController::class, 'update']);
```

您仍然可以對 `Illuminate\Http\Request` 進行型別提示，並透過如下定義您的控制器方法來存取 `id` 路由參數：

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

`Illuminate\Http\Request` 實例提供了多種檢查傳入 HTTP 請求的方法，並繼承自 `Symfony\Component\HttpFoundation\Request` 類別。我們將在下方討論幾個最重要的方法。


<a name="retrieving-the-request-path"></a>
#### 取得請求路徑

`path` 方法會回傳請求的路徑資訊。因此，如果傳入的請求目標是 `http://example.com/foo/bar`，`path` 方法將回傳 `foo/bar`：

```php
$uri = $request->path();
```


<a name="inspecting-the-request-path"></a>
#### 檢查請求路徑 / 路由

`is` 方法允許您驗證傳入的請求路徑是否符合給定的模式。在使用此方法時，您可以使用 `*` 字元作為萬用字元：

```php
if ($request->is('admin/*')) {
    // ...
}
```

使用 `routeIs` 方法，您可以判定傳入的請求是否符合[具名路由](/docs/{{version}}/routing#named-routes)：

```php
if ($request->routeIs('admin.*')) {
    // ...
}
```


<a name="retrieving-the-request-url"></a>
#### 取得請求 URL

若要取得傳入請求的完整 URL，您可以使用 `url` 或 `fullUrl` 方法。`url` 方法將回傳不含查詢字串的 URL，而 `fullUrl` 方法則包含查詢字串：

```php
$url = $request->url();

$urlWithQueryString = $request->fullUrl();
```

如果您想將查詢字串資料附加到目前 URL，可以呼叫 `fullUrlWithQuery` 方法。此方法會將給定的查詢字串變數陣列與目前的查詢字串合併：

```php
$request->fullUrlWithQuery(['type' => 'phone']);
```

如果您想取得不含特定查詢字串參數的目前 URL，可以使用 `fullUrlWithoutQuery` 方法：

```php
$request->fullUrlWithoutQuery(['type']);
```


<a name="retrieving-the-request-host"></a>
#### 取得請求主機

您可以使用 `host`、`httpHost` 和 `schemeAndHttpHost` 方法來取得傳入請求的「主機 (host)」：

```php
$request->host();
$request->httpHost();
$request->schemeAndHttpHost();
```


<a name="retrieving-the-request-method"></a>
#### 取得請求方法

`method` 方法會回傳請求的 HTTP 動詞。您可以使用 `isMethod` 方法來驗證 HTTP 動詞是否符合給定的字串：

```php
$method = $request->method();

if ($request->isMethod('post')) {
    // ...
}
```


<a name="request-headers"></a>
### 請求標頭

您可以使用 `header` 方法從 `Illuminate\Http\Request` 實例中取得請求標頭。如果請求中不存在該標頭，將回傳 `null`。不過，`header` 方法接受一個可選的第二個引數，如果請求中不存在該標頭，則會回傳此值：

```php
$value = $request->header('X-Header-Name');

$value = $request->header('X-Header-Name', 'default');
```

`hasHeader` 方法可用於判定請求是否包含給定的標頭：

```php
if ($request->hasHeader('X-Header-Name')) {
    // ...
}
```

為了方便起見，可以使用 `bearerToken` 方法從 `Authorization` 標頭中取得 Bearer 令牌。如果不存在此標頭，將回傳空字串：

```php
$token = $request->bearerToken();
```


<a name="request-ip-address"></a>
### 請求 IP 位址

`ip` 方法可用於取得向您的應用程式發出請求之客戶端的 IP 位址：

```php
$ipAddress = $request->ip();
```

如果您想取得 IP 位址陣列（包含所有由代理伺服器轉發的客戶端 IP 位址），可以使用 `ips` 方法。「原始」客戶端 IP 位址將位於陣列的末端：

```php
$ipAddresses = $request->ips();
```

一般而言，IP 位址應被視為不可信且由使用者控制的輸入，僅應將其用於資訊參考目的。


<a name="content-negotiation"></a>
### 內容協商

Laravel 提供了多種方法，可透過 `Accept` 標頭檢查傳入請求所要求的內容類型。首先，`getAcceptableContentTypes` 方法將回傳一個包含請求所接受之所有內容類型的陣列：

```php
$contentTypes = $request->getAcceptableContentTypes();
```

`accepts` 方法接受一個內容類型陣列，如果其中任何內容類型被請求接受，則回傳 `true`；否則回傳 `false`：

```php
if ($request->accepts(['text/html', 'application/json'])) {
    // ...
}
```

您可以使用 `prefers` 方法來判定在給定的內容類型陣列中，請求最偏好哪一種內容類型。如果提供的內容類型都沒有被請求接受，將回傳 `null`：

```php
$preferred = $request->prefers(['text/html', 'application/json']);
```

由於許多應用程式僅提供 HTML 或 JSON，您可以使用 `expectsJson` 方法快速判定傳入請求是否預期 JSON 回應：

```php
if ($request->expectsJson()) {
    // ...
}
```

如果您需要判定請求是否特別偏好 Markdown，或者在其他內容類型中是否接受 Markdown（例如在為 AI 代理或其他消耗 Markdown 回應的客戶端提供服務時），可以使用 `wantsMarkdown` 和 `acceptsMarkdown` 方法：

```php
if ($request->wantsMarkdown()) {
    // The client's most preferred content type is text/markdown...
}

if ($request->acceptsMarkdown()) {
    // The client accepts Markdown responses...
}
```

<a name="psr7-requests"></a>
### PSR-7 請求

[PSR-7 標準](https://www.php-fig.org/psr/psr-7/) 定義了 HTTP 訊息的介面，包括請求與回應。如果您想要取得 PSR-7 請求的實例而非 Laravel 請求，您首先需要安裝幾個函式庫。Laravel 使用 *Symfony HTTP Message Bridge* 元件將典型的 Laravel 請求與回應轉換為相容於 PSR-7 的實作：

```shell
composer require symfony/psr-http-message-bridge
composer require nyholm/psr7
```

安裝這些函式庫後，您可以在路由閉包或控制器方法中對請求介面進行型別提示，以取得 PSR-7 請求：

```php
use Psr\Http\Message\ServerRequestInterface;

Route::get('/', function (ServerRequestInterface $request) {
    // ...
});
```

> [!NOTE]
> 如果您從路由或控制器回傳一個 PSR-7 回應實例，它將會被自動轉換回 Laravel 回應實例並由框架顯示。

<a name="input"></a>
## 輸入

<a name="retrieving-input"></a>
### 取得輸入


<a name="retrieving-all-input-data"></a>
#### 取得所有輸入資料

您可以使用 `all` 方法將所有傳入請求的輸入資料取得為一個 `array`。無論傳入請求是來自 HTML 表單還是 XHR 請求，都可以使用此方法：

```php
$input = $request->all();
```

使用 `collect` 方法，您可以將所有傳入請求的輸入資料取得為一個 [collection](/docs/{{version}}/collections)：

```php
$input = $request->collect();
```

`collect` 方法還允許您將傳入請求輸入的一個子集取得為集合：

```php
$request->collect('users')->each(function (string $user) {
    // ...
});
```


<a name="retrieving-an-input-value"></a>
#### 取得單一輸入值

透過幾個簡單的方法，您可以從 `Illuminate\Http\Request` 實例中存取所有使用者輸入，而無需擔心請求使用了哪種 HTTP 動詞。無論使用何種 HTTP 動詞，都可以使用 `input` 方法來取得使用者輸入：

```php
$name = $request->input('name');
```

您可以將預設值作為 `input` 方法的第二個引數傳入。如果請求中不存在所請求的輸入值，將會回傳此值：

```php
$name = $request->input('name', 'Sally');
```

在處理包含陣列輸入的表單時，請使用「點」符號表示法來存取陣列：

```php
$name = $request->input('products.0.name');

$names = $request->input('products.*.name');
```

您可以呼叫不帶任何引數的 `input` 方法，以將所有輸入值取得為關聯陣列：

```php
$input = $request->input();
```


<a name="retrieving-input-from-the-query-string"></a>
#### 從查詢字串取得輸入

雖然 `input` 方法會從整個請求內容（payload，包括查詢字串）中取得值，但 `query` 方法將僅從查詢字串中取得值：

```php
$name = $request->query('name');
```

如果請求的查詢字串值不存在，將回傳此方法的第二個引數：

```php
$name = $request->query('name', 'Helen');
```

您可以呼叫不帶任何引數的 `query` 方法，以將所有查詢字串值取得為關聯陣列：

```php
$query = $request->query();
```


<a name="retrieving-json-input-values"></a>
#### 取得 JSON 輸入值

當向您的應用程式發送 JSON 請求時，只要請求的 `Content-Type` 標頭正確設定為 `application/json`，您就可以透過 `input` 方法存取 JSON 資料。您甚至可以使用「點」符號語法來取得巢狀在 JSON 陣列 / 物件中的值：

```php
$name = $request->input('user.name');
```


<a name="retrieving-stringable-input-values"></a>
#### 取得可字串操作的輸入值

與其將請求的輸入資料取得為原生 `string`，您可以使用 `string` 方法將請求資料取得為 [Illuminate\Support\Stringable](/docs/{{version}}/strings) 的實例：

```php
$name = $request->string('name')->trim();
```


<a name="retrieving-integer-input-values"></a>
#### 取得整數輸入值

若要將輸入值取得為整數，您可以使用 `integer` 方法。此方法會嘗試將輸入值轉換為整數。如果輸入值不存在或轉換失敗，它將回傳您指定的預設值。這對於分頁或其他數字輸入特別有用：

```php
$perPage = $request->integer('per_page');
```


<a name="retrieving-boolean-input-values"></a>
#### 取得布林輸入值

在處理像核取方塊 (checkbox) 這樣的 HTML 元素時，您的應用程式可能會收到實際上是字串的「真值 (truthy)」值。例如 "true" 或 "on"。為了方便起見，您可以使用 `boolean` 方法將這些值取得為布林值。`boolean` 方法對於 1, "1", true, "true", "on", 和 "yes" 會回傳 `true`。所有其他值將回傳 `false`：

```php
$archived = $request->boolean('archived');
```


<a name="retrieving-array-input-values"></a>
#### 取得陣列輸入值

包含陣列的輸入值可以使用 `array` 方法取得。此方法將始終將輸入值轉換為陣列。如果請求中不包含指定名稱的輸入值，將回傳一個空陣列：

```php
$versions = $request->array('versions');
```


<a name="retrieving-date-input-values"></a>
#### 取得日期輸入值

為了方便起見，包含日期 / 時間的輸入值可以使用 `date` 方法取得為 Carbon 實例。如果請求中不包含指定名稱的輸入值，將回傳 `null`：

```php
$birthday = $request->date('birthday');
```

`date` 方法接收的第二和第三個引數可分別用於指定日期的格式和時區：

```php
$elapsed = $request->date('elapsed', '!H:i', 'Europe/Madrid');
```

如果輸入值存在但格式無效，將拋出 `InvalidArgumentException`；因此，建議您在呼叫 `date` 方法之前先驗證輸入。


<a name="retrieving-interval-input-values"></a>
#### 取得間隔輸入值

包含時長的輸入值可以使用 `interval` 方法取得為 `CarbonInterval` 實例。如果請求中不包含指定名稱的輸入值，將回傳 `null`：

```php
$duration = $request->interval('duration');
```

如果輸入值是數字，您可以在第二個引數中提供單位。單位可以是字串（如 `second`、`minute` 或 `day`），或者是 `Carbon\Unit` 列舉實例：

```php
use Carbon\Unit;

$timeout = $request->interval('timeout', 'second');

$delay = $request->interval('delay', Unit::Minute);
```

如果輸入值存在但格式無效，將拋出 `InvalidArgumentException`；因此，建議您在呼叫 `interval` 方法之前先驗證輸入。


<a name="retrieving-enum-input-values"></a>
#### 取得列舉輸入值

對應於 [PHP 列舉 (enums)](https://www.php.net/manual/en/language.types.enumerations.php) 的輸入值也可以從請求中取得。如果請求中不包含指定名稱的輸入值，或者列舉沒有與輸入值相匹配的後備值 (backing value)，將回傳 `null`。`enum` 方法將輸入值的名稱和列舉類別分別作為其第一個和第二個引數：

```php
use App\Enums\Status;

$status = $request->enum('status', Status::class);
```

您也可以提供一個預設值，如果在缺少值或值無效時將回傳該預設值：

```php
$status = $request->enum('status', Status::class, Status::Pending);
```

如果輸入值是一個對應於 PHP 列舉的值陣列，您可以使用 `enums` 方法將該值陣列取得為列舉實例：

```php
use App\Enums\Product;

$products = $request->enums('products', Product::class);
```


<a name="retrieving-input-via-dynamic-properties"></a>
#### 透過動態屬性取得輸入

您也可以使用 `Illuminate\Http\Request` 實例上的動態屬性來存取使用者輸入。例如，如果您的應用程式表單中包含一個 `name` 欄位，您可以像這樣存取該欄位的值：

```php
$name = $request->name;
```

當使用動態屬性時，Laravel 會首先在請求內容 (payload) 中尋找參數的值。如果不存在，Laravel 將在匹配路由的參數中搜尋該欄位。


<a name="retrieving-a-portion-of-the-input-data"></a>
#### 取得部分輸入資料

如果您需要取得輸入資料的一個子集，可以使用 `only` 和 `except` 方法。這兩個方法都接收單個 `array` 或動態的引數列表：

```php
$input = $request->only(['username', 'password']);

$input = $request->only('username', 'password');

$input = $request->except(['credit_card']);

$input = $request->except('credit_card');
```

> [!WARNING]
> `only` 方法會回傳您請求的所有鍵 / 值對；然而，它不會回傳請求中不存在的鍵 / 值對。

<a name="input-presence"></a>
### 輸入存在判定

您可以使用 `has` 方法來判定某個值是否存在於請求中。如果該值存在於請求中，`has` 方法將回傳 `true`：

```php
if ($request->has('name')) {
    // ...
}
```

當傳入一個陣列時，`has` 方法會判定所有指定的值是否全部都存在：

```php
if ($request->has(['name', 'email'])) {
    // ...
}
```

`hasAny` 方法在指定的值中只要有任何一個存在，就會回傳 `true`：

```php
if ($request->hasAny(['name', 'email'])) {
    // ...
}
```

`whenHas` 方法會在某個值存在於請求中時，執行給定的閉包：

```php
$request->whenHas('name', function (string $input) {
    // ...
});
```

您也可以傳遞第二個閉包給 `whenHas` 方法，當指定的值不存在於請求中時，就會執行該閉包：

```php
$request->whenHas('name', function (string $input) {
    // The "name" value is present...
}, function () {
    // The "name" value is not present...
});
```

如果您想判定某個值是否存在於請求中，且該值不是空字串，可以使用 `filled` 方法：

```php
if ($request->filled('name')) {
    // ...
}
```

如果您想判定某個值在請求中缺失或是空字串，可以使用 `isNotFilled` 方法：

```php
if ($request->isNotFilled('name')) {
    // ...
}
```

當傳入一個陣列時，`isNotFilled` 方法會判定所有指定的值是否全部都缺失或為空：

```php
if ($request->isNotFilled(['name', 'email'])) {
    // ...
}
```

`anyFilled` 方法在指定的值中只要有任何一個不是空字串，就會回傳 `true`：

```php
if ($request->anyFilled(['name', 'email'])) {
    // ...
}
```

`whenFilled` 方法會在某個值存在於請求中且不是空字串時，執行給定的閉包：

```php
$request->whenFilled('name', function (string $input) {
    // ...
});
```

您也可以傳遞第二個閉包給 `whenFilled` 方法，當指定的值沒有被「填充 (filled)」時，就會執行該閉包：

```php
$request->whenFilled('name', function (string $input) {
    // The "name" value is filled...
}, function () {
    // The "name" value is not filled...
});
```

要判定某個給定的鍵是否在請求中缺失，您可以使用 `missing` 和 `whenMissing` 方法：

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

有時候您可能需要手動將額外的輸入合併到請求現有的輸入資料中。為了實現這一點，您可以使用 `merge` 方法。如果給定的輸入鍵已經存在於請求中，它將會被傳遞給 `merge` 方法的資料所覆蓋：

```php
$request->merge(['votes' => 0]);
```

`mergeIfMissing` 方法可用於在對應的鍵尚未存在於請求輸入資料中時，將輸入合併到請求中：

```php
$request->mergeIfMissing(['votes' => 0]);
```


<a name="old-input"></a>
### 舊輸入

Laravel 允許您在下一次請求中保留上一次請求的輸入。這項功能在偵測到驗證錯誤後重新填充表單時特別有用。然而，如果您使用 Laravel 內建的 [驗證功能](/docs/{{version}}/validation)，您可能不需要直接手動使用這些會話輸入快閃 (flashing) 方法，因為 Laravel 的一些內建驗證機制會自動呼叫它們。


<a name="flashing-input-to-the-session"></a>
#### 將輸入快閃至會話

`Illuminate\Http\Request` 類別中的 `flash` 方法會將目前的輸入快閃至 [會話 (session)](/docs/{{version}}/session)，以便在使用者下一次請求應用程式時可以使用：

```php
$request->flash();
```

您也可以使用 `flashOnly` 和 `flashExcept` 方法將請求資料的子集快閃至會話。這些方法可用於將密碼等敏感資訊排除在會話之外：

```php
$request->flashOnly(['username', 'email']);

$request->flashExcept('password');
```


<a name="flashing-input-then-redirecting"></a>
#### 快閃輸入後重新導向

由於您通常會希望將輸入快閃至會話然後重新導向回前一個頁面，您可以使用 `withInput` 方法輕鬆地將輸入快閃鏈接至重新導向：

```php
return redirect('/form')->withInput();

return redirect()->route('user.create')->withInput();

return redirect('/form')->withInput(
    $request->except('password')
);
```


<a name="retrieving-old-input"></a>
#### 取得舊輸入

要從上一次請求中取得快閃的輸入，請在 `Illuminate\Http\Request` 實例上呼叫 `old` 方法。`old` 方法會從 [會話 (session)](/docs/{{version}}/session) 中提取先前快閃的輸入資料：

```php
$username = $request->old('username');
```

Laravel 還提供了一個全域的 `old` 輔助函數。如果您是在 [Blade 模板](/docs/{{version}}/blade) 中顯示舊輸入，使用 `old` 輔助函數來重新填充表單會更方便。如果給定欄位沒有舊輸入，將回傳 `null`：

```blade
<input type="text" name="username" value="{{ old('username') }}">
```


<a name="cookies"></a>
### Cookies


<a name="retrieving-cookies-from-requests"></a>
#### 從請求中取得 Cookies

所有由 Laravel 框架建立的 Cookies 都經過加密並使用認證碼簽署，這意味著如果它們被客戶端修改，將被視為無效。要從請求中取得 Cookie 值，請在 `Illuminate\Http\Request` 實例上使用 `cookie` 方法：

```php
$value = $request->cookie('name');
```

<a name="input-trimming-and-normalization"></a>
## 輸入修剪與標準化

預設情況下，Laravel 在應用程式的全域中介層堆疊中包含了 `Illuminate\Foundation\Http\Middleware\TrimStrings` 與 `Illuminate\Foundation\Http\Middleware\ConvertEmptyStringsToNull` 中介層。這些中介層會自動修剪請求中所有傳入的字串欄位，並將任何空字串欄位轉換為 `null`。這讓您在定義路由與控制器時，不必擔心這些標準化的問題。


#### 停用輸入標準化

如果您希望針對所有請求停用此行為，可以在應用程式的 `bootstrap/app.php` 檔案中呼叫 `$middleware->remove` 方法，將這兩個中介層從應用程式的中介層堆疊中移除：

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

如果您希望針對應用程式的部分請求停用字串修剪與空字串轉換，可以在應用程式的 `bootstrap/app.php` 檔案中使用 `trimStrings` 與 `convertEmptyStringsToNull` 中介層方法。這兩個方法都接受一個閉包陣列，閉包應回傳 `true` 或 `false` 以指示是否應跳過輸入標準化：

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

您可以使用 `file` 方法或使用動態屬性從 `Illuminate\Http\Request` 實例中取得上傳的檔案。`file` 方法會回傳一個 `Illuminate\Http\UploadedFile` 類別的實例，該類別擴展了 PHP 的 `SplFileInfo` 類別，並提供了多種與檔案互動的方法：

```php
$file = $request->file('photo');

$file = $request->photo;
```

您可以使用 `hasFile` 方法來判斷請求中是否存在某個檔案：

```php
if ($request->hasFile('photo')) {
    // ...
}
```


<a name="validating-successful-uploads"></a>
#### 驗證成功上傳

除了檢查檔案是否存在之外，您還可以使用 `isValid` 方法來驗證上傳檔案的過程中沒有發生問題：

```php
if ($request->file('photo')->isValid()) {
    // ...
}
```


<a name="file-paths-extensions"></a>
#### 檔案路徑與副檔名

`UploadedFile` 類別還包含可用於存取檔案完整路徑與其副檔名的方法。`extension` 方法會嘗試根據檔案內容來推測其副檔名。此副檔名可能與客戶端提供的副檔名不同：

```php
$path = $request->photo->path();

$extension = $request->photo->extension();
```


<a name="other-file-methods"></a>
#### 其他檔案方法

`UploadedFile` 實例還提供多種其他方法。請參閱 [該類別的 API 文件](https://github.com/symfony/symfony/blob/6.0/src/Symfony/Component/HttpFoundation/File/UploadedFile.php) 以獲取更多關於這些方法的資訊。


<a name="storing-uploaded-files"></a>
### 儲存上傳檔案

要儲存上傳的檔案，您通常會使用配置好的 [檔案系統](/docs/{{version}}/filesystem) 之一。`UploadedFile` 類別具有一個 `store` 方法，可將上傳的檔案移動到其中一個磁碟中，該磁碟可以是本地檔案系統上的位置或像 Amazon S3 這樣的雲端儲存位置。

`store` 方法接受一個相對於檔案系統配置根目錄的儲存路徑。此路徑不應包含檔名，因為系統會自動生成一個唯一 ID 作為檔名。

`store` 方法還接受一個可選的第二個引數，用於指定儲存檔案的磁碟名稱。該方法將回傳相對於磁碟根目錄的檔案路徑：

```php
$path = $request->photo->store('images');

$path = $request->photo->store('images', 's3');
```

如果您不希望自動生成檔名，可以使用 `storeAs` 方法，該方法接受路徑、檔名與磁碟名稱作為引數：

```php
$path = $request->photo->storeAs('images', 'filename.jpg');

$path = $request->photo->storeAs('images', 'filename.jpg', 's3');
```

> [!NOTE]
> 關於 Laravel 檔案儲存的更多資訊，請參閱完整的 [檔案儲存文件](/docs/{{version}}/filesystem)。


<a name="configuring-trusted-proxies"></a>
## 設定受信任的代理伺服器

當您的應用程式運行在會終止 TLS / SSL 憑證的負載平衡器後方時，您可能會發現使用 `url` 輔助函數時，應用程式有時不會生成 HTTPS 連結。這通常是因為您的應用程式是透過 80 端口接收來自負載平衡器的轉發流量，因此不知道應該生成安全連結。

要解決這個問題，您可以啟用 Laravel 應用程式中包含的 `Illuminate\Http\Middleware\TrustProxies` 中介層，這讓您可以快速自訂應用程式應信任的負載平衡器或代理伺服器。您應該在應用程式的 `bootstrap/app.php` 檔案中使用 `trustProxies` 中介層方法來指定受信任的代理伺服器：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->trustProxies(at: [
        '192.168.1.1',
        '10.0.0.0/8',
    ]);
})
```

除了設定受信任的代理伺服器之外，您還可以設定應受信任的代理伺服器標頭：

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
> 如果您使用 AWS Elastic Load Balancing，`headers` 的值應為 `Request::HEADER_X_FORWARDED_AWS_ELB`。如果您的負載平衡器使用 [RFC 7239](https://www.rfc-editor.org/rfc/rfc7239#section-4) 的標準 `Forwarded` 標頭，則 `headers` 的值應為 `Request::HEADER_FORWARDED`。關於可用於 `headers` 值的常數之更多資訊，請參閱 Symfony 關於 [信任代理伺服器](https://symfony.com/doc/current/deployment/proxies.html) 的文件。


<a name="trusting-all-proxies"></a>
#### 信任所有代理伺服器

如果您使用 Amazon AWS 或其他「雲端」負載平衡器提供商，您可能不知道實際平衡器的 IP 位址。在這種情況下，您可以使用 `*` 來信任所有代理伺服器：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->trustProxies(at: '*');
})
```

<a name="configuring-trusted-hosts"></a>
## 設定受信任的主機

預設情況下，Laravel 會回應它接收到的所有請求，而不管 HTTP 請求的 `Host` 標頭內容為何。此外，在 Web 請求期間產生指向您應用程式的絕對 URL 時，會使用 `Host` 標頭的值。

通常，您應該設定您的 Web 伺服器（例如 Nginx 或 Apache），使其僅將符合指定主機名稱的請求發送到您的應用程式。然而，如果您無法直接自訂 Web 伺服器，且需要指示 Laravel 僅回應特定的主機名稱，您可以透過為應用程式啟用 `Illuminate\Http\Middleware\TrustHosts` 中介層來實現。

要啟用 `TrustHosts` 中介層，您應該在應用程式的 `bootstrap/app.php` 檔案中呼叫 `trustHosts` 中介層方法。您可以使用此方法的 `at` 引數來指定您的應用程式應該回應的主機名稱。主機名稱字串會被視為正規表達式。具有其他 `Host` 標頭的傳入請求將被拒絕：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->trustHosts(at: ['^laravel\.test$']);
})
```

預設情況下，來自應用程式 URL 子網域的請求也會自動被信任。如果您想要停用此行為，可以使用 `subdomains` 引數：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->trustHosts(at: ['^laravel\.test$'], subdomains: false);
})
```

如果您需要存取應用程式的設定檔或資料庫來決定受信任的主機，您可以為 `at` 引數提供一個閉包：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->trustHosts(at: fn () => config('app.trusted_hosts'));
})
```