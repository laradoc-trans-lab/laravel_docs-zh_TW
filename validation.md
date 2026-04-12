# 驗證

- [簡介](#introduction)
- [驗證快速上手](#validation-quickstart)
    - [定義路由](#quick-defining-the-routes)
    - [建立控制器](#quick-creating-the-controller)
    - [撰寫驗證邏輯](#quick-writing-the-validation-logic)
    - [顯示驗證錯誤](#quick-displaying-the-validation-errors)
    - [重新填充表單](#repopulating-forms)
    - [關於可選欄位的注意事項](#a-note-on-optional-fields)
    - [驗證錯誤回應格式](#validation-error-response-format)
- [表單請求驗證](#form-request-validation)
    - [建立表單請求](#creating-form-requests)
    - [授權表單請求](#authorizing-form-requests)
    - [自訂錯誤訊息](#customizing-the-error-messages)
    - [準備驗證輸入](#preparing-input-for-validation)
- [手動建立驗證器](#manually-creating-validators)
    - [自動重新導向](#automatic-redirection)
    - [具名錯誤袋](#named-error-bags)
    - [自訂錯誤訊息](#manual-customizing-the-error-messages)
    - [執行額外驗證](#performing-additional-validation)
- [處理已驗證的輸入](#working-with-validated-input)
- [處理錯誤訊息](#working-with-error-messages)
    - [在語言檔案中指定自訂訊息](#specifying-custom-messages-in-language-files)
    - [在語言檔案中指定屬性](#specifying-attribute-in-language-files)
    - [在語言檔案中指定值](#specifying-values-in-language-files)
- [可用的驗證規則](#available-validation-rules)
- [條件式新增規則](#conditionally-adding-rules)
- [驗證陣列](#validating-arrays)
    - [驗證巢狀陣列輸入](#validating-nested-array-input)
    - [錯誤訊息的索引與位置](#error-message-indexes-and-positions)
- [驗證檔案](#validating-files)
- [驗證密碼](#validating-passwords)
- [自訂驗證規則](#custom-validation-rules)
    - [使用規則物件](#using-rule-objects)
    - [使用閉包](#using-closures)
    - [隱含規則](#implicit-rules)

<a name="introduction"></a>
## 簡介

Laravel 提供了幾種不同的方法來驗證應用程式接收到的資料。最常見的是使用所有傳入 HTTP 請求中可用的 `validate` 方法。不過，我們也會討論其他驗證方法。

Laravel 包含許多方便的驗證規則可套用於資料，甚至提供了驗證值在特定資料庫表格中是否唯一的功能。我們將詳細介紹每一項驗證規則，讓您能熟悉 Laravel 的所有驗證功能。

<a name="validation-quickstart"></a>
## 驗證快速上手

為了了解 Laravel 強大的驗證功能，讓我們來看一個驗證表單並將錯誤訊息顯示回使用者的完整範例。透過這個高層級的概觀，您將能對如何使用 Laravel 驗證傳入的請求資料有一個良好的整體理解：


<a name="quick-defining-the-routes"></a>
### 定義路由

首先，假設我們在 `routes/web.php` 檔案中定義了以下路由：

```php
use App\Http\Controllers\PostController;

Route::get('/post/create', [PostController::class, 'create']);
Route::post('/post', [PostController::class, 'store']);
```

`GET` 路由將顯示一個讓使用者建立新部落格文章的表單，而 `POST` 路由則會將新的部落格文章儲存在資料庫中。


<a name="quick-creating-the-controller"></a>
### 建立控制器

接下來，讓我們看看一個處理這些路由傳入請求的簡單控制器。我們暫時將 `store` 方法保持空白：

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;
use Illuminate\View\View;

class PostController extends Controller
{
    /**
     * Show the form to create a new blog post.
     */
    public function create(): View
    {
        return view('post.create');
    }

    /**
     * Store a new blog post.
     */
    public function store(Request $request): RedirectResponse
    {
        // Validate and store the blog post...

        $post = /** ... */

        return to_route('post.show', ['post' => $post->id]);
    }
}
```


<a name="quick-writing-the-validation-logic"></a>
### 撰寫驗證邏輯

現在我們準備好在 `store` 方法中填入驗證新部落格文章的邏輯了。為此，我們將使用 `Illuminate\Http\Request` 物件提供的 `validate` 方法。如果驗證規則通過，您的程式碼將繼續正常執行；然而，如果驗證失敗，則會拋出 `Illuminate\Validation\ValidationException` 異常，並自動將適當的錯誤回應傳回給使用者。

如果在傳統的 HTTP 請求中驗證失敗，將會產生一個重新導向至前一個 URL 的回應。如果傳入的是 XHR 請求，則會回傳一個 [包含驗證錯誤訊息的 JSON 回應](#validation-error-response-format)。

為了更深入了解 `validate` 方法，讓我們回到 `store` 方法：

```php
/**
 * Store a new blog post.
 */
public function store(Request $request): RedirectResponse
{
    $validated = $request->validate([
        'title' => 'required|unique:posts|max:255',
        'body' => 'required',
    ]);

    // The blog post is valid...

    return redirect('/posts');
}
```

如您所見，驗證規則被傳遞到 `validate` 方法中。別擔心 — 所有可用的驗證規則都已 [記錄在案](#available-validation-rules)。再次強調，如果驗證失敗，系統會自動產生適當的回應。如果驗證通過，我們的控制器將繼續正常執行。

或者，驗證規則也可以指定為規則陣列，而非單一由 `|` 分隔的字串：

```php
$validatedData = $request->validate([
    'title' => ['required', 'unique:posts', 'max:255'],
    'body' => ['required'],
]);
```

此外，您可以使用 `validateWithBag` 方法來驗證請求，並將任何錯誤訊息儲存在 [具名錯誤袋](#named-error-bags) 中：

```php
$validatedData = $request->validateWithBag('post', [
    'title' => ['required', 'unique:posts', 'max:255'],
    'body' => ['required'],
]);
```


<a name="stopping-on-first-validation-failure"></a>
#### 在第一次驗證失敗時停止

有時候，您可能希望在屬性發生第一次驗證失敗後就停止執行後續的驗證規則。若要達成此目的，請將 `bail` 規則分配給該屬性：

```php
$request->validate([
    'title' => 'bail|required|unique:posts|max:255',
    'body' => 'required',
]);
```

在此範例中，如果 `title` 屬性的 `unique` 規則失敗，則不會檢查 `max` 規則。規則將按照分配的順序進行驗證。


<a name="a-note-on-nested-attributes"></a>
#### 關於巢狀屬性的注意事項

如果傳入的 HTTP 請求包含「巢狀」欄位資料，您可以在驗證規則中使用「點」語法來指定這些欄位：

```php
$request->validate([
    'title' => 'required|unique:posts|max:255',
    'author.name' => 'required',
    'author.description' => 'required',
]);
```

另一方面，如果您的欄位名稱包含實際的句點，您可以使用反斜線轉義該句點，以明確防止其被解釋為「點」語法：

```php
$request->validate([
    'title' => 'required|unique:posts|max:255',
    'v1\.0' => 'required',
]);
```

<a name="quick-displaying-the-validation-errors"></a>
### 顯示驗證錯誤

那麼，如果傳入的請求欄位沒有通過給定的驗證規則會怎麼辦？如前所述，Laravel 會自動將使用者重新導向回他們之前的位置。此外，所有的驗證錯誤與 [請求輸入](/docs/{{version}}/requests#retrieving-old-input) 都會自動被 [快照到會話 (flashed to the session)](/docs/{{version}}/session#flash-data)。

`$errors` 變數是由 `web` 中介層群組提供的 `Illuminate\View\Middleware\ShareErrorsFromSession` 中介層與應用程式的所有視圖共用的。當此中介層被應用時，您的視圖中將始終可以使用 `$errors` 變數，讓您可以方便地假設 `$errors` 變數一向已定義且可以安全使用。`$errors` 變數將會是 `Illuminate\Support\MessageBag` 的實例。如需更多關於處理此物件的資訊，請[查看其文件](#working-with-error-messages)。

因此，在我們的範例中，當驗證失敗時，使用者將被重新導向到控制器的 `create` 方法，讓我們能在視圖中顯示錯誤訊息：

```blade
<!-- /resources/views/post/create.blade.php -->

<h1>Create Post</h1>

@if ($errors->any())
    <div class="alert alert-danger">
        <ul>
            @foreach ($errors->all() as $error)
                <li>{{ $error }}</li>
            @endforeach
        </ul>
    </div>
@endif

<!-- Create Post Form -->
```


<a name="quick-customizing-the-error-messages"></a>
#### 自訂錯誤訊息

Laravel 內建的驗證規則 each 都有一個錯誤訊息，位於您應用程式的 `lang/en/validation.php` 檔案中。如果您的應用程式沒有 `lang` 目錄，您可以使用 `lang:publish` Artisan 指令讓 Laravel 建立它。

在 `lang/en/validation.php` 檔案中，您會發現每個驗證規則都有一個翻譯條目。您可以根據應用程式的需求自由更改或修改這些訊息。

此外，您可以將此檔案複製到另一個語言目錄，以將訊息翻譯成您應用程式所使用的語言。如需了解更多關於 Laravel 在地化的資訊，請查看完整的 [在地化文件](/docs/{{version}}/localization)。

> [!WARNING]
> 預設情況下，Laravel 應用程式骨架不包含 `lang` 目錄。如果您想自訂 Laravel 的語言檔案，可以透過 `lang:publish` Artisan 指令來發布它們。


<a name="quick-xhr-requests-and-validation"></a>
#### XHR 請求與驗證

在這個範例中，我們使用傳統表單將資料發送到應用程式。然而，許多應用程式會接收來自 JavaScript 驅動的前端的 XHR 請求。在 XHR 請求中使用 `validate` 方法時，Laravel 不會產生重新導向回應。相反地，Laravel 會產生一個 [包含所有驗證錯誤的 JSON 回應](#validation-error-response-format)。此 JSON 回應將伴隨 422 HTTP 狀態碼發送。


<a name="the-at-error-directive"></a>
#### `@error` 指令

您可以使用 `@error` [Blade](/docs/{{version}}/blade) 指令來快速判斷特定屬性是否存在驗證錯誤訊息。在 `@error` 指令內，您可以印出 `$message` 變數來顯示錯誤訊息：

```blade
<!-- /resources/views/post/create.blade.php -->

<label for="title">Post Title</label>

<input
    id="title"
    type="text"
    name="title"
    class="@error('title') is-invalid @enderror"
/>

@error('title')
    <div class="alert alert-danger">{{ $message }}</div>
@enderror
```

如果您使用 [具名錯誤袋](#named-error-bags)，您可以將錯誤袋的名稱作為第二個引數傳遞給 `@error` 指令：

```blade
<input ... class="@error('title', 'post') is-invalid @enderror">
```


<a name="repopulating-forms"></a>
### 重新填充表單

當 Laravel 因為驗證錯誤而產生重新導向回應時，框架會自動將 [所有請求輸入快照到會話 (flash all of the request's input to the session)](/docs/{{version}}/session#flash-data)。這樣做的目的是讓您能在下一次請求期間方便地存取輸入內容，並重新填充使用者嘗試提交的表單。

要從之前的請求中檢索快照的輸入，請在 `Illuminate\Http\Request` 實例上呼叫 `old` 方法。`old` 方法將從 [會話](/docs/{{version}}/session) 中提取先前快照的輸入資料：

```php
$title = $request->old('title');
```

Laravel 還提供了一個全域的 `old` 輔助函式。如果您在 [Blade 模板](/docs/{{version}}/blade) 中顯示舊輸入，使用 `old` 輔助函式來重新填充表單會更方便。如果給定欄位沒有舊輸入，將返回 `null`：

```blade
<input type="text" name="title" value="{{ old('title') }}">
```


<a name="a-note-on-optional-fields"></a>
### 關於可選欄位的注意事項

預設情況下，Laravel 在應用程式的全域中介層堆疊中包含了 `TrimStrings` 和 `ConvertEmptyStringsToNull` 中介層。因此，如果您不希望驗證器將 `null` 值視為無效，通常需要將您的「可選」請求欄位標記為 `nullable`。例如：

```php
$request->validate([
    'title' => 'required|unique:posts|max:255',
    'body' => 'required',
    'publish_at' => 'nullable|date',
]);
```

在這個範例中，我們指定 `publish_at` 欄位可以是 `null` 或有效的日期表示方式。如果規則定義中沒有加上 `nullable` 修飾符，驗證器會將 `null` 視為無效日期。


<a name="validation-error-response-format"></a>
### 驗證錯誤回應格式

當您的應用程式拋出 `Illuminate\Validation\ValidationException` 異常，且傳入的 HTTP 請求預期接收 JSON 回應時，Laravel 會自動為您格式化錯誤訊息，並返回 `422 Unprocessable Entity` HTTP 回應。

下方您可以查看驗證錯誤的 JSON 回應格式範例。請注意，巢狀錯誤鍵會被扁平化為「點」標記法格式：

```json
{
    "message": "The team name must be a string. (and 4 more errors)",
    "errors": {
        "team_name": [
            "The team name must be a string.",
            "The team name must be at least 1 characters."
        ],
        "authorization.role": [
            "The selected authorization.role is invalid."
        ],
        "users.0.email": [
            "The users.0.email field is required."
        ],
        "users.2.email": [
            "The users.2.email must be a valid email address."
        ]
    }
}
```

<a name="form-request-validation"></a>
## 表單請求驗證


<a name="creating-form-requests"></a>
### 建立表單請求

針對更複雜的驗證情境，您可能希望建立一個「表單請求」。表單請求是自訂的請求類別，將其自身的驗證與授權邏輯封裝在一起。要建立表單請求類別，您可以使用 `make:request` Artisan CLI 指令：

```shell
php artisan make:request StorePostRequest
```

產生的表單請求類別將被放置在 `app/Http/Requests` 目錄中。如果該目錄不存在，在執行 `make:request` 指令時會自動建立。Laravel 產生的每個表單請求都有兩個方法：`authorize` 與 `rules`。

正如您所料，`authorize` 方法負責判斷目前已認證的使用者是否能執行該請求所代表的操作，而 `rules` 方法則回傳應應用於該請求資料的驗證規則：

```php
/**
 * Get the validation rules that apply to the request.
 *
 * @return array<string, \Illuminate\Contracts\Validation\ValidationRule|array<mixed>|string>
 */
public function rules(): array
{
    return [
        'title' => 'required|unique:posts|max:255',
        'body' => 'required',
    ];
}
```

> [!NOTE]
> 您可以在 `rules` 方法的簽署中型別提示 (type-hint) 任何您需要的依賴項。它們將透過 Laravel [服務容器](/docs/{{version}}/container) 自動解析。

那麼，驗證規則是如何被評估的？您只需要在控制器的方法中對請求進行型別提示即可。進入的表單請求會在控制器方法被呼叫之前就先經過驗證，這意味著您不需要在控制器中堆疊任何驗證邏輯：

```php
/**
 * Store a new blog post.
 */
public function store(StorePostRequest $request): RedirectResponse
{
    // The incoming request is valid...

    // Retrieve the validated input data...
    $validated = $request->validated();

    // Retrieve a portion of the validated input data...
    $validated = $request->safe()->only(['name', 'email']);
    $validated = $request->safe()->except(['name', 'email']);

    // Store the blog post...

    return redirect('/posts');
}
```

如果驗證失敗，將會產生一個重新導向回應，將使用者送回之前的位置。錯誤訊息也會被快照 (flash) 到 session 中，以便顯示。如果該請求是 XHR 請求，則會回傳一個 422 狀態碼的 HTTP 回應，其中包含 [驗證錯誤的 JSON 表示形式](#validation-error-response-format)。

> [!NOTE]
> 需要在由 Inertia 驅動的 Laravel 前端加入即時表單請求驗證？請參考 [Laravel Precognition](/docs/{{version}}/precognition)。


<a name="performing-additional-validation-on-form-requests"></a>
#### 執行額外驗證

有時候您需要在初始驗證完成後執行額外的驗證。您可以使用表單請求的 `after` 方法來達成此目的。

`after` 方法應回傳一個可呼叫對象 (callables) 或閉包 (closures) 的陣列，這些對象將在驗證完成後被執行。這些可呼叫對象將收到一個 `Illuminate\Validation\Validator` 實例，允許您在必要時提出額外的錯誤訊息：

```php
use Illuminate\Validation\Validator;

/**
 * Get the "after" validation callables for the request.
 */
public function after(): array
{
    return [
        function (Validator $validator) {
            if ($this->somethingElseIsInvalid()) {
                $validator->errors()->add(
                    'field',
                    'Something is wrong with this field!'
                );
            }
        }
    ];
}
```

如前所述，`after` 方法回傳的陣列也可以包含可呼叫類別 (invokable classes)。這些類別的 `__invoke` 方法將收到一個 `Illuminate\Validation\Validator` 實例：

```php
use App\Validation\ValidateShippingTime;
use App\Validation\ValidateUserStatus;
use Illuminate\Validation\Validator;

/**
 * Get the "after" validation callables for the request.
 */
public function after(): array
{
    return [
        new ValidateUserStatus,
        new ValidateShippingTime,
        function (Validator $validator) {
            //
        }
    ];
}
```


<a name="request-stopping-on-first-validation-rule-failure"></a>
#### 在第一次驗證失敗時停止

藉由在請求類別中加入 `StopOnFirstFailure` 屬性，您可以通知驗證器在發生單次驗證失敗後，就停止驗證所有屬性：

```php
<?php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\Attributes\StopOnFirstFailure;
use Illuminate\Foundation\Http\FormRequest;

#[StopOnFirstFailure]
class StorePostRequest extends FormRequest
{
    // ...
}
```


<a name="request-failing-on-unknown-fields"></a>
#### 針對未知欄位失敗

藉由在請求類別中加入 `FailOnUnknownFields` 屬性，您可以指示 Laravel 拒絕任何未在請求驗證規則中定義的進入欄位：

```php
<?php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\Attributes\FailOnUnknownFields;
use Illuminate\Foundation\Http\FormRequest;

#[FailOnUnknownFields]
class StorePostRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'title' => ['required', 'string'],
            'body' => ['required', 'string'],
        ];
    }
}
```

您也可以在 `AppServiceProvider` 中為所有表單請求全域啟用此行為：

```php
use Illuminate\Foundation\Http\FormRequest;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    FormRequest::failOnUnknownFields();
}
```

如果需要，您可以透過將 `false` 傳遞給屬性來為特定請求禁用此行為：

```php
#[FailOnUnknownFields(false)]
class PublicWebhookRequest extends FormRequest
{
    // ...
}
```

拒絕未知欄位可以防止非預期的輸入鍵進入應用程式深層，進而提供額外的保護，防止大量賦值 (mass-assignment) 風格的問題。儘管如此，您仍應設定模型 的 `$fillable` / `$guarded` 屬性，並且僅持久化受信任且經過驗證的輸入。


<a name="customizing-the-redirect-location"></a>
#### 自訂重新導向位置

當表單請求驗證失敗時，會產生一個重新導向回應，將使用者送回之前的位置。不過，您可以自訂此行為。要這樣做，您可以在表單請求上使用 `RedirectTo` 屬性：

```php
<?php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\Attributes\RedirectTo;
use Illuminate\Foundation\Http\FormRequest;

#[RedirectTo('/dashboard')]
class StorePostRequest extends FormRequest
{
    // ...
}
```

或者，如果您想將使用者重新導向到具名路由，可以使用 `RedirectToRoute` 屬性：

```php
<?php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\Attributes\RedirectToRoute;
use Illuminate\Foundation\Http\FormRequest;

#[RedirectToRoute('dashboard')]
class StorePostRequest extends FormRequest
{
    // ...
}
```


<a name="customizing-the-error-bag"></a>
#### 自訂錯誤袋

當表單請求驗證失敗時，錯誤會被快照到 `default` 錯誤袋中。如果您需要將錯誤儲存在不同的 [具名錯誤袋](#named-error-bags) 中，可以在表單請求上使用 `ErrorBag` 屬性：

```php
<?php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\Attributes\ErrorBag;
use Illuminate\Foundation\Http\FormRequest;

#[ErrorBag('login')]
class LoginRequest extends FormRequest
{
    // ...
}
```

<a name="authorizing-form-requests"></a>
### 授權表單請求

表單請求類別還包含一個 `authorize` 方法。在此方法中，您可以判斷已認證的使用者是否確實擁有更新指定資源的權限。例如，您可以判斷使用者是否確實擁有他們正嘗試更新的部落格留言。您很可能會在此方法中與您的 [授權 Gate 與 Policy](/docs/{{version}}/authorization) 進行互動：

```php
use App\Models\Comment;

/**
 * Determine if the user is authorized to make this request.
 */
public function authorize(): bool
{
    $comment = Comment::find($this->route('comment'));

    return $comment && $this->user()->can('update', $comment);
}
```

由於所有表單請求都繼承自 Laravel 的基礎請求類別，我們可以使用 `user` 方法來存取目前已認證的使用者。此外，請注意上述範例中對 `route` 方法的呼叫。此方法允許您存取所呼叫路由上定義的 URI 參數，例如下方範例中的 `{comment}` 參數：

```php
Route::post('/comment/{comment}');
```

因此，如果您的應用程式利用了 [路由模型綁定](/docs/{{version}}/routing#route-model-binding)，您可以透過將解析後的模型作為請求的屬性來存取，使程式碼更加簡潔：

```php
return $this->user()->can('update', $this->comment);
```

如果 `authorize` 方法回傳 `false`，系統將自動回傳一個 403 狀態碼的 HTTP 回應，且您的控制器方法將不會執行。

如果您計畫在應用程式的其他部分處理該請求的授權邏輯，您可以完全移除 `authorize` 方法，或者簡單地回傳 `true`：

```php
/**
 * Determine if the user is authorized to make this request.
 */
public function authorize(): bool
{
    return true;
}
```

> [!NOTE]
> 您可以在 `authorize` 方法的簽名中對任何需要的依賴進行型別提示 (type-hint)。它們將會透過 Laravel 的 [服務容器](/docs/{{version}}/container) 自動解析。


<a name="customizing-the-error-messages"></a>
### 自訂錯誤訊息

您可以透過覆寫 `messages` 方法來自訂表單請求所使用的錯誤訊息。此方法應回傳一個包含屬性 / 規則配對及其對應錯誤訊息的陣列：

```php
/**
 * Get the error messages for the defined validation rules.
 *
 * @return array<string, string>
 */
public function messages(): array
{
    return [
        'title.required' => 'A title is required',
        'body.required' => 'A message is required',
    ];
}
```


<a name="customizing-the-validation-attributes"></a>
#### 自訂驗證屬性

Laravel 許多內建的驗證規則錯誤訊息都包含 `:attribute` 佔位符。如果您希望驗證訊息中的 `:attribute` 佔位符被替換為自訂的屬性名稱，您可以透過覆寫 `attributes` 方法來指定自訂名稱。此方法應回傳一個包含屬性 / 名稱配對的陣列：

```php
/**
 * Get custom attributes for validator errors.
 *
 * @return array<string, string>
 */
public function attributes(): array
{
    return [
        'email' => 'email address',
    ];
}
```


<a name="preparing-input-for-validation"></a>
### 準備驗證輸入

如果您需要在套用驗證規則之前準備或清理請求中的任何資料，可以使用 `prepareForValidation` 方法：

```php
use Illuminate\Support\Str;

/**
 * Prepare the data for validation.
 */
protected function prepareForValidation(): void
{
    $this->merge([
        'slug' => Str::slug($this->slug),
    ]);
}
```

同樣地，如果您需要在驗證完成後將任何請求資料標準化，可以使用 `passedValidation` 方法：

```php
/**
 * Handle a passed validation attempt.
 */
protected function passedValidation(): void
{
    $this->replace(['name' => 'Taylor']);
}
```

<a name="manually-creating-validators"></a>
## 手動建立驗證器

如果您不想在請求上使用 `validate` 方法，可以使用 `Validator` [Facade](/docs/{{version}}/facades) 手動建立一個驗證器實例。Facade 上的 `make` 方法會產生一個新的驗證器實例：

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Validator;

class PostController extends Controller
{
    /**
     * Store a new blog post.
     */
    public function store(Request $request): RedirectResponse
    {
        $validator = Validator::make($request->all(), [
            'title' => 'required|unique:posts|max:255',
            'body' => 'required',
        ]);

        if ($validator->fails()) {
            return redirect('/post/create')
                ->withErrors($validator)
                ->withInput();
        }

        // Retrieve the validated input...
        $validated = $validator->validated();

        // Retrieve a portion of the validated input...
        $validated = $validator->safe()->only(['name', 'email']);
        $validated = $validator->safe()->except(['name', 'email']);

        // Store the blog post...

        return redirect('/posts');
    }
}
```

傳遞給 `make` 方法的第一個引數是待驗證的資料。第二個引數則是應應用於該資料的驗證規則陣列。

在確定請求驗證是否失敗後，您可以使用 `withErrors` 方法將錯誤訊息快閃 (flash) 到會話 (session) 中。使用此方法後，重新導向時 `$errors` 變數會自動與您的視圖共享，讓您能輕鬆地將錯誤顯示給使用者。`withErrors` 方法接受一個驗證器、一個 `MessageBag` 或一個 PHP `array`。


#### 在第一次驗證失敗時停止

`stopOnFirstFailure` 方法會告知驗證器，一旦發生單次驗證失敗，就應停止驗證所有屬性：

```php
if ($validator->stopOnFirstFailure()->fails()) {
    // ...
}
```


<a name="automatic-redirection"></a>
### 自動重新導向

如果您想手動建立驗證器實例，但仍想利用 HTTP 請求的 `validate` 方法所提供的自動重新導向功能，可以在現有的驗證器實例上呼叫 `validate` 方法。如果驗證失敗，使用者將被自動重新導向；如果是 XHR 請求，則會 [返回一個 JSON 回應](#validation-error-response-format)：

```php
Validator::make($request->all(), [
    'title' => 'required|unique:posts|max:255',
    'body' => 'required',
])->validate();
```

如果驗證失敗，您可以使用 `validateWithBag` 方法將錯誤訊息儲存在 [具名錯誤袋](#named-error-bags) 中：

```php
Validator::make($request->all(), [
    'title' => 'required|unique:posts|max:255',
    'body' => 'required',
])->validateWithBag('post');
```


<a name="named-error-bags"></a>
### 具名錯誤袋

如果您在單一頁面上有多個表單，您可能希望為包含驗證錯誤的 `MessageBag` 命名，以便檢索特定表單的錯誤訊息。若要實現此功能，請將名稱作為 `withErrors` 的第二個引數傳遞：

```php
return redirect('/register')->withErrors($validator, 'login');
```

接著，您可以從 `$errors` 變數中存取該具名 `MessageBag` 實例：

```blade
{{ $errors->login->first('email') }}
```


<a name="manual-customizing-the-error-messages"></a>
### 自訂錯誤訊息

如果需要，您可以提供自訂錯誤訊息，讓驗證器實例使用這些訊息來取代 Laravel 提供的預設錯誤訊息。有幾種方式可以指定自訂訊息。首先，您可以將自訂訊息作為 `Validator::make` 方法的第三個引數傳遞：

```php
$validator = Validator::make($input, $rules, $messages = [
    'required' => 'The :attribute field is required.',
]);
```

在此範例中，`:attribute` 佔位符將被替換為待驗證欄位的實際名稱。您也可以在驗證訊息中使用其他佔位符。例如：

```php
$messages = [
    'same' => 'The :attribute and :other must match.',
    'size' => 'The :attribute must be exactly :size.',
    'between' => 'The :attribute value :input is not between :min - :max.',
    'in' => 'The :attribute must be one of the following types: :values',
];
```


<a name="specifying-a-custom-message-for-a-given-attribute"></a>
#### 為特定屬性指定自訂訊息

有時您可能只想為特定屬性指定自訂錯誤訊息。您可以使用「點」符號 (dot notation) 來實現。請先指定屬性名稱，接著是規則：

```php
$messages = [
    'email.required' => 'We need to know your email address!',
];
```


<a name="specifying-custom-attribute-values"></a>
#### 指定自訂屬性值

Laravel 的許多內建錯誤訊息都包含一個 `:attribute` 佔位符，它會被替換為待驗證的欄位或屬性名稱。若要自訂特定欄位用於替換這些佔位符的值，您可以將自訂屬性陣列作為 `Validator::make` 方法的第四個引數傳遞：

```php
$validator = Validator::make($input, $rules, $messages, [
    'email' => 'email address',
]);
```


<a name="performing-additional-validation"></a>
### 執行額外驗證

有時您需要在初始驗證完成後執行額外的驗證。您可以使用驗證器的 `after` 方法來實現。`after` 方法接受一個閉包或一個可呼叫物件 (callables) 陣列，這些物件將在驗證完成後被執行。傳遞的可呼叫物件將會收到一個 `Illuminate\Validation\Validator` 實例，讓您在必要時能產生額外的錯誤訊息：

```php
use Illuminate\Support\Facades\Validator;

$validator = Validator::make(/* ... */);

$validator->after(function ($validator) {
    if ($this->somethingElseIsInvalid()) {
        $validator->errors()->add(
            'field', 'Something is wrong with this field!'
        );
    }
});

if ($validator->fails()) {
    // ...
}
```

如前所述，`after` 方法也接受一個可呼叫物件陣列，如果您的「驗證後」邏輯封裝在可呼叫類別 (invokable classes) 中，這將非常方便，這些類別將透過其 `__invoke` 方法收到一個 `Illuminate\Validation\Validator` 實例：

```php
use App\Validation\ValidateShippingTime;
use App\Validation\ValidateUserStatus;

$validator->after([
    new ValidateUserStatus,
    new ValidateShippingTime,
    function ($validator) {
        // ...
    },
]);
```

<a name="working-with-validated-input"></a>
## 處理已驗證的輸入

在使用表單請求或手動建立的驗證器實例驗證傳入的請求資料後，您可能希望檢索實際經過驗證的請求資料。這可以透過幾種方式達成。首先，您可以呼叫表單請求或驗證器實例上的 `validated` 方法。此方法會回傳一個已驗證資料的陣列：

```php
$validated = $request->validated();

$validated = $validator->validated();
```

或者，您也可以呼叫表單請求或驗證器實例上的 `safe` 方法。此方法會回傳一個 `Illuminate\Support\ValidatedInput` 實例。此物件提供了 `only`、`except` 和 `all` 方法，用以檢索已驗證資料的子集或完整的已驗證資料陣列：

```php
$validated = $request->safe()->only(['name', 'email']);

$validated = $request->safe()->except(['name', 'email']);

$validated = $request->safe()->all();
```

此外，`Illuminate\Support\ValidatedInput` 實例可以被迭代並像陣列一樣被存取：

```php
// Validated data may be iterated...
foreach ($request->safe() as $key => $value) {
    // ...
}

// Validated data may be accessed as an array...
$validated = $request->safe();

$email = $validated['email'];
```

如果您想在已驗證的資料中新增額外欄位，可以呼叫 `merge` 方法：

```php
$validated = $request->safe()->merge(['name' => 'Taylor Otwell']);
```

如果您想將已驗證的資料作為 [collection](/docs/{{version}}/collections) 實例檢索，可以呼叫 `collect` 方法：

```php
$collection = $request->safe()->collect();
```


<a name="working-with-error-messages"></a>
## 處理錯誤訊息

在 `Validator` 實例上呼叫 `errors` 方法後，您將收到一個 `Illuminate\Support\MessageBag` 實例，它提供了許多方便的方法來處理錯誤訊息。自動提供給所有視圖的 `$errors` 變數也是 `MessageBag` 類別的一個實例。


<a name="retrieving-the-first-error-message-for-a-field"></a>
#### 檢索特定欄位的第一個錯誤訊息

若要檢索給定欄位的第一個錯誤訊息，請使用 `first` 方法：

```php
$errors = $validator->errors();

echo $errors->first('email');
```


<a name="retrieving-all-error-messages-for-a-field"></a>
#### 檢索特定欄的所有錯誤訊息

如果您需要檢索給定欄位所有訊息的陣列，請使用 `get` 方法：

```php
foreach ($errors->get('email') as $message) {
    // ...
}
```

如果您正在驗證陣列形式的表單欄位，可以使用 `*` 字元來檢索陣列中每個元素的完整訊息：

```php
foreach ($errors->get('attachments.*') as $message) {
    // ...
}
```


<a name="retrieving-all-error-messages-for-all-fields"></a>
#### 檢索所有欄位的所有錯誤訊息

若要檢索所有欄位所有訊息的陣列，請使用 `all` 方法：

```php
foreach ($errors->all() as $message) {
    // ...
}
```


<a name="determining-if-messages-exist-for-a-field"></a>
#### 判斷特定欄位是否存在訊息

`has` 方法可用於判斷給定欄位是否存在任何錯誤訊息：

```php
if ($errors->has('email')) {
    // ...
}
```


<a name="specifying-custom-messages-in-language-files"></a>
### 在語言檔案中指定自訂訊息

Laravel 內建的驗證規則各自都有一個錯誤訊息，位於應用程式的 `lang/en/validation.php` 檔案中。如果您的應用程式沒有 `lang` 目錄，您可以使用 `lang:publish` Artisan 命令指示 Laravel 建立該目錄。

在 `lang/en/validation.php` 檔案中，您會發現每個驗證規則都有一個翻譯條目。您可以根據應用程式的需求自由更改或修改這些訊息。

此外，您可以將此檔案複製到另一個語言目錄，以便為應用程式的語言翻譯這些訊息。若要了解更多關於 Laravel 在地化的資訊，請參閱完整的 [在地化文件](/docs/{{version}}/localization)。

> [!WARNING]
> 預設情況下，Laravel 應用程式骨架不包含 `lang` 目錄。如果您想自訂 Laravel 的語言檔案，可以使用 `lang:publish` Artisan 命令來發布它們。


<a name="custom-messages-for-specific-attributes"></a>
#### 特定屬性的自訂訊息

您可以在應用程式的驗證語言檔案中，為指定的屬性與規則組合自訂錯誤訊息。若要這樣做，請將您的訊息自訂內容新增至應用程式 `lang/xx/validation.php` 語言檔案的 `custom` 陣列中：

```php
'custom' => [
    'email' => [
        'required' => 'We need to know your email address!',
        'max' => 'Your email address is too long!'
    ],
],
```


<a name="specifying-attribute-in-language-files"></a>
### 在語言檔案中指定屬性

許多 Laravel 內建的錯誤訊息包含一個 `:attribute` 佔位符，它會被替換為正在驗證的欄位或屬性名稱。如果您希望驗證訊息中的 `:attribute` 部分被替換為自訂的屬性名稱，可以在 `lang/xx/validation.php` 語言檔案的 `attributes` 陣列中指定自訂屬性名稱：

```php
'attributes' => [
    'email' => 'email address',
],
```

> [!WARNING]
> 預設情況下，Laravel 應用程式骨架不包含 `lang` 目錄。如果您想自訂 Laravel 的語言檔案，可以使用 `lang:publish` Artisan 命令來發布它們。


<a name="specifying-values-in-language-files"></a>
### 在語言檔案中指定值

某些 Laravel 內建的驗證規則錯誤訊息包含一個 `:value` 佔位符，它會被替換為請求屬性的目前值。然而，您偶爾可能需要將驗證訊息中的 `:value` 部分替換為該值的自訂表示方式。例如，考慮以下規則：如果 `payment_type` 的值為 `cc`，則需要提供信用卡號碼：

```php
Validator::make($request->all(), [
    'credit_card_number' => 'required_if:payment_type,cc'
]);
```

如果此驗證規則失敗，將產生以下錯誤訊息：

```text
The credit card number field is required when payment type is cc.
```

您可以在 `lang/xx/validation.php` 語言檔案中定義一個 `values` 陣列，以指定一個更易於理解的值表示方式，而不是直接顯示 `cc` 作為付款類型的值：

```php
'values' => [
    'payment_type' => [
        'cc' => 'credit card'
    ],
],
```

> [!WARNING]
> 預設情況下，Laravel 應用程式骨架不包含 `lang` 目錄。如果您想自訂 Laravel 的語言檔案，可以使用 `lang:publish` Artisan 命令來發布它們。

定義此值後，驗證規則將產生以下錯誤訊息：

```text
The credit card number field is required when payment type is credit card.
```

<a name="available-validation-rules"></a>
## 可用的驗證規則

以下是所有可用的驗證規則及其功能的列表：

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


#### 布林值

<div class="collection-method-list" markdown="1">

[Accepted](#rule-accepted)
[Accepted If](#rule-accepted-if)
[Boolean](#rule-boolean)
[Declined](#rule-declined)
[Declined If](#rule-declined-if)

</div>


#### 字串

<div class="collection-method-list" markdown="1">

[Active URL](#rule-active-url)
[Alpha](#rule-alpha)
[Alpha Dash](#rule-alpha-dash)
[Alpha Numeric](#rule-alpha-num)
[Ascii](#rule-ascii)
[Confirmed](#rule-confirmed)
[Current Password](#rule-current-password)
[Different](#rule-different)
[Doesnt Start With](#rule-doesnt-start-with)
[Doesnt End With](#rule-doesnt-end-with)
[Email](#rule-email)
[Ends With](#rule-ends-with)
[Enum](#rule-enum)
[Hex Color](#rule-hex-color)
[In](#rule-in)
[IP Address](#rule-ip)
[JSON](#rule-json)
[Lowercase](#rule-lowercase)
[MAC Address](#rule-mac)
[Max](#rule-max)
[Min](#rule-min)
[Not In](#rule-not-in)
[Regular Expression](#rule-regex)
[Not Regular Expression](#rule-not-regex)
[Same](#rule-same)
[Size](#rule-size)
[Starts With](#rule-starts-with)
[String](#rule-string)
[Uppercase](#rule-uppercase)
[URL](#rule-url)
[ULID](#rule-ulid)
[UUID](#rule-uuid)

</div>


#### 數字

<div class="collection-method-list" markdown="1">

[Between](#rule-between)
[Decimal](#rule-decimal)
[Different](#rule-different)
[Digits](#rule-digits)
[Digits Between](#rule-digits-between)
[Greater Than](#rule-gt)
[Greater Than Or Equal](#rule-gte)
[Integer](#rule-integer)
[Less Than](#rule-lt)
[Less Than Or Equal](#rule-lte)
[Max](#rule-max)
[Max Digits](#rule-max-digits)
[Min](#rule-min)
[Min Digits](#rule-min-digits)
[Multiple Of](#rule-multiple-of)
[Numeric](#rule-numeric)
[Same](#rule-same)
[Size](#rule-size)

</div>


#### 陣列

<div class="collection-method-list" markdown="1">

[Array](#rule-array)
[Between](#rule-between)
[Contains](#rule-contains)
[Doesnt Contain](#rule-doesnt-contain)
[Distinct](#rule-distinct)
[In Array](#rule-in-array)
[In Array Keys](#rule-in-array-keys)
[List](#rule-list)
[Max](#rule-max)
[Min](#rule-min)
[Size](#rule-size)

</div>


#### 日期

<div class="collection-method-list" markdown="1">

[After](#rule-after)
[After Or Equal](#rule-after-or-equal)
[Before](#rule-before)
[Before Or Equal](#rule-before-or-equal)
[Date](#rule-date)
[Date Equals](#rule-date-equals)
[Date Format](#rule-date-format)
[Different](#rule-different)
[Timezone](#rule-timezone)

</div>


#### 檔案

<div class="collection-method-list" markdown="1">

[Between](#rule-between)
[Dimensions](#rule-dimensions)
[Encoding](#rule-encoding)
[Extensions](#rule-extensions)
[File](#rule-file)
[Image](#rule-image)
[Max](#rule-max)
[MIME Types](#rule-mimetypes)
[MIME Type By File Extension](#rule-mimes)
[Size](#rule-size)

</div>


#### 資料庫

<div class="collection-method-list" markdown="1">

[Exists](#rule-exists)
[Unique](#rule-unique)

</div>


#### 工具

<div class="collection-method-list" markdown="1">

[Any Of](#rule-anyof)
[Bail](#rule-bail)
[Exclude](#rule-exclude)
[Exclude If](#rule-exclude-if)
[Exclude Unless](#rule-exclude-unless)
[Exclude With](#rule-exclude-with)
[Exclude Without](#rule-exclude-without)
[Filled](#rule-filled)
[Missing](#rule-missing)
[Missing If](#rule-missing-if)
[Missing Unless](#rule-missing-unless)
[Missing With](#rule-missing-with)
[Missing With All](#rule-missing-with-all)
[Nullable](#rule-nullable)
[Present](#rule-present)
[Present If](#rule-present-if)
[Present Unless](#rule-present-unless)
[Present With](#rule-present-with)
[Present With All](#rule-present-with-all)
[Prohibited](#rule-prohibited)
[Prohibited If](#rule-prohibited-if)
[Prohibited If Accepted](#rule-prohibited-if-accepted)
[Prohibited If Declined](#rule-prohibited-if-declined)
[Prohibited Unless](#rule-prohibited-unless)
[Prohibits](#rule-prohibits)
[Required](#rule-required)
[Required If](#rule-required-if)
[Required If Accepted](#rule-required-if-accepted)
[Required If Declined](#rule-required-if-declined)
[Required Unless](#rule-required-unless)
[Required With](#rule-required-with)
[Required With All](#rule-required-with-all)
[Required Without](#rule-required-without)
[Required Without All](#rule-required-without-all)
[Required Array Keys](#rule-required-array-keys)
[Sometimes](#validating-when-present)

</div>


<a name="rule-accepted"></a>
#### accepted

被驗證的欄位必須為 `"yes"`, `"on"`, `1`, `"1"`, `true`, 或 `"true"`。這對於驗證「服務條款」的接受或類似欄位非常有用。


<a name="rule-accepted-if"></a>
#### accepted_if:anotherfield,value,...

如果另一個被驗證的欄位等於指定的值，則被驗證的欄位必須為 `"yes"`, `"on"`, `1`, `"1"`, `true`, 或 `"true"`。這對於驗證「服務條款」的接受或類似欄位非常有用。


<a name="rule-active-url"></a>
#### active_url

根據 PHP 的 `dns_get_record` 函數，被驗證的欄位必須具有有效的 A 或 AAAA 記錄。在傳遞給 `dns_get_record` 之前，會使用 PHP 的 `parse_url` 函數提取所提供 URL 的主機名稱。


<a name="rule-after"></a>
#### after:_date_

被驗證的欄位必須是一個在指定日期之後的值。日期將被傳遞到 PHP 的 `strtotime` 函數中，以便轉換為有效的 `DateTime` 實例：

```php
'start_date' => 'required|date|after:tomorrow'
```

您可以指定另一個欄位來與該日期進行比較，而不是傳遞一個由 `strtotime` 評估的日期字串：

```php
'finish_date' => 'required|date|after:start_date'
```

為了方便起見，可以使用流暢的 `date` 規則建構器來建立基於日期的規則：

```php
use Illuminate\Validation\Rule;

'start_date' => [
    'required',
    Rule::date()->after(today()->addDays(7)),
],
```

`afterToday` 和 `todayOrAfter` 方法可用於流暢地表達日期，分別代表必須在今天之後，或者今天或之後：

```php
'start_date' => [
    'required',
    Rule::date()->afterToday(),
],
```


<a name="rule-after-or-equal"></a>
#### after\_or\_equal:_date_

被驗證的欄位必須是一個在指定日期之後或等於該日期的值。更多資訊請參閱 [after](#rule-after) 規則。

為了方便起見，可以使用流暢的 `date` 規則建構器來建立基於日期的規則：

```php
use Illuminate\Validation\Rule;

'start_date' => [
    'required',
    Rule::date()->afterOrEqual(today()->addDays(7)),
],
```


<a name="rule-anyof"></a>
#### anyOf

`Rule::anyOf` 驗證規則允許您指定被驗證的欄位必須滿足給定的任何一組驗證規則。例如，以下規則將驗證 `username` 欄位必須是電子郵件地址，或是長度至少 6 個字元的英數字串（包含連字號）：

```php
use Illuminate\Validation\Rule;

'username' => [
    'required',
    Rule::anyOf([
        ['string', 'email'],
        ['string', 'alpha_dash', 'min:6'],
    ]),
],
```


<a name="rule-alpha"></a>
#### alpha

被驗證的欄位必須完全由包含在 [\p{L}](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AL%3A%5D&g=&i=) 和 [\p{M}](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AM%3A%5D&g=&i=) 中的 Unicode 字母字元組成。

若要將此驗證規則限制在 ASCII 範圍（`a-z` 和 `A-Z`）內的字元，您可以為驗證規則提供 `ascii` 選項：

```php
'username' => 'alpha:ascii',
```

<a name="rule-alpha-dash"></a>
#### alpha_dash

驗證欄位必須完全由 Unicode 字母數字字元組成，包含 [\p{L}](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AL%3A%5D&g=&i=)、[\p{M}](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AM%3A%5D&g=&i=)、[\p{N}](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AN%3A%5D&g=&i=)，以及 ASCII 破折號 (`-`) 和 ASCII 底線 (`_`)。

若要將此驗證規則限制在 ASCII 範圍（`a-z`、`A-Z` 和 `0-9`）內的字元，您可以為驗證規則提供 `ascii` 選項：

```php
'username' => 'alpha_dash:ascii',
```


<a name="rule-alpha-num"></a>
#### alpha_num

驗證欄位必須完全由 Unicode 字母數字字元組成，包含 [\p{L}](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AL%3A%5D&g=&i=)、[\p{M}](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AM%3A%5D&g=&i=) 和 [\p{N}](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AN%3A%5D&g=&i=)。

若要將此驗證規則限制在 ASCII 範圍（`a-z`、`A-Z` 和 `0-9`）內的字元，您可以為驗證規則提供 `ascii` 選項：

```php
'username' => 'alpha_num:ascii',
```


<a name="rule-array"></a>
#### array

驗證欄位必須是一個 PHP `array`。

當為 `array` 規則提供額外的值時，輸入陣列中的每個鍵必須存在於提供給該規則的值列表中。在以下範例中，輸入陣列中的 `admin` 鍵是無效的，因為它不包含在提供給 `array` 規則的值列表中：

```php
use Illuminate\Support\Facades\Validator;

$input = [
    'user' => [
        'name' => 'Taylor Otwell',
        'username' => 'taylorotwell',
        'admin' => true,
    ],
];

Validator::make($input, [
    'user' => 'array:name,username',
]);
```

一般而言，您應該始終指定允許存在於陣列中的陣列鍵。


<a name="rule-ascii"></a>
#### ascii

驗證欄位必須完全由 7 位元 ASCII 字元組成。


<a name="rule-bail"></a>
#### bail

在第一次驗證失敗後，停止執行該欄位的驗證規則。

雖然 `bail` 規則僅在遇到驗證失敗時停止驗證特定欄位，但 `stopOnFirstFailure` 方法會告知驗證器，一旦發生單次驗證失敗，就應停止驗證所有屬性：

```php
if ($validator->stopOnFirstFailure()->fails()) {
    // ...
}
```


<a name="rule-before"></a>
#### before:_date_

驗證欄位的值必須在給定日期之前。日期將被傳遞給 PHP 的 `strtotime` 函數，以便轉換為有效的 `DateTime` 實例。此外，與 [after](#rule-after) 規則一樣，也可以提供另一個驗證欄位的名稱作為 `date` 的值。

為了方便起見，可以使用流暢的 `date` 規則建構器來建立基於日期的規則：

```php
use Illuminate\Validation\Rule;

'start_date' => [
    'required',
    Rule::date()->before(today()->subDays(7)),
],
```

可以使用 `beforeToday` 和 `todayOrBefore` 方法來流暢地表達日期必須在今天之前，或在今天或今天之前：

```php
'start_date' => [
    'required',
    Rule::date()->beforeToday(),
],
```


<a name="rule-before-or-equal"></a>
#### before\_or\_equal:_date_

驗證欄位的值必須在給定日期之前或相等。日期將被傳遞給 PHP 的 `strtotime` 函數，以便轉換為有效的 `DateTime` 實例。此外，與 [after](#rule-after) 規則一樣，也可以提供另一個驗證欄位的名稱作為 `date` 的值。

為了方便起見，可以使用流暢的 `date` 規則建構器來建立基於日期的規則：

```php
use Illuminate\Validation\Rule;

'start_date' => [
    'required',
    Rule::date()->beforeOrEqual(today()->subDays(7)),
],
```


<a name="rule-between"></a>
#### between:_min_,_max_

驗證欄位的大小必須在給定的 _min_ 和 _max_ 之間（包含邊界）。字串、數值、陣列和檔案的評估方式與 [size](#rule-size) 規則相同。


<a name="rule-boolean"></a>
#### boolean

驗證欄位必須能夠被轉換為布林值。接受的輸入為 `true`、`false`、`1`、`0`、`"1"` 和 `"0"`。

您可以使用 `strict` 參數，僅在值為 `true` 或 `false` 時才認為該欄位有效：

```php
'foo' => 'boolean:strict'
```


<a name="rule-confirmed"></a>
#### confirmed

驗證欄位必須有一個匹配的 `{field}_confirmation` 欄位。例如，如果驗證欄位是 `password`，則輸入中必須存在匹配的 `password_confirmation` 欄位。

您也可以傳遞自訂的確認欄位名稱。例如，`confirmed:repeat_username` 會要求 `repeat_username` 欄位與驗證欄位匹配。


<a name="rule-contains"></a>
#### contains:_foo_,_bar_,...

驗證欄位必須是一個包含所有給定參數值的陣列。由於此規則通常需要您 `implode` 陣列，因此可以使用 `Rule::contains` 方法來流暢地建構規則：

```php
use Illuminate\Support\Facades\Validator;
use Illuminate\Validation\Rule;

Validator::make($data, [
    'roles' => [
        'required',
        'array',
        Rule::contains(['admin', 'editor']),
    ],
]);
```


<a name="rule-doesnt-contain"></a>
#### doesnt_contain:_foo_,_bar_,...

驗證欄位必須是一個不包含任何給定參數值的陣列。由於此規則通常需要您 `implode` 陣列，因此可以使用 `Rule::doesntContain` 方法來流暢地建構規則：

```php
use Illuminate\Support\Facades\Validator;
use Illuminate\Validation\Rule;

Validator::make($data, [
    'roles' => [
        'required',
        'array',
        Rule::doesntContain(['admin', 'editor']),
    ],
]);
```


<a name="rule-current-password"></a>
#### current_password

驗證欄位必須符合已認證使用者的密碼。您可以使用規則的第一個參數來指定 [認證守衛(authentication guard)](/docs/{{version}}/authentication)：

```php
'password' => 'current_password:api'
```


<a name="rule-date"></a>
#### date

驗證欄位必須是一個有效的、非相對的日期（根據 PHP 的 `strtotime` 函數而定）。


<a name="rule-date-equals"></a>
#### date_equals:_date_

驗證欄位必須等於給定日期。日期將被傳遞給 PHP 的 `strtotime` 函數，以便轉換為有效的 `DateTime` 實例。


<a name="rule-date-format"></a>
#### date_format:_format_,...

驗證欄位必須符合其中一個給定的 _格式_。在驗證欄位時，您應該**僅**使用 `date` 或 `date_format` 其中之一，而不是兩者都使用。此驗證規則支持 PHP [DateTime](https://www.php.net/manual/en/class.datetime.php) 類支持的所有格式。

為了方便起見，可以使用流暢的 `date` 規則建構器來建立基於日期的規則：

```php
use Illuminate\Validation\Rule;

'start_date' => [
    'required',
    Rule::date()->format('Y-m-d'),
],
```


<a name="rule-decimal"></a>
#### decimal:_min_,_max_

驗證欄位必須是數值，且必須包含指定的小數位：

```php
// Must have exactly two decimal places (9.99)...
'price' => 'decimal:2'

// Must have between 2 and 4 decimal places...
'price' => 'decimal:2,4'
```


<a name="rule-declined"></a>
#### declined

驗證欄位必須為 `"no"`、`"off"`、`0`、`"0"`、`false` 或 `"false"`。


<a name="rule-declined-if"></a>
#### declined_if:anotherfield,value,...

如果另一個驗證欄位等於指定值，則驗證欄位必須為 `"no"`、`"off"`、`0`、`"0"`、`false` 或 `"false"`。


<a name="rule-different"></a>
#### different:_field_

驗證欄位的值必須與 _field_ 的值不同。


<a name="rule-digits"></a>
#### digits:_value_

驗證中的整數長度必須正好為 _value_。

<a name="rule-digits-between"></a>
#### digits_between:_min_,_max_

待驗證的整數長度必須介於給定的 _min_ 與 _max_ 之間。


<a name="rule-dimensions"></a>
#### dimensions

待驗證的檔案必須是一張符合規則參數所指定的尺寸限制之圖片：

```php
'avatar' => 'dimensions:min_width=100,min_height=200'
```

可用的限制有：_min\_width_、_max\_width_、_min\_height_、_max\_height_、_width_、_height_、_ratio_。

_ratio_ 限制應表示為寬度除以高度。這可以使用像 `3/2` 這樣的分數或像 `1.5` 這樣的浮點數來指定：

```php
'avatar' => 'dimensions:ratio=3/2'
```

由於此規則需要多個引數，通常使用 `Rule::dimensions` 方法來流暢地構建規則會更方便：

```php
use Illuminate\Support\Facades\Validator;
use Illuminate\Validation\Rule;

Validator::make($data, [
    'avatar' => [
        'required',
        Rule::dimensions()
            ->maxWidth(1000)
            ->maxHeight(500)
            ->ratio(3 / 2),
    ],
]);
```


<a name="rule-distinct"></a>
#### distinct

在驗證陣列時，待驗證的欄位不得有任何重複的值：

```php
'foo.*.id' => 'distinct'
```

Distinct 預設使用鬆散的變數比較。若要使用嚴格比較，您可以在驗證規則定義中加入 `strict` 參數：

```php
'foo.*.id' => 'distinct:strict'
```

您可以在驗證規則的引數中加入 `ignore_case`，以使該規則忽略大小寫差異：

```php
'foo.*.id' => 'distinct:ignore_case'
```


<a name="rule-doesnt-start-with"></a>
#### doesnt_start_with:_foo_,_bar_,...

待驗證的欄位不得以給定的其中一個值開頭。


<a name="rule-doesnt-end-with"></a>
#### doesnt_end_with:_foo_,_bar_,...

待驗證的欄位不得以給定的其中一個值結尾。


<a name="rule-email"></a>
#### email

待驗證的欄位必須符合電子郵件地址格式。此驗證規則使用 [egulias/email-validator](https://github.com/egulias/EmailValidator) 套件來驗證電子郵件地址。預設會套用 `RFCValidation` 驗證器，但您也可以套用其他驗證樣式：

```php
'email' => 'email:rfc,dns'
```

上方的範例將套用 `RFCValidation` 與 `DNSCheckValidation` 驗證。以下是您可以套用的驗證樣式完整列表：

<div class="content-list" markdown="1">

- `rfc`: `RFCValidation` - 根據 [支援的 RFC](https://github.com/egulias/EmailValidator?tab=readme-ov-file#supported-rfcs) 驗證電子郵件地址。
- `strict`: `NoRFCWarningsValidation` - 根據 [支援的 RFC](https://github.com/egulias/EmailValidator?tab=readme-ov-file#supported-rfcs) 驗證電子郵件，若發現警告（例如尾隨句點或多個連續句點）則驗證失敗。
- `dns`: `DNSCheckValidation` - 確保電子郵件地址的網域具有有效的 MX 紀錄。
- `spoof`: `SpoofCheckValidation` - 確保電子郵件地址不包含同形字或欺騙性的 Unicode 字元。
- `filter`: `FilterEmailValidation` - 確保電子郵件地址根據 PHP 的 `filter_var` 函數是有效的。
- `filter_unicode`: `FilterEmailValidation::unicode()` - 確保電子郵件地址根據 PHP 的 `filter_var` 函數是有效的，並允許某些 Unicode 字元。

</div>

為了方便起見，電子郵件驗證規則可以使用流暢的規則構建器來建立：

```php
use Illuminate\Validation\Rule;

$request->validate([
    'email' => [
        'required',
        Rule::email()
            ->rfcCompliant(strict: false)
            ->validateMxRecord()
            ->preventSpoofing()
    ],
]);
```

> [!WARNING]
> `dns` 與 `spoof` 驗證器需要 PHP 的 `intl` 擴充功能。


<a name="rule-encoding"></a>
#### encoding:*encoding_type*

待驗證的欄位必須符合指定的字元編碼。此規則使用 PHP 的 `mb_check_encoding` 函數來驗證給定檔案或字串值的編碼。為了方便起見，`encoding` 規則可以使用 Laravel 的流暢檔案規則構建器來建立：

```php
use Illuminate\Support\Facades\Validator;
use Illuminate\Validation\Rules\File;

Validator::validate($input, [
    'attachment' => [
        'required',
        File::types(['csv'])
            ->encoding('utf-8'),
    ],
]);
```


<a name="rule-ends-with"></a>
#### ends_with:_foo_,_bar_,...

待驗證的欄位必須以給定的其中一個值結尾。


<a name="rule-enum"></a>
#### enum

`Enum` 規則是一個基於類別的規則，用於驗證待驗證的欄位是否包含有效的 enum 值。`Enum` 規則僅接受 enum 的名稱作為其建構子引數。在驗證原始值 (primitive values) 時，應向 `Enum` 規則提供一個 backed Enum：

```php
use App\Enums\ServerStatus;
use Illuminate\Validation\Rule;

$request->validate([
    'status' => [Rule::enum(ServerStatus::class)],
]);
```

`Enum` 規則的 `only` 與 `except` 方法可用於限制哪些 enum 案例應被視為有效：

```php
Rule::enum(ServerStatus::class)
    ->only([ServerStatus::Pending, ServerStatus::Active]);

Rule::enum(ServerStatus::class)
    ->except([ServerStatus::Pending, ServerStatus::Active]);
```

`when` 方法可用於條件式地修改 `Enum` 規則：

```php
use Illuminate\Support\Facades\Auth;
use Illuminate\Validation\Rule;

Rule::enum(ServerStatus::class)
    ->when(
        Auth::user()->isAdmin(),
        fn ($rule) => $rule->only(...),
        fn ($rule) => $rule->only(...),
    );
```


<a name="rule-exclude"></a>
#### exclude

待驗證的欄位將從 `validate` 與 `validated` 方法回傳的請求資料中排除。


<a name="rule-exclude-if"></a>
#### exclude_if:_anotherfield_,_value_

如果 _anotherfield_ 欄位等於 _value_，則待驗證的欄位將從 `validate` 與 `validated` 方法回傳的請求資料中排除。

如果需要複雜的條件式排除邏輯，您可以使用 `Rule::excludeIf` 方法。此方法接受一個布林值或一個閉包。當提供閉包時，該閉包應回傳 `true` 或 `false` 以指示待驗證的欄位是否應被排除：

```php
use Illuminate\Support\Facades\Validator;
use Illuminate\Validation\Rule;

Validator::make($request->all(), [
    'role_id' => Rule::excludeIf($request->user()->is_admin),
]);

Validator::make($request->all(), [
    'role_id' => Rule::excludeIf(fn () => $request->user()->is_admin),
]);
```


<a name="rule-exclude-unless"></a>
#### exclude_unless:_anotherfield_,_value_

除非 _anotherfield_ 欄位等於 _value_，否則待驗證的欄位將從 `validate` 與 `validated` 方法回傳的請求資料中排除。如果 _value_ 為 `null` (`exclude_unless:name,null`)，則除非比較欄位為 `null` 或比較欄位在請求資料中缺失，否則待驗證的欄位將被排除。

如果需要複雜的條件式排除邏輯，您可以使用 `Rule::excludeUnless` 方法。此方法接受一個布林值或一個閉包。當提供閉包時，該閉包應回傳 `true` 或 `false` 以指示待驗證的欄位是否不應被排除：

```php
use Illuminate\Support\Facades\Validator;
use Illuminate\Validation\Rule;

Validator::make($request->all(), [
    'role_id' => Rule::excludeUnless($request->user()->is_admin),
]);

Validator::make($request->all(), [
    'role_id' => Rule::excludeUnless(fn () => $request->user()->is_admin),
]);
```


<a name="rule-exclude-with"></a>
#### exclude_with:_anotherfield_

如果 _anotherfield_ 欄位存在，則待驗證的欄位將從 `validate` 與 `validated` 方法回傳的請求資料中排除。


<a name="rule-exclude-without"></a>
#### exclude_without:_anotherfield_

如果 _anotherfield_ 欄位不存在，則待驗證的欄位將從 `validate` 與 `validated` 方法回傳的請求資料中排除。

<a name="rule-exists"></a>
#### exists:_table_,_column_

被驗證的欄位必須存在於指定的資料庫資料表中。


<a name="basic-usage-of-exists-rule"></a>
#### Exists 規則的基本用法

```php
'state' => 'exists:states'
```

如果沒有指定 `column` 選項，將使用欄位名稱。因此，在此例中，該規則將驗證 `states` 資料庫資料表中是否包含一筆紀錄，其 `state` 欄位的值與請求中的 `state` 屬性值相符。


<a name="specifying-a-custom-column-name"></a>
#### 指定自訂欄位名稱

您可以透過將資料庫欄位名稱放在資料表名稱之後，來明確指定驗證規則應使用的資料庫欄位名稱：

```php
'state' => 'exists:states,abbreviation'
```

有時，您可能需要指定 `exists` 查詢要使用的特定資料庫連線。您可以透過在資料表名稱前加上連線名稱來達成：

```php
'email' => 'exists:connection.staff,email'
```

您也可以直接指定用於確定資料表名稱的 Eloquent 模型，而不是直接指定資料表名稱：

```php
'user_id' => 'exists:App\Models\User,id'
```

如果您想要自訂驗證規則所執行的查詢，可以使用 `Rule` 類別來流暢地定義規則。在此例中，我們還將驗證規則指定為陣列，而不是使用 `|` 字元來分隔它們：

```php
use Illuminate\Database\Query\Builder;
use Illuminate\Support\Facades\Validator;
use Illuminate\Validation\Rule;

Validator::make($data, [
    'email' => [
        'required',
        Rule::exists('staff')->where(function (Builder $query) {
            $query->where('account_id', 1);
        }),
    ],
]);
```

您可以透過將欄位名稱作為 `exists` 方法的第二個引數，來明確指定由 `Rule::exists` 方法生成的 `exists` 規則應使用的資料庫欄位名稱：

```php
'state' => Rule::exists('states', 'abbreviation'),
```

有時，您可能希望驗證一組值是否都存在於資料庫中。您可以將 `exists` 和 [array](#rule-array) 規則同時新增至被驗證的欄位中：

```php
'states' => ['array', Rule::exists('states', 'abbreviation')],
```

當這兩個規則都被分配給一個欄位時，Laravel 會自動建立單一查詢，以確定所有給定的值都存在於指定的資料表中。


<a name="rule-extensions"></a>
#### extensions:_foo_,_bar_,...

被驗證的檔案必須具有對應於列出擴展名之一的使用者指定擴展名：

```php
'photo' => ['required', 'extensions:jpg,png'],
```

> [!WARNING]
> 您絕對不應該僅依賴使用者指定的擴展名來驗證檔案。此規則通常應與 [mimes](#rule-mimes) 或 [mimetypes](#rule-mimetypes) 規則結合使用。


<a name="rule-file"></a>
#### file

被驗證的欄位必須是一個成功上傳的檔案。


<a name="rule-filled"></a>
#### filled

當被驗證的欄位存在時，其內容不能為空。


<a name="rule-gt"></a>
#### gt:_field_

被驗證的欄位必須大於給定的 _field_ 或 _value_。兩個欄位必須是相同的型別。字串、數值、陣列和檔案的評估方式與 [size](#rule-size) 規則相同。


<a name="rule-gte"></a>
#### gte:_field_

被驗證的欄位必須大於或等於給定的 _field_ 或 _value_。兩個欄位必須是相同的型別。字串、數值、陣列和檔案的評估方式與 [size](#rule-size) 規則相同。


<a name="rule-hex-color"></a>
#### hex_color

被驗證的欄位必須包含一個有效的 [十六進位](https://developer.mozilla.org/en-US/docs/Web/CSS/hex-color) 格式的顏色值。


<a name="rule-image"></a>
#### image

被驗證的檔案必須是一張圖片 (jpg, jpeg, png, bmp, gif 或 webp)。

> [!WARNING]
> 預設情況下，由於 XSS 漏洞的可能性，image 規則不允許 SVG 檔案。如果您需要允許 SVG 檔案，可以為 `image` 規則提供 `allow_svg` 指令 (`image:allow_svg`)。


<a name="rule-in"></a>
#### in:_foo_,_bar_,...

被驗證的欄位必須包含在給定的值列表中。由於此規則通常需要您 `implode` 一個陣列，因此可以使用 `Rule::in` 方法來流暢地構建規則：

```php
use Illuminate\Support\Facades\Validator;
use Illuminate\Validation\Rule;

Validator::make($data, [
    'zones' => [
        'required',
        Rule::in(['first-zone', 'second-zone']),
    ],
]);
```

當 `in` 規則與 `array` 規則結合使用時，輸入陣列中的每個值都必須存在於提供給 `in` 規則的值列表中。在以下範例中，輸入陣列中的 `LAS` 機場代碼是無效的，因為它未包含在提供給 `in` 規則的機場列表中：

```php
use Illuminate\Support\Facades\Validator;
use Illuminate\Validation\Rule;

$input = [
    'airports' => ['NYC', 'LAS'],
];

Validator::make($input, [
    'airports' => [
        'required',
        'array',
    ],
    'airports.*' => Rule::in(['NYC', 'LIT']),
]);
```


<a name="rule-in-array"></a>
#### in_array:_anotherfield_.*

被驗證的欄位必須存在於 _anotherfield_ 的值中。


<a name="rule-in-array-keys"></a>
#### in_array_keys:_value_.*

被驗證的欄位必須是一個陣列，且該陣列中至少有一個給定的 _values_ 作為鍵：

```php
'config' => 'array|in_array_keys:timezone'
```


<a name="rule-integer"></a>
#### integer

被驗證的欄位必須是一個整數。

您可以使用 `strict` 參數來指定僅在型別為 `integer` 時才視該欄位為有效。具有整數值的字串將被視為無效：

```php
'age' => 'integer:strict'
```

> [!WARNING]
> 此驗證規則並不驗證輸入是否為 "integer" 變數型別，而僅驗證輸入是否為 PHP `FILTER_VALIDATE_INT` 規則所接受的型別。如果您需要驗證輸入是否為數字，請將此規則與 [the `numeric` validation rule](#rule-numeric) 結合使用。


<a name="rule-ip"></a>
#### ip

被驗證的欄位必須是一個 IP 位址。


<a name="ipv4"></a>
#### ipv4

被驗證的欄位必須是一個 IPv4 位址。


<a name="ipv6"></a>
#### ipv6

被驗證的欄位必須是一個 IPv6 位址。


<a name="rule-json"></a>
#### json

被驗證的欄位必須是一個有效的 JSON 字串。


<a name="rule-lt"></a>
#### lt:_field_

被驗證的欄位必須小於給定的 _field_。兩個欄位必須是相同的型別。字串、數值、陣列和檔案的評估方式與 [size](#rule-size) 規則相同。


<a name="rule-lte"></a>
#### lte:_field_

被驗證的欄位必須小於或等於給定的 _field_。兩個欄位必須是相同的型別。字串、數值、陣列和檔案的評估方式與 [size](#rule-size) 規則相同。


<a name="rule-lowercase"></a>
#### lowercase

被驗證的欄位必須為小寫。


<a name="rule-list"></a>
#### list

被驗證的欄位必須是一個屬於清單 (list) 的陣列。如果陣列的鍵由從 0 到 `count($array) - 1` 的連續數字組成，則該陣列被視為清單。


<a name="rule-mac"></a>
#### mac_address

被驗證的欄位必須是一個 MAC 位址。


<a name="rule-max"></a>
#### max:_value_

被驗證的欄位必須小於或等於最大值 _value_。字串、數值、陣列和檔案的評估方式與 [size](#rule-size) 規則相同。


<a name="rule-max-digits"></a>
#### max_digits:_value_

被驗證的整數必須具有最大長度 _value_。

<a name="rule-mimetypes"></a>
#### mimetypes:_text/plain_,...

待驗證的檔案必須符合其中一個給定的 MIME 類型：

```php
'video' => 'mimetypes:video/avi,video/mpeg,video/quicktime',

'media' => 'mimetypes:image/*,video/*',
```

為了確定上傳檔案的 MIME 類型，框架會讀取檔案內容並嘗試推測其 MIME 類型，這可能與客戶端提供的 MIME 類型不同。


<a name="rule-mimes"></a>
#### mimes:_foo_,_bar_,...

待驗證的檔案必須具有與列出的擴展名之一相對應的 MIME 類型：

```php
'photo' => 'mimes:jpg,bmp,png'
```

儘管您只需要指定擴展名，但此規則實際上是透過讀取檔案內容並推測其 MIME 類型來驗證檔案的 MIME 類型。完整的 MIME 類型及其對應擴展名列表可以在以下位置找到：

[https://svn.apache.org/repos/asf/httpd/httpd/trunk/docs/conf/mime.types](https://svn.apache.org/repos/asf/httpd/httpd/trunk/docs/conf/mime.types)


<a name="mime-types-and-extensions"></a>
#### MIME 類型與擴展名

此驗證規則不會驗證 MIME 類型與使用者指定給檔案的擴展名是否一致。例如，`mimes:png` 驗證規則會將包含有效 PNG 內容的檔案視為有效的 PNG 圖片，即使該檔案被命名為 `photo.txt`。如果您想要驗證使用者指定的檔案擴展名，可以使用 [extensions](#rule-extensions) 規則。


<a name="rule-min"></a>
#### min:_value_

待驗證的欄位必須具有最小 _value_。字串、數字、陣列與檔案的評估方式與 [size](#rule-size) 規則相同。


<a name="rule-min-digits"></a>
#### min_digits:_value_

待驗證的整數必須具有最小長度 _value_。


<a name="rule-multiple-of"></a>
#### multiple_of:_value_

待驗證的欄位必須是 _value_ 的倍數。


<a name="rule-missing"></a>
#### missing

待驗證的欄位不得出現在輸入資料中。


<a name="rule-missing-if"></a>
#### missing_if:_anotherfield_,_value_,...

如果 _anotherfield_ 欄位等於任何 _value_，則待驗證的欄位不得出現。


<a name="rule-missing-unless"></a>
#### missing_unless:_anotherfield_,_value_

除非 _anotherfield_ 欄位等於任何 _value_，否則待驗證的欄位不得出現。


<a name="rule-missing-with"></a>
#### missing_with:_foo_,_bar_,...

_僅當_ 其他任何指定欄位出現時，待驗證的欄位不得出現。


<a name="rule-missing-with-all"></a>
#### missing_with_all:_foo_,_bar_,...

_僅當_ 所有其他指定欄位都出現時，待驗證的欄位不得出現。


<a name="rule-not-in"></a>
#### not_in:_foo_,_bar_,...

待驗證的欄位不得包含在給定的值列表中。可以使用 `Rule::notIn` 方法來流暢地建構此規則：

```php
use Illuminate\Validation\Rule;

Validator::make($data, [
    'toppings' => [
        'required',
        Rule::notIn(['sprinkles', 'cherries']),
    ],
]);
```


<a name="rule-not-regex"></a>
#### not_regex:_pattern_

待驗證的欄位不得符合給定的正規表達式。

在內部，此規則使用 PHP 的 `preg_match` 函數。指定的模式應遵守 `preg_match` 所要求的相同格式，因此也應包含有效的定界符。例如：`'email' => 'not_regex:/^.+$/i'`。

> [!WARNING]
> 使用 `regex` / `not_regex` 模式時，可能需要使用陣列來指定驗證規則，而不是使用 `|` 定界符，尤其是當正規表達式包含 `|` 字元時。


<a name="rule-nullable"></a>
#### nullable

待驗證的欄位可以是 `null`。


<a name="rule-numeric"></a>
#### numeric

待驗證的欄位必須是 [數字(numeric)](https://www.php.net/manual/en/function.is-numeric.php)。

您可以使用 `strict` 參數，僅在值為整數或浮點數類型時才將該欄位視為有效。數字字串將被視為無效：

```php
'amount' => 'numeric:strict'
```


<a name="rule-present"></a>
#### present

待驗證的欄位必須存在於輸入資料中。


<a name="rule-present-if"></a>
#### present_if:_anotherfield_,_value_,...

如果 _anotherfield_ 欄位等於任何 _value_，則待驗證的欄位必須存在。


<a name="rule-present-unless"></a>
#### present_unless:_anotherfield_,_value_

除非 _anotherfield_ 欄位等於任何 _value_，否則待驗證的欄位必須存在。


<a name="rule-present-with"></a>
#### present_with:_foo_,_bar_,...

_僅當_ 其他任何指定欄位存在時，待驗證的欄位必須存在。


<a name="rule-present-with-all"></a>
#### present_with_all:_foo_,_bar_,...

_僅當_ 所有其他指定欄位都存在時，待驗證的欄位必須存在。


<a name="rule-prohibited"></a>
#### prohibited

待驗證的欄位必須缺失或為空。如果欄位符合以下任一條件，則被視為「空」：

<div class="content-list" markdown="1">

- 值為 `null`。
- 值為空字串。
- 值為空陣列或空的 `Countable` 物件。
- 值為路徑為空的上傳檔案。

</div>


<a name="rule-prohibited-if"></a>
#### prohibited_if:_anotherfield_,_value_,...

如果 _anotherfield_ 欄位等於任何 _value_，則待驗證的欄位必須缺失或為空。如果欄位符合以下任一條件，則被視為「空」：

<div class="content-list" markdown="1">

- 值為 `null`。
- 值為空字串。
- 值為空陣列或空的 `Countable` 物件。
- 值為路徑為空的上傳檔案。

</div>

如果需要複雜的條件式禁止邏輯，您可以使用 `Rule::prohibitedIf` 方法。此方法接受一個布林值或一個閉包。當給定閉包時，閉包應回傳 `true` 或 `false` 以指示待驗證的欄位是否應被禁止：

```php
use Illuminate\Support\Facades\Validator;
use Illuminate\Validation\Rule;

Validator::make($request->all(), [
    'role_id' => Rule::prohibitedIf($request->user()->is_admin),
]);

Validator::make($request->all(), [
    'role_id' => Rule::prohibitedIf(fn () => $request->user()->is_admin),
]);
```

<a name="rule-prohibited-if-accepted"></a>
#### prohibited_if_accepted:_anotherfield_,...

如果 _anotherfield_ 欄位等於 `"yes"`, `"on"`, `1`, `"1"`, `true` 或 `"true"`，則待驗證的欄位必須缺失或為空。


<a name="rule-prohibited-if-declined"></a>
#### prohibited_if_declined:_anotherfield_,...

如果 _anotherfield_ 欄位等於 `"no"`, `"off"`, `0`, `"0"`, `false` 或 `"false"`，則待驗證的欄位必須缺失或為空。


<a name="rule-prohibited-unless"></a>
#### prohibited_unless:_anotherfield_,_value_,...

除非 _anotherfield_ 欄位等於任何 _value_，否則待驗證的欄位必須缺失或為空。如果欄位符合以下任一條件，則被視為「空」：

<div class="content-list" markdown="1">

- 值為 `null`。
- 值為空字串。
- 值為空陣列或空的 `Countable` 物件。
- 值為路徑為空的上傳檔案。

</div>

如果需要複雜的條件式禁止邏輯，您可以使用 `Rule::prohibitedUnless` 方法。此方法接受一個布林值或一個閉包。當給定閉包時，閉包應回傳 `true` 或 `false` 以指示待驗證的欄位是否不應被禁止：

```php
use Illuminate\Support\Facades\Validator;
use Illuminate\Validation\Rule;

Validator::make($request->all(), [
    'role_id' => Rule::prohibitedUnless($request->user()->is_admin),
]);

Validator::make($request->all(), [
    'role_id' => Rule::prohibitedUnless(fn () => $request->user()->is_admin),
]);
```

<a name="rule-prohibits"></a>
#### prohibits:_anotherfield_,...

如果被驗證的欄位不是缺失或空的，則 _anotherfield_ 中的所有欄位必須是缺失或空的。如果欄位符合以下任一條件，則被視為「空的」：

<div class="content-list" markdown="1">

- 數值為 `null`。
- 數值為空字串。
- 數值為空陣列或空的 `Countable` 物件。
- 數值為路徑為空的上傳檔案。

</div>


<a name="rule-regex"></a>
#### regex:_pattern_

被驗證的欄位必須符合給定的正規表達式。

在內部，此規則使用 PHP 的 `preg_match` 函數。指定的模式應遵守 `preg_match` 所要求的格式，因此也必須包含有效的定界符 (delimiters)。例如：`'email' => 'regex:/^.+@.+$/i'`。

> [!WARNING]
> 當使用 `regex` / `not_regex` 模式時，可能需要將規則指定為陣列而非使用 `|` 定界符，特別是當正規表達式本身包含 `|` 字元時。


<a name="rule-required"></a>
#### required

被驗證的欄位必須存在於輸入資料中且不能為空。如果欄位符合以下任一條件，則被視為「空的」：

<div class="content-list" markdown="1">

- 數值為 `null`。
- 數值為空字串。
- 數值為空陣列或空的 `Countable` 物件。
- 數值為沒有路徑的上傳檔案。

</div>


<a name="rule-required-if"></a>
#### required_if:_anotherfield_,_value_,...

如果 _anotherfield_ 欄位等於任何 _value_，則被驗證的欄位必須存在且不能為空。

如果您想要為 `required_if` 規則建構更複雜的條件，可以使用 `Rule::requiredIf` 方法。此方法接受一個布林值或一個閉包。當傳入閉包時，該閉包應返回 `true` 或 `false` 以指示被驗證的欄位是否為必填：

```php
use Illuminate\Support\Facades\Validator;
use Illuminate\Validation\Rule;

Validator::make($request->all(), [
    'role_id' => Rule::requiredIf($request->user()->is_admin),
]);

Validator::make($request->all(), [
    'role_id' => Rule::requiredIf(fn () => $request->user()->is_admin),
]);
```


<a name="rule-required-if-accepted"></a>
#### required_if_accepted:_anotherfield_,...

如果 _anotherfield_ 欄位等於 `"yes"`, `"on"`, `1`, `"1"`, `true`, 或 `"true"`，則被驗證的欄位必須存在且不能為空。


<a name="rule-required-if-declined"></a>
#### required_if_declined:_anotherfield_,...

如果 _anotherfield_ 欄位等於 `"no"`, `"off"`, `0`, `"0"`, `false`, 或 `"false"`，則被驗證的欄位必須存在且不能為空。


<a name="rule-required-unless"></a>
#### required_unless:_anotherfield_,_value_,...

除非 _anotherfield_ 欄位等於任何 _value_，否則被驗證的欄位必須存在且不能為空。這也意味著除非 _value_ 為 `null`，否則 _anotherfield_ 必須存在於請求資料中。如果 _value_ 為 `null` (`required_unless:name,null`)，則除非比較欄位為 `null` 或比較欄位在請求資料中缺失，否則被驗證的欄位將是必填的。

如果您想要為 `required_unless` 規則建構更複雜的條件，可以使用 `Rule::requiredUnless` 方法。此方法接受一個布林值或一個閉包。當傳入閉包時，該閉包應返回 `true` 或 `false` 以指示被驗證的欄位是否非必填：

```php
use Illuminate\Support\Facades\Validator;
use Illuminate\Validation\Rule;

Validator::make($request->all(), [
    'role_id' => Rule::requiredUnless($request->user()->is_admin),
]);

Validator::make($request->all(), [
    'role_id' => Rule::requiredUnless(fn () => $request->user()->is_admin),
]);
```


<a name="rule-required-with"></a>
#### required_with:_foo_,_bar_,...

_僅當_ 其他指定的任何欄位存在且不為空時，被驗證的欄位必須存在且不能為空。


<a name="rule-required-with-all"></a>
#### required_with_all:_foo_,_bar_,...

_僅當_ 其他所有指定的欄位都存在且不為空時，被驗證的欄位必須存在且不能為空。


<a name="rule-required-without"></a>
#### required_without:_foo_,_bar_,...

_僅當_ 其他指定的任何欄位為空或不存在時，被驗證的欄位必須存在且不能為空。


<a name="rule-required-without-all"></a>
#### required_without_all:_foo_,_bar_,...

_僅當_ 其他所有指定的欄位都為空或不存在時，被驗證的欄位必須存在且不能為空。


<a name="rule-required-array-keys"></a>
#### required_array_keys:_foo_,_bar_,...

被驗證的欄位必須是一個陣列，且必須包含至少指定的鍵 (keys)。


<a name="rule-same"></a>
#### same:_field_

給定的 _field_ 必須與被驗證的欄位相匹配。


<a name="rule-size"></a>
#### size:_value_

被驗證的欄位必須具有與給定 _value_ 相匹配的大小。對於字串資料，_value_ 對應於字元數。對於數值資料，_value_ 對應於給定的整數值（該屬性還必須具有 `numeric` 或 `integer` 規則）。對於陣列，_size_ 對應於陣列的 `count`。對於檔案，_size_ 對應於以 KB 為單位的檔案大小。讓我們看一些範例：

```php
// Validate that a string is exactly 12 characters long...
'title' => 'size:12';

// Validate that a provided integer equals 10...
'seats' => 'integer|size:10';

// Validate that an array has exactly 5 elements...
'tags' => 'array|size:5';

// Validate that an uploaded file is exactly 512 kilobytes...
'image' => 'file|size:512';
```


<a name="rule-starts-with"></a>
#### starts_with:_foo_,_bar_,...

被驗證的欄位必須以給定值中的其中一個開頭。


<a name="rule-string"></a>
#### string

被驗證的欄位必須是一個字串。如果您希望允許該欄位也可以為 `null`，則應將 `nullable` 規則分配給該欄位。

為了方便起見，字串驗證規則也可以使用流暢的 `Rule::string()` 規則建構器來建構：

```php
use Illuminate\Validation\Rule;

'title' => [
    'required',
    Rule::string()
        ->min(3)
        ->max(255)
        ->alphaDash(ascii: true),
],
```

字串規則建構器提供了常用字串約束的方法，包括 `alpha`, `alphaDash`, `alphaNumeric`, `ascii`, `between`, `doesntEndWith`, `doesntStartWith`, `endsWith`, `exactly`, `lowercase`, `max`, `min`, `startsWith`, 以及 `uppercase`。由於規則建構器是可條件化的，您也可以使用 `when` 和 `unless` 方法來條件式地應用約束。


<a name="rule-timezone"></a>
#### timezone

根據 `DateTimeZone::listIdentifiers` 方法，被驗證的欄位必須是一個有效的時區識別碼。

此驗證規則也可以提供 [ `DateTimeZone::listIdentifiers` 方法所接受](https://www.php.net/manual/en/datetimezone.listidentifiers.php) 的引數：

```php
'timezone' => 'required|timezone:all';

'timezone' => 'required|timezone:Africa';

'timezone' => 'required|timezone:per_country,US';
```

<a name="rule-unique"></a>
#### unique:_table_,_column_

待驗證的欄位必須不存在於給定的資料庫資料表中。

**指定自訂的資料表 / 欄位名稱：**

您可以指定 Eloquent 模型來決定資料表名稱，而不是直接指定資料表名稱：

```php
'email' => 'unique:App\Models\User,email_address'
```

`column` 選項可用於指定欄位對應的資料庫欄位。如果未指定 `column` 選項，將使用待驗證欄位的名稱。

```php
'email' => 'unique:users,email_address'
```

**指定自訂的資料庫連線**

有時您可能需要為 Validator 執行的資料庫查詢設定自訂連線。若要達成此目的，您可以在資料表名稱前加上連線名稱：

```php
'email' => 'unique:connection.users,email_address'
```

**強制 unique 規則忽略給定的 ID：**

有時您可能希望在 unique 驗證期間忽略給定的 ID。例如，考慮一個包含使用者名稱、電子郵件地址和位置的「更新設定」畫面。您可能想要驗證電子郵件地址是否唯一。然而，如果使用者僅更改了名稱欄位而未更改電子郵件欄位，您不希望拋出驗證錯誤，因為使用者已經是該電子郵件地址的所有者。

若要指示驗證器忽略使用者的 ID，我們將使用 `Rule` 類別來流暢地定義規則。在此範例中，我們還將驗證規則指定為陣列，而不是使用 `|` 字元來分隔規則：

```php
use Illuminate\Support\Facades\Validator;
use Illuminate\Validation\Rule;

Validator::make($data, [
    'email' => [
        'required',
        Rule::unique('users')->ignore($user->id),
    ],
]);
```

> [!WARNING]
> 您絕不應將任何由使用者控制的請求輸入傳遞給 `ignore` 方法。相反地，您應該僅傳遞系統產生的唯一 ID，例如來自 Eloquent 模型實例的自動遞增 ID 或 UUID。否則，您的應用程式將容易受到 SQL 注入攻擊。

您也可以將整個模型實例傳遞給 `ignore` 方法，而不是傳遞模型鍵值。Laravel 將自動從模型中提取鍵值：

```php
Rule::unique('users')->ignore($user)
```

如果您的資料表使用的主鍵欄位名稱不是 `id`，您可以在呼叫 `ignore` 方法時指定欄位名稱：

```php
Rule::unique('users')->ignore($user->id, 'user_id')
```

預設情況下，`unique` 規則將檢查與被驗證屬性名稱相符之欄位的唯一性。然而，您可以將不同的欄位名稱作為 `unique` 方法的第二個引數傳遞：

```php
Rule::unique('users', 'email_address')->ignore($user->id)
```

**新增額外的 Where 子句：**

您可以使用 `where` 方法自訂查詢來指定額外的查詢條件。例如，讓我們新增一個查詢條件，將查詢範圍限定為僅搜尋 `account_id` 欄位值為 `1` 的記錄：

```php
'email' => Rule::unique('users')->where(fn (Builder $query) => $query->where('account_id', 1))
```

**在唯一性檢查中忽略軟刪除記錄：**

預設情況下，unique 規則在判定唯一性時會包含軟刪除的記錄。若要從唯一性檢查中排除軟刪除的記錄，您可以呼叫 `withoutTrashed` 方法：

```php
Rule::unique('users')->withoutTrashed();
```

如果您的模型對軟刪除記錄使用的欄位名稱不是 `deleted_at`，您可以在呼叫 `withoutTrashed` 方法時提供欄位名稱：

```php
Rule::unique('users')->withoutTrashed('was_deleted_at');
```


<a name="rule-uppercase"></a>
#### uppercase

待驗證的欄位必須為大寫。


<a name="rule-url"></a>
#### url

待驗證的欄位必須是一個有效的 URL。

如果您想要指定應被視為有效的 URL 協定，您可以將協定作為驗證規則的參數傳遞：

```php
'url' => 'url:http,https',

'game' => 'url:minecraft,steam',
```


<a name="rule-ulid"></a>
#### ulid

待驗證的欄位必須是一個有效的 [Universally Unique Lexicographically Sortable Identifier](https://github.com/ulid/spec) (ULID)。


<a name="rule-uuid"></a>
#### uuid

待驗證的欄位必須是一個有效的 RFC 9562 (版本 1, 3, 4, 5, 6, 7 或 8) 通用唯一識別碼 (UUID)。

您也可以驗證給定的 UUID 是否符合特定版本的 UUID 規範：

```php
'uuid' => 'uuid:4'
```

<a name="conditionally-adding-rules"></a>
## 條件式新增規則


<a name="skipping-validation-when-fields-have-certain-values"></a>
#### 當欄位具有特定值時跳過驗證

您可能偶爾希望在另一個欄位具有特定值時，不對給定欄位進行驗證。您可以使用 `exclude_if` 驗證規則來達成此目的。在此範例中，如果 `has_appointment` 欄位的值為 `false`，則 `appointment_date` 和 `doctor_name` 欄位將不會被驗證：

```php
use Illuminate\Support\Facades\Validator;

$validator = Validator::make($data, [
    'has_appointment' => 'required|boolean',
    'appointment_date' => 'exclude_if:has_appointment,false|required|date',
    'doctor_name' => 'exclude_if:has_appointment,false|required|string',
]);
```

或者，您可以使用 `exclude_unless` 規則，除非另一個欄位具有特定值，否則不對給定欄位進行驗證：

```php
$validator = Validator::make($data, [
    'has_appointment' => 'required|boolean',
    'appointment_date' => 'exclude_unless:has_appointment,true|required|date',
    'doctor_name' => 'exclude_unless:has_appointment,true|required|string',
]);
```


<a name="validating-when-present"></a>
#### 僅在存在時驗證

在某些情況下，您可能希望**僅**在該欄位存在於被驗證的資料中時，才對其執行驗證檢查。要快速達成此目的，請將 `sometimes` 規則新增至您的規則清單中：

```php
$validator = Validator::make($data, [
    'email' => 'sometimes|required|email',
]);
```

在上述範例中，`email` 欄位僅在 `$data` 陣列中存在時才會被驗證。

> [!NOTE]
> 如果您嘗試驗證一個應該始終存在但可能為空的欄位，請參閱[關於可選欄位的注意事項](#a-note-on-optional-fields)。


<a name="complex-conditional-validation"></a>
#### 複雜的條件式驗證

有時您可能希望根據更複雜的條件邏輯來新增驗證規則。例如，您可能希望僅在另一個欄位的值大於 100 時，才要求某個給定欄位必填。或者，您可能需要僅在另一個欄位存在時，才要求兩個欄位具有特定值。新增這些驗證規則並不困難。首先，使用永不變動的 _靜態規則_ 建立一個 `Validator` 實例：

```php
use Illuminate\Support\Facades\Validator;

$validator = Validator::make($request->all(), [
    'email' => 'required|email',
    'games' => 'required|integer|min:0',
]);
```

假設我們的網頁應用程式是為遊戲收藏家設計的。如果一名遊戲收藏家在我們的應用程式中註冊，且他們擁有超過 100 款遊戲，我們希望他們解釋為什麼擁有這麼多遊戲。例如，也許他們經營一家遊戲轉售店，或者他們單純享受收藏遊戲的樂趣。要條件式地新增此要求，我們可以使用 `Validator` 實例上的 `sometimes` 方法。

```php
use Illuminate\Support\Fluent;

$validator->sometimes('reason', 'required|max:500', function (Fluent $input) {
    return $input->games >= 100;
});
```

傳遞給 `sometimes` 方法的第一個引數是我們要進行條件式驗證的欄位名稱。第二個引數是我們要新增的規則清單。如果作為第三個引數傳遞的閉包回傳 `true`，則會新增這些規則。這個方法讓建立複雜的條件式驗證變得輕而易舉。您甚至可以一次為多個欄位新增條件式驗證：

```php
$validator->sometimes(['reason', 'cost'], 'required', function (Fluent $input) {
    return $input->games >= 100;
});
```

> [!NOTE]
> 傳遞給閉包的 `$input` 參數將是一個 `Illuminate\Support\Fluent` 實例，可用於存取正在驗證的輸入與檔案。


<a name="complex-conditional-array-validation"></a>
#### 複雜的條件式陣列驗證

有時您可能希望根據同一個巢狀陣列中的另一個欄位來驗證某個欄位，而您並不知道其索引。在這種情況下，您可以讓閉包接收第二個引數，該引數將是目前正在被驗證的陣列中的單個項目：

```php
$input = [
    'channels' => [
        [
            'type' => 'email',
            'address' => 'abigail@example.com',
        ],
        [
            'type' => 'url',
            'address' => 'https://example.com',
        ],
    ],
];

$validator->sometimes('channels.*.address', 'email', function (Fluent $input, Fluent $item) {
    return $item->type === 'email';
});

$validator->sometimes('channels.*.address', 'url', function (Fluent $input, Fluent $item) {
    return $item->type !== 'email';
});
```

與傳遞給閉包的 `$input` 參數一樣，當屬性資料為陣列時，`$item` 參數是一個 `Illuminate\Support\Fluent` 實例；否則，它是一個字串。

<a name="validating-arrays"></a>
## 驗證陣列

如 [array 驗證規則文件](#rule-array) 中所述，`array` 規則接受一個允許的陣列鍵名清單。如果陣列中存在任何額外的鍵名，驗證將會失敗：

```php
use Illuminate\Support\Facades\Validator;

$input = [
    'user' => [
        'name' => 'Taylor Otwell',
        'username' => 'taylorotwell',
        'admin' => true,
    ],
];

Validator::make($input, [
    'user' => 'array:name,username',
]);
```

一般而言，您應該始終指定允許存在於陣列中的鍵名。否則，驗證器的 `validate` 與 `validated` 方法將會回傳所有已驗證的資料，包括該陣列及其所有鍵名，即使這些鍵名沒有被其他巢狀陣列驗證規則驗證過。


<a name="validating-nested-array-input"></a>
### 驗證巢狀陣列輸入

驗證基於陣列的巢狀表單輸入欄位並不困難。您可以使用「點號表示法 (dot notation)」來驗證陣列中的屬性。例如，如果傳入的 HTTP 請求包含一個 `photos[profile]` 欄位，您可以像這樣驗證它：

```php
use Illuminate\Support\Facades\Validator;

$validator = Validator::make($request->all(), [
    'photos.profile' => 'required|image',
]);
```

您也可以驗證陣列中的每個元素。例如，若要驗證給定陣列輸入欄位中的每個電子郵件都是唯一的，您可以執行以下操作：

```php
$validator = Validator::make($request->all(), [
    'users.*.email' => 'email|unique:users',
    'users.*.first_name' => 'required_with:users.*.last_name',
]);
```

同樣地，在[語言檔案中指定自訂驗證訊息](#custom-messages-for-specific-attributes)時，您可以使用 `*` 字元，讓為基於陣列的欄位使用單一驗證訊息變得非常簡單：

```php
'custom' => [
    'users.*.email' => [
        'unique' => 'Each user must have a unique email address',
    ]
],
```


<a name="accessing-nested-array-data"></a>
#### 存取巢狀陣列資料

有時候在為屬性指定驗證規則時，您可能需要存取特定巢狀陣列元素的值。您可以使用 `Rule::forEach` 方法來實現。`forEach` 方法接受一個閉包，該閉包會在驗證陣列屬性的每次迭代中被呼叫，並接收該屬性的值以及明確且完整展開的屬性名稱。該閉包應回傳一個要分配給陣列元素的規則陣列：

```php
use App\Rules\HasPermission;
use Illuminate\Support\Facades\Validator;
use Illuminate\Validation\Rule;

$validator = Validator::make($request->all(), [
    'companies.*.id' => Rule::forEach(function (string|null $value, string $attribute) {
        return [
            Rule::exists(Company::class, 'id'),
            new HasPermission('manage-company', $value),
        ];
    }),
]);
```


<a name="error-message-indexes-and-positions"></a>
### 錯誤訊息的索引與位置

驗證陣列時，您可能希望在應用程式顯示的錯誤訊息中引用失敗驗證項目的索引或位置。為了實現這一點，您可以在[自訂驗證訊息](#manual-customizing-the-error-messages)中包含 `:index` (從 `0` 開始)、`:position` (從 `1` 開始) 或 `:ordinal-position` (從 `1st` 開始) 佔位符：

```php
use Illuminate\Support\Facades\Validator;

$input = [
    'photos' => [
        [
            'name' => 'BeachVacation.jpg',
            'description' => 'A photo of my beach vacation!',
        ],
        [
            'name' => 'GrandCanyon.jpg',
            'description' => '',
        ],
    ],
];

Validator::validate($input, [
    'photos.*.description' => 'required',
], [
    'photos.*.description.required' => 'Please describe photo #:position.',
]);
```

根據上述範例，驗證將會失敗，且使用者將看到如下錯誤訊息：_"Please describe photo #2."_

如有必要，您可以使用 `second-index`、`second-position`、`third-index`、`third-position` 等來引用更深層的巢狀索引與位置。

```php
'photos.*.attributes.*.string' => 'Invalid attribute for photo #:second-position.',
```


<a name="validating-files"></a>
## 驗證檔案

Laravel 提供了多種可用於驗證上傳檔案的驗證規則，例如 `mimes`、`image`、`min` 與 `max`。雖然您可以在驗證檔案時單獨指定這些規則，但 Laravel 還提供了一個流暢的檔案驗證規則建構器，您可能會覺得很方便：

```php
use Illuminate\Support\Facades\Validator;
use Illuminate\Validation\Rules\File;

Validator::validate($input, [
    'attachment' => [
        'required',
        File::types(['mp3', 'wav'])
            ->min(1024)
            ->max(12 * 1024),
    ],
]);
```


<a name="validating-files-file-types"></a>
#### 驗證檔案類型

儘管在呼叫 `types` 方法時只需要指定擴展名，但此方法實際上會透過讀取檔案內容並推測其 MIME 類型來驗證檔案的 MIME 類型。完整的所有 MIME 類型及其對應擴展名清單可以在以下位置找到：

[https://svn.apache.org/repos/asf/httpd/httpd/trunk/docs/conf/mime.types](https://svn.apache.org/repos/asf/httpd/httpd/trunk/docs/conf/mime.types)


<a name="validating-files-file-sizes"></a>
#### 驗證檔案大小

為了方便起見，最小和最大檔案大小可以指定為帶有表示檔案大小單位的後綴字串。支援 `kb`、`mb`、`gb` 與 `tb` 後綴：

```php
File::types(['mp3', 'wav'])
    ->min('1kb')
    ->max('10mb');
```


<a name="validating-files-image-files"></a>
#### 驗證圖片檔案

如果您的應用程式接受使用者上傳的圖片，您可以使用 `File` 規則的 `image` 建構方法來確保被驗證的檔案是一張圖片 (jpg, jpeg, png, bmp, gif, 或 webp)。

此外，`dimensions` 規則可用於限制圖片的尺寸：

```php
use Illuminate\Support\Facades\Validator;
use Illuminate\Validation\Rule;
use Illuminate\Validation\Rules\File;

Validator::validate($input, [
    'photo' => [
        'required',
        File::image()
            ->min(1024)
            ->max(12 * 1024)
            ->dimensions(Rule::dimensions()->maxWidth(1000)->maxHeight(500)),
    ],
]);
```

> [!NOTE]
> 關於驗證圖片尺寸的更多資訊，請參閱 [dimension 規則文件](#rule-dimensions)。

> [!WARNING]
> 由於可能存在 XSS 漏洞，預設情況下 `image` 規則不允許 SVG 檔案。如果您需要允許 SVG 檔案，可以將 `allowSvg: true` 傳遞給 `image` 規則：`File::image(allowSvg: true)`。


<a name="validating-files-image-dimensions"></a>
#### 驗證圖片尺寸

您也可以驗證圖片的尺寸。例如，若要驗證上傳的圖片寬度至少為 1000 像素且高度至少為 500 像素，可以使用 `dimensions` 規則：

```php
use Illuminate\Validation\Rule;
use Illuminate\Validation\Rules\File;

File::image()->dimensions(
    Rule::dimensions()
        ->maxWidth(1000)
        ->maxHeight(500)
)
```

> [!NOTE]
> 關於驗證圖片尺寸的更多資訊，請參閱 [dimension 規則文件](#rule-dimensions)。

<a name="validating-passwords"></a>
## 驗證密碼

為了確保密碼具有足夠的複雜度，您可以使用 Laravel 的 `Password` 規則物件：

```php
use Illuminate\Support\Facades\Validator;
use Illuminate\Validation\Rules\Password;

$validator = Validator::make($request->all(), [
    'password' => ['required', 'confirmed', Password::min(8)],
]);
```

`Password` 規則物件讓您可以輕鬆地為應用程式自訂密碼複雜度要求，例如指定密碼必須包含至少一個字母、數字、符號或大小寫混合的字元：

```php
// Require at least 8 characters...
Password::min(8)

// Require at least one letter...
Password::min(8)->letters()

// Require at least one uppercase and one lowercase letter...
Password::min(8)->mixedCase()

// Require at least one number...
Password::min(8)->numbers()

// Require at least one symbol...
Password::min(8)->symbols()
```

此外，您可以使用 `uncompromised` 方法來確保密碼沒有在公開的密碼資料外洩事件中被洩漏：

```php
Password::min(8)->uncompromised()
```

在內部，`Password` 規則物件使用 [k-Anonymity](https://en.wikipedia.org/wiki/K-anonymity) 模型來判斷密碼是否透過 [haveibeenpwned.com](https://haveibeenpwned.com) 服務洩漏，且不會犧牲使用者的隱私或安全性。

預設情況下，如果密碼在資料外洩中出現至少一次，它將被視為已洩漏。您可以使用 `uncompromised` 方法的第一個引數來自訂此閾值：

```php
// Ensure the password appears less than 3 times in the same data leak...
Password::min(8)->uncompromised(3);
```

當然，您可以將上述範例中的所有方法串接在一起：

```php
Password::min(8)
    ->letters()
    ->mixedCase()
    ->numbers()
    ->symbols()
    ->uncompromised()
```


<a name="defining-default-password-rules"></a>
#### 定義預設密碼規則

您可能會覺得在應用程式的單一位置指定密碼的預設驗證規則會很方便。您可以使用 `Password::defaults` 方法輕鬆實現這一點，該方法接受一個閉包。傳遞給 `defaults` 方法的閉包應該回傳 Password 規則的預設設定。通常，`defaults` 規則應該在應用程式服務提供者(Service Providers)的 `boot` 方法中被呼叫：

```php
use Illuminate\Validation\Rules\Password;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Password::defaults(function () {
        $rule = Password::min(8);

        return $this->app->isProduction()
            ? $rule->mixedCase()->uncompromised()
            : $rule;
    });
}
```

接著，當您想要將預設規則應用於某個正在進行驗證的密碼時，您可以不帶引數地呼叫 `defaults` 方法：

```php
'password' => ['required', Password::defaults()],
```

有時，您可能希望在預設的密碼驗證規則中附加額外的驗證規則。您可以使用 `rules` 方法來實現這一點：

```php
use App\Rules\ZxcvbnRule;

Password::defaults(function () {
    $rule = Password::min(8)->rules([new ZxcvbnRule]);

    // ...
});
```

<a name="custom-validation-rules"></a>
## 自訂驗證規則


<a name="using-rule-objects"></a>
### 使用規則物件

Laravel 提供了許多實用的驗證規則；然而，您可能希望定義一些自己的規則。註冊自訂驗證規則的一種方法是使用規則物件。要產生一個新的規則物件，您可以使用 `make:rule` Artisan 指令。讓我們使用這個指令來產生一個驗證字串是否為大寫的規則。Laravel 會將新規則放置在 `app/Rules` 目錄中。如果該目錄不存在，Laravel 會在您執行 Artisan 指令建立規則時自動建立它：

```shell
php artisan make:rule Uppercase
```

規則建立完成後，我們就可以定義其行為。規則物件包含一個單一方法：`validate`。此方法會接收屬性名稱、其值，以及一個在驗證失敗時應被調用的回呼函式（callback），並傳入驗證錯誤訊息：

```php
<?php

namespace App\Rules;

use Closure;
use Illuminate\Contracts\Validation\ValidationRule;

class Uppercase implements ValidationRule
{
    /**
     * Run the validation rule.
     */
    public function validate(string $attribute, mixed $value, Closure $fail): void
    {
        if (strtoupper($value) !== $value) {
            $fail('The :attribute must be uppercase.');
        }
    }
}
```

定義好規則後，您可以透過將規則物件的實例與其他驗證規則一起傳遞給驗證器，來將其附加到驗證器上：

```php
use App\Rules\Uppercase;

$request->validate([
    'name' => ['required', 'string', new Uppercase],
]);
```


#### 翻譯驗證訊息

除了向 `$fail` 閉包提供字面上的錯誤訊息外，您也可以提供 [翻譯字串金鑰](/docs/{{version}}/localization) 並指示 Laravel 翻譯錯誤訊息：

```php
if (strtoupper($value) !== $value) {
    $fail('validation.uppercase')->translate();
}
```

如有需要，您可以將佔位符替換內容與偏好的語言，分別作為 `translate` 方法的第一與第二個引數：

```php
$fail('validation.location')->translate([
    'value' => $this->value,
], 'fr');
```


#### 存取額外資料

如果您的自訂驗證規則類別需要存取所有正在進行驗證的其他資料，您的規則類別可以實作 `Illuminate\Contracts\Validation\DataAwareRule` 介面。此介面要求您的類別定義一個 `setData` 方法。此方法將由 Laravel 自動調用（在驗證進行之前），並傳入所有正在驗證的資料：

```php
<?php

namespace App\Rules;

use Illuminate\Contracts\Validation\DataAwareRule;
use Illuminate\Contracts\Validation\ValidationRule;

class Uppercase implements DataAwareRule, ValidationRule
{
    /**
     * All of the data under validation.
     *
     * @var array<string, mixed>
     */
    protected $data = [];

    // ...

    /**
     * Set the data under validation.
     *
     * @param  array<string, mixed>  $data
     */
    public function setData(array $data): static
    {
        $this->data = $data;

        return $this;
    }
}
```

或者，如果您的驗證規則需要存取執行驗證的驗證器實例，您可以實作 `ValidatorAwareRule` 介面：

```php
<?php

namespace App\Rules;

use Illuminate\Contracts\Validation\ValidationRule;
use Illuminate\Contracts\Validation\ValidatorAwareRule;
use Illuminate\Validation\Validator;

class Uppercase implements ValidationRule, ValidatorAwareRule
{
    /**
     * The validator instance.
     *
     * @var \Illuminate\Validation\Validator
     */
    protected $validator;

    // ...

    /**
     * Set the current validator.
     */
    public function setValidator(Validator $validator): static
    {
        $this->validator = $validator;

        return $this;
    }
}
```


<a name="using-closures"></a>
### 使用閉包

如果您在整個應用程式中只需要使用一次自訂規則的功能，您可以使用閉包來替代規則物件。閉包會接收屬性名稱、屬性值，以及一個在驗證失敗時應被調用的 `$fail` 回呼函式：

```php
use Illuminate\Support\Facades\Validator;
use Closure;

$validator = Validator::make($request->all(), [
    'title' => [
        'required',
        'max:255',
        function (string $attribute, mixed $value, Closure $fail) {
            if ($value === 'foo') {
                $fail("The {$attribute} is invalid.");
            }
        },
    ],
]);
```


<a name="implicit-rules"></a>
### 隱含規則

預設情況下，當被驗證的屬性不存在或包含空字串時，一般的驗證規則（包括自訂規則）將不會執行。例如，[unique](#rule-unique) 規則將不會針對空字串執行：

```php
use Illuminate\Support\Facades\Validator;

$rules = ['name' => 'unique:users,name'];

$input = ['name' => ''];

Validator::make($input, $rules)->passes(); // true
```

若要讓自訂規則在屬性為空時仍能執行，該規則必須隱含該屬性為必填。要快速產生一個新的隱含規則物件，您可以使用帶有 `--implicit` 選項的 `make:rule` Artisan 指令：

```shell
php artisan make:rule Uppercase --implicit
```

> [!WARNING]
> 一個「隱含」規則僅僅是 _隱含_ 該屬性為必填。至於它是否真的會使缺失或空的屬性失效，則由您決定。