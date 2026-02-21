# Eloquent：關聯

- [簡介](#introduction)
- [定義關聯](#defining-relationships)
    - [一對一 / Has One](#one-to-one)
    - [一對多 / Has Many](#one-to-many)
    - [一對多 (反向) / Belongs To](#one-to-many-inverse)
    - [Has One of Many](#has-one-of-many)
    - [Has One Through](#has-one-through)
    - [Has Many Through](#has-many-through)
- [範圍關聯](#scoped-relationships)
- [多對多關聯](#many-to-many)
    - [取得中間表欄位](#retrieving-intermediate-table-columns)
    - [透過中間表欄位篩選查詢](#filtering-queries-via-intermediate-table-columns)
    - [透過中間表欄位排序查詢](#ordering-queries-via-intermediate-table-columns)
    - [定義自定義中間表模型](#defining-custom-intermediate-table-models)
- [多型關聯](#polymorphic-relationships)
    - [一對一](#one-to-one-polymorphic-relations)
    - [一對多](#one-to-many-polymorphic-relations)
    - [One of Many](#one-of-many-polymorphic-relations)
    - [多對多](#many-to-many-polymorphic-relations)
    - [自定義多型類型](#custom-polymorphic-types)
- [動態關聯](#dynamic-relationships)
- [查詢關聯](#querying-relations)
    - [關聯方法 vs. 動態屬性](#relationship-methods-vs-dynamic-properties)
    - [查詢關聯是否存在](#querying-relationship-existence)
    - [查詢關聯是否不存在](#querying-relationship-absence)
    - [查詢 Morph To 關聯](#querying-morph-to-relationships)
- [聚合關聯模型](#aggregating-related-models)
    - [計算關聯模型數量](#counting-related-models)
    - [其他聚合函式](#other-aggregate-functions)
    - [計算 Morph To 關聯的模型數量](#counting-related-models-on-morph-to-relationships)
- [積極載入](#eager-loading)
    - [限制積極載入](#constraining-eager-loads)
    - [延遲積極載入](#lazy-eager-loading)
    - [自動積極載入](#automatic-eager-loading)
    - [防止懶惰載入](#preventing-lazy-loading)
- [新增與更新關聯模型](#inserting-and-updating-related-models)
    - [`save` 方法](#the-save-method)
    - [`create` 方法](#the-create-method)
    - [Belongs To 關聯](#updating-belongs-to-relationships)
    - [多對多關聯](#updating-many-to-many-relationships)
- [更新父模型時間戳記](#touching-parent-timestamps)

<a name="introduction"></a>
## 簡介

資料庫資料表通常彼此相關。例如，一篇部落格文章可能有許多留言，或者一筆訂單可能與下單的使用者相關。Eloquent 讓管理和處理這些關聯變得簡單，並支援多種常見的關聯：

<div class="content-list" markdown="1">

- [一對一](#one-to-one)
- [一對多](#one-to-many)
- [多對多](#many-to-many)
- [Has One Through](#has-one-through)
- [Has Many Through](#has-many-through)
- [一對一 (多型)](#one-to-one-polymorphic-relations)
- [一對多 (多型)](#one-to-many-polymorphic-relations)
- [多對多 (多型)](#many-to-many-polymorphic-relations)

</div>

<a name="defining-relationships"></a>
## 定義關聯

Eloquent 關聯定義在你的 Eloquent 模型類別中。由於關聯也可以作為強大的 [查詢產生器 (query builders)](/docs/{{version}}/queries)，因此將關聯定義為方法提供了強大的方法鏈結與查詢能力。例如，我們可以在這個 `posts` 關聯上鏈結額外的查詢約束：

```php
$user->posts()->where('active', 1)->get();
```

但在深入探討如何使用關聯之前，讓我們學習如何定義 Eloquent 支援的每種關聯類型。


<a name="one-to-one"></a>
### 一對一 / Has One

一對一關聯是一種非常基礎的資料庫關聯類型。例如，一個 `User` 模型可能與一個 `Phone` 模型關聯。要定義此關聯，我們將在 `User` 模型中放置一個 `phone` 方法。`phone` 方法應該呼叫 `hasOne` 方法並回傳其結果。透過模型的 `Illuminate\Database\Eloquent\Model` 基底類別，你的模型可以使用 `hasOne` 方法：

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

`hasOne` 方法的第一個參數是關聯模型的類別名稱。定義好關聯後，我們可以使用 Eloquent 的動態屬性取得關聯記錄。動態屬性讓你可以像存取模型上定義的屬性一樣存取關聯方法：

```php
$phone = User::find(1)->phone;
```

Eloquent 會根據父模型名稱決定關聯的外鍵。在這種情況下，自動假設 `Phone` 模型有一個 `user_id` 外鍵。如果你想覆寫此慣例，可以將第二個參數傳遞給 `hasOne` 方法：

```php
return $this->hasOne(Phone::class, 'foreign_key');
```

此外，Eloquent 假設外鍵的值應該與父模型的主鍵欄位匹配。換句話說，Eloquent 會在 `Phone` 記錄的 `user_id` 欄位中尋找使用者 `id` 欄位的值。如果你希望關聯使用 `id` 以外的主鍵值或模型中的 `$primaryKey` 屬性，可以將第三個參數傳遞給 `hasOne` 方法：

```php
return $this->hasOne(Phone::class, 'foreign_key', 'local_key');
```


<a name="one-to-one-defining-the-inverse-of-the-relationship"></a>
#### 定義反向關聯

現在，我們可以從 `User` 模型存取 `Phone` 模型。接下來，讓我們在 `Phone` 模型上定義一個關聯，讓我們能存取擁有該電話的使用者。我們可以使用 `belongsTo` 方法來定義 `hasOne` 關聯的反向關聯：

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

呼叫 `user` 方法時，Eloquent 會嘗試尋找一個 `id` 與 `Phone` 模型上的 `user_id` 欄位相匹配的 `User` 模型。

Eloquent 透過檢查關聯方法的名稱，並在方法名稱後綴加上 `_id` 來決定外鍵名稱。因此，在這種情況下，Eloquent 假設 `Phone` 模型有一個 `user_id` 欄位。但是，如果 `Phone` 模型上的外鍵不是 `user_id`，你可以將自定義鍵名作為第二個參數傳遞給 `belongsTo` 方法：

```php
/**
 * Get the user that owns the phone.
 */
public function user(): BelongsTo
{
    return $this->belongsTo(User::class, 'foreign_key');
}
```

如果父模型不使用 `id` 作為其主鍵，或者你希望使用不同的欄位來尋找關聯模型，你可以將第三個參數傳遞給 `belongsTo` 方法，指定父資料表的自定義鍵：

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

一對多關聯用於定義單個模型是核心（父模型），且擁有一或多個子模型的關聯。例如，一篇部落格文章可能有無限數量的留言。與所有其他 Eloquent 關聯一樣，一對多關聯是透過在你的 Eloquent 模型中定義一個方法來定義的：

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

請記住，Eloquent 會自動為 `Comment` 模型決定合適的外鍵欄位。依照慣例，Eloquent 會採用父模型的「蛇形命名 (snake case)」名稱並加上 `_id` 後綴。因此，在此範例中，Eloquent 會假設 `Comment` 模型上的外鍵欄位是 `post_id`。

定義好關聯方法後，我們可以透過存取 `comments` 屬性來取得關聯留言的 [集合 (collection)](/docs/{{version}}/eloquent-collections)。請記住，由於 Eloquent 提供「動態關聯屬性」，我們可以像存取模型上定義的屬性一樣存取關聯方法：

```php
use App\Models\Post;

$comments = Post::find(1)->comments;

foreach ($comments as $comment) {
    // ...
}
```

由於所有關聯也可以作為查詢產生器，你可以透過呼叫 `comments` 方法並繼續在查詢中鏈結條件，為關聯查詢增加進一步的限制：

```php
$comment = Post::find(1)->comments()
    ->where('title', 'foo')
    ->first();
```

與 `hasOne` 方法一樣，你也可以透過向 `hasMany` 方法傳遞額外參數來覆寫外鍵與本地鍵：

```php
return $this->hasMany(Comment::class, 'foreign_key');

return $this->hasMany(Comment::class, 'foreign_key', 'local_key');
```


<a name="automatically-hydrating-parent-models-on-children"></a>
#### 自動在子模型中填充父模型

即使利用 Eloquent 的積極載入 (eager loading)，如果你在迴圈走訪子模型時嘗試從子模型存取父模型，仍可能會出現 「N + 1」查詢問題：

```php
$posts = Post::with('comments')->get();

foreach ($posts as $post) {
    foreach ($post->comments as $comment) {
        echo $comment->post->title;
    }
}
```

在上述範例中，引入了 「N + 1」查詢問題，因為即使已為每個 `Post` 模型積極載入留言，Eloquent 也不會自動在每個子 `Comment` 模型上填充 (hydrate) 父級 `Post`。

如果你希望 Eloquent 自動將父模型填充到其子模型中，可以在定義 `hasMany` 關聯時呼叫 `chaperone` 方法：

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

或者，如果你想在執行時選擇加入自動父模型填充，可以在積極載入關聯時呼叫 `chaperone` 方法：

```php
use App\Models\Post;

$posts = Post::with([
    'comments' => fn ($comments) => $comments->chaperone(),
])->get();
```

<a name="one-to-many-inverse"></a>
### 一對多 (反向) / Belongs To

既然我們已經可以存取文章的所有留言，接著讓我們定義一個關聯，允許留言存取其所屬的文章。要定義 `hasMany` 關聯的反向關聯，請在子模型上定義一個呼叫 `belongsTo` 方法的關聯方法：

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

關聯定義完成後，我們可以透過存取 `post` 「動態關聯屬性」來取得留言的父級文章：

```php
use App\Models\Comment;

$comment = Comment::find(1);

return $comment->post->title;
```

在上述範例中，Eloquent 會嘗試尋找 `id` 與 `Comment` 模型上 `post_id` 欄位匹配的 `Post` 模型。

Eloquent 透過檢查關聯方法的名稱，並在方法名稱後加上 `_` 及父模型主鍵欄位的名稱來決定預設的外鍵名稱。因此在目前的範例中，Eloquent 會假設 `Post` 模型在 `comments` 資料表上的外鍵是 `post_id`。

然而，如果您的關聯外鍵不符合這些慣例，您可以將自定義外鍵名稱作為 `belongsTo` 方法的第二個參數傳入：

```php
/**
 * Get the post that owns the comment.
 */
public function post(): BelongsTo
{
    return $this->belongsTo(Post::class, 'foreign_key');
}
```

如果您的父模型不使用 `id` 作為主鍵，或者您希望使用不同的欄位來尋找關聯模型，您可以向 `belongsTo` 方法傳遞第三個參數，指定父資料表的自定義鍵：

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
#### 預設模型

`belongsTo`、`hasOne`、`hasOneThrough` 與 `morphOne` 關聯允許您定義一個預設模型，當給定的關聯為 `null` 時，將會回傳該預設模型。這種模式通常被稱為 [Null Object 模式](https://en.wikipedia.org/wiki/Null_Object_pattern)，可以幫助您減少程式碼中的條件檢查。在以下範例中，如果 `Post` 模型沒有關聯任何使用者，則 `user` 關聯將回傳一個空的 `App\Models\User` 模型：

```php
/**
 * Get the author of the post.
 */
public function user(): BelongsTo
{
    return $this->belongsTo(User::class)->withDefault();
}
```

若要使用屬性填充預設模型，您可以將陣列或閉包傳遞給 `withDefault` 方法：

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

查詢「屬於」關聯的子模型時，您可以手動建構 `where` 子句來取得對應的 Eloquent 模型：

```php
use App\Models\Post;

$posts = Post::where('user_id', $user->id)->get();
```

然而，您可能會發現使用 `whereBelongsTo` 方法更方便，它會自動為給定的模型判斷正確的關聯和外鍵：

```php
$posts = Post::whereBelongsTo($user)->get();
```

您也可以將 [Collection](/docs/{{version}}/eloquent-collections) 實例傳遞給 `whereBelongsTo` 方法。這樣做時，Laravel 會取得屬於該 Collection 內任何父模型的子模型：

```php
$users = User::where('vip', true)->get();

$posts = Post::whereBelongsTo($users)->get();
```

預設情況下，Laravel 會根據模型的類別名稱來判斷與給定模型相關聯的關聯；不過，您可以透過將關聯名稱作為第二個參數提供給 `whereBelongsTo` 方法來手動指定關聯名稱：

```php
$posts = Post::whereBelongsTo($user, 'author')->get();
```

<a name="has-one-of-many"></a>
### Has One of Many

有時一個模型可能具有多個關聯模型，但你希望能輕鬆取得該關聯中「最新」或「最舊」的相關模型。例如，一個 `User` 模型可能關聯多個 `Order` 模型，但你希望能定義一個便捷的方式來與使用者最近一次下的訂單進行互動。你可以使用 `hasOne` 關聯類型結合 `ofMany` 方法來完成：

```php
/**
 * Get the user's most recent order.
 */
public function latestOrder(): HasOne
{
    return $this->hasOne(Order::class)->latestOfMany();
}
```

同樣地，你可以定義一個方法來取得關聯中「最舊」或第一個相關模型：

```php
/**
 * Get the user's oldest order.
 */
public function oldestOrder(): HasOne
{
    return $this->hasOne(Order::class)->oldestOfMany();
}
```

預設情況下，`latestOfMany` 和 `oldestOfMany` 方法會根據模型的初級鍵（Primary Key）來取得最新或最舊的相關模型，而該初級鍵必須是可排序的。然而，有時你可能希望使用不同的排序標準從較大的關聯中取得單一模型。

例如，使用 `ofMany` 方法，你可以取得使用者金額最高的訂單。`ofMany` 方法接受可排序的欄位作為第一個參數，以及在查詢關聯模型時要套用的聚合函式（`min` 或 `max`）作為第二個參數：

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
> 由於 PostgreSQL 不支援對 UUID 欄位執行 `MAX` 函式，目前無法在 PostgreSQL UUID 欄位上結合使用 one-of-many 關聯。


<a name="converting-many-relationships-to-has-one-relationships"></a>
#### Converting "Many" Relationships to Has One Relationships

通常，當使用 `latestOfMany`、`oldestOfMany` 或 `ofMany` 方法來取得單一模型時，你已經為同一個模型定義了一個「has many」關聯。為了方便起見，Laravel 允許你透過在關聯上呼叫 `one` 方法，輕鬆地將此關聯轉換為「has one」關聯：

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
#### Advanced Has One of Many Relationships

你也可以構建更進階的「has one of many」關聯。例如，一個 `Product` 模型可能有多個關聯的 `Price` 模型，即使在發布新定價後，這些模型仍保留在系統中。此外，產品的新定價數據可能可以透過 `published_at` 欄位提前發布，以便在未來的某個日期生效。

總結來說，我們需要取得最新發布且發布日期不在未來的定價。此外，如果兩個價格具有相同的發布日期，我們將優先選擇 ID 較大的價格。為了實現這一點，我們必須向 `ofMany` 方法傳遞一個陣列，其中包含用於決定最新價格的可排序欄位。此外，還需提供一個閉包作為 `ofMany` 方法的第二個參數。該閉包將負責為關聯查詢添加額外的發布日期限制：

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

「has-one-through」關聯定義了與另一個模型的一對一關聯。然而，此關聯表示宣告模型可以透過「經過」第三個模型，與另一個模型的一個實例進行匹配。

例如，在一個修車廠應用程式中，每個 `Mechanic` 模型可能關聯一個 `Car` 模型，而每個 `Car` 模型可能關聯一個 `Owner` 模型。雖然技師（mechanic）與車主（owner）在資料庫中沒有直接關聯，但技師可以透過 `Car` 模型存取車主。讓我們看看定義此關聯所需的資料表：

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

既然我們已經檢查了該關聯的資料表結構，接著讓我們在 `Mechanic` 模型上定義這個關聯：

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

傳遞給 `hasOneThrough` 方法的第一個參數是我們希望存取的最終模型名稱，而第二個參數則是中間模型的名稱。

或者，如果涉及關聯的所有模型都已經定義了相關的關聯，你可以透過呼叫 `through` 方法並提供這些關聯的名稱，流暢地定義「has-one-through」關聯。例如，如果 `Mechanic` 模型有一個 `cars` 關聯，且 `Car` 模型有一個 `owner` 關聯，你可以像這樣定義連結技師與車主的「has-one-through」關聯：

```php
// String based syntax...
return $this->through('cars')->has('owner');

// Dynamic syntax...
return $this->throughCars()->hasOwner();
```


<a name="has-one-through-key-conventions"></a>
#### Key Conventions

執行關聯查詢時，將使用典型的 Eloquent 外鍵慣例。如果你希望自定義關聯的鍵，可以將它們作為第三個和第四個參數傳遞給 `hasOneThrough` 方法。第三個參數是中間模型上的外鍵名稱。第四個參數是最終模型上的外鍵名稱。第五個參數是本地鍵（local key），而第六個參數是中間模型的本地鍵：

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

或者，如前所述，如果涉及關聯的所有模型都已經定義了相關的關聯，你可以透過呼叫 `through` 方法並提供這些關聯的名稱，流暢地定義「has-one-through」關聯。這種方法的優點是能重複使用已經定義在現有關聯上的鍵慣例：

```php
// String based syntax...
return $this->through('cars')->has('owner');

// Dynamic syntax...
return $this->throughCars()->hasOwner();
```

<a name="has-many-through"></a>
### Has Many Through

「has-many-through」關聯提供了一種透過中間關聯來存取遠端關聯的便利方式。例如，假設我們正在建立一個像 [Laravel Cloud](https://cloud.laravel.com) 這樣的部署平台。一個 `Application` 模型可能會透過中間的 `Environment` 模型存取許多 `Deployment` 模型。在這個範例中，您可以輕鬆地取得特定應用程式的所有部署。讓我們來看看定義此關聯所需的資料表：

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

既然我們已經檢查了該關聯的資料表結構，接著讓我們在 `Application` 模型上定義這個關聯：

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

傳遞給 `hasManyThrough` 方法的第一個參數是我們希望存取的最終模型名稱，而第二個參數是中間模型的名稱。

或者，如果參與關聯的所有模型都已經定義了相關的關聯，您可以透過呼叫 `through` 方法並提供這些關聯的名稱，來流暢地定義「has-many-through」關聯。例如，如果 `Application` 模型具有 `environments` 關聯，且 `Environment` 模型具有 `deployments` 關聯，您可以像這樣定義連接應用程式和部署的「has-many-through」關聯：

```php
// String based syntax...
return $this->through('environments')->has('deployments');

// Dynamic syntax...
return $this->throughEnvironments()->hasDeployments();
```

雖然 `Deployment` 模型的資料表不包含 `application_id` 欄位，但 `hasManyThrough` 關聯提供了透過 `$application->deployments` 存取應用程式部署的方法。為了取得這些模型，Eloquent 會檢查中間模型 `Environment` 資料表中的 `application_id` 欄位。在找到相關的環境 ID 後，它們會被用於查詢 `Deployment` 模型的資料表。

<a name="has-many-through-key-conventions"></a>
#### 鍵的慣例 (Key Conventions)

在執行關聯查詢時，會使用典型的 Eloquent 外鍵慣例。如果您想要自定義關聯的鍵，可以將它們作為第三個和第四個參數傳遞給 `hasManyThrough` 方法。第三個參數是中間模型上的外鍵名稱。第四個參數是最終模型上的外鍵名稱。第五個參數是區域鍵 (Local Key)，而第六個參數是中間模型的區域鍵：

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

或者，如前所述，如果參與關聯的所有模型都已經定義了相關的關聯，您可以透過呼叫 `through` 方法並提供這些關聯的名稱，來流暢地定義「has-many-through」關聯。這種方法的優點是可以重複使用現有關聯中已定義的鍵慣例：

```php
// String based syntax...
return $this->through('environments')->has('deployments');

// Dynamic syntax...
return $this->throughEnvironments()->hasDeployments();
```

<a name="scoped-relationships"></a>
### 範圍關聯

通常會在模型中增加額外的方法來限制關聯。例如，您可以在 `User` 模型中增加一個 `featuredPosts` 方法，透過額外的 `where` 約束來限制更廣泛的 `posts` 關聯：

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

然而，如果您嘗試透過 `featuredPosts` 方法建立模型，其 `featured` 屬性不會被設置為 `true`。如果您希望透過關聯方法建立模型，並指定應添加到透過該關聯建立的所有模型中的屬性，您可以在建立關聯查詢時使用 `withAttributes` 方法：

```php
/**
 * Get the user's featured posts.
 */
public function featuredPosts(): HasMany
{
    return $this->posts()->withAttributes(['featured' => true]);
}
```

`withAttributes` 方法會使用給定的屬性在查詢中加入 `where` 條件，並且也會將給定的屬性添加到透過該關聯方法建立的任何模型中：

```php
$post = $user->featuredPosts()->create(['title' => 'Featured Post']);

$post->featured; // true
```

若要指示 `withAttributes` 方法不要在查詢中加入 `where` 條件，您可以將 `asConditions` 參數設置為 `false`：

```php
return $this->posts()->withAttributes(['featured' => true], asConditions: false);
```

<a name="many-to-many"></a>
## 多對多關聯

多對多關聯比 `hasOne` 及 `hasMany` 關聯稍微複雜一些。多對多關聯的一個例子是：一位使用者擁有多個角色，而這些角色也被應用程式中的其他使用者共享。例如，一位使用者可能被分配了「作者」和「編輯」的角色；然而，這些角色也可以被分配給其他使用者。因此，一位使用者擁有多個角色，而一個角色擁有多位使用者。

<a name="many-to-many-table-structure"></a>
#### 資料表結構

為了定義這種關聯，需要三個資料庫表：`users`、`roles` 和 `role_user`。`role_user` 資料表是根據相關模型名稱的字母順序命名的，並包含 `user_id` 和 `role_id` 欄位。此資料表被用作連結使用者和角色的中間表。

請記住，由於一個角色可以屬於多個使用者，我們不能簡單地在 `roles` 資料表中放置一個 `user_id` 欄位。這意味著一個角色只能屬於單一使用者。為了支援將角色分配給多個使用者，需要 `role_user` 資料表。我們可以將關聯的資料表結構總結如下：

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
#### 模型結構

多對多關聯是透過編寫一個回傳 `belongsToMany` 方法結果的方法來定義的。`belongsToMany` 方法由所有應用程式的 Eloquent 模型所使用的 `Illuminate\Database\Eloquent\Model` 基底類別提供。例如，讓我們在 `User` 模型中定義一個 `roles` 方法。傳遞給此方法的第一個引數是相關模型類別的名稱：

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

定義好關聯後，您可以使用 `roles` 動態關聯屬性來存取使用者的角色：

```php
use App\Models\User;

$user = User::find(1);

foreach ($user->roles as $role) {
    // ...
}
```

由於所有關聯也都充當查詢產生器，您可以透過呼叫 `roles` 方法並繼續鏈結查詢條件，為關聯查詢增加進一步的限制：

```php
$roles = User::find(1)->roles()->orderBy('name')->get();
```

為了確定關聯中間表的名稱，Eloquent 會按照字母順序組合兩個相關的模型名稱。但是，您可以隨意覆蓋此慣例。您可以透過向 `belongsToMany` 方法傳遞第二個引數來做到這一點：

```php
return $this->belongsToMany(Role::class, 'role_user');
```

除了自定義中間表的名稱外，您還可以透過向 `belongsToMany` 方法傳遞額外的引數來自定義資料表上的鍵名。第三個引數是您定義關聯的模型的外鍵名稱，而第四個引數是您要連接的模型的外鍵名稱：

```php
return $this->belongsToMany(Role::class, 'role_user', 'user_id', 'role_id');
```

<a name="many-to-many-defining-the-inverse-of-the-relationship"></a>
#### 定義反向關聯

要定義多對多關聯的「反向」，您應該在相關模型上定義一個方法，該方法也回傳 `belongsToMany` 方法的結果。為了完成我們的使用者/角色範例，讓我們在 `Role` 模型上定義 `users` 方法：

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

如您所見，除了引用 `App\Models\User` 模型外，該關聯的定義與其 `User` 模型對應部分的定義完全相同。由於我們重複使用了 `belongsToMany` 方法，因此在定義多對多關聯的「反向」時，所有常用的資料表和鍵名自定義選項都是可用的。

<a name="retrieving-intermediate-table-columns"></a>
### 取得中間表欄位

如您所知，處理多對多關聯需要中間表的存在。Eloquent 提供了一些非常有用的方式來與此資料表互動。例如，假設我們的 `User` 模型與許多 `Role` 模型相關聯。在存取此關聯後，我們可以使用模型上的 `pivot` 屬性來存取中間表：

```php
use App\Models\User;

$user = User::find(1);

foreach ($user->roles as $role) {
    echo $role->pivot->created_at;
}
```

請注意，我們取得的每個 `Role` 模型都會自動被分配一個 `pivot` 屬性。此屬性包含一個代表中間表的模型。

預設情況下，`pivot` 模型上只會存在模型鍵。如果您的中間表包含額外的屬性，您必須在定義關聯時指定它們：

```php
return $this->belongsToMany(Role::class)->withPivot('active', 'created_by');
```

如果您希望中間表具有由 Eloquent 自動維護的 `created_at` 和 `updated_at` 時間戳記，請在定義關聯時呼叫 `withTimestamps` 方法：

```php
return $this->belongsToMany(Role::class)->withTimestamps();
```

> [!WARNING]
> 使用 Eloquent 自動維護時間戳記的中間表，必須同時具有 `created_at` 和 `updated_at` 時間戳記欄位。

<a name="customizing-the-pivot-attribute-name"></a>
#### 自定義 `pivot` 屬性名稱

如前所述，屬性可以透過 `pivot` 屬性在模型上存取中間表的資料。但是，您可以隨意自定義此屬性的名稱，以更好地反映其在應用程式中的用途。

例如，如果您的應用程式包含可以訂閱 Podcast 的使用者，那麼使用者和 Podcast 之間可能存在多對多關聯。在這種情況下，您可能希望將中間表屬性重新命名為 `subscription` 而不是 `pivot`。這可以在定義關聯時使用 `as` 方法來完成：

```php
return $this->belongsToMany(Podcast::class)
    ->as('subscription')
    ->withTimestamps();
```

一旦指定了自定義的中間表屬性，您就可以使用該自定義名稱存取中間表資料：

```php
$users = User::with('podcasts')->get();

foreach ($users->flatMap->podcasts as $podcast) {
    echo $podcast->subscription->created_at;
}
```

<a name="filtering-queries-via-intermediate-table-columns"></a>
### 透過中間表欄位篩選查詢

在定義關聯時，您也可以使用 `wherePivot`、`wherePivotIn`、`wherePivotNotIn`、`wherePivotBetween`、`wherePivotNotBetween`、`wherePivotNull` 以及 `wherePivotNotNull` 方法來篩選 `belongsToMany` 關聯查詢所回傳的結果：

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

`wherePivot` 會在查詢中加入 where 子句限制，但在透過定義的關聯建立新模型時，並不會加入指定的值。如果您需要同時查詢以及使用特定的 Pivot 值來建立關聯，您可以使用 `withPivotValue` 方法：

```php
return $this->belongsToMany(Role::class)
    ->withPivotValue('approved', 1);
```


<a name="ordering-queries-via-intermediate-table-columns"></a>
### 透過中間表欄位排序查詢

您可以使用 `orderByPivot` 和 `orderByPivotDesc` 方法對 `belongsToMany` 關聯查詢回傳的結果進行排序。在以下範例中，我們將為使用者取得所有最新的徽章：

```php
return $this->belongsToMany(Badge::class)
    ->where('rank', 'gold')
    ->orderByPivotDesc('created_at');
```


<a name="defining-custom-intermediate-table-models"></a>
### 定義自定義中間表模型

如果您想定義一個自定義模型來代表多對多關聯的中間表，您可以在定義關聯時呼叫 `using` 方法。自定義 Pivot 模型讓您有機會在 Pivot 模型上定義額外的行為，例如方法和型別轉換 (Casts)。

自定義的多對多 Pivot 模型應該擴充 `Illuminate\Database\Eloquent\Relations\Pivot` 類別，而自定義的多型多對多 Pivot 模型應該擴充 `Illuminate\Database\Eloquent\Relations\MorphPivot` 類別。例如，我們可以定義一個使用自定義 `RoleUser` Pivot 模型的 `Role` 模型：

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

在定義 `RoleUser` 模型時，您應該擴充 `Illuminate\Database\Eloquent\Relations\Pivot` 類別：

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
> Pivot 模型不能使用 `SoftDeletes` trait。如果您需要軟刪除 Pivot 紀錄，請考慮將您的 Pivot 模型轉換為實際的 Eloquent 模型。


<a name="custom-pivot-models-and-incrementing-ids"></a>
#### 自定義 Pivot 模型與遞增 ID

如果您定義了一個使用自定義 Pivot 模型的多對多關聯，且該 Pivot 模型具有自動遞增的主鍵，則應確保您的自定義 Pivot 模型類別定義了一個設定為 `true` 的 `incrementing` 屬性。

```php
/**
 * Indicates if the IDs are auto-incrementing.
 *
 * @var bool
 */
public $incrementing = true;
```

<a name="polymorphic-relationships"></a>
## 多型關聯

多型關聯允許子模型透過單個關聯隸屬於多種模型。例如，假設您正在建立一個允許使用者分享部落格文章與影片的應用程式。在這樣的應用程式中，`Comment` 模型可能同時隸屬於 `Post` 與 `Video` 模型。


<a name="one-to-one-polymorphic-relations"></a>
### 一對一


<a name="one-to-one-polymorphic-table-structure"></a>
#### 資料表結構

一對一多型關聯類似於典型的一對一關聯；然而，子模型可以透過單個關聯隸屬於多種模型。例如，部落格 `Post` 與 `User` 可能會共享一個到 `Image` 模型的多型關聯。使用一對一多型關聯讓您擁有一張包含唯一圖片的資料表，並可將其與文章和使用者關聯。首先，讓我們看看資料表結構：

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
    imageable_id - integer
    imageable_type - string
```

請注意 `images` 表中的 `imageable_id` 與 `imageable_type` 欄位。`imageable_id` 欄位將包含文章或使用者的 ID 值，而 `imageable_type` 欄位將包含父模型的類別名稱。`imageable_type` 欄位由 Eloquent 用來在存取 `imageable` 關聯時判斷要回傳哪種「類型」的父模型。在此案例中，該欄位將包含 `App\Models\Post` 或 `App\Models\User`。


<a name="one-to-one-polymorphic-model-structure"></a>
#### 模型結構

接著，讓我們看看建立此關聯所需的模型定義：

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

一旦定義了資料庫表與模型，您就可以透過模型存取這些關聯。例如，要取得文章的圖片，我們可以存取 `image` 動態關聯屬性：

```php
use App\Models\Post;

$post = Post::find(1);

$image = $post->image;
```

您可以透過存取執行 `morphTo` 呼叫的方法名稱來取得多型模型的父級。在此案例中，即是 `Image` 模型上的 `imageable` 方法。因此，我們將以動態關聯屬性的方式存取該方法：

```php
use App\Models\Image;

$image = Image::find(1);

$imageable = $image->imageable;
```

`Image` 模型上的 `imageable` 關聯將回傳 `Post` 或 `User` 實例，取決於哪種類型的模型擁有該圖片。


<a name="morph-one-to-one-key-conventions"></a>
#### 鍵值慣例

如有必要，您可以指定多型子模型所使用的 「id」 與 「type」 欄位名稱。如果您這樣做，請確保始終將關聯名稱作為第一個參數傳遞給 `morphTo` 方法。通常，此值應與方法名稱相符，因此您可以使用 PHP 的 `__FUNCTION__` 常數：

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

一對多多型關聯類似於典型的一對多關聯；然而，子模型可以使用單一關聯屬於多種類型的模型。例如，想像您的應用程式使用者可以對文章 (posts) 和影片 (videos) 進行「留言 (comment)」。透過使用多型關聯，您可以使用單一的 `comments` 資料表來包含文章與影片的留言。首先，讓我們看看建立此關聯所需的資料表結構：

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
    commentable_id - integer
    commentable_type - string
```


<a name="one-to-many-polymorphic-model-structure"></a>
#### 模型結構

接下來，讓我們檢查建立此關聯所需的模型定義：

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

一旦定義了資料表與模型，您就可以透過模型的動態關聯屬性來存取關聯。例如，要存取一篇文章的所有留言，我們可以使用 `comments` 動態屬性：

```php
use App\Models\Post;

$post = Post::find(1);

foreach ($post->comments as $comment) {
    // ...
}
```

您也可以透過存取呼叫 `morphTo` 的方法名稱來取得多型子模型的父模型。在此範例中，即是 `Comment` 模型上的 `commentable` 方法。因此，我們將該方法作為動態關聯屬性存取，以存取留言的父模型：

```php
use App\Models\Comment;

$comment = Comment::find(1);

$commentable = $comment->commentable;
```

`Comment` 模型上的 `commentable` 關聯將回傳 `Post` 或 `Video` 實例，具體取決於哪種類型的模型是留言的父模型。


<a name="polymorphic-automatically-hydrating-parent-models-on-children"></a>
#### 自動在子模型中填充父模型

即使利用了 Eloquent 的積極載入，如果您在迴圈遍歷子模型時嘗試從子模型存取父模型，仍可能出現「N + 1」查詢問題：

```php
$posts = Post::with('comments')->get();

foreach ($posts as $post) {
    foreach ($post->comments as $comment) {
        echo $comment->commentable->title;
    }
}
```

在上述範例中，由於即使為每個 `Post` 模型積極載入了留言，Eloquent 也不會自動在每個子 `Comment` 模型中填充父 `Post` 模型，因此引入了「N + 1」查詢問題。

如果您希望 Eloquent 自動將父模型填充到其子模型中，您可以在定義 `morphMany` 關聯時呼叫 `chaperone` 方法：

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

或者，如果您想在執行時選擇加入自動父模型填充，可以在積極載入關聯時呼叫 `chaperone` 方法：

```php
use App\Models\Post;

$posts = Post::with([
    'comments' => fn ($comments) => $comments->chaperone(),
])->get();
```


<a name="one-of-many-polymorphic-relations"></a>
### One of Many

有時一個模型可能有許多相關聯的模型，但您希望輕鬆取得該關聯中「最新」或「最舊」的相關聯模型。例如，一個 `User` 模型可能與許多 `Image` 模型相關聯，但您想定義一種方便的方式來與使用者上傳的最新圖片進行互動。您可以使用 `morphOne` 關聯類型結合 `ofMany` 方法來達成此目的：

```php
/**
 * Get the user's most recent image.
 */
public function latestImage(): MorphOne
{
    return $this->morphOne(Image::class, 'imageable')->latestOfMany();
}
```

同樣地，您可以定義一個方法來取得關聯中「最舊」或第一個相關聯的模型：

```php
/**
 * Get the user's oldest image.
 */
public function oldestImage(): MorphOne
{
    return $this->morphOne(Image::class, 'imageable')->oldestOfMany();
}
```

預設情況下，`latestOfMany` 和 `oldestOfMany` 方法將根據模型的初級鍵（必須是可排序的）取得最新或最舊的相關聯模型。然而，有時您可能希望使用不同的排序標準從較大的關聯中取得單一模型。

例如，使用 `ofMany` 方法，您可以取得使用者獲得最多「讚」的圖片。`ofMany` 方法接受可排序的欄位作為其第一個參數，以及在查詢相關聯模型時要套用的聚合函式 (`min` 或 `max`)：

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
> 可以建構更進階的「one of many」關聯。如需更多資訊，請參閱 [has one of many 文件](#advanced-has-one-of-many-relationships)。

<a name="many-to-many-polymorphic-relations"></a>
### 多對多


<a name="many-to-many-polymorphic-table-structure"></a>
#### 資料表結構

多對多多型關聯比「morph one」和「morph many」關聯稍微複雜一些。例如，`Post` 模型和 `Video` 模型可以共享指向 `Tag` 模型的多型關聯。在這種情況下使用多對多多型關聯，可以讓您的應用程式擁有一個包含唯一標籤的單一資料表，這些標籤可以與貼文或影片相關聯。首先，讓我們看看建立此關聯所需的資料表結構：

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
    taggable_id - integer
    taggable_type - string
```

> [!NOTE]
> 在深入研究多型多對多關聯之前，閱讀有關典型[多對多關聯](#many-to-many)的說明文件可能會對您有所幫助。


<a name="many-to-many-polymorphic-model-structure"></a>
#### 模型結構

接下來，我們準備在模型上定義關聯。`Post` 和 `Video` 模型都將包含一個 `tags` 方法，該方法呼叫基礎 Eloquent 模型類別提供的 `morphToMany` 方法。

`morphToMany` 方法接受關聯模型的名稱以及「關聯名稱」。根據我們分配給中間表的名稱及其包含的鍵，我們將此關聯稱為「taggable」：

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

接下來，在 `Tag` 模型上，您應該為其每個可能的父模型定義一個方法。因此，在此範例中，我們將定義一個 `posts` 方法和一個 `videos` 方法。這兩個方法都應該回傳 `morphedByMany` 方法的結果。

`morphedByMany` 方法接受關聯模型的名稱以及「關聯名稱」。根據我們分配給中間表的名稱及其包含的鍵，我們將此關聯稱為「taggable」：

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

一旦定義好資料表和模型，您就可以透過模型存取關聯。例如，要存取貼文的所有標籤，您可以使用 `tags` 動態關聯屬性：

```php
use App\Models\Post;

$post = Post::find(1);

foreach ($post->tags as $tag) {
    // ...
}
```

您可以透過存取執行 `morphedByMany` 呼叫的方法名稱，從多型子模型中取得多型關聯的父模型。在這種情況下，即 `Tag` 模型上的 `posts` 或 `videos` 方法：

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
### 自定義多型類型

預設情況下，Laravel 將使用完整路徑類別名稱來儲存關聯模型的「類型」。例如，給定上述的一對多關聯範例，其中 `Comment` 模型可能屬於 `Post` 或 `Video` 模型，預設的 `commentable_type` 將分別為 `App\Models\Post` 或 `App\Models\Video`。然而，您可能希望將這些值與應用程式的內部結構解耦。

例如，我們可以使用簡單的字串如 `post` 和 `video` 來代替模型名稱作為「類型」。藉由這樣做，即使模型被重新命名，我們資料庫中的多型「類型」欄位值仍將保持有效：

```php
use Illuminate\Database\Eloquent\Relations\Relation;

Relation::enforceMorphMap([
    'post' => 'App\Models\Post',
    'video' => 'App\Models\Video',
]);
```

您可以將 `enforceMorphMap` 方法呼叫放在 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中，或者如果您願意，也可以建立一個單獨的服務提供者。

您可以在執行時使用模型的 `getMorphClass` 方法來確定給定模型的多型別名。相反地，您可以使用 `Relation::getMorphedModel` 方法來確定與多型別名關聯的完整路徑類別名稱：

```php
use Illuminate\Database\Eloquent\Relations\Relation;

$alias = $post->getMorphClass();

$class = Relation::getMorphedModel($alias);
```

> [!WARNING]
> 當為現有的應用程式新增「多型對照表 (Morph Map)」時，資料庫中每個仍包含完整路徑類別的動態 `*_type` 欄位值都需要轉換為其對應的「對照表」名稱。


<a name="dynamic-relationships"></a>
### 動態關聯

您可以使用 `resolveRelationUsing` 方法在執行時定義 Eloquent 模型之間的關聯。雖然通常不建議用於一般的應用程式開發，但在開發 Laravel 套件時，這偶爾會很有用。

`resolveRelationUsing` 方法的第一個參數接受所需的關聯名稱。傳遞給該方法的第二個參數應該是一個閉包，該閉包接受模型實例並回傳一個有效的 Eloquent 關聯定義。通常，您應該在[服務提供者](/docs/{{version}}/providers)的 boot 方法中設定動態關聯：

```php
use App\Models\Order;
use App\Models\Customer;

Order::resolveRelationUsing('customer', function (Order $orderModel) {
    return $orderModel->belongsTo(Customer::class, 'customer_id');
});
```

> [!WARNING]
> 在定義動態關聯時，請務必為 Eloquent 關聯方法提供明確的鍵名參數。

<a name="querying-relations"></a>
## 查詢關聯

由於所有 Eloquent 關聯都是透過方法定義的，您可以呼叫這些方法來取得關聯的實例，而無需實際執行查詢來載入關聯模型。此外，所有類型的 Eloquent 關聯也都充當 [查詢語句建構器 (Query Builder)](/docs/{{version}}/queries)，讓您可以在最終針對資料庫執行 SQL 查詢之前，繼續在關聯查詢上串接約束條件。

例如，假設一個部落格應用程式中，`User` 模型擁有許多關聯的 `Post` 模型：

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

您可以查詢 `posts` 關聯並對該關聯新增額外的約束條件，如下所示：

```php
use App\Models\User;

$user = User::find(1);

$user->posts()->where('active', 1)->get();
```

您可以在關聯上使用任何 Laravel [查詢語句建構器](/docs/{{version}}/queries) 的方法，因此請務必探索查詢語句建構器文件，以了解所有可用的方法。


<a name="chaining-orwhere-clauses-after-relationships"></a>
#### 在關聯後串接 `orWhere` 子句

如上述範例所示，您在查詢關聯時可以自由地新增額外的約束條件。但是，在關聯上串接 `orWhere` 子句時請務必小心，因為 `orWhere` 子句在邏輯上會被歸類在與關聯約束相同的層級：

```php
$user->posts()
    ->where('active', 1)
    ->orWhere('votes', '>=', 100)
    ->get();
```

上述範例將產生以下 SQL。如您所見，`or` 子句指示查詢回傳 _任何_ 票數大於 100 的貼文。該查詢不再受限於特定的使用者：

```sql
select *
from posts
where user_id = ? and active = 1 or votes >= 100
```

在大多數情況下，您應該使用 [邏輯分組](/docs/{{version}}/queries#logical-grouping) 將條件檢查放在括號中進行分組：

```php
use Illuminate\Database\Eloquent\Builder;

$user->posts()
    ->where(function (Builder $query) {
        return $query->where('active', 1)
            ->orWhere('votes', '>=', 100);
    })
    ->get();
```

上述範例將產生以下 SQL。請注意，邏輯分組已正確地將約束條件分組，且查詢仍然受限於特定的使用者：

```sql
select *
from posts
where user_id = ? and (active = 1 or votes >= 100)
```


<a name="relationship-methods-vs-dynamic-properties"></a>
### 關聯方法 vs. 動態屬性

如果您不需要對 Eloquent 關聯查詢新增額外的約束條件，您可以像存取屬性一樣存取該關聯。例如，繼續使用我們的 `User` 和 `Post` 範例模型，我們可以像這樣存取使用者的所有貼文：

```php
use App\Models\User;

$user = User::find(1);

foreach ($user->posts as $post) {
    // ...
}
```

動態關聯屬性執行的是「懶惰載入 (Lazy Loading)」，這意味著只有在您實際存取它們時，它們才會載入關聯資料。因此，開發者通常會使用 [積極載入 (Eager Loading)](#eager-loading) 來預先載入他們知道在載入模型後會被存取的關聯。積極載入大幅減少了載入模型關聯時必須執行的 SQL 查詢數量。


<a name="querying-relationship-existence"></a>
### 查詢關聯是否存在

在取得模型紀錄時，您可能希望根據關聯的存在與否來限制結果。例如，假設您想取得所有至少有一條留言的部落格貼文。為此，您可以將關聯名稱傳遞給 `has` 和 `orHas` 方法：

```php
use App\Models\Post;

// Retrieve all posts that have at least one comment...
$posts = Post::has('comments')->get();
```

您也可以指定運算子和數量值來進一步自定義查詢：

```php
// Retrieve all posts that have three or more comments...
$posts = Post::has('comments', '>=', 3)->get();
```

可以使用「點」標記法建構嵌套的 `has` 語句。例如，您可以取得所有至少有一條留言且該留言至少有一個圖片的貼文：

```php
// Retrieve posts that have at least one comment with images...
$posts = Post::has('comments.images')->get();
```

如果您需要更強大的功能，可以使用 `whereHas` 和 `orWhereHas` 方法在 `has` 查詢上定義額外的查詢約束條件，例如檢查留言的內容：

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
> Eloquent 目前不支援跨資料庫查詢關聯的存在性。關聯必須存在於同一個資料庫中。


<a name="many-to-many-relationship-existence-queries"></a>
#### 多對多關聯存在性查詢

`whereAttachedTo` 方法可用於查詢與某個模型或模型集合具有多對多附加關係的模型：

```php
$users = User::whereAttachedTo($role)->get();
```

您也可以向 `whereAttachedTo` 方法提供一個 [集合 (Collection)](/docs/{{version}}/eloquent-collections) 實例。這樣做時，Laravel 將取得附加到集合中任何一個模型的模型：

```php
$tags = Tag::whereLike('name', '%laravel%')->get();

$posts = Post::whereAttachedTo($tags)->get();
```


<a name="inline-relationship-existence-queries"></a>
#### 行內關聯存在性查詢

如果您想透過附加在關聯查詢上的單個簡單 where 條件來查詢關聯的存在性，您可能會發現使用 `whereRelation`、`orWhereRelation`、`whereMorphRelation` 和 `orWhereMorphRelation` 方法更為方便。例如，我們可以查詢所有具有未核准留言的貼文：

```php
use App\Models\Post;

$posts = Post::whereRelation('comments', 'is_approved', false)->get();
```

當然，就像呼叫查詢語句建構器的 `where` 方法一樣，您也可以指定一個運算子：

```php
$posts = Post::whereRelation(
    'comments', 'created_at', '>=', now()->minus(hours: 1)
)->get();
```


<a name="querying-relationship-absence"></a>
### 查詢關聯是否不存在

在取得模型紀錄時，您可能希望根據關聯的不存在來限制結果。例如，假設您想取得所有 **沒有** 任何留言的部落格貼文。為此，您可以將關聯名稱傳遞給 `doesntHave` 和 `orDoesntHave` 方法：

```php
use App\Models\Post;

$posts = Post::doesntHave('comments')->get();
```

如果您需要更強大的功能，可以使用 `whereDoesntHave` 和 `orWhereDoesntHave` 方法在您的 `doesntHave` 查詢中新增額外的查詢約束，例如檢查留言的內容：

```php
use Illuminate\Database\Eloquent\Builder;

$posts = Post::whereDoesntHave('comments', function (Builder $query) {
    $query->where('content', 'like', 'code%');
})->get();
```

您可以使用「點」標記法對嵌套關聯執行查詢。例如，以下查詢將取得所有沒有留言的貼文，以及雖有留言但所有留言都不是來自被停權使用者的貼文：

```php
use Illuminate\Database\Eloquent\Builder;

$posts = Post::whereDoesntHave('comments.author', function (Builder $query) {
    $query->where('banned', 1);
})->get();
```

<a name="querying-morph-to-relationships"></a>
### 查詢 Morph To 關聯

若要查詢 「morph to」 關聯是否存在，你可以使用 `whereHasMorph` 與 `whereDoesntHaveMorph` 方法。這些方法接受關聯名稱作為其第一個參數。接著，這些方法接受你希望包含在查詢中的相關模型名稱。最後，你可以提供一個用來自定義關聯查詢的閉包：

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

你偶爾可能需要根據相關多型模型的「類型」來增加查詢限制。傳遞給 `whereHasMorph` 方法的閉包可以接收 `$type` 值作為其第二個參數。此參數讓你可以檢查正在建立的查詢「類型」：

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

有時你可能想查詢「morph to」關聯之父模型的子模型。你可以使用 `whereMorphedTo` 與 `whereNotMorphedTo` 方法來達成此目的，這些方法會自動為指定的模型判斷正確的多型類型映射。這些方法接受 `morphTo` 關聯名稱作為其第一個參數，並將相關父模型作為其第二個參數：

```php
$comments = Comment::whereMorphedTo('commentable', $post)
    ->orWhereMorphedTo('commentable', $video)
    ->get();
```


<a name="querying-all-morph-to-related-models"></a>
#### 查詢所有相關模型

除了傳遞可能的多型模型陣列之外，你也可以提供 `*` 作為萬用字元值。這將指示 Laravel 從資料庫中取得所有可能的多型類型。為了執行此操作，Laravel 將會執行一個額外的查詢：

```php
use Illuminate\Database\Eloquent\Builder;

$comments = Comment::whereHasMorph('commentable', '*', function (Builder $query) {
    $query->where('title', 'like', 'foo%');
})->get();
```

<a name="aggregating-related-models"></a>
## 聚合關聯模型

<a name="counting-related-models"></a>
### 計算關聯模型數量

有時您可能想計算某個關聯的模型數量，但不想實際載入這些模型。為此，您可以使用 `withCount` 方法。`withCount` 方法會在產生的模型上放置一個 `{relation}_count` 屬性：

```php
use App\Models\Post;

$posts = Post::withCount('comments')->get();

foreach ($posts as $post) {
    echo $post->comments_count;
}
```

藉由傳遞一個陣列給 `withCount` 方法，您可以同時為多個關聯新增「計數」，並為查詢新增額外的約束：

```php
use Illuminate\Database\Eloquent\Builder;

$posts = Post::withCount(['votes', 'comments' => function (Builder $query) {
    $query->where('content', 'like', 'code%');
}])->get();

echo $posts[0]->votes_count;
echo $posts[0]->comments_count;
```

您也可以為關聯計數結果設定別名，從而允許在同一個關聯上進行多次計數：

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
#### 延遲計數載入

使用 `loadCount` 方法，您可以在父模型已經取得後，再載入關聯計數：

```php
$book = Book::first();

$book->loadCount('genres');
```

如果您需要在計數查詢中設定額外的查詢約束，可以傳遞一個以您想計數的關聯為鍵 (Key) 的陣列。陣列的值應為接收查詢產生器實例的閉包：

```php
$book->loadCount(['reviews' => function (Builder $query) {
    $query->where('rating', 5);
}])
```

<a name="relationship-counting-and-custom-select-statements"></a>
#### 關聯計數與自定義 Select 語句

如果您將 `withCount` 與 `select` 語句結合使用，請確保在 `select` 方法之後呼叫 `withCount`：

```php
$posts = Post::select(['title', 'body'])
    ->withCount('comments')
    ->get();
```

<a name="other-aggregate-functions"></a>
### 其他聚合函式

除了 `withCount` 方法外，Eloquent 還提供了 `withMin`、`withMax`、`withAvg`、`withSum` 與 `withExists` 方法。這些方法會在您產生的模型上放置一個 `{relation}_{function}_{column}` 屬性：

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

就像 `loadCount` 方法一樣，這些方法也有延遲版本。這些額外的聚合操作可以在已經取得的 Eloquent 模型上執行：

```php
$post = Post::first();

$post->loadSum('comments', 'votes');
```

如果您將這些聚合方法與 `select` 語句結合使用，請確保在 `select` 方法之後呼叫這些聚合方法：

```php
$posts = Post::select(['title', 'body'])
    ->withExists('comments')
    ->get();
```

<a name="counting-related-models-on-morph-to-relationships"></a>
### 計算 Morph To 關聯的模型數量

如果您想積極載入一個 「morph to」 關聯，以及該關聯可能回傳的各種實體之相關模型計數，您可以結合使用 `with` 方法與 `morphTo` 關聯的 `morphWithCount` 方法。

在此範例中，假設 `Photo` 與 `Post` 模型可能會建立 `ActivityFeed` 模型。我們假設 `ActivityFeed` 模型定義了一個名為 `parentable` 的「morph to」關聯，讓我們可以取得給定 `ActivityFeed` 實例的父級 `Photo` 或 `Post` 模型。此外，假設 `Photo` 模型「has many」`Tag` 模型，而 `Post` 模型「has many」`Comment` 模型。

現在，想像我們要取得 `ActivityFeed` 實例，並為每個 `ActivityFeed` 實例積極載入 `parentable` 父模型。此外，我們還想取得與每個父級相片相關聯的標籤數量，以及與每個父級文章相關聯的留言數量：

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
#### 延遲計數載入

假設我們已經取得了一組 `ActivityFeed` 模型，現在我們想為與動態摘要相關聯的各種 `parentable` 模型載入嵌套關聯計數。您可以使用 `loadMorphCount` 方法來達成此目的：

```php
$activities = ActivityFeed::with('parentable')->get();

$activities->loadMorphCount('parentable', [
    Photo::class => ['tags'],
    Post::class => ['comments'],
]);
```

<a name="eager-loading"></a>
## 積極載入

當以屬性方式存取 Eloquent 關聯時，關聯模型是「懶惰載入」的。這意味著直到您首次存取該屬性之前，關聯資料實際上並未被載入。然而，Eloquent 可以在您查詢父模型時「積極載入」關聯。積極載入減輕了「N + 1」查詢問題。為了說明 N + 1 查詢問題，請考慮一個「屬於」`Author` 模型的 `Book` 模型：

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

此迴圈將執行一次查詢以取得資料庫表中的所有書籍，然後針對每本書執行另一次查詢以取得該書的作者。因此，如果我們有 25 本書，上述程式碼將執行 26 次查詢：一次查詢原始書籍，以及 25 次額外查詢以取得每本書的作者。

慶幸地，我們可以使用積極載入將此操作減少到僅兩次查詢。在建立查詢時，您可以使用 `with` 方法指定哪些關聯應該被積極載入：

```php
$books = Book::with('author')->get();

foreach ($books as $book) {
    echo $book->author->name;
}
```

對於此操作，將僅執行兩次查詢 - 一次查詢取得所有書籍，一次查詢取得所有書籍的所有作者：

```sql
select * from books

select * from authors where id in (1, 2, 3, 4, 5, ...)
```


<a name="eager-loading-multiple-relationships"></a>
#### 積極載入多個關聯

有時您可能需要積極載入多個不同的關聯。若要執行此操作，只需將關聯陣列傳遞給 `with` 方法：

```php
$books = Book::with(['author', 'publisher'])->get();
```


<a name="nested-eager-loading"></a>
#### 巢狀積極載入

若要積極載入關聯的關聯，您可以使用「點」語法。例如，讓我們積極載入書籍的所有作者以及作者的所有個人聯絡人：

```php
$books = Book::with('author.contacts')->get();
```

或者，您可以透過向 `with` 方法提供巢狀陣列來指定巢狀積極載入關聯，這在積極載入多個巢狀關聯時非常方便：

```php
$books = Book::with([
    'author' => [
        'contacts',
        'publisher',
    ],
])->get();
```


<a name="nested-eager-loading-morphto-relationships"></a>
#### 積極載入巢狀 `morphTo` 關聯

如果您想積極載入 `morphTo` 關聯，以及該關聯可能回傳的各種實體上的巢狀關聯，您可以將 `with` 方法與 `morphTo` 關聯的 `morphWith` 方法結合使用。為了幫助說明此方法，讓我們考慮以下模型：

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

在此範例中，讓我們假設 `Event`、`Photo` 和 `Post` 模型可能會建立 `ActivityFeed` 模型。此外，讓我們假設 `Event` 模型屬於 `Calendar` 模型，`Photo` 模型與 `Tag` 模型相關聯，而 `Post` 模型屬於 `Author` 模型。

使用這些模型定義和關聯，我們可以取得 `ActivityFeed` 模型實例並積極載入所有 `parentable` 模型及其各自的巢狀關聯：

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
#### 積極載入特定欄位

您可能並不總是需要從取得的關聯中取得每個欄位。因此，Eloquent 允許您指定想要取得關聯的哪些欄位：

```php
$books = Book::with('author:id,name,book_id')->get();
```

> [!WARNING]
> 使用此功能時，您應始終在想要取得的欄位清單中包含 `id` 欄位和任何相關的外鍵欄位。


<a name="eager-loading-by-default"></a>
#### 預設積極載入

有時您可能希望在取得模型時始終載入某些關聯。若要實現此目的，您可以在模型上定義一個 `$with` 屬性：

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

如果您想在單個查詢中從 `$with` 屬性中移除某個項目，可以使用 `without` 方法：

```php
$books = Book::without('author')->get();
```

如果您想在單個查詢中覆蓋 `$with` 屬性中的所有項目，可以使用 `withOnly` 方法：

```php
$books = Book::withOnly('genre')->get();
```

<a name="constraining-eager-loads"></a>
### 限制積極載入

有時您可能希望積極載入某個關聯，但同時也想為該積極載入查詢指定額外的查詢條件。您可以透過傳遞一個關聯陣列給 `with` 方法來達成此目的，其中陣列的鍵 (Key) 是關聯名稱，而值 (Value) 則是一個為積極載入查詢增加額外限制的閉包 (Closure)：

```php
use App\Models\User;

$users = User::with(['posts' => function ($query) {
    $query->where('title', 'like', '%code%');
}])->get();
```

在此範例中，Eloquent 將只會積極載入貼文 `title` 欄位包含單字 `code` 的貼文。您可以呼叫其他的 [查詢產生器 (Query Builder)](/docs/{{version}}/queries) 方法來進一步自定義積極載入操作：

```php
$users = User::with(['posts' => function ($query) {
    $query->orderBy('created_at', 'desc');
}])->get();
```


<a name="constraining-eager-loading-of-morph-to-relationships"></a>
#### 限制 `morphTo` 關聯的積極載入

如果您正在積極載入一個 `morphTo` 關聯，Eloquent 將會執行多個查詢來取得每種型別的相關模型。您可以使用 `MorphTo` 關聯的 `constrain` 方法，為這些查詢中的每一個加入額外的限制：

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

在此範例中，Eloquent 將只會積極載入尚未被隱藏的貼文以及 `type` 值為 「educational」 的影片。


<a name="constraining-eager-loads-with-relationship-existence"></a>
#### 根據關聯是否存在來限制積極載入

有時您可能會發現自己需要檢查某個關聯是否存在，同時又想根據相同的條件來載入該關聯。例如，您可能只想取得擁有符合特定查詢條件的子代 `Post` 模型的 `User` 模型，同時積極載入這些符合條件的貼文。您可以使用 `withWhereHas` 方法來完成此操作：

```php
use App\Models\User;

$users = User::withWhereHas('posts', function ($query) {
    $query->where('featured', true);
})->get();
```


<a name="lazy-eager-loading"></a>
### 延遲積極載入

有時您可能需要在取得父模型之後才積極載入某個關聯。例如，當您需要動態決定是否載入關聯模型時，這會非常有用：

```php
use App\Models\Book;

$books = Book::all();

if ($condition) {
    $books->load('author', 'publisher');
}
```

如果您需要為積極載入查詢設定額外的查詢限制，可以傳遞一個以您想載入的關聯為鍵的陣列。陣列的值應為接收查詢實例的閉包實例：

```php
$author->load(['books' => function ($query) {
    $query->orderBy('published_date', 'asc');
}]);
```

若要僅在關聯尚未被載入時才載入它，請使用 `loadMissing` 方法：

```php
$book->loadMissing('author');
```


<a name="nested-lazy-eager-loading-morphto"></a>
#### 巢狀延遲積極載入與 `morphTo`

如果您想積極載入 `morphTo` 關聯，以及該關聯可能回傳的各種實體上的巢狀關聯，可以使用 `loadMorph` 方法。

此方法接受 `morphTo` 關聯的名稱作為第一個參數，並接受一個模型 / 關聯配對陣列作為第二個參數。為了說明此方法，讓我們考慮以下模型：

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

在此範例中，讓我們假設 `Event`、`Photo` 和 `Post` 模型都可以建立 `ActivityFeed` 模型。此外，假設 `Event` 模型屬於一個 `Calendar` 模型，`Photo` 模型與 `Tag` 模型相關聯，而 `Post` 模型屬於一個 `Author` 模型。

使用這些模型定義和關聯，我們可以取得 `ActivityFeed` 模型實例，並積極載入所有 `parentable` 模型及其各自的巢狀關聯：

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
### 自動積極載入

> [!WARNING]
> 此功能目前處於 Beta 階段，以收集社群回饋。此功能的行為與功能甚至可能在補丁版本 (Patch Releases) 中發生變化。

在許多情況下，Laravel 可以自動積極載入您存取的關聯。要啟用自動積極載入，您應該在應用程式 `AppServiceProvider` 的 `boot` 方法中呼叫 `Model::automaticallyEagerLoadRelationships` 方法：

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

當啟用此功能時，Laravel 將嘗試自動載入您存取且先前尚未載入的任何關聯。例如，考慮以下場景：

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

通常情況下，上述程式碼會為每個使用者執行一個查詢以取得其貼文，並為每篇貼文執行一個查詢以取得其留言。然而，當 `automaticallyEagerLoadRelationships` 功能啟用時，當您嘗試存取任何已取得使用者的貼文時，Laravel 將自動為使用者集合中的所有使用者「延遲積極載入 (Lazy Eager Load)」貼文。同樣地，當您嘗試存取任何已取得貼文的留言時，所有留言都會為原本取得的所有貼文進行延遲積極載入。

如果您不想全域啟用自動積極載入，您仍然可以透過在集合上呼叫 `withRelationshipAutoloading` 方法，來為單個 Eloquent 集合實例啟用此功能：

```php
$users = User::where('vip', true)->get();

return $users->withRelationshipAutoloading();
```

<a name="preventing-lazy-loading"></a>
### 防止懶惰載入

如前所述，積極載入關聯通常可以為您的應用程式提供顯著的效能優勢。因此，如果您願意，可以指示 Laravel 始終防止關聯的懶惰載入。要實現此目的，您可以呼叫基礎 Eloquent 模型類別提供的 `preventLazyLoading` 方法。通常，您應該在應用程式的 `AppServiceProvider` 類別的 `boot` 方法中呼叫此方法。

`preventLazyLoading` 方法接受一個選填的布林值參數，用以指示是否應防止懶惰載入。例如，您可能希望僅在非正式環境中停用懶惰載入，以便即使正式環境的程式碼中意外出現懶惰載入關聯，正式環境仍能繼續正常運作：

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

防止懶惰載入後，當您的應用程式嘗試懶惰載入任何 Eloquent 關聯時，Eloquent 將會拋出 `Illuminate\Database\LazyLoadingViolationException` 例外。

您可以使用 `handleLazyLoadingViolationsUsing` 方法來自定義懶惰載入違規的行為。例如，透過此方法，您可以指示僅記錄懶惰載入違規，而不是以例外中斷應用程式的執行：

```php
Model::handleLazyLoadingViolationUsing(function (Model $model, string $relation) {
    $class = $model::class;

    info("Attempted to lazy load [{$relation}] on model [{$class}].");
});
```

<a name="inserting-and-updating-related-models"></a>
## 新增與更新關聯模型

<a name="the-save-method"></a>
### `save` 方法

Eloquent 提供了一些方便的方法來將新模型新增到關聯中。例如，您可能需要為貼文新增一則評論。與其手動設定 `Comment` 模型上的 `post_id` 屬性，不如使用關聯的 `save` 方法來插入評論：

```php
use App\Models\Comment;
use App\Models\Post;

$comment = new Comment(['message' => 'A new comment.']);

$post = Post::find(1);

$post->comments()->save($comment);
```

請注意，我們沒有將 `comments` 關聯當作動態屬性來存取。相反地，我們呼叫了 `comments` 方法來取得關聯的實例。`save` 方法會自動將適當的 `post_id` 值新增到新的 `Comment` 模型中。

如果您需要儲存多個關聯模型，可以使用 `saveMany` 方法：

```php
$post = Post::find(1);

$post->comments()->saveMany([
    new Comment(['message' => 'A new comment.']),
    new Comment(['message' => 'Another new comment.']),
]);
```

`save` 與 `saveMany` 方法會將給定的模型實例持久化，但不會將新持久化的模型新增到任何已載入到父模型上的記憶體中關聯。如果您打算在呼叫 `save` 或 `saveMany` 方法後存取該關聯，您可能需要使用 `refresh` 方法來重新載入模型及其關聯：

```php
$post->comments()->save($comment);

$post->refresh();

// All comments, including the newly saved comment...
$post->comments;
```

<a name="the-push-method"></a>
#### 遞迴儲存模型與關聯

如果您想要 `save` 您的模型及其所有關聯的關聯，您可以使用 `push` 方法。在此範例中，`Post` 模型及其評論與評論的作者都將被儲存：

```php
$post = Post::find(1);

$post->comments[0]->message = 'Message';
$post->comments[0]->author->name = 'Author Name';

$post->push();
```

`pushQuietly` 方法可用於儲存模型及其關聯的關聯，且不會觸發任何事件：

```php
$post->pushQuietly();
```

<a name="the-create-method"></a>
### `create` 方法

除了 `save` 與 `saveMany` 方法之外，您還可以使用 `create` 方法。該方法接受一個屬性陣列，建立模型，並將其插入資料庫。`save` 與 `create` 之間的區別在於 `save` 接受一個完整的 Eloquent 模型實例，而 `create` 接受一個純 PHP 的 `array`。`create` 方法會回傳新建立的模型：

```php
use App\Models\Post;

$post = Post::find(1);

$comment = $post->comments()->create([
    'message' => 'A new comment.',
]);
```

您可以使用 `createMany` 方法來建立多個關聯模型：

```php
$post = Post::find(1);

$post->comments()->createMany([
    ['message' => 'A new comment.'],
    ['message' => 'Another new comment.'],
]);
```

`createQuietly` 與 `createManyQuietly` 方法可用於在不發送任何事件的情況下建立模型：

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

您還可以使用 `findOrNew`、`firstOrNew`、`firstOrCreate` 與 `updateOrCreate` 方法在[關聯上建立與更新模型](/docs/{{version}}/eloquent#upserts)。

> [!NOTE]
> 在使用 `create` 方法之前，請務必查閱[批量賦值 (Mass Assignment)](/docs/{{version}}/eloquent#mass-assignment) 說明文件。

<a name="updating-belongs-to-relationships"></a>
### Belongs To 關聯

如果您想要將子模型指派給新的父模型，可以使用 `associate` 方法。在此範例中，`User` 模型對 `Account` 模型定義了一個 `belongsTo` 關聯。此 `associate` 方法將設定子模型上的外鍵：

```php
use App\Models\Account;

$account = Account::find(10);

$user->account()->associate($account);

$user->save();
```

要從子模型中移除父模型，可以使用 `dissociate` 方法。此方法會將關聯的外鍵設為 `null`：

```php
$user->account()->dissociate();

$user->save();
```

<a name="updating-many-to-many-relationships"></a>
### 多對多關聯

<a name="attaching-detaching"></a>
#### 附加 / 分離

Eloquent 也提供了一些方法，讓處理多對多關聯更加方便。例如，假設一個使用者可以擁有多個角色，而一個角色也可以擁有多個使用者。您可以使用 `attach` 方法透過在關聯的中間表中插入紀錄來將角色附加給使用者：

```php
use App\Models\User;

$user = User::find(1);

$user->roles()->attach($roleId);
```

在將關聯附加到模型時，您也可以傳遞一個包含額外資料的陣列，以插入到中間表中：

```php
$user->roles()->attach($roleId, ['expires' => $expires]);
```

有時可能需要從使用者中移除某個角色。要移除多對多關聯紀錄，請使用 `detach` 方法。`detach` 方法會將適當的紀錄從中間表中刪除；然而，兩個模型都將保留在資料庫中：

```php
// Detach a single role from the user...
$user->roles()->detach($roleId);

// Detach all roles from the user...
$user->roles()->detach();
```

為了方便起見， `attach` 與 `detach` 也接受 ID 陣列作為輸入：

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

您還可以使用 `sync` 方法來建構多對多關聯。`sync` 方法接受一個 ID 陣列，並將其放入中間表中。任何不在給定陣列中的 ID 都將從中間表中移除。因此，在此操作完成後，中間表中將僅存在給定陣列中的 ID：

```php
$user->roles()->sync([1, 2, 3]);
```

您也可以隨 ID 傳遞額外的新增中間表值：

```php
$user->roles()->sync([1 => ['expires' => true], 2, 3]);
```

如果您想要為每個同步的模型 ID 插入相同的中間表值，可以使用 `syncWithPivotValues` 方法：

```php
$user->roles()->syncWithPivotValues([1, 2, 3], ['active' => true]);
```

如果您不想分離不在給定陣列中的現有 ID，可以使用 `syncWithoutDetaching` 方法：

```php
$user->roles()->syncWithoutDetaching([1, 2, 3]);
```

<a name="toggling-associations"></a>
#### 切換關聯

多對多關聯還提供了一個 `toggle` 方法，它可以「切換」給定關聯模型 ID 的附加狀態。如果給定的 ID 目前已附加，它將被分離。同樣地，如果目前已分離，它將被附加：

```php
$user->roles()->toggle([1, 2, 3]);
```

您也可以隨 ID 傳遞額外的新增中間表值：

```php
$user->roles()->toggle([
    1 => ['expires' => true],
    2 => ['expires' => true],
]);
```

<a name="updating-a-record-on-the-intermediate-table"></a>
#### 更新中間表上的紀錄

如果您需要更新關聯中間表中的現有行，可以使用 `updateExistingPivot` 方法。此方法接受中間紀錄的外鍵以及一個要更新的屬性陣列：

```php
$user = User::find(1);

$user->roles()->updateExistingPivot($roleId, [
    'active' => false,
]);
```

<a name="touching-parent-timestamps"></a>
## 更新父模型時間戳記

當模型定義了對另一個模型的 `belongsTo` 或 `belongsToMany` 關聯時，例如隸屬於 `Post` 的 `Comment`，有時在子模型被更新時同步更新父模型的時間戳記會很有幫助。

例如，當 `Comment` 模型被更新時，你可能希望自動「觸碰 (touch)」所屬 `Post` 的 `updated_at` 時間戳記，使其設為當前日期與時間。若要達成此目的，你可以在子模型中加入一個 `touches` 屬性，其中包含在子模型更新時，應同步更新其 `updated_at` 時間戳記的關聯名稱：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class Comment extends Model
{
    /**
     * All of the relationships to be touched.
     *
     * @var array
     */
    protected $touches = ['post'];

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
> 父模型的時間戳記只有在子模型使用 Eloquent 的 `save` 方法更新時才會被更新。