# 驗證

- [介紹](#introduction)
- [快速開始驗證](#validation-quickstart)
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
    - [為驗證準備輸入資料](#preparing-input-for-validation)
- [手動建立驗證器](#manually-creating-validators)
    - [自動重新導向](#automatic-redirection)
    - [具名錯誤袋](#named-error-bags)
    - [自訂錯誤訊息](#manual-customizing-the-error-messages)
    - [執行額外驗證](#performing-additional-validation)
- [處理已驗證的輸入](#working-with-validated-input)
- [處理錯誤訊息](#working-with-error-messages)
    - [在語言檔中指定自訂訊息](#specifying-custom-messages-in-language-files)
    - [在語言檔中指定屬性](#specifying-attribute-in-language-files)
    - [在語言檔中指定值](#specifying-values-in-language-files)
- [可用的驗證規則](#available-validation-rules)
- [有條件地加入規則](#conditionally-adding-rules)
- [驗證陣列](#validating-arrays)
    - [驗證巢狀陣列輸入](#validating-nested-array-input)
    - [錯誤訊息索引與位置](#error-message-indexes-and-positions)
- [驗證檔案](#validating-files)
- [驗證密碼](#validating-passwords)
- [自訂驗證規則](#custom-validation-rules)
    - [使用規則物件](#using-rule-objects)
    - [使用閉包](#using-closures)
    - [隱含規則](#implicit-rules)

<a name="introduction"></a>
## 介紹

Laravel 提供了幾種不同的方法來驗證應用程式的傳入資料。最常見的做法是使用所有傳入 HTTP 請求上可用的 `validate` 方法。然而，我們也會討論其他的驗證方法。

Laravel 包含了各種方便的驗證規則，可套用到資料上，甚至提供了驗證值在給定資料庫表格中是否唯一的能力。我們將會詳細涵蓋這些驗證規則的每個細節，讓您熟悉 Laravel 的所有驗證功能。

<a name="validation-quickstart"></a>
## 快速開始驗證

為了瞭解 Laravel 強大的驗證功能，我們將透過一個完整的範例來驗證表單並將錯誤訊息顯示給使用者。透過閱讀這份高階概述，您將對如何使用 Laravel 驗證傳入的請求資料有一個良好的總體理解：


<a name="quick-defining-the-routes"></a>
### 定義路由

首先，假設我們在 `routes/web.php` 檔案中定義了以下路由：

    use App\Http\Controllers\PostController;

    Route::get('/post/create', [PostController::class, 'create']);
    Route::post('/post', [PostController::class, 'store']);

`GET` 路由將顯示一個表單供使用者建立新的部落格文章，而 `POST` 路由則會將新的部落格文章儲存到資料庫中。


<a name="quick-creating-the-controller"></a>
### 建立控制器

接下來，讓我們看看一個處理這些路由傳入請求的簡單控制器。我們目前會將 `store` 方法留空：

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


<a name="quick-writing-the-validation-logic"></a>
### 編寫驗證邏輯

現在我們準備好在 `store` 方法中填入驗證新部落格文章的邏輯。為此，我們將使用 `Illuminate\Http\Request` 物件提供的 `validate` 方法。如果驗證規則通過，您的程式碼將正常執行；但是，如果驗證失敗，將會拋出 `Illuminate\Validation\ValidationException` 例外，並自動向使用者傳回適當的錯誤回應。

如果在傳統的 HTTP 請求期間驗證失敗，將會生成一個重新導向回應到先前的 URL。如果傳入的請求是 XHR 請求，則會傳回[包含驗證錯誤訊息的 JSON 回應](#validation-error-response-format)。

為了更好地理解 `validate` 方法，讓我們回到 `store` 方法：

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

如您所見，驗證規則會傳遞給 `validate` 方法。不用擔心，所有可用的驗證規則都[已文件化](#available-validation-rules)。再次強調，如果驗證失敗，將會自動生成適當的回應。如果驗證通過，我們的控制器將會正常執行。

此外，驗證規則也可以指定為規則陣列，而不是單個 `|` 分隔的字串：

    $validatedData = $request->validate([
        'title' => ['required', 'unique:posts', 'max:255'],
        'body' => ['required'],
    ]);

此外，您可以使用 `validateWithBag` 方法來驗證請求並將任何錯誤訊息儲存在[具名錯誤袋](#named-error-bags)中：

    $validatedData = $request->validateWithBag('post', [
        'title' => ['required', 'unique:posts', 'max:255'],
        'body' => ['required'],
    ]);


<a name="stopping-on-first-validation-failure"></a>
#### 在第一次驗證失敗時停止

有時候，您可能希望在第一次驗證失敗後，停止對屬性執行驗證規則。為此，請將 `bail` 規則指派給該屬性：

    $request->validate([
        'title' => 'bail|required|unique:posts|max:255',
        'body' => 'required',
    ]);

在此範例中，如果 `title` 屬性上的 `unique` 規則失敗，則不會檢查 `max` 規則。規則將按照它們被指派的順序進行驗證。


<a name="a-note-on-nested-attributes"></a>
#### 關於巢狀屬性的注意事項

如果傳入的 HTTP 請求包含「巢狀」欄位資料，您可以使用「點號」語法在驗證規則中指定這些欄位：

    $request->validate([
        'title' => 'required|unique:posts|max:255',
        'author.name' => 'required',
        'author.description' => 'required',
    ]);

另一方面，如果您的欄位名稱包含字面上的句點，您可以透過使用反斜線跳脫句點，明確防止其被解釋為「點號」語法：

    $request->validate([
        'title' => 'required|unique:posts|max:255',
        'v1\.0' => 'required',
    ]);

<a name="quick-displaying-the-validation-errors"></a>
### 顯示驗證錯誤

那麼，如果傳入的請求欄位不符合給定的驗證規則該怎麼辦？如前所述，Laravel 會自動將使用者重新導向回他們之前的位置。此外，所有驗證錯誤和[請求輸入](/docs/{{version}}/requests#retrieving-old-input)都將自動[快閃至 Session](/docs/{{version}}/session#flash-data)。

`Illuminate\View\Middleware\ShareErrorsFromSession` 中介層（由 `web` 中介層群組提供）會將 `$errors` 變數與您應用程式的所有視圖共享。當此中介層被應用時，`$errors` 變數將始終在您的視圖中可用，讓您可以方便地假設 `$errors` 變數始終已定義且可以安全使用。`$errors` 變數將是一個 `Illuminate\Support\MessageBag` 實例。有關如何使用此物件的更多資訊，請[參閱其文件](#working-with-error-messages)。

因此，在我們的範例中，當驗證失敗時，使用者將被重新導向到我們控制器的 `create` 方法，讓您可以在視圖中顯示錯誤訊息：

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

Laravel 內建的驗證規則都有一個錯誤訊息，位於您應用程式的 `lang/en/validation.php` 檔案中。如果您的應用程式沒有 `lang` 目錄，您可以使用 `lang:publish` Artisan 指令來指示 Laravel 建立它。

在 `lang/en/validation.php` 檔案中，您會找到每個驗證規則的翻譯項目。您可以根據應用程式的需求自由更改或修改這些訊息。

此外，您可以將此檔案複製到另一個語言目錄，以翻譯應用程式語言的訊息。要了解更多關於 Laravel localization 的資訊，請查閱完整的[本地化文件](/docs/{{version}}/localization)。

> [!WARNING]  
> 預設情況下，Laravel 應用程式骨架不包含 `lang` 目錄。如果您想自訂 Laravel 的語言檔案，您可以透過 `lang:publish` Artisan 指令發布它們。

<a name="quick-xhr-requests-and-validation"></a>
#### XHR 請求與驗證

在此範例中，我們使用傳統表單向應用程式發送資料。然而，許多應用程式從由 JavaScript 驅動的前端接收 XHR 請求。在 XHR 請求期間使用 `validate` 方法時，Laravel 將不會產生重新導向回應。相反地，Laravel 會產生一個[包含所有驗證錯誤的 JSON 回應](#validation-error-response-format)。此 JSON 回應將以 422 HTTP 狀態碼發送。

<a name="the-at-error-directive"></a>
#### `@error` 指令

您可以使用 `@error` [Blade](/docs/{{version}}/blade) 指令快速判斷特定屬性是否存在驗證錯誤訊息。在 `@error` 指令中，您可以 echo `$message` 變數來顯示錯誤訊息：

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

如果您正在使用[具名錯誤袋](#named-error-bags)，您可以將錯誤袋的名稱作為第二個引數傳遞給 `@error` 指令：

```blade
<input ... class="@error('title', 'post') is-invalid @enderror">
```

<a name="repopulating-forms"></a>
### 重新填入表單

當 Laravel 因為驗證錯誤而產生重新導向回應時，框架會自動[將所有請求的輸入快閃至 Session](/docs/{{version}}/session#flash-data)。這樣做的目的是讓您可以在下次請求時方便地存取輸入，並重新填入使用者嘗試提交的表單。

要從先前的請求中檢索快閃輸入，請在 `Illuminate\Http\Request` 實例上呼叫 `old` 方法。`old` 方法將從 [Session](/docs/{{version}}/session) 中取出先前快閃的輸入資料：

    $title = $request->old('title');

Laravel 還提供了一個全域的 `old` 輔助函式。如果您在 [Blade 範本](/docs/{{version}}/blade)中顯示舊輸入，使用 `old` 輔助函式來重新填入表單會更方便。如果指定欄位沒有舊輸入，則會傳回 `null`：

```blade
<input type="text" name="title" value="{{ old('title') }}">
```

<a name="a-note-on-optional-fields"></a>
### 關於選填欄位的注意事項

預設情況下，Laravel 在您的應用程式全域中介層堆疊中包含了 `TrimStrings` 和 `ConvertEmptyStringsToNull` 中介層。因此，如果您不希望驗證器將 `null` 值視為無效，通常需要將您的「選填」請求欄位標記為 `nullable`。例如：

    $request->validate([
        'title' => 'required|unique:posts|max:255',
        'body' => 'required',
        'publish_at' => 'nullable|date',
    ]);

在此範例中，我們指定 `publish_at` 欄位可以是 `null` 或有效的日期表示。如果未將 `nullable` 修飾符加入規則定義中，驗證器會將 `null` 視為無效日期。

<a name="validation-error-response-format"></a>
### 驗證錯誤回應格式

當您的應用程式拋出 `Illuminate\Validation\ValidationException` 異常，且傳入的 HTTP 請求預期是 JSON 回應時，Laravel 會自動為您格式化錯誤訊息，並傳回 `422 Unprocessable Entity` HTTP 回應。

下面，您可以檢閱驗證錯誤的 JSON 回應格式範例。請注意，巢狀錯誤鍵將會被扁平化為「點」表示法格式：

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

對於更複雜的驗證情境，您可能會希望建立一個「form request」。Form requests 是自訂的請求類別，它們封裝了自身的驗證與授權邏輯。若要建立一個 form request 類別，您可以使用 `make:request` Artisan CLI 命令：

```shell
php artisan make:request StorePostRequest
```

產生的 form request 類別將會放在 `app/Http/Requests` 目錄中。如果此目錄不存在，當您執行 `make:request` 命令時將會建立它。每個由 Laravel 產生的 form request 都會有兩個方法：`authorize` 和 `rules`。

如您所料，`authorize` 方法負責判斷當前已驗證的使用者是否能執行請求所代表的動作，而 `rules` 方法則回傳應該應用於請求資料的驗證規則：

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

> [!NOTE]  
> 您可以在 `rules` 方法的簽章中型別提示任何您需要的依賴，它們將會透過 Laravel [service container](/docs/{{version}}/container) 自動解析。

那麼，驗證規則是如何被評估的呢？您所需要做的，就是在控制器方法中型別提示該請求。傳入的 form request 會在控制器方法被呼叫之前進行驗證，這表示您不需要在控制器中塞入任何驗證邏輯：

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

如果驗證失敗，將會產生一個重新導向的回應，將使用者導回他們先前的位置。錯誤訊息也會快閃到 session 中，以便顯示。如果該請求是 XHR 請求，則會向使用者傳回一個 HTTP 狀態碼為 422 的回應，其中包含驗證錯誤的 [JSON 表示](#validation-error-response-format)。

> [!NOTE]  
> 需要為您由 Inertia 驅動的 Laravel 前端新增即時 form request 驗證嗎？請查看 [Laravel Precognition](/docs/{{version}}/precognition)。

<a name="performing-additional-validation-on-form-requests"></a>
#### 執行額外驗證

有時，在初始驗證完成後，您可能需要執行額外的驗證。您可以使用 form request 的 `after` 方法來完成此操作。

`after` 方法應該回傳一個可呼叫 (callable) 或閉包 (closure) 的陣列，這些將會在驗證完成後被呼叫。這些可呼叫項目將會收到一個 `Illuminate\Validation\Validator` 實例，允許您在必要時拋出額外的錯誤訊息：

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

如前所述，`after` 方法回傳的陣列中也可以包含可呼叫的類別。這些類別的 `__invoke` 方法將會收到一個 `Illuminate\Validation\Validator` 實例：

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
#### 在首次驗證失敗時停止

透過在您的請求類別中新增 `stopOnFirstFailure` 屬性，您可以通知驗證器在發生單一驗證失敗時，應停止驗證所有屬性：

    /**
     * Indicates if the validator should stop on the first rule failure.
     *
     * @var bool
     */
    protected $stopOnFirstFailure = true;

<a name="customizing-the-redirect-location"></a>
#### 自訂重新導向位置

當 form request 驗證失敗時，將會產生一個重新導向回應，將使用者導回他們先前的位置。然而，您可以自由自訂此行為。若要這樣做，請在您的 form request 上定義一個 `$redirect` 屬性：

    /**
     * The URI that users should be redirected to if validation fails.
     *
     * @var string
     */
    protected $redirect = '/dashboard';

或者，如果您想將使用者重新導向到一個具名路由，您可以改為定義一個 `$redirectRoute` 屬性：

    /**
     * The route that users should be redirected to if validation fails.
     *
     * @var string
     */
    protected $redirectRoute = 'dashboard';

<a name="authorizing-form-requests"></a>
### 授權表單請求

Form request 類別也包含了一個 `authorize` 方法。在此方法中，您可以判斷已驗證的使用者是否確實有權限更新給定的資源。例如，您可以判斷使用者是否確實擁有他們嘗試更新的部落格留言。最有可能的是，您將在此方法中與您的 [授權 Gates 及 Policies](/docs/{{version}}/authorization) 進行互動：

    use App\Models\Comment;

    /**
     * Determine if the user is authorized to make this request.
     */
    public function authorize(): bool
    {
        $comment = Comment::find($this->route('comment'));

        return $comment && $this->user()->can('update', $comment);
    }

由於所有的 form requests 都繼承了基礎的 Laravel 請求類別，我們可以使用 `user` 方法來存取當前已驗證的使用者。此外，請注意上面範例中對 `route` 方法的呼叫。此方法讓您能存取正在被呼叫的路由上定義的 URI 參數，例如下面範例中的 `{comment}` 參數：

    Route::post('/comment/{comment}');

因此，如果您的應用程式有利用 [路由模型綁定](/docs/{{version}}/routing#route-model-binding)，您的程式碼可以透過將解析後的模型作為請求的屬性來存取，從而變得更加簡潔：

    return $this->user()->can('update', $this->comment);

如果 `authorize` 方法回傳 `false`，則將會自動回傳一個 HTTP 狀態碼為 403 的回應，並且您的控制器方法將不會執行。

如果您打算在應用程式的其他部分處理請求的授權邏輯，您可以完全移除 `authorize` 方法，或者僅回傳 `true`：

    /**
     * Determine if the user is authorized to make this request.
     */
    public function authorize(): bool
    {
        return true;
    }

> [!NOTE]  
> 您可以在 `authorize` 方法的簽章中型別提示任何您需要的依賴。它們將會透過 Laravel [service container](/docs/{{version}}/container) 自動解析。

<a name="customizing-the-error-messages"></a>
### 自訂錯誤訊息

您可以透過覆寫 `messages` 方法來自訂表單請求使用的錯誤訊息。此方法應回傳一個屬性 / 規則配對及其對應錯誤訊息的陣列：

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


<a name="customizing-the-validation-attributes"></a>
#### 自訂驗證屬性

許多 Laravel 內建的驗證規則錯誤訊息都包含一個 `:attribute` 佔位符。如果您想將驗證訊息中的 `:attribute` 佔位符替換為自訂屬性名稱，您可以透過覆寫 `attributes` 方法來指定自訂名稱。此方法應回傳一個屬性 / 名稱配對的陣列：

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


<a name="preparing-input-for-validation"></a>
### 為驗證準備輸入資料

如果您需要在應用驗證規則之前準備或清理請求中的任何資料，您可以使用 `prepareForValidation` 方法：

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

同樣地，如果您需要在驗證完成後正規化任何請求資料，您可以使用 `passedValidation` 方法：

    /**
     * Handle a passed validation attempt.
     */
    protected function passedValidation(): void
    {
        $this->replace(['name' => 'Taylor']);
    }

<a name="manually-creating-validators"></a>
## 手動建立驗證器

如果您不想在請求中使用 `validate` 方法，可以透過 `Validator` [Facade](/docs/{{version}}/facades) 手動建立一個驗證器實例。此 Facade 的 `make` 方法會產生一個新的驗證器實例：

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

傳遞給 `make` 方法的第一個參數是待驗證的資料。第二個參數是應套用至資料的驗證規則陣列。

在判斷請求驗證是否失敗後，您可以使用 `withErrors` 方法將錯誤訊息閃存到 Session 中。當使用此方法時，`$errors` 變數在重新導向後會自動與您的視圖共用，讓您可以輕鬆地將它們顯示給使用者。`withErrors` 方法接受一個驗證器、一個 `MessageBag` 或一個 PHP `array`。

#### 在首次驗證失敗時停止

`stopOnFirstFailure` 方法會告知驗證器，一旦發生單一驗證失敗，它應停止驗證所有屬性：

    if ($validator->stopOnFirstFailure()->fails()) {
        // ...
    }

<a name="automatic-redirection"></a>
### 自動重新導向

如果您想手動建立一個驗證器實例，但仍想利用 HTTP 請求的 `validate` 方法提供的自動重新導向功能，您可以在現有的驗證器實例上呼叫 `validate` 方法。如果驗證失敗，使用者將會自動被重新導向，或者，如果是 XHR 請求，則會傳回一個 [JSON 格式的錯誤回應](#validation-error-response-format)。

    Validator::make($request->all(), [
        'title' => 'required|unique:posts|max:255',
        'body' => 'required',
    ])->validate();

如果驗證失敗，您可以使用 `validateWithBag` 方法將錯誤訊息儲存在 [具名錯誤袋](#named-error-bags) 中：

    Validator::make($request->all(), [
        'title' => 'required|unique:posts|max:255',
        'body' => 'required',
    ])->validateWithBag('post');

<a name="named-error-bags"></a>
### 具名錯誤袋

如果單一頁面上有多個表單，您可能希望為包含驗證錯誤的 `MessageBag` 命名，以便為特定表單擷取錯誤訊息。要實現這一點，請將名稱作為第二個參數傳遞給 `withErrors`：

    return redirect('/register')->withErrors($validator, 'login');

然後，您可以從 `$errors` 變數中存取具名的 `MessageBag` 實例：

```blade
{{ $errors->login->first('email') }}
```

<a name="manual-customizing-the-error-messages"></a>
### 自訂錯誤訊息

如果需要，您可以提供自訂錯誤訊息，讓驗證器實例使用，而不是 Laravel 提供的預設錯誤訊息。有幾種方式可以指定自訂訊息。首先，您可以將自訂訊息作為第三個參數傳遞給 `Validator::make` 方法：

    $validator = Validator::make($input, $rules, $messages = [
        'required' => 'The :attribute field is required.',
    ]);

在此範例中，`:attribute` 預留位置將被實際的驗證欄位名稱取代。您也可以在驗證訊息中利用其他預留位置。例如：

    $messages = [
        'same' => 'The :attribute and :other must match.',
        'size' => 'The :attribute must be exactly :size.',
        'between' => 'The :attribute value :input is not between :min - :max.',
        'in' => 'The :attribute must be one of the following types: :values',
    ];

<a name="specifying-a-custom-message-for-a-given-attribute"></a>
#### 為指定屬性指定自訂訊息

有時您可能希望僅為特定屬性指定自訂錯誤訊息。您可以使用「點」符號來做到這一點。首先指定屬性的名稱，然後是規則：

    $messages = [
        'email.required' => 'We need to know your email address!',
    ];

<a name="specifying-custom-attribute-values"></a>
#### 指定自訂屬性值

Laravel 許多內建的錯誤訊息包含 `:attribute` 預留位置，該預留位置會被驗證欄位或屬性的名稱取代。若要自訂用於取代特定欄位這些預留位置的值，您可以將一個自訂屬性陣列作為第四個參數傳遞給 `Validator::make` 方法：

    $validator = Validator::make($input, $rules, $messages, [
        'email' => 'email address',
    ]);

<a name="performing-additional-validation"></a>
### 執行額外驗證

有時您需要在初始驗證完成後執行額外驗證。您可以使用驗證器的 `after` 方法來完成此操作。`after` 方法接受一個閉包或一個可呼叫的陣列，這些在驗證完成後將被調用。給定的可呼叫項目將收到一個 `Illuminate\Validation\Validator` 實例，讓您可以在必要時引發額外的錯誤訊息：

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

如前所述，`after` 方法也接受一個可呼叫的陣列，如果您的「驗證後」邏輯封裝在可呼叫類別中，這會特別方便，這些類別將透過其 `__invoke` 方法接收一個 `Illuminate\Validation\Validator` 實例：

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

在使用表單請求或手動建立的驗證器實例驗證傳入的請求資料後，您可能希望擷取實際經過驗證的請求資料。這可以透過幾種方式達成。首先，您可以在表單請求或驗證器實例上呼叫 `validated` 方法。此方法會回傳已驗證資料的陣列：

    $validated = $request->validated();

    $validated = $validator->validated();

或者，您可以在表單請求或驗證器實例上呼叫 `safe` 方法。此方法會回傳 `Illuminate\Support\ValidatedInput` 實例。此物件提供 `only`、`except` 和 `all` 方法，以擷取已驗證資料的子集或整個已驗證資料陣列：

    $validated = $request->safe()->only(['name', 'email']);

    $validated = $request->safe()->except(['name', 'email']);

    $validated = $request->safe()->all();

此外，`Illuminate\Support\ValidatedInput` 實例可以被迭代並像陣列一樣存取：

    // Validated data may be iterated...
    foreach ($request->safe() as $key => $value) {
        // ...
    }

    // Validated data may be accessed as an array...
    $validated = $request->safe();

    $email = $validated['email'];

如果您想為已驗證的資料加入額外欄位，可以呼叫 `merge` 方法：

    $validated = $request->safe()->merge(['name' => 'Taylor Otwell']);

如果您想將已驗證的資料作為 [collection](/docs/{{version}}/collections) 實例擷取，可以呼叫 `collect` 方法：

    $collection = $request->safe()->collect();


<a name="working-with-error-messages"></a>
## 處理錯誤訊息

在 `Validator` 實例上呼叫 `errors` 方法後，您將會收到一個 `Illuminate\Support\MessageBag` 實例，它提供各種便捷的方法來處理錯誤訊息。自動提供給所有視圖的 `$errors` 變數也是 `MessageBag` 類別的實例。


<a name="retrieving-the-first-error-message-for-a-field"></a>
#### 擷取欄位的第一個錯誤訊息

要擷取指定欄位的第一個錯誤訊息，請使用 `first` 方法：

    $errors = $validator->errors();

    echo $errors->first('email');


<a name="retrieving-all-error-messages-for-a-field"></a>
#### 擷取欄位的所有錯誤訊息

如果您需要擷取指定欄位的所有訊息陣列，請使用 `get` 方法：

    foreach ($errors->get('email') as $message) {
        // ...
    }

如果您正在驗證陣列表單欄位，您可以使用 `*` 字元擷取每個陣列元素的所有訊息：

    foreach ($errors->get('attachments.*') as $message) {
        // ...
    }


<a name="retrieving-all-error-messages-for-all-fields"></a>
#### 擷取所有欄位的所有錯誤訊息

要擷取所有欄位的所有訊息陣列，請使用 `all` 方法：

    foreach ($errors->all() as $message) {
        // ...
    }


<a name="determining-if-messages-exist-for-a-field"></a>
#### 判斷欄位是否存在訊息

`has` 方法可用於判斷指定欄位是否存在任何錯誤訊息：

    if ($errors->has('email')) {
        // ...
    }


<a name="specifying-custom-messages-in-language-files"></a>
### 在語言檔中指定自訂訊息

Laravel 內建的驗證規則，每個都有一個錯誤訊息，位於應用程式的 `lang/en/validation.php` 檔案中。如果您的應用程式沒有 `lang` 目錄，您可以透過 `lang:publish` Artisan 命令指示 Laravel 建立它。

在 `lang/en/validation.php` 檔案中，您會找到每個驗證規則的翻譯項目。您可以根據應用程式的需求自由更改或修改這些訊息。

此外，您可以將此檔案複製到另一個語言目錄，以翻譯應用程式語言的訊息。要了解更多關於 Laravel 本地化的資訊，請查閱完整的 [本地化文件](/docs/{{version}}/localization)。

> [!WARNING]  
> 預設情況下，Laravel 應用程式骨架不包含 `lang` 目錄。如果您想自訂 Laravel 的語言檔案，可以透過 `lang:publish` Artisan 命令發佈它們。


<a name="custom-messages-for-specific-attributes"></a>
#### 特定屬性的自訂訊息

您可以在應用程式的驗證語言檔中，為指定的屬性和規則組合自訂錯誤訊息。要這麼做，請將您的訊息自訂內容新增到應用程式 `lang/xx/validation.php` 語言檔的 `custom` 陣列中：

    'custom' => [
        'email' => [
            'required' => 'We need to know your email address!',
            'max' => 'Your email address is too long!'
        ],
    ],


<a name="specifying-attribute-in-language-files"></a>
### 在語言檔中指定屬性

Laravel 的許多內建錯誤訊息都包含一個 `:attribute` 佔位符，該佔位符會替換為正在驗證的欄位或屬性名稱。如果您希望驗證訊息的 `:attribute` 部分替換為自訂值，您可以在 `lang/xx/validation.php` 語言檔的 `attributes` 陣列中指定自訂屬性名稱：

    'attributes' => [
        'email' => 'email address',
    ],

> [!WARNING]  
> 預設情況下，Laravel 應用程式骨架不包含 `lang` 目錄。如果您想自訂 Laravel 的語言檔案，可以透過 `lang:publish` Artisan 命令發佈它們。


<a name="specifying-values-in-language-files"></a>
### 在語言檔中指定值

Laravel 的某些內建驗證規則錯誤訊息包含一個 `:value` 佔位符，該佔位符會替換為請求屬性的當前值。然而，您偶爾可能需要將驗證訊息的 `:value` 部分替換為值的自訂表示。例如，考慮以下規則，該規則指定如果 `payment_type` 的值為 `cc` 則需要信用卡號碼：

    Validator::make($request->all(), [
        'credit_card_number' => 'required_if:payment_type,cc'
    ]);

如果此驗證規則失敗，它將產生以下錯誤訊息：

```none
The credit card number field is required when payment type is cc.
```

與其將 `cc` 顯示為付款類型值，您可以在 `lang/xx/validation.php` 語言檔中定義一個 `values` 陣列，以指定一個更友善的表示值：

    'values' => [
        'payment_type' => [
            'cc' => 'credit card'
        ],
    ],

> [!WARNING]  
> 預設情況下，Laravel 應用程式骨架不包含 `lang` 目錄。如果您想自訂 Laravel 的語言檔案，可以透過 `lang:publish` Artisan 命令發佈它們。

定義此值後，驗證規則將產生以下錯誤訊息：

```none
The credit card number field is required when payment type is credit card.
```

<a name="available-validation-rules"></a>
## 可用的驗證規則

下方列出了所有可用的驗證規則及其功能：

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
[Distinct](#rule-distinct)
[In Array](#rule-in-array)
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


#### 公用程式

<div class="collection-method-list" markdown="1">

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

驗證中的欄位必須是 `"yes"`、`"on"`、`1`、`"1"`、`true` 或 `"true"`。這對於驗證「服務條款」接受或其他類似欄位非常有用。


<a name="rule-accepted-if"></a>
#### accepted_if:anotherfield,value,...

驗證中的欄位必須是 `"yes"`、`"on"`、`1`、`"1"`、`true` 或 `"true"`，如果另一個驗證中的欄位等於指定的值。這對於驗證「服務條款」接受或其他類似欄位非常有用。


<a name="rule-active-url"></a>
#### active_url

驗證中的欄位必須根據 PHP `dns_get_record` 函式擁有有效的 A 或 AAAA 紀錄。提供的 URL 主機名稱會在使用 `dns_get_record` 前，先透過 PHP `parse_url` 函式提取。


<a name="rule-after"></a>
#### after:_date_

驗證中的欄位必須是在指定日期之後的值。日期將會傳入 PHP `strtotime` 函式，以轉換為有效的 `DateTime` 實例：

    'start_date' => 'required|date|after:tomorrow'

您可以指定另一個欄位來比較日期，而不是傳遞一個日期字串讓 `strtotime` 評估：

    'finish_date' => 'required|date|after:start_date'

為了方便起見，日期規則可以使用流暢的 `date` 規則建立器來建構：

    use Illuminate\Validation\Rule;

    'start_date' => [
        'required',
        Rule::date()->after(today()->addDays(7)),
    ],

方法 `afterToday` 和 `todayOrAfter` 可用於流暢地表示日期必須分別在今天之後、或今天之後及包含今天：

    'start_date' => [
        'required',
        Rule::date()->afterToday(),
    ],


<a name="rule-after-or-equal"></a>
#### after\_or\_equal:_date_

驗證中的欄位必須是在指定日期之後或相等的值。更多資訊請參閱 [after](#rule-after) 規則。

為了方便起見，日期規則可以使用流暢的 `date` 規則建立器來建構：

    use Illuminate\Validation\Rule;

    'start_date' => [
        'required',
        Rule::date()->afterOrEqual(today()->addDays(7)),
    ],


<a name="rule-alpha"></a>
#### alpha

驗證中的欄位必須完全是 [`\p{L}`](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AL%3A%5D&g=&i=) 和 [`\p{M}`](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AM%3A%5D&g=&i=) 中包含的 Unicode 字母字元。

若要將此驗證規則限制為 ASCII 範圍 (`a-z` 和 `A-Z`) 中的字元，您可以為此驗證規則提供 `ascii` 選項：

```php
'username' => 'alpha:ascii',
```


<a name="rule-alpha-dash"></a>
#### alpha_dash

驗證中的欄位必須完全是 [`\p{L}`](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AL%3A%5D&g=&i=)、[`\p{M}`](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AM%3A%5D&g=&i=)、[`\p{N}`](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AN%3A%5D&g=&i=) 中包含的 Unicode 英數字元，以及 ASCII 破折號 (`-`) 和 ASCII 底線 (`_`)。

若要將此驗證規則限制為 ASCII 範圍 (`a-z` 和 `A-Z`) 中的字元，您可以為此驗證規則提供 `ascii` 選項：

```php
'username' => 'alpha_dash:ascii',
```

<a name="rule-alpha-num"></a>
#### alpha_num

被驗證的欄位必須完全由包含在 [`\p{L}`](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AL%3A%5D&g=&i=)、[`\p{M}`](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AM%3A%5D&g=&i=) 和 [`\p{N}`](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AN%3A%5D&g=&i=) 中的 Unicode 字母數字字元組成。

要將此驗證規則限制為 ASCII 範圍 (`a-z` 和 `A-Z`) 中的字元，您可以為驗證規則提供 `ascii` 選項：

```php
'username' => 'alpha_num:ascii',
```

<a name="rule-array"></a>
#### array

被驗證的欄位必須是 PHP `array`。

當提供額外值給 `array` 規則時，輸入陣列中的每個鍵都必須存在於提供給該規則的值列表中。在以下範例中，輸入陣列中的 `admin` 鍵是無效的，因為它不包含在提供給 `array` 規則的值列表中：

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

一般來說，您應該始終指定允許出現在陣列中的鍵。

<a name="rule-ascii"></a>
#### ascii

被驗證的欄位必須完全由 7 位元 ASCII 字元組成。

<a name="rule-bail"></a>
#### bail

在第一次驗證失敗後停止執行該欄位的驗證規則。

雖然 `bail` 規則只會在特定欄位遇到驗證失敗時停止驗證該欄位，但 `stopOnFirstFailure` 方法會通知驗證器，一旦發生單一驗證失敗，就應停止驗證所有屬性：

    if ($validator->stopOnFirstFailure()->fails()) {
        // ...
    }

<a name="rule-before"></a>
#### before:_date_

被驗證的欄位必須是一個早於指定日期的值。日期將傳遞給 PHP `strtotime` 函數以轉換為有效的 `DateTime` 實例。此外，與 [`after`](#rule-after) 規則一樣，也可以將另一個被驗證欄位的名稱作為 `date` 的值。

為了方便，基於日期的規則也可以使用 Fluent `date` 規則建構器來建構：

    use Illuminate\Validation\Rule;

    'start_date' => [
        'required',
        Rule::date()->before(today()->subDays(7)),
    ],

`beforeToday` 和 `todayOrBefore` 方法可以用來流暢地表達日期必須在今天之前，或者在今天或今天之前：

    'start_date' => [
        'required',
        Rule::date()->beforeToday(),
    ],

<a name="rule-before-or-equal"></a>
#### before\_or\_equal:_date_

被驗證的欄位必須是一個早於或等於指定日期的值。日期將傳遞給 PHP `strtotime` 函數以轉換為有效的 `DateTime` 實例。此外，與 [`after`](#rule-after) 規則一樣，也可以將另一個被驗證欄位的名稱作為 `date` 的值。

為了方便，基於日期的規則也可以使用 Fluent `date` 規則建構器來建構：

    use Illuminate\Validation\Rule;

    'start_date' => [
        'required',
        Rule::date()->beforeOrEqual(today()->subDays(7)),
    ],

<a name="rule-between"></a>
#### between:_min_,_max_

被驗證的欄位的大小必須在給定的 _min_ 和 _max_ 之間 (包含兩者)。字串、數字、陣列和檔案的評估方式與 [`size`](#rule-size) 規則相同。

<a name="rule-boolean"></a>
#### boolean

被驗證的欄位必須可以被轉換為布林值。接受的輸入值為 `true`、`false`、`1`、`0`、`"1"` 和 `"0"`。

<a name="rule-confirmed"></a>
#### confirmed

被驗證的欄位必須有一個匹配的 `{field}_confirmation` 欄位。例如，如果被驗證的欄位是 `password`，那麼輸入中必須存在一個匹配的 `password_confirmation` 欄位。

您也可以傳遞自訂的確認欄位名稱。例如，`confirmed:repeat_username` 會預期 `repeat_username` 欄位與被驗證的欄位匹配。

<a name="rule-contains"></a>
#### contains:_foo_,_bar_,...

被驗證的欄位必須是一個包含所有指定參數值的陣列。

<a name="rule-current-password"></a>
#### current_password

被驗證的欄位必須與已驗證使用者的密碼匹配。您可以使用規則的第一個參數來指定 [authentication guard](/docs/{{version}}/authentication)：

    'password' => 'current_password:api'

<a name="rule-date"></a>
#### date

被驗證的欄位必須是根據 PHP `strtotime` 函數的有效且非相對日期。

<a name="rule-date-equals"></a>
#### date\_equals:_date_

被驗證的欄位必須等於指定日期。日期將傳遞給 PHP `strtotime` 函數以轉換為有效的 `DateTime` 實例。

<a name="rule-date-format"></a>
#### date\_format:_format_,...

被驗證的欄位必須匹配指定的 _formats_ 之一。驗證欄位時，您應該**只使用** `date` 或 `date_format`，不能兩者都用。此驗證規則支援 PHP [DateTime](https://www.php.net/manual/en/class.datetime.php) 類別所支援的所有格式。

為了方便，基於日期的規則可以使用 Fluent `date` 規則建構器來建構：

    use Illuminate\Validation\Rule;

    'start_date' => [
        'required',
        Rule::date()->format('Y-m-d'),
    ],

<a name="rule-decimal"></a>
#### decimal:_min_,_max_

被驗證的欄位必須是數字，並且必須包含指定的小數位數：

    // 必須恰好有兩位小數 (9.99)...
    'price' => 'decimal:2'

    // 必須有 2 到 4 位小數...
    'price' => 'decimal:2,4'

<a name="rule-declined"></a>
#### declined

被驗證的欄位必須是 `"no"`、`"off"`、`0`、`"0"`、`false` 或 `"false"`。

<a name="rule-declined-if"></a>
#### declined\_if:anotherfield,value,...

如果另一個被驗證的欄位等於指定值，則被驗證的欄位必須是 `"no"`、`"off"`、`0`、`"0"`、`false` 或 `"false"`。

<a name="rule-different"></a>
#### different:_field_

被驗證的欄位必須與 _field_ 有不同的值。

<a name="rule-digits"></a>
#### digits:_value_

被驗證的整數必須具有 _value_ 的確切長度。

<a name="rule-digits-between"></a>
#### digits\_between:_min_,_max_

整數驗證必須具有介於給定 _min_ 和 _max_ 之間的長度。

<a name="rule-dimensions"></a>
#### dimensions

被驗證的檔案必須是符合規則參數指定尺寸限制的圖片：

    'avatar' => 'dimensions:min_width=100,min_height=200'

可用的限制條件有：_min\_width_、_max\_width_、_min\_height_、_max\_height_、_width_、_height_、_ratio_。

_ratio_ 限制應表示為寬度除以高度。這可以透過分數如 `3/2` 或浮點數如 `1.5` 來指定：

    'avatar' => 'dimensions:ratio=3/2'

由於此規則需要多個參數，因此使用 `Rule::dimensions` 方法流暢地建構規則通常更方便：

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

<a name="rule-distinct"></a>
#### distinct

驗證陣列時，被驗證的欄位不得有任何重複的值：

    'foo.*.id' => 'distinct'

Distinct 預設使用寬鬆的變數比較。若要使用嚴格比較，您可以將 `strict` 參數新增到驗證規則定義中：

    'foo.*.id' => 'distinct:strict'

您可以將 `ignore_case` 新增到驗證規則的參數中，以使規則忽略大小寫差異：

    'foo.*.id' => 'distinct:ignore_case'

<a name="rule-doesnt-start-with"></a>
#### doesnt_start_with:_foo_,_bar_,...

被驗證的欄位不能以給定的其中一個值開頭。


<a name="rule-doesnt-end-with"></a>
#### doesnt_end_with:_foo_,_bar_,...

被驗證的欄位不能以給定的其中一個值結尾。


<a name="rule-email"></a>
#### email

被驗證的欄位必須是電子郵件地址格式。此驗證規則使用 [`egulias/email-validator`](https://github.com/egulias/EmailValidator) 套件來驗證電子郵件地址。預設情況下，`RFCValidation` 驗證器會被套用，但您也可以套用其他驗證樣式：

    'email' => 'email:rfc,dns'

上述範例將套用 `RFCValidation` 和 `DNSCheckValidation` 驗證。以下是您可以套用的驗證樣式完整列表：

<div class="content-list" markdown="1">

- `rfc`: `RFCValidation` - 根據 RFC 5322 驗證電子郵件地址。
- `strict`: `NoRFCWarningsValidation` - 根據 RFC 5322 驗證電子郵件，拒絕結尾句號或多個連續句號。
- `dns`: `DNSCheckValidation` - 確保電子郵件地址的網域具有有效的 MX 記錄。
- `spoof`: `SpoofCheckValidation` - 確保電子郵件地址不包含同形異義字或欺騙性 Unicode 字元。
- `filter`: `FilterEmailValidation` - 確保電子郵件地址根據 PHP 的 `filter_var` 函數是有效的。
- `filter_unicode`: `FilterEmailValidation::unicode()` - 確保電子郵件地址根據 PHP 的 `filter_var` 函數是有效的，允許某些 Unicode 字元。

</div>

為了方便，電子郵件驗證規則可以使用流暢的規則建構器來建構：

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

被驗證的欄位必須以給定的其中一個值結尾。


<a name="rule-enum"></a>
#### enum

`Enum` 規則是一個基於類別的規則，用於驗證被驗證的欄位是否包含有效的 enum 值。`Enum` 規則接受 enum 的名稱作為其唯一的建構子引數。當驗證原始值時，應將一個 backed Enum 提供給 `Enum` 規則：

    use App\Enums\ServerStatus;
    use Illuminate\Validation\Rule;

    $request->validate([
        'status' => [Rule::enum(ServerStatus::class)],
    ]);

`Enum` 規則的 `only` 和 `except` 方法可用於限制哪些 enum 案例應被視為有效：

    Rule::enum(ServerStatus::class)
        ->only([ServerStatus::Pending, ServerStatus::Active]);

    Rule::enum(ServerStatus::class)
        ->except([ServerStatus::Pending, ServerStatus::Active]);

`when` 方法可用於有條件地修改 `Enum` 規則：

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

被驗證的欄位將從 `validate` 和 `validated` 方法返回的請求資料中排除。


<a name="rule-exclude-if"></a>
#### exclude_if:_anotherfield_,_value_

如果 _anotherfield_ 欄位等於 _value_，則被驗證的欄位將從 `validate` 和 `validated` 方法返回的請求資料中排除。

如果需要複雜的條件式排除邏輯，您可以使用 `Rule::excludeIf` 方法。此方法接受一個布林值或一個閉包。當給予一個閉包時，該閉包應返回 `true` 或 `false`，以指示是否應排除被驗證的欄位：

    use Illuminate\Support\Facades\Validator;
    use Illuminate\Validation\Rule;

    Validator::make($request->all(), [
        'role_id' => Rule::excludeIf($request->user()->is_admin),
    ]);

    Validator::make($request->all(), [
        'role_id' => Rule::excludeIf(fn () => $request->user()->is_admin),
    ]);


<a name="rule-exclude-unless"></a>
#### exclude_unless:_anotherfield_,_value_

除非 _anotherfield_ 欄位等於 _value_，否則被驗證的欄位將從 `validate` 和 `validated` 方法返回的請求資料中排除。如果 _value_ 為 `null` (`exclude_unless:name,null`)，則除非比較欄位為 `null` 或比較欄位在請求資料中遺失，否則被驗證的欄位將被排除。


<a name="rule-exclude-with"></a>
#### exclude_with:_anotherfield_

如果 _anotherfield_ 欄位存在，則被驗證的欄位將從 `validate` 和 `validated` 方法返回的請求資料中排除。


<a name="rule-exclude-without"></a>
#### exclude_without:_anotherfield_

如果 _anotherfield_ 欄位不存在，則被驗證的欄位將從 `validate` 和 `validated` 方法返回的請求資料中排除。


<a name="rule-exists"></a>
#### exists:_table_,_column_

被驗證的欄位必須存在於給定的資料庫資料表中。


<a name="basic-usage-of-exists-rule"></a>
#### exists 規則的基本用法

    'state' => 'exists:states'

如果沒有指定 `column` 選項，將使用欄位名稱。因此，在此情況下，該規則將驗證 `states` 資料庫資料表是否包含一條 `state` 欄位值與請求的 `state` 屬性值相符的記錄。


<a name="specifying-a-custom-column-name"></a>
#### 指定自訂欄位名稱

您可以透過將資料庫欄位名稱放在資料庫資料表名稱之後，明確指定驗證規則應使用的資料庫欄位名稱：

    'state' => 'exists:states,abbreviation'

有時，您可能需要指定用於 `exists` 查詢的特定資料庫連線。您可以透過在資料表名稱前加上連線名稱來實現此目的：

    'email' => 'exists:connection.staff,email'

您不必直接指定資料表名稱，您可以指定應用於確定資料表名稱的 Eloquent 模型：

    'user_id' => 'exists:App\Models\User,id'

如果您想自訂驗證規則執行的查詢，可以使用 `Rule` 類別以流暢的方式定義規則。在此範例中，我們還會將驗證規則指定為陣列，而不是使用 `|` 字元來分隔它們：

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

您可以透過將欄位名稱作為第二個引數提供給 `exists` 方法，明確指定由 `Rule::exists` 方法生成的 `exists` 規則應使用的資料庫欄位名稱：

    'state' => Rule::exists('states', 'abbreviation'),


<a name="rule-extensions"></a>
#### extensions:_foo_,_bar_,...

被驗證的檔案必須具有與列出的其中一個副檔名相對應的使用者指定的副檔名：

    'photo' => ['required', 'extensions:jpg,png'],

> [!WARNING]  
> 您絕不應單獨依靠使用者指定的副檔名來驗證檔案。此規則通常應與 [`mimes`](#rule-mimes) 或 [`mimetypes`](#rule-mimetypes) 規則結合使用。


<a name="rule-file"></a>
#### file

被驗證的欄位必須是成功上傳的檔案。


<a name="rule-filled"></a>
#### filled

當存在時，被驗證的欄位不能為空。


<a name="rule-gt"></a>
#### gt:_field_

被驗證的欄位必須大於給定的 _field_ 或 _value_。這兩個欄位必須是相同類型。字串、數字、陣列和檔案是使用與 [`size`](#rule-size) 規則相同的約定進行評估。

<a name="rule-gte"></a>
#### gte:_field_

驗證中的欄位必須大於或等於給定的 _field_ 或 _value_。這兩個欄位必須是相同的型別。字串、數值、陣列和檔案的評估方式與 [`size`](#rule-size) 規則相同。

<a name="rule-hex-color"></a>
#### hex_color

驗證中的欄位必須包含有效的 [十六進位](https://developer.mozilla.org/en-US/docs/Web/CSS/hex-color) 顏色值。

<a name="rule-image"></a>
#### image

驗證中的檔案必須是圖片 (jpg, jpeg, png, bmp, gif, svg 或 webp)。

<a name="rule-in"></a>
#### in:_foo_,_bar_,...

驗證中的欄位必須包含在給定的值清單中。由於此規則通常要求您 `implode` 一個陣列，因此可以使用 `Rule::in` 方法流暢地建構規則：

    use Illuminate\Support\Facades\Validator;
    use Illuminate\Validation\Rule;

    Validator::make($data, [
        'zones' => [
            'required',
            Rule::in(['first-zone', 'second-zone']),
        ],
    ]);

當 `in` 規則與 `array` 規則結合時，輸入陣列中的每個值都必須存在於 `in` 規則提供的值清單中。在以下範例中，輸入陣列中的 `LAS` 機場代碼無效，因為它不包含在 `in` 規則提供的機場清單中：

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

<a name="rule-in-array"></a>
#### in_array:_anotherfield_.*

驗證中的欄位必須存在於 _anotherfield_ 的值中。

<a name="rule-integer"></a>
#### integer

驗證中的欄位必須是整數。

> [!WARNING]  
> 此驗證規則不會驗證輸入是否為「整數」變數型別，只會驗證輸入是否為 PHP `FILTER_VALIDATE_INT` 規則所接受的型別。如果您需要驗證輸入為數字，請將此規則與 [數值驗證規則 (`numeric`)](#rule-numeric) 結合使用。

<a name="rule-ip"></a>
#### ip

驗證中的欄位必須是 IP 位址。

<a name="ipv4"></a>
#### ipv4

驗證中的欄位必須是 IPv4 位址。

<a name="ipv6"></a>
#### ipv6

驗證中的欄位必須是 IPv6 位址。

<a name="rule-json"></a>
#### json

驗證中的欄位必須是有效的 JSON 字串。

<a name="rule-lt"></a>
#### lt:_field_

驗證中的欄位必須小於給定的 _field_。這兩個欄位必須是相同的型別。字串、數值、陣列和檔案的評估方式與 [`size`](#rule-size) 規則相同。

<a name="rule-lte"></a>
#### lte:_field_

驗證中的欄位必須小於或等於給定的 _field_。這兩個欄位必須是相同的型別。字串、數值、陣列和檔案的評估方式與 [`size`](#rule-size) 規則相同。

<a name="rule-lowercase"></a>
#### lowercase

驗證中的欄位必須是小寫。

<a name="rule-list"></a>
#### list

驗證中的欄位必須是列表陣列。如果陣列的鍵由 0 到 `count($array) - 1` 的連續數字組成，則該陣列被視為列表。

<a name="rule-mac"></a>
#### mac_address

驗證中的欄位必須是 MAC 位址。

<a name="rule-max"></a>
#### max:_value_

驗證中的欄位必須小於或等於最大 _value_。字串、數值、陣列和檔案的評估方式與 [`size`](#rule-size) 規則相同。

<a name="rule-max-digits"></a>
#### max_digits:_value_

驗證中的整數必須具有最大長度 _value_。

<a name="rule-mimetypes"></a>
#### mimetypes:_text/plain_,...

驗證中的檔案必須符合給定的 MIME 型別之一：

    'video' => 'mimetypes:video/avi,video/mpeg,video/quicktime'

為了確定上傳檔案的 MIME 型別，框架將讀取檔案內容並嘗試猜測 MIME 型別，這可能與客戶端提供的 MIME 型別不同。

<a name="rule-mimes"></a>
#### mimes:_foo_,_bar_,...

驗證中的檔案必須具有與所列副檔名之一相對應的 MIME 型別：

    'photo' => 'mimes:jpg,bmp,png'

即使您只需要指定副檔名，此規則實際上會透過讀取檔案內容並猜測其 MIME 型別來驗證檔案的 MIME 型別。MIME 型別及其對應副檔名的完整列表可在此處找到：

[https://svn.apache.org/repos/asf/httpd/httpd/trunk/docs/conf/mime.types](https://svn.apache.org/repos/asf/httpd/httpd/trunk/docs/conf/mime.types)

<a name="mime-types-and-extensions"></a>
#### MIME 型別與副檔名

此驗證規則不會驗證 MIME 型別與使用者指定給檔案的副檔名之間的一致性。例如，`mimes:png` 驗證規則會認為包含有效 PNG 內容的檔案是有效的 PNG 圖片，即使該檔案命名為 `photo.txt`。如果您想驗證使用者指定給檔案的副檔名，可以使用 [`extensions`](#rule-extensions) 規則。

<a name="rule-min"></a>
#### min:_value_

驗證中的欄位必須具有最小 _value_。字串、數值、陣列和檔案的評估方式與 [`size`](#rule-size) 規則相同。

<a name="rule-min-digits"></a>
#### min_digits:_value_

驗證中的整數必須具有最小長度 _value_。

<a name="rule-multiple-of"></a>
#### multiple_of:_value_

驗證中的欄位必須是 _value_ 的倍數。

<a name="rule-missing"></a>
#### missing

驗證中的欄位不得出現在輸入資料中。

<a name="rule-missing-if"></a>
#### missing_if:_anotherfield_,_value_,...

如果 _anotherfield_ 欄位等於任何 _value_，則驗證中的欄位不得出現在輸入資料中。

<a name="rule-missing-unless"></a>
#### missing_unless:_anotherfield_,_value_

除非 _anotherfield_ 欄位等於任何 _value_，否則驗證中的欄位不得出現在輸入資料中。

<a name="rule-missing-with"></a>
#### missing_with:_foo_,_bar_,...

驗證中的欄位不得出現在輸入資料中，_僅當_ 任何其他指定欄位存在時。

<a name="rule-missing-with-all"></a>
#### missing_with_all:_foo_,_bar_,...

驗證中的欄位不得出現在輸入資料中，_僅當_ 所有其他指定欄位都存在時。

<a name="rule-not-in"></a>
#### not_in:_foo_,_bar_,...

驗證中的欄位不得包含在給定的值清單中。可以使用 `Rule::notIn` 方法流暢地建構規則：

    use Illuminate\Validation\Rule;

    Validator::make($data, [
        'toppings' => [
            'required',
            Rule::notIn(['sprinkles', 'cherries']),
        ],
    ]);

<a name="rule-not-regex"></a>
#### not_regex:_pattern_

驗證中的欄位不得符合給定的正規表達式。

在內部，此規則使用 PHP `preg_match` 函數。指定的模式應遵守 `preg_match` 所要求的相同格式，因此也應包含有效的分隔符。例如：`'email' => 'not_regex:/^.+$/i'`。

> [!WARNING]  
> 使用 `regex` / `not_regex` 模式時，可能需要使用陣列而不是 `|` 分隔符來指定驗證規則，特別是當正規表達式包含 `|` 字元時。

<a name="rule-nullable"></a>
#### nullable

驗證中的欄位可以為 `null`。

<a name="rule-numeric"></a>
#### numeric

驗證中的欄位必須是 [數值型](https://www.php.net/manual/en/function.is-numeric.php)。

<a name="rule-present"></a>
#### present

驗證中的欄位必須存在於輸入資料中。

<a name="rule-present-if"></a>
#### present_if:_anotherfield_,_value_,...

如果 _anotherfield_ 欄位等於任何 _value_，則驗證中的欄位必須存在。

<a name="rule-present-unless"></a>
#### present_unless:_anotherfield_,_value_

待驗證欄位必須存在，除非 _anotherfield_ 欄位等於任何 _value_。


<a name="rule-present-with"></a>
#### present_with:_foo_,_bar_,...

待驗證欄位必須存在，**僅當**任何其他指定欄位存在時。


<a name="rule-present-with-all"></a>
#### present_with_all:_foo_,_bar_,...

待驗證欄位必須存在，**僅當**所有其他指定欄位都存在時。


<a name="rule-prohibited"></a>
#### prohibited

待驗證欄位必須不存在或為空。欄位為「空」的條件如下：

<div class="content-list" markdown="1">

- 值為 `null`。
- 值為空字串。
- 值為空陣列或空的 `Countable` 物件。
- 值為路徑為空的已上傳檔案。

</div>


<a name="rule-prohibited-if"></a>
#### prohibited_if:_anotherfield_,_value_,...

待驗證欄位必須不存在或為空，如果 _anotherfield_ 欄位等於任何 _value_。欄位為「空」的條件如下：

<div class="content-list" markdown="1">

- 值為 `null`。
- 值為空字串。
- 值為空陣列或空的 `Countable` 物件。
- 值為路徑為空的已上傳檔案。

</div>

如果需要複雜的條件式禁止邏輯，您可以使用 `Rule::prohibitedIf` 方法。此方法接受一個布林值或一個閉包。當給定閉包時，閉包應回傳 `true` 或 `false` 以指示待驗證欄位是否應被禁止：

    use Illuminate\Support\Facades\Validator;
    use Illuminate\Validation\Rule;

    Validator::make($request->all(), [
        'role_id' => Rule::prohibitedIf($request->user()->is_admin),
    ]);

    Validator::make($request->all(), [
        'role_id' => Rule::prohibitedIf(fn () => $request->user()->is_admin),
    ]);


<a name="rule-prohibited-unless"></a>
#### prohibited_unless:_anotherfield_,_value_,...

待驗證欄位必須不存在或為空，除非 _anotherfield_ 欄位等於任何 _value_。欄位為「空」的條件如下：

<div class="content-list" markdown="1">

- 值為 `null`。
- 值為空字串。
- 值為空陣列或空的 `Countable` 物件。
- 值為路徑為空的已上傳檔案。

</div>


<a name="rule-prohibits"></a>
#### prohibits:_anotherfield_,...

如果待驗證欄位不存在或不為空，則 _anotherfield_ 中的所有欄位必須不存在或為空。欄位為「空」的條件如下：

<div class="content-list" markdown="1">

- 值為 `null`。
- 值為空字串。
- 值為空陣列或空的 `Countable` 物件。
- 值為路徑為空的已上傳檔案。

</div>


<a name="rule-regex"></a>
#### regex:_pattern_

待驗證欄位必須符合給定的正規表達式。

在內部，此規則使用 PHP `preg_match` 函式。指定的模式應遵循 `preg_match` 所需的相同格式，因此也應包含有效的分隔符號。例如：`'email' => 'regex:/^.+@.+$/i'`。

> [!WARNING]  
> 當使用 `regex` / `not_regex` 模式時，可能需要將規則指定為陣列而不是使用 `|` 分隔符號，特別是當正規表達式包含 `|` 字元時。


<a name="rule-required"></a>
#### required

待驗證欄位必須存在於輸入資料中且不為空。欄位為「空」的條件如下：

<div class="content-list" markdown="1">

- 值為 `null`。
- 值為空字串。
- 值為空陣列或空的 `Countable` 物件。
- 值為路徑為空的已上傳檔案。

</div>


<a name="rule-required-if"></a>
#### required_if:_anotherfield_,_value_,...

待驗證欄位必須存在且不為空，如果 _anotherfield_ 欄位等於任何 _value_。

如果您想為 `required_if` 規則建構更複雜的條件，您可以使用 `Rule::requiredIf` 方法。此方法接受一個布林值或一個閉包。當傳入閉包時，閉包應回傳 `true` 或 `false` 以指示待驗證欄位是否為必填：

    use Illuminate\Support\Facades\Validator;
    use Illuminate\Validation\Rule;

    Validator::make($request->all(), [
        'role_id' => Rule::requiredIf($request->user()->is_admin),
    ]);

    Validator::make($request->all(), [
        'role_id' => Rule::requiredIf(fn () => $request->user()->is_admin),
    ]);


<a name="rule-required-if-accepted"></a>
#### required_if_accepted:_anotherfield_,...

待驗證欄位必須存在且不為空，如果 _anotherfield_ 欄位等於 `"yes"`、`"on"`、`1`、`"1"`、`true` 或 `"true"`。


<a name="rule-required-if-declined"></a>
#### required_if_declined:_anotherfield_,...

待驗證欄位必須存在且不為空，如果 _anotherfield_ 欄位等於 `"no"`、`"off"`、`0`、`"0"`、`false` 或 `"false"`。


<a name="rule-required-unless"></a>
#### required_unless:_anotherfield_,_value_,...

待驗證欄位必須存在且不為空，除非 _anotherfield_ 欄位等於任何 _value_。這也表示 _anotherfield_ 必須存在於請求資料中，除非 _value_ 為 `null`。如果 _value_ 為 `null` (`required_unless:name,null`)，則待驗證欄位為必填，除非比較欄位為 `null` 或該比較欄位不存在於請求資料中。


<a name="rule-required-with"></a>
#### required_with:_foo_,_bar_,...

待驗證欄位必須存在且不為空，**僅當**任何其他指定欄位存在且不為空時。


<a name="rule-required-with-all"></a>
#### required_with_all:_foo_,_bar_,...

待驗證欄位必須存在且不為空，**僅當**所有其他指定欄位都存在且不為空時。


<a name="rule-required-without"></a>
#### required_without:_foo_,_bar_,...

待驗證欄位必須存在且不為空，**僅當**任何其他指定欄位為空或不存在時。


<a name="rule-required-without-all"></a>
#### required_without_all:_foo_,_bar_,...

待驗證欄位必須存在且不為空，**僅當**所有其他指定欄位為空或不存在時。


<a name="rule-required-array-keys"></a>
#### required_array_keys:_foo_,_bar_,...

待驗證欄位必須是陣列，並且必須包含至少指定的鍵。


<a name="rule-same"></a>
#### same:_field_

給定的 _field_ 必須與待驗證欄位匹配。


<a name="rule-size"></a>
#### size:_value_

待驗證欄位的大小必須符合給定的 _value_。對於字串資料，_value_ 對應於字元數。對於數值資料，_value_ 對應於給定的整數值（該屬性也必須具有 `numeric` 或 `integer` 規則）。對於陣列，_size_ 對應於陣列的 `count`。對於檔案，_size_ 對應於以 kilobyte 為單位的檔案大小。讓我們看一些例子：

    // Validate that a string is exactly 12 characters long...
    'title' => 'size:12';

    // Validate that a provided integer equals 10...
    'seats' => 'integer|size:10';

    // Validate that an array has exactly 5 elements...
    'tags' => 'array|size:5';

    // Validate that an uploaded file is exactly 512 kilobytes...
    'image' => 'file|size:512';


<a name="rule-starts-with"></a>
#### starts_with:_foo_,_bar_,...

待驗證欄位必須以給定值之一開頭。


<a name="rule-string"></a>
#### string

待驗證欄位必須是字串。如果您想允許該欄位也為 `null`，則應為該欄位指定 `nullable` 規則。

<a name="rule-timezone"></a>
#### timezone

待驗證的欄位必須是根據 `DateTimeZone::listIdentifiers` 方法的有效時區識別符。

[由 `DateTimeZone::listIdentifiers` 方法接受的參數](https://www.php.net/manual/en/datetimezone.listidentifiers.php) 也可用於此驗證規則：

    'timezone' => 'required|timezone:all';

    'timezone' => 'required|timezone:Africa';

    'timezone' => 'required|timezone:per_country,US';


<a name="rule-unique"></a>
#### unique:_table_,_column_

待驗證的欄位值不得存在於給定的資料庫表格中。

**指定自訂表格/欄位名稱：**

除了直接指定表格名稱，您也可以指定 Eloquent Model 來決定表格名稱：

    'email' => 'unique:App\Models\User,email_address'

您可以使用 `column` 選項來指定欄位對應的資料庫欄位。如果未指定 `column` 選項，則會使用待驗證欄位的名稱。

    'email' => 'unique:users,email_address'

**指定自訂資料庫連線**

有時，您可能需要為 Validator 所做的資料庫查詢設定自訂連線。您可以透過在表格名稱前加上連線名稱來達成此目的：

    'email' => 'unique:connection.users,email_address'

**強制 unique 規則忽略給定 ID：**

有時，您可能希望在 unique 驗證期間忽略給定的 ID。例如，考慮一個「更新個人資料」畫面，其中包含使用者的姓名、電子郵件地址和位置。您可能希望驗證電子郵件地址是唯一的。然而，如果使用者只更改了姓名欄位而未更改電子郵件欄位，您就不希望拋出驗證錯誤，因為該使用者已經是該電子郵件地址的擁有者。

為了指示驗證器忽略使用者的 ID，我們將使用 `Rule` 類別以流暢的方式定義規則。在此範例中，我們還將驗證規則指定為陣列，而不是使用 `|` 字元來分隔規則：

    use Illuminate\Support\Facades\Validator;
    use Illuminate\Validation\Rule;

    Validator::make($data, [
        'email' => [
            'required',
            Rule::unique('users')->ignore($user->id),
        ],
    ]);

> [!WARNING]  
> 您絕不應將任何使用者控制的請求輸入傳遞給 `ignore` 方法。相反，您應該只傳遞系統生成的唯一 ID，例如 Eloquent Model 實例中的自動遞增 ID 或 UUID。否則，您的應用程式將容易受到 SQL injection 攻擊。

除了將模型鍵的值傳遞給 `ignore` 方法外，您還可以傳遞整個模型實例。Laravel 將自動從模型中提取鍵：

    Rule::unique('users')->ignore($user)

如果您的表格使用的主鍵欄位名稱不是 `id`，您可以在呼叫 `ignore` 方法時指定該欄位的名稱：

    Rule::unique('users')->ignore($user->id, 'user_id')

依預設，`unique` 規則會檢查與待驗證屬性名稱相符的欄位唯一性。然而，您可以將不同的欄位名稱作為第二個參數傳遞給 `unique` 方法：

    Rule::unique('users', 'email_address')->ignore($user->id)

**新增額外條件子句：**

您可以透過使用 `where` 方法自訂查詢來指定額外的查詢條件。例如，讓我們新增一個查詢條件，將查詢範圍限定為僅搜尋 `account_id` 欄位值為 `1` 的記錄：

    'email' => Rule::unique('users')->where(fn (Builder $query) => $query->where('account_id', 1))

**在唯一性檢查中忽略軟刪除記錄：**

依預設，unique 規則在判斷唯一性時會包含軟刪除記錄。若要從唯一性檢查中排除軟刪除記錄，您可以呼叫 `withoutTrashed` 方法：

    Rule::unique('users')->withoutTrashed();

如果您的模型使用的主鍵欄位名稱不是 `deleted_at`，您可以在呼叫 `withoutTrashed` 方法時指定該欄位的名稱：

    Rule::unique('users')->withoutTrashed('was_deleted_at');


<a name="rule-uppercase"></a>
#### uppercase

待驗證的欄位必須為大寫。


<a name="rule-url"></a>
#### url

待驗證的欄位必須是有效的 URL。

如果您想指定應視為有效的 URL 協定，您可以將協定作為驗證規則參數傳遞：

```php
'url' => 'url:http,https',

'game' => 'url:minecraft,steam',
```


<a name="rule-ulid"></a>
#### ulid

待驗證的欄位必須是有效的 [Universally Unique Lexicographically Sortable Identifier](https://github.com/ulid/spec) (ULID)。


<a name="rule-uuid"></a>
#### uuid

待驗證的欄位必須是有效的 RFC 9562 (版本 1, 3, 4, 5, 6, 7, 或 8) 普遍唯一識別碼 (UUID)。

<a name="conditionally-adding-rules"></a>
## 有條件地加入規則

<a name="skipping-validation-when-fields-have-certain-values"></a>
#### 當欄位具有特定值時跳過驗證

您可能偶爾會希望在另一個欄位具有特定值時，不對某個給定欄位進行驗證。您可以使用 `exclude_if` 驗證規則來達成此目的。在這個範例中，如果 `has_appointment` 欄位的值為 `false`，則 `appointment_date` 和 `doctor_name` 欄位將不會被驗證：

    use Illuminate\Support\Facades\Validator;

    $validator = Validator::make($data, [
        'has_appointment' => 'required|boolean',
        'appointment_date' => 'exclude_if:has_appointment,false|required|date',
        'doctor_name' => 'exclude_if:has_appointment,false|required|string',
    ]);

或者，您可以使用 `exclude_unless` 規則，除非另一個欄位具有給定值，否則不對給定欄位進行驗證：

    $validator = Validator::make($data, [
        'has_appointment' => 'required|boolean',
        'appointment_date' => 'exclude_unless:has_appointment,true|required|date',
        'doctor_name' => 'exclude_unless:has_appointment,true|required|string',
    ]);

<a name="validating-when-present"></a>
#### 僅當欄位存在時才驗證

在某些情況下，您可能希望**僅在**該欄位存在於要驗證的資料中時，才對其執行驗證檢查。為了快速達成此目的，請將 `sometimes` 規則加入到您的規則清單中：

    $validator = Validator::make($data, [
        'email' => 'sometimes|required|email',
    ]);

在上述範例中，`email` 欄位只會在 `$data` 陣列中存在時才會被驗證。

> [!NOTE]
> 如果您要驗證一個應該始終存在但可能為空的欄位，請參閱[關於選填欄位的注意事項](#a-note-on-optional-fields)。

<a name="complex-conditional-validation"></a>
#### 複雜的條件式驗證

有時您可能希望根據更複雜的條件邏輯來添加驗證規則。例如，您可能希望只有在另一個欄位的值大於 100 時，才要求一個給定欄位。或者，您可能需要兩個欄位只有在另一個欄位存在時才具有給定值。添加這些驗證規則不應該很麻煩。首先，使用您永遠不會改變的_靜態規則_來建立一個 `Validator` 實例：

    use Illuminate\Support\Facades\Validator;

    $validator = Validator::make($request->all(), [
        'email' => 'required|email',
        'games' => 'required|numeric',
    ]);

讓我們假設我們的網路應用程式是用於遊戲收藏家。如果一位遊戲收藏家在我們的應用程式中註冊，並且他們擁有超過 100 款遊戲，我們希望他們解釋為何擁有這麼多遊戲。例如，他們可能是經營遊戲轉售商店，或者只是喜歡收藏遊戲。為了有條件地添加此要求，我們可以使用 `Validator` 實例上的 `sometimes` 方法。

    use Illuminate\Support\Fluent;

    $validator->sometimes('reason', 'required|max:500', function (Fluent $input) {
        return $input->games >= 100;
    });

傳遞給 `sometimes` 方法的第一個參數是我們要進行條件式驗證的欄位名稱。第二個參數是我們要添加的規則列表。如果作為第三個參數傳遞的閉包回傳 `true`，則這些規則將會被添加。這個方法讓建立複雜的條件式驗證變得輕而易舉。您甚至可以同時為多個欄位添加條件式驗證：

    $validator->sometimes(['reason', 'cost'], 'required', function (Fluent $input) {
        return $input->games >= 100;
    });

> [!NOTE]
> 傳遞給您的閉包的 `$input` 參數將是 `Illuminate\Support\Fluent` 的實例，可用於存取您要驗證的輸入資料和檔案。

<a name="complex-conditional-array-validation"></a>
#### 複雜的條件式陣列驗證

有時您可能希望根據同一個巢狀陣列中另一個您不知道索引的欄位來驗證某個欄位。在這些情況下，您可以讓您的閉包接收第二個參數，該參數將是正在驗證的陣列中的當前個別項目：

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

與傳遞給閉包的 `$input` 參數類似，當屬性資料是陣列時，`$item` 參數是 `Illuminate\Support\Fluent` 的實例；否則，它是一個字串。

<a name="validating-arrays"></a>
## 驗證陣列

如 [`array` 驗證規則文件](#rule-array) 中所述，`array` 規則接受一個允許的陣列鍵列表。如果陣列中存在任何額外鍵值，驗證將會失敗：

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

通常來說，您應總是指定允許存在於陣列中的鍵。否則，驗證器的 `validate` 和 `validated` 方法將會回傳所有已驗證的資料，包含陣列及其所有鍵，即使這些鍵未經其他巢狀陣列驗證規則驗證。

<a name="validating-nested-array-input"></a>
### 驗證巢狀陣列輸入

驗證基於巢狀陣列的表單輸入欄位不一定是件麻煩事。您可以使用「點式標記法」來驗證陣列中的屬性。例如，如果傳入的 HTTP 請求包含一個 `photos[profile]` 欄位，您可以這樣驗證：

    use Illuminate\Support\Facades\Validator;

    $validator = Validator::make($request->all(), [
        'photos.profile' => 'required|image',
    ]);

您也可以驗證陣列中的每個元素。例如，要驗證指定陣列輸入欄位中的每個 email 都獨一無二，您可以這樣做：

    $validator = Validator::make($request->all(), [
        'person.*.email' => 'email|unique:users',
        'person.*.first_name' => 'required_with:person.*.last_name',
    ]);

同樣地，在 [語言檔中指定自訂驗證訊息](#custom-messages-for-specific-attributes) 時，您可以使用 `*` 字元，使得為基於陣列的欄位使用單一驗證訊息變得輕而易舉：

    'custom' => [
        'person.*.email' => [
            'unique' => 'Each person must have a unique email address',
        ]
    ],

<a name="accessing-nested-array-data"></a>
#### 存取巢狀陣列資料

有時，在將驗證規則指派給屬性時，您可能需要存取指定巢狀陣列元素的值。您可以使用 `Rule::forEach` 方法來實現此目的。`forEach` 方法接受一個閉包，該閉包將在驗證陣列屬性的每次迭代中被呼叫，並會接收屬性的值，以及明確、完整展開的屬性名稱。此閉包應回傳一個規則陣列，以指派給該陣列元素：

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

<a name="error-message-indexes-and-positions"></a>
### 錯誤訊息索引與位置

驗證陣列時，您可能希望在應用程式顯示的錯誤訊息中引用驗證失敗的特定項目的索引或位置。為此，您可以在 [自訂驗證訊息](#manual-customizing-the-error-messages) 中包含 `:index` (從 `0` 開始) 和 `:position` (從 `1` 開始) 佔位符：

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

根據上述範例，驗證將會失敗，使用者將會看到以下錯誤：「請描述照片 #2。」

如有必要，您可以透過 `second-index`、`second-position`、`third-index`、`third-position` 等方式，引用更深層次的巢狀索引和位置。

    'photos.*.attributes.*.string' => 'Invalid attribute for photo #:second-position.',

<a name="validating-files"></a>
## 驗證檔案

Laravel 提供多種可用於驗證上傳檔案的規則，例如 `mimes`、`image`、`min` 和 `max`。儘管您可以單獨指定這些規則，但 Laravel 也提供一個流暢的檔案驗證規則建構器，您可能會覺得很方便：

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

<a name="validating-files-file-types"></a>
#### 驗證檔案類型

儘管您在呼叫 `types` 方法時只需要指定副檔名，但此方法實際上是透過讀取檔案內容並猜測其 MIME 類型來驗證檔案的 MIME 類型。完整的 MIME 類型及其對應副檔名列表可於以下位置找到：

[https://svn.apache.org/repos/asf/httpd/httpd/trunk/docs/conf/mime.types](https://svn.apache.org/repos/asf/httpd/httpd/trunk/docs/conf/mime.types)

<a name="validating-files-file-sizes"></a>
#### 驗證檔案大小

為求方便，最小和最大檔案大小可以字串形式指定，並帶有表示檔案大小單位的後綴。支援 `kb`、`mb`、`gb` 和 `tb` 後綴：

```php
File::types(['mp3', 'wav'])
    ->min('1kb')
    ->max('10mb');
```

<a name="validating-files-image-files"></a>
#### 驗證圖片檔案

要驗證上傳的檔案是否為圖片，您可以使用 `File` 規則的 `image` 建構式方法。`File::image()` 規則確保受驗證的檔案為圖片 (jpg、jpeg、png、bmp、gif、svg 或 webp)：

```php
use Illuminate\Support\Facades\Validator;
use Illuminate\Validation\Rule;
use Illuminate\Validation\Rules\File;

Validator::validate($input, [
    'photo' => [
        'required',
        File::image(),
    ],
]);
```

<a name="validating-files-image-dimensions"></a>
#### 驗證圖片尺寸

您也可以驗證圖片的尺寸。例如，要驗證上傳的圖片至少寬 1000 像素，高 500 像素，您可以使用 `dimensions` 規則：

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

為了確保密碼具備足夠的複雜度，你可以使用 Laravel 的 `Password` 規則物件：

    use Illuminate\Support\Facades\Validator;
    use Illuminate\Validation\Rules\Password;

    $validator = Validator::make($request->all(), [
        'password' => ['required', 'confirmed', Password::min(8)],
    ]);

`Password` 規則物件允許你輕鬆自訂應用程式的密碼複雜度要求，例如指定密碼至少需要一個字母、數字、符號或混合大小寫的字元：

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

此外，你可以使用 `uncompromised` 方法來確保密碼未在公開的密碼資料洩露中被洩露：

    Password::min(8)->uncompromised()

在內部，`Password` 規則物件使用 [k-Anonymity](https://en.wikipedia.org/wiki/K-anonymity) 模型來判斷密碼是否已透過 [haveibeenpwned.com](https://haveibeenpwned.com) 服務洩露，同時不犧牲使用者的隱私或安全性。

預設情況下，如果密碼在資料洩露中至少出現一次，則將被視為已洩露。你可以使用 `uncompromised` 方法的第一個引數來自訂此閾值：

    // Ensure the password appears less than 3 times in the same data leak...
    Password::min(8)->uncompromised(3);

當然，你可以鏈接上述所有方法：

    Password::min(8)
        ->letters()
        ->mixedCase()
        ->numbers()
        ->symbols()
        ->uncompromised()


<a name="defining-default-password-rules"></a>
#### 定義預設密碼規則

你可能會覺得方便，在應用程式的單一位置指定密碼的預設驗證規則。你可以使用 `Password::defaults` 方法輕鬆實現此目的，該方法接受一個閉包。提供給 `defaults` 方法的閉包應回傳 Password 規則的預設組態。通常，`defaults` 規則應在應用程式的其中一個服務提供者 `boot` 方法中呼叫：

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

然後，當你想要將預設規則應用於正在驗證的特定密碼時，可以不帶任何引數地呼叫 `defaults` 方法：

    'password' => ['required', Password::defaults()],

有時，你可能希望將額外的驗證規則附加到你的預設密碼驗證規則中。你可以使用 `rules` 方法來實現此目的：

    use App\Rules\ZxcvbnRule;

    Password::defaults(function () {
        $rule = Password::min(8)->rules([new ZxcvbnRule]);

        // ...
    });

<a name="custom-validation-rules"></a>
## 自訂驗證規則

<a name="using-rule-objects"></a>
### 使用規則物件

Laravel 提供了各種實用的驗證規則；然而，您可能希望指定一些自己的規則。註冊自訂驗證規則的一種方法是使用規則物件。若要產生新的規則物件，您可以使用 `make:rule` Artisan 命令。讓我們使用此命令來產生一個驗證字串是否為大寫的規則。Laravel 會將新的規則放置於 `app/Rules` 目錄中。如果此目錄不存在，當您執行 Artisan 命令建立規則時，Laravel 會自動建立它：

```shell
php artisan make:rule Uppercase
```

規則建立後，我們就可以定義其行為。規則物件包含一個方法：`validate`。此方法會接收屬性名稱、其值，以及一個在驗證失敗時應被呼叫並帶有驗證錯誤訊息的回呼：

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

規則定義完成後，您可以將規則物件的實例與其他驗證規則一起傳遞，將其附加到驗證器：

    use App\Rules\Uppercase;

    $request->validate([
        'name' => ['required', 'string', new Uppercase],
    ]);

#### 翻譯驗證訊息

您可以不向 `$fail` 閉包提供字面錯誤訊息，而是提供一個[翻譯字串鍵](/docs/{{version}}/localization)並指示 Laravel 翻譯錯誤訊息：

    if (strtoupper($value) !== $value) {
        $fail('validation.uppercase')->translate();
    }

如有必要，您可以將佔位符替換和首選語言作為 `translate` 方法的第一個和第二個參數提供：

    $fail('validation.location')->translate([
        'value' => $this->value,
    ], 'fr')

#### 存取額外資料

如果您的自訂驗證規則類別需要存取所有其他正在驗證的資料，您的規則類別可以實作 `Illuminate\Contracts\Validation\DataAwareRule` 介面。此介面要求您的類別定義一個 `setData` 方法。此方法將由 Laravel 自動呼叫 (在驗證進行之前)，並帶有所有正在驗證的資料：

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

或者，如果您的驗證規則需要存取執行驗證的驗證器實例，您可以實作 `ValidatorAwareRule` 介面：

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

<a name="using-closures"></a>
### 使用閉包

如果您只需要在應用程式中一次性使用自訂規則的功能，您可以使用閉包而不是規則物件。閉包會接收屬性名稱、屬性值，以及一個在驗證失敗時應該被呼叫的 `$fail` 回呼：

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

<a name="implicit-rules"></a>
### 隱含規則

預設情況下，當正在驗證的屬性不存在或包含空字串時，正常的驗證規則（包括自訂規則）不會執行。例如，`unique` 規則不會針對空字串執行：

    use Illuminate\Support\Facades\Validator;

    $rules = ['name' => 'unique:users,name'];

    $input = ['name' => ''];

    Validator::make($input, $rules)->passes(); // true

為了讓自訂規則即使在屬性為空時也能執行，該規則必須隱含該屬性是必需的。若要快速產生新的隱含規則物件，您可以使用 `make:rule` Artisan 命令，並加上 `--implicit` 選項：

```shell
php artisan make:rule Uppercase --implicit
```

> [!WARNING]
> 「隱含」規則僅**暗示**該屬性是必需的。它是否真的會將遺失或空的屬性視為無效，則取決於您。