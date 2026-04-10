# Eloquent: API 資源

- [簡介](#introduction)
- [產生資源](#generating-resources)
- [概念概覽](#concept-overview)
    - [資源集合](#resource-collections)
- [撰寫資源](#writing-resources)
    - [資料包裹](#data-wrapping)
    - [分頁](#pagination)
    - [條件屬性](#conditional-attributes)
    - [條件關聯](#conditional-relationships)
    - [新增中繼資料](#adding-meta-data)
- [JSON:API 資源](#jsonapi-resources)
    - [產生 JSON:API 資源](#generating-jsonapi-resources)
    - [定義屬性](#defining-jsonapi-attributes)
    - [定義關聯](#defining-jsonapi-relationships)
    - [資源類型與 ID](#jsonapi-resource-type-and-id)
    - [稀疏欄位集與包含](#jsonapi-sparse-fieldsets-and-includes)
    - [連結與中繼資料](#jsonapi-links-and-meta)
- [資源回應](#resource-responses)

<a name="introduction"></a>
## 簡介

當您在建構 API 時，您可能需要一個位於 Eloquent 模型與實際回傳給應用程式使用者的 JSON 回應之間的轉換層。例如，您可能希望針對部分使用者顯示某些屬性而對其他使用者隱藏，或者您可能希望在模型的 JSON 表示中始終包含某些關聯。Eloquent 的資源類別讓您可以直觀且輕鬆地將模型與模型集合轉換為 JSON。

當然，您隨時可以使用 `toJson` 方法將 Eloquent 模型或集合轉換為 JSON；然而，Eloquent 資源針對模型及其關聯的 JSON 序列化提供了更精細且強大的控制。


<a name="generating-resources"></a>
## 產生資源

要產生資源類別，您可以使用 `make:resource` Artisan 命令。預設情況下，資源將被放置在應用程式的 `app/Http/Resources` 目錄中。資源繼承自 `Illuminate\Http\Resources\Json\JsonResource` 類別：

```shell
php artisan make:resource UserResource
```


<a name="generating-resource-collections"></a>
#### 資源集合

除了產生用來轉換單一模型的資源外，您還可以產生負責轉換模型集合的資源。這讓您的 JSON 回應能夠包含與該資源完整集合相關的連結或其他中繼資料。

要建立資源集合，您在建立資源時應該使用 `--collection` 標記。或者，在資源名稱中包含 `Collection` 字樣會告知 Laravel 應該建立一個集合資源。集合資源繼承自 `Illuminate\Http\Resources\Json\ResourceCollection` 類別：

```shell
php artisan make:resource User --collection

php artisan make:resource UserCollection
```

<a name="concept-overview"></a>
## 概念概覽

> [!NOTE]
> 這是關於資源與資源集合的高階概覽。強烈建議您閱讀本文件的其他章節，以深入了解資源為您提供的自定義功能與強大能力。

在深入研究撰寫資源的所有選項之前，我們先來高階地看看資源在 Laravel 中是如何使用的。一個資源類別代表一個需要被轉換為 JSON 結構的單一模型。例如，這裡有一個簡單的 `UserResource` 資源類別：

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

每個資源類別都定義了一個 `toArray` 方法，該方法會回傳一個屬性陣列，當資源作為路由或控制器方法的回應回傳時，這些屬性將被轉換為 JSON。

請注意，我們可以直接從 `$this` 變數存取模型屬性。這是因為資源類別會自動將屬性和方法的存取代理到底層模型，以便於存取。一旦定義了資源，就可以從路由或控制器中回傳。資源透過其建構子接收底層模型實例：

```php
use App\Http\Resources\UserResource;
use App\Models\User;

Route::get('/user/{id}', function (string $id) {
    return new UserResource(User::findOrFail($id));
});
```

為了方便起見，您可以使用模型的 `toResource` 方法，它將使用框架慣例來自動偵測模型的底層資源：

```php
return User::findOrFail($id)->toResource();
```

當調用 `toResource` 方法時，Laravel 會嘗試在與模型命名空間最接近的 `Http\Resources` 命名空間中，尋找一個與模型名稱匹配且（可選擇地）以 `Resource` 為後綴的資源。

如果您的資源類別不符合此命名慣例，或位於不同的命名空間中，您可以使用 `UseResource` 屬性為模型指定預設資源：

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

或者，您也可以透過將資源類別傳遞給 `toResource` 方法來指定它：

```php
return User::findOrFail($id)->toResource(CustomUserResource::class);
```

<a name="resource-collections"></a>
### 資源集合

如果您要回傳資源集合或分頁回應，在路由或控制器中建立資源實例時，應使用資源類別提供的 `collection` 方法：

```php
use App\Http\Resources\UserResource;
use App\Models\User;

Route::get('/users', function () {
    return UserResource::collection(User::all());
});
```

或者，為了方便起見，您可以使用 Eloquent 集合的 `toResourceCollection` 方法，它會使用框架慣例來自動偵測模型對應的底層資源集合：

```php
return User::all()->toResourceCollection();
```

當呼叫 `toResourceCollection` 方法時，Laravel 會嘗試在與模型命名空間最接近的 `Http\Resources` 命名空間中，尋找一個名稱與模型相符且以 `Collection` 為後綴的資源集合。

如果您的資源集合類別不符合此命名慣例，或位於不同的命名空間中，您可以使用 `UseResourceCollection` 屬性為模型指定預設的資源集合：

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

或者，您也可以透過將資源集合類別傳遞給 `toResourceCollection` 方法來指定：

```php
return User::all()->toResourceCollection(CustomUserCollection::class);
```


<a name="custom-resource-collections"></a>
#### 自訂資源集合

預設情況下，資源集合不允許增加任何可能需要隨集合一同回傳的自訂中繼資料。如果您想要自訂資源集合的回應，可以建立一個專用的資源來代表該集合：

```shell
php artisan make:resource UserCollection
```

一旦資源集合類別產生後，您可以輕鬆地定義應該包含在回應中的任何中繼資料：

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

定義好資源集合後，即可從路由或控制器中回傳：

```php
use App\Http\Resources\UserCollection;
use App\Models\User;

Route::get('/users', function () {
    return new UserCollection(User::all());
});
```

或者，為了方便起見，您可以使用 Eloquent 集合的 `toResourceCollection` 方法，它會使用框架慣例來自動偵測模型對應的底層資源集合：

```php
return User::all()->toResourceCollection();
```

當呼叫 `toResourceCollection` 方法時，Laravel 會嘗試在與模型命名空間最接近的 `Http\Resources` 命名空間中，尋找一個名稱與模型相符且以 `Collection` 為後綴的資源集合。


<a name="preserving-collection-keys"></a>
#### 保留集合鍵值

從路由回傳資源集合時，Laravel 會重設集合的鍵值，使其成為數字順序。然而，您可以在資源類別上使用 `PreserveKeys` 屬性，來指定是否應保留集合的原始鍵值：

```php
<?php

namespace App\Http\Resources;

use Illuminate\Http\Resources\Attributes\PreserveKeys;
use Illuminate\Http\Resources\Json\JsonResource;

#[PreserveKeys]
class UserResource extends JsonResource
{
    // ...
}
```

當 `preserveKeys` 屬性設定為 `true` 時，從路由或控制器回傳集合時將會保留集合的鍵值：

```php
use App\Http\Resources\UserResource;
use App\Models\User;

Route::get('/users', function () {
    return UserResource::collection(User::all()->keyBy->id);
});
```


<a name="customizing-the-underlying-resource-class"></a>
#### 自訂底層資源類別

通常，資源集合的 `$this->collection` 屬性會自動填充將集合中的每個項目對應到其單數資源類別的結果。單數資源類別被假設為集合的類別名稱去掉末尾的 `Collection` 部分。此外，根據您的個人偏好，單數資源類別可以加上或不加上 `Resource` 後綴。

例如，`UserCollection` 會嘗試將給定的使用者實例對應到 `UserResource` 資源。若要自訂此行為，您可以在資源集合上使用 `Collects` 屬性：

```php
<?php

namespace App\Http\Resources;

use Illuminate\Http\Resources\Attributes\Collects;
use Illuminate\Http\Resources\Json\ResourceCollection;

#[Collects(Member::class)]
class UserCollection extends ResourceCollection
{
    // ...
}
```

<a name="writing-resources"></a>
## 撰寫資源

> [!NOTE]
> 如果您尚未閱讀 [概念概覽](#concept-overview)，強烈建議在繼續閱讀本文件之前先閱讀該章節。

資源只需要將給定的模型轉換為陣列。因此，每個資源都包含一個 `toArray` 方法，該方法將您的模型屬性轉換為適合 API 的陣列，可用於從您的應用程式路由或控制器中回傳：

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

一旦定義了資源，就可以直接從路由或控制器中回傳：

```php
use App\Models\User;

Route::get('/user/{id}', function (string $id) {
    return User::findOrFail($id)->toUserResource();
});
```


<a name="relationships"></a>
#### 關聯

如果您想在回應中包含關聯資源，可以將其添加到資源的 `toArray` 方法所回傳的陣列中。在此範例中，我們將使用 `PostResource` 資源的 `collection` 方法，將使用者的部落格文章添加到資源回應中：

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
> 如果您只想在關聯已經被載入時才包含它們，請參閱 [條件關聯](#conditional-relationships) 的文件。


<a name="writing-resource-collections"></a>
#### 資源集合

資源將單個模型轉換為陣列，而資源集合則將模型集合轉換為陣列。然而，並不絕對有必要為每個模型定義資源集合類別，因為所有 Eloquent 模型集合都提供了一個 `toResourceCollection` 方法，可以用於即時產生一個「臨時」的資源集合：

```php
use App\Models\User;

Route::get('/users', function () {
    return User::all()->toResourceCollection();
});
```

但是，如果您需要自定義集合回傳的中繼資料，則必須定義您自己的資源集合：

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

與單數資源一樣，資源集合可以直接從路由或控制器中回傳：

```php
use App\Http\Resources\UserCollection;
use App\Models\User;

Route::get('/users', function () {
    return new UserCollection(User::all());
});
```

或者，為了方便起見，您可以使用 Eloquent 集合的 `toResourceCollection` 方法，它將使用框架慣例自動發現模型底層的資源集合：

```php
return User::all()->toResourceCollection();
```

在調用 `toResourceCollection` 方法時，Laravel 將嘗試在與模型命名空間最接近的 `Http\Resources` 命名空間中，尋找與模型名稱匹配且後綴為 `Collection` 的資源集合。


<a name="data-wrapping"></a>
### 資料包裹

預設情況下，當資源回應被轉換為 JSON 時，您的最外層資源會被包裹在一個 `data` 鍵中。例如，典型的資源集合回應如下所示：

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

如果您想禁用最外層資源的包裹，應該在基礎的 `Illuminate\Http\Resources\Json\JsonResource` 類別上調用 `withoutWrapping` 方法。通常，您應該在 `AppServiceProvider` 或另一個在每次請求應用程式時都會載入的 [服務提供者(Service Providers)](/docs/{{version}}/providers) 中調用此方法：

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
> `withoutWrapping` 方法僅影響最外層回應，不會移除您手動添加到自定義資源集合中的 `data` 鍵。


<a name="wrapping-nested-resources"></a>
#### 包裹巢狀資源

您可以完全自由地決定如何包裹資源的關聯。如果您希望所有資源集合都被包裹在 `data` 鍵中（無論其巢狀層級如何），您應該為每個資源定義一個資源集合類別，並在 `data` 鍵中回傳該集合。

您可能會好奇這是否會導致您的最外層資源被包裹在兩個 `data` 鍵中。請放心，Laravel 絕不會讓您的資源被意外地重複包裹，因此您不必擔心所轉換之資源集合的巢狀層級：

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
#### 資料包裹與分頁

當透過資源回應回傳分頁集合時，即使調用了 `withoutWrapping` 方法，Laravel 仍會將您的資源資料包裹在 `data` 鍵中。這是因為分頁回應總是包含 `meta` 和 `links` 鍵，其中包含關於分頁器狀態的資訊：

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

您可以將 Laravel 分頁器實例傳遞給資源的 `collection` 方法或自定義的資源集合：

```php
use App\Http\Resources\UserCollection;
use App\Models\User;

Route::get('/users', function () {
    return new UserCollection(User::paginate());
});
```

或者，為了方便起見，您可以使用分頁器的 `toResourceCollection` 方法，它將使用框架慣例來自動發現分頁模型對應的底層資源集合：

```php
return User::paginate()->toResourceCollection();
```

分頁回應總是包含 `meta` 和 `links` 鍵，其中包含關於分頁器狀態的資訊：

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
#### 自定義分頁資訊

如果您想要自定義分頁回應中 `links` 或 `meta` 鍵所包含的資訊，您可以在資源上定義 `paginationInformation` 方法。此方法將接收 `$paginated` 資料以及 `$default` 資訊陣列，該陣列包含了 `links` 和 `meta` 鍵：

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
### 條件屬性

有時候，您可能希望只有在滿足特定條件時，才在資源回應中包含某個屬性。例如，您可能希望只有在目前使用者是「管理員」時才包含某個值。Laravel 提供了多種輔助方法來協助您處理這種情況。`when` 方法可用於有條件地將屬性添加到資源回應中：

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

在這個範例中，只有當已認證使用者的 `isAdmin` 方法回傳 `true` 時，最終的資源回應才會回傳 `secret` 鍵。如果該方法回傳 `false`，`secret` 鍵將在發送到客戶端之前從資源回應中移除。`when` 方法讓您在建構陣列時，能夠以具表達力的方式定義資源，而無需使用條件陳述式。

`when` 方法還接受一個閉包作為其第二個引數，讓您僅在給定條件為 `true` 時才計算結果值：

```php
'secret' => $this->when($request->user()->isAdmin(), function () {
    return 'secret-value';
}),
```

`whenHas` 方法可用於在屬性實際存在於底層模型時包含該屬性：

```php
'name' => $this->whenHas('name'),
```

此外，`whenNotNull` 方法可用於在屬性不為 null 時將其包含在資源回應中：

```php
'name' => $this->whenNotNull($this->name),
```


<a name="merging-conditional-attributes"></a>
#### 合併條件屬性

有時候，您可能有數個屬性應該基於相同的條件才包含在資源回應中。在這種情況下，您可以使用 `mergeWhen` 方法，僅在給定條件為 `true` 時將這些屬性包含在回應中：

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

同樣地，如果給定條件為 `false`，這些屬性將在發送到客戶端之前從資源回應中移除。

> [!WARNING]
> `mergeWhen` 方法不應在混合使用字串和數字鍵的陣列中使用。此外，它也不應在數字鍵非連續排序的陣列中使用。

<a name="conditional-relationships"></a>
### 條件關聯

除了條件式載入屬性外，您還可以根據關聯是否已在模型中載入，來決定是否在資源回應中包含關聯。這讓您的控制器可以決定模型應該載入哪些關聯，而資源則可以在關聯確實被載入時才將其包含在內。最終，這讓您更容易避免資源中的 "N+1" 查詢問題。

`whenLoaded` 方法可用於條件式載入關聯。為了避免不必要地載入關聯，此方法接收的是關聯的名稱而非關聯本身：

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

在這個範例中，如果關聯尚未載入，`posts` 鍵會在回應發送到客戶端之前從資源回應中移除。


<a name="conditional-relationship-counts"></a>
#### 條件式關聯計數

除了條件式包含關聯外，您還可以根據關聯計數是否已在模型中載入，來決定是否在資源回應中包含關聯的「計數(counts)」：

```php
new UserResource($user->loadCount('posts'));
```

`whenCounted` 方法可用於在資源回應中條件式包含關聯計數。如果關聯計數不存在，此方法會避免不必要地包含該屬性：

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

在這個範例中，如果 `posts` 關聯的計數尚未載入，`posts_count` 鍵會在回應發送到客戶端之前從資源回應中移除。

其他類型的聚合函數，例如 `avg`、`sum`、`min` 和 `max`，也可以使用 `whenAggregated` 方法來條件式載入：

```php
'words_avg' => $this->whenAggregated('posts', 'words', 'avg'),
'words_sum' => $this->whenAggregated('posts', 'words', 'sum'),
'words_min' => $this->whenAggregated('posts', 'words', 'min'),
'words_max' => $this->whenAggregated('posts', 'words', 'max'),
```


<a name="conditional-pivot-information"></a>
#### 條件式樞紐(Pivot)資訊

除了在資源回應中條件式包含關聯資訊外，您還可以使用 `whenPivotLoaded` 方法，條件式包含多對多關聯中中間表 (intermediate tables) 的資料。`whenPivotLoaded` 方法接收樞紐表的名稱作為第一個引數。第二個引數應該是一個閉包 (closure)，如果在模型中可取得樞紐資訊，則返回要回傳的值：

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

如果您的關聯使用了 [自定義中間表模型](/docs/{{version}}/eloquent-relationships#defining-custom-intermediate-table-models)，您可以將中間表模型的實例作為 `whenPivotLoaded` 方法的第一個引數：

```php
'expires_at' => $this->whenPivotLoaded(new Membership, function () {
    return $this->pivot->expires_at;
}),
```

如果您的中間表使用 `pivot` 以外的存取器 (accessor)，您可以使用 `whenPivotLoadedAs` 方法：

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

某些 JSON API 標準要求在資源和資源集合的回應中新增中繼資料 (meta data)。這通常包括指向該資源或相關資源的 `links`，或是關於資源本身的中繼資料。如果您需要回傳關於資源的額外中繼資料，請將其包含在 `toArray` 方法中。例如，您可以在轉換資源集合時包含 `links` 資訊：

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

從資源回傳額外中繼資料時，您不必擔心會意外覆蓋 Laravel 在回傳分頁回應時自動新增的 `links` 或 `meta` 鍵。您定義的任何額外 `links` 都會與分頁器提供的連結合併。


<a name="top-level-meta-data"></a>
#### 頂層中繼資料

有時您可能希望僅在資源是回傳的最外層資源時，才在資源回應中包含某些中繼資料。這通常包含關於整個回應的中繼資訊。要定義這些中繼資料，請在您的資源類別中新增 `with` 方法。此方法應回傳一個中繼資料陣列，且僅在該資源是被轉換的最外層資源時，才會被包含在資源回應中：

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
#### 在建構資源時新增中繼資料

您也可以在路由或控制器中建構資源實例時新增頂層資料。所有資源都可用的 `additional` 方法接收一個資料陣列，該陣列將被新增到資源回應中：

```php
return User::all()
    ->load('roles')
    ->toResourceCollection()
    ->additional(['meta' => [
        'key' => 'value',
    ]]);
```

<a name="jsonapi-resources"></a>
## JSON:API 資源

Laravel 內建 `JsonApiResource`，這是一個能產生符合 [JSON:API 規範](https://jsonapi.org/) 回應的資源類別。它繼承自標準的 `JsonResource` 類別，並自動處理資源物件結構、關聯、稀疏欄位集 (sparse fieldsets)、包含 (includes)、延遲屬性評估，並將 `Content-Type` 標頭設定為 `application/vnd.api+json`。

> [!NOTE]
> Laravel 的 JSON:API 資源負責處理回應的序列化。如果您還需要解析傳入的 JSON:API 查詢參數（例如篩選和排序），[Spatie's Laravel Query Builder](https://spatie.be/docs/laravel-query-builder) 是一個非常棒的配套套件。


<a name="generating-jsonapi-resources"></a>
### 產生 JSON:API 資源

要產生 JSON:API 資源，請使用 `make:resource` Artisan 指令並加上 `--json-api` 旗標：

```shell
php artisan make:resource PostResource --json-api
```

產生的類別將繼承 `Illuminate\Http\Resources\JsonApi\JsonApiResource` 並包含 `$attributes` 與 `$relationships` 屬性供您定義：

```php
<?php

namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\JsonApi\JsonApiResource;

class PostResource extends JsonApiResource
{
    /**
     * The resource's attributes.
     */
    public $attributes = [
        // ...
    ];

    /**
     * The resource's relationships.
     */
    public $relationships = [
        // ...
    ];
}
```

JSON:API 資源可以像標準資源一樣從路由和控制器回傳：

```php
use App\Http\Resources\PostResource;
use App\Models\Post;

Route::get('/api/posts/{post}', function (Post $post) {
    return new PostResource($post);
});
```

或者，為了方便起見，您可以使用模型的 `toResource` 方法：

```php
Route::get('/api/posts/{post}', function (Post $post) {
    return $post->toResource();
});
```

這將會產生一個符合 JSON:API 規範的回應：

```json
{
    "data": {
        "id": "1",
        "type": "posts",
        "attributes": {
            "title": "Hello World",
            "body": "This is my first post."
        }
    }
}
```

要回傳 JSON:API 資源的集合，請使用 `collection` 方法或 `toResourceCollection` 便利方法：

```php
return PostResource::collection(Post::all());

return Post::all()->toResourceCollection();
```


<a name="defining-jsonapi-attributes"></a>
### 定義屬性

定義 JSON:API 資源中包含哪些屬性有兩種方式。

最簡單的方法是在資源中定義 `$attributes` 屬性。您可以將屬性名稱列為值，這些屬性將直接從底層模型中讀取：

```php
public $attributes = [
    'title',
    'body',
    'created_at',
];
```

如果某個屬性的計算成本很高，您可以透過 `toAttributes` 以閉包 (closure) 的形式回傳，這樣只有在回應中確實需要該屬性時才會進行評估。

或者，如果您想完全控制資源的屬性，可以覆寫資源中的 `toAttributes` 方法：

```php
/**
 * Get the resource's attributes.
 *
 * @return array<string, mixed>
 */
public function toAttributes(Request $request): array
{
    return [
        'title' => $this->title,
        'body' => $this->body,
        'is_published' => fn () => $this->published_at !== null,
        'created_at' => $this->created_at,
        'updated_at' => $this->updated_at,
    ];
}
```


<a name="defining-jsonapi-relationships"></a>
### 定義關聯

JSON:API 資源支援定義符合 JSON:API 規範的關聯。關聯僅在客戶端透過 `include` 查詢參數要求時才會被序列化。


#### The `$relationships` 屬性

您可以在資源中透過 `$relationships` 屬性定義可被包含的關聯：

```php
public $relationships = [
    'author',
    'comments',
];
```

當將關聯名稱列為值時，Laravel 將解析相對應的 Eloquent 關聯並自動偵測合適的資源類別。如果您需要明確指定資源類別，可以將關聯定義為鍵 / 類別對：

```php
use App\Http\Resources\UserResource;

public $relationships = [
    'author' => UserResource::class,
    'comments',
];
```

或者，您可以覆寫資源中的 `toRelationships` 方法：

```php
/**
 * Get the resource's relationships.
 */
public function toRelationships(Request $request): array
{
    return [
        'author' => UserResource::class,
        'comments' => fn () => CommentResource::collection(
            $request->user()->is($this->resource)
                ? $this->comments
                : $this->comments->where('is_public', true),
        ),
    ];
}
```

使用閉包可以讓您更精準地控制關聯的負載 (payload)，同時仍能確保只有在客戶端要求時才解析關聯。


#### 包含關聯

客戶端可以使用 `include` 查詢參數來要求相關資源：

```
GET /api/posts/1?include=author,comments
```

這會產生一個回應，其中 `relationships` 鍵包含資源識別碼物件，而頂層的 `included` 陣列則包含完整的資源物件：

```json
{
    "data": {
        "id": "1",
        "type": "posts",
        "attributes": {
            "title": "Hello World"
        },
        "relationships": {
            "author": {
                "data": {
                    "id": "1",
                    "type": "users"
                }
            },
            "comments": {
                "data": [
                    {
                        "id": "1",
                        "type": "comments"
                    }
                ]
            }
        }
    },
    "included": [
        {
            "id": "1",
            "type": "users",
            "attributes": {
                "name": "Taylor Otwell"
            }
        },
        {
            "id": "1",
            "type": "comments",
            "attributes": {
                "body": "Great post!"
            }
        }
    ]
}
```

巢狀關聯可以使用點號表示法 (dot notation) 來包含：

```
GET /api/posts/1?include=comments.author
```


<a name="jsonapi-relationship-depth"></a>
#### 關聯深度

預設情況下，巢狀關聯的包含被限制在最大深度內。您可以使用 `maxRelationshipDepth` 方法來自訂此限制，通常設定在應用程式的服務提供者中：

```php
use Illuminate\Http\Resources\JsonApi\JsonApiResource;

JsonApiResource::maxRelationshipDepth(3);
```


<a name="jsonapi-resource-type-and-id"></a>
### 資源類型與 ID

預設情況下，資源的 `type` 是根據資源類別名稱衍生而來。例如，`PostResource` 會產生 `posts` 類型，而 `BlogPostResource` 會產生 `blog-posts`。資源的 `id` 則從模型的主鍵解析而來。

如果您需要自訂這些值，可以覆寫資源中的 `toType` 和 `toId` 方法：

```php
/**
 * Get the resource's type.
 */
public function toType(Request $request): string
{
    return 'articles';
}

/**
 * Get the resource's ID.
 */
public function toId(Request $request): string
{
    return (string) $this->uuid;
}
```

當資源的類型應該與其類別名稱不同時，這特別有用。例如，當 `AuthorResource` 包裹一個 `User` 模型且應該輸出 `authors` 類型時。

<a name="jsonapi-sparse-fieldsets-and-includes"></a>
### 稀疏欄位集與包含

JSON:API 資源支援 [稀疏欄位集(sparse fieldsets)](https://jsonapi.org/format/#fetching-sparse-fieldsets)，允許客戶端使用 `fields` 查詢參數為每個資源類型僅請求特定的屬性：

```
GET /api/posts?fields[posts]=title,created_at&fields[users]=name
```

這將使 `posts` 資源僅包含 `title` 與 `created_at` 屬性，且 `users` 資源僅包含 `name` 屬性。


<a name="jsonapi-ignoring-query-string"></a>
#### 忽略查詢字串

如果您想要為特定的資源回應停用稀疏欄位集過濾，可以呼叫 `ignoreFieldsAndIncludesInQueryString` 方法：

```php
return $post->toResource()
    ->ignoreFieldsAndIncludesInQueryString();
```


<a name="jsonapi-including-previously-loaded-relationships"></a>
#### 包含先前已載入的關聯

預設情況下，關聯僅在透過 `include` 查詢參數請求時才會包含在回應中。如果您希望無論查詢字串為何，都要包含所有先前已預先載入 (eager-loaded) 的關聯，可以呼叫 `includePreviouslyLoadedRelationships` 方法：

```php
return $post->load('author', 'comments')
    ->toResource()
    ->includePreviouslyLoadedRelationships();
```


<a name="jsonapi-links-and-meta"></a>
### 連結與中繼資料

您可以透過覆寫資源上的 `toLinks` 與 `toMeta` 方法，為您的 JSON:API 資源物件新增連結與中繼資料：

```php
/**
 * Get the resource's links.
 */
public function toLinks(Request $request): array
{
    return [
        'self' => route('api.posts.show', $this->resource),
    ];
}

/**
 * Get the resource's meta information.
 */
public function toMeta(Request $request): array
{
    return [
        'readable_created_at' => $this->created_at->diffForHumans(),
    ];
}
```

這將在回應的資源物件中新增 `links` 與 `meta` 鍵：

```json
{
    "data": {
        "id": "1",
        "type": "posts",
        "attributes": {
            "title": "Hello World"
        },
        "links": {
            "self": "https://example.com/api/posts/1"
        },
        "meta": {
            "readable_created_at": "2 hours ago"
        }
    }
}
```

<a name="resource-responses"></a>
## 資源回應

正如您先前所讀到的，資源可以直接從路由與控制器回傳：

```php
use App\Models\User;

Route::get('/user/{id}', function (string $id) {
    return User::findOrFail($id)->toResource();
});
```

然而，有時您可能需要在 HTTP 回應傳送到用戶端之前對其進行自定義。有兩種方法可以實現此目的。首先，您可以在資源後方鏈接 `response` 方法。此方法將回傳一個 `Illuminate\Http\JsonResponse` 實例，讓您可以完全控制回應的標頭：

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

或者，您可以在資源本身定義 `withResponse` 方法。當資源作為回應中的最外層資源被回傳時，此方法會被呼叫：

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