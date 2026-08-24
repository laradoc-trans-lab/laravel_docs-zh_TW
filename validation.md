# 驗證 (Validation)

- [簡介](#introduction)
- [驗證快速入門](#validation-quickstart)
    - [定義路由](#quick-defining-the-routes)
    - [建立控制器](#quick-creating-the-controller)
    - [撰寫驗證邏輯](#quick-writing-the-validation-logic)
    - [顯示驗證錯誤](#quick-displaying-the-validation-errors)
    - [重新填入表單](#repopulating-forms)
    - [關於選填欄位的說明](#a-note-on-optional-fields)
    - [驗證錯誤回應格式](#validation-error-response-format)
- [表單請求(Form Request) 驗證](#form-request-validation)
    - [建立表單請求](#creating-form-requests)
    - [授權表單請求](#authorizing-form-requests)
    - [自訂錯誤訊息](#customizing-the-error-messages)
    - [準備驗證的輸入資料](#preparing-input-for-validation)
- [手動建立驗證器](#manually-creating-validators)
    - [自動重定向](#automatic-redirection)
    - [具名錯誤包](#named-error-bags)
    - [自訂錯誤訊息](#manual-customizing-the-error-messages)
    - [進行額外驗證](#performing-additional-validation)
- [使用驗證後的輸入資料](#working-with-validated-input)
- [使用錯誤訊息](#working-with-error-messages)
    - [在語言檔中指定自訂訊息](#specifying-custom-messages-in-language-files)
    - [在語言檔中指定屬性](#specifying-attribute-in-language-files)
    - [在語言檔中指定數值](#specifying-values-in-language-files)
- [可用的驗證規則](#available-validation-rules)
- [條件式新增規則](#conditionally-adding-rules)
- [驗證陣列](#validating-arrays)
    - [驗證巢狀陣列輸入](#validating-nested-array-input)
    - [錯誤訊息索引與位置](#error-message-indexes-and-positions)
- [驗證檔案](#validating-files)
- [驗證密碼](#validating-passwords)
- [自訂驗證規則](#custom-validation-rules)
    - [使用規則物件](#using-rule-objects)
    - [使用 Closure](#using-closures)
    - [隱式規則](#implicit-rules)

<a name="introduction"></a>
## 簡介

Laravel 提供了幾種不同的方法來驗證傳入應用程式的資料。最常見的方式是使用所有傳入 HTTP 請求上都可用的 `validate` 方法。不過，我們也會討論其他驗證方式。

Laravel 包含許多方便的驗證規則，您可以將其套用到資料上，甚至提供驗證數值在指定資料庫資料表中是否唯一的能力。我們將詳細介紹這些驗證規則，以便您熟悉 Laravel 的所有驗證功能。

<a name="validation-quickstart"></a>
## 驗證快速入門

若要了解 Laravel 強大的驗證功能，讓我們來看一個驗證表單並將錯誤訊息顯示給使用者的完整範例。透過閱讀這個高階概述，您將能很好地通盤瞭解如何使用 Laravel 驗證傳入的請求資料：


<a name="quick-defining-the-routes"></a>
### 定義路由

首先，假設我們在 `routes/web.php` 檔案中定義了以下路由：

```php
use App\Http\Controllers\PostController;

Route::get('/post/create', [PostController::class, 'create']);
Route::post('/post', [PostController::class, 'store']);
```

`GET` 路由會顯示供使用者建立新部落格文章的表單，而 `POST` 路由則會將新的部落格文章儲存到資料庫中。


<a name="quick-creating-the-controller"></a>
### 建立控制器

接下來，讓我們看看處理這些傳入路由請求的簡單控制器。我們暫時將 `store` 方法留空：

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

現在我們準備在 `store` 方法中填入驗證新部落格文章的邏輯。為此，我們將使用 `Illuminate\Http\Request` 物件提供的 `validate` 方法。如果驗證規則通過，您的程式碼將繼續正常執行；然而，如果驗證失敗，將會拋出 `Illuminate\Validation\ValidationException` 異常，且適當的錯誤回應會自動發回給使用者。

如果在傳統的 HTTP 請求期間驗證失敗，則會產生重定向至前一個 URL 的回應。如果傳入的請求是 XHR 請求，則會傳回[包含驗證錯誤訊息的 JSON 回應](#validation-error-response-format)。

為了更深入了解 `validate` 方法，讓我們回到 `store` 方法：

```php
/**
 * Store a new blog post.
 */
public function store(Request $request): RedirectResponse
{
    $validated = $request->validate([
        'title' => ['required', 'unique:posts', 'max:255'],
        'body' => ['required'],
    ]);

    // The blog post is valid...

    return redirect('/posts');
}
```

如您所見，驗證規則被傳入 `validate` 方法中。別擔心——所有可用的驗證規則都有[文件說明](#available-validation-rules)。同樣地，如果驗證失敗，適當的回應會自動產生。如果驗證通過，我們的控制器將繼續正常執行。

此外，您可以使用 `validateWithBag` 方法來驗證請求，並將任何錯誤訊息儲存在[具名錯誤包](#named-error-bags)中：

```php
$validated = $request->validateWithBag('post', [
    'title' => ['required', 'unique:posts', 'max:255'],
    'body' => ['required'],
]);
```


<a name="stopping-on-first-validation-failure"></a>
#### 首次驗證失敗時停止

有時，您可能希望在某個屬性發生首次驗證失敗後，停止執行該屬性其餘的驗證規則。為此，請將 `bail` 規則指派給該屬性：

```php
$request->validate([
    'title' => ['bail', 'required', 'unique:posts', 'max:255'],
    'body' => ['required'],
]);
```

在此範例中，如果 `title` 屬性上的 `unique` 規則失敗，則不會檢查 `max` 規則。規則將按照指派的順序進行驗證。


<a name="a-note-on-nested-attributes"></a>
#### 關於巢狀屬性的說明

如果傳入的 HTTP 請求包含「巢狀」欄位資料，您可以使用「點」語法在驗證規則中指定這些欄位：

```php
$request->validate([
    'title' => ['required', 'unique:posts', 'max:255'],
    'author.name' => ['required'],
    'author.description' => ['required'],
]);
```

另一方面，如果您的欄位名稱本身包含句點字元，您可以使用反斜線轉義句點，以明確防止其被解釋為「點」語法：

```php
$request->validate([
    'title' => ['required', 'unique:posts', 'max:255'],
    'v1\.0' => ['required'],
]);
```

<a name="quick-displaying-the-validation-errors"></a>
### 顯示驗證錯誤

那麼，如果傳入的請求欄位沒有通過指定的驗證規則會怎樣？如前所述，Laravel 將會自動將使用者重定向回先前的頁面。此外，所有的驗證錯誤與 [請求輸入](/docs/{{version}}/requests#retrieving-old-input) 都會自動被 [暫存至 Session](/docs/{{version}}/session#flash-data)。

由 `web` 中介層群組所提供的 `Illuminate\View\Middleware\ShareErrorsFromSession` 中介層，會將 `$errors` 變數共享給應用程式的所有視圖。套用此中介層後，視圖中將始終可以使用 `$errors` 變數，讓您可以方便地假設 `$errors` 變數已經定義且可以安全地使用。`$errors` 變數會是 `Illuminate\Support\MessageBag` 的實例。關於如何使用該物件的更多資訊，請[參考其說明文件](#working-with-error-messages)。

因此，在我們的範例中，當驗證失敗時，使用者將會被重定向至控制器的 `create` 方法，讓我們可以在視圖中顯示錯誤訊息：

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

Laravel 內建的驗證規則各自都有對應的錯誤訊息，這些訊息位於應用程式的 `lang/en/validation.php` 檔案中。若您的應用程式沒有 `lang` 目錄，您可以使用 `lang:publish` Artisan 指令來指示 Laravel 建立該目錄。

在 `lang/en/validation.php` 檔案中，您會找到每個驗證規則的翻譯項目。您可以根據應用程式的需求隨意更改或修改這些訊息。

此外，您可以將此檔案複製到另一個語言目錄中，以將訊息翻譯為您應用程式所使用的語言。若要瞭解更多關於 Laravel 在地化的資訊，請參考完整的 [在地化文件](/docs/{{version}}/localization)。

> [!WARNING]
> 預設情況下，Laravel 應用程式骨架並不包含 `lang` 目錄。若您想要自訂 Laravel 的語言檔，可以透過 `lang:publish` Artisan 指令來發布它們。


<a name="quick-xhr-requests-and-validation"></a>
#### XHR 請求與驗證

在此範例中，我們使用傳統表單將資料發送到應用程式。然而，許多應用程式會接收來自 JavaScript 前端的 XHR 請求。當在 XHR 請求期間使用 `validate` 方法時，Laravel 不會產生重定向回應。相反地，Laravel 會產生一個 [包含所有驗證錯誤的 JSON 回應](#validation-error-response-format)。這個 JSON 回應將附帶 422 HTTP 狀態碼傳送。


<a name="the-at-error-directive"></a>
#### `@error` 指令

您可以使用 `@error` [Blade](/docs/{{version}}/blade) 指令來快速判斷指定的屬性是否存在驗證錯誤訊息。在 `@error` 指令內部，您可以印出 `$message` 變數來顯示錯誤訊息：

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

如果您正在使用 [具名錯誤包](#named-error-bags)，您可以將錯誤包的名稱作為第二個引數傳遞給 `@error` 指令：

```blade
<input ... class="@error('title', 'post') is-invalid @enderror">
```


<a name="repopulating-forms"></a>
### 重新填入表單

當 Laravel 因驗證錯誤而產生重定向回應時，框架會自動將 [所有請求輸入暫存至 Session](/docs/{{version}}/session#flash-data)。這樣做是為了讓您可以在下一次請求時方便地存取這些輸入資料，並重新填入使用者嘗試提交的表單。

若要從上一次請求中取得暫存的輸入資料，請在 `Illuminate\Http\Request` 實例上呼叫 `old` 方法。`old` 方法將會從 [Session](/docs/{{version}}/session) 中拉取先前暫存的輸入資料：

```php
$title = $request->old('title');
```

Laravel 還提供了一個全域的 `old` 輔助函數。如果您要在 [Blade 樣板](/docs/{{version}}/blade) 中顯示舊有的輸入資料，使用 `old` 輔助函數來重新填入表單會更加方便。如果指定的欄位不存在舊的輸入資料，則會傳回 `null`：

```blade
<input type="text" name="title" value="{{ old('title') }}">
```


<a name="a-note-on-optional-fields"></a>
### 關於選填欄位的說明

預設情況下，Laravel 在您的應用程式全域中介層堆疊中包含了 `TrimStrings` 與 `ConvertEmptyStringsToNull` 中介層。因此，如果您不希望驗證器將 `null` 值視為無效，通常需要將「選填 (Optional)」請求欄位標記為 `nullable`。例如：

```php
$request->validate([
    'title' => ['required', 'unique:posts', 'max:255'],
    'body' => ['required'],
    'publish_at' => ['nullable', 'date'],
]);
```

在這個範例中，我們指定了 `publish_at` 欄位可以是 `null` 或有效的日期表示法。如果沒有在規則定義中增加 `nullable` 修飾詞，驗證器會將 `null` 視為無效的日期。


<a name="validation-error-response-format"></a>
### 驗證錯誤回應格式

當您的應用程式拋出 `Illuminate\Validation\ValidationException` 例外且傳入的 HTTP 請求期望一個 JSON 回應時，Laravel 將會自動為您格式化錯誤訊息，並傳回 `422 Unprocessable Entity` HTTP 回應。

下方您可以檢視驗證錯誤 JSON 回應格式的範例。請注意，巢狀錯誤鍵名會被平坦化為「點」號表示法格式：

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
## 表單請求(Form Request) 驗證


<a name="creating-form-requests"></a>
### 建立表單請求

對於更複雜的驗證情境，您可能會想要建立「表單請求 (Form Request)」。表單請求是封裝了自身驗證與授權邏輯的自訂請求類別。若要建立表單請求類別，您可以使用 `make:request` Artisan CLI 命令：

```shell
php artisan make:request StorePostRequest
```

產生的表單請求類別將會放置於 `app/Http/Requests` 目錄中。如果該目錄不存在，當您執行 `make:request` 命令時將會自動建立。Laravel 產生的每個表單請求都有兩個方法：`authorize` 與 `rules`。

正如您可能猜到的，`authorize` 方法負責判斷目前已認證的使用者是否可以執行該請求所代表的動作，而 `rules` 方法則傳回應該套用到請求資料的驗證規則：

```php
/**
 * Get the validation rules that apply to the request.
 *
 * @return array<string, \Illuminate\Contracts\Validation\ValidationRule|array<mixed>|string>
 */
public function rules(): array
{
    return [
        'title' => ['required', 'unique:posts', 'max:255'],
        'body' => ['required'],
    ];
}
```

> [!NOTE]
> 您可以在 `rules` 方法的簽名中型別提示 (type-hint) 任何您需要的依賴項目。它們將會透過 Laravel [服務容器](/docs/{{version}}/container)自動解構。

那麼，驗證規則是如何進行評估的呢？您只需要在控制器的動作方法上型別提示該請求即可。傳入的表單請求會在呼叫控制器方法之前先進行驗證，這意味著您不需要讓控制器充斥任何驗證邏輯：

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

如果驗證失敗，系統會產生一個重定向回應，將使用者送回他們先前的位址。錯誤訊息也會快閃 (flash) 到 Session 中，以便顯示。如果該請求是 XHR 請求，則會傳回一個 HTTP 422 狀態碼的回應給使用者，其中包含[驗證錯誤的 JSON 格式](#validation-error-response-format)。

> [!NOTE]
> 需要為由 Inertia 驅動的 Laravel 前端新增即時表單請求驗證嗎？請參考 [Laravel Precognition](/docs/{{version}}/precognition)。


<a name="performing-additional-validation-on-form-requests"></a>
#### 進行額外驗證

有時候，您需要在初始驗證完成後執行額外的驗證。您可以透過表單請求的 `after` 方法來實現。

`after` 方法應該傳回一個由 callable 或 Closure 組成的陣列，這些內容將會在驗證完成後被調用。給定的 callable 將接收一個 `Illuminate\Validation\Validator` 實例，允許您在需要時引發額外的錯誤訊息：

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

如前所述，`after` 方法傳回的陣列也可以包含可呼叫類別 (invokable classes)。這些類別的 `__invoke` 方法將會接收一個 `Illuminate\Validation\Validator` 實例：

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
#### 在第一個驗證失敗時停止

透過將 `StopOnFirstFailure` Attribute 新增至您的請求類別，您可以告知驗證器，只要發生單一驗證失敗，就應停止驗證所有屬性：

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
#### 在未知欄位上失敗

透過將 `FailOnUnknownFields` Attribute 新增至您的請求類別，您可以指示 Laravel 拒絕任何未由請求驗證規則定義的傳入欄位：

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

您也可以從 `AppServiceProvider` 中為所有表單請求全域啟用此行為：

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

如果需要，您可以透過將 `false` 傳遞給 Attribute 來停用特定請求的此行為：

```php
#[FailOnUnknownFields(false)]
class PublicWebhookRequest extends FormRequest
{
    // ...
}
```

拒絕未知欄位可以透過防止非預期的輸入鍵流向應用程式更深處，來提供額外的防護以防止大量指派 (mass-assignment) 類型的問題。然而，您仍應設定模型中的 `$fillable` / `$guarded` 屬性，並僅持久化可信、已驗證的輸入。


<a name="customizing-the-redirect-location"></a>
#### 自訂重定向位置

當表單請求驗證失敗時，會產生重定向回應將使用者送回先前的位址。不過，您可以自由地自訂此行為。為此，您可以使用表單請求上的 `RedirectTo` Attribute：

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

或者，如果您想要將使用者重定向至具名路由，您可以使用 `RedirectToRoute` Attribute 代替：

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
#### 自訂錯誤包

當表單請求驗證失敗時，錯誤將會快閃 (flash) 至 `default` 錯誤包。如果您需要將錯誤儲存在不同的[具名錯誤包](#named-error-bags)中，您可以使用表單請求上的 `ErrorBag` Attribute：

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

表單請求類別還包含一個 `authorize` 方法。在此方法中，您可以確認已通過認證的使用者是否真的有權限更新給定的資源。例如，您可以判斷使用者是否確實擁有他們試圖更新的部落格留言。您最有可能在此方法中與您的[授權 Gate 與 Policy](/docs/{{version}}/authorization) 進行互動：

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

由於所有表單請求都繼承了基礎的 Laravel 請求類別，因此我們可以使用 `user` 方法來存取目前已通過認證的使用者。此外，請注意上例中對 `route` 方法的呼叫。此方法允許您存取正在被呼叫的路由所定義的 URI 參數，例如以下範例中的 `{comment}` 參數：

```php
Route::post('/comment/{comment}');
```

因此，如果您的應用程式使用了[路由模型綁定](/docs/{{version}}/routing#route-model-binding)，透過將解析後的模型作為請求的屬性來存取，您的程式碼可以變得更加精簡：

```php
return $this->user()->can('update', $this->comment);
```

如果 `authorize` 方法回傳 `false`，系統將自動回傳 HTTP 403 狀態碼的回應，且您的控制器方法將不會執行。

如果您打算在應用程式的其他部分處理請求的授權邏輯，您可以完全移除 `authorize` 方法，或是直接回傳 `true`：

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
> 您可以在 `authorize` 方法的簽名中型別提示任何您需要的依賴項目。它們將會透過 Laravel [服務容器(Service Container)](/docs/{{version}}/container)自動解析。


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

Laravel 許多內建的驗證規則錯誤訊息都包含 `:attribute` 佔位符。如果您希望將驗證訊息中的 `:attribute` 佔位符替換為自訂的屬性名稱，您可以透過覆寫 `attributes` 方法來指定自訂名稱。此方法應回傳一個包含屬性 / 名稱配對的陣列：

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
### 準備驗證的輸入資料

如果您需要在套用驗證規則之前對來自請求的任何資料進行準備或清理 (Sanitize)，您可以使用 `prepareForValidation` 方法：

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

同樣地，如果您需要在驗證完成後正規化 (Normalize) 任何請求資料，您可以使用 `passedValidation` 方法：

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

如果您不想在請求上使用 `validate` 方法，可以使用 `Validator` [Facade](/docs/{{version}}/facades) 手動建立驗證器實例。Facade 上的 `make` 方法會產生一個新的驗證器實例：

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
            'title' => ['required', 'unique:posts', 'max:255'],
            'body' => ['required'],
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

傳給 `make` 方法的第一個引數是要被驗證的資料。第二個引數則是要套用到該資料的驗證規則陣列。

在判斷請求驗證是否失敗後，您可以使用 `withErrors` 方法將錯誤訊息快閃 (Flash) 存入 Session。使用此方法時，在重定向之後 `$errors` 變數會自動共享給您的 View，讓您可以輕鬆地將錯誤呈現給使用者。`withErrors` 方法接受驗證器、`MessageBag` 或 PHP `array`。


#### 在第一個驗證失敗時停止

`stopOnFirstFailure` 方法會通知驗證器，一旦發生單一驗證失敗，就應該停止驗證所有屬性：

```php
if ($validator->stopOnFirstFailure()->fails()) {
    // ...
}
```


<a name="automatic-redirection"></a>
### 自動重定向

如果您想手動建立驗證器實例，但仍想利用 HTTP 請求的 `validate` 方法所提供的自動重定向功能，可以在現有的驗證器實例上呼叫 `validate` 方法。若驗證失敗，使用者將會被自動重定向；或者如果是 XHR 請求，則會[回傳 JSON 回應](#validation-error-response-format)：

```php
Validator::make($request->all(), [
    'title' => ['required', 'unique:posts', 'max:255'],
    'body' => ['required'],
])->validate();
```

如果驗證失敗，您可以使用 `validateWithBag` 方法將錯誤訊息儲存於[具名錯誤包](#named-error-bags)中：

```php
Validator::make($request->all(), [
    'title' => ['required', 'unique:posts', 'max:255'],
    'body' => ['required'],
])->validateWithBag('post');
```


<a name="named-error-bags"></a>
### 具名錯誤包

若您在單一頁面上有多個表單，您可能希望為包含驗證錯誤的 `MessageBag` 命名，以便能取得特定表單的錯誤訊息。若要做到這點，請將名稱作為第二個引數傳入 `withErrors`：

```php
return redirect('/register')->withErrors($validator, 'login');
```

接著，您就可以從 `$errors` 變數中存取該具名的 `MessageBag` 實例：

```blade
{{ $errors->login->first('email') }}
```


<a name="manual-customizing-the-error-messages"></a>
### 自訂錯誤訊息

如有需要，您可以提供自訂的錯誤訊息，讓驗證器實例使用它們來取代 Laravel 提供的預設錯誤訊息。有幾種方式可以指定自訂訊息。首先，您可以將自訂訊息作為第三個引數傳入 `Validator::make` 方法：

```php
$validator = Validator::make($input, $rules, $messages = [
    'required' => 'The :attribute field is required.',
]);
```

在這個範例中，`:attribute` 佔位符將會被替換為正在被驗證欄位的實際名稱。您也可以在驗證訊息中使用其他佔位符。例如：

```php
$messages = [
    'same' => 'The :attribute and :other must match.',
    'size' => 'The :attribute must be exactly :size.',
    'between' => 'The :attribute value :input is not between :min - :max.',
    'in' => 'The :attribute must be one of the following types: :values',
];
```


<a name="specifying-a-custom-message-for-a-given-attribute"></a>
#### 為指定屬性指定自訂訊息

有時候您可能只想為特定的屬性指定自訂錯誤訊息。您可以使用「點 (dot)」記法來做到這一點。先指定屬性名稱，後面加上規則：

```php
$messages = [
    'email.required' => 'We need to know your email address!',
];
```


<a name="specifying-custom-attribute-values"></a>
#### 指定自訂屬性數值

許多 Laravel 的內建錯誤訊息都包含 `:attribute` 佔位符，該佔位符會被替換為正在驗證的欄位或屬性名稱。若要為特定欄位自訂用來替換這些佔位符的名稱，您可以將自訂屬性陣列作為第四個引數傳給 `Validator::make` 方法：

```php
$validator = Validator::make($input, $rules, $messages, [
    'email' => 'email address',
]);
```


<a name="performing-additional-validation"></a>
### 進行額外驗證

有時您需要在初始驗證完成後執行額外的驗證。您可以透過驗證器的 `after` 方法來完成。`after` 方法接受 Closure 或可呼叫項目的陣列，這些會在驗證完成後被呼叫。傳入的可呼叫項目將會接收一個 `Illuminate\Validation\Validator` 實例，讓您能在必要時發起額外的錯誤訊息：

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

如上所述，`after` 方法也接受可呼叫項目的陣列，如果您的「驗證後」邏輯被封裝在可呼叫的類別中，這特別方便，這些類別將透過其 `__invoke` 方法接收 `Illuminate\Validation\Validator` 實例：

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
## 使用驗證後的輸入資料

在使用表單請求(Form request)或手動建立的驗證器實例驗證傳入的請求資料後，您可能希望取得真正通過驗證的傳入請求資料。這可以透過幾種方式來完成。首先，您可以呼叫表單請求或驗證器實例上的 `validated` 方法。此方法會傳回已通過驗證的資料陣列：

```php
$validated = $request->validated();

$validated = $validator->validated();
```

或者，您可以呼叫表單請求或驗證器實例上的 `safe` 方法。此方法會傳回一個 `Illuminate\Support\ValidatedInput` 實例。該物件提供了 `only`、`except` 與 `all` 方法，用於取得部分已驗證資料或整組已驗證資料的陣列：

```php
$validated = $request->safe()->only(['name', 'email']);

$validated = $request->safe()->except(['name', 'email']);

$validated = $request->safe()->all();
```

此外，`Illuminate\Support\ValidatedInput` 實例可以被迭代並像陣列一樣進行存取：

```php
// Validated data may be iterated...
foreach ($request->safe() as $key => $value) {
    // ...
}

// Validated data may be accessed as an array...
$validated = $request->safe();

$email = $validated['email'];
```

如果您想在已驗證的資料中新增額外的欄位，您可以呼叫 `merge` 方法：

```php
$validated = $request->safe()->merge(['name' => 'Taylor Otwell']);
```

如果您想將已驗證的資料作為 [集合(collection)](/docs/{{version}}/collections) 實例取得，您可以呼叫 `collect` 方法：

```php
$collection = $request->safe()->collect();
```


<a name="working-with-error-messages"></a>
## 使用錯誤訊息

在呼叫 `Validator` 實例上的 `errors` 方法後，您將收到一個 `Illuminate\Support\MessageBag` 實例，該實例提供多種便利的方法來處理錯誤訊息。自動提供給所有視圖的 `$errors` 變數也是 `MessageBag` 類別的一個實例。


<a name="retrieving-the-first-error-message-for-a-field"></a>
#### 取得欄位的第一條錯誤訊息

若要取得指定欄位的第一條錯誤訊息，請使用 `first` 方法：

```php
$errors = $validator->errors();

echo $errors->first('email');
```


<a name="retrieving-all-error-messages-for-a-field"></a>
#### 取得欄位的所有錯誤訊息

如果您需要取得指定欄位的所有訊息陣列，請使用 `get` 方法：

```php
foreach ($errors->get('email') as $message) {
    // ...
}
```

如果您正在驗證陣列表單欄位，可以使用 `*` 字元來取得每個陣列元素的所有訊息：

```php
foreach ($errors->get('attachments.*') as $message) {
    // ...
}
```


<a name="retrieving-all-error-messages-for-all-fields"></a>
#### 取得所有欄位的所有錯誤訊息

若要取得所有欄位的所有訊息陣列，請使用 `all` 方法：

```php
foreach ($errors->all() as $message) {
    // ...
}
```


<a name="determining-if-messages-exist-for-a-field"></a>
#### 判斷欄位是否存在錯誤訊息

`has` 方法可用於判斷指定欄位是否存在任何錯誤訊息：

```php
if ($errors->has('email')) {
    // ...
}
```


<a name="specifying-custom-messages-in-language-files"></a>
### 在語言檔中指定自訂訊息

Laravel 的內建驗證規則各自有一條錯誤訊息，部位於您應用程式的 `lang/en/validation.php` 檔案中。如果您的應用程式沒有 `lang` 目錄，您可以使用 `lang:publish` Artisan 指令指示 Laravel 建立該目錄。

在 `lang/en/validation.php` 檔案中，您會找到每個驗證規則的翻譯項目。您可以根據應用程式的需求隨意更改或修改這些訊息。

此外，您可以將此檔案複製到另一個語言目錄，以翻譯您應用程式所用語言的訊息。若要瞭解更多關於 Laravel 在地化的資訊，請參考完整的[在地化說明文件](/docs/{{version}}/localization)。

> [!WARNING]
> 預設情況下，Laravel 應用程式骨架不包含 `lang` 目錄。如果您想自訂 Laravel 的語言檔，可以透過 `lang:publish` Artisan 指令來發布它們。


<a name="custom-messages-for-specific-attributes"></a>
#### 特定屬性的自訂訊息

您可以在應用程式的驗證語言檔中，自訂指定屬性與規則組合所使用的錯誤訊息。為此，請將您的訊息自訂內容新增至應用程式 `lang/xx/validation.php` 語言檔的 `custom` 陣列中：

```php
'custom' => [
    'email' => [
        'required' => 'We need to know your email address!',
        'max' => 'Your email address is too long!'
    ],
],
```


<a name="specifying-attribute-in-language-files"></a>
### 在語言檔中指定屬性

Laravel 的許多內建錯誤訊息都包含 `:attribute` 預留位置，該預留位置會替換為正在驗證的欄位或屬性名稱。如果您希望驗證訊息中的 `:attribute` 部分替換為自訂數值，您可以在 `lang/xx/validation.php` 語言檔的 `attributes` 陣列中指定自訂屬性名稱：

```php
'attributes' => [
    'email' => 'email address',
],
```

> [!WARNING]
> 預設情況下，Laravel 應用程式骨架不包含 `lang` 目錄。如果您想自訂 Laravel 的語言檔，可以透過 `lang:publish` Artisan 指令來發布它們。


<a name="specifying-values-in-language-files"></a>
### 在語言檔中指定數值

Laravel 的某些內建驗證規則錯誤訊息包含 `:value` 預留位置，該預留位置會替換為請求屬性的當前數值。但是，有時您可能需要將驗證訊息中的 `:value` 部分替換為更平易近人的數值表示方式。例如，考慮以下規則，該規則指定如果 `payment_type` 的數值為 `cc`，則必須填寫信用卡卡號：

```php
Validator::make($request->all(), [
    'credit_card_number' => ['required_if:payment_type,cc']
]);
```

如果此驗證規則失敗，它將產生以下錯誤訊息：

```text
The credit card number field is required when payment type is cc.
```

您可以透過在 `lang/xx/validation.php` 語言檔中定義 `values` 陣列，來指定更符合使用者習慣的數值表示方式，而不是顯示 `cc` 作為付款方式的數值：

```php
'values' => [
    'payment_type' => [
        'cc' => 'credit card'
    ],
],
```

> [!WARNING]
> 預設情況下，Laravel 應用程式骨架不包含 `lang` 目錄。如果您想自訂 Laravel 的語言檔，可以透過 `lang:publish` Artisan 指令來發布它們。

定義此數值後，驗證規則將產生以下錯誤訊息：

```text
The credit card number field is required when payment type is credit card.
```

<a name="available-validation-rules"></a>
## 可用的驗證規則

以下是所有可用的驗證規則及其功能的清單：

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
[Array Keys](#rule-array-keys)
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
[Min](#rule-min)
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

驗證中的欄位必須為 `"yes"`、`"on"`、`1`、`"1"`、`true` 或 `"true"`。這對於驗證「服務條款」的同意或類似欄位很有用。


<a name="rule-accepted-if"></a>
#### accepted_if:anotherfield,value,...

當另一個驗證中的欄位等於指定數值時，驗證中的欄位必須為 `"yes"`、`"on"`、`1`、`"1"`、`true` 或 `"true"`。這對於驗證「服務條款」的同意或類似欄位很有用。


<a name="rule-active-url"></a>
#### active_url

根據 PHP 的 `dns_get_record` 函式，驗證中的欄位必須擁有有效的 A 或 AAAA 記錄。在傳送至 `dns_get_record` 之前，會先使用 PHP 的 `parse_url` 函式擷取所提供 URL 的主機名稱 (Hostname)。

當測試執行 DNS 尋找 (DNS Lookups) 的驗證規則（例如 `active_url` 及 `email:dns`）時，您可以使用 `Validator::fakeDnsLookups` 方法。這會模擬 DNS 尋找，同時保留規則的其他驗證行為：

```php
use Illuminate\Support\Facades\Validator;

Validator::fakeDnsLookups();
```


<a name="rule-after"></a>
#### after:_date_

驗證中的欄位必須是指定日期之後的值。這些日期將傳入 PHP 的 `strtotime` 函式，以便轉換為有效的 `DateTime` 實例：

```php
'start_date' => ['required', 'date', 'after:tomorrow']
```

除了傳入由 `strtotime` 評估的日期字串外，您也可以指定另一個欄位來與該日期進行比較：

```php
'finish_date' => ['required', 'date', 'after:start_date']
```

為了方便起見，基於日期的規則可以使用流暢的 `date` 規則建立器來建構：

```php
use Illuminate\Validation\Rule;

'start_date' => [
    'required',
    Rule::date()->after(today()->addDays(7)),
],
```

可以使用 `afterToday` 及 `todayOrAfter` 方法以流暢方式表達日期，分別代表必須在今天之後、或是今天（含）之後：

```php
'start_date' => [
    'required',
    Rule::date()->afterToday(),
],
```


<a name="rule-after-or-equal"></a>
#### after\_or\_equal:_date_

驗證中的欄位必須是晚於或等於指定日期的值。更多資訊請參考 [after](#rule-after) 規則。

為了方便起見，基於日期的規則可以使用流暢的 `date` 規則建立器來建構：

```php
use Illuminate\Validation\Rule;

'start_date' => [
    'required',
    Rule::date()->afterOrEqual(today()->addDays(7)),
],
```


<a name="rule-anyof"></a>
#### anyOf

`Rule::anyOf` 驗證規則允許您指定驗證中的欄位必須滿足任何一組給定的驗證規則集。例如，以下規則將驗證 `username` 欄位必須是電子郵件地址，或者是至少 6 個字元長的英數字字串（包含破折號）：

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

驗證中的欄位必須完全是由包含在 [\p{L}](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AL%3A%5D&g=&i=) 與 [\p{M}](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AM%3A%5D&g=&i=) 中的 Unicode 字母字元所組成。

若要將此驗證規則限制為 ASCII 範圍內的字元（`a-z` 與 `A-Z`），您可以為驗證規則提供 `ascii` 選項：

```php
'username' => ['alpha:ascii'],
```

<a name="rule-alpha-dash"></a>
#### alpha_dash

驗證中的欄位必須完全是由包含在 [\p{L}](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AL%3A%5D&g=&i=)、[\p{M}](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AM%3A%5D&g=&i=)、[\p{N}](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AN%3A%5D&g=&i=) 中的 Unicode 字母與數字字元，以及 ASCII 破折號 (`-`) 與 ASCII 底線 (`_`) 所組成。

若要將此驗證規則限制為 ASCII 範圍內的字元（`a-z`、`A-Z` 與 `0-9`），您可以為該驗證規則提供 `ascii` 選項：

```php
'username' => ['alpha_dash:ascii'],
```


<a name="rule-alpha-num"></a>
#### alpha_num

驗證中的欄位必須完全是由包含在 [\p{L}](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AL%3A%5D&g=&i=)、[\p{M}](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AM%3A%5D&g=&i=) 與 [\p{N}](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AN%3A%5D&g=&i=) 中的 Unicode 字母與數字字元所組成。

若要將此驗證規則限制為 ASCII 範圍內的字元（`a-z`、`A-Z` 與 `0-9`），您可以為該驗證規則提供 `ascii` 選項：

```php
'username' => ['alpha_num:ascii'],
```


<a name="rule-array"></a>
#### array

驗證中的欄位必須是一個 PHP `array`。

當為 `array` 規則提供額外的數值時，輸入陣列中的每個鍵值（Key）都必須存在於提供給該規則的數值清單中。在以下範例中，輸入陣列中的 `admin` 鍵值是無效的，因為它並未包含在提供給 `array` 規則的數值清單中：

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
    'user' => ['array:name,username'],
]);
```

一般來說，您應該始終指定允許存在於陣列中的陣列鍵值。


<a name="rule-array-keys"></a>
#### array_keys:_foo_,_bar_,...

驗證中的欄位必須是一個 PHP `array`，且其所有鍵值都包含在給定的清單中。必須至少提供一個鍵值：

```php
'user' => ['array_keys:name,username'],
```

為求方便，您可以使用 `Rule::arrayKeys` 方法：

```php
'user' => [Rule::arrayKeys('name', 'username')],
```


<a name="rule-ascii"></a>
#### ascii

驗證中的欄位必須完全由 7 位元 ASCII 字元組成。


<a name="rule-bail"></a>
#### bail

當欄位發生第一次驗證失敗時，即停止對該欄位執行後續的驗證規則。

雖然 `bail` 規則只會在遇到驗證失敗時停止驗證特定的欄位，但 `stopOnFirstFailure` 方法會通知驗證器，一旦發生單一驗證失敗時，就應該停止驗證所有屬性：

```php
if ($validator->stopOnFirstFailure()->fails()) {
    // ...
}
```


<a name="rule-before"></a>
#### before:_date_

驗證中的欄位數值必須早於給定的日期。這些日期將會傳入 PHP 的 `strtotime` 函式，以便轉換為有效的 `DateTime` 實例。此外，與 [after](#rule-after) 規則一樣，也可以將另一個驗證中欄位的名稱作為 `date` 的數值傳入。

為求方便，基於日期的規則也可以使用流暢的 `date` 規則建構器來建立：

```php
use Illuminate\Validation\Rule;

'start_date' => [
    'required',
    Rule::date()->before(today()->subDays(7)),
],
```

`beforeToday` 與 `todayOrBefore` 方法可用於流暢地表達日期，並且分別必須早於今天，或是今天或早於今天：

```php
'start_date' => [
    'required',
    Rule::date()->beforeToday(),
],
```


<a name="rule-before-or-equal"></a>
#### before\_or\_equal:_date_

驗證中的欄位數值必須早於或等於給定的日期。這些日期將會傳入 PHP 的 `strtotime` 函式，以便轉換為有效的 `DateTime` 實例。此外，與 [after](#rule-after) 規則一樣，也可以將另一個驗證中欄位的名稱作為 `date` 的數值傳入。

為求方便，基於日期的規則也可以使用流暢的 `date` 規則建構器來建立：

```php
use Illuminate\Validation\Rule;

'start_date' => [
    'required',
    Rule::date()->beforeOrEqual(today()->subDays(7)),
],
```


<a name="rule-between"></a>
#### between:_min_,_max_

驗證中的欄位大小必須介於給定的 _min_ 與 _max_（包含）之間。字串、數字、陣列與檔案的評估方式與 [size](#rule-size) 規則相同。


<a name="rule-boolean"></a>
#### boolean

驗證中的欄位必須能夠轉型為布林值。接受的輸入包含 `true`、`false`、`1`、`0`、`"1"` 以及 `"0"`。

您可以使用 `strict` 參數，只有當欄位的數值為 `true` 或 `false` 時才視為有效：

```php
'foo' => ['boolean:strict']
```


<a name="rule-confirmed"></a>
#### confirmed

驗證中的欄位必須有一個相配對的 `{field}_confirmation` 欄位。例如，若驗證中的欄位是 `password`，輸入資料中就必須存在一個相配對的 `password_confirmation` 欄位。

您也可以傳入自訂的確認欄位名稱。例如，`confirmed:repeat_username` 會期望 `repeat_username` 欄位與驗證中的欄位相符。


<a name="rule-contains"></a>
#### contains:_foo_,_bar_,...

驗證中的欄位必須是一個包含所有給定參數數值的陣列。由於此規則通常需要對陣列進行 `implode`，因此可以使用 `Rule::contains` 方法來流暢地建構此規則：

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

驗證中的欄位必須是一個不包含任何給定參數數值的陣列。由於此規則通常需要對陣列進行 `implode`，因此可以使用 `Rule::doesntContain` 方法來流暢地建構此規則：

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

驗證中的欄位必須與已認證使用者的密碼相符。您可以使用該規則的第一個參數來指定 [認證 Guard](/docs/{{version}}/authentication)：

```php
'password' => ['current_password:api']
```


<a name="rule-date"></a>
#### date

根據 PHP 的 `strtotime` 函式，驗證中的欄位必須是一個有效的、非相對時間的日期。


<a name="rule-date-equals"></a>
#### date_equals:_date_

驗證中的欄位必須等於給定的日期。這些日期將傳入 PHP 的 `strtotime` 函式，以便轉換為有效的 `DateTime` 實例。


<a name="rule-date-format"></a>
#### date_format:_format_,...

驗證中的欄位必須符合給定 _format_（格式）中的其中之一。驗證欄位時，您應該選擇使用 `date` **或是** `date_format`，而非兩者皆用。此驗證規則支援 PHP 的 [DateTime](https://www.php.net/manual/en/class.datetime.php) 類別所支援的所有格式。

為求方便，基於日期的規則可以使用流暢的 `date` 規則建構器來建立：

```php
use Illuminate\Validation\Rule;

'start_date' => [
    'required',
    Rule::date()->format('Y-m-d'),
],
```


<a name="rule-decimal"></a>
#### decimal:_min_,_max_

驗證中的欄位必須是數字，且必須包含指定的小數位數：

```php
// Must have exactly two decimal places (9.99)...
'price' => ['decimal:2']

// Must have between 2 and 4 decimal places...
'price' => ['decimal:2,4']
```


<a name="rule-declined"></a>
#### declined

驗證中的欄位必須是 `"no"`、`"off"`、`0`、`"0"`、`false` 或 `"false"`。

<a name="rule-declined-if"></a>
#### declined_if:anotherfield,value,...

當指定的另一個欄位 _anotherfield_ 等於 _value_ 時，受驗證的欄位必須為 `"no"`、`"off"`、`0`、`"0"`、`false` 或 `"false"`。


<a name="rule-different"></a>
#### different:_field_

受驗證的欄位值必須與指定的欄位 _field_ 不同。


<a name="rule-digits"></a>
#### digits:_value_

受驗證的整數必須具備精確的長度 _value_。


<a name="rule-digits-between"></a>
#### digits_between:_min_,_max_

受驗證的整數長度必須介於指定的 _min_ 與 _max_ 之間。


<a name="rule-dimensions"></a>
#### dimensions

受驗證的檔案必須是符合規則參數所指定尺寸限制的圖片：

```php
'avatar' => ['dimensions:min_width=100,min_height=200']
```

可用的限制條件有：_min\_width_、_max\_width_、_min\_height_、_max\_height_、_width_、_height_、_ratio_、_min\_ratio_、_max\_ratio_。

_ratio_ 限制條件應表示為寬度除以高度。這可以透過像 `3/2` 這樣的分數或像 `1.5` 這樣的浮點數來指定：

```php
'avatar' => ['dimensions:ratio=3/2']
```

_min\_ratio_ 與 _max\_ratio_ 限制條件可用於定義可接受的長寬比範圍：

```php
'avatar' => ['dimensions:min_ratio=1/2,max_ratio=3/2']
```

由於此規則需要多個引數，使用 `Rule::dimensions` 方法來順暢建構規則通常更為方便：

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

你也可以使用 `minRatio`、`maxRatio` 和 `ratioBetween` 方法來順暢地定義比例限制：

```php
Rule::dimensions()->ratioBetween(min: 1 / 2, max: 3 / 2)
```


<a name="rule-distinct"></a>
#### distinct

在驗證陣列時，受驗證的欄位不能包含任何重複的值：

```php
'foo.*.id' => ['distinct']
```

Distinct 預設使用鬆散的變數比較。若要使用嚴格比較，可在驗證規則定義中加入 `strict` 參數：

```php
'foo.*.id' => ['distinct:strict']
```

可在驗證規則的引數中加入 `ignore_case`，讓規則忽略大小寫的差異：

```php
'foo.*.id' => ['distinct:ignore_case']
```


<a name="rule-doesnt-start-with"></a>
#### doesnt_start_with:_foo_,_bar_,...

受驗證的欄位不能以任何給定的值開頭。


<a name="rule-doesnt-end-with"></a>
#### doesnt_end_with:_foo_,_bar_,...

受驗證的欄位不能以任何給定的值結尾。


<a name="rule-email"></a>
#### email

受驗證的欄位必須符合電子郵件地址格式。此驗證規則採用 [egulias/email-validator](https://github.com/egulias/EmailValidator) 套件來驗證電子郵件地址。預設會套用 `RFCValidation` 驗證器，但你也可以套用其他的驗證樣式：

```php
'email' => ['email:rfc,dns']
```

上述範例將套用 `RFCValidation` 與 `DNSCheckValidation` 驗證。以下是你可以套用的完整驗證樣式清單：

<div class="content-list" markdown="1">

- `rfc`: `RFCValidation` - 依據[支援的 RFC](https://github.com/egulias/EmailValidator?tab=readme-ov-file#supported-rfcs) 驗證電子郵件地址。
- `strict`: `NoRFCWarningsValidation` - 依據[支援的 RFC](https://github.com/egulias/EmailValidator?tab=readme-ov-file#supported-rfcs) 驗證電子郵件，當發現警告時（例如結尾句點與連續多個句點）即驗證失敗。
- `dns`: `DNSCheckValidation` - 確保電子郵件地址的網域具有有效的 MX 記錄。
- `spoof`: `SpoofCheckValidation` - 確保電子郵件地址不包含同形異義詞或具欺騙性的 Unicode 字元。
- `filter`: `FilterEmailValidation` - 依據 PHP 的 `filter_var` 函式確保電子郵件地址有效。
- `filter_unicode`: `FilterEmailValidation::unicode()` - 依據 PHP 的 `filter_var` 函式確保電子郵件地址有效，並允許部分 Unicode 字元。

</div>

為方便起見，可以使用流暢的規則建構器來建立電子郵件驗證規則：

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

受驗證的欄位必須符合指定的字元編碼。此規則使用 PHP 的 `mb_check_encoding` 函式來確認指定檔案或字串值的編碼。為方便起見，可以使用 Laravel 的流暢檔案規則建構器來建構 `encoding` 規則：

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

受驗證的欄位必須以給定的值之一結尾。


<a name="rule-enum"></a>
#### enum

`Enum` 規則是一個基於類別的規則，用於驗證受驗證的欄位是否包含有效的 Enum 值。`Enum` 規則接受 Enum 的名稱作為其唯一的建構子引數。在驗證基本型別值時，應提供 Backed Enum 給 `Enum` 規則：

```php
use App\Enums\ServerStatus;
use Illuminate\Validation\Rule;

$request->validate([
    'status' => [Rule::enum(ServerStatus::class)],
]);
```

`Enum` 規則的 `only` 和 `except` 方法可用於限制哪些 Enum 案件應被視為有效：

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

受驗證的欄位將被排除在 `validate` 和 `validated` 方法所返回的請求資料之外。


<a name="rule-exclude-if"></a>
#### exclude_if:_anotherfield_,_value_

如果指定的欄位 _anotherfield_ 等於 _value_，受驗證的欄位將被排除在 `validate` 和 `validated` 方法所返回的請求資料之外。

如果需要複雜的條件式排除邏輯，可以使用 `Rule::excludeIf` 方法。此方法接受布林值或 Closure。當給定 Closure 時，Closure 應返回 `true` 或 `false` 以表示受驗證的欄位是否應該被排除：

```php
use Illuminate\Support\Facades\Validator;
use Illuminate\Validation\Rule;

Validator::make($request->all(), [
    'role_id' => [Rule::excludeIf($request->user()->is_admin)],
]);

Validator::make($request->all(), [
    'role_id' => [Rule::excludeIf(fn () => $request->user()->is_admin)],
]);
```

<a name="rule-exclude-unless"></a>
#### exclude_unless:_anotherfield_,_value_

除非 _anotherfield_ 欄位等於 _value_，否則受驗證的欄位將從 `validate` 與 `validated` 方法所回傳的請求資料中排除。若 _value_ 為 `null` (`exclude_unless:name,null`)，則除非比較欄位為 `null` 或請求資料中缺少該比較欄位，否則受驗證欄位將被排除。

若需要複雜的條件式排除邏輯，您可以使用 `Rule::excludeUnless` 方法。此方法接受布林值或 Closure。當給定 Closure 時，Closure 應回傳 `true` 或 `false` 以指出受驗證的欄位是否不應被排除：

```php
use Illuminate\Support\Facades\Validator;
use Illuminate\Validation\Rule;

Validator::make($request->all(), [
    'role_id' => [Rule::excludeUnless($request->user()->is_admin)],
]);

Validator::make($request->all(), [
    'role_id' => [Rule::excludeUnless(fn () => $request->user()->is_admin)],
]);
```


<a name="rule-exclude-with"></a>
#### exclude_with:_anotherfield_

若 _anotherfield_ 欄位存在，受驗證的欄位將從 `validate` 與 `validated` 方法所回傳的請求資料中排除。


<a name="rule-exclude-without"></a>
#### exclude_without:_anotherfield_

若 _anotherfield_ 欄位不存在，受驗證的欄位將從 `validate` 與 `validated` 方法所回傳的請求資料中排除。


<a name="rule-exists"></a>
#### exists:_table_,_column_

受驗證的欄位必須存在於給定的資料庫資料表中。


<a name="basic-usage-of-exists-rule"></a>
#### Exists 規則的基本用法

```php
'state' => ['exists:states']
```

若未指定 `column` 選項，將會使用欄位名稱。因此在此例中，該規則將驗證 `states` 資料庫資料表中是否包含一筆 `state` 欄位值與請求的 `state` 屬性值相符的紀錄。


<a name="specifying-a-custom-column-name"></a>
#### 指定自訂欄位名稱

您可以透過在資料庫資料表名稱後方加上欄位名稱，明確指定驗證規則所應使用的資料庫欄位名稱：

```php
'state' => ['exists:states,abbreviation']
```

有時，您可能需要指定用於 `exists` 查詢的特定資料庫連線。您可以透過將連線名稱加在資料表名稱前來完成：

```php
'email' => ['exists:connection.staff,email']
```

除了直接指定資料表名稱外，您也可以指定用於確定資料表名稱的 Eloquent 模型：

```php
'user_id' => ['exists:App\Models\User,id']
```

若您想自訂驗證規則所執行的查詢，可以使用 `Rule` 類別流暢地定義規則。

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

您可以將欄位名稱作為第二個引數傳遞給 `exists` 方法，明確指定由 `Rule::exists` 方法產生的 `exists` 規則所應使用的資料庫欄位名稱：

```php
'state' => [Rule::exists('states', 'abbreviation')],
```

有時，您可能想要驗證陣列中的數值是否存在於資料庫中。您可以透過將 `exists` 與 [array](#rule-array) 規則一併賦予給受驗證欄位來實現：

```php
'states' => ['array', Rule::exists('states', 'abbreviation')],
```

當這兩個規則都被賦予給某個欄位時，Laravel 將會自動建立單一查詢，以確定給定的所有數值是否都存在於指定的資料表中。


<a name="rule-extensions"></a>
#### extensions:_foo_,_bar_,...

受驗證的檔案必須具有對應於所列副檔名之一的使用者自訂副檔名：

```php
'photo' => ['required', 'extensions:jpg,png'],
```

> [!WARNING]
> 您絕不應該僅依賴使用者自訂的副檔名來驗證檔案。此規則通常應一律與 [mimes](#rule-mimes) 或 [mimetypes](#rule-mimetypes) 規則搭配使用。


<a name="rule-file"></a>
#### file

受驗證的欄位必須是成功上傳的檔案。


<a name="rule-filled"></a>
#### filled

受驗證的欄位在存在時不得為空。


<a name="rule-gt"></a>
#### gt:_field_

受驗證的欄位必須大於給定的 _field_ 或 _value_。這兩個欄位必須是相同型別。字串、數值、陣列與檔案會使用與 [size](#rule-size) 規則相同的慣例進行評估。


<a name="rule-gte"></a>
#### gte:_field_

受驗證的欄位必須大於或等於給定的 _field_ 或 _value_。這兩個欄位必須是相同型別。字串、數值、陣列與檔案會使用與 [size](#rule-size) 規則相同的慣例進行評估。


<a name="rule-hex-color"></a>
#### hex_color

受驗證的欄位必須包含 [十六進位](https://developer.mozilla.org/en-US/docs/Web/CSS/hex-color) 格式的有效顏色值。


<a name="rule-image"></a>
#### image

受驗證的檔案必須是圖片 (jpg, jpeg, png, bmp, gif, 或 webp)。

> [!WARNING]
> 預設情況下，image 規則因考量到可能存在 XSS 漏洞而不允許 SVG 檔案。若您需要允許 SVG 檔案，您可以向 `image` 規則提供 `allow_svg` 指令 (`image:allow_svg`)。


<a name="rule-in"></a>
#### in:_foo_,_bar_,...

受驗證的欄位必須包含在給定的數值清單中。由於此規則通常需要對陣列進行 `implode`，您可以使用 `Rule::in` 方法來流暢地建構此規則：

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

當 `in` 規則與 `array` 規則組合使用時，輸入陣列中的每個值都必須存在於提供給 `in` 規則的數值清單中。在以下範例中，輸入陣列中的 `LAS` 機場代碼是無效的，因為它未包含在提供給 `in` 規則的機場清單中：

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

受驗證的欄位必須存在於 _anotherfield_ 的數值中。


<a name="rule-in-array-keys"></a>
#### in_array_keys:_value_.*

受驗證的欄位必須是一個陣列，且該陣列的鍵 (Key) 中至少包含給定 _values_ 的其中一個：

```php
'config' => ['array', 'in_array_keys:timezone']
```


<a name="rule-integer"></a>
#### integer

受驗證的欄位必須是整數。

您可以使用 `strict` 參數，使其僅在欄位型別為 `integer` 時才視為有效。具有整數值的字串將被視為無效：

```php
'age' => ['integer:strict']
```

> [!WARNING]
> 此驗證規則不會驗證輸入是否為「整數」變數型別，僅驗證輸入是否為 PHP 的 `FILTER_VALIDATE_INT` 規則所接受的型別。若您需要驗證輸入是否為數字，請將此規則與 [`numeric` 驗證規則](#rule-numeric) 結合使用。


<a name="rule-ip"></a>
#### ip

受驗證的欄位必須是 IP 位址。


<a name="ipv4"></a>
#### ipv4

受驗證的欄位必須是 IPv4 位址。


<a name="ipv6"></a>
#### ipv6

受驗證的欄位必須是 IPv6 位址。


<a name="rule-json"></a>
#### json

受驗證的欄位必須是有效的 JSON 字串。


<a name="rule-lt"></a>
#### lt:_field_

受驗證的欄位必須小於給定的 _field_。這兩個欄位必須是相同型別。字串、數值、陣列與檔案會使用與 [size](#rule-size) 規則相同的慣例進行評估。

<a name="rule-lte"></a>
#### lte:_field_

驗證欄位的值必須小於或等於給定的 _field_。這兩個欄位必須具有相同的型別。字串、數值、陣列和檔案的評估方式與 [size](#rule-size) 規則相同的慣例。


<a name="rule-lowercase"></a>
#### lowercase

驗證欄位必須為小寫。


<a name="rule-list"></a>
#### list

驗證欄位必須是一個列表 (list) 陣列。如果一個陣列的鍵由從 0 到 `count($array) - 1` 的連續數字組成，則該陣列會被視為列表。


<a name="rule-mac"></a>
#### mac_address

驗證欄位必須為 MAC 位址。


<a name="rule-max"></a>
#### max:_value_

驗證欄位必須小於或等於最大值 _value_。字串、數值、陣列和檔案的評估方式與 [size](#rule-size) 規則相同。


<a name="rule-max-digits"></a>
#### max_digits:_value_

驗證的整數長度最多為 _value_。


<a name="rule-mimetypes"></a>
#### mimetypes:_text/plain_,...

驗證的檔案必須符合給定的 MIME 類型之一：

```php
'video' => ['mimetypes:video/avi,video/mpeg,video/quicktime'],

'media' => ['mimetypes:image/*,video/*'],
```

為確定上傳檔案的 MIME 類型，框架將讀取檔案內容並嘗試猜測其 MIME 類型，這可能與用戶端提供的 MIME 類型不同。


<a name="rule-mimes"></a>
#### mimes:_foo_,_bar_,...

驗證的檔案必須具有與列出的副檔名之一對應的 MIME 類型：

```php
'photo' => ['mimes:jpg,bmp,png']
```

儘管您只需要指定副檔名，但此規則實際上會透過讀取檔案內容並猜測其 MIME 類型來驗證檔案的 MIME 類型。MIME 類型及其對應副檔名的完整列表可在以下位置找到：

[https://svn.apache.org/repos/asf/httpd/httpd/trunk/docs/conf/mime.types](https://svn.apache.org/repos/asf/httpd/httpd/trunk/docs/conf/mime.types)


<a name="mime-types-and-extensions"></a>
#### MIME 類型與副檔名

此驗證規則不會確認 MIME 類型與使用者賦予檔案的副檔名之間是否一致。例如，`mimes:png` 驗證規則會將包含有效 PNG 內容的檔案視為有效的 PNG 圖片，即使該檔案命名為 `photo.txt`。若您想驗證使用者賦予檔案的副檔名，可以使用 [extensions](#rule-extensions) 規則。


<a name="rule-min"></a>
#### min:_value_

驗證欄位必須具有最小值 _value_。字串、數值、陣列和檔案的評估方式與 [size](#rule-size) 規則相同。


<a name="rule-min-digits"></a>
#### min_digits:_value_

驗證的整數長度最少為 _value_。


<a name="rule-multiple-of"></a>
#### multiple_of:_value_

驗證欄位必須是 _value_ 的倍數。


<a name="rule-missing"></a>
#### missing

驗證欄位不得存在於輸入資料中。


<a name="rule-missing-if"></a>
#### missing_if:_anotherfield_,_value_,...

若 _anotherfield_ 欄位等於任何 _value_，則驗證欄位不得存在。


<a name="rule-missing-unless"></a>
#### missing_unless:_anotherfield_,_value_

除非 _anotherfield_ 欄位等於任何 _value_，否則驗證欄位不得存在。


<a name="rule-missing-with"></a>
#### missing_with:_foo_,_bar_,...

_僅當_指定的任何其他欄位存在時，驗證欄位才不得存在。


<a name="rule-missing-with-all"></a>
#### missing_with_all:_foo_,_bar_,...

_僅當_指定的所有其他欄位皆存在時，驗證欄位才不得存在。


<a name="rule-not-in"></a>
#### not_in:_foo_,_bar_,...

驗證欄位不得包含在給定的數值列表中。可以使用 `Rule::notIn` 方法來順暢地建構規則：

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

驗證欄位不得符合給定的正規表示式。

在內部，此規則使用 PHP 的 `preg_match` 函式。指定的模式應遵循 `preg_match` 所需的相同格式，因此也包含有效的界定符。例如：`'email' => ['not_regex:/^.+$/i']`。


<a name="rule-nullable"></a>
#### nullable

驗證欄位可以為 `null`。


<a name="rule-numeric"></a>
#### numeric

驗證欄位必須為[數字 (numeric)](https://www.php.net/manual/en/function.is-numeric.php)。

您可以使用 `strict` 參數，僅在其值為整數或浮點數型別時才將欄位視為有效。數字字串將被視為無效：

```php
'amount' => ['numeric:strict']
```


<a name="rule-present"></a>
#### present

驗證欄位必須存在於輸入資料中。


<a name="rule-present-if"></a>
#### present_if:_anotherfield_,_value_,...

若 _anotherfield_ 欄位等於任何 _value_，則驗證欄位必須存在。


<a name="rule-present-unless"></a>
#### present_unless:_anotherfield_,_value_

除非 _anotherfield_ 欄位等於任何 _value_，否則驗證欄位必須存在。


<a name="rule-present-with"></a>
#### present_with:_foo_,_bar_,...

_僅當_指定的任何其他欄位存在時，驗證欄位才必須存在。


<a name="rule-present-with-all"></a>
#### present_with_all:_foo_,_bar_,...

_僅當_指定的所有其他欄位皆存在時，驗證欄位才必須存在。


<a name="rule-prohibited"></a>
#### prohibited

驗證欄位必須缺失或是為空。如果欄位符合以下標準之一，則該欄位為「空」：

<div class="content-list" markdown="1">

- 數值為 `null`。
- 數值為空字串。
- 數值為空陣列或空的 `Countable` 物件。
- 數值為路徑為空的上傳檔案。

</div>


<a name="rule-prohibited-if"></a>
#### prohibited_if:_anotherfield_,_value_,...

若 _anotherfield_ 欄位等於任何 _value_，則驗證欄位必須缺失或是為空。如果欄位符合以下標準之一，則該欄位為「空」：

<div class="content-list" markdown="1">

- 數值為 `null`。
- 數值為空字串。
- 數值為空陣列或空的 `Countable` 物件。
- 數值為路徑為空的上傳檔案。

</div>

如果需要複雜的條件式禁止邏輯，您可以使用 `Rule::prohibitedIf` 方法。此方法接受布林值或 Closure。當給定 Closure 時，Closure 應傳回 `true` 或 `false` 以指示是否應禁止驗證欄位：

```php
use Illuminate\Support\Facades\Validator;
use Illuminate\Validation\Rule;

Validator::make($request->all(), [
    'role_id' => [Rule::prohibitedIf($request->user()->is_admin)],
]);

Validator::make($request->all(), [
    'role_id' => [Rule::prohibitedIf(fn () => $request->user()->is_admin)],
]);
```

<a name="rule-prohibited-if-accepted"></a>
#### prohibited_if_accepted:_anotherfield_,...

若 _anotherfield_ 欄位等於 `"yes"`、`"on"`、`1`、`"1"`、`true` 或 `"true"`，則驗證欄位必須缺失或是為空。


<a name="rule-prohibited-if-declined"></a>
#### prohibited_if_declined:_anotherfield_,...

若 _anotherfield_ 欄位等於 `"no"`、`"off"`、`0`、`"0"`、`false` 或 `"false"`，則驗證欄位必須缺失或是為空。

<a name="rule-prohibited-unless"></a>
#### prohibited_unless:_anotherfield_,_value_,...

正在驗證的欄位必須不存在或是空的，除非 _anotherfield_ 欄位等於任何一個 _value_。如果符合以下任一條件，該欄位會被視為「空的」：

<div class="content-list" markdown="1">

- 數值為 `null`。
- 數值為空字串。
- 數值為空陣列或空的 `Countable` 物件。
- 數值為路徑為空的上傳檔案。

</div>

如果需要更複雜的條件式禁止邏輯，您可以使用 `Rule::prohibitedUnless` 方法。此方法接收布林值或 Closure。當傳入 Closure 時，Closure 應回傳 `true` 或 `false` 以指示正在驗證的欄位是否不應被禁止：

```php
use Illuminate\Support\Facades\Validator;
use Illuminate\Validation\Rule;

Validator::make($request->all(), [
    'role_id' => [Rule::prohibitedUnless($request->user()->is_admin)],
]);

Validator::make($request->all(), [
    'role_id' => [Rule::prohibitedUnless(fn () => $request->user()->is_admin)],
]);
```


<a name="rule-prohibits"></a>
#### prohibits:_anotherfield_,...

如果正在驗證的欄位並非不存在或非空，則 _anotherfield_ 中的所有欄位都必須不存在或是空的。如果符合以下任一條件，該欄位會被視為「空的」：

<div class="content-list" markdown="1">

- 數值為 `null`。
- 數值為空字串。
- 數值為空陣列或空的 `Countable` 物件。
- 數值為路徑為空的上傳檔案。

</div>


<a name="rule-regex"></a>
#### regex:_pattern_

正在驗證的欄位必須符合給定的正規表示式。

在內部，此規則使用 PHP 的 `preg_match` 函式。指定的模式應遵循 `preg_match` 所需的相同格式，因此也必須包含有效的定界符 (Delimiters)。例如：`'email' => ['regex:/^.+@.+$/i']`。


<a name="rule-required"></a>
#### required

正在驗證的欄位必須存在於輸入資料中且不能為空。如果符合以下任一條件，該欄位會被視為「空的」：

<div class="content-list" markdown="1">

- 數值為 `null`。
- 數值為空字串。
- 數值為空陣列或空的 `Countable` 物件。
- 數值為沒有路徑的上傳檔案。

</div>


<a name="rule-required-if"></a>
#### required_if:_anotherfield_,_value_,...

如果 _anotherfield_ 欄位等於任何一個 _value_，則正在驗證的欄位必須存在且不能為空。

如果您想要為 `required_if` 規則建構更複雜的條件，可以使用 `Rule::requiredIf` 方法。此方法接收布林值或 Closure。當傳入 Closure 時，Closure 應回傳 `true` 或 `false` 以指示正在驗證的欄位是否為必填：

```php
use Illuminate\Support\Facades\Validator;
use Illuminate\Validation\Rule;

Validator::make($request->all(), [
    'role_id' => [Rule::requiredIf($request->user()->is_admin)],
]);

Validator::make($request->all(), [
    'role_id' => [Rule::requiredIf(fn () => $request->user()->is_admin)],
]);
```


<a name="rule-required-if-accepted"></a>
#### required_if_accepted:_anotherfield_,...

如果 _anotherfield_ 欄位等於 `"yes"`、`"on"`、`1`、`"1"`、`true` 或 `"true"`，則正在驗證的欄位必須存在且不能為空。


<a name="rule-required-if-declined"></a>
#### required_if_declined:_anotherfield_,...

如果 _anotherfield_ 欄位等於 `"no"`、`"off"`、`0`、`"0"`、`false` 或 `"false"`，則正在驗證的欄位必須存在且不能為空。


<a name="rule-required-unless"></a>
#### required_unless:_anotherfield_,_value_,...

正在驗證的欄位必須存在且不能為空，除非 _anotherfield_ 欄位等於任何一個 _value_。這也意味著 _anotherfield_ 必須存在於請求資料中，除非 _value_ 為 `null`。如果 _value_ 為 `null`（`required_unless:name,null`），則正在驗證的欄位將是必填的，除非比較欄位為 `null` 或比較欄位在請求資料中缺失。

如果您想要為 `required_unless` 規則建構更複雜的條件，可以使用 `Rule::requiredUnless` 方法。此方法接收布林值或 Closure。當傳入 Closure 時，Closure 應回傳 `true` 或 `false` 以指示正在驗證的欄位是否不為必填：

```php
use Illuminate\Support\Facades\Validator;
use Illuminate\Validation\Rule;

Validator::make($request->all(), [
    'role_id' => [Rule::requiredUnless($request->user()->is_admin)],
]);

Validator::make($request->all(), [
    'role_id' => [Rule::requiredUnless(fn () => $request->user()->is_admin)],
]);
```


<a name="rule-required-with"></a>
#### required_with:_foo_,_bar_,...

_只有當_ 任何其他指定的欄位存在且不為空時，正在驗證的欄位才必須存在且不能為空。


<a name="rule-required-with-all"></a>
#### required_with_all:_foo_,_bar_,...

_只有當_ 所有其他指定的欄位皆存在且不為空時，正在驗證的欄位才必須存在且不能為空。


<a name="rule-required-without"></a>
#### required_without:_foo_,_bar_,...

_只有當_ 任何其他指定的欄位為空或不存在時，正在驗證的欄位才必須存在且不能為空。


<a name="rule-required-without-all"></a>
#### required_without_all:_foo_,_bar_,...

_只有當_ 所有其他指定的欄位皆為空或不存在時，正在驗證的欄位才必須存在且不能為空。


<a name="rule-required-array-keys"></a>
#### required_array_keys:_foo_,_bar_,...

正在驗證的欄位必須是一個陣列，且必須至少包含指定的鍵值 (Key)。


<a name="rule-same"></a>
#### same:_field_

給定的 _field_ 必須與正在驗證的欄位相符。


<a name="rule-size"></a>
#### size:_value_

正在驗證的欄位其大小必須符合給定的 _value_。對於字串資料，_value_ 對應字元數量。對於數值資料，_value_ 對應給定的整數值（該屬性同時必須具備 `numeric` 或 `integer` 規則）。對於陣列，_size_ 對應陣列的 `count`。對於檔案，_size_ 對應以 KB (Kilobytes) 為單位的檔案大小。讓我們看一些範例：

```php
// Validate that a string is exactly 12 characters long...
'title' => ['size:12'];

// Validate that a provided integer equals 10...
'seats' => ['integer', 'size:10'];

// Validate that an array has exactly 5 elements...
'tags' => ['array', 'size:5'];

// Validate that an uploaded file is exactly 512 kilobytes...
'image' => ['file', 'size:512'];
```


<a name="rule-starts-with"></a>
#### starts_with:_foo_,_bar_,...

正在驗證的欄位必須以給定的數值之一開頭。


<a name="rule-string"></a>
#### string

正在驗證的欄位必須是字串。如果您希望該欄位也可以為 `null`，則應該為該欄位分配 `nullable` 規則。

為了方便起見，也可以使用流暢的 `Rule::string()` 規則建構器來建立字串驗證規則：

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

字串規則建構器提供了常用字串限制條件的方法，包含 `alpha`、`alphaDash`、`alphaNumeric`、`ascii`、`between`、`doesntEndWith`、`doesntStartWith`、`endsWith`、`exactly`、`lowercase`、`max`、`min`、`startsWith` 以及 `uppercase`。由於規則建構器支援條件式，您還可以使用 `when` 和 `unless` 方法來條件式地套用限制。


<a name="rule-timezone"></a>
#### timezone

根據 `DateTimeZone::listIdentifiers` 方法，正在驗證的欄位必須是有效的時區識別碼。

[`DateTimeZone::listIdentifiers` 方法所接收的引數](https://www.php.net/manual/en/datetimezone.listidentifiers.php) 也可以提供給此驗證規則：

```php
'timezone' => ['required', 'timezone:all'];

'timezone' => ['required', 'timezone:Africa'];

'timezone' => ['required', 'timezone:per_country,US'];
```

<a name="rule-unique"></a>
#### unique:_table_,_column_

受驗證的欄位值不得存在於給定的資料庫資料表中。

**指定自訂資料表／欄位名稱：**

除了直接指定資料表名稱外，您也可以指定用來確定資料表名稱的 Eloquent 模型：

```php
'email' => ['unique:App\Models\User,email_address']
```

`column` 選項可以用於指定欄位對應的資料庫欄位。若未指定 `column` 選項，則會使用受驗證的欄位名稱。

```php
'email' => ['unique:users,email_address']
```

**指定自訂資料庫連線**

有時，您可能需要為驗證器進行的資料庫查詢設定自訂連線。若要做到這一點，您可以在資料表名稱前面加上連線名稱：

```php
'email' => ['unique:connection.users,email_address']
```

**強制 Unique 規則忽略指定的 ID：**

有時，您可能希望在進行唯一性驗證時忽略特定的 ID。例如，考慮一個包含使用者姓名、電子郵件地址和位置的「更新個人資料」畫面。您可能會想要驗證電子郵件地址是否唯一。但是，如果使用者只修改了姓名欄位而沒有修改電子郵件欄位，您不希望因為使用者已經是該電子郵件地址的擁有者而拋出驗證錯誤。

為了指示驗證器忽略使用者的 ID，我們將使用 `Rule` 類別流暢地定義規則。

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
> 您絕不應該將任何由使用者控制的請求輸入傳入 `ignore` 方法中。相對地，您應該只傳入由系統生成的唯一 ID，例如來自 Eloquent 模型執行個體的自動遞增 ID 或 UUID。否則，您的應用程式將容易受到 SQL 注入攻擊。

除了將模型主鍵的值傳給 `ignore` 方法外，您也可以傳入整個模型執行個體。Laravel 會自動從模型中擷取主鍵：

```php
Rule::unique('users')->ignore($user)
```

如果您的資料表使用的主鍵欄位名稱不是 `id`，您可以在呼叫 `ignore` 方法時指定欄位名稱：

```php
Rule::unique('users')->ignore($user->id, 'user_id')
```

預設情況下，`unique` 規則會檢查與受驗證屬性名稱相符的欄位唯一性。然而，您可以傳入不同的欄位名稱作為 `unique` 方法的第二個引數：

```php
Rule::unique('users', 'email_address')->ignore($user->id)
```

**新增額外的 Where 子句：**

您可以透過使用 `where` 方法自訂查詢來指定額外的查詢條件。例如，讓我們新增一個查詢條件，將查詢限制為僅搜尋 `account_id` 欄位值為 `1` 的紀錄：

```php
'email' => Rule::unique('users')->where(fn (Builder $query) => $query->where('account_id', 1))
```

**在唯一性檢查中忽略軟刪除紀錄：**

預設情況下，唯一性規則在確定唯一性時會包含軟刪除紀錄。若要在唯一性檢查中排除軟刪除紀錄，您可以呼叫 `withoutTrashed` 方法：

```php
Rule::unique('users')->withoutTrashed();
```

如果您的模型在軟刪除紀錄上使用的欄位名稱不是 `deleted_at`，您可以在呼叫 `withoutTrashed` 方法時提供該欄位名稱：

```php
Rule::unique('users')->withoutTrashed('was_deleted_at');
```


<a name="rule-uppercase"></a>
#### uppercase

受驗證的欄位必須為大寫。


<a name="rule-url"></a>
#### url

受驗證的欄位必須是有效的 URL。

如果您想指定哪些 URL 通訊協定應被視為有效，您可以將通訊協定作為驗證規則的參數傳入：

```php
'url' => ['url:http,https'],

'game' => ['url:minecraft,steam'],
```


<a name="rule-ulid"></a>
#### ulid

受驗證的欄位必須是有效的 [Universally Unique Lexicographically Sortable Identifier](https://github.com/ulid/spec) (ULID)。


<a name="rule-uuid"></a>
#### uuid

受驗證的欄位必須是有效的 RFC 9562（版本 1、3、4、5、6、7 或 8）通用唯一識別碼 (UUID)。

您也可以驗證給定的 UUID 是否符合特定的版本規範：

```php
'uuid' => ['uuid:4']
```

<a name="conditionally-adding-rules"></a>
## 條件式新增規則


<a name="skipping-validation-when-fields-have-certain-values"></a>
#### 當欄位具有特定數值時跳過驗證

有時您可能希望在另一個欄位具有特定數值時，不驗證指定的欄位。您可以使用 `exclude_if` 驗證規則來達成此目的。在此範例中，若 `has_appointment` 欄位的值為 `false`，則不會驗證 `appointment_date` 和 `doctor_name` 欄位：

```php
use Illuminate\Support\Facades\Validator;

$validator = Validator::make($data, [
    'has_appointment' => ['required', 'boolean'],
    'appointment_date' => ['exclude_if:has_appointment,false', 'required', 'date'],
    'doctor_name' => ['exclude_if:has_appointment,false', 'required', 'string'],
]);
```

或者，您可以選擇使用 `exclude_unless` 規則，除非另一個欄位具有指定的值，否則不驗證該欄位：

```php
$validator = Validator::make($data, [
    'has_appointment' => ['required', 'boolean'],
    'appointment_date' => ['exclude_unless:has_appointment,true', 'required', 'date'],
    'doctor_name' => ['exclude_unless:has_appointment,true', 'required', 'string'],
]);
```


<a name="validating-when-present"></a>
#### 當存在時才進行驗證

在某些情況下，您可能希望**僅**在被驗證的資料中存在該欄位時，才對其執行驗證檢查。要快速實現此目的，請將 `sometimes` 規則新增至您的規則清單中：

```php
$validator = Validator::make($data, [
    'email' => ['sometimes', 'required', 'email'],
]);
```

在上述範例中，僅當 `email` 欄位存在於 `$data` 陣列中時，才會為其進行驗證。

> [!NOTE]
> 如果您嘗試驗證一個應該總是存在但可能為空的欄位，請參考[關於選填欄位的說明](#a-note-on-optional-fields)。


<a name="complex-conditional-validation"></a>
#### 複雜條件式驗證

有時您可能希望根據更複雜的條件邏輯來新增驗證規則。例如，您可能希望僅在另一個欄位的值大於 100 時才需要某個特定欄位；或者，您可能需要僅在另一個欄位存在時，兩個欄位才具有指定的值。新增這些驗證規則並不困難。首先，使用永遠不變的_靜態規則_建立一個 `Validator` 實例：

```php
use Illuminate\Support\Facades\Validator;

$validator = Validator::make($request->all(), [
    'email' => ['required', 'email'],
    'games' => ['required', 'integer', 'min:0'],
]);
```

假設我們的 Web 應用程式是用於遊戲收藏家。如果遊戲收藏家在我們的應用程式註冊且擁有超過 100 款遊戲，我們希望他們解釋為什麼擁有這麼多遊戲。例如，也許他們經營一家遊戲轉售店，或者他們只是喜歡收集遊戲。若要條件式新增此需求，我們可以在 `Validator` 實例上使用 `sometimes` 方法。

```php
use Illuminate\Support\Fluent;

$validator->sometimes('reason', ['required', 'max:500'], function (Fluent $input) {
    return $input->games >= 100;
});
```

傳遞給 `sometimes` 方法的第一個引數是我們有條件驗證的欄位名稱。第二個引數是我們想要新增的規則清單。如果作為第三個引數傳遞的 Closure 回傳 `true`，則會新增這些規則。這個方法讓建立複雜的條件式驗證變得非常輕鬆。您甚至可以一次為多個欄位新增條件式驗證：

```php
$validator->sometimes(['reason', 'cost'], 'required', function (Fluent $input) {
    return $input->games >= 100;
});
```

> [!NOTE]
> 傳遞給 Closure 的 `$input` 參數將是 `Illuminate\Support\Fluent` 的實例，可用於存取您正在進行驗證的輸入資料和檔案。


<a name="complex-conditional-array-validation"></a>
#### 複雜條件式陣列驗證

有時您可能想根據同一巢狀陣列中的另一個欄位來驗證某個欄位，但您不知道該項目的索引。在這些情況下，您可以讓您的 Closure 接收第二個引數，該引數將是目前被驗證的陣列中的單一項目：

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

如同傳遞給 Closure 的 `$input` 參數，當屬性資料為陣列時，`$item` 參數是 `Illuminate\Support\Fluent` 的實例；否則它是一個字串。

<a name="validating-arrays"></a>
## 驗證陣列

如同 [array 驗證規則文件](#rule-array) 中所述，`array` 規則接受允許的陣列鍵值列表。如果陣列中存在任何額外的鍵值，驗證將會失敗：

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
    'user' => ['array:name,username'],
]);
```

一般來說，您應該總是指定允許存在於陣列中的陣列鍵值。否則，驗證器的 `validate` 與 `validated` 方法將回傳所有已驗證的資料（包含該陣列及其所有鍵值），即使這些鍵值並未經過其他巢狀陣列驗證規則的驗證。


<a name="validating-nested-array-input"></a>
### 驗證巢狀陣列輸入

驗證巢狀陣列結構的表單輸入欄位並不困難。您可以使用「點號表示法 (dot notation)」來驗證陣列中的屬性。例如，若傳入的 HTTP 請求包含 `photos[profile]` 欄位，您可以像這樣進行驗證：

```php
use Illuminate\Support\Facades\Validator;

$validator = Validator::make($request->all(), [
    'photos.profile' => ['required', 'image'],
]);
```

您也可以驗證陣列中的每個元素。例如，若要驗證給定陣列輸入欄位中的每個 Email 是否唯一，可以這樣做：

```php
$validator = Validator::make($request->all(), [
    'users.*.email' => ['email', 'unique:users'],
    'users.*.first_name' => ['required_with:users.*.last_name'],
]);
```

同樣地，當您在[語言檔中指定自訂驗證訊息](#custom-messages-for-specific-attributes)時，可以使用 `*` 字元，這讓您能輕鬆地為陣列欄位統一使用單一驗證訊息：

```php
'custom' => [
    'users.*.email' => [
        'unique' => 'Each user must have a unique email address',
    ]
],
```


<a name="accessing-nested-array-data"></a>
#### 存取巢狀陣列資料

有時您可能需要在為屬性設定驗證規則時存取特定巢狀陣列元素的值。您可以透過 `Rule::forEach` 方法達成此目的。`forEach` 方法接收一個 Closure，該 Closure 會在每次疊代正受驗證的陣列屬性時被呼叫，並接收該屬性的值以及明確且完全展開的屬性名稱。該 Closure 應回傳要分配給該陣列元素的規則陣列：

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
### 錯誤訊息索引與位置

驗證陣列時，您可能希望在應用程式顯示的錯誤訊息中引用驗證失敗項目的索引或位置。為此，您可以在[自訂驗證訊息](#manual-customizing-the-error-messages)中使用 `:index`（從 `0` 開始）、`:position`（從 `1` 開始）或 `:ordinal-position`（從 `1st` 開始）等預留位置：

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
    'photos.*.description' => ['required'],
], [
    'photos.*.description.required' => 'Please describe photo #:position.',
]);
```

在上面的範例中，驗證將會失敗，且使用者將會收到以下錯誤訊息：_"Please describe photo #2."_

如有需要，您可以透過 `second-index`、`second-position`、`third-index`、`third-position` 等方式引用更深層巢狀結構的索引與位置。

```php
'photos.*.attributes.*.string' => 'Invalid attribute for photo #:second-position.',
```


<a name="validating-files"></a>
## 驗證檔案

Laravel 提供多種可用於驗證上傳檔案的驗證規則，例如 `mimes`、`image`、`min` 與 `max`。雖然您可以在驗證檔案時單獨指定這些規則，但 Laravel 也提供了一個流暢的檔案驗證規則建構器，您可能會覺得非常方便：

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

即使您在呼叫 `types` 方法時只需要指定副檔名，該方法實際上會透過讀取檔案內容並猜測其 MIME 類型來驗證檔案的 MIME 類型。完整的 MIME 類型及其對應副檔名列表可在以下位置找到：

[https://svn.apache.org/repos/asf/httpd/httpd/trunk/docs/conf/mime.types](https://svn.apache.org/repos/asf/httpd/httpd/trunk/docs/conf/mime.types)


<a name="validating-files-file-sizes"></a>
#### 驗證檔案大小

為求方便，最小與最大檔案大小可以指定為帶有表示檔案大小單位字尾的字串。支援 `kb`、`mb`、`gb` 及 `tb` 等字尾：

```php
File::types(['mp3', 'wav'])
    ->min('1kb')
    ->max('10mb');
```


<a name="validating-files-image-files"></a>
#### 驗證圖檔

若您的應用程式接受使用者上傳的圖檔，您可以使用 `File` 規則的 `image` 建構子方法來確保受驗證的檔案是圖片（jpg、jpeg、png、bmp、gif 或 webp）。

此外，還可以使用 `dimensions` 規則來限制圖片的尺寸：

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
> 有關驗證圖片尺寸的更多資訊，請參閱 [dimension 規則文件](#rule-dimensions)。

> [!WARNING]
> 預設情況下，由於可能存在 XSS 漏洞，`image` 規則不允許 SVG 檔案。若您需要允許 SVG 檔案，可以傳遞 `allowSvg: true` 給 `image` 規則：`File::image(allowSvg: true)`。


<a name="validating-files-image-dimensions"></a>
#### 驗證圖片尺寸

您也可以驗證圖片的尺寸。例如，若要驗證上傳的圖片寬度至少為 1000 像素且高度至少為 500 像素，您可以使用 `dimensions` 規則：

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
> 有關驗證圖片尺寸的更多資訊，請參閱 [dimension 規則文件](#rule-dimensions)。

<a name="validating-passwords"></a>
## 驗證密碼

為確保密碼具備足夠的複雜度，您可以使用 Laravel 的 `Password` 規則物件：

```php
use Illuminate\Support\Facades\Validator;
use Illuminate\Validation\Rules\Password;

$validator = Validator::make($request->all(), [
    'password' => ['required', 'confirmed', Password::min(8)],
]);
```

`Password` 規則物件讓您能輕鬆自訂應用程式的密碼複雜度需求，例如指定密碼必須包含至少一個字母、數字、符號，或是大小寫混合的字元：

```php
// Require at least 8 characters...
Password::min(8)

// Require at most 256 characters...
Password::min(16)->max(256)

// Require at least one letter...
Password::min(8)->letters()

// Require at least one uppercase and one lowercase letter...
Password::min(8)->mixedCase()

// Require at least one number...
Password::min(8)->numbers()

// Require at least one symbol...
Password::min(8)->symbols()
```

此外，您可以使用 `uncompromised` 方法確保密碼未在公開的密碼資料洩漏事件中遭妥協：

```php
Password::min(8)->uncompromised()
```

在內部，`Password` 規則物件採用 [k-Anonymity](https://en.wikipedia.org/wiki/K-anonymity) 模型，透過 [haveibeenpwned.com](https://haveibeenpwned.com) 服務來判斷密碼是否已經外洩，同時不會犧牲使用者的隱私或安全性。

預設情況下，如果密碼在資料洩漏中出現過至少一次，就會被視為遭妥協。您可以透過 `uncompromised` 方法的第一個引數來自訂此門檻：

```php
// Ensure the password appears less than 3 times in the same data leak...
Password::min(8)->uncompromised(3);
```

當然，您可以將上述範例中的所有方法串接在一起：

```php
Password::min(8)
    ->max(256)
    ->letters()
    ->mixedCase()
    ->numbers()
    ->symbols()
    ->uncompromised()
```

您可以使用 `toPasswordRulesString` 方法將 `Password` 規則物件轉換為適合用於 HTML `passwordrules` 屬性的字串：

```blade
<input
    type="password"
    name="password"
    autocomplete="new-password"
    passwordrules="{{ Password::defaults()->toPasswordRulesString() }}"
/>
```

<a name="defining-default-password-rules"></a>
#### 定義預設密碼規則

您可能會發現，在應用程式的單一位置指定密碼的預設驗證規則會非常便利。使用接收 Closure 的 `Password::defaults` 方法即可輕鬆達成此目的。傳給 `defaults` 方法的 Closure 應該回傳 Password 規則的預設設定。通常，應該在應用程式的其中一個服務提供者(Service Providers)的 `boot` 方法內呼叫 `defaults` 規則：

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

接著，當您想要將預設規則套用到正在進行驗證的特定密碼時，您可以呼叫不帶引數的 `defaults` 方法：

```php
'password' => ['required', Password::defaults()],
```

有時候，您可能會想在預設的密碼驗證規則中附加額外的驗證規則。您可以使用 `rules` 方法來達成這一點：

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

Laravel 提供許多有用的驗證規則；然而，你可能希望指定自訂的規則。註冊自訂驗證規則的一種方法是使用規則物件。要產生新的規則物件，你可以使用 `make:rule` Artisan 命令。讓我們使用此命令來產生一個驗證字串是否為大寫的規則。Laravel 會將新規則放在 `app/Rules` 目錄中。如果該目錄不存在，當你執行 Artisan 命令建立規則時，Laravel 會自動建立它：

```shell
php artisan make:rule Uppercase
```

建立規則後，我們就可以定義其行為。規則物件包含一個 `validate` 方法。此方法接收屬性名稱、屬性值以及一個失敗時應該被呼叫的閉包 (Callback)，並帶入驗證錯誤訊息：

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

定義好規則後，你可以透過將規則物件的實例與其他驗證規則一起傳入，將其附加到驗證器：

```php
use App\Rules\Uppercase;

$request->validate([
    'name' => ['required', 'string', new Uppercase],
]);
```

#### 翻譯驗證訊息

除了向 `$fail` 閉包提供字面錯誤訊息外，你也可以提供一個[翻譯字串鍵值](/docs/{{version}}/localization)並指示 Laravel 翻譯該錯誤訊息：

```php
if (strtoupper($value) !== $value) {
    $fail('validation.uppercase')->translate();
}
```

如果需要，你可以將占位符替換陣列與偏好語言分別作為第一與第二個引數傳給 `translate` 方法：

```php
$fail('validation.location')->translate([
    'value' => $this->value,
], 'fr');
```

#### 存取額外資料

如果你的自訂驗證規則類別需要存取正在進行驗證的所有其他資料，你的規則類別可以實作 `Illuminate\Contracts\Validation\DataAwareRule` 介面。這個介面要求你的類別定義一個 `setData` 方法。在驗證進行前，Laravel 會自動呼叫此方法並帶入所有正在驗證的資料：

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

或者，如果你的驗證規則需要存取執行驗證的驗證器實例，你可以實作 `ValidatorAwareRule` 介面：

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
### 使用 Closure

如果你在整個應用程式中只需要使用一次自訂規則的功能，你可以使用 Closure 來代替規則物件。該 Closure 接收屬性的名稱、屬性的值，以及一個驗證失敗時應呼叫的 `$fail` 回調：

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
### 隱式規則

預設情況下，當正在驗證的屬性不存在或包含空字串時，一般的驗證規則（包含自訂規則）都不會執行。例如，[unique](#rule-unique) 規則不會針對空字串執行：

```php
use Illuminate\Support\Facades\Validator;

$rules = ['name' => ['unique:users,name']];

$input = ['name' => ''];

Validator::make($input, $rules)->passes(); // true
```

若要讓自訂規則在屬性為空時也能執行，該規則必須隱含該屬性為必填。若要快速產生新的隱式規則物件，你可以使用帶有 `--implicit` 選項的 `make:rule` Artisan 命令：

```shell
php artisan make:rule Uppercase --implicit
```

> [!WARNING]
> 「隱式」規則僅僅是 _隱含_ 該屬性為必填。至於它是否真的會使缺失或為空的屬性不通過驗證，完全取決於你的實作。