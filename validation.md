# 驗證

- [簡介](#introduction)
- [驗證快速上手](#validation-quickstart)
    - [定義路由](#quick-defining-the-routes)
    - [建立控制器](#quick-creating-the-controller)
    - [編寫驗證邏輯](#quick-writing-the-validation-logic)
    - [顯示驗證錯誤](#quick-displaying-the-validation-errors)
    - [重新填充表單](#repopulating-forms)
    - [關於選填欄位的說明](#a-note-on-optional-fields)
    - [驗證錯誤回應格式](#validation-error-response-format)
- [表單請求驗證(Form request Validation)](#form-request-validation)
    - [建立表單請求](#creating-form-requests)
    - [授權表單請求](#authorizing-form-requests)
    - [自訂錯誤訊息](#customizing-the-error-messages)
    - [準備驗證輸入](#preparing-input-for-validation)
- [手動建立驗證器](#manually-creating-validators)
    - [自動重新導向](#automatic-redirection)
    - [具名錯誤袋 (Named Error Bags)](#named-error-bags)
    - [自訂錯誤訊息](#manual-customizing-the-error-messages)
    - [執行額外驗證](#performing-additional-validation)
- [處理已驗證的輸入](#working-with-validated-input)
- [處理錯誤訊息](#working-with-error-messages)
    - [在語言檔案中指定自訂訊息](#specifying-custom-messages-in-language-files)
    - [在語言檔案中指定屬性](#specifying-attribute-in-language-files)
    - [在語言檔案中指定值](#specifying-values-in-language-files)
- [可用的驗證規則](#available-validation-rules)
- [條件式地添加規則](#conditionally-adding-rules)
- [驗證陣列](#validating-arrays)
    - [驗證巢狀陣列輸入](#validating-nested-array-input)
    - [錯誤訊息的索引與位置](#error-message-indexes-and-positions)
- [驗證檔案](#validating-files)
- [驗證密碼](#validating-passwords)
- [自訂驗證規則](#custom-validation-rules)
    - [使用規則物件](#using-rule-objects)
    - [使用閉包 (Closures)](#using-closures)
    - [隱含規則 (Implicit Rules)](#implicit-rules)

<a name="introduction"></a>
## 簡介

Laravel 提供了幾種不同的方法來驗證應用程式傳入的資料。最常見的是使用所有傳入 HTTP 請求中都可用的 `validate` 方法。不過，我們也會討論其他的驗證方法。

Laravel 包含了許多方便的驗證規則可用於資料，甚至提供了驗證值在給定資料庫資料表中是否唯一的功能。我們將詳細介紹這些驗證規則，讓您熟悉 Laravel 的所有驗證功能。

<a name="validation-quickstart"></a>
## 驗證快速上手

為了學習 Laravel 強大的驗證功能，讓我們來看一個完整地驗證表單並將錯誤訊息顯示給使用者的範例。透過閱讀這個高階概述，您將能對如何使用 Laravel 驗證傳入的請求資料有良好的整體理解：


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

接下來，讓我們看一個處理這些路由請求的簡單控制器。我們暫時將 `store` 方法留空：

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
### 編寫驗證邏輯

現在我們準備在 `store` 方法中填入驗證新部落格文章的邏輯。為此，我們將使用 `Illuminate\Http\Request` 物件提供的 `validate` 方法。如果驗證規則通過，您的程式碼將繼續正常執行；然而，如果驗證失敗，將會拋出一個 `Illuminate\Validation\ValidationException` 異常，並自動將正確的錯誤回應傳回給使用者。

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

如您所見，驗證規則被傳入 `validate` 方法中。別擔心 —— 所有可用的驗證規則都有 [詳細文件說明](#available-validation-rules)。再次強調，如果驗證失敗，系統會自動產生正確的回應。如果驗證通過，我們的控制器將繼續正常執行。

或者，驗證規則也可以指定為規則陣列，而非單一以 `|` 分隔的字串：

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

有時候您可能希望在某個屬性第一次驗證失敗後就停止執行後續的驗證規則。若要達成此目的，請將 `bail` 規則指派給該屬性：

```php
$request->validate([
    'title' => 'bail|required|unique:posts|max:255',
    'body' => 'required',
]);
```

在這個範例中，如果 `title` 屬性的 `unique` 規則失敗，則不會檢查 `max` 規則。規則將依照指派的順序進行驗證。


<a name="a-note-on-nested-attributes"></a>
#### 關於巢狀屬性的說明

如果傳入的 HTTP 請求包含「巢狀」欄位資料，您可以在驗證規則中使用「點」語法 (dot syntax) 來指定這些欄位：

```php
$request->validate([
    'title' => 'required|unique:posts|max:255',
    'author.name' => 'required',
    'author.description' => 'required',
]);
```

另一方面，如果您的欄位名稱包含實際的句點，您可以透過在句點前加上反斜線來轉義，以明確防止其被解釋為「點」語法：

```php
$request->validate([
    'title' => 'required|unique:posts|max:255',
    'v1\.0' => 'required',
]);
```

<a name="quick-displaying-the-validation-errors"></a>
### 顯示驗證錯誤

那麼，如果傳入的請求欄位沒有通過給定的驗證規則會怎樣？如前所述，Laravel 會自動將使用者重新導向回他們之前的位置。此外，所有的驗證錯誤與 [請求輸入](/docs/{{version}}/requests#retrieving-old-input) 都會自動被 [快閃到會話](/docs/{{version}}/session#flash-data) 中。

`$errors` 變數會由 `web` 中介層群組提供的 `Illuminate\View\Middleware\ShareErrorsFromSession` 中介層分享給您應用程式的所有視圖。當此中介層被套用時，您的視圖中將始終可以使用 `$errors` 變數，讓您可以方便地假設 `$errors` 變數總是已定義且可以安全使用。`$errors` 變數將會是 `Illuminate\Support\MessageBag` 的一個實例。如需更多關於操作此物件的資訊，請[參閱其文件](#working-with-error-messages)。

因此，在我們的範例中，當驗證失敗時，使用者將被重新導向至控制器的 `create` 方法，讓我們能在視圖中顯示錯誤訊息：

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

Laravel 內建的驗證規則 each 都有一個錯誤訊息，位於您應用程式的 `lang/en/validation.php` 檔案中。如果您的應用程式沒有 `lang` 目錄，您可以使用 `lang:publish` Artisan 指令要求 Laravel 建立該目錄。

在 `lang/en/validation.php` 檔案中，您會發現每個驗證規則都有一個翻譯條目。您可以根據應用程式的需求自由地更改或修改這些訊息。

此外，您可以將此檔案複製到另一個語言目錄，以將訊息翻譯成您應用程式所使用的語言。若要了解更多關於 Laravel 本地化的資訊，請參閱完整的 [本地化文件](/docs/{{version}}/localization)。

> [!WARNING]
> 預設情況下，Laravel 應用程式骨架不包含 `lang` 目錄。如果您想要自訂 Laravel 的語言檔案，可以透過 `lang:publish` Artisan 指令來發布它們。


<a name="quick-xhr-requests-and-validation"></a>
#### XHR 請求與驗證

在此範例中，我們使用傳統表單將資料發送到應用程式。然而，許多應用程式會從由 JavaScript 驅動的前端接收 XHR 請求。在 XHR 請求中使用 `validate` 方法時，Laravel 不會產生重新導向回應。相反地，Laravel 會產生一個 [包含所有驗證錯誤的 JSON 回應](#validation-error-response-format)。此 JSON 回應將隨 422 HTTP 狀態碼一同發送。


<a name="the-at-error-directive"></a>
#### `@error` 指令

您可以使用 `@error` [Blade](/docs/{{version}}/blade) 指令來快速判斷特定屬性是否存在驗證錯誤訊息。在 `@error` 指令中，您可以印出 `$message` 變數來顯示錯誤訊息：

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

如果您使用的是 [具名錯誤袋 (named error bags)](#named-error-bags)，您可以將錯誤袋的名稱作為第二個引數傳遞給 `@error` 指令：

```blade
<input ... class="@error('title', 'post') is-invalid @enderror">
```


<a name="repopulating-forms"></a>
### 重新填充表單

當 Laravel 因為驗證錯誤而產生重新導向回應時，框架會自動將 [請求的所有輸入快閃到會話中](/docs/{{version}}/session#flash-data)。這樣做是為了讓您在下一次請求期間能方便地存取輸入內容，並重新填充使用者嘗試提交的表單。

若要從之前的請求中檢索快閃輸入，請在 `Illuminate\Http\Request` 的實例上呼叫 `old` 方法。`old` 方法會從 [會話](/docs/{{version}}/session) 中提取先前快閃的輸入資料：

```php
$title = $request->old('title');
```

Laravel 還提供了一個全域的 `old` 輔助函式。如果您是在 [Blade 模板](/docs/{{version}}/blade) 中顯示舊輸入，使用 `old` 輔助函式來重新填充表單會更方便。如果給定欄位沒有舊輸入，將會返回 `null`：

```blade
<input type="text" name="title" value="{{ old('title') }}">
```


<a name="a-note-on-optional-fields"></a>
### 關於選填欄位的說明

預設情況下，Laravel 在應用程式的全域中介層堆疊中包含了 `TrimStrings` 與 `ConvertEmptyStringsToNull` 中介層。因此，如果您不希望驗證器將 `null` 值視為無效，通常需要將您的「選填」請求欄位標記為 `nullable`。例如：

```php
$request->validate([
    'title' => 'required|unique:posts|max:255',
    'body' => 'required',
    'publish_at' => 'nullable|date',
]);
```

在此範例中，我們指定 `publish_at` 欄位可以是 `null` 或有效的日期表示形式。如果沒有在規則定義中添加 `nullable` 修飾詞，驗證器會將 `null` 視為無效日期。


<a name="validation-error-response-format"></a>
### 驗證錯誤回應格式

當您的應用程式拋出 `Illuminate\Validation\ValidationException` 異常且傳入的 HTTP 請求期望收到 JSON 回應時，Laravel 會自動為您格式化錯誤訊息，並返回 `422 Unprocessable Entity` HTTP 回應。

下方您可以查看驗證錯誤的 JSON 回應格式範例。請注意，巢狀錯誤鍵會被扁平化為「點」符號表示法格式：

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
## 表單請求驗證(Form request Validation)


<a name="creating-form-requests"></a>
### 建立表單請求

對於更複雜的驗證場景，您可能希望建立一個「表單請求」。表單請求是自訂的請求類別，封裝了其本身的驗證與授權邏輯。要建立表單請求類別，您可以使用 `make:request` Artisan CLI 命令：

```shell
php artisan make:request StorePostRequest
```

生成的表單請求類別將被放置在 `app/Http/Requests` 目錄中。如果此目錄不存在，當您執行 `make:request` 命令時會自動建立。Laravel 生成的每個表單請求都有兩個方法：`authorize` 與 `rules`。

正如您所料，`authorize` 方法負責決定目前經過認證的使用者是否可以執行該請求所代表的操作，而 `rules` 方法則回傳應應用於請求資料的驗證規則：

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
> 您可以在 `rules` 方法的簽署中對任何所需的依賴項進行類型提示 (type-hint)。它們將透過 Laravel 的 [服務容器](/docs/{{version}}/container) 自動解析。

那麼，驗證規則是如何被評估的？您只需要在控制器方法中對請求進行類型提示 (type-hint) 即可。進入的表單請求在控制器方法被呼叫之前就會被驗證，這意味著您不需要在控制器中填滿任何驗證邏輯：

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

如果驗證失敗，將會生成一個重新導向回應，將使用者送回之前的位置。錯誤訊息也會被快閃 (flashed) 到 session 中，以便顯示。如果該請求是一個 XHR 請求，則會回傳一個狀態碼為 422 的 HTTP 回應，其中包含 [驗證錯誤的 JSON 格式](#validation-error-response-format)。

> [!NOTE]
> 需要為您由 Inertia 驅動的 Laravel 前端添加即時表單請求驗證嗎？請參考 [Laravel Precognition](/docs/{{version}}/precognition)。


<a name="performing-additional-validation-on-form-requests"></a>
#### 執行額外驗證

有時您需要在初始驗證完成後執行額外驗證。您可以使用表單請求的 `after` 方法來實現。

`after` 方法應回傳一個包含可呼叫物件 (callables) 或閉包的陣列，這些物件將在驗證完成後被呼叫。給定的可呼叫物件將接收到一個 `Illuminate\Validation\Validator` 實例，允許您在必要時提出額外的錯誤訊息：

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

如前所述，`after` 方法回傳的陣列也可以包含可呼叫類別 (invokable classes)。這些類別的 `__invoke` 方法將接收到一個 `Illuminate\Validation\Validator` 實例：

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

透過將 `StopOnFirstFailure` 屬性添加到您的請求類別中，您可以告知驗證器一旦發生單次驗證失敗，就應停止驗證所有屬性：

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


<a name="customizing-the-redirect-location"></a>
#### 自訂重新導向位置

當表單請求驗證失敗時，將會生成一個重新導向回應，將使用者送回之前的位置。不過，您可以自訂此行為。若要如此，您可以在表單請求上使用 `RedirectTo` 屬性：

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

當表單請求驗證失敗時，錯誤會被快閃到 `default` 錯誤袋中。如果您需要將錯誤儲存在不同的 [具名錯誤袋 (Named Error Bags)](#named-error-bags) 中，可以在表單請求上使用 `ErrorBag` 屬性：

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

表單請求類別還包含一個 `authorize` 方法。在此方法中，您可以決定已認證的使用者是否真正具有更新給定資源的權限。例如，您可以決定使用者是否確實擁有他們正嘗試更新的部落格留言。在大多數情況下，您將在此方法中與您的 [授權 Gate 與 Policy](/docs/{{version}}/authorization) 進行互動：

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

由於所有表單請求都繼承自 Laravel 的基礎請求類別，因此我們可以使用 `user` 方法來存取目前已認證的使用者。此外，請注意上方範例中對 `route` 方法的呼叫。此方法讓您可以存取在被呼叫之路由上定義的 URI 參數，例如下方範例中的 `{comment}` 參數：

```php
Route::post('/comment/{comment}');
```

因此，如果您的應用程式利用了 [路由模型綁定](/docs/{{version}}/routing#route-model-binding)，您可以直接將解析後的模型作為請求的屬性來存取，使程式碼更加簡潔：

```php
return $this->user()->can('update', $this->comment);
```

如果 `authorize` 方法回傳 `false`，系統將自動回傳一個 403 狀態碼的 HTTP 回應，且您的控制器方法將不會執行。

如果您計畫在應用程式的其他部分處理請求的授權邏輯，您可以完全移除 `authorize` 方法，或者簡單地回傳 `true`：

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
> 您可以在 `authorize` 方法的簽名中型別提示 (type-hint) 您需要的任何依賴項。它們將透過 Laravel [服務容器](/docs/{{version}}/container) 自動解析。


<a name="customizing-the-error-messages"></a>
### 自訂錯誤訊息

您可以透過覆寫 `messages` 方法來自訂表單請求所使用的錯誤訊息。此方法應回傳一個由屬性/規則配對及其對應錯誤訊息所組成的陣列：

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

Laravel 許多內建的驗證規則錯誤訊息中包含一個 `:attribute` 佔位符。如果您希望驗證訊息中的 `:attribute` 佔位符被替換為自訂的屬性名稱，您可以透過覆寫 `attributes` 方法來指定自訂名稱。此方法應回傳一個屬性/名稱配對的陣列：

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

如果您需要在套用驗證規則之前，準備或清理請求中的任何資料，可以使用 `prepareForValidation` 方法：

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

同樣地，如果您需要在驗證完成後正規化任何請求資料，可以使用 `passedValidation` 方法：

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

如果您不想在請求上使用 `validate` 方法，您可以使用 `Validator` [Facade](/docs/{{version}}/facades) 手動建立驗證器實例。Facade 上的 `make` 方法會產生一個新的驗證器實例：

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

傳遞給 `make` 方法的第一個引數是待驗證的資料。第二個引數是應應用於該資料的驗證規則陣列。

在確定請求驗證是否失敗後，您可以使用 `withErrors` 方法將錯誤訊息快閃 (flash) 到 session 中。使用此方法後，`$errors` 變數在重新導向後會自動與您的視圖共享，讓您能輕鬆地將錯誤訊息顯示給使用者。`withErrors` 方法接受一個驗證器、一個 `MessageBag` 或一個 PHP `array`。


#### 於第一次驗證失敗時停止

`stopOnFirstFailure` 方法將告知驗證器，一旦發生單次驗證失敗，就應停止驗證所有屬性：

```php
if ($validator->stopOnFirstFailure()->fails()) {
    // ...
}
```


<a name="automatic-redirection"></a>
### 自動重新導向

如果您想手動建立驗證器實例，但仍想利用 HTTP 請求之 `validate` 方法所提供的自動重新導向功能，您可以在現有的驗證器實例上呼叫 `validate` 方法。如果驗證失敗，使用者將自動被重新導向，或者在 XHR 請求的情況下，將回傳一個 [JSON 回應](#validation-error-response-format)：

```php
Validator::make($request->all(), [
    'title' => 'required|unique:posts|max:255',
    'body' => 'required',
])->validate();
```

如果驗證失敗，您可以使用 `validateWithBag` 方法將錯誤訊息儲存在 [具名錯誤袋 (named error bag)](#named-error-bags) 中：

```php
Validator::make($request->all(), [
    'title' => 'required|unique:posts|max:255',
    'body' => 'required',
])->validateWithBag('post');
```


<a name="named-error-bags"></a>
### 具名錯誤袋 (Named Error Bags)

如果您在單一頁面上有多個表單，您可能希望為包含驗證錯誤的 `MessageBag` 命名，讓您能夠檢索特定表單的錯誤訊息。要達成此目的，請將名稱作為第二個引數傳遞給 `withErrors`：

```php
return redirect('/register')->withErrors($validator, 'login');
```

您接著可以從 `$errors` 變數中存取該具名 `MessageBag` 實例：

```blade
{{ $errors->login->first('email') }}
```


<a name="manual-customizing-the-error-messages"></a>
### 自訂錯誤訊息

如果需要，您可以提供自訂錯誤訊息，讓驗證器實例使用，而非使用 Laravel 提供的預設錯誤訊息。有幾種方式可以指定自訂訊息。首先，您可以將自訂訊息作為第三個引數傳遞給 `Validator::make` 方法：

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

有時您可能只想為特定屬性指定自訂錯誤訊息。您可以使用「點」語法 (dot notation) 來達成。先指定屬性名稱，接著是規則：

```php
$messages = [
    'email.required' => 'We need to know your email address!',
];
```


<a name="specifying-custom-attribute-values"></a>
#### 指定自訂屬性值

Laravel 的許多內建錯誤訊息包含一個 `:attribute` 佔位符，該佔位符會被替換為待驗證的欄位或屬性名稱。若要自訂特定欄位用來替換這些佔位符的值，您可以將自訂屬性陣列作為第四個引數傳遞給 `Validator::make` 方法：

```php
$validator = Validator::make($input, $rules, $messages, [
    'email' => 'email address',
]);
```


<a name="performing-additional-validation"></a>
### 執行額外驗證

有時您需要在初始驗證完成後執行額外的驗證。您可以使用驗證器的 `after` 方法來達成。`after` 方法接受一個閉包 (closure) 或一個可呼叫對象 (callables) 陣列，這些對象將在驗證完成後被呼叫。這些可呼叫對象將接收一個 `Illuminate\Validation\Validator` 實例，讓您在必要時能增加額外的錯誤訊息：

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

如前所述，`after` 方法也接受一個可呼叫對象陣列，如果您的「驗證後」邏輯被封裝在可呼叫類別 (invokable classes) 中，這會特別方便，這些類別將透過其 `__invoke` 方法接收一個 `Illuminate\Validation\Validator` 實例：

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

在使用表單請求或手動建立的驗證器實例對傳入的請求資料進行驗證後，您可能希望取得實際上經過驗證的請求資料。這可以透過幾種方式達成。首先，您可以在表單請求或驗證器實例上呼叫 `validated` 方法。此方法會回傳一個已驗證資料的陣列：

```php
$validated = $request->validated();

$validated = $validator->validated();
```

或者，您可以在表單請求或驗證器實例上呼叫 `safe` 方法。此方法會回傳一個 `Illuminate\Support\ValidatedInput` 實例。此物件提供了 `only`、`except` 和 `all` 方法，可用於取得已驗證資料的子集或整個已驗證資料陣列：

```php
$validated = $request->safe()->only(['name', 'email']);

$validated = $request->safe()->except(['name', 'email']);

$validated = $request->safe()->all();
```

此外，`Illuminate\Support\ValidatedInput` 實例可以被迭代，且能像陣列一樣被存取：

```php
// Validated data may be iterated...
foreach ($request->safe() as $key => $value) {
    // ...
}

// Validated data may be accessed as an array...
$validated = $request->safe();

$email = $validated['email'];
```

如果您想在已驗證的資料中添加額外欄位，可以呼叫 `merge` 方法：

```php
$validated = $request->safe()->merge(['name' => 'Taylor Otwell']);
```

如果您想將已驗證的資料以 [集合(collection)](/docs/{{version}}/collections) 實例的形式取得，可以呼叫 `collect` 方法：

```php
$collection = $request->safe()->collect();
```


<a name="working-with-error-messages"></a>
## 處理錯誤訊息

在 `Validator` 實例上呼叫 `errors` 方法後，您將收到一個 `Illuminate\Support\MessageBag` 實例，它提供了許多方便處理錯誤訊息的方法。自動提供給所有視圖 (views) 的 `$errors` 變數也是 `MessageBag` 類別的一個實例。


<a name="retrieving-the-first-error-message-for-a-field"></a>
#### 取得特定欄位的第一條錯誤訊息

若要取得特定欄位的第一條錯誤訊息，請使用 `first` 方法：

```php
$errors = $validator->errors();

echo $errors->first('email');
```


<a name="retrieving-all-error-messages-for-a-field"></a>
#### 取得特定欄位的所有錯誤訊息

如果您需要取得特定欄位所有訊息的陣列，請使用 `get` 方法：

```php
foreach ($errors->get('email') as $message) {
    // ...
}
```

如果您正在驗證陣列形式的表單欄位，可以使用 `*` 字元來取得該陣列中每個元素的所有訊息：

```php
foreach ($errors->get('attachments.*') as $message) {
    // ...
}
```


<a name="retrieving-all-error-messages-for-all-fields"></a>
#### 取得所有欄位的所有錯誤訊息

若要取得所有欄位所有訊息的陣列，請使用 `all` 方法：

```php
foreach ($errors->all() as $message) {
    // ...
}
```


<a name="determining-if-messages-exist-for-a-field"></a>
#### 判斷特定欄位是否存在訊息

`has` 方法可用於判斷特定欄位是否存在任何錯誤訊息：

```php
if ($errors->has('email')) {
    // ...
}
```


<a name="specifying-custom-messages-in-language-files"></a>
### 在語言檔案中指定自訂訊息

Laravel 內建的驗證規則 each 都有對應的錯誤訊息，位於您應用程式的 `lang/en/validation.php` 檔案中。如果您的應用程式沒有 `lang` 目錄，可以使用 `lang:publish` Artisan 指令要求 Laravel 建立該目錄。

在 `lang/en/validation.php` 檔案中，您會發現每個驗證規則都有一個翻譯項目。您可以根據應用程式的需求自由更改或修改這些訊息。

此外，您可以將此檔案複製到另一個語言目錄，以針對您應用程式的語言翻譯這些訊息。若要了解更多關於 Laravel 在地化 (localization) 的資訊，請參閱完整的 [在地化文件](/docs/{{version}}/localization)。

> [!WARNING]
> 預設情況下，Laravel 應用程式骨架不包含 `lang` 目錄。如果您想自訂 Laravel 的語言檔案，可以透過 `lang:publish` Artisan 指令來發布它們。


<a name="custom-messages-for-specific-attributes"></a>
#### 針對特定屬性的自訂訊息

您可以在應用程式的驗證語言檔案中，針對指定的屬性與規則組合自訂錯誤訊息。若要達成此目的，請將自訂訊息添加到應用程式 `lang/xx/validation.php` 語言檔案的 `custom` 陣列中：

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

Laravel 許多內建的錯誤訊息包含一個 `:attribute` 佔位符，它會被替換為正在驗證的欄位或屬性名稱。如果您希望驗證訊息中的 `:attribute` 部分被替換為自訂值，可以在 `lang/xx/validation.php` 語言檔案的 `attributes` 陣列中指定自訂屬性名稱：

```php
'attributes' => [
    'email' => 'email address',
],
```

> [!WARNING]
> 預設情況下，Laravel 應用程式骨架不包含 `lang` 目錄。如果您想自訂 Laravel 的語言檔案，可以透過 `lang:publish` Artisan 指令來發布它們。


<a name="specifying-values-in-language-files"></a>
### 在語言檔案中指定值

部分 Laravel 內建的驗證規則錯誤訊息包含一個 `:value` 佔位符，它會被替換為請求屬性的目前值。然而，您偶爾可能需要將驗證訊息中的 `:value` 部分替換為該值的自訂表示方式。例如，考慮以下規則：如果 `payment_type` 的值為 `cc`，則需要提供信用卡號碼：

```php
Validator::make($request->all(), [
    'credit_card_number' => 'required_if:payment_type,cc'
]);
```

如果此驗證規則失敗，將會產生以下錯誤訊息：

```text
The credit card number field is required when payment type is cc.
```

您可以在 `lang/xx/validation.php` 語言檔案中定義 `values` 陣列，以指定更友善的使用者值表示方式，而不是直接顯示 `cc` 作為付款類型值：

```php
'values' => [
    'payment_type' => [
        'cc' => 'credit card'
    ],
],
```

> [!WARNING]
> 預設情況下，Laravel 應用程式骨架不包含 `lang` 目錄。如果您想自訂 Laravel 的語言檔案，可以透過 `lang:publish` Artisan 指令來發布它們。

定義此值後，驗證規則將會產生以下錯誤訊息：

```text
The credit card number field is required when payment type is credit card.
```

<a name="available-validation-rules"></a>
## 可用的驗證規則

以下是所有可用驗證規則及其功能的列表：

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


#### 布林值 (Booleans)

<div class="collection-method-list" markdown="1">

[Accepted](#rule-accepted)
[Accepted If](#rule-accepted-if)
[Boolean](#rule-boolean)
[Declined](#rule-declined)
[Declined If](#rule-declined-if)

</div>


#### 字串 (Strings)

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


#### 數字 (Numbers)

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


#### 陣列 (Arrays)

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


#### 日期 (Dates)

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


#### 檔案 (Files)

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


#### 資料庫 (Database)

<div class="collection-method-list" markdown="1">

[Exists](#rule-exists)
[Unique](#rule-unique)

</div>


#### 工具 (Utilities)

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

根據 PHP 的 `dns_get_record` 函數，被驗證的欄位必須具有有效的 A 或 AAAA 記錄。在將提供的 URL 傳遞給 `dns_get_record` 之前，會使用 PHP 的 `parse_url` 函數提取其主機名稱。


<a name="rule-after"></a>
#### after:_date_

被驗證的欄位必須是一個在給定日期之後的值。日期將被傳遞到 PHP 的 `strtotime` 函數中，以便轉換為有效的 `DateTime` 實例：

```php
'start_date' => 'required|date|after:tomorrow'
```

除了傳遞一個由 `strtotime` 評估的日期字串外，您也可以指定另一個欄位來與該日期進行比較：

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

`afterToday` 和 `todayOrAfter` 方法可用於流暢地表達該日期必須分別在今天之後，或今天或之後：

```php
'start_date' => [
    'required',
    Rule::date()->afterToday(),
],
```


<a name="rule-after-or-equal"></a>
#### after\_or\_equal:_date_

被驗證的欄位必須是一個在給定日期之後或等於該日期的值。更多資訊請參閱 [after](#rule-after) 規則。

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

`Rule::anyOf` 驗證規則允許您指定被驗證的欄位必須滿足給定的任何一組驗證規則。例如，以下規則將驗證 `username` 欄位必須是一個電子郵件地址，或者是一個至少 6 個字元的英數字串（包括連字號）：

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

驗證欄位必須完全由包含在 [\p{L}](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AL%3A%5D&g=&i=), [\p{M}](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AM%3A%5D&g=&i=), [\p{N}](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AN%3A%5D&g=&i=) 中的 Unicode 英數字元，以及 ASCII 破折號 (`-`) 和 ASCII 底線 (`_`) 組成。

若要將此驗證規則限制在 ASCII 範圍 (`a-z`, `A-Z`, 和 `0-9`) 的字元，您可以為驗證規則提供 `ascii` 參數：

```php
'username' => 'alpha_dash:ascii',
```


<a name="rule-alpha-num"></a>
#### alpha_num

驗證欄位必須完全由包含在 [\p{L}](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AL%3A%5D&g=&i=), [\p{M}](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AM%3A%5D&g=&i=), 和 [\p{N}](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AN%3A%5D&g=&i=) 中的 Unicode 英數字元組成。

若要將此驗證規則限制在 ASCII 範圍 (`a-z`, `A-Z`, 和 `0-9`) 的字元，您可以為驗證規則提供 `ascii` 參數：

```php
'username' => 'alpha_num:ascii',
```


<a name="rule-array"></a>
#### array

驗證欄位必須是一個 PHP `array`。

當 `array` 規則被提供額外的值時，輸入陣列中的每個鍵 (key) 必須存在於提供給該規則的值列表中。在以下範例中，輸入陣列中的 `admin` 鍵是無效的，因為它不包含在提供給 `array` 規則的值列表中：

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

在第一次驗證失敗後，停止對該欄位執行驗證規則。

雖然 `bail` 規則僅在遇到驗證失敗時停止驗證特定欄位，但 `stopOnFirstFailure` 方法會通知驗證器，一旦發生單次驗證失敗，就應停止驗證所有屬性：

```php
if ($validator->stopOnFirstFailure()->fails()) {
    // ...
}
```


<a name="rule-before"></a>
#### before:_date_

驗證欄位必須是一個早於指定日期的值。日期將被傳遞給 PHP 的 `strtotime` 函數，以便轉換為有效的 `DateTime` 實例。此外，與 [after](#rule-after) 規則一樣，也可以提供另一個驗證欄位的名稱作為 `date` 的值。

為了方便起見，日期相關的規則可以使用流暢的 `date` 規則建構器來建立：

```php
use Illuminate\Validation\Rule;

'start_date' => [
    'required',
    Rule::date()->before(today()->subDays(7)),
],
```

`beforeToday` 和 `todayOrBefore` 方法可用於流暢地表示日期必須在今天之前，或今天或之前：

```php
'start_date' => [
    'required',
    Rule::date()->beforeToday(),
],
```


<a name="rule-before-or-equal"></a>
#### before\_or\_equal:_date_

驗證欄位必須是一個早於或等於指定日期的值。日期將被傳遞給 PHP 的 `strtotime` 函數，以便轉換為有效的 `DateTime` 實例。此外，與 [after](#rule-after) 規則一樣，也可以提供另一個驗證欄位的名稱作為 `date` 的值。

為了方便起見，日期相關的規則可以使用流暢的 `date` 規則建構器來建立：

```php
use Illuminate\Validation\Rule;

'start_date' => [
    'required',
    Rule::date()->beforeOrEqual(today()->subDays(7)),
],
```


<a name="rule-between"></a>
#### between:_min_,_max_

驗證欄位的大小必須在給定的 _min_ 和 _max_ 之間（含）。字串、數字、陣列和檔案的評估方式與 [size](#rule-size) 規則相同。


<a name="rule-boolean"></a>
#### boolean

驗證欄位必須能夠被轉換為布林值。接受的輸入為 `true`, `false`, `1`, `0`, `"1"`, 和 `"0"`。

您可以使用 `strict` 參數來僅在值為 `true` 或 `false` 時才視為該欄位有效：

```php
'foo' => 'boolean:strict'
```


<a name="rule-confirmed"></a>
#### confirmed

驗證欄位必須有一個匹配的 `{field}_confirmation` 欄位。例如，如果驗證欄位是 `password`，則輸入中必須存在匹配的 `password_confirmation` 欄位。

您也可以傳遞自訂的確認欄位名稱。例如，`confirmed:repeat_username` 將要求 `repeat_username` 欄位與驗證欄位匹配。


<a name="rule-contains"></a>
#### contains:_foo_,_bar_,...

驗證欄位必須是一個包含所有給定參數值的陣列。由於此規則通常需要您對陣列執行 `implode`，因此可以使用 `Rule::contains` 方法來流暢地建構規則：

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

驗證欄位必須是一個不包含任何給定參數值的陣列。由於此規則通常需要您對陣列執行 `implode`，因此可以使用 `Rule::doesntContain` 方法來流暢地建構規則：

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

驗證欄位必須與已認證使用者的密碼匹配。您可以使用規則的第一個參數指定[認證守衛 (authentication guard)](/docs/{{version}}/authentication)：

```php
'password' => 'current_password:api'
```


<a name="rule-date"></a>
#### date

驗證欄位必須是一個根據 PHP `strtotime` 函數定義的有效且非相對的日期。


<a name="rule-date-equals"></a>
#### date_equals:_date_

驗證欄位必須等於給定日期。日期將被傳遞給 PHP 的 `strtotime` 函數，以便轉換為有效的 `DateTime` 實例。


<a name="rule-date-format"></a>
#### date_format:_format_,...

驗證欄位必須匹配給定的 _formats_ 之一。在驗證欄位時，您應該**僅**使用 `date` 或 `date_format` 其中之一，而不是兩者都使用。此驗證規則支援 PHP [DateTime](https://www.php.net/manual/en/class.datetime.php) 類別支援的所有格式。

為了方便起見，日期相關的規則可以使用流暢的 `date` 規則建構器來建立：

```php
use Illuminate\Validation\Rule;

'start_date' => [
    'required',
    Rule::date()->format('Y-m-d'),
],
```


<a name="rule-decimal"></a>
#### decimal:_min_,_max_

驗證欄位必須為數字，且必須包含指定的小數點位數：

```php
// Must have exactly two decimal places (9.99)...
'price' => 'decimal:2'

// Must have between 2 and 4 decimal places...
'price' => 'decimal:2,4'
```


<a name="rule-declined"></a>
#### declined

驗證欄位必須為 `"no"`, `"off"`, `0`, `"0"`, `false`, 或 `"false"`。


<a name="rule-declined-if"></a>
#### declined_if:anotherfield,value,...

如果另一個驗證欄位等於指定值，則驗證欄位必須為 `"no"`, `"off"`, `0`, `"0"`, `false`, 或 `"false"`。


<a name="rule-different"></a>
#### different:_field_

驗證欄位必須具有與 _field_ 不同的值。


<a name="rule-digits"></a>
#### digits:_value_

驗證中的整數必須具有確切的 _value_ 長度。

<a name="rule-digits-between"></a>
#### digits_between:_min_,_max_

驗證中的整數長度必須在給定的 _min_ 與 _max_ 之間。


<a name="rule-dimensions"></a>
#### dimensions

驗證中的檔案必須是一張符合規則參數所指定的尺寸限制的圖片：

```php
'avatar' => 'dimensions:min_width=100,min_height=200'
```

可用的限制包括：_min\_width_, _max\_width_, _min\_height_, _max\_height_, _width_, _height_, _ratio_。

_ratio_ 限制應表示為寬度除以高度。這可以使用像 `3/2` 這樣的分數或像 `1.5` 這樣的浮點數來指定：

```php
'avatar' => 'dimensions:ratio=3/2'
```

由於此規則需要多個引數，因此使用 `Rule::dimensions` 方法以流暢的介面來建構規則通常會更方便：

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

當驗證陣列時，驗證中的欄位不得包含任何重複的值：

```php
'foo.*.id' => 'distinct'
```

distinct 預設使用鬆散的變數比較。若要使用嚴格比較，您可以在驗證規則定義中加入 `strict` 參數：

```php
'foo.*.id' => 'distinct:strict'
```

您可以在驗證規則的引數中加入 `ignore_case` 以使規則忽略大小寫差異：

```php
'foo.*.id' => 'distinct:ignore_case'
```


<a name="rule-doesnt-start-with"></a>
#### doesnt_start_with:_foo_,_bar_,...

驗證中的欄位不得以給定的其中一個值開頭。


<a name="rule-doesnt-end-with"></a>
#### doesnt_end_with:_foo_,_bar_,...

驗證中的欄位不得以給定的其中一個值結尾。


<a name="rule-email"></a>
#### email

驗證中的欄位必須符合電子郵件地址格式。此驗證規則利用 [egulias/email-validator](https://github.com/egulias/EmailValidator) 套件來驗證電子郵件地址。預設情況下會套用 `RFCValidation` 驗證器，但您也可以套用其他驗證樣式：

```php
'email' => 'email:rfc,dns'
```

上面的範例將套用 `RFCValidation` 和 `DNSCheckValidation` 驗證。以下是您可以套用的完整驗證樣式列表：

<div class="content-list" markdown="1">

- `rfc`: `RFCValidation` - 根據 [支援的 RFCs](https://github.com/egulias/EmailValidator?tab=readme-ov-file#supported-rfcs) 驗證電子郵件地址。
- `strict`: `NoRFCWarningsValidation` - 根據 [支援的 RFCs](https://github.com/egulias/EmailValidator?tab=readme-ov-file#supported-rfcs) 驗證電子郵件，若發現警告（例如末尾有句號或多個連續句號）則驗證失敗。
- `dns`: `DNSCheckValidation` - 確保電子郵件地址的網域具有有效的 MX 記錄。
- `spoof`: `SpoofCheckValidation` - 確保電子郵件地址不包含同形字或欺騙性的 Unicode 字元。
- `filter`: `FilterEmailValidation` - 確保電子郵件地址根據 PHP 的 `filter_var` 函數是有效的。
- `filter_unicode`: `FilterEmailValidation::unicode()` - 確保電子郵件地址根據 PHP 的 `filter_var` 函數是有效的，且允許部分 Unicode 字元。

</div>

為了方便起見，電子郵件驗證規則可以使用流暢的規則建構器來建立：

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
> `dns` 和 `spoof` 驗證器需要 PHP 的 `intl` 擴充功能。


<a name="rule-encoding"></a>
#### encoding:*encoding_type*

驗證中的欄位必須符合指定的字元編碼。此規則使用 PHP 的 `mb_check_encoding` 函數來驗證給定檔案或字串值的編碼。為了方便起見，`encoding` 規則可以使用 Laravel 的流暢檔案規則建構器來建構：

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

驗證中的欄位必須以給定的其中一個值結尾。


<a name="rule-enum"></a>
#### enum

`Enum` 規則是一個基於類別的規則，用於驗證驗證中的欄位是否包含有效的列舉 (enum) 值。`Enum` 規則僅接受列舉名稱作為其建構子引數。在驗證原始值 (primitive values) 時，應為 `Enum` 規則提供一個 Backed Enum：

```php
use App\Enums\ServerStatus;
use Illuminate\Validation\Rule;

$request->validate([
    'status' => [Rule::enum(ServerStatus::class)],
]);
```

`Enum` 規則的 `only` 和 `except` 方法可用於限制哪些列舉案例應被視為有效：

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

驗證中的欄位將被排除在 `validate` 和 `validated` 方法回傳的請求資料之外。


<a name="rule-exclude-if"></a>
#### exclude_if:_anotherfield_,_value_

如果 _anotherfield_ 欄位等於 _value_，則驗證中的欄位將被排除在 `validate` 和 `validated` 方法回傳的請求資料之外。

如果需要複雜的條件式排除邏輯，您可以使用 `Rule::excludeIf` 方法。此方法接受一個布林值或一個閉包。當提供閉包時，閉包應回傳 `true` 或 `false` 以指示驗證中的欄位是否應被排除：

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

除非 _anotherfield_ 欄位等於 _value_，否則驗證中的欄位將被排除在 `validate` 和 `validated` 方法回傳的請求資料之外。如果 _value_ 為 `null` (`exclude_unless:name,null`)，除非比較欄位為 `null` 或比較欄位在請求資料中缺失，否則驗證中的欄位將被排除。

如果需要複雜的條件式排除邏輯，您可以使用 `Rule::excludeUnless` 方法。此方法接受一個布林值或一個閉包。當提供閉包時，閉包應回傳 `true` 或 `false` 以指示驗證中的欄位不應被排除：

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

如果 _anotherfield_ 欄位存在，則驗證中的欄位將被排除在 `validate` 和 `validated` 方法回傳的請求資料之外。


<a name="rule-exclude-without"></a>
#### exclude_without:_anotherfield_

如果 _anotherfield_ 欄位不存在，則驗證中的欄位將被排除在 `validate` 和 `validated` 方法回傳的請求資料之外。

<a name="rule-exists"></a>
#### exists:_table_,_column_

驗證的欄位必須存在於指定的資料庫資料表中。


<a name="basic-usage-of-exists-rule"></a>
#### Exists 規則的基本用法

```php
'state' => 'exists:states'
```

如果沒有指定 `column` 選項，將會使用欄位名稱。因此，在此範例中，該規則會驗證 `states` 資料庫資料表是否包含一筆 `state` 欄位值與請求的 `state` 屬性值相匹配的紀錄。


<a name="specifying-a-custom-column-name"></a>
#### 指定自訂欄位名稱

您可以將資料庫欄位名稱放在資料表名稱之後，來明確指定驗證規則應使用的資料庫欄位名稱：

```php
'state' => 'exists:states,abbreviation'
```

有時候，您可能需要指定 `exists` 查詢所使用的特定資料庫連線。您可以將連線名稱前置於資料表名稱之前來達成此目的：

```php
'email' => 'exists:connection.staff,email'
```

您可以指定用來確定資料表名稱的 Eloquent 模型，而不是直接指定資料表名稱：

```php
'user_id' => 'exists:App\Models\User,id'
```

如果您想要自訂驗證規則所執行的查詢，可以使用 `Rule` 類別來流暢地定義規則。在此範例中，我們還將驗證規則指定為陣列，而不是使用 `|` 字元來分隔它們：

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

您可以透過將欄位名稱作為 `exists` 方法的第二個引數，來明確指定由 `Rule::exists` 方法產生的 `exists` 規則應使用的資料庫欄位名稱：

```php
'state' => Rule::exists('states', 'abbreviation'),
```

有時候，您可能希望驗證一組值是否存在於資料庫中。您可以將 `exists` 和 [array](#rule-array) 規則同時添加到被驗證的欄位中來達成此目的：

```php
'states' => ['array', Rule::exists('states', 'abbreviation')],
```

當這兩個規則都被分配給一個欄位時，Laravel 會自動建立單一查詢，以確定所有給定的值是否存在於指定的資料表中。


<a name="rule-extensions"></a>
#### extensions:_foo_,_bar_,...

驗證的檔案必須具有與列出擴充名稱對應的使用者指定擴充名稱：

```php
'photo' => ['required', 'extensions:jpg,png'],
```

> [!WARNING]
> 您絕對不應該僅僅依賴使用者指定的擴充名稱來驗證檔案。此規則通常應與 [mimes](#rule-mimes) 或 [mimetypes](#rule-mimetypes) 規則結合使用。


<a name="rule-file"></a>
#### file

驗證的欄位必須是一個成功上傳的檔案。


<a name="rule-filled"></a>
#### filled

當欄位存在時，驗證的欄位不得為空。


<a name="rule-gt"></a>
#### gt:_field_

驗證的欄位必須大於給定的 _field_ 或 _value_。這兩個欄位必須具有相同的型別。字串、數值、陣列和檔案的評估方式與 [size](#rule-size) 規則相同。


<a name="rule-gte"></a>
#### gte:_field_

驗證的欄位必須大於或等於給定的 _field_ 或 _value_。這兩個欄位必須具有相同的型別。字串、數值、陣列和檔案的評估方式與 [size](#rule-size) 規則相同。


<a name="rule-hex-color"></a>
#### hex_color

驗證的欄位必須包含一個[十六進位](https://developer.mozilla.org/en-US/docs/Web/CSS/hex-color)格式的有效顏色值。


<a name="rule-image"></a>
#### image

驗證的檔案必須是一張圖片 (jpg, jpeg, png, bmp, gif 或 webp)。

> [!WARNING]
> 由於 XSS 漏洞的可能性，預設情況下 image 規則不允許 SVG 檔案。如果您需要允許 SVG 檔案，可以將 `allow_svg` 指令傳遞給 `image` 規則 (`image:allow_svg`)。


<a name="rule-in"></a>
#### in:_foo_,_bar_,...

驗證的欄位必須包含在給定的值列表中。由於此規則通常需要您 `implode` 一個陣列，因此可以使用 `Rule::in` 方法來流暢地建構規則：

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

當 `in` 規則與 `array` 規則結合時，輸入陣列中的每個值都必須存在於提供給 `in` 規則的值列表中。在以下範例中，輸入陣列中的 `LAS` 機場代碼是無效的，因為它不包含在提供給 `in` 規則的機場列表中：

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

驗證的欄位必須存在於 _anotherfield_ 的值中。


<a name="rule-in-array-keys"></a>
#### in_array_keys:_value_.*

驗證的欄位必須是一個陣列，且該陣列中至少有一個給定的 _values_ 作為鍵 (key)：

```php
'config' => 'array|in_array_keys:timezone'
```


<a name="rule-integer"></a>
#### integer

驗證的欄位必須是一個整數。

您可以使用 `strict` 參數，僅在欄位型別為 `integer` 時才視為有效。具有整數值的字串將被視為無效：

```php
'age' => 'integer:strict'
```

> [!WARNING]
> 此驗證規則不會驗證輸入是否為「integer」變數型別，僅驗證輸入是否為 PHP 的 `FILTER_VALIDATE_INT` 規則所接受的型別。如果您需要驗證輸入是否為數字，請將此規則與 [the `numeric` validation rule](#rule-numeric) 結合使用。


<a name="rule-ip"></a>
#### ip

驗證的欄位必須是一個 IP 位址。


<a name="ipv4"></a>
#### ipv4

驗證的欄位必須是一個 IPv4 位址。


<a name="ipv6"></a>
#### ipv6

驗證的欄位必須是一個 IPv6 位址。


<a name="rule-json"></a>
#### json

驗證的欄位必須是一個有效的 JSON 字串。


<a name="rule-lt"></a>
#### lt:_field_

驗證的欄位必須小於給定的 _field_。這兩個欄位必須具有相同的型別。字串、數值、陣列和檔案的評估方式與 [size](#rule-size) 規則相同。


<a name="rule-lte"></a>
#### lte:_field_

驗證的欄位必須小於或等於給定的 _field_。這兩個欄位必須具有相同的型別。字串、數值、陣列和檔案的評估方式與 [size](#rule-size) 規則相同。


<a name="rule-lowercase"></a>
#### lowercase

驗證的欄位必須為小寫。


<a name="rule-list"></a>
#### list

驗證的欄位必須是一個作為列表 (list) 的陣列。如果陣列的鍵是由 0 到 `count($array) - 1` 的連續數字組成，則該陣列被視為列表。


<a name="rule-mac"></a>
#### mac_address

驗證的欄位必須是一個 MAC 位址。


<a name="rule-max"></a>
#### max:_value_

驗證的欄位必須小於或等於最大值 _value_。字串、數值、陣列和檔案的評估方式與 [size](#rule-size) 規則相同。


<a name="rule-max-digits"></a>
#### max_digits:_value_

驗證的整數必須具有最大長度 _value_。

<a name="rule-mimetypes"></a>
#### mimetypes:_text/plain_,...

待驗證的檔案必須符合給定的其中一個 MIME 類型：

```php
'video' => 'mimetypes:video/avi,video/mpeg,video/quicktime',

'media' => 'mimetypes:image/*,video/*',
```

為了確定上傳檔案的 MIME 類型，框架會讀取檔案內容並嘗試推測其 MIME 類型，這可能與用戶端提供的 MIME 類型不同。


<a name="rule-mimes"></a>
#### mimes:_foo_,_bar_,...

待驗證的檔案必須具有與列出的其中一個擴展名對應的 MIME 類型：

```php
'photo' => 'mimes:jpg,bmp,png'
```

儘管您只需要指定擴展名，但此規則實際上會透過讀取檔案內容並推測其 MIME 類型來驗證檔案的 MIME 類型。您可以在以下位置找到 MIME 類型及其對應擴展名的完整列表：

[https://svn.apache.org/repos/asf/httpd/httpd/trunk/docs/conf/mime.types](https://svn.apache.org/repos/asf/httpd/httpd/trunk/docs/conf/mime.types)


<a name="mime-types-and-extensions"></a>
#### MIME 類型與擴展名

此驗證規則不會驗證 MIME 類型與使用者為檔案指定的擴展名是否一致。例如，`mimes:png` 驗證規則會將包含有效 PNG 內容的檔案視為有效的 PNG 圖片，即使該檔案被命名為 `photo.txt`。如果您想驗證使用者指定的檔案擴展名，可以使用 [extensions](#rule-extensions) 規則。


<a name="rule-min"></a>
#### min:_value_

待驗證的欄位必須具有最小值 _value_。字串、數值、陣列和檔案的評估方式與 [size](#rule-size) 規則相同。


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

僅當其他指定的任何欄位出現時，待驗證的欄位不得出現。


<a name="rule-missing-with-all"></a>
#### missing_with_all:_foo_,_bar_,...

僅當所有其他指定的欄位都出現時，待驗證的欄位不得出現。


<a name="rule-not-in"></a>
#### not_in:_foo_,_bar_,...

待驗證的欄位不得包含在給定的值列表中。您可以使用 `Rule::notIn` 方法來流暢地構建此規則：

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
> 使用 `regex` / `not_regex` 模式時，可能需要使用陣列而非 `|` 定界符來指定驗證規則，特別是當正規表達式包含 `|` 字元時。


<a name="rule-nullable"></a>
#### nullable

待驗證的欄位可以為 `null`。


<a name="rule-numeric"></a>
#### numeric

待驗證的欄位必須是[數值](https://www.php.net/manual/en/function.is-numeric.php)。

您可以使用 `strict` 參數來僅在該值為整數或浮點數型別時才視為有效。數值字串將被視為無效：

```php
'amount' => 'numeric:strict'
```


<a name="rule-present"></a>
#### present

待驗證的欄位必須存在於輸入資料中。


<a name="rule-present-if"></a>
#### present_if:_anotherfield_,_value_,...

如果 _anotherfield_ 欄位等於任何 _value_，則待驗證的欄位必須出現。


<a name="rule-present-unless"></a>
#### present_unless:_anotherfield_,_value_

除非 _anotherfield_ 欄位等於任何 _value_，否則待驗證的欄位必須出現。


<a name="rule-present-with"></a>
#### present_with:_foo_,_bar_,...

僅當其他指定的任何欄位出現時，待驗證的欄位必須出現。


<a name="rule-present-with-all"></a>
#### present_with_all:_foo_,_bar_,...

僅當所有其他指定的欄位都出現時，待驗證的欄位必須出現。


<a name="rule-prohibited"></a>
#### prohibited

待驗證的欄位必須缺失或為空。如果欄位符合以下任一條件，則視為「空」：

<div class="content-list" markdown="1">

- 該值為 `null`。
- 該值為空字串。
- 該值為空陣列或空 `Countable` 物件。
- 該值為路徑為空的上傳檔案。

</div>


<a name="rule-prohibited-if"></a>
#### prohibited_if:_anotherfield_,_value_,...

如果 _anotherfield_ 欄位等於任何 _value_，則待驗證的欄位必須缺失或為空。如果欄位符合以下任一條件，則視為「空」：

<div class="content-list" markdown="1">

- 該值為 `null`。
- 該值為空字串。
- 該值為空陣列或空 `Countable` 物件。
- 該值為路徑為空的上傳檔案。

</div>

如果需要複雜的條件禁止邏輯，您可以使用 `Rule::prohibitedIf` 方法。此方法接受一個布林值或一個閉包。當提供閉包時，閉包應返回 `true` 或 `false` 以指示待驗證的欄位是否應被禁止：

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

如果 _anotherfield_ 欄位等於 `"yes"`, `"on"`, `1`, `"1"`, `true`, 或 `"true"`，則待驗證的欄位必須缺失或為空。


<a name="rule-prohibited-if-declined"></a>
#### prohibited_if_declined:_anotherfield_,...

如果 _anotherfield_ 欄位等於 `"no"`, `"off"`, `0`, `"0"`, `false`, 或 `"false"`，則待驗證的欄位必須缺失或為空。


<a name="rule-prohibited-unless"></a>
#### prohibited_unless:_anotherfield_,_value_,...

除非 _anotherfield_ 欄位等於任何 _value_，否則待驗證的欄位必須缺失或為空。如果欄位符合以下任一條件，則視為「空」：

<div class="content-list" markdown="1">

- 該值為 `null`。
- 該值為空字串。
- 該值為空陣列或空 `Countable` 物件。
- 該值為路徑為空的上傳檔案。

</div>

如果需要複雜的條件禁止邏輯，您可以使用 `Rule::prohibitedUnless` 方法。此方法接受一個布林值或一個閉包。當提供閉包時，閉包應返回 `true` 或 `false` 以指示待驗證的欄位是否不應被禁止：

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

若正在驗證的欄位並非缺失或為空，則 _anotherfield_ 中的所有欄位必須缺失或為空。欄位在滿足以下任一條件時被視為「空」：

<div class="content-list" markdown="1">

- 該值為 `null`。
- 該值為空字串。
- 該值為空陣列或空的 `Countable` 物件。
- 該值為路徑為空的上傳檔案。

</div>


<a name="rule-regex"></a>
#### regex:_pattern_

正在驗證的欄位必須符合給定的正規表示式。

在內部，此規則使用 PHP 的 `preg_match` 函數。指定的模式應遵守 `preg_match` 所要求的格式，因此也必須包含有效的定界符 (delimiters)。例如：`'email' => 'regex:/^.+@.+$/i'`。

> [!WARNING]
> 當使用 `regex` / `not_regex` 模式時，可能需要使用陣列來指定規則而非使用 `|` 定界符，特別是當正規表示式中包含 `|` 字元時。


<a name="rule-required"></a>
#### required

正在驗證的欄位必須存在於輸入資料中且不能為空。欄位在滿足以下任一條件時被視為「空」：

<div class="content-list" markdown="1">

- 該值為 `null`。
- 該值為空字串。
- 該值為空陣列或空的 `Countable` 物件。
- 該值為沒有路徑的上傳檔案。

</div>


<a name="rule-required-if"></a>
#### required_if:_anotherfield_,_value_,...

若 _anotherfield_ 欄位等於任何 _value_，則正在驗證的欄位必須存在且不能為空。

如果您想為 `required_if` 規則建構更複雜的條件，可以使用 `Rule::requiredIf` 方法。此方法接受一個布林值或閉包。當傳入閉包時，該閉包應回傳 `true` 或 `false` 以表示正在驗證的欄位是否為必填：

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

若 _anotherfield_ 欄位等於 `"yes"`, `"on"`, `1`, `"1"`, `true`, 或 `"true"`，則正在驗證的欄位必須存在且不能為空。


<a name="rule-required-if-declined"></a>
#### required_if_declined:_anotherfield_,...

若 _anotherfield_ 欄位等於 `"no"`, `"off"`, `0`, `"0"`, `false`, 或 `"false"`，則正在驗證的欄位必須存在且不能為空。


<a name="rule-required-unless"></a>
#### required_unless:_anotherfield_,_value_,...

除非 _anotherfield_ 欄位等於任何 _value_，否則正在驗證的欄位必須存在且不能為空。這也意味著 _anotherfield_ 必須存在於請求資料中，除非 _value_ 為 `null`。若 _value_ 為 `null` (`required_unless:name,null`)，則除非比較欄位為 `null` 或比較欄位在請求資料中缺失，否則正在驗證的欄位將被要求必填。

如果您想為 `required_unless` 規則建構更複雜的條件，可以使用 `Rule::requiredUnless` 方法。此方法接受一個布林值或閉包。當傳入閉包時，該閉包應回傳 `true` 或 `false` 以表示正在驗證的欄位是否不需要必填：

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

_僅當_ 其他任何指定的欄位存在且不為空時，正在驗證的欄位必須存在且不能為空。


<a name="rule-required-with-all"></a>
#### required_with_all:_foo_,_bar_,...

_僅當_ 所有其他指定的欄位都存在且不為空時，正在驗證的欄位必須存在且不能為空。


<a name="rule-required-without"></a>
#### required_without:_foo_,_bar_,...

_僅當_ 任何其他指定的欄位為空或不存在時，正在驗證的欄位必須存在且不能為空。


<a name="rule-required-without-all"></a>
#### required_without_all:_foo_,_bar_,...

_僅當_ 所有其他指定的欄位都為空或不存在時，正在驗證的欄位必須存在且不能為空。


<a name="rule-required-array-keys"></a>
#### required_array_keys:_foo_,_bar_,...

正在驗證的欄位必須是一個陣列，且必須包含至少指定的鍵。


<a name="rule-same"></a>
#### same:_field_

給定的 _field_ 必須與正在驗證的欄位相匹配。


<a name="rule-size"></a>
#### size:_value_

正在驗證的欄位必須具有與給定 _value_ 相匹配的大小。對於字串資料，_value_ 對應於字元數。對於數值資料，_value_ 對應於給定的整數值（該屬性還必須具有 `numeric` 或 `integer` 規則）。對於陣列，_size_ 對應於陣列的 `count`（數量）。對於檔案，_size_ 對應於以 KB 為單位的檔案大小。讓我們看一些範例：

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

正在驗證的欄位必須以給定值之一開始。


<a name="rule-string"></a>
#### string

正在驗證的欄位必須是一個字串。如果您希望允許該欄位也可以為 `null`，則應將 `nullable` 規則分配給該欄位。

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

字串規則建構器提供了常用字串限制的方法，包括 `alpha`, `alphaDash`, `alphaNumeric`, `ascii`, `between`, `doesntEndWith`, `doesntStartWith`, `endsWith`, `exactly`, `lowercase`, `max`, `min`, `startsWith`, 以及 `uppercase`。由於規則建構器是可條件化的，您也可以使用 `when` 和 `unless` 方法來條件式地應用限制。


<a name="rule-timezone"></a>
#### timezone

正在驗證的欄位必須是一個有效的時區識別碼，符合 `DateTimeZone::listIdentifiers` 方法的定義。

此驗證規則也可以提供 [`DateTimeZone::listIdentifiers` 方法所接受](https://www.php.net/manual/en/datetimezone.listidentifiers.php) 的參數：

```php
'timezone' => 'required|timezone:all';

'timezone' => 'required|timezone:Africa';

'timezone' => 'required|timezone:per_country,US';
```

<a name="rule-unique"></a>
#### unique:_table_,_column_

被驗證的欄位必須不存在於給定的資料庫資料表中。

**指定自訂資料表 / 欄位名稱：**

您可以指定 Eloquent 模型來決定資料表名稱，而不是直接指定資料表名稱：

```php
'email' => 'unique:App\Models\User,email_address'
```

`column` 選項可用於指定該欄位對應的資料庫欄位。如果未指定 `column` 選項，則將使用被驗證欄位的名稱。

```php
'email' => 'unique:users,email_address'
```

**指定自訂資料庫連線**

有時您可能需要為驗證器 (Validator) 執行的資料庫查詢設定自訂連線。若要達成此目的，您可以在資料表名稱前加上連線名稱：

```php
'email' => 'unique:connection.users,email_address'
```

**強制 unique 規則忽略給定的 ID：**

有時您可能希望在 unique 驗證期間忽略給定的 ID。例如，假設有一個包含使用者名稱、電子郵件地址和位置的「更新個人資料」畫面。您可能想要驗證電子郵件地址是唯一的。然而，如果使用者僅更改名稱欄位而未更改電子郵件欄位，您不希望因為使用者已經是該電子郵件地址的所有者而拋出驗證錯誤。

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
> 您絕對不應該將任何由使用者控制的請求輸入傳遞給 `ignore` 方法。相反地，您應該僅傳遞系統產生的唯一 ID，例如來自 Eloquent 模型實體的自動遞增 ID 或 UUID。否則，您的應用程式將容易受到 SQL 注入攻擊。

您可以傳遞整個模型實體給 `ignore` 方法，而不是傳遞模型鍵的值。Laravel 會自動從模型中提取鍵：

```php
Rule::unique('users')->ignore($user)
```

如果您的資料表使用的主鍵欄位名稱不是 `id`，您可以在呼叫 `ignore` 方法時指定欄位名稱：

```php
Rule::unique('users')->ignore($user->id, 'user_id')
```

預設情況下，`unique` 規則會檢查與被驗證屬性名稱相匹配的欄位唯一性。不過，您可以將不同的欄位名稱作為 `unique` 方法的第二個引數傳遞：

```php
Rule::unique('users', 'email_address')->ignore($user->id)
```

**添加額外的 Where 子句：**

您可以使用 `where` 方法自訂查詢，以指定額外的查詢條件。例如，讓我們添加一個查詢條件，將查詢範圍限制在 `account_id` 欄位值為 `1` 的記錄中：

```php
'email' => Rule::unique('users')->where(fn (Builder $query) => $query->where('account_id', 1))
```

**在唯一性檢查中忽略軟刪除記錄：**

預設情況下，unique 規則在判定唯一性時會包含軟刪除的記錄。若要從唯一性檢查中排除軟刪除的記錄，您可以呼叫 `withoutTrashed` 方法：

```php
Rule::unique('users')->withoutTrashed();
```

如果您的模型對軟刪除記錄使用了 `deleted_at` 以外的欄位名稱，您可以在呼叫 `withoutTrashed` 方法時提供該欄位名稱：

```php
Rule::unique('users')->withoutTrashed('was_deleted_at');
```


<a name="rule-uppercase"></a>
#### uppercase

被驗證的欄位必須是大寫。


<a name="rule-url"></a>
#### url

被驗證的欄位必須是一個有效的 URL。

如果您想要指定哪些 URL 協定被視為有效，您可以將協定作為驗證規則的參數傳遞：

```php
'url' => 'url:http,https',

'game' => 'url:minecraft,steam',
```


<a name="rule-ulid"></a>
#### ulid

被驗證的欄位必須是一個有效的 [Universally Unique Lexicographically Sortable Identifier](https://github.com/ulid/spec) (ULID)。


<a name="rule-uuid"></a>
#### uuid

被驗證的欄位必須是一個有效的 RFC 9562 (版本 1, 3, 4, 5, 6, 7 或 8) 通用唯一識別碼 (UUID)。

您也可以驗證給定的 UUID 是否符合特定版本的 UUID 規範：

```php
'uuid' => 'uuid:4'
```

<a name="conditionally-adding-rules"></a>
## 條件式地添加規則


<a name="skipping-validation-when-fields-have-certain-values"></a>
#### 當欄位具有特定值時跳過驗證

您有時可能希望在另一個欄位具有特定值時，不對給定欄位進行驗證。您可以使用 `exclude_if` 驗證規則來達成此目的。在這個範例中，如果 `has_appointment` 欄位的值為 `false`，則 `appointment_date` 和 `doctor_name` 欄位將不會被驗證：

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
#### 僅在欄位存在時驗證

在某些情況下，您可能希望**僅**在該欄位存在於被驗證的資料中時，才對該欄位執行驗證檢查。要快速達成此目的，請將 `sometimes` 規則添加到您的規則列表中：

```php
$validator = Validator::make($data, [
    'email' => 'sometimes|required|email',
]);
```

在上述範例中，只有當 `email` 欄位存在於 `$data` 陣列中時，才會對其進行驗證。

> [!NOTE]
> 如果您嘗試驗證一個應該始終存在但可能為空的欄位，請參閱 [關於選填欄位的說明](#a-note-on-optional-fields)。


<a name="complex-conditional-validation"></a>
#### 複雜的條件式驗證

有時您可能希望根據更複雜的條件邏輯來添加驗證規則。例如，您可能希望僅在另一個欄位的值大於 100 時，才要求給定欄位必須填寫。或者，您可能需要僅在另一個欄位存在時，兩個欄位才必須具有特定值。添加這些驗證規則並不困難。首先，使用永不改變的 _靜態規則_ 建立一個 `Validator` 實例：

```php
use Illuminate\Support\Facades\Validator;

$validator = Validator::make($request->all(), [
    'email' => 'required|email',
    'games' => 'required|integer|min:0',
]);
```

假設我們的網頁應用程式是為遊戲收藏家設計的。如果一位遊戲收藏家在我們的應用程式中註冊，且他們擁有超過 100 款遊戲，我們希望他們解釋為什麼擁有這麼多遊戲。例如，也許他們經營一家遊戲轉售店，或者他們只是喜歡收集遊戲。要條件式地添加此要求，我們可以使用 `Validator` 實例上的 `sometimes` 方法。

```php
use Illuminate\Support\Fluent;

$validator->sometimes('reason', 'required|max:500', function (Fluent $input) {
    return $input->games >= 100;
});
```

傳遞給 `sometimes` 方法的第一個引數是要條件式驗證的欄位名稱。第二個引數是要添加的規則列表。如果作為第三個引數傳遞的閉包返回 `true`，則將添加這些規則。這個方法讓建立複雜的條件式驗證變得非常簡單。您甚至可以一次為多個欄位添加條件式驗證：

```php
$validator->sometimes(['reason', 'cost'], 'required', function (Fluent $input) {
    return $input->games >= 100;
});
```

> [!NOTE]
> 傳遞給閉包的 `$input` 參數將是一個 `Illuminate\Support\Fluent` 實例，可用於存取正在驗證的輸入和檔案。


<a name="complex-conditional-array-validation"></a>
#### 複雜的條件式陣列驗證

有時您可能希望根據同一個巢狀陣列中另一個索引不明的欄位來驗證某個欄位。在這些情況下，您可以讓閉包接收第二個引數，該引數將是目前正在驗證的陣列中的單個項目：

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

正如 [陣列驗證規則文件](#rule-array) 中所討論的，`array` 規則接受一個允許的陣列鍵名列表。如果陣列中存在任何額外的鍵名，驗證將會失敗：

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

一般而言，您應該始終指定允許存在於陣列中的鍵名。否則，驗證器的 `validate` 與 `validated` 方法將會回傳所有已驗證的資料，包括該陣列及其所有鍵名，即使這些鍵名沒有被其他巢狀陣列驗證規則所驗證。


<a name="validating-nested-array-input"></a>
### 驗證巢狀陣列輸入

驗證基於巢狀陣列的表單輸入欄位並不困難。您可以使用「點號表示法 (dot notation)」來驗證陣列中的屬性。例如，如果傳入的 HTTP 請求包含一個 `photos[profile]` 欄位，您可以像這樣驗證它：

```php
use Illuminate\Support\Facades\Validator;

$validator = Validator::make($request->all(), [
    'photos.profile' => 'required|image',
]);
```

您也可以驗證陣列中的每個元素。例如，要驗證給定陣列輸入欄位中的每個電子郵件是否唯一，您可以執行以下操作：

```php
$validator = Validator::make($request->all(), [
    'users.*.email' => 'email|unique:users',
    'users.*.first_name' => 'required_with:users.*.last_name',
]);
```

同樣地，在 [語言檔案中指定自訂驗證訊息](#custom-messages-for-specific-attributes) 時，您可以使用 `*` 字元，這讓為陣列欄位使用單一驗證訊息變得非常簡單：

```php
'custom' => [
    'users.*.email' => [
        'unique' => 'Each user must have a unique email address',
    ]
],
```


<a name="accessing-nested-array-data"></a>
#### 存取巢狀陣列資料

有時候，在為屬性指定驗證規則時，您可能需要存取給定巢狀陣列元素的數值。您可以使用 `Rule::forEach` 方法來實現這一點。`forEach` 方法接受一個閉包，該閉包會在驗證陣列屬性的每次迭代時被調用，並接收屬性的數值以及明確的、完全展開的屬性名稱。該閉包應回傳一個要分配給陣列元素的規則陣列：

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

在驗證陣列時，您可能希望在應用程式顯示的錯誤訊息中引用驗證失敗之特定項目的索引或位置。為了實現這一點，您可以在 [自訂驗證訊息](#manual-customizing-the-error-messages) 中包含 `:index`（從 `0` 開始）、`:position`（從 `1` 開始）或 `:ordinal-position`（從 `1st` 開始）佔位符：

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

根據上述範例，驗證將會失敗，使用者將會看到 _"Please describe photo #2."_ 的錯誤訊息。

如果需要，您可以使用 `second-index`、`second-position`、`third-index`、`third-position` 等來引用更深層的巢狀索引與位置。

```php
'photos.*.attributes.*.string' => 'Invalid attribute for photo #:second-position.',
```


<a name="validating-files"></a>
## 驗證檔案

Laravel 提供了各種可用於驗證上傳檔案的驗證規則，例如 `mimes`、`image`、`min` 和 `max`。雖然您可以在驗證檔案時單獨指定這些規則，但 Laravel 還提供了一個流暢的檔案驗證規則建構器，您可能會覺得很方便：

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

雖然在調用 `types` 方法時只需要指定副檔名，但此方法實際上是透過讀取檔案內容並推測其 MIME 類型來驗證檔案的 MIME 類型。完整的 MIME 類型及其對應副檔名列表可以在以下位置找到：

[https://svn.apache.org/repos/asf/httpd/httpd/trunk/docs/conf/mime.types](https://svn.apache.org/repos/asf/httpd/httpd/trunk/docs/conf/mime.types)


<a name="validating-files-file-sizes"></a>
#### 驗證檔案大小

為了方便起見，最小和最大檔案大小可以用帶有表示檔案大小單位的後綴字串來指定。支援 `kb`、`mb`、`gb` 和 `tb` 後綴：

```php
File::types(['mp3', 'wav'])
    ->min('1kb')
    ->max('10mb');
```


<a name="validating-files-image-files"></a>
#### 驗證圖片檔案

如果您的應用程式接受使用者上傳的圖片，您可以使用 `File` 規則的 `image` 建構方法來確保驗證中的檔案是一張圖片 (jpg, jpeg, png, bmp, gif, 或 webp)。

此外，可以使用 `dimensions` 規則來限制圖片的尺寸：

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
> 關於驗證圖片尺寸的更多資訊，請參閱 [尺寸規則文件](#rule-dimensions)。

> [!WARNING]
> 由於 XSS 漏洞的可能性，預設情況下 `image` 規則不允許 SVG 檔案。如果您需要允許 SVG 檔案，可以將 `allowSvg: true` 傳遞給 `image` 規則：`File::image(allowSvg: true)`。


<a name="validating-files-image-dimensions"></a>
#### 驗證圖片尺寸

您也可以驗證圖片的尺寸。例如，要驗證上傳的圖片寬度至少為 1000 像素且高度至少為 500 像素，您可以使用 `dimensions` 規則：

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
> 關於驗證圖片尺寸的更多資訊，請參閱 [尺寸規則文件](#rule-dimensions)。

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

在內部，`Password` 規則物件使用了 [k-Anonymity](https://en.wikipedia.org/wiki/K-anonymity) 模型，透過 [haveibeenpwned.com](https://haveibeenpwned.com) 服務來判斷密碼是否已被洩漏，且不會犧牲使用者的隱私或安全性。

預設情況下，如果密碼在資料外洩中出現過至少一次，它將被視為已洩漏。您可以使用 `uncompromised` 方法的第一個參數來自訂此門檻：

```php
// Ensure the password appears less than 3 times in the same data leak...
Password::min(8)->uncompromised(3);
```

當然，您可以將上述範例中的所有方法鏈接起來：

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

您可能會覺得在應用程式的單一位置指定密碼的預設驗證規則會很方便。您可以使用 `Password::defaults` 方法輕鬆實現，該方法接受一個閉包。傳遞給 `defaults` 方法的閉包應該回傳 Password 規則的預設配置。通常，`defaults` 規則應該在應用程式服務提供者(Service Providers)的 `boot` 方法中被呼叫：

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

接著，當您想要將預設規則應用於某個正在進行驗證的密碼時，您可以不傳遞任何參數地呼叫 `defaults` 方法：

```php
'password' => ['required', Password::defaults()],
```

有時候，您可能想要在預設的密碼驗證規則中附加額外的驗證規則。您可以使用 `rules` 方法來達成此目的：

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

Laravel 提供了許多實用的驗證規則；然而，您可能希望指定一些自己的規則。註冊自訂驗證規則的一種方法是使用規則物件。要產生一個新的規則物件，您可以使用 `make:rule` Artisan 命令。讓我們使用此命令來產生一個驗證字串是否為大寫的規則。Laravel 會將新規則放置在 `app/Rules` 目錄中。如果此目錄不存在，Laravel 會在您執行 Artisan 命令建立規則時自動建立它：

```shell
php artisan make:rule Uppercase
```

一旦規則建立完成，我們就可以定義它的行為。一個規則物件包含單一的方法：`validate`。此方法會接收屬性名稱、其值，以及一個在驗證失敗時應被調用的回呼函數，用以傳遞驗證錯誤訊息：

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

一旦定義好規則，您就可以透過將規則物件的實例與其他驗證規則一起傳遞，將其附加到驗證器中：

```php
use App\Rules\Uppercase;

$request->validate([
    'name' => ['required', 'string', new Uppercase],
]);
```


#### 翻譯驗證訊息

除了向 `$fail` 閉包提供字面上的錯誤訊息外，您也可以提供一個 [翻譯字串金鑰](/docs/{{version}}/localization) 並指示 Laravel 翻譯該錯誤訊息：

```php
if (strtoupper($value) !== $value) {
    $fail('validation.uppercase')->translate();
}
```

如有必要，您可以將佔位符替換內容和偏好語言分別作為 `translate` 方法的第一和第二個引數：

```php
$fail('validation.location')->translate([
    'value' => $this->value,
], 'fr');
```


#### 存取額外資料

如果您的自訂驗證規則類別需要存取所有正在進行驗證的其他資料，您的規則類別可以實作 `Illuminate\Contracts\Validation\DataAwareRule` 介面。此介面要求您的類別定義一個 `setData` 方法。此方法將由 Laravel 自動調用（在驗證繼續進行之前），並傳入所有正在驗證的資料：

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
### 使用閉包 (Closures)

如果您在整個應用程式中只需要一次使用自訂規則的功能，您可以使用閉包來代替規則物件。閉包會接收屬性名稱、屬性值以及一個在驗證失敗時應被調用的 `$fail` 回呼函數：

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
### 隱含規則 (Implicit Rules)

預設情況下，當被驗證的屬性不存在或包含空字串時，一般的驗證規則（包括自訂規則）不會被執行。例如，unique 規則不會對空字串執行驗證：

```php
use Illuminate\Support\Facades\Validator;

$rules = ['name' => 'unique:users,name'];

$input = ['name' => ''];

Validator::make($input, $rules)->passes(); // true
```

若要讓自訂規則在屬性為空時仍然執行，該規則必須「隱含」該屬性是必填的。要快速產生一個新的隱含規則物件，您可以使用 `make:rule` Artisan 命令並加上 `--implicit` 選項：

```shell
php artisan make:rule Uppercase --implicit
```

> [!WARNING]
> 「隱含」規則僅是 _隱含_ 該屬性為必填。至於它是否真的會使缺失或空的屬性失效，則由您自行決定。