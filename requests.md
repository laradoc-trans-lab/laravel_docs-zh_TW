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
    - [輸入是否存在](#input-presence)
    - [合併額外輸入](#merging-additional-input)
    - [舊有輸入](#old-input)
    - [Cookie](#cookies)
    - [輸入字串修剪與正規化](#input-trimming-and-normalization)
- [檔案](#files)
    - [取得上傳檔案](#retrieving-uploaded-files)
    - [儲存上傳檔案](#storing-uploaded-files)
- [設定信任的 Proxy](#configuring-trusted-proxies)
- [設定信任的主機](#configuring-trusted-hosts)

<a name="introduction"></a>
## 簡介

Laravel 的 `Illuminate\Http\Request` 類別提供了一種物件導向的方式，來與應用程式目前正在處理的 HTTP 請求互動，並取得隨請求提交的輸入、Cookie 和檔案。

<a name="interacting-with-the-request"></a>
## 與請求互動

<a name="accessing-the-request"></a>
### 存取請求

若要透過依賴注入 (dependency injection) 取得目前 HTTP 請求的實例，您應該在路由閉包 (route closure) 或控制器方法上型別提示 (type-hint) `Illuminate\Http\Request` 類別。傳入的請求實例將會由 Laravel [服務容器](/docs/{{version}}/container)自動注入：

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

如前所述，您也可以在路由閉包上型別提示 `Illuminate\Http\Request` 類別。服務容器會在閉包執行時自動注入傳入的請求：

    use Illuminate\Http\Request;

    Route::get('/', function (Request $request) {
        // ...
    });

<a name="dependency-injection-route-parameters"></a>
#### 依賴注入與路由參數

如果您的控制器方法也預期從路由參數取得輸入，您應該將路由參數列在其他依賴之後。例如，如果您的路由定義如下：

    use App\Http\Controllers\UserController;

    Route::put('/user/{id}', [UserController::class, 'update']);

您仍然可以型別提示 `Illuminate\Http\Request` 並透過以下方式定義控制器方法來存取您的 `id` 路由參數：

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

<a name="request-path-and-method"></a>
### 請求路徑、主機與方法

`Illuminate\Http\Request` 實例提供了多種方法來檢查傳入的 HTTP 請求，並擴展了 `Symfony\Component\HttpFoundation\Request` 類別。我們將在下面討論一些最重要的方法。

<a name="retrieving-the-request-path"></a>
#### 取得請求路徑

`path` 方法會回傳請求的路徑資訊。因此，如果傳入的請求目標是 `http://example.com/foo/bar`，`path` 方法將會回傳 `foo/bar`：

    $uri = $request->path();

<a name="inspecting-the-request-path"></a>
#### 檢查請求路徑 / 路由

`is` 方法允許您驗證傳入的請求路徑是否符合給定的模式。在使用此方法時，您可以使用 `*` 字元作為萬用字元：

    if ($request->is('admin/*')) {
        // ...
    }

使用 `routeIs` 方法，您可以判斷傳入的請求是否符合[具名路由](/docs/{{version}}/routing#named-routes)：

    if ($request->routeIs('admin.*')) {
        // ...
    }

<a name="retrieving-the-request-url"></a>
#### 取得請求 URL

若要取得傳入請求的完整 URL，您可以使用 `url` 或 `fullUrl` 方法。`url` 方法會回傳不含查詢字串的 URL，而 `fullUrl` 方法則包含查詢字串：

    $url = $request->url();

    $urlWithQueryString = $request->fullUrl();

如果您想將查詢字串資料附加到目前的 URL，您可以呼叫 `fullUrlWithQuery` 方法。此方法會將給定的查詢字串變數陣列與目前的查詢字串合併：

    $request->fullUrlWithQuery(['type' => 'phone']);

如果您想取得不含給定查詢字串參數的目前 URL，您可以使用 `fullUrlWithoutQuery` 方法：

```php
$request->fullUrlWithoutQuery(['type']);
```

<a name="retrieving-the-request-host"></a>
#### 取得請求主機

您可以透過 `host`、`httpHost` 和 `schemeAndHttpHost` 方法來取得傳入請求的「主機」：

    $request->host();
    $request->httpHost();
    $request->schemeAndHttpHost();

<a name="retrieving-the-request-method"></a>
#### 取得請求方法

`method` 方法會回傳請求的 HTTP 動詞。您可以使用 `isMethod` 方法來驗證 HTTP 動詞是否符合給定的字串：

    $method = $request->method();

    if ($request->isMethod('post')) {
        // ...
    }

<a name="request-headers"></a>
### 請求標頭

您可以使用 `header` 方法從 `Illuminate\Http\Request` 實例中取得請求標頭。如果請求中不存在該標頭，則會回傳 `null`。然而，`header` 方法接受一個可選的第二個參數，如果請求中不存在該標頭，則會回傳該參數：

    $value = $request->header('X-Header-Name');

    $value = $request->header('X-Header-Name', 'default');

`hasHeader` 方法可用於判斷請求是否包含給定的標頭：

    if ($request->hasHeader('X-Header-Name')) {
        // ...
    }

為方便起見，`bearerToken` 方法可用於從 `Authorization` 標頭中取得 Bearer Token。如果不存在此標頭，則會回傳空字串：

    $token = $request->bearerToken();

<a name="request-ip-address"></a>
### 請求 IP 位址

`ip` 方法可用於取得向您的應用程式發出請求的用戶端 IP 位址：

    $ipAddress = $request->ip();

如果您想取得 IP 位址陣列，包括所有由 Proxy 轉發的用戶端 IP 位址，您可以使用 `ips` 方法。「原始」用戶端 IP 位址將位於陣列的末尾：

    $ipAddresses = $request->ips();

一般來說，IP 位址應被視為不受信任、由使用者控制的輸入，僅用於資訊目的。

<a name="content-negotiation"></a>
### 內容協商

Laravel 提供了幾種方法，透過 `Accept` 標頭來檢查傳入請求所要求的內容類型。首先，`getAcceptableContentTypes` 方法會回傳一個陣列，其中包含請求接受的所有內容類型：

    $contentTypes = $request->getAcceptableContentTypes();

`accepts` 方法接受一個內容類型陣列，如果請求接受其中任何一種內容類型，則回傳 `true`。否則，將回傳 `false`：

    if ($request->accepts(['text/html', 'application/json'])) {
        // ...
    }

您可以使用 `prefers` 方法來判斷在給定的內容類型陣列中，請求最偏好哪種內容類型。如果請求不接受任何提供的內容類型，則會回傳 `null`：

    $preferred = $request->prefers(['text/html', 'application/json']);

由於許多應用程式只提供 HTML 或 JSON，您可以使用 `expectsJson` 方法來快速判斷傳入的請求是否預期 JSON 回應：

    if ($request->expectsJson()) {
        // ...
    }

<a name="psr7-requests"></a>
### PSR-7 請求

[PSR-7 標準](https://www.php-fig.org/psr/psr-7/) 指定了 HTTP 訊息的介面，包括請求和回應。如果您想取得 PSR-7 請求的實例而不是 Laravel 請求，您需要先安裝一些函式庫。Laravel 使用 *Symfony HTTP Message Bridge* 元件將典型的 Laravel 請求和回應轉換為與 PSR-7 相容的實作：

```shell
composer require symfony/psr-http-message-bridge
composer require nyholm/psr7
```

安裝這些函式庫後，您可以透過在路由閉包或控制器方法上型別提示請求介面來取得 PSR-7 請求：

    use Psr\Http\Message\ServerRequestInterface;

    Route::get('/', function (ServerRequestInterface $request) {
        // ...
    });

> [!NOTE]
> 如果您從路由或控制器回傳 PSR-7 回應實例，它將會自動轉換回 Laravel 回應實例並由框架顯示。

<a name="input"></a>
## 輸入

<a name="retrieving-input"></a>
### 取得輸入

<a name="retrieving-all-input-data"></a>
#### 取得所有輸入資料

您可以使用 `all` 方法將所有傳入請求的輸入資料作為 `array` 取得。無論傳入請求是來自 HTML 表單還是 XHR 請求，都可以使用此方法：

    $input = $request->all();

使用 `collect` 方法，您可以將所有傳入請求的輸入資料作為[集合](/docs/{{version}}/collections)取得：

    $input = $request->collect();

`collect` 方法也允許您將傳入請求輸入的子集作為集合取得：

    $request->collect('users')->each(function (string $user) {
        // ...
    });

<a name="retrieving-an-input-value"></a>
#### 取得輸入值

使用幾個簡單的方法，您可以從 `Illuminate\Http\Request` 實例存取所有使用者輸入，而無需擔心請求使用了哪個 HTTP 動詞。無論 HTTP 動詞為何，`input` 方法都可用於取得使用者輸入：

    $name = $request->input('name');

您可以將預設值作為 `input` 方法的第二個參數傳遞。如果請求中不存在所請求的輸入值，則會回傳此值：

    $name = $request->input('name', 'Sally');

當處理包含陣列輸入的表單時，請使用「點」符號來存取陣列：

    $name = $request->input('products.0.name');

    $names = $request->input('products.*.name');

您可以不帶任何參數呼叫 `input` 方法，以取得所有輸入值作為關聯陣列：

    $input = $request->input();

<a name="retrieving-input-from-the-query-string"></a>
#### 從查詢字串取得輸入

雖然 `input` 方法從整個請求負載 (包括查詢字串) 中取得值，但 `query` 方法只會從查詢字串中取得值：

    $name = $request->query('name');

如果請求的查詢字串值資料不存在，則會回傳此方法的第二個參數：

    $name = $request->query('name', 'Helen');

您可以不帶任何參數呼叫 `query` 方法，以取得所有查詢字串值作為關聯陣列：

    $query = $request->query();

<a name="retrieving-json-input-values"></a>
#### 取得 JSON 輸入值

當向您的應用程式發送 JSON 請求時，只要請求的 `Content-Type` 標頭正確設定為 `application/json`，您就可以透過 `input` 方法存取 JSON 資料。您甚至可以使用「點」語法來取得巢狀 JSON 陣列/物件中的值：

    $name = $request->input('user.name');

<a name="retrieving-stringable-input-values"></a>
#### 取得可字串化輸入值

您可以使用 `string` 方法將請求的輸入資料作為 [`Illuminate\Support\Stringable`](/docs/{{version}}/strings) 實例取得，而不是作為原始 `string` 取得：

    $name = $request->string('name')->trim();

<a name="retrieving-integer-input-values"></a>
#### 取得整數輸入值

若要將輸入值作為整數取得，您可以使用 `integer` 方法。此方法會嘗試將輸入值轉換為整數。如果輸入不存在或轉換失敗，它將回傳您指定的預設值。這對於分頁或其他數字輸入特別有用：

    $perPage = $request->integer('per_page');

<a name="retrieving-boolean-input-values"></a>
#### 取得布林輸入值

當處理像核取方塊這樣的 HTML 元素時，您的應用程式可能會收到實際上是字串的「真值」。例如，「true」或「on」。為方便起見，您可以使用 `boolean` 方法將這些值作為布林值取得。`boolean` 方法對於 1、「1」、true、「true」、「on」和「yes」回傳 `true`。所有其他值將回傳 `false`：

    $archived = $request->boolean('archived');

<a name="retrieving-date-input-values"></a>
#### 取得日期輸入值

為方便起見，包含日期/時間的輸入值可以使用 `date` 方法作為 Carbon 實例取得。如果請求中不包含具有給定名稱的輸入值，則會回傳 `null`：

    $birthday = $request->date('birthday');

`date` 方法接受的第二個和第三個參數可用於分別指定日期的格式和時區：

    $elapsed = $request->date('elapsed', '!H:i', 'Europe/Madrid');

如果輸入值存在但格式無效，則會拋出 `InvalidArgumentException`；因此，建議您在呼叫 `date` 方法之前驗證輸入。

<a name="retrieving-enum-input-values"></a>
#### 取得 Enum 輸入值

與 [PHP Enum](https://www.php.net/manual/en/language.types.enumerations.php) 對應的輸入值也可以從請求中取得。如果請求中不包含具有給定名稱的輸入值，或者 Enum 沒有與輸入值匹配的後備值，則會回傳 `null`。`enum` 方法接受輸入值的名稱和 Enum 類別作為其第一個和第二個參數：

    use App\Enums\Status;

    $status = $request->enum('status', Status::class);

如果輸入值是與 PHP Enum 對應的值陣列，您可以使用 `enums` 方法將值陣列作為 Enum 實例取得：

    use App\Enums\Product;

    $products = $request->enums('products', Product::class);

<a name="retrieving-input-via-dynamic-properties"></a>
#### 透過動態屬性取得輸入

您也可以使用 `Illuminate\Http\Request` 實例上的動態屬性來存取使用者輸入。例如，如果您的應用程式表單之一包含 `name` 欄位，您可以像這樣存取該欄位的值：

    $name = $request->name;

當使用動態屬性時，Laravel 會首先在請求負載中尋找參數的值。如果不存在，Laravel 會在匹配路由的參數中尋找該欄位。

<a name="retrieving-a-portion-of-the-input-data"></a>
#### 取得部分輸入資料

如果您需要取得輸入資料的子集，您可以使用 `only` 和 `except` 方法。這兩個方法都接受單個 `array` 或動態參數列表：

    $input = $request->only(['username', 'password']);

    $input = $request->only('username', 'password');

    $input = $request->except(['credit_card']);

    $input = $request->except('credit_card');

> [!WARNING]
> `only` 方法會回傳您請求的所有鍵/值對；但是，它不會回傳請求中不存在的鍵/值對。

<a name="input-presence"></a>
### 輸入是否存在

您可以使用 `has` 方法來判斷請求中是否存在某個值。如果請求中存在該值，`has` 方法會回傳 `true`：

    if ($request->has('name')) {
        // ...
    }

當給定一個陣列時，`has` 方法會判斷所有指定的值是否存在：

    if ($request->has(['name', 'email'])) {
        // ...
    }

`hasAny` 方法會回傳 `true`，如果任何指定的值存在：

    if ($request->hasAny(['name', 'email'])) {
        // ...
    }

如果請求中存在某個值，`whenHas` 方法將會執行給定的閉包：

    $request->whenHas('name', function (string $input) {
        // ...
    });

第二個閉包可以傳遞給 `whenHas` 方法，如果指定的值不存在於請求中，則會執行該閉包：

    $request->whenHas('name', function (string $input) {
        // The "name" value is present...
    }, function () {
        // The "name" value is not present...
    });

如果您想判斷請求中是否存在某個值且不是空字串，您可以使用 `filled` 方法：

    if ($request->filled('name')) {
        // ...
    }

如果您想判斷請求中是否缺少某個值或該值是空字串，您可以使用 `isNotFilled` 方法：

    if ($request->isNotFilled('name')) {
        // ...
    }

當給定一個陣列時，`isNotFilled` 方法會判斷所有指定的值是否缺失或為空：

    if ($request->isNotFilled(['name', 'email'])) {
        // ...
    }

`anyFilled` 方法會回傳 `true`，如果任何指定的值不是空字串：

    if ($request->anyFilled(['name', 'email'])) {
        // ...
    }

如果請求中存在某個值且不是空字串，`whenFilled` 方法將會執行給定的閉包：

    $request->whenFilled('name', function (string $input) {
        // ...
    });

第二個閉包可以傳遞給 `whenFilled` 方法，如果指定的值未「填寫」，則會執行該閉包：

    $request->whenFilled('name', function (string $input) {
        // The "name" value is filled...
    }, function () {
        // The "name" value is not filled...
    });

若要判斷請求中是否缺少給定的鍵，您可以使用 `missing` 和 `whenMissing` 方法：

    if ($request->missing('name')) {
        // ...
    }

    $request->whenMissing('name', function () {
        // The "name" value is missing...
    }, function () {
        // The "name" value is present...
    });

<a name="merging-additional-input"></a>
### 合併額外輸入

有時您可能需要手動將額外輸入合併到請求現有的輸入資料中。為此，您可以使用 `merge` 方法。如果請求中已存在給定的輸入鍵，它將被提供給 `merge` 方法的資料覆寫：

    $request->merge(['votes' => 0]);

`mergeIfMissing` 方法可用於在請求中不存在對應鍵的情況下，將輸入合併到請求中：

    $request->mergeIfMissing(['votes' => 0]);

<a name="old-input"></a>
### 舊有輸入

Laravel 允許您在下一個請求期間保留一個請求的輸入。此功能對於在偵測到驗證錯誤後重新填寫表單特別有用。但是，如果您使用 Laravel 內建的[驗證功能](/docs/{{version}}/validation)，您可能不需要直接手動使用這些 Session 輸入快閃方法，因為 Laravel 的一些內建驗證功能會自動呼叫它們。

<a name="flashing-input-to-the-session"></a>
#### 將輸入快閃到 Session

`Illuminate\Http\Request` 類別上的 `flash` 方法會將目前的輸入快閃到 [Session](/docs/{{version}}/session)，以便在使用者下次向應用程式發出請求時可用：

    $request->flash();

您也可以使用 `flashOnly` 和 `flashExcept` 方法將請求資料的子集快閃到 Session。這些方法對於將敏感資訊 (例如密碼) 保留在 Session 之外很有用：

    $request->flashOnly(['username', 'email']);

    $request->flashExcept('password');

<a name="flashing-input-then-redirecting"></a>
#### 快閃輸入然後重新導向

由於您通常會希望將輸入快閃到 Session 然後重新導向到上一頁，因此您可以使用 `withInput` 方法輕鬆地將輸入快閃鏈接到重新導向：

    return redirect('/form')->withInput();

    return redirect()->route('user.create')->withInput();

    return redirect('/form')->withInput(
        $request->except('password')
    );

<a name="retrieving-old-input"></a>
#### 取得舊有輸入

若要從先前的請求中取得快閃輸入，請在 `Illuminate\Http\Request` 實例上呼叫 `old` 方法。`old` 方法會從 [Session](/docs/{{version}}/session) 中提取先前快閃的輸入資料：

    $username = $request->old('username');

Laravel 也提供了一個全域的 `old` 輔助函式。如果您在 [Blade 模板](/docs/{{version}}/blade)中顯示舊有輸入，使用 `old` 輔助函式重新填寫表單會更方便。如果給定欄位沒有舊有輸入，則會回傳 `null`：

    <input type="text" name="username" value="{{ old('username') }}">

<a name="cookies"></a>
### Cookie

<a name="retrieving-cookies-from-requests"></a>
#### 從請求中取得 Cookie

所有由 Laravel 框架建立的 Cookie 都會被加密並使用驗證碼簽署，這表示如果它們被用戶端更改，將被視為無效。若要從請求中取得 Cookie 值，請在 `Illuminate\Http\Request` 實例上使用 `cookie` 方法：

    $value = $request->cookie('name');

<a name="input-trimming-and-normalization"></a>
## 輸入字串修剪與正規化

預設情況下，Laravel 在您的應用程式全域 Middleware 堆疊中包含了 `Illuminate\Foundation\Http\Middleware\TrimStrings` 和 `Illuminate\Foundation\Http\Middleware\ConvertEmptyStringsToNull` Middleware。這些 Middleware 會自動修剪請求中所有傳入的字串欄位，並將任何空字串欄位轉換為 `null`。這讓您無需在路由和控制器中擔心這些正規化問題。

#### 停用輸入正規化

如果您想為所有請求停用此行為，您可以透過在應用程式的 `bootstrap/app.php` 檔案中呼叫 `$middleware->remove` 方法來從應用程式的 Middleware 堆疊中移除這兩個 Middleware：

    use Illuminate\Foundation\Http\Middleware\ConvertEmptyStringsToNull;
    use Illuminate\Foundation\Http\Middleware\TrimStrings;

    ->withMiddleware(function (Middleware $middleware) {
        $middleware->remove([
            ConvertEmptyStringsToNull::class,
            TrimStrings::class,
        ]);
    })

如果您想為應用程式的某些請求停用字串修剪和空字串轉換，您可以在應用程式的 `bootstrap/app.php` 檔案中使用 `trimStrings` 和 `convertEmptyStringsToNull` Middleware 方法。這兩個方法都接受一個閉包陣列，該閉包應回傳 `true` 或 `false` 以指示是否應跳過輸入正規化：

    ->withMiddleware(function (Middleware $middleware) {
        $middleware->convertEmptyStringsToNull(except: [
            fn (Request $request) => $request->is('admin/*'),
        ]);

        $middleware->trimStrings(except: [
            fn (Request $request) => $request->is('admin/*'),
        ]);
    })

<a name="files"></a>
## 檔案

<a name="retrieving-uploaded-files"></a>
### 取得上傳檔案

您可以使用 `file` 方法或動態屬性從 `Illuminate\Http\Request` 實例中取得上傳檔案。`file` 方法會回傳 `Illuminate\Http\UploadedFile` 類別的實例，該類別擴展了 PHP `SplFileInfo` 類別，並提供了多種與檔案互動的方法：

    $file = $request->file('photo');

    $file = $request->photo;

您可以使用 `hasFile` 方法判斷請求中是否存在檔案：

    if ($request->hasFile('photo')) {
        // ...
    }

<a name="validating-successful-uploads"></a>
#### 驗證成功上傳

除了檢查檔案是否存在之外，您還可以透過 `isValid` 方法驗證檔案上傳沒有問題：

    if ($request->file('photo')->isValid()) {
        // ...
    }

<a name="file-paths-extensions"></a>
#### 檔案路徑與副檔名

`UploadedFile` 類別還包含用於存取檔案的完整路徑及其副檔名的方法。`extension` 方法會嘗試根據檔案內容猜測檔案的副檔名。此副檔名可能與用戶端提供的副檔名不同：

    $path = $request->photo->path();

    $extension = $request->photo->extension();

<a name="other-file-methods"></a>
#### 其他檔案方法

`UploadedFile` 實例上還有許多其他可用方法。請查閱[該類別的 API 文件](https://github.com/symfony/symfony/blob/6.0/src/Symfony/Component/HttpFoundation/File/UploadedFile.php)以獲取有關這些方法的更多資訊。

<a name="storing-uploaded-files"></a>
### 儲存上傳檔案

若要儲存上傳的檔案，您通常會使用其中一個已設定的[檔案系統](/docs/{{version}}/filesystem)。`UploadedFile` 類別有一個 `store` 方法，它會將上傳的檔案移動到您的其中一個磁碟，該磁碟可以是您本機檔案系統上的位置，也可以是 Amazon S3 等雲端儲存位置。

`store` 方法接受檔案應儲存的路徑，該路徑相對於檔案系統已設定的根目錄。此路徑不應包含檔案名稱，因為會自動產生唯一的 ID 作為檔案名稱。

`store` 方法還接受一個可選的第二個參數，用於指定應用於儲存檔案的磁碟名稱。該方法將回傳檔案相對於磁碟根目錄的路徑：

    $path = $request->photo->store('images');

    $path = $request->photo->store('images', 's3');

如果您不希望自動產生檔案名稱，您可以使用 `storeAs` 方法，該方法接受路徑、檔案名稱和磁碟名稱作為其參數：

    $path = $request->photo->storeAs('images', 'filename.jpg');

    $path = $request->photo->storeAs('images', 'filename.jpg', 's3');

> [!NOTE]
> 有關 Laravel 中檔案儲存的更多資訊，請查閱完整的[檔案儲存文件](/docs/{{version}}/filesystem)。

<a name="configuring-trusted-proxies"></a>
## 設定信任的 Proxy

當您的應用程式在終止 TLS / SSL 憑證的負載平衡器後執行時，您可能會注意到在使用 `url` 輔助函式時，您的應用程式有時不會產生 HTTPS 連結。通常這是因為您的應用程式正在從負載平衡器接收來自 80 埠的流量，並且不知道它應該產生安全連結。

為了解決這個問題，您可以啟用 Laravel 應用程式中包含的 `Illuminate\Http\Middleware\TrustProxies` Middleware，它允許您快速自訂應用程式應信任的負載平衡器或 Proxy。您的信任 Proxy 應使用應用程式 `bootstrap/app.php` 檔案中的 `trustProxies` Middleware 方法指定：

    ->withMiddleware(function (Middleware $middleware) {
        $middleware->trustProxies(at: [
            '192.168.1.1',
            '10.0.0.0/8',
        ]);
    })

除了設定信任的 Proxy 之外，您還可以設定應信任的 Proxy 標頭：

    ->withMiddleware(function (Middleware $middleware) {
        $middleware->trustProxies(headers: Request::HEADER_X_FORWARDED_FOR |
            Request::HEADER_X_FORWARDED_HOST |
            Request::HEADER_X_FORWARDED_PORT |
            Request::HEADER_X_FORWARDED_PROTO |
            Request::HEADER_X_FORWARDED_AWS_ELB
        );
    })

> [!NOTE]
> 如果您使用 AWS Elastic Load Balancing，`headers` 值應為 `Request::HEADER_X_FORWARDED_AWS_ELB`。如果您的負載平衡器使用 [RFC 7239](https://www.rfc-editor.org/rfc/rfc7239#section-4) 中的標準 `Forwarded` 標頭，則 `headers` 值應為 `Request::HEADER_FORWARDED`。有關可用於 `headers` 值的常數的更多資訊，請查閱 Symfony 關於[信任 Proxy](https://symfony.com/doc/7.0/deployment/proxies.html) 的文件。

<a name="trusting-all-proxies"></a>
#### 信任所有 Proxy

如果您使用 Amazon AWS 或其他「雲端」負載平衡器提供商，您可能不知道實際平衡器的 IP 位址。在這種情況下，您可以使用 `*` 來信任所有 Proxy：

    ->withMiddleware(function (Middleware $middleware) {
        $middleware->trustProxies(at: '*');
    })

<a name="configuring-trusted-hosts"></a>
## 設定信任的主機

預設情況下，Laravel 會回應它收到的所有請求，無論 HTTP 請求的 `Host` 標頭內容為何。此外，在 Web 請求期間產生應用程式的絕對 URL 時，將使用 `Host` 標頭的值。

通常，您應該設定您的 Web 伺服器 (例如 Nginx 或 Apache) 只將符合給定主機名稱的請求發送到您的應用程式。但是，如果您無法直接自訂您的 Web 伺服器，並且需要指示 Laravel 只回應某些主機名稱，您可以透過為您的應用程式啟用 `Illuminate\Http\Middleware\TrustHosts` Middleware 來實現。

若要啟用 `TrustHosts` Middleware，您應該在應用程式的 `bootstrap/app.php` 檔案中呼叫 `trustHosts` Middleware 方法。使用此方法的 `at` 參數，您可以指定您的應用程式應回應的主機名稱。具有其他 `Host` 標頭的傳入請求將被拒絕：

    ->withMiddleware(function (Middleware $middleware) {
        $middleware->trustHosts(at: ['laravel.test']);
    })

預設情況下，來自應用程式 URL 子網域的請求也會自動被信任。如果您想停用此行為，您可以使用 `subdomains` 參數：

    ->withMiddleware(function (Middleware $middleware) {
        $middleware->trustHosts(at: ['laravel.test'], subdomains: false);
    })

如果您需要存取應用程式的設定檔或資料庫來判斷您的信任主機，您可以向 `at` 參數提供一個閉包：

    ->withMiddleware(function (Middleware $middleware) {
        $middleware->trustHosts(at: fn () => config('app.trusted_hosts'));
    })
