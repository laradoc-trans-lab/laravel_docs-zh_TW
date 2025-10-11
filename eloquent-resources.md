# Eloquent: API 資源

- [簡介](#introduction)
- [生成資源](#generating-resources)
- [概念概覽](#concept-overview)
    - [資源集合](#resource-collections)
- [撰寫資源](#writing-resources)
    - [資料包裝](#data-wrapping)
    - [分頁](#pagination)
    - [條件式屬性](#conditional-attributes)
    - [條件式關聯](#conditional-relationships)
    - [新增中繼資料](#adding-meta-data)
- [資源回應](#resource-responses)

<a name="introduction"></a>
## 簡介

建立 API 時，您可能需要一個轉換層，介於 Eloquent 模型與實際回傳給應用程式使用者的 JSON 回應之間。舉例來說，您可能希望僅向部分使用者顯示某些屬性，而不向其他使用者顯示；或者您可能希望在模型的 JSON 表示法中始終包含某些關聯。Eloquent 的資源類別讓您能夠明確且輕鬆地將模型與模型集合轉換為 JSON。

當然，您始終可以使用 Eloquent 模型或集合的 `toJson` 方法將其轉換為 JSON；然而，Eloquent 資源提供了對模型及其關聯的 JSON 序列化更精細且強大的控制。

<a name="generating-resources"></a>
## 生成資源

要生成資源類別，您可以使用 `make:resource` Artisan 命令。預設情況下，資源會被放置在應用程式的 `app/Http/Resources` 目錄中。資源繼承自 `Illuminate\Http\Resources\Json\JsonResource` 類別：

```shell
php artisan make:resource UserResource
```

<a name="generating-resource-collections"></a>
#### 資源集合

除了生成轉換單一模型的資源外，您還可以生成負責轉換模型集合的資源。這使得您的 JSON 回應可以包含與指定資源的整個集合相關的連結及其他中繼資訊。

要建立資源集合，您應在建立資源時使用 `--collection` 旗標。或者，在資源名稱中包含 `Collection` 一詞將指示 Laravel 應建立一個集合資源。集合資源繼承自 `Illuminate\Http\Resources\Json\ResourceCollection` 類別：

```shell
php artisan make:resource User --collection

php artisan make:resource UserCollection
```

<a name="concept-overview"></a>
## 概念概覽

> [!NOTE]  
> 這是資源與資源集合的高層次概覽。強烈建議您閱讀本文檔的其他章節，以深入了解資源為您提供的自訂與強大功能。

在深入探討撰寫資源時所有可用的選項之前，讓我們先高層次地了解資源如何在 Laravel 中使用。資源類別代表一個需要轉換為 JSON 結構的單一模型。舉例來說，這是一個簡單的 `UserResource` 資源類別：

    <?php

    namespace App\Http\Resources;

    use Illuminate\Http\Request;
    use Illuminate\Http\Resources\Json\JsonResource;

    class UserResource extends JsonResource
    {
        /**
         * Transform the resource into an array.
         *
         * @return array<string, mixed>
         */
        public function toArray(Request $request): array
        {
            return [
                'id' => $this->id,
                'name' => $this->name,
                'email' => $this->email,
                'created_at' => $this->created_at,
                'updated_at' => $this->updated_at,
            ];
        }
    }

每個資源類別都定義了一個 `toArray` 方法，該方法返回一個屬性陣列，當資源從路由或控制器方法作為回應返回時，這些屬性應轉換為 JSON。

請注意，我們可以直接從 `$this` 變數存取模型屬性。這是因為資源類別會自動將屬性與方法的存取代理到底層模型，以便於存取。一旦資源被定義，它就可以從路由或控制器返回。資源透過其建構子接受底層模型實例：

    use App\Http\Resources\UserResource;
    use App\Models\User;

    Route::get('/user/{id}', function (string $id) {
        return new UserResource(User::findOrFail($id));
    });

<a name="resource-collections"></a>
### 資源集合

如果您正在返回資源集合或分頁回應，您應該在路由或控制器中建立資源實例時，使用資源類別提供的 `collection` 方法：

    use App\Http\Resources\UserResource;
    use App\Models\User;

    Route::get('/users', function () {
        return UserResource::collection(User::all());
    });

請注意，這不允許為您的集合新增任何可能需要返回的自訂中繼資料。如果您想自訂資源集合回應，您可以建立一個專門的資源來代表該集合：

```shell
php artisan make:resource UserCollection
```

一旦資源集合類別生成，您可以輕鬆定義應包含在回應中的任何中繼資料：

    <?php

    namespace App\Http\Resources;

    use Illuminate\Http\Request;
    use Illuminate\Http\Resources\Json\ResourceCollection;

    class UserCollection extends ResourceCollection
    {
        /**
         * Transform the resource collection into an array.
         *
         * @return array<int|string, mixed>
         */
        public function toArray(Request $request): array
        {
            return [
                'data' => $this->collection,
                'links' => [
                    'self' => 'link-value',
                ],
            ];
        }
    }

定義您的資源集合後，它就可以從路由或控制器返回：

    use App\Http\Resources\UserCollection;
    use App\Models\User;

    Route::get('/users', function () {
        return new UserCollection(User::all());
    });

<a name="preserving-collection-keys"></a>
#### 保留集合鍵值

從路由返回資源集合時，Laravel 會重設集合的鍵值，使其依數值順序排列。然而，您可以在資源類別中新增一個 `preserveKeys` 屬性，以指示是否應保留集合的原始鍵值：

    <?php

    namespace App\Http\Resources;

    use Illuminate\Http\Resources\Json\JsonResource;

    class UserResource extends JsonResource
    {
        /**
         * Indicates if the resource's collection keys should be preserved.
         *
         * @var bool
         */
        public $preserveKeys = true;
    }

當 `preserveKeys` 屬性設為 `true` 時，當集合從路由或控制器返回時，集合鍵值將會被保留。

    use App\Http\Resources\UserResource;
    use App\Models\User;

    Route::get('/users', function () {
        return UserResource::collection(User::all()->keyBy->id);
    });

<a name="customizing-the-underlying-resource-class"></a>
#### 自訂底層資源類別

通常，資源集合的 `$this->collection` 屬性會自動填充，其結果是將集合中的每個項目映射到其單一資源類別。單一資源類別假定為集合類別名稱，但不包含類別名稱末尾的 `Collection` 部分。此外，根據您的個人偏好，單一資源類別可能帶有 `Resource` 後綴，也可能不帶。

舉例來說，`UserCollection` 將嘗試把給定的使用者實例映射到 `UserResource` 資源。要自訂此行為，您可以覆寫資源集合的 `$collects` 屬性：

    <?php

    namespace App\Http\Resources;

    use Illuminate\Http\Resources\Json\ResourceCollection;

    class UserCollection extends ResourceCollection
    {
        /**
         * The resource that this resource collects.
         *
         * @var string
         */
        public $collects = Member::class;
    }

<a name="writing-resources"></a>
## 撰寫資源

> [!NOTE]  
> 如果您尚未閱讀 [概念概覽](#concept-overview)，強烈建議您在繼續閱讀本文件之前先閱讀該部分。

資源只需將給定的模型轉換為陣列。因此，每個資源都包含一個 `toArray` 方法，該方法會將您的模型屬性轉換為可從應用程式路由或控制器回傳的 API 友善陣列：

    <?php

    namespace App\Http\Resources;

    use Illuminate\Http\Request;
    use Illuminate\Http\Resources\Json\JsonResource;

    class UserResource extends JsonResource
    {
        /**
         * Transform the resource into an array.
         *
         * @return array<string, mixed>
         */
        public function toArray(Request $request): array
        {
            return [
                'id' => $this->id,
                'name' => $this->name,
                'email' => $this->email,
                'created_at' => $this->created_at,
                'updated_at' => $this->updated_at,
            ];
        }
    }

一旦資源被定義，它就可以直接從路由或控制器中回傳：

    use App\Http\Resources\UserResource;
    use App\Models\User;

    Route::get('/user/{id}', function (string $id) {
        return new UserResource(User::findOrFail($id));
    });


<a name="relationships"></a>
#### 關聯

如果您希望在回應中包含相關資源，可以將它們添加到資源 `toArray` 方法回傳的陣列中。在此範例中，我們將使用 `PostResource` 資源的 `collection` 方法將使用者的部落格文章添加到資源回應中：

    use App\Http\Resources\PostResource;
    use Illuminate\Http\Request;

    /**
     * Transform the resource into an array.
     *
     * @return array<string, mixed>
     */
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'email' => $this->email,
            'posts' => PostResource::collection($this->posts),
            'created_at' => $this->created_at,
            'updated_at' => $this->updated_at,
        ];
    }

> [!NOTE]  
> 如果您希望僅在關聯已經載入的情況下才包含它們，請查閱 [條件式關聯](#conditional-relationships) 的文件。


<a name="writing-resource-collections"></a>
#### 資源集合

資源將單一模型轉換為陣列，而資源集合則將模型集合轉換為陣列。然而，並非絕對需要為每個模型定義一個資源集合類別，因為所有資源都提供了一個 `collection` 方法來即時生成一個「臨時」資源集合：

    use App\Http\Resources\UserResource;
    use App\Models\User;

    Route::get('/users', function () {
        return UserResource::collection(User::all());
    });

但是，如果您需要自訂與集合一起回傳的中繼資料，則有必要定義自己的資源集合：

    <?php

    namespace App\Http\Resources;

    use Illuminate\Http\Request;
    use Illuminate\Http\Resources\Json\ResourceCollection;

    class UserCollection extends ResourceCollection
    {
        /**
         * Transform the resource collection into an array.
         *
         * @return array<string, mixed>
         */
        public function toArray(Request $request): array
        {
            return [
                'data' => $this->collection,
                'links' => [
                    'self' => 'link-value',
                ],
            ];
        }
    }

就像單一資源一樣，資源集合可以直接從路由或控制器中回傳：

    use App\Http\Resources\UserCollection;
    use App\Models\User;

    Route::get('/users', function () {
        return new UserCollection(User::all());
    });


<a name="data-wrapping"></a>
### 資料包裝

預設情況下，當資源回應轉換為 JSON 時，最外層的資源會被包裝在 `data` 鍵中。因此，例如，典型的資源集合回應如下所示：

```json
{
    "data": [
        {
            "id": 1,
            "name": "Eladio Schroeder Sr.",
            "email": "therese28@example.com"
        },
        {
            "id": 2,
            "name": "Liliana Mayert",
            "email": "evandervort@example.com"
        }
    ]
}
```

如果您希望禁用最外層資源的包裝，您應該在基礎 `Illuminate\Http\Resources\Json\JsonResource` 類別上呼叫 `withoutWrapping` 方法。通常，您應該從您的 `AppServiceProvider` 或應用程式每次請求都會載入的其他 [服務提供者](/docs/{{version}}/providers) 中呼叫此方法：

    <?php

    namespace App\Providers;

    use Illuminate\Http\Resources\Json\JsonResource;
    use Illuminate\Support\ServiceProvider;

    class AppServiceProvider extends ServiceProvider
    {
        /**
         * Register any application services.
         */
        public function register(): void
        {
            // ...
        }

        /**
         * Bootstrap any application services.
         */
        public function boot(): void
        {
            JsonResource::withoutWrapping();
        }
    }

> [!WARNING]  
> `withoutWrapping` 方法僅影響最外層的回應，並不會移除您手動新增到自己資源集合中的 `data` 鍵。


<a name="wrapping-nested-resources"></a>
#### 包裝巢狀資源

您可以完全自由地決定如何包裝資源的關聯。如果您希望所有資源集合（無論其巢狀層級如何）都包裝在 `data` 鍵中，則應為每個資源定義一個資源集合類別，並在 `data` 鍵中回傳該集合。

您可能想知道這是否會導致您的最外層資源被包裝在兩個 `data` 鍵中。不用擔心，Laravel 絕不會讓您的資源意外地重複包裝，因此您不必擔心您正在轉換的資源集合的巢狀層級：

    <?php

    namespace App\Http\Resources;

    use Illuminate\Http\Resources\Json\ResourceCollection;

    class CommentsCollection extends ResourceCollection
    {
        /**
         * Transform the resource collection into an array.
         *
         * @return array<string, mixed>
         */
        public function toArray(Request $request): array
        {
            return ['data' => $this->collection];
        }
    }


<a name="data-wrapping-and-pagination"></a>
#### 資料包裝與分頁

當透過資源回應回傳分頁集合時，即使已呼叫 `withoutWrapping` 方法，Laravel 仍會將您的資源資料包裝在 `data` 鍵中。這是因為分頁回應總是包含帶有分頁器狀態資訊的 `meta` 和 `links` 鍵：

```json
{
    "data": [
        {
            "id": 1,
            "name": "Eladio Schroeder Sr.",
            "email": "therese28@example.com"
        },
        {
            "id": 2,
            "name": "Liliana Mayert",
            "email": "evandervort@example.com"
        }
    ],
    "links":{
        "first": "http://example.com/users?page=1",
        "last": "http://example.com/users?page=1",
        "prev": null,
        "next": null
    },
    "meta":{
        "current_page": 1,
        "from": 1,
        "last_page": 1,
        "path": "http://example.com/users",
        "per_page": 15,
        "to": 10,
        "total": 10
    }
}
```

<a name="pagination"></a>
### 分頁

您可以將 Laravel 分頁器實例傳遞給資源的 `collection` 方法，或傳遞給自訂的資源集合：

    use App\Http\Resources\UserCollection;
    use App\Models\User;

    Route::get('/users', function () {
        return new UserCollection(User::paginate());
    });

分頁回應始終包含 `meta` 和 `links` 鍵，其中帶有關於分頁器狀態的資訊：

```json
{
    "data": [
        {
            "id": 1,
            "name": "Eladio Schroeder Sr.",
            "email": "therese28@example.com"
        },
        {
            "id": 2,
            "name": "Liliana Mayert",
            "email": "evandervort@example.com"
        }
    ],
    "links":{
        "first": "http://example.com/users?page=1",
        "last": "http://example.com/users?page=1",
        "prev": null,
        "next": null
    },
    "meta":{
        "current_page": 1,
        "from": 1,
        "last_page": 1,
        "path": "http://example.com/users",
        "per_page": 15,
        "to": 10,
        "total": 10
    }
}
```

<a name="customizing-the-pagination-information"></a>
#### 自訂分頁資訊

如果您想自訂分頁回應中 `links` 或 `meta` 鍵所包含的資訊，您可以在資源上定義一個 `paginationInformation` 方法。此方法將接收 `$paginated` 資料和 `$default` 資訊陣列，該陣列包含 `links` 和 `meta` 鍵：

    /**
     * Customize the pagination information for the resource.
     *
     * @param  \Illuminate\Http\Request  $request
     * @param  array $paginated
     * @param  array $default
     * @return array
     */
    public function paginationInformation($request, $paginated, $default)
    {
        $default['links']['custom'] = 'https://example.com';

        return $default;
    }

<a name="conditional-attributes"></a>
### 條件式屬性

有時您可能希望僅在滿足特定條件時，才在資源回應中包含某個屬性。例如，您可能希望只有當前使用者是「administrator」時才包含某個值。Laravel 提供了多種輔助方法來幫助您處理這種情況。`when` 方法可用於有條件地將屬性添加到資源回應中：

    /**
     * Transform the resource into an array.
     *
     * @return array<string, mixed>
     */
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'email' => $this->email,
            'secret' => $this->when($request->user()->isAdmin(), 'secret-value'),
            'created_at' => $this->created_at,
            'updated_at' => $this->updated_at,
        ];
    }

在此範例中，只有當經過身份驗證的使用者之 `isAdmin` 方法回傳 `true` 時，`secret` 鍵才會在最終的資源回應中返回。如果該方法回傳 `false`，則在將回應傳送給客戶端之前，`secret` 鍵將從資源回應中移除。`when` 方法讓您能夠清晰地定義資源，而無需在建構陣列時使用條件語句。

`when` 方法也接受一個閉包作為其第二個參數，讓您僅在給定條件為 `true` 時才計算出結果值：

    'secret' => $this->when($request->user()->isAdmin(), function () {
        return 'secret-value';
    }),

`whenHas` 方法可用於在屬性確實存在於底層模型上時才包含該屬性：

    'name' => $this->whenHas('name'),

此外，`whenNotNull` 方法可用於在屬性不為 null 時，才將該屬性包含在資源回應中：

    'name' => $this->whenNotNull($this->name),

<a name="merging-conditional-attributes"></a>
#### 合併條件式屬性

有時您可能有多個屬性，僅應在滿足相同條件時才包含在資源回應中。在這種情況下，您可以使用 `mergeWhen` 方法，僅在給定條件為 `true` 時才將這些屬性包含在回應中：

    /**
     * Transform the resource into an array.
     *
     * @return array<string, mixed>
     */
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'email' => $this->email,
            $this->mergeWhen($request->user()->isAdmin(), [
                'first-secret' => 'value',
                'second-secret' => 'value',
            ]),
            'created_at' => $this->created_at,
            'updated_at' => $this->updated_at,
        ];
    }

同樣地，如果給定條件為 `false`，則在將回應傳送給客戶端之前，這些屬性將從資源回應中移除。

> [!WARNING]
> `mergeWhen` 方法不應在混合了字串和數字鍵的陣列中使用。此外，它也不應在數字鍵未依序排列的陣列中使用。

<a name="conditional-relationships"></a>
### 條件式關聯

除了條件式載入屬性外，您還可以根據關聯是否已在模型上載入，來條件式地將關聯包含在您的資源回應中。這讓您的控制器可以決定模型上應該載入哪些關聯，而您的資源則可以輕鬆地只在關聯實際載入時才包含它們。最終，這使得在資源中更容易避免「N+1」查詢問題。

`whenLoaded` 方法可用於條件式載入關聯。為了避免不必要的關聯載入，此方法接受關聯的名稱而不是關聯本身：

    use App\Http\Resources\PostResource;

    /**
     * Transform the resource into an array.
     *
     * @return array<string, mixed>
     */
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'email' => $this->email,
            'posts' => PostResource::collection($this->whenLoaded('posts')),
            'created_at' => $this->created_at,
            'updated_at' => $this->updated_at,
        ];
    }

在這個範例中，如果關聯尚未載入，`posts` 鍵將在資源回應傳送給客戶端之前從中移除。

<a name="conditional-relationship-counts"></a>
#### 條件式關聯計數

除了條件式包含關聯之外，您還可以根據關聯的計數是否已在模型上載入，來條件式地將關聯「計數」包含在您的資源回應中：

    new UserResource($user->loadCount('posts'));

`whenCounted` 方法可用於條件式包含關聯的計數在您的資源回應中。如果關聯的計數不存在，此方法會避免不必要地包含該屬性：

    /**
     * Transform the resource into an array.
     *
     * @return array<string, mixed>
     */
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'email' => $this->email,
            'posts_count' => $this->whenCounted('posts'),
            'created_at' => $this->created_at,
            'updated_at' => $this->updated_at,
        ];
    }

在這個範例中，如果 `posts` 關聯的計數尚未載入，`posts_count` 鍵將在資源回應傳送給客戶端之前從中移除。

其他類型的聚合，例如 `avg`、`sum`、`min` 和 `max`，也可以使用 `whenAggregated` 方法進行條件式載入：

```php
'words_avg' => $this->whenAggregated('posts', 'words', 'avg'),
'words_sum' => $this->whenAggregated('posts', 'words', 'sum'),
'words_min' => $this->whenAggregated('posts', 'words', 'min'),
'words_max' => $this->whenAggregated('posts', 'words', 'max'),
```

<a name="conditional-pivot-information"></a>
#### 條件式 Pivot 資訊

除了條件式包含關聯資訊在您的資源回應中，您還可以使用 `whenPivotLoaded` 方法，條件式包含多對多關聯中中間表的資料。`whenPivotLoaded` 方法將 Pivot 表的名稱作為第一個參數。第二個參數應該是一個閉包，如果 Pivot 資訊在模型上可用，則返回要返回的值：

    /**
     * Transform the resource into an array.
     *
     * @return array<string, mixed>
     */
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'expires_at' => $this->whenPivotLoaded('role_user', function () {
                return $this->pivot->expires_at;
            }),
        ];
    }

如果您的關聯使用 [自訂中間表模型](/docs/{{version}}/eloquent-relationships#defining-custom-intermediate-table-models)，您可以將中間表模型的實例作為第一個參數傳遞給 `whenPivotLoaded` 方法：

    'expires_at' => $this->whenPivotLoaded(new Membership, function () {
        return $this->pivot->expires_at;
    }),

如果您的中間表使用的存取器不是 `pivot`，您可以使用 `whenPivotLoadedAs` 方法：

    /**
     * Transform the resource into an array.
     *
     * @return array<string, mixed>
     */
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'expires_at' => $this->whenPivotLoadedAs('subscription', 'role_user', function () {
                return $this->subscription->expires_at;
            }),
        ];
    }

