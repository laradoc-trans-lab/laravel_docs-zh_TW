# HTTP 請求

- [簡介](#introduction)
- [與請求互動](#interacting-with-the-request)
    - [取得請求](#accessing-the-request)
    - [請求路徑、主機與方法](#request-path-and-method)
    - [請求標頭](#request-headers)
    - [請求 IP 位址](#request-ip-address)
    - [內容協商](#content-negotiation)
    - [PSR-7 請求](#psr7-requests)
- [輸入資料](#input)
    - [取得輸入資料](#retrieving-input)
    - [確認輸入資料是否存在](#input-presence)
    - [合併額外的輸入資料](#merging-additional-input)
    - [舊輸入資料](#old-input)
    - [Cookie](#cookies)
    - [輸入資料修剪與規格化](#input-trimming-and-normalization)
- [檔案](#files)
    - [取得上傳的檔案](#retrieving-uploaded-files)
    - [儲存上傳的檔案](#storing-uploaded-files)
- [設定信任的代理伺服器](#configuring-trusted-proxies)
- [設定信任的主機](#configuring-trusted-hosts)

<a name="introduction"></a>
## 簡介

Laravel 的 `Illuminate\Http\Request` 類別提供了一種物件導向的方式，用於與應用程式正在處理的當前 HTTP 請求進行互動，並能取得隨請求送出的輸入資料、Cookie 和檔案。

<a name="interacting-with-the-request"></a>
## 與請求互動

<a name="accessing-the-request"></a>
### 取得請求

若要透過依賴注入取得當前 HTTP 請求的實例，你應該在路由 Closure 或控制器方法中對 `Illuminate\Http\Request` 類別進行型別提示 (type-hint)。傳入的請求實例將會自動由 Laravel [服務容器](/docs/{{version}}/container)注入：

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

如前所述，你也可以在路由 Closure 上對 `Illuminate\Http\Request` 類別進行型別提示。當 Closure 執行時，服務容器會自動將傳入的請求注入到 Closure 中：

```php
use Illuminate\Http\Request;

Route::get('/', function (Request $request) {
    // ...
});
```

<a name="dependency-injection-route-parameters"></a>
#### 依賴注入與路由參數

如果你的控制器方法同時期待接收來自路由參數的輸入資料，你應該將路由參數列在其他依賴項目之後。例如，如果你的路由定義如下：

```php
use App\Http\Controllers\UserController;

Route::put('/user/{id}', [UserController::class, 'update']);
```

你仍然可以對 `Illuminate\Http\Request` 進行型別提示，並透過如下定義控制器方法來存取你的 `id` 路由參數：

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

`Illuminate\Http\Request` 實例提供各種方法來檢視傳入的 HTTP 請求，並且繼承了 `Symfony\Component\HttpFoundation\Request` 類別。我們會在下方討論幾個最重要的方法。

<a name="retrieving-the-request-path"></a>
#### 取得請求路徑

`path` 方法會傳回請求的路徑資訊。因此，如果傳入的請求目標是 `http://example.com/foo/bar`，則 `path` 方法將傳回 `foo/bar`：

```php
$uri = $request->path();
```

<a name="inspecting-the-request-path"></a>
#### 檢查請求路徑／路由

`is` 方法允許你驗證傳入的請求路徑是否符合給定的模式。使用此方法時，你可以使用 `*` 字元作為萬用字元：

```php
if ($request->is('admin/*')) {
    // ...
}
```

使用 `routeIs` 方法，你可以確認傳入的請求是否符合指定的[具名路由](/docs/{{version}}/routing#named-routes)：

```php
if ($request->routeIs('admin.*')) {
    // ...
}
```

<a name="retrieving-the-request-url"></a>
#### 取得請求 URL

若要取得傳入請求的完整 URL，可以使用 `url` 或 `fullUrl` 方法。`url` 方法會傳回不帶查詢字串 (Query String) 的 URL，而 `fullUrl` 方法則包含查詢字串：

```php
$url = $request->url();

$urlWithQueryString = $request->fullUrl();
```

如果你想將查詢字串資料附加到當前 URL，可以呼叫 `fullUrlWithQuery` 方法。此方法會將給定的查詢字串變數陣列與當前查詢字串進行合併：

```php
$request->fullUrlWithQuery(['type' => 'phone']);
```

如果你想取得不包含特定查詢字串參數的當前 URL，可以使用 `fullUrlWithoutQuery` 方法：

```php
$request->fullUrlWithoutQuery(['type']);
```

<a name="retrieving-the-request-host"></a>
#### 取得請求主機

你可以透過 `host`、`httpHost` 及 `schemeAndHttpHost` 方法來取得傳入請求的「主機 (Host)」：

```php
// http://localhost:8000
$request->host(); // localhost
$request->httpHost(); // localhost:8000
$request->schemeAndHttpHost(); // http://localhost:8000
```

<a name="retrieving-the-request-method"></a>
#### 取得請求 HTTP 方法

`method` 方法會傳回請求的 HTTP 動詞。你可以使用 `isMethod` 方法來驗證 HTTP 動詞是否符合給定的字串：

```php
$method = $request->method();

if ($request->isMethod('post')) {
    // ...
}
```

<a name="request-headers"></a>
### 請求標頭

你可以使用 `header` 方法從 `Illuminate\Http\Request` 實例中取得請求標頭。如果請求中不存在該標頭，將傳回 `null`。不過，`header` 方法接受選填的第二個引數，若請求中不存在該標頭，則會傳回該值：

```php
$value = $request->header('X-Header-Name');

$value = $request->header('X-Header-Name', 'default');
```

`hasHeader` 方法可用於確定請求是否包含給定的標頭：

```php
if ($request->hasHeader('X-Header-Name')) {
    // ...
}
```

為方便起見，可以使用 `bearerToken` 方法從 `Authorization` 標頭中取得 Bearer 令牌。如果不存在此類標頭，將傳回空字串：

```php
$token = $request->bearerToken();
```

<a name="request-ip-address"></a>
### 請求 IP 位址

`ip` 方法可用於取得向你的應用程式發送請求的用戶端 IP 位址：

```php
$ipAddress = $request->ip();
```

如果你想取得包含由代理伺服器轉發的所有用戶端 IP 位址陣列，可以使用 `ips` 方法。「原始」用戶端 IP 位址將位於陣列的末端：

```php
$ipAddresses = $request->ips();
```

一般來說，IP 位址應被視為不可信任、由使用者控制的輸入，且僅用於資訊參考目的。

<a name="content-negotiation"></a>
### 內容協商

Laravel 提供多種方法，透過 `Accept` 標頭來檢視傳入請求所要求的內容型別。首先，`getAcceptableContentTypes` 方法會傳回一個陣列，包含該請求所接受的所有內容型別：

```php
$contentTypes = $request->getAcceptableContentTypes();
```

`accepts` 方法接受一個內容型別陣列，如果請求接受其中任何一個內容型別，則傳回 `true`。否則，將傳回 `false`：

```php
if ($request->accepts(['text/html', 'application/json'])) {
    // ...
}
```

你可以使用 `prefers` 方法來確定給定內容型別陣列中，請求最偏好哪一種內容型別。如果請求不接受任何提供的內容型別，將傳回 `null`：

```php
$preferred = $request->prefers(['text/html', 'application/json']);
```

由於許多應用程式僅提供 HTML 或 JSON，你可以使用 `expectsJson` 方法快速確定傳入的請求是否期待 JSON 回應：

```php
if ($request->expectsJson()) {
    // ...
}
```

如果你需要確定請求是否特別偏好 Markdown 或是否能在其他內容型別中接受 Markdown（例如在服務 AI 代理或其他消耗 Markdown 回應的用戶端時），可以使用 `wantsMarkdown` 與 `acceptsMarkdown` 方法：

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

[PSR-7 標準](https://www.php-fig.org/psr/psr-7/)規定了 HTTP 訊息的介面，包含請求與回應。若您想要取得 PSR-7 請求實例而非 Laravel 請求實例，您需要先安裝幾個套件庫。Laravel 使用 *Symfony HTTP Message Bridge* 元件將典型的 Laravel 請求與回應轉換為相容 PSR-7 的實作：

```shell
composer require symfony/psr-http-message-bridge
composer require nyholm/psr7
```

安裝這些套件庫後，您可以在路由閉包或控制器方法中，對請求介面進行型別提示來取得 PSR-7 請求：

```php
use Psr\Http\Message\ServerRequestInterface;

Route::get('/', function (ServerRequestInterface $request) {
    // ...
});
```

> [!NOTE]
> 如果您從路由或控制器回傳 PSR-7 回應實例，它將會自動轉換回 Laravel 回應實例並由框架顯示出來。

<a name="input"></a>
## 輸入資料

<a name="retrieving-input"></a>
### 取得輸入資料


<a name="retrieving-all-input-data"></a>
#### 取得所有輸入資料

您可以使用 `all` 方法將傳入請求的所有輸入資料作為 `array` 取得。無論傳入的請求是來自 HTML 表單還是 XHR 請求，都可以使用此方法：

```php
$input = $request->all();
```

使用 `collect` 方法，您可以將傳入請求的所有輸入資料作為 [集合](/docs/{{version}}/collections) 取得：

```php
$input = $request->collect();
```

`collect` 方法也允許您將傳入請求的部分輸入資料作為集合取得：

```php
$request->collect('users')->each(function (string $user) {
    // ...
});
```


<a name="retrieving-an-input-value"></a>
#### 取得單一輸入值

透過幾個簡單的方法，您可以從 `Illuminate\Http\Request` 實例中存取所有使用者輸入資料，而無需擔心請求使用的是哪種 HTTP 動作。無論 HTTP 動作為何，都可以使用 `input` 方法來取得使用者輸入：

```php
$name = $request->input('name');
```

您可以將預設值作為第二個引數傳遞給 `input` 方法。如果請求中不存在所請求的輸入值，則會傳回此值：

```php
$name = $request->input('name', 'Sally');
```

在處理包含陣列輸入的表單時，可以使用「點」記號來存取陣列：

```php
$name = $request->input('products.0.name');

$names = $request->input('products.*.name');
```

您可以不傳入任何引數呼叫 `input` 方法，以取得所有輸入值組成的關聯陣列：

```php
$input = $request->input();
```


<a name="retrieving-input-from-the-query-string"></a>
#### 從查詢字串取得輸入資料

雖然 `input` 方法可以從整個請求有效負載（包含查詢字串）中取得值，但 `query` 方法只會從查詢字串中取得值：

```php
$name = $request->query('name');
```

如果所請求的查詢字串資料不存在，將會傳回此方法的第二個引數：

```php
$name = $request->query('name', 'Helen');
```

您可以不傳入任何引數呼叫 `query` 方法，以取得所有查詢字串值組成的關聯陣列：

```php
$query = $request->query();
```


<a name="retrieving-json-input-values"></a>
#### 取得 JSON 輸入值

當傳送 JSON 請求給您的應用程式時，只要請求的 `Content-Type` 標頭正確設定為 `application/json`，您就可以透過 `input` 方法存取 JSON 資料。您甚至可以使用「點」語法來取得巢狀於 JSON 陣列或物件中的值：

```php
$name = $request->input('user.name');
```


<a name="retrieving-stringable-input-values"></a>
#### 取得 Stringable 輸入值

除了將請求的輸入資料作為原生 `string` 取得之外，您也可以使用 `string` 方法將請求資料作為 [Illuminate\Support\Stringable](/docs/{{version}}/strings) 的實例取得：

```php
$name = $request->string('name')->trim();
```


<a name="retrieving-integer-input-values"></a>
#### 取得整數輸入值

若要將輸入值作為整數取得，您可以使用 `integer` 方法。此方法會嘗試將輸入值型別轉換為整數。如果輸入值不存在或轉換失敗，它將傳回您指定的預設值。這對於分頁或其他數值輸入特別有用：

```php
$perPage = $request->integer('per_page');
```


<a name="retrieving-boolean-input-values"></a>
#### 取得布林輸入值

在處理像核取方塊 (checkbox) 這類的 HTML 元素時，您的應用程式可能會收到實際上是字串的「真值 (truthy)」。例如："true" 或 "on"。為了方便起見，您可以使用 `boolean` 方法將這些值作為布林值取得。`boolean` 方法針對 1、"1"、true、"true"、"on" 與 "yes" 都會傳回 `true`。所有其他值都將傳回 `false`：

```php
$archived = $request->boolean('archived');
```


<a name="retrieving-array-input-values"></a>
#### 取得陣列輸入值

包含陣列的輸入值可以使用 `array` 方法取得。此方法總是會將輸入值型別轉換為陣列。如果請求不包含給定名稱的輸入值，則會傳回空陣列：

```php
$versions = $request->array('versions');
```


<a name="retrieving-date-input-values"></a>
#### 取得日期輸入值

為方便起見，包含日期/時間的輸入值可以使用 `date` 方法作為 Carbon 實例取得。如果請求不包含給定名稱的輸入值，將傳回 `null`：

```php
$birthday = $request->date('birthday');
```

`date` 方法接受的第二個和第三個引數可用於分別指定日期的格式與時區：

```php
$elapsed = $request->date('elapsed', '!H:i', 'Europe/Madrid');
```

如果輸入值存在但格式無效，將會拋出 `InvalidArgumentException`；因此，建議您在呼叫 `date` 方法前先驗證輸入資料。


<a name="retrieving-interval-input-values"></a>
#### 取得時間間隔輸入值

包含持續時間的輸入值可以使用 `interval` 方法作為 `CarbonInterval` 實例取得。如果請求不包含給定名稱的輸入值，將傳回 `null`：

```php
$duration = $request->interval('duration');
```

如果輸入值是數值，您可以提供單位作為第二個引數。單位可以是像是 `second`、`minute` 或 `day` 的字串，或者是 `Carbon\Unit` Enum 實例：

```php
use Carbon\Unit;

$timeout = $request->interval('timeout', 'second');

$delay = $request->interval('delay', Unit::Minute);
```

如果輸入值存在但格式無效，將會拋出 `InvalidArgumentException`；因此，建議您在呼叫 `interval` 方法前先驗證輸入資料。


<a name="retrieving-enum-input-values"></a>
#### 取得 Enum 輸入值

對應於 [PHP Enum](https://www.php.net/manual/en/language.types.enumerations.php) 的輸入值也可以從請求中取得。如果請求不包含給定名稱的輸入值，或者 Enum 沒有與輸入值相符的背書值 (backing value)，將會傳回 `null`。`enum` 方法接受輸入值的名稱和 Enum 類別作為第一個與第二個引數：

```php
use App\Enums\Status;

$status = $request->enum('status', Status::class);
```

您也可以提供一個預設值，當值缺少或無效時將會傳回該預設值：

```php
$status = $request->enum('status', Status::class, Status::Pending);
```

如果輸入值是包含多個對應於 PHP Enum 的值陣列，您可以使用 `enums` 方法將該值陣列作為 Enum 實例取得：

```php
use App\Enums\Product;

$products = $request->enums('products', Product::class);
```


<a name="retrieving-input-via-dynamic-properties"></a>
#### 透過動態屬性取得輸入資料

您也可以透過 `Illuminate\Http\Request` 實例上的動態屬性來存取使用者輸入資料。例如，如果您的應用程式表單之一包含 `name` 欄位，您可以像這樣存取該欄位的值：

```php
$name = $request->name;
```

在使用動態屬性時，Laravel 會優先在請求的有效負載中尋找參數的值。如果不存在，Laravel 會在符合的路由參數中搜尋該欄位。


<a name="retrieving-a-portion-of-the-input-data"></a>
#### 取得部分輸入資料

如果您需要取得輸入資料的子集，可以使用 `only` 與 `except` 方法。這兩個方法都接受單一 `array` 或動態引數清單：

```php
$input = $request->only(['username', 'password']);

$input = $request->only('username', 'password');

$input = $request->except(['credit_card']);

$input = $request->except('credit_card');
```

> [!WARNING]
> `only` 方法會傳回您請求的所有鍵/值對；然而，它不會傳回請求中不存在的鍵/值對。

<a name="input-presence"></a>
### 確認輸入資料是否存在

你可以使用 `has` 方法來確認請求中是否存在某個數值。如果請求中存在該數值，`has` 方法會回傳 `true`：

```php
if ($request->has('name')) {
    // ...
}
```

當傳入陣列時，`has` 方法會確認是否所有指定的數值都存在：

```php
if ($request->has(['name', 'email'])) {
    // ...
}
```

`hasAny` 方法會在任何指定的數值存在時回傳 `true`：

```php
if ($request->hasAny(['name', 'email'])) {
    // ...
}
```

`whenHas` 方法會在請求中存在該數值時執行給定的閉包：

```php
$request->whenHas('name', function (string $input) {
    // ...
});
```

亦可傳入第二個閉包至 `whenHas` 方法，該閉包會在請求中不存在指定的數值時執行：

```php
$request->whenHas('name', function (string $input) {
    // The "name" value is present...
}, function () {
    // The "name" value is not present...
});
```

如果你想確認數值是否存在於請求中且不是空字串，可以使用 `filled` 方法：

```php
if ($request->filled('name')) {
    // ...
}
```

如果你想確認數值是否自請求中缺失或是空字串，可以使用 `isNotFilled` 方法：

```php
if ($request->isNotFilled('name')) {
    // ...
}
```

當傳入陣列時，`isNotFilled` 方法會確認是否所有指定的數值皆缺失或是空的：

```php
if ($request->isNotFilled(['name', 'email'])) {
    // ...
}
```

`anyFilled` 方法會在任何指定的數值不是空字串時回傳 `true`：

```php
if ($request->anyFilled(['name', 'email'])) {
    // ...
}
```

`whenFilled` 方法會在數值存在於請求中且不是空字串時執行給定的閉包：

```php
$request->whenFilled('name', function (string $input) {
    // ...
});
```

亦可傳入第二個閉包至 `whenFilled` 方法，該閉包會在指定的數值未「填寫」時執行：

```php
$request->whenFilled('name', function (string $input) {
    // The "name" value is filled...
}, function () {
    // The "name" value is not filled...
});
```

若要確認請求中是否缺少給定的鍵值，可以使用 `missing` 與 `whenMissing` 方法：

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
### 合併額外的輸入資料

有時候你可能需要手動將額外的輸入資料合併至請求現有的輸入資料中。若要達到此目的，可以使用 `merge` 方法。如果給定的輸入鍵值已經存在於請求中，它將會被提供給 `merge` 方法的資料所覆蓋：

```php
$request->merge(['votes' => 0]);
```

`mergeIfMissing` 方法可用於在對應鍵值尚未存在於請求的輸入資料時，將輸入資料合併至請求中：

```php
$request->mergeIfMissing(['votes' => 0]);
```


<a name="old-input"></a>
### 舊輸入資料

Laravel 允許你在進行下一次請求時保留上一次請求的輸入資料。這個功能在偵測到驗證錯誤後重新填寫表單時特別有用。然而，如果你使用的是 Laravel 內建的[驗證功能](/docs/{{version}}/validation)，你可能不需要直接手動使用這些 Session 輸入資料快閃（Flash）方法，因為 Laravel 部分內建的驗證機制會自動呼叫它們。


<a name="flashing-input-to-the-session"></a>
#### Flashing Input to the Session

`Illuminate\Http\Request` 類別上的 `flash` 方法會將當前的輸入資料暫存（Flash）至 [Session](/docs/{{version}}/session) 中，以便在使用者對應用程式發起下一次請求時可以使用：

```php
$request->flash();
```

你也可以使用 `flashOnly` 和 `flashExcept` 方法將請求資料的子集暫存至 Session 中。這些方法對於避免將密碼等敏感資訊存入 Session 非常有用：

```php
$request->flashOnly(['username', 'email']);

$request->flashExcept('password');
```


<a name="flashing-input-then-redirecting"></a>
#### Flashing Input Then Redirecting

由於你通常會希望將輸入資料暫存至 Session 後再重導向至前一個頁面，因此你可以使用 `withInput` 方法輕鬆地將輸入資料暫存串接在重導向操作上：

```php
return redirect('/form')->withInput();

return redirect()->route('user.create')->withInput();

return redirect('/form')->withInput(
    $request->except('password')
);
```


<a name="retrieving-old-input"></a>
#### Retrieving Old Input

若要取得前一次請求暫存的輸入資料，請在 `Illuminate\Http\Request` 實例上呼叫 `old` 方法。`old` 方法會從 [Session](/docs/{{version}}/session) 中拉取先前暫存的輸入資料：

```php
$username = $request->old('username');
```

Laravel 還提供了全域的 `old` 輔助函式。如果你要在 [Blade 模版](/docs/{{version}}/blade) 中顯示舊輸入資料，使用 `old` 輔助函式來重新填寫表單會更加方便。如果指定的欄位不存在舊輸入資料，將會回傳 `null`：

```blade
<input type="text" name="username" value="{{ old('username') }}">
```


<a name="cookies"></a>
### Cookie


<a name="retrieving-cookies-from-requests"></a>
#### Retrieving Cookies From Requests

所有由 Laravel 框架建立的 Cookie 皆經過加密並加上驗證碼簽署，這意味著如果客戶端對其進行了修改，它們將被視為無效。若要從請求中取得 Cookie 值，請在 `Illuminate\Http\Request` 實例上使用 `cookie` 方法：

```php
$value = $request->cookie('name');
```

<a name="input-trimming-and-normalization"></a>
## 輸入資料修剪與規格化

預設情況下，Laravel 在您應用程式的全域中介層堆疊中包含了 `Illuminate\Foundation\Http\Middleware\TrimStrings` 和 `Illuminate\Foundation\Http\Middleware\ConvertEmptyStringsToNull` 中介層。這些中介層會自動修剪請求中所有傳入的字串欄位，並將任何空字串欄位轉換為 `null`。這樣您就不必在路由和控制器中擔心這些規格化的問題。

#### 停用輸入資料規格化

如果您想對所有請求停用此行為，可以在應用程式的 `bootstrap/app.php` 檔案中透過呼叫 `$middleware->remove` 方法，從應用程式的中介層堆疊中移除這兩個中介層：

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

如果您想對傳入應用程式的一部分請求停用字串修剪和空字串轉換，可以在應用程式的 `bootstrap/app.php` 檔案中使用 `trimStrings` 與 `convertEmptyStringsToNull` 中介層方法。這兩個方法都接受一個閉包陣列，閉包應回傳 `true` 或 `false` 以指示是否應跳過輸入資料規格化：

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
### 取得上傳的檔案

您可以透過 `file` 方法或使用動態屬性從 `Illuminate\Http\Request` 實例取得上傳的檔案。`file` 方法回傳 `Illuminate\Http\UploadedFile` 類別的實例，該類別繼承了 PHP 的 `SplFileInfo` 類別，並提供各種與檔案互動的方法：

```php
$file = $request->file('photo');

$file = $request->photo;
```

您可以使用 `hasFile` 方法來確認請求中是否存在檔案：

```php
if ($request->hasFile('photo')) {
    // ...
}
```

若上傳的檔案是圖片，且您需要在儲存前對其進行處理，可以使用 `image` 方法取得 `Illuminate\Image\Image` 實例；若檔案不存在則回傳 `null`：

```php
$image = $request->image('photo');
```

關於圖像處理的更多資訊，請參考完整的[圖片處理文件](/docs/{{version}}/images)。

<a name="validating-successful-uploads"></a>
#### 驗證成功上傳

除了檢查檔案是否存在外，您還可以透過 `isValid` 方法驗證上傳檔案的過程中是否沒有發生問題：

```php
if ($request->file('photo')->isValid()) {
    // ...
}
```

<a name="file-paths-extensions"></a>
#### 檔案路徑與副檔名

`UploadedFile` 類別還包含存取檔案完整路徑及其副檔名的方法。`extension` 方法會嘗試根據檔案內容猜測檔案的副檔名。此副檔名可能與用戶端所提供的副檔名不同：

```php
$path = $request->photo->path();

$extension = $request->photo->extension();
```

<a name="other-file-methods"></a>
#### 其他檔案方法

`UploadedFile` 實例上還有許多其他可用的方法。請查看[該類別的 API 文件](https://github.com/symfony/symfony/blob/6.0/src/Symfony/Component/HttpFoundation/File/UploadedFile.php)以取得關於這些方法的更多資訊。

<a name="storing-uploaded-files"></a>
### 儲存上傳的檔案

要儲存上傳的檔案，您通常會使用已設定的[檔案系統](/docs/{{version}}/filesystem)之一。`UploadedFile` 類別擁有一個 `store` 方法，會將上傳的檔案移至您的其中一個磁碟，這可以是您本機檔案系統上的位置，也可以是像 Amazon S3 這樣的雲端儲存位置。

`store` 方法接受相對於檔案系統所設定之根目錄的檔案儲存路徑。此路徑不應包含檔名，因為會自動產生一個唯一的 ID 作為檔名。

`store` 方法還接受選擇性的第二個引數，用來指定應用於儲存檔案的磁碟名稱。該方法將回傳相對於磁碟根目錄的檔案路徑：

```php
$path = $request->photo->store('images');

$path = $request->photo->store('images', 's3');
```

如果您不希望自動產生檔名，可以使用 `storeAs` 方法，該方法接受路徑、檔名與磁碟名稱作為其引數：

```php
$path = $request->photo->storeAs('images', 'filename.jpg');

$path = $request->photo->storeAs('images', 'filename.jpg', 's3');
```

> [!NOTE]
> 關於 Laravel 中檔案儲存的更多資訊，請查看完整的[檔案儲存文件](/docs/{{version}}/filesystem)。

<a name="configuring-trusted-proxies"></a>
## 設定信任的代理伺服器

當您的應用程式執行在終止 TLS / SSL 憑證的負載平衡器後方時，您可能會注意到在使用 `url` 輔助函式時，應用程式有時不會產生 HTTPS 連結。通常這是因為您的應用程式接收到了從負載平衡器透過通訊埠 80 轉發過來的流量，卻不知道它應該要產生安全的連結。

為了處理這個問題，您可以啟用 Laravel 應用程式中包含的 `Illuminate\Http\Middleware\TrustProxies` 中介層，這讓您可以快速自訂應該被應用程式信任的負載平衡器或代理伺服器。您信任的代理伺服器應在應用程式的 `bootstrap/app.php` 檔案中使用 `trustProxies` 中介層方法來指定：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->trustProxies(at: [
        '192.168.1.1',
        '10.0.0.0/8',
    ]);
})
```

除了設定信任的代理伺服器之外，您還可以設定應該信任的代理伺服器標頭：

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
> 如果您使用的是 AWS Elastic Load Balancing，`headers` 的值應為 `Request::HEADER_X_FORWARDED_AWS_ELB`。如果您的負載平衡器使用來自 [RFC 7239](https://www.rfc-editor.org/rfc/rfc7239#section-4) 的標準 `Forwarded` 標頭，`headers` 的值應為 `Request::HEADER_FORWARDED`。關於可以在 `headers` 值中使用的常數的更多資訊，請查看 Symfony 關於[信任代理伺服器](https://symfony.com/doc/current/deployment/proxies.html)的文件。

<a name="trusting-all-proxies"></a>
#### 信任所有代理伺服器

如果您使用的是 Amazon AWS 或其他「雲端」負載平衡器提供者，您可能不知道實際負載平衡器的 IP 位址。在這種情況下，您可以使用 `*` 來信任所有代理伺服器：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->trustProxies(at: '*');
})
```

<a name="configuring-trusted-hosts"></a>
## 設定信任的主機

預設情況下，無論 HTTP 請求的 `Host` 標頭內容為何，Laravel 都會回應接收到的所有請求。此外，在 Web 請求期間產生指向應用程式的絕對 URL 時，也會使用 `Host` 標頭的值。

通常，您應該設定 Web 伺服器（例如 Nginx 或 Apache），使其僅將符合特定主機名稱的請求發送到您的應用程式。但是，若您無法直接自訂 Web 伺服器，且需要指示 Laravel 只回應特定主機名稱，您可以透過為應用程式啟用 `Illuminate\Http\Middleware\TrustHosts` 中介層來達成此目的。

要啟用 `TrustHosts` 中介層，您應該在應用程式的 `bootstrap/app.php` 檔案中呼叫 `trustHosts` 中介層方法。透過使用該方法的 `at` 引數，您可以指定應用程式應回應的主機名稱。主機名稱字串會被視為正規表示式處理。帶有其他 `Host` 標頭的傳入請求將會被拒絕：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->trustHosts(at: ['^laravel\.test$']);
})
```

預設情況下，來自應用程式 URL 子網域的請求也會被自動信任。如果您想要停用此行為，可以使用 `subdomains` 引數：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->trustHosts(at: ['^laravel\.test$'], subdomains: false);
})
```

如果您需要存取應用程式的設定檔或資料庫來確定信任的主機，可以為 `at` 引數提供一個 Closure：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->trustHosts(at: fn () => config('app.trusted_hosts'));
})
```