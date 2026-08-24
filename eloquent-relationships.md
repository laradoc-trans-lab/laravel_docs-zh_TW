# Eloquent：關聯

- [簡介](#introduction)
- [定義關聯](#defining-relationships)
    - [一對一 / Has One](#one-to-one)
    - [一對多 / Has Many](#one-to-many)
    - [一對多（反向）/ Belongs To](#one-to-many-inverse)
    - [Has One of Many](#has-one-of-many)
    - [Has One Through](#has-one-through)
    - [Has Many Through](#has-many-through)
- [作用域關聯](#scoped-relationships)
- [多對多關聯](#many-to-many)
    - [取得中間資料表欄位](#retrieving-intermediate-table-columns)
    - [透過中間資料表欄位篩選查詢](#filtering-queries-via-intermediate-table-columns)
    - [透過中間資料表欄位排序查詢](#ordering-queries-via-intermediate-table-columns)
    - [定義自訂中間資料表 Model](#defining-custom-intermediate-table-models)
- [多型關聯](#polymorphic-relationships)
    - [一對一](#one-to-one-polymorphic-relations)
    - [一對多](#one-to-many-polymorphic-relations)
    - [One of Many](#one-of-many-polymorphic-relations)
    - [多對多](#many-to-many-polymorphic-relations)
    - [自訂多型型別](#custom-polymorphic-types)
- [動態關聯](#dynamic-relationships)
- [查詢關聯](#querying-relations)
    - [關聯方法與動態屬性](#relationship-methods-vs-dynamic-properties)
    - [查詢關聯是否存在](#querying-relationship-existence)
    - [查詢關聯是否不存在](#querying-relationship-absence)
    - [查詢 Morph To 關聯](#querying-morph-to-relationships)
- [聚合關聯 Model](#aggregating-related-models)
    - [計算關聯 Model 數量](#counting-related-models)
    - [其他聚合函式](#other-aggregate-functions)
    - [計算 Morph To 關聯上的關聯 Model 數量](#counting-related-models-on-morph-to-relationships)
- [預先載入](#eager-loading)
    - [限制預先載入](#constraining-eager-loads)
    - [延遲預先載入](#lazy-eager-loading)
    - [自動預先載入](#automatic-eager-loading)
    - [防止延遲載入](#preventing-lazy-loading)
- [新增與更新關聯 Model](#inserting-and-updating-related-models)
    - [`save` 方法](#the-save-method)
    - [`create` 方法](#the-create-method)
    - [Belongs To 關聯](#updating-belongs-to-relationships)
    - [多對多關聯](#updating-many-to-many-relationships)
- [觸發更新父層時間戳記](#touching-parent-timestamps)

<a name="introduction"></a>
## 簡介

資料庫資料表通常彼此相互關聯。例如，一篇部落格文章可能有多則留言，或者一筆訂單可能與建立該訂單的使用者相關聯。Eloquent 讓管理與操作這些關聯變得輕鬆簡單，並且支援多種常見的關聯類型：

<div class="content-list" markdown="1">

- [一對一](#one-to-one)
- [一對多](#one-to-many)
- [多對多](#many-to-many)
- [Has One Through](#has-one-through)
- [Has Many Through](#has-many-through)
- [一對一（多型）](#one-to-one-polymorphic-relations)
- [一對多（多型）](#one-to-many-polymorphic-relations)
- [多對多（多型）](#many-to-many-polymorphic-relations)

</div>

<a name="defining-relationships"></a>
## 定義關聯

Eloquent 關聯是定義在 Eloquent Model 類別中的方法。由於關聯也兼具強大的[查詢建構器](/docs/{{version}}/queries)功能，因此將關聯定義為方法能提供強大的方法鏈結與查詢能力。例如，我們可以在此 `posts` 關聯上鏈結額外的查詢條件約束：

```php
$user->posts()->where('active', 1)->get();
```

不過，在深入探討如何使用關聯之前，讓我們先來了解如何定義 Eloquent 所支援的每種類型關聯。


<a name="one-to-one"></a>
### 一對一 / Has One

一對一關聯是一種非常基礎的資料庫關聯型態。例如，一個 `User` Model 可能會關聯一個 `Phone` Model。為了定義此關聯，我們會在 `User` Model 中放置一個 `phone` 方法。該 `phone` 方法應該呼叫 `hasOne` 方法並回傳其結果。你的 Model 可以透過其 `Illuminate\Database\Eloquent\Model` 基底類別來使用 `hasOne` 方法：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\HasOne;

class User extends Model
{
    /**
     * Get the phone associated with the user.
     */
    public function phone(): HasOne
    {
        return $this->hasOne(Phone::class);
    }
}
```

傳入 `hasOne` 方法的第一個引數是關聯 Model 的類別名稱。一旦定義了關聯，我們就可以使用 Eloquent 的動態屬性來取得關聯的紀錄。動態屬性允許你像存取 Model 上定義的屬性一樣存取關聯方法：

```php
$phone = User::find(1)->phone;
```

Eloquent 會根據父 Model 名稱來判斷關聯的外鍵。在此例中，`Phone` Model 會被自動假設擁有一個 `user_id` 外鍵。如果你希望覆寫此慣例，可以傳遞第二個引數給 `hasOne` 方法：

```php
return $this->hasOne(Phone::class, 'foreign_key');
```

此外，Eloquent 預設外鍵的值應該與父層的主鍵欄位值相符。換句話說，Eloquent 會在 `Phone` 紀錄的 `user_id` 欄位中尋找使用者的 `id` 欄位值。如果你希望關聯使用除了 `id` 或 Model 主鍵之外的值，可以傳遞第三個引數給 `hasOne` 方法：

```php
return $this->hasOne(Phone::class, 'foreign_key', 'local_key');
```


<a name="one-to-one-defining-the-inverse-of-the-relationship"></a>
#### 定義反向關聯

現在我們可以從 `User` Model 存取 `Phone` Model。接著，讓我們在 `Phone` Model 上定義一個能讓我們存取擁有該電話之使用者的關聯。我們可以使用 `belongsTo` 方法來定義 `hasOne` 關聯的反向關聯：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class Phone extends Model
{
    /**
     * Get the user that owns the phone.
     */
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }
}
```

當呼叫 `user` 方法時，Eloquent 會嘗試尋找其 `id` 與 `Phone` Model 上的 `user_id` 欄位相符的 `User` Model。

Eloquent 會透過檢查關聯方法的名稱，並在方法名稱後加上後綴 `_id` 來決定外鍵名稱。因此在此例中，Eloquent 會假設 `Phone` Model 擁有一個 `user_id` 欄位。然而，如果 `Phone` Model 上的外鍵不是 `user_id`，你可以傳遞自訂鍵名作為 `belongsTo` 方法的第二個引數：

```php
/**
 * Get the user that owns the phone.
 */
public function user(): BelongsTo
{
    return $this->belongsTo(User::class, 'foreign_key');
}
```

如果父 Model 不是使用 `id` 作為其主鍵，或者你希望使用不同的欄位來尋找關聯的 Model，你可以傳遞第三個引數給 `belongsTo` 方法，以指定父資料表的自訂鍵名：

```php
/**
 * Get the user that owns the phone.
 */
public function user(): BelongsTo
{
    return $this->belongsTo(User::class, 'foreign_key', 'owner_key');
}
```


<a name="one-to-many"></a>
### 一對多 / Has Many

一對多關聯用於定義單一 Model 是單個或多個子 Model 之父層的關聯。例如，一篇部落格文章可能有無數則留言。如同所有其他 Eloquent 關聯一樣，一對多關聯是透過在 Eloquent Model 上定義一個方法來建立的：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\HasMany;

class Post extends Model
{
    /**
     * Get the comments for the blog post.
     */
    public function comments(): HasMany
    {
        return $this->hasMany(Comment::class);
    }
}
```

請記住，Eloquent 會自動為 `Comment` Model 決定適當的外鍵欄位。依照慣例，Eloquent 會採用父 Model 名稱的 "snake case" 形式並加上後綴 `_id`。因此在此範例中，Eloquent 會假設 `Comment` Model 上的外鍵欄位是 `post_id`。

一旦定義了關聯方法，我們就可以透過存取 `comments` 屬性來取得關聯留言的[集合](/docs/{{version}}/eloquent-collections)。請記住，由於 Eloquent 提供了「動態關聯屬性」，我們可以像存取 Model 上定義的屬性一樣存取關聯方法：

```php
use App\Models\Post;

$comments = Post::find(1)->comments;

foreach ($comments as $comment) {
    // ...
}
```

由於所有關聯也都兼具查詢建構器的功能，你可以透過呼叫 `comments` 方法並繼續在查詢上鏈結條件約束，為關聯查詢增加更多限制：

```php
$comment = Post::find(1)->comments()
    ->where('title', 'foo')
    ->first();
```

與 `hasOne` 方法相同，你也可以透過傳遞額外的引數給 `hasMany` 方法來覆寫外鍵與本地鍵：

```php
return $this->hasMany(Comment::class, 'foreign_key');

return $this->hasMany(Comment::class, 'foreign_key', 'local_key');
```


<a name="automatically-hydrating-parent-models-on-children"></a>
#### 自動在子 Model 上載入父 Model

即使使用了 Eloquent 預先載入，當你在迴圈遍歷子 Model 時嘗試從子 Model 存取父 Model，仍可能產生「N + 1」查詢問題：

```php
$posts = Post::with('comments')->get();

foreach ($posts as $post) {
    foreach ($post->comments as $comment) {
        echo $comment->post->title;
    }
}
```

在上面的範例中，即使每個 `Post` Model 都已經預先載入了 comments，但由於 Eloquent 並不會自動在每個子 `Comment` Model 上載入父層 `Post`，因而引入了「N + 1」查詢問題。

如果你希望 Eloquent 自動將父 Model 填入到它們的子 Model 中，可以在定義 `hasMany` 關聯時呼叫 `chaperone` 方法：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\HasMany;

class Post extends Model
{
    /**
     * Get the comments for the blog post.
     */
    public function comments(): HasMany
    {
        return $this->hasMany(Comment::class)->chaperone();
    }
}
```

或者，如果你想在執行階段選擇性啟用自動父層載入，可以在預先載入關聯時呼叫 `chaperone` 方法：

```php
use App\Models\Post;

$posts = Post::with([
    'comments' => fn ($comments) => $comments->chaperone(),
])->get();
```

<a name="one-to-many-inverse"></a>
### 一對多（反向）/ Belongs To

現在我們已經可以存取文章的所有留言，接著讓我們定義一個關聯，讓留言能夠存取其所屬的文章。若要定義 `hasMany` 關聯的反向關聯，可以在子 Model 上定義一個呼叫 `belongsTo` 方法的關聯方法：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class Comment extends Model
{
    /**
     * Get the post that owns the comment.
     */
    public function post(): BelongsTo
    {
        return $this->belongsTo(Post::class);
    }
}
```

定義好關聯後，我們可以透過存取 `post`「動態關聯屬性」來取得留言所屬的文章：

```php
use App\Models\Comment;

$comment = Comment::find(1);

return $comment->post->title;
```

在上述範例中，Eloquent 會嘗試尋找其 `id` 與 `Comment` Model 上的 `post_id` 欄位相互符合的 `Post` Model。

Eloquent 預設的外鍵名稱是透過檢查關聯方法的名稱，並在方法名稱後加上 `_` 以及父層 Model 的主鍵欄位名稱來決定的。因此，在這個範例中，Eloquent 會假設 `Post` Model 在 `comments` 資料表上的外鍵為 `post_id`。

不過，如果你的關聯外鍵並未遵循這些慣例，你可以傳入自訂的外鍵名稱作為 `belongsTo` 方法的第二個引數：

```php
/**
 * Get the post that owns the comment.
 */
public function post(): BelongsTo
{
    return $this->belongsTo(Post::class, 'foreign_key');
}
```

若你的父層 Model 不是使用 `id` 作為其主鍵，或者你希望透過不同的欄位來尋找關聯的 Model，可以傳入第三個引數給 `belongsTo` 方法來指定父層資料表的自訂鍵：

```php
/**
 * Get the post that owns the comment.
 */
public function post(): BelongsTo
{
    return $this->belongsTo(Post::class, 'foreign_key', 'owner_key');
}
```


<a name="default-models"></a>
#### 預設 Model

`belongsTo`、`hasOne`、`hasOneThrough` 以及 `morphOne` 關聯允許你定義一個預設 Model，當給定的關聯為 `null` 時將會回傳該預設 Model。這種模式通常被稱為[空物件模式 (Null Object pattern)](https://en.wikipedia.org/wiki/Null_Object_pattern)，有助於減少程式碼中的條件判斷式。在以下範例中，若 `Post` Model 沒有關聯任何使用者，則 `user` 關聯將回傳一個空的 `App\Models\User` Model：

```php
/**
 * Get the author of the post.
 */
public function user(): BelongsTo
{
    return $this->belongsTo(User::class)->withDefault();
}
```

若要為預設 Model 填入預設屬性，你可以傳入陣列或閉包 (Closure) 給 `withDefault` 方法：

```php
/**
 * Get the author of the post.
 */
public function user(): BelongsTo
{
    return $this->belongsTo(User::class)->withDefault([
        'name' => 'Guest Author',
    ]);
}

/**
 * Get the author of the post.
 */
public function user(): BelongsTo
{
    return $this->belongsTo(User::class)->withDefault(function (User $user, Post $post) {
        $user->name = 'Guest Author';
    });
}
```


<a name="querying-belongs-to-relationships"></a>
#### 查詢 Belongs To 關聯

當查詢「belongs to」關聯的子資料時，你可以手動建立 `where` 子句來取得對應的 Eloquent Model：

```php
use App\Models\Post;

$posts = Post::where('user_id', $user->id)->get();
```

然而，使用 `whereBelongsTo` 方法會更加方便，它會自動為給定的 Model 判斷正確的關聯與外鍵：

```php
$posts = Post::whereBelongsTo($user)->get();
```

你也可以傳入一個[集合 (Collection)](/docs/{{version}}/eloquent-collections) 實例給 `whereBelongsTo` 方法。這樣做時，Laravel 將會取得屬於該集合中任何父層 Model 的所有 Model：

```php
$users = User::where('vip', true)->get();

$posts = Post::whereBelongsTo($users)->get();
```

預設情況下，Laravel 會根據 Model 的類別名稱來判斷與給定 Model 相關聯的關係；不過，你也可以透過提供第二個引數給 `whereBelongsTo` 方法來手動指定關聯名稱：

```php
$posts = Post::whereBelongsTo($user, 'author')->get();
```

<a name="has-one-of-many"></a>
### Has One of Many

有時一個 Model 可能擁有許多關聯的 Model，但你希望能輕鬆取得該關聯中「最新」或「最舊」的關聯 Model。例如，一個 `User` Model 可能關聯許多 `Order` Model，但你想定義一個方便的方法來與該使用者最近建立的訂單互動。你可以透過將 `hasOne` 關聯型別與 `ofMany` 方法結合來達成此目的：

```php
/**
 * Get the user's most recent order.
 */
public function latestOrder(): HasOne
{
    return $this->hasOne(Order::class)->latestOfMany();
}
```

同樣地，你可以定義一個方法來取得關聯中「最舊」或第一筆關聯的 Model：

```php
/**
 * Get the user's oldest order.
 */
public function oldestOrder(): HasOne
{
    return $this->hasOne(Order::class)->oldestOfMany();
}
```

預設情況下，`latestOfMany` 與 `oldestOfMany` 方法會根據 Model 的主鍵（該主鍵必須是可排序的）來取得最新或最舊的關聯 Model。然而，有時你可能希望使用不同的排序條件從較大的關聯中取得單一 Model。

例如，使用 `ofMany` 方法，你可以取得使用者金額最高的訂單。`ofMany` 方法的第一個引數接受可排序的欄位，第二個引數則接受在查詢關聯 Model 時要套用的聚合函式（`min` 或 `max`）：

```php
/**
 * Get the user's largest order.
 */
public function largestOrder(): HasOne
{
    return $this->hasOne(Order::class)->ofMany('price', 'max');
}
```

> [!WARNING]
> 由於 PostgreSQL 不支援對 UUID 欄位執行 `MAX` 函式，因此目前無法將 one-of-many 關聯與 PostgreSQL 的 UUID 欄位結合使用。

<a name="converting-many-relationships-to-has-one-relationships"></a>
#### 將「多」關聯轉換為 Has One 關聯

通常，當使用 `latestOfMany`、`oldestOfMany` 或 `ofMany` 方法取得單一 Model 時，你已經為同一個 Model 定義了「has many」關聯。為了方便起見，Laravel 允許你藉由在關聯上呼叫 `one` 方法，輕鬆地將此關聯轉換為「has one」關聯：

```php
/**
 * Get the user's orders.
 */
public function orders(): HasMany
{
    return $this->hasMany(Order::class);
}

/**
 * Get the user's largest order.
 */
public function largestOrder(): HasOne
{
    return $this->orders()->one()->ofMany('price', 'max');
}
```

你也可以使用 `one` 方法將 `HasManyThrough` 關聯轉換為 `HasOneThrough` 關聯：

```php
public function latestDeployment(): HasOneThrough
{
    return $this->deployments()->one()->latestOfMany();
}
```

<a name="advanced-has-one-of-many-relationships"></a>
#### 進階 Has One of Many 關聯

構建更進階的「has one of many」關聯是可行的。例如，一個 `Product` Model 可能有許多關聯的 `Price` Model，即使發布了新定價，這些 Model 仍會保留在系統中。此外，產品的新定價資料可能可以透過 `published_at` 欄位提前發布，以便在未來的日期生效。

總結來說，我們需要取得發布日期不在未來的最新發布定價。此外，如果兩個定價具有相同的發布日期，我們將偏好選擇 ID 較大的定價。為了達成這一點，我們必須傳遞一個包含決定最新價格之可排序欄位的陣列給 `ofMany` 方法。此外，還會提供一個 Closure 作為 `ofMany` 方法的第二個引數。此 Closure 將負責為關聯查詢加入額外的發布日期條件限制：

```php
/**
 * Get the current pricing for the product.
 */
public function currentPricing(): HasOne
{
    return $this->hasOne(Price::class)->ofMany([
        'published_at' => 'max',
        'id' => 'max',
    ], function (Builder $query) {
        $query->where('published_at', '<', now());
    });
}
```

<a name="has-one-through"></a>
### Has One Through

「has-one-through」關聯定義了與另一個 Model 的一對一關聯。然而，此關聯表示宣告的 Model 可以 _透過_ 第三個 Model 與另一個 Model 的實例進行匹配。

例如，在修車廠應用程式中，每個 `Mechanic` Model 可以與一個 `Car` Model 關聯，而每個 `Car` Model 可以與一個 `Owner` Model 關聯。雖然技工與車主在資料庫中沒有直接關聯，但技工可以 _透過_ `Car` Model 存取車主。讓我們來看看定義此關聯所需的資料表：

```text
mechanics
    id - integer
    name - string

cars
    id - integer
    model - string
    mechanic_id - integer

owners
    id - integer
    name - string
    car_id - integer
```

既然我們已經檢視了該關聯的資料表結構，接著讓我們在 `Mechanic` Model 上定義該關聯：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\HasOneThrough;

class Mechanic extends Model
{
    /**
     * Get the car's owner.
     */
    public function carOwner(): HasOneThrough
    {
        return $this->hasOneThrough(Owner::class, Car::class);
    }
}
```

傳遞給 `hasOneThrough` 方法的第一個引數是我們希望存取的最終 Model 名稱，而第二個引數是中間 Model 的名稱。

或者，如果關聯中涉及的所有 Model 上都已經定義了相關的關聯，你可以藉由呼叫 `through` 方法並提供這些關聯的名稱，以流暢的方式定義「has-one-through」關聯。例如，如果 `Mechanic` Model 具有 `cars` 關聯，而 `Car` Model 具有 `owner` 關聯，你可以像這樣定義一個連接技工與車主的「has-one-through」關聯：

```php
// String based syntax...
return $this->through('cars')->has('owner');

// Dynamic syntax...
return $this->throughCars()->hasOwner();
```

<a name="has-one-through-key-conventions"></a>
#### 鍵值慣例

執行關聯查詢時將會使用典型的 Eloquent 外鍵慣例。如果你想要自訂關聯的鍵值，可以將它們作為第三和第四個引數傳遞給 `hasOneThrough` 方法。第三個引數是中間 Model 上的外鍵名稱。第四個引數是最終 Model 上的外鍵名稱。第五個引數是本地鍵，而第六個引數則是中間 Model 的本地鍵：

```php
class Mechanic extends Model
{
    /**
     * Get the car's owner.
     */
    public function carOwner(): HasOneThrough
    {
        return $this->hasOneThrough(
            Owner::class,
            Car::class,
            'mechanic_id', // Foreign key on the cars table...
            'car_id', // Foreign key on the owners table...
            'id', // Local key on the mechanics table...
            'id' // Local key on the cars table...
        );
    }
}
```

或者，如前所述，如果關聯中涉及的所有 Model 都已經定義了相關的關聯，你可以藉由呼叫 `through` 方法並提供這些關聯的名稱，流暢地定義「has-one-through」關聯。這種做法的好處是可以重複使用既有關聯上已定義的鍵值慣例：

```php
// String based syntax...
return $this->through('cars')->has('owner');

// Dynamic syntax...
return $this->throughCars()->hasOwner();
```

<a name="has-many-through"></a>
### Has Many Through

「Has Many Through」關聯提供了透過中間關聯來存取遠端關聯的便捷方式。例如，假設我們正在建立一個類似 [Laravel Cloud](https://cloud.laravel.com) 的部署平台。`Application` Model 可以透過中間的 `Environment` Model 存取多個 `Deployment` Model。使用這個範例，你可以輕鬆收集給定應用程式的所有部署。讓我們看看定義此關聯所需的資料表：

```text
applications
    id - integer
    name - string

environments
    id - integer
    application_id - integer
    name - string

deployments
    id - integer
    environment_id - integer
    commit_hash - string
```

現在我們已經檢查了關聯的資料表結構，接著讓我們在 `Application` Model 上定義此關聯：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\HasManyThrough;

class Application extends Model
{
    /**
     * Get all of the deployments for the application.
     */
    public function deployments(): HasManyThrough
    {
        return $this->hasManyThrough(Deployment::class, Environment::class);
    }
}
```

傳遞給 `hasManyThrough` 方法的第一個引數是我們希望存取的最終 Model 名稱，而第二個引數則是中間 Model 的名稱。

或者，如果相關關聯已經在關聯中涉及的所有 Model 上定義過，你可以透過呼叫 `through` 方法並提供這些關聯的名稱，以流暢的方式定義「Has Many Through」關聯。例如，如果 `Application` Model 具有 `environments` 關聯，且 `Environment` Model 具有 `deployments` 關聯，你可以像這樣定義連接應用程式與部署的「Has Many Through」關聯：

```php
// String based syntax...
return $this->through('environments')->has('deployments');

// Dynamic syntax...
return $this->throughEnvironments()->hasDeployments();
```

雖然 `Deployment` Model 的資料表不包含 `application_id` 欄位，但 `hasManyThrough` 關聯提供了透過 `$application->deployments` 存取應用程式部署的方式。為了取得這些 Model，Eloquent 會檢查中間 `Environment` Model 資料表上的 `application_id` 欄位。找到相關的環境 ID 後，它們將用於查詢 `Deployment` Model 的資料表。


<a name="has-many-through-key-conventions"></a>
#### 鍵值慣例

執行關聯查詢時，將使用標準的 Eloquent 外鍵慣例。如果你想要自訂關聯的鍵值，可以將它們作為第三和第四個引數傳遞給 `hasManyThrough` 方法。第三個引數是中間 Model 上的外鍵名稱。第四個引數是最終 Model 上的外鍵名稱。第五個引數是本地鍵，而第六個引數則是中間 Model 的本地鍵：

```php
class Application extends Model
{
    public function deployments(): HasManyThrough
    {
        return $this->hasManyThrough(
            Deployment::class,
            Environment::class,
            'application_id', // Foreign key on the environments table...
            'environment_id', // Foreign key on the deployments table...
            'id', // Local key on the applications table...
            'id' // Local key on the environments table...
        );
    }
}
```

或者，如前所述，如果相關關聯已經在關聯中涉及的所有 Model 上定義過，你可以透過呼叫 `through` 方法並提供這些關聯的名稱，以流暢的方式定義「Has Many Through」關聯。這種方法的優點在於可以重複使用現有關聯上已定義的鍵值慣例：

```php
// String based syntax...
return $this->through('environments')->has('deployments');

// Dynamic syntax...
return $this->throughEnvironments()->hasDeployments();
```


<a name="scoped-relationships"></a>
### 作用域關聯

為 Model 新增用來約束關聯的額外方法是很常見的做法。例如，你可以在 `User` Model 中新增一個 `featuredPosts` 方法，該方法透過額外的 `where` 約束來限制更廣泛的 `posts` 關聯：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\HasMany;

class User extends Model
{
    /**
     * Get the user's posts.
     */
    public function posts(): HasMany
    {
        return $this->hasMany(Post::class)->latest();
    }

    /**
     * Get the user's featured posts.
     */
    public function featuredPosts(): HasMany
    {
        return $this->posts()->where('featured', true);
    }
}
```

然而，如果你嘗試透過 `featuredPosts` 方法建立 Model，其 `featured` 屬性不會被設定為 `true`。如果你想要透過關聯方法建立 Model，同時指定應該新增至透過該關聯建立的所有 Model 上的屬性，可以在建立關聯查詢時使用 `withAttributes` 方法：

```php
/**
 * Get the user's featured posts.
 */
public function featuredPosts(): HasMany
{
    return $this->posts()->withAttributes(['featured' => true]);
}
```

`withAttributes` 方法將使用給定的屬性向查詢新增 `where` 條件，並且還會將給定的屬性新增至透過關聯方法建立的任何 Model 中：

```php
$post = $user->featuredPosts()->create(['title' => 'Featured Post']);

$post->featured; // true
```

若要指示 `withAttributes` 方法不要向查詢新增 `where` 條件，可以將 `asConditions` 引數設定為 `false`：

```php
return $this->posts()->withAttributes(['featured' => true], asConditions: false);
```

<a name="many-to-many"></a>
## 多對多關聯

多對多關聯比 `hasOne` 和 `hasMany` 關聯稍微複雜一些。多對多關聯的一個範例是一位使用者擁有多個角色，而這些角色也同時被應用程式中的其他使用者共享。例如，一位使用者可能被分配「作者」和「編輯」角色；然而，這些角色也可能同時被分配給其他使用者。因此，一位使用者擁有多個角色，而一個角色也擁有多位使用者。


<a name="many-to-many-table-structure"></a>
#### 資料表結構

要定義此關聯，需要三個資料庫資料表：`users`、`roles` 和 `role_user`。`role_user` 資料表是依照關聯 Model 名稱的字母順序命名的，並包含 `user_id` 和 `role_id` 欄位。此資料表用作連接使用者與角色的中間資料表。

請記住，由於一個角色可以屬於多位使用者，因此我們不能簡單地在 `roles` 資料表上放置 `user_id` 欄位。這將意味著一個角色只能屬於單一使用者。為了支援將角色分配給多位使用者，需要 `role_user` 資料表。我們可以將此關聯的資料表結構總結如下：

```text
users
    id - integer
    name - string

roles
    id - integer
    name - string

role_user
    user_id - integer
    role_id - integer
```


<a name="many-to-many-model-structure"></a>
#### Model 結構

多對多關聯是透過撰寫一個回傳 `belongsToMany` 方法結果的方法來定義的。`belongsToMany` 方法由你應用程式中所有 Eloquent Model 所使用的 `Illuminate\Database\Eloquent\Model` 基底類別提供。例如，讓我們在 `User` Model 上定義一個 `roles` 方法。傳遞給此方法的第一個引數是關聯 Model 類別的名稱：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsToMany;

class User extends Model
{
    /**
     * The roles that belong to the user.
     */
    public function roles(): BelongsToMany
    {
        return $this->belongsToMany(Role::class);
    }
}
```

一旦定義了關聯，你就可以使用 `roles` 動態關聯屬性來存取使用者的角色：

```php
use App\Models\User;

$user = User::find(1);

foreach ($user->roles as $role) {
    // ...
}
```

由於所有關聯同時也作為查詢建構器，因此你可以透過呼叫 `roles` 方法並繼續將條件鏈結到查詢上，來為關聯查詢加入進一步的限制條件：

```php
$roles = User::find(1)->roles()->orderBy('name')->get();
```

為了決定關聯中間資料表的名稱，Eloquent 會按照字母順序組合兩個關聯 Model 的名稱。然而，你可以自由覆寫此慣例。你可以透過傳遞第二個引數給 `belongsToMany` 方法來實現：

```php
return $this->belongsToMany(Role::class, 'role_user');
```

除了自訂中間資料表的名稱之外，你還可以透過傳遞額外的引數給 `belongsToMany` 方法來自訂資料表上的鍵值欄位名稱。第三個引數是你在其上定義關聯的 Model 外鍵名稱，而第四個引數是你所連接到的 Model 外鍵名稱：

```php
return $this->belongsToMany(Role::class, 'role_user', 'user_id', 'role_id');
```


<a name="many-to-many-defining-the-inverse-of-the-relationship"></a>
#### 定義關聯的反向

若要定義多對多關聯的「反向」，你應該在關聯的 Model 上定義一個同樣回傳 `belongsToMany` 方法結果的方法。為了完成我們使用者與角色的範例，讓我們在 `Role` Model 上定義 `users` 方法：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsToMany;

class Role extends Model
{
    /**
     * The users that belong to the role.
     */
    public function users(): BelongsToMany
    {
        return $this->belongsToMany(User::class);
    }
}
```

如你所見，除了參照 `App\Models\User` Model 之外，該關聯的定義方式與其在 `User` Model 中的對應項完全相同。由於我們重複使用了 `belongsToMany` 方法，因此在定義多對多關聯的「反向」時，所有常用的資料表和鍵值自訂選項皆可使用。


<a name="retrieving-intermediate-table-columns"></a>
### 取得中間資料表欄位

正如你所了解的，使用多對多關聯需要中間資料表的存在。Eloquent 提供了一些非常有用的方式來與此資料表進行互動。例如，假設我們的 `User` Model 關聯了許多 `Role` Model。存取此關聯後，我們可以使用 Model 上的 `pivot` 屬性存取中間資料表：

```php
use App\Models\User;

$user = User::find(1);

foreach ($user->roles as $role) {
    echo $role->pivot->created_at;
}
```

請注意，我們取得的每個 `Role` Model 都會自動被指派一個 `pivot` 屬性。此屬性包含一個代表中間資料表的 Model。

預設情況下，`pivot` Model 上僅會存在 Model 的鍵值。如果你的中間資料表包含額外的屬性，則必須在定義關聯時明確指定它們：

```php
return $this->belongsToMany(Role::class)->withPivot('active', 'created_by');
```

如果你希望中間資料表擁有由 Eloquent 自動維護的 `created_at` 和 `updated_at` 時間戳記，請在定義關聯時呼叫 `withTimestamps` 方法：

```php
return $this->belongsToMany(Role::class)->withTimestamps();
```

> [!WARNING]
> 使用 Eloquent 自動維護時間戳記的中間資料表必須同時具備 `created_at` 與 `updated_at` 時間戳記欄位。


<a name="customizing-the-pivot-attribute-name"></a>
#### 自訂 `pivot` 屬性名稱

如前所述，中間資料表的屬性可以透過 Model 上的 `pivot` 屬性來存取。然而，你可以自由自訂此屬性的名稱，以更好地反映其在應用程式中的用途。

例如，如果你的應用程式包含可以訂閱 Podcast 的使用者，那麼在使用者和 Podcast 之間可能存在多對多關聯。在這種情況下，你可能希望將中間資料表的屬性重新命名為 `subscription` 而不是 `pivot`。這可以在定義關聯時使用 `as` 方法來完成：

```php
return $this->belongsToMany(Podcast::class)
    ->as('subscription')
    ->withTimestamps();
```

一旦指定了自訂的中間資料表屬性，你就可以使用自訂的名稱來存取中間資料表的資料：

```php
$users = User::with('podcasts')->get();

foreach ($users->flatMap->podcasts as $podcast) {
    echo $podcast->subscription->created_at;
}
```

<a name="filtering-queries-via-intermediate-table-columns"></a>
### 透過中間資料表欄位篩選查詢

在定義關聯時，你也可以使用 `wherePivot`、`wherePivotIn`、`wherePivotNotIn`、`wherePivotBetween`、`wherePivotNotBetween`、`wherePivotNull` 以及 `wherePivotNotNull` 方法來篩選 `belongsToMany` 關聯查詢所返回的結果：

```php
return $this->belongsToMany(Role::class)
    ->wherePivot('approved', 1);

return $this->belongsToMany(Role::class)
    ->wherePivotIn('priority', [1, 2]);

return $this->belongsToMany(Role::class)
    ->wherePivotNotIn('priority', [1, 2]);

return $this->belongsToMany(Podcast::class)
    ->as('subscriptions')
    ->wherePivotBetween('created_at', ['2020-01-01 00:00:00', '2020-12-31 00:00:00']);

return $this->belongsToMany(Podcast::class)
    ->as('subscriptions')
    ->wherePivotNotBetween('created_at', ['2020-01-01 00:00:00', '2020-12-31 00:00:00']);

return $this->belongsToMany(Podcast::class)
    ->as('subscriptions')
    ->wherePivotNull('expired_at');

return $this->belongsToMany(Podcast::class)
    ->as('subscriptions')
    ->wherePivotNotNull('expired_at');
```

`wherePivot` 會在查詢中加入 where 子句限制，但透過已定義關聯建立新 Model 時不會自動加入指定的值。如果你需要同時查詢並使用特定的 pivot 值建立關聯，可以使用 `withPivotValue` 方法：

```php
return $this->belongsToMany(Role::class)
    ->withPivotValue('approved', 1);
```


<a name="ordering-queries-via-intermediate-table-columns"></a>
### 透過中間資料表欄位排序查詢

你可以使用 `orderByPivot` 與 `orderByPivotDesc` 方法來對 `belongsToMany` 關聯查詢所返回的結果進行排序。在以下範例中，我們將取得該使用者所有最新的徽章：

```php
return $this->belongsToMany(Badge::class)
    ->where('rank', 'gold')
    ->orderByPivotDesc('created_at');
```


<a name="defining-custom-intermediate-table-models"></a>
### 定義自訂中間資料表 Model

如果你想要定義一個自訂 Model 來表示多對多關聯的中間資料表，可以在定義關聯時呼叫 `using` 方法。自訂 pivot Model 讓你有機會在 pivot Model 上定義額外的行為，例如方法與型別轉換。

自訂的多對多 pivot Model 應繼承 `Illuminate\Database\Eloquent\Relations\Pivot` 類別，而自訂的多型多對多 pivot Model 應繼承 `Illuminate\Database\Eloquent\Relations\MorphPivot` 類別。例如，我們可以定義一個使用自訂 `RoleUser` pivot Model 的 `Role` Model：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsToMany;

class Role extends Model
{
    /**
     * The users that belong to the role.
     */
    public function users(): BelongsToMany
    {
        return $this->belongsToMany(User::class)->using(RoleUser::class);
    }
}
```

定義 `RoleUser` Model 時，應繼承 `Illuminate\Database\Eloquent\Relations\Pivot` 類別：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Relations\Pivot;

class RoleUser extends Pivot
{
    // ...
}
```

> [!WARNING]
> Pivot Model 不能使用 `SoftDeletes` trait。如果你需要軟刪除 pivot 記錄，請考慮將你的 pivot Model 轉換為實際的 Eloquent Model。


<a name="custom-pivot-models-and-incrementing-ids"></a>
#### 自訂 Pivot Model 與遞增 ID

如果你定義了使用自訂 pivot Model 的多對多關聯，且該 pivot Model 具有自動遞增的主鍵，你應確保自訂 pivot Model 類別使用 `Table` 屬性並將 `incrementing` 設為 `true`：

```php
use Illuminate\Database\Eloquent\Attributes\Table;
use Illuminate\Database\Eloquent\Relations\Pivot;

#[Table(incrementing: true)]
class RoleUser extends Pivot
{
    // ...
}
```

<a name="polymorphic-relationships"></a>
## 多型關聯

多型關聯允許子 Model 透過單一關聯屬於多種類型的 Model。例如，想像你正在建構一個允許使用者分享部落格文章和影片的應用程式。在這樣的應用程式中，`Comment` Model 可以同時屬於 `Post` 和 `Video` Model。

<a name="one-to-one-polymorphic-relations"></a>
### 一對一

<a name="one-to-one-polymorphic-table-structure"></a>
#### 資料表結構

一對一多型關聯類似於典型的一對一關聯；然而，子 Model 可以透過單一關聯屬於多種類型的 Model。例如，部落格 `Post` 和 `User` 可以共享對 `Image` Model 的多型關聯。使用一對一多型關聯可以讓你擁有一個儲存唯一圖片的單一資料表，並能與文章和使用者建立關聯。首先，讓我們來看看資料表結構：

```text
posts
    id - integer
    name - string

users
    id - integer
    name - string

images
    id - integer
    url - string
    imageable_type - string
    imageable_id - integer
```

請注意 `images` 資料表上的 `imageable_id` 和 `imageable_type` 欄位。`imageable_id` 欄位將包含文章或使用者的 ID 值，而 `imageable_type` 欄位將包含父 Model 的類別名稱。Eloquent 使用 `imageable_type` 欄位來判斷存取 `imageable` 關聯時應回傳哪種類型的父 Model。在此範例中，該欄位將包含 `App\Models\Post` 或 `App\Models\User`。

<a name="one-to-one-polymorphic-model-structure"></a>
#### Model 結構

接下來，讓我們看看建構此關聯所需的 Model 定義：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\MorphTo;

class Image extends Model
{
    /**
     * Get the parent imageable model (user or post).
     */
    public function imageable(): MorphTo
    {
        return $this->morphTo();
    }
}

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\MorphOne;

class Post extends Model
{
    /**
     * Get the post's image.
     */
    public function image(): MorphOne
    {
        return $this->morphOne(Image::class, 'imageable');
    }
}

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\MorphOne;

class User extends Model
{
    /**
     * Get the user's image.
     */
    public function image(): MorphOne
    {
        return $this->morphOne(Image::class, 'imageable');
    }
}
```

<a name="one-to-one-polymorphic-retrieving-the-relationship"></a>
#### 取得關聯

定義好資料庫資料表與 Model 後，你就可以透過 Model 存取關聯。例如，要取得文章的圖片，我們可以存取 `image` 動態關聯屬性：

```php
use App\Models\Post;

$post = Post::find(1);

$image = $post->image;
```

你可以透過存取呼叫 `morphTo` 的方法名稱來取得多型 Model 的父層。在此範例中，即為 `Image` Model 上的 `imageable` 方法。因此，我們將把該方法當作動態關聯屬性來存取：

```php
use App\Models\Image;

$image = Image::find(1);

$imageable = $image->imageable;
```

`Image` Model 上的 `imageable` 關聯將回傳 `Post` 或 `User` 實例，具體取決於擁有該圖片的 Model 類型。

<a name="morph-one-to-one-key-conventions"></a>
#### 鍵名慣例

如有需要，你可以指定多型子 Model 所使用的 "id" 與 "type" 欄位名稱。如果這樣做，請確保一律將關聯名稱作為第一個引數傳入 `morphTo` 方法。通常，此值應與方法名稱相符，因此你可以使用 PHP 的 `__FUNCTION__` 常數：

```php
/**
 * Get the model that the image belongs to.
 */
public function imageable(): MorphTo
{
    return $this->morphTo(__FUNCTION__, 'imageable_type', 'imageable_id');
}
```

<a name="one-to-many-polymorphic-relations"></a>
### 一對多

<a name="one-to-many-polymorphic-table-structure"></a>
#### 資料表結構

一對多多型關聯類似於典型的一對多關聯；然而，子 Model 可以透過單一關聯屬於多種類型的 Model。例如，假設你應用程式的使用者可以對文章 (Post) 和影片 (Video) 發表「留言 (Comment)」。使用多型關聯，你可以使用單一 `comments` 資料表來同時儲存文章和影片的留言。首先，讓我們檢視建立此關聯所需的資料表結構：

```text
posts
    id - integer
    title - string
    body - text

videos
    id - integer
    title - string
    url - string

comments
    id - integer
    body - text
    commentable_type - string
    commentable_id - integer
```

<a name="one-to-many-polymorphic-model-structure"></a>
#### Model 結構

接著，讓我們檢視建立此關聯所需的 Model 定義：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\MorphTo;

class Comment extends Model
{
    /**
     * Get the parent commentable model (post or video).
     */
    public function commentable(): MorphTo
    {
        return $this->morphTo();
    }
}

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\MorphMany;

class Post extends Model
{
    /**
     * Get all of the post's comments.
     */
    public function comments(): MorphMany
    {
        return $this->morphMany(Comment::class, 'commentable');
    }
}

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\MorphMany;

class Video extends Model
{
    /**
     * Get all of the video's comments.
     */
    public function comments(): MorphMany
    {
        return $this->morphMany(Comment::class, 'commentable');
    }
}
```

<a name="one-to-many-polymorphic-retrieving-the-relationship"></a>
#### 取得關聯

一旦定義好資料庫資料表與 Model，你就可以透過 Model 的動態關聯屬性來存取關聯。例如，要存取文章的所有留言，我們可以使用 `comments` 動態屬性：

```php
use App\Models\Post;

$post = Post::find(1);

foreach ($post->comments as $comment) {
    // ...
}
```

你也可以透過存取呼叫 `morphTo` 的方法名稱來取得多型子 Model 的父層。在此範例中，即是 `Comment` Model 上的 `commentable` 方法。因此，我們將把該方法作為動態關聯屬性來存取，以取得留言的父 Model：

```php
use App\Models\Comment;

$comment = Comment::find(1);

$commentable = $comment->commentable;
```

`Comment` Model 上的 `commentable` 關聯將回傳 `Post` 或 `Video` 實例，取決於哪種類型的 Model 擁有該留言。

<a name="polymorphic-automatically-hydrating-parent-models-on-children"></a>
#### 自動在子 Model 上注入父 Model

即使使用了 Eloquent 預先載入，當你在迴圈遍歷子 Model 時嘗試從子 Model 存取父 Model，仍可能產生「N + 1」查詢問題：

```php
$posts = Post::with('comments')->get();

foreach ($posts as $post) {
    foreach ($post->comments as $comment) {
        echo $comment->commentable->title;
    }
}
```

在上述範例中，引發了「N + 1」查詢問題，因為即使為每個 `Post` Model 預先載入了留言，Eloquent 也不會自動在每個子 `Comment` Model 上注入父 `Post`。

如果你希望 Eloquent 自動將父 Model 注入到其子 Model 上，可以在定義 `morphMany` 關聯時呼叫 `chaperone` 方法：

```php
class Post extends Model
{
    /**
     * Get all of the post's comments.
     */
    public function comments(): MorphMany
    {
        return $this->morphMany(Comment::class, 'commentable')->chaperone();
    }
}
```

或者，如果你想在執行階段選擇啟用自動注入父 Model，可以在預先載入關聯時呼叫 `chaperone` 方法：

```php
use App\Models\Post;

$posts = Post::with([
    'comments' => fn ($comments) => $comments->chaperone(),
])->get();
```

<a name="one-of-many-polymorphic-relations"></a>
### One of Many

有時一個 Model 可能有許多關聯的 Model，但你希望能輕鬆取得該關聯中「最新」或「最舊」的關聯 Model。例如，一個 `User` Model 可能關聯到許多 `Image` Model，但你希望定義一個方便的方式來存取該使用者上傳的最新圖片。你可以使用 `morphOne` 關聯型別結合 `ofMany` 方法來達成此目的：

```php
/**
 * Get the user's most recent image.
 */
public function latestImage(): MorphOne
{
    return $this->morphOne(Image::class, 'imageable')->latestOfMany();
}
```

同樣地，你可以定義一個方法來取得關聯中「最舊」或第一個關聯 Model：

```php
/**
 * Get the user's oldest image.
 */
public function oldestImage(): MorphOne
{
    return $this->morphOne(Image::class, 'imageable')->oldestOfMany();
}
```

預設情況下，`latestOfMany` 和 `oldestOfMany` 方法將根據 Model 的主鍵（必須可排序）來取得最新或最舊的關聯 Model。然而，有時你可能希望使用不同的排序條件從較大的關聯中取得單一 Model。

例如，使用 `ofMany` 方法，你可以取得使用者獲得最多「讚 (liked)」的圖片。`ofMany` 方法接受可排序的欄位作為其第一個引數，以及在查詢關聯 Model 時要套用的聚合函式（`min` 或 `max`）：

```php
/**
 * Get the user's most popular image.
 */
public function bestImage(): MorphOne
{
    return $this->morphOne(Image::class, 'imageable')->ofMany('likes', 'max');
}
```

> [!NOTE]
> 可以建構更進階的「one of many」關聯。如需更多資訊，請參閱 [Has One of Many 說明文件](#advanced-has-one-of-many-relationships)。

<a name="many-to-many-polymorphic-relations"></a>
### 多對多

<a name="many-to-many-polymorphic-table-structure"></a>
#### 資料表結構

多對多多型關聯比「一對一多型（morph one）」與「一對多多型（morph many）」關聯稍微複雜一些。例如，`Post` Model 與 `Video` Model 可以共用對 `Tag` Model 的多型關聯。在這種情況下使用多對多多型關聯，可讓您的應用程式擁有一張獨立的標籤資料表，並能與貼文或影片建立關聯。首先，讓我們檢視建立此關聯所需的資料表結構：

```text
posts
    id - integer
    name - string

videos
    id - integer
    name - string

tags
    id - integer
    name - string

taggables
    tag_id - integer
    taggable_type - string
    taggable_id - integer
```

> [!NOTE]
> 在深入探討多對多多型關聯之前，建議先閱讀一般[多對多關聯](#many-to-many)的說明文件。

<a name="many-to-many-polymorphic-model-structure"></a>
#### Model 結構

接著，我們準備在 Model 上定義關聯。`Post` 與 `Video` Model 都將包含一個 `tags` 方法，該方法會呼叫 Eloquent 基底 Model 類別所提供的 `morphToMany` 方法。

`morphToMany` 方法接受關聯 Model 的名稱以及「關聯名稱」。根據我們為中間資料表指定的名稱及其包含的欄位鍵，我們將這個關聯稱為「taggable」：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\MorphToMany;

class Post extends Model
{
    /**
     * Get all of the tags for the post.
     */
    public function tags(): MorphToMany
    {
        return $this->morphToMany(Tag::class, 'taggable');
    }
}
```

<a name="many-to-many-polymorphic-defining-the-inverse-of-the-relationship"></a>
#### 定義反向關聯

接著，在 `Tag` Model 上，您應該為其每一個可能的父層 Model 定義一個方法。因此在這個範例中，我們將定義一個 `posts` 方法與一個 `videos` 方法。這兩個方法都應該回傳 `morphedByMany` 方法的結果。

`morphedByMany` 方法接受關聯 Model 的名稱以及「關聯名稱」。根據我們為中間資料表指定的名稱及其包含的欄位鍵，我們將這個關聯稱為「taggable」：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\MorphToMany;

class Tag extends Model
{
    /**
     * Get all of the posts that are assigned this tag.
     */
    public function posts(): MorphToMany
    {
        return $this->morphedByMany(Post::class, 'taggable');
    }

    /**
     * Get all of the videos that are assigned this tag.
     */
    public function videos(): MorphToMany
    {
        return $this->morphedByMany(Video::class, 'taggable');
    }
}
```

<a name="many-to-many-polymorphic-retrieving-the-relationship"></a>
#### 取得關聯

一旦定義好資料表與 Model，您便可以透過 Model 來存取關聯。例如，若要存取貼文的所有標籤，可以使用 `tags` 動態關聯屬性：

```php
use App\Models\Post;

$post = Post::find(1);

foreach ($post->tags as $tag) {
    // ...
}
```

您可以從多型子 Model 中，藉由存取呼叫 `morphedByMany` 的方法名稱來取得多型關聯的父層。在此範例中，即為 `Tag` Model 上的 `posts` 或 `videos` 方法：

```php
use App\Models\Tag;

$tag = Tag::find(1);

foreach ($tag->posts as $post) {
    // ...
}

foreach ($tag->videos as $video) {
    // ...
}
```

<a name="custom-polymorphic-types"></a>
### 自訂多型型別

預設情況下，Laravel 會使用完整類別名稱（Fully Qualified Class Name）來儲存關聯 Model 的「型別（Type）」。例如，在上述一對多關聯範例中，當 `Comment` Model 可能屬於 `Post` 或 `Video` Model 時，預設的 `commentable_type` 會分別是 `App\Models\Post` 或 `App\Models\Video`。然而，您可能希望將這些值與應用程式的內部結構解耦。

例如，我們可以使用簡單的字串如 `post` 和 `video`，而不是使用 Model 名稱作為「型別」。這樣一來，即使 Model 被重新命名，資料庫中的多型「型別」欄位值仍然有效：

```php
use Illuminate\Database\Eloquent\Relations\Relation;

Relation::enforceMorphMap([
    'post' => 'App\Models\Post',
    'video' => 'App\Models\Video',
]);
```

您可以在 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫 `enforceMorphMap` 方法，或者視需要建立一個獨立的服務提供者。

您可以在執行時期使用 Model 的 `getMorphClass` 方法來取得該 Model 的多型別名（Morph Alias）。反之，您可以使用 `Relation::getMorphedModel` 方法來取得與多型別名相關聯的完整類別名稱：

```php
use Illuminate\Database\Eloquent\Relations\Relation;

$alias = $post->getMorphClass();

$class = Relation::getMorphedModel($alias);
```

> [!WARNING]
> 當為現有應用程式新增「多型對應（Morph Map）」時，資料庫中仍包含完整類別名稱的每一個多型 `*_type` 欄位值都需要轉換為其「對應」名稱。

<a name="dynamic-relationships"></a>
### 動態關聯

您可以使用 `resolveRelationUsing` 方法在執行時期定義 Eloquent Model 之間的關聯。雖然在一般應用程式開發中通常不建議這麼做，但在開發 Laravel 套件時這可能偶爾會很有用。

`resolveRelationUsing` 方法接受所需的關聯名稱作為其第一個引數。傳遞給該方法的第二個引數應該是一個閉包，該閉包接受 Model 實例並回傳一個有效的 Eloquent 關聯定義。通常，您應該在[服務提供者(Service Providers)](/docs/{{version}}/providers)的 boot 方法中設定動態關聯：

```php
use App\Models\Order;
use App\Models\Customer;

Order::resolveRelationUsing('customer', function (Order $orderModel) {
    return $orderModel->belongsTo(Customer::class, 'customer_id');
});
```

> [!WARNING]
> 定義動態關聯時，請務必為 Eloquent 關聯方法提供明確的欄位鍵名稱引數。

<a name="querying-relations"></a>
## 查詢關聯

由於所有 Eloquent 關聯都是透過方法定義的，因此你可以呼叫這些方法來取得關聯的實例，而無需實際執行查詢來載入關聯的 Model。此外，所有類型的 Eloquent 關聯同時也是[查詢建構器 (Query Builder)](/docs/{{version}}/queries)，允許你在最終對資料庫執行 SQL 查詢之前，繼續在關聯查詢上鏈結約束條件。

例如，假設有一個部落格應用程式，其中 `User` Model 擁有許多關聯的 `Post` Model：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\HasMany;

class User extends Model
{
    /**
     * Get all of the posts for the user.
     */
    public function posts(): HasMany
    {
        return $this->hasMany(Post::class);
    }
}
```

你可以查詢 `posts` 關聯並在關聯上加入額外的約束條件，如下所示：

```php
use App\Models\User;

$user = User::find(1);

$user->posts()->where('active', 1)->get();
```

你可以在關聯上使用 Laravel [查詢建構器](/docs/{{version}}/queries)的任何方法，因此請務必閱讀查詢建構器的說明文件，以瞭解所有可用的方法。


<a name="chaining-orwhere-clauses-after-relationships"></a>
#### 在關聯後鏈結 `orWhere` 子句

如上面的範例所示，你在查詢關聯時可以自由加入額外的約束條件。但是，在關聯上鏈結 `orWhere` 子句時請特別小心，因為 `orWhere` 子句在邏輯上會與關聯約束條件處於同一層級：

```php
$user->posts()
    ->where('active', 1)
    ->orWhere('votes', '>=', 100)
    ->get();
```

上面的範例將產生以下 SQL。如你所見，`or` 子句指示查詢返回超過 100 票的「任何」文章。該查詢不再受限於特定使用者：

```sql
select *
from posts
where user_id = ? and active = 1 or votes >= 100
```

在多數情況下，你應該使用[邏輯分組](/docs/{{version}}/queries#logical-grouping)將條件檢查包裹在括號中：

```php
use Illuminate\Database\Eloquent\Builder;

$user->posts()
    ->where(function (Builder $query) {
        return $query->where('active', 1)
            ->orWhere('votes', '>=', 100);
    })
    ->get();
```

上面的範例將產生以下 SQL。請注意，邏輯分組已正確分組了約束條件，並且查詢仍然受限於特定使用者：

```sql
select *
from posts
where user_id = ? and (active = 1 or votes >= 100)
```


<a name="relationship-methods-vs-dynamic-properties"></a>
### 關聯方法與動態屬性

如果你不需要在 Eloquent 關聯查詢中加入額外的約束條件，你可以像存取屬性一樣存取該關聯。例如，繼續使用我們的 `User` 和 `Post` 範例 Model，我們可以像這樣存取某位使用者的所有文章：

```php
use App\Models\User;

$user = User::find(1);

foreach ($user->posts as $post) {
    // ...
}
```

動態關聯屬性會執行「延遲載入 (Lazy Loading)」，這表示它們只會在您實際存取該屬性時才載入關聯資料。因此，開發者經常使用[預先載入](#eager-loading)來預先載入他們知道在載入 Model 後將會存取的關聯。預先載入大幅減少了載入 Model 關聯時必須執行的 SQL 查詢次數。


<a name="querying-relationship-existence"></a>
### 查詢關聯是否存在

取得 Model 記錄時，你可能會希望根據關聯是否存在來限制查詢結果。例如，假設你想取得所有至少有一則留言的部落格文章。為此，你可以將關聯的名稱傳遞給 `has` 和 `orHas` 方法：

```php
use App\Models\Post;

// Retrieve all posts that have at least one comment...
$posts = Post::has('comments')->get();
```

你也可以指定運算子和數量值來進一步自訂查詢：

```php
// Retrieve all posts that have three or more comments...
$posts = Post::has('comments', '>=', 3)->get();
```

可以使用「點 (dot)」標記法建構巢狀的 `has` 語句。例如，你可以取得所有至少有一則留言且該留言至少包含一張圖片的文章：

```php
// Retrieve posts that have at least one comment with images...
$posts = Post::has('comments.images')->get();
```

如果你需要更強大的功能，可以使用 `whereHas` 和 `orWhereHas` 方法在 `has` 查詢上定義額外的查詢約束條件，例如檢查留言的內容：

```php
use Illuminate\Database\Eloquent\Builder;

// Retrieve posts with at least one comment containing words like code%...
$posts = Post::whereHas('comments', function (Builder $query) {
    $query->where('content', 'like', 'code%');
})->get();

// Retrieve posts with at least ten comments containing words like code%...
$posts = Post::whereHas('comments', function (Builder $query) {
    $query->where('content', 'like', 'code%');
}, '>=', 10)->get();
```

> [!WARNING]
> Eloquent 目前不支援跨資料庫查詢關聯是否存在。關聯必須存在於同一個資料庫內。


<a name="many-to-many-relationship-existence-queries"></a>
#### 多對多關聯存在性查詢

`whereAttachedTo` 方法可用於查詢與某個 Model 或 Model 集合具有多對多附加關係的 Model：

```php
$users = User::whereAttachedTo($role)->get();
```

你也可以傳遞 [Collection](/docs/{{version}}/eloquent-collections) 實例給 `whereAttachedTo` 方法。這樣做時，Laravel 將檢索附加到集合中任何一個 Model 的所有 Model：

```php
$tags = Tag::whereLike('name', '%laravel%')->get();

$posts = Post::whereAttachedTo($tags)->get();
```


<a name="inline-relationship-existence-queries"></a>
#### 行內關聯存在性查詢

如果你想要在關聯查詢中附加單一、簡單的 where 條件來查詢關聯是否存在，使用 `whereRelation`、`orWhereRelation`、`whereMorphRelation` 和 `orWhereMorphRelation` 方法會更加方便。例如，我們可以查詢所有具有未核准留言的文章：

```php
use App\Models\Post;

$posts = Post::whereRelation('comments', 'is_approved', false)->get();
```

當然，如同呼叫查詢建構器的 `where` 方法一樣，你也可以指定運算子：

```php
$posts = Post::whereRelation(
    'comments', 'created_at', '>=', now()->minus(hours: 1)
)->get();
```


<a name="querying-relationship-absence"></a>
### 查詢關聯是否不存在

取得 Model 記錄時，你可能會希望根據關聯是否「不存在」來限制查詢結果。例如，假設你想取得所有**沒有**任何留言的部落格文章。為此，你可以將關聯的名稱傳遞給 `doesntHave` 和 `orDoesntHave` 方法：

```php
use App\Models\Post;

$posts = Post::doesntHave('comments')->get();
```

如果你需要更強大的功能，可以使用 `whereDoesntHave` 和 `orWhereDoesntHave` 方法在 `doesntHave` 查詢中加入額外的查詢條件，例如檢查留言的內容：

```php
use Illuminate\Database\Eloquent\Builder;

$posts = Post::whereDoesntHave('comments', function (Builder $query) {
    $query->where('content', 'like', 'code%');
})->get();
```

你可以使用「點」標記法對巢狀關聯執行查詢。例如，以下查詢將取得所有沒有留言的文章，以及有留言但留言都不是來自被封鎖使用者的文章：

```php
use Illuminate\Database\Eloquent\Builder;

$posts = Post::whereDoesntHave('comments.author', function (Builder $query) {
    $query->where('banned', 1);
})->get();
```

<a name="querying-morph-to-relationships"></a>
### 查詢 Morph To 關聯

若要查詢「morph to」關聯是否存在，您可以使用 `whereHasMorph` 與 `whereDoesntHaveMorph` 方法。這些方法接受關聯名稱作為其第一個引數。接著，這些方法接受您希望包含在查詢中的關聯 Model 名稱。最後，您可以提供一個閉包來自訂關聯查詢：

```php
use App\Models\Comment;
use App\Models\Post;
use App\Models\Video;
use Illuminate\Database\Eloquent\Builder;

// Retrieve comments associated to posts or videos with a title like code%...
$comments = Comment::whereHasMorph(
    'commentable',
    [Post::class, Video::class],
    function (Builder $query) {
        $query->where('title', 'like', 'code%');
    }
)->get();

// Retrieve comments associated to posts with a title not like code%...
$comments = Comment::whereDoesntHaveMorph(
    'commentable',
    Post::class,
    function (Builder $query) {
        $query->where('title', 'like', 'code%');
    }
)->get();
```

有時您可能需要根據關聯多型 Model 的「型別」來新增查詢條件。傳遞給 `whereHasMorph` 方法的閉包可以接收一個 `$type` 值作為其第二個引數。此引數允許您檢視正在建構的查詢「型別」：

```php
use Illuminate\Database\Eloquent\Builder;

$comments = Comment::whereHasMorph(
    'commentable',
    [Post::class, Video::class],
    function (Builder $query, string $type) {
        $column = $type === Post::class ? 'content' : 'title';

        $query->where($column, 'like', 'code%');
    }
)->get();
```

有時您可能想要查詢某個「morph to」關聯之父層的子層 Model。您可以使用 `whereMorphedTo` 與 `whereNotMorphedTo` 方法來達成此目的，這些方法會自動為給定的 Model 判斷正確的多型對應。這些方法接受 `morphTo` 關聯的名稱作為其第一個引數，並接受關聯的父層 Model 作為其第二個引數：

```php
$comments = Comment::whereMorphedTo('commentable', $post)
    ->orWhereMorphedTo('commentable', $video)
    ->get();
```


<a name="querying-all-morph-to-related-models"></a>
#### 查詢所有關聯 Model

除了傳遞可能的多型 Model 陣列外，您還可以提供 `*` 作為萬用字元值。這將指示 Laravel 從資料庫中檢索所有可能的多型型別。Laravel 將執行額外的查詢以執行此操作：

```php
use Illuminate\Database\Eloquent\Builder;

$comments = Comment::whereHasMorph('commentable', '*', function (Builder $query) {
    $query->where('title', 'like', 'foo%');
})->get();
```

<a name="aggregating-related-models"></a>
## 聚合關聯 Model


<a name="counting-related-models"></a>
### 計算關聯 Model 數量

有時候您可能想要計算給定關聯的相關 Model 數量，而不需要實際載入這些 Model。為達成此目的，您可以使用 `withCount` 方法。`withCount` 方法會在產生的 Model 上放置一個 `{relation}_count` 屬性：

```php
use App\Models\Post;

$posts = Post::withCount('comments')->get();

foreach ($posts as $post) {
    echo $post->comments_count;
}
```

透過傳遞一個陣列給 `withCount` 方法，您可以為多個關聯新增「計數」，也可以為查詢新增額外的條件限制：

```php
use Illuminate\Database\Eloquent\Builder;

$posts = Post::withCount(['votes', 'comments' => function (Builder $query) {
    $query->where('content', 'like', 'code%');
}])->get();

echo $posts[0]->votes_count;
echo $posts[0]->comments_count;
```

您也可以為關聯計數結果設定別名，從而在同一個關聯上執行多個計數：

```php
use Illuminate\Database\Eloquent\Builder;

$posts = Post::withCount([
    'comments',
    'comments as pending_comments_count' => function (Builder $query) {
        $query->where('approved', false);
    },
])->get();

echo $posts[0]->comments_count;
echo $posts[0]->pending_comments_count;
```


<a name="deferred-count-loading"></a>
#### 延遲載入計數

使用 `loadCount` 方法，您可以在父層 Model 已經被檢索後再載入關聯計數：

```php
$book = Book::first();

$book->loadCount('genres');
```

如果您需要對計數查詢設定額外的查詢條件限制，可以傳遞一個以您想要計數的關聯為鍵的陣列。陣列的值應為接收查詢建構器實例的閉包：

```php
$book->loadCount(['reviews' => function (Builder $query) {
    $query->where('rating', 5);
}])
```


<a name="relationship-counting-and-custom-select-statements"></a>
#### 關聯計數與自訂 Select 語句

如果您將 `withCount` 與 `select` 語句結合使用，請確保在 `select` 方法之後呼叫 `withCount`：

```php
$posts = Post::select(['title', 'body'])
    ->withCount('comments')
    ->get();
```


<a name="other-aggregate-functions"></a>
### 其他聚合函式

除了 `withCount` 方法之外，Eloquent 還提供了 `withMin`、`withMax`、`withAvg`、`withSum` 和 `withExists` 方法。這些方法會在產生的 Model 上放置一個 `{relation}_{function}_{column}` 屬性：

```php
use App\Models\Post;

$posts = Post::withSum('comments', 'votes')->get();

foreach ($posts as $post) {
    echo $post->comments_sum_votes;
}
```

如果您希望使用其他名稱存取聚合函式的結果，可以指定您自己的別名：

```php
$posts = Post::withSum('comments as total_comments', 'votes')->get();

foreach ($posts as $post) {
    echo $post->total_comments;
}
```

就像 `loadCount` 方法一樣，這些方法也有延遲版本。可以在已經檢索出來的 Eloquent Model 上執行這些額外的聚合操作：

```php
$post = Post::first();

$post->loadSum('comments', 'votes');
```

如果您將這些聚合方法與 `select` 語句結合使用，請確保在 `select` 方法之後呼叫聚合方法：

```php
$posts = Post::select(['title', 'body'])
    ->withExists('comments')
    ->get();
```


<a name="counting-related-models-on-morph-to-relationships"></a>
### 計算 Morph To 關聯上的關聯 Model 數量

如果您想要預先載入「morph to」關聯，以及由該關聯返回的各個實體的關聯 Model 計數，可以將 `with` 方法與 `morphTo` 關聯的 `morphWithCount` 方法結合使用。

在這個範例中，我們假設 `Photo` 和 `Post` Model 可以建立 `ActivityFeed` Model。我們將假設 `ActivityFeed` Model 定義了一個名為 `parentable` 的「morph to」關聯，它允許我們檢索給定 `ActivityFeed` 實例的父層 `Photo` 或 `Post` Model。此外，假設 `Photo` Model「擁有許多」`Tag` Model，而 `Post` Model「擁有許多」`Comment` Model。

現在，想像我們想要檢索 `ActivityFeed` 實例，並為每個 `ActivityFeed` 實例預先載入 `parentable` 父層 Model。此外，我們還想檢索與每個父層相片關聯的標籤數量，以及與每個父層文章關聯的留言數量：

```php
use Illuminate\Database\Eloquent\Relations\MorphTo;

$activities = ActivityFeed::with([
    'parentable' => function (MorphTo $morphTo) {
        $morphTo->morphWithCount([
            Photo::class => ['tags'],
            Post::class => ['comments'],
        ]);
    }])->get();
```


<a name="morph-to-deferred-count-loading"></a>
#### 延遲載入計數

假設我們已經檢索了一組 `ActivityFeed` Model，現在想要載入與動態摘要相關聯的各種 `parentable` Model 的巢狀關聯計數。您可以使用 `loadMorphCount` 方法來達成此目的：

```php
$activities = ActivityFeed::with('parentable')->get();

$activities->loadMorphCount('parentable', [
    Photo::class => ['tags'],
    Post::class => ['comments'],
]);
```

<a name="eager-loading"></a>
## 預先載入

當將 Eloquent 關聯當作屬性存取時，關聯的 Model 會被「延遲載入 (lazy loaded)」。這意味著關聯資料直到你首次存取該屬性時才會真正載入。然而，Eloquent 可以在你查詢父層 Model 的同時「預先載入 (eager load)」關聯。預先載入能緩解「N + 1」查詢問題。為了說明 N + 1 查詢問題，請考慮一個「belongs to」`Author` Model 的 `Book` Model：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class Book extends Model
{
    /**
     * Get the author that wrote the book.
     */
    public function author(): BelongsTo
    {
        return $this->belongsTo(Author::class);
    }
}
```

現在，讓我們取得所有書籍及其作者：

```php
use App\Models\Book;

$books = Book::all();

foreach ($books as $book) {
    echo $book->author->name;
}
```

此迴圈將執行一次查詢以取得資料庫資料表中的所有書籍，接著針對每本書執行另一次查詢以取得該書的作者。因此，如果我們有 25 本書，上述程式碼將會執行 26 次查詢：1 次用於取得原始書籍，以及 25 次額外查詢用於取得每本書的作者。

慶幸的是，我們可以使用預先載入將此操作減少至僅需兩次查詢。在建構查詢時，你可以使用 `with` 方法指定應該預先載入哪些關聯：

```php
$books = Book::with('author')->get();

foreach ($books as $book) {
    echo $book->author->name;
}
```

在此操作中，只會執行兩次查詢——一次查詢用於取得所有書籍，另一次查詢用於取得所有書籍的所有作者：

```sql
select * from books

select * from authors where id in (1, 2, 3, 4, 5, ...)
```


<a name="eager-loading-multiple-relationships"></a>
#### 預先載入多個關聯

有時你可能需要預先載入數個不同的關聯。為此，只需將關聯陣列傳遞給 `with` 方法即可：

```php
$books = Book::with(['author', 'publisher'])->get();
```


<a name="nested-eager-loading"></a>
#### 巢狀預先載入

若要預先載入關聯的關聯，你可以使用「點 (dot)」語法。例如，讓我們預先載入書籍的所有作者以及作者的所有個人聯絡人：

```php
$books = Book::with('author.contacts')->get();
```

或者，你也可以透過向 `with` 方法提供巢狀陣列來指定巢狀預先載入關聯，這在預先載入多個巢狀關聯時會非常方便：

```php
$books = Book::with([
    'author' => [
        'contacts',
        'publisher',
    ],
])->get();
```


<a name="nested-eager-loading-morphto-relationships"></a>
#### 巢狀預先載入 `morphTo` 關聯

如果你想預先載入 `morphTo` 關聯，以及該關聯可能返回的各種實體上的巢狀關聯，你可以將 `with` 方法與 `morphTo` 關聯的 `morphWith` 方法結合使用。為了幫助說明此方法，讓我們考慮以下 Model：

```php
<?php

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\MorphTo;

class ActivityFeed extends Model
{
    /**
     * Get the parent of the activity feed record.
     */
    public function parentable(): MorphTo
    {
        return $this->morphTo();
    }
}
```

在此範例中，假設 `Event`、`Photo` 和 `Post` Model 都可以建立 `ActivityFeed` Model。此外，假設 `Event` Model 屬於 `Calendar` Model，`Photo` Model 與 `Tag` Model 相關聯，而 `Post` Model 則屬於 `Author` Model。

使用這些 Model 定義與關聯，我們可以取得 `ActivityFeed` Model 實例，並預先載入所有 `parentable` Model 及其各自的巢狀關聯：

```php
use Illuminate\Database\Eloquent\Relations\MorphTo;

$activities = ActivityFeed::query()
    ->with(['parentable' => function (MorphTo $morphTo) {
        $morphTo->morphWith([
            Event::class => ['calendar'],
            Photo::class => ['tags'],
            Post::class => ['author'],
        ]);
    }])->get();
```


<a name="eager-loading-specific-columns"></a>
#### 預先載入特定欄位

在取得關聯時，你可能並不總是需要每個欄位。因此，Eloquent 允許你指定想要取得該關聯的哪些欄位：

```php
$books = Book::with('author:id,name,book_id')->get();
```

> [!WARNING]
> 使用此功能時，你應該始終在要取得的欄位清單中包含 `id` 欄位以及任何相關的外部鍵欄位。


<a name="eager-loading-by-default"></a>
#### 預設預先載入

有時你可能希望在取得 Model 時始終載入某些關聯。為此，你可以在 Model 上定義 `$with` 屬性：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class Book extends Model
{
    /**
     * The relationships that should always be loaded.
     *
     * @var array
     */
    protected $with = ['author'];

    /**
     * Get the author that wrote the book.
     */
    public function author(): BelongsTo
    {
        return $this->belongsTo(Author::class);
    }

    /**
     * Get the genre of the book.
     */
    public function genre(): BelongsTo
    {
        return $this->belongsTo(Genre::class);
    }
}
```

如果你想在單一查詢中從 `$with` 屬性中移除某個項目，可以使用 `without` 方法：

```php
$books = Book::without('author')->get();
```

如果你想在單一查詢中覆寫 `$with` 屬性中的所有項目，可以使用 `withOnly` 方法：

```php
$books = Book::withOnly('genre')->get();
```

<a name="constraining-eager-loads"></a>
### 限制預先載入

有時你可能會希望預先載入某個關聯，但同時也想為該預先載入查詢指定額外的查詢條件。你可以透過傳遞一個關聯陣列給 `with` 方法來達成此目的，其中陣列的鍵 (Key) 為關聯名稱，而陣列的值則是一個為預先載入查詢增加額外條件約束的閉包 (Closure)：

```php
use App\Models\User;

$users = User::with(['posts' => function ($query) {
    $query->where('title', 'like', '%code%');
}])->get();
```

在這個範例中，Eloquent 只會預先載入文章的 `title` 欄位包含單字 `code` 的文章。你也可以呼叫其他[查詢建構器 (Query Builder)](/docs/{{version}}/queries) 方法來進一步自訂預先載入操作：

```php
$users = User::with(['posts' => function ($query) {
    $query->orderBy('created_at', 'desc');
}])->get();
```


<a name="constraining-eager-loading-of-morph-to-relationships"></a>
#### 限制 `morphTo` 關聯的預先載入

如果你正在預先載入一個 `morphTo` 關聯，Eloquent 將會執行多個查詢來取得每種類型的關聯 Model。你可以使用 `MorphTo` 關聯的 `constrain` 方法為這些查詢中的每一個增加額外的條件約束：

```php
use Illuminate\Database\Eloquent\Relations\MorphTo;

$comments = Comment::with(['commentable' => function (MorphTo $morphTo) {
    $morphTo->constrain([
        Post::class => function ($query) {
            $query->whereNull('hidden_at');
        },
        Video::class => function ($query) {
            $query->where('type', 'educational');
        },
    ]);
}])->get();
```

在這個範例中，Eloquent 只會預先載入未被隱藏的文章，以及 `type` 值為 "educational" 的影片。


<a name="constraining-eager-loads-with-relationship-existence"></a>
#### 結合關聯存在性限制預先載入

有時你可能會發現自己需要檢查關聯是否存在，同時又需要基於相同條件載入該關聯。例如，你可能只想取得擁有符合給定查詢條件的子 `Post` Model 的 `User` Model，同時也預先載入這些符合條件的文章。你可以使用 `withWhereHas` 方法來達成此目的：

```php
use App\Models\User;

$users = User::withWhereHas('posts', function ($query) {
    $query->where('featured', true);
})->get();
```


<a name="lazy-eager-loading"></a>
### 延遲預先載入

有時你可能需要在取得父層 Model 之後才預先載入關聯。例如，當你需要動態決定是否載入關聯 Model 時，這會很有用：

```php
use App\Models\Book;

$books = Book::all();

if ($condition) {
    $books->load('author', 'publisher');
}
```

如果你需要在延遲預先載入查詢上設定額外的查詢條件約束，可以傳遞一個以欲載入的關聯為鍵 (Key) 的陣列。陣列的值應該是接收查詢實例的閉包 (Closure)：

```php
$author->load(['books' => function ($query) {
    $query->orderBy('published_date', 'asc');
}]);
```

若只想在關聯尚未被載入時才載入該關聯，請使用 `loadMissing` 方法：

```php
$book->loadMissing('author');
```


<a name="nested-lazy-eager-loading-morphto"></a>
#### 巢狀延遲預先載入與 `morphTo`

如果你想要預先載入 `morphTo` 關聯，以及該關聯可能返回的各種實體上的巢狀關聯，可以使用 `loadMorph` 方法。

該方法接受 `morphTo` 關聯的名稱作為其第一個引數，並接受一個 Model / 關聯配對的陣列作為其第二個引數。為了說明這個方法，讓我們參考以下 Model：

```php
<?php

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\MorphTo;

class ActivityFeed extends Model
{
    /**
     * Get the parent of the activity feed record.
     */
    public function parentable(): MorphTo
    {
        return $this->morphTo();
    }
}
```

在這個範例中，我們假設 `Event`、`Photo` 與 `Post` Model 可以建立 `ActivityFeed` Model。此外，我們假設 `Event` Model 屬於 `Calendar` Model，`Photo` Model 與 `Tag` Model 關聯，而 `Post` Model 則屬於 `Author` Model。

使用這些 Model 定義與關聯，我們可以取得 `ActivityFeed` Model 實例，並預先載入所有 `parentable` Model 及其各自的巢狀關聯：

```php
$activities = ActivityFeed::with('parentable')
    ->get()
    ->loadMorph('parentable', [
        Event::class => ['calendar'],
        Photo::class => ['tags'],
        Post::class => ['author'],
    ]);
```


<a name="automatic-eager-loading"></a>
### 自動預先載入

在許多情況下，Laravel 可以自動預先載入你所存取的關聯。若要啟用自動預先載入，你應該在應用程式的 `AppServiceProvider` 中的 `boot` 方法內呼叫 `Model::automaticallyEagerLoadRelationships` 方法：

```php
use Illuminate\Database\Eloquent\Model;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Model::automaticallyEagerLoadRelationships();
}
```

啟用此功能後，Laravel 將嘗試自動載入你存取但先前尚未載入的任何關聯。例如，考慮以下情境：

```php
use App\Models\User;

$users = User::all();

foreach ($users as $user) {
    foreach ($user->posts as $post) {
        foreach ($post->comments as $comment) {
            echo $comment->content;
        }
    }
}
```

通常，上述程式碼會針對每個使用者執行一次查詢以取得其文章，並針對每篇文章執行一次查詢以取得其留言。然而，當 `automaticallyEagerLoadRelationships` 功能啟用時，當你嘗試存取已取得使用者中的任何一位使用者的文章時，Laravel 將自動為使用者集合中的所有使用者[延遲預先載入](#lazy-eager-loading)文章。同樣地，當你嘗試存取任何已取得文章的留言時，將會為原本取得的所有文章延遲預先載入所有留言。

如果你不想全域啟用自動預先載入，你仍然可以透過在單一 Eloquent Collection 實例上呼叫 `withRelationshipAutoloading` 方法來為其啟用此功能：

```php
$users = User::where('vip', true)->get();

return $users->withRelationshipAutoloading();
```

<a name="preventing-lazy-loading"></a>
### 防止延遲載入

如前所述，預先載入關聯通常可以為您的應用程式帶來顯著的效能優勢。因此，如果您願意，可以指示 Laravel 始終防止關聯的延遲載入。為此，您可以調用基礎 Eloquent Model 類別所提供的 `preventLazyLoading` 方法。通常，您應該在應用程式的 `AppServiceProvider` 類別的 `boot` 方法中調用此方法。

`preventLazyLoading` 方法接受一個可選的布林引數，用於指定是否應防止延遲載入。例如，您可能希望僅在非正式環境中停用延遲載入，這樣即使正式環境程式碼中意外出現延遲載入的關聯，您的正式環境也能繼續正常運行：

```php
use Illuminate\Database\Eloquent\Model;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Model::preventLazyLoading(! $this->app->isProduction());
}
```

防止延遲載入後，當您的應用程式嘗試延遲載入任何 Eloquent 關聯時，Eloquent 將拋出 `Illuminate\Database\LazyLoadingViolationException` 例外。

您可以使用 `handleLazyLoadingViolationsUsing` 方法自訂違反延遲載入時的行為。例如，使用此方法，您可以指示僅記錄違反延遲載入的日誌，而不是以例外中斷應用程式的執行：

```php
Model::handleLazyLoadingViolationUsing(function (Model $model, string $relation) {
    $class = $model::class;

    info("Attempted to lazy load [{$relation}] on model [{$class}].");
});
```

<a name="inserting-and-updating-related-models"></a>
## 新增與更新關聯 Model


<a name="the-save-method"></a>
### `save` 方法

Eloquent 提供了便捷的方法來將新 Model 加入關聯中。例如，也許您需要為一篇文章新增一則新的留言。您可以使用關聯的 `save` 方法來插入該留言，而不是手動在 `Comment` Model 上設定 `post_id` 屬性：

```php
use App\Models\Comment;
use App\Models\Post;

$comment = new Comment(['message' => 'A new comment.']);

$post = Post::find(1);

$post->comments()->save($comment);
```

請注意，我們沒有將 `comments` 關聯作為動態屬性來存取。相反地，我們呼叫了 `comments` 方法來取得該關聯的實例。`save` 方法會自動為新的 `Comment` Model 加入適當的 `post_id` 值。

如果您需要儲存多個關聯 Model，可以使用 `saveMany` 方法：

```php
$post = Post::find(1);

$post->comments()->saveMany([
    new Comment(['message' => 'A new comment.']),
    new Comment(['message' => 'Another new comment.']),
]);
```

`save` 與 `saveMany` 方法會將給定的 Model 實例寫入資料庫，但不會將新保存的 Model 新增至已載入父層 Model 的任何記憶體內部關聯中。如果您打算在使用 `save` 或 `saveMany` 方法後存取該關聯，建議使用 `refresh` 方法來重新載入該 Model 及其關聯：

```php
$post->comments()->save($comment);

$post->refresh();

// All comments, including the newly saved comment...
$post->comments;
```


<a name="the-push-method"></a>
#### 遞迴儲存 Model 與關聯

如果您想要 `save` 您的 Model 及其所有相關聯的 Model，可以使用 `push` 方法。在此範例中，`Post` Model 將會被儲存，其留言以及留言的作者也都會一併儲存：

```php
$post = Post::find(1);

$post->comments[0]->message = 'Message';
$post->comments[0]->author->name = 'Author Name';

$post->push();
```

`pushQuietly` 方法可用於儲存 Model 及其相關聯的 Model，且不會觸發任何事件：

```php
$post->pushQuietly();
```


<a name="the-create-method"></a>
### `create` 方法

除了 `save` 和 `saveMany` 方法外，您還可以使用 `create` 方法，它接受一個屬性陣列，建立一個 Model 並將其插入資料庫中。`save` 與 `create` 之間的差異在於 `save` 接受一個完整的 Eloquent Model 實例，而 `create` 接受純 PHP `array`。新建立的 Model 將會由 `create` 方法回傳：

```php
use App\Models\Post;

$post = Post::find(1);

$comment = $post->comments()->create([
    'message' => 'A new comment.',
]);
```

您可以使用 `createMany` 方法來建立多個關聯 Model：

```php
$post = Post::find(1);

$post->comments()->createMany([
    ['message' => 'A new comment.'],
    ['message' => 'Another new comment.'],
]);
```

`createQuietly` 和 `createManyQuietly` 方法可用於建立 Model 且不分派任何事件：

```php
$user = User::find(1);

$user->posts()->createQuietly([
    'title' => 'Post title.',
]);

$user->posts()->createManyQuietly([
    ['title' => 'First post.'],
    ['title' => 'Second post.'],
]);
```

您還可以使用 `findOrNew`、`firstOrNew`、`firstOrCreate` 和 `updateOrCreate` 方法來[在關聯上建立與更新 Model](/docs/{{version}}/eloquent#upserts)。

> [!NOTE]
> 在使用 `create` 方法之前，請務必參閱[大量賦值](/docs/{{version}}/eloquent#mass-assignment)文件。


<a name="updating-belongs-to-relationships"></a>
### Belongs To 關聯

如果您想將子 Model 指派給新的父 Model，可以使用 `associate` 方法。在此範例中，`User` Model 定義了對 `Account` Model 的 `belongsTo` 關聯。此 `associate` 方法將會在子 Model 上設定外鍵：

```php
use App\Models\Account;

$account = Account::find(10);

$user->account()->associate($account);

$user->save();
```

若要從子 Model 中移除父 Model，可以使用 `dissociate` 方法。此方法會將該關聯的外鍵設定為 `null`：

```php
$user->account()->dissociate();

$user->save();
```

<a name="updating-many-to-many-relationships"></a>
### 多對多關聯

<a name="attaching-detaching"></a>
#### 附加 / 分離

Eloquent 還提供了多種方法，讓操作多對多關聯更加方便。舉例來說，假設一個使用者可以擁有多個角色，而一個角色也可以被多個使用者擁有。你可以使用 `attach` 方法透過在關聯的中間資料表中插入一筆紀錄，來為使用者附加一個角色：

```php
use App\Models\User;

$user = User::find(1);

$user->roles()->attach($roleId);
```

在將關聯附加到 Model 時，你還可以傳入要插入中間資料表的額外資料陣列：

```php
$user->roles()->attach($roleId, ['expires' => $expires]);
```

有時候可能需要從使用者身上移除某個角色。若要移除多對多關聯紀錄，請使用 `detach` 方法。`detach` 方法會從中間資料表中刪除對應的紀錄；不過，兩個 Model 都仍會保留在資料庫中：

```php
// Detach a single role from the user...
$user->roles()->detach($roleId);

// Detach all roles from the user...
$user->roles()->detach();
```

為了方便起見，`attach` 與 `detach` 也接受 ID 陣列作為輸入：

```php
$user = User::find(1);

$user->roles()->detach([1, 2, 3]);

$user->roles()->attach([
    1 => ['expires' => $expires],
    2 => ['expires' => $expires],
]);
```

<a name="syncing-associations"></a>
#### 同步關聯

你也可以使用 `sync` 方法來建立多對多關聯。`sync` 方法接受一個要放置在中間資料表中的 ID 陣列。任何不在給定陣列中的 ID 都會從中間資料表中移除。因此，在此操作完成後，中間資料表中將只存在給定陣列中的 ID：

```php
$user->roles()->sync([1, 2, 3]);
```

你也可以隨 ID 一併傳遞額外的中間資料表數值：

```php
$user->roles()->sync([1 => ['expires' => true], 2, 3]);
```

如果你希望為每個同步的 Model ID 插入相同的中間資料表數值，可以使用 `syncWithPivotValues` 方法：

```php
$user->roles()->syncWithPivotValues([1, 2, 3], ['active' => true]);
```

如果你不想分離給定陣列中缺少的現有 ID，可以使用 `syncWithoutDetaching` 方法：

```php
$user->roles()->syncWithoutDetaching([1, 2, 3]);
```

<a name="toggling-associations"></a>
#### 切換關聯狀態

多對多關聯還提供了一個 `toggle` 方法，用來「切換」給定關聯 Model ID 的附加狀態。如果給定的 ID 目前已附加，它將會被分離；同樣地，如果它目前已被分離，則會被附加：

```php
$user->roles()->toggle([1, 2, 3]);
```

你也可以隨 ID 一併傳遞額外的中間資料表數值：

```php
$user->roles()->toggle([
    1 => ['expires' => true],
    2 => ['expires' => true],
]);
```

<a name="transactional-pivot-operations"></a>
#### 交易式 Pivot 操作

上述討論的每個 pivot 操作也都有一個 `OrFail` 變體（`attachOrFail`、`detachOrFail`、`syncOrFail`、`syncWithoutDetachingOrFail` 與 `toggleOrFail`），它們會將操作包裝在資料庫交易中，以便在拋出例外時自動還原所有變更：

```php
$user->roles()->attachOrFail([1, 2, 3]);

$user->roles()->syncOrFail([1, 2, 3]);
```

<a name="updating-a-record-on-the-intermediate-table"></a>
#### 更新中間資料表上的紀錄

如果你需要更新關聯中間資料表中的現有資料列，可以使用 `updateExistingPivot` 方法。此方法接受中間紀錄的外鍵以及要更新的屬性陣列：

```php
$user = User::find(1);

$user->roles()->updateExistingPivot($roleId, [
    'active' => false,
]);
```

<a name="touching-parent-timestamps"></a>
## 觸發更新父層時間戳記

當 Model 定義了對另一個 Model 的 `belongsTo` 或 `belongsToMany` 關聯時（例如屬於 `Post` 的 `Comment`），在子 Model 更新時同時更新父層的時間戳記有時會很有幫助。

例如，當 `Comment` Model 被更新時，您可能希望自動「觸碰 (touch)」所屬 `Post` 的 `updated_at` 時間戳記，將其設定為目前的日期與時間。為達成此目的，您可以在子 Model 上使用 `Touches` Attribute，其中包含在子 Model 更新時應該更新其 `updated_at` 時間戳記的關聯名稱：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Attributes\Touches;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

#[Touches(['post'])]
class Comment extends Model
{
    /**
     * Get the post that the comment belongs to.
     */
    public function post(): BelongsTo
    {
        return $this->belongsTo(Post::class);
    }
}
```

> [!WARNING]
> 父層 Model 的時間戳記僅會在子 Model 使用 Eloquent 的 `save` 方法更新時才會被更新。