<a name="adding-meta-data"></a>
### 新增中繼資料

一些 JSON API 標準要求在您的資源和資源集合回應中新增中繼資料。這通常包括指向資源或相關資源的 `links`，或關於資源本身的中繼資料。如果您需要返回關於資源的額外中繼資料，請將其包含在您的 `toArray` 方法中。例如，您在轉換資源集合時可能會包含 `links` 資訊：

    /**
     * Transform the resource into an array.
     *
     * @return array<string, mixed>
     */
    public function toArray(Request $request): array
    {
        return [
            'data' => $this->collection,
            'links' => [
                'self' => 'link-value',
            ],
        ];
    }

從您的資源返回額外中繼資料時，您無需擔心在返回分頁回應時，Laravel 自動新增的 `links` 或 `meta` 鍵會被意外覆寫。您定義的任何額外 `links` 將與分頁器提供的連結合併。

<a name="top-level-meta-data"></a>
#### 頂層中繼資料

有時候，您可能希望僅在資源是回傳的最外層資源時，才在資源回應中包含某些中繼資料。通常，這包括關於整個回應的中繼資訊。要定義此中繼資料，請在您的資源類別中新增一個 `with` 方法。此方法應返回一個中繼資料陣列，該陣列僅在資源是正在轉換的最外層資源時才包含在資源回應中：

    <?php

    namespace App\Http\Resources;

    use Illuminate\Http\Resources\Json\ResourceCollection;

    class UserCollection extends ResourceCollection
    {
        /**
         * Transform the resource collection into an array.
         *
         * @return array<string, mixed>
         */
        public function toArray(Request $request): array
        {
            return parent::toArray($request);
        }

        /**
         * Get additional data that should be returned with the resource array.
         *
         * @return array<string, mixed>
         */
        public function with(Request $request): array
        {
            return [
                'meta' => [
                    'key' => 'value',
                ],
            ];
        }
    }

