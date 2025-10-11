# Eloquent：關聯

- [簡介](#introduction)
- [定義關聯](#defining-relationships)
    - [一對一 / Has One](#one-to-one)
    - [一對多 / Has Many](#one-to-many)
    - [一對多 (反向) / Belongs To](#one-to-many-inverse)
    - [Has One of Many](#has-one-of-many)
    - [Has One Through](#has-one-through)
    - [Has Many Through](#has-many-through)
- [限定範圍的關聯](#scoped-relationships)
- [多對多關聯](#many-to-many)
    - [取得中介資料表的欄位](#retrieving-intermediate-table-columns)
    - [透過中介資料表欄位來篩選查詢](#filtering-queries-via-intermediate-table-columns)
    - [透過中介資料表欄位來排序查詢](#ordering-queries-via-intermediate-table-columns)
    - [定義自訂的中介資料表 Model](#defining-custom-intermediate-table-models)
- [Polymorphic (多態) 關聯](#polymorphic-relationships)
    - [一對一](#one-to-one-polymorphic-relations)
    - [一對多](#one-to-many-polymorphic-relations)
    - [One of Many](#one-of-many-polymorphic-relations)
    - [多對多](#many-to-many-polymorphic-relations)
    - [自訂 Polymorphic (多態) 型別](#custom-polymorphic-types)
- [動態關聯](#dynamic-relationships)
- [查詢關聯](#querying-relations)
    - [關聯方法 vs. 動態屬性](#relationship-methods-vs-dynamic-properties)
    - [查詢關聯是否存在](#querying-relationship-existence)
    - [查詢關聯是否不存在](#querying-relationship-absence)
    - [查詢 Morph To 關聯](#querying-morph-to-relationships)
- [彙總關聯 Model](#aggregating-related-models)
    - [計算關聯 Model](#counting-related-models)
    - [其他彙總函式](#other-aggregate-functions)
    - [在 Morph To 關聯上計算關聯 Model](#counting-related-models-on-morph-to-relationships)
- [預載入 (Eager Loading)](#eager-loading)
    - [限制預載入](#constraining-eager-loads)
    - [延遲預載入 (Lazy Eager Loading)](#lazy-eager-loading)
    - [防止延遲載入 (Lazy Loading)](#preventing-lazy-loading)
- [新增與更新關聯 Model](#inserting-and-updating-related-models)
    - [`save` 方法](#the-save-method)
    - [`create` 方法](#the-create-method)
    - [Belongs To 關聯](#updating-belongs-to-relationships)
    - [多對多關聯](#updating-many-to-many-relationships)
- [接觸父項的時間戳記](#touching-parent-timestamps)

<a name="introduction"></a>
## 簡介

資料庫資料表通常會互相有關聯。例如，一篇部落格文章可能會有多則留言，或一筆訂單可能會關聯到下訂單的使用者。Eloquent 讓管理與操作這些關聯變得簡單，且支援多種常見的關聯：

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

Eloquent 關련是在 Eloquent Model 類別上定義為方法。由於關聯同時也作為強大的[查詢產生器](/docs/{{version}}/queries)，因此將關聯定義為方法可提供強大的方法鏈結與查詢功能。舉例來說，我們可以在這個 `posts` 關聯上鏈結其他的查詢限制條件：

    $user->posts()->where('active', 1)->get();

不過，在深入研究如何使用關聯前，讓我們先來學習如何定義 Eloquent 支援的每種關聯類型。

<a name="one-to-one"></a>
### 一對一 / Has One

一對一關聯是一種非常基本的資料庫關聯類型。舉例來說，一個 `User` Model 可能會關聯到一個 `Phone` Model。若要定義此關聯，我們要在 `User` Model 上放置一個 `phone` 方法。`phone` 方法應呼叫 `hasOne` 方法並回傳其結果。`hasOne` 方法可透過 Model 的 `Illuminate\Database\Eloquent\Model` 基底類別來使用：

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

傳給 `hasOne` 方法的第一個引數是關聯 Model 類別的名稱。定義好關聯後，我們就可以使用 Eloquent 的動態屬性來取得關聯的記錄。動態屬性可讓我們像存取 Model 上已定義的屬性一樣來存取關聯方法：

    $phone = User::find(1)->phone;

Eloquent 會根據父 Model 的名稱來判斷關聯的外鍵。在這個例子中，`Phone` Model 會被自動假定有一個 `user_id` 外鍵。若想覆寫這個慣例，可以傳入第二個引數給 `hasOne` 方法：

    return $this->hasOne(Phone::class, 'foreign_key');

此外，Eloquent 會假定外鍵的值應符合父項的主鍵欄位。換句話說，Eloquent 會在 `Phone` 記錄的 `user_id` 欄位中尋找使用者 `id` 欄位的值。若想讓關聯使用 `id` 或 Model 的 `$primaryKey` 屬性以外的主鍵值，可以傳入第三個引數給 `hasOne` 方法：

    return $this->hasOne(Phone::class, 'foreign_key', 'local_key');

<a name="one-to-one-defining-the-inverse-of-the-relationship"></a>
#### 定義反向關聯

這樣一來，我們就能從 `User` Model 中存取 `Phone` Model。接著，我們在 `Phone` Model 上定義一個關聯，讓我們能存取擁有該電話的使用者。我們可以使用 `belongsTo` 方法來定義 `hasOne` 關聯的反向關聯：

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

叫用 `user` 方法時，Eloquent 會嘗試尋找一個 `id` 符合 `Phone` Model 上 `user_id` 欄位的 `User` Model。

Eloquent 會檢查關聯方法的名稱，並在方法名稱後加上 `_id` 來判斷外鍵名稱。因此，在這個例子中，Eloquent 會假定 `Phone` Model 有一個 `user_id` 欄位。不過，若 `Phone` Model 上的外鍵不是 `user_id`，則可以傳入自訂的鍵名作為 `belongsTo` 方法的第二個引數：

    /**
     * Get the user that owns the phone.
     */
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class, 'foreign_key');
    }

若父 Model 不使用 `id` 作為主鍵，或想使用不同的欄位來尋找關聯的 Model，可以傳入第三個引數給 `belongsTo` 方法，以指定父資料表的自訂鍵：

    /**
     * Get the user that owns the phone.
     */
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class, 'foreign_key', 'owner_key');
    }

<a name="one-to-many"></a>
### 一對多 / Has Many

一對多關聯用來定義某個 Model 是一個或多個子 Model 的父項的關聯。舉例來說，一篇部落格文章可能會有無限多則留言。與其他所有 Eloquent 關聯一樣，一對多關聯也是在 Eloquent Model 上定義一個方法來定義的：

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

請記得，Eloquent 會自動為 `Comment` Model 判斷正確的外鍵欄位。依照慣例，Eloquent 會取父 Model 的「蛇形 (snake case)」名稱，並在後面加上 `_id`。因此，在這個例子中，Eloquent 會假定 `Comment` Model 上的外鍵欄位是 `post_id`。

定義好關聯方法後，我們就可以存取 `comments` 屬性來取得關聯留言的[集合](/docs/{{version}}/eloquent-collections)。請記得，由於 Eloquent 提供了「動態關聯屬性」，因此我們可以像存取 Model 上已定義的屬性一樣來存取關聯方法：

    use App\Models\Post;

    $comments = Post::find(1)->comments;

    foreach ($comments as $comment) {
        // ...
    }

由於所有關聯也都作為查詢產生器，因此可以呼叫 `comments` 方法並繼續在查詢上鏈結條件，來為關聯查詢加上更多限制條件：

    $comment = Post::find(1)->comments()
        ->where('title', 'foo')
        ->first();

與 `hasOne` 方法一樣，也可以傳入額外的引數給 `hasMany` 方法來覆寫外鍵與本地鍵：

    return $this->hasMany(Comment::class, 'foreign_key');

    return $this->hasMany(Comment::class, 'foreign_key', 'local_key');

<a name="automatically-hydrating-parent-models-on-children"></a>
#### 自動在子項上填充父 Model

即使使用了 Eloquent 預載入，若在遍覽子 Model 時嘗試從子 Model 存取父 Model，仍可能發生「N + 1」查詢問題：

```php
$posts = Post::with('comments')->get();

foreach ($posts as $post) {
    foreach ($post->comments as $comment) {
        echo $comment->post->title;
    }
}
```

在上述範例中，之所以會發生「N + 1」查詢問題，是因為即使已為每個 `Post` Model 預載入留言，Eloquent 也不會自動在每個子 `Comment` Model 上填充父 `Post` Model。

若想讓 Eloquent 自動在子項上填充父 Model，可以在定義 `hasMany` 關聯時叫用 `chaperone` 方法：

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

或者，若想在執行階段選擇性啟用自動父項填充，可以在預載入關聯時叫用 `chaperone` Model：

```php
use App\Models\Post;

$posts = Post::with([
    'comments' => fn ($comments) => $comments->chaperone(),
])->get();
```

<a name="one-to-many-inverse"></a>
### 一對多 (反向) / Belongs To

現在我們可以存取一篇文章的所有留言了，接著讓我們來定義一個關聯，讓留言可以存取其父文章。若要定義 `hasMany` 關聯的反向關聯，請在子 Model 上定義一個關聯方法，並在其中呼叫 `belongsTo` 方法：

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

關聯定義好後，我們就可以通過存取 `post`「動態關聯屬性」來取得留言的父文章：

    use App\Models\Comment;

    $comment = Comment::find(1);

    return $comment->post->title;

在上述範例中，Eloquent 會嘗試尋找 `id` 符合 `Comment` Model 上 `post_id` 欄位的 `Post` Model。

Eloquent 會檢查關聯方法的名稱，並在方法名稱後加上 `_` 與父 Model 的主鍵欄位名稱來作為預設的外鍵名稱。因此，在這個範例中，Eloquent 會假設 `comments` 資料表中 `Post` Model 的外鍵為 `post_id`。

不過，若關聯的外鍵不符合此慣例，則可以傳入一個自訂的外鍵名稱作為 `belongsTo` 方法的第二個引數：

    /**
     * Get the post that owns the comment.
     */
    public function post(): BelongsTo
    {
        return $this->belongsTo(Post::class, 'foreign_key');
    }

若父 Model 不使用 `id` 作為主鍵，或想用不同的欄位來尋找關聯的 Model，則可以傳入第三個引數給 `belongsTo` 方法來指定父資料表的自訂索引鍵：

    /**
     * Get the post that owns the comment.
     */
    public function post(): BelongsTo
    {
        return $this->belongsTo(Post::class, 'foreign_key', 'owner_key');
    }


<a name="default-models"></a>
#### 預設 Model

`belongsTo`、`hasOne`、`hasOneThrough`、與 `morphOne` 關聯可讓開發者定義一個預設 Model。當指定的關聯為 `null` 時，就會回傳這個預設 Model。這個模式通常被稱為 [Null 物件模式 (Null Object pattern)](https://en.wikipedia.org/wiki/Null_Object_pattern)，能幫助我們移除程式碼中的條件式檢查。在下列範例中，若沒有使用者附加至 `Post` Model，則 `user` 關聯會回傳一個空的 `App\Models\User` Model：

    /**
     * Get the author of the post.
     */
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class)->withDefault();
    }

若要為預設 Model 填入屬性，可以傳入一個陣列或閉包至 `withDefault` 方法：

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


<a name="querying-belongs-to-relationships"></a>
#### 查詢 Belongs To 關聯

當查詢「belongs to」關聯的子項時，可以手動建立 `where` 子句來取得對應的 Eloquent Model：

    use App\Models\Post;

    $posts = Post::where('user_id', $user->id)->get();

不過，使用 `whereBelongsTo` 方法可能會更方便，該方法會自動為給定的 Model 判斷正確的關聯與外鍵：

    $posts = Post::whereBelongsTo($user)->get();

也可以提供一個[集合](/docs/{{version}}/eloquent-collections)實體給 `whereBelongsTo` 方法。這麼做的話，Laravel 就會取得屬於該集合中任何一個父 Model 的 Model：

    $users = User::where('vip', true)->get();

    $posts = Post::whereBelongsTo($users)->get();

預設情況下，Laravel 會根據給定 Model 的類別名稱來判斷其關聯；不過，也可以手動指定關聯名稱，只要將其作為第二個引數提供給 `whereBelongsTo` 方法即可：

    $posts = Post::whereBelongsTo($user, 'author')->get();

<a name="has-one-of-many"></a>
### Has One of Many

有時候，一個 Model 可能會有很多個關聯 Model，但我們只希望能簡單地取得關聯中「最新」或「最舊」的那個關聯 Model。舉例來說，一個 `User` Model 可能會關聯到多個 `Order` Model，但我們希望能定義一個方便的方法來操作該使用者最近下的一筆訂單。我們可以結合 `hasOne` 關聯類型與 `ofMany` 等方法來達成這個目的：

```php
/**
 * Get the user's most recent order.
 */
public function latestOrder(): HasOne
{
    return $this->hasOne(Order::class)->latestOfMany();
}
```

同樣地，我們也可以定義一個方法來取得關聯中「最舊」或第一筆的關聯 Model：

```php
/**
 * Get the user's oldest order.
 */
public function oldestOrder(): HasOne
{
    return $this->hasOne(Order::class)->oldestOfMany();
}
```

預設情況下，`latestOfMany` 與 `oldestOfMany` 會根據 Model 的主鍵來取得最新或最舊的關聯 Model，因此主鍵必須是可排序的。不過，有時候我們可能會希望能用不同的排序條件來從一個較大的關聯中取得單一 Model。

舉例來說，使用 `ofMany` 方法，我們可以取得使用者最貴的一筆訂單。`ofMany` 方法的第一個引數為要排序的欄位，第二個引數則為查詢關聯 Model 時要套用的彙總函式 (`min` 或 `max`)：

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
> 由於 PostgreSQL 不支援對 UUID 欄位執行 `MAX` 函式，因此目前無法將 one-of-many 關聯與 PostgreSQL 的 UUID 欄位一起使用。

<a name="converting-many-relationships-to-has-one-relationships"></a>
#### 將「多筆」關聯轉換為 Has One 關聯

通常，在使用 `latestOfMany`、`oldestOfMany`、或 `ofMany` 來取得單一 Model 時，我們通常都已經為同一個 Model 定義好了「has many」關聯。為了方便起見，Laravel 允許我們在關聯上叫用 `one` 方法，來將這個關聯簡單地轉換為「has one」關聯：

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

<a name="advanced-has-one-of-many-relationships"></a>
#### 進階 Has One of Many 關聯

我們也可以建構更進階的「has one of many」關聯。舉例來說，一個 `Product` Model 可能會關聯到多個 `Price` Model，而這些價格在發布新價格後仍會保留在系統中。此外，產品的新價格資料也可能會預先發布，並透過 `published_at` 欄位來指定於未來的某個日期生效。

總結來說，我們需要取得已發布的價格中，發布日期不大於現在的最新價格。此外，若兩個價格有相同的發布日期，我們會偏好使用 ID 較大的那個價格。為達成此目的，我們必須傳遞一個陣列給 `ofMany` 方法，其中包含用來決定最新價格的可排序欄位。此外，我們還會提供一個閉包 (Closure) 作為 `ofMany` 方法的第二個引數。這個閉包會負責為關聯查詢加上額外的發布日期限制：

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

「has-one-through」關聯用來定義與另一個 Model 的一對一關聯。不過，這個關聯代表，定義關聯的 Model 可藉由 _透過_ 第三個 Model 來與另一個 Model 的其中一個實體 (Instance) 對應。

舉例來說，在一個車輛維修廠的應用程式中，每個 `Mechanic` (技師) Model 都可能會關聯到一個 `Car` (汽車) Model，而每個 `Car` Model 也都可能會關聯到一個 `Owner` (車主) Model。雖然技師與車主在資料庫中沒有直接的關聯，但技師可以 _透過_ `Car` Model 來存取車主。我們來看看定義此關聯所需的資料表：

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

在檢視過這個關聯的資料表結構後，我們來在 `Mechanic` Model 上定義關聯：

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

傳給 `hasOneThrough` 方法的第一個引數是我們想存取的最終 Model 名稱，而第二個引數則是中介 Model 的名稱。

或者，若關聯中所有相關的 Model 都已經定義好了關聯，我們也可以流暢地 (Fluently) 叫用 `through` 方法並提供這些關聯的名稱來定義一個「has-one-through」關聯。舉例來說，若 `Mechanic` Model 有一個 `cars` 關聯，而 `Car` Model 有一個 `owner` 關聯，則可以像這樣定義一個連接技師與車主的「has-one-through」關聯：

```php
// String based syntax...
return $this->through('cars')->has('owner');

// Dynamic syntax...
return $this->throughCars()->hasOwner();
```

<a name="has-one-through-key-conventions"></a>
#### 索引鍵慣例

執行關聯查詢時，會使用典型的 Eloquent 外鍵慣例。若想自訂關聯的鍵，可以將其作為第三與第四個引數傳給 `hasOneThrough` 方法。第三個引數是中介 Model 上的外鍵名稱。第四個引數是最終 Model 上的外鍵名稱。第五個引數是本地鍵 (Local Key)，而第六個引數則是中介 Model 上的本地鍵：

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

或者，如前所述，若關聯中所有相關的 Model 都已經定義好了關聯，我們也可以流暢地 (Fluently) 叫用 `through` 方法並提供這些關聯的名稱來定義一個「has-one-through」關聯。這種做法的好處是，可以重用既有關聯上已定義的索引鍵慣例：

```php
// String based syntax...
return $this->through('cars')->has('owner');

// Dynamic syntax...
return $this->throughCars()->hasOwner();
```

<a name="has-many-through"></a>
### Has Many Through

「遠程一對多 (has-many-through)」關聯提供了一個方便的方法，可透過中介關聯來存取遠程的關聯。舉例來說，假設我們正在建置一個像 [Laravel Vapor](https://vapor.laravel.com) 這樣的部署平台。一個 `Project` Model 可能會透過中介的 `Environment` Model 來存取多個 `Deployment` Model。使用這個範例，我們可以輕易地取得給定專案的所有部署。讓我們來看看定義此關聯所需的資料表：

    projects
        id - integer
        name - string

    environments
        id - integer
        project_id - integer
        name - string

    deployments
        id - integer
        environment_id - integer
        commit_hash - string

看完這個關聯的資料表結構後，讓我們來在 `Project` Model 上定義這個關聯：

    <?php

    namespace App\Models;

    use Illuminate\Database\Eloquent\Model;
    use Illuminate\Database\Eloquent\Relations\HasManyThrough;

    class Project extends Model
    {
        /**
         * Get all of the deployments for the project.
         */
        public function deployments(): HasManyThrough
        {
            return $this->hasManyThrough(Deployment::class, Environment::class);
        }
    }

傳給 `hasManyThrough` 方法的第一個引數是我們希望存取的最終 Model 名稱，而第二個引數則是中介 Model 的名稱。

或者，若關聯中包含的所有 Model 上都已經定義了相關的關聯，則我們也可以叫用 `through` 方法並提供這些關聯的名稱來流暢地 (fluently) 定義「遠程一對多」關聯。舉例來說，若 `Project` Model 上有 `environments` 關聯，且 `Environment` Model 上有 `deployments` 關聯，則我們可以像這樣定義一個連接 Project 與部署的「遠程一對多」關聯：

```php
// String based syntax...
return $this->through('environments')->has('deployments');

// Dynamic syntax...
return $this->throughEnvironments()->hasDeployments();
```

雖然 `Deployment` Model 的資料表不包含 `project_id` 欄位，但 `hasManyThrough` 關聯讓我們能透過 `$project->deployments` 來存取專案的部署。為了取得這些 Model，Eloquent 會檢查中介的 `Environment` Model 資料表上的 `project_id` 欄位。在找到相關的 Environment ID 後，Eloquent 會用這些 ID 來查詢 `Deployment` Model 的資料表。

<a name="has-many-through-key-conventions"></a>
#### 鍵的慣例

執行關聯查詢時，會使用典型的 Eloquent 外部鍵慣例。若想自訂關聯的鍵，可以將其作為第三個與第四個引數傳給 `hasManyThrough` 方法。第三個引數是中介 Model 上的外部鍵名稱。第四個引數是最終 Model 上的外部鍵名稱。第五個引數是本地鍵，而第六個引數則是中介 Model 的本地鍵：

    class Project extends Model
    {
        public function deployments(): HasManyThrough
        {
            return $this->hasManyThrough(
                Deployment::class,
                Environment::class,
                'project_id', // Foreign key on the environments table...
                'environment_id', // Foreign key on the deployments table...
                'id', // Local key on the projects table...
                'id' // Local key on the environments table...
            );
        }
    }

或者，如先前所述，若關聯中包含的所有 Model 上都已經定義了相關的關聯，則我們也可以叫用 `through` 方法並提供這些關聯的名稱來流暢地 (fluently) 定義「遠程一對多」關聯。這個方法的好處是，我們可以重用現有關聯上已經定義好的鍵慣例：

```php
// String based syntax...
return $this->through('environments')->has('deployments');

// Dynamic syntax...
return $this->throughEnvironments()->hasDeployments();
```

<a name="scoped-relationships"></a>
### 限定範圍的關聯

通常，我們會為 Model 新增一些額外的方法來限制 (constrain) 關聯。舉例來說，我們可能會在 `User` Model 上新增一個 `featuredPosts` 方法，這個方法會用額外的 `where` 限制式來限制較廣的 `posts` 關聯：

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

不過，若嘗試透過 `featuredPosts` 方法來建立 Model，則其 `featured` 屬性將不會被設為 `true`。若想透過關聯方法來建立 Model，並同時指定應被新增到所有透過該關聯建立的 Model 上的屬性，則可以在建構關聯查詢時使用 `withAttributes` 方法：

    /**
     * Get the user's featured posts.
     */
    public function featuredPosts(): HasMany
    {
        return $this->posts()->withAttributes(['featured' => true]);
    }

`withAttributes` 方法會使用給定的屬性為查詢新增 `where` 子句限制式，且也會將給定的屬性新增到所有透過該關聯方法建立的 Model 上：

    $post = $user->featuredPosts()->create(['title' => 'Featured Post']);

    $post->featured; // true

<a name="many-to-many"></a>
## 多對多關聯

多對多關聯比 `hasOne` 與 `hasMany` 關聯要稍微複雜一點。舉個多對多關聯的例子，一個使用者可以擁有多個角色，而這些角色也可被應用程式中的其他使用者共用。例如，一個使用者可能會被指派為「作者」與「編輯」的角色；同時，這些角色也可能被指派給其他使用者。所以，一個使用者擁有多個角色，而一個角色也擁有多個使用者。


<a name="many-to-many-table-structure"></a>
#### 資料表結構

若要定義這個關聯，則需要三個資料表：`users`、`roles`、`role_user`。`role_user` 資料表的名稱是由兩個關聯 Model 的名稱按字母順序排列推導而來，其中包含了 `user_id` 與 `role_id` 欄位。這個資料表是用來連結使用者與角色的中介資料表。

請記得，因為一個角色可屬於多個使用者，所以我們不能只在 `roles` 資料表上放一個 `user_id` 欄位。這樣做的話，就代表一個角色只能屬於單一使用者。為了要支援角色可被指派給多個使用者，我們需要 `role_user` 資料表。我們可以像這樣總結這個關聯的資料表結構：

    users
        id - integer
        name - string

    roles
        id - integer
        name - string

    role_user
        user_id - integer
        role_id - integer


<a name="many-to-many-model-structure"></a>
#### Model 結構

多對多關聯的定義方式是，撰寫一個方法來回傳 `belongsToMany` 方法的執行結果。`belongsToMany` 方法是由 `Illuminate\Database\Eloquent\Model` 基底類別提供的，你應用程式中所有的 Eloquent Model 都有使用該基底類別。舉例來說，讓我們先在 `User` Model 上定義一個 `roles` 方法。傳給這個方法的第一個引數是關聯 Model 的類別名稱：

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

定義好關聯後，就可以使用 `roles` 動態關聯屬性來存取使用者的角色：

    use App\Models\User;

    $user = User::find(1);

    foreach ($user->roles as $role) {
        // ...
    }

由於所有的關聯都可作為查詢產生器，因此我們可以在呼叫 `roles` 方法後鏈式地為關聯查詢加上更多限制條件：

    $roles = User::find(1)->roles()->orderBy('name')->get();

為了判斷關聯中介資料表的名稱，Eloquent 會將兩個關聯 Model 的名稱按字母順序組合起來。不過，你也可以不照這個慣例。只要傳遞第二個引數給 `belongsToMany` 方法即可：

    return $this->belongsToMany(Role::class, 'role_user');

除了可自訂中介資料表的名稱外，我們還可以傳遞額外的引數給 `belongsToMany` 方法來個自訂資料表上的鍵(Key)的欄位名稱。第三個引數是正在定義關聯的 Model 的外鍵名稱，而第四個引數則是要 Join 的 Model 的外鍵名稱：

    return $this->belongsToMany(Role::class, 'role_user', 'user_id', 'role_id');


<a name="many-to-many-defining-the-inverse-of-the-relationship"></a>
#### 定義反向關聯

若要定義多對多關聯的「反向」關聯，我們應在關聯的 Model 上定義一個也回傳 `belongsToMany` 方法結果的方法。為了完成我們的使用者／角色範例，讓我們先在 `Role` Model 上定義 `users` 方法：

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

如你所見，除了參考 `App\Models\User` Model 外，這個關聯的定義與其對應的 `User` Model 完全相同。由於我們重複使用了 `belongsToMany` 方法，因此在定義多對多關聯的「反向」關聯時，所有常用的資料表與鍵(Key)的自訂選項都可使用。


<a name="retrieving-intermediate-table-columns"></a>
### 取得中介資料表的欄位

如前所述，在使用多對多關聯時需要有個中介資料表。Eloquent 提供了一些與該資料表互動的實用方法。举例来说，假设我们的 `User` Model 有许多与其关联的 `Role` Model。在存取這個關聯後，我們可以使用 Model 上的 `pivot` 屬性來存取中介資料表：

    use App\Models\User;

    $user = User::find(1);

    foreach ($user->roles as $role) {
        echo $role->pivot->created_at;
    }

請注意，我們取出的每個 `Role` Model 都會自動被指派 `pivot` 屬性。這個屬性包含了一個代表中介資料表的 Model。

預設情況下，`pivot` Model 上只會有 Model 的鍵(Key)。若你的中介資料表包含額外的屬性，則必須在定義關聯時指定它們：

    return $this->belongsToMany(Role::class)->withPivot('active', 'created_by');

若想讓中介資料表有 `created_at` 與 `updated_at` 時間戳記並由 Eloquent 自動維護，請在定義關聯時呼叫 `withTimestamps` 方法：

    return $this->belongsToMany(Role::class)->withTimestamps();

> [!WARNING]  
> 使用 Eloquent 自動維護時間戳記的中介資料表，必須同時包含 `created_at` 與 `updated_at` 這兩個時間戳記欄位。


<a name="customizing-the-pivot-attribute-name"></a>
#### 自訂 `pivot` 屬性名稱

如前所述，中介資料表的屬性可透過 Model 上的 `pivot` 屬性來存取。不過，你也可以自訂這個屬性的名稱，以更好地反映其在應用程式中的目的。

舉例來說，若你的應用程式中有可訂閱 Podcast 的使用者，則你可能會有個使用者與 Podcast 間的多對多關聯。在這種情況下，你可能會想把中介資料表的屬性從 `pivot` 重新命名為 `subscription`。只要在定義關聯時使用 `as` 方法即可：

    return $this->belongsToMany(Podcast::class)
        ->as('subscription')
        ->withTimestamps();

指定了自訂的中介資料表屬性後，就可以用自訂的名稱來存取中介資料表資料：

    $users = User::with('podcasts')->get();

    foreach ($users->flatMap->podcasts as $podcast) {
        echo $podcast->subscription->created_at;
    }

<a name="filtering-queries-via-intermediate-table-columns"></a>
### 透過中介資料表欄位來篩選查詢

我們也可以在定義關聯時，使用 `wherePivot`、`wherePivotIn`、`wherePivotNotIn`、`wherePivotBetween`、`wherePivotNotBetween`、`wherePivotNull`、`wherePivotNotNull` 等方法來篩選 `belongsToMany` 關聯查詢所回傳的結果：

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

`wherePivot` 方法會為查詢新增一個 where 子句限制，但不會在透過已定義的關聯建立新 Model 時新增指定的值。若需要查詢與建立都使用特定的 pivot 值，請改用 `withPivotValue` 方法：

    return $this->belongsToMany(Role::class)
            ->withPivotValue('approved', 1);


<a name="ordering-queries-via-intermediate-table-columns"></a>
### 透過中介資料表欄位來排序查詢

我們也可以在定義 `belongsToMany` 關聯查詢時，使用 `orderByPivot` 方法來為回傳的結果排序。在下列範例中，我們會取得該使用者的所有最新徽章：

    return $this->belongsToMany(Badge::class)
        ->where('rank', 'gold')
        ->orderByPivot('created_at', 'desc');


<a name="defining-custom-intermediate-table-models"></a>
### 定義自訂的中介資料表 Model

若想為多對多關聯定義一個自訂的 Model 來代表中介資料表，可在定義關聯時呼叫 `using` 方法。自訂的 Pivot Model 讓我們有機會可在 Pivot Model 上定義額外的行為，如方法與類型轉換。

自訂的多對多 Pivot Model 應繼承 `Illuminate\Database\Eloquent\Relations\Pivot` 類別，而自訂的 Polymorphic (多態) 多對多 Pivot Model 則應繼承 `Illuminate\Database\Eloquent\Relations\MorphPivot` 類別。舉例來說，我們可以定義一個 `Role` Model，並讓它使用自訂的 `RoleUser` Pivot Model：

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

在定義 `RoleUser` Model 時，應繼承 `Illuminate\Database\Eloquent\Relations\Pivot` 類別：

    <?php

    namespace App\Models;

    use Illuminate\Database\Eloquent\Relations\Pivot;

    class RoleUser extends Pivot
    {
        // ...
    }

> [!WARNING]  
> Pivot Model 不可使用 `SoftDeletes` Trait。若需要對 Pivot 記錄進行軟刪除，請考慮將 Pivot Model 轉換為實際的 Eloquent Model。


<a name="custom-pivot-models-and-incrementing-ids"></a>
#### 自訂 Pivot Model 與遞增 ID

若已定義了使用自訂 Pivot Model 的多對多關聯，且該 Pivot Model 有自動遞增的主鍵，則應確保自訂的 Pivot Model 類別中有定義一個設為 `true` 的 `incrementing` 屬性。

    /**
     * Indicates if the IDs are auto-incrementing.
     *
     * @var bool
     */
    public $incrementing = true;

<a name="polymorphic-relationships"></a>
## Polymorphic (多態) 關聯

Polymorphic (多態) 關聯可讓子 Model 能使用單個關聯來從屬於多種類型的 Model。舉例來說，假設我們正在建立一個能讓使用者分享部落格文章與影片的應用程式。在這樣的應用程式中，一個 `Comment` Model 可能會同時從屬於 `Post` 與 `Video` Model。


<a name="one-to-one-polymorphic-relations"></a>
### 一對一


<a name="one-to-one-polymorphic-table-structure"></a>
#### 資料表結構

「一對一 Polymorphic (多態)」關聯類似於一般的一對一關聯；不過，子 Model 可使用單一關聯來從屬於多種類型的 Model。舉例來說，一篇部落格的 `Post` 與一個 `User` 可共享一個與 `Image` Model 的 Polymorphic (多態) 關聯。使用一對一 Polymorphic (多態) 關聯可讓我們用單個資料表來儲存可與文章或使用者關聯的獨立圖片。首先，我們先來看看資料表結構：

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

請注意 `images` 資料表上的 `imageable_id` 與 `imageable_type` 欄位。`imageable_id` 欄位會包含文章或使用者的 ID 值，而 `imageable_type` 欄位則會包含父 Model 的類別名稱。Eloquent 會使用 `imageable_type` 欄位來判斷在存取 `imageable` 關聯時要回傳何種「類型」的父 Model。在這個例子中，該欄位會包含 `App\Models\Post` 或 `App\Models\User`。


<a name="one-to-one-polymorphic-model-structure"></a>
#### Model 結構

接著，我們來看看建立此關聯所需的 Model 定義：

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


<a name="one-to-one-polymorphic-retrieving-the-relationship"></a>
#### 取得關聯

定義好資料表與 Model 後，就可以透過 Model 來存取關聯了。舉例來說，若要取得文章的圖片，我們可以存取 `image` 動態關聯屬性：

    use App\Models\Post;

    $post = Post::find(1);

    $image = $post->image;

我們可以存取呼叫 `morphTo` 的方法名稱來取得 Polymorphic (多態) Model 的父項。在這個例子中，也就是 `Image` Model 上的 `imageable` 方法。所以，我們要將該方法作為動態關聯屬性來存取：

    use App\Models\Image;

    $image = Image::find(1);

    $imageable = $image->imageable;

`Image` Model 上的 `imageable` 關聯會回傳 `Post` 或 `User` 的實體，取決於是哪種 Model 擁有該圖片。


<a name="morph-one-to-one-key-conventions"></a>
#### 主鍵慣例

如有需要，可以指定 Polymorphic (多態) 子 Model 所使用的「id」與「type」欄位名稱。若這麼做，請務必將關聯名稱作為第一個引數傳給 `morphTo` 方法。一般來說，這個值應符合方法名稱，所以可以使用 PHP 的 `__FUNCTION__` 常數：

    /**
     * Get the model that the image belongs to.
     */
    public function imageable(): MorphTo
    {
        return $this->morphTo(__FUNCTION__, 'imageable_type', 'imageable_id');
    }

<a name="one-to-many-polymorphic-relations"></a>
### 一對多

<a name="one-to-many-polymorphic-table-structure"></a>
#### 資料表結構

一對多多態關聯與一般的一對多關聯類似；不過，子 Model 可透過單一關聯屬於多種不同型別的 Model。舉例來說，假設應用程式中的使用者可以對文章或影片進行「留言」(Comment)。使用多態關聯，就可以用單一的 `comments` 資料表來包含文章與影片的留言。首先，我們先來看看建立此關聯所需的資料表結構：

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

<a name="one-to-many-polymorphic-model-structure"></a>
#### Model 結構

接著，我們來看看建立此關聯所需的 Model 定義：

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

<a name="one-to-many-polymorphic-retrieving-the-relationship"></a>
#### 取得關聯

定義好資料表與 Model 後，就可以透過 Model 的動態關聯屬性來存取關聯。舉例來說，若要存取某篇文章的所有留言，可使用 `comments` 動態屬性：

    use App\Models\Post;

    $post = Post::find(1);

    foreach ($post->comments as $comment) {
        // ...
    }

我們也可以透過存取呼叫 `morphTo` 方法的方法名稱來取得多態子 Model 的父項。在這個例子中，這個方法就是 `Comment` Model 上的 `commentable` 方法。因此，我們會將該方法作為動態關聯屬性來存取留言的父 Model：

    use App\Models\Comment;

    $comment = Comment::find(1);

    $commentable = $comment->commentable;

`Comment` Model 上的 `commentable` 關聯會回傳一個 `Post` 或 `Video` 的實體，取決於是哪種型別的 Model 為該留言的父項。

<a name="polymorphic-automatically-hydrating-parent-models-on-children"></a>
#### 在子 Model 上自動 Hydrate (填充) 父 Model

即使在使用 Eloquent 預載入時，若在遍歷子 Model 的迴圈中嘗試存取父 Model，仍可能發生「N + 1」查詢問題：

```php
$posts = Post::with('comments')->get();

foreach ($posts as $post) {
    foreach ($post->comments as $comment) {
        echo $comment->commentable->title;
    }
}
```

在上述範例中，會發生「N + 1」查詢問題，這是因為，即使我們為每個 `Post` Model 都預載入了留言，Eloquent 也不會自動在每個子 `Comment` Model 上 Hydrate (填充) 其父 `Post` Model。

若想讓 Eloquent 自動將父 Model Hydrate (填充) 到其子 Model 上，可以在定義 `morphMany` 關聯時叫用 `chaperone` 方法：

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

或者，若想在執行時期才選擇是否要自動 Hydrate (填充) 父 Model，可以在預載入關聯時叫用 `chaperone` Model：

```php
use App\Models\Post;

$posts = Post::with([
    'comments' => fn ($comments) => $comments->chaperone(),
])->get();
```

<a name="one-of-many-polymorphic-relations"></a>
### One of Many

有時候，一個 Model 可能會有很多個關聯 Model，但我們只想輕鬆地取得關聯中「最新」或「最舊」的那個關聯 Model。舉例來說，一個 `User` Model 可能會關聯到多個 `Image` Model，但我們想定義一個方便的方法來操作使用者最近上傳的圖片。我們可以透過 `morphOne` 關聯型別並搭配 `ofMany` 系列方法來達成此目的：

```php
/**
 * Get the user's most recent image.
 */
public function latestImage(): MorphOne
{
    return $this->morphOne(Image::class, 'imageable')->latestOfMany();
}
```

同樣地，我們也可以定義一個方法來取得關聯中「最舊」或第一個關聯 Model：

```php
/**
 * Get the user's oldest image.
 */
public function oldestImage(): MorphOne
{
    return $this->morphOne(Image::class, 'imageable')->oldestOfMany();
}
```

預設情況下，`latestOfMany` 與 `oldestOfMany` 方法會根據 Model 的主鍵 (必須可排序) 來取得最新或最舊的關聯 Model。不過，有時候我們可能會想用不同的排序條件從一個較大的關聯中只取出單一 Model。

舉例來說，透過 `ofMany` 方法，我們可以取得使用者「最多讚」的圖片。`ofMany` 方法的第一個引數為可排序的欄位，第二個引數則為查詢關聯 Model 時要套用的彙總函式 (`min` 或 `max`)：

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
> 我們可以建構更進階的「One of Many」關聯。更多資訊，請參考 [Has One of Many 說明文件](#advanced-has-one-of-many-relationships)。

<a name="many-to-many-polymorphic-relations"></a>
### 多對多

<a name="many-to-many-polymorphic-table-structure"></a>
#### 資料表結構

多對多 Polymorphic 關聯比「morph one」與「morph many」關聯要來得稍微複雜一些。舉例來說，一個 `Post` Model 與一個 `Video` Model 可以共享一個與 `Tag` Model 的 Polymorphic 關聯。在這種情況下，使用多對多 Polymorphic 關聯可讓你的應用程式使用單一張唯一的標籤 (Tag) 資料表，並將這些標籤關聯到文章或影片上。首先，我們先來看看建立這個關聯所需的資料表結構：

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

> [!NOTE]  
> 在深入了解多對多 Polymorphic 關聯前，建議先閱讀有關一般[多對多關聯](#many-to-many)的文件，這會對你有所幫助。

<a name="many-to-many-polymorphic-model-structure"></a>
#### Model 結構

接下來，我們就可以來定義 Model 上的關聯了。`Post` 與 `Video` Model 都會包含一個 `tags` 方法，該方法會呼叫基礎 Eloquent Model Class 所提供的 `morphToMany` 方法。

`morphToMany` 方法可接受關聯 Model 的名稱以及「關聯名稱」。根據我們為中介資料表指定的名稱以及其中包含的索引鍵，我們會將這個關聯稱為「taggable」：

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

<a name="many-to-many-polymorphic-defining-the-inverse-of-the-relationship"></a>
#### 定義反向關聯

接下來，在 `Tag` Model 上，我們應為每個可能的父 Model 都定義一個方法。因此，在這個範例中，我們會定義一個 `posts` 方法與一個 `videos` 方法。這兩個方法都應回傳 `morphedByMany` 方法的結果。

`morphedByMany` 方法可接受關聯 Model 的名稱以及「關聯名稱」。根據我們為中介資料表指定的名稱以及其中包含的索引鍵，我們會將這個關聯稱為「taggable」：

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

<a name="many-to-many-polymorphic-retrieving-the-relationship"></a>
#### 取得關聯

在定義好資料庫資料表與 Model 後，就可以透過 Model 來存取關聯了。舉例來說，若要存取某篇文章的所有標籤，可以使用 `tags` 動態關聯屬性：

    use App\Models\Post;

    $post = Post::find(1);

    foreach ($post->tags as $tag) {
        // ...
    }

我們可以透過存取呼叫 `morphedByMany` 的方法名稱來從 Polymorphic 子 Model 上取得 Polymorphic 關聯的父項。在這個例子中，也就是 `Tag` Model 上的 `posts` 或 `videos` 方法：

    use App\Models\Tag;

    $tag = Tag::find(1);

    foreach ($tag->posts as $post) {
        // ...
    }

    foreach ($tag->videos as $video) {
        // ...
    }

<a name="custom-polymorphic-types"></a>
### 自訂 Polymorphic (多態) 型別

預設情況下，Laravel 會使用完整的 Class 名稱來儲存關聯 Model 的「型別 (Type)」。舉例來說，在上述的一對多關聯範例中，`Comment` Model 可屬於 `Post` 或 `Video` Model，則預設的 `commentable_type` 就會分別是 `App\Models\Post` 或 `App\Models\Video`。不過，我們可能會想將這些值與應用程式的內部結構解耦 (Decouple)。

舉例來說，與其使用 Model 名稱作為「型別」，我們可以使用如 `post` 與 `video` 等簡單的字串。這麼一來，即使 Model 被重新命名，資料庫中多態「型別」欄位的值也仍然有效：

    use Illuminate\Database\Eloquent\Relations\Relation;

    Relation::enforceMorphMap([
        'post' => 'App\Models\Post',
        'video' => 'App\Models\Video',
    ]);

我們可以在 `App\Providers\AppServiceProvider` Class 的 `boot` 方法中呼叫 `enforceMorphMap` 方法，或也可以建立一個獨立的 Service Provider 來做這件事。

我們可以在執行階段時使用 Model 的 `getMorphClass` 方法來判斷某個 Model 的 Morph 別名。反過來，也可以使用 `Relation::getMorphedModel` 方法來判斷與 Morph 別名關聯的完整 Class 名稱：

    use Illuminate\Database\Eloquent\Relations\Relation;

    $alias = $post->getMorphClass();

    $class = Relation::getMorphedModel($alias);

> [!WARNING]  
> 在將「Morph Map」加到現有應用程式中時，資料庫中每個仍包含完整 Class 名稱的可 Morph 的 `*_type` 欄位值都必須被轉換為其「Map」名稱。

<a name="dynamic-relationships"></a>
### 動態關聯

我們可以使用 `resolveRelationUsing` 方法來在執行階段定義 Eloquent Model 間的關聯。雖然一般不建議在正常的應用程式開發中使用此方法，但在開發 Laravel 套件時偶爾會很有用。

`resolveRelationUsing` 方法的第一個引數是想要的關聯名稱。傳給該方法的第二個引數則應為一個 Closure，該 Closure 可接受 Model 實體並回傳一個有效的 Eloquent 關聯定義。一般來說，我們應在 [Service Provider](/docs/{{version}}/providers) 的 boot 方法中設定動態關聯：

    use App\Models\Order;
    use App\Models\Customer;

    Order::resolveRelationUsing('customer', function (Order $orderModel) {
        return $orderModel->belongsTo(Customer::class, 'customer_id');
    });

> [!WARNING]  
> 在定義動態關聯時，請務必為 Eloquent 關聯方法提供明確的索引鍵名稱引數。

<a name="querying-relations"></a>
## 查詢關聯

由於所有的 Eloquent 關聯都是透過方法來定義的，因此我們可以呼叫這些方法來取得關聯的實體，而不需要實際執行查詢來載入關聯 Model。此外，所有 Eloquent 關聯的類型也都可以作為[查詢產生器](/docs/{{version}}/queries)來使用，讓開發者能在最終對資料庫執行 SQL 查詢前，先在關聯查詢上繼續鏈結條件。

舉例來說，假設有個部落格應用程式，其中 `User` Model 有許多關聯的 `Post` Model：

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

我們可以像這樣查詢 `posts` 關聯，並為該關聯加上額外的條件：

    use App\Models\User;

    $user = User::find(1);

    $user->posts()->where('active', 1)->get();

我們可以在關聯上使用任何 Laravel [查詢產生器](/docs/{{version}}/queries)的方法，請務必參閱查詢產生器的說明文件來了解所有可用的方法。

<a name="chaining-orwhere-clauses-after-relationships"></a>
#### 在關聯後鏈結 `orWhere` 子句

如上例所示，我們可以在查詢關聯時自由地為其加上額外的條件。不過，在關聯上鏈結 `orWhere` 子句時要特別小心，因為 `orWhere` 子句會與關聯的條件被邏輯分組在同一個層級上：

    $user->posts()
            ->where('active', 1)
            ->orWhere('votes', '>=', 100)
            ->get();

上方程式碼會產生下列 SQL。如你所見，`or` 子句會讓查詢回傳票數大於 100 的*任何*文章。查詢不再被限制在特定使用者上：

```sql
select *
from posts
where user_id = ? and active = 1 or votes >= 100
```

在大多數情況下，我們應使用[邏輯分組](/docs/{{version}}/queries#logical-grouping)來將條件檢查群組在括號之間：

    use Illuminate\Database\Eloquent\Builder;

    $user->posts()
        ->where(function (Builder $query) {
            return $query->where('active', 1)
                ->orWhere('votes', '>=', 100);
        })
        ->get();

上方程式碼會產生下列 SQL。請注意，邏輯分組已正確地將條件群組起來，且查詢仍被限制在特定使用者上：

```sql
select *
from posts
where user_id = ? and (active = 1 or votes >= 100)
```

<a name="relationship-methods-vs-dynamic-properties"></a>
### 關聯方法 vs. 動態屬性

若不需要為 Eloquent 關聯查詢加上額外條件，我們可以像存取屬性一樣來存取關聯。舉例來說，繼續使用剛才的 `User` 與 `Post` Model 範例，我們可以像這樣存取某個使用者的所有文章：

    use App\Models\User;

    $user = User::find(1);

    foreach ($user->posts as $post) {
        // ...
    }

動態關聯屬性會執行「延遲載入 (Lazy Loading)」，代表只有在實際存取到關聯資料時才會載入。因此，開發人員通常會使用[預載入 (Eager Loading)](#eager-loading) 來預先載入在載入 Model 後確定會存取到的關聯。預載入可以大幅減少載入 Model 關聯時所需的 SQL 查詢次數。

<a name="querying-relationship-existence"></a>
### 查詢關聯是否存在

在擷取 Model 記錄時，我們可能希望根據關聯是否存在來限制結果。舉例來說，假設我們想擷取所有至少有一則留言的文章。為此，我們可以將關聯的名稱傳給 `has` 與 `orHas` 方法：

    use App\Models\Post;

    // Retrieve all posts that have at least one comment...
    $posts = Post::has('comments')->get();

我們也可以指定一個運算子與計數值來進一步自訂查詢：

    // Retrieve all posts that have three or more comments...
    $posts = Post::has('comments', '>=', 3)->get();

巢狀的 `has` 陳述式可使用「點」號來建構。舉例來說，我們可以擷取所有至少有一則留言、且該留言至少有一張圖片的文章：

    // Retrieve posts that have at least one comment with images...
    $posts = Post::has('comments.images')->get();

若需要更強大的功能，我們可以使用 `whereHas` 與 `orWhereHas` 方法來為 `has` 查詢定義額外的查詢條件，例如檢查留言的內容：

    use Illuminate\Database\Eloquent\Builder;

    // Retrieve posts with at least one comment containing words like code%...
    $posts = Post::whereHas('comments', function (Builder $query) {
        $query->where('content', 'like', 'code%');
    })->get();

    // Retrieve posts with at least ten comments containing words like code%...
    $posts = Post::whereHas('comments', function (Builder $query) {
        $query->where('content', 'like', 'code%');
    }, '>=', 10)->get();

> [!WARNING]  
> Eloquent 目前不支援跨資料庫查詢關聯是否存在。關聯必須存在於同一個資料庫中。

<a name="inline-relationship-existence-queries"></a>
#### 行內關聯存在查詢

若想查詢某個關聯是否存在，且只須在關聯查詢上附加一個簡單的 where 條件，那麼使用 `whereRelation`、`orWhereRelation`、`whereMorphRelation` 與 `orWhereMorphRelation` 方法會更方便。舉例來說，我們可以查詢所有有未核准留言的文章：

    use App\Models\Post;

    $posts = Post::whereRelation('comments', 'is_approved', false)->get();

當然，就像呼叫查詢產生器的 `where` 方法一樣，我們也可以指定運算子：

    $posts = Post::whereRelation(
        'comments', 'created_at', '>=', now()->subHour()
    )->get();

<a name="querying-relationship-absence"></a>
### 查詢關聯是否不存在

在擷取 Model 記錄時，我們可能希望根據某個關聯不存在來限制結果。舉例來說，假設我們想擷取所有**沒有**任何留言的文章。為此，我們可以將關聯的名稱傳給 `doesntHave` 與 `orDoesntHave` 方法：

    use App\Models\Post;

    $posts = Post::doesntHave('comments')->get();

若需要更強大的功能，我們可以使用 `whereDoesntHave` 與 `orWhereDoesntHave` 方法來為 `doesntHave` 查詢加上額外的查詢條件，例如檢查留言的內容：

    use Illuminate\Database\Eloquent\Builder;

    $posts = Post::whereDoesntHave('comments', function (Builder $query) {
        $query->where('content', 'like', 'code%');
    })->get();

我們可以使用「點」號來對巢狀的關聯執行查詢。舉例來說，下列查詢會擷取所有沒有留言的文章；不過，若文章有留言，但留言的作者未被封鎖，則該文章還是會被包含在結果中：

    use Illuminate\Database\Eloquent\Builder;

    $posts = Post::whereDoesntHave('comments.author', function (Builder $query) {
        $query->where('banned', 0);
    })->get();

<a name="querying-morph-to-relationships"></a>
### 查詢 Morph To 關聯

若要查詢「morph to」關聯是否存在，可使用 `whereHasMorph` 與 `whereDoesntHaveMorph` 方法。這些方法的第一個引數為關聯的名稱。接下來，這些方法可接受要包含在查詢內的關聯 Model 名稱。最後，我們還可以提供一個閉包來自訂關聯查詢：

    use App\Models\Comment;
    use App\Models\Post;
    use App\Models\Video;
    use Illuminate\Database\Eloquent\Builder;

    // 取得標題類似 code% 且與貼文或影片關聯的留言⋯⋯
    $comments = Comment::whereHasMorph(
        'commentable',
        [Post::class, Video::class],
        function (Builder $query) {
            $query->where('title', 'like', 'code%');
        }
    )->get();

    // 取得標題不類似 code% 且與貼文關聯的留言⋯⋯
    $comments = Comment::whereDoesntHaveMorph(
        'commentable',
        Post::class,
        function (Builder $query) {
            $query->where('title', 'like', 'code%');
        }
    )->get();

有時候，我們可能需要根據關聯的多態 Model「型別 (type)」來加上查詢限制。傳給 `whereHasMorph` 方法的閉包可接受 `$type` 值作為其第二個引數。這個引數可讓我們檢查正在建立的查詢的「型別」：

    use Illuminate\Database\Eloquent\Builder;

    $comments = Comment::whereHasMorph(
        'commentable',
        [Post::class, Video::class],
        function (Builder $query, string $type) {
            $column = $type === Post::class ? 'content' : 'title';

            $query->where($column, 'like', 'code%');
        }
    )->get();

有時候，我們可能想查詢某個「morph to」關聯的父項的子項。我們可以使用 `whereMorphedTo` 與 `whereNotMorphedTo` 方法來達成。這兩個方法會自動為給定的 Model 判斷正確的 morph 型別對應。這些方法的第一個引數為 `morphTo` 關聯的名稱，第二個引數則為相關的父 Model：

    $comments = Comment::whereMorphedTo('commentable', $post)
        ->orWhereMorphedTo('commentable', $video)
        ->get();


<a name="querying-all-morph-to-related-models"></a>
#### 查詢所有關聯 Model

除了傳入一個包含可能的多態 Model 的陣列外，我們也可以提供 `*` 作為萬用字元值。這會讓 Laravel 從資料庫中取得所有可能的多態型別。Laravel 會執行一個額外的查詢來執行此操作：

    use Illuminate\Database\Eloquent\Builder;

    $comments = Comment::whereHasMorph('commentable', '*', function (Builder $query) {
        $query->where('title', 'like', 'foo%');
    })->get();

<a name="aggregating-related-models"></a>
## 彙總關聯 Model


<a name="counting-related-models"></a>
### 計算關聯 Model

有時候，我們可能會想在不實際載入 Model 的情況下，計算某個關聯的關聯 Model 數量。為此，可使用 `withCount` 方法。`withCount` 方法會將一個 `{relation}_count` 屬性放到結果 Model 上：

    use App\Models\Post;

    $posts = Post::withCount('comments')->get();

    foreach ($posts as $post) {
        echo $post->comments_count;
    }

只要傳遞一個陣列給 `withCount` 方法，就可以為多個關聯新增「計數」，並為查詢加上額外的限制：

    use Illuminate\Database\Eloquent\Builder;

    $posts = Post::withCount(['votes', 'comments' => function (Builder $query) {
        $query->where('content', 'like', 'code%');
    }])->get();

    echo $posts[0]->votes_count;
    echo $posts[0]->comments_count;

也可以為關聯計數的結果取別名，以便在同一個關聯上進行多次計數：

    use Illuminate\Database\Eloquent\Builder;

    $posts = Post::withCount([
        'comments',
        'comments as pending_comments_count' => function (Builder $query) {
            $query->where('approved', false);
        },
    ])->get();

    echo $posts[0]->comments_count;
    echo $posts[0]->pending_comments_count;


<a name="deferred-count-loading"></a>
#### 延遲計數載入

使用 `loadCount` 方法，可在父項 Model 已被擷取後，才載入關聯計數：

    $book = Book::first();

    $book->loadCount('genres');

若需要在計數查詢上設定額外的查詢限制，可傳遞一個以要計數的關聯為索引鍵的陣列。該陣列的值應為一個閉包，其中會收到查詢產生器實體：

    $book->loadCount(['reviews' => function (Builder $query) {
        $query->where('rating', 5);
    }])


<a name="relationship-counting-and-custom-select-statements"></a>
#### 關聯計數與自訂 Select 陳述式

若要將 `withCount` 與 `select` 陳述式合併使用，請確保在 `select` 方法之後才呼叫 `withCount`：

    $posts = Post::select(['title', 'body'])
        ->withCount('comments')
        ->get();


<a name="other-aggregate-functions"></a>
### 其他彙總函式

除了 `withCount` 方法外，Eloquent 還提供了 `withMin`、`withMax`、`withAvg`、`withSum` 與 `withExists` 方法。這些方法會將 `{relation}_{function}_{column}` 屬性放到結果 Model 上：

    use App\Models\Post;

    $posts = Post::withSum('comments', 'votes')->get();

    foreach ($posts as $post) {
        echo $post->comments_sum_votes;
    }

若想用另一個名稱來存取彙總函式的結果，可指定自己的別名：

    $posts = Post::withSum('comments as total_comments', 'votes')->get();

    foreach ($posts as $post) {
        echo $post->total_comments;
    }

與 `loadCount` 方法類似，這些方法也有延遲版本。這些額外的彙總操作可在已經被擷取的 Eloquent Model 上執行：

    $post = Post::first();

    $post->loadSum('comments', 'votes');

若要將這些彙總方法與 `select` 陳述式合併使用，請確保在 `select` 方法之後才呼叫這些彙總方法：

    $posts = Post::select(['title', 'body'])
        ->withExists('comments')
        ->get();


<a name="counting-related-models-on-morph-to-relationships"></a>
### 在 Morph To 關聯上計算關聯 Model

若想預載入 (Eager Load) 一個「Morph To」關聯，以及該關聯可能回傳的各種實體的關聯 Model 計數，可使用 `with` 方法並搭配 `morphTo` 關聯的 `morphWithCount` 方法。

在此範例中，假設 `Photo` 與 `Post` Model 可建立 `ActivityFeed` Model。我們假設 `ActivityFeed` Model 定義了一個名為 `parentable` 的「Morph To」關聯，讓我們能為給定的 `ActivityFeed` 實體擷取父項 `Photo` 或 `Post` Model。此外，假設 `Photo` Model 「有很多」`Tag` Model，而 `Post` Model「有很多」`Comment` Model。

現在，假設我們想擷取 `ActivityFeed` 實體，並為每個 `ActivityFeed` 實體預載入 (Eager Load) 其 `parentable` 父項 Model。此外，我們還想擷取與每個父項 Photo 關聯的 Tag 數量，以及與每個父項 Post 關聯的 Comment 數量：

    use Illuminate\Database\Eloquent\Relations\MorphTo;

    $activities = ActivityFeed::with([
        'parentable' => function (MorphTo $morphTo) {
            $morphTo->morphWithCount([
                Photo::class => ['tags'],
                Post::class => ['comments'],
            ]);
        }])->get();


<a name="morph-to-deferred-count-loading"></a>
#### 延遲計數載入

假設我們已擷取了一組 `ActivityFeed` Model，而現在想為這些 Activity Feed 關聯的各種 `parentable` Model 載入巢狀的關聯計數。可使用 `loadMorphCount` 方法來達成此目的：

    $activities = ActivityFeed::with('parentable')->get();

    $activities->loadMorphCount('parentable', [
        Photo::class => ['tags'],
        Post::class => ['comments'],
    ]);

<a name="eager-loading"></a>
## 預載入 (Eager Loading)

在把 Eloquent 關聯當作屬性來存取時，關聯 Model 會被「延遲載入 (Lazy Loaded)」。這表示，關聯資料要一直到我們第一次存取該屬性時才會被載入。不過，Eloquent 可以在查詢父項 Model 的時候就「預載入 (Eager Load)」關聯。預載入可以緩解「N + 1」查詢問題。為了說明 N + 1 查詢問題，我們來看看一個「屬於 (belongs to)」`Author` Model 的 `Book` Model：

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

現在，我們來取得所有的書與其作者：

    use App\Models\Book;

    $books = Book::all();

    foreach ($books as $book) {
        echo $book->author->name;
    }

這個迴圈會執行 1 次查詢來從資料表中取得所有的書，然後，為了取得每本書的作者，又為每本書各執行了 1 次查詢。所以，若我們有 25 本書，則上面的程式碼會執行 26 次查詢：1 次用來取得原本的書，另外 25 次查詢則用來取得每本書的作者。

幸好，我們可以使用預載入 (Eager Loading) 來將這個操作的查詢次數減少到只有 2 次。在建立查詢時，可以使用 `with` 方法來指定應被預載入的關聯：

    $books = Book::with('author')->get();

    foreach ($books as $book) {
        echo $book->author->name;
    }

在這個操作中，只會執行 2 次查詢 —— 1 次查詢用來取得所有的書，另 1 次查詢用來取得所有書的所有作者：

```sql
select * from books

select * from authors where id in (1, 2, 3, 4, 5, ...)
```

<a name="eager-loading-multiple-relationships"></a>
#### 預載入多個關聯

有時候，我們可能會需要預載入好幾個不同的關聯。若要這麼做，只需要傳入一個包含關聯的陣列給 `with` 方法即可：

    $books = Book::with(['author', 'publisher'])->get();

<a name="nested-eager-loading"></a>
#### 巢狀預載入

若要預載入某個關련中的關聯，可以使用「點 (.)」語法。舉例來說，我們來預載入所有書的作者，以及所有作者的聯絡人：

    $books = Book::with('author.contacts')->get();

或者，我們也可以提供一個巢狀陣列給 `with` 方法來指定巢狀的預載入關聯。在預載入多個巢狀關聯時，這個方法很方便：

    $books = Book::with([
        'author' => [
            'contacts',
            'publisher',
        ],
    ])->get();

<a name="nested-eager-loading-morphto-relationships"></a>
#### 巢狀預載入 `morphTo` 關聯

若想預載入 `morphTo` 關聯，以及該關聯可能回傳的各種實體上的巢狀關聯，可以使用 `with` 方法，並搭配 `morphTo` 關聯的 `morphWith` 方法。為了說明這個方法，我們先來看看下列 Model：

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

在這個範例中，我們假設 `Event`、`Photo`、`Post` 等 Model 可以建立 `ActivityFeed` Model。此外，我們也假設 `Event` Model 屬於 `Calendar` Model、`Photo` Model 與 `Tag` Model 有關聯、而 `Post` Model 則屬於 `Author` Model。

使用這些 Model 定義與關聯，我們可以取得 `ActivityFeed` Model 的實體，並預載入所有的 `parentable` Model 與其各自的巢狀關聯：

    use Illuminate\Database\Eloquent\Relations\MorphTo;

    $activities = ActivityFeed::query()
        ->with(['parentable' => function (MorphTo $morphTo) {
            $morphTo->morphWith([
                Event::class => ['calendar'],
                Photo::class => ['tags'],
                Post::class => ['author'],
            ]);
        }])->get();

<a name="eager-loading-specific-columns"></a>
#### 預載入特定欄位

有時候我們不一定需要關聯中所有的欄位。因此，Eloquent 讓你可以指定關聯中想取得的欄位：

    $books = Book::with('author:id,name,book_id')->get();

> [!WARNING]
> 在使用這個功能時，應總是在想取得的欄位列表中包含 `id` 欄位與任何相關的外鍵欄位。

<a name="eager-loading-by-default"></a>
#### 預設預載入

有時候我們可能會想在取得某個 Model 時總是載入某些關聯。若要這麼做，可以在該 Model 上定義一個 `$with` 屬性：

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

若想在某個查詢中從 `$with` 屬性移除某個項目，可以使用 `without` 方法：

    $books = Book::without('author')->get();

若想在某個查詢中覆寫 `$with` 屬性內的所有項目，可以使用 `withOnly` 方法：

    $books = Book::withOnly('genre')->get();

<a name="constraining-eager-loads"></a>
### 限制預載入

有時候，我們可能會想預載入關聯，但同時也想為該預載入查詢加上額外的查詢條件。我們可以傳遞一個關聯陣列給 `with` 方法來達成這個目的。在該陣列中，Key 為關聯的名稱，而 Value 則為一個 Closure，可用來為預載入查詢加上額外限制：

    use App\Models\User;
    use Illuminate\Contracts\Database\Eloquent\Builder;

    $users = User::with(['posts' => function (Builder $query) {
        $query->where('title', 'like', '%code%');
    }])->get();

在此範例中，Eloquent 只會預載入 `title` 欄位包含 `code` 字眼的 Post。我們也可以呼叫其他的[查詢產生器](/docs/{{version}}/queries)方法來進一步自訂預載入操作：

    $users = User::with(['posts' => function (Builder $query) {
        $query->orderBy('created_at', 'desc');
    }])->get();

<a name="constraining-eager-loading-of-morph-to-relationships"></a>
#### 限制 `morphTo` 關聯的預載入

若要預載入一個 `morphTo` 關聯，Eloquent 會執行多個查詢來擷取各種類型的關聯 Model。我們可以使用 `MorphTo` 關聯的 `constrain` 方法來為這些查詢加上額外的限制：

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

在此範例中，Eloquent 只會預載入尚未被隱藏的 Post、以及 `type` 值為「educational」的 Video。

<a name="constraining-eager-loads-with-relationship-existence"></a>
#### 透過關聯存在性來限制預載入

有時候我們可能會需要在檢查關聯存在性的同時，根據相同的條件來載入關聯。举例来说，我們可能只想擷取有符合特定查詢條件的子 `Post` Model 的 `User` Model，並同時預載入這些符合條件的 Post。這時可以使用 `withWhereHas` 方法來達成：

    use App\Models\User;

    $users = User::withWhereHas('posts', function ($query) {
        $query->where('featured', true);
    })->get();

<a name="lazy-eager-loading"></a>
### 延遲預載入 (Lazy Eager Loading)

有時候，我們可能會需要在擷取完父項 Model 後才需要預載入關聯。舉例來說，當我們需要動態決定是否要載入關聯 Model 時，這個功能就很有用：

    use App\Models\Book;

    $books = Book::all();

    if ($someCondition) {
        $books->load('author', 'publisher');
    }

若要為預載入查詢設定額外的查詢條件，可以傳遞一個以關聯為 Key 的陣列。陣列的 Value 應為 Closure 的實體，該 Closure 會收到查詢的實體：

    $author->load(['books' => function (Builder $query) {
        $query->orderBy('published_date', 'asc');
    }]);

若只要在關聯尚未載入時才載入關聯，請使用 `loadMissing` 方法：

    $book->loadMissing('author');

<a name="nested-lazy-eager-loading-morphto"></a>
#### 巢狀延遲預載入與 `morphTo`

若想預載入 `morphTo` 關聯，以及該關聯可能回傳的各種實體上的巢狀關聯，可以使用 `loadMorph` 方法。

這個方法的第一個引數是 `morphTo` 關聯的名稱，而第二個引數則是一個 Model / 關聯的配對陣列。為了說明這個方法，我們先來看看下列 Model：

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

在這個範例中，我們假設 `Event`、`Photo`、與 `Post` Model 都可以建立 `ActivityFeed` Model。此外，我們也假設 `Event` Model 屬於 `Calendar` Model、`Photo` Model 與 `Tag` Model 有關聯、而 `Post` Model 則屬於 `Author` Model。

使用這些 Model 定義與關聯，我們就可以擷取 `ActivityFeed` Model 的實體，並預載入所有的 `parentable` Model 與其各自的巢狀關聯：

    $activities = ActivityFeed::with('parentable')
        ->get()
        ->loadMorph('parentable', [
            Event::class => ['calendar'],
            Photo::class => ['tags'],
            Post::class => ['author'],
        ]);

<a name="preventing-lazy-loading"></a>
### 防止延遲載入 (Lazy Loading)

如前所述，預載入關聯通常能為應用程式帶來顯著的效能提升。因此，如有需要，我們可以指示 Laravel 一律防止延遲載入關聯。為此，我們可以叫用 Eloquent 基底 Model 類別提供的 `preventLazyLoading` 方法。一般來說，我們應在應用程式 `AppServiceProvider` 類別的 `boot` 方法中呼叫此方法。

`preventLazyLoading` 方法可接受一個可選的布林引數，用來表示是否應防止延遲載入。舉例來說，我們可能只想在非正式版 (Production) 環境中停用延遲載入，這樣一來，即使正式版程式碼中不小心出現了延遲載入的關聯，正式版環境還是能正常運作：

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

防止延遲載入後，當應用程式嘗試延遲載入任何 Eloquent 關聯時，Eloquent 都會擲回 `Illuminate\Database\LazyLoadingViolationException` 例外。

我們可以使用 `handleLazyLoadingViolationsUsing` 方法來自訂延遲載入違規的行為。舉例來說，使用這個方法，我們可以指示延遲載入違規只需被記錄下來，而不需要擲回例外來中斷應用程式的執行：

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

Eloquent 提供了方便的方法來為關聯新增新的 Model。舉例來說，假設我們要為一篇文章新增一則留言。與其手動設定 `Comment` Model 上的 `post_id` 屬性，我們可以使用關聯的 `save` 方法來新增留言：

    use App\Models\Comment;
    use App\Models\Post;

    $comment = new Comment(['message' => 'A new comment.']);

    $post = Post::find(1);

    $post->comments()->save($comment);

注意，這裡我們不是以動態屬性來存取 `comments` 關聯，而是呼叫 `comments` 方法來取得關聯的實體。`save` 方法會自動將 `post_id` 的值新增到新的 `Comment` Model 上。

若要儲存多個關聯 Model，可使用 `saveMany` 方法：

    $post = Post::find(1);

    $post->comments()->saveMany([
        new Comment(['message' => 'A new comment.']),
        new Comment(['message' => 'Another new comment.']),
    ]);

`save` 與 `saveMany` 方法會持久化給定的 Model 實體，但不會將新持久化的 Model 新增到已載入到父項 Model 上的記憶體內關聯。若打算在使用 `save` 或 `saveMany` 方法後存取關聯，可以使用 `refresh` 方法來重新載入 Model 及其關聯：

    $post->comments()->save($comment);

    $post->refresh();

    // All comments, including the newly saved comment...
    $post->comments;


<a name="the-push-method"></a>
#### 遞迴式地儲存 Model 與關聯

若想 `save` 你的 Model 及其所有關聯，可使用 `push` 方法。在這個範例中，`Post` Model 會被儲存，同時也會儲存其留言與留言作者：

    $post = Post::find(1);

    $post->comments[0]->message = 'Message';
    $post->comments[0]->author->name = 'Author Name';

    $post->push();

`pushQuietly` 方法可用來儲存 Model 及其關聯，且不會引發任何事件：

    $post->pushQuietly();


<a name="the-create-method"></a>
### `create` 方法

除了 `save` 與 `saveMany` 方法外，也可以使用 `create` 方法，該方法可接受一組屬性陣列，並用來建立 Model、再將其新增到資料庫中。`save` 與 `create` 的不同之處在於，`save` 接受的是一個完整的 Eloquent Model 實體，而 `create` 接受的則是一個純 PHP `array`。`create` 方法會回傳新建的 Model：

    use App\Models\Post;

    $post = Post::find(1);

    $comment = $post->comments()->create([
        'message' => 'A new comment.',
    ]);

可使用 `createMany` 方法來建立多個關聯 Model：

    $post = Post::find(1);

    $post->comments()->createMany([
        ['message' => 'A new comment.'],
        ['message' => 'Another new comment.'],
    ]);

`createQuietly` 與 `createManyQuietly` 方法可用來建立 Model，且不會分派任何事件：

    $user = User::find(1);

    $user->posts()->createQuietly([
        'title' => 'Post title.',
    ]);

    $user->posts()->createManyQuietly([
        ['title' => 'First post.'],
        ['title' => 'Second post.'],
    ]);

也可以使用 `findOrNew`、`firstOrNew`、`firstOrCreate`、`updateOrCreate` 等方法來[在關聯上建立與更新 Model](/docs/{{version}}/eloquent#upserts)。

> [!NOTE]  
> 在使用 `create` 方法前，請務必先閱讀[批量賦值](/docs/{{version}}/eloquent#mass-assignment)的文件。


<a name="updating-belongs-to-relationships"></a>
### Belongs To 關聯

若想指派一個子 Model 給新的父項 Model，可使用 `associate` 方法。在這個範例中，`User` Model 定義了一個與 `Account` Model 的 `belongsTo` 關聯。`associate` 方法會設定子 Model 上的外鍵：

    use App\Models\Account;

    $account = Account::find(10);

    $user->account()->associate($account);

    $user->save();

若要從子 Model 上移除父項 Model，可使用 `dissociate` 方法。這個方法會將關聯的外鍵設為 `null`：

    $user->account()->dissociate();

    $user->save();


<a name="updating-many-to-many-relationships"></a>
### 多對多關聯


<a name="attaching-detaching"></a>
#### 附加／分離 (Attaching / Detaching)

Eloquent 也提供了一些方法來讓操作多對多關聯變得更方便。舉例來說，假設一個使用者可以有多個角色，而一個角色可以屬於多個使用者。我們可以使用 `attach` 方法，透過在關聯的中介資料表內新增一筆紀錄，來為使用者附加一個角色：

    use App\Models\User;

    $user = User::find(1);

    $user->roles()->attach($roleId);

在為 Model 附加關聯時，也可以傳入一組要新增到中介資料表內的額外資料陣列：

    $user->roles()->attach($roleId, ['expires' => $expires]);

有時候，可能需要從使用者身上移除某個角色。若要移除一筆多對多關聯的紀錄，請使用 `detach` 方法。`detach` 方法會從中介資料表刪除對應的紀錄；不過，兩個 Model 都會保留在資料庫內：

    // Detach a single role from the user...
    $user->roles()->detach($roleId);

    // Detach all roles from the user...
    $user->roles()->detach();

為了方便，`attach` 與 `detach` 也接受 ID 陣列作為輸入：

    $user = User::find(1);

    $user->roles()->detach([1, 2, 3]);

    $user->roles()->attach([
        1 => ['expires' => $expires],
        2 => ['expires' => $expires],
    ]);


<a name="syncing-associations"></a>
#### 同步關聯 (Syncing Associations)

也可以使用 `sync` 方法來建立多對多關聯。`sync` 方法可接受一組 ID 陣列來放置到中介資料表上。所有不在給定陣列內的 ID 都會從中介資料表內被移除。因此，在這個操作完成後，中介資料表內就只會存在給定陣列內的 ID：

    $user->roles()->sync([1, 2, 3]);

也可以跟 ID 一起傳入額外的中介資料表值：

    $user->roles()->sync([1 => ['expires' => true], 2, 3]);

若想為每個同步的 Model ID 都插入相同的中介資料表值，可使用 `syncWithPivotValues` 方法：

    $user->roles()->syncWithPivotValues([1, 2, 3], ['active' => true]);

若不想分離掉給定陣列中沒有的現有 ID，可使用 `syncWithoutDetaching` 方法：

    $user->roles()->syncWithoutDetaching([1, 2, 3]);


<a name="toggling-associations"></a>
#### 切換關聯 (Toggling Associations)

多對多關聯還提供了一個 `toggle` 方法，可用來「切換」給定關聯 Model ID 的附加狀態。若給定的 ID 目前為附加狀態，則會被分離。反之，若目前為分離狀態，則會被附加：

    $user->roles()->toggle([1, 2, 3]);

也可以跟 ID 一起傳入額外的中介資料表值：

    $user->roles()->toggle([
        1 => ['expires' => true],
        2 => ['expires' => true],
    ]);


<a name="updating-a-record-on-the-intermediate-table"></a>
#### 更新中介資料表上的紀錄

若要更新關聯中介資料表內現有的資料列，可使用 `updateExistingPivot` 方法。這個方法可接受中介紀錄的外鍵與一組要更新的屬性陣列：

    $user = User::find(1);

    $user->roles()->updateExistingPivot($roleId, [
        'active' => false,
    ]);

<a name="touching-parent-timestamps"></a>
## 接觸父項的時間戳記

當一個 Model 為另一個 Model 定義了 `belongsTo` 或 `belongsToMany` 關聯時 (例如，屬於某個 `Post` 的 `Comment`)，在更新子 Model 時，有時會需要一併更新父項的時間戳記。

舉例來說，當 `Comment` Model 更新時，我們可能會想自動「接觸 (Touch)」所屬 `Post` 的 `updated_at` 時間戳記，好讓該時間戳記被設為目前的日期與時間。要達成這個目的，只要在子 Model 上加上一個 `touches` 屬性即可。該屬性包含了一些關聯的名稱，這些關聯的 `updated_at` 時間戳記應在子 Model 更新時一併更新：

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

> [!WARNING]  
> 只有在使用 Eloquent 的 `save` 方法來更新子 Model 時，父項 Model 的時間戳記才會被更新。