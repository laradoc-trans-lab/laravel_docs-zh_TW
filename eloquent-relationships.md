# Eloquent：關聯

- [簡介](#introduction)
- [定義關聯](#defining-relationships)
    - [一對一 / Has One](#one-to-one)
    - [一對多 / Has Many](#one-to-many)
    - [一對多 (反向) / Belongs To](#one-to-many-inverse)
    - [Has One of Many](#has-one-of-many)
    - [Has One Through](#has-one-through)
    - [Has Many Through](#has-many-through)
- [範圍限定關聯](#scoped-relationships)
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
    - [關聯方法與動態屬性的區別](#relationship-methods-vs-dynamic-properties)
    - [查詢關聯是否存在](#querying-relationship-existence)
    - [查詢關聯是否不存在](#querying-relationship-absence)
    - [查詢 Morph To 關聯](#querying-morph-to-relationships)
- [聚合關聯模型](#aggregating-related-models)
    - [計算關聯模型數量](#counting-related-models)
    - [其他聚合函數](#other-aggregate-functions)
    - [在 Morph To 關聯上計算關聯模型數量](#counting-related-models-on-morph-to-relationships)
- [預先載入](#eager-loading)
    - [限制預先載入](#constraining-eager-loads)
    - [延遲預先載入](#lazy-eager-loading)
    - [自動預先載入](#automatic-eager-loading)
    - [防止延遲載入](#preventing-lazy-loading)
- [插入與更新關聯模型](#inserting-and-updating-related-models)
    - [`save` 方法](#the-save-method)
    - [`create` 方法](#the-create-method)
    - [Belongs To 關聯](#updating-belongs-to-relationships)
    - [多對多關聯](#updating-many-to-many-relationships)
- [更新父層時間戳記](#touching-parent-timestamps)

<a name="introduction"></a>
## 簡介

資料庫表通常彼此相關聯。例如，一篇部落格文章可能有許多評論，或者一張訂單可能與下單的使用者相關聯。Eloquent 讓管理與處理這些關聯變得簡單，並支援多種常見的關聯：

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

Eloquent 關聯定義在 Eloquent 模型類別的方法中。由於關聯同時也作為強大的 [查詢建立器](/docs/{{version}}/queries)，將關聯定義為方法可以提供強大的方法鏈結與查詢能力。例如，我們可以在這個 `posts` 關聯上鏈結額外的查詢限制：

```php
$user->posts()->where('active', 1)->get();
```

但在深入研究如何使用關聯之前，讓我們來學習如何定義 Eloquent 支援的每種關聯類型。


<a name="one-to-one"></a>
### 一對一 / Has One

一對一關聯是一種非常基礎的資料庫關聯類型。例如，一個 `User` 模型可能與一個 `Phone` 模型相關聯。要定義此關聯，我們會在 `User` 模型中放置一個 `phone` 方法。`phone` 方法應該呼叫 `hasOne` 方法並返回其結果。`hasOne` 方法透過模型的 `Illuminate\Database\Eloquent\Model` 基類提供給您的模型使用：

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

傳遞給 `hasOne` 方法的第一個引數是相關聯的模型類別名稱。一旦定義了關聯，我們就可以使用 Eloquent 的動態屬性來取得相關紀錄。動態屬性允許您像存取定義在模型上的屬性一樣，來存取關聯方法：

```php
$phone = User::find(1)->phone;
```

Eloquent 會根據父模型名稱來決定關聯的外鍵。在這種情況下，`Phone` 模型會自動被假設具有 `user_id` 外鍵。如果您希望覆寫此慣例，可以傳遞第二個引數給 `hasOne` 方法：

```php
return $this->hasOne(Phone::class, 'foreign_key');
```

此外，Eloquent 假設外鍵的值應該與父層的主鍵欄位相匹配。換句話說，Eloquent 會在 `Phone` 紀錄的 `user_id` 欄位中尋找使用者的 `id` 欄位值。如果您希望關聯使用 `id` 或模型主鍵以外的主鍵值，可以傳遞第三個引數給 `hasOne` 方法：

```php
return $this->hasOne(Phone::class, 'foreign_key', 'local_key');
```


<a name="one-to-one-defining-the-inverse-of-the-relationship"></a>
#### 定義關聯的反向

現在我們可以使用 `User` 模型存取 `Phone` 模型。接下來，讓我們在 `Phone` 模型上定義一個關聯，讓我們能夠存取擁有該電話的使用者。我們可以使用 `belongsTo` 方法來定義 `hasOne` 關聯的反向：

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

當呼叫 `user` 方法時，Eloquent 會嘗試尋找一個 `id` 與 `Phone` 模型上的 `user_id` 欄位相匹配的 `User` 模型。

Eloquent 透過檢查關聯方法的名稱並在方法名稱後加上 `_id` 來決定外鍵名稱。因此，在這種情況下，Eloquent 假設 `Phone` 模型具有 `user_id` 欄位。然而，如果 `Phone` 模型上的外鍵不是 `user_id`，您可以將自定義的鍵名稱作為第二個引數傳遞給 `belongsTo` 方法：

```php
/**
 * Get the user that owns the phone.
 */
public function user(): BelongsTo
{
    return $this->belongsTo(User::class, 'foreign_key');
}
```

如果父模型不使用 `id` 作為主鍵，或者您希望使用不同的欄位來尋找關聯模型，您可以傳遞第三個引數給 `belongsTo` 方法，指定父資料表的自定義鍵：

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

一對多關聯用於定義單一模型作為一個或多個子模型父層的關聯。例如，一篇部落格文章可能有無限數量的評論。與所有其他 Eloquent 關聯一樣，一對多關聯是透過在 Eloquent 模型中定義方法來實現的：

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

請記得，Eloquent 會自動為 `Comment` 模型決定適當的外鍵欄位。根據慣例，Eloquent 會取得父模型的「蛇形命名法 (snake case)」名稱並在其後加上 `_id`。因此，在此範例中，Eloquent 會假設 `Comment` 模型上的外鍵欄位是 `post_id`。

一旦定義了關聯方法，我們就可以透過存取 `comments` 屬性來取得相關評論的 [集合](/docs/{{version}}/eloquent-collections)。請記得，由於 Eloquent 提供了「動態關聯屬性」，我們可以像存取定義在模型上的屬性一樣，來存取關聯方法：

```php
use App\Models\Post;

$comments = Post::find(1)->comments;

foreach ($comments as $comment) {
    // ...
}
```

由於所有關聯也同時作為查詢建立器，您可以透過呼叫 `comments` 方法並繼續鏈結條件到查詢中，為關聯查詢添加進一步的限制：

```php
$comment = Post::find(1)->comments()
    ->where('title', 'foo')
    ->first();
```

與 `hasOne` 方法類似，您也可以透過傳遞額外引數給 `hasMany` 方法來覆寫外鍵和本地鍵：

```php
return $this->hasMany(Comment::class, 'foreign_key');

return $this->hasMany(Comment::class, 'foreign_key', 'local_key');
```


<a name="automatically-hydrating-parent-models-on-children"></a>
#### 在子模型上自動填充父模型

即使使用了 Eloquent 預先載入，如果您在迴圈遍歷子模型時嘗試從子模型存取父模型，仍可能會出現「N + 1」查詢問題：

```php
$posts = Post::with('comments')->get();

foreach ($posts as $post) {
    foreach ($post->comments as $comment) {
        echo $comment->post->title;
    }
}
```

在上面的範例中，引入了「N + 1」查詢問題，因為儘管每個 `Post` 模型的評論都已預先載入，但 Eloquent 並不會自動在每個子 `Comment` 模型上填充父層 `Post`。

如果您希望 Eloquent 自動將父模型填充到其子模型中，可以在定義 `hasMany` 關聯時呼叫 `chaperone` 方法：

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

或者，如果您希望在執行時選擇加入自動父層填充，可以在預先載入關聯時呼叫 `chaperone` 模型：

```php
use App\Models\Post;

$posts = Post::with([
    'comments' => fn ($comments) => $comments->chaperone(),
])->get();
```

<a name="one-to-many-inverse"></a>
### 一對多 (反向) / Belongs To

現在我們已經可以存取文章的所有留言了，接著讓我們定義一個關聯，讓留言能夠存取其父層文章。要定義 `hasMany` 關聯的反向關聯，請在子模型中定義一個呼叫 `belongsTo` 方法的關聯方法：

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

一旦定義好關聯後，我們就可以透過存取 `post` 「動態關聯屬性」來取得留言的父層文章：

```php
use App\Models\Comment;

$comment = Comment::find(1);

return $comment->post->title;
```

在上述範例中，Eloquent 會嘗試尋找一個 `id` 與 `Comment` 模型上的 `post_id` 欄位相符的 `Post` 模型。

Eloquent 透過檢查關聯方法的名稱，並在方法名稱後加上 `_` 以及父層模型的主鍵欄位名稱來決定預設的外鍵名稱。因此，在此範例中，Eloquent 會假設 `comments` 表中 `Post` 模型的外鍵是 `post_id`。

然而，如果您的關聯外鍵不符合這些慣例，您可以將自定義的外鍵名稱作為 `belongsTo` 方法的第二個參數傳入：

```php
/**
 * Get the post that owns the comment.
 */
public function post(): BelongsTo
{
    return $this->belongsTo(Post::class, 'foreign_key');
}
```

如果您的父層模型未使用 `id` 作為主鍵，或者您希望使用不同的欄位來尋找關聯模型，您可以傳入第三個參數給 `belongsTo` 方法，以指定父層表的自定義鍵：

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

`belongsTo`、`hasOne`、`hasOneThrough` 以及 `morphOne` 關聯允許您定義一個預設模型，當給定的關聯為 `null` 時將會回傳該模型。這種模式通常被稱為 [空物件模式(Null Object pattern)](https://en.wikipedia.org/wiki/Null_Object_pattern)，可以幫助您移除程式碼中的條件判斷。在以下範例中，如果 `Post` 模型沒有關聯任何使用者，`user` 關聯將回傳一個空的 `App\Models\User` 模型：

```php
/**
 * Get the author of the post.
 */
public function user(): BelongsTo
{
    return $this->belongsTo(User::class)->withDefault();
}
```

若要為預設模型填充屬性，您可以將陣列或閉包 (closure) 傳遞給 `withDefault` 方法：

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

當查詢「belongs to」關聯的子項目時，您可以手動構建 `where` 子句來取得對應的 Eloquent 模型：

```php
use App\Models\Post;

$posts = Post::where('user_id', $user->id)->get();
```

然而，您可能會發現使用 `whereBelongsTo` 方法更為方便，它會自動為給定的模型決定正確的關聯與外鍵：

```php
$posts = Post::whereBelongsTo($user)->get();
```

您也可以向 `whereBelongsTo` 方法提供一個 [集合(collection)](/docs/{{version}}/eloquent-collections) 實例。這樣做時，Laravel 會取得屬於該集合中任何父層模型的模型：

```php
$users = User::where('vip', true)->get();

$posts = Post::whereBelongsTo($users)->get();
```

預設情況下，Laravel 會根據模型的類別名稱來決定與給定模型相關聯的關聯；不過，您也可以透過將關聯名稱作為 `whereBelongsTo` 方法的第二個參數來手動指定關聯名稱：

```php
$posts = Post::whereBelongsTo($user, 'author')->get();
```

<a name="has-one-of-many"></a>
### Has One of Many

有時候一個模型可能具有許多相關模型，但您希望輕鬆地取得該關聯中「最新」或「最舊」的相關模型。例如，一個 `User` 模型可能與許多 `Order` 模型相關聯，但您希望定義一種便捷的方式來與使用者下過的最近一次訂單進行互動。您可以使用 `hasOne` 關聯類型結合 `ofMany` 方法來實現此功能：

```php
/**
 * Get the user's most recent order.
 */
public function latestOrder(): HasOne
{
    return $this->hasOne(Order::class)->latestOfMany();
}
```

同樣地，您也可以定義一個方法來取得關聯中「最舊」或第一個相關模型：

```php
/**
 * Get the user's oldest order.
 */
public function oldestOrder(): HasOne
{
    return $this->hasOne(Order::class)->oldestOfMany();
}
```

預設情況下，`latestOfMany` 和 `oldestOfMany` 方法將根據模型的主鍵（必須是可排序的）來取得最新或最舊的相關模型。然而，有時候您可能希望使用不同的排序標準，從較大的關聯中取得單一模型。

例如，使用 `ofMany` 方法，您可以取得使用者最昂貴的訂單。`ofMany` 方法接受可排序的欄位作為第一個引數，以及在查詢相關模型時要套用的聚合函數（`min` 或 `max`）：

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
> 由於 PostgreSQL 不支援對 UUID 欄位執行 `MAX` 函數，目前無法在 PostgreSQL UUID 欄位中使用 one-of-many 關聯。

<a name="converting-many-relationships-to-has-one-relationships"></a>
#### 將「多」關聯轉換為 Has One 關聯

通常，當您使用 `latestOfMany`、`oldestOfMany` 或 `ofMany` 方法取得單一模型時，您可能已經為該模型定義了一個「has many」關聯。為了方便起見，Laravel 允許您透過在關聯上呼叫 `one` 方法，輕鬆地將此關聯轉換為「has one」關聯：

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

您也可以使用 `one` 方法將 `HasManyThrough` 關聯轉換為 `HasOneThrough` 關聯：

```php
public function latestDeployment(): HasOneThrough
{
    return $this->deployments()->one()->latestOfMany();
}
```

<a name="advanced-has-one-of-many-relationships"></a>
#### 進階 Has One of Many 關聯

您可以建構更進階的「has one of many」關聯。例如，一個 `Product` 模型可能有許多相關聯的 `Price` 模型，即使在發布新價格後，這些價格仍保留在系統中。此外，產品的新價格數據可能會提前發布，並透過 `published_at` 欄位在未來某個日期生效。

總結來說，我們需要取得最新已發布且發布日期不在未來的價格。此外，如果兩個價格具有相同的發布日期，我們將優先選擇 ID 最大的價格。為了實現這一點，我們必須向 `ofMany` 方法傳遞一個包含決定最新價格的可排序欄位的陣列。此外，將提供一個閉包作為 `ofMany` 方法的第二個引數，該閉包將負責為關聯查詢添加額外的發布日期限制：

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

「has-one-through」關聯定義了與另一個模型的 一對一 關聯。然而，這種關聯表示宣告模型可以透過 _經過_ 第三個模型來與另一個模型的一個實例進行匹配。

例如，在一個汽車修理廠應用程式中，每個 `Mechanic` 模型可能與一個 `Car` 模型相關聯，而每個 `Car` 模型可能與一個 `Owner` 模型相關聯。雖然技師和車主在資料庫中沒有直接關係，但技師可以 _透過_ `Car` 模型來存取車主。讓我們看看定義此關聯所需的資料表：

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

現在我們已經檢查了關聯的資料表結構，讓我們在 `Mechanic` 模型上定義此關聯：

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

傳遞給 `hasOneThrough` 方法的第一個引數是我們希望存取的最終模型名稱，而第二個引數是中間模型的名稱。

或者，如果相關關聯已經在所有參與關聯的模型中定義，您可以透過呼叫 `through` 方法並提供這些關聯的名稱，來流暢地定義「has-one-through」關聯。例如，如果 `Mechanic` 模型具有 `cars` 關聯且 `Car` 模型具有 `owner` 關聯，您可以像這樣定義連接技師和車主的「has-one-through」關聯：

```php
// String based syntax...
return $this->through('cars')->has('owner');

// Dynamic syntax...
return $this->throughCars()->hasOwner();
```

<a name="has-one-through-key-conventions"></a>
#### 金鑰慣例

在執行關聯查詢時，將使用典型的 Eloquent 外鍵慣例。如果您想自定義關聯的金鑰，可以將它們作為 `hasOneThrough` 方法的第三和第四個引數傳遞。第三個引數是中間模型上的外鍵名稱。第四個引數是最終模型上的外鍵名稱。第五個引數是本地金鑰，而第六個引數是中間模型的本地金鑰：

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

或者，如前所述，如果相關關聯已經在所有參與關聯的模型中定義，您可以透過呼叫 `through` 方法並提供這些關聯的名稱，來流暢地定義「has-one-through」關聯。這種方法具有重複使用現有關聯中已定義金鑰慣例的優勢：

```php
// String based syntax...
return $this->through('cars')->has('owner');

// Dynamic syntax...
return $this->throughCars()->hasOwner();
```

<a name="has-many-through"></a>
### Has Many Through

「Has Many Through」關聯提供了一種便捷的方式，透過一個中介關聯來存取遠端關聯。例如，假設我們正在構建一個像 [Laravel Cloud](https://cloud.laravel.com) 這樣的部署平台。一個 `Application` 模型可能會透過一個中介 `Environment` 模型來存取許多 `Deployment` 模型。使用這個範例，您可以輕鬆地收集特定應用程式的所有部署。讓我們看看定義此關聯所需的資料表：

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

既然我們已經檢查了關聯的資料表結構，讓我們在 `Application` 模型上定義此關聯：

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

傳遞給 `hasManyThrough` 方法的第一個引數是我們希望存取的最終模型名稱，而第二個引數則是中介模型的名稱。

或者，如果相關的關聯已經在所有涉及該關聯的模型上定義好了，您可以透過呼叫 `through` 方法並提供這些關聯的名稱，以流暢的鏈式語法來定義「Has Many Through」關聯。例如，如果 `Application` 模型具有 `environments` 關聯，且 `Environment` 模型具有 `deployments` 關聯，您可以像這樣定義一個連接應用程式與部署的「Has Many Through」關聯：

```php
// String based syntax...
return $this->through('environments')->has('deployments');

// Dynamic syntax...
return $this->throughEnvironments()->hasDeployments();
```

儘管 `Deployment` 模型的資料表並不包含 `application_id` 欄位，但 `hasManyThrough` 關聯讓您可以透過 `$application->deployments` 存取應用程式的部署。為了取得這些模型，Eloquent 會檢查中介 `Environment` 模型資料表上的 `application_id` 欄位。在找到相關的環境 ID 後，它們將被用於查詢 `Deployment` 模型的資料表。


<a name="has-many-through-key-conventions"></a>
#### Key Conventions

在執行關聯查詢時，將使用 Eloquent 的典型外鍵慣例。如果您想自定義關聯的鍵，可以將它們作為第三和第四個引數傳遞給 `hasManyThrough` 方法。第三個引數是中介模型上的外鍵名稱，第四個引數是最終模型上的外鍵名稱，第五個引數是本地鍵，而第六個引數則是中介模型的本地鍵：

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

或者，如前所述，如果相關的關聯已經在所有涉及該關聯的模型上定義好了，您可以透過呼叫 `through` 方法並提供這些關聯的名稱，以流暢的鏈式語法來定義「Has Many Through」關聯。這種方法的好處是可以重複使用現有關聯中已定義的鍵慣例：

```php
// String based syntax...
return $this->through('environments')->has('deployments');

// Dynamic syntax...
return $this->throughEnvironments()->hasDeployments();
```


<a name="scoped-relationships"></a>
### 範圍限定關聯

在模型中加入額外的方法來限制關聯是很常見的。例如，您可能會在 `User` 模型中加入一個 `featuredPosts` 方法，該方法透過額外的 `where` 限制來約束更廣泛的 `posts` 關聯：

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

然而，如果您嘗試透過 `featuredPosts` 方法建立模型，其 `featured` 屬性將不會被設置為 `true`。如果您希望透過關聯方法建立模型，並指定應添加到透過該關聯建立的所有模型中的屬性，您可以在構建關聯查詢時使用 `withAttributes` 方法：

```php
/**
 * Get the user's featured posts.
 */
public function featuredPosts(): HasMany
{
    return $this->posts()->withAttributes(['featured' => true]);
}
```

`withAttributes` 方法會使用給定的屬性將 `where` 條件添加到查詢中，並且也會將這些屬性添加到透過該關聯方法建立的任何模型中：

```php
$post = $user->featuredPosts()->create(['title' => 'Featured Post']);

$post->featured; // true
```

若要指示 `withAttributes` 方法不要將 `where` 條件添加到查詢中，您可以將 `asConditions` 引數設置為 `false`：

```php
return $this->posts()->withAttributes(['featured' => true], asConditions: false);
```

<a name="many-to-many"></a>
## 多對多關聯

多對多關聯比 `hasOne` 與 `hasMany` 關聯稍微複雜一些。多對多關聯的一個例子是：一名使用者擁有多個角色，而這些角色也被應用程式中的其他使用者共用。例如，一名使用者可能被分配為「作者」與「編輯」的角色；然而，這些角色也可能被分配給其他使用者。因此，一名使用者擁有多個角色，而一個角色也擁有多名使用者。


<a name="many-to-many-table-structure"></a>
#### 表格結構

要定義這種關聯，需要三個資料庫表格：`users`、`roles` 以及 `role_user`。`role_user` 表格的名稱是由相關模型名稱的字母順序衍生而來，並包含 `user_id` 與 `role_id` 欄位。此表格被用作連結使用者與角色的中間表。

請記得，由於一個角色可以屬於多名使用者，我們不能簡單地在 `roles` 表格中放置 `user_id` 欄位。這樣會意味著一個角色只能屬於一名使用者。為了支援角色被分配給多名使用者，因此需要 `role_user` 表格。我們可以將此關聯的表格結構總結如下：

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

多對多關聯是透過撰寫一個回傳 `belongsToMany` 方法結果的方法來定義的。`belongsToMany` 方法是由所有應用程式 Eloquent 模型所使用的 `Illuminate\Database\Eloquent\Model` 基底類別提供的。例如，讓我們在 `User` 模型中定義一個 `roles` 方法。傳遞給此方法的第一個引數是相關模型類別的名稱：

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

一旦定義了關聯，您就可以使用 `roles` 動態關聯屬性來存取使用者的角色：

```php
use App\Models\User;

$user = User::find(1);

foreach ($user->roles as $role) {
    // ...
}
```

由於所有關聯也同時充當查詢建構器，您可以透過呼叫 `roles` 方法並繼續在查詢上鏈接條件，來為關聯查詢添加進一步的限制：

```php
$roles = User::find(1)->roles()->orderBy('name')->get();
```

為了決定關聯中間表的名稱，Eloquent 會將兩個相關的模型名稱按字母順序連接起來。不過，您可以自由地覆寫此慣例。您可以透過將第二個引數傳遞給 `belongsToMany` 方法來實現：

```php
return $this->belongsToMany(Role::class, 'role_user');
```

除了自定義中間表的名稱外，您還可以透過將額外引數傳遞給 `belongsToMany` 方法來自定義表格中鍵值的欄位名稱。第三個引數是您定義關聯之模型的外鍵名稱，而第四個引數則是您要連接之模型的外鍵名稱：

```php
return $this->belongsToMany(Role::class, 'role_user', 'user_id', 'role_id');
```


<a name="many-to-many-defining-the-inverse-of-the-relationship"></a>
#### 定義關聯的反向

要定義多對多關聯的「反向」，您應該在相關模型中定義一個同樣回傳 `belongsToMany` 方法結果的方法。為了完成我們的使用者 / 角色範例，讓我們在 `Role` 模型中定義 `users` 方法：

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

如您所見，除了引用 `App\Models\User` 模型外，此關聯的定義與其對應的 `User` 模型完全相同。由於我們重複使用了 `belongsToMany` 方法，因此在定義多對多關聯的反向時，所有常用的表格與鍵值自定義選項均可用。


<a name="retrieving-intermediate-table-columns"></a>
### 取得中間表欄位

正如您已經學到的，處理多對多關聯需要中間表。Eloquent 提供了一些非常方便的方式來與此表格互動。例如，假設我們的 `User` 模型與多個 `Role` 模型相關聯。在存取此關聯後，我們可以使用模型上的 `pivot` 屬性來存取中間表：

```php
use App\Models\User;

$user = User::find(1);

foreach ($user->roles as $role) {
    echo $role->pivot->created_at;
}
```

請注意，我們取得的每個 `Role` 模型都會自動被分配一個 `pivot` 屬性。此屬性包含一個代表中間表的模型。

預設情況下，`pivot` 模型中僅會包含模型鍵值。如果您的中間表包含額外屬性，您必須在定義關聯時指定它們：

```php
return $this->belongsToMany(Role::class)->withPivot('active', 'created_by');
```

如果您希望中間表擁有由 Eloquent 自動維護的 `created_at` 與 `updated_at` 時間戳記，請在定義關聯時呼叫 `withTimestamps` 方法：

```php
return $this->belongsToMany(Role::class)->withTimestamps();
```

> [!WARNING]
> 利用 Eloquent 自動維護時間戳記的中間表，必須同時擁有 `created_at` 與 `updated_at` 時間戳記欄位。


<a name="customizing-the-pivot-attribute-name"></a>
#### 自定義 `pivot` 屬性名稱

如前所述，中間表的屬性可以透過模型的 `pivot` 屬性來存取。不過，您可以自由地自定義此屬性的名稱，以更好地反映其在應用程式中的用途。

例如，如果您的應用程式包含可以訂閱播客(Podcasts)的使用者，您可能在使用者與播客之間具有多對多關聯。在這種情況下，您可能希望將中間表屬性重新命名為 `subscription` 而不是 `pivot`。這可以在定義關聯時使用 `as` 方法來完成：

```php
return $this->belongsToMany(Podcast::class)
    ->as('subscription')
    ->withTimestamps();
```

一旦指定了自定義的中間表屬性，您就可以使用自定義的名稱來存取中間表數據：

```php
$users = User::with('podcasts')->get();

foreach ($users->flatMap->podcasts as $podcast) {
    echo $podcast->subscription->created_at;
}
```

<a name="filtering-queries-via-intermediate-table-columns"></a>
### 透過中間表欄位篩選查詢

您也可以在定義關聯時，使用 `wherePivot`、`wherePivotIn`、`wherePivotNotIn`、`wherePivotBetween`、`wherePivotNotBetween`、`wherePivotNull` 以及 `wherePivotNotNull` 方法，來篩選 `belongsToMany` 關聯查詢所回傳的結果：

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

`wherePivot` 會在查詢中加入 where 子句限制，但在透過定義的關聯建立新模型時，不會加入指定的值。如果您需要同時針對特定的 pivot 值進行查詢並建立關聯，可以使用 `withPivotValue` 方法：

```php
return $this->belongsToMany(Role::class)
    ->withPivotValue('approved', 1);
```


<a name="ordering-queries-via-intermediate-table-columns"></a>
### 透過中間表欄位排序查詢

您可以使用 `orderByPivot` 與 `orderByPivotDesc` 方法，對 `belongsToMany` 關聯查詢所回傳的結果進行排序。在以下範例中，我們將取得該使用者的所有最新徽章：

```php
return $this->belongsToMany(Badge::class)
    ->where('rank', 'gold')
    ->orderByPivotDesc('created_at');
```


<a name="defining-custom-intermediate-table-models"></a>
### 定義自定義中間表模型

如果您想要定義一個自定義模型來代表多對多關聯的中間表，可以在定義關聯時呼叫 `using` 方法。自定義 pivot 模型讓您有機會在 pivot 模型上定義額外的行為，例如方法與型別轉換 (casts)。

自定義的多對多 pivot 模型應該繼承 `Illuminate\Database\Eloquent\Relations\Pivot` 類別，而自定義的多型多對多 pivot 模型則應該繼承 `Illuminate\Database\Eloquent\Relations\MorphPivot` 類別。例如，我們可以定義一個使用自定義 `RoleUser` pivot 模型的 `Role` 模型：

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

在定義 `RoleUser` 模型時，您應該繼承 `Illuminate\Database\Eloquent\Relations\Pivot` 類別：

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
> Pivot 模型不能使用 `SoftDeletes` trait。如果您需要對 pivot 記錄進行軟刪除，請考慮將您的 pivot 模型轉換為實際的 Eloquent 模型。


<a name="custom-pivot-models-and-incrementing-ids"></a>
#### 自定義 Pivot 模型與自動遞增 ID

如果您定義了一個使用自定義 pivot 模型的多對多關聯，且該 pivot 模型具有自動遞增的主鍵，您應該確保您的自定義 pivot 模型類別使用了 `Table` 屬性並將 `incrementing` 設定為 `true`：

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

多型關聯允許子模型透過單一的關聯，隸屬於多個不同類型的模型。例如，假設您正在開發一個允許使用者分享部落格文章與影片的應用程式。在這樣的應用程式中，一個 `Comment` 模型可能會同時隸屬於 `Post` 和 `Video` 模型。


<a name="one-to-one-polymorphic-relations"></a>
### 一對一


<a name="one-to-one-polymorphic-table-structure"></a>
#### 表格結構

一對一多型關聯與典型的一對一關聯非常相似；然而，子模型可以透過單一關聯隸屬於多個不同類型的模型。例如，部落格的 `Post` 和 `User` 可能會與 `Image` 模型共享一個多型關聯。使用一對一多型關聯，可以讓您擁有單一的唯一圖片表，且這些圖片可以與文章或使用者關聯。首先，讓我們來看看表格結構：

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

請注意 `images` 表中的 `imageable_id` 與 `imageable_type` 欄位。`imageable_id` 欄位將包含文章或使用者的 ID 值，而 `imageable_type` 欄位則將包含父模型的類別名稱。`imageable_type` 欄位被 Eloquent 用於在存取 `imageable` 關聯時，決定要回傳哪一種「類型」的父模型。在此範例中，該欄位將包含 `App\Models\Post` 或 `App\Models\User`。


<a name="one-to-one-polymorphic-model-structure"></a>
#### 模型結構

接下來，讓我們看看建立此關聯所需定義的模型：

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

一旦定義好資料庫表與模型，您就可以透過模型存取關聯。例如，若要取得文章的圖片，我們可以存取 `image` 動態關聯屬性：

```php
use App\Models\Post;

$post = Post::find(1);

$image = $post->image;
```

您可以透過存取呼叫 `morphTo` 的方法名稱，來取得多型模型的父模型。在此範例中，即為 `Image` 模型上的 `imageable` 方法。因此，我們將該方法作為動態關聯屬性來存取：

```php
use App\Models\Image;

$image = Image::find(1);

$imageable = $image->imageable;
```

`Image` 模型上的 `imageable` 關聯將根據誰擁有該圖片，回傳 `Post` 或 `User` 的實例。


<a name="morph-one-to-one-key-conventions"></a>
#### 金鑰慣例

如有必要，您可以指定多型子模型所使用的 "id" 與 "type" 欄位名稱。若您這樣做，請確保始終將關聯名稱作為 `morphTo` 方法的第一個引數傳入。通常此值應與方法名稱一致，因此您可以使用 PHP 的 `__FUNCTION__` 常數：

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

一對多多型關聯與典型的一對多關聯類似；然而，子模型可以使用單一關聯隸屬於多種類型的模型。例如，想像您的應用程式使用者可以在文章 (posts) 和影片 (videos) 上發表 「評論」。使用多型關聯，您可以使用單個 `comments` 資料表來儲存文章和影片的評論。首先，讓我們 examine 建立此關聯所需的資料表結構：

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

接下來，讓我們 examine 建立此關聯所需的模型定義：

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

一旦定義好資料表與模型，您就可以透過模型的動態關聯屬性來存取這些關聯。例如，若要存取文章的所有評論，我們可以使用 `comments` 動態屬性：

```php
use App\Models\Post;

$post = Post::find(1);

foreach ($post->comments as $comment) {
    // ...
}
```

您也可以透過存取執行 `morphTo` 呼叫的方法名稱，來取得多型子模型的父模型。在這種情況下，就是 `Comment` 模型上的 `commentable` 方法。因此，我們將該方法作為動態關聯屬性來存取，以取得評論的父模型：

```php
use App\Models\Comment;

$comment = Comment::find(1);

$commentable = $comment->commentable;
```

`Comment` 模型上的 `commentable` 關聯將根據哪種類型的模型擁有該評論，而返回 `Post` 或 `Video` 實例。

<a name="polymorphic-automatically-hydrating-parent-models-on-children"></a>
#### 自動為子模型填充父模型

即使使用了 Eloquent 的預先載入，如果您在遍歷子模型時嘗試從子模型存取父模型，仍可能會出現 「N + 1」 查詢問題：

```php
$posts = Post::with('comments')->get();

foreach ($posts as $post) {
    foreach ($post->comments as $comment) {
        echo $comment->commentable->title;
    }
}
```

在上述範例中，引入了 「N + 1」 查詢問題，因為儘管每個 `Post` 模型都預先載入了評論，但 Eloquent 並不會自動在每個子 `Comment` 模型上填充父 `Post` 模型。

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

或者，如果您想在執行時選擇加入自動父模型填充，您可以在預先載入關聯時呼叫 `chaperone` 方法：

```php
use App\Models\Post;

$posts = Post::with([
    'comments' => fn ($comments) => $comments->chaperone(),
])->get();
```

<a name="one-of-many-polymorphic-relations"></a>
### One of Many

有時候一個模型可能擁有多個關聯模型，但您希望能輕鬆地取得該關聯中 「最新」 或 「最舊」 的關聯模型。例如，一個 `User` 模型可能與許多 `Image` 模型相關聯，但您想定義一種方便的方式來與使用者上傳的最新圖片進行互動。您可以使用 `morphOne` 關聯類型結合 `ofMany` 方法來實現這一點：

```php
/**
 * Get the user's most recent image.
 */
public function latestImage(): MorphOne
{
    return $this->morphOne(Image::class, 'imageable')->latestOfMany();
}
```

同樣地，您可以定義一個方法來取得關聯中 「最舊」 或第一個相關模型：

```php
/**
 * Get the user's oldest image.
 */
public function oldestImage(): MorphOne
{
    return $this->morphOne(Image::class, 'imageable')->oldestOfMany();
}
```

預設情況下，`latestOfMany` 和 `oldestOfMany` 方法會根據模型的主鍵（必須是可排序的）來取得最新或最舊的關聯模型。然而，有時候您可能希望使用不同的排序標準從較大的關聯中取得單一模型。

例如，使用 `ofMany` 方法，您可以取得使用者最受歡迎（最多讚）的圖片。`ofMany` 方法接受可排序的欄位作為第一個引數，以及在查詢關聯模型時要套用的聚合函數 (`min` 或 `max`)：

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
> 可以構建更進階的 「One of Many」 關聯。欲了解更多資訊，請參閱 [has one of many 文件](#advanced-has-one-of-many-relationships)。

<a name="many-to-many-polymorphic-relations"></a>
### 多對多

<a name="many-to-many-polymorphic-table-structure"></a>
#### 表格結構

多對多多型關聯比 「morph one」 和 「morph many」 關聯稍微複雜一些。例如，`Post` 模型和 `Video` 模型可能與 `Tag` 模型共享一個多型關聯。在這種情況下，使用多對多多型關聯可以讓您的應用程式擁有一個單一的唯一標籤表，該表可以與文章或影片相關聯。首先，讓我們來查看建立此關聯所需的表格結構：

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
> 在深入研究多對多多型關聯之前，您可能會受益於閱讀關於一般[多對多關聯](#many-to-many)的說明文件。


<a name="many-to-many-polymorphic-model-structure"></a>
#### 模型結構

接下來，我們準備在模型上定義關聯。`Post` 和 `Video` 模型都將包含一個 `tags` 方法，該方法會呼叫由 Eloquent 基礎模型類別提供的 `morphToMany` 方法。

`morphToMany` 方法接受相關模型的名稱以及 「關聯名稱」。根據我們為中間表名稱及其包含的鍵所指定的名稱，我們將此關聯稱為 「taggable」：

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
#### 定義關聯的反向定義

接下來，在 `Tag` 模型上，您應該為每個可能的父層模型定義一個方法。因此，在本範例中，我們將定義 `posts` 方法和 `videos` 方法。這兩個方法都應該回傳 `morphedByMany` 方法的結果。

`morphedByMany` 方法接受相關模型的名稱以及 「關聯名稱」。根據我們為中間表名稱及其包含的鍵所指定的名稱，我們將此關聯稱為 「taggable」：

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

一旦定義好資料庫表格和模型，您就可以透過模型存取關聯。例如，要取得文章的所有標籤，可以使用 `tags` 動態關聯屬性：

```php
use App\Models\Post;

$post = Post::find(1);

foreach ($post->tags as $tag) {
    // ...
}
```

您可以透過存取執行 `morphedByMany` 呼叫的方法名稱，從多型子模型中取得多型關聯的父層。在這種情況下，即為 `Tag` 模型上的 `posts` 或 `videos` 方法。因此，我們將該方法作為動態關聯屬性來存取：

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

預設情況下，Laravel 會使用全限定類別名稱 (fully qualified class name) 來儲存相關模型的 「類型」。例如，在上述的一對多關聯範例中，`Comment` 模型可能屬於 `Post` 或 `Video` 模型，預設的 `commentable_type` 將分別為 `App\Models\Post` 或 `App\Models\Video`。然而，您可能希望將這些值與應用程式的內部結構解耦。

例如，我們可以使用簡單的字串（如 `post` 和 `video`）來代替模型名稱作為 「類型」。這樣一來，即使模型重新命名，資料庫中的多型 「類型」 欄位值仍然有效：

```php
use Illuminate\Database\Eloquent\Relations\Relation;

Relation::enforceMorphMap([
    'post' => 'App\Models\Post',
    'video' => 'App\Models\Video',
]);
```

您可以在 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫 `enforceMorphMap` 方法，或者如果您願意，可以建立一個獨立的服務提供者。

您可以使用模型的 `getMorphClass` 方法在執行時 (runtime) 確定給定模型的多型別名。相反地，您可以使用 `Relation::getMorphedModel` 方法確定與多型別名相關聯的全限定類別名稱：

```php
use Illuminate\Database\Eloquent\Relations\Relation;

$alias = $post->getMorphClass();

$class = Relation::getMorphedModel($alias);
```

> [!WARNING]
> 當在現有應用程式中添加 「多型映射 (morph map)」 時，資料庫中所有仍包含全限定類別的 `*_type` 欄位值都需要轉換為其 「映射」 名稱。


<a name="dynamic-relationships"></a>
### 動態關聯

您可以使用 `resolveRelationUsing` 方法在執行時定義 Eloquent 模型之間的關聯。雖然通常不建議在一般應用程式開發中使用，但在開發 Laravel 套件時，這可能會非常有用。

`resolveRelationUsing` 方法接受所需的關聯名稱作為第一個引數。傳遞給該方法的第二個引數應該是一個閉包，該閉包接受模型實例並回傳一個有效的 Eloquent 關聯定義。通常，您應該在[服務提供者](/docs/{{version}}/providers)的 boot 方法中配置動態關聯：

```php
use App\Models\Order;
use App\Models\Customer;

Order::resolveRelationUsing('customer', function (Order $orderModel) {
    return $orderModel->belongsTo(Customer::class, 'customer_id');
});
```

> [!WARNING]
> 定義動態關聯時，請務必為 Eloquent 關聯方法提供明確的鍵名稱引數。

<a name="querying-relations"></a>
## 查詢關聯

由於所有 Eloquent 關聯都是透過方法定義的，您可以呼叫這些方法來取得關聯的實例，而不需要實際執行查詢來載入關聯模型。此外，所有類型的 Eloquent 關聯也同時作為 [查詢構建器](/docs/{{version}}/queries)，讓您可以在最終對資料庫執行 SQL 查詢之前，繼續在關聯查詢上鏈接約束條件。

例如，想像一個部落格應用程式，其中 `User` 模型具有許多關聯的 `Post` 模型：

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

您可以查詢 `posts` 關聯並像這樣為關聯添加額外的約束條件：

```php
use App\Models\User;

$user = User::find(1);

$user->posts()->where('active', 1)->get();
```

您可以在關聯上使用 Laravel [查詢構建器](/docs/{{version}}/queries) 的任何方法，因此請務必探索查詢構建器的文件，以了解所有可用於您的方法。


<a name="chaining-orwhere-clauses-after-relationships"></a>
#### 在關聯之後鏈接 `orWhere` 子句

如上面的範例所示，您可以在查詢關聯時自由地添加額外的約束條件。然而，在關聯上鏈接 `orWhere` 子句時請務必小心，因為 `orWhere` 子句將與關聯約束處於相同的邏輯分組層級：

```php
$user->posts()
    ->where('active', 1)
    ->orWhere('votes', '>=', 100)
    ->get();
```

上面的範例將產生以下 SQL。如您所見，`or` 子句會指示查詢回傳 *任何* 投票數大於 100 的文章。該查詢不再被限制在特定的使用者內：

```sql
select *
from posts
where user_id = ? and active = 1 or votes >= 100
```

在大多數情況下，您應該使用 [邏輯分組](/docs/{{version}}/queries#logical-grouping) 將條件檢查包含在括號中：

```php
use Illuminate\Database\Eloquent\Builder;

$user->posts()
    ->where(function (Builder $query) {
        return $query->where('active', 1)
            ->orWhere('votes', '>=', 100);
    })
    ->get();
```

上面的範例將產生以下 SQL。請注意，邏輯分組已正確地將約束條件分組，且查詢仍然被限制在特定的使用者內：

```sql
select *
from posts
where user_id = ? and (active = 1 or votes >= 100)
```


<a name="relationship-methods-vs-dynamic-properties"></a>
### 關聯方法與動態屬性的區別

如果您不需要在 Eloquent 關聯查詢中添加額外的約束條件，您可以將關聯視為屬性來存取。例如，繼續使用我們的 `User` 和 `Post` 範例模型，我們可以像這樣存取使用者所有的文章：

```php
use App\Models\User;

$user = User::find(1);

foreach ($user->posts as $post) {
    // ...
}
```

動態關聯屬性執行的是「延遲載入 (lazy loading)」，這意味著它們只有在您實際存取時才會載入關聯數據。因此，開發者通常會使用 [預先載入](#eager-loading) 來預先載入他們知道在載入模型後會被存取的關聯。預先載入能顯著減少載入模型關聯時必須執行的 SQL 查詢數量。


<a name="querying-relationship-existence"></a>
### 查詢關聯是否存在

在取得模型記錄時，您可能希望根據關聯是否存在來限制結果。例如，想像您想要取得所有至少有一則評論的部落格文章。為此，您可以將關聯名稱傳遞給 `has` 和 `orHas` 方法：

```php
use App\Models\Post;

// Retrieve all posts that have at least one comment...
$posts = Post::has('comments')->get();
```

您還可以指定運算子和計數值來進一步自定義查詢：

```php
// Retrieve all posts that have three or more comments...
$posts = Post::has('comments', '>=', 3)->get();
```

可以使用「點號表示法」來構建巢狀的 `has` 語句。例如，您可以取得所有至少有一則評論且該評論至少有一張圖片的文章：

```php
// Retrieve posts that have at least one comment with images...
$posts = Post::has('comments.images')->get();
```

如果您需要更強的功能，可以使用 `whereHas` 和 `orWhereHas` 方法在 `has` 查詢中定義額外的查詢約束，例如檢查評論的內容：

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
> Eloquent 目前不支援跨資料庫查詢關聯是否存在。關聯必須存在於同一個資料庫中。


<a name="many-to-many-relationship-existence-queries"></a>
#### 多對多關聯存在查詢

`whereAttachedTo` 方法可用於查詢與某個模型或模型集合具有多對多附加關係的模型：

```php
$users = User::whereAttachedTo($role)->get();
```

您也可以將 [集合](/docs/{{version}}/eloquent-collections) 實例傳遞給 `whereAttachedTo` 方法。這樣做時，Laravel 將取得與集合中任何模型相關聯的模型：

```php
$tags = Tag::whereLike('name', '%laravel%')->get();

$posts = Post::whereAttachedTo($tags)->get();
```


<a name="inline-relationship-existence-queries"></a>
#### 行內關聯存在查詢

如果您想在關聯查詢中附加單一且簡單的 where 條件來查詢關聯是否存在，您可能會發現使用 `whereRelation`、`orWhereRelation`、`whereMorphRelation` 和 `orWhereMorphRelation` 方法更方便。例如，我們可以查詢所有具有未核准評論的文章：

```php
use App\Models\Post;

$posts = Post::whereRelation('comments', 'is_approved', false)->get();
```

當然，就像呼叫查詢構建器的 `where` 方法一樣，您也可以指定運算子：

```php
$posts = Post::whereRelation(
    'comments', 'created_at', '>=', now()->minus(hours: 1)
)->get();
```


<a name="querying-relationship-absence"></a>
### 查詢關聯是否不存在

在取得模型記錄時，您可能希望根據關聯是否不存在來限制結果。例如，想像您想要取得所有**沒有**任何評論的部落格文章。為此，您可以將關聯名稱傳遞給 `doesntHave` 和 `orDoesntHave` 方法：

```php
use App\Models\Post;

$posts = Post::doesntHave('comments')->get();
```

如果您需要更強的功能，可以使用 `whereDoesntHave` 和 `orWhereDoesntHave` 方法在 `doesntHave` 查詢中添加額外的查詢約束，例如檢查評論的內容：

```php
use Illuminate\Database\Eloquent\Builder;

$posts = Post::whereDoesntHave('comments', function (Builder $query) {
    $query->where('content', 'like', 'code%');
})->get();
```

您可以使用「點號表示法」對巢狀關聯執行查詢。例如，以下查詢將取得所有沒有評論的文章，以及具有評論但沒有任何評論來自於被封鎖使用者的文章：

```php
use Illuminate\Database\Eloquent\Builder;

$posts = Post::whereDoesntHave('comments.author', function (Builder $query) {
    $query->where('banned', 1);
})->get();
```

<a name="querying-morph-to-relationships"></a>
### 查詢 Morph To 關聯

若要查詢 'morph to' 關聯是否存在，您可以使用 `whereHasMorph` 與 `whereDoesntHaveMorph` 方法。這些方法接受關聯名稱作為第一個引數。接下來，方法接受您希望包含在查詢中的相關模型名稱。最後，您可以提供一個閉包來自定義關聯查詢：

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

您有時可能需要根據相關多型模型的 "type" 來增加查詢限制。傳遞給 `whereHasMorph` 方法的閉包可以接收 `$type` 值作為其第二個引數。此引數允許您檢視目前正在建構的查詢 "type"：

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

有時您可能想要查詢 'morph to' 關聯父層的子項目。您可以使用 `whereMorphedTo` 與 `whereNotMorphedTo` 方法來達成，這將會自動為給定的模型決定正確的多型類型對映。這些方法接受 `morphTo` 關聯名稱作為第一個引數，並接受相關的父模型作為第二個引數：

```php
$comments = Comment::whereMorphedTo('commentable', $post)
    ->orWhereMorphedTo('commentable', $video)
    ->get();
```


<a name="querying-all-morph-to-related-models"></a>
#### 查詢所有相關模型

您可以提供 `*` 作為萬用字元值，而非傳遞可能的多型模型陣列。這將指示 Laravel 從資料庫中取得所有可能的多型類型。Laravel 將執行一次額外的查詢以完成此操作：

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

有時候您可能想要計算特定關聯的相關模型數量，而不需要實際載入這些模型。為了達成此目的，您可以使用 `withCount` 方法。`withCount` 方法會在結果模型中放置一個 `{relation}_count` 屬性：

```php
use App\Models\Post;

$posts = Post::withCount('comments')->get();

foreach ($posts as $post) {
    echo $post->comments_count;
}
```

透過向 `withCount` 方法傳遞一個陣列，您可以為多個關聯增加「數量」計算，並為查詢添加額外的限制條件：

```php
use Illuminate\Database\Eloquent\Builder;

$posts = Post::withCount(['votes', 'comments' => function (Builder $query) {
    $query->where('content', 'like', 'code%');
}])->get();

echo $posts[0]->votes_count;
echo $posts[0]->comments_count;
```

您也可以為關聯數量結果設定別名，以便在同一個關聯上進行多次計數：

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
#### 延遲數量載入

使用 `loadCount` 方法，您可以在父模型已被取得後，再載入關聯數量：

```php
$book = Book::first();

$book->loadCount('genres');
```

如果您需要在數量查詢上設定額外的限制條件，可以傳遞一個以關聯名稱為鍵的陣列。陣列的值應該是接收查詢建構器實體的閉包：

```php
$book->loadCount(['reviews' => function (Builder $query) {
    $query->where('rating', 5);
}])
```


<a name="relationship-counting-and-custom-select-statements"></a>
#### 關聯數量計算與自定義 Select 語句

如果您將 `withCount` 與 `select` 語句結合使用，請確保在 `select` 方法之後呼叫 `withCount`：

```php
$posts = Post::select(['title', 'body'])
    ->withCount('comments')
    ->get();
```


<a name="other-aggregate-functions"></a>
### 其他聚合函數

除了 `withCount` 方法外，Eloquent 還提供了 `withMin`、`withMax`、`withAvg`、`withSum` 以及 `withExists` 方法。這些方法會在結果模型中放置一個 `{relation}_{function}_{column}` 屬性：

```php
use App\Models\Post;

$posts = Post::withSum('comments', 'votes')->get();

foreach ($posts as $post) {
    echo $post->comments_sum_votes;
}
```

如果您希望使用另一個名稱來存取聚合函數的結果，可以指定自己的別名：

```php
$posts = Post::withSum('comments as total_comments', 'votes')->get();

foreach ($posts as $post) {
    echo $post->total_comments;
}
```

與 `loadCount` 方法類似，這些方法也提供延遲版本。這些額外的聚合操作可以在已經被取得的 Eloquent 模型上執行：

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
### 在 Morph To 關聯上計算關聯模型數量

如果您想要預先載入一個 "morph to" 關聯，以及該關聯可能回傳的各種實體的相關模型數量，您可以將 `with` 方法與 `morphTo` 關聯的 `morphWithCount` 方法結合使用。

在這個範例中，假設 `Photo` 和 `Post` 模型可以建立 `ActivityFeed` 模型。我們假設 `ActivityFeed` 模型定義了一個名為 `parentable` 的 "morph to" 關聯，讓我們可以為給定的 `ActivityFeed` 實體取得其父層 `Photo` 或 `Post` 模型。此外，假設 `Photo` 模型「具有許多 (have many)」`Tag` 模型，而 `Post` 模型「具有許多 (have many)」`Comment` 模型。

現在，想像我們想要取得 `ActivityFeed` 實體，並為每個 `ActivityFeed` 實體預先載入 `parentable` 父模型。此外，我們還想要取得與每個父相片相關的標籤數量，以及與每個父文章相關的評論數量：

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
#### 延遲數量載入

假設我們已經取得了一組 `ActivityFeed` 模型，現在我們想要載入與這些活動餵送 (activity feeds) 相關的各種 `parentable` 模型的巢狀關聯數量。您可以使用 `loadMorphCount` 方法來達成此目的：

```php
$activities = ActivityFeed::with('parentable')->get();

$activities->loadMorphCount('parentable', [
    Photo::class => ['tags'],
    Post::class => ['comments'],
]);
```

<a name="eager-loading"></a>
## 預先載入

當將 Eloquent 關聯作為屬性存取時，相關模型會被「延遲載入(lazy loaded)」。這意味著直到您第一次存取該屬性之前，關聯數據實際上並不會被載入。然而，Eloquent 可以在您查詢父模型時「預先載入(eager load)」關聯。預先載入可以緩解「N + 1」查詢問題。為了說明 N + 1 查詢問題，請考慮一個 "belongs to" 於 `Author` 模型的 `Book` 模型：

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

現在，讓我們取得所有的書籍及其作者：

```php
use App\Models\Book;

$books = Book::all();

foreach ($books as $book) {
    echo $book->author->name;
}
```

此迴圈將執行一次查詢以取得資料庫表中的所有書籍，然後針對每本書再執行一次查詢以取得該書的作者。因此，如果我們有 25 本書，上述程式碼將執行 26 次查詢：一次用於原始書籍，以及 25 次額外查詢以取得每本書的作者。

幸運的是，我們可以使用預先載入將此操作減少到僅僅兩次查詢。在建立查詢時，您可以使用 `with` 方法指定哪些關聯應該被預先載入：

```php
$books = Book::with('author')->get();

foreach ($books as $book) {
    echo $book->author->name;
}
```

對於此操作，僅會執行兩次查詢 — 一次查詢以取得所有書籍，以及一次查詢以取得所有書籍的所有作者：

```sql
select * from books

select * from authors where id in (1, 2, 3, 4, 5, ...)
```


<a name="eager-loading-multiple-relationships"></a>
#### 預先載入多個關聯

有時您可能需要預先載入多個不同的關聯。若要這樣做，只需將關聯陣列傳遞給 `with` 方法：

```php
$books = Book::with(['author', 'publisher'])->get();
```


<a name="nested-eager-loading"></a>
#### 巢狀預先載入

若要預先載入關聯的關聯，您可以使用「點」語法。例如，讓我們預先載入所有書籍的作者以及所有作者的個人聯絡人：

```php
$books = Book::with('author.contacts')->get();
```

或者，您也可以透過向 `with` 方法提供巢狀陣列來指定巢狀預先載入的關聯，這在預先載入多個巢狀關聯時會非常方便：

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

如果您想要預先載入 `morphTo` 關聯，以及該關聯可能返回之各種實體上的巢狀關聯，您可以使用 `with` 方法結合 `morphTo` 關聯的 `morphWith` 方法。為了說明此方法，讓我們考慮以下模型：

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

在這個範例中，假設 `Event`、`Photo` 和 `Post` 模型可以建立 `ActivityFeed` 模型。此外，假設 `Event` 模型屬於 `Calendar` 模型，`Photo` 模型與 `Tag` 模型相關聯，而 `Post` 模型屬於 `Author` 模型。

使用這些模型定義和關聯，我們可以取得 `ActivityFeed` 模型實例，並預先載入所有 `parentable` 模型及其各自的巢狀關聯：

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

您可能並不總是需要所取得關聯中的每個欄位。因此，Eloquent 允許您指定想要取得的關聯欄位：

```php
$books = Book::with('author:id,name,book_id')->get();
```

> [!WARNING]
> 使用此功能時，您應該始終在想要取得的欄位列表中包含 `id` 欄位以及任何相關的外鍵欄位。


<a name="eager-loading-by-default"></a>
#### 預設預先載入

有時您可能希望在取得模型時始終載入某些關聯。若要實現此目的，您可以在模型上定義 `$with` 屬性：

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

如果您想在單次查詢中從 `$with` 屬性中移除某個項目，可以使用 `without` 方法：

```php
$books = Book::without('author')->get();
```

如果您想在單次查詢中覆寫 `$with` 屬性中的所有項目，可以使用 `withOnly` 方法：

```php
$books = Book::withOnly('genre')->get();
```

<a name="constraining-eager-loads"></a>
### 限制預先載入

有時候您可能希望預先載入關聯，但同時也想為預先載入的查詢指定額外的查詢條件。您可以將關聯陣列傳遞給 `with` 方法來實現此功能，其中陣列的鍵（key）是關聯名稱，而陣列的值（value）是一個用於為預先載入查詢添加額外限制的閉包（closure）：

```php
use App\Models\User;

$users = User::with(['posts' => function ($query) {
    $query->where('title', 'like', '%code%');
}])->get();
```

在這個範例中，Eloquent 只會預先載入 `title` 欄位包含 `code` 字眼的貼文。您可以呼叫其他 [查詢建構器 (query builder)](/docs/{{version}}/queries) 方法來進一步自定義預先載入操作：

```php
$users = User::with(['posts' => function ($query) {
    $query->orderBy('created_at', 'desc');
}])->get();
```


<a name="constraining-eager-loading-of-morph-to-relationships"></a>
#### 限制 `morphTo` 關聯的預先載入

如果您正在預先載入 `morphTo` 關聯，Eloquent 會執行多個查詢來獲取每種類型的關聯模型。您可以使用 `MorphTo` 關聯的 `constrain` 方法為這些查詢中的每一個添加額外限制：

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

在這個範例中，Eloquent 只會預先載入尚未隱藏的貼文以及 `type` 值为 "educational" 的影片。


<a name="constraining-eager-loads-with-relationship-existence"></a>
#### 結合關聯存在性限制預先載入

有時候您可能需要檢查關聯是否存在，同時根據相同的條件載入該關聯。例如，您可能希望僅檢索具有符合特定查詢條件之子 `Post` 模型的 `User` 模型，同時預先載入這些符合條件的貼文。您可以使用 `withWhereHas` 方法來實現此功能：

```php
use App\Models\User;

$users = User::withWhereHas('posts', function ($query) {
    $query->where('featured', true);
})->get();
```


<a name="lazy-eager-loading"></a>
### 延遲預先載入

有時候您可能需要在父模型已被檢索後才預先載入關聯。例如，如果您需要動態決定是否載入關聯模型時，這會非常有用：

```php
use App\Models\Book;

$books = Book::all();

if ($condition) {
    $books->load('author', 'publisher');
}
```

如果您需要在預先載入查詢中設定額外的查詢限制，可以傳遞一個以您要載入的關聯為鍵的陣列。陣列的值應該是接收查詢實例的閉包實例：

```php
$author->load(['books' => function ($query) {
    $query->orderBy('published_date', 'asc');
}]);
```

若要僅在關聯尚未被載入時才載入，請使用 `loadMissing` 方法：

```php
$book->loadMissing('author');
```


<a name="nested-lazy-eager-loading-morphto"></a>
#### 巢狀延遲預先載入與 `morphTo`

如果您想要預先載入 `morphTo` 關聯，以及該關聯可能返回之各種實體上的巢狀關聯，您可以使用 `loadMorph` 方法。

此方法接收 `morphTo` 關聯名稱作為第一個引數，以及模型與關聯配對的陣列作為第二個引數。為了說明此方法，讓我們考慮以下模型：

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

在這個範例中，假設 `Event`、`Photo` 和 `Post` 模型可能會建立 `ActivityFeed` 模型。此外，假設 `Event` 模型屬於一個 `Calendar` 模型，`Photo` 模型與 `Tag` 模型相關聯，而 `Post` 模型屬於一個 `Author` 模型。

使用這些模型定義和關聯，我們可以檢索 `ActivityFeed` 模型實例，並預先載入所有 `parentable` 模型及其各自的巢狀關聯：

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
> 此功能目前處於 beta 階段，旨在收集社群回饋。即使在修正版本 (patch releases) 中，此功能的行為和功能也可能會有所變動。

在許多情況下，Laravel 可以自動預先載入您存取的關聯。要啟用自動預先載入，您應該在應用程式 `AppServiceProvider` 的 `boot` 方法中呼叫 `Model::automaticallyEagerLoadRelationships` 方法：

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

當此功能啟用時，Laravel 會嘗試自動載入您存取但先前尚未載入的任何關聯。例如，考慮以下情境：

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

通常情況下，上述程式碼會為每個使用者執行一次查詢以檢索其貼文，並為每篇貼文執行一次查詢以檢索其評論。然而，當 `automaticallyEagerLoadRelationships` 功能啟用時，當您嘗試存取任何已檢索使用者的貼文時，Laravel 會自動為使用者集合中的所有使用者 [延遲預先載入](#lazy-eager-loading) 貼文。同樣地，當您嘗試存取任何已檢索貼文的評論時，所有原本被檢索的貼文之評論都將被延遲預先載入。

如果您不想全域啟用自動預先載入，您仍然可以透過在集合上呼叫 `withRelationshipAutoloading` 方法，為單個 Eloquent 集合實例啟用此功能：

```php
$users = User::where('vip', true)->get();

return $users->withRelationshipAutoloading();
```

<a name="preventing-lazy-loading"></a>
### 防止延遲載入

正如先前所討論的，預先載入關聯通常能為您的應用程式提供顯著的效能提升。因此，如果您希望的話，可以指示 Laravel 始終防止關聯的延遲載入。要達成此目的，您可以調用 Eloquent 模型基底類別所提供的 `preventLazyLoading` 方法。通常，您應該在應用程式 `AppServiceProvider` 類別的 `boot` 方法中調用此方法。

`preventLazyLoading` 方法接受一個可選的布林值參數，用以指示是否應防止延遲載入。例如，您可能希望僅在非正式環境中禁用延遲載入，以便即使正式環境的程式碼中意外出現了延遲載入的關聯，正式環境仍能繼續正常運作：

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

在防止延遲載入後，當您的應用程式嘗試延遲載入任何 Eloquent 關聯時，Eloquent 將會拋出 `Illuminate\Database\LazyLoadingViolationException` 異常。

您可以使用 `handleLazyLoadingViolationsUsing` 方法來自定義延遲載入違規的行為。例如，使用此方法，您可以指示將延遲載入違規僅記錄在日誌中，而不是透過異常來中斷應用程式的執行：

```php
Model::handleLazyLoadingViolationUsing(function (Model $model, string $relation) {
    $class = $model::class;

    info("Attempted to lazy load [{$relation}] on model [{$class}].");
});
```

<a name="inserting-and-updating-related-models"></a>
## 插入與更新關聯模型


<a name="the-save-method"></a>
### `save` 方法

Eloquent 提供了方便的方法將新模型新增至關聯中。例如，您可能需要為一篇文章新增一則評論。您可以透過關聯的 `save` 方法來插入該評論，而不需要手動設定 `Comment` 模型的 `post_id` 屬性：

```php
use App\Models\Comment;
use App\Models\Post;

$comment = new Comment(['message' => 'A new comment.']);

$post = Post::find(1);

$post->comments()->save($comment);
```

請注意，我們並非將 `comments` 關聯視為動態屬性來存取，而是呼叫 `comments` 方法以取得該關聯的實例。`save` 方法會自動將適當的 `post_id` 值新增至新的 `Comment` 模型中。

如果您需要儲存多個關聯模型，可以使用 `saveMany` 方法：

```php
$post = Post::find(1);

$post->comments()->saveMany([
    new Comment(['message' => 'A new comment.']),
    new Comment(['message' => 'Another new comment.']),
]);
```

`save` 與 `saveMany` 方法會將給定的模型實例持久化，但不會將新持久化的模型新增至已載入到父模型中的記憶體內關聯中。如果您打算在執行 `save` 或 `saveMany` 方法後存取該關聯，您可能需要使用 `refresh` 方法來重新載入模型及其關聯：

```php
$post->comments()->save($comment);

$post->refresh();

// All comments, including the newly saved comment...
$post->comments;
```


<a name="the-push-method"></a>
#### 遞迴儲存模型與關聯

如果您想要 `save` 您的模型及其所有相關聯的關聯，可以使用 `push` 方法。在這個範例中，`Post` 模型將與其評論以及評論的作者一起被儲存：

```php
$post = Post::find(1);

$post->comments[0]->message = 'Message';
$post->comments[0]->author->name = 'Author Name';

$post->push();
```

`pushQuietly` 方法可用於儲存模型及其相關聯的關聯，而不會觸發任何事件：

```php
$post->pushQuietly();
```


<a name="the-create-method"></a>
### `create` 方法

除了 `save` 和 `saveMany` 方法外，您還可以使用 `create` 方法。該方法接受一個屬性陣列，建立一個模型並將其插入資料庫。`save` 與 `create` 的區別在於，`save` 接受一個完整的 Eloquent 模型實例，而 `create` 則接受一個純 PHP `array`。新建立的模型將由 `create` 方法回傳：

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

您也可以使用 `findOrNew`、`firstOrNew`、`firstOrCreate` 與 `updateOrCreate` 方法來[在關聯上建立與更新模型](/docs/{{version}}/eloquent#upserts)。

> [!NOTE]
> 在使用 `create` 方法之前，請務必閱讀[大量賦值(mass assignment)](/docs/{{version}}/eloquent#mass-assignment)文件。


<a name="updating-belongs-to-relationships"></a>
### Belongs To 關聯

如果您想將子模型指定給一個新的父模型，可以使用 `associate` 方法。在這個範例中，`User` 模型定義了一個對 `Account` 模型的 `belongsTo` 關聯。此 `associate` 方法會設定子模型上的外鍵：

```php
use App\Models\Account;

$account = Account::find(10);

$user->account()->associate($account);

$user->save();
```

若要從子模型中移除父模型，可以使用 `dissociate` 方法。此方法會將關聯的外鍵設定為 `null`：

```php
$user->account()->dissociate();

$user->save();
```

<a name="updating-many-to-many-relationships"></a>
### 多對多關聯


<a name="attaching-detaching"></a>
#### 附加 / 分離

Eloquent 還提供了許多方法，讓處理多對多關聯變得更加方便。例如，想像一個使用者可以擁有許多角色，而一個角色也可以被許多使用者共有。您可以使用 `attach` 方法，透過在關聯的中間表中插入一筆記錄，將角色附加至使用者：

```php
use App\Models\User;

$user = User::find(1);

$user->roles()->attach($roleId);
```

在將關聯附加到模型時，您也可以傳入一個陣列，將額外資料插入到中間表中：

```php
$user->roles()->attach($roleId, ['expires' => $expires]);
```

有時可能需要從使用者中移除角色。若要移除多對多關聯記錄，請使用 `detach` 方法。`detach` 方法會從中間表中刪除對應的記錄，但兩個模型仍會保留在資料庫中：

```php
// Detach a single role from the user...
$user->roles()->detach($roleId);

// Detach all roles from the user...
$user->roles()->detach();
```

為了方便起見，`attach` 和 `detach` 也接受 ID 陣列作為輸入：

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

您也可以使用 `sync` 方法來建立多對多關聯。`sync` 方法接受一個要放入中間表的 ID 陣列。任何不在該陣列中的 ID 都將從中間表中被移除。因此，在此操作完成後，中間表中將只存在該陣列中定義的 ID：

```php
$user->roles()->sync([1, 2, 3]);
```

您也可以在傳入 ID 的同時，傳入額外的中間表值：

```php
$user->roles()->sync([1 => ['expires' => true], 2, 3]);
```

如果您希望為每個同步的模型 ID 插入相同的中間表值，可以使用 `syncWithPivotValues` 方法：

```php
$user->roles()->syncWithPivotValues([1, 2, 3], ['active' => true]);
```

如果您不想分離那些不在給定陣列中的現有 ID，可以使用 `syncWithoutDetaching` 方法：

```php
$user->roles()->syncWithoutDetaching([1, 2, 3]);
```


<a name="toggling-associations"></a>
#### 切換關聯

多對多關聯還提供了一個 `toggle` 方法，可用於「切換」給定相關模型 ID 的附加狀態。如果給定的 ID 目前已附加，它將被分離；反之，如果目前處於分離狀態，它將被附加：

```php
$user->roles()->toggle([1, 2, 3]);
```

您也可以在傳入 ID 的同時，傳入額外的中間表值：

```php
$user->roles()->toggle([
    1 => ['expires' => true],
    2 => ['expires' => true],
]);
```


<a name="transactional-pivot-operations"></a>
#### 交易性樞紐操作

上述每項樞紐操作都提供了一個 `OrFail` 變體（例如 `attachOrFail`、`detachOrFail`、`syncOrFail`、`syncWithoutDetachingOrFail` 以及 `toggleOrFail`），它會將操作封裝在資料庫交易 (transaction) 中，因此如果拋出異常，所有變更都會自動回滾：

```php
$user->roles()->attachOrFail([1, 2, 3]);

$user->roles()->syncOrFail([1, 2, 3]);
```


<a name="updating-a-record-on-the-intermediate-table"></a>
#### 更新中間表記錄

如果您需要更新關聯中間表中的現有列，可以使用 `updateExistingPivot` 方法。此方法接受中間記錄的外鍵以及要更新的屬性陣列：

```php
$user = User::find(1);

$user->roles()->updateExistingPivot($roleId, [
    'active' => false,
]);
```

<a name="touching-parent-timestamps"></a>
## 更新父層時間戳記

當模型定義了對另一個模型的 `belongsTo` 或 `belongsToMany` 關聯時（例如一個屬於 `Post` 的 `Comment`），在更新子模型時同時更新父模型的時間戳記有時會很有幫助。

例如，當 `Comment` 模型被更新時，您可能希望自動「觸碰 (touch)」擁有它的 `Post` 模型的 `updated_at` 時間戳記，使其設定為目前的日期與時間。要實現這一點，您可以在子模型上使用 `Touches` 屬性，其中包含在子模型更新時應同步更新 `updated_at` 時間戳記的關聯名稱：

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
> 父模型的時間戳記僅在子模型使用 Eloquent 的 `save` 方法更新時才會被更新。