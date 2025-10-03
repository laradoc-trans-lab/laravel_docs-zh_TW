# Eloquent：關聯

- [介紹](#introduction)
- [定義關聯](#defining-relationships)
    - [一對一 / Has One](#one-to-one)
    - [一對多 / Has Many](#one-to-many)
    - [一對多 (反向) / Belongs To](#one-to-many-inverse)
    - [Has One of Many](#has-one-of-many)
    - [Has One Through](#has-one-through)
    - [Has Many Through](#has-many-through)
- [作用域關聯](#scoped-relationships)
- [多對多關聯](#many-to-many)
    - [取得中介資料表欄位](#retrieving-intermediate-table-columns)
    - [透過中介資料表欄位篩選查詢](#filtering-queries-via-intermediate-table-columns)
    - [透過中介資料表欄位排序查詢](#ordering-queries-via-intermediate-table-columns)
    - [定義自訂中介資料表 Model](#defining-custom-intermediate-table-models)
- [多態關聯](#polymorphic-relationships)
    - [一對一](#one-to-one-polymorphic-relations)
    - [一對多](#one-to-many-polymorphic-relations)
    - [一對多中的其一](#one-of-many-polymorphic-relations)
    - [多對多 (Polymorphic)](#many-to-many-polymorphic-relations)
    - [自訂多態型別](#custom-polymorphic-types)
- [動態關聯](#dynamic-relationships)
- [查詢關聯](#querying-relations)
    - [關聯方法 vs. 動態屬性](#relationship-methods-vs-dynamic-properties)
    - [查詢關聯的存在](#querying-relationship-existence)
    - [查詢關聯的不存在](#querying-relationship-absence)
    - [查詢 Morph To 關聯](#querying-morph-to-relationships)
- [聚合關聯 Model](#aggregating-related-models)
    - [計數關聯 Model](#counting-related-models)
    - [其他聚合函式](#other-aggregate-functions)
    - [在 Morph To 關聯上計數關聯 Model](#counting-related-models-on-morph-to-relationships)
- [預先載入 (Eager Loading)](#eager-loading)
    - [限制預先載入](#constraining-eager-loads)
    - [延遲預先載入 (Lazy Eager Loading)](#lazy-eager-loading)
    - [自動預先載入](#automatic-eager-loading)
    - [避免延遲載入 (Lazy Loading)](#preventing-lazy-loading)
- [新增與更新關聯 Model](#inserting-and-updating-related-models)
    - [`save` 方法](#the-save-method)
    - [`create` 方法](#the-create-method)
    - [Belongs To 關聯](#updating-belongs-to-relationships)
    - [多對多關聯](#updating-many-to-many-relationships)
- [接觸 (Touch) 父項時間戳記](#touching-parent-timestamps)

<a name="introduction"></a>
## 介紹

資料庫的資料表之間常有關聯。舉例來說，一篇部落格文章可能有很多則留言，或是一筆訂單可能與下單的使用者有關。Eloquent 讓管理與操作這些關聯變得簡單，並支援多種常見的關聯：

<div class="content-list" markdown="1">

- [一對一](#one-to-one)
- [一對多](#one-to-many)
- [多對多](#many-to-many)
- [Has One Through](#has-one-through)
- [Has Many Through](#has-many-through)
- [一對一 (多態)](#one-to-one-polymorphic-relations)
- [一對多 (多態)](#one-to-many-polymorphic-relations)
- [多對多 (多態)](#many-to-many-polymorphic-relations)

</div>

<a name="defining-relationships"></a>
## 定義關聯

Eloquent 關련是以方法的形式定義在 Eloquent Model 類別上。由於關聯同時也作為強大的[查詢產生器](/docs/{{version}}/queries)，因此將關聯定義為方法可提供強大的方法鏈式呼叫與查詢功能。舉例來說，我們可以在這個 `posts` 關聯上鏈式呼叫其他的查詢限制：

```php
$user->posts()->where('active', 1)->get();
```

不過，在深入使用關聯前，讓我們先來學習如何定義 Eloquent 所支援的每種關聯類型。

<a name="one-to-one"></a>
### 一對一 / Has One

「一對一」關聯是一種非常基本的資料庫關聯。舉例來說，一個 `User` Model 可能會關聯到一個 `Phone` Model。為定義這個關聯，我們會在 `User` Model 上放置一個 `phone` 方法。`phone` 方法應呼叫 `hasOne` 方法並回傳其結果。`hasOne` 方法是透過 Model 的 `Illuminate\Database\Eloquent\Model` 基底類別提供給你的 Model 使用：

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

傳給 `hasOne` 方法的第一個引數是關聯 Model 的類別名稱。關聯定義好後，我們就可以使用 Eloquent 的動態屬性來取得關聯的紀錄。動態屬性能讓你在存取關聯方法時，像在存取 Model 上定義的屬性一樣：

```php
$phone = User::find(1)->phone;
```

Eloquent 會根據父 Model 的名稱來判斷關聯的外鍵。在這個例子中，`Phone` Model 會自動被假設有一個 `user_id` 外鍵。若想覆寫這個慣例，可以傳入第二個引數給 `hasOne` 方法：

```php
return $this->hasOne(Phone::class, 'foreign_key');
```

此外，Eloquent 會假設外鍵應有一個符合父項主鍵欄位的值。換句話說，Eloquent 會在 `Phone` 紀錄的 `user_id` 欄位中尋找使用者的 `id` 欄位的值。若想讓關聯使用 `id` 或 Model 的 `$primaryKey` 屬性以外的主鍵值，可以傳入第三個引數給 `hasOne` 方法：

```php
return $this->hasOne(Phone::class, 'foreign_key', 'local_key');
```

<a name="one-to-one-defining-the-inverse-of-the-relationship"></a>
#### 定義關聯的反向

這樣，我們就能從 `User` Model 存取 `Phone` Model 了。接著，讓我們在 `Phone` Model 上定義一個關聯，讓我們能存取擁有該電話的使用者。我們可以使用 `belongsTo` 方法來定義 `hasOne` 關聯的反向：

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

叫用 `user` 方法時，Eloquent 會嘗試尋找一個 `id` 符合 `Phone` Model 上 `user_id` 欄位的 `User` Model。

Eloquent 會檢查關聯方法的名稱並在後面加上 `_id` 來判斷外鍵的名稱。因此，在這個例子中，Eloquent 會假設 `Phone` Model 有一個 `user_id` 欄位。不過，若 `Phone` Model 上的外鍵不是 `user_id`，則可以傳入一個自訂的鍵名作為 `belongsTo` 方法的第二個引數：

```php
/**
 * Get the user that owns the phone.
 */
public function user(): BelongsTo
{
    return $this->belongsTo(User::class, 'foreign_key');
}
```

若父 Model 不使用 `id` 作為主鍵，或想使用不同的欄位來尋找關聯的 Model，則可以傳入第三個引數給 `belongsTo` 方法，以指定父資料表的自訂鍵：

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

「一對多」關聯是用來定義一個 Model 是多個子 Model 的父項的關聯。舉例來說，一篇部落格文章可能會有無限則留言。與所有其他的 Eloquent 關聯一樣，一對多關聯也是透過在 Eloquent Model 上定義方法來定義的：

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

記得，Eloquent 會自動為 `Comment` Model 判斷正確的外鍵欄位。依照慣例，Eloquent 會取用父 Model 名稱的「蛇形命名法 (snake case)」並在後面加上 `_id`。因此，在這個例子中，Eloquent 會假設 `Comment` Model 上的外鍵欄位是 `post_id`。

關聯方法定義好後，我們就可以透過存取 `comments` 屬性來存取關聯留言的[集合 (Collection)](/docs/{{version}}/eloquent-collections)。記得，因為 Eloquent 提供了「動態關聯屬性」，所以我們可以在存取關聯方法時，像在存取 Model 上定義的屬性一樣：

```php
use App\Models\Post;

$comments = Post::find(1)->comments;

foreach ($comments as $comment) {
    // ...
}
```

由於所有的關聯都可作為查詢產生器，因此可以透過呼叫 `comments` 方法並繼續鏈式呼叫條件到查詢上，來為關聯查詢加上更多限制：

```php
$comment = Post::find(1)->comments()
    ->where('title', 'foo')
    ->first();
```

與 `hasOne` 方法一樣，也可以傳入額外的引數給 `hasMany` 方法來覆寫外鍵與本地鍵：

```php
return $this->hasMany(Comment::class, 'foreign_key');

return $this->hasMany(Comment::class, 'foreign_key', 'local_key');
```

<a name="automatically-hydrating-parent-models-on-children"></a>
#### 自動填充子 Model 上的父 Model

即便使用了 Eloquent 的預先載入 (Eager Loading)，若在遍歷子 Model 時嘗試從子 Model 存取父 Model，還是可能會出現「N + 1」查詢問題：

```php
$posts = Post::with('comments')->get();

foreach ($posts as $post) {
    foreach ($post->comments as $comment) {
        echo $comment->post->title;
    }
}
```

在上方程式碼中，之所以會出現「N + 1」查詢問題，是因為雖然每個 `Post` Model 都預先載入了留言，但 Eloquent 並沒有自動在每個子 `Comment` Model 上填充 (Hydrate) 父 `Post`。

若想讓 Eloquent 自動在子 Model 上填充父 Model，可以在定義 `hasMany` 關聯時叫用 `chaperone` 方法：

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

或者，若想在執行時才選擇性地自動填充父項，可以在預先載入關聯時叫用 `chaperone` 方法：

```php
use App\Models\Post;

$posts = Post::with([
    'comments' => fn ($comments) => $comments->chaperone(),
])->get();
```

<a name="one-to-many-inverse"></a>
### 一對多 (反向) / Belongs To

現在我們已經可以存取一篇文章的所有留言了，接著讓我們來定義一個關聯，讓留言可以存取其父文章。若要定義 `hasMany` 關聯的反向關聯，請在子 Model 上定義一個關聯方法，並在其中呼叫 `belongsTo` 方法：

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

關聯定義好後，我們就可以存取 `post` 這個「動態關聯屬性」來取得留言的父文章：

```php
use App\Models\Comment;

$comment = Comment::find(1);

return $comment->post->title;
```

在上方程式碼中，Eloquent 會試著在 `Comment` Model 上找到一個 `id` 與 `post_id` 欄位符合的 `Post` Model。

Eloquent 會檢查關聯方法的名稱，並在方法名稱後方加上 `_` 與父 Model 的主鍵欄位名稱來當作預設的外鍵名稱。因此，在這個範例中，Eloquent 會假設 `comments` 資料表上 `Post` Model 的外鍵為 `post_id`。

不過，若關聯的外鍵不符合這些慣例，則可以傳入一個自訂的外鍵名稱作為 `belongsTo` 方法的第二個引數：

```php
/**
 * Get the post that owns the comment.
 */
public function post(): BelongsTo
{
    return $this->belongsTo(Post::class, 'foreign_key');
}
```

若父 Model 不使用 `id` 作為主鍵，或希望能使用不同的欄位來尋找關聯的 Model，則可以傳入第三個引數到 `belongsTo` 方法來指定父資料表的自訂鍵：

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

`belongsTo`、`hasOne`、`hasOneThrough`、`morphOne` 等關聯可讓你在給定關聯為 `null` 的情況下定義一個要回傳的預設 Model。這個模式通常被稱為 [空物件模式 (Null Object pattern)](https://en.wikipedia.org/wiki/Null_Object_pattern)，且有助於移除程式碼中的條件式檢查。在下列範例中，若沒有使用者附加到 `Post` Model，則 `user` 關聯會回傳一個空的 `App\Models\User` Model：

```php
/**
 * Get the author of the post.
 */
public function user(): BelongsTo
{
    return $this->belongsTo(User::class)->withDefault();
}
```

若要為預設 Model 填入屬性，可以傳入陣列或閉包到 `withDefault` 方法：

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

在查詢「belongs to」關聯的子項時，可以手動建立 `where` 子句來取得對應的 Eloquent Model：

```php
use App\Models\Post;

$posts = Post::where('user_id', $user->id)->get();
```

不過，我們可能會覺得使用 `whereBelongsTo` 方法更方便，該方法會自動為給定的 Model 判斷正確的關聯與外鍵：

```php
$posts = Post::whereBelongsTo($user)->get();
```

也可以提供 [Collection](/docs/{{version}}/eloquent-collections) 實體給 `whereBelongsTo` 方法。當這麼做時，Laravel 會擷取屬於該 Collection 中任何一個父 Model 的 Model：

```php
$users = User::where('vip', true)->get();

$posts = Post::whereBelongsTo($users)->get();
```

預設情況下，Laravel 會根據給定 Model 的類別名稱來判斷要與哪個關聯關聯；不過，我們也可以手動將關聯名稱作為第二個引數傳給 `whereBelongsTo` 方法來指定關聯名稱：

```php
$posts = Post::whereBelongsTo($user, 'author')->get();
```

<a name="has-one-of-many"></a>
### Has One of Many

有時候，一個 Model 可能會有很多個關聯 Model，但我們只想輕鬆地取得關聯中「最新」或「最舊」的那個關聯 Model。舉例來說，一個 `User` Model 可能會關聯到多個 `Order` Model，但我們想定義一個方便的方法來與該使用者最近下的一筆訂單互動。我們可以結合 `hasOne` 關聯類型與 `ofMany` 方法來達成此目的：

```php
/**
 * Get the user's most recent order.
 */
public function latestOrder(): HasOne
{
    return $this->hasOne(Order::class)->latestOfMany();
}
```

同樣地，我們也可以定義一個方法來取得關聯中「最舊的」，也就是第一個關聯 Model：

```php
/**
 * Get the user's oldest order.
 */
public function oldestOrder(): HasOne
{
    return $this->hasOne(Order::class)->oldestOfMany();
}
```

預設情況下，`latestOfMany` 與 `oldestOfMany` 方法會根據 Model 的主鍵來取得最新或最舊的關聯 Model，因此主鍵必須是可排序的。不過，有時候我們可能會想用不同的排序條件來從一個較大的關聯中取得單一 Model。

舉例來說，使用 `ofMany` 方法，我們可以取得某個使用者最貴的一筆訂單。`ofMany` 方法的第一個引數是可排序的欄位，第二個引數則是要在查詢關聯 Model 時要套用的聚合函式 (`min` 或 `max`)：

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
> 由於 PostgreSQL 不支援對 UUID 欄位執行 `MAX` 函式，因此目前無法將「一對多中的其一」關聯與 PostgreSQL 的 UUID 欄位一起使用。

<a name="converting-many-relationships-to-has-one-relationships"></a>
#### 將「多」關聯轉為 Has One 關聯

通常，在使用 `latestOfMany`、`oldestOfMany`、或 `ofMany` 方法來取得單一 Model 時，我們可能已經為同一個 Model 定義了一個「一對多」關聯。為方便起見，Laravel 允許我們在關聯上叫用 `one` 方法，來輕鬆地將這個關聯轉為「一對一」關聯：

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

也可以使用 `one` 方法來將 `HasManyThrough` 關聯轉為 `HasOneThrough` 關聯：

```php
public function latestDeployment(): HasOneThrough
{
    return $this->deployments()->one()->latestOfMany();
}
```

<a name="advanced-has-one-of-many-relationships"></a>
#### 進階 Has One of Many 關聯

我們也可以建構更進階的「一對多中的其一」關聯。举例来说，一个 `Product` Model 可能会关联到多个 `Price` Model，且这些价格在发布新价格后仍会保留在系统中。此外，產品的新價格資料也可能可以預先發布，並透過一個 `published_at` 欄位在未來的某個日期生效。

總結來說，我們需要取得已發布的最新價格，且其發布日期不能在未來。此外，若有兩個價格的發布日期相同，我們會偏好 ID 較大的那個價格。為此，我們必須傳遞一個陣列給 `ofMany` 方法，其中包含用來判斷最新價格的可排序欄位。此外，我們還會提供一個閉包作為 `ofMany` 方法的第二個引數。這個閉包會負責為關聯查詢加上額外的發布日期限制：

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

「Has-one-through (遠層一對一)」關聯用來定義與另一個 Model 的一對一關聯。不過，這個關聯表示，定義關聯的 Model 可以藉由 _通過_ 第三個 Model 來與另一個 Model 的一個實體匹配。

舉例來說，在一個車輛維修廠的應用程式中，每個 `Mechanic` (技師) Model 可能會關聯到一個 `Car` (汽車) Model，而每個 `Car` Model 可能會關聯到一個 `Owner` (車主) Model。雖然技師與車主在資料庫中沒有直接的關聯，但技師可以 _透過_ `Car` Model 來存取車主。讓我們先來看看定義這個關聯所需的資料表：

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

看完關聯的資料表結構後，讓我在 `Mechanic` Model 上定義關聯：

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

傳給 `hasOneThrough` 方法的第一個引數是我們想存取的最終 Model 名稱，而第二個引數則是中介 Model 的名稱。

或者，若關聯中牽涉到的所有 Model 都已經定義好了相關的關聯，我們也可以叫用 `through` 方法並提供這些關聯的名稱來流暢地定義一個「has-one-through」關聯。舉例來說，若 `Mechanic` Model 有一個 `cars` 關聯、且 `Car` Model 有一個 `owner` 關聯，我們可以像這樣定義一個連接技師與車主的「has-one-through」關聯：

```php
// String based syntax...
return $this->through('cars')->has('owner');

// Dynamic syntax...
return $this->throughCars()->hasOwner();
```

<a name="has-one-through-key-conventions"></a>
#### 鍵的慣例

在執行關聯查詢時，會使用典型的 Eloquent 外鍵慣例。若想自訂關聯的鍵，可以將其作為第三與第四個引數傳給 `hasOneThrough` 方法。第三個引數是中介 Model 上的外鍵名稱。第四個引數是最終 Model 上的外鍵名稱。第五個引數是本地鍵，而第六個引數則是中介 Model 上的本地鍵：

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

或者，如前所述，若關聯中牽涉到的所有 Model 都已經定義好了相關的關聯，我們也可以叫用 `through` 方法並提供這些關聯的名稱來流暢地定義一個「has-one-through」關聯。這個做法的優點是可以重用已定義在現有關聯上的鍵慣例：

```php
// String based syntax...
return $this->through('cars')->has('owner');

// Dynamic syntax...
return $this->throughCars()->hasOwner();
```

<a name="has-many-through"></a>
### Has Many Through

「has-many-through」關聯提供了一種方便的方法，能透過一個中介關聯來存取遠程的關聯。舉例來說，假設我們正在建立一個像 [Laravel Cloud](https://cloud.laravel.com) 這樣的部署平台。一個 `Application` Model 可能會透過一個中介的 `Environment` Model 來存取多個 `Deployment` Model。使用這個範例，你可以輕鬆地為給定的應用程式收集所有的部署。讓我們來看看定義這個關聯所需的資料表：

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

現在我們已經檢查了關聯的資料表結構，讓我們在 `Application` Model 上定義關聯：

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

傳遞給 `hasManyThrough` 方法的第一個引數是我們希望存取的最終 Model 的名稱，而第二個引數是中介 Model 的名稱。

或者，如果關聯中涉及的所有 Model 上都已經定義了相關的關聯，你可以透過叫用 `through` 方法並提供這些關聯的名稱，來流暢地定義一個「has-many-through」關聯。例如，如果 `Application` Model 有一個 `environments` 關聯，而 `Environment` Model 有一個 `deployments` 關聯，你可以像這樣定義一個連接應用程式和部署的「has-many-through」關聯：

```php
// String based syntax...
return $this->through('environments')->has('deployments');

// Dynamic syntax...
return $this->throughEnvironments()->hasDeployments();
```

雖然 `Deployment` Model 的資料表不包含 `application_id` 欄位，但 `hasManyThrough` 關聯提供了透過 `$application->deployments` 存取應用程式部署的功能。為了取得這些 Model，Eloquent 會檢查中介的 `Environment` Model 資料表上的 `application_id` 欄位。找到相關的環境 ID 後，它們會被用來查詢 `Deployment` Model 的資料表。


<a name="has-many-through-key-conventions"></a>
#### 鍵的慣例

在執行關聯查詢時，將會使用典型的 Eloquent 外鍵慣例。如果你想要自訂關聯的鍵，你可以將它們作為第三和第四個引數傳遞給 `hasManyThrough` 方法。第三個引數是中介 Model 上的外鍵名稱。第四個引數是最終 Model 上的外鍵名稱。第五個引數是本地鍵，而第六個引數是中介 Model 的本地鍵：

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

或者，如前所述，如果關聯中涉及的所有 Model 上都已經定義了相關的關聯，你可以透過叫用 `through` 方法並提供這些關聯的名稱，來流暢地定義一個「has-many-through」關聯。這種方法的好處是能重用現有關聯上已定義的鍵慣例：

```php
// String based syntax...
return $this->through('environments')->has('deployments');

// Dynamic syntax...
return $this->throughEnvironments()->hasDeployments();
```


<a name="scoped-relationships"></a>
### 作用域關聯

在 Model 中新增額外的方法來限制關聯是很常見的。例如，你可能會在 `User` Model 中新增一個 `featuredPosts` 方法，該方法會以一個額外的 `where` 限制式來限制更廣泛的 `posts` 關聯：

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

然而，如果你嘗試透過 `featuredPosts` 方法來建立一個 Model，它的 `featured` 屬性將不會被設定為 `true`。如果你希望透過關聯方法來建立 Model，同時也指定應被新增到所有透過該關聯建立的 Model 上的屬性，你可以在建立關聯查詢時使用 `withAttributes` 方法：

```php
/**
 * Get the user's featured posts.
 */
public function featuredPosts(): HasMany
{
    return $this->posts()->withAttributes(['featured' => true]);
}
```

`withAttributes` 方法會使用給定的屬性在查詢中新增 `where` 條件，並且它也會將給定的屬性新增到任何透過該關聯方法建立的 Model 上：

```php
$post = $user->featuredPosts()->create(['title' => 'Featured Post']);

$post->featured; // true
```

若要指示 `withAttributes` 方法不要在查詢中新增 `where` 條件，你可以將 `asConditions` 引數設定為 `false`：

```php
return $this->posts()->withAttributes(['featured' => true], asConditions: false);
```

<a name="many-to-many"></a>
## 多對多關聯

多對多關聯比 `hasOne` 與 `hasMany` 關聯要來得複雜一些。多對多關聯的一個例子是，一個使用者可以有多個角色，而這些角色也可以被應用程式中的其他使用者共用。例如，一個使用者可能會被指派「作者」與「編輯」的角色；然而，這些角色也可能會被指派給其他使用者。所以，一個使用者有多個角色，而一個角色也有多個使用者。


<a name="many-to-many-table-structure"></a>
#### 資料表結構

為了定義這種關聯，需要三個資料庫資料表：`users`、`roles`、與 `role_user`。`role_user` 資料表的名稱是從關聯 Model 名稱的字母順序衍生而來，其中包含 `user_id` 與 `role_id` 欄位。這個資料表被用來作為連結使用者與角色的中介資料表。

請記住，由於一個角色可以屬於多個使用者，我們不能只在 `roles` 資料表上放一個 `user_id` 欄位。這樣做的話，一個角色就只能屬於單一使用者。為了支援將角色指派給多個使用者，就需要 `role_user` 這個資料表。我們可以像這樣總結這個關聯的資料表結構：

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

多對多關聯是透過撰寫一個方法來定義的，該方法會回傳 `belongsToMany` 方法的結果。`belongsToMany` 方法是由 `Illuminate\Database\Eloquent\Model` 這個基礎類別提供的，你應用程式中所有的 Eloquent Model 都會使用到這個基礎類別。舉例來說，我們來在 `User` Model 上定義一個 `roles` 方法。傳給這個方法的第一個引數是關聯 Model 的類別名稱：

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

定義好關聯後，就可以使用 `roles` 動態關聯屬性來存取使用者的角色：

```php
use App\Models\User;

$user = User::find(1);

foreach ($user->roles as $role) {
    // ...
}
```

由於所有的關聯同時也都是查詢產生器，因此可以呼叫 `roles` 方法並繼續在查詢上鏈式地附加條件來為關聯查詢增加更多限制：

```php
$roles = User::find(1)->roles()->orderBy('name')->get();
```

為了判斷關聯中介資料表的名稱，Eloquent 會將兩個關聯 Model 的名稱按字母順序合併。不過，你也可以自由地覆寫這個慣例。只要傳遞第二個引數給 `belongsToMany` 方法即可：

```php
return $this->belongsToMany(Role::class, 'role_user');
```

除了自訂中介資料表的名稱外，你也可以透過傳遞額外的引數給 `belongsToMany` 方法來自訂資料表上鍵(Key)的欄位名稱。第三個引數是你正在定義關聯的那個 Model 的外鍵名稱，而第四個引數則是要連結的那個 Model 的外鍵名稱：

```php
return $this->belongsToMany(Role::class, 'role_user', 'user_id', 'role_id');
```


<a name="many-to-many-defining-the-inverse-of-the-relationship"></a>
#### 定義反向關聯

若要定義多對多關聯的「反向」關聯，應在關聯的 Model 上定義一個方法，該方法也應回傳 `belongsToMany` 方法的結果。為了完成我們的使用者／角色範例，我們來在 `Role` Model 上定義 `users` 方法：

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

如你所見，除了參照的 Model 是 `App\Models\User` 外，這個關聯的定義與其在 `User` Model 上的對應部分完全相同。由於我們重複使用了 `belongsToMany` 方法，因此在定義多對多關聯的「反向」關聯時，所有常見的資料表與鍵的自訂選項都可以使用。


<a name="retrieving-intermediate-table-columns"></a>
### 取得中介資料表欄位

如你所學，處理多對多關聯需要一個中介資料表。Eloquent 提供了一些非常有用的方法來與這個資料表互動。舉例來說，假設我們的 `User` Model 與多個 `Role` Model 有關聯。在存取這個關聯後，我們可以使用 Model 上的 `pivot` 屬性來存取中介資料表：

```php
use App\Models\User;

$user = User::find(1);

foreach ($user->roles as $role) {
    echo $role->pivot->created_at;
}
```

請注意，我們取出的每個 `Role` Model 都會自動被指派一個 `pivot` 屬性。這個屬性包含一個代表中介資料表的 Model。

預設情況下，只有 Model 的鍵會出現在 `pivot` Model 上。如果你的中介資料表包含額外的屬性，則必須在定義關聯時指定它們：

```php
return $this->belongsToMany(Role::class)->withPivot('active', 'created_by');
```

若想讓你的中介資料表有 `created_at` 與 `updated_at` 時間戳記，並由 Eloquent 自動維護，請在定義關聯時呼叫 `withTimestamps` 方法：

```php
return $this->belongsToMany(Role::class)->withTimestamps();
```

> [!WARNING]
> 使用 Eloquent 自動維護的時間戳記的中介資料表，必須同時有 `created_at` 與 `updated_at` 這兩個時間戳記欄位。


<a name="customizing-the-pivot-attribute-name"></a>
#### 自訂 `pivot` 屬性名稱

如前所述，可以透過 `pivot` 屬性在 Model 上存取中介資料表的屬性。不過，你也可以自由地自訂這個屬性的名稱，以更好地反映其在你應用程式中的用途。

例如，若你的應用程式中有使用者可以訂閱 Podcast，那你很可能在使用者與 Podcast 之間有一個多對多關聯。在這種情況下，你可能會想把中介資料表的屬性從 `pivot` 重新命名為 `subscription`。這可以在定義關聯時使用 `as` 方法來完成：

```php
return $this->belongsToMany(Podcast::class)
    ->as('subscription')
    ->withTimestamps();
```

指定了自訂的中介資料表屬性後，就可以使用自訂的名稱來存取中介資料表的資料：

```php
$users = User::with('podcasts')->get();

foreach ($users->flatMap->podcasts as $podcast) {
    echo $podcast->subscription->created_at;
}
```

<a name="filtering-queries-via-intermediate-table-columns"></a>
### 透過中介資料表欄位篩選查詢

在定義關聯時，也可以使用 `wherePivot`、`wherePivotIn`、`wherePivotNotIn`、`wherePivotBetween`、`wherePivotNotBetween`、`wherePivotNull`、`wherePivotNotNull` 等方法來篩選 `belongsToMany` 關聯查詢所回傳的結果：

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

`wherePivot` 會為查詢加上一個 where 子句限制，但在透過已定義的關聯來建立新 Model 時，並不會加上指定的值。若需要查詢並建立帶有特定中介值的關聯，可使用 `withPivotValue` 方法：

```php
return $this->belongsToMany(Role::class)
    ->withPivotValue('approved', 1);
```

<a name="ordering-queries-via-intermediate-table-columns"></a>
### 透過中介資料表欄位排序查詢

可以使用 `orderByPivot` 方法來為 `belongsToMany` 關聯查詢回傳的結果排序。在下列範例中，我們會擷取該使用者所有最新的徽章：

```php
return $this->belongsToMany(Badge::class)
    ->where('rank', 'gold')
    ->orderByPivot('created_at', 'desc');
```

<a name="defining-custom-intermediate-table-models"></a>
### 定義自訂中介資料表 Model

若想定義一個自訂的 Model 來代表多對多關聯的中介資料表，可以在定義關聯時呼叫 `using` 方法。自訂的中介(Pivot) Model 讓你有機會在中介 Model 上定義額外的行為，如方法 (Method) 與類型轉換 (Cast)。

自訂的多對多中介 Model 應繼承 `Illuminate\Database\Eloquent\Relations\Pivot` Class，而自訂的多態多對多中介 Model 則應繼承 `Illuminate\Database\Eloquent\Relations\MorphPivot` Class。舉例來說，我們可以定義一個使用自訂 `RoleUser` 中介 Model 的 `Role` Model：

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

在定義 `RoleUser` Model 時，應繼承 `Illuminate\Database\Eloquent\Relations\Pivot` Class：

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
> 中介 Model (Pivot Model) 不能使用 `SoftDeletes` Trait。若需要對中介資料表記錄進行軟刪除，請考慮將中介 Model 轉換為一個真正的 Eloquent Model。

<a name="custom-pivot-models-and-incrementing-ids"></a>
#### 自訂中介 Model 與遞增 ID

若已定義了使用自訂中介 Model 的多對多關聯，且該中介 Model 有一個自動遞增的主鍵，則應確保自訂中介 Model 的 Class 中有定義一個設為 `true` 的 `incrementing` 屬性。

```php
/**
 * Indicates if the IDs are auto-incrementing.
 *
 * @var bool
 */
public $incrementing = true;
```

<a name="polymorphic-relationships"></a>
## 多態關聯

多態關聯允許子 Model 使用單一關聯來屬於多種類型的 Model。舉例來說，假設你正在開發一個能讓使用者分享部落格文章與影片的應用程式。在這樣的應用程式中，一個 `Comment` Model 可能會同時屬於 `Post` 與 `Video` Model。

<a name="one-to-one-polymorphic-relations"></a>
### 一對一

<a name="one-to-one-polymorphic-table-structure"></a>
#### 資料表結構

一對一多態關聯與一般的一對一關聯類似；不過，子 Model 可使用單一關聯來屬於多種類型的 Model。舉例來說，一篇部落格的 `Post` 與一個 `User` Model 可以共享一個與 `Image` Model 的多態關聯。使用一對一多態關聯可以讓我們用單一資料表來儲存可關聯至文章與使用者的獨立圖片。首先，我們先來看看資料表結構：

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

請注意 `images` 資料表上的 `imageable_id` 與 `imageable_type` 欄位。`imageable_id` 欄位會包含文章或使用者的 ID 值，而 `imageable_type` 欄位則會包含父 Model 的類別名稱。Eloquent 會使用 `imageable_type` 欄位來判斷在存取 `imageable` 關聯時要回傳哪個「類型」的父 Model。在這個例子中，該欄位會包含 `App\Models\Post` 或 `App\Models\User`。

<a name="one-to-one-polymorphic-model-structure"></a>
<h4>Model 結構</h4>

接著，我們來看看要建立此關聯所需的 Model 定義：

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

在定義好資料表與 Model 後，就可以透過 Model 來存取關聯。舉例來說，若要取得一篇文章的圖片，我們可以存取 `image` 動態關聯屬性：

```php
use App\Models\Post;

$post = Post::find(1);

$image = $post->image;
```

我們可以藉由存取呼叫 `morphTo` 的方法名稱來從多態 Model 中取得父項。在這個例子中，也就是 `Image` Model 上的 `imageable` 方法。因此，我們可以將該方法作為動態關聯屬性來存取：

```php
use App\Models\Image;

$image = Image::find(1);

$imageable = $image->imageable;
```

`Image` Model 上的 `imageable` 關聯會回傳 `Post` 或 `User` 的實體，取決於是哪種類型的 Model 擁有該圖片。

<a name="morph-one-to-one-key-conventions"></a>
#### 主鍵慣例

若有需要，可以為多態子 Model 指定要使用的「id」與「type」欄位。若這麼做，請確保總是在 `morphTo` 方法的第一個引數中傳入關聯的名稱。一般來說，這個值應符合方法名稱，因此可以使用 PHP 的 `__FUNCTION__` 常數：

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

「一對多」的多態關聯與一般的「一對多」關聯很像；不過，子 Model 可使用單一關聯屬於多種不同型別的 Model。舉例來說，假設某個應用程式中的使用者可以「留言 (Comment)」給文章 (Post) 與影片 (Video)。使用多態關聯時，我們就可以用單一個 `comments` 資料表來儲存給文章與影片的留言。首先，我們先來看看要建立這個關聯所需的資料表結構：

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
#### Model 結構

接著，我們來看看要建立這個關聯所需的 Model 定義：

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

定義好資料表與 Model 後，就可以透過 Model 上的動態關聯屬性來存取關聯。舉例來說，若要存取某篇文章的所有留言，可使用 `comments` 動態屬性：

```php
use App\Models\Post;

$post = Post::find(1);

foreach ($post->comments as $comment) {
    // ...
}
```

我們也可以透過存取呼叫 `morphTo` 的那個方法名稱來從多態子 Model 上取得父 Model。在這個例子中，就是 `Comment` Model 上的 `commentable` 方法。因此，我們會將該方法作為動態關聯屬性來存取，以取得留言的父 Model：

```php
use App\Models\Comment;

$comment = Comment::find(1);

$commentable = $comment->commentable;
```

`Comment` Model 上的 `commentable` 關聯會回傳 `Post` 或 `Video` 的實體，取決於是哪種型別的 Model 擁有該留言。


<a name="polymorphic-automatically-hydrating-parent-models-on-children"></a>
#### 自動在子 Model 上 Hydrate 父 Model

就算使用 Eloquent 的預先載入，若在遍歷子 Model 的迴圈中嘗試存取父 Model，還是可能會出現「N + 1」查詢問題：

```php
$posts = Post::with('comments')->get();

foreach ($posts as $post) {
    foreach ($post->comments as $comment) {
        echo $comment->commentable->title;
    }
}
```

在上面的範例中，會出現「N + 1」查詢問題。這是因為，就算我們為每個 `Post` Model 預先載入了其留言，Eloquent 也不會在每個子 `Comment` Model 上自動 Hydrate (填充) 其父 `Post`。

若想讓 Eloquent 自動將父 Model Hydrate 到其子 Model 上，可以在定義 `morphMany` 關聯時叫用 `chaperone` 方法：

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

或者，若想在執行時才選擇性地啟用自動父項 Hydration，可以在預先載入關聯時叫用 `chaperone` Model：

```php
use App\Models\Post;

$posts = Post::with([
    'comments' => fn ($comments) => $comments->chaperone(),
])->get();
```


<a name="one-of-many-polymorphic-relations"></a>
### 一對多中的其一

有時候一個 Model 可能會有多個關聯 Model，但我們只想要輕鬆地取得關聯中「最新」或「最舊」的關聯 Model。舉例來說，一個 `User` Model 可能會關聯到多個 `Image` Model，但我們想定義一個方便的方法來操作該使用者最近上傳的圖片。我們可以使用 `morphOne` 關聯型別並搭配 `ofMany` 系列方法來達成此目標：

```php
/**
 * Get the user's most recent image.
 */
public function latestImage(): MorphOne
{
    return $this->morphOne(Image::class, 'imageable')->latestOfMany();
}
```

同樣地，我們也可以定義一個方法來取得關聯中「最舊」(也就是第一個) 的關聯 Model：

```php
/**
 * Get the user's oldest image.
 */
public function oldestImage(): MorphOne
{
    return $this->morphOne(Image::class, 'imageable')->oldestOfMany();
}
```

預設情況下，`latestOfMany` 與 `oldestOfMany` 方法會根據 Model 中可排序的主鍵來取得最新或最舊的關聯 Model。不過，有時候我們可能會想用不同的排序條件來從一個較大的關聯中取得單一 Model。

舉例來說，使用 `ofMany` 方法，我們可以取得使用者「最受歡迎 (liked)」的圖片。`ofMany` 方法的第一個引數為可排序的欄位，第二個引數則為查詢關聯 Model 時要套用的聚合函式 (`min` 或 `max`)：

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
> 可以建立更進階的「一對多中的其一」關聯。更多資訊請參考 [has one of many 文件的說明](#advanced-has-one-of-many-relationships)。

<a name="many-to-many-polymorphic-relations"></a>
### 多對多 (Polymorphic)


<a name="many-to-many-polymorphic-table-structure"></a>
#### 資料表結構

多態多對多關聯比「Morph One」與「Morph Many」關聯要稍微複雜一些。舉例來說，一個 `Post` Model 與一個 `Video` Model 可以共享與 `Tag` Model 的多態關聯。在這種情況下使用多態多對多關聯，可以讓你的應用程式使用單一的唯一標籤資料表，並將這些標籤關聯到文章或影片上。首先，我們先來看看建立這個關聯所需的資料表結構：

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
> 在深入瞭解多態多對多關聯前，建議先閱讀有關一般 [多對多關聯](#many-to-many) 的說明文件。


<a name="many-to-many-polymorphic-model-structure"></a>
#### Model 結構

接著，我們就可以來在 Model 上定義關聯了。`Post` 與 `Video` Model 都會包含一個 `tags` 方法，該方法會呼叫基礎 Eloquent Model Class 所提供的 `morphToMany` 方法。

`morphToMany` 方法的第一個引數為關聯 Model 的名稱，第二個引數則為「關聯名稱」。根據我們為中介資料表指定的名稱以及其中包含的索引鍵，我們將這個關聯稱為「taggable」：

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

接著，在 `Tag` Model 上，我們應為每個可能的父項 Model 都定義一個方法。因此，在這個範例中，我們會定義一個 `posts` 方法與一個 `videos` 方法。這兩個方法都應回傳 `morphedByMany` 方法的結果。

`morphedByMany` 方法的第一個引數為關聯 Model 的名稱，第二個引數則為「關聯名稱」。根據我們為中介資料表指定的名稱以及其中包含的索引鍵，我們將這個關聯稱為「taggable」：

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

定義好資料表與 Model 後，就可以透過 Model 來存取關聯了。舉例來說，若要存取某篇文章的所有標籤，可以使用 `tags` 動態關聯屬性：

```php
use App\Models\Post;

$post = Post::find(1);

foreach ($post->tags as $tag) {
    // ...
}
```

我們也可以透過存取呼叫 `morphedByMany` 的方法名稱，來從多態子項 Model 中取得多態關聯的父項。在這個例子中，這個方法就是 `Tag` Model 上的 `posts` 或 `videos` 方法：

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
### 自訂多態型別

預設情況下，Laravel 會使用完整的 Class 名稱來儲存關聯 Model 的「型別」。舉例來說，在上面的一對多關聯範例中，`Comment` Model 可能屬於 `Post` 或 `Video` Model，因此預設的 `commentable_type` 會分別是 `App\Models\Post` 或 `App\Models\Video`。不過，有時候我們可能會想讓這些值與應用程式的內部結構脫鉤。

舉例來說，我們可以使用 `post` 或 `video` 這種簡單的字串來取代 Model 名稱作為「型別」。這麼一來，就算 Model 被重新命名了，資料庫中的多態「型別」欄位值也仍然有效：

```php
use Illuminate\Database\Eloquent\Relations\Relation;

Relation::enforceMorphMap([
    'post' => 'App\Models\Post',
    'video' => 'App\Models\Video',
]);
```

我們可以在 `App\Providers\AppServiceProvider` Class 的 `boot` 方法中呼叫 `enforceMorphMap` 方法，或也可以在需要時建立一個獨立的 Service Provider。

在執行階段時，可以使用 Model 的 `getMorphClass` 方法來判斷給定 Model 的 Morph 別名。反過來說，也可以使用 `Relation::getMorphedModel` 方法來判斷與某個 Morph 別名相關的完整 Class 名稱：

```php
use Illuminate\Database\Eloquent\Relations\Relation;

$alias = $post->getMorphClass();

$class = Relation::getMorphedModel($alias);
```

> [!WARNING]
> 在現有專案中新增「Morph Map」時，資料庫中所有仍包含完整 Class 名稱的可多態 `*_type` 欄位值都必須被轉換為其「Map」名稱。


<a name="dynamic-relationships"></a>
### 動態關聯

我們可以使用 `resolveRelationUsing` 方法在執行階段定義 Eloquent Model 之間的關聯。雖然在一般的應用程式開發中不建議這麼做，但在開發 Laravel 套件時偶爾會很有用。

`resolveRelationUsing` 方法的第一個引數為想要的關聯名稱。傳給該方法的第二個引數則應為一個閉包，該閉包會接收 Model 實體並回傳一個有效的 Eloquent 關聯定義。一般來說，我們應在 [Service Provider](/docs/{{version}}/providers) 的 boot 方法中設定動態關聯：

```php
use App\Models\Order;
use App\Models\Customer;

Order::resolveRelationUsing('customer', function (Order $orderModel) {
    return $orderModel->belongsTo(Customer::class, 'customer_id');
});
```

> [!WARNING]
> 定義動態關聯時，請務必為 Eloquent 關聯方法提供明確的索引鍵名稱引數。

<a name="querying-relations"></a>
## 查詢關聯

由於所有的 Eloquent 關聯都是透過方法定義的，因此我們可以呼叫這些方法來取得關聯的實體，而不需要實際執行查詢來載入關聯的 Model。此外，所有 Eloquent 關聯的類型也都可作為[查詢產生器](/docs/{{version}}/queries)使用，讓我們可以在最終對資料庫執行 SQL 查詢前，繼續在關聯查詢上鏈式呼叫並加上條件約束。

舉例來說，假設有個部落格應用程式，其中 `User` Model 有許多關聯的 `Post` Model：

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

我們可以像這樣查詢 `posts` 關聯並為其加上額外的條件約束：

```php
use App\Models\User;

$user = User::find(1);

$user->posts()->where('active', 1)->get();
```

你可以在關聯上使用任何 Laravel [查詢產生器](/docs/{{version}}/queries)的方法，所以請務必詳閱查詢產生器的說明文件來瞭解所有可用的方法。

<a name="chaining-orwhere-clauses-after-relationships"></a>
#### 在關聯後鏈式呼叫 `orWhere` 子句

如上例所示，在查詢關聯時，我們可以自由地為關聯加上額外的條件約束。不過，在關聯上鏈式呼叫 `orWhere` 子句時要小心，因為 `orWhere` 子句在邏輯上會與關聯的條件約束在同一層級上分組：

```php
$user->posts()
    ->where('active', 1)
    ->orWhere('votes', '>=', 100)
    ->get();
```

上例會產生下列的 SQL。如你所見，`or` 子句會讓查詢回傳 _任何_ 票數大於 100 的文章。這個查詢已不再被限制於特定使用者：

```sql
select *
from posts
where user_id = ? and active = 1 or votes >= 100
```

在大多數情況下，應使用[邏輯分組](/docs/{{version}}/queries#logical-grouping)來將條件檢查放在括號中：

```php
use Illuminate\Database\Eloquent\Builder;

$user->posts()
    ->where(function (Builder $query) {
        return $query->where('active', 1)
            ->orWhere('votes', '>=', 100);
    })
    ->get();
```

上例會產生下列的 SQL。請注意，邏輯分組已將條件約束正確地分組，且查詢仍被限制於特定使用者：

```sql
select *
from posts
where user_id = ? and (active = 1 or votes >= 100)
```

<a name="relationship-methods-vs-dynamic-properties"></a>
### 關聯方法 vs. 動態屬性

若不需要為 Eloquent 關聯查詢加上額外的條件約束，我們可以像存取屬性一樣存取關聯。舉例來說，繼續使用我們的 `User` 與 `Post` 範例 Model，我們可以像這樣存取某個使用者的所有文章：

```php
use App\Models\User;

$user = User::find(1);

foreach ($user->posts as $post) {
    // ...
}
```

動態關聯屬性會執行「延遲載入 (Lazy Loading)」，代表只有在實際存取到關聯資料時才會載入。因此，開發人員通常會使用[預先載入 (Eager Loading)](#eager-loading) 來預先載入他們知道在載入 Model 後會存取到的關聯。預先載入能大幅減少載入 Model 關聯時必須執行的 SQL 查詢。

<a name="querying-relationship-existence"></a>
### 查詢關聯的存在

在擷取 Model 紀錄時，我們可能希望根據關聯的存在與否來限制結果。舉例來說，假設我們想擷取所有至少有一則留言的部落格文章。為此，我們可以將關聯的名稱傳給 `has` 與 `orHas` 方法：

```php
use App\Models\Post;

// Retrieve all posts that have at least one comment...
$posts = Post::has('comments')->get();
```

我們也可以指定運算子與計數值來進一步自訂查詢：

```php
// Retrieve all posts that have three or more comments...
$posts = Post::has('comments', '>=', 3)->get();
```

巢狀的 `has` 陳述式可使用「點 (.)」標記法來建構。舉例來說，我們可以擷取所有至少有一則留言，且該留言至少有一張圖片的文章：

```php
// Retrieve posts that have at least one comment with images...
$posts = Post::has('comments.images')->get();
```

若需要更強大的功能，我們可以使用 `whereHas` 與 `orWhereHas` 方法來為 `has` 查詢定義額外的查詢條件約束，例如檢查留言的內容：

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
> Eloquent 目前不支援跨資料庫查詢關聯的存在。關聯必須存在於同一個資料庫內。

<a name="many-to-many-relationship-existence-queries"></a>
#### 多對多關聯存在查詢

`whereAttachedTo` 方法可用於查詢與某個 Model 或 Model 集合有多對多附件的 Model：

```php
$users = User::whereAttachedTo($role)->get();
```

你也可以提供一個 [collection](/docs/{{version}}/eloquent-collections) 實體給 `whereAttachedTo` 方法。這麼做時，Laravel 將會擷取與集合中任何一個 Model 有關聯的 Model：

```php
$tags = Tag::whereLike('name', '%laravel%')->get();

$posts = Post::whereAttachedTo($tags)->get();
```

<a name="inline-relationship-existence-queries"></a>
#### 行內關聯存在查詢

如果你想用單一、簡單的 where 條件來查詢關聯的存在，你可能會覺得使用 `whereRelation`、`orWhereRelation`、`whereMorphRelation` 與 `orWhereMorphRelation` 方法更為方便。舉例來說，我們可以查詢所有有未經審核留言的文章：

```php
use App\Models\Post;

$posts = Post::whereRelation('comments', 'is_approved', false)->get();
```

當然，就像查詢產生器的 `where` 方法一樣，你也可以指定一個運算子：

```php
$posts = Post::whereRelation(
    'comments', 'created_at', '>=', now()->subHour()
)->get();
```

<a name="querying-relationship-absence"></a>
### 查詢關聯的不存在

在擷取 Model 紀錄時，我們可能希望根據關聯的不存在來限制結果。舉例來說，假設我們想擷取所有**沒有**任何留言的部落格文章。為此，我們可以將關聯的名稱傳給 `doesntHave` 與 `orDoesntHave` 方法：

```php
use App\Models\Post;

$posts = Post::doesntHave('comments')->get();
```

若需要更強大的功能，我們可以使用 `whereDoesntHave` 與 `orWhereDoesntHave` 方法來為 `doesntHave` 查詢加上額外的查詢條件約束，例如檢查留言的內容：

```php
use Illuminate\Database\Eloquent\Builder;

$posts = Post::whereDoesntHave('comments', function (Builder $query) {
    $query->where('content', 'like', 'code%');
})->get();
```

我們可以使用「點 (.)」標記法來對巢狀關聯執行查詢。舉例來說，下列查詢會擷取所有沒有留言的文章，以及有留言但留言者都不是被封鎖使用者的文章：

```php
use Illuminate\Database\Eloquent\Builder;

$posts = Post::whereDoesntHave('comments.author', function (Builder $query) {
    $query->where('banned', 1);
})->get();
```

<a name="querying-morph-to-relationships"></a>
### 查詢 Morph To 關聯

若要查詢「morph to」關聯是否存在，可使用 `whereHasMorph` 與 `whereDoesntHaveMorph` 方法。這些方法的第一個引數為關聯的名稱。接著，這些方法的第二個引數則為要包含在查詢內的關聯 Model 名稱。最後，還可以提供一個閉包來自訂關聯查詢：

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

有時候，我們可能會需要根據多態關聯 Model 的「型別」來新增查詢限制。傳給 `whereHasMorph` 方法的閉包可接收 `$type` 值作為其第二個引數。此引數可讓我們檢查正在建構的查詢的「型別」：

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

有時候，我們可能會想查詢某個「morph to」關聯的父項的子項。我們可以使用 `whereMorphedTo` 與 `whereNotMorphedTo` 方法來達成。這兩個方法會自動為給定的 Model 判斷正確的多態型別對應。這些方法會接收 `morphTo` 關聯的名稱作為其第一個引數，並接收關聯的父 Model 作為其第二個引數：

```php
$comments = Comment::whereMorphedTo('commentable', $post)
    ->orWhereMorphedTo('commentable', $video)
    ->get();
```


<a name="querying-all-morph-to-related-models"></a>
#### 查詢所有關聯 Model

除了傳遞一個包含可能的多態 Model 的陣列外，我們也可以提供 `*` 作為萬用字元值。這會讓 Laravel 從資料庫中取得所有可能的多態型別。Laravel 會執行一個額外的查詢來執行此操作：

```php
use Illuminate\Database\Eloquent\Builder;

$comments = Comment::whereHasMorph('commentable', '*', function (Builder $query) {
    $query->where('title', 'like', 'foo%');
})->get();
```

<a name="aggregating-related-models"></a>
## 聚合關聯 Model


<a name="counting-related-models"></a>
### 計數關聯 Model

有時候，我們可能會想在不實際載入 Model 的情況下，計算某個關聯的關聯 Model 數量。為此，可使用 `withCount` 方法。`withCount` 方法會在產生的 Model 上加上一個 `{relation}_count` 屬性：

```php
use App\Models\Post;

$posts = Post::withCount('comments')->get();

foreach ($posts as $post) {
    echo $post->comments_count;
}
```

`withCount` 方法也接受傳入一個陣列，藉此可為多個關聯加上「計數」，並為查詢加上額外的限制：

```php
use Illuminate\Database\Eloquent\Builder;

$posts = Post::withCount(['votes', 'comments' => function (Builder $query) {
    $query->where('content', 'like', 'code%');
}])->get();

echo $posts[0]->votes_count;
echo $posts[0]->comments_count;
```

我們也可以為關聯計數結果設定別名 (Alias)，以在同一個關聯上進行多次計數：

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

使用 `loadCount` 方法，就可以在父項 Model 已被擷取後，才載入關聯計數：

```php
$book = Book::first();

$book->loadCount('genres');
```

若需要為計數查詢設定額外的查詢限制，可以傳入一個以要計數的關聯為索引鍵的陣列。該陣列的值應為一個會收到查詢產生器實體的閉包：

```php
$book->loadCount(['reviews' => function (Builder $query) {
    $query->where('rating', 5);
}])
```


<a name="relationship-counting-and-custom-select-statements"></a>
#### 關聯計數與自訂 Select 陳述式

若要將 `withCount` 與 `select` 陳述式合併使用，請確保在 `select` 方法後才呼叫 `withCount`：

```php
$posts = Post::select(['title', 'body'])
    ->withCount('comments')
    ->get();
```


<a name="other-aggregate-functions"></a>
### 其他聚合函式

除了 `withCount` 方法外，Eloquent 還提供了 `withMin`、`withMax`、`withAvg`、`withSum`、與 `withExists` 等方法。這些方法會在產生的 Model 上加上一個 `{relation}_{function}_{column}` 屬性：

```php
use App\Models\Post;

$posts = Post::withSum('comments', 'votes')->get();

foreach ($posts as $post) {
    echo $post->comments_sum_votes;
}
```

若想用另一個名稱來存取聚合函式的結果，可以指定自己的別名：

```php
$posts = Post::withSum('comments as total_comments', 'votes')->get();

foreach ($posts as $post) {
    echo $post->total_comments;
}
```

與 `loadCount` 方法類似，這些方法也提供延遲的版本。這些額外的聚合操作可在已擷取的 Eloquent Model 上執行：

```php
$post = Post::first();

$post->loadSum('comments', 'votes');
```

若要將這些聚合方法與 `select` 陳述式合併使用，請確保在 `select` 方法後才呼叫這些聚合方法：

```php
$posts = Post::select(['title', 'body'])
    ->withExists('comments')
    ->get();
```


<a name="counting-related-models-on-morph-to-relationships"></a>
### 在 Morph To 關聯上計數關聯 Model

若想預先載入 (Eager Load)「Morph To」關聯，並為該關聯可能回傳的各個實體取得關聯 Model 計數，可使用 `with` 方法並搭配 `morphTo` 關聯的 `morphWithCount` 方法。

在此範例中，我們假設 `Photo` 與 `Post` Model 可用來建立 `ActivityFeed` Model。我們會假設 `ActivityFeed` Model 定義了一個名為 `parentable` 的「Morph To」關聯，讓我們能為給定的 `ActivityFeed` 實體擷取其父項 `Photo` 或 `Post` Model。此外，我們也假設 `Photo` Model「擁有多個 (Have Many)」`Tag` Model，而 `Post` Model 則「擁有多個」`Comment` Model。

現在，假設我們要擷取 `ActivityFeed` 實體，並為每個 `ActivityFeed` 實體預先載入其 `parentable` 父項 Model。此外，我們還想擷取與每個父項照片關聯的標籤數量，以及與每個父項文章關聯的留言數量：

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

假設我們已經擷取了一組 `ActivityFeed` Model，而現在想為與這些活動饋給 (Activity Feed) 關聯的各個 `parentable` Model 載入其巢狀的關聯計數。可使用 `loadMorphCount` 方法來達成：

```php
$activities = ActivityFeed::with('parentable')->get();

$activities->loadMorphCount('parentable', [
    Photo::class => ['tags'],
    Post::class => ['comments'],
]);
```

<a name="eager-loading"></a>
## 預先載入 (Eager Loading)

當將 Eloquent 關聯當作屬性來存取時，關聯 Model 會被「延遲載入 (lazy loaded)」。這代表關聯資料並不會在存取該屬性前就載入。不過，Eloquent 可以在查詢父項 Model 時「預先載入 (eager load)」關聯。預先載入可以緩解「N + 1」查詢問題。為了說明 N + 1 查詢問題，我們假設有個 `Book` Model「屬於 (belongs to)」一個 `Author` Model：

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

現在，來取得所有的書與其作者：

```php
use App\Models\Book;

$books = Book::all();

foreach ($books as $book) {
    echo $book->author->name;
}
```

這個迴圈會先執行一個查詢來從資料庫資料表中取得所有的書，接著，每一本書都會再執行一個查詢來取得該書的作者。所以，若我們有 25 本書，上面的程式碼就會執行 26 個查詢：一個查詢用來取得原本的書，而另外 25 個查詢則用來取得每一本書的作者。

幸好，我們可以使用預先載入來將這個操作降到只有兩個查詢。在建立查詢時，可以使用 `with` 方法來指定應預先載入哪些關聯：

```php
$books = Book::with('author')->get();

foreach ($books as $book) {
    echo $book->author->name;
}
```

對於這個操作，只會執行兩個查詢 —— 一個查詢用來取得所有的書，另一個查詢則用來取得所有這些書的作者：

```sql
select * from books

select * from authors where id in (1, 2, 3, 4, 5, ...)
```

<a name="eager-loading-multiple-relationships"></a>
#### 預先載入多個關聯

有時候，可能需要預先載入數個不同的關聯。若要這麼做，只需要傳遞一個包含關聯的陣列給 `with` 方法：

```php
$books = Book::with(['author', 'publisher'])->get();
```

<a name="nested-eager-loading"></a>
#### 巢狀預先載入

若要預先載入某個關聯的關聯，可以使用「點 (.)」語法。舉例來說，我們來預先載入所有書的作者，以及所有作者的個人聯絡人：

```php
$books = Book::with('author.contacts')->get();
```

或者，我們也可以提供一個巢狀陣列給 `with` 方法來指定巢狀預先載入的關聯，在預先載入多個巢狀關聯時會很方便：

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

若想預先載入一個 `morphTo` 關聯、以及該關聯可能回傳的不同實體上的巢狀關聯，可以使用 `with` 方法搭配 `morphTo` 關聯的 `morphWith` 方法。為了說明這個方法，我們先來看看下列這個 Model：

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

在這個範例中，我們假設 `Event`、`Photo`、與 `Post` Model 可以建立 `ActivityFeed` Model。此外，我們也假設 `Event` Model 屬於 `Calendar` Model、`Photo` Model 則與 `Tag` Model 相關聯、而 `Post` Model 則屬於 `Author` Model。

有了這些 Model 定義與關聯，我們就可以取得 `ActivityFeed` Model 的實體，並預先載入所有的 `parentable` Model 與其各自的巢狀關聯：

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

有時候，我們不一定需要從關聯中取得所有欄位。為此，Eloquent 允許我們指定想從關聯中取得哪些欄位：

```php
$books = Book::with('author:id,name,book_id')->get();
```

> [!WARNING]
> 使用此功能時，應總是在想取得的欄位列表中包含 `id` 欄位與任何相關的外鍵欄位。

<a name="eager-loading-by-default"></a>
#### 預設預先載入

有時候，我們可能會想在取得 Model 時總是載入某些關聯。若要這麼做，可以在 Model 上定義一個 `$with` 屬性：

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

若想在單一查詢中從 `$with` 屬性移除某個項目，可以使用 `without` 方法：

```php
$books = Book::without('author')->get();
```

若想在單一查詢中覆寫 `$with` 屬性內的所有項目，可以使用 `withOnly` 方法：

```php
$books = Book::withOnly('genre')->get();
```

<a name="constraining-eager-loads"></a>
### 限制預先載入

有時候，你可能會想預先載入某個關聯，但同時也為該預先載入查詢指定額外的查詢條件。要達成這個目的，可以傳遞一個關聯陣列到 `with` 方法中。其中，陣列的鍵為關聯名稱，而陣列的值則為一個閉包 (Closure)，用來為預先載入查詢增加額外限制：

```php
use App\Models\User;
use Illuminate\Database\Eloquent\Builder;

$users = User::with(['posts' => function (Builder $query) {
    $query->where('title', 'like', '%code%');
}])->get();
```

在此範例中，Eloquent 只會預先載入文章 `title` 欄位中包含 `code` 這個字的貼文。你也可以呼叫其他 [查詢產生器](/docs/{{version}}/queries) 的方法來進一步自訂預先載入的操作：

```php
$users = User::with(['posts' => function (Builder $query) {
    $query->orderBy('created_at', 'desc');
}])->get();
```


<a name="constraining-eager-loading-of-morph-to-relationships"></a>
#### 限制 `morphTo` 關聯的預先載入

若要預先載入 `morphTo` 關聯，Eloquent 會執行多個查詢來擷取各種類型的關聯 Model。可使用 `MorphTo` 關聯的 `constrain` 方法來為這些查詢加上額外的限制：

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

在此範例中，Eloquent 只會預先載入尚未被隱藏的貼文，以及 `type` 值為「educational」的影片。


<a name="constraining-eager-loads-with-relationship-existence"></a>
#### 透過關聯存在限制預先載入

有時候，你可能會需要在檢查某個關聯是否存在的同時，又根據相同的條件來載入該關聯。舉例來說，你可能只想擷取帶有符合特定查詢條件的子 `Post` Model 的 `User` Model，並同時預先載入這些符合條件的貼文。要達成這個目的，可使用 `withWhereHas` 方法：

```php
use App\Models\User;

$users = User::withWhereHas('posts', function ($query) {
    $query->where('featured', true);
})->get();
```


<a name="lazy-eager-loading"></a>
### 延遲預先載入 (Lazy Eager Loading)

有時候，你可能會需要在已擷取到父 Model 後才需要預先載入關聯。舉例來說，當你需要動態決定是否載入關聯 Model 時，這個功能就很有用：

```php
use App\Models\Book;

$books = Book::all();

if ($condition) {
    $books->load('author', 'publisher');
}
```

若想為預先載入查詢設定額外的查詢限制，可以傳遞一個以你想載入的關聯為鍵的陣列。陣列的值應為一個會收到查詢實體的閉包實體：

```php
$author->load(['books' => function (Builder $query) {
    $query->orderBy('published_date', 'asc');
}]);
```

若只要在關聯尚未載入時才載入關聯，請使用 `loadMissing` 方法：

```php
$book->loadMissing('author');
```


<a name="nested-lazy-eager-loading-morphto"></a>
#### 巢狀延遲預先載入與 `morphTo`

若想預先載入 `morphTo` 關聯，以及該關聯可能回傳的各種實體上的巢狀關聯，可使用 `loadMorph` 方法。

此方法的第一個引數為 `morphTo` 關聯的名稱，第二個引數則為一個包含 Model / 關聯配對的陣列。為了說明這個方法，讓我們先來看看下列 Model：

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

在此範例中，我們假設 `Event`、`Photo`、`Post` 這幾個 Model 都可以建立 `ActivityFeed` Model。此外，我們也假設 `Event` Model 屬於 `Calendar` Model、`Photo` Model 與 `Tag` Model 關聯、而 `Post` Model 則屬於 `Author` Model。

透過這些 Model 定義與關聯，我們就可以擷取 `ActivityFeed` Model 的實體，並預先載入所有的 `parentable` Model 與其各自的巢狀關聯：

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

> [!WARNING]
> This feature is currently in beta in order to gather community feedback. The behavior and functionality of this feature may change even on patch releases.

在許多情況下，Laravel 可以自動預先載入你存取的關聯。若要啟用自動預先載入，應在應用程式的 `AppServiceProvider` 的 `boot` 方法中叫用 `Model::automaticallyEagerLoadRelationships` 方法：

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

啟用此功能後，Laravel 會在你存取任何先前未載入的關聯時，嘗試自動載入這些關聯。舉例來說，請參考下列情境：

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

一般來說，上述程式碼會為每個使用者執行一次查詢來擷取其貼文，並為每個貼文執行一次查詢來擷取其留言。不過，當 `automaticallyEagerLoadRelationships` 功能啟用後，在你嘗試存取任何一個已擷取使用者的貼文時，Laravel 就會自動為使用者集合中的所有使用者 [延遲預先載入](#lazy-eager-loading) 貼文。同樣地，當你嘗試存取任何已擷取貼文的留言時，所有原先擷取的貼文的留言也會被延遲預先載入。

若不想全域啟用自動預先載入，還是可以透過在 Eloquent 集合實體上叫用 `withRelationshipAutoloading` 方法，來為單一集合啟用此功能：

```php
$users = User::where('vip', true)->get();

return $users->withRelationshipAutoloading();
```

<a name="preventing-lazy-loading"></a>
### 避免延遲載入 (Lazy Loading)

如先前所述，預先載入關聯通常能為應用程式帶來顯著的效能提升。因此，若有需要，可以讓 Laravel 總是避免延遲載入關聯。為此，可以叫用基礎 Eloquent Model 類別提供的 `preventLazyLoading` 方法。一般來說，應在應用程式的 `AppServiceProvider` 類別中的 `boot` 方法內叫用此方法。

`preventLazyLoading` 方法可接受一個選用的布林值引數，用以表示是否應避免延遲載入。舉例來說，我們可能會想只在非正式版 (non-production) 環境中停用延遲載入，這樣一來，就算正式版程式碼中不小心出現了延遲載入的關聯，正式版環境還是能正常運作：

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

避免了延遲載入後，當應用程式嘗試延遲載入任何 Eloquent 關聯時，Eloquent 就會擲回一個 `Illuminate\Database\LazyLoadingViolationException` 例外。

我們可以使用 `handleLazyLoadingViolationsUsing` 方法來自訂延遲載入違規時的行為。舉例來說，使用此方法，我們可以讓延遲載入違規只被記錄下來，而不是用例外中斷應用程式的執行：

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

Eloquent 提供了方便的方法來將新的 Model 新增到關聯中。舉例來說，假設我們需要為一篇文章新增一則留言。與其手動在 `Comment` Model 上設定 `post_id` 屬性，我們可以直接用關聯的 `save` 方法來新增留言：

```php
use App\Models\Comment;
use App\Models\Post;

$comment = new Comment(['message' => 'A new comment.']);

$post = Post::find(1);

$post->comments()->save($comment);
```

請注意，我們不是以動態屬性來存取 `comments` 關聯。而是呼叫 `comments` 方法來取得關聯的實體。`save` 方法會自動將合適的 `post_id` 值加到新的 `Comment` Model 中。

若需要儲存多個關聯 Model，可使用 `saveMany` 方法：

```php
$post = Post::find(1);

$post->comments()->saveMany([
    new Comment(['message' => 'A new comment.']),
    new Comment(['message' => 'Another new comment.']),
]);
```

`save` 與 `saveMany` 方法會將給定的 Model 實體 Persist (儲存) 起來，但不會將新建的 Model 加到父項 Model 上已載入的記憶體內關聯中。若打算在 `save` 或 `saveMany` 方法後存取關聯，則可以使用 `refresh` 方法來重新載入 Model 與其關聯：

```php
$post->comments()->save($comment);

$post->refresh();

// All comments, including the newly saved comment...
$post->comments;
```


<a name="the-push-method"></a>
#### 遞迴式儲存 Model 與關聯

若想 `save` (儲存) Model 及其所有關聯，可使用 `push` 方法。在本範例中，`Post` Model 及其留言、留言的作者都會被儲存：

```php
$post = Post::find(1);

$post->comments[0]->message = 'Message';
$post->comments[0]->author->name = 'Author Name';

$post->push();
```

`pushQuietly` 方法可用於儲存 Model 及其關聯，且不觸發任何事件：

```php
$post->pushQuietly();
```


<a name="the-create-method"></a>
### `create` 方法

除了 `save` 與 `saveMany` 方法外，你還可以使用 `create` 方法，該方法接受一個屬性陣列、建立一個 Model、並將其插入資料庫。`save` 與 `create` 的不同之處在於，`save` 接受一個完整的 Eloquent Model 實體，而 `create` 則接受一個普通的 PHP `array`。`create` 方法會回傳新建的 Model：

```php
use App\Models\Post;

$post = Post::find(1);

$comment = $post->comments()->create([
    'message' => 'A new comment.',
]);
```

可使用 `createMany` 方法來建立多個關聯 Model：

```php
$post = Post::find(1);

$post->comments()->createMany([
    ['message' => 'A new comment.'],
    ['message' => 'Another new comment.'],
]);
```

`createQuietly` 與 `createManyQuietly` 方法可用於建立一或多個 Model 且不分派任何事件：

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

你也可以使用 `findOrNew`、`firstOrNew`、`firstOrCreate` 與 `updateOrCreate` 方法來[在關聯上建立與更新 Model](/docs/{{version}}/eloquent#upserts)。

> [!NOTE]
> 在使用 `create` 方法前，請務必先閱讀[批量指派](/docs/{{version}}/eloquent#mass-assignment)的說明文件。


<a name="updating-belongs-to-relationships"></a>
### Belongs To 關聯

若想將子 Model 指派給新的父項 Model，可使用 `associate` 方法。在本範例中，`User` Model 定義了一個與 `Account` Model 的 `belongsTo` 關聯。`associate` 方法會在子 Model 上設定外鍵：

```php
use App\Models\Account;

$account = Account::find(10);

$user->account()->associate($account);

$user->save();
```

若要從子 Model 上移除父項 Model，可使用 `dissociate` 方法。該方法會將關聯的外鍵設為 `null`：

```php
$user->account()->dissociate();

$user->save();
```


<a name="updating-many-to-many-relationships"></a>
### 多對多關聯


<a name="attaching-detaching"></a>
#### 附加 (Attaching) / 分離 (Detaching)

Eloquent 也提供了一些方法，讓操作多對多關聯更為方便。舉例來說，假設一個使用者可以有多個角色，而一個角色可以屬於多個使用者。我們可以使用 `attach` 方法來將一個角色附加給使用者，這麼做會在關聯的中介資料表內新增一筆紀錄：

```php
use App\Models\User;

$user = User::find(1);

$user->roles()->attach($roleId);
```

將關聯附加到 Model 上時，也可以傳入一組要插入到中介資料表中的額外資料陣列：

```php
$user->roles()->attach($roleId, ['expires' => $expires]);
```

有時候，我們也需要從使用者身上移除某個角色。若要移除一筆多對多關聯的紀錄，可使用 `detach` 方法。`detach` 方法會從中介資料表中刪除對應的紀錄；不過，兩個 Model 都會保留在資料庫中：

```php
// Detach a single role from the user...
$user->roles()->detach($roleId);

// Detach all roles from the user...
$user->roles()->detach();
```

為了方便起見，`attach` 與 `detach` 也都接受 ID 陣列作為輸入：

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

我們還可以使用 `sync` 方法來建立多對多關聯。`sync` 方法接受一組要放到中介資料表中的 ID 陣列。任何不在給定陣列中的 ID 都會從中介資料表中移除。因此，此操作結束後，中介資料表中只會存在給定陣列中的 ID：

```php
$user->roles()->sync([1, 2, 3]);
```

你也可以與 ID 一起傳入額外的中介資料表值：

```php
$user->roles()->sync([1 => ['expires' => true], 2, 3]);
```

若想為每個同步的 Model ID 都插入相同的中介資料表值，可使用 `syncWithPivotValues` 方法：

```php
$user->roles()->syncWithPivotValues([1, 2, 3], ['active' => true]);
```

若不想分離掉不存在於給定陣列中的既有 ID，可使用 `syncWithoutDetaching` 方法：

```php
$user->roles()->syncWithoutDetaching([1, 2, 3]);
```


<a name="toggling-associations"></a>
#### 切換關聯

多對多關聯也提供了一個 `toggle` 方法，可用於「切換 (Toggle)」給定關聯 Model ID 的附加狀態。若給定的 ID 目前是已附加，則會被分離。同樣地，若目前為已分離，則會被附加：

```php
$user->roles()->toggle([1, 2, 3]);
```

你也可以與 ID 一起傳入額外的中介資料表值：

```php
$user->roles()->toggle([
    1 => ['expires' => true],
    2 => ['expires' => true],
]);
```


<a name="updating-a-record-on-the-intermediate-table"></a>
#### 更新中介資料表上的紀錄

若需要更新關聯中介資料表上的既有資料列，可使用 `updateExistingPivot` 方法。此方法接受中介紀錄的外鍵與一組要更新的屬性陣列：

```php
$user = User::find(1);

$user->roles()->updateExistingPivot($roleId, [
    'active' => false,
]);
```

<a name="touching-parent-timestamps"></a>
## 接觸 (Touch) 父項時間戳記

當一個 Model 為另一個 Model 定義了 `belongsTo` 或 `belongsToMany` 關聯時 (例如，屬於 `Post` 的 `Comment`)，有時候在更新子 Model 時，能夠一併更新父項的時間戳記會很有幫助。

舉例來說，當 `Comment` Model 更新時，我們可能想自動「接觸 (touch)」其所屬 `Post` 的 `updated_at` 時間戳記，來將其設為目前的日期與時間。為達成此目的，我們可以在子 Model 中加入一個 `touches` 屬性，其中包含在子 Model 更新時，應一同更新其 `updated_at` 時間戳記的關聯名稱：

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
> 只有在使用 Eloquent 的 `save` 方法更新子 Model 時，才會更新父項 Model 的時間戳記。