<a name="adding-meta-data-when-constructing-resources"></a>
#### 建構資源時新增中繼資料

您也可以在您的路由或控制器中建構資源實例時新增頂層資料。`additional` 方法適用於所有資源，它接受一個應新增至資源回應的資料陣列：

    return (new UserCollection(User::all()->load('roles')))
        ->additional(['meta' => [
            'key' => 'value',
        ]]);

<a name="resource-responses"></a>
## 資源回應

如您所見，資源可直接從路由和控制器回傳：

    use App\Http\Resources\UserResource;
    use App\Models\User;

    Route::get('/user/{id}', function (string $id) {
        return new UserResource(User::findOrFail($id));
    });

然而，有時您可能需要在 HTTP 回應傳送給用戶端之前，對其進行客製化。有兩種方法可以實現此目的。首先，您可以將 `response` 方法串接到資源上。這個方法會回傳一個 `Illuminate\Http\JsonResponse` 實例，讓您完全控制回應的標頭：

    use App\Http\Resources\UserResource;
    use App\Models\User;

    Route::get('/user', function () {
        return (new UserResource(User::find(1)))
            ->response()
            ->header('X-Value', 'True');
    });

或者，您可以在資源本身中定義一個 `withResponse` 方法。當資源作為回應中最外層的資源回傳時，就會呼叫這個方法：

    <?php

    namespace App\Http\Resources;

    use Illuminate\Http\JsonResponse;
    use Illuminate\Http\Request;
    use Illuminate\Http\Resources\Json\JsonResource;

    class UserResource extends JsonResource
    {
        /**
         * Transform the resource into an array.
         *
         * @return array<string, mixed>
         */
        public function toArray(Request $request): array
        {
            return [
                'id' => $this->id,
            ];
        }

        /**
         * Customize the outgoing response for the resource.
         */
        public function withResponse(Request $request, JsonResponse $response): void
        {
            $response->header('X-Value', 'True');
        }
    }