# Eloquent：API 資源

- [簡介](#introduction)
- [建立資源](#generating-resources)
- [概念總覽](#concept-overview)
    - [資源集合](#resource-collections)
- [撰寫資源](#writing-resources)
    - [資料封裝](#data-wrapping)
    - [分頁](#pagination)
    - [條件式屬性](#conditional-attributes)
    - [條件式關聯](#conditional-relationships)
    - [新增中繼資料](#adding-meta-data)
- [資源回應](#resource-responses)

<a name="introduction"></a>
## 簡介

建立 API 時，您可能需要一個轉換層，它位於 Eloquent 模型和實際傳回給應用程式使用者的 JSON 回應之間。例如，您可能希望僅向部分使用者顯示特定屬性，而不向其他使用者顯示；或者您可能希望在模型的 JSON 表示中始終包含特定關聯。Eloquent 的資源類別讓您可以清楚簡潔地將模型和模型集合轉換為 JSON。

當然，您隨時可以使用 Eloquent 模型或集合的 `toJson` 方法將其轉換為 JSON；然而，Eloquent 資源為模型及其關聯的 JSON 序列化提供了更精細且強大的控制。

<a name="generating-resources"></a>
## 建立資源

要建立資源類別，您可以使用 `make:resource` Artisan 命令。預設情況下，資源將會放置在應用程式的 `app/Http/Resources` 目錄中。資源會繼承 `Illuminate\Http\Resources\Json\JsonResource` 類別：

```shell
php artisan make:resource UserResource
```

<a name="generating-resource-collections"></a>
#### 建立資源集合

除了產生轉換個別模型的資源之外，您還可以產生負責轉換模型集合的資源。這讓您的 JSON 回應可以包含連結和其他與指定資源的整個集合相關的中繼資訊。

要建立資源集合，您在建立資源時應該使用 `--collection` 旗標。或者，在資源名稱中包含 `Collection` 字樣會告訴 Laravel 應建立一個集合資源。集合資源會繼承 `Illuminate\Http\Resources\Json\ResourceCollection` 類別：

```shell
php artisan make:resource User --collection

php artisan make:resource UserCollection
```

<a name="concept-overview"></a>
## 概念總覽

> [!NOTE]
> 這是資源與資源集合的高層次概述。強烈建議您閱讀本文件的其他章節，以更深入地理解資源所提供的客製化與強大功能。

在深入探討撰寫資源時可用的所有選項之前，讓我們先從高層次了解資源如何在 Laravel 中使用。資源類別代表一個單一模型，它需要被轉換為 JSON 結構。例如，以下是一個簡單的 `UserResource` 資源類別：

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

每個資源類別都定義了一個 `toArray` 方法，該方法會回傳一個屬性陣列，這些屬性應該被轉換為 JSON，當資源作為路由或控制器方法的回應回傳時。

請注意，我們可以直接從 `$this` 變數存取模型屬性。這是因為資源類別會自動將屬性與方法的存取代理到底層模型，以方便存取。一旦資源被定義，它就可以從路由或控制器中回傳。該資源透過其建構子接受底層模型實例：

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

當呼叫 `toResource` 方法時，Laravel 將嘗試尋找一個與模型名稱相符的資源，該資源可選擇性地以 `Resource` 作為字尾，且位於 `Http\Resources` 命名空間內最接近模型的命名空間。

如果您的資源類別不遵循此命名慣例或者位於不同的命名空間中，您可以為模型指定預設資源，使用 `UseResource` 屬性：

```php
<?php

namespace App\Models;

use App\Http\Resources\CustomUserResource;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Attributes\UseResource;

#[UseResource(CustomUserResource::class)]
class User extends Model
{
    // ...
}
```

或者，您可以透過傳遞資源類別給 `toResource` 方法來指定它：

```php
return User::findOrFail($id)->toResource(CustomUserResource::class);
```

<a name="resource-collections"></a>
### 資源集合

如果您正在回傳資源集合或分頁回應，在路由或控制器方法中建立資源實例時，應該使用資源類別提供的 `collection` 方法：

```php
use App\Http\Resources\UserResource;
use App\Models\User;

Route::get('/users', function () {
    return UserResource::collection(User::all());
});
```

或者，為了方便起見，您可以使用 Eloquent 集合的 `toResourceCollection` 方法，它將透過框架慣例自動發現模型底層的資源集合：

```php
return User::all()->toResourceCollection();
```

當呼叫 `toResourceCollection` 方法時，Laravel 將嘗試在最接近模型命名空間的 `Http\Resources` 命名空間中，找到一個與模型名稱相符並以 `Collection` 為後綴的資源集合。

如果您的資源集合類別不遵循此命名慣例或位於不同的命名空間中，您可以使用 `UseResourceCollection` 屬性為模型指定預設的資源集合：

```php
<?php

namespace App\Models;

use App\Http\Resources\CustomUserCollection;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Attributes\UseResourceCollection;

#[UseResourceCollection(CustomUserCollection::class)]
class User extends Model
{
    // ...
}
```

或者，您可以透過將資源集合類別傳遞給 `toResourceCollection` 方法來指定它：

```php
return User::all()->toResourceCollection(CustomUserCollection::class);
```


<a name="custom-resource-collections"></a>
#### 自訂資源集合

依預設，資源集合不允許新增任何可能需要與您的集合一起回傳的自訂中繼資料。如果您想自訂資源集合回應，您可以建立一個專用的資源來表示該集合：

```shell
php artisan make:resource UserCollection
```

資源集合類別產生後，您可以輕鬆定義任何應包含在回應中的中繼資料：

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

定義資源集合後，它可以從路由或控制器中回傳：

```php
use App\Http\Resources\UserCollection;
use App\Models\User;

Route::get('/users', function () {
    return new UserCollection(User::all());
});
```

或者，為了方便起見，您可以使用 Eloquent 集合的 `toResourceCollection` 方法，它將透過框架慣例自動發現模型底層的資源集合：

```php
return User::all()->toResourceCollection();
```

當呼叫 `toResourceCollection` 方法時，Laravel 將嘗試在最接近模型命名空間的 `Http\Resources` 命名空間中，找到一個與模型名稱相符並以 `Collection` 為後綴的資源集合。


<a name="preserving-collection-keys"></a>
#### 保留集合的鍵值

從路由回傳資源集合時，Laravel 會重置集合的鍵值，使其按數值順序排列。不過，您可以在資源類別中新增一個 `preserveKeys` 屬性，指示是否應保留集合的原始鍵值：

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

當 `preserveKeys` 屬性設定為 `true` 時，當集合從路由或控制器回傳時，集合的鍵值將會被保留：

```php
use App\Http\Resources\UserResource;
use App\Models\User;

Route::get('/users', function () {
    return UserResource::collection(User::all()->keyBy->id);
});
```


<a name="customizing-the-underlying-resource-class"></a>
#### 自訂底層資源類別

通常，資源集合的 `$this->collection` 屬性會自動填入集合中每個項目映射到其單一資源類別的結果。單一資源類別被假定為集合類別名稱中不帶末尾 `Collection` 部分的類別名稱。此外，根據您的個人偏好，單一資源類別可能帶有或不帶有 `Resource` 後綴。

例如，`UserCollection` 將嘗試將給定的使用者實例映射到 `UserResource` 資源中。要自訂此行為，您可以覆寫資源集合的 `$collects` 屬性：

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
> 如果您尚未閱讀[概念總覽](#concept-overview)，強烈建議您在繼續閱讀本文檔之前先閱讀該章節。

資源只需要將給定的模型轉換為陣列。因此，每個資源都包含一個 `toArray` 方法，該方法會將您模型的屬性轉換為對 API 友善的陣列，以便從應用程式的路由或控制器中返回：

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

一旦定義了資源，就可以直接從路由或控制器中返回：

```php
use App\Models\User;

Route::get('/user/{id}', function (string $id) {
    return User::findOrFail($id)->toUserResource();
});
```

<a name="relationships"></a>
#### 關聯

如果您想在回應中包含相關資源，可以將它們添加到資源的 `toArray` 方法返回的陣列中。在此範例中，我們將使用 `PostResource` 資源的 `collection` 方法，將使用者的部落格文章添加到資源回應中：

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
> 如果您只想在關聯已經載入時才包含它們，請查閱[條件式關聯](#conditional-relationships)的文件。

<a name="writing-resource-collections"></a>
#### 資源集合

雖然資源將單一模型轉換為陣列，但資源集合會將模型的集合轉換為陣列。然而，由於所有 Eloquent 模型集合都提供了一個 `toResourceCollection` 方法來動態生成「ad-hoc」資源集合，因此並非絕對需要為每個模型定義一個資源集合類別：

```php
use App\Models\User;

Route::get('/users', function () {
    return User::all()->toResourceCollection();
});
```

但是，如果您需要自訂隨集合返回的中繼資料，則有必要定義自己的資源集合：

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

或者，為了方便起見，您可以使用 Eloquent 集合的 `toResourceCollection` 方法，該方法將利用框架慣例自動發現模型底層的資源集合：

```php
return User::all()->toResourceCollection();
```

調用 `toResourceCollection` 方法時，Laravel 將嘗試在最接近模型命名空間的 `Http\Resources` 命名空間中，尋找與模型名稱匹配且後綴為 `Collection` 的資源集合。

<a name="data-wrapping"></a>
### 資料封裝

預設情況下，當資源回應轉換為 JSON 時，最外層的資源會被封裝在 `data` 鍵中。因此，舉例來說，典型的資源集合回應如下所示：

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

如果您想禁用最外層資源的封裝，應該在基礎 `Illuminate\Http\Resources\Json\JsonResource` 類別上調用 `withoutWrapping` 方法。通常，您應該在 `AppServiceProvider` 或另一個每次應用程式請求時載入的[服務提供者](/docs/{{version}}/providers)中調用此方法：

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
> `withoutWrapping` 方法只會影響最外層的回應，並且不會移除您手動添加到自己的資源集合中的 `data` 鍵。

<a name="wrapping-nested-resources"></a>
#### 封裝巢狀資源

您可以完全自由地決定資源關聯的封裝方式。如果您希望所有資源集合都封裝在 `data` 鍵中，無論其巢狀層級如何，您都應該為每個資源定義一個資源集合類別，並在 `data` 鍵中返回該集合。

您可能會想，這是否會導致最外層的資源被兩個 `data` 鍵封裝。別擔心，Laravel 絕不會讓您的資源意外地重複封裝，因此您不必擔心您正在轉換的資源集合的巢狀層級：

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

當透過資源回應返回分頁集合時，即使已調用 `withoutWrapping` 方法，Laravel 仍會將您的資源資料封裝在 `data` 鍵中。這是因為分頁回應總是包含帶有分頁器狀態資訊的 `meta` 和 `links` 鍵：

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

您可以將 Laravel 分頁器實例傳遞給資源的 `collection` 方法或客製化資源集合：

```php
use App\Http\Resources\UserCollection;
use App\Models\User;

Route::get('/users', function () {
    return new UserCollection(User::paginate());
});
```

或者，為了方便起見，您可以使用分頁器的 `toResourceCollection` 方法，它將透過框架慣例自動發現已分頁模型底層的資源集合：

```php
return User::paginate()->toResourceCollection();
```

分頁回應總是包含 `meta` 和 `links` 鍵，其中包含有關分頁器狀態的資訊：

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
#### 客製化分頁資訊

如果您想要客製化分頁回應中 `links` 或 `meta` 鍵所包含的資訊，您可以在資源上定義 `paginationInformation` 方法。此方法將接收 `$paginated` 資料以及 `$default` 資訊陣列，該陣列包含 `links` 和 `meta` 鍵：

```php
/**
 * Customize the pagination information for the resource.
 *
 * @param  \Illuminate\Http\Request  $request
 * @param  array  $paginated
 * @param  array  $default
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

有時您可能希望僅在滿足給定條件時才將屬性包含在資源回應中。例如，您可能希望僅在當前使用者是「管理員」時才包含某個值。Laravel 提供了多種輔助方法來協助您處理這種情況。`when` 方法可用於條件式地將屬性新增到資源回應中：

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

在此範例中，只有在已驗證使用者的 `isAdmin` 方法回傳 `true` 時，`secret` 鍵才會包含在最終的資源回應中。如果該方法回傳 `false`，`secret` 鍵將在發送給客戶端之前從資源回應中移除。`when` 方法讓您能夠以表達性方式定義資源，而無需在建構陣列時訴諸於條件判斷式。

`when` 方法也接受一個閉包作為其第二個引數，讓您可以在給定條件為 `true` 時才計算結果值：

```php
'secret' => $this->when($request->user()->isAdmin(), function () {
    return 'secret-value';
}),
```

`whenHas` 方法可用於在屬性實際存在於底層模型上時，才將其包含進去：

```php
'name' => $this->whenHas('name'),
```

此外，`whenNotNull` 方法可用於在屬性不為 null 時，將其包含在資源回應中：

```php
'name' => $this->whenNotNull($this->name),
```

<a name="merging-conditional-attributes"></a>
#### 合併條件式屬性

有時您可能會有幾個屬性，僅應在基於相同條件時才包含在資源回應中。在這種情況下，您可以使用 `mergeWhen` 方法，僅在給定條件為 `true` 時才將屬性包含在回應中：

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

[!WARNING]
`mergeWhen` 方法不應該用於混合字串和數字鍵的陣列中。此外，它也不應該用於數字鍵未按順序排列的陣列中。

<a name="conditional-relationships"></a>
### 條件式關聯

除了條件式載入屬性之外，您還可以根據關聯是否已經載入到模型中，來條件式地在資源回應中包含關聯。這讓您的控制器可以決定應該在模型上載入哪些關聯，而您的資源可以輕鬆地僅在關聯實際載入時才包含它們。最終，這使得在資源中更容易避免「N+1」查詢問題。

`whenLoaded` 方法可用於條件式地載入關聯。為了避免不必要的關聯載入，此方法接受關聯的名稱而不是關聯本身：

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

在此範例中，如果關聯尚未載入，`posts` 鍵將在資源回應發送給客戶端之前從中移除。

<a name="conditional-relationship-counts"></a>
#### 條件式關聯計數

除了條件式地包含關聯之外，您還可以根據關聯的計數是否已載入到模型中，來條件式地在資源回應中包含關聯的「計數」。

```php
new UserResource($user->loadCount('posts'));
```

`whenCounted` 方法可用於條件式地在您的資源回應中包含關聯的計數。如果關聯的計數不存在，此方法可以避免不必要地包含該屬性：

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

在此範例中，如果 `posts` 關聯的計數尚未載入，`posts_count` 鍵將在資源回應發送給客戶端之前從中移除。

其他類型的聚合，例如 `avg`、`sum`、`min` 和 `max`，也可以使用 `whenAggregated` 方法進行條件式載入：

```php
'words_avg' => $this->whenAggregated('posts', 'words', 'avg'),
'words_sum' => $this->whenAggregated('posts', 'words', 'sum'),
'words_min' => $this->whenAggregated('posts', 'words', 'min'),
'words_max' => $this->whenAggregated('posts', 'words', 'max'),
```

<a name="conditional-pivot-information"></a>
#### 條件式中介資料表資訊

除了條件式地在資源回應中包含關聯資訊之外，您還可以使用 `whenPivotLoaded` 方法，條件式地包含多對多關聯中介資料表中的資料。`whenPivotLoaded` 方法將中介資料表的名稱作為其第一個參數。第二個參數應該是一個閉包，如果中介資料表資訊在模型上可用，則返回該閉包的值：

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

如果您的關聯使用 [自訂中介資料表模型](/docs/{{version}}/eloquent-relationships#defining-custom-intermediate-table-models)，您可以將中介資料表模型的一個實例作為第一個參數傳遞給 `whenPivotLoaded` 方法：

```php
'expires_at' => $this->whenPivotLoaded(new Membership, function () {
    return $this->pivot->expires_at;
}),
```

如果您的中介資料表使用 `pivot` 以外的存取器，您可以使用 `whenPivotLoadedAs` 方法：

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

某些 JSON API 標準要求在您的資源和資源集合回應中新增中繼資料 (meta data)。這通常包括諸如指向資源或相關資源的 `links`，或關於資源本身的中繼資料。如果您需要返回關於資源的額外中繼資料，請將其包含在您的 `toArray` 方法中。例如，您在轉換資源集合時可能會包含 `links` 資訊：

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

當您的資源返回額外中繼資料時，您無需擔心意外覆寫 Laravel 在返回分頁回應時自動添加的 `links` 或 `meta` 鍵。您定義的任何額外 `links` 都將與分頁器提供的連結合併。

<a name="top-level-meta-data"></a>
#### 頂層中繼資料

有時您可能希望僅在資源是回傳的最外層資源時，才在資源回應中包含某些中繼資料。通常，這包括關於整個回應的中繼資訊。要定義此中繼資料，請在您的資源類別中新增一個 `with` 方法。此方法應返回一個中繼資料陣列，僅當資源是最外層正在轉換的資源時，才會包含在資源回應中：

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

在路由或控制器中建構資源實例時，您也可以新增頂層資料。所有資源都可用的 `additional` 方法接受一個資料陣列，該陣列應新增到資源回應中：

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

如您所讀，資源可以直接從路由與控制器中回傳：

```php
use App\Models\User;

Route::get('/user/{id}', function (string $id) {
    return User::findOrFail($id)->toResource();
});
```

然而，有時候您可能需要在對外的 HTTP 回應送回給客戶端之前對其進行自訂。有兩種方法可以實現這一點。首先，您可以將 `response` 方法鏈接到資源上。此方法將回傳一個 `Illuminate\Http\JsonResponse` 實例，讓您能夠完全控制回應的標頭：

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

此外，您也可以在資源本身中定義一個 `withResponse` 方法。當資源作為回應中最外層的資源回傳時，將會呼叫此方法：

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