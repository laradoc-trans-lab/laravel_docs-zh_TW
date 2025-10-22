# 驗證

- [簡介](#introduction)
- [驗證快速入門](#validation-quickstart)
    - [定義路由](#quick-defining-the-routes)
    - [建立控制器](#quick-creating-the-controller)
    - [編寫驗證邏輯](#quick-writing-the-validation-logic)
    - [顯示驗證錯誤](#quick-displaying-the-validation-errors)
    - [重新填入表單](#repopulating-forms)
    - [關於選填欄位的注意事項](#a-note-on-optional-fields)
    - [驗證錯誤回應格式](#validation-error-response-format)
- [表單請求驗證](#form-request-validation)
    - [建立表單請求](#creating-form-requests)
    - [授權表單請求](#authorizing-form-requests)
    - [自訂錯誤訊息](#customizing-the-error-messages)
    - [準備用於驗證的輸入](#preparing-input-for-validation)
- [手動建立驗證器](#manually-creating-validators)
    - [自動重新導向](#automatic-redirection)
    - [命名錯誤袋](#named-error-bags)
    - [自訂錯誤訊息](#manual-customizing-the-error-messages)
    - [執行額外驗證](#performing-additional-validation)
- [處理已驗證的輸入](#working-with-validated-input)
- [處理錯誤訊息](#working-with-error-messages)
    - [在語系檔中指定自訂訊息](#specifying-custom-messages-in-language-files)
    - [在語系檔中指定屬性](#specifying-attribute-in-language-files)
    - [在語系檔中指定值](#specifying-values-in-language-files)
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
    - [隱式規則](#implicit-rules)

<a name="introduction"></a>
## 簡介

Laravel 提供了幾種不同的方法來驗證您應用程式的傳入資料。最常見的是使用所有傳入 HTTP 請求上可用的 `validate` 方法。不過，我們也會探討其他驗證方法。

Laravel 包含了各式各樣方便的驗證規則，您可以將其套用到資料上，甚至提供了驗證值在給定資料庫資料表中是否唯一的功能。我們會詳細介紹這些驗證規則，讓您熟悉所有 Laravel 的驗證功能。

<a name="validation-quickstart"></a>
## 驗證快速入門

為了了解 Laravel 強大的驗證功能，我們來看一個完整的範例，示範如何驗證表單並向使用者顯示錯誤訊息。透過閱讀此高層次概述，您將能夠對如何使用 Laravel 驗證傳入的請求資料有一個良好的總體理解：


<a name="quick-defining-the-routes"></a>
### 定義路由

首先，我們假設在 `routes/web.php` 檔案中定義了以下路由：

```php
use App\Http\Controllers\PostController;

Route::get('/post/create', [PostController::class, 'create']);
Route::post('/post', [PostController::class, 'store']);
```

`GET` 路由將顯示一個表單供使用者建立新的部落格文章，而 `POST` 路由則會將新的部落格文章儲存到資料庫中。


<a name="quick-creating-the-controller"></a>
### 建立控制器

接下來，我們來看看一個簡單的控制器，它處理這些路由的傳入請求。我們暫時將 `store` 方法留空：

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

現在，我們準備在 `store` 方法中填入驗證新部落格文章的邏輯。為此，我們將使用 `Illuminate\Http\Request` 物件提供的 `validate` 方法。如果驗證規則通過，您的程式碼將會正常執行；但是，如果驗證失敗，將會拋出 `Illuminate\Validation\ValidationException` 異常，並且會自動將正確的錯誤回應傳回給使用者。

如果在傳統 HTTP 請求期間驗證失敗，將會產生重新導向回應至先前的 URL。如果傳入的請求是 XHR 請求，則會傳回包含驗證錯誤訊息的 [JSON 回應](#validation-error-response-format)。

為了更好地理解 `validate` 方法，我們回到 `store` 方法：

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

如您所見，驗證規則被傳遞給 `validate` 方法。別擔心 —— 所有可用的驗證規則都已 [記錄](#available-validation-rules) 在案。再者，如果驗證失敗，將會自動產生正確的回應。如果驗證通過，我們的控制器將會正常繼續執行。

此外，驗證規則也可以指定為規則陣列，而不是單個以 `|` 分隔的字串：

```php
$validatedData = $request->validate([
    'title' => ['required', 'unique:posts', 'max:255'],
    'body' => ['required'],
]);
```

此外，您可以使用 `validateWithBag` 方法來驗證請求並將任何錯誤訊息儲存在 [命名錯誤袋](#named-error-bags) 中：

```php
$validatedData = $request->validateWithBag('post', [
    'title' => ['required', 'unique:posts', 'max:255'],
    'body' => ['required'],
]);
```


<a name="stopping-on-first-validation-failure"></a>
#### 於第一次驗證失敗時停止

有時您可能希望在第一次驗證失敗後，停止對屬性執行驗證規則。為此，請將 `bail` 規則分配給該屬性：

```php
$request->validate([
    'title' => 'bail|required|unique:posts|max:255',
    'body' => 'required',
]);
```

在此範例中，如果 `unique` 規則在 `title` 屬性上失敗，則不會檢查 `max` 規則。規則將按照它們被賦予的順序進行驗證。


<a name="a-note-on-nested-attributes"></a>
#### 關於巢狀屬性的注意事項

如果傳入的 HTTP 請求包含「巢狀」欄位資料，您可以使用「點」語法在驗證規則中指定這些欄位：

```php
$request->validate([
    'title' => 'required|unique:posts|max:255',
    'author.name' => 'required',
    'author.description' => 'required',
]);
```

另一方面，如果您的欄位名稱包含文字句點，您可以透過使用反斜線跳脫該句點，來明確阻止其被解釋為「點」語法：

```php
$request->validate([
    'title' => 'required|unique:posts|max:255',
    'v1\.0' => 'required',
]);
```

<a name="quick-displaying-the-validation-errors"></a>
### 顯示驗證錯誤

那麼，如果傳入的請求欄位未能通過指定的驗證規則該怎麼辦呢？如前所述，Laravel 會自動將使用者重新導向回他們先前的位置。此外，所有驗證錯誤和 [請求輸入](/docs/{{version}}/requests#retrieving-old-input) 都會自動 [快閃到 Session](/docs/{{version}}/session#flash-data)。

透過 `web` 中介層群組所提供的 `Illuminate\View\Middleware\ShareErrorsFromSession` 中介層，應用程式的所有視圖都會共享一個 `$errors` 變數。當此中介層被套用時，`$errors` 變數將始終在您的視圖中可用，讓您方便地假設 `$errors` 變數總是已定義並且可以安全使用。`$errors` 變數將會是 `Illuminate\Support\MessageBag` 實例。有關如何使用此物件的更多資訊，請 [查看其文件](#working-with-error-messages)。

因此，在我們的範例中，當驗證失敗時，使用者將會被重新導向到我們控制器的 `create` 方法，讓您可以在視圖中顯示錯誤訊息：

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

Laravel 內建的驗證規則各自有一個錯誤訊息，這些訊息位於您應用程式的 `lang/en/validation.php` 檔案中。如果您的應用程式沒有 `lang` 目錄，您可以使用 `lang:publish` Artisan 指令來指示 Laravel 建立它。

在 `lang/en/validation.php` 檔案中，您會找到每個驗證規則的翻譯項目。您可以根據應用程式的需求自由修改這些訊息。

此外，您可以將此檔案複製到另一個語言目錄，以翻譯您應用程式的語言訊息。要了解更多關於 Laravel 語系本地化的資訊，請查看完整的 [語系本地化文件](/docs/{{version}}/localization)。

> [!WARNING]
> 預設情況下，Laravel 應用程式骨架不包含 `lang` 目錄。如果您想自訂 Laravel 的語言檔案，可以透過 `lang:publish` Artisan 指令發布它們。

<a name="quick-xhr-requests-and-validation"></a>
#### XHR 請求與驗證

在此範例中，我們使用傳統表單向應用程式發送資料。然而，許多應用程式從由 JavaScript 驅動的前端接收 XHR 請求。在 XHR 請求期間使用 `validate` 方法時，Laravel 將不會生成重新導向回應。相反地，Laravel 會生成一個 [包含所有驗證錯誤的 JSON 回應](#validation-error-response-format)。此 JSON 回應將會以 422 HTTP 狀態碼發送。

<a name="the-at-error-directive"></a>
#### `@error` 指令

您可以使用 `@error` [Blade](/docs/{{version}}/blade) 指令快速判斷指定屬性是否存在驗證錯誤訊息。在 `@error` 指令中，您可以輸出 `$message` 變數來顯示錯誤訊息：

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

如果您正在使用 [命名錯誤袋](#named-error-bags)，您可以將錯誤袋的名稱作為 `@error` 指令的第二個參數傳入：

```blade
<input ... class="@error('title', 'post') is-invalid @enderror">
```

<a name="repopulating-forms"></a>
### 重新填入表單

當 Laravel 因為驗證錯誤生成重新導向回應時，框架將會自動 [將所有請求輸入快閃到 Session](/docs/{{version}}/session#flash-data)。這樣做是為了方便您在下一個請求中存取輸入，並重新填入使用者嘗試提交的表單。

若要從先前的請求中檢索快閃輸入，請在 `Illuminate\Http\Request` 的實例上呼叫 `old` 方法。`old` 方法將會從 [Session](/docs/{{version}}/session) 中取出先前快閃的輸入資料：

```php
$title = $request->old('title');
```

Laravel 也提供了一個全域 `old` 輔助函式。如果您正在 [Blade 模板](/docs/{{version}}/blade) 中顯示舊輸入，使用 `old` 輔助函式來重新填入表單會更方便。如果指定欄位沒有舊輸入，將會返回 `null`：

```blade
<input type="text" name="title" value="{{ old('title') }}">
```

<a name="a-note-on-optional-fields"></a>
### 關於選填欄位的注意事項

預設情況下，Laravel 會在應用程式的全域中介層堆疊中包含 `TrimStrings` 和 `ConvertEmptyStringsToNull` 中介層。因此，如果您不希望驗證器將 `null` 值視為無效，通常需要將您的「選填」請求欄位標記為 `nullable`。例如：

```php
$request->validate([
    'title' => 'required|unique:posts|max:255',
    'body' => 'required',
    'publish_at' => 'nullable|date',
]);
```

在此範例中，我們指定 `publish_at` 欄位可以是 `null` 或有效的日期表示。如果 `nullable` 修飾符沒有新增到規則定義中，驗證器將會將 `null` 視為無效日期。

<a name="validation-error-response-format"></a>
### 驗證錯誤回應格式

當您的應用程式拋出 `Illuminate\Validation\ValidationException` 例外且傳入的 HTTP 請求預期接收 JSON 回應時，Laravel 會自動為您格式化錯誤訊息並返回 `422 Unprocessable Entity` HTTP 回應。

以下是驗證錯誤的 JSON 回應格式範例。請注意，巢狀錯誤鍵會扁平化為「點」表示法格式：

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

對於更複雜的驗證情境，您可能會希望建立一個「Form Request」。Form Request 是一種自訂的請求類別，它封裝了自己的驗證與授權邏輯。若要建立 Form Request 類別，您可以使用 `make:request` Artisan CLI 指令：

```shell
php artisan make:request StorePostRequest
```

產生的 Form Request 類別將會被放置在 `app/Http/Requests` 目錄中。如果此目錄不存在，它會在您執行 `make:request` 指令時建立。每個由 Laravel 產生的 Form Request 都有兩個方法：`authorize` 與 `rules`。

如您所料，`authorize` 方法負責判斷目前通過身份驗證的使用者是否可以執行該請求所代表的動作，而 `rules` 方法則回傳應該應用於該請求資料的驗證規則：

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
> 您可以在 `rules` 方法的簽名中 Type-Hint 您所需的任何依賴。它們將會透過 Laravel [服務容器](/docs/{{version}}/container)自動解析。

那麼，驗證規則是如何評估的呢？您只需在控制器方法中 Type-Hint 該請求即可。傳入的 Form Request 在控制器方法被呼叫之前就會被驗證，這表示您不需要在控制器中混入任何驗證邏輯：

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

如果驗證失敗，將會產生重新導向回應，將使用者送回他們之前的位置。錯誤也會被暫存到 Session 中，以便顯示。如果請求是 XHR 請求，則會向使用者回傳一個 HTTP 狀態碼為 422 的回應，其中包含驗證錯誤的 [JSON 表示](#validation-error-response-format)。

> [!NOTE]
> 需要為您的 Inertia 驅動的 Laravel 前端新增即時 Form Request 驗證嗎？請參考 [Laravel Precognition](/docs/{{version}}/precognition)。


<a name="performing-additional-validation-on-form-requests"></a>
#### 執行額外驗證

有時候，在初始驗證完成後，您需要執行額外的驗證。您可以使用 Form Request 的 `after` 方法來實現此目的。

`after` 方法應該回傳一個可呼叫 (callable) 或閉包 (closure) 陣列，這些可呼叫或閉包將在驗證完成後被呼叫。給定的可呼叫將接收一個 `Illuminate\Validation\Validator` 實例，以便您在必要時拋出額外的錯誤訊息：

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

如上所述，`after` 方法回傳的陣列也可以包含可呼叫類別 (invokable class)。這些類別的 `__invoke` 方法將會接收一個 `Illuminate\Validation\Validator` 實例：

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
#### 停止於第一個驗證失敗

透過在您的請求類別中新增一個 `stopOnFirstFailure` 屬性，您可以通知驗證器，一旦發生單一驗證失敗，它就應該停止驗證所有屬性：

```php
/**
 * Indicates if the validator should stop on the first rule failure.
 *
 * @var bool
 */
protected $stopOnFirstFailure = true;
```


<a name="customizing-the-redirect-location"></a>
#### 自訂重新導向位置

當 Form Request 驗證失敗時，將會產生重新導向回應，將使用者送回他們之前的位置。然而，您可以自由自訂此行為。為此，請在您的 Form Request 上定義一個 `$redirect` 屬性：

```php
/**
 * The URI that users should be redirected to if validation fails.
 *
 * @var string
 */
protected $redirect = '/dashboard';
```

或者，如果您想將使用者重新導向到一個命名路由 (named route)，您可以改為定義一個 `$redirectRoute` 屬性：

```php
/**
 * The route that users should be redirected to if validation fails.
 *
 * @var string
 */
protected $redirectRoute = 'dashboard';
```


<a name="authorizing-form-requests"></a>
### 授權表單請求

Form Request 類別也包含一個 `authorize` 方法。在這個方法中，您可以判斷通過身份驗證的使用者是否真的有權限更新給定的資源。例如，您可以判斷使用者是否確實擁有他們嘗試更新的部落格留言。您很可能會在這個方法中與您的 [授權閘門 (Gates) 和 Policy](/docs/{{version}}/authorization) 互動：

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

由於所有的 Form Request 都繼承了基礎的 Laravel 請求類別，我們可以使用 `user` 方法來存取目前通過身份驗證的使用者。此外，請注意上面範例中對 `route` 方法的呼叫。這個方法讓您能夠存取所呼叫路由上定義的 URI 參數，例如下面範例中的 `{comment}` 參數：

```php
Route::post('/comment/{comment}');
```

因此，如果您的應用程式利用了 [路由模型綁定 (route model binding)](/docs/{{version}}/routing#route-model-binding) 的優勢，您可以透過將已解析的模型作為請求的屬性來存取，使您的程式碼更為簡潔：

```php
return $this->user()->can('update', $this->comment);
```

如果 `authorize` 方法回傳 `false`，將會自動回傳一個 HTTP 狀態碼為 403 的回應，並且您的控制器方法將不會執行。

如果您打算在應用程式的其他部分處理請求的授權邏輯，您可以完全移除 `authorize` 方法，或者直接回傳 `true`：

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
> 您可以在 `authorize` 方法的簽名中 Type-Hint 您所需的任何依賴。它們將會透過 Laravel [服務容器](/docs/{{version}}/container)自動解析。

<a name="customizing-the-error-messages"></a>
### 自訂錯誤訊息

您可以使用覆寫 `messages` 方法的方式，自訂表單請求使用的錯誤訊息。此方法應該回傳一個屬性/規則配對及其對應錯誤訊息的陣列：

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

Laravel 許多內建驗證規則的錯誤訊息都包含 `:attribute` 佔位符。如果您希望將驗證訊息中的 `:attribute` 佔位符替換為自訂屬性名稱，您可以覆寫 `attributes` 方法來指定這些自訂名稱。此方法應該回傳一個屬性/名稱配對的陣列：

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
### 準備用於驗證的輸入

如果您需要在套用驗證規則之前，準備或淨化請求中的任何資料，您可以使用 `prepareForValidation` 方法：

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

同樣地，如果您需要在驗證完成後正規化任何請求資料，您可以使用 `passedValidation` 方法：

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

如果您不想使用請求上的 `validate` 方法，您可以手動使用 `Validator` [外觀](/docs/{{version}}/facades) 來建立一個驗證器實例。該外觀上的 `make` 方法會產生一個新的驗證器實例：

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

傳遞給 `make` 方法的第一個參數是要驗證的資料。第二個參數是應該應用於該資料的驗證規則陣列。

在判斷請求驗證是否失敗後，您可以使用 `withErrors` 方法將錯誤訊息快閃至 Session。使用此方法時，`$errors` 變數將在重新導向後自動與您的視圖共享，讓您能夠輕鬆地將它們顯示回使用者。`withErrors` 方法接受一個驗證器、一個 `MessageBag` 或一個 PHP `array`。

#### 停止於第一個驗證失敗

`stopOnFirstFailure` 方法會告知驗證器，一旦發生單一驗證失敗，就應該停止驗證所有屬性：

```php
if ($validator->stopOnFirstFailure()->fails()) {
    // ...
}
```

<a name="automatic-redirection"></a>
### 自動重新導向

如果您想手動建立一個驗證器實例，但仍想利用 HTTP 請求 `validate` 方法所提供的自動重新導向功能，您可以在現有的驗證器實例上呼叫 `validate` 方法。如果驗證失敗，使用者將自動被重新導向，或者，如果是 XHR 請求，則會傳回 [包含驗證錯誤的 JSON 回應](#validation-error-response-format)：

```php
Validator::make($request->all(), [
    'title' => 'required|unique:posts|max:255',
    'body' => 'required',
])->validate();
```

如果驗證失敗，您可以使用 `validateWithBag` 方法將錯誤訊息儲存在 [命名錯誤袋](#named-error-bags) 中：

```php
Validator::make($request->all(), [
    'title' => 'required|unique:posts|max:255',
    'body' => 'required',
])->validateWithBag('post');
```

<a name="named-error-bags"></a>
### 命名錯誤袋

如果單一頁面上有不只一個表單，您可能會想命名包含驗證錯誤的 `MessageBag`，以便您可以檢索特定表單的錯誤訊息。為此，請將名稱作為第二個參數傳遞給 `withErrors`：

```php
return redirect('/register')->withErrors($validator, 'login');
```

接著您可以從 `$errors` 變數中存取已命名的 `MessageBag` 實例：

```blade
{{ $errors->login->first('email') }}
```

<a name="manual-customizing-the-error-messages"></a>
### 自訂錯誤訊息

如果需要，您可以提供自訂錯誤訊息，供驗證器實例使用，而非 Laravel 提供的預設錯誤訊息。有幾種方法可以指定自訂訊息。首先，您可以將自訂訊息作為第三個參數傳遞給 `Validator::make` 方法：

```php
$validator = Validator::make($input, $rules, $messages = [
    'required' => 'The :attribute field is required.',
]);
```

在此範例中，`:attribute` 預留位置將被驗證欄位的實際名稱取代。您也可以在驗證訊息中使用其他預留位置。例如：

```php
$messages = [
    'same' => 'The :attribute and :other must match.',
    'size' => 'The :attribute must be exactly :size.',
    'between' => 'The :attribute value :input is not between :min - :max.',
    'in' => 'The :attribute must be one of the following types: :values',
];
```

<a name="specifying-a-custom-message-for-a-given-attribute"></a>
#### 為給定屬性指定自訂訊息

有時您可能希望僅為特定屬性指定自訂錯誤訊息。您可以使用「點」符號來這樣做。首先指定屬性名稱，然後是規則：

```php
$messages = [
    'email.required' => 'We need to know your email address!',
];
```

<a name="specifying-custom-attribute-values"></a>
#### 指定自訂屬性值

許多 Laravel 內建的錯誤訊息都包含一個 `:attribute` 預留位置，該預留位置會被驗證中的欄位或屬性名稱取代。若要自訂用於特定欄位替換這些預留位置的值，您可以將一個自訂屬性陣列作為第四個參數傳遞給 `Validator::make` 方法：

```php
$validator = Validator::make($input, $rules, $messages, [
    'email' => 'email address',
]);
```

<a name="performing-additional-validation"></a>
### 執行額外驗證

有時您需要在初始驗證完成後執行額外驗證。您可以使用驗證器的 `after` 方法來實現此目的。`after` 方法接受一個閉包或一個可呼叫陣列，這些閉包或可呼叫陣列將在驗證完成後被調用。給定的可呼叫將接收一個 `Illuminate\Validation\Validator` 實例，允許您在必要時提出額外錯誤訊息：

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

如前所述，`after` 方法也接受一個可呼叫陣列，如果您的「後驗證」邏輯封裝在可呼叫類別中，這會特別方便，這些類別將透過其 `__invoke` 方法接收一個 `Illuminate\Validation\Validator` 實例：

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

在使用表單請求 (form request) 或手動建立的驗證器實例驗證傳入的請求資料後，您可能希望檢索實際經過驗證的傳入請求資料。這可以透過幾種方式完成。首先，您可以呼叫表單請求或驗證器實例上的 `validated` 方法。此方法會回傳經過驗證的資料陣列：

```php
$validated = $request->validated();

$validated = $validator->validated();
```

或者，您可以呼叫表單請求或驗證器實例上的 `safe` 方法。此方法會回傳 `Illuminate\Support\ValidatedInput` 的實例。此物件會公開 `only`、`except` 和 `all` 方法，以檢索經過驗證資料的子集或整個已驗證資料陣列：

```php
$validated = $request->safe()->only(['name', 'email']);

$validated = $request->safe()->except(['name', 'email']);

$validated = $request->safe()->all();
```

此外，`Illuminate\Support\ValidatedInput` 實例可以像陣列一樣進行疊代和存取：

```php
// Validated data may be iterated...
foreach ($request->safe() as $key => $value) {
    // ...
}

// Validated data may be accessed as an array...
$validated = $request->safe();

$email = $validated['email'];
```

如果您想向已驗證的資料新增額外欄位，您可以呼叫 `merge` 方法：

```php
$validated = $request->safe()->merge(['name' => 'Taylor Otwell']);
```

如果您想將已驗證的資料檢索為 [collection](/docs/{{version}}/collections) 實例，您可以呼叫 `collect` 方法：

```php
$collection = $request->safe()->collect();
```


<a name="working-with-error-messages"></a>
## 處理錯誤訊息

在呼叫 `Validator` 實例上的 `errors` 方法後，您將收到一個 `Illuminate\Support\MessageBag` 實例，它具有多種處理錯誤訊息的便利方法。所有視圖自動可用的 `$errors` 變數也是 `MessageBag` 類別的一個實例。


<a name="retrieving-the-first-error-message-for-a-field"></a>
#### 檢索欄位的第一個錯誤訊息

要檢索給定欄位的第一個錯誤訊息，請使用 `first` 方法：

```php
$errors = $validator->errors();

echo $errors->first('email');
```


<a name="retrieving-all-error-messages-for-a-field"></a>
#### 檢索欄位的所有錯誤訊息

如果您需要檢索給定欄位的所有訊息陣列，請使用 `get` 方法：

```php
foreach ($errors->get('email') as $message) {
    // ...
}
```

如果您正在驗證陣列表單欄位，您可以使用 `*` 字元檢索每個陣列元素的所有訊息：

```php
foreach ($errors->get('attachments.*') as $message) {
    // ...
}
```


<a name="retrieving-all-error-messages-for-all-fields"></a>
#### 檢索所有欄位的所有錯誤訊息

要檢索所有欄位的所有訊息陣列，請使用 `all` 方法：

```php
foreach ($errors->all() as $message) {
    // ...
}
```


<a name="determining-if-messages-exist-for-a-field"></a>
#### 判斷欄位是否存在訊息

`has` 方法可用於判斷給定欄位是否存在任何錯誤訊息：

```php
if ($errors->has('email')) {
    // ...
}
```


<a name="specifying-custom-messages-in-language-files"></a>
### 在語系檔中指定自訂訊息

Laravel 內建的每個驗證規則都有一個錯誤訊息，位於應用程式的 `lang/en/validation.php` 檔案中。如果您的應用程式沒有 `lang` 目錄，您可以指示 Laravel 使用 `lang:publish` Artisan 指令建立它。

在 `lang/en/validation.php` 檔案中，您會找到每個驗證規則的翻譯項目。您可以根據應用程式的需求自由變更或修改這些訊息。

此外，您可以將此檔案複製到另一個語系目錄，以翻譯應用程式語系的訊息。要了解有關 Laravel 本地化的更多資訊，請查閱完整的 [本地化文檔](/docs/{{version}}/localization)。

> [!WARNING]
> 依預設，Laravel 應用程式骨架不包含 `lang` 目錄。如果您想自訂 Laravel 的語系檔，可以透過 `lang:publish` Artisan 指令發佈它們。


<a name="custom-messages-for-specific-attributes"></a>
#### 特定屬性的自訂訊息

您可以自訂應用程式驗證語系檔中用於指定屬性與規則組合的錯誤訊息。為此，請將您的訊息自訂新增到應用程式 `lang/xx/validation.php` 語系檔的 `custom` 陣列中：

```php
'custom' => [
    'email' => [
        'required' => 'We need to know your email address!',
        'max' => 'Your email address is too long!'
    ],
],
```


<a name="specifying-attribute-in-language-files"></a>
### 在語系檔中指定屬性

Laravel 許多內建的錯誤訊息都包含一個 `:attribute` 佔位符，該佔位符會替換為驗證中欄位或屬性的名稱。如果您希望驗證訊息的 `:attribute` 部分被自訂值替換，您可以在 `lang/xx/validation.php` 語系檔的 `attributes` 陣列中指定自訂屬性名稱：

```php
'attributes' => [
    'email' => 'email address',
],
```

> [!WARNING]
> 依預設，Laravel 應用程式骨架不包含 `lang` 目錄。如果您想自訂 Laravel 的語系檔，可以透過 `lang:publish` Artisan 指令發佈它們。


<a name="specifying-values-in-language-files"></a>
### 在語系檔中指定值

Laravel 某些內建驗證規則的錯誤訊息包含一個 `:value` 佔位符，該佔位符會替換為請求屬性的目前值。然而，您有時可能需要將驗證訊息的 `:value` 部分替換為該值的自訂表示。例如，考慮以下規則，該規則指定如果 `payment_type` 的值為 `cc`，則需要信用卡號碼：

```php
Validator::make($request->all(), [
    'credit_card_number' => 'required_if:payment_type,cc'
]);
```

如果此驗證規則失敗，它將產生以下錯誤訊息：

```text
The credit card number field is required when payment type is cc.
```

您可以不是顯示 `cc` 作為付款類型值，而是在 `lang/xx/validation.php` 語系檔中透過定義 `values` 陣列來指定更友善的使用者值表示：

```php
'values' => [
    'payment_type' => [
        'cc' => 'credit card'
    ],
],
```

> [!WARNING]
> 依預設，Laravel 應用程式骨架不包含 `lang` 目錄。如果您想自訂 Laravel 的語系檔，可以透過 `lang:publish` Artisan 指令發佈它們。

定義此值後，驗證規則將產生以下錯誤訊息：

```text
The credit card number field is required when payment type is credit card.
```

<a name="available-validation-rules"></a>
## 可用的驗證規則

以下列出所有可用的驗證規則及其功能：

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

[接受](#rule-accepted)
[接受 If](#rule-accepted-if)
[布林值](#rule-boolean)
[拒絕](#rule-declined)
[拒絕 If](#rule-declined-if)

</div>


#### 字串

<div class="collection-method-list" markdown="1">

[有效網址](#rule-active-url)
[英文字母](#rule-alpha)
[英數字與連字號](#rule-alpha-dash)
[英數字](#rule-alpha-num)
[ASCII](#rule-ascii)
[確認](#rule-confirmed)
[目前密碼](#rule-current-password)
[不同](#rule-different)
[不以...開頭](#rule-doesnt-start-with)
[不以...結尾](#rule-doesnt-end-with)
[電子郵件](#rule-email)
[以...結尾](#rule-ends-with)
[列舉](#rule-enum)
[十六進位顏色](#rule-hex-color)
[包含於](#rule-in)
[IP 位址](#rule-ip)
[JSON](#rule-json)
[小寫](#rule-lowercase)
[MAC 位址](#rule-mac)
[最大值](#rule-max)
[最小值](#rule-min)
[不包含於](#rule-not-in)
[正規表達式](#rule-regex)
[非正規表達式](#rule-not-regex)
[相同](#rule-same)
[大小](#rule-size)
[以...開頭](#rule-starts-with)
[字串](#rule-string)
[大寫](#rule-uppercase)
[網址](#rule-url)
[ULID](#rule-ulid)
[UUID](#rule-uuid)

</div>


#### 數字

<div class="collection-method-list" markdown="1">

[介於](#rule-between)
[小數](#rule-decimal)
[不同](#rule-different)
[位數](#rule-digits)
[介於...位數之間](#rule-digits-between)
[大於](#rule-gt)
[大於或等於](#rule-gte)
[整數](#rule-integer)
[小於](#rule-lt)
[小於或等於](#rule-lte)
[最大值](#rule-max)
[最大位數](#rule-max-digits)
[最小值](#rule-min)
[最小位數](#rule-min-digits)
[倍數](#rule-multiple-of)
[數值](#rule-numeric)
[相同](#rule-same)
[大小](#rule-size)

</div>


#### 陣列

<div class="collection-method-list" markdown="1">

[陣列](#rule-array)
[介於](#rule-between)
[包含](#rule-contains)
[不包含](#rule-doesnt-contain)
[唯一](#rule-distinct)
[在陣列中](#rule-in-array)
[在陣列鍵中](#rule-in-array-keys)
[列表](#rule-list)
[最大值](#rule-max)
[最小值](#rule-min)
[大小](#rule-size)

</div>


#### 日期

<div class="collection-method-list" markdown="1">

[之後](#rule-after)
[之後或等於](#rule-after-or-equal)
[之前](#rule-before)
[之前或等於](#rule-before-or-equal)
[日期](#rule-date)
[日期等於](#rule-date-equals)
[日期格式](#rule-date-format)
[不同](#rule-different)
[時區](#rule-timezone)

</div>


#### 檔案

<div class="collection-method-list" markdown="1">

[介於](#rule-between)
[尺寸](#rule-dimensions)
[副檔名](#rule-extensions)
[檔案](#rule-file)
[圖片](#rule-image)
[最大值](#rule-max)
[MIME 類型](#rule-mimetypes)
[MIME 類型 (依副檔名)](#rule-mimes)
[大小](#rule-size)

</div>


#### 資料庫

<div class="collection-method-list" markdown="1">

[存在](#rule-exists)
[唯一](#rule-unique)

</div>


#### 公用程式

<div class="collection-method-list" markdown="1">

[任意一項](#rule-anyof)
[立即失敗](#rule-bail)
[排除](#rule-exclude)
[排除 If](#rule-exclude-if)
[排除 Unless](#rule-exclude-unless)
[排除 With](#rule-exclude-with)
[排除 Without](#rule-exclude-without)
[已填寫](#rule-filled)
[遺失](#rule-missing)
[遺失 If](#rule-missing-if)
[遺失 Unless](#rule-missing-unless)
[遺失 With](#rule-missing-with)
[遺失 With All](#rule-missing-with-all)
[可為空](#rule-nullable)
[存在](#rule-present)
[存在 If](#rule-present-if)
[存在 Unless](#rule-present-unless)
[存在 With](#rule-present-with)
[存在 With All](#rule-present-with-all)
[禁止](#rule-prohibited)
[禁止 If](#rule-prohibited-if)
[禁止 If Accepted](#rule-prohibited-if-accepted)
[禁止 If Declined](#rule-prohibited-if-declined)
[禁止 Unless](#rule-prohibited-unless)
[禁止 (Prohibits)](#rule-prohibits)
[必填](#rule-required)
[必填 If](#rule-required-if)
[必填 If Accepted](#rule-required-if-accepted)
[必填 If Declined](#rule-required-if-declined)
[必填 Unless](#rule-required-unless)
[必填 With](#rule-required-with)
[必填 With All](#rule-required-with-all)
[必填 Without](#rule-required-without)
[必填 Without All](#rule-required-without-all)
[必填陣列鍵](#rule-required-array-keys)
[有時](#validating-when-present)

</div>


<a name="rule-accepted"></a>
#### accepted

待驗證欄位必須是 `"yes"`、`"on"`、`1`、`"1"`、`true` 或 `"true"`。這對於驗證「服務條款」接受或類似欄位很有用。


<a name="rule-accepted-if"></a>
#### accepted_if:anotherfield,value,...

如果另一個待驗證欄位等於指定值，則待驗證欄位必須是 `"yes"`、`"on"`、`1`、`"1"`、`true` 或 `"true"`。這對於驗證「服務條款」接受或類似欄位很有用。


<a name="rule-active-url"></a>
#### active_url

根據 PHP 函式 `dns_get_record`，待驗證欄位必須具有有效的 A 或 AAAA 記錄。提供的 URL 的主機名將使用 PHP 函式 `parse_url` 提取，然後再傳遞給 `dns_get_record`。


<a name="rule-after"></a>
#### after:_date_

待驗證欄位的值必須晚於給定日期。日期將傳遞給 PHP 函式 `strtotime` 以轉換為有效的 `DateTime` 實例：

```php
'start_date' => 'required|date|after:tomorrow'
```

您也可以指定另一個欄位來與日期進行比較，而不是傳遞由 `strtotime` 評估的日期字串：

```php
'finish_date' => 'required|date|after:start_date'
```

為方便起見，日期規則可以使用流暢的 `date` 規則建立器來建構：

```php
use Illuminate\Validation\Rule;

'start_date' => [
    'required',
    Rule::date()->after(today()->addDays(7)),
],
```

`afterToday` 和 `todayOrAfter` 方法可用於流暢地表達日期必須在今天之後，或在今天之後或等於今天：

```php
'start_date' => [
    'required',
    Rule::date()->afterToday(),
],
```


<a name="rule-after-or-equal"></a>
#### after\_or\_equal:_date_

待驗證欄位的值必須晚於或等於給定日期。更多資訊請參閱 [after](#rule-after) 規則。

為方便起見，日期規則可以使用流暢的 `date` 規則建立器來建構：

```php
use Illuminate\Validation\Rule;

'start_date' => [
    'required',
    Rule::date()->afterOrEqual(today()->addDays(7)),
],
```


<a name="rule-anyof"></a>
#### anyOf

`Rule::anyOf` 驗證規則允許您指定待驗證欄位必須滿足給定驗證規則集中的任何一個。例如，以下規則將驗證 `username` 欄位是電子郵件位址，或是一個至少包含 6 個字元的英數字串 (包含連字號)：

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

待驗證欄位必須完全由 [\p{L}](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AL%3A%5D&g=&i=) 和 [\p{M}](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AM%3A%5D&g=&i=) 中包含的 Unicode 字母字元組成。

若要將此驗證規則限制為 ASCII 範圍 (`a-z` 和 `A-Z`) 中的字元，您可以為驗證規則提供 `ascii` 選項：

```php
'username' => 'alpha:ascii',
```

<a name="rule-alpha-dash"></a>
#### alpha_dash

驗證中的欄位必須完全是 Unicode 英數字元，包含 [\p{L}](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AL%3A%5D&g=&i=)、[\p{M}](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AM%3A%5D&g=&i=)、[\p{N}](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AN%3A%5D&g=&i=)，以及 ASCII 破折號 (`-`) 和 ASCII 底線 (`_`)。

若要將此驗證規則限制在 ASCII 範圍 (`a-z`、`A-Z` 和 `0-9`) 中的字元，您可以為此驗證規則提供 `ascii` 選項：

```php
'username' => 'alpha_dash:ascii',
```


<a name="rule-alpha-num"></a>
#### alpha_num

驗證中的欄位必須完全是 Unicode 英數字元，包含 [\p{L}](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AL%3A%5D&g=&i=)、[\p{M}](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AM%3A%5D&g=&i=) 和 [\p{N}](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AN%3A%5D&g=&i=)。

若要將此驗證規則限制在 ASCII 範圍 (`a-z`、`A-Z` 和 `0-9`) 中的字元，您可以為此驗證規則提供 `ascii` 選項：

```php
'username' => 'alpha_num:ascii',
```


<a name="rule-array"></a>
#### array

驗證中的欄位必須是一個 PHP `array`。

當提供額外的值給 `array` 規則時，輸入陣列中的每個鍵都必須存在於規則提供的值清單中。在以下範例中，輸入陣列中的 `admin` 鍵是無效的，因為它不包含在 `array` 規則提供的值清單中：

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

一般來說，您應該始終指定允許存在於陣列中的鍵。


<a name="rule-ascii"></a>
#### ascii

驗證中的欄位必須完全是 7 位元的 ASCII 字元。


<a name="rule-bail"></a>
#### bail

在第一個驗證失敗後，停止對該欄位執行驗證規則。

儘管 `bail` 規則只會在遇到驗證失敗時停止驗證特定欄位，但 `stopOnFirstFailure` 方法會告知驗證器，一旦發生單一驗證失敗，它應該停止驗證所有屬性：

```php
if ($validator->stopOnFirstFailure()->fails()) {
    // ...
}
```


<a name="rule-before"></a>
#### before:_date_

驗證中的欄位必須是指定日期之前的值。這些日期將會傳入 PHP `strtotime` 函數，以便轉換成有效的 `DateTime` 實例。此外，與 [after](#rule-after) 規則一樣，也可以提供另一個驗證中欄位的名稱作為 `date` 的值。

為了方便，日期型規則也可以使用流暢的 `date` 規則建立器來建構：

```php
use Illuminate\Validation\Rule;

'start_date' => [
    'required',
    Rule::date()->before(today()->subDays(7)),
],
```

`beforeToday` 和 `todayOrBefore` 方法可以用來流暢地表達日期，並且必須分別為今天之前、或今天或今天之前：

```php
'start_date' => [
    'required',
    Rule::date()->beforeToday(),
],
```


<a name="rule-before-or-equal"></a>
#### before\_or\_equal:_date_

驗證中的欄位必須是指定日期之前或等於該日期的值。這些日期將會傳入 PHP `strtotime` 函數，以便轉換成有效的 `DateTime` 實例。此外，與 [after](#rule-after) 規則一樣，也可以提供另一個驗證中欄位的名稱作為 `date` 的值。

為了方便，日期型規則也可以使用流暢的 `date` 規則建立器來建構：

```php
use Illuminate\Validation\Rule;

'start_date' => [
    'required',
    Rule::date()->beforeOrEqual(today()->subDays(7)),
],
```


<a name="rule-between"></a>
#### between:_min_,_max_

驗證中的欄位大小必須在指定的 _min_ 與 _max_ (含) 之間。字串、數字、陣列和檔案的評估方式與 [size](#rule-size) 規則相同。


<a name="rule-boolean"></a>
#### boolean

驗證中的欄位必須能夠被轉型為布林值。接受的輸入為 `true`、`false`、`1`、`0`、`"1"` 和 `"0"`。

您可以使用 `strict` 參數，僅在欄位值為 `true` 或 `false` 時才視為有效：

```php
'foo' => 'boolean:strict'
```


<a name="rule-confirmed"></a>
#### confirmed

驗證中的欄位必須有一個名稱為 `{field}_confirmation` 的匹配欄位。例如，如果驗證中的欄位是 `password`，則輸入中必須存在一個匹配的 `password_confirmation` 欄位。

您也可以傳遞自訂的確認欄位名稱。例如，`confirmed:repeat_username` 將會要求 `repeat_username` 欄位與驗證中的欄位匹配。


<a name="rule-contains"></a>
#### contains:_foo_,_bar_,...

驗證中的欄位必須是一個包含所有指定參數值的陣列。由於此規則通常需要您 `implode` 一個陣列，因此可以使用 `Rule::contains` 方法來流暢地建構規則：

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

驗證中的欄位必須是一個不包含任何指定參數值的陣列。由於此規則通常需要您 `implode` 一個陣列，因此可以使用 `Rule::doesntContain` 方法來流暢地建構規則：

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

驗證中的欄位必須與已驗證使用者的密碼匹配。您可以使用規則的第一個參數來指定 [身份驗證守衛](/docs/{{version}}/authentication)：

```php
'password' => 'current_password:api'
```


<a name="rule-date"></a>
#### date

驗證中的欄位必須是根據 PHP `strtotime` 函數的有效非相對日期。


<a name="rule-date-equals"></a>
#### date_equals:_date_

驗證中的欄位必須等於指定的日期。這些日期將會傳入 PHP `strtotime` 函數，以便轉換成有效的 `DateTime` 實例。


<a name="rule-date-format"></a>
#### date_format:_format_,...

驗證中的欄位必須符合指定的其中一個 _formats_。在驗證欄位時，您應該使用 `date` **或** `date_format`，而不是兩者都用。此驗證規則支援 PHP [DateTime](https://www.php.net/manual/en/class.datetime.php) 類別所支援的所有格式。

為了方便，日期型規則也可以使用流暢的 `date` 規則建立器來建構：

```php
use Illuminate\Validation\Rule;

'start_date' => [
    'required',
    Rule::date()->format('Y-m-d'),
],
```


<a name="rule-decimal"></a>
#### decimal:_min_,_max_

驗證中的欄位必須是數字，並且必須包含指定的小數位數：

```php
// Must have exactly two decimal places (9.99)...
'price' => 'decimal:2'

// Must have between 2 and 4 decimal places...
'price' => 'decimal:2,4'
```


<a name="rule-declined"></a>
#### declined

驗證中的欄位必須是 `"no"`、`"off"`、`0`、`"0"`、`false` 或 `"false"`。


<a name="rule-declined-if"></a>
#### declined_if:anotherfield,value,...

如果另一個驗證中的欄位等於指定的值，則驗證中的欄位必須是 `"no"`、`"off"`、`0`、`"0"`、`false` 或 `"false"`。


<a name="rule-different"></a>
#### different:_field_

驗證中的欄位必須與 _field_ 有不同的值。


<a name="rule-digits"></a>
#### digits:_value_

驗證中的整數必須具有 _value_ 的確切長度。

<a name="rule-digits-between"></a>
#### digits_between:_min_,_max_

正在驗證的整數長度必須介於給定的 _min_ 與 _max_ 之間。


<a name="rule-dimensions"></a>
#### dimensions

正在驗證的檔案必須是一個符合規則參數指定尺寸限制的圖片：

```php
'avatar' => 'dimensions:min_width=100,min_height=200'
```

可用的限制有：_min\_width_、_max\_width_、_min\_height_、_max\_height_、_width_、_height_、_ratio_。

_比例_限制應表示為寬度除以高度。這可以透過分數表示，例如 `3/2`，或浮點數表示，例如 `1.5`：

```php
'avatar' => 'dimensions:ratio=3/2'
```

由於此規則需要多個參數，因此通常更方便使用 `Rule::dimensions` 方法來流暢地建構規則：

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

在驗證陣列時，正在驗證的欄位不能有重複的值：

```php
'foo.*.id' => 'distinct'
```

Distinct 預設使用寬鬆的變數比較。若要使用嚴格比較，您可以在驗證規則定義中添加 `strict` 參數：

```php
'foo.*.id' => 'distinct:strict'
```

您可以將 `ignore_case` 添加到驗證規則的參數中，以使規則忽略大小寫差異：

```php
'foo.*.id' => 'distinct:ignore_case'
```


<a name="rule-doesnt-start-with"></a>
#### doesnt_start_with:_foo_,_bar_,...

正在驗證的欄位不能以給定的值之一開頭。


<a name="rule-doesnt-end-with"></a>
#### doesnt_end_with:_foo_,_bar_,...

正在驗證的欄位不能以給定的值之一結尾。


<a name="rule-email"></a>
#### email

正在驗證的欄位必須是電子郵件地址格式。此驗證規則使用 [egulias/email-validator](https://github.com/egulias/EmailValidator) 套件來驗證電子郵件地址。預設情況下，會套用 `RFCValidation` 驗證器，但您也可以套用其他驗證樣式：

```php
'email' => 'email:rfc,dns'
```

上面的範例將套用 `RFCValidation` 和 `DNSCheckValidation` 驗證。以下是您可以套用的完整驗證樣式列表：

<div class="content-list" markdown="1">

- `rfc`: `RFCValidation` - 根據 [支援的 RFCs](https://github.com/egulias/EmailValidator?tab=readme-ov-file#supported-rfcs) 驗證電子郵件地址。
- `strict`: `NoRFCWarningsValidation` - 根據 [支援的 RFCs](https://github.com/egulias/EmailValidator?tab=readme-ov-file#supported-rfcs) 驗證電子郵件，若發現警告 (例如，尾隨句點和多個連續句點) 則驗證失敗。
- `dns`: `DNSCheckValidation` - 確保電子郵件地址的網域具有有效的 MX 記錄。
- `spoof`: `SpoofCheckValidation` - 確保電子郵件地址不包含同形字或欺騙性 Unicode 字元。
- `filter`: `FilterEmailValidation` - 確保電子郵件地址根據 PHP 的 `filter_var` 函式是有效的。
- `filter_unicode`: `FilterEmailValidation::unicode()` - 確保電子郵件地址根據 PHP 的 `filter_var` 函式是有效的，並允許某些 Unicode 字元。

</div>

為方便起見，電子郵件驗證規則可以使用流暢的規則建構器建立：

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


<a name="rule-ends-with"></a>
#### ends_with:_foo_,_bar_,...

正在驗證的欄位必須以給定的值之一結尾。


<a name="rule-enum"></a>
#### enum

`Enum` 規則是一個基於類別的規則，它驗證正在驗證的欄位是否包含有效的 enum 值。`Enum` 規則將 enum 的名稱作為其唯一的建構子參數。在驗證原始值時，應向 `Enum` 規則提供一個支援的 Enum：

```php
use App\Enums\ServerStatus;
use Illuminate\Validation\Rule;

$request->validate([
    'status' => [Rule::enum(ServerStatus::class)],
]);
```

`Enum` 規則的 `only` 和 `except` 方法可用來限制哪些 enum 情況應被視為有效：

```php
Rule::enum(ServerStatus::class)
    ->only([ServerStatus::Pending, ServerStatus::Active]);

Rule::enum(ServerStatus::class)
    ->except([ServerStatus::Pending, ServerStatus::Active]);
```

`when` 方法可用來有條件地修改 `Enum` 規則：

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

正在驗證的欄位將從 `validate` 和 `validated` 方法返回的請求資料中排除。


<a name="rule-exclude-if"></a>
#### exclude_if:_anotherfield_,_value_

如果 _anotherfield_ 欄位等於 _value_，則正在驗證的欄位將從 `validate` 和 `validated` 方法返回的請求資料中排除。

如果需要複雜的條件排除邏輯，您可以使用 `Rule::excludeIf` 方法。此方法接受布林值或閉包。當給予閉包時，閉包應返回 `true` 或 `false`，以指示是否應排除正在驗證的欄位：

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

正在驗證的欄位將從 `validate` 和 `validated` 方法返回的請求資料中排除，除非 _anotherfield_ 欄位等於 _value_。如果 _value_ 為 `null` (`exclude_unless:name,null`)，則正在驗證的欄位將被排除，除非比較欄位為 `null` 或比較欄位在請求資料中缺失。


<a name="rule-exclude-with"></a>
#### exclude_with:_anotherfield_

如果 _anotherfield_ 欄位存在，則正在驗證的欄位將從 `validate` 和 `validated` 方法返回的請求資料中排除。


<a name="rule-exclude-without"></a>
#### exclude_without:_anotherfield_

如果 _anotherfield_ 欄位不存在，則正在驗證的欄位將從 `validate` 和 `validated` 方法返回的請求資料中排除。


<a name="rule-exists"></a>
#### exists:_table_,_column_

正在驗證的欄位必須存在於給定的資料庫表格中。


<a name="basic-usage-of-exists-rule"></a>
#### exists 規則的基本用法

```php
'state' => 'exists:states'
```

如果未指定 `column` 選項，將使用欄位名稱。因此，在此情況下，規則將驗證 `states` 資料庫表格是否包含具有與請求的 `state` 屬性值相符的 `state` 欄位值的記錄。

<a name="specifying-a-custom-column-name"></a>
#### 指定自訂欄位名稱

您可以透過在資料庫表格名稱後方放置驗證規則應使用的資料庫欄位名稱來明確指定：

```php
'state' => 'exists:states,abbreviation'
```

有時，您可能需要指定用於 `exists` 查詢的特定資料庫連線。您可以透過將連線名稱前置到表格名稱來實現：

```php
'email' => 'exists:connection.staff,email'
```

您可以不直接指定表格名稱，而是指定應該用於確定表格名稱的 Eloquent 模型：

```php
'user_id' => 'exists:App\Models\User,id'
```

如果您想自訂驗證規則執行的查詢，您可以使用 `Rule` 類別流暢地定義規則。在此範例中，我們還將驗證規則指定為陣列，而不是使用 `|` 字元來分隔它們：

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

您可以透過將欄位名稱作為第二個引數傳遞給 `exists` 方法，來明確指定由 `Rule::exists` 方法生成的 `exists` 規則應使用的資料庫欄位名稱：

```php
'state' => Rule::exists('states', 'abbreviation'),
```

有時，您可能希望驗證資料庫中是否存在一組值。您可以透過將 `exists` 和 [array](#rule-array) 規則同時添加到要驗證的欄位來實現：

```php
'states' => ['array', Rule::exists('states', 'abbreviation')],
```

當這兩個規則都分配給一個欄位時，Laravel 將自動建立一個單一查詢來判斷所有給定值是否存在於指定表格中。


<a name="rule-extensions"></a>
#### extensions:_foo_,_bar_,...

待驗證的檔案必須具有與所列擴充功能之一相對應的使用者指定擴充功能：

```php
'photo' => ['required', 'extensions:jpg,png'],
```

> [!WARNING]
> 您不應僅憑使用者指定的副檔名來驗證檔案。此規則通常應與 [mimes](#rule-mimes) 或 [mimetypes](#rule-mimetypes) 規則結合使用。


<a name="rule-file"></a>
#### file

待驗證的欄位必須是一個成功上傳的檔案。


<a name="rule-filled"></a>
#### filled

待驗證的欄位存在時，不得為空。


<a name="rule-gt"></a>
#### gt:_field_

待驗證的欄位必須大於給定的 _field_ 或 _value_。這兩個欄位必須是相同的類型。字串、數字、陣列和檔案的評估方式與 [size](#rule-size) 規則相同。


<a name="rule-gte"></a>
#### gte:_field_

待驗證的欄位必須大於或等於給定的 _field_ 或 _value_。這兩個欄位必須是相同的類型。字串、數字、陣列和檔案的評估方式與 [size](#rule-size) 規則相同。


<a name="rule-hex-color"></a>
#### hex_color

待驗證的欄位必須包含有效的 [十六進位](https://developer.mozilla.org/en-US/docs/Web/CSS/hex-color) 顏色值。


<a name="rule-image"></a>
#### image

待驗證的檔案必須是圖片 (jpg, jpeg, png, bmp, gif 或 webp)。

> [!WARNING]
> 預設情況下，image 規則不允許 SVG 檔案，因為存在 XSS 漏洞的可能性。如果您需要允許 SVG 檔案，您可以將 `allow_svg` 指令提供給 `image` 規則 (`image:allow_svg`)。


<a name="rule-in"></a>
#### in:_foo_,_bar_,...

待驗證的欄位必須包含在給定的值清單中。由於此規則通常需要您 `implode` 一個陣列，因此可以使用 `Rule::in` 方法來流暢地建構規則：

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

當 `in` 規則與 `array` 規則結合使用時，輸入陣列中的每個值都必須存在於提供給 `in` 規則的值清單中。在以下範例中，輸入陣列中的 `LAS` 機場代碼無效，因為它不包含在提供給 `in` 規則的機場清單中：

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

待驗證的欄位必須存在於 _anotherfield_ 的值中。


<a name="rule-in-array-keys"></a>
#### in_array_keys:_value_.*

待驗證的欄位必須是一個陣列，並且該陣列中至少有一個鍵是給定的 _values_ 之一：

```php
'config' => 'array|in_array_keys:timezone'
```


<a name="rule-integer"></a>
#### integer

待驗證的欄位必須是一個整數。

您可以使用 `strict` 參數，僅當欄位類型為 `integer` 時才將其視為有效。帶有整數值的字串將被視為無效：

```php
'age' => 'integer:strict'
```

> [!WARNING]
> 此驗證規則不會驗證輸入是否為「整數」變數類型，僅驗證輸入是否為 PHP 的 `FILTER_VALIDATE_INT` 規則所接受的類型。如果您需要驗證輸入是否為數字，請將此規則與 [numeric 驗證規則](#rule-numeric) 結合使用。


<a name="rule-ip"></a>
#### ip

待驗證的欄位必須是 IP 位址。


<a name="ipv4"></a>
#### ipv4

待驗證的欄位必須是 IPv4 位址。


<a name="ipv6"></a>
#### ipv6

待驗證的欄位必須是 IPv6 位址。


<a name="rule-json"></a>
#### json

待驗證的欄位必須是有效的 JSON 字串。


<a name="rule-lt"></a>
#### lt:_field_

待驗證的欄位必須小於給定的 _field_。這兩個欄位必須是相同的類型。字串、數字、陣列和檔案的評估方式與 [size](#rule-size) 規則相同。


<a name="rule-lte"></a>
#### lte:_field_

待驗證的欄位必須小於或等於給定的 _field_。這兩個欄位必須是相同的類型。字串、數字、陣列和檔案的評估方式與 [size](#rule-size) 規則相同。


<a name="rule-lowercase"></a>
#### lowercase

待驗證的欄位必須是小寫。


<a name="rule-list"></a>
#### list

待驗證的欄位必須是一個列表形式的陣列。如果陣列的鍵由從 0 到 `count($array) - 1` 的連續數字組成，則該陣列被視為列表。


<a name="rule-mac"></a>
#### mac_address

待驗證的欄位必須是 MAC 位址。


<a name="rule-max"></a>
#### max:_value_

待驗證的欄位必須小於或等於最大 _value_。字串、數字、陣列和檔案的評估方式與 [size](#rule-size) 規則相同。


<a name="rule-max-digits"></a>
#### max_digits:_value_

待驗證的整數必須具有最大長度 _value_。


<a name="rule-mimetypes"></a>
#### mimetypes:_text/plain_,...

待驗證的檔案必須符合給定的 MIME 類型之一：

```php
'video' => 'mimetypes:video/avi,video/mpeg,video/quicktime'
```

為了確定上傳檔案的 MIME 類型，將會讀取檔案內容，框架將嘗試猜測 MIME 類型，這可能與客戶端提供的 MIME 類型不同。

<a name="rule-mimes"></a>
#### mimes:_foo_,_bar_,...

正在驗證的檔案必須具有與所列副檔名之一對應的 MIME 類型：

```php
'photo' => 'mimes:jpg,bmp,png'
```

即使您只需要指定副檔名，此規則實際上會透過讀取檔案內容並猜測其 MIME 類型來驗證檔案的 MIME 類型。MIME 類型及其對應副檔名的完整列表可在以下位置找到：

[https://svn.apache.org/repos/asf/httpd/httpd/trunk/docs/conf/mime.types](https://svn.apache.org/repos/asf/httpd/httpd/trunk/docs/conf/mime.types)

<a name="mime-types-and-extensions"></a>
#### MIME Types and Extensions

此驗證規則不會驗證 MIME 類型與使用者指派給檔案的副檔名之間的一致性。例如，`mimes:png` 驗證規則會將包含有效 PNG 內容的檔案視為有效的 PNG 圖片，即使該檔案命名為 `photo.txt`。如果您想驗證檔案的使用者指派副檔名，您可以使用 [extensions](#rule-extensions) 規則。

<a name="rule-min"></a>
#### min:_value_

正在驗證的欄位必須具有最小的 _值_。字串、數字、陣列和檔案的評估方式與 [size](#rule-size) 規則相同。

<a name="rule-min-digits"></a>
#### min_digits:_value_

正在驗證的整數必須具有最小長度為 _值_。

<a name="rule-multiple-of"></a>
#### multiple_of:_value_

正在驗證的欄位必須是 _值_ 的倍數。

<a name="rule-missing"></a>
#### missing

正在驗證的欄位不得出現在輸入資料中。

<a name="rule-missing-if"></a>
#### missing_if:_anotherfield_,_value_,...

如果 _anotherfield_ 欄位等於任何 _值_，則正在驗證的欄位不得存在。

<a name="rule-missing-unless"></a>
#### missing_unless:_anotherfield_,_value_

除非 _anotherfield_ 欄位等於任何 _值_，否則正在驗證的欄位不得存在。

<a name="rule-missing-with"></a>
#### missing_with:_foo_,_bar_,...

正在驗證的欄位必須 _僅當_ 任何其他指定欄位存在時才不得存在。

<a name="rule-missing-with-all"></a>
#### missing_with_all:_foo_,_bar_,...

正在驗證的欄位必須 _僅當_ 所有其他指定欄位都存在時才不得存在。

<a name="rule-not-in"></a>
#### not_in:_foo_,_bar_,...

正在驗證的欄位不得包含在給定的值列表中。`Rule::notIn` 方法可用於流暢地建構此規則：

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

正在驗證的欄位不得匹配給定的正規表達式。

在內部，此規則使用 PHP `preg_match` 函數。指定的模式應遵守 `preg_match` 所需的相同格式，因此也應包含有效的定界符。例如：`'email' => 'not_regex:/^.+$/i'`。

> [!WARNING]
> 當使用 `regex` / `not_regex` 模式時，可能需要使用陣列而非 `|` 分隔符號來指定您的驗證規則，特別是當正規表達式包含 `|` 字元時。

<a name="rule-nullable"></a>
#### nullable

正在驗證的欄位可以為 `null`。

<a name="rule-numeric"></a>
#### numeric

正在驗證的欄位必須是 [數字](https://www.php.net/manual/en/function.is-numeric.php)。

您可以使用 `strict` 參數，僅在欄位的值為整數或浮點類型時才將其視為有效。數字字串將被視為無效：

```php
'amount' => 'numeric:strict'
```

<a name="rule-present"></a>
#### present

正在驗證的欄位必須存在於輸入資料中。

<a name="rule-present-if"></a>
#### present_if:_anotherfield_,_value_,...

如果 _anotherfield_ 欄位等於任何 _值_，則正在驗證的欄位必須存在。

<a name="rule-present-unless"></a>
#### present_unless:_anotherfield_,_value_

除非 _anotherfield_ 欄位等於任何 _值_，否則正在驗證的欄位必須存在。

<a name="rule-present-with"></a>
#### present_with:_foo_,_bar_,...

正在驗證的欄位必須 _僅當_ 任何其他指定欄位存在時才存在。

<a name="rule-present-with-all"></a>
#### present_with_all:_foo_,_bar_,...

正在驗證的欄位必須 _僅當_ 所有其他指定欄位都存在時才存在。

<a name="rule-prohibited"></a>
#### prohibited

正在驗證的欄位必須不存在或為空。欄位在符合以下任一條件時為「空」：

<div class="content-list" markdown="1">

- 值為 `null`。
- 值為空字串。
- 值為空陣列或空的 `Countable` 物件。
- 值為具有空路徑的上傳檔案。

</div>

<a name="rule-prohibited-if"></a>
#### prohibited_if:_anotherfield_,_value_,...

如果 _anotherfield_ 欄位等於任何 _值_，則正在驗證的欄位必須不存在或為空。欄位在符合以下任一條件時為「空」：

<div class="content-list" markdown="1">

- 值為 `null`。
- 值為空字串。
- 值為空陣列或空的 `Countable` 物件。
- 值為具有空路徑的上傳檔案。

</div>

如果需要複雜的條件式禁止邏輯，您可以使用 `Rule::prohibitedIf` 方法。此方法接受布林值或閉包。當給定閉包時，閉包應回傳 `true` 或 `false` 以指示是否應禁止正在驗證的欄位：

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

如果 _anotherfield_ 欄位等於 `"yes"`、`"on"`、`1`、`"1"`、`true` 或 `"true"`，則正在驗證的欄位必須不存在或為空。

<a name="rule-prohibited-if-declined"></a>
#### prohibited_if_declined:_anotherfield_,...

如果 _anotherfield_ 欄位等於 `"no"`、`"off"`、`0`、`"0"`、`false` 或 `"false"`，則正在驗證的欄位必須不存在或為空。

<a name="rule-prohibited-unless"></a>
#### prohibited_unless:_anotherfield_,_value_,...

除非 _anotherfield_ 欄位等於任何 _值_，否則正在驗證的欄位必須不存在或為空。欄位在符合以下任一條件時為「空」：

<div class="content-list" markdown="1">

- 值為 `null`。
- 值為空字串。
- 值為空陣列或空的 `Countable` 物件。
- 值為具有空路徑的上傳檔案。

</div>

<a name="rule-prohibits"></a>
#### prohibits:_anotherfield_,...

如果正在驗證的欄位不存在或不為空，則 _anotherfield_ 中的所有欄位都必須不存在或為空。欄位在符合以下任一條件時為「空」：

<div class="content-list" markdown="1">

- 值為 `null`。
- 值為空字串。
- 值為空陣列或空的 `Countable` 物件。
- 值為具有空路徑的上傳檔案。

</div>

<a name="rule-regex"></a>
#### regex:_pattern_

正在驗證的欄位必須匹配給定的正規表達式。

在內部，此規則使用 PHP `preg_match` 函數。指定的模式應遵守 `preg_match` 所需的相同格式，因此也應包含有效的定界符。例如：`'email' => 'regex:/^.+@.+$/i'`。

> [!WARNING]
> 當使用 `regex` / `not_regex` 模式時，可能需要使用陣列而非 `|` 分隔符號來指定規則，特別是當正規表達式包含 `|` 字元時。

<a name="rule-required"></a>
#### required

驗證中的欄位必須存在於輸入資料中且不能為空。欄位若符合以下任一條件，則為「空」：

<div class="content-list" markdown="1">

- 值為 `null`。
- 值為空字串。
- 值為空陣列或空的 `Countable` 物件。
- 值為沒有路徑的已上傳檔案。

</div>


<a name="rule-required-if"></a>
#### required_if:_anotherfield_,_value_,...

如果 _anotherfield_ 欄位等於任何 _value_，則驗證中的欄位必須存在且不能為空。

如果您想要為 `required_if` 規則建構更複雜的條件，可以使用 `Rule::requiredIf` 方法。此方法接受布林值或閉包。當傳入閉包時，該閉包應回傳 `true` 或 `false` 以指示驗證中的欄位是否為必填：

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

如果 _anotherfield_ 欄位等於 `"yes"`、`"on"`、`1`、`"1"`、`true` 或 `"true"`，則驗證中的欄位必須存在且不能為空。


<a name="rule-required-if-declined"></a>
#### required_if_declined:_anotherfield_,...

如果 _anotherfield_ 欄位等於 `"no"`、`"off"`、`0`、`"0"`、`false` 或 `"false"`，則驗證中的欄位必須存在且不能為空。


<a name="rule-required-unless"></a>
#### required_unless:_anotherfield_,_value_,...

除非 _anotherfield_ 欄位等於任何 _value_，否則驗證中的欄位必須存在且不能為空。這也意味著除非 _value_ 為 `null`，否則 _anotherfield_ 必須存在於請求資料中。如果 _value_ 為 `null` (`required_unless:name,null`)，則驗證中的欄位將為必填，除非比較欄位為 `null` 或比較欄位在請求資料中遺失。


<a name="rule-required-with"></a>
#### required_with:_foo_,_bar_,...

驗證中的欄位必須存在且不能為空，**僅當**任何其他指定的欄位存在且不為空時。


<a name="rule-required-with-all"></a>
#### required_with_all:_foo_,_bar_,...

驗證中的欄位必須存在且不能為空，**僅當**所有其他指定的欄位存在且不為空時。


<a name="rule-required-without"></a>
#### required_without:_foo_,_bar_,...

驗證中的欄位必須存在且不能為空，**僅當**任何其他指定的欄位為空或不存在時。


<a name="rule-required-without-all"></a>
#### required_without_all:_foo_,_bar_,...

驗證中的欄位必須存在且不能為空，**僅當**所有其他指定的欄位為空或不存在時。


<a name="rule-required-array-keys"></a>
#### required_array_keys:_foo_,_bar_,...

驗證中的欄位必須是陣列，且必須至少包含指定的鍵。


<a name="rule-same"></a>
#### same:_field_

給定的 _field_ 必須與驗證中的欄位相符。


<a name="rule-size"></a>
#### size:_value_

驗證中的欄位大小必須與給定的 _value_ 相符。對於字串資料，_value_ 對應字元數。對於數值資料，_value_ 對應給定的整數值 (該屬性也必須具備 `numeric` 或 `integer` 規則)。對於陣列，_size_ 對應陣列的 `count` 值。對於檔案，_size_ 對應以 KB 為單位的檔案大小。讓我們看一些範例：

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

驗證中的欄位必須以給定值中的其中一個開頭。


<a name="rule-string"></a>
#### string

驗證中的欄位必須是字串。如果您想允許該欄位也為 `null`，則應將 `nullable` 規則指派給該欄位。


<a name="rule-timezone"></a>
#### timezone

驗證中的欄位必須是根據 `DateTimeZone::listIdentifiers` 方法定義的有效時區識別符。

`DateTimeZone::listIdentifiers` 方法[接受的參數](https://www.php.net/manual/en/datetimezone.listidentifiers.php)也可以提供給此驗證規則：

```php
'timezone' => 'required|timezone:all';

'timezone' => 'required|timezone:Africa';

'timezone' => 'required|timezone:per_country,US';
```

<a name="rule-unique"></a>
#### unique:_table_,_column_

驗證中的欄位必須在給定的資料庫表格中不存在。

**指定自訂資料表 / 欄位名稱：**

除了直接指定資料表名稱外，您還可以指定 Eloquent Model 來決定資料表名稱：

```php
'email' => 'unique:App\Models\User,email_address'
```

`column` 選項可用於指定欄位所對應的資料庫欄位。如果未指定 `column` 選項，則將使用驗證中的欄位名稱。

```php
'email' => 'unique:users,email_address'
```

**指定自訂資料庫連線**

有時，您可能需要為驗證器執行的資料庫查詢設定自訂連線。為此，您可以將連線名稱加在資料表名稱之前：

```php
'email' => 'unique:connection.users,email_address'
```

**強制唯一性規則忽略給定 ID：**

有時，您可能希望在唯一性驗證期間忽略給定的 ID。例如，考慮一個「更新個人檔案」的畫面，其中包含使用者的姓名、電子郵件地址和位置。您可能會想驗證電子郵件地址的唯一性。但是，如果使用者只更改姓名欄位而未更改電子郵件欄位，您不希望因為使用者已經是該電子郵件地址的擁有者而拋出驗證錯誤。

為了指示驗證器忽略使用者的 ID，我們將使用 `Rule` 類別流暢地定義規則。在這個範例中，我們還會將驗證規則指定為陣列，而不是使用 `|` 字元來分隔規則：

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
> 您絕對不應該將任何使用者控制的請求輸入傳遞給 `ignore` 方法。相反地，您應該只傳遞系統生成的唯一 ID，例如 Eloquent Model 實例中的自動遞增 ID 或 UUID。否則，您的應用程式將容易受到 SQL 注入攻擊。

除了將 Model 鍵的值傳遞給 `ignore` 方法外，您也可以傳遞整個 Model 實例。Laravel 會自動從 Model 中提取鍵：

```php
Rule::unique('users')->ignore($user)
```

如果您的資料表使用的主鍵欄位名稱不是 `id`，您可以在呼叫 `ignore` 方法時指定欄位名稱：

```php
Rule::unique('users')->ignore($user->id, 'user_id')
```

預設情況下，`unique` 規則會檢查與驗證中屬性名稱相符的欄位唯一性。但是，您可以將不同的欄位名稱作為第二個參數傳遞給 `unique` 方法：

```php
Rule::unique('users', 'email_address')->ignore($user->id)
```

**新增額外的 Where 條件：**

您可以透過使用 `where` 方法自訂查詢來指定額外的查詢條件。例如，讓我們新增一個查詢條件，將查詢範圍限定為只搜尋 `account_id` 欄位值為 `1` 的記錄：

```php
'email' => Rule::unique('users')->where(fn (Builder $query) => $query->where('account_id', 1))
```

**忽略唯一性檢查中的軟刪除記錄：**

預設情況下，`unique` 規則在判斷唯一性時會包含軟刪除的記錄。若要從唯一性檢查中排除軟刪除的記錄，您可以呼叫 `withoutTrashed` 方法：

```php
Rule::unique('users')->withoutTrashed();
```

如果您的 Model 對於軟刪除記錄使用 `deleted_at` 以外的欄位名稱，您可以在呼叫 `withoutTrashed` 方法時提供該欄位名稱：

```php
Rule::unique('users')->withoutTrashed('was_deleted_at');
```

<a name="rule-uppercase"></a>
#### uppercase

驗證中的欄位必須是大寫。

<a name="rule-url"></a>
#### url

驗證中的欄位必須是有效的 URL。

如果您想指定應視為有效的 URL 協定，您可以將這些協定作為驗證規則參數傳遞：

```php
'url' => 'url:http,https',

'game' => 'url:minecraft,steam',
```

<a name="rule-ulid"></a>
#### ulid

驗證中的欄位必須是有效的 [通用唯一詞典排序識別碼](https://github.com/ulid/spec) (ULID)。

<a name="rule-uuid"></a>
#### uuid

驗證中的欄位必須是有效的 RFC 9562 (版本 1、3、4、5、6、7 或 8) 通用唯一識別碼 (UUID)。

您也可以驗證給定的 UUID 是否符合按版本指定的 UUID 規範：

```php
'uuid' => 'uuid:4'
```

<a name="conditionally-adding-rules"></a>
## 條件式新增規則


<a name="skipping-validation-when-fields-have-certain-values"></a>
#### 當欄位具有特定值時跳過驗證

您偶爾可能會希望，如果另一個欄位具有特定值時，則不驗證給定的欄位。您可以使用 `exclude_if` 驗證規則來實現此目的。在此範例中，如果 `has_appointment` 欄位的值為 `false`，則 `appointment_date` 和 `doctor_name` 欄位將不會被驗證：

```php
use Illuminate\Support\Facades\Validator;

$validator = Validator::make($data, [
    'has_appointment' => 'required|boolean',
    'appointment_date' => 'exclude_if:has_appointment,false|required|date',
    'doctor_name' => 'exclude_if:has_appointment,false|required|string',
]);
```

或者，您可以使用 `exclude_unless` 規則，除非另一個欄位具有特定值，否則不驗證給定的欄位：

```php
$validator = Validator::make($data, [
    'has_appointment' => 'required|boolean',
    'appointment_date' => 'exclude_unless:has_appointment,true|required|date',
    'doctor_name' => 'exclude_unless:has_appointment,true|required|string',
]);
```


<a name="validating-when-present"></a>
#### 僅當存在時才進行驗證

在某些情況下，您可能希望**僅在**驗證資料中存在某個欄位時才對其執行驗證檢查。要快速實現此目的，請將 `sometimes` 規則添加到您的規則列表：

```php
$validator = Validator::make($data, [
    'email' => 'sometimes|required|email',
]);
```

在上面的範例中，`email` 欄位將**只會**在 `$data` 陣列中存在時才進行驗證。

> [!NOTE]
> 如果您嘗試驗證一個應該始終存在但可能為空的欄位，請查看 [關於選填欄位的注意事項](#a-note-on-optional-fields)。


<a name="complex-conditional-validation"></a>
#### 複雜的條件式驗證

有時您可能希望根據更複雜的條件邏輯來添加驗證規則。例如，您可能希望僅當另一個欄位的值大於 100 時才要求某個給定欄位。或者，您可能需要兩個欄位僅在另一個欄位存在時才具有特定值。添加這些驗證規則不應該是件麻煩事。首先，使用您的_靜態規則_（永不改變的規則）來建立一個 `Validator` 實例：

```php
use Illuminate\Support\Facades\Validator;

$validator = Validator::make($request->all(), [
    'email' => 'required|email',
    'games' => 'required|integer|min:0',
]);
```

假設我們的網路應用程式是為遊戲收藏家設計的。如果一位遊戲收藏家在我們的應用程式註冊並擁有超過 100 款遊戲，我們希望他們解釋為何擁有這麼多遊戲。例如，他們可能經營一家遊戲轉售商店，或者只是喜歡收集遊戲。要條件式地添加此要求，我們可以使用 `Validator` 實例上的 `sometimes` 方法。

```php
use Illuminate\Support\Fluent;

$validator->sometimes('reason', 'required|max:500', function (Fluent $input) {
    return $input->games >= 100;
});
```

傳遞給 `sometimes` 方法的第一個參數是我們條件式驗證的欄位名稱。第二個參數是我們想要添加的規則列表。如果作為第三個參數傳遞的閉包返回 `true`，則這些規則將被添加。這種方法讓建立複雜的條件式驗證變得輕而易舉。您甚至可以一次為多個欄位添加條件式驗證：

```php
$validator->sometimes(['reason', 'cost'], 'required', function (Fluent $input) {
    return $input->games >= 100;
});
```

> [!NOTE]
> 傳遞給閉包的 `$input` 參數將是 `Illuminate\Support\Fluent` 的實例，可用於存取您正在驗證的輸入和檔案。


<a name="complex-conditional-array-validation"></a>
#### 複雜的條件式陣列驗證

有時您可能希望根據同一個巢狀陣列中您不知道索引的另一個欄位來驗證某個欄位。在這些情況下，您可以讓閉包接收第二個參數，該參數將是正在驗證的陣列中當前的個別項目：

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

與傳遞給閉包的 `$input` 參數一樣，當屬性資料是陣列時，`$item` 參數是 `Illuminate\Support\Fluent` 的實例；否則，它是一個字串。

<a name="validating-arrays"></a>
## 驗證陣列

正如 [array 驗證規則說明](#rule-array) 中所述，`array` 規則接受一個允許的陣列鍵列表。如果陣列中存在任何額外的鍵，驗證將會失敗：

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

一般來說，您應該始終指定陣列中允許存在的鍵。否則，驗證器的 `validate` 和 `validated` 方法將會回傳所有已驗證的資料，包括陣列及其所有鍵，即使這些鍵沒有被其他巢狀陣列驗證規則驗證。

<a name="validating-nested-array-input"></a>
### 驗證巢狀陣列輸入

驗證巢狀陣列形式的表單輸入欄位不必很麻煩。您可以使用「點符號 (dot notation)」來驗證陣列內的屬性。例如，如果傳入的 HTTP 請求包含 `photos[profile]` 欄位，您可以這樣驗證它：

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

同樣地，您可以在 [語系檔中指定自訂驗證訊息](#custom-messages-for-specific-attributes) 時使用 `*` 字元，這使得為陣列形式的欄位使用單一驗證訊息變得輕而易舉：

```php
'custom' => [
    'users.*.email' => [
        'unique' => 'Each user must have a unique email address',
    ]
],
```

<a name="accessing-nested-array-data"></a>
#### 存取巢狀陣列資料

有時您可能需要在為屬性指派驗證規則時，存取特定巢狀陣列元素的值。您可以使用 `Rule::forEach` 方法來達成此目的。`forEach` 方法接受一個閉包，該閉包將在驗證的陣列屬性每次迭代時被呼叫，並會收到該屬性的值和明確、完整展開的屬性名稱。該閉包應回傳一個規則陣列，以指派給該陣列元素：

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

驗證陣列時，您可能希望在應用程式顯示的錯誤訊息中，引用特定失敗驗證項目的索引或位置。為此，您可以在 [自訂驗證訊息](#manual-customizing-the-error-messages) 中包含 `:index` (從 `0` 開始)、`:position` (從 `1` 開始) 或 `:ordinal-position` (從 `1st` 開始) 佔位符：

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

根據上述範例，驗證將會失敗，使用者將會收到以下錯誤訊息：_「請描述照片 #2。」_

如有必要，您可以透過 `second-index`、`second-position`、`third-index`、`third-position` 等來引用更深層的巢狀索引和位置。

```php
'photos.*.attributes.*.string' => 'Invalid attribute for photo #:second-position.',
```

<a name="validating-files"></a>
## 驗證檔案

Laravel 提供了多種可用於驗證上傳檔案的驗證規則，例如 `mimes`、`image`、`min` 和 `max`。雖然您可以自由地單獨指定這些規則來驗證檔案，但 Laravel 也提供了一個流暢的檔案驗證規則建立器，您可能會覺得很方便：

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

儘管您在呼叫 `types` 方法時只需指定副檔名，但此方法實際上會透過讀取檔案內容並猜測其 MIME 類型來驗證檔案的 MIME 類型。您可以在以下位置找到 MIME 類型及其對應副檔名的完整列表：

[https://svn.apache.org/repos/asf/httpd/httpd/trunk/docs/conf/mime.types](https://svn.apache.org/repos/asf/httpd/httpd/trunk/docs/conf/mime.types)

<a name="validating-files-file-sizes"></a>
#### 驗證檔案大小

為方便起見，最小和最大檔案大小可以字串形式指定，並附帶一個指示檔案大小單位的後綴。支援 `kb`、`mb`、`gb` 和 `tb` 後綴：

```php
File::types(['mp3', 'wav'])
    ->min('1kb')
    ->max('10mb');
```

<a name="validating-files-image-files"></a>
#### 驗證圖片檔案

如果您的應用程式接受使用者上傳的圖片，您可以使用 `File` 規則的 `image` 建構式方法來確保被驗證的檔案是圖片 (jpg、jpeg、png、bmp、gif 或 webp)。

此外，`dimensions` 規則可用來限制圖片的尺寸：

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
> 有關驗證圖片尺寸的更多資訊，請參閱 [dimension 規則說明](#rule-dimensions)。

> [!WARNING]
> 預設情況下，`image` 規則不允許 SVG 檔案，因為存在 XSS 漏洞的可能性。如果您需要允許 SVG 檔案，可以將 `allowSvg: true` 傳遞給 `image` 規則：`File::image(allowSvg: true)`。

<a name="validating-files-image-dimensions"></a>
#### 驗證圖片尺寸

您也可以驗證圖片的尺寸。例如，若要驗證上傳的圖片寬度至少為 1000 像素，高度至少為 500 像素，您可以使用 `dimensions` 規則：

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
> 有關驗證圖片尺寸的更多資訊，請參閱 [dimension 規則說明](#rule-dimensions)。

<a name="validating-passwords"></a>
## 驗證密碼

為確保密碼具有足夠的複雜度，您可以使用 Laravel 的 `Password` 規則物件：

```php
use Illuminate\Support\Facades\Validator;
use Illuminate\Validation\Rules\Password;

$validator = Validator::make($request->all(), [
    'password' => ['required', 'confirmed', Password::min(8)],
]);
```

`Password` 規則物件讓您可以輕鬆自訂應用程式的密碼複雜度要求，例如指定密碼至少需要一個字母、數字、符號或大小寫混合的字元：

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

此外，您還可以使用 `uncompromised` 方法，確保密碼沒有在公開的密碼洩露事件中被洩露：

```php
Password::min(8)->uncompromised()
```

在內部，`Password` 規則物件使用 [k-Anonymity](https://en.wikipedia.org/wiki/K-anonymity) 模型，透過 [haveibeenpwned.com](https://haveibeenpwned.com) 服務來判斷密碼是否已被洩露，同時不犧牲使用者的隱私或安全性。

預設情況下，如果密碼在資料洩露中至少出現一次，則將被視為已洩露。您可以使用 `uncompromised` 方法的第一個參數來自訂此閾值：

```php
// Ensure the password appears less than 3 times in the same data leak...
Password::min(8)->uncompromised(3);
```

當然，您可以將上述範例中的所有方法串聯起來使用：

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

您可能會發現在應用程式的單一位置指定密碼的預設驗證規則很方便。您可以使用 `Password::defaults` 方法輕鬆實現這一點，該方法接受一個閉包。傳遞給 `defaults` 方法的閉包應該返回 Password 規則的預設配置。通常，`defaults` 規則應該在應用程式某個服務提供者的 `boot` 方法中呼叫：

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

然後，當您想將預設規則應用於正在驗證的特定密碼時，可以呼叫不帶任何參數的 `defaults` 方法：

```php
'password' => ['required', Password::defaults()],
```

偶爾，您可能希望將額外的驗證規則附加到您的預設密碼驗證規則中。您可以使用 `rules` 方法來實現這一點：

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

Laravel 提供了多種實用的驗證規則；但是，您可能希望指定一些自己的規則。註冊自訂驗證規則的一種方法是使用規則物件。要生成新的規則物件，您可以使用 `make:rule` Artisan 指令。讓我們使用此指令來生成一個驗證字串是否為大寫的規則。Laravel 會將新規則放置在 `app/Rules` 目錄中。如果此目錄不存在，Laravel 會在您執行 Artisan 指令建立規則時創建它：

```shell
php artisan make:rule Uppercase
```

一旦規則建立後，我們就可以定義其行為。規則物件包含一個方法：`validate`。此方法接收屬性名稱、其值，以及一個在驗證失敗時應呼叫並帶有驗證錯誤訊息的回呼函式：

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

一旦定義了規則，您可以將其與其他驗證規則一起傳遞規則物件的實例，以將其附加到驗證器：

```php
use App\Rules\Uppercase;

$request->validate([
    'name' => ['required', 'string', new Uppercase],
]);
```


#### 翻譯驗證訊息

您也可以提供 [翻譯字串鍵](/docs/{{version}}/localization) 並指示 Laravel 翻譯錯誤訊息，而不是向 `$fail` 閉包提供字面上的錯誤訊息：

```php
if (strtoupper($value) !== $value) {
    $fail('validation.uppercase')->translate();
}
```

如有需要，您可以將預留位置替換和首選語言作為 `translate` 方法的第一個和第二個參數提供：

```php
$fail('validation.location')->translate([
    'value' => $this->value,
], 'fr');
```


#### 存取額外資料

如果您的自訂驗證規則類別需要存取所有其他正在驗證的資料，您的規則類別可以實作 `Illuminate\Contracts\Validation\DataAwareRule` 介面。此介面要求您的類別定義 `setData` 方法。此方法將由 Laravel 自動呼叫（在驗證進行之前），並帶有所有正在驗證的資料：

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

如果您的應用程式只需要一次自訂規則的功能，您可以使用閉包而不是規則物件。閉包接收屬性名稱、屬性值以及一個在驗證失敗時應呼叫的 `$fail` 回呼函式：

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

預設情況下，當正在驗證的屬性不存在或包含空字串時，包括自訂規則在內的正常驗證規則將不會執行。例如，[unique](#rule-unique) 規則將不會針對空字串執行：

```php
use Illuminate\Support\Facades\Validator;

$rules = ['name' => 'unique:users,name'];

$input = ['name' => ''];

Validator::make($input, $rules)->passes(); // true
```

若要讓自訂規則即使在屬性為空時也能執行，該規則必須暗示該屬性為必填。要快速生成一個新的隱式規則物件，您可以使用帶有 `--implicit` 選項的 `make:rule` Artisan 指令：

```shell
php artisan make:rule Uppercase --implicit
```

> [!WARNING]
> 「隱式」規則僅 _暗示_ 該屬性為必填。它是否真的會使缺少或為空的屬性失效取決於您。