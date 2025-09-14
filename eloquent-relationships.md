# Eloquent: 關聯

- [簡介](#introduction)
- [定義關聯](#defining-relationships)
    - [一對一 / Has One](#one-to-one)
    - [一對多 / Has Many](#one-to-many)
    - [一對多 (反向) / Belongs To](#one-to-many-inverse)
    - [多個中的一個 / Has One of Many](#has-one-of-many)
    - [透過一個擁有多個 / Has One Through](#has-one-through)
    - [透過多個擁有多個 / Has Many Through](#has-many-through)
- [範圍關聯](#scoped-relationships)
- [多對多關聯](#many-to-many)
    - [擷取中間表格欄位](#retrieving-intermediate-table-columns)
    - [透過中間表格欄位篩選查詢](#filtering-queries-via-intermediate-table-columns)
    - [透過中間表格欄位排序查詢](#ordering-queries-via-intermediate-table-columns)
    - [定義自訂中間表格模型](#defining-custom-intermediate-table-models)
- [多型關聯](#polymorphic-relationships)
    - [一對一](#one-to-one-polymorphic-relations)
    - [一對多](#one-to-many-polymorphic-relations)
    - [多個中的一個](#one-of-many-polymorphic-relations)
    - [多對多](#many-to-many-polymorphic-relations)
    - [自訂多型類型](#custom-polymorphic-types)
- [動態關聯](#dynamic-relationships)
- [查詢關聯](#querying-relations)
    - [關聯方法與動態屬性](#relationship-methods-vs-dynamic-properties)
    - [查詢關聯是否存在](#querying-relationship-existence)
    - [查詢關聯是否不存在](#querying-relationship-absence)
    - [查詢 Morph To 關聯](#querying-morph-to-relationships)
- [聚合關聯模型](#aggregating-related-models)
    - [計算關聯模型數量](#counting-related-models)
    - [其他聚合函數](#other-aggregate-functions)
    - [計算 Morph To 關聯上的關聯模型數量](#counting-related-models-on-morph-to-relationships)
- [預載入 (Eager Loading)](#eager-loading)
    - [限制預載入](#constraining-eager-loads)
    - [延遲預載入 (Lazy Eager Loading)](#lazy-eager-loading)
    - [自動預載入](#automatic-eager-loading)
    - [防止延遲載入](#preventing-lazy-loading)
- [插入與更新關聯模型](#inserting-and-updating-related-models)
    - [`save` 方法](#the-save-method)
    - [`create` 方法](#the-create-method)
    - [Belongs To 關聯](#updating-belongs-to-relationships)
    - [多對多關聯](#updating-many-to-many-relationships)
- [更新父級時間戳記](#touching-parent-timestamps)

<a name="introduction"></a>
## 簡介

資料庫表格之間通常存在關聯。例如，一篇部落格文章可能有多個評論，或者一個訂單可能與下訂單的使用者相關。Eloquent 讓管理和操作這些關聯變得輕而易舉，並支援多種常見的關聯類型：

<div class="content-list" markdown="1">

- [一對一](#one-to-one)
- [一對多](#one-to-many)
- [多對多](#many-to-many)
- [透過一個擁有多個](#has-one-through)
- [透過多個擁有多個](#has-many-through)
- [一對一 (多型)](#one-to-one-polymorphic-relations)
- [一對多 (多型)](#one-to-many-polymorphic-relations)
- [多對多 (多型)](#many-to-many-polymorphic-relations)

</div>

<a name="defining-relationships"></a>
## 定義關聯

Eloquent 關聯是作為 Eloquent 模型類別上的方法來定義的。由於關聯也作為強大的 [查詢建構器](/docs/{{version}}/queries)，將關聯定義為方法提供了強大的方法鏈接和查詢功能。例如，我們可以在這個 `posts` 關聯上鏈接額外的查詢限制：

```php
$user->posts()->where('active', 1)->get();
```

但是，在深入使用關聯之前，讓我們先學習如何定義 Eloquent 支援的每種關聯類型。

<a name="one-to-one"></a>
### 一對一 / Has One

一對一關聯是一種非常基本的資料庫關聯類型。例如，一個 `User` 模型可能與一個 `Phone` 模型相關聯。為了定義這個關聯，我們將在 `User` 模型上放置一個 `phone` 方法。`phone` 方法應該呼叫 `hasOne` 方法並回傳其結果。`hasOne` 方法透過模型的 `Illuminate\Database\Eloquent\Model` 基底類別提供給您的模型：

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

傳遞給 `hasOne` 方法的第一個引數是相關模型類別的名稱。一旦定義了關聯，我們就可以使用 Eloquent 的動態屬性來擷取相關記錄。動態屬性允許您像存取模型上定義的屬性一樣存取關聯方法：

```php
$phone = User::find(1)->phone;
```

Eloquent 根據父模型名稱決定關聯的外鍵。在此情況下，`Phone` 模型會自動假定具有 `user_id` 外鍵。如果您希望覆寫此慣例，可以將第二個引數傳遞給 `hasOne` 方法：

```php
return $this->hasOne(Phone::class, 'foreign_key');
```

此外，Eloquent 假定外鍵的值應與父級的主鍵欄位匹配。換句話說，Eloquent 將在 `Phone` 記錄的 `user_id` 欄位中尋找使用者 `id` 欄位的值。如果您希望關聯使用 `id` 或模型 `$primaryKey` 屬性以外的主鍵值，您可以將第三個引數傳遞給 `hasOne` 方法：

```php
return $this->hasOne(Phone::class, 'foreign_key', 'local_key');
```

<a name="one-to-one-defining-the-inverse-of-the-relationship"></a>
#### 定義關聯的反向

因此，我們可以從 `User` 模型存取 `Phone` 模型。接下來，讓我們在 `Phone` 模型上定義一個關聯，讓它能夠存取擁有該手機的使用者。我們可以使用 `belongsTo` 方法來定義 `hasOne` 關聯的反向：

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

當呼叫 `user` 方法時，Eloquent 將嘗試尋找一個 `User` 模型，其 `id` 與 `Phone` 模型上的 `user_id` 欄位匹配。

Eloquent 透過檢查關聯方法的名稱並在方法名稱後加上 `_id` 來決定外鍵名稱。因此，在此範例中，Eloquent 假定 `Phone` 模型具有 `user_id` 欄位。但是，如果 `Phone` 模型上的外鍵不是 `user_id`，您可以將自訂鍵名作為第二個引數傳遞給 `belongsTo` 方法：

```php
/**
 * Get the user that owns the phone.
 */
public function user(): BelongsTo
{
    return $this->belongsTo(User::class, 'foreign_key');
}
```

如果父模型不使用 `id` 作為主鍵，或者您希望使用不同的欄位來尋找相關模型，您可以將第三個引數傳遞給 `belongsTo` 方法，指定父表格的自訂鍵：

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

一對多關聯用於定義一個模型是多個子模型的父級的關聯。例如，一篇部落格文章可以有無數個評論。與所有其他 Eloquent 關聯一樣，一對多關聯是透過在 Eloquent 模型上定義一個方法來定義的：

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

請記住，Eloquent 會自動為 `Comment` 模型決定正確的外鍵欄位。按照慣例，Eloquent 會取父模型的「蛇式命名」名稱並在其後加上 `_id`。因此，在此範例中，Eloquent 會假定 `Comment` 模型上的外鍵欄位是 `post_id`。

一旦定義了關聯方法，我們就可以透過存取 `comments` 屬性來存取相關評論的 [集合](/docs/{{version}}/eloquent-collections)。請記住，由於 Eloquent 提供了「動態關聯屬性」，我們可以像存取模型上定義的屬性一樣存取關聯方法：

```php
use App\Models\Post;

$comments = Post::find(1)->comments;

foreach ($comments as $comment) {
    // ...
}
```

由於所有關聯也作為查詢建構器，您可以透過呼叫 `comments` 方法並繼續將條件鏈接到查詢來為關聯查詢添加更多限制：

```php
$comment = Post::find(1)->comments()
    ->where('title', 'foo')
    ->first();
```

與 `hasOne` 方法一樣，您也可以透過將額外引數傳遞給 `hasMany` 方法來覆寫外鍵和本地鍵：

```php
return $this->hasMany(Comment::class, 'foreign_key');

return $this->hasMany(Comment::class, 'foreign_key', 'local_key');
```

<a name="automatically-hydrating-parent-models-on-children"></a>
#### 自動將父模型注入子模型

即使使用 Eloquent 預載入，如果您在遍歷子模型時嘗試從子模型存取父模型，也可能會出現「N + 1」查詢問題：

```php
$posts = Post::with('comments')->get();

foreach ($posts as $post) {
    foreach ($post->comments as $comment) {
        echo $comment->post->title;
    }
}
```

在上面的範例中，引入了「N + 1」查詢問題，因為即使為每個 `Post` 模型預載入了評論，Eloquent 也不會自動將父 `Post` 注入每個子 `Comment` 模型。

如果您希望 Eloquent 自動將父模型注入其子模型，您可以在定義 `hasMany` 關聯時呼叫 `chaperone` 方法：

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

或者，如果您希望在執行時選擇自動父級注入，您可以在預載入關聯時呼叫 `chaperone` 模型：

```php
use App\Models\Post;

$posts = Post::with([
    'comments' => fn ($comments) => $comments->chaperone(),
])->get();
```

<a name="one-to-many-inverse"></a>
### 一對多 (反向) / Belongs To

現在我們可以存取一篇文章的所有評論，讓我們定義一個關聯，允許評論存取其父級文章。為了定義 `hasMany` 關聯的反向，請在子模型上定義一個呼叫 `belongsTo` 方法的關聯方法：

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

一旦定義了關聯，我們就可以透過存取 `post`「動態關聯屬性」來擷取評論的父級文章：

```php
use App\Models\Comment;

$comment = Comment::find(1);

return $comment->post->title;
```

在上面的範例中，Eloquent 將嘗試尋找一個 `Post` 模型，其 `id` 與 `Comment` 模型上的 `post_id` 欄位匹配。

Eloquent 透過檢查關聯方法的名稱，並在方法名稱後加上 `_` 和父模型主鍵欄位的名稱來決定預設的外鍵名稱。因此，在此範例中，Eloquent 將假定 `comments` 表格上 `Post` 模型的外鍵是 `post_id`。

但是，如果您的關聯外鍵不遵循這些慣例，您可以將自訂外鍵名稱作為第二個引數傳遞給 `belongsTo` 方法：

```php
/**
 * Get the post that owns the comment.
 */
public function post(): BelongsTo
{
    return $this->belongsTo(Post::class, 'foreign_key');
}
```

如果您的父模型不使用 `id` 作為主鍵，或者您希望使用不同的欄位來尋找相關模型，您可以將第三個引數傳遞給 `belongsTo` 方法，指定父表格的自訂鍵：

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

`belongsTo`、`hasOne`、`hasOneThrough` 和 `morphOne` 關聯允許您定義一個預設模型，如果給定的關聯為 `null`，則會回傳該模型。這種模式通常被稱為 [Null Object pattern](https://en.wikipedia.org/wiki/Null_Object_pattern)，可以幫助消除程式碼中的條件檢查。在以下範例中，如果沒有使用者附加到 `Post` 模型，`user` 關聯將回傳一個空的 `App\Models\User` 模型：

```php
/**
 * Get the author of the post.
 */
public function user(): BelongsTo
{
    return $this->belongsTo(User::class)->withDefault();
}
```

要使用屬性填充預設模型，您可以將陣列或閉包傳遞給 `withDefault` 方法：

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

當查詢「belongs to」關聯的子級時，您可以手動建構 `where` 子句以擷取相應的 Eloquent 模型：

```php
use App\Models\Post;

$posts = Post::where('user_id', $user->id)->get();
```

但是，您可能會發現使用 `whereBelongsTo` 方法更方便，它會自動為給定模型決定正確的關聯和外鍵：

```php
$posts = Post::whereBelongsTo($user)->get();
```

您也可以向 `whereBelongsTo` 方法提供一個 [集合](/docs/{{version}}/eloquent-collections) 實例。這樣做時，Laravel 將擷取屬於集合中任何父模型的模型：

```php
$users = User::where('vip', true)->get();

$posts = Post::whereBelongsTo($users)->get();
```

預設情況下，Laravel 將根據模型的類別名稱來決定與給定模型相關聯的關聯；但是，您可以透過將關聯名稱作為第二個引數提供給 `whereBelongsTo` 方法來手動指定關聯名稱：

```php
$posts = Post::whereBelongsTo($user, 'author')->get();
```

<a name="has-one-of-many"></a>
### 多個中的一個 / Has One of Many

有時一個模型可能有多個相關模型，但您希望輕鬆擷取關聯中「最新」或「最舊」的相關模型。例如，一個 `User` 模型可能與許多 `Order` 模型相關聯，但您希望定義一種方便的方式來與使用者下達的最新訂單互動。您可以使用 `hasOne` 關聯類型結合 `ofMany` 方法來實現此目的：

```php
/**
 * Get the user's most recent order.
 */
public function latestOrder(): HasOne
{
    return $this->hasOne(Order::class)->latestOfMany();
}
```

同樣地，您可以定義一個方法來擷取關聯中「最舊」或第一個相關模型：

```php
/**
 * Get the user's oldest order.
 */
public function oldestOrder(): HasOne
{
    return $this->hasOne(Order::class)->oldestOfMany();
}
```

預設情況下，`latestOfMany` 和 `oldestOfMany` 方法將根據模型的主鍵（必須可排序）擷取最新或最舊的相關模型。但是，有時您可能希望使用不同的排序條件從較大的關聯中擷取單個模型。

例如，使用 `ofMany` 方法，您可以擷取使用者最昂貴的訂單。`ofMany` 方法接受可排序欄位作為其第一個引數，以及在查詢相關模型時要套用的聚合函數 (`min` 或 `max`)：

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
> 由於 PostgreSQL 不支援對 UUID 欄位執行 `MAX` 函數，因此目前無法將多個中的一個關聯與 PostgreSQL UUID 欄位結合使用。

<a name="converting-many-relationships-to-has-one-relationships"></a>
#### 將「多個」關聯轉換為 Has One 關聯

通常，當使用 `latestOfMany`、`oldestOfMany` 或 `ofMany` 方法擷取單個模型時，您已經為同一個模型定義了「has many」關聯。為了方便起見，Laravel 允許您透過在關聯上呼叫 `one` 方法，輕鬆地將此關聯轉換為「has one」關聯：

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
#### 進階的多個中的一個關聯

可以建構更進階的「多個中的一個」關聯。例如，一個 `Product` 模型可能有多個相關的 `Price` 模型，即使發布了新價格，這些模型仍會保留在系統中。此外，產品的新價格資料可能會提前發布，以便透過 `published_at` 欄位在未來生效。

因此，總之，我們需要擷取最新發布的價格，其中發布日期不是未來。此外，如果兩個價格具有相同的發布日期，我們將優先選擇 ID 最大的價格。為了實現此目的，我們必須將一個陣列傳遞給 `ofMany` 方法，該陣列包含決定最新價格的可排序欄位。此外，一個閉包將作為第二個引數提供給 `ofMany` 方法。此閉包將負責向關聯查詢添加額外的發布日期限制：

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
### 透過一個擁有多個 / Has One Through

「has-one-through」關聯定義了與另一個模型的一對一關聯。但是，此關聯表示宣告模型可以透過第三個模型與另一個模型的一個實例匹配。

例如，在車輛維修店應用程式中，每個 `Mechanic` 模型可能與一個 `Car` 模型相關聯，每個 `Car` 模型可能與一個 `Owner` 模型相關聯。雖然技工和車主在資料庫中沒有直接關聯，但技工可以透過 `Car` 模型存取車主。讓我們看看定義此關聯所需的表格：

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

現在我們已經檢查了關聯的表格結構，讓我們在 `Mechanic` 模型上定義關聯：

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

傳遞給 `hasOneThrough` 方法的第一個引數是我們希望存取的最終模型名稱，而第二個引數是中間模型名稱。

或者，如果相關關聯已在關聯中涉及的所有模型上定義，您可以透過呼叫 `through` 方法並提供這些關聯的名稱來流暢地定義「has-one-through」關聯。例如，如果 `Mechanic` 模型具有 `cars` 關聯，並且 `Car` 模型具有 `owner` 關聯，您可以像這樣定義連接技工和車主的「has-one-through」關聯：

```php
// String based syntax...
return $this->through('cars')->has('owner');

// Dynamic syntax...
return $this->throughCars()->hasOwner();
```

<a name="has-one-through-key-conventions"></a>
#### 鍵值慣例

執行關聯查詢時將使用典型的 Eloquent 外鍵慣例。如果您希望自訂關聯的鍵，您可以將它們作為第三個和第四個引數傳遞給 `hasOneThrough` 方法。第三個引數是中間模型上的外鍵名稱。第四個引數是最終模型上的外鍵名稱。第五個引數是本地鍵，而第六個引數是中間模型的本地鍵：

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

或者，如前所述，如果相關關聯已在關聯中涉及的所有模型上定義，您可以透過呼叫 `through` 方法並提供這些關聯的名稱來流暢地定義「has-one-through」關聯。這種方法提供了重複使用現有關聯上已定義的鍵值慣例的優勢：

```php
// String based syntax...
return $this->through('cars')->has('owner');

// Dynamic syntax...
return $this->throughCars()->hasOwner();
```

<a name="has-many-through"></a>
### 透過多個擁有多個 / Has Many Through

「has-many-through」關聯提供了一種方便的方式，透過中間關聯來存取遠端關聯。例如，假設我們正在建構一個部署平台，例如 [Laravel Cloud](https://cloud.laravel.com)。一個 `Application` 模型可能透過中間的 `Environment` 模型存取許多 `Deployment` 模型。使用此範例，您可以輕鬆地為給定的應用程式收集所有部署。讓我們看看定義此關聯所需的表格：

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

現在我們已經檢查了關聯的表格結構，讓我們在 `Application` 模型上定義關聯：

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

傳遞給 `hasManyThrough` 方法的第一個引數是我們希望存取的最終模型名稱，而第二個引數是中間模型名稱。

或者，如果相關關聯已在關聯中涉及的所有模型上定義，您可以透過呼叫 `through` 方法並提供這些關聯的名稱來流暢地定義「has-many-through」關聯。例如，如果 `Application` 模型具有 `environments` 關聯，並且 `Environment` 模型具有 `deployments` 關聯，您可以像這樣定義連接應用程式和部署的「has-many-through」關聯：

```php
// String based syntax...
return $this->through('environments')->has('deployments');

// Dynamic syntax...
return $this->throughEnvironments()->hasDeployments();
```

儘管 `Deployment` 模型表格不包含 `application_id` 欄位，但 `hasManyThrough` 關聯透過 `$application->deployments` 提供對應用程式部署的存取。為了擷取這些模型，Eloquent 會檢查中間 `Environment` 模型表格上的 `application_id` 欄位。找到相關的環境 ID 後，它們將用於查詢 `Deployment` 模型表格。

<a name="has-many-through-key-conventions"></a>
#### 鍵值慣例

執行關聯查詢時將使用典型的 Eloquent 外鍵慣例。如果您希望自訂關聯的鍵，您可以將它們作為第三個和第四個引數傳遞給 `hasManyThrough` 方法。第三個引數是中間模型上的外鍵名稱。第四個引數是最終模型上的外鍵名稱。第五個引數是本地鍵，而第六個引數是中間模型的本地鍵：

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

或者，如前所述，如果相關關聯已在關聯中涉及的所有模型上定義，您可以透過呼叫 `through` 方法並提供這些關聯的名稱來流暢地定義「has-many-through」關聯。這種方法提供了重複使用現有關聯上已定義的鍵值慣例的優勢：

```php
// String based syntax...
return $this->through('environments')->has('deployments');

// Dynamic syntax...
return $this->throughEnvironments()->hasDeployments();
```

<a name="scoped-relationships"></a>
### 範圍關聯

通常會向模型添加額外的方法來限制關聯。例如，您可能會向 `User` 模型添加一個 `featuredPosts` 方法，該方法會使用額外的 `where` 限制來限制更廣泛的 `posts` 關聯：

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

但是，如果您嘗試透過 `featuredPosts` 方法建立模型，其 `featured` 屬性將不會設定為 `true`。如果您希望透過關聯方法建立模型，並指定應添加到透過該關聯建立的所有模型的屬性，您可以在建構關聯查詢時使用 `withAttributes` 方法：

```php
/**
 * Get the user's featured posts.
 */
public function featuredPosts(): HasMany
{
    return $this->posts()->withAttributes(['featured' => true]);
}
```

`withAttributes` 方法將使用給定的屬性向查詢添加 `where` 條件，並且還會將給定的屬性添加到透過關聯方法建立的任何模型：

```php
$post = $user->featuredPosts()->create(['title' => 'Featured Post']);

$post->featured; // true
```

要指示 `withAttributes` 方法不向查詢添加 `where` 條件，您可以將 `asConditions` 引數設定為 `false`：

```php
return $this->posts()->withAttributes(['featured' => true], asConditions: false);
```

<a name="many-to-many"></a>
## 多對多關聯

多對多關聯比 `hasOne` 和 `hasMany` 關聯稍微複雜一些。多對多關聯的一個範例是使用者擁有多個角色，並且這些角色也由應用程式中的其他使用者共享。例如，一個使用者可能被分配「作者」和「編輯」的角色；但是，這些角色也可能被分配給其他使用者。因此，一個使用者擁有多個角色，一個角色擁有多個使用者。

<a name="many-to-many-table-structure"></a>
#### 表格結構

為了定義此關聯，需要三個資料庫表格：`users`、`roles` 和 `role_user`。`role_user` 表格是根據相關模型名稱的字母順序派生而來的，並包含 `user_id` 和 `role_id` 欄位。此表格用作連接使用者和角色的中間表格。

請記住，由於一個角色可以屬於多個使用者，我們不能簡單地在 `roles` 表格上放置一個 `user_id` 欄位。這將意味著一個角色只能屬於一個使用者。為了支援將角色分配給多個使用者，需要 `role_user` 表格。我們可以將關聯的表格結構總結如下：

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

多對多關聯是透過編寫一個回傳 `belongsToMany` 方法結果的方法來定義的。`belongsToMany` 方法由所有應用程式 Eloquent 模型使用的 `Illuminate\Database\Eloquent\Model` 基底類別提供。例如，讓我們在 `User` 模型上定義一個 `roles` 方法。傳遞給此方法的第一個引數是相關模型類別的名稱：

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

由於所有關聯也作為查詢建構器，您可以透過呼叫 `roles` 方法並繼續將條件鏈接到查詢來為關聯查詢添加更多限制：

```php
$roles = User::find(1)->roles()->orderBy('name')->get();
```

為了決定關聯中間表格的表格名稱，Eloquent 將按字母順序連接兩個相關模型名稱。但是，您可以自由覆寫此慣例。您可以透過將第二個引數傳遞給 `belongsToMany` 方法來實現：

```php
return $this->belongsToMany(Role::class, 'role_user');
```

除了自訂中間表格的名稱之外，您還可以透過將額外引數傳遞給 `belongsToMany` 方法來自訂表格上鍵的欄位名稱。第三個引數是您正在定義關聯的模型的外鍵名稱，而第四個引數是您正在連接的模型的外鍵名稱：

```php
return $this->belongsToMany(Role::class, 'role_user', 'user_id', 'role_id');
```

<a name="many-to-many-defining-the-inverse-of-the-relationship"></a>
#### 定義關聯的反向

為了定義多對多關聯的「反向」，您應該在相關模型上定義一個方法，該方法也回傳 `belongsToMany` 方法的結果。為了完成我們的使用者/角色範例，讓我們在 `Role` 模型上定義 `users` 方法：

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

如您所見，關聯的定義與其 `User` 模型對應項完全相同，只是引用了 `App\Models\User` 模型。由於我們重複使用了 `belongsToMany` 方法，因此在定義多對多關聯的「反向」時，所有常見的表格和鍵自訂選項都可用。

<a name="retrieving-intermediate-table-columns"></a>
### 擷取中間表格欄位

如您所知，處理多對多關聯需要中間表格的存在。Eloquent 提供了一些非常有用的方法來與此表格互動。例如，假設我們的 `User` 模型與許多 `Role` 模型相關聯。存取此關聯後，我們可以透過模型上的 `pivot` 屬性存取中間表格：

```php
use App\Models\User;

$user = User::find(1);

foreach ($user->roles as $role) {
    echo $role->pivot->created_at;
}
```

請注意，我們擷取的每個 `Role` 模型都會自動分配一個 `pivot` 屬性。此屬性包含一個代表中間表格的模型。

預設情況下，`pivot` 模型上只會存在模型鍵。如果您的中間表格包含額外屬性，您必須在定義關聯時指定它們：

```php
return $this->belongsToMany(Role::class)->withPivot('active', 'created_by');
```

如果您希望您的中間表格具有由 Eloquent 自動維護的 `created_at` 和 `updated_at` 時間戳記，請在定義關聯時呼叫 `withTimestamps` 方法：

```php
return $this->belongsToMany(Role::class)->withTimestamps();
```

> [!WARNING]
> 使用 Eloquent 自動維護時間戳記的中間表格必須同時具有 `created_at` 和 `updated_at` 時間戳記欄位。

<a name="customizing-the-pivot-attribute-name"></a>
#### 自訂 `pivot` 屬性名稱

如前所述，可以透過 `pivot` 屬性存取中間表格的屬性。但是，您可以自由自訂此屬性的名稱，以更好地反映其在應用程式中的用途。

例如，如果您的應用程式包含可以訂閱 Podcast 的使用者，您可能在使用者和 Podcast 之間存在多對多關聯。在這種情況下，您可能希望將中間表格屬性重新命名為 `subscription` 而不是 `pivot`。這可以在定義關聯時使用 `as` 方法來完成：

```php
return $this->belongsToMany(Podcast::class)
    ->as('subscription')
    ->withTimestamps();
```

一旦指定了自訂中間表格屬性，您就可以使用自訂名稱存取中間表格資料：

```php
$users = User::with('podcasts')->get();

foreach ($users->flatMap->podcasts as $podcast) {
    echo $podcast->subscription->created_at;
}
```

<a name="filtering-queries-via-intermediate-table-columns"></a>
### 透過中間表格欄位篩選查詢

您還可以在定義關聯時使用 `wherePivot`、`wherePivotIn`、`wherePivotNotIn`、`wherePivotBetween`、`wherePivotNotBetween`、`wherePivotNull` 和 `wherePivotNotNull` 方法來篩選 `belongsToMany` 關聯查詢回傳的結果：

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

`wherePivot` 會向查詢添加一個 where 子句限制，但不會在透過定義的關聯建立新模型時添加指定的值。如果您需要同時查詢和建立具有特定 pivot 值的關聯，您可以使用 `withPivotValue` 方法：

```php
return $this->belongsToMany(Role::class)
    ->withPivotValue('approved', 1);
```

<a name="ordering-queries-via-intermediate-table-columns"></a>
### 透過中間表格欄位排序查詢

您可以使用 `orderByPivot` 方法來排序 `belongsToMany` 關聯查詢回傳的結果。在以下範例中，我們將擷取使用者所有最新的徽章：

```php
return $this->belongsToMany(Badge::class)
    ->where('rank', 'gold')
    ->orderByPivot('created_at', 'desc');
```

<a name="defining-custom-intermediate-table-models"></a>
### 定義自訂中間表格模型

如果您希望定義一個自訂模型來表示多對多關聯的中間表格，您可以在定義關聯時呼叫 `using` 方法。自訂 pivot 模型讓您有機會在 pivot 模型上定義額外的行為，例如方法和型別轉換。

自訂多對多 pivot 模型應擴展 `Illuminate\Database\Eloquent\Relations\Pivot` 類別，而自訂多型多對多 pivot 模型應擴展 `Illuminate\Database\Eloquent\Relations\MorphPivot` 類別。例如，我們可以定義一個使用自訂 `RoleUser` pivot 模型的 `Role` 模型：

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

在定義 `RoleUser` 模型時，您應該擴展 `Illuminate\Database\Eloquent\Relations\Pivot` 類別：

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
> Pivot 模型不能使用 `SoftDeletes` trait。如果您需要軟刪除 pivot 記錄，請考慮將您的 pivot 模型轉換為實際的 Eloquent 模型。

<a name="custom-pivot-models-and-incrementing-ids"></a>
#### 自訂 Pivot 模型與遞增 ID

如果您定義了一個使用自訂 pivot 模型的多對多關聯，並且該 pivot 模型具有自動遞增的主鍵，您應該確保您的自訂 pivot 模型類別定義了一個設定為 `true` 的 `incrementing` 屬性。

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

多型關聯允許子模型使用單一關聯屬於多種類型的模型。例如，想像您正在建構一個允許使用者分享部落格文章和影片的應用程式。在此類應用程式中，一個 `Comment` 模型可能同時屬於 `Post` 和 `Video` 模型。

<a name="one-to-one-polymorphic-relations"></a>
### 一對一 (多型)

<a name="one-to-one-polymorphic-table-structure"></a>
#### 表格結構

一對一多型關聯與典型的一對一關聯相似；但是，子模型可以使用單一關聯屬於多種類型的模型。例如，部落格 `Post` 和 `User` 可能與 `Image` 模型共享多型關聯。使用一對一多型關聯允許您擁有一個單一的唯一圖片表格，該表格可以與文章和使用者相關聯。首先，讓我們檢查表格結構：

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

請注意 `images` 表格上的 `imageable_id` 和 `imageable_type` 欄位。`imageable_id` 欄位將包含文章或使用者的 ID 值，而 `imageable_type` 欄位將包含父模型的類別名稱。`imageable_type` 欄位由 Eloquent 用於決定在存取 `imageable` 關聯時要回傳哪種「類型」的父模型。在此情況下，該欄位將包含 `App\Models\Post` 或 `App\Models\User`。

<a name="one-to-one-polymorphic-model-structure"></a>
#### 模型結構

接下來，讓我們檢查建構此關聯所需的模型定義：

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
#### 擷取關聯

一旦定義了資料庫表格和模型，您就可以透過模型存取關聯。例如，要擷取文章的圖片，我們可以存取 `image` 動態關聯屬性：

```php
use App\Models\Post;

$post = Post::find(1);

$image = $post->image;
```

您可以透過存取執行 `morphTo` 呼叫的方法名稱來擷取多型模型的父級。在此情況下，這是 `Image` 模型上的 `imageable` 方法。因此，我們將該方法作為動態關聯屬性存取：

```php
use App\Models\Image;

$image = Image::find(1);

$imageable = $image->imageable;
```

`Image` 模型上的 `imageable` 關聯將回傳 `Post` 或 `User` 實例，具體取決於哪種類型的模型擁有該圖片。

<a name="morph-one-to-one-key-conventions"></a>
#### 鍵值慣例

如有必要，您可以指定多型子模型使用的「id」和「type」欄位名稱。如果您這樣做，請確保始終將關聯名稱作為第一個引數傳遞給 `morphTo` 方法。通常，此值應與方法名稱匹配，因此您可以使用 PHP 的 `__FUNCTION__` 常數：

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
### 一對多 (多型)

<a name="one-to-many-polymorphic-table-structure"></a>
#### 表格結構

一對多多型關聯與典型的一對多關聯相似；但是，子模型可以使用單一關聯屬於多種類型的模型。例如，想像您的應用程式使用者可以「評論」文章和影片。使用多型關聯，您可以使用單一 `comments` 表格來包含文章和影片的評論。首先，讓我們檢查建構此關聯所需的表格結構：

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

接下來，讓我們檢查建構此關聯所需的模型定義：

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
#### 擷取關聯

一旦定義了資料庫表格和模型，您就可以透過模型的動態關聯屬性存取關聯。例如，要存取文章的所有評論，我們可以使用 `comments` 動態屬性：

```php
use App\Models\Post;

$post = Post::find(1);

foreach ($post->comments as $comment) {
    // ...
}
```

您也可以透過存取執行 `morphTo` 呼叫的方法名稱來擷取多型子模型的父級。在此情況下，這是 `Comment` 模型上的 `commentable` 方法。因此，我們將該方法作為動態關聯屬性存取，以存取評論的父模型：

```php
use App\Models\Comment;

$comment = Comment::find(1);

$commentable = $comment->commentable;
```

`Comment` 模型上的 `commentable` 關聯將回傳 `Post` 或 `Video` 實例，具體取決於哪種類型的模型是評論的父級。

<a name="polymorphic-automatically-hydrating-parent-models-on-children"></a>
#### 自動將父模型注入子模型

即使使用 Eloquent 預載入，如果您在遍歷子模型時嘗試從子模型存取父模型，也可能會出現「N + 1」查詢問題：

```php
$posts = Post::with('comments')->get();

foreach ($posts as $post) {
    foreach ($post->comments as $comment) {
        echo $comment->commentable->title;
    }
}
```

在上面的範例中，引入了「N + 1」查詢問題，因為即使為每個 `Post` 模型預載入了評論，Eloquent 也不會自動將父 `Post` 注入每個子 `Comment` 模型。

如果您希望 Eloquent 自動將父模型注入其子模型，您可以在定義 `morphMany` 關聯時呼叫 `chaperone` 方法：

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

或者，如果您希望在執行時選擇自動父級注入，您可以在預載入關聯時呼叫 `chaperone` 模型：

```php
use App\Models\Post;

$posts = Post::with([
    'comments' => fn ($comments) => $comments->chaperone(),
])->get();
```

<a name="one-of-many-polymorphic-relations"></a>
### 多個中的一個 (多型)

有時一個模型可能有多個相關模型，但您希望輕鬆擷取關聯中「最新」或「最舊」的相關模型。例如，一個 `User` 模型可能與許多 `Image` 模型相關聯，但您希望定義一種方便的方式來與使用者上傳的最新圖片互動。您可以使用 `morphOne` 關聯類型結合 `ofMany` 方法來實現此目的：

```php
/**
 * Get the user's most recent image.
 */
public function latestImage(): MorphOne
{
    return $this->morphOne(Image::class, 'imageable')->latestOfMany();
}
```

同樣地，您可以定義一個方法來擷取關聯中「最舊」或第一個相關模型：

```php
/**
 * Get the user's oldest image.
 */
public function oldestImage(): MorphOne
{
    return $this->morphOne(Image::class, 'imageable')->oldestOfMany();
}
```

預設情況下，`latestOfMany` 和 `oldestOfMany` 方法將根據模型的主鍵（必須可排序）擷取最新或最舊的相關模型。但是，有時您可能希望使用不同的排序條件從較大的關聯中擷取單個模型。

例如，使用 `ofMany` 方法，您可以擷取使用者最「受歡迎」的圖片。`ofMany` 方法接受可排序欄位作為其第一個引數，以及在查詢相關模型時要套用的聚合函數 (`min` 或 `max`)：

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
> 可以建構更進階的「多個中的一個」關聯。有關更多資訊，請參閱 [多個中的一個關聯文件](#advanced-has-one-of-many-relationships)。

<a name="many-to-many-polymorphic-relations"></a>
### 多對多 (多型)

<a name="many-to-many-polymorphic-table-structure"></a>
#### 表格結構

多對多多型關聯比「morph one」和「morph many」關聯稍微複雜一些。例如，一個 `Post` 模型和 `Video` 模型可以與 `Tag` 模型共享多型關聯。在這種情況下使用多對多多型關聯將允許您的應用程式擁有一個單一的唯一標籤表格，該表格可以與文章或影片相關聯。首先，讓我們檢查建構此關聯所需的表格結構：

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
> 在深入了解多型多對多關聯之前，您可能會從閱讀典型 [多對多關聯](#many-to-many) 的文件獲益。

<a name="many-to-many-polymorphic-model-structure"></a>
#### 模型結構

接下來，我們準備在模型上定義關聯。`Post` 和 `Video` 模型都將包含一個 `tags` 方法，該方法呼叫基底 Eloquent 模型類別提供的 `morphToMany` 方法。

`morphToMany` 方法接受相關模型的名稱以及「關聯名稱」。根據我們分配給中間表格名稱的名稱及其包含的鍵，我們將關聯稱為「taggable」：

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
#### 定義關聯的反向

接下來，在 `Tag` 模型上，您應該為其每個可能的父模型定義一個方法。因此，在此範例中，我們將定義一個 `posts` 方法和一個 `videos` 方法。這兩個方法都應該回傳 `morphedByMany` 方法的結果。

`morphedByMany` 方法接受相關模型的名稱以及「關聯名稱」。根據我們分配給中間表格名稱的名稱及其包含的鍵，我們將關聯稱為「taggable」：

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
#### 擷取關聯

一旦定義了資料庫表格和模型，您就可以透過模型存取關聯。例如，要存取文章的所有標籤，您可以使用 `tags` 動態關聯屬性：

```php
use App\Models\Post;

$post = Post::find(1);

foreach ($post->tags as $tag) {
    // ...
}
```

您可以透過存取執行 `morphedByMany` 呼叫的方法名稱來擷取多型子模型的多型關聯的父級。在此情況下，這是 `Tag` 模型上的 `posts` 或 `videos` 方法：

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
### 自訂多型類型

預設情況下，Laravel 將使用完全限定的類別名稱來儲存相關模型的「類型」。例如，考慮上面一對多關聯的範例，其中 `Comment` 模型可能屬於 `Post` 或 `Video` 模型，預設的 `commentable_type` 將分別是 `App\Models\Post` 或 `App\Models\Video`。但是，您可能希望將這些值與應用程式的內部結構解耦。

例如，我們可以使用簡單的字串（例如 `post` 和 `video`）而不是模型名稱作為「類型」。這樣做，即使模型重新命名，資料庫中的多型「類型」欄位值也將保持有效：

```php
use Illuminate\Database\Eloquent\Relations\Relation;

Relation::enforceMorphMap([
    'post' => 'App\Models\Post',
    'video' => 'App\Models\Video',
]);
```

您可以在 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫 `enforceMorphMap` 方法，或者如果您願意，可以建立一個單獨的服務提供者。

您可以使用模型的 `getMorphClass` 方法在執行時決定給定模型的多型別名。相反，您可以使用 `Relation::getMorphedModel` 方法決定與多型別名相關聯的完全限定類別名稱：

```php
use Illuminate\Database\Eloquent\Relations\Relation;

$alias = $post->getMorphClass();

$class = Relation::getMorphedModel($alias);
```

> [!WARNING]
> 當向現有應用程式添加「多型映射」時，資料庫中所有仍包含完全限定類別的多型 `*_type` 欄位值都需要轉換為其「映射」名稱。

<a name="dynamic-relationships"></a>
### 動態關聯

您可以使用 `resolveRelationUsing` 方法在執行時定義 Eloquent 模型之間的關聯。雖然通常不建議用於正常的應用程式開發，但在開發 Laravel 套件時，這偶爾會很有用。

`resolveRelationUsing` 方法接受所需的關聯名稱作為其第一個引數。傳遞給方法的第二個引數應該是一個閉包，它接受模型實例並回傳有效的 Eloquent 關聯定義。通常，您應該在 [服務提供者](/docs/{{version}}/providers) 的 boot 方法中配置動態關聯：

```php
use App\Models\Order;
use App\Models\Customer;

Order::resolveRelationUsing('customer', function (Order $orderModel) {
    return $orderModel->belongsTo(Customer::class, 'customer_id');
});
```

> [!WARNING]
> 定義動態關聯時，請務必向 Eloquent 關聯方法提供明確的鍵名引數。

<a name="querying-relations"></a>
## 查詢關聯

由於所有 Eloquent 關聯都是透過方法定義的，您可以呼叫這些方法來取得關聯實例，而無需實際執行查詢來載入相關模型。此外，所有類型的 Eloquent 關聯也作為 [查詢建構器](/docs/{{version}}/queries)，允許您在最終對資料庫執行 SQL 查詢之前，繼續將限制鏈接到關聯查詢。

例如，想像一個部落格應用程式，其中 `User` 模型擁有多個相關的 `Post` 模型：

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

您可以查詢 `posts` 關聯並向關聯添加額外限制，如下所示：

```php
use App\Models\User;

$user = User::find(1);

$user->posts()->where('active', 1)->get();
```

您可以使用 Laravel [查詢建構器](/docs/{{version}}/queries) 的任何方法來處理關聯，因此請務必探索查詢建構器文件以了解所有可用的方法。

<a name="chaining-orwhere-clauses-after-relationships"></a>
#### 在關聯之後鏈接 `orWhere` 子句

如上例所示，您可以在查詢關聯時自由地添加額外限制。但是，在關聯之後鏈接 `orWhere` 子句時請務必小心，因為 `orWhere` 子句將與關聯限制在同一層級進行邏輯分組：

```php
$user->posts()
    ->where('active', 1)
    ->orWhere('votes', '>=', 100)
    ->get();
```

上面的範例將產生以下 SQL。如您所見，`or` 子句指示查詢回傳任何投票數大於 100 的文章。查詢不再受限於特定使用者：

```sql
select *
from posts
where user_id = ? and active = 1 or votes >= 100
```

在大多數情況下，您應該使用 [邏輯分組](/docs/{{version}}/queries#logical-grouping) 將條件檢查分組在括號內：

```php
use Illuminate\Database\Eloquent\Builder;

$user->posts()
    ->where(function (Builder $query) {
        return $query->where('active', 1)
            ->orWhere('votes', '>=', 100);
    })
    ->get();
```

上面的範例將產生以下 SQL。請注意，邏輯分組已正確分組限制，並且查詢仍受限於特定使用者：

```sql
select *
from posts
where user_id = ? and (active = 1 or votes >= 100)
```

<a name="relationship-methods-vs-dynamic-properties"></a>
### 關聯方法與動態屬性

如果您不需要向 Eloquent 關聯查詢添加額外限制，您可以像存取屬性一樣存取關聯。例如，繼續使用我們的 `User` 和 `Post` 範例模型，我們可以像這樣存取使用者所有文章：

```php
use App\Models\User;

$user = User::find(1);

foreach ($user->posts as $post) {
    // ...
}
```

動態關聯屬性執行「延遲載入」，這表示它們只會在您實際存取它們時才載入其關聯資料。因此，開發人員通常使用 [預載入](#eager-loading) 來預載入他們知道在載入模型後將被存取的關聯。預載入顯著減少了載入模型關聯必須執行的 SQL 查詢數量。

<a name="querying-relationship-existence"></a>
### 查詢關聯是否存在

擷取模型記錄時，您可能希望根據關聯的存在來限制結果。例如，想像您想要擷取所有至少有一個評論的部落格文章。為此，您可以將關聯名稱傳遞給 `has` 和 `orHas` 方法：

```php
use App\Models\Post;

// Retrieve all posts that have at least one comment...
$posts = Post::has('comments')->get();
```

您還可以指定運算子和計數值以進一步自訂查詢：

```php
// Retrieve all posts that have three or more comments...
$posts = Post::has('comments', '>=', 3)->get();
```

可以使用「點」符號建構巢狀 `has` 語句。例如，您可以擷取所有至少有一個評論且該評論至少有一張圖片的文章：

```php
// Retrieve posts that have at least one comment with images...
$posts = Post::has('comments.images')->get();
```

如果您需要更強大的功能，您可以使用 `whereHas` 和 `orWhereHas` 方法來定義 `has` 查詢的額外查詢限制，例如檢查評論的內容：

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

`whereAttachedTo` 方法可用於查詢與模型或模型集合具有多對多附件的模型：

```php
$users = User::whereAttachedTo($role)->get();
```

您也可以向 `whereAttachedTo` 方法提供一個 [集合](/docs/{{version}}/eloquent-collections) 實例。這樣做時，Laravel 將擷取附加到集合中任何模型的模型：

```php
$tags = Tag::whereLike('name', '%laravel%')->get();

$posts = Post::whereAttachedTo($tags)->get();
```

<a name="inline-relationship-existence-queries"></a>
#### 行內關聯存在查詢

如果您希望使用附加到關聯查詢的單一、簡單的 where 條件來查詢關聯的存在，您可能會發現使用 `whereRelation`、`orWhereRelation`、`whereMorphRelation` 和 `orWhereMorphRelation` 方法更方便。例如，我們可以查詢所有具有未批准評論的文章：

```php
use App\Models\Post;

$posts = Post::whereRelation('comments', 'is_approved', false)->get();
```

當然，與呼叫查詢建構器的 `where` 方法一樣，您也可以指定運算子：

```php
$posts = Post::whereRelation(
    'comments', 'created_at', '>=', now()->subHour()
)->get();
```

<a name="querying-relationship-absence"></a>
### 查詢關聯是否不存在

擷取模型記錄時，您可能希望根據關聯的不存在來限制結果。例如，想像您想要擷取所有**沒有**任何評論的部落格文章。為此，您可以將關聯名稱傳遞給 `doesntHave` 和 `orDoesntHave` 方法：

```php
use App\Models\Post;

$posts = Post::doesntHave('comments')->get();
```

如果您需要更強大的功能，您可以使用 `whereDoesntHave` 和 `orWhereDoesntHave` 方法來為 `doesntHave` 查詢添加額外查詢限制，例如檢查評論的內容：

```php
use Illuminate\Database\Eloquent\Builder;

$posts = Post::whereDoesntHave('comments', function (Builder $query) {
    $query->where('content', 'like', 'code%');
})->get();
```

您可以使用「點」符號對巢狀關聯執行查詢。例如，以下查詢將擷取所有沒有評論的文章，以及有評論但沒有任何評論來自被禁止使用者的文章：

```php
use Illuminate\Database\Eloquent\Builder;

$posts = Post::whereDoesntHave('comments.author', function (Builder $query) {
    $query->where('banned', 1);
})->get();
```

<a name="querying-morph-to-relationships"></a>
### 查詢 Morph To 關聯

要查詢「morph to」關聯的存在，您可以使用 `whereHasMorph` 和 `whereDoesntHaveMorph` 方法。這些方法接受關聯名稱作為其第一個引數。接下來，這些方法接受您希望包含在查詢中的相關模型名稱。最後，您可以提供一個閉包來自訂關聯查詢：

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

您可能偶爾需要根據相關多型模型的「類型」添加查詢限制。傳遞給 `whereHasMorph` 方法的閉包可以將 `$type` 值作為其第二個引數接收。此引數允許您檢查正在建構的查詢的「類型」：

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

有時您可能希望查詢「morph to」關聯的父級的子級。您可以使用 `whereMorphedTo` 和 `whereNotMorphedTo` 方法來實現此目的，這些方法將自動為給定模型決定正確的多型類型映射。這些方法接受 `morphTo` 關聯的名稱作為其第一個引數，並將相關父模型作為其第二個引數：

```php
$comments = Comment::whereMorphedTo('commentable', $post)
    ->orWhereMorphedTo('commentable', $video)
    ->get();
```

<a name="querying-all-morph-to-related-models"></a>
#### 查詢所有相關模型

您可以提供 `*` 作為萬用字元值，而不是傳遞可能的多型模型陣列。這將指示 Laravel 從資料庫中擷取所有可能的多型類型。Laravel 將執行額外的查詢來執行此操作：

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

有時您可能希望計算給定關聯的相關模型數量，而無需實際載入模型。為了實現此目的，您可以使用 `withCount` 方法。`withCount` 方法將在結果模型上放置一個 `{relation}_count` 屬性：

```php
use App\Models\Post;

$posts = Post::withCount('comments')->get();

foreach ($posts as $post) {
    echo $post->comments_count;
}
```

透過將陣列傳遞給 `withCount` 方法，您可以添加多個關聯的「計數」，並向查詢添加額外限制：

```php
use Illuminate\Database\Eloquent\Builder;

$posts = Post::withCount(['votes', 'comments' => function (Builder $query) {
    $query->where('content', 'like', 'code%');
}])->get();

echo $posts[0]->votes_count;
echo $posts[0]->comments_count;
```

您還可以為關聯計數結果設定別名，允許在同一個關聯上進行多次計數：

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

使用 `loadCount` 方法，您可以在父模型已經擷取後載入關聯計數：

```php
$book = Book::first();

$book->loadCount('genres');
```

如果您需要為計數查詢設定額外的查詢限制，您可以傳遞一個以您希望計數的關聯為鍵的陣列。陣列值應該是接收查詢建構器實例的閉包：

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
### 其他聚合函數

除了 `withCount` 方法之外，Eloquent 還提供了 `withMin`、`withMax`、`withAvg`、`withSum` 和 `withExists` 方法。這些方法將在您的結果模型上放置一個 `{relation}_{function}_{column}` 屬性：

```php
use App\Models\Post;

$posts = Post::withSum('comments', 'votes')->get();

foreach ($posts as $post) {
    echo $post->comments_sum_votes;
}
```

如果您希望使用另一個名稱存取聚合函數的結果，您可以指定自己的別名：

```php
$posts = Post::withSum('comments as total_comments', 'votes')->get();

foreach ($posts as $post) {
    echo $post->total_comments;
}
```

與 `loadCount` 方法一樣，這些方法的延遲版本也可用。這些額外的聚合操作可以在已經擷取的 Eloquent 模型上執行：

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
### 計算 Morph To 關聯上的關聯模型數量

如果您希望預載入「morph to」關聯，以及該關聯可能回傳的各種實體的相關模型計數，您可以使用 `with` 方法結合 `morphTo` 關聯的 `morphWithCount` 方法。

在此範例中，假設 `Photo` 和 `Post` 模型可以建立 `ActivityFeed` 模型。我們將假設 `ActivityFeed` 模型定義了一個名為 `parentable` 的「morph to」關聯，該關聯允許我們擷取給定 `ActivityFeed` 實例的父 `Photo` 或 `Post` 模型。此外，讓我們假設 `Photo` 模型「擁有多個」`Tag` 模型，並且 `Post` 模型「擁有多個」`Comment` 模型。

現在，讓我們想像我們想要擷取 `ActivityFeed` 實例並預載入每個 `ActivityFeed` 實例的 `parentable` 父模型。此外，我們想要擷取與每個父圖片相關聯的標籤數量，以及與每個父文章相關聯的評論數量：

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
#### Morph To 延遲計數載入

讓我們假設我們已經擷取了一組 `ActivityFeed` 模型，現在我們希望載入與活動摘要相關聯的各種 `parentable` 模型的巢狀關聯計數。您可以使用 `loadMorphCount` 方法來實現此目的：

```php
$activities = ActivityFeed::with('parentable')->get();

$activities->loadMorphCount('parentable', [
    Photo::class => ['tags'],
    Post::class => ['comments'],
]);
```

<a name="eager-loading"></a>
## 預載入 (Eager Loading)

當將 Eloquent 關聯作為屬性存取時，相關模型是「延遲載入」的。這表示關聯資料直到您第一次存取屬性時才實際載入。但是，Eloquent 可以在您查詢父模型時「預載入」關聯。預載入緩解了「N + 1」查詢問題。為了說明 N + 1 查詢問題，請考慮一個「屬於」`Author` 模型的 `Book` 模型：

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

現在，讓我們擷取所有書籍及其作者：

```php
use App\Models\Book;

$books = Book::all();

foreach ($books as $book) {
    echo $book->author->name;
}
```

此迴圈將執行一個查詢以擷取資料庫表格中的所有書籍，然後為每本書執行另一個查詢以擷取該書的作者。因此，如果我們有 25 本書，上面的程式碼將執行 26 個查詢：一個用於原始書籍，另外 25 個查詢用於擷取每本書的作者。

幸運的是，我們可以使用預載入將此操作減少到僅兩個查詢。在建構查詢時，您可以使用 `with` 方法指定應預載入哪些關聯：

```php
$books = Book::with('author')->get();

foreach ($books as $book) {
    echo $book->author->name;
}
```

對於此操作，只會執行兩個查詢——一個查詢以擷取所有書籍，一個查詢以擷取所有書籍的所有作者：

```sql
select * from books

select * from authors where id in (1, 2, 3, 4, 5, ...)
```

<a name="eager-loading-multiple-relationships"></a>
#### 預載入多個關聯

有時您可能需要預載入幾個不同的關聯。為此，只需將關聯陣列傳遞給 `with` 方法：

```php
$books = Book::with(['author', 'publisher'])->get();
```

<a name="nested-eager-loading"></a>
#### 巢狀預載入

要預載入關聯的關聯，您可以使用「點」語法。例如，讓我們預載入所有書籍的作者以及作者的個人聯絡人：

```php
$books = Book::with('author.contacts')->get();
```

或者，您可以透過向 `with` 方法提供巢狀陣列來指定巢狀預載入關聯，這在預載入多個巢狀關聯時會很方便：

```php
$books = Book::with([
    'author' => [
        'contacts',
        'publisher',
    ],
])->get();
```

<a name="nested-eager-loading-morphto-relationships"></a>
#### 巢狀預載入 `morphTo` 關聯

如果您希望預載入 `morphTo` 關聯，以及該關聯可能回傳的各種實體上的巢狀關聯，您可以使用 `with` 方法結合 `morphTo` 關聯的 `morphWith` 方法。為了幫助說明此方法，讓我們考慮以下模型：

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

在此範例中，假設 `Event`、`Photo` 和 `Post` 模型可以建立 `ActivityFeed` 模型。此外，假設 `Event` 模型屬於 `Calendar` 模型，`Photo` 模型與 `Tag` 模型相關聯，並且 `Post` 模型屬於 `Author` 模型。

使用這些模型定義和關聯，我們可以擷取 `ActivityFeed` 模型實例並預載入所有 `parentable` 模型及其各自的巢狀關聯：

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
#### 預載入特定欄位

您可能不總是需要從您正在擷取的關聯中獲取所有欄位。因此，Eloquent 允許您指定您希望擷取關聯的哪些欄位：

```php
$books = Book::with('author:id,name,book_id')->get();
```

> [!WARNING]
> 使用此功能時，您應該始終在您希望擷取的欄位清單中包含 `id` 欄位和任何相關的外鍵欄位。

<a name="eager-loading-by-default"></a>
#### 預設預載入

有時您可能希望在擷取模型時始終載入某些關聯。為了實現此目的，您可以在模型上定義一個 `$with` 屬性：

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

如果您希望為單一查詢從 `$with` 屬性中移除項目，您可以使用 `without` 方法：

```php
$books = Book::without('author')->get();
```

如果您希望為單一查詢覆寫 `$with` 屬性中的所有項目，您可以使用 `withOnly` 方法：

```php
$books = Book::withOnly('genre')->get();
```

<a name="constraining-eager-loads"></a>
### 限制預載入

有時您可能希望預載入關聯，但同時也為預載入查詢指定額外的查詢條件。您可以透過將關聯陣列傳遞給 `with` 方法來實現此目的，其中陣列鍵是關聯名稱，陣列值是一個閉包，它向預載入查詢添加額外限制：

```php
use App\Models\User;
use Illuminate\Database\Eloquent\Builder;

$users = User::with(['posts' => function (Builder $query) {
    $query->where('title', 'like', '%code%');
}])->get();
```

在此範例中，Eloquent 只會預載入文章 `title` 欄位包含「code」字樣的文章。您可以呼叫其他 [查詢建構器](/docs/{{version}}/queries) 方法來進一步自訂預載入操作：

```php
$users = User::with(['posts' => function (Builder $query) {
    $query->orderBy('created_at', 'desc');
}])->get();
```

<a name="constraining-eager-loading-of-morph-to-relationships"></a>
#### 限制 `morphTo` 關聯的預載入

如果您正在預載入 `morphTo` 關聯，Eloquent 將執行多個查詢以擷取每種類型的相關模型。您可以使用 `MorphTo` 關聯的 `constrain` 方法向每個查詢添加額外限制：

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

在此範例中，Eloquent 只會預載入未隱藏的文章和 `type` 值為「educational」的影片。

<a name="constraining-eager-loads-with-relationship-existence"></a>
#### 透過關聯存在限制預載入

您有時可能會發現自己需要檢查關聯是否存在，同時根據相同的條件載入關聯。例如，您可能希望只擷取具有符合給定查詢條件的子 `Post` 模型，同時也預載入匹配的文章。您可以使用 `withWhereHas` 方法來實現此目的：

```php
use App\Models\User;

$users = User::withWhereHas('posts', function ($query) {
    $query->where('featured', true);
})->get();
```

<a name="lazy-eager-loading"></a>
### 延遲預載入 (Lazy Eager Loading)

有時您可能需要在父模型已經擷取後預載入關聯。例如，如果您需要動態決定是否載入相關模型，這可能會很有用：

```php
use App\Models\Book;

$books = Book::all();

if ($condition) {
    $books->load('author', 'publisher');
}
```

如果您需要為預載入查詢設定額外的查詢限制，您可以傳遞一個以您希望載入的關聯為鍵的陣列。陣列值應該是接收查詢實例的閉包實例：

```php
$author->load(['books' => function (Builder $query) {
    $query->orderBy('published_date', 'asc');
}]);
```

要僅在關聯尚未載入時載入它，請使用 `loadMissing` 方法：

```php
$book->loadMissing('author');
```

<a name="nested-lazy-eager-loading-morphto"></a>
#### 巢狀延遲預載入與 `morphTo`

如果您希望預載入 `morphTo` 關聯，以及該關聯可能回傳的各種實體上的巢狀關聯，您可以使用 `loadMorph` 方法。

此方法接受 `morphTo` 關聯的名稱作為其第一個引數，以及模型/關聯對的陣列作為其第二個引數。為了幫助說明此方法，讓我們考慮以下模型：

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

在此範例中，假設 `Event`、`Photo` 和 `Post` 模型可以建立 `ActivityFeed` 模型。此外，假設 `Event` 模型屬於 `Calendar` 模型，`Photo` 模型與 `Tag` 模型相關聯，並且 `Post` 模型屬於 `Author` 模型。

使用這些模型定義和關聯，我們可以擷取 `ActivityFeed` 模型實例並預載入所有 `parentable` 模型及其各自的巢狀關聯：

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
### 自動預載入

> [!WARNING]
> 此功能目前處於 Beta 階段，以收集社群回饋。此功能的行為和功能可能會在修補程式版本中發生變化。

在許多情況下，Laravel 可以自動預載入您存取的關聯。要啟用自動預載入，您應該在應用程式 `AppServiceProvider` 的 `boot` 方法中呼叫 `Model::automaticallyEagerLoadRelationships` 方法：

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

啟用此功能後，Laravel 將嘗試自動載入您存取但尚未載入的任何關聯。例如，考慮以下情境：

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

通常，上面的程式碼會為每個使用者執行一個查詢以擷取其文章，並為每篇文章執行一個查詢以擷取其評論。但是，當啟用 `automaticallyEagerLoadRelationships` 功能時，當您嘗試存取任何擷取到的使用者上的文章時，Laravel 將自動 [延遲預載入](#lazy-eager-loading) 使用者集合中所有使用者的文章。同樣，當您嘗試存取任何擷取到的文章的評論時，所有評論都將為所有最初擷取的文章延遲預載入。

如果您不想全域啟用自動預載入，您仍然可以透過在集合上呼叫 `withRelationshipAutoloading` 方法來為單個 Eloquent 集合實例啟用此功能：

```php
$users = User::where('vip', true)->get();

return $users->withRelationshipAutoloading();
```

<a name="preventing-lazy-loading"></a>
### 防止延遲載入

如前所述，預載入關聯通常可以為您的應用程式提供顯著的效能優勢。因此，如果您願意，您可以指示 Laravel 始終防止關聯的延遲載入。為了實現此目的，您可以呼叫基底 Eloquent 模型類別提供的 `preventLazyLoading` 方法。通常，您應該在應用程式 `AppServiceProvider` 類別的 `boot` 方法中呼叫此方法。

`preventLazyLoading` 方法接受一個可選的布林引數，該引數指示是否應防止延遲載入。例如，您可能希望只在非生產環境中禁用延遲載入，以便您的生產環境即使在生產程式碼中意外存在延遲載入關聯時也能正常運作：

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

防止延遲載入後，當您的應用程式嘗試延遲載入任何 Eloquent 關聯時，Eloquent 將拋出 `Illuminate\Database\LazyLoadingViolationException` 異常。

您可以使用 `handleLazyLoadingViolationsUsing` 方法自訂延遲載入違規的行為。例如，使用此方法，您可以指示延遲載入違規只記錄日誌，而不是透過異常中斷應用程式的執行：

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

Eloquent 提供了方便的方法來向關聯添加新模型。例如，您可能需要向文章添加新評論。您可以透過使用關聯的 `save` 方法插入評論，而不是手動設定 `Comment` 模型上的 `post_id` 屬性：

```php
use App\Models\Comment;
use App\Models\Post;

$comment = new Comment(['message' => 'A new comment.']);

$post = Post::find(1);

$post->comments()->save($comment);
```

請注意，我們沒有將 `comments` 關聯作為動態屬性存取。相反，我們呼叫了 `comments` 方法以取得關聯實例。`save` 方法將自動向新的 `Comment` 模型添加適當的 `post_id` 值。

如果您需要儲存多個相關模型，您可以使用 `saveMany` 方法：

```php
$post = Post::find(1);

$post->comments()->saveMany([
    new Comment(['message' => 'A new comment.']),
    new Comment(['message' => 'Another new comment.']),
]);
```

`save` 和 `saveMany` 方法將持久化給定的模型實例，但不會將新持久化的模型添加到已載入到父模型上的任何記憶體中關聯。如果您計劃在使用 `save` 或 `saveMany` 方法後存取關聯，您可能希望使用 `refresh` 方法重新載入模型及其關聯：

```php
$post->comments()->save($comment);

$post->refresh();

// All comments, including the newly saved comment...
$post->comments;
```

<a name="the-push-method"></a>
#### 遞迴儲存模型與關聯

如果您希望 `save` 您的模型及其所有相關關聯，您可以使用 `push` 方法。在此範例中，`Post` 模型及其評論和評論的作者都將被儲存：

```php
$post = Post::find(1);

$post->comments[0]->message = 'Message';
$post->comments[0]->author->name = 'Author Name';

$post->push();
```

`pushQuietly` 方法可用於儲存模型及其相關關聯，而不會引發任何事件：

```php
$post->pushQuietly();
```

<a name="the-create-method"></a>
### `create` 方法

除了 `save` 和 `saveMany` 方法之外，您還可以使用 `create` 方法，該方法接受一個屬性陣列，建立一個模型，並將其插入資料庫。`save` 和 `create` 之間的區別在於 `save` 接受一個完整的 Eloquent 模型實例，而 `create` 接受一個普通的 PHP `array`。新建立的模型將由 `create` 方法回傳：

```php
use App\Models\Post;

$post = Post::find(1);

$comment = $post->comments()->create([
    'message' => 'A new comment.',
]);
```

您可以使用 `createMany` 方法建立多個相關模型：

```php
$post = Post::find(1);

$post->comments()->createMany([
    ['message' => 'A new comment.'],
    ['message' => 'Another new comment.'],
]);
```

`createQuietly` 和 `createManyQuietly` 方法可用於建立模型而不會分派任何事件：

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

您還可以使用 `findOrNew`、`firstOrNew`、`firstOrCreate` 和 `updateOrCreate` 方法來 [在關聯上建立和更新模型](/docs/{{version}}/eloquent#upserts)。

> [!NOTE]
> 在使用 `create` 方法之前，請務必查閱 [大量賦值](/docs/{{version}}/eloquent#mass-assignment) 文件。

<a name="updating-belongs-to-relationships"></a>
### Belongs To 關聯

如果您希望將子模型分配給新的父模型，您可以使用 `associate` 方法。在此範例中，`User` 模型定義了與 `Account` 模型的 `belongsTo` 關聯。此 `associate` 方法將設定子模型上的外鍵：

```php
use App\Models\Account;

$account = Account::find(10);

$user->account()->associate($account);

$user->save();
```

要從子模型中移除父模型，您可以使用 `dissociate` 方法。此方法將關聯的外鍵設定為 `null`：

```php
$user->account()->dissociate();

$user->save();
```

<a name="updating-many-to-many-relationships"></a>
### 多對多關聯

<a name="attaching-detaching"></a>
#### 附加 / 分離

Eloquent 還提供了使多對多關聯更方便的方法。例如，讓我們想像一個使用者可以擁有多個角色，一個角色可以擁有多個使用者。您可以使用 `attach` 方法將角色附加到使用者，方法是在關聯的中間表格中插入一條記錄：

```php
use App\Models\User;

$user = User::find(1);

$user->roles()->attach($roleId);
```

將關聯附加到模型時，您還可以傳遞一個額外資料陣列以插入中間表格：

```php
$user->roles()->attach($roleId, ['expires' => $expires]);
```

有時可能需要從使用者中移除角色。要移除多對多關聯記錄，請使用 `detach` 方法。`detach` 方法將從中間表格中刪除相應的記錄；但是，兩個模型都將保留在資料庫中：

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

您還可以使用 `sync` 方法來建構多對多關聯。`sync` 方法接受一個 ID 陣列以放置在中間表格上。任何不在給定陣列中的 ID 都將從中間表格中移除。因此，此操作完成後，中間表格中只會存在給定陣列中的 ID：

```php
$user->roles()->sync([1, 2, 3]);
```

您還可以傳遞額外的中間表格值和 ID：

```php
$user->roles()->sync([1 => ['expires' => true], 2, 3]);
```

如果您希望將相同的中間表格值與每個同步的模型 ID 一起插入，您可以使用 `syncWithPivotValues` 方法：

```php
$user->roles()->syncWithPivotValues([1, 2, 3], ['active' => true]);
```

如果您不想分離給定陣列中缺少現有 ID，您可以使用 `syncWithoutDetaching` 方法：

```php
$user->roles()->syncWithoutDetaching([1, 2, 3]);
```

<a name="toggling-associations"></a>
#### 切換關聯

多對多關聯還提供了一個 `toggle` 方法，該方法「切換」給定相關模型 ID 的附加狀態。如果給定的 ID 當前已附加，它將被分離。同樣，如果它當前已分離，它將被附加：

```php
$user->roles()->toggle([1, 2, 3]);
```

您還可以傳遞額外的中間表格值和 ID：

```php
$user->roles()->toggle([
    1 => ['expires' => true],
    2 => ['expires' => true],
]);
```

<a name="updating-a-record-on-the-intermediate-table"></a>
#### 更新中間表格上的記錄

如果您需要更新關聯中間表格中的現有行，您可以使用 `updateExistingPivot` 方法。此方法接受中間記錄外鍵和要更新的屬性陣列：

```php
$user = User::find(1);

$user->roles()->updateExistingPivot($roleId, [
    'active' => false,
]);
```

<a name="touching-parent-timestamps"></a>
## 更新父級時間戳記

當模型定義了與另一個模型的 `belongsTo` 或 `belongsToMany` 關聯時，例如屬於 `Post` 的 `Comment`，有時在子模型更新時更新父級的時間戳記會很有幫助。

例如，當 `Comment` 模型更新時，您可能希望自動「更新」擁有 `Post` 的 `updated_at` 時間戳記，使其設定為當前日期和時間。為了實現此目的，您可以向子模型添加一個 `touches` 屬性，其中包含在子模型更新時應更新其 `updated_at` 時間戳記的關聯名稱：

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
> 父模型時間戳記只會在子模型使用 Eloquent 的 `save` 方法更新時更新。
