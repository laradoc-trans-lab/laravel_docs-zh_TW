# 資料庫：查詢生成器

- [簡介](#introduction)
- [執行資料庫查詢](#running-database-queries)
    - [分塊處理結果](#chunking-results)
    - [惰性串流結果](#streaming-results-lazily)
    - [聚合](#aggregates)
- [Select 陳述式](#select-statements)
- [原生運算式](#raw-expressions)
- [Joins (連接)](#joins)
- [Unions (聯集)](#unions)
- [基礎 Where 子句](#basic-where-clauses)
    - [Where 子句](#where-clauses)
    - [Or Where 子句](#or-where-clauses)
    - [Where Not 子句](#where-not-clauses)
    - [Where Any / All / None 子句](#where-any-all-none-clauses)
    - [JSON Where 子句](#json-where-clauses)
    - [其他 Where 子句](#additional-where-clauses)
    - [邏輯分組](#logical-grouping)
- [進階 Where 子句](#advanced-where-clauses)
    - [Where Exists 子句](#where-exists-clauses)
    - [子查詢 Where 子句](#subquery-where-clauses)
    - [全文檢索 Where 子句](#full-text-where-clauses)
    - [向量相似度子句](#vector-similarity-clauses)
- [排序、分組、Limit 與 Offset](#ordering-grouping-limit-and-offset)
    - [排序](#ordering)
    - [分組](#grouping)
    - [Limit 與 Offset](#limit-and-offset)
- [條件子句](#conditional-clauses)
- [Insert 陳述式](#insert-statements)
    - [Upsert (更新或新增)](#upserts)
- [Update 陳述式](#update-statements)
    - [更新 JSON 欄位](#updating-json-columns)
    - [遞增與遞減](#increment-and-decrement)
- [Delete 陳述式](#delete-statements)
- [悲觀鎖定](#pessimistic-locking)
- [可重複使用的查詢元件](#reusable-query-components)
- [偵錯](#debugging)

<a name="introduction"></a>
## 簡介

Laravel 的資料庫查詢生成器提供了一個方便且順暢的介面，用於建立與執行資料庫查詢。它可以用來執行您應用程式中的大部分資料庫操作，並且能完美相容於所有 Laravel 支援的資料庫系統。

Laravel 查詢生成器使用 PDO 參數綁定來保護您的應用程式免受 SQL 注入攻擊。您無需清理或過濾作為查詢綁定傳遞給查詢生成器的字串。

> [!WARNING]
> PDO 不支援綁定欄位名稱。因此，您絕不應該讓使用者的輸入決定查詢中所引用的欄位名稱，包含 "order by" 欄位。

<a name="running-database-queries"></a>
## 執行資料庫查詢

<a name="retrieving-all-rows-from-a-table"></a>
#### 從資料表中取得所有資料列

您可以使用 `DB` Facade 所提供的 `table` 方法來開始進行查詢。`table` 方法會針對指定的資料表傳回一個流暢的查詢生成器實例，讓您可以在查詢上鏈結更多的約束條件，最後再使用 `get` 方法取得查詢結果：

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

`get` 方法會傳回一個包含查詢結果的 `Illuminate\Support\Collection` 實例，其中每個結果都是 PHP `stdClass` 物件的實例。您可以將欄位視為該物件的屬性來存取每個欄位的值：

```php
use Illuminate\Support\Facades\DB;

$users = DB::table('users')->get();

foreach ($users as $user) {
    echo $user->name;
}
```

> [!NOTE]
> Laravel 集合提供了各種極為強大的方法來映射與轉換資料。如需更多關於 Laravel 集合的資訊，請參考[集合文件](/docs/{{version}}/collections)。

<a name="retrieving-a-single-row-column-from-a-table"></a>
#### 從資料表中取得單一資料列 / 欄位

如果您只需要從資料表中取得單一資料列，可以使用 `DB` Facade 的 `first` 方法。此方法將傳回單一 `stdClass` 物件：

```php
$user = DB::table('users')->where('name', 'John')->first();

return $user->email;
```

如果您想從資料表中取得單一資料列，但若找不到匹配的資料列時希望拋出 `Illuminate\Database\RecordNotFoundException`，您可以使用 `firstOrFail` 方法。如果 `RecordNotFoundException` 未被捕捉，系統將會自動傳回 404 HTTP 回應給用戶端：

```php
$user = DB::table('users')->where('name', 'John')->firstOrFail();
```

如果您不需要整行資料列，可以使用 `value` 方法從紀錄中選取單一數值。此方法將直接傳回該欄位的值：

```php
$email = DB::table('users')->where('name', 'John')->value('email');
```

若要透過 `id` 欄位的值來取得單一資料列，請使用 `find` 方法：

```php
$user = DB::table('users')->find(3);
```

<a name="retrieving-a-list-of-column-values"></a>
#### 取得單一欄位的值列表

如果您想取得包含單一欄位所有值的 `Illuminate\Support\Collection` 實例，可以使用 `pluck` 方法。在這個範例中，我們將取得使用者頭銜的集合：

```php
use Illuminate\Support\Facades\DB;

$titles = DB::table('users')->pluck('title');

foreach ($titles as $title) {
    echo $title;
}
```

您可以透過傳入第二個引數給 `pluck` 方法，來指定傳回的集合要使用哪個欄位作為鍵名：

```php
$titles = DB::table('users')->pluck('title', 'name');

foreach ($titles as $name => $title) {
    echo $title;
}
```

<a name="chunking-results"></a>
### 分塊處理結果

如果您需要處理數千條資料庫紀錄，可以考慮使用 `DB` Facade 提供的 `chunk` 方法。此方法一次只檢索一小部分（分塊）結果，並將每個分塊傳入閉包進行處理。例如，讓我們以一次 100 條紀錄的分塊方式檢索整個 `users` 資料表：

```php
use Illuminate\Support\Collection;
use Illuminate\Support\Facades\DB;

DB::table('users')->orderBy('id')->chunk(100, function (Collection $users) {
    foreach ($users as $user) {
        // ...
    }
});
```

您可以透過從閉包傳回 `false` 來停止後續分塊的處理：

```php
DB::table('users')->orderBy('id')->chunk(100, function (Collection $users) {
    // Process the records...

    return false;
});
```

如果您在分塊處理結果的同時更新資料庫紀錄，您的分塊結果可能會以非預期的方式改變。如果您打算在分塊時更新檢索到的紀錄，最好改用 `chunkById` 方法。此方法會自動根據紀錄的主鍵對結果進行分頁：

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

由於 `chunkById` 與 `lazyById` 方法會在執行的查詢中加入自己的「where」條件，您通常應該在閉包內對您自己的條件進行[邏輯分組](#logical-grouping)：

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
> 在分塊回呼函式中更新或刪除紀錄時，對主鍵或外鍵進行的任何變更都可能會影響分塊查詢。這可能會導致某些紀錄未能包含在分塊結果中。

<a name="streaming-results-lazily"></a>
### 惰性串流結果

`lazy` 方法運作方式與[分塊處理結果](#chunking-results)類似，因為它也是以分塊方式執行查詢。然而，`lazy()` 方法不會將每個分塊傳入回呼函式，而是傳回一個 [LazyCollection](/docs/{{version}}/collections#lazy-collections)，讓您可以將結果視為單一串流來進行互動：

```php
use Illuminate\Support\Facades\DB;

DB::table('users')->orderBy('id')->lazy()->each(function (object $user) {
    // ...
});
```

同樣地，如果您打算在迭代紀錄時進行更新，最好改用 `lazyById` 或 `lazyByIdDesc` 方法。這些方法會自動根據紀錄的主鍵對結果進行分頁：

```php
DB::table('users')->where('active', false)
    ->lazyById()->each(function (object $user) {
        DB::table('users')
            ->where('id', $user->id)
            ->update(['active' => true]);
    });
```

> [!WARNING]
> 在迭代紀錄的同時進行更新或刪除時，對主鍵或外鍵進行的任何變更都可能會影響分塊查詢。這可能會導致某些紀錄未能包含在結果中。

<a name="aggregates"></a>
### 聚合

查詢生成器還提供了各種用於取得聚合值的方法，例如 `count`、`max`、`min`、`avg` 以及 `sum`。您可以在建立查詢後呼叫這些方法中的任何一個：

```php
use Illuminate\Support\Facades\DB;

$users = DB::table('users')->count();

$price = DB::table('orders')->max('price');
```

當然，您可以將這些方法與其他子句結合使用，以精細調整聚合值的計算方式：

```php
$price = DB::table('orders')
    ->where('finalized', 1)
    ->avg('price');
```

<a name="determining-if-records-exist"></a>
#### 判斷紀錄是否存在

除了使用 `count` 方法來判斷是否存在符合您查詢約束條件的紀錄之外，您還可以使用 `exists` 與 `doesntExist` 方法：

```php
if (DB::table('orders')->where('finalized', 1)->exists()) {
    // ...
}

if (DB::table('orders')->where('finalized', 1)->doesntExist()) {
    // ...
}
```

<a name="select-statements"></a>
## Select 陳述式


<a name="specifying-a-select-clause"></a>
#### 指定 Select 子句

您可能不總是想要選取資料庫表單中的所有欄位。使用 `select` 方法，您可以為查詢指定自訂的「select」子句：

```php
use Illuminate\Support\Facades\DB;

$users = DB::table('users')
    ->select('name', 'email as user_email')
    ->get();
```

`distinct` 方法允許您強制查詢回傳不重複的結果：

```php
$users = DB::table('users')->distinct()->get();
```

如果您已經有一個查詢生成器實例，且希望在現有的 select 子句中新增欄位，可以使用 `addSelect` 方法：

```php
$query = DB::table('users')->select('name');

$users = $query->addSelect('age')->get();
```


<a name="raw-expressions"></a>
## 原生運算式

有時您可能需要在查詢中插入任意字串。若要建立原生字串運算式，可以使用 `DB` Facade 提供的 `raw` 方法：

```php
$users = DB::table('users')
    ->select(DB::raw('count(*) as user_count, status'))
    ->where('status', '<>', 1)
    ->groupBy('status')
    ->get();
```

> [!WARNING]
> 原生陳述式會以字串形式注入到查詢中，因此您應該非常小心，以避免產生 SQL 注入 (SQL injection) 漏洞。


<a name="raw-methods"></a>
### 原生方法

除了使用 `DB::raw` 方法外，您還可以使用以下方法將原生運算式插入到查詢的不同部分。**請記住，Laravel 無法保證任何使用原生運算式的查詢都能防止 SQL 注入漏洞。**


<a name="selectraw"></a>
#### `selectRaw`

`selectRaw` 方法可以用來取代 `addSelect(DB::raw(/* ... */))`。該方法接受一個選擇性的綁定陣列作為其第二個引數：

```php
$orders = DB::table('orders')
    ->selectRaw('price * ? as price_with_tax', [1.0825])
    ->get();
```


<a name="whereraw-orwhereraw"></a>
#### `whereRaw / orWhereRaw`

`whereRaw` 和 `orWhereRaw` 方法可以用於將原生的「where」子句注入到您的查詢中。這些方法接受一個選擇性的綁定陣列作為其第二個引數：

```php
$orders = DB::table('orders')
    ->whereRaw('price > IF(state = "TX", ?, 100)', [200])
    ->get();
```


<a name="havingraw-orhavingraw"></a>
#### `havingRaw / orHavingRaw`

`havingRaw` 和 `orHavingRaw` 方法可用於提供原生字串作為「having」子句的值。這些方法接受一個選擇性的綁定陣列作為其第二個引數：

```php
$orders = DB::table('orders')
    ->select('department', DB::raw('SUM(price) as total_sales'))
    ->groupBy('department')
    ->havingRaw('SUM(price) > ?', [2500])
    ->get();
```


<a name="orderbyraw"></a>
#### `orderByRaw`

`orderByRaw` 方法可用於提供原生字串作為「order by」子句的值：

```php
$orders = DB::table('orders')
    ->orderByRaw('updated_at - created_at DESC')
    ->get();
```


<a name="groupbyraw"></a>
#### `groupByRaw`

`groupByRaw` 方法可用於提供原生字串作為 `group by` 子句的值：

```php
$orders = DB::table('orders')
    ->select('city', 'state')
    ->groupByRaw('city, state')
    ->get();
```


<a name="joins"></a>
## Joins (連接)


<a name="inner-join-clause"></a>
#### Inner Join 子句

查詢生成器也可用於在您的查詢中新增 join 子句。要執行基礎的「inner join」，可以在查詢生成器實例上使用 `join` 方法。傳遞給 `join` 方法的第一個引數是您需要連接的資料表名稱，而其餘引數則指定 join 的欄位條件限制。您甚至可以在單一查詢中連接多個資料表：

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

如果您想執行「left join」或「right join」而不是「inner join」，請使用 `leftJoin` 或 `rightJoin` 方法。這些方法與 `join` 方法具有相同的簽名 (signature)：

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

您可以選用 `crossJoin` 方法來執行「cross join」。Cross join 會在第一個資料表與被連接的資料表之間產生笛卡兒積 (Cartesian product)：

```php
$sizes = DB::table('sizes')
    ->crossJoin('colors')
    ->get();
```


<a name="advanced-join-clauses"></a>
#### 進階 Join 子句

您還可以指定更進階的 join 子句。首先，將閉包 (closure) 作為第二個引數傳遞給 `join` 方法。該閉包將接收一個 `Illuminate\Database\Query\JoinClause` 實例，允許您在「join」子句上指定條件限制：

```php
DB::table('users')
    ->join('contacts', function (JoinClause $join) {
        $join->on('users.id', '=', 'contacts.user_id')->orOn(/* ... */);
    })
    ->get();
```

如果您想在 join 上使用「where」子句，可以使用 `JoinClause` 實例提供的 `where` 和 `orWhere` 方法。這些方法將欄位與一個數值進行比較，而不是比較兩個欄位：

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

您可以使用 `joinSub`、`leftJoinSub` 與 `rightJoinSub` 方法將查詢與子查詢連接起來。這些方法各接收三個引數：子查詢、其資料表別名，以及定義相關欄位的閉包。在這個範例中，我們將取得使用者集合，其中每個使用者記錄還包含該使用者最近發布的部落格文章的 `created_at` 時間戳記：

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
#### Lateral Joins

> [!WARNING]
> Lateral join 目前支援 PostgreSQL、MySQL >= 8.0.14 以及 SQL Server。

您可以使用 `joinLateral` 和 `leftJoinLateral` 方法來對子查詢執行「lateral join」。這些方法各接收兩個引數：子查詢及其資料表別名。Join 條件應在給定子查詢的 `where` 子句中指定。Lateral join 會對每一行進行評估，並可以引用子查詢外部的欄位。

在這個範例中，我們將取得使用者集合以及該使用者最新的三篇部落格文章。每個使用者在結果集中最多可產生三行：代表其最新部落格文章中的每一篇。Join 條件是在子查詢內部使用 `whereColumn` 子句來指定的，該子句引用了當前的使用者資料列：

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
## Unions (聯集)

查詢生成器也提供了一個方便的方法將兩個或多個查詢「聯集 (union)」在一起。例如，您可以建立一個初始查詢，並使用 `union` 方法將其與更多查詢進行聯集：

```php
use Illuminate\Support\Facades\DB;

$usersWithoutFirstName = DB::table('users')
    ->whereNull('first_name');

$users = DB::table('users')
    ->whereNull('last_name')
    ->union($usersWithoutFirstName)
    ->get();
```

除了 `union` 方法外，查詢生成器還提供了 `unionAll` 方法。使用 `unionAll` 方法結合的查詢將不會移除重複的結果。`unionAll` 方法具有與 `union` 方法相同的方法簽章。

<a name="basic-where-clauses"></a>
## 基礎 Where 子句


<a name="where-clauses"></a>
### Where 子句

您可以使用查詢生成器的 `where` 方法在查詢中加入「where」子句。呼叫 `where` 方法最基本需要三個引數。第一個引數是欄位名稱。第二個引數是運算子，可以是資料庫支援的任何運算子。第三個引數則是要與欄位值進行比較的數值。

例如，以下查詢會擷取 `votes` 欄位值等於 `100` 且 `age` 欄位值大於 `35` 的使用者：

```php
$users = DB::table('users')
    ->where('votes', '=', 100)
    ->where('age', '>', 35)
    ->get();
```

為求方便，如果您只想驗證欄位是否 `=` 指定數值，可以直接將該數值作為第二個引數傳給 `where` 方法。Laravel 會預設您想使用 `=` 運算子：

```php
$users = DB::table('users')->where('votes', 100)->get();
```

您也可以傳遞關聯陣列給 `where` 方法，以快速查詢多個欄位：

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

您也可以將包含多個條件的陣列傳給 `where` 函式。陣列中的每個元素都應該是一個包含通常傳給 `where` 方法的三個引數的陣列：

```php
$users = DB::table('users')->where([
    ['status', '=', '1'],
    ['subscribed', '<>', '1'],
])->get();
```

> [!WARNING]
> PDO 不支援綁定欄位名稱。因此，您絕不應該允許使用者輸入來決定查詢所引用的欄位名稱，包含「order by」的欄位。

> [!WARNING]
> MySQL 與 MariaDB 在進行字串與數字比較時，會自動將字串型別轉換為整數。在這個過程中，非數字字串會被轉換為 `0`，這可能會導致意料之外的結果。例如，若您的資料表中有一個 `secret` 欄位值為 `aaa`，而您執行了 `User::where('secret', 0)`，該筆資料就會被回傳。為了避免這種情況，請確保所有數值在查詢中使用前都已轉換為適當的型別。


<a name="or-where-clauses"></a>
### Or Where 子句

當連續呼叫查詢生成器的 `where` 方法時，這些「where」子句會使用 `and` 運算子組合在一起。不過，您可以改用 `orWhere` 方法，透過 `or` 運算子將子句加入查詢中。`orWhere` 方法接受與 `where` 方法相同的引數：

```php
$users = DB::table('users')
    ->where('votes', '>', 100)
    ->orWhere('name', 'John')
    ->get();
```

如果您需要將「or」條件用括號括起來分組，可以將閉包作為第一個引數傳給 `orWhere` 方法：

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
> 您應該總是將 `orWhere` 呼叫進行分組，以避免在套用全域作用域 (Global Scopes) 時發生非預期的行為。


<a name="where-not-clauses"></a>
### Where Not 子句

`whereNot` 與 `orWhereNot` 方法可用於否定指定的一組查詢約束條件。例如，以下查詢排除正在清倉拍賣或價格小於 10 的商品：

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

有時候您可能需要將相同的查詢約束條件套用到多個欄位。例如，您可能想擷取指定清單中任一欄位 `LIKE` 指定數值的所有紀錄。您可以使用 `whereAny` 方法來達成此目的：

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

同理，`whereAll` 方法可用於擷取所有指定欄位皆符合特定約束條件的紀錄：

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

`whereNone` 方法可用於擷取所有指定欄位皆不符合特定約束條件的紀錄：

```php
$albums = DB::table('albums')
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

Laravel 也支援在提供 JSON 欄位型別支援的資料庫上查詢 JSON 欄位型別。目前包括 MariaDB 10.3+、MySQL 8.0+、PostgreSQL 12.0+、SQL Server 2017+ 以及 SQLite 3.39.0+。若要查詢 JSON 欄位，請使用 `->` 運算子：

```php
$users = DB::table('users')
    ->where('preferences->dining->meal', 'salad')
    ->get();

$users = DB::table('users')
    ->whereIn('preferences->dining->meal', ['pasta', 'salad', 'sandwiches'])
    ->get();
```

您可以使用 `whereJsonContains` 與 `whereJsonDoesntContain` 方法來查詢 JSON 陣列：

```php
$users = DB::table('users')
    ->whereJsonContains('options->languages', 'en')
    ->get();

$users = DB::table('users')
    ->whereJsonDoesntContain('options->languages', 'en')
    ->get();
```

如果您的應用程式使用 MariaDB、MySQL 或 PostgreSQL 資料庫，您可以傳遞一個包含多個數值的陣列給 `whereJsonContains` 與 `whereJsonDoesntContain` 方法：

```php
$users = DB::table('users')
    ->whereJsonContains('options->languages', ['en', 'de'])
    ->get();

$users = DB::table('users')
    ->whereJsonDoesntContain('options->languages', ['en', 'de'])
    ->get();
```

此外，您可以使用 `whereJsonContainsKey` 或 `whereJsonDoesntContainKey` 方法來擷取包含或不包含指定 JSON 鍵 (Key) 的結果：

```php
$users = DB::table('users')
    ->whereJsonContainsKey('preferences->dietary_requirements')
    ->get();

$users = DB::table('users')
    ->whereJsonDoesntContainKey('preferences->dietary_requirements')
    ->get();
```

最後，您可以使用 `whereJsonLength` 方法依據 JSON 陣列的長度來進行查詢：

```php
$users = DB::table('users')
    ->whereJsonLength('options->languages', 0)
    ->get();

$users = DB::table('users')
    ->whereJsonLength('options->languages', '>', 1)
    ->get();
```

<a name="additional-where-clauses"></a>
### 其他 Where 子句

**whereLike / orWhereLike / whereNotLike / orWhereNotLike**

`whereLike` 方法允許你向查詢加入 "LIKE" 子句以進行模式比對。這些方法提供了一種獨立於資料庫的字串比對查詢方式，並支援切換是否區分大小寫。預設情況下，字串比對是不區分大小寫的：

```php
$users = DB::table('users')
    ->whereLike('name', '%John%')
    ->get();
```

你可以透過 `caseSensitive` 引數來啟用區分大小寫的搜尋：

```php
$users = DB::table('users')
    ->whereLike('name', '%John%', caseSensitive: true)
    ->get();
```

`orWhereLike` 方法允許你加入包含 LIKE 條件的 "or" 子句：

```php
$users = DB::table('users')
    ->where('votes', '>', 100)
    ->orWhereLike('name', '%John%')
    ->get();
```

`whereNotLike` 方法允許你向查詢加入 "NOT LIKE" 子句：

```php
$users = DB::table('users')
    ->whereNotLike('name', '%John%')
    ->get();
```

同樣地，你可以使用 `orWhereNotLike` 來加入包含 NOT LIKE 條件的 "or" 子句：

```php
$users = DB::table('users')
    ->where('votes', '>', 100)
    ->orWhereNotLike('name', '%John%')
    ->get();
```

> [!WARNING]
> SQL Server 目前不支援 `whereLike` 的區分大小寫搜尋選項。

**whereIn / whereNotIn / orWhereIn / orWhereNotIn**

`whereIn` 方法驗證指定欄位的值是否包含於給定的陣列中：

```php
$users = DB::table('users')
    ->whereIn('id', [1, 2, 3])
    ->get();
```

`whereNotIn` 方法驗證指定欄位的值是否不包含於給定的陣列中：

```php
$users = DB::table('users')
    ->whereNotIn('id', [1, 2, 3])
    ->get();
```

你也可以提供一個查詢物件作為 `whereIn` 方法的第二個引數：

```php
$activeUsers = DB::table('users')->select('id')->where('is_active', 1);

$comments = DB::table('comments')
    ->whereIn('user_id', $activeUsers)
    ->get();
```

上面的範例將產生以下 SQL：

```sql
select * from comments where user_id in (
    select id
    from users
    where is_active = 1
)
```

> [!WARNING]
> 若要向查詢加入大量的整數綁定陣列，可以使用 `whereIntegerInRaw` 或 `whereIntegerNotInRaw` 方法，這可以大幅降低記憶體使用量。

**whereBetween / orWhereBetween**

`whereBetween` 方法驗證欄位的值是否介於兩個值之間：

```php
$users = DB::table('users')
    ->whereBetween('votes', [1, 100])
    ->get();
```

**whereNotBetween / orWhereNotBetween**

`whereNotBetween` 方法驗證欄位的值是否落在兩個值之外：

```php
$users = DB::table('users')
    ->whereNotBetween('votes', [1, 100])
    ->get();
```

**whereBetweenColumns / whereNotBetweenColumns / orWhereBetweenColumns / orWhereNotBetweenColumns**

`whereBetweenColumns` 方法驗證欄位的值是否介於同一資料表列中另外兩個欄位的值之間：

```php
$patients = DB::table('patients')
    ->whereBetweenColumns('weight', ['minimum_allowed_weight', 'maximum_allowed_weight'])
    ->get();
```

`whereNotBetweenColumns` 方法驗證欄位的值是否落在同一資料表列中另外兩個欄位的值之外：

```php
$patients = DB::table('patients')
    ->whereNotBetweenColumns('weight', ['minimum_allowed_weight', 'maximum_allowed_weight'])
    ->get();
```

**whereValueBetween / whereValueNotBetween / orWhereValueBetween / orWhereValueNotBetween**

`whereValueBetween` 方法驗證給定的值是否介於同一資料表列中相同型別的兩個欄位值之間：

```php
$products = DB::table('products')
    ->whereValueBetween(100, ['min_price', 'max_price'])
    ->get();
```

`whereValueNotBetween` 方法驗證給定的值是否落在同一資料表列中兩個欄位的值之外：

```php
$products = DB::table('products')
    ->whereValueNotBetween(100, ['min_price', 'max_price'])
    ->get();
```

**whereNull / whereNotNull / orWhereNull / orWhereNotNull**

`whereNull` 方法驗證指定欄位的值是否為 `NULL`：

```php
$users = DB::table('users')
    ->whereNull('updated_at')
    ->get();
```

`whereNotNull` 方法驗證欄位的值是否不為 `NULL`：

```php
$users = DB::table('users')
    ->whereNotNull('updated_at')
    ->get();
```

**whereNullSafeEquals / orWhereNullSafeEquals**

`whereNullSafeEquals` 和 `orWhereNullSafeEquals` 方法可用於比較欄位的值與給定的值，同時將兩個 `NULL` 值視為相等：

```php
$lastLoginIp = $request->input('last_login_ip');

$users = DB::table('users')
    ->whereNullSafeEquals('last_login_ip', $lastLoginIp)
    ->get();
```

**whereDate / whereMonth / whereDay / whereYear / whereTime**

`whereDate` 方法可用於將欄位的值與日期進行比較：

```php
$users = DB::table('users')
    ->whereDate('created_at', '2016-12-31')
    ->get();
```

`whereMonth` 方法可用於將欄位的值與特定的月份進行比較：

```php
$users = DB::table('users')
    ->whereMonth('created_at', '12')
    ->get();
```

`whereDay` 方法可用於將欄位的值與一個月中的特定幾號進行比較：

```php
$users = DB::table('users')
    ->whereDay('created_at', '31')
    ->get();
```

`whereYear` 方法可用於將欄位的值與特定的年份進行比較：

```php
$users = DB::table('users')
    ->whereYear('created_at', '2016')
    ->get();
```

`whereTime` 方法可用於將欄位的值與特定的時間進行比較：

```php
$users = DB::table('users')
    ->whereTime('created_at', '=', '11:20:45')
    ->get();
```

**wherePast / whereFuture / whereToday / whereBeforeToday / whereAfterToday**

`wherePast` 與 `whereFuture` 方法可以用來判斷欄位的值是否屬於過去或未來：

```php
$invoices = DB::table('invoices')
    ->wherePast('due_at')
    ->get();

$invoices = DB::table('invoices')
    ->whereFuture('due_at')
    ->get();
```

`whereNowOrPast` 與 `whereNowOrFuture` 方法可以用來判斷欄位的值是否屬於過去或未來，包含目前的日期與時間：

```php
$invoices = DB::table('invoices')
    ->whereNowOrPast('due_at')
    ->get();

$invoices = DB::table('invoices')
    ->whereNowOrFuture('due_at')
    ->get();
```

`whereToday`、`whereBeforeToday` 與 `whereAfterToday` 方法可用於分別判斷欄位的值是否為今天、今天之前或今天之後：

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

同樣地，`whereTodayOrBefore` 與 `whereTodayOrAfter` 方法可用於判斷欄位的值是否在今天之前或今天之後，包含今天的日期：

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

你也可以傳遞一個比較運算子給 `whereColumn` 方法：

```php
$users = DB::table('users')
    ->whereColumn('updated_at', '>', 'created_at')
    ->get();
```

你還可以傳遞一個欄位比較的陣列給 `whereColumn` 方法。這些條件將使用 `and` 運算子組合：

```php
$users = DB::table('users')
    ->whereColumn([
        ['first_name', '=', 'last_name'],
        ['updated_at', '>', 'created_at'],
    ])->get();
```

<a name="logical-grouping"></a>
### 邏輯分組

有時候，您可能需要將數個 "where" 子句放在括號內進行分組，以達到查詢所需的邏輯分組。事實上，為了避免未預期的查詢行為，您通常應該總是將對 `orWhere` 方法的呼叫包裹在括號中。若要達成此目的，您可以傳遞一個 Closure 至 `where` 方法：

```php
$users = DB::table('users')
    ->where('name', '=', 'John')
    ->where(function (Builder $query) {
        $query->where('votes', '>', 100)
            ->orWhere('title', '=', 'Admin');
    })
    ->get();
```

如您所見，傳遞 Closure 給 `where` 方法會指示查詢生成器開始一個條件約束分組。該 Closure 會接收一個查詢生成器實例，您可以使用它來設定應該包含在括號分組內的條件約束。上述範例將會產生如下的 SQL：

```sql
select * from users where name = 'John' and (votes > 100 or title = 'Admin')
```

> [!WARNING]
> 您應該總是將 `orWhere` 呼叫進行分組，以避免套用全域範疇 (Global Scopes) 時產生未預期的行為。

<a name="advanced-where-clauses"></a>
## 進階 Where 子句


<a name="where-exists-clauses"></a>
### Where Exists 子句

`whereExists` 方法允許您撰寫 "where exists" SQL 子句。`whereExists` 方法接受一個閉包，該閉包會接收一個查詢生成器實例，讓您定義應放置於 "exists" 子句內部的查詢：

```php
$users = DB::table('users')
    ->whereExists(function (Builder $query) {
        $query->select(DB::raw(1))
            ->from('orders')
            ->whereColumn('orders.user_id', 'users.id');
    })
    ->get();
```

此外，您也可以向 `whereExists` 方法提供一個查詢物件，而非閉包：

```php
$orders = DB::table('orders')
    ->select(DB::raw(1))
    ->whereColumn('orders.user_id', 'users.id');

$users = DB::table('users')
    ->whereExists($orders)
    ->get();
```

上述兩個範例都將產生以下 SQL：

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

有時您可能需要建構一個 "where" 子句，將子查詢的結果與給定的值進行比較。您可以透過將閉包與值傳遞給 `where` 方法來達成此目的。例如，以下查詢將檢索所有最近擁有指定類型「會員資格 (membership)」的使用者：

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

或者，您可能需要建構一個 "where" 子句，將欄位與子查詢的結果進行比較。您可以透過將欄位、運算子與閉包傳遞給 `where` 方法來達成此目的。例如，以下查詢將檢索金額小於平均值的所有收入記錄：

```php
use App\Models\Income;
use Illuminate\Database\Query\Builder;

$incomes = Income::where('amount', '<', function (Builder $query) {
    $query->selectRaw('avg(i.amount)')->from('incomes as i');
})->get();
```


<a name="full-text-where-clauses"></a>
### 全文檢索 Where 子句

> [!WARNING]
> 全文檢索 where 子句目前支援 MariaDB、MySQL 與 PostgreSQL。

`whereFullText` 與 `orWhereFullText` 方法可用於為擁有[全文索引](/docs/{{version}}/migrations#available-index-types)的欄位新增全文檢索 "where" 子句。Laravel 會將這些方法轉換為適合底層資料庫系統的 SQL。例如，使用 MariaDB 或 MySQL 的應用程式將會產生 `MATCH AGAINST` 子句：

```php
$users = DB::table('users')
    ->whereFullText('bio', 'web developer')
    ->get();
```


<a name="vector-similarity-clauses"></a>
### 向量相似度子句

> [!NOTE]
> 向量相似度子句目前僅支援使用 `pgvector` 擴充套件的 PostgreSQL 連線。有關定義向量欄位與索引的資訊，請參閱[遷移文件](/docs/{{version}}/migrations#available-column-types)。

`whereVectorSimilarTo` 方法透過計算與給定向量的餘弦相似度 (cosine similarity) 來篩選結果，並依相關性排序結果。`minSimilarity` 門檻值應為 `0.0` 到 `1.0` 之間的值，其中 `1.0` 代表完全相同：

```php
$documents = DB::table('documents')
    ->whereVectorSimilarTo('embedding', $queryEmbedding, minSimilarity: 0.4)
    ->limit(10)
    ->get();
```

當傳入純字串作為向量引數時，Laravel 會自動使用 [Laravel AI SDK](/docs/{{version}}/ai-sdk#embeddings) 為其產生嵌入向量 (embeddings)：

```php
$documents = DB::table('documents')
    ->whereVectorSimilarTo('embedding', 'Best wineries in Napa Valley')
    ->limit(10)
    ->get();
```

預設情況下，`whereVectorSimilarTo` 還會按距離排序結果（最相似的排在最前面）。您可以透過將 `false` 作為 `order` 引數傳入來停用此排序：

```php
$documents = DB::table('documents')
    ->whereVectorSimilarTo('embedding', $queryEmbedding, minSimilarity: 0.4, order: false)
    ->orderBy('created_at', 'desc')
    ->limit(10)
    ->get();
```

如果您需要更多控制權，可以單獨使用 `selectVectorDistance`、`whereVectorDistanceLessThan` 和 `orderByVectorDistance` 方法：

```php
$documents = DB::table('documents')
    ->select('*')
    ->selectVectorDistance('embedding', $queryEmbedding, as: 'distance')
    ->whereVectorDistanceLessThan('embedding', $queryEmbedding, maxDistance: 0.3)
    ->orderByVectorDistance('embedding', $queryEmbedding)
    ->limit(10)
    ->get();
```

使用 PostgreSQL 時，必須先載入 `pgvector` 擴充套件才能建立 `vector` 欄位：

```php
Schema::ensureVectorExtensionExists();
```

<a name="ordering-grouping-limit-and-offset"></a>
## 排序、分組、Limit 與 Offset


<a name="ordering"></a>
### 排序


<a name="orderby"></a>
#### `orderBy` 方法

`orderBy` 方法允許您根據給定的欄位對查詢結果進行排序。`orderBy` 方法接受的第一個引數應該是您希望排序的欄位，而第二個引數則決定排序的方向，可以是 `asc` 或 `desc`：

```php
$users = DB::table('users')
    ->orderBy('name', 'desc')
    ->get();
```

若要依多個欄位排序，您只需根據需要多次呼叫 `orderBy` 即可：

```php
$users = DB::table('users')
    ->orderBy('name', 'desc')
    ->orderBy('email', 'asc')
    ->get();
```

排序方向是可選的，預設為升冪 (ascending)。如果您想以降冪排序，可以為 `orderBy` 方法指定第二個參數，或者直接使用 `orderByDesc`：

```php
$users = DB::table('users')
    ->orderByDesc('verified_at')
    ->get();
```

最後，使用 `->` 運算子，也可以根據 JSON 欄位內的值進行排序：

```php
$corporations = DB::table('corporations')
    ->where('country', 'US')
    ->orderBy('location->state')
    ->get();
```


<a name="latest-oldest"></a>
#### `latest` 與 `oldest` 方法

`latest` 和 `oldest` 方法讓您可以輕鬆地按日期對結果進行排序。預設情況下，結果將按資料表的 `created_at` 欄位進行排序。或者，您也可以傳入希望排序的欄位名稱：

```php
$user = DB::table('users')
    ->latest()
    ->first();
```


<a name="random-ordering"></a>
#### 隨機排序

`inRandomOrder` 方法可用於將查詢結果進行隨機排序。例如，您可以使用此方法獲取隨機的使用者：

```php
$randomUser = DB::table('users')
    ->inRandomOrder()
    ->first();
```


<a name="removing-existing-orderings"></a>
#### 移除現有的排序

`reorder` 方法會移除先前已套用到查詢的所有 "order by" 子句：

```php
$query = DB::table('users')->orderBy('name');

$unorderedUsers = $query->reorder()->get();
```

呼叫 `reorder` 方法時，您可以傳入欄位與排序方向，以便移除所有現有的 "order by" 子句並向查詢套用全新的排序：

```php
$query = DB::table('users')->orderBy('name');

$usersOrderedByEmail = $query->reorder('email', 'desc')->get();
```

為了方便起見，您可以使用 `reorderDesc` 方法將查詢結果重新以降冪排序：

```php
$query = DB::table('users')->orderBy('name');

$usersOrderedByEmail = $query->reorderDesc('email')->get();
```


<a name="grouping"></a>
### 分組


<a name="groupby-having"></a>
#### `groupBy` 與 `having` 方法

正如您所預期的，`groupBy` 和 `having` 方法可用於對查詢結果進行分組。`having` 方法的簽名與 `where` 方法類似：

```php
$users = DB::table('users')
    ->groupBy('account_id')
    ->having('account_id', '>', 100)
    ->get();
```

您可以使用 `havingBetween` 方法來篩選特定範圍內的結果：

```php
$report = DB::table('orders')
    ->selectRaw('count(id) as number_of_orders, customer_id')
    ->groupBy('customer_id')
    ->havingBetween('number_of_orders', [5, 15])
    ->get();
```

您可以傳遞多個引數給 `groupBy` 方法，以依多個欄位進行分組：

```php
$users = DB::table('users')
    ->groupBy('first_name', 'status')
    ->having('account_id', '>', 100)
    ->get();
```

若要建構更進階的 `having` 陳述式，請參考 [havingRaw](#raw-methods) 方法。


<a name="limit-and-offset"></a>
### Limit 與 Offset

您可以使用 `limit` 和 `offset` 方法來限制查詢返回的結果數量，或在查詢中跳過給定數量的結果：

```php
$users = DB::table('users')
    ->offset(10)
    ->limit(5)
    ->get();
```


<a name="conditional-clauses"></a>
## 條件子句

有時您可能希望某些查詢子句僅在滿足另一個條件時才套用到查詢上。例如，您可能只想在傳入的 HTTP 請求中存在給定的輸入值時，才套用 `where` 陳述式。您可以透過 `when` 方法來實現此目的：

```php
$role = $request->input('role');

$users = DB::table('users')
    ->when($role, function (Builder $query, string $role) {
        $query->where('role_id', $role);
    })
    ->get();
```

`when` 方法僅會在第一個引數為 `true` 時執行給定的閉包。如果第一個引數為 `false`，則不會執行該閉包。因此，在上面的範例中，傳給 `when` 方法的閉包只有在傳入的請求中存在 `role` 欄位且求值為 `true` 時才會被呼叫。

您可以傳遞另一個閉包作為 `when` 方法的第三個引數。這個閉包只有在第一個引數求值為 `false` 時才會執行。為了說明如何使用此功能，我們將用它來設定查詢的預設排序：

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
## Insert 陳述式

查詢生成器還提供了 `insert` 方法，可用於將紀錄插入資料庫資料表中。`insert` 方法接受欄位名稱與值的陣列：

```php
DB::table('users')->insert([
    'email' => 'kayla@example.com',
    'votes' => 0
]);
```

您可以透過傳入陣列的陣列來一次插入多筆紀錄。每個陣列代表應該插入資料表中的一筆紀錄：

```php
DB::table('users')->insert([
    ['email' => 'picard@example.com', 'votes' => 0],
    ['email' => 'janeway@example.com', 'votes' => 0],
]);
```

`insertOrIgnore` 方法會在將紀錄插入資料庫時忽略錯誤。使用此方法時，您應該注意重複紀錄的錯誤將被忽略，並且根據資料庫引擎的不同，其他類型的錯誤也可能會被忽略。例如，`insertOrIgnore` 將會[繞過 MySQL 的嚴格模式](https://dev.mysql.com/doc/refman/en/sql-mode.html#ignore-effect-on-execution)：

```php
DB::table('users')->insertOrIgnore([
    ['id' => 1, 'email' => 'sisko@example.com'],
    ['id' => 2, 'email' => 'archer@example.com'],
]);
```

`insertUsing` 方法會在將新紀錄插入資料表時，使用子查詢來決定應插入的資料：

```php
DB::table('pruned_users')->insertUsing([
    'id', 'name', 'email', 'email_verified_at'
], DB::table('users')->select(
    'id', 'name', 'email', 'email_verified_at'
)->where('updated_at', '<=', now()->minus(months: 1)));
```

<a name="auto-incrementing-ids"></a>
#### 自動遞增 ID

如果資料表有一個自動遞增的 id，請使用 `insertGetId` 方法來插入紀錄並取得該 ID：

```php
$id = DB::table('users')->insertGetId(
    ['email' => 'john@example.com', 'votes' => 0]
);
```

> [!WARNING]
> 使用 PostgreSQL 時，`insertGetId` 方法預設自動遞增欄位名稱為 `id`。如果您想從不同的「序列 (sequence)」取得 ID，可以將欄位名稱作為第二個參數傳給 `insertGetId` 方法。

<a name="upserts"></a>
### Upsert (更新或新增)

`upsert` 方法會插入不存在的紀錄，並使用您指定的全新數值更新已經存在的紀錄。該方法的第一個引數包含要插入或更新的值，而第二個引數則列出用於唯一識別相關資料表中紀錄的欄位。該方法的第三個也是最後一個引數是一個欄位陣列，當資料庫中已存在匹配的紀錄時，這些欄位應該被更新：

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

在上例中，Laravel 將嘗試插入兩筆紀錄。如果已存在具有相同 `departure` 和 `destination` 欄位值的紀錄，Laravel 將更新該紀錄的 `price` 欄位。

> [!WARNING]
> 除 SQL Server 外的所有資料庫，都要求 `upsert` 方法第二個引數中的欄位必須具有「主鍵 (primary)」或「唯一 (unique)」索引。此外，MariaDB 和 MySQL 資料庫驅動程式會忽略 `upsert` 方法的第二個引數，且總是使用資料表的主鍵和唯一索引來檢測現有紀錄。

<a name="update-statements"></a>
## Update 陳述式

除了將紀錄插入資料庫之外，查詢生成器還可以使用 `update` 方法更新現有紀錄。`update` 方法與 `insert` 方法一樣，接受欄位與值成對組成的陣列，指定要更新的欄位。`update` 方法會回傳受影響的資料列筆數。您可以使用 `where` 子句來限制 `update` 查詢：

```php
$affected = DB::table('users')
    ->where('id', 1)
    ->update(['votes' => 1]);
```

<a name="update-or-insert"></a>
#### Update 或 Insert

有時候您可能想要更新資料庫中的現有紀錄，或者在不存在匹配紀錄時建立該紀錄。在此情境下，可以使用 `updateOrInsert` 方法。`updateOrInsert` 方法接受兩個引數：用於搜尋紀錄的條件陣列，以及指定要更新欄位的欄位與值成對組成的陣列。

`updateOrInsert` 方法將嘗試使用第一個引數的欄位和值成對資料來尋找匹配的資料庫紀錄。如果紀錄存在，將使用第二個引數中的值來更新它。如果找不到紀錄，將插入一筆合併了兩個引數屬性的新紀錄：

```php
DB::table('users')
    ->updateOrInsert(
        ['email' => 'john@example.com', 'name' => 'John'],
        ['votes' => '2']
    );
```

您可以向 `updateOrInsert` 方法提供一個閉包，以根據是否存在匹配紀錄來自訂更新或插入資料庫的屬性：

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

更新 JSON 欄位時，您應該使用 `->` 語法來更新 JSON 物件中對應的鍵 (Key)。此操作在 MariaDB 10.3+、MySQL 5.7+ 和 PostgreSQL 9.5+ 上均有支援：

```php
$affected = DB::table('users')
    ->where('id', 1)
    ->update(['options->enabled' => true]);
```

<a name="increment-and-decrement"></a>
### 遞增與遞減

查詢生成器還提供了方便的方法來遞增或遞減指定欄位的值。這兩種方法都至少接受一個引數：要修改的欄位。可以提供第二個引數來指定該欄位應遞增或遞減的數值：

```php
DB::table('users')->increment('votes');

DB::table('users')->increment('votes', 5);

DB::table('users')->decrement('votes');

DB::table('users')->decrement('votes', 5);
```

如果需要，您還可以在遞增或遞減操作期間指定要更新的其他欄位：

```php
DB::table('users')->increment('votes', 1, ['name' => 'John']);
```

此外，您可以使用 `incrementEach` 和 `decrementEach` 方法一次遞增或遞減多個欄位：

```php
DB::table('users')->incrementEach([
    'votes' => 5,
    'balance' => 100,
]);
```

<a name="delete-statements"></a>
## Delete 陳述式

查詢生成器的 `delete` 方法可用於從資料表中刪除紀錄。`delete` 方法會回傳受影響的筆數。您可以在呼叫 `delete` 方法之前透過新增 "where" 子句來限制 `delete` 陳述式：

```php
$deleted = DB::table('users')->delete();

$deleted = DB::table('users')->where('votes', '>', 100)->delete();
```

<a name="pessimistic-locking"></a>
## 悲觀鎖定

查詢生成器也包含了一些函式，可協助您在執行 `select` 陳述式時實現「悲觀鎖定 (pessimistic locking)」。若要執行帶有「共享鎖定 (shared lock)」的陳述式，您可以呼叫 `sharedLock` 方法。共享鎖定可防止選取的資料列在您的交易 (transaction) 提交之前被修改：

```php
DB::table('users')
    ->where('votes', '>', 100)
    ->sharedLock()
    ->get();
```

或者，您可以使用 `lockForUpdate` 方法。「for update」鎖定可防止選取的紀錄被修改或被另一個共享鎖定所選取：

```php
DB::table('users')
    ->where('votes', '>', 100)
    ->lockForUpdate()
    ->get();
```

雖然不是強制性的，但建議將悲觀鎖定包裹在[交易](/docs/{{version}}/database#database-transactions)之中。這能確保檢索到的資料在整個操作完成之前，在資料庫中保持不變。萬一失敗，交易會自動復原任何變更並釋放鎖定：

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
## 可重複使用的查詢元件

如果在您的應用程式中有重複的查詢邏輯，您可以使用查詢生成器的 `tap` 和 `pipe` 方法將邏輯抽離成可重複使用的物件。想像一下在您的應用程式中有這兩個不同的查詢：

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

您可能會想要將這些查詢之間共有的目的地篩選邏輯抽離成一個可重複使用的物件：

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

接著，您可以使用查詢生成器的 `tap` 方法將該物件的邏輯套用到查詢中：

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

`tap` 方法總是會回傳查詢生成器。如果您想要抽離一個執行查詢並回傳另一個值的物件，您可以改用 `pipe` 方法。

考慮以下包含整個應用程式中共用的[分頁](/docs/{{version}}/pagination)邏輯之查詢物件。與將查詢條件套用到查詢的 `DestinationFilter` 不同，`Paginate` 物件會執行查詢並回傳分頁器 (paginator) 實例：

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

使用查詢生成器的 `pipe` 方法，我們可以利用此物件來套用共用的分頁邏輯：

```php
$flights = DB::table('flights')
    ->tap(new DestinationFilter($destination))
    ->pipe(new Paginate);
```


<a name="debugging"></a>
## 偵錯

您可以在建構查詢時使用 `dd` 和 `dump` 方法來傾印 (dump) 當前的查詢綁定與 SQL。`dd` 方法會顯示偵錯資訊並停止執行請求。`dump` 方法則會顯示偵錯資訊，但允許請求繼續執行：

```php
DB::table('users')->where('votes', '>', 100)->dd();

DB::table('users')->where('votes', '>', 100)->dump();
```

您可以在查詢上呼叫 `dumpRawSql` 和 `ddRawSql` 方法，以傾印已將所有參數綁定正確替換後的查詢 SQL：

```php
DB::table('users')->where('votes', '>', 100)->dumpRawSql();

DB::table('users')->where('votes', '>', 100)->ddRawSql();
```