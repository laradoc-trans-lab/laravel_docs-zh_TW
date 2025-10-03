# 資料庫：查詢產生器

- [簡介](#introduction)
- [執行資料庫查詢](#running-database-queries)
    - [分批處理結果](#chunking-results)
    - [惰性串流處理結果](#streaming-results-lazily)
    - [聚合](#aggregates)
- [Select 語句](#select-statements)
- [原始運算式](#raw-expressions)
- [聯結 (Joins)](#joins)
- [聯集 (Unions)](#unions)
- [基本 Where 子句](#basic-where-clauses)
    - [Where 子句](#where-clauses)
    - [Or Where 子句](#or-where-clauses)
    - [Where Not 子句](#where-not-clauses)
    - [Where Any / All / None 子句](#where-any-all-none-clauses)
    - [JSON Where 子句](#json-where-clauses)
    - [額外 Where 子句](#additional-where-clauses)
    - [邏輯分組](#logical-grouping)
- [進階 Where 子句](#advanced-where-clauses)
    - [Where Exists 子句](#where-exists-clauses)
    - [子查詢 Where 子句](#subquery-where-clauses)
    - [全文搜尋 Where 子句](#full-text-where-clauses)
- [排序、分組、限制與偏移](#ordering-grouping-limit-and-offset)
    - [排序](#ordering)
    - [分組](#grouping)
    - [限制與偏移](#limit-and-offset)
- [條件式子句](#conditional-clauses)
- [Insert 語句](#insert-statements)
    - [增量更新 (Upserts)](#upserts)
- [Update 語句](#update-statements)
    - [更新 JSON 欄位](#updating-json-columns)
    - [增量與減量](#increment-and-decrement)
- [Delete 語句](#delete-statements)
- [悲觀鎖](#pessimistic-locking)
- [可重複使用查詢組件](#reusable-query-components)
- [除錯](#debugging)

<a name="introduction"></a>
## 簡介

Laravel 的資料庫查詢產生器提供了一個方便、流暢的介面，用於建立和執行資料庫查詢。它可以用來執行應用程式中大部分的資料庫操作，並能完美地搭配所有 Laravel 支援的資料庫系統。

Laravel 查詢產生器使用 PDO 參數綁定來保護您的應用程式免受 SQL injection 攻擊。無需清理或淨化作為查詢綁定傳遞給查詢產生器的字串。

> [!WARNING]
> PDO 不支援綁定欄位名稱。因此，您絕不應該允許使用者輸入來決定您的查詢所參考的欄位名稱，包含「order by」欄位。

<a name="running-database-queries"></a>
## 執行資料庫查詢


<a name="retrieving-all-rows-from-a-table"></a>
#### 從資料表中取得所有列

您可以使用 `DB` Facade 提供的 `table` 方法來啟動查詢。`table` 方法會針對指定的資料表回傳一個 Fluent 查詢產生器實例，讓您可以將更多約束鏈接到查詢上，然後最終使用 `get` 方法取得查詢結果：

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Support\Facades\DB;
use Illuminate\View\View;

class UserController extends Controller
{
    /**
     * Show a list of all of the application's users.
     */
    public function index(): View
    {
        $users = DB::table('users')->get();

        return view('user.index', ['users' => $users]);
    }
}
```

`get` 方法會回傳一個 `Illuminate\Support\Collection` 實例，其中包含查詢結果，每個結果都是 PHP `stdClass` 物件的實例。您可以透過存取物件的屬性來存取每個欄位的值：

```php
use Illuminate\Support\Facades\DB;

$users = DB::table('users')->get();

foreach ($users as $user) {
    echo $user->name;
}
```

> [!NOTE]
> Laravel Collection 提供了各種極其強大的方法來映射與精簡資料。有關 Laravel Collection 的更多資訊，請查閱 [Collection 文件](/docs/{{version}}/collections)。


<a name="retrieving-a-single-row-column-from-a-table"></a>
#### 從資料表中取得單一列 / 欄位

如果您只需要從資料庫表中取得單一列，可以使用 `DB` Facade 的 `first` 方法。此方法將會回傳一個單一的 `stdClass` 物件：

```php
$user = DB::table('users')->where('name', 'John')->first();

return $user->email;
```

如果您想從資料庫表中取得單一列，但如果找不到匹配的列就拋出 `Illuminate\Database\RecordNotFoundException`，可以使用 `firstOrFail` 方法。如果 `RecordNotFoundException` 未被捕獲，則會自動回傳 404 HTTP 回應給客戶端：

```php
$user = DB::table('users')->where('name', 'John')->firstOrFail();
```

如果您不需要整個列，可以使用 `value` 方法從記錄中提取單一值。此方法將直接回傳欄位的值：

```php
$email = DB::table('users')->where('name', 'John')->value('email');
```

若要依據 `id` 欄位的值取得單一列，請使用 `find` 方法：

```php
$user = DB::table('users')->find(3);
```


<a name="retrieving-a-list-of-column-values"></a>
#### 取得欄位值的列表

如果您想取得一個包含單一欄位值的 `Illuminate\Support\Collection` 實例，可以使用 `pluck` 方法。在這個範例中，我們將取得一個使用者稱謂的 Collection：

```php
use Illuminate\Support\Facades\DB;

$titles = DB::table('users')->pluck('title');

foreach ($titles as $title) {
    echo $title;
}
```

您可以透過向 `pluck` 方法提供第二個參數，來指定結果 Collection 應該使用哪個欄位作為其鍵：

```php
$titles = DB::table('users')->pluck('title', 'name');

foreach ($titles as $name => $title) {
    echo $title;
}
```


<a name="chunking-results"></a>
### 分批處理結果

如果您需要處理數千條資料庫記錄，請考慮使用 `DB` Facade 提供的 `chunk` 方法。此方法一次取得一小批結果，並將每批結果饋送到閉包中進行處理。例如，讓我們以一次 100 條記錄的分批方式取得整個 `users` 資料表：

```php
use Illuminate\Support\Collection;
use Illuminate\Support\Facades\DB;

DB::table('users')->orderBy('id')->chunk(100, function (Collection $users) {
    foreach ($users as $user) {
        // ...
    }
});
```

您可以從閉包中回傳 `false` 來停止進一步的分批處理：

```php
DB::table('users')->orderBy('id')->chunk(100, function (Collection $users) {
    // Process the records...

    return false;
});
```

如果您在分批處理結果時更新資料庫記錄，您的分批結果可能會以意想不到的方式改變。如果您打算在分批處理時更新取得的記錄，最好改用 `chunkById` 方法。此方法將根據記錄的主鍵自動分頁結果：

```php
DB::table('users')->where('active', false)
    ->chunkById(100, function (Collection $users) {
        foreach ($users as $user) {
            DB::table('users')
                ->where('id', $user->id)
                ->update(['active' => true]);
        }
    });
```

由於 `chunkById` 和 `lazyById` 方法會將自己的「where」條件添加到正在執行的查詢中，您通常應該在閉包中 [邏輯分組](#logical-grouping) 您的條件：

```php
DB::table('users')->where(function ($query) {
    $query->where('credits', 1)->orWhere('credits', 2);
})->chunkById(100, function (Collection $users) {
    foreach ($users as $user) {
        DB::table('users')
            ->where('id', $user->id)
            ->update(['credits' => 3]);
    }
});
```

> [!WARNING]
> 在 chunk 回呼中更新或刪除記錄時，主鍵或外部鍵的任何變更都可能會影響 chunk 查詢。這可能會導致記錄不包含在分批結果中。


<a name="streaming-results-lazily"></a>
### 惰性串流處理結果

`lazy` 方法的功能類似於 [分批處理方法](#chunking-results)，都是以分批的方式執行查詢。然而，`lazy()` 方法不是將每個分批傳遞給回呼，而是回傳一個 [LazyCollection](/docs/{{version}}/collections#lazy-collections)，讓您可以將結果作為單一串流進行互動：

```php
use Illuminate\Support\Facades\DB;

DB::table('users')->orderBy('id')->lazy()->each(function (object $user) {
    // ...
});
```

再次強調，如果您打算在迭代記錄時更新已取得的記錄，最好改用 `lazyById` 或 `lazyByIdDesc` 方法。這些方法會根據記錄的主鍵自動分頁結果：

```php
DB::table('users')->where('active', false)
    ->lazyById()->each(function (object $user) {
        DB::table('users')
            ->where('id', $user->id)
            ->update(['active' => true]);
    });
```

> [!WARNING]
> 在迭代記錄時更新或刪除記錄時，主鍵或外部鍵的任何變更都可能會影響 chunk 查詢。這可能會導致記錄不包含在結果中。


<a name="aggregates"></a>
### 聚合

查詢產生器還提供了各種方法來取得聚合值，例如 `count`、`max`、`min`、`avg` 和 `sum`。您可以在建構查詢後呼叫這些方法中的任何一個：

```php
use Illuminate\Support\Facades\DB;

$users = DB::table('users')->count();

$price = DB::table('orders')->max('price');
```

當然，您可以將這些方法與其他子句結合使用，以微調聚合值的計算方式：

```php
$price = DB::table('orders')
    ->where('finalized', 1)
    ->avg('price');
```


<a name="determining-if-records-exist"></a>
#### 判斷記錄是否存在

您可以使用 `exists` 和 `doesntExist` 方法來判斷是否存在任何符合查詢約束的記錄，而不是使用 `count` 方法：

```php
if (DB::table('orders')->where('finalized', 1)->exists()) {
    // ...
}

if (DB::table('orders')->where('finalized', 1)->doesntExist()) {
    // ...
}
```

<a name="select-statements"></a>
## Select 語句

<a name="specifying-a-select-clause"></a>
#### 指定 Select 子句

您可能不總是希望從資料庫資料表中選取所有欄位。透過使用 `select` 方法，您可以為查詢指定一個自訂的「select」子句：

```php
use Illuminate\Support\Facades\DB;

$users = DB::table('users')
    ->select('name', 'email as user_email')
    ->get();
```

`distinct` 方法允許您強制查詢回傳不同的結果：

```php
$users = DB::table('users')->distinct()->get();
```

如果您已經有一個查詢產生器實例，並且希望將欄位新增到其現有的 select 子句中，您可以使用 `addSelect` 方法：

```php
$query = DB::table('users')->select('name');

$users = $query->addSelect('age')->get();
```

<a name="raw-expressions"></a>
## 原始運算式

有時您可能需要將任意字串插入到查詢中。要建立原始字串運算式，您可以使用 `DB` Facade 提供的 `raw` 方法：

```php
$users = DB::table('users')
    ->select(DB::raw('count(*) as user_count, status'))
    ->where('status', '<>', 1)
    ->groupBy('status')
    ->get();
```

> [!WARNING]
> 原始語句將以字串形式注入查詢，因此您應極度小心，避免產生 SQL 注入弱點。

<a name="raw-methods"></a>
### 原始方法

除了使用 `DB::raw` 方法外，您也可以使用以下方法將原始運算式插入查詢的不同部分。**請記住，Laravel 無法保證任何使用原始運算式的查詢都能免受 SQL 注入弱點的影響。**

<a name="selectraw"></a>
#### `selectRaw`

`selectRaw` 方法可以用來取代 `addSelect(DB::raw(/* ... */))`。此方法接受一個選用的綁定陣列作為其第二個參數：

```php
$orders = DB::table('orders')
    ->selectRaw('price * ? as price_with_tax', [1.0825])
    ->get();
```

<a name="whereraw-orwhereraw"></a>
#### `whereRaw / orWhereRaw`

`whereRaw` 和 `orWhereRaw` 方法可以用來將原始的「where」子句注入您的查詢。這些方法接受一個選用的綁定陣列作為其第二個參數：

```php
$orders = DB::table('orders')
    ->whereRaw('price > IF(state = "TX", ?, 100)', [200])
    ->get();
```

<a name="havingraw-orhavingraw"></a>
#### `havingRaw / orHavingRaw`

`havingRaw` 和 `orHavingRaw` 方法可用來提供一個原始字串作為「having」子句的值。這些方法接受一個選用的綁定陣列作為其第二個參數：

```php
$orders = DB::table('orders')
    ->select('department', DB::raw('SUM(price) as total_sales'))
    ->groupBy('department')
    ->havingRaw('SUM(price) > ?', [2500])
    ->get();
```

<a name="orderbyraw"></a>
#### `orderByRaw`

`orderByRaw` 方法可用來提供一個原始字串作為「order by」子句的值：

```php
$orders = DB::table('orders')
    ->orderByRaw('updated_at - created_at DESC')
    ->get();
```

<a name="groupbyraw"></a>
### `groupByRaw`

`groupByRaw` 方法可用來提供一個原始字串作為 `group by` 子句的值：

```php
$orders = DB::table('orders')
    ->select('city', 'state')
    ->groupByRaw('city, state')
    ->get();
```

<a name="joins"></a>
## 聯結 (Joins)

<a name="inner-join-clause"></a>
#### Inner Join 子句

查詢產生器也可用來為您的查詢新增 join 子句。要執行基本的「inner join」，您可以在查詢產生器實例上使用 `join` 方法。`join` 方法的第一個參數是您需要 join 的資料表名稱，而其餘參數則指定 join 的欄位約束條件。您甚至可以在單一查詢中 join 多個資料表：

```php
use Illuminate\Support\Facades\DB;

$users = DB::table('users')
    ->join('contacts', 'users.id', '=', 'contacts.user_id')
    ->join('orders', 'users.id', '=', 'orders.user_id')
    ->select('users.*', 'contacts.phone', 'orders.price')
    ->get();
```

<a name="left-join-right-join-clause"></a>
#### Left Join / Right Join 子句

如果您想執行「left join」或「right join」而不是「inner join」，請使用 `leftJoin` 或 `rightJoin` 方法。這些方法與 `join` 方法具有相同的簽名：

```php
$users = DB::table('users')
    ->leftJoin('posts', 'users.id', '=', 'posts.user_id')
    ->get();

$users = DB::table('users')
    ->rightJoin('posts', 'users.id', '=', 'posts.user_id')
    ->get();
```

<a name="cross-join-clause"></a>
#### Cross Join 子句

您可以使用 `crossJoin` 方法執行「cross join」。Cross join 會在第一個資料表與 join 的資料表之間產生一個笛卡兒積：

```php
$sizes = DB::table('sizes')
    ->crossJoin('colors')
    ->get();
```

<a name="advanced-join-clauses"></a>
#### 進階 Join 子句

您也可以指定更進階的 join 子句。首先，將一個閉包作為第二個參數傳遞給 `join` 方法。該閉包將收到一個 `Illuminate\Database\Query\JoinClause` 實例，它允許您指定「join」子句上的約束條件：

```php
DB::table('users')
    ->join('contacts', function (JoinClause $join) {
        $join->on('users.id', '=', 'contacts.user_id')->orOn(/* ... */);
    })
    ->get();
```

如果您想在您的 join 上使用「where」子句，您可以使用 `JoinClause` 實例提供的 `where` 和 `orWhere` 方法。這些方法不是比較兩個欄位，而是將欄位與一個值進行比較：

```php
DB::table('users')
    ->join('contacts', function (JoinClause $join) {
        $join->on('users.id', '=', 'contacts.user_id')
            ->where('contacts.user_id', '>', 5);
    })
    ->get();
```

<a name="subquery-joins"></a>
#### 子查詢 Join

您可以使用 `joinSub`、`leftJoinSub` 和 `rightJoinSub` 方法將查詢 join 到子查詢。這些方法中的每一個都接受三個參數：子查詢、其資料表別名，以及定義相關欄位的閉包。在此範例中，我們將檢索一個使用者集合，其中每個使用者記錄也包含該使用者最新發布的部落格文章的 `created_at` 時間戳記：

```php
$latestPosts = DB::table('posts')
    ->select('user_id', DB::raw('MAX(created_at) as last_post_created_at'))
    ->where('is_published', true)
    ->groupBy('user_id');

$users = DB::table('users')
    ->joinSub($latestPosts, 'latest_posts', function (JoinClause $join) {
        $join->on('users.id', '=', 'latest_posts.user_id');
    })->get();
```

<a name="lateral-joins"></a>
#### Lateral Join

> [!WARNING]
> Lateral join 目前受到 PostgreSQL、MySQL >= 8.0.14 和 SQL Server 的支援。

您可以使用 `joinLateral` 和 `leftJoinLateral` 方法來執行帶有子查詢的「lateral join」。這些方法中的每一個都接受兩個參數：子查詢及其資料表別名。join 條件應在給定子查詢的 `where` 子句中指定。Lateral join 會對每一列進行評估，並且可以引用子查詢外部的欄位。

在此範例中，我們將檢索一個使用者集合以及該使用者最新的三篇部落格文章。每個使用者在結果集中最多可以產生三列：每列代表其一篇最新的部落格文章。join 條件在子查詢中使用 `whereColumn` 子句指定，並參考當前使用者列：

```php
$latestPosts = DB::table('posts')
    ->select('id as post_id', 'title as post_title', 'created_at as post_created_at')
    ->whereColumn('user_id', 'users.id')
    ->orderBy('created_at', 'desc')
    ->limit(3);

$users = DB::table('users')
    ->joinLateral($latestPosts, 'latest_posts')
    ->get();
```

<a name="unions"></a>
## 聯集 (Unions)

查詢產生器也提供了一個便利的方法，能將兩個或更多查詢進行「聯集」。例如，您可以建立一個初始查詢，並使用 `union` 方法將其與更多查詢進行聯集：

```php
use Illuminate\Support\Facades\DB;

$first = DB::table('users')
    ->whereNull('first_name');

$users = DB::table('users')
    ->whereNull('last_name')
    ->union($first)
    ->get();
```

除了 `union` 方法之外，查詢產生器還提供了一個 `unionAll` 方法。使用 `unionAll` 方法組合的查詢將不會移除其重複的結果。`unionAll` 方法具有與 `union` 方法相同的方法簽章。

<a name="basic-where-clauses"></a>
## 基本 Where 子句


<a name="where-clauses"></a>
### Where 子句

您可以使用查詢產生器的 `where` 方法為查詢新增「where」子句。對 `where` 方法最基本的呼叫需要三個參數。第一個參數是欄位的名稱。第二個參數是一個運算子，可以是資料庫支援的任何運算子。第三個參數是要與欄位值比較的數值。

例如，以下查詢會擷取 `votes` 欄位的值等於 `100` 且 `age` 欄位的值大於 `35` 的使用者：

```php
$users = DB::table('users')
    ->where('votes', '=', 100)
    ->where('age', '>', 35)
    ->get();
```

為了方便，如果您想驗證某個欄位是否 `=` 某個指定值，您可以將該值作為第二個參數傳遞給 `where` 方法。Laravel 會假設您想使用 `=` 運算子：

```php
$users = DB::table('users')->where('votes', 100)->get();
```

您也可以提供一個關聯陣列給 `where` 方法，以便快速查詢多個欄位：

```php
$users = DB::table('users')->where([
    'first_name' => 'Jane',
    'last_name' => 'Doe',
])->get();
```

如前所述，您可以使用資料庫系統支援的任何運算子：

```php
$users = DB::table('users')
    ->where('votes', '>=', 100)
    ->get();

$users = DB::table('users')
    ->where('votes', '<>', 100)
    ->get();

$users = DB::table('users')
    ->where('name', 'like', 'T%')
    ->get();
```

您也可以傳遞一個條件陣列給 `where` 函數。陣列中的每個元素都應是一個包含通常傳遞給 `where` 方法的三個參數的陣列：

```php
$users = DB::table('users')->where([
    ['status', '=', '1'],
    ['subscribed', '<>', '1'],
])->get();
```

> [!WARNING]
> PDO does not support binding column names. Therefore, you should never allow user input to dictate the column names referenced by your queries, including "order by" columns.

> [!WARNING]
> MySQL and MariaDB automatically typecast strings to integers in string-number comparisons. In this process, non-numeric strings are converted to `0`, which can lead to unexpected results. For example, if your table has a `secret` column with a value of `aaa` and you run `User::where('secret', 0)`, that row will be returned. To avoid this, ensure all values are typecast to their appropriate types before using them in queries.


<a name="or-where-clauses"></a>
### Or Where 子句

當將查詢產生器的 `where` 方法呼叫鏈結在一起時，「where」子句將使用 `and` 運算子結合。但是，您可以使用 `orWhere` 方法將子句使用 `or` 運算子加入查詢。`orWhere` 方法接受與 `where` 方法相同的參數：

```php
$users = DB::table('users')
    ->where('votes', '>', 100)
    ->orWhere('name', 'John')
    ->get();
```

如果您需要將「or」條件在括號內分組，您可以將閉包作為第一個參數傳遞給 `orWhere` 方法：

```php
use Illuminate\Database\Query\Builder; 

$users = DB::table('users')
    ->where('votes', '>', 100)
    ->orWhere(function (Builder $query) {
        $query->where('name', 'Abigail')
            ->where('votes', '>', 50);
        })
    ->get();
```

上述範例將產生以下 SQL：

```sql
select * from users where votes > 100 or (name = 'Abigail' and votes > 50)
```

> [!WARNING]
> You should always group `orWhere` calls in order to avoid unexpected behavior when global scopes are applied.


<a name="where-not-clauses"></a>
### Where Not 子句

`whereNot` 和 `orWhereNot` 方法可用於否定一組給定的查詢限制。例如，以下查詢會排除正在促銷或價格低於十元的產品：

```php
$products = DB::table('products')
    ->whereNot(function (Builder $query) {
        $query->where('clearance', true)
            ->orWhere('price', '<', 10);
        })
    ->get();
```


<a name="where-any-all-none-clauses"></a>
### Where Any / All / None 子句

有時您可能需要將相同的查詢限制應用於多個欄位。例如，您可能希望擷取所有記錄，其中指定列表中的任何欄位 `LIKE` 指定值。您可以使用 `whereAny` 方法來達成此目的：

```php
$users = DB::table('users')
    ->where('active', true)
    ->whereAny([
        'name',
        'email',
        'phone',
    ], 'like', 'Example%')
    ->get();
```

上述查詢將產生以下 SQL：

```sql
SELECT *
FROM users
WHERE active = true AND (
    name LIKE 'Example%' OR
    email LIKE 'Example%' OR
    phone LIKE 'Example%'
)
```

同樣地，`whereAll` 方法可用於擷取所有指定欄位都符合給定限制的記錄：

```php
$posts = DB::table('posts')
    ->where('published', true)
    ->whereAll([
        'title',
        'content',
    ], 'like', '%Laravel%')
    ->get();
```

上述查詢將產生以下 SQL：

```sql
SELECT *
FROM posts
WHERE published = true AND (
    title LIKE '%Laravel%' AND
    content LIKE '%Laravel%'
)
```

`whereNone` 方法可用於擷取所有指定欄位都不符合給定限制的記錄：

```php
$posts = DB::table('albums')
    ->where('published', true)
    ->whereNone([
        'title',
        'lyrics',
        'tags',
    ], 'like', '%explicit%')
    ->get();
```

上述查詢將產生以下 SQL：

```sql
SELECT *
FROM albums
WHERE published = true AND NOT (
    title LIKE '%explicit%' OR
    lyrics LIKE '%explicit%' OR
    tags LIKE '%explicit%'
)
```


<a name="json-where-clauses"></a>
### JSON Where 子句

Laravel 也支援在提供 JSON 欄位類型支援的資料庫上查詢 JSON 欄位類型。目前，這包括 MariaDB 10.3+、MySQL 8.0+、PostgreSQL 12.0+、SQL Server 2017+ 和 SQLite 3.39.0+。若要查詢 JSON 欄位，請使用 `->` 運算子：

```php
$users = DB::table('users')
    ->where('preferences->dining->meal', 'salad')
    ->get();

$users = DB::table('users')
    ->whereIn('preferences->dining->meal', ['pasta', 'salad', 'sandwiches'])
    ->get();
```

您可以使用 `whereJsonContains` 和 `whereJsonDoesntContain` 方法來查詢 JSON 陣列：

```php
$users = DB::table('users')
    ->whereJsonContains('options->languages', 'en')
    ->get();

$users = DB::table('users')
    ->whereJsonDoesntContain('options->languages', 'en')
    ->get();
```

如果您的應用程式使用 MariaDB、MySQL 或 PostgreSQL 資料庫，您可以將值陣列傳遞給 `whereJsonContains` 和 `whereJsonDoesntContain` 方法：

```php
$users = DB::table('users')
    ->whereJsonContains('options->languages', ['en', 'de'])
    ->get();

$users = DB::table('users')
    ->whereJsonDoesntContain('options->languages', ['en', 'de'])
    ->get();
```

此外，您可以使用 `whereJsonContainsKey` 或 `whereJsonDoesntContainKey` 方法來擷取包含或不包含 JSON 鍵的結果：

```php
$users = DB::table('users')
    ->whereJsonContainsKey('preferences->dietary_requirements')
    ->get();

$users = DB::table('users')
    ->whereJsonDoesntContainKey('preferences->dietary_requirements')
    ->get();
```

最後，您可以使用 `whereJsonLength` 方法根據 JSON 陣列的長度進行查詢：

```php
$users = DB::table('users')
    ->whereJsonLength('options->languages', 0)
    ->get();

$users = DB::table('users')
    ->whereJsonLength('options->languages', '>', 1)
    ->get();
```

<a name="additional-where-clauses"></a>
### 額外 Where 子句

**whereLike / orWhereLike / whereNotLike / orWhereNotLike**

`whereLike` 方法允許您在查詢中加入「LIKE」子句，用於模式比對。這些方法提供了一種與資料庫無關的方式來執行字串比對查詢，並能夠切換區分大小寫。預設情況下，字串比對是不區分大小寫的：

```php
$users = DB::table('users')
    ->whereLike('name', '%John%')
    ->get();
```

您可以透過 `caseSensitive` 引數啟用區分大小寫的搜尋：

```php
$users = DB::table('users')
    ->whereLike('name', '%John%', caseSensitive: true)
    ->get();
```

`orWhereLike` 方法允許您加入一個帶有 LIKE 條件的「或」子句：

```php
$users = DB::table('users')
    ->where('votes', '>', 100)
    ->orWhereLike('name', '%John%')
    ->get();
```

`whereNotLike` 方法允許您在查詢中加入「NOT LIKE」子句：

```php
$users = DB::table('users')
    ->whereNotLike('name', '%John%')
    ->get();
```

同樣地，您可以使用 `orWhereNotLike` 加入一個帶有 NOT LIKE 條件的「或」子句：

```php
$users = DB::table('users')
    ->where('votes', '>', 100)
    ->orWhereNotLike('name', '%John%')
    ->get();
```

> [!WARNING]
> `whereLike` 的區分大小寫搜尋選項目前不支援 SQL Server。

**whereIn / whereNotIn / orWhereIn / orWhereNotIn**

`whereIn` 方法驗證指定欄位的值是否包含在給定陣列中：

```php
$users = DB::table('users')
    ->whereIn('id', [1, 2, 3])
    ->get();
```

`whereNotIn` 方法驗證指定欄位的值是否不包含在給定陣列中：

```php
$users = DB::table('users')
    ->whereNotIn('id', [1, 2, 3])
    ->get();
```

您也可以將查詢物件作為 `whereIn` 方法的第二個引數提供：

```php
$activeUsers = DB::table('users')->select('id')->where('is_active', 1);

$users = DB::table('comments')
    ->whereIn('user_id', $activeUsers)
    ->get();
```

上述範例將產生以下 SQL：

```sql
select * from comments where user_id in (
    select id
    from users
    where is_active = 1
)
```

> [!WARNING]
> 如果您正在為查詢加入一個大型的整數繫結陣列，可以使用 `whereIntegerInRaw` 或 `whereIntegerNotInRaw` 方法來大幅減少記憶體使用量。

**whereBetween / orWhereBetween**

`whereBetween` 方法驗證欄位的值介於兩個值之間：

```php
$users = DB::table('users')
    ->whereBetween('votes', [1, 100])
    ->get();
```

**whereNotBetween / orWhereNotBetween**

`whereNotBetween` 方法驗證欄位的值位於兩個值之外：

```php
$users = DB::table('users')
    ->whereNotBetween('votes', [1, 100])
    ->get();
```

**whereBetweenColumns / whereNotBetweenColumns / orWhereBetweenColumns / orWhereNotBetweenColumns**

`whereBetweenColumns` 方法驗證欄位的值介於同一個資料表列中兩個欄位的兩個值之間：

```php
$patients = DB::table('patients')
    ->whereBetweenColumns('weight', ['minimum_allowed_weight', 'maximum_allowed_weight'])
    ->get();
```

`whereNotBetweenColumns` 方法驗證欄位的值位於同一個資料表列中兩個欄位的兩個值之外：

```php
$patients = DB::table('patients')
    ->whereNotBetweenColumns('weight', ['minimum_allowed_weight', 'maximum_allowed_weight'])
    ->get();
```

**whereValueBetween / whereValueNotBetween / orWhereValueBetween / orWhereValueNotBetween**

`whereValueBetween` 方法驗證給定值介於同一個資料表列中兩個相同型別欄位的值之間：

```php
$patients = DB::table('products')
    ->whereValueBetween(100, ['min_price', 'max_price'])
    ->get();
```

`whereValueNotBetween` 方法驗證給定值位於同一個資料表列中兩個欄位的值之外：

```php
$patients = DB::table('products')
    ->whereValueNotBetween(100, ['min_price', 'max_price'])
    ->get();
```

**whereNull / whereNotNull / orWhereNull / orWhereNotNull**

`whereNull` 方法驗證指定欄位的值為 `NULL`：

```php
$users = DB::table('users')
    ->whereNull('updated_at')
    ->get();
```

`whereNotNull` 方法驗證欄位的值不是 `NULL`：

```php
$users = DB::table('users')
    ->whereNotNull('updated_at')
    ->get();
```

**whereDate / whereMonth / whereDay / whereYear / whereTime**

`whereDate` 方法可用於將欄位的值與日期進行比對：

```php
$users = DB::table('users')
    ->whereDate('created_at', '2016-12-31')
    ->get();
```

`whereMonth` 方法可用於將欄位的值與指定月份進行比對：

```php
$users = DB::table('users')
    ->whereMonth('created_at', '12')
    ->get();
```

`whereDay` 方法可用於將欄位的值與指定日期進行比對：

```php
$users = DB::table('users')
    ->whereDay('created_at', '31')
    ->get();
```

`whereYear` 方法可用於將欄位的值與指定年份進行比對：

```php
$users = DB::table('users')
    ->whereYear('created_at', '2016')
    ->get();
```

`whereTime` 方法可用於將欄位的值與指定時間進行比對：

```php
$users = DB::table('users')
    ->whereTime('created_at', '=', '11:20:45')
    ->get();
```

**wherePast / whereFuture / whereToday / whereBeforeToday / whereAfterToday**

`wherePast` 和 `whereFuture` 方法可用於判斷欄位的值是在過去還是未來：

```php
$invoices = DB::table('invoices')
    ->wherePast('due_at')
    ->get();

$invoices = DB::table('invoices')
    ->whereFuture('due_at')
    ->get();
```

`whereNowOrPast` 和 `whereNowOrFuture` 方法可用於判斷欄位的值是在過去還是未來，包含當前的日期和時間：

```php
$invoices = DB::table('invoices')
    ->whereNowOrPast('due_at')
    ->get();

$invoices = DB::table('invoices')
    ->whereNowOrFuture('due_at')
    ->get();
```

`whereToday`、`whereBeforeToday` 和 `whereAfterToday` 方法可用於判斷欄位的值是否分別是今天、今天之前或今天之後：

```php
$invoices = DB::table('invoices')
    ->whereToday('due_at')
    ->get();

$invoices = DB::table('invoices')
    ->whereBeforeToday('due_at')
    ->get();

$invoices = DB::table('invoices')
    ->whereAfterToday('due_at')
    ->get();
```

同樣地，`whereTodayOrBefore` 和 `whereTodayOrAfter` 方法可用於判斷欄位的值是否在今天之前或今天之後，包含今天日期：

```php
$invoices = DB::table('invoices')
    ->whereTodayOrBefore('due_at')
    ->get();

$invoices = DB::table('invoices')
    ->whereTodayOrAfter('due_at')
    ->get();
```

**whereColumn / orWhereColumn**

`whereColumn` 方法可用於驗證兩個欄位是否相等：

```php
$users = DB::table('users')
    ->whereColumn('first_name', 'last_name')
    ->get();
```

您也可以將比較運算子傳遞給 `whereColumn` 方法：

```php
$users = DB::table('users')
    ->whereColumn('updated_at', '>', 'created_at')
    ->get();
```

您也可以將欄位比較陣列傳遞給 `whereColumn` 方法。這些條件將使用 `and` 運算子聯結：

```php
$users = DB::table('users')
    ->whereColumn([
        ['first_name', '=', 'last_name'],
        ['updated_at', '>', 'created_at'],
    ])->get();
```

<a name="logical-grouping"></a>
### 邏輯分組

有時候你可能需要將多個「Where 子句」在括號內分組，以實現你查詢所期望的邏輯分組。事實上，你通常應該始終將 `orWhere` 方法的呼叫放在括號內，以避免不可預期的查詢行為。為此，你可以將一個閉包傳遞給 `where` 方法：

```php
$users = DB::table('users')
    ->where('name', '=', 'John')
    ->where(function (Builder $query) {
        $query->where('votes', '>', 100)
            ->orWhere('title', '=', 'Admin');
    })
    ->get();
```

如你所見，將閉包傳遞給 `where` 方法指示查詢產生器開始一個約束群組。該閉包將會接收一個查詢產生器實例，你可以使用它來設定將包含在括號群組內的約束條件。上述範例將產生以下 SQL：

```sql
select * from users where name = 'John' and (votes > 100 or title = 'Admin')
```

> [!WARNING]
> 你應該始終將 `orWhere` 呼叫進行分組，以避免在應用全域 Scope (global scopes) 時產生不可預期的行為。

<a name="advanced-where-clauses"></a>
## 進階 Where 子句


<a name="where-exists-clauses"></a>
### Where Exists 子句

`whereExists` 方法允許您編寫「where exists」SQL 子句。`whereExists` 方法接受一個閉包，該閉包將收到一個查詢產生器實例，讓您定義應放入「exists」子句中的查詢：

```php
$users = DB::table('users')
    ->whereExists(function (Builder $query) {
        $query->select(DB::raw(1))
            ->from('orders')
            ->whereColumn('orders.user_id', 'users.id');
    })
    ->get();
```

或者，您可以向 `whereExists` 方法提供一個查詢物件而不是閉包：

```php
$orders = DB::table('orders')
    ->select(DB::raw(1))
    ->whereColumn('orders.user_id', 'users.id');

$users = DB::table('users')
    ->whereExists($orders)
    ->get();
```

以上兩個範例都將產生以下 SQL：

```sql
select * from users
where exists (
    select 1
    from orders
    where orders.user_id = users.id
)
```


<a name="subquery-where-clauses"></a>
### 子查詢 Where 子句

有時您可能需要建構一個「where」子句，用於將子查詢的結果與給定值進行比較。您可以透過向 `where` 方法傳遞閉包和一個值來實現。例如，以下查詢將檢索所有最近有特定類型「會員資格」的使用者；

```php
use App\Models\User;
use Illuminate\Database\Query\Builder;

$users = User::where(function (Builder $query) {
    $query->select('type')
        ->from('membership')
        ->whereColumn('membership.user_id', 'users.id')
        ->orderByDesc('membership.start_date')
        ->limit(1);
}, 'Pro')->get();
```

或者，您可能需要建構一個「where」子句，用於將欄位與子查詢結果進行比較。您可以透過向 `where` 方法傳遞欄位、運算子和閉包來實現。例如，以下查詢將檢索所有金額小於平均值的所有收入記錄；

```php
use App\Models\Income;
use Illuminate\Database\Query\Builder;

$incomes = Income::where('amount', '<', function (Builder $query) {
    $query->selectRaw('avg(i.amount)')->from('incomes as i');
})->get();
```


<a name="full-text-where-clauses"></a>
### 全文搜尋 Where 子句

> [!WARNING]
> 目前 MariaDB、MySQL 和 PostgreSQL 支援全文搜尋 where 子句。

`whereFullText` 和 `orWhereFullText` 方法可用於為具有[全文索引](/docs/{{version}}/migrations#available-index-types)的欄位新增全文「where」子句。這些方法將由 Laravel 轉換為底層資料庫系統的適當 SQL。例如，對於使用 MariaDB 或 MySQL 的應用程式，將會產生 `MATCH AGAINST` 子句：

```php
$users = DB::table('users')
    ->whereFullText('bio', 'web developer')
    ->get();
```


<a name="ordering-grouping-limit-and-offset"></a>
## 排序、分組、限制與偏移


<a name="ordering"></a>
### 排序


<a name="orderby"></a>
#### `orderBy` 方法

`orderBy` 方法允許您根據給定欄位對查詢結果進行排序。`orderBy` 方法接受的第一個參數應為您希望排序的欄位，而第二個參數則決定排序方向，可以是 `asc` 或 `desc`：

```php
$users = DB::table('users')
    ->orderBy('name', 'desc')
    ->get();
```

若要依多個欄位排序，您可以簡單地多次呼叫 `orderBy`：

```php
$users = DB::table('users')
    ->orderBy('name', 'desc')
    ->orderBy('email', 'asc')
    ->get();
```

排序方向是可選的，預設為升序。若您想以降序排序，可以為 `orderBy` 方法指定第二個參數，或直接使用 `orderByDesc`：

```php
$users = DB::table('users')
    ->orderByDesc('verified_at')
    ->get();
```

最後，使用 `->` 運算子，結果可以根據 JSON 欄位中的值進行排序：

```php
$corporations = DB::table('corporations')
    ->where('country', 'US')
    ->orderBy('location->state')
    ->get();
```


<a name="latest-oldest"></a>
#### `latest` 與 `oldest` 方法

`latest` 與 `oldest` 方法讓您可以輕鬆地按日期排序結果。預設情況下，結果將按資料表的 `created_at` 欄位排序。或者，您可以傳遞您希望排序的欄位名稱：

```php
$user = DB::table('users')
    ->latest()
    ->first();
```


<a name="random-ordering"></a>
#### 隨機排序

`inRandomOrder` 方法可用於隨機排序查詢結果。例如，您可以使用此方法來取得一個隨機使用者：

```php
$randomUser = DB::table('users')
    ->inRandomOrder()
    ->first();
```


<a name="removing-existing-orderings"></a>
#### 移除現有排序

`reorder` 方法會移除先前已套用至查詢的所有「order by」子句：

```php
$query = DB::table('users')->orderBy('name');

$unorderedUsers = $query->reorder()->get();
```

呼叫 `reorder` 方法時，您可以傳遞欄位和方向，以移除所有現有的「order by」子句，並為查詢套用全新的排序：

```php
$query = DB::table('users')->orderBy('name');

$usersOrderedByEmail = $query->reorder('email', 'desc')->get();
```

為了方便起見，您可以使用 `reorderDesc` 方法以降序重新排序查詢結果：

```php
$query = DB::table('users')->orderBy('name');

$usersOrderedByEmail = $query->reorderDesc('email')->get();
```


<a name="grouping"></a>
### 分組


<a name="groupby-having"></a>
#### `groupBy` 與 `having` 方法

正如您所預期的，`groupBy` 和 `having` 方法可用於對查詢結果進行分組。`having` 方法的簽章與 `where` 方法類似：

```php
$users = DB::table('users')
    ->groupBy('account_id')
    ->having('account_id', '>', 100)
    ->get();
```

您可以使用 `havingBetween` 方法來篩選指定範圍內的結果：

```php
$report = DB::table('orders')
    ->selectRaw('count(id) as number_of_orders, customer_id')
    ->groupBy('customer_id')
    ->havingBetween('number_of_orders', [5, 15])
    ->get();
```

您可以向 `groupBy` 方法傳遞多個參數，以按多個欄位分組：

```php
$users = DB::table('users')
    ->groupBy('first_name', 'status')
    ->having('account_id', '>', 100)
    ->get();
```

要建構更進階的 `having` 語句，請參閱 [havingRaw](#raw-methods) 方法。


<a name="limit-and-offset"></a>
### 限制與偏移

您可以使用 `limit` 和 `offset` 方法來限制查詢返回的結果數量，或跳過查詢中給定數量的結果：

```php
$users = DB::table('users')
    ->offset(10)
    ->limit(5)
    ->get();
```

<a name="conditional-clauses"></a>
## 條件式子句

有時您可能希望根據某個條件，將特定的查詢子句套用至查詢。例如，您可能只想在傳入的 HTTP 請求中存在給定輸入值時，才套用 `where` 語句。您可以使用 `when` 方法來實現此目的：

```php
$role = $request->input('role');

$users = DB::table('users')
    ->when($role, function (Builder $query, string $role) {
        $query->where('role_id', $role);
    })
    ->get();
```

`when` 方法僅在第一個參數為 `true` 時執行給定的閉包。如果第一個參數為 `false`，則閉包將不會被執行。因此，在上述範例中，傳給 `when` 方法的閉包僅在傳入的請求中存在 `role` 欄位且其值為 `true` 時才會被調用。

您可以將另一個閉包作為 `when` 方法的第三個參數傳入。此閉包僅在第一個參數評估為 `false` 時執行。為了說明此功能的用途，我們將使用它來配置查詢的預設排序：

```php
$sortByVotes = $request->boolean('sort_by_votes');

$users = DB::table('users')
    ->when($sortByVotes, function (Builder $query, bool $sortByVotes) {
        $query->orderBy('votes');
    }, function (Builder $query) {
        $query->orderBy('name');
    })
    ->get();
```


<a name="insert-statements"></a>
## Insert 語句

查詢產生器還提供了 `insert` 方法，可用於將記錄插入資料庫資料表。`insert` 方法接受一個欄位名稱和值組成的陣列：

```php
DB::table('users')->insert([
    'email' => 'kayla@example.com',
    'votes' => 0
]);
```

您可以透過傳遞一個陣列的陣列，一次插入多筆記錄。每個陣列都代表一筆應該插入到資料表中的記錄：

```php
DB::table('users')->insert([
    ['email' => 'picard@example.com', 'votes' => 0],
    ['email' => 'janeway@example.com', 'votes' => 0],
]);
```

`insertOrIgnore` 方法會在將記錄插入資料庫時忽略錯誤。使用此方法時，您應該注意重複記錄錯誤將會被忽略，並且根據資料庫引擎的不同，其他類型的錯誤也可能被忽略。例如，`insertOrIgnore` 將會[繞過 MySQL 的嚴格模式](https://dev.mysql.com/doc/refman/en/sql-mode.html#ignore-effect-on-execution)：

```php
DB::table('users')->insertOrIgnore([
    ['id' => 1, 'email' => 'sisko@example.com'],
    ['id' => 2, 'email' => 'archer@example.com'],
]);
```

`insertUsing` 方法會利用子查詢來決定要插入的資料，從而將新記錄插入到資料表：

```php
DB::table('pruned_users')->insertUsing([
    'id', 'name', 'email', 'email_verified_at'
], DB::table('users')->select(
    'id', 'name', 'email', 'email_verified_at'
)->where('updated_at', '<=', now()->subMonth()));
```


<a name="auto-incrementing-ids"></a>
#### 自動遞增 ID

如果資料表具有自動遞增 ID，可以使用 `insertGetId` 方法插入一筆記錄並取回 ID：

```php
$id = DB::table('users')->insertGetId(
    ['email' => 'john@example.com', 'votes' => 0]
);
```

> [!WARNING]
> 當使用 PostgreSQL 時，`insertGetId` 方法預期自動遞增欄位被命名為 `id`。如果您想從不同的「序列」中取回 ID，可以將欄位名稱作為 `insertGetId` 方法的第二個參數傳遞。


<a name="upserts"></a>
### 增量更新 (Upserts)

`upsert` 方法會插入不存在的記錄，並使用您可以指定的新值更新已存在的記錄。該方法的第一個參數包含要插入或更新的值，而第二個參數列出唯一識別相關資料表中記錄的欄位。該方法的第三個也是最後一個參數是一個欄位陣列，當資料庫中已存在匹配的記錄時，這些欄位應該被更新：

```php
DB::table('flights')->upsert(
    [
        ['departure' => 'Oakland', 'destination' => 'San Diego', 'price' => 99],
        ['departure' => 'Chicago', 'destination' => 'New York', 'price' => 150]
    ],
    ['departure', 'destination'],
    ['price']
);
```

在上述範例中，Laravel 將會嘗試插入兩筆記錄。如果已存在具有相同 `departure` 和 `destination` 欄位值的記錄，Laravel 將會更新該記錄的 `price` 欄位。

> [!WARNING]
> 除 SQL Server 外的所有資料庫都要求 `upsert` 方法的第二個參數中的欄位具有「主鍵」或「唯一」索引。此外，MariaDB 和 MySQL 資料庫驅動程式會忽略 `upsert` 方法的第二個參數，並總是使用資料表的主鍵和唯一索引來檢測現有記錄。


<a name="update-statements"></a>
## Update 語句

除了將記錄插入資料庫，查詢產生器還可以使用 `update` 方法更新現有記錄。`update` 方法與 `insert` 方法一樣，接受一個欄位與值配對的陣列，指示要更新的欄位。`update` 方法會回傳受影響的列數。您可以使用 `where` 子句來約束 `update` 查詢：

```php
$affected = DB::table('users')
    ->where('id', 1)
    ->update(['votes' => 1]);
```


<a name="update-or-insert"></a>
#### 更新或插入

有時您可能希望更新資料庫中現有的記錄，如果不存在匹配的記錄則建立它。在這種情況下，可以使用 `updateOrInsert` 方法。`updateOrInsert` 方法接受兩個參數：一個是用於尋找記錄的條件陣列，另一個是欄位與值配對的陣列，指示要更新的欄位。

`updateOrInsert` 方法會嘗試使用第一個參數的欄位與值配對來定位匹配的資料庫記錄。如果記錄存在，它將會使用第二個參數中的值進行更新。如果找不到記錄，則會插入一筆新記錄，其屬性為兩個參數合併後的結果：

```php
DB::table('users')
    ->updateOrInsert(
        ['email' => 'john@example.com', 'name' => 'John'],
        ['votes' => '2']
    );
```

您可以向 `updateOrInsert` 方法提供一個閉包，以根據是否存在匹配記錄來自訂要更新或插入到資料庫中的屬性：

```php
DB::table('users')->updateOrInsert(
    ['user_id' => $user_id],
    fn ($exists) => $exists ? [
        'name' => $data['name'],
        'email' => $data['email'],
    ] : [
        'name' => $data['name'],
        'email' => $data['email'],
        'marketable' => true,
    ],
);
```


<a name="updating-json-columns"></a>
### 更新 JSON 欄位

更新 JSON 欄位時，您應該使用 `->` 語法來更新 JSON 物件中適當的鍵。此操作支援 MariaDB 10.3+、MySQL 5.7+ 和 PostgreSQL 9.5+：

```php
$affected = DB::table('users')
    ->where('id', 1)
    ->update(['options->enabled' => true]);
```


<a name="increment-and-decrement"></a>
### 增量與減量

查詢產生器還提供了用於增量或減量指定欄位值的便利方法。這兩種方法都接受至少一個參數：要修改的欄位。可以提供第二個參數來指定欄位應增量或減量的數量：

```php
DB::table('users')->increment('votes');

DB::table('users')->increment('votes', 5);

DB::table('users')->decrement('votes');

DB::table('users')->decrement('votes', 5);
```

如果需要，您還可以指定要在增量或減量操作期間更新的額外欄位：

```php
DB::table('users')->increment('votes', 1, ['name' => 'John']);
```

此外，您可以使用 `incrementEach` 和 `decrementEach` 方法一次增量或減量多個欄位：

```php
DB::table('users')->incrementEach([
    'votes' => 5,
    'balance' => 100,
]);
```

<a name="delete-statements"></a>
## Delete 語句

查詢產生器的 `delete` 方法可用於從資料表中刪除記錄。`delete` 方法會傳回受影響的列數。您可以在呼叫 `delete` 方法之前，透過新增「where」子句來約束 `delete` 語句：

```php
$deleted = DB::table('users')->delete();

$deleted = DB::table('users')->where('votes', '>', 100)->delete();
```


<a name="pessimistic-locking"></a>
## 悲觀鎖

查詢產生器也包含一些功能，可協助您在執行 `select` 語句時實現「悲觀鎖」。若要執行具有「共享鎖」的語句，您可以呼叫 `sharedLock` 方法。共享鎖會阻止選定的列被修改，直到您的交易被提交為止：

```php
DB::table('users')
    ->where('votes', '>', 100)
    ->sharedLock()
    ->get();
```

或者，您可以使用 `lockForUpdate` 方法。「for update」鎖會阻止選定的記錄被修改，或被其他共享鎖選取：

```php
DB::table('users')
    ->where('votes', '>', 100)
    ->lockForUpdate()
    ->get();
```

雖然不是強制性的，但建議將悲觀鎖包裝在 [交易](/docs/{{version}}/database#database-transactions) 中。這確保了在整個操作完成之前，資料庫中檢索到的資料保持不變。如果發生故障，交易將自動回滾任何變更並釋放鎖定：

```php
DB::transaction(function () {
    $sender = DB::table('users')
        ->lockForUpdate()
        ->find(1);

    $receiver = DB::table('users')
        ->lockForUpdate()
        ->find(2);

    if ($sender->balance < 100) {
        throw new RuntimeException('Balance too low.');
    }

    DB::table('users')
        ->where('id', $sender->id)
        ->update([
            'balance' => $sender->balance - 100
        ]);

    DB::table('users')
        ->where('id', $receiver->id)
        ->update([
            'balance' => $receiver->balance + 100
        ]);
});
```


<a name="reusable-query-components"></a>
## 可重複使用查詢組件

如果您的應用程式中存在重複的查詢邏輯，您可以使用查詢產生器的 `tap` 和 `pipe` 方法將邏輯提取到可重複使用的物件中。想像一下您的應用程式中有以下兩個不同的查詢：

```php
use Illuminate\Database\Query\Builder;
use Illuminate\Support\Facades\DB;

$destination = $request->query('destination');

DB::table('flights')
    ->when($destination, function (Builder $query, string $destination) {
        $query->where('destination', $destination);
    })
    ->orderByDesc('price')
    ->get();

// ...

$destination = $request->query('destination');

DB::table('flights')
    ->when($destination, function (Builder $query, string $destination) {
        $query->where('destination', $destination);
    })
    ->where('user', $request->user()->id)
    ->orderBy('destination')
    ->get();
```

您可能會希望將這些查詢中常見的目的地過濾邏輯提取到一個可重複使用的物件中：

```php
<?php

namespace App\Scopes;

use Illuminate\Database\Query\Builder;

class DestinationFilter
{
    public function __construct(
        private ?string $destination,
    ) {
        //
    }

    public function __invoke(Builder $query): void
    {
        $query->when($this->destination, function (Builder $query) {
            $query->where('destination', $this->destination);
        });
    }
}
```

然後，您可以使用查詢產生器的 `tap` 方法將該物件的邏輯應用於查詢：

```php
use App\Scopes\DestinationFilter;
use Illuminate\Database\Query\Builder;
use Illuminate\Support\Facades\DB;

DB::table('flights')
    ->when($destination, function (Builder $query, string $destination) { // [tl! remove]
        $query->where('destination', $destination); // [tl! remove]
    }) // [tl! remove]
    ->tap(new DestinationFilter($destination)) // [tl! add]
    ->orderByDesc('price')
    ->get();

// ...

DB::table('flights')
    ->when($destination, function (Builder $query, string $destination) { // [tl! remove]
        $query->where('destination', $destination); // [tl! remove]
    }) // [tl! remove]
    ->tap(new DestinationFilter($destination)) // [tl! add]
    ->where('user', $request->user()->id)
    ->orderBy('destination')
    ->get();
```


<a name="query-pipes"></a>
#### 查詢管道

`tap` 方法將始終傳回查詢產生器。如果您希望提取一個執行查詢並傳回另一個值的物件，則可以使用 `pipe` 方法。

考慮以下查詢物件，其中包含應用程式中使用的共享 [分頁](/docs/{{version}}/pagination) 邏輯。與對查詢應用查詢條件的 `DestinationFilter` 不同，`Paginate` 物件執行查詢並傳回一個分頁器實例：

```php
<?php

namespace App\Scopes;

use Illuminate\Contracts\Pagination\LengthAwarePaginator;
use Illuminate\Database\Query\Builder;

class Paginate
{
    public function __construct(
        private string $sortBy = 'timestamp',
        private string $sortDirection = 'desc',
        private int $perPage = 25,
    ) {
        //
    }

    public function __invoke(Builder $query): LengthAwarePaginator
    {
        return $query->orderBy($this->sortBy, $this->sortDirection)
            ->paginate($this->perPage, pageName: 'p');
    }
}
```

使用查詢產生器的 `pipe` 方法，我們可以利用此物件來應用我們的共享分頁邏輯：

```php
$flights = DB::table('flights')
    ->tap(new DestinationFilter($destination))
    ->pipe(new Paginate);
```


<a name="debugging"></a>
## 除錯

您可以在建構查詢時使用 `dd` 和 `dump` 方法來輸出目前的查詢綁定和 SQL。`dd` 方法將顯示除錯資訊，然後停止執行請求。`dump` 方法將顯示除錯資訊，但允許請求繼續執行：

```php
DB::table('users')->where('votes', '>', 100)->dd();

DB::table('users')->where('votes', '>', 100)->dump();
```

`dumpRawSql` 和 `ddRawSql` 方法可以在查詢上呼叫，以輸出已正確替換所有參數綁定的查詢 SQL：

```php
DB::table('users')->where('votes', '>', 100)->dumpRawSql();

DB::table('users')->where('votes', '>', 100)->ddRawSql();
```