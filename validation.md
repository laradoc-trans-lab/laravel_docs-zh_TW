# 驗證 (Validation)

- [簡介](#introduction)
- [驗證快速入門](#validation-quickstart)
    - [定義路由](#quick-defining-the-routes)
    - [建立控制器](#quick-creating-the-controller)
    - [撰寫驗證邏輯](#quick-writing-the-validation-logic)
    - [顯示驗證錯誤訊息](#quick-displaying-the-validation-errors)
    - [重新填入表單](#repopulating-forms)
    - [關於選填欄位的注意事項](#a-note-on-optional-fields)
    - [驗證錯誤的回應格式](#validation-error-response-format)
- [表單請求(Form request)驗證](#form-request-validation)
    - [建立表單請求](#creating-form-requests)
    - [授權表單請求](#authorizing-form-requests)
    - [自訂錯誤訊息](#customizing-the-error-messages)
    - [在驗證前預處理輸入資料](#preparing-input-for-validation)
- [手動建立驗證器](#manually-creating-validators)
    - [自動轉址](#automatic-redirection)
    - [命名錯誤包 (Named Error Bags)](#named-error-bags)
    - [自訂錯誤訊息](#manual-customizing-the-error-messages)
    - [執行額外驗證](#performing-additional-validation)
- [處理通過驗證的輸入資料](#working-with-validated-input)
- [處理錯誤訊息](#working-with-error-messages)
    - [在語言檔中指定自訂訊息](#specifying-custom-messages-in-language-files)
    - [在語言檔中指定屬性](#specifying-attribute-in-language-files)
    - [在語言檔中指定數值](#specifying-values-in-language-files)
- [可用的驗證規則](#available-validation-rules)
- [條件式新增規則](#conditionally-adding-rules)
- [驗證陣列](#validating-arrays)
    - [驗證巢狀陣列輸入](#validating-nested-array-input)
    - [錯誤訊息的索引與位置](#error-message-indexes-and-positions)
- [驗證檔案](#validating-files)
- [驗證密碼](#validating-passwords)
- [自訂驗證規則](#custom-validation-rules)
    - [使用規則物件](#using-rule-objects)
    - [使用 Closure](#using-closures)
    - [隱式規則](#implicit-rules)

<a name="introduction"></a>
## 簡介

Laravel 提供數種不同的方式來驗證應用程式接收到的資料。最常見的做法是使用所有傳入 HTTP 請求皆可用的 `validate` 方法。不過，我們也會探討其他驗證方式。

Laravel 內建了豐富且便利的驗證規則可套用到資料上，甚至能驗證數值在指定的資料庫資料表中是否為唯一值。我們將詳細介紹這些驗證規則，讓你能熟悉 Laravel 的所有驗證功能。

<a name="validation-quickstart"></a>
## 驗證快速入門

為了瞭解 Laravel 強大的驗證功能，讓我們來看一個驗證表單並將錯誤訊息顯示給使用者的完整範例。閱讀這份高階概覽後，您將能基本掌握如何使用 Laravel 驗證傳入的請求資料：


<a name="quick-defining-the-routes"></a>
### 定義路由

首先，假設我們在 `routes/web.php` 檔案中定義了以下路由：

```php
use App\Http\Controllers\PostController;

Route::get('/post/create', [PostController::class, 'create']);
Route::post('/post', [PostController::class, 'store']);
```

`GET` 路由會顯示一個表單供使用者建立新的部落格文章，而 `POST` 路由則會將新的部落格文章儲存到資料庫中。


<a name="quick-creating-the-controller"></a>
### 建立控制器

接下來，讓我們看一下處理這些傳入請求的簡單控制器。我們暫時將 `store` 方法留空：

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

現在我們準備在 `store` 方法中填入驗證新部落格文章的邏輯。為此，我們將使用 `Illuminate\Http\Request` 物件提供的 `validate` 方法。若通過驗證規則，您的程式碼將繼續正常執行；然而，若驗證失敗，系統會拋出 `Illuminate\Validation\ValidationException` 異常，並自動將相應的錯誤回應發回給使用者。

如果在傳統的 HTTP 請求過程中驗證失敗，系統會產生一個轉址至前一個 URL 的轉址回應。如果傳入的請求是 XHR 請求，則會傳回[包含驗證錯誤訊息的 JSON 回應](#validation-error-response-format)。

為了更能理解 `validate` 方法，讓我們回到 `store` 方法：

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

如您所見，驗證規則被傳入 `validate` 方法中。別擔心——所有可用的驗證規則都有[文件紀錄](#available-validation-rules)。再次說明，如果驗證失敗，系統會自動產生相應的回應。如果驗證通過，我們的控制器將繼續正常執行。

此外，您可以使用 `validateWithBag` 方法來驗證請求，並將任何錯誤訊息儲存在[命名錯誤包 (named error bag)](#named-error-bags)中：

```php
$validated = $request->validateWithBag('post', [
    'title' => ['required', 'unique:posts', 'max:255'],
    'body' => ['required'],
]);
```


<a name="stopping-on-first-validation-failure"></a>
#### 首次驗證失敗時停止

有時您可能希望在某個屬性發生首次驗證失敗後，停止對該屬性執行後續的驗證規則。為此，可以將 `bail` 規則指派給該屬性：

```php
$request->validate([
    'title' => ['bail', 'required', 'unique:posts', 'max:255'],
    'body' => ['required'],
]);
```

在這個範例中，如果 `title` 屬性上的 `unique` 規則驗證失敗，就不會檢查 `max` 規則。規則將按照指派的順序進行驗證。


<a name="a-note-on-nested-attributes"></a>
#### 關於巢狀屬性的注意事項

如果傳入的 HTTP 請求包含「巢狀」欄位資料，您可以使用「點 (dot)」語法在驗證規則中指定這些欄位：

```php
$request->validate([
    'title' => ['required', 'unique:posts', 'max:255'],
    'author.name' => ['required'],
    'author.description' => ['required'],
]);
```

另一方面，如果您的欄位名稱本身就包含句點文字，您可以透過使用反斜線轉義該句點，明確防止其被解析為「點」語法：

```php
$request->validate([
    'title' => ['required', 'unique:posts', 'max:255'],
    'v1\.0' => ['required'],
]);
```

<a name="quick-displaying-the-validation-errors"></a>
### 顯示驗證錯誤訊息

那麼，如果傳入的請求欄位沒有通過指定的驗證規則呢？如前所述，Laravel 會自動將使用者轉址回他們之前的頁面。此外，所有的驗證錯誤與 [請求輸入資料](/docs/{{version}}/requests#retrieving-old-input) 都會自動 [快閃至 Session](/docs/{{version}}/session#flash-data)。

由 `web` 中介層群組提供的 `Illuminate\View\Middleware\ShareErrorsFromSession` 中介層會將 `$errors` 變數共享給您應用程式中的所有視圖。當套用此中介層時，`$errors` 變數將永遠在您的視圖中可用，讓您可以方便地假設 `$errors` 變數總是已被定義且能安全地使用。`$errors` 變數會是 `Illuminate\Support\MessageBag` 的實例。關於使用此物件的更多資訊，請[參考其說明文件](#working-with-error-messages)。

因此，在我們的範例中，當驗證失敗時，使用者會被轉址回我們控制器的 `create` 方法，讓我們能在視圖中顯示錯誤訊息：

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

Laravel 內建的驗證規則各自都有錯誤訊息，位於您應用程式的 `lang/en/validation.php` 檔案中。若您的應用程式沒有 `lang` 目錄，您可以使用 `lang:publish` Artisan 指令讓 Laravel 建立該目錄。

在 `lang/en/validation.php` 檔案中，您會找到每個驗證規則對應的翻譯項目。您可以根據應用程式的需求自由修改這些訊息。

此外，您可以將此檔案複製到其他語言目錄，以便將訊息翻譯成您應用程式所使用的語言。若要瞭解更多關於 Laravel 在地化的資訊，請參考完整的[在地化文件](/docs/{{version}}/localization)。

> [!WARNING]
> 預設情況下，Laravel 應用程式骨架並不包含 `lang` 目錄。若您想要自訂 Laravel 的語言檔，可以透過 `lang:publish` Artisan 指令來發布它們。


<a name="quick-xhr-requests-and-validation"></a>
#### XHR 請求與驗證

在這個範例中，我們使用傳統表單將資料發送到應用程式。然而，許多應用程式接收來自 JavaScript 前端的 XHR 請求。當在 XHR 請求期間使用 `validate` 方法時，Laravel 不會產生轉址回應。相反地，Laravel 會產生一個[包含所有驗證錯誤的 JSON 回應](#validation-error-response-format)。此 JSON 回應將會伴隨 422 HTTP 狀態碼傳送。


<a name="the-at-error-directive"></a>
#### `@error` 指令

您可以使用 `@error` [Blade](/docs/{{version}}/blade) 指令來快速判斷指定屬性是否存在驗證錯誤訊息。在 `@error` 指令內，您可以印出 `$message` 變數來顯示錯誤訊息：

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

如果您使用[命名錯誤包 (Named Error Bags)](#named-error-bags)，可以將錯誤包的名稱作為第二個引數傳給 `@error` 指令：

```blade
<input ... class="@error('title', 'post') is-invalid @enderror">
```


<a name="repopulating-forms"></a>
### 重新填入表單

當 Laravel 因驗證錯誤而產生轉址回應時，框架會自動將[請求的所有輸入資料快閃至 Session](/docs/{{version}}/session#flash-data)。這樣做能讓您在下一次請求時方便地存取這些輸入資料，並重新填入使用者嘗試提交的表單。

若要取得前一次請求快閃的輸入資料，請在 `Illuminate\Http\Request` 實例上呼叫 `old` 方法。`old` 方法會從 [Session](/docs/{{version}}/session) 中拉取先前快閃的輸入資料：

```php
$title = $request->old('title');
```

Laravel 還提供了全域的 `old` 輔助函式。如果您要在 [Blade 模板](/docs/{{version}}/blade)內顯示舊的輸入資料，使用 `old` 輔助函式來重新填入表單會更加方便。若指定欄位不存在舊的輸入資料，則會傳回 `null`：

```blade
<input type="text" name="title" value="{{ old('title') }}">
```


<a name="a-note-on-optional-fields"></a>
### 關於選填欄位的注意事項

預設情況下，Laravel 在您應用程式的全域中介層堆疊中包含了 `TrimStrings` 和 `ConvertEmptyStringsToNull` 中介層。因此，如果您不希望驗證器將 `null` 值視為無效，通常需要將「選填」的請求欄位標記為 `nullable`。例如：

```php
$request->validate([
    'title' => ['required', 'unique:posts', 'max:255'],
    'body' => ['required'],
    'publish_at' => ['nullable', 'date'],
]);
```

在此範例中，我們指定 `publish_at` 欄位可以是 `null` 或有效的日期格式。若未在規則定義中加入 `nullable` 修飾詞，驗證器將會把 `null` 視為無效的日期。


<a name="validation-error-response-format"></a>
### 驗證錯誤的回應格式

當您的應用程式拋出 `Illuminate\Validation\ValidationException` 異常，且傳入的 HTTP 請求期望一個 JSON 回應時，Laravel 會自動為您格式化錯誤訊息，並傳回 `422 Unprocessable Entity` HTTP 回應。

您可以在下方檢視驗證錯誤 JSON 回應格式的範例。請注意，巢狀錯誤鍵會被展平為「點」號表示法格式：

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
## 表單請求(Form request)驗證

<a name="creating-form-requests"></a>
### 建立表單請求

對於更複雜的驗證情境，你可能希望建立「表單請求(Form request)」。表單請求是自訂的請求類別，其中封裝了自身的驗證與授權邏輯。若要建立表單請求類別，可以使用 `make:request` Artisan CLI 命令：

```shell
php artisan make:request StorePostRequest
```

產生的表單請求類別將會置於 `app/Http/Requests` 目錄中。若此目錄不存在，將會在執行 `make:request` 命令時自動建立。Laravel 產生的每個表單請求都包含兩個方法：`authorize` 與 `rules`。

正如你所猜想的，`authorize` 方法負責判斷當前已通過認證的使用者是否能執行該請求所代表的操作，而 `rules` 方法則傳回應套用至請求資料的驗證規則：

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
> 你可以在 `rules` 方法的型別提示中注入所需的任何依賴項。它們將透過 Laravel [服務容器(Service Container)](/docs/{{version}}/container) 自動解析。

那麼，驗證規則是如何進行評估的呢？你只需要在控制器的動作方法中，將傳入的請求型別提示為該表單請求即可。傳入的表單請求會在控制器方法被呼叫之前完成驗證，這意味著你不需要在控制器中寫滿任何驗證邏輯：

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

如果驗證失敗，系統會產生一個轉址回應，將使用者引導回前一個頁面。這些錯誤訊息也會被暫存至 Session，以便在畫面中顯示。若該請求為 XHR 請求，則會傳回 HTTP 狀態碼為 422 的回應給使用者，其中包含[驗證錯誤的 JSON 格式內容](#validation-error-response-format)。

> [!NOTE]
> 需要為你基於 Inertia 的 Laravel 前端新增即時的表單請求驗證嗎？請參考 [Laravel Precognition](/docs/{{version}}/precognition)。

<a name="performing-additional-validation-on-form-requests"></a>
#### 執行額外驗證

有時你需要在初始驗證完成後執行額外的驗證。你可以使用表單請求的 `after` 方法來達成此目的。

`after` 方法應傳回一個包含 Callable 或 Closure 的陣列，這些內容將在驗證完成後被呼叫。傳入的 Callable 將會接收到一個 `Illuminate\Validation\Validator` 實例，讓你可以在有需要時觸發額外的錯誤訊息：

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

如前所述，`after` 方法傳回的陣列也可以包含可呼叫的類別 (Invokable Classes)。這些類別的 `__invoke` 方法將會接收到一個 `Illuminate\Validation\Validator` 實例：

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
#### 遇到第一個驗證失敗時停止

透過在請求類別中新增 `StopOnFirstFailure` 屬性，你可以告知驗證器一旦發生單一驗證失敗，就應該停止驗證所有屬性：

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
#### 未知欄位驗證失敗

透過在請求類別中新增 `FailOnUnknownFields` 屬性，你可以指示 Laravel 拒絕任何未在該請求驗證規則中定義的傳入欄位：

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

你也可以在 `AppServiceProvider` 中為所有表單請求全域啟用此行為：

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

如果需要，你可以透過傳入 `false` 給該屬性來停用特定請求的此行為：

```php
#[FailOnUnknownFields(false)]
class PublicWebhookRequest extends FormRequest
{
    // ...
}
```

拒絕未知欄位可以防止未預期的輸入鍵流入應用程式深處，從而為批量賦值 (Mass-assignment) 類型的安全性問題提供額外的保護。然而，你仍應設定 Model 的 `$fillable` / `$guarded` 屬性，且僅持久化受信任、通過驗證的輸入資料。

<a name="customizing-the-redirect-location"></a>
#### 自訂轉址位置

當表單請求驗證失敗時，系統會產生一個轉址回應，將使用者引導回先前的位置。然而，你可以自由地自訂此行為。若要自訂，可在表單請求上使用 `RedirectTo` 屬性：

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

或者，如果你想將使用者轉址到具名路由，則可以改用 `RedirectToRoute` 屬性：

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

當表單請求驗證失敗時，錯誤訊息會被暫存至 `default` 錯誤包中。若你需要將錯誤訊息儲存於不同的[命名錯誤包 (Named Error Bags)](#named-error-bags) 中，可以在表單請求上使用 `ErrorBag` 屬性：

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

表單請求類別也包含了一個 `authorize` 方法。在此方法中，你可以確認已通過認證的使用者是否真的有權限更新給定的資源。例如，你可以判斷使用者是否確實擁有他們試圖更新的部落格留言。你最有可能在這個方法中呼叫你的[授權 Gates 與 Policies](/docs/{{version}}/authorization)：

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

由於所有表單請求都繼承了 Laravel 的基礎請求類別，因此我們可以使用 `user` 方法來取得當前通過認證的使用者。此外，請注意上述範例中對 `route` 方法的呼叫。該方法讓你能夠存取被呼叫路由上定義的 URI 參數，例如以下範例中的 `{comment}` 參數：

```php
Route::post('/comment/{comment}');
```

因此，如果你的應用程式利用了[路由模型綁定](/docs/{{version}}/routing#route-model-binding)，透過將解析出的模型作為請求的屬性來存取，可以讓你的程式碼變得更加精簡：

```php
return $this->user()->can('update', $this->comment);
```

如果 `authorize` 方法回傳 `false`，系統會自動回傳 HTTP 403 狀態碼的回應，且你的控制器方法將不會執行。

如果你打算在應用程式的其他地方處理請求的授權邏輯，你可以完全移除 `authorize` 方法，或是直接回傳 `true`：

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
> 你可以在 `authorize` 方法的簽名中型態提示 (Type-hint) 任何所需的依賴項目。它們將會透過 Laravel [服務容器](/docs/{{version}}/container)自動解析。


<a name="customizing-the-error-messages"></a>
### 自訂錯誤訊息

你可以透過覆寫 `messages` 方法來自訂表單請求所使用的錯誤訊息。此方法應回傳一個包含屬性 / 規則對及其對應錯誤訊息的陣列：

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

許多 Laravel 內建的驗證規則錯誤訊息都包含 `:attribute` 佔位符。如果你希望將驗證訊息中的 `:attribute` 佔位符替換為自訂的屬性名稱，你可以透過覆寫 `attributes` 方法來指定自訂名稱。此方法應回傳一個包含屬性 / 名稱對的陣列：

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
### 在驗證前預處理輸入資料

如果你需要在套用驗證規則之前預處理或清理來自請求的任何資料，可以使用 `prepareForValidation` 方法：

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

同樣地，如果你需要在驗證完成後正規化任何請求資料，可以使用 `passedValidation` 方法：

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

若您不想使用請求物件上的 `validate` 方法，您可以使用 `Validator` [Facade](/docs/{{version}}/facades) 手動建立驗證器實例。Facade 上的 `make` 方法會產生一個新的驗證器實例：

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

傳入 `make` 方法的第一個引數為要被驗證的資料。第二個引數則是要套用到該資料的驗證規則陣列。

在判斷請求驗證是否失敗後，您可以使用 `withErrors` 方法將錯誤訊息快閃 (Flash) 至 Session 中。使用此方法時，在轉址後，`$errors` 變數會自動與您的 View 共用，讓您能輕鬆地將錯誤訊息顯示給使用者。`withErrors` 方法接受驗證器、`MessageBag` 實例或 PHP `array`。


#### 首次驗證失敗即停止

`stopOnFirstFailure` 方法會告知驗證器，一旦發生單一驗證失敗時，就應該停止驗證所有屬性：

```php
if ($validator->stopOnFirstFailure()->fails()) {
    // ...
}
```


<a name="automatic-redirection"></a>
### 自動轉址

若您想要手動建立驗證器實例，但仍想利用 HTTP 請求的 `validate` 方法所提供的自動轉址功能，您可以在現有的驗證器實例上呼叫 `validate` 方法。若驗證失敗，使用者會自動被轉址；或者在 XHR 請求的情況下，會[傳回 JSON 回應](#validation-error-response-format)：

```php
Validator::make($request->all(), [
    'title' => ['required', 'unique:posts', 'max:255'],
    'body' => ['required'],
])->validate();
```

若驗證失敗，您可以使用 `validateWithBag` 方法將錯誤訊息儲存在[命名錯誤包 (Named Error Bags)](#named-error-bags) 中：

```php
Validator::make($request->all(), [
    'title' => ['required', 'unique:posts', 'max:255'],
    'body' => ['required'],
])->validateWithBag('post');
```


<a name="named-error-bags"></a>
### 命名錯誤包 (Named Error Bags)

若您在單一頁面擁有多個表單，您可能希望為包含驗證錯誤的 `MessageBag` 命名，這能讓您取得特定表單的錯誤訊息。若要達成此目的，請將名稱作為第二個引數傳遞給 `withErrors`：

```php
return redirect('/register')->withErrors($validator, 'login');
```

接著，您就可以從 `$errors` 變數中存取該命名的 `MessageBag` 實例：

```blade
{{ $errors->login->first('email') }}
```


<a name="manual-customizing-the-error-messages"></a>
### 自訂錯誤訊息

如有需要，您可以提供自訂錯誤訊息供驗證器實例使用，以取代 Laravel 提供的預設錯誤訊息。有幾種方法可以指定自訂訊息。首先，您可以將自訂訊息作為第三個引數傳遞給 `Validator::make` 方法：

```php
$validator = Validator::make($input, $rules, $messages = [
    'required' => 'The :attribute field is required.',
]);
```

在這個範例中，`:attribute` 占位符會替換為要驗證的欄位實際名稱。您也可以在驗證訊息中使用其他占位符。例如：

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

有時您可能希望僅為特定屬性指定自訂錯誤訊息。您可以使用「點 (Dot)」標記法來做到這一點。先指定屬性名稱，後面加上規則：

```php
$messages = [
    'email.required' => 'We need to know your email address!',
];
```


<a name="specifying-custom-attribute-values"></a>
#### 指定自訂屬性數值

許多 Laravel 內建的錯誤訊息都包含一個 `:attribute` 占位符，該占位符會替換為被驗證的欄位或屬性名稱。若要為特定欄位自訂用於替換這些占位符的值，您可以將自訂屬性陣列作為第四個引數傳遞給 `Validator::make` 方法：

```php
$validator = Validator::make($input, $rules, $messages, [
    'email' => 'email address',
]);
```


<a name="performing-additional-validation"></a>
### 執行額外驗證

有時您需要在初始驗證完成後執行額外驗證。您可以使用驗證器的 `after` 方法來完成此操作。`after` 方法接受一個 Closure 或 callable 陣列，這些內容將在驗證完成後被呼叫。傳入的 callable 將接收一個 `Illuminate\Validation\Validator` 實例，允許您在必要時引發額外錯誤訊息：

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

如上所述，`after` 方法也接受 callable 陣列，如果您的「驗證後」邏輯封裝在可調用 (Invokable) 的類別中，這會特別方便，這些類別將透過其 `__invoke` 方法接收 `Illuminate\Validation\Validator` 實例：

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
## 處理通過驗證的輸入資料

在使用表單請求或手動建立的驗證器實例來驗證傳入的請求資料後，您可能希望取得實際上通過驗證的傳入請求資料。這可以透過幾種方式來實現。首先，您可以呼叫表單請求或驗證器實例上的 `validated` 方法。這個方法會傳回包含所有通過驗證資料的陣列：

```php
$validated = $request->validated();

$validated = $validator->validated();
```

或者，您也可以呼叫表單請求或驗證器實例上的 `safe` 方法。這個方法會傳回 `Illuminate\Support\ValidatedInput` 的實例。該物件提供了 `only`、`except` 與 `all` 方法，用來取得通過驗證資料的子集或是完整的驗證資料陣列：

```php
$validated = $request->safe()->only(['name', 'email']);

$validated = $request->safe()->except(['name', 'email']);

$validated = $request->safe()->all();
```

此外，`Illuminate\Support\ValidatedInput` 實例可以被迭代，且能像陣列一樣進行存取：

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

如果您想將已驗證的資料作為 [集合 (Collection)](/docs/{{version}}/collections) 實例取得，可以呼叫 `collect` 方法：

```php
$collection = $request->safe()->collect();
```


<a name="working-with-error-messages"></a>
## 處理錯誤訊息

在呼叫 `Validator` 實例上的 `errors` 方法後，您將會收到一個 `Illuminate\Support\MessageBag` 實例，該實例擁有許多方便的方法來處理錯誤訊息。自動共享給所有視圖使用的 `$errors` 變數，也是 `MessageBag` 類別的實例。


<a name="retrieving-the-first-error-message-for-a-field"></a>
#### 取得欄位的第一條錯誤訊息

若要取得指定欄位的第一條錯誤訊息，請使用 `first` 方法：

```php
$errors = $validator->errors();

echo $errors->first('email');
```


<a name="retrieving-all-error-messages-for-a-field"></a>
#### 取得欄位的所有錯誤訊息

如果需要取得指定欄位所有訊息的陣列，請使用 `get` 方法：

```php
foreach ($errors->get('email') as $message) {
    // ...
}
```

若您正在驗證陣列形式的表單欄位，可以使用 `*` 字元來取得每個陣列元素的所有訊息：

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

Laravel 內建的每一個驗證規則都有一個錯誤訊息，位於您應用程式的 `lang/en/validation.php` 檔案中。如果您的應用程式沒有 `lang` 目錄，您可以使用 `lang:publish` Artisan 命令指示 Laravel 來建立它。

在 `lang/en/validation.php` 檔案中，您會找到每個驗證規則的翻譯項目。您可以根據應用程式的需求隨意更改或修改這些訊息。

此外，您可以將此檔案複製到另一個語言目錄中，以將訊息翻譯為您應用程式所使用的語言。若要深入瞭解 Laravel 的在地化，請參考完整的[在地化說明文件](/docs/{{version}}/localization)。

> [!WARNING]
> 預設情況下，Laravel 應用程式骨架並不包含 `lang` 目錄。如果您想要自訂 Laravel 的語言檔，可以透過 `lang:publish` Artisan 命令來發布它們。


<a name="custom-messages-for-specific-attributes"></a>
#### 為特定屬性自訂訊息

您可以在應用程式的驗證語言檔中，為特定的屬性與規則組合自訂錯誤訊息。若要做到這一點，請將自訂訊息新增至應用程式 `lang/xx/validation.php` 語言檔中的 `custom` 陣列：

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

Laravel 的許多內建錯誤訊息都包含 `:attribute` 預留位置，該預留位置會被替換為正在進行驗證的欄位或屬性名稱。如果您希望驗證訊息中的 `:attribute` 部分替換為自訂名稱，可以在 `lang/xx/validation.php` 語言檔中的 `attributes` 陣列指定自訂屬性名稱：

```php
'attributes' => [
    'email' => 'email address',
],
```

> [!WARNING]
> 預設情況下，Laravel 應用程式骨架並不包含 `lang` 目錄。如果您想要自訂 Laravel 的語言檔，可以透過 `lang:publish` Artisan 命令來發布它們。


<a name="specifying-values-in-language-files"></a>
### 在語言檔中指定數值

部分 Laravel 內建的驗證規則錯誤訊息含有 `:value` 預留位置，它會被替換為請求屬性的當前數值。然而，有時您可能需要將驗證訊息中的 `:value` 部分替換為更易讀的自訂表示方式。例如，考慮以下規則，該規則指定當 `payment_type` 的數值為 `cc` 時，必須提供信用卡號碼：

```php
Validator::make($request->all(), [
    'credit_card_number' => ['required_if:payment_type,cc']
]);
```

如果此驗證規則未通過，將會產生以下錯誤訊息：

```text
The credit card number field is required when payment type is cc.
```

除了將 `cc` 顯示為付款類型數值外，您也可以透過在 `lang/xx/validation.php` 語言檔中定義 `values` 陣列，來指定對使用者更友善的數值表示方式：

```php
'values' => [
    'payment_type' => [
        'cc' => 'credit card'
    ],
],
```

> [!WARNING]
> 預設情況下，Laravel 應用程式骨架並不包含 `lang` 目錄。如果您想要自訂 Laravel 的語言檔，可以透過 `lang:publish` Artisan 命令來發布它們。

定義好此數值後，該驗證規則將會產生以下錯誤訊息：

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
[Min](#rule-min)
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

正在驗證的欄位必須為 `"yes"`、`"on"`、`1`、`"1"`、`true` 或 `"true"`。這對於驗證「服務條款」的同意或類似欄位非常有用。


<a name="rule-accepted-if"></a>
#### accepted_if:anotherfield,value,...

若另一個正在驗證的欄位等於指定的值，則正在驗證的欄位必須為 `"yes"`、`"on"`、`1`、`"1"`、`true` 或 `"true"`。這對於驗證「服務條款」的同意或類似欄位非常有用。


<a name="rule-active-url"></a>
#### active_url

根據 PHP 的 `dns_get_record` 函式，正在驗證的欄位必須具有有效的 A 或 AAAA 記錄。傳入 URL 的主機名稱會在傳給 `dns_get_record` 之前先使用 PHP 的 `parse_url` 函式進行解析與提取。

當測試執行 DNS 查詢的驗證規則時（例如 `active_url` 與 `email:dns`），你可以使用 `Validator::fakeDnsLookups` 方法。這會模擬 DNS 查詢，同時保留規則的其他驗證行為：

```php
use Illuminate\Support\Facades\Validator;

Validator::fakeDnsLookups();
```


<a name="rule-after"></a>
#### after:_date_

正在驗證的欄位必須是指定日期之後的值。日期會被傳入 PHP 的 `strtotime` 函式，以轉換為有效的 `DateTime` 實例：

```php
'start_date' => ['required', 'date', 'after:tomorrow']
```

除了傳入由 `strtotime` 評估的日期字串外，你也可以指定另一個欄位來與該日期進行比較：

```php
'finish_date' => ['required', 'date', 'after:start_date']
```

為求方便，基於日期的規則可以使用流暢的 `date` 規則建構器來建立：

```php
use Illuminate\Validation\Rule;

'start_date' => [
    'required',
    Rule::date()->after(today()->addDays(7)),
],
```

`afterToday` 與 `todayOrAfter` 方法可用於流暢地表示日期必須分別在今天之後，或在今天及今天之後：

```php
'start_date' => [
    'required',
    Rule::date()->afterToday(),
],
```


<a name="rule-after-or-equal"></a>
#### after\_or\_equal:_date_

正在驗證的欄位必須是落在或等於指定日期之後的值。更多資訊請參閱 [after](#rule-after) 規則。

為求方便，基於日期的規則可以使用流暢的 `date` 規則建構器來建立：

```php
use Illuminate\Validation\Rule;

'start_date' => [
    'required',
    Rule::date()->afterOrEqual(today()->addDays(7)),
],
```


<a name="rule-anyof"></a>
#### anyOf

`Rule::anyOf` 驗證規則允許你指定正在驗證的欄位必須符合任何一個給定的驗證規則集。例如，以下規則將驗證 `username` 欄位是一個 Email 地址，或者是長度至少為 6 個字元的英數字字串（包含破折號）：

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

正在驗證的欄位必須完全是由包含在 [\p{L}](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AL%3A%5D&g=&i=) 與 [\p{M}](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AM%3A%5D&g=&i=) 中的 Unicode 字母字元所組成。

若要將此驗證規則限制為 ASCII 範圍內的字元（`a-z` 與 `A-Z`），你可以為該驗證規則提供 `ascii` 選項：

```php
'username' => ['alpha:ascii'],
```

<a name="rule-alpha-dash"></a>
#### alpha_dash

驗證中的欄位必須完全是由包含在 [\p{L}](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AL%3A%5D&g=&i=)、[\p{M}](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AM%3A%5D&g=&i=)、[\p{N}](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AN%3A%5D&g=&i=) 中的 Unicode 字母數字字元，以及 ASCII 破折號 (`-`) 與 ASCII 底線 (`_`) 所組成。

若要將此驗證規則限制為 ASCII 範圍內的字元（`a-z`、`A-Z` 與 `0-9`），您可以為驗證規則提供 `ascii` 選項：

```php
'username' => ['alpha_dash:ascii'],
```


<a name="rule-alpha-num"></a>
#### alpha_num

驗證中的欄位必須完全是由包含在 [\p{L}](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AL%3A%5D&g=&i=)、[\p{M}](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AM%3A%5D&g=&i=) 與 [\p{N}](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AN%3A%5D&g=&i=) 中的 Unicode 字母數字字元所組成。

若要將此驗證規則限制為 ASCII 範圍內的字元（`a-z`、`A-Z` 與 `0-9`），您可以為驗證規則提供 `ascii` 選項：

```php
'username' => ['alpha_num:ascii'],
```


<a name="rule-array"></a>
#### array

驗證中的欄位必須是一個 PHP `array`。

當傳入額外的數值給 `array` 規則時，輸入陣列中的每個鍵（Key）都必須存在於提供給該規則的數值清單中。在以下範例中，輸入陣列中的 `admin` 鍵是無效的，因為它未包含在提供給 `array` 規則的數值清單中：

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

一般而言，您應該總是明確指定允許存在於陣列中的陣列鍵。


<a name="rule-array-keys"></a>
#### array_keys:_foo_,_bar_,...

驗證中的欄位必須是一個 PHP `array`，且其所有鍵都包含在給定的清單中。必須至少提供一個鍵：

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

在該欄位首次驗證失敗後，停止對該欄位執行其餘的驗證規則。

雖然 `bail` 規則只會在遇到驗證失敗時停止驗證特定欄位，但 `stopOnFirstFailure` 方法會通知驗證器，一旦發生單一驗證失敗，就應該停止驗證所有屬性：

```php
if ($validator->stopOnFirstFailure()->fails()) {
    // ...
}
```


<a name="rule-before"></a>
#### before:_date_

驗證中的欄位必須是早於給定日期的值。日期將會傳入 PHP 的 `strtotime` 函式，以轉換為有效的 `DateTime` 實例。此外，如同 [after](#rule-after) 規則，也可以提供另一個驗證中的欄位名稱作為 `date` 的數值。

為求方便，基於日期的規則也可以使用流暢的 `date` 規則建構器來建立：

```php
use Illuminate\Validation\Rule;

'start_date' => [
    'required',
    Rule::date()->before(today()->subDays(7)),
],
```

`beforeToday` 與 `todayOrBefore` 方法可用於流暢地表達日期必須分別早於今天，或是今天或更早：

```php
'start_date' => [
    'required',
    Rule::date()->beforeToday(),
],
```


<a name="rule-before-or-equal"></a>
#### before\_or\_equal:_date_

驗證中的欄位必須是早於或等於給定日期的值。日期將會傳入 PHP 的 `strtotime` 函式，以轉換為有效的 `DateTime` 實例。此外，如同 [after](#rule-after) 規則，也可以提供另一個驗證中的欄位名稱作為 `date` 的數值。

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

驗證中的欄位大小必須介於給定的 _min_ 與 _max_（含）之間。字串、數字、陣列與檔案的評估方式與 [size](#rule-size) 規則相同。


<a name="rule-boolean"></a>
#### boolean

驗證中的欄位必須能夠被轉型為布林值。接受的輸入值為 `true`、`false`、`1`、`0`、`"1"` 與 `"0"`。

您可以使用 `strict` 參數，使欄位僅在其數值為 `true` 或 `false` 時才被視為有效：

```php
'foo' => ['boolean:strict']
```


<a name="rule-confirmed"></a>
#### confirmed

驗證中的欄位必須擁有一個與之對應的 `{field}_confirmation` 欄位。例如，若驗證中的欄位是 `password`，則輸入資料中必須存在對應的 `password_confirmation` 欄位。

您也可以傳入自訂的確認欄位名稱。例如，`confirmed:repeat_username` 將會預期 `repeat_username` 欄位必須與驗證中的欄位一致。


<a name="rule-contains"></a>
#### contains:_foo_,_bar_,...

驗證中的欄位必須是一個包含所有給定參數值的陣列。由於此規則通常需要您對陣列進行 `implode`，因此可以使用 `Rule::contains` 方法來流暢地建構此規則：

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

驗證中的欄位必須是一個不包含任何給定參數值的陣列。由於此規則通常需要您對陣列進行 `implode`，因此可以使用 `Rule::doesntContain` 方法來流暢地建構此規則：

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

根據 PHP 的 `strtotime` 函式，驗證中的欄位必須是一個有效且非相對位置的日期。


<a name="rule-date-equals"></a>
#### date_equals:_date_

驗證中的欄位必須等於給定的日期。日期將會傳入 PHP 的 `strtotime` 函式，以轉換為有效的 `DateTime` 實例。


<a name="rule-date-format"></a>
#### date_format:_format_,...

驗證中的欄位必須符合給定的 _formats_ 格式之一。在驗證欄位時，您應該選擇使用 `date` **或** `date_format` **其中之一**，而非兩者皆用。此驗證規則支援 PHP [DateTime](https://www.php.net/manual/en/class.datetime.php) 類別所支援的所有格式。

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

驗證中的欄位必須為數字，且必須包含指定的小數位數：

```php
// Must have exactly two decimal places (9.99)...
'price' => ['decimal:2']

// Must have between 2 and 4 decimal places...
'price' => ['decimal:2,4']
```


<a name="rule-declined"></a>
#### declined

驗證中的欄位必須為 `"no"`、`"off"`、`0`、`"0"`、`false` 或 `"false"`。

<a name="rule-declined-if"></a>
#### declined_if:anotherfield,value,...

當另一個驗證中的欄位等於指定數值時，驗證中的欄位必須為 `"no"`、`"off"`、`0`、`"0"`、`false` 或 `"false"`。


<a name="rule-different"></a>
#### different:_field_

驗證中的欄位必須具有與 _field_ 不同的數值。


<a name="rule-digits"></a>
#### digits:_value_

驗證中的整數長度必須正好為 _value_。


<a name="rule-digits-between"></a>
#### digits_between:_min_,_max_

驗證中的整數長度必須介於給定的 _min_ 與 _max_ 之間。


<a name="rule-dimensions"></a>
#### dimensions

驗證中的檔案必須是符合規則參數所指定的尺寸限制之圖片：

```php
'avatar' => ['dimensions:min_width=100,min_height=200']
```

可用的限制條件有：_min\_width_、_max\_width_、_min\_height_、_max\_height_、_width_、_height_、_ratio_、_min\_ratio_、_max\_ratio_。

_ratio_ 限制應該表示為寬度除以高度。這可以用像 `3/2` 這樣的分數或像 `1.5` 這樣的浮點數來指定：

```php
'avatar' => ['dimensions:ratio=3/2']
```

_min\_ratio_ 與 _max\_ratio_ 限制可用於定義可接受的長寬比範圍：

```php
'avatar' => ['dimensions:min_ratio=1/2,max_ratio=3/2']
```

由於此規則需要多個引數，使用 `Rule::dimensions` 方法來順暢地建構規則通常更為方便：

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

您也可以使用 `minRatio`、`maxRatio` 與 `ratioBetween` 方法來順暢地定義比例限制：

```php
Rule::dimensions()->ratioBetween(min: 1 / 2, max: 3 / 2)
```


<a name="rule-distinct"></a>
#### distinct

驗證陣列時，驗證中的欄位不能有任何重複的值：

```php
'foo.*.id' => ['distinct']
```

Distinct 預設使用鬆散的變數比較。若要使用嚴格比較，您可以在驗證規則定義中加入 `strict` 參數：

```php
'foo.*.id' => ['distinct:strict']
```

您可以在驗證規則的引數中加入 `ignore_case`，使規則忽略大小寫的差異：

```php
'foo.*.id' => ['distinct:ignore_case']
```


<a name="rule-doesnt-start-with"></a>
#### doesnt_start_with:_foo_,_bar_,...

驗證中的欄位不能以給定值的其中任何一個開頭。


<a name="rule-doesnt-end-with"></a>
#### doesnt_end_with:_foo_,_bar_,...

驗證中的欄位不能以給定值的其中任何一個結尾。


<a name="rule-email"></a>
#### email

驗證中的欄位必須符合 Email 格式。此驗證規則使用 [egulias/email-validator](https://github.com/egulias/EmailValidator) 套件來驗證 Email 地址。預設會套用 `RFCValidation` 驗證器，但您也可以套用其他的驗證樣式：

```php
'email' => ['email:rfc,dns']
```

上述範例將會套用 `RFCValidation` 與 `DNSCheckValidation` 驗證。以下是您可以套用的完整驗證樣式清單：

<div class="content-list" markdown="1">

- `rfc`：`RFCValidation` - 根據 [支援的 RFC](https://github.com/egulias/EmailValidator?tab=readme-ov-file#supported-rfcs) 驗證 Email 地址。
- `strict`：`NoRFCWarningsValidation` - 根據 [支援的 RFC](https://github.com/egulias/EmailValidator?tab=readme-ov-file#supported-rfcs) 驗證 Email，若發現警告（例如末尾句點或多個連續句點）則驗證失敗。
- `dns`：`DNSCheckValidation` - 確保 Email 地址的網域具有有效的 MX 記錄。
- `spoof`：`SpoofCheckValidation` - 確保 Email 地址不包含同形文字或具欺騙性的 Unicode 字元。
- `filter`：`FilterEmailValidation` - 確保 Email 地址符合 PHP 的 `filter_var` 函式驗證。
- `filter_unicode`：`FilterEmailValidation::unicode()` - 確保 Email 地址符合 PHP 的 `filter_var` 函式驗證，並允許某些 Unicode 字元。

</div>

為了便利起見，Email 驗證規則可以使用順暢的規則建構器進行建構：

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

`dns` 驗證器會執行真實的 DNS 查詢，以確認該地址的網域具有有效的 MX 記錄。它不會確認個人信箱是否存在。

由於您的測試不應依賴真實的 DNS 查詢，您可以使用 `Validator::fakeDnsLookups` 方法來[虛擬化 DNS 查詢](#rule-active-url)，同時讓任何其他被請求的驗證（例如 `rfc`）繼續執行：

```php
use Illuminate\Support\Facades\Validator;

Validator::fakeDnsLookups();
```

這能讓您的應用程式在測試時繼續使用其現有的驗證規則：

```php
'email' => ['required', 'email:rfc,dns'],
```

> [!WARNING]
> `dns` 與 `spoof` 驗證器需要 PHP `intl` 擴充套件。


<a name="rule-encoding"></a>
#### encoding:*encoding_type*

驗證中的欄位必須符合指定的字元編碼。此規則使用 PHP 的 `mb_check_encoding` 函式來驗證給定檔案或字串值的編碼。為了便利起見，可以使用 Laravel 的流暢檔案規則建構器來建立 `encoding` 規則：

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

驗證中的欄位必須以給定值的其中任何一個結尾。


<a name="rule-enum"></a>
#### enum

`Enum` 規則是一個基於類別的規則，用於驗證驗證中的欄位是否包含有效的 Enum 數值。`Enum` 規則接受 Enum 的名稱作為其唯一的建構子引數。驗證基本型別數值時，應向 `Enum` 規則提供 Backed Enum：

```php
use App\Enums\ServerStatus;
use Illuminate\Validation\Rule;

$request->validate([
    'status' => [Rule::enum(ServerStatus::class)],
]);
```

`Enum` 規則的 `only` 與 `except` 方法可用於限制哪些 Enum 情況應被視為有效：

```php
Rule::enum(ServerStatus::class)
    ->only([ServerStatus::Pending, ServerStatus::Active]);

Rule::enum(ServerStatus::class)
    ->except([ServerStatus::Pending, ServerStatus::Active]);
```

`when` 方法可用於條件式修改 `Enum` 規則：

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

驗證中的欄位將從 `validate` 與 `validated` 方法傳回的請求資料中排除。


<a name="rule-exclude-if"></a>
#### exclude_if:_anotherfield_,_value_

若 _anotherfield_ 欄位等於 _value_，驗證中的欄位將從 `validate` 與 `validated` 方法傳回的請求資料中排除。

如果需要複雜的條件排除邏輯，您可以使用 `Rule::excludeIf` 方法。此方法接受布林值或 Closure。當給定 Closure 時，Closure 應傳回 `true` 或 `false` 以表示是否應排除驗證中的欄位：

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

除非 _anotherfield_ 的欄位等於 _value_，否則被驗證的欄位將會從 `validate` 與 `validated` 方法所回傳的請求資料中排除。若 _value_ 為 `null` (`exclude_unless:name,null`)，則除非比較的欄位為 `null` 或請求資料中缺少該比較的欄位，否則被驗證的欄位將會被排除。

若需要更複雜的條件式排除邏輯，您可以使用 `Rule::excludeUnless` 方法。該方法接受一個布林值或 Closure。當傳入 Closure 時，Closure 應回傳 `true` 或 `false` 以指示被驗證的欄位是否不應被排除：

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

如果 _anotherfield_ 欄位存在，則被驗證的欄位將會從 `validate` 與 `validated` 方法所回傳的請求資料中排除。


<a name="rule-exclude-without"></a>
#### exclude_without:_anotherfield_

如果 _anotherfield_ 欄位不存在，則被驗證的欄位將會從 `validate` 與 `validated` 方法所回傳的請求資料中排除。


<a name="rule-exists"></a>
#### exists:_table_,_column_

被驗證的欄位必須存在於指定的資料庫資料表中。


<a name="basic-usage-of-exists-rule"></a>
#### Exists 規則的基本用法

```php
'state' => ['exists:states']
```

若未指定 `column` 選項，將會使用該欄位名稱。因此在此範例中，該規則將驗證 `states` 資料庫表中是否包含一筆 `state` 欄位值與請求的 `state` 屬性值相符的紀錄。


<a name="specifying-a-custom-column-name"></a>
#### 指定自訂欄位名稱

您可以在資料庫資料表名稱後面加上欄位名稱，以明確指定驗證規則應使用的資料庫欄位名稱：

```php
'state' => ['exists:states,abbreviation']
```

有時候，您可能需要為 `exists` 查詢指定特定的資料庫連線。您可以透過在資料表名稱前加上連線名稱來達成：

```php
'email' => ['exists:connection.staff,email']
```

除了直接指定資料表名稱外，您也可以指定應用於確定資料表名稱的 Eloquent Model：

```php
'user_id' => ['exists:App\Models\User,id']
```

若您想自訂驗證規則所執行的查詢，可以使用 `Rule` 類別來流暢地定義該規則：

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

您可以將欄位名稱作為第二個引數傳遞給 `exists` 方法，以明確指定由 `Rule::exists` 方法產生的 `exists` 規則所應使用的資料庫欄位名稱：

```php
'state' => [Rule::exists('states', 'abbreviation')],
```

有時您可能想要驗證陣列中的值是否存在於資料庫中。您可以透過同時將 `exists` 與 [array](#rule-array) 規則指定給被驗證的欄位來達成：

```php
'states' => ['array', Rule::exists('states', 'abbreviation')],
```

當這兩個規則都被指定給某個欄位時，Laravel 將自動建置單一查詢，以確定所有給定的值是否存在於指定的資料表中。


<a name="rule-extensions"></a>
#### extensions:_foo_,_bar_,...

被驗證的檔案必須具有符合列出副檔名之一的使用者自訂副檔名：

```php
'photo' => ['required', 'extensions:jpg,png'],
```

> [!WARNING]
> 您絕不應該僅依賴使用者指定的副檔名來驗證檔案。此規則通常應始終與 [mimes](#rule-mimes) 或 [mimetypes](#rule-mimetypes) 規則結合使用。


<a name="rule-file"></a>
#### file

被驗證的欄位必須是成功上傳的檔案。


<a name="rule-filled"></a>
#### filled

被驗證的欄位當存在時不得為空。


<a name="rule-gt"></a>
#### gt:_field_

被驗證的欄位必須大於給定的 _field_ 或 _value_。這兩個欄位必須是相同型別。字串、數值、陣列與檔案會採用與 [size](#rule-size) 規則相同的慣例進行評估。


<a name="rule-gte"></a>
#### gte:_field_

被驗證的欄位必須大於或等於給定的 _field_ 或 _value_。這兩個欄位必須是相同型別。字串、數值、陣列與檔案會採用與 [size](#rule-size) 規則相同的慣例進行評估。


<a name="rule-hex-color"></a>
#### hex_color

被驗證的欄位必須包含符合[十六進位 (Hexadecimal)](https://developer.mozilla.org/en-US/docs/Web/CSS/hex-color) 格式的有效顏色值。


<a name="rule-image"></a>
#### image

被驗證的檔案必須是圖片 (jpg, jpeg, png, bmp, gif, 或 webp)。

> [!WARNING]
> 預設情況下，image 規則因考量 XSS 漏洞的風險而不允許 SVG 檔案。若您需要允許 SVG 檔案，可以向 `image` 規則提供 `allow_svg` 指令 (`image:allow_svg`)。


<a name="rule-in"></a>
#### in:_foo_,_bar_,...

被驗證的欄位必須包含在給定的值清單中。由於此規則通常需要您對陣列進行 `implode`，因此可以使用 `Rule::in` 方法來流暢地建構該規則：

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

當 `in` 規則與 `array` 規則結合使用時，輸入陣列中的每個值都必須存在於提供給 `in` 規則的值清單中。在以下範例中，輸入陣列中的 `LAS` 機場代碼是無效的，因為它未包含在提供給 `in` 規則的機場清單中：

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

被驗證的欄位必須是一個陣列，且至少包含給定的 _values_ 之一作為該陣列中的鍵名：

```php
'config' => ['array', 'in_array_keys:timezone']
```


<a name="rule-integer"></a>
#### integer

被驗證的欄位必須是整數。

您可以使用 `strict` 參數，僅在欄位型別為 `integer` 時才視為有效。包含整數值的字串將被視為無效：

```php
'age' => ['integer:strict']
```

> [!WARNING]
> 此驗證規則不會驗證輸入資料是否屬於 "integer" 變數型態，僅會驗證輸入資料是否為 PHP `FILTER_VALIDATE_INT` 規則所接受的型別。若需要將輸入資料驗證為數字，請將此規則與 [`numeric` 驗證規則](#rule-numeric) 結合使用。


<a name="rule-ip"></a>
#### ip

被驗證的欄位必須是 IP 位址。


<a name="ipv4"></a>
#### ipv4

被驗證的欄位必須是 IPv4 位址。


<a name="ipv6"></a>
#### ipv6

被驗證的欄位必須是 IPv6 位址。


<a name="rule-json"></a>
#### json

被驗證的欄位必須是有效的 JSON 字串。


<a name="rule-lt"></a>
#### lt:_field_

被驗證的欄位必須小於給定的 _field_。這兩個欄位必須是相同型別。字串、數值、陣列與檔案會採用與 [size](#rule-size) 規則相同的慣例進行評估。

<a name="rule-lte"></a>
#### lte:_field_

驗證中的欄位必須小於或等於給定的 _field_。這兩個欄位必須是相同的型態。字串、數值、陣列和檔案的評估方式與 [size](#rule-size) 規則的慣例相同。


<a name="rule-lowercase"></a>
#### lowercase

驗證中的欄位必須全為小寫。


<a name="rule-list"></a>
#### list

驗證中的欄位必須是列表 (List) 形式的陣列。如果陣列的鍵 (Key) 是由從 0 到 `count($array) - 1` 的連續數字所組成，則該陣列會被視為列表。


<a name="rule-mac"></a>
#### mac_address

驗證中的欄位必須是 MAC 位址。


<a name="rule-max"></a>
#### max:_value_

驗證中的欄位必須小於或等於最大 _value_。字串、數值、陣列和檔案的評估方式與 [size](#rule-size) 規則的方式相同。


<a name="rule-max-digits"></a>
#### max_digits:_value_

驗證中的整數最大長度必須為 _value_。


<a name="rule-mimetypes"></a>
#### mimetypes:_text/plain_,...

驗證中的檔案必須符合給定的 MIME 類型之一：

```php
'video' => ['mimetypes:video/avi,video/mpeg,video/quicktime'],

'media' => ['mimetypes:image/*,video/*'],
```

為了確定上傳檔案的 MIME 類型，系統會讀取該檔案的內容，且框架將嘗試猜測其 MIME 類型，這可能與用戶端提供的 MIME 類型不同。


<a name="rule-mimes"></a>
#### mimes:_foo_,_bar_,...

驗證中的檔案必須具有與列出的副檔名之一相對應的 MIME 類型：

```php
'photo' => ['mimes:jpg,bmp,png']
```

即使你只需要指定副檔名，此規則實際上會透過讀取檔案內容並猜測其 MIME 類型來驗證檔案的 MIME 類型。可以在以下位置找到完整的 MIME 類型及其對應副檔名清單：

[https://svn.apache.org/repos/asf/httpd/httpd/trunk/docs/conf/mime.types](https://svn.apache.org/repos/asf/httpd/httpd/trunk/docs/conf/mime.types)


<a name="mime-types-and-extensions"></a>
#### MIME 類型與副檔名 (MIME Types and Extensions)

此驗證規則不會驗證 MIME 類型與使用者為檔案所指定的副檔名之間是否一致。例如，即使檔案命名為 `photo.txt`，只要其包含有效的 PNG 內容，`mimes:png` 驗證規則就會將其視為有效的 PNG 圖片。如果你想要驗證使用者為檔案指定的副檔名，可以使用 [extensions](#rule-extensions) 規則。


<a name="rule-min"></a>
#### min:_value_

驗證中的欄位必須具有最小 _value_。字串、數值、陣列和檔案的評估方式與 [size](#rule-size) 規則的方式相同。


<a name="rule-min-digits"></a>
#### min_digits:_value_

驗證中的整數最小長度必須為 _value_。


<a name="rule-multiple-of"></a>
#### multiple_of:_value_

驗證中的欄位必須是 _value_ 的倍數。


<a name="rule-missing"></a>
#### missing

驗證中的欄位不得存在於輸入資料中。


<a name="rule-missing-if"></a>
#### missing_if:_anotherfield_,_value_,...

當 _anotherfield_ 欄位等於任何 _value_ 時，驗證中的欄位不得存在。


<a name="rule-missing-unless"></a>
#### missing_unless:_anotherfield_,_value_

除非 _anotherfield_ 欄位等於任何 _value_，否則驗證中的欄位不得存在。


<a name="rule-missing-with"></a>
#### missing_with:_foo_,_bar_,...

_當且僅當_ 任何其他指定的欄位存在時，驗證中的欄位才不得存在。


<a name="rule-missing-with-all"></a>
#### missing_with_all:_foo_,_bar_,...

_當且僅當_ 所有其他指定的欄位都存在時，驗證中的欄位才不得存在。


<a name="rule-not-in"></a>
#### not_in:_foo_,_bar_,...

驗證中的欄位不得包含在給定的數值清單中。可以使用 `Rule::notIn` 方法流暢地建構此規則：

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

驗證中的欄位不得符合給定的正規表示式。

在內部，此規則使用 PHP 的 `preg_match` 函式。指定的模式應遵循 `preg_match` 所要求的相同格式，因此也應包含有效的定界符 (Delimiter)。例如：`'email' => ['not_regex:/^.+$/i']`。


<a name="rule-nullable"></a>
#### nullable

驗證中的欄位可以為 `null`。


<a name="rule-numeric"></a>
#### numeric

驗證中的欄位必須是[數值 (Numeric)](https://www.php.net/manual/en/function.is-numeric.php)。

你可以使用 `strict` 參數，讓該欄位僅在其值為整數 (Integer) 或浮點數 (Float) 型態時才被視為有效。數字字串將被視為無效：

```php
'amount' => ['numeric:strict']
```


<a name="rule-present"></a>
#### present

驗證中的欄位必須存在於輸入資料中。


<a name="rule-present-if"></a>
#### present_if:_anotherfield_,_value_,...

當 _anotherfield_ 欄位等於任何 _value_ 時，驗證中的欄位必須存在。


<a name="rule-present-unless"></a>
#### present_unless:_anotherfield_,_value_

除非 _anotherfield_ 欄位等於任何 _value_，否則驗證中的欄位必須存在。


<a name="rule-present-with"></a>
#### present_with:_foo_,_bar_,...

_當且僅當_ 任何其他指定的欄位存在時，驗證中的欄位才必須存在。


<a name="rule-present-with-all"></a>
#### present_with_all:_foo_,_bar_,...

_當且僅當_ 所有其他指定的欄位都存在時，驗證中的欄位才必須存在。


<a name="rule-prohibited"></a>
#### prohibited

驗證中的欄位必須不存在或為空。如果欄位符合以下條件之一，則視為「空」：

<div class="content-list" markdown="1">

- 數值為 `null`。
- 數值為空字串。
- 數值為空陣列或空的 `Countable` 物件。
- 數值為路徑為空的上傳檔案。

</div>


<a name="rule-prohibited-if"></a>
#### prohibited_if:_anotherfield_,_value_,...

如果 _anotherfield_ 欄位等於任何 _value_，則驗證中的欄位必須不存在或為空。如果欄位符合以下條件之一，則視為「空」：

<div class="content-list" markdown="1">

- 數值為 `null`。
- 數值為空字串。
- 數值為空陣列或空的 `Countable` 物件。
- 數值為路徑為空的上傳檔案。

</div>

如果需要複雜的條件式禁止邏輯，可以使用 `Rule::prohibitedIf` 方法。此方法接受布林值或 Closure。當傳入 Closure 時，Closure 應回傳 `true` 或 `false` 以指示是否應禁止驗證中的欄位：

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

如果 _anotherfield_ 欄位等於 `"yes"`、`"on"`、`1`、`"1"`、`true` 或 `"true"`，則驗證中的欄位必須不存在或為空。


<a name="rule-prohibited-if-declined"></a>
#### prohibited_if_declined:_anotherfield_,...

如果 _anotherfield_ 欄位等於 `"no"`、`"off"`、`0`、`"0"`、`false` 或 `"false"`，則驗證中的欄位必須不存在或為空。

<a name="rule-prohibited-unless"></a>
#### prohibited_unless:_anotherfield_,_value_,...

驗證中的欄位必須不存在或為空，除非 _anotherfield_ 欄位等於任何 _value_。若符合以下任一條件，則該欄位會被視為「為空」：

<div class="content-list" markdown="1">

- 數值為 `null`。
- 數值為空字串。
- 數值為空陣列或空的 `Countable` 物件。
- 數值為路徑為空的上傳檔案。

</div>

如果需要更複雜的條件式禁止邏輯，可以使用 `Rule::prohibitedUnless` 方法。此方法接受布林值或 Closure。當傳入 Closure 時，該 Closure 應回傳 `true` 或 `false`，用來指出驗證中的欄位是否不應被禁止：

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

如果驗證中的欄位非不存在且非為空，則 _anotherfield_ 中的所有欄位必須不存在或為空。若符合以下任一條件，則該欄位會被視為「為空」：

<div class="content-list" markdown="1">

- 數值為 `null`。
- 數值為空字串。
- 數值為空陣列或空的 `Countable` 物件。
- 數值為路徑為空的上傳檔案。

</div>


<a name="rule-regex"></a>
#### regex:_pattern_

驗證中的欄位必須符合給定的正則表達式。

在內部，此規則使用的是 PHP 的 `preg_match` 函式。指定的 Pattern 應遵循 `preg_match` 所要求的相同格式，因此也必須包含有效的界定符（Delimiter）。例如：`'email' => ['regex:/^.+@.+$/i']`。


<a name="rule-required"></a>
#### required

驗證中的欄位必須存在於輸入資料中且不能為空。若符合以下任一條件，則該欄位會被視為「為空」：

<div class="content-list" markdown="1">

- 數值為 `null`。
- 數值為空字串。
- 數值為空陣列或空的 `Countable` 物件。
- 數值為沒有路徑的上傳檔案。

</div>


<a name="rule-required-if"></a>
#### required_if:_anotherfield_,_value_,...

當 _anotherfield_ 欄位等於任何 _value_ 時，驗證中的欄位必須存在且不能為空。

如果你想為 `required_if` 規則建立更複雜的條件，可以使用 `Rule::requiredIf` 方法。此方法接受布林值或 Closure。當傳入 Closure 時，該 Closure 應回傳 `true` 或 `false`，用來指出驗證中的欄位是否為必填：

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

當 _anotherfield_ 欄位等於 `"yes"`、`"on"`、`1`、`"1"`、`true` 或 `"true"` 時，驗證中的欄位必須存在且不能為空。


<a name="rule-required-if-declined"></a>
#### required_if_declined:_anotherfield_,...

當 _anotherfield_ 欄位等於 `"no"`、`"off"`、`0`、`"0"`、`false` 或 `"false"` 時，驗證中的欄位必須存在且不能為空。


<a name="rule-required-unless"></a>
#### required_unless:_anotherfield_,_value_,...

驗證中的欄位必須存在且不能為空，除非 _anotherfield_ 欄位等於任何 _value_。這也代表 _anotherfield_ 必須存在於請求資料中，除非 _value_ 為 `null`。如果 _value_ 為 `null`（例如 `required_unless:name,null`），則驗證中的欄位將會是必填，除非被比較的欄位為 `null` 或被比較的欄位不存在於請求資料中。

如果你想為 `required_unless` 規則建立更複雜的條件，可以使用 `Rule::requiredUnless` 方法。此方法接受布林值或 Closure。當傳入 Closure 時，該 Closure 應回傳 `true` 或 `false`，用來指出驗證中的欄位是否為非必填：

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

_只有當_任何其他指定欄位存在且不為空時，驗證中的欄位才必須存在且不能為空。


<a name="rule-required-with-all"></a>
#### required_with_all:_foo_,_bar_,...

_只有當_所有其他指定欄位皆存在且不為空時，驗證中的欄位才必須存在且不能為空。


<a name="rule-required-without"></a>
#### required_without:_foo_,_bar_,...

_只有當_任何其他指定欄位為空或不存在時，驗證中的欄位才必須存在且不能為空。


<a name="rule-required-without-all"></a>
#### required_without_all:_foo_,_bar_,...

_只有當_所有其他指定欄位皆為空或不存在時，驗證中的欄位才必須存在且不能為空。


<a name="rule-required-array-keys"></a>
#### required_array_keys:_foo_,_bar_,...

驗證中的欄位必須是一個陣列，且必須至少包含指定的鍵名（Keys）。


<a name="rule-same"></a>
#### same:_field_

給定的 _field_ 必須與驗證中的欄位數值相符。


<a name="rule-size"></a>
#### size:_value_

驗證中的欄位大小必須符合給定的 _value_。對於字串資料，_value_ 對應字元個數。對於數值資料，_value_ 對應給定的整數值（該屬性也必須擁有 `numeric` 或 `integer` 規則）。對於陣列，_size_ 對應陣列的 `count`（元素數量）。對於檔案，_size_ 對應檔案大小（單位為千位元組 KB）。讓我們看一些範例：

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

驗證中的欄位必須以給定的其中一個數值開頭。


<a name="rule-string"></a>
#### string

驗證中的欄位必須是字串。如果你想允許該欄位也可以是 `null`，你應該為該欄位分配 `nullable` 規則。

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

字串規則建構器為常見的字串約束提供了多種方法，包括 `alpha`、`alphaDash`、`alphaNumeric`、`ascii`、`between`、`doesntEndWith`、`doesntStartWith`、`endsWith`、`exactly`、`lowercase`、`max`、`min`、`startsWith` 以及 `uppercase`。由於規則建構器支援條件控制，你也可以使用 `when` 和 `unless` 方法來條件式地套用約束條件。


<a name="rule-timezone"></a>
#### timezone

驗證中的欄位必須是根據 `DateTimeZone::listIdentifiers` 方法判斷有效的時區識別碼。

[`DateTimeZone::listIdentifiers` 方法所接受的引數](https://www.php.net/manual/en/datetimezone.listidentifiers.php)也可以提供給此驗證規則：

```php
'timezone' => ['required', 'timezone:all'];

'timezone' => ['required', 'timezone:Africa'];

'timezone' => ['required', 'timezone:per_country,US'];
```

<a name="rule-unique"></a>
#### unique:_table_,_column_

正被驗證的欄位值不得存在於指定的資料庫資料表中。

**指定自訂資料表 / 欄位名稱：**

除了直接指定資料表名稱外，您也可以指定應用於確定資料表名稱的 Eloquent 模型：

```php
'email' => ['unique:App\Models\User,email_address']
```

`column` 選項可用於指定欄位對應的資料庫欄位。如果未指定 `column` 選項，將會使用正被驗證的欄位名稱。

```php
'email' => ['unique:users,email_address']
```

**指定自訂資料庫連線**

有時，您可能需要為驗證器執行的資料庫查詢設定自訂連線。若要達到此目的，您可以在資料表名稱前加上連線名稱：

```php
'email' => ['unique:connection.users,email_address']
```

**強制 Unique 規則忽略指定的 ID：**

有時，您可能希望在進行唯一性驗證時忽略特定的 ID。例如，考慮一個包含使用者姓名、電子郵件地址和位置的「更新個人資料」畫面。您可能希望驗證電子郵件地址是否唯一。然而，如果使用者只更改了姓名欄位而沒有更改電子郵件欄位，您不會希望拋出驗證錯誤，因為該使用者本來就是該電子郵件地址的擁有者。

為了指示驗證器忽略使用者的 ID，我們將使用 `Rule` 類別來流暢地定義規則：

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
> 您絕不應該將任何由使用者控制的請求輸入傳遞給 `ignore` 方法。相對地，您應該只傳遞系統產生的唯一 ID，例如來自 Eloquent 模型實例的自動遞增 ID 或 UUID。否則，您的應用程式將容易受到 SQL 注入攻擊。

除了將模型主鍵的值傳遞給 `ignore` 方法外，您也可以傳遞整個模型實例。Laravel 會自動從模型中擷取主鍵：

```php
Rule::unique('users')->ignore($user)
```

如果您的資料表使用的是 `id` 以外的主鍵欄位名稱，您可以在呼叫 `ignore` 方法時指定該欄位名稱：

```php
Rule::unique('users')->ignore($user->id, 'user_id')
```

預設情況下，`unique` 規則會檢查與正被驗證的屬性名稱相符的欄位唯一性。不過，您可以傳遞不同的欄位名稱作為 `unique` 方法的第二個引數：

```php
Rule::unique('users', 'email_address')->ignore($user->id)
```

**新增額外的 Where 子句：**

您可以透過使用 `where` 方法自訂查詢來指定額外的查詢條件。例如，讓我們新增一個查詢條件，將查詢範圍限制為僅搜尋 `account_id` 欄位值為 `1` 的紀錄：

```php
'email' => Rule::unique('users')->where(fn (Builder $query) => $query->where('account_id', 1))
```

**在唯一性檢查中忽略軟刪除紀錄：**

預設情況下，唯一性規則在判定唯一性時會包含軟刪除的紀錄。若要從唯一性檢查中排除軟刪除紀錄，您可以呼叫 `withoutTrashed` 方法：

```php
Rule::unique('users')->withoutTrashed();
```

如果您的模型對軟刪除紀錄使用 `deleted_at` 以外的欄位名稱，可以在呼叫 `withoutTrashed` 方法時提供該欄位名稱：

```php
Rule::unique('users')->withoutTrashed('was_deleted_at');
```


<a name="rule-uppercase"></a>
#### uppercase

正被驗證的欄位必須是大寫。


<a name="rule-url"></a>
#### url

正被驗證的欄位必須是有效的 URL。

如果您想指定應視為有效的 URL 通訊協定，可以將這些協定作為驗證規則參數傳遞：

```php
'url' => ['url:http,https'],

'game' => ['url:minecraft,steam'],
```


<a name="rule-ulid"></a>
#### ulid

正被驗證的欄位必須是有效的[通用唯一可字典排序識別碼](https://github.com/ulid/spec) (ULID)。


<a name="rule-uuid"></a>
#### uuid

正被驗證的欄位必須是有效的 RFC 9562（版本 1、3、4、5、6、7 或 8）通用唯一識別碼 (UUID)。

您也可以驗證給定的 UUID 是否符合特定版本的 UUID 規範：

```php
'uuid' => ['uuid:4']
```

<a name="conditionally-adding-rules"></a>
## 條件式新增規則


<a name="skipping-validation-when-fields-have-certain-values"></a>
#### 當欄位具有特定數值時跳過驗證

當另一個欄位具有特定數值時，您有時可能希望不對指定的欄位進行驗證。您可以使用 `exclude_if` 驗證規則來達成此目的。在此範例中，若 `has_appointment` 欄位的值為 `false`，則不會驗證 `appointment_date` 和 `doctor_name` 欄位：

```php
use Illuminate\Support\Facades\Validator;

$validator = Validator::make($data, [
    'has_appointment' => ['required', 'boolean'],
    'appointment_date' => ['exclude_if:has_appointment,false', 'required', 'date'],
    'doctor_name' => ['exclude_if:has_appointment,false', 'required', 'string'],
]);
```

或者，您可以使用 `exclude_unless` 規則，除非另一個欄位具有特定數值，否則不驗證指定的欄位：

```php
$validator = Validator::make($data, [
    'has_appointment' => ['required', 'boolean'],
    'appointment_date' => ['exclude_unless:has_appointment,true', 'required', 'date'],
    'doctor_name' => ['exclude_unless:has_appointment,true', 'required', 'string'],
]);
```


<a name="validating-when-present"></a>
#### 當欄位存在時才進行驗證

在某些情況下，您可能希望**僅在**被驗證的資料中存在某個欄位時，才對該欄位執行驗證檢查。要快速達到這個目的，只需將 `sometimes` 規則加入您的規則列表中：

```php
$validator = Validator::make($data, [
    'email' => ['sometimes', 'required', 'email'],
]);
```

在上述範例中，只有當 `email` 欄位存在於 `$data` 陣列中時，才會對其進行驗證。

> [!NOTE]
> 如果您正嘗試驗證一個應該始終存在但可能為空的欄位，請參考[關於選填欄位的注意事項](#a-note-on-optional-fields)。


<a name="complex-conditional-validation"></a>
#### 複雜條件式驗證

有時候，您可能希望基於更複雜的條件邏輯來新增驗證規則。例如，您可能希望僅在另一個欄位的值大於 100 時，才要求填寫某個欄位。或者，您可能需要僅在另一個欄位存在時，兩個欄位才具有指定的值。新增這些驗證規則並不麻煩。首先，使用永遠不會改變的_靜態規則_建立一個 `Validator` 實例：

```php
use Illuminate\Support\Facades\Validator;

$validator = Validator::make($request->all(), [
    'email' => ['required', 'email'],
    'games' => ['required', 'integer', 'min:0'],
]);
```

假設我們的 Web 應用程式是專為遊戲收藏家設計的。如果遊戲收藏家在我們的應用程式中註冊，且擁有超過 100 款遊戲，我們希望他們解釋為什麼擁有這麼多遊戲。例如，也許他們開了一家遊戲轉賣店，或者只是單純喜歡收集遊戲。若要條件式地新增這個需求，我們可以使用 `Validator` 實例上的 `sometimes` 方法。

```php
use Illuminate\Support\Fluent;

$validator->sometimes('reason', ['required', 'max:500'], function (Fluent $input) {
    return $input->games >= 100;
});
```

傳遞給 `sometimes` 方法的第一個引數是我們要進行條件式驗證的欄位名稱。第二個引數是我們想要新增的規則列表。如果作為第三個引數傳遞的 Closure 回傳 `true`，這些規則就會被新增。這個方法讓建立複雜的條件式驗證變得非常輕鬆。您甚至可以一次為多個欄位新增條件式驗證：

```php
$validator->sometimes(['reason', 'cost'], 'required', function (Fluent $input) {
    return $input->games >= 100;
});
```

> [!NOTE]
> 傳遞給 Closure 的 `$input` 參數會是 `Illuminate\Support\Fluent` 的實例，可用於存取正在進行驗證的輸入資料與檔案。


<a name="complex-conditional-array-validation"></a>
#### 複雜條件式陣列驗證

有時候，您可能想根據同一個巢狀陣列中的另一個欄位來驗證某個欄位，但您並不知道該欄位的索引值。在這些情況下，您可以讓 Closure 接收第二個引數，該引數將會是目前正在被驗證的陣列單一項目：

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

就像傳遞給 Closure 的 `$input` 參數一樣，當屬性資料為陣列時，`$item` 參數是 `Illuminate\Support\Fluent` 的實例；否則，它是一個字串。

<a name="validating-arrays"></a>
## 驗證陣列

正如[陣列驗證規則文件](#rule-array)中所討論的，`array` 規則接受允許的陣列鍵值清單。若陣列中存在任何額外的鍵值，驗證將會失敗：

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

一般來說，你應該總是指定允許出現在陣列中的陣列鍵值。否則，驗證器的 `validate` 和 `validated` 方法將會回傳所有通過驗證的資料（包含陣列及其所有鍵值），即使這些鍵值並未通過其他巢狀陣列驗證規則的驗證。


<a name="validating-nested-array-input"></a>
### 驗證巢狀陣列輸入

驗證巢狀陣列架構的表單輸入欄位並不需要很痛苦。你可以使用「點語法 (Dot Notation)」來驗證陣列內的屬性。例如，若傳入的 HTTP 請求包含 `photos[profile]` 欄位，你可以像這樣驗證它：

```php
use Illuminate\Support\Facades\Validator;

$validator = Validator::make($request->all(), [
    'photos.profile' => ['required', 'image'],
]);
```

你也可以驗證陣列中的每個元素。例如，若要驗證給定陣列輸入欄位中的每個 Email 都是唯一的，你可以這樣做：

```php
$validator = Validator::make($request->all(), [
    'users.*.email' => ['email', 'unique:users'],
    'users.*.first_name' => ['required_with:users.*.last_name'],
]);
```

同樣地，在[語言檔中指定自訂驗證訊息](#custom-messages-for-specific-attributes)時，你可以使用 `*` 字元，這讓你能夠輕鬆地為陣列欄位使用單一驗證訊息：

```php
'custom' => [
    'users.*.email' => [
        'unique' => 'Each user must have a unique email address',
    ]
],
```


<a name="accessing-nested-array-data"></a>
#### 存取巢狀陣列資料

有時在為屬性指派驗證規則時，你可能需要存取特定巢狀陣列元素的值。你可以使用 `Rule::forEach` 方法來達成。`forEach` 方法接受一個 Closure，該 Closure 會在被驗證的陣列屬性每次迭代時被呼叫，並接收該屬性值與明確且完全展開的屬性名稱。該 Closure 應回傳要指派給該陣列元素的規則陣列：

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

在驗證陣列時，你可能希望在應用程式顯示的錯誤訊息中，引用驗證失敗之特定項目的索引或位置。為達成此目的，你可以在[自訂錯誤訊息](#manual-customizing-the-error-messages)中使用 `:index`（從 `0` 開始）、`:position`（從 `1` 開始）或 `:ordinal-position`（從 `1st` 開始）預留位置：

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

在上述範例中，驗證將會失敗，且使用者將看到以下錯誤訊息：_"Please describe photo #2."_

若有需要，你可以透過 `second-index`、`second-position`、`third-index`、`third-position` 等方式引用更深層巢狀的索引與位置。

```php
'photos.*.attributes.*.string' => 'Invalid attribute for photo #:second-position.',
```


<a name="validating-files"></a>
## 驗證檔案

Laravel 提供各種可用於驗證上傳檔案的驗證規則，例如 `mimes`、`image`、`min` 和 `max`。雖然你在驗證檔案時可以自由個別指定這些規則，但 Laravel 也提供了流暢 (Fluent) 的檔案驗證規則建構器，你可能會覺得非常方便：

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

即使你在呼叫 `types` 方法時只需要指定副檔名，該方法實際上是透過讀取檔案內容並猜測其 MIME 類型來驗證檔案的 MIME 類型。MIME 類型及其對應副檔名的完整清單可以在以下位置找到：

[https://svn.apache.org/repos/asf/httpd/httpd/trunk/docs/conf/mime.types](https://svn.apache.org/repos/asf/httpd/httpd/trunk/docs/conf/mime.types)


<a name="validating-files-file-sizes"></a>
#### 驗證檔案大小

為了方便起見，檔案的最大值與最小值可以指定為帶有表示檔案大小單位字尾的字串。支援 `kb`、`mb`、`gb` 及 `tb` 字尾：

```php
File::types(['mp3', 'wav'])
    ->min('1kb')
    ->max('10mb');
```


<a name="validating-files-image-files"></a>
#### 驗證圖檔

若你的應用程式接受使用者上傳圖片，你可以使用 `File` 規則的 `image` 建構子方法，以確保被驗證的檔案是圖片（jpg、jpeg、png、bmp、gif 或 webp）。

此外，`dimensions` 規則可以用來限制圖片的尺寸：

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
> 更多關於驗證圖片尺寸的資訊，可以在 [dimension 規則文件](#rule-dimensions)中找到。

> [!WARNING]
> 預設情況下，出於 XSS 漏洞的可能性，`image` 規則並不允許 SVG 檔案。如果你需要允許 SVG 檔案，可以將 `allowSvg: true` 傳遞給 `image` 規則：`File::image(allowSvg: true)`。


<a name="validating-files-image-dimensions"></a>
#### 驗證圖片尺寸

你也可以驗證圖片的尺寸。例如，要驗證上傳的圖片寬度至少為 1000 像素且高度至少為 500 像素，你可以使用 `dimensions` 規則：

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
> 更多關於驗證圖片尺寸的資訊，可以在 [dimension 規則文件](#rule-dimensions)中找到。

<a name="validating-passwords"></a>
## 驗證密碼

為了確保密碼具備足夠的複雜度，你可以使用 Laravel 的 `Password` 規則物件：

```php
use Illuminate\Support\Facades\Validator;
use Illuminate\Validation\Rules\Password;

$validator = Validator::make($request->all(), [
    'password' => ['required', 'confirmed', Password::min(8)],
]);
```

`Password` 規則物件讓你能輕鬆自訂應用程式的密碼複雜度需求，例如指定密碼至少需要一個字母、數字、符號或是大小寫混合的字元：

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

此外，你還可以使用 `uncompromised` 方法來確保密碼未曾在大眾密碼資料外洩事件中洩露：

```php
Password::min(8)->uncompromised()
```

在內部，`Password` 規則物件使用了 [k-Anonymity](https://en.wikipedia.org/wiki/K-anonymity) 模型，能在不犧牲使用者隱私或安全的情況下，透過 [haveibeenpwned.com](https://haveibeenpwned.com) 服務檢查密碼是否已外洩。

預設情況下，只要密碼在資料外洩中出現過一次，就會被認定為已受干擾或外洩。你可以透過 `uncompromised` 方法的第一個引數來自訂此門檻值：

```php
// Ensure the password appears less than 3 times in the same data leak...
Password::min(8)->uncompromised(3);
```

當然，你可以將上述範例中的所有方法串接在一起：

```php
Password::min(8)
    ->max(256)
    ->letters()
    ->mixedCase()
    ->numbers()
    ->symbols()
    ->uncompromised()
```

你可以使用 `toPasswordRulesString` 方法將 `Password` 規則物件轉換成適合 HTML `passwordrules` 屬性的字串：

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

你可能會發現，在應用程式的單一位置集中指定密碼的預設驗證規則非常方便。你可以使用接收 Closure 的 `Password::defaults` 方法輕鬆達成此目的。傳給 `defaults` 方法的 Closure 應傳回 Password 規則的預設設定。通常，`defaults` 規則應該在應用程式的其中一個服務提供者(Service Providers)的 `boot` 方法中呼叫：

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

接著，當你想將預設規則套用到正在進行驗證的特定密碼時，只需呼叫不帶引數的 `defaults` 方法即可：

```php
'password' => ['required', Password::defaults()],
```

有時，你可能想在預設的密碼驗證規則中額外附加其他的驗證規則。你可以使用 `rules` 方法來完成此操作：

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

Laravel 提供許多實用的驗證規則；然而，您可能希望定義自己專屬的規則。註冊自訂驗證規則的一種方法是使用規則物件。若要產生新的規則物件，您可以使用 `make:rule` Artisan 命令。讓我們使用此命令來產生一個驗證字串是否為大寫的規則。Laravel 會將新規則放在 `app/Rules` 目錄中。如果此目錄不存在，當您執行 Artisan 命令建立規則時，Laravel 會自動建立它：

```shell
php artisan make:rule Uppercase
```

建立規則後，我們就可以開始定義其行為。規則物件包含一個 `validate` 方法。此方法接收屬性名稱、屬性值以及驗證失敗時應呼叫的 Callback，並附上驗證錯誤訊息：

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

定義好規則後，您可以將規則物件的實例與其他驗證規則一起傳入，藉此附加到驗證器：

```php
use App\Rules\Uppercase;

$request->validate([
    'name' => ['required', 'string', new Uppercase],
]);
```

#### 翻譯驗證訊息

除了向 `$fail` Closure 提供字面上的錯誤訊息外，您也可以提供一個[翻譯字串鍵](/docs/{{version}}/localization)，並指示 Laravel 翻譯該錯誤訊息：

```php
if (strtoupper($value) !== $value) {
    $fail('validation.uppercase')->translate();
}
```

如有需要，您可以將預位符替換值與偏好的語言分別作為第一與第二個引數傳遞給 `translate` 方法：

```php
$fail('validation.location')->translate([
    'value' => $this->value,
], 'fr');
```

#### 存取額外資料

若您的自訂驗證規則類別需要存取所有正在進行驗證的其他資料，您的規則類別可以實作 `Illuminate\Contracts\Validation\DataAwareRule` 介面。此介面要求您的類別必須定義一個 `setData` 方法。Laravel 會在進行驗證前，自動呼叫此方法並帶入所有正在驗證的資料：

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

或者，若您的驗證規則需要存取執行驗證的驗證器實例，您可以實作 `ValidatorAwareRule` 介面：

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

若您在整個應用程式中只需要使用一次自訂規則的功能，您可以使用 Closure 來代替規則物件。該 Closure 接收屬性的名稱、屬性的值，以及一個在驗證失敗時應該呼叫的 `$fail` Callback：

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

若要讓自訂規則在屬性為空時也能執行，該規則必須隱式表明該屬性為必填。若要快速產生一個新的隱式規則物件，您可以使用帶有 `--implicit` 選項的 `make:rule` Artisan 命令：

```shell
php artisan make:rule Uppercase --implicit
```

> [!WARNING]
> 「隱式」規則僅 _暗示_ 該屬性為必填。至於它是否真的會判定缺少或為空的屬性無效，則取決於您的設定。