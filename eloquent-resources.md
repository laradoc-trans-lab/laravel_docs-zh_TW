# Eloquent：API 資源

- [介紹](#introduction)
- [產生資源](#generating-resources)
- [概念總覽](#concept-overview)
    - [資源集合](#resource-collections)
- [撰寫資源](#writing-resources)
    - [資料封裝](#data-wrapping)
    - [分頁](#pagination)
    - [條件式屬性](#conditional-attributes)
    - [條件式關聯](#conditional-relationships)
    - [新增 Meta 資料](#adding-meta-data)
- [資源回應](#resource-responses)

<a name="introduction"></a>
## 介紹

在建構 API 時，你可能需要一個轉換層，位於你的 Eloquent 模型與實際回傳給應用程式使用者的 JSON 回應之間。舉例來說，你可能希望只向部分使用者顯示特定屬性，或者你可能希望在模型的 JSON 表達形式中總是包含某些關聯。Eloquent 的資源類別讓你能清晰且輕鬆地將模型和模型集合轉換成 JSON。

當然，你總是可以使用 Eloquent 模型或集合的 `toJson` 方法將其轉換為 JSON；然而，Eloquent 資源提供了對模型及其關聯的 JSON 序列化更細緻且強大的控制。

<a name="generating-resources"></a>
## 產生資源

若要產生資源類別，你可以使用 `make:resource` Artisan 指令。預設情況下，資源將會被放置在應用程式的 `app/Http/Resources` 目錄中。資源繼承了 `Illuminate\Http\Resources\Json\JsonResource` 類別：

```shell
php artisan make:resource UserResource
```

<a name="generating-resource-collections"></a>
#### 資源集合

除了產生轉換單一模型的資源外，你也可以產生負責轉換模型集合的資源。這讓你的 JSON 回應可以包含與指定資源的整個集合相關的連結及其他 Meta 資訊。

若要建立資源集合，你應該在建立資源時使用 `--collection` 旗標。或者，在資源名稱中包含 `Collection` 這個詞，將會指示 Laravel 應該建立一個集合資源。集合資源繼承了 `Illuminate\Http\Resources\Json\ResourceCollection` 類別：

```shell
php artisan make:resource User --collection

php artisan make:resource UserCollection
```

<a name="concept-overview"></a>
## 概念總覽

> [!NOTE]
> 這是對資源和資源集合的一個高層次概覽。強烈建議您閱讀本文檔的其他章節，以更深入地了解資源所提供的自訂功能和強大能力。

在深入探討撰寫資源時所有可用的選項之前，我們先來概括地了解資源在 Laravel 中的使用方式。資源類別代表需要轉換為 JSON 結構的單一 model。例如，這是一個簡單的 `UserResource` 資源類別：

```php
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
```

每個資源類別都定義了一個 `toArray` 方法，該方法返回一個屬性陣列，當資源從路由或控制器方法作為回應返回時，這些屬性應轉換為 JSON。

請注意，我們可以直接從 `$this` 變數存取 model 屬性。這是因為資源類別會自動將屬性與方法存取代理到底層 model，以方便存取。一旦定義了資源，就可以從路由或控制器返回。資源透過其建構函式接受底層的 model 實例：

```php
use App\Http\Resources\UserResource;
use App\Models\User;

Route::get('/user/{id}', function (string $id) {
    return new UserResource(User::findOrFail($id));
});
```

為了方便，您可以使用 model 的 `toResource` 方法，該方法會利用框架慣例自動探索 model 的底層資源：

```php
return User::findOrFail($id)->toResource();
```

呼叫 `toResource` 方法時，Laravel 會嘗試在最接近 model 命名空間的 `Http\Resources` 命名空間中，尋找名稱與 model 相符且可選地以 `Resource` 結尾的資源。

<a name="resource-collections"></a>
### 資源集合

如果您要返回資源集合或分頁回應，則在路由或控制器中建立資源實例時，應使用資源類別提供的 `collection` 方法：

```php
use App\Http\Resources\UserResource;
use App\Models\User;

Route::get('/users', function () {
    return UserResource::collection(User::all());
});
```

或者，為了方便，您可以使用 Eloquent 集合的 `toResourceCollection` 方法，該方法會利用框架慣例自動探索 model 的底層資源集合：

```php
return User::all()->toResourceCollection();
```

呼叫 `toResourceCollection` 方法時，Laravel 會嘗試在最接近 model 命名空間的 `Http\Resources` 命名空間中，尋找名稱與 model 相符且以 `Collection` 結尾的資源集合。

<a name="custom-resource-collections"></a>
#### 自訂資源集合

預設情況下，資源集合不允許添加任何可能需要隨您的集合一起返回的自訂 meta 資料。如果您想自訂資源集合回應，可以建立一個專用的資源來表示該集合：

```shell
php artisan make:resource UserCollection
```

一旦產生了資源集合類別，您就可以輕鬆定義應包含在回應中的任何 meta 資料：

```php
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
```

定義資源集合後，就可以從路由或控制器返回它：

```php
use App\Http\Resources\UserCollection;
use App\Models\User;

Route::get('/users', function () {
    return new UserCollection(User::all());
});
```

或者，為了方便，您可以使用 Eloquent 集合的 `toResourceCollection` 方法，該方法會利用框架慣例自動探索 model 的底層資源集合：

```php
return User::all()->toResourceCollection();
```

呼叫 `toResourceCollection` 方法時，Laravel 會嘗試在最接近 model 命名空間的 `Http\Resources` 命名空間中，尋找名稱與 model 相符且以 `Collection` 結尾的資源集合。

<a name="preserving-collection-keys"></a>
#### 保留集合鍵

從路由返回資源集合時，Laravel 會重設集合的鍵，使其按數字順序排列。但是，您可以為資源類別添加一個 `preserveKeys` 屬性，指示是否應保留集合的原始鍵：

```php
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
```

當 `preserveKeys` 屬性設為 `true` 時，當集合從路由或控制器返回時，集合鍵將會被保留：

```php
use App\Http\Resources\UserResource;
use App\Models\User;

Route::get('/users', function () {
    return UserResource::collection(User::all()->keyBy->id);
});
```

<a name="customizing-the-underlying-resource-class"></a>
#### 自訂底層資源類別

通常，資源集合的 `$this->collection` 屬性會自動填入將集合中的每個項目映射到其單一資源類別的結果。單一資源類別被假定為集合類別名稱，但不包含末尾的 `Collection` 部分。此外，根據您的個人偏好，單一資源類別可能會或可能不會以 `Resource` 作為字尾。

例如，`UserCollection` 將嘗試將給定的使用者實例映射到 `UserResource` 資源。要自訂此行為，您可以覆寫資源集合的 `$collects` 屬性：

```php
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
```

<a name="writing-resources"></a>
## 撰寫資源

> [!NOTE]
> 如果你尚未閱讀過[概念總覽](#concept-overview)，我們強烈建議你在繼續閱讀本文件前先閱讀它。

資源只需將給定的模型轉換為陣列。因此，每個資源都包含一個 `toArray` 方法，該方法會將你的模型屬性轉換為適合 API 的陣列，可從你的應用程式路由或控制器中返回。

```php
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
```

一旦資源被定義，它就可以直接從路由或控制器中返回。

```php
use App\Models\User;

Route::get('/user/{id}', function (string $id) {
    return User::findOrFail($id)->toUserResource();
});
```

<a name="relationships"></a>
#### 關聯

如果你想在回應中包含相關資源，你可以將它們添加到資源的 `toArray` 方法所返回的陣列中。在此範例中，我們將使用 `PostResource` 資源的 `collection` 方法將使用者的部落格文章添加到資源回應中：

```php
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
```

> [!NOTE]
> 如果你想只在關聯已被載入時才包含它，請查閱關於[條件式關聯](#conditional-relationships)的文件。

<a name="writing-resource-collections"></a>
#### 資源集合

資源將單一模型轉換為陣列，而資源集合則將模型集合轉換為陣列。然而，為每個模型定義一個資源集合類別並非絕對必要，因為所有 Eloquent 模型集合都提供了 `toResourceCollection` 方法，可以即時生成一個「臨時」的資源集合：

```php
use App\Models\User;

Route::get('/users', function () {
    return User::all()->toResourceCollection();
});
```

然而，如果你需要自訂與集合一起返回的 meta 資料，則必須定義自己的資源集合：

```php
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
```

與單一資源一樣，資源集合可以直接從路由或控制器中返回：

```php
use App\Http\Resources\UserCollection;
use App\Models\User;

Route::get('/users', function () {
    return new UserCollection(User::all());
});
```

或者，為了方便起見，你可以使用 Eloquent 集合的 `toResourceCollection` 方法，該方法將利用框架慣例自動發現模型底層的資源集合：

```php
return User::all()->toResourceCollection();
```

當調用 `toResourceCollection` 方法時，Laravel 會嘗試在最接近模型命名空間的 `Http\Resources` 命名空間中，尋找與模型名稱相符並以 `Collection` 作為後綴的資源集合。

<a name="data-wrapping"></a>
### 資料封裝

預設情況下，當資源回應轉換為 JSON 時，最外層的資源會被 `data` 鍵包裝。例如，典型的資源集合回應如下所示：

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

如果你想禁用最外層資源的封裝，你應該在基礎的 `Illuminate\Http\Resources\Json\JsonResource` 類別上調用 `withoutWrapping` 方法。通常，你應該從你的 `AppServiceProvider` 或其他每個應用程式請求都會載入的[服務提供者](/docs/{{version}}/providers)中調用此方法：

```php
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
```

> [!WARNING]
> `withoutWrapping` 方法只會影響最外層的回應，並且不會移除你手動添加到自己的資源集合中的 `data` 鍵。

<a name="wrapping-nested-resources"></a>
#### 封裝巢狀資源

你完全可以自由決定資源的關聯如何被封裝。如果你希望所有資源集合，無論其巢狀層級如何，都封裝在 `data` 鍵中，你應該為每個資源定義一個資源集合類別，並在 `data` 鍵中返回該集合。

你可能會想，這會不會導致最外層資源被兩個 `data` 鍵包裝。別擔心，Laravel 絕不會讓你的資源意外地被雙重包裝，所以你不必擔心你正在轉換的資源集合的巢狀層級：

```php
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
```

<a name="data-wrapping-and-pagination"></a>
#### 資料封裝與分頁

當透過資源回應返回分頁集合時，即使已呼叫 `withoutWrapping` 方法，Laravel 仍會將你的資源資料包裝在 `data` 鍵中。這是因為分頁回應總是包含 `meta` 和 `links` 鍵，其中包含分頁器狀態的資訊：

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

您可以將 Laravel 分頁器實例傳遞給資源的 `collection` 方法或自訂資源集合：

```php
use App\Http\Resources\UserCollection;
use App\Models\User;

Route::get('/users', function () {
    return new UserCollection(User::paginate());
});
```

或者，為方便起見，您可以使用分頁器的 `toResourceCollection` 方法，該方法將利用框架慣例自動探索分頁模型底層的資源集合：

```php
return User::paginate()->toResourceCollection();
```

分頁回應總是包含 `meta` 和 `links` 鍵，其中包含分頁器狀態的資訊：

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

如果您想自訂分頁回應中 `links` 或 `meta` 鍵所包含的資訊，可以在資源上定義 `paginationInformation` 方法。此方法將接收 `$paginated` 資料和 `$default` 資訊陣列，該陣列包含 `links` 和 `meta` 鍵：

```php
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
```


<a name="conditional-attributes"></a>
### 條件式屬性

有時您可能希望僅在符合特定條件時才將屬性包含在資源回應中。例如，您可能希望僅在目前使用者是「管理員」時才包含某個值。Laravel 提供了各種輔助方法來協助處理這種情況。`when` 方法可用於條件式地將屬性新增到資源回應中：

```php
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
```

在此範例中，只有在已認證使用者的 `isAdmin` 方法回傳 `true` 時，`secret` 鍵才會在最終資源回應中回傳。如果該方法回傳 `false`，則 `secret` 鍵將在傳送給客戶端之前從資源回應中移除。`when` 方法讓您能夠以更具表達力的方式定義資源，而無需在建構陣列時訴諸於條件陳述。

`when` 方法也接受一個閉包作為其第二個參數，允許您僅在給定條件為 `true` 時才計算結果值：

```php
'secret' => $this->when($request->user()->isAdmin(), function () {
    return 'secret-value';
}),
```

`whenHas` 方法可用於在底層模型中實際存在屬性時將其包含在內：

```php
'name' => $this->whenHas('name'),
```

此外，`whenNotNull` 方法可用於在屬性不為空值時將其包含在資源回應中：

```php
'name' => $this->whenNotNull($this->name),
```


<a name="merging-conditional-attributes"></a>
#### 合併條件式屬性

有時您可能有幾個屬性應根據相同的條件包含在資源回應中。在此情況下，您可以使用 `mergeWhen` 方法，僅在給定條件為 `true` 時才將這些屬性包含在回應中：

```php
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
```

同樣地，如果給定條件為 `false`，這些屬性將在傳送給客戶端之前從資源回應中移除。

> [!WARNING]
> `mergeWhen` 方法不應在混合字串和數字鍵的陣列中使用。此外，它不應在具有非依序排列的數字鍵的陣列中使用。

<a name="conditional-relationships"></a>
### 條件式關聯

除了條件式載入屬性外，您還可以根據關聯是否已在模型上載入，來條件式地將關聯包含在資源回應中。這讓您的控制器可以決定要在模型上載入哪些關聯，而您的資源可以輕鬆地僅在這些關聯實際載入後才包含它們。最終，這讓您可以更容易地避免資源中的「N+1」查詢問題。

`whenLoaded` 方法可用於條件式載入關聯。為了避免不必要的關聯載入，此方法接受關聯的名稱，而非關聯本身：

```php
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
```

在此範例中，如果關聯尚未載入，`posts` 鍵將在資源回應傳送給客戶端之前從回應中移除。


<a name="conditional-relationship-counts"></a>
#### 條件式關聯計數

除了條件式地包含關聯外，您還可以根據關聯的計數是否已在模型上載入，來條件式地將關聯「計數」包含在資源回應中：

```php
new UserResource($user->loadCount('posts'));
```

`whenCounted` 方法可用於條件式地將關聯計數包含在資源回應中。此方法避免了在關聯計數不存在時不必要地包含該屬性：

```php
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
```

在此範例中，如果 `posts` 關聯的計數尚未載入，`posts_count` 鍵將在資源回應傳送給客戶端之前從回應中移除。

其他類型的聚合函數，例如 `avg`、`sum`、`min` 和 `max`，也可以使用 `whenAggregated` 方法進行條件式載入：

```php
'words_avg' => $this->whenAggregated('posts', 'words', 'avg'),
'words_sum' => $this->whenAggregated('posts', 'words', 'sum'),
'words_min' => $this->whenAggregated('posts', 'words', 'min'),
'words_max' => $this->whenAggregated('posts', 'words', 'max'),
```


<a name="conditional-pivot-information"></a>
#### 條件式樞紐資料

除了條件式地包含關聯資訊在您的資源回應中，您還可以利用 `whenPivotLoaded` 方法，條件式地包含多對多關聯中間表中的資料。`whenPivotLoaded` 方法接受樞紐表名稱作為其第一個引數。第二個引數應該是一個閉包，如果樞紐資料在模型上可用，該閉包將傳回要傳回的值：

```php
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
```

如果您的關聯使用 [自訂中間表模型](/docs/{{version}}/eloquent-relationships#defining-custom-intermediate-table-models)，您可以將中間表模型實例作為 `whenPivotLoaded` 方法的第一個引數傳入：

```php
'expires_at' => $this->whenPivotLoaded(new Membership, function () {
    return $this->pivot->expires_at;
}),
```

如果您的中間表使用 `pivot` 以外的存取器，您可以使用 `whenPivotLoadedAs` 方法：

```php
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
```


<a name="adding-meta-data"></a>
### 新增 Meta 資料

有些 JSON API 標準要求在您的資源和資源集合回應中新增 Meta 資料。這通常包括指向資源或相關資源的 `links` 等內容，或關於資源本身的 Meta 資料。如果您需要傳回關於資源的額外 Meta 資料，請將其包含在您的 `toArray` 方法中。例如，您可以在轉換資源集合時包含 `links` 資訊：

```php
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
```

當您從資源傳回額外的 Meta 資料時，您無需擔心會意外覆寫 Laravel 在傳回分頁回應時自動新增的 `links` 或 `meta` 鍵。您定義的任何額外 `links` 都將與分頁器提供的連結合併。


<a name="top-level-meta-data"></a>
#### 頂層 Meta 資料

有時您可能希望僅當資源是正在傳回的最外層資源時，才在資源回應中包含某些 Meta 資料。通常，這包括關於整個回應的 Meta 資訊。要定義此 Meta 資料，請在您的資源類別中新增一個 `with` 方法。此方法應傳回一個 Meta 資料陣列，該陣列僅在資源是最外層資源被轉換時才包含在資源回應中：

```php
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
```


<a name="adding-meta-data-when-constructing-resources"></a>
#### 建構資源時新增 Meta 資料

您也可以在您的路由或控制器中建構資源實例時，新增頂層資料。`additional` 方法適用於所有資源，它接受一個應新增至資源回應的資料陣列：

```php
return User::all()
    ->load('roles')
    ->toResourceCollection()
    ->additional(['meta' => [
        'key' => 'value',
    ]]);
```

<a name="resource-responses"></a>
## 資源回應

如您先前所讀，資源可以直接從路由與控制器中回傳：

```php
use App\Models\User;

Route::get('/user/{id}', function (string $id) {
    return User::findOrFail($id)->toResource();
});
```

然而，有時您可能需要在將傳出的 HTTP 回應傳送給客戶端之前對其進行自訂。有兩種方式可以達成此目的。首先，您可以將 `response` 方法鏈結到資源上。這個方法會回傳一個 `Illuminate\Http\JsonResponse` 實例，讓您能完全控制回應的標頭：

```php
use App\Http\Resources\UserResource;
use App\Models\User;

Route::get('/user', function () {
    return User::find(1)
        ->toResource()
        ->response()
        ->header('X-Value', 'True');
});
```

另一種方式是，您可以在資源本身中定義一個 `withResponse` 方法。當資源作為回應中最外層的資源回傳時，就會呼叫此方法：

```php
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
```