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

在建構 API 時，您可能需要一個轉換層，位於您的 Eloquent 模型與實際回傳給應用程式使用者的 JSON 回應之間。例如，您可能希望針對部分使用者顯示某些屬性，而對其他使用者則不顯示；或者您可能希望在模型的 JSON 表示中始終包含某些關聯。Eloquent 的資源類別讓您可以表達性地、輕鬆地將您的模型和模型集合轉換為 JSON。

當然，您始終可以使用 Eloquent 模型或集合的 `toJson` 方法將其轉換為 JSON；然而，Eloquent 資源提供了對模型及其關聯的 JSON 序列化更細緻且強大的控制。

<a name="generating-resources"></a>
## 生成資源

要生成資源類別，您可以使用 `make:resource` Artisan 命令。預設情況下，資源將放置在您應用程式的 `app/Http/Resources` 目錄中。資源會繼承 `Illuminate\Http\Resources\Json\JsonResource` 類別：

```shell
php artisan make:resource UserResource
```

<a name="generating-resource-collections"></a>
#### 資源集合

除了生成轉換單一模型的資源外，您還可以生成負責轉換模型集合的資源。這允許您的 JSON 回應包含與給定資源的整個集合相關的連結和其他中繼資訊。

要建立資源集合，您應該在建立資源時使用 `--collection` 旗標。或者，在資源名稱中包含 `Collection` 一詞將會指示 Laravel 建立一個集合資源。集合資源會繼承 `Illuminate\Http\Resources\Json\ResourceCollection` 類別：

```shell
php artisan make:resource User --collection

php artisan make:resource UserCollection
```

<a name="concept-overview"></a>
## 概念概覽

> [!NOTE]
> 這是資源和資源集合的高層次概覽。強烈建議您閱讀本文件中的其他章節，以更深入地了解資源為您提供的自訂和強大功能。

在深入探討撰寫資源時可用的所有選項之前，讓我們先高層次地了解資源在 Laravel 中的使用方式。資源類別代表需要轉換為 JSON 結構的單一模型。例如，這是一個簡單的 `UserResource` 資源類別：

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

每個資源類別都定義了一個 `toArray` 方法，該方法回傳當資源從路由或控制器方法回傳作為回應時，應轉換為 JSON 的屬性陣列。

請注意，我們可以直接從 `$this` 變數存取模型屬性。這是因為資源類別會自動將屬性和方法的存取代理到底層模型，以便於存取。一旦定義了資源，就可以從路由或控制器回傳。資源透過其建構函式接受底層模型實例：

```php
use App\Http\Resources\UserResource;
use App\Models\User;

Route::get('/user/{id}', function (string $id) {
    return new UserResource(User::findOrFail($id));
});
```

為了方便起見，您可以使用模型的 `toResource` 方法，該方法將使用框架慣例自動發現模型的底層資源：

```php
return User::findOrFail($id)->toResource();
```

當呼叫 `toResource` 方法時，Laravel 將嘗試在最接近模型命名空間的 `Http\Resources` 命名空間中，尋找與模型名稱匹配且可選地以 `Resource` 為後綴的資源。

<a name="resource-collections"></a>
### 資源集合

如果您要回傳資源集合或分頁回應，您應該在路由或控制器中建立資源實例時，使用資源類別提供的 `collection` 方法：

```php
use App\Http\Resources\UserResource;
use App\Models\User;

Route::get('/users', function () {
    return UserResource::collection(User::all());
});
```

或者，為了方便起見，您可以使用 Eloquent 集合的 `toResourceCollection` 方法，該方法將使用框架慣例自動發現模型的底層資源集合：

```php
return User::all()->toResourceCollection();
```

當呼叫 `toResourceCollection` 方法時，Laravel 將嘗試在最接近模型命名空間的 `Http\Resources` 命名空間中，尋找與模型名稱匹配且以 `Collection` 為後綴的資源集合。

<a name="custom-resource-collections"></a>
#### 自訂資源集合

預設情況下，資源集合不允許新增任何可能需要與您的集合一起回傳的自訂中繼資料。如果您想自訂資源集合回應，您可以建立一個專用資源來表示該集合：

```shell
php artisan make:resource UserCollection
```

一旦生成了資源集合類別，您就可以輕鬆定義應包含在回應中的任何中繼資料：

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

定義資源集合後，可以從路由或控制器回傳：

```php
use App\Http\Resources\UserCollection;
use App\Models\User;

Route::get('/users', function () {
    return new UserCollection(User::all());
});
```

或者，為了方便起見，您可以使用 Eloquent 集合的 `toResourceCollection` 方法，該方法將使用框架慣例自動發現模型的底層資源集合：

```php
return User::all()->toResourceCollection();
```

當呼叫 `toResourceCollection` 方法時，Laravel 將嘗試在最接近模型命名空間的 `Http\Resources` 命名空間中，尋找與模型名稱匹配且以 `Collection` 為後綴的資源集合。

<a name="preserving-collection-keys"></a>
#### 保留集合鍵

當從路由回傳資源集合時，Laravel 會重設集合的鍵，使其按數字順序排列。但是，您可以將 `preserveKeys` 屬性新增到您的資源類別中，指示是否應保留集合的原始鍵：

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

當 `preserveKeys` 屬性設定為 `true` 時，當集合從路由或控制器回傳時，集合鍵將被保留：

```php
use App\Http\Resources\UserResource;
use App\Models\User;

Route::get('/users', function () {
    return UserResource::collection(User::all()->keyBy->id);
});
```

<a name="customizing-the-underlying-resource-class"></a>
#### 自訂底層資源類別

通常，資源集合的 `$this->collection` 屬性會自動填充，其結果是將集合的每個項目映射到其單數資源類別。單數資源類別被假定為集合的類別名稱，不帶類別名稱末尾的 `Collection` 部分。此外，根據您的個人偏好，單數資源類別可能帶有或不帶有 `Resource` 後綴。

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
> 如果您尚未閱讀[概念概覽](#concept-overview)，強烈建議您在繼續閱讀本文件之前先閱讀。

資源只需要將給定模型轉換為陣列。因此，每個資源都包含一個 `toArray` 方法，該方法將模型的屬性轉換為 API 友好的陣列，可以從應用程式的路由或控制器回傳：

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

一旦定義了資源，就可以直接從路由或控制器回傳：

```php
use App\Models\User;

Route::get('/user/{id}', function (string $id) {
    return User::findOrFail($id)->toUserResource();
});
```

<a name="relationships"></a>
#### 關聯

如果您想在回應中包含相關資源，您可以將它們新增到資源的 `toArray` 方法回傳的陣列中。在此範例中，我們將使用 `PostResource` 資源的 `collection` 方法將使用者的部落格文章新增到資源回應中：

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
> 如果您只想在關聯已載入時才包含它們，請查看有關[條件式關聯](#conditional-relationships)的文件。

<a name="writing-resource-collections"></a>
#### 資源集合

雖然資源將單一模型轉換為陣列，但資源集合將模型集合轉換為陣列。然而，對於每個模型來說，定義一個資源集合類別並非絕對必要，因為所有 Eloquent 模型集合都提供了一個 `toResourceCollection` 方法，可以即時生成一個「臨時」資源集合：

```php
use App\Models\User;

Route::get('/users', function () {
    return User::all()->toResourceCollection();
});
```

但是，如果您需要自訂與集合一起回傳的中繼資料，則必須定義自己的資源集合：

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

與單數資源一樣，資源集合可以直接從路由或控制器回傳：

```php
use App\Http\Resources\UserCollection;
use App\Models\User;

Route::get('/users', function () {
    return new UserCollection(User::all());
});
```

或者，為了方便起見，您可以使用 Eloquent 集合的 `toResourceCollection` 方法，該方法將使用框架慣例自動發現模型的底層資源集合：

```php
return User::all()->toResourceCollection();
```

當呼叫 `toResourceCollection` 方法時，Laravel 將嘗試在最接近模型命名空間的 `Http\Resources` 命名空間中，尋找與模型名稱匹配且以 `Collection` 為後綴的資源集合。

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

如果您想禁用最外層資源的包裝，您應該在基礎 `Illuminate\Http\Resources\Json\JsonResource` 類別上呼叫 `withoutWrapping` 方法。通常，您應該從您的 `AppServiceProvider` 或另一個在應用程式的每個請求上載入的[服務提供者](/docs/{{version}}/providers)中呼叫此方法：

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
> `withoutWrapping` 方法僅影響最外層的回應，不會移除您手動新增到自己資源集合中的 `data` 鍵。

<a name="wrapping-nested-resources"></a>
#### 包裝巢狀資源

您可以完全自由地決定如何包裝資源的關聯。如果您希望所有資源集合都包裝在 `data` 鍵中，無論其巢狀深度如何，您都應該為每個資源定義一個資源集合類別，並將集合回傳在 `data` 鍵中。

您可能會想，這是否會導致您的最外層資源被包裝在兩個 `data` 鍵中。別擔心，Laravel 絕不會讓您的資源意外地被雙重包裝，因此您無需擔心您正在轉換的資源集合的巢狀層級：

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
#### 資料包裝與分頁

當透過資源回應回傳分頁集合時，即使已呼叫 `withoutWrapping` 方法，Laravel 也會將您的資源資料包裝在 `data` 鍵中。這是因為分頁回應始終包含 `meta` 和 `links` 鍵，其中包含有關分頁器狀態的資訊：

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

或者，為了方便起見，您可以使用分頁器的 `toResourceCollection` 方法，該方法將使用框架慣例自動發現分頁模型的底層資源集合：

```php
return User::paginate()->toResourceCollection();
```

分頁回應始終包含 `meta` 和 `links` 鍵，其中包含有關分頁器狀態的資訊：

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

如果您想自訂分頁回應的 `links` 或 `meta` 鍵中包含的資訊，您可以在資源上定義一個 `paginationInformation` 方法。此方法將接收 `$paginated` 資料和 `$default` 資訊陣列，後者是一個包含 `links` 和 `meta` 鍵的陣列：

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

有時您可能希望僅在滿足給定條件時才將屬性包含在資源回應中。例如，您可能希望僅在當前使用者是「管理員」時才包含某個值。Laravel 提供了各種輔助方法來幫助您處理這種情況。`when` 方法可用於有條件地將屬性新增到資源回應中：

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

在此範例中，`secret` 鍵僅在經過身份驗證的使用者的 `isAdmin` 方法回傳 `true` 時才會在最終資源回應中回傳。如果該方法回傳 `false`，則 `secret` 鍵將在發送給客戶端之前從資源回應中移除。`when` 方法允許您在建構陣列時，無需訴諸條件語句即可表達性地定義資源。

`when` 方法也接受一個閉包作為其第二個參數，允許您僅在給定條件為 `true` 時才計算結果值：

```php
'secret' => $this->when($request->user()->isAdmin(), function () {
    return 'secret-value';
}),
```

`whenHas` 方法可用於在底層模型上實際存在屬性時包含該屬性：

```php
'name' => $this->whenHas('name'),
```

此外，`whenNotNull` 方法可用於在屬性不為 null 時將其包含在資源回應中：

```php
'name' => $this->whenNotNull($this->name),
```

<a name="merging-conditional-attributes"></a>
#### 合併條件式屬性

有時您可能有幾個屬性應該僅在相同條件下包含在資源回應中。在這種情況下，您可以使用 `mergeWhen` 方法僅在給定條件為 `true` 時才將屬性包含在回應中：

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

同樣，如果給定條件為 `false`，這些屬性將在發送給客戶端之前從資源回應中移除。

> [!WARNING]
> `mergeWhen` 方法不應在混合字串和數字鍵的陣列中使用。此外，它不應在數字鍵未按順序排列的陣列中使用。

<a name="conditional-relationships"></a>
### 條件式關聯

除了有條件地載入屬性之外，您還可以根據關聯是否已在模型上載入，有條件地將關聯包含在資源回應中。這允許您的控制器決定應在模型上載入哪些關聯，並且您的資源可以輕鬆地僅在實際載入時才包含它們。最終，這使得在資源中避免「N+1」查詢問題變得更容易。

`whenLoaded` 方法可用於有條件地載入關聯。為了避免不必要地載入關聯，此方法接受關聯的名稱而不是關聯本身：

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

在此範例中，如果關聯尚未載入，則 `posts` 鍵將在發送給客戶端之前從資源回應中移除。

<a name="conditional-relationship-counts"></a>
#### 條件式關聯計數

除了有條件地包含關聯之外，您還可以根據關聯的計數是否已在模型上載入，有條件地將關聯「計數」包含在資源回應中：

```php
new UserResource($user->loadCount('posts'));
```

`whenCounted` 方法可用於有條件地將關聯的計數包含在您的資源回應中。此方法避免了在關聯的計數不存在時不必要地包含該屬性：

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

在此範例中，如果 `posts` 關聯的計數尚未載入，則 `posts_count` 鍵將在發送給客戶端之前從資源回應中移除。

其他類型的聚合，例如 `avg`、`sum`、`min` 和 `max` 也可以使用 `whenAggregated` 方法有條件地載入：

```php
'words_avg' => $this->whenAggregated('posts', 'words', 'avg'),
'words_sum' => $this->whenAggregated('posts', 'words', 'sum'),
'words_min' => $this->whenAggregated('posts', 'words', 'min'),
'words_max' => $this->whenAggregated('posts', 'words', 'max'),
```

<a name="conditional-pivot-information"></a>
#### 條件式樞紐資訊

除了有條件地在資源回應中包含關聯資訊之外，您還可以使用 `whenPivotLoaded` 方法有條件地包含多對多關聯的中間表中的資料。`whenPivotLoaded` 方法接受樞紐表的名稱作為其第一個參數。第二個參數應該是一個閉包，如果樞紐資訊在模型上可用，則回傳要回傳的值：

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

如果您的關聯使用[自訂中間表模型](/docs/{{version}}/eloquent-relationships#defining-custom-intermediate-table-models)，您可以將中間表模型的實例作為第一個參數傳遞給 `whenPivotLoaded` 方法：

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
### 新增中繼資料

某些 JSON API 標準要求在資源和資源集合回應中新增中繼資料。這通常包括諸如資源或相關資源的 `links`，或資源本身的中繼資料。如果您需要回傳有關資源的其他中繼資料，請將其包含在您的 `toArray` 方法中。例如，您可以在轉換資源集合時包含 `links` 資訊：

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

當從資源回傳額外中繼資料時，您無需擔心意外覆寫 Laravel 在回傳分頁回應時自動新增的 `links` 或 `meta` 鍵。您定義的任何額外 `links` 都將與分頁器提供的連結合併。

<a name="top-level-meta-data"></a>
#### 頂層中繼資料

有時您可能希望僅在資源是回傳的最外層資源時才將某些中繼資料包含在資源回應中。通常，這包括有關整個回應的中繼資訊。要定義此中繼資料，請在您的資源類別中新增一個 `with` 方法。此方法應回傳一個中繼資料陣列，該陣列僅在資源是正在轉換的最外層資源時才包含在資源回應中：

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
#### 建構資源時新增中繼資料

您也可以在路由或控制器中建構資源實例時新增頂層資料。`additional` 方法在所有資源上都可用，它接受一個應新增到資源回應中的資料陣列：

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

如您所讀，資源可以直接從路由和控制器回傳：

```php
use App\Models\User;

Route::get('/user/{id}', function (string $id) {
    return User::findOrFail($id)->toResource();
});
```

然而，有時您可能需要在將 HTTP 回應發送給客戶端之前自訂它。有兩種方法可以實現這一點。首先，您可以將 `response` 方法鏈接到資源上。此方法將回傳一個 `Illuminate\Http\JsonResponse` 實例，讓您可以完全控制回應的標頭：

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

或者，您可以在資源本身中定義一個 `withResponse` 方法。當資源作為回應中最外層的資源回傳時，將呼叫此方法：

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

