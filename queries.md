# 資料庫：查詢生成器

- [簡介](#introduction)
- [執行資料庫查詢](#running-database-queries)
    - [結果分塊](#chunking-results)
    - [惰性串流結果](#streaming-results-lazily)
    - [聚合函數](#aggregates)
- [Select 陳述式](#select-statements)
- [原生運算式](#raw-expressions)
- [Join](#joins)
- [Union](#unions)
- [基本 Where 子句](#basic-where-clauses)
    - [Where 子句](#where-clauses)
    - [Or Where 子句](#or-where-clauses)
    - [Where Not 子句](#where-not-clauses)
    - [Where Any / All / None 子句](#where-any-all-none-clauses)
    - [JSON Where 子句](#json-where-clauses)
    - [額外的 Where 子句](#additional-where-clauses)
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
    - [Upsert](#upserts)
- [Update 陳述式](#update-statements)
    - [更新 JSON 欄位](#updating-json-columns)
    - [遞增與遞減](#increment-and-decrement)
- [Delete 陳述式](#delete-statements)
- [悲觀鎖定](#pessimistic-locking)
- [可重複使用的查詢元件](#reusable-query-components)
- [除錯](#debugging)

<a name="introduction"></a>
## 簡介

Laravel 的資料庫查詢生成器提供了一個方便且流暢的介面，用於建立與執行資料庫查詢。它可以用來執行應用程式中的大多數資料庫操作，並且完美支援 Laravel 所支援的所有資料庫系統。

Laravel 查詢生成器使用 PDO 參數綁定來保護您的應用程式免受 SQL 注入攻擊。作為查詢綁定傳遞給查詢生成器的字串，完全不需要先進行清理或淨化處理。

> [!WARNING]
> PDO 不支援綁定欄位名稱。因此，您絕不應該讓使用者輸入來決定查詢中所參考的欄位名稱，包括 "order by" 欄位。

<a name="running-database-queries"></a>
## 執行資料庫查詢

<a name="retrieving-all-rows-from-a-table"></a>
#### 從資料表中取得所有資料列

您可以使用 `DB` Facade 提供的 `table` 方法來開始建立查詢。`table` 方法會針對指定的資料表傳回一個流暢的查詢生成器實例，讓您可以在查詢上鏈結更多約束條件，最後再使用 `get` 方法取得查詢結果：

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

`get` 方法會傳回一個包含查詢結果的 `Illuminate\Support\Collection` 實例，其中每個結果都是 PHP `stdClass` 物件的實例。您可以透過存取物件的屬性來存取每個欄位的值：

```php
use Illuminate\Support\Facades\DB;

$users = DB::table('users')->get();

foreach ($users as $user) {
    echo $user->name;
}
```

> [!NOTE]
> Laravel Collection 提供了許多非常強大的方法來對資料進行映射與縮減。如需更多關於 Laravel Collection 的資訊，請參考 [Collection 說明文件](/docs/{{version}}/collections)。

<a name="retrieving-a-single-row-column-from-a-table"></a>
#### 從資料表中取得單一資料列 / 欄位

如果您只需要從資料表中取得單一資料列，可以使用 `DB` Facade 的 `first` 方法。此方法將傳回單一 `stdClass` 物件：

```php
$user = DB::table('users')->where('name', 'John')->first();

return $user->email;
```

如果您想要從資料表中取得單一資料列，但若找不到符合的資料列時拋出 `Illuminate\Database\RecordNotFoundException`，可以使用 `firstOrFail` 方法。如果 `RecordNotFoundException` 未被擷取，系統會自動將 404 HTTP 回應傳回給用戶端：

```php
$user = DB::table('users')->where('name', 'John')->firstOrFail();
```

如果您不需要整行資料列，可以使用 `value` 方法從紀錄中取出單一值。此方法將直接傳回該欄位的值：

```php
$email = DB::table('users')->where('name', 'John')->value('email');
```

若要透過 `id` 欄位的值取得單一資料列，請使用 `find` 方法：

```php
$user = DB::table('users')->find(3);
```

<a name="retrieving-a-list-of-column-values"></a>
#### 取得單一欄位的值清單

如果您想要取得包含單一欄位所有值的 `Illuminate\Support\Collection` 實例，可以使用 `pluck` 方法。在本例中，我們將取得使用者頭銜的集合：

```php
use Illuminate\Support\Facades\DB;

$titles = DB::table('users')->pluck('title');

foreach ($titles as $title) {
    echo $title;
}
```

您可以傳入第二個引數給 `pluck` 方法，以指定產生的 Collection 應使用哪個欄位作為其鍵值：

```php
$titles = DB::table('users')->pluck('title', 'name');

foreach ($titles as $name => $title) {
    echo $title;
}
```

<a name="chunking-results"></a>
### 結果分塊

如果您需要處理數千條資料庫紀錄，可以考慮使用 `DB` Facade 提供的 `chunk` 方法。此方法一次只取得一小塊結果，並將每個區塊傳入閉包 (Closure) 進行處理。例如，讓我們以一次 100 條紀錄的方式，分塊取得整個 `users` 資料表：

```php
use Illuminate\Support\Collection;
use Illuminate\Support\Facades\DB;

DB::table('users')->orderBy('id')->chunk(100, function (Collection $users) {
    foreach ($users as $user) {
        // ...
    }
});
```

您可以透過從閉包傳回 `false` 來停止繼續處理後續的分塊：

```php
DB::table('users')->orderBy('id')->chunk(100, function (Collection $users) {
    // Process the records...

    return false;
});
```

如果您在對結果進行分塊時更新資料庫紀錄，分塊結果可能會以未預期的形式改變。如果您預計在分塊處理時更新取得的紀錄，最好改用 `chunkById` 方法。該方法會自動根據紀錄的主鍵對結果進行分頁：

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

由於 `chunkById` 與 `lazyById` 方法會為執行的查詢加入自己的 "where" 條件，因此您通常應該在閉包內對您自己的條件進行[邏輯分組](#logical-grouping)：

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
> 在分塊回呼函式內更新或刪除紀錄時，對主鍵或外鍵的任何變更都可能影響分塊查詢。這可能會導致某些紀錄未包含在分塊結果中。

<a name="streaming-results-lazily"></a>
### 惰性串流結果

`lazy` 方法運作方式與[分塊方法](#chunking-results)類似，都是以分塊方式執行查詢。然而，`lazy()` 方法不會將每個分塊傳入回呼函式，而是傳回一個 [LazyCollection](/docs/{{version}}/collections#lazy-collections)，讓您可以將結果作為單一串流進行互動：

```php
use Illuminate\Support\Facades\DB;

DB::table('users')->orderBy('id')->lazy()->each(function (object $user) {
    // ...
});
```

同樣地，如果您打算在迭代時更新取得的紀錄，最好改用 `lazyById` 或 `lazyByIdDesc` 方法。這些方法會自動根據紀錄的主鍵對結果進行分頁：

```php
DB::table('users')->where('active', false)
    ->lazyById()->each(function (object $user) {
        DB::table('users')
            ->where('id', $user->id)
            ->update(['active' => true]);
    });
```

> [!WARNING]
> 在迭代紀錄的同時進行更新或刪除時，對主鍵或外鍵的任何變更都可能影響分塊查詢。這可能會導致某些紀錄未包含在結果中。

<a name="aggregates"></a>
### 聚合函數

查詢生成器還提供了各種用於取得聚合值的方法，例如 `count`、`max`、`min`、`avg` 和 `sum`。您可以在建立查詢後呼叫其中任何方法：

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
#### 確認紀錄是否存在

除了使用 `count` 方法來判斷是否有符合查詢限制的紀錄外，您還可以使用 `exists` 和 `doesntExist` 方法：

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

您可能並不總是想從資料庫資料表中選取所有欄位。使用 `select` 方法，您可以為查詢指定自訂的「select」子句：

```php
use Illuminate\Support\Facades\DB;

$users = DB::table('users')
    ->select('name', 'email as user_email')
    ->get();
```

`distinct` 方法允許您強制查詢傳回不重複的結果：

```php
$users = DB::table('users')->distinct()->get();
```

若您已經有一個查詢生成器實例，且希望在其現有的 select 子句中新增欄位，可以使用 `addSelect` 方法：

```php
$query = DB::table('users')->select('name');

$users = $query->addSelect('age')->get();
```


<a name="raw-expressions"></a>
## 原生運算式

有時候您可能需要將任意字串插入查詢中。若要建立原生的字串運算式，可以使用 `DB` Facade 所提供的 `raw` 方法：

```php
$users = DB::table('users')
    ->select(DB::raw('count(*) as user_count, status'))
    ->where('status', '<>', 1)
    ->groupBy('status')
    ->get();
```

> [!WARNING]
> 原生陳述式會以字串形式注入到查詢中，因此您應該非常小心，避免產生 SQL 注入漏洞。


<a name="raw-methods"></a>
### 原生方法

除了使用 `DB::raw` 方法之外，您還可以使用以下方法將原生運算式插入查詢的不同部分。**請記住，Laravel 無法保證任何使用原生運算式的查詢都能防止 SQL 注入漏洞。**


<a name="selectraw"></a>
#### `selectRaw`

`selectRaw` 方法可以用來替代 `addSelect(DB::raw(/* ... */))`。此方法接受可選的綁定陣列作為其第二個引數：

```php
$orders = DB::table('orders')
    ->selectRaw('price * ? as price_with_tax', [1.0825])
    ->get();
```


<a name="whereraw-orwhereraw"></a>
#### `whereRaw / orWhereRaw`

`whereRaw` 和 `orWhereRaw` 方法可用於將原生的「where」子句注入到您的查詢中。這些方法接受可選的綁定陣列作為其第二個引數：

```php
$orders = DB::table('orders')
    ->whereRaw('price > IF(state = "TX", ?, 100)', [200])
    ->get();
```


<a name="havingraw-orhavingraw"></a>
#### `havingRaw / orHavingRaw`

`havingRaw` 和 `orHavingRaw` 方法可用於提供原生字串作為「having」子句的值。這些方法接受可選的綁定陣列作為其第二個引數：

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
### `groupByRaw`

`groupByRaw` 方法可用於提供原生字串作為 `group by` 子句的值：

```php
$orders = DB::table('orders')
    ->select('city', 'state')
    ->groupByRaw('city, state')
    ->get();
```


<a name="joins"></a>
## Join


<a name="inner-join-clause"></a>
#### Inner Join 子句

查詢生成器也可以用來在您的查詢中新增 join 子句。若要執行基本的「inner join」，可以在查詢生成器實例上使用 `join` 方法。傳遞給 `join` 方法的第一個引數是您需要連接的資料表名稱，而其餘引數則指定 join 的欄位條件約束。您甚至可以在單一查詢中連接多個資料表：

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

如果您想執行「left join」或「right join」而非「inner join」，可以使用 `leftJoin` 或 `rightJoin` 方法。這些方法與 `join` 方法具有相同的簽名（Signature）：

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

您可以使用 `crossJoin` 方法來執行「cross join」。Cross join 會在第一個資料表與連接的資料表之間產生笛卡兒積（Cartesian Product）：

```php
$sizes = DB::table('sizes')
    ->crossJoin('colors')
    ->get();
```


<a name="advanced-join-clauses"></a>
#### 進階 Join 子句

您也可以指定更進階的 join 子句。首先，將一個閉包（Closure）作為第二個引數傳遞給 `join` 方法。該閉包將接收一個 `Illuminate\Database\Query\JoinClause` 實例，允許您在「join」子句上指定條件約束：

```php
DB::table('users')
    ->join('contacts', function (JoinClause $join) {
        $join->on('users.id', '=', 'contacts.user_id')->orOn(/* ... */);
    })
    ->get();
```

如果您想在 join 上使用「where」子句，可以使用 `JoinClause` 實例提供的 `where` 和 `orWhere` 方法。這些方法不是比較兩個欄位，而是將欄位與一個值進行比較：

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

您可以使用 `joinSub`、`leftJoinSub` 和 `rightJoinSub` 方法將查詢與子查詢進行連接。這些方法中的每一個都接收三個引數：子查詢、其資料表別名，以及定義相關欄位的閉包。在這個範例中，我們將檢索使用者集合，其中每個使用者記錄還包含該使用者最新發布的部落格文章的 `created_at` 時間戳記：

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
> Lateral join 目前支援 PostgreSQL、MySQL >= 8.0.14 以及 SQL Server。

您可以使用 `joinLateral` 和 `leftJoinLateral` 方法對子查詢執行「lateral join」。這些方法中的每一個都接收兩個引數：子查詢及其資料表別名。Join 條件應在給定子查詢的 `where` 子句中指定。Lateral join 會針對每一列進行評估，且可以引用子查詢外部的欄位。

在這個範例中，我們將檢索使用者集合以及該使用者最新的三篇部落格文章。每個使用者在結果集中最多可產生三列：分別對應其最新發布的每一篇文章。Join 條件是在子查詢內部使用 `whereColumn` 子句指定的，並引用當前使用者列：

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
## Union

查詢生成器也提供了一個方便的方法來將兩個或多個查詢「Union（聯合）」在一起。例如，您可以建立一個初始查詢，並使用 `union` 方法將其與其他查詢進行聯合：

```php
use Illuminate\Support\Facades\DB;

$usersWithoutFirstName = DB::table('users')
    ->whereNull('first_name');

$users = DB::table('users')
    ->whereNull('last_name')
    ->union($usersWithoutFirstName)
    ->get();
```

除了 `union` 方法之外，查詢生成器還提供了 `unionAll` 方法。使用 `unionAll` 方法結合的查詢不會移除其重複的結果。`unionAll` 方法擁有與 `union` 方法相同的方法簽名。

<a name="basic-where-clauses"></a>
## 基本 Where 子句


<a name="where-clauses"></a>
### Where 子句

您可以使用查詢生成器的 `where` 方法在查詢中加入「where」子句。對 `where` 方法的最基本呼叫需要三個引數。第一個引數是欄位名稱。第二個引數是運算子，可以是資料庫支援的任何運算子。第三個引數是要與欄位值進行比較的數值。

例如，以下查詢會取得 `votes` 欄位值等於 `100` 且 `age` 欄位值大於 `35` 的使用者：

```php
$users = DB::table('users')
    ->where('votes', '=', 100)
    ->where('age', '>', 35)
    ->get();
```

為了方便起見，如果您想驗證欄位是否等於 (`=`) 給定的值，可以將該值作為第二個引數傳遞給 `where` 方法。Laravel 會自動假設您想要使用 `=` 運算子：

```php
$users = DB::table('users')->where('votes', 100)->get();
```

您也可以向 `where` 方法提供一個關聯陣列，以快速對多個欄位進行查詢：

```php
$users = DB::table('users')->where([
    'first_name' => 'Jane',
    'last_name' => 'Doe',
])->get();
```

如前所述，您可以使用的任何資料庫系統支援的運算子：

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

您也可以傳遞一個條件陣列給 `where` 函式。陣列中的每個元素都應該是一個包含通常傳遞給 `where` 方法的三個引數的陣列：

```php
$users = DB::table('users')->where([
    ['status', '=', '1'],
    ['subscribed', '<>', '1'],
])->get();
```

> [!WARNING]
> PDO 不支援綁定欄位名稱。因此，您絕不應該允許使用者輸入來決定查詢所引用的欄位名稱，包括「order by」欄位。

> [!WARNING]
> MySQL 與 MariaDB 在字串與數字進行比較時，會自動將字串型別轉換為整數。在此過程中，非數字字串會被轉換為 `0`，這可能會導致意外的結果。例如，若您的資料表中有一個 `secret` 欄位值為 `aaa`，而您執行了 `User::where('secret', 0)`，該資料列將會被回傳。為了避免這種情況，請確保所有數值在用於查詢之前都已型別轉換為適當的型別。


<a name="or-where-clauses"></a>
### Or Where 子句

當鏈結呼叫查詢生成器的 `where` 方法時，「where」子句將會使用 `and` 運算子組合在一起。不過，您可以使用 `orWhere` 方法，透過 `or` 運算子將子句附加到查詢中。`orWhere` 方法接受與 `where` 方法相同的引數：

```php
$users = DB::table('users')
    ->where('votes', '>', 100)
    ->orWhere('name', 'John')
    ->get();
```

如果您需要在括號內對「or」條件進行分組，可以將閉包作為第一個引數傳遞給 `orWhere` 方法：

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
> 您應該總是對 `orWhere` 呼叫進行分組，以避免在套用全域作用域 (Global Scopes) 時產生未預期的行為。


<a name="where-not-clauses"></a>
### Where Not 子句

`whereNot` 與 `orWhereNot` 方法可用於否定給定的一組查詢限制條件。例如，以下查詢排除了正在出清或價格小於 10 的產品：

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

有時您可能需要將相同的查詢限制條件套用到多個欄位。例如，您可能想要取得給定清單中的任何欄位為 `LIKE` 給定數值的所有紀錄。您可以使用 `whereAny` 方法來達成此目的：

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

同樣地，`whereAll` 方法可用於取得所有給定欄位皆符合給定限制條件的紀錄：

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

`whereNone` 方法可用於取得給定欄位皆不符合給定限制條件的紀錄：

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

Laravel 也支援在提供 JSON 欄位型別支援的資料庫上查詢 JSON 欄位型別。目前，這包括 MariaDB 10.3+、MySQL 8.0+、PostgreSQL 12.0+、SQL Server 2017+ 以及 SQLite 3.39.0+。若要查詢 JSON 欄位，請使用 `->` 運算子：

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

如果您的應用程式使用 MariaDB、MySQL 或 PostgreSQL 資料庫，您可以向 `whereJsonContains` 與 `whereJsonDoesntContain` 方法傳遞一個數值陣列：

```php
$users = DB::table('users')
    ->whereJsonContains('options->languages', ['en', 'de'])
    ->get();

$users = DB::table('users')
    ->whereJsonDoesntContain('options->languages', ['en', 'de'])
    ->get();
```

此外，您可以使用 `whereJsonContainsKey` 或 `whereJsonDoesntContainKey` 方法來取得包含或不包含某個 JSON 鍵的結果：

```php
$users = DB::table('users')
    ->whereJsonContainsKey('preferences->dietary_requirements')
    ->get();

$users = DB::table('users')
    ->whereJsonDoesntContainKey('preferences->dietary_requirements')
    ->get();
```

最後，您可以使用 `whereJsonLength` 方法根據 JSON 陣列長度進行查詢：

```php
$users = DB::table('users')
    ->whereJsonLength('options->languages', 0)
    ->get();

$users = DB::table('users')
    ->whereJsonLength('options->languages', '>', 1)
    ->get();
```

<a name="additional-where-clauses"></a>
### 額外的 Where 子句

**whereLike / orWhereLike / whereNotLike / orWhereNotLike**

`whereLike` 方法允許你在查詢中加入 "LIKE" 子句以進行模式比對。這些方法提供了一種獨立於資料庫的方式來執行字串比對查詢，並支援切換是否區分大小寫。預設情況下，字串比對是不區分大小寫的：

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

`orWhereLike` 方法允許你加入帶有 LIKE 條件的 "or" 子句：

```php
$users = DB::table('users')
    ->where('votes', '>', 100)
    ->orWhereLike('name', '%John%')
    ->get();
```

`whereNotLike` 方法允許你在查詢中加入 "NOT LIKE" 子句：

```php
$users = DB::table('users')
    ->whereNotLike('name', '%John%')
    ->get();
```

同樣地，你可以使用 `orWhereNotLike` 來加入帶有 NOT LIKE 條件的 "or" 子句：

```php
$users = DB::table('users')
    ->where('votes', '>', 100)
    ->orWhereNotLike('name', '%John%')
    ->get();
```

> [!WARNING]
> SQL Server 目前不支援 `whereLike` 的區分大小寫搜尋選項。

**whereIn / whereNotIn / orWhereIn / orWhereNotIn**

`whereIn` 方法驗證指定欄位的值是否包含在給定的陣列中：

```php
$users = DB::table('users')
    ->whereIn('id', [1, 2, 3])
    ->get();
```

`whereNotIn` 方法驗證指定欄位的值是否不包含在給定的陣列中：

```php
$users = DB::table('users')
    ->whereNotIn('id', [1, 2, 3])
    ->get();
```

你也可以將查詢物件作為 `whereIn` 方法的第二個引數：

```php
$activeUsers = DB::table('users')->select('id')->where('is_active', 1);

$comments = DB::table('comments')
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
> 如果你要向查詢中加入大量整數綁定的陣列，可以使用 `whereIntegerInRaw` 或 `whereIntegerNotInRaw` 方法來大幅減少記憶體使用量。

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

`whereBetweenColumns` 方法驗證欄位的值是否介於同一資料表列中兩個欄位的值之間：

```php
$patients = DB::table('patients')
    ->whereBetweenColumns('weight', ['minimum_allowed_weight', 'maximum_allowed_weight'])
    ->get();
```

`whereNotBetweenColumns` 方法驗證欄位的值是否落在同一資料表列中兩個欄位的值之外：

```php
$patients = DB::table('patients')
    ->whereNotBetweenColumns('weight', ['minimum_allowed_weight', 'maximum_allowed_weight'])
    ->get();
```

**whereValueBetween / whereValueNotBetween / orWhereValueBetween / orWhereValueNotBetween**

`whereValueBetween` 方法驗證給定值是否介於同一資料表列中相同型別的兩個欄位值之間：

```php
$products = DB::table('products')
    ->whereValueBetween(100, ['min_price', 'max_price'])
    ->get();
```

`whereValueNotBetween` 方法驗證給定值是否落在同一資料表列中兩個欄位的值之外：

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

`whereNullSafeEquals` 與 `orWhereNullSafeEquals` 方法可用於將欄位的值與給定值進行比較，同時將兩個 `NULL` 值視為相等：

```php
$lastLoginIp = $request->input('last_login_ip');

$users = DB::table('users')
    ->whereNullSafeEquals('last_login_ip', $lastLoginIp)
    ->get();
```

**whereDate / whereMonth / whereDay / whereYear / whereTime**

`whereDate` 方法可以用來將欄位的值與指定日期進行比較：

```php
$users = DB::table('users')
    ->whereDate('created_at', '2016-12-31')
    ->get();
```

`whereMonth` 方法可以用來將欄位的值與特定月份進行比較：

```php
$users = DB::table('users')
    ->whereMonth('created_at', '12')
    ->get();
```

`whereDay` 方法可以用來將欄位的值與當月的特定日期進行比較：

```php
$users = DB::table('users')
    ->whereDay('created_at', '31')
    ->get();
```

`whereYear` 方法可以用來將欄位的值與特定年份進行比較：

```php
$users = DB::table('users')
    ->whereYear('created_at', '2016')
    ->get();
```

`whereTime` 方法可以用來將欄位的值與特定時間進行比較：

```php
$users = DB::table('users')
    ->whereTime('created_at', '=', '11:20:45')
    ->get();
```

**wherePast / whereFuture / whereToday / whereBeforeToday / whereAfterToday**

`wherePast` 與 `whereFuture` 方法可以用來判斷欄位的值是否在過去或未來：

```php
$invoices = DB::table('invoices')
    ->wherePast('due_at')
    ->get();

$invoices = DB::table('invoices')
    ->whereFuture('due_at')
    ->get();
```

`whereNowOrPast` 與 `whereNowOrFuture` 方法可以用來判斷欄位的值是否在過去或未來（包含當前的日期與時間）：

```php
$invoices = DB::table('invoices')
    ->whereNowOrPast('due_at')
    ->get();

$invoices = DB::table('invoices')
    ->whereNowOrFuture('due_at')
    ->get();
```

`whereToday`、`whereBeforeToday` 以及 `whereAfterToday` 方法可以用來分別判斷欄位的值是否為今天、今天之前或今天之後：

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

同樣地，`whereTodayOrBefore` 與 `whereTodayOrAfter` 方法可以用來判斷欄位的值是否在今天之前或今天之後（包含今天的日期）：

```php
$invoices = DB::table('invoices')
    ->whereTodayOrBefore('due_at')
    ->get();

$invoices = DB::table('invoices')
    ->whereTodayOrAfter('due_at')
    ->get();
```

**whereColumn / orWhereColumn**

`whereColumn` 方法可以用來驗證兩個欄位是否相等：

```php
$users = DB::table('users')
    ->whereColumn('first_name', 'last_name')
    ->get();
```

你也可以傳遞比較運算子給 `whereColumn` 方法：

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

有時您可能需要將數個「where」子句用括號括起來，以達成查詢所需的邏輯分組。事實上，通常您應該總是將對 `orWhere` 方法的呼叫用括號進行分組，以避免非預期的查詢行為。若要達成此目的，您可以傳遞一個閉包給 `where` 方法：

```php
$users = DB::table('users')
    ->where('name', '=', 'John')
    ->where(function (Builder $query) {
        $query->where('votes', '>', 100)
            ->orWhere('title', '=', 'Admin');
    })
    ->get();
```

如您所見，傳入閉包給 `where` 方法會指示查詢生成器開始一個條件群組。該閉包會接收一個查詢生成器實例，您可以使用它來設定應包含在括號群組內的條件約束。上述範例將產生以下 SQL：

```sql
select * from users where name = 'John' and (votes > 100 or title = 'Admin')
```

> [!WARNING]
> 您應該總是將 `orWhere` 呼叫進行分組，以避免在套用全域範疇時發生非預期的行為。

<a name="advanced-where-clauses"></a>
## 進階 Where 子句


<a name="where-exists-clauses"></a>
### Where Exists 子句

`whereExists` 方法允許您撰寫 "where exists" SQL 子句。`whereExists` 方法接收一個閉包，該閉包將會接收一個查詢生成器實例，讓您定義應該放在 "exists" 子句內部的查詢：

```php
$users = DB::table('users')
    ->whereExists(function (Builder $query) {
        $query->select(DB::raw(1))
            ->from('orders')
            ->whereColumn('orders.user_id', 'users.id');
    })
    ->get();
```

或者，您可以提供一個查詢物件給 `whereExists` 方法，而不是傳入閉包：

```php
$orders = DB::table('orders')
    ->select(DB::raw(1))
    ->whereColumn('orders.user_id', 'users.id');

$users = DB::table('users')
    ->whereExists($orders)
    ->get();
```

上述兩個範例都會產生以下 SQL：

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

有時您可能需要建構一個將子查詢結果與給定值進行比較的 "where" 子句。您可以透過傳遞一個閉包和一個值給 `where` 方法來實現這一點。例如，以下查詢將取得所有最近擁有指定類型 "membership" 的使用者：

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

或者，您可能需要建構一個將欄位與子查詢結果進行比較的 "where" 子句。您可以透過傳遞欄位、運算子與閉包給 `where` 方法來實現這一點。例如，以下查詢將取得所有金額低於平均值的收入紀錄：

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
> 全文檢索 where 子句目前由 MariaDB、MySQL 和 PostgreSQL 支援。

`whereFullText` 與 `orWhereFullText` 方法可用於向擁有 [全文索引](/docs/{{version}}/migrations#available-index-types) 的欄位新增全文檢索 "where" 子句。這些方法會由 Laravel 轉換為適合底層資料庫系統的 SQL。例如，對於使用 MariaDB 或 MySQL 的應用程式，將會產生 `MATCH AGAINST` 子句：

```php
$users = DB::table('users')
    ->whereFullText('bio', 'web developer')
    ->get();
```


<a name="vector-similarity-clauses"></a>
### 向量相似度子句

> [!NOTE]
> 向量相似度子句目前僅支援使用 `pgvector` 擴充套件的 PostgreSQL 連線。有關定義向量欄位與索引的資訊，請參閱 [遷移文件](/docs/{{version}}/migrations#available-column-types)。

`whereVectorSimilarTo` 方法透過與給定向量的餘弦相似度 (cosine similarity) 來過濾結果，並依相關性對結果進行排序。`minSimilarity` 門檻值應為 `0.0` 到 `1.0` 之間的值，其中 `1.0` 表示完全相同：

```php
$documents = DB::table('documents')
    ->whereVectorSimilarTo('embedding', $queryEmbedding, minSimilarity: 0.4)
    ->limit(10)
    ->get();
```

當給定純字串作為向量引數時，Laravel 會自動使用 [Laravel AI SDK](/docs/{{version}}/ai-sdk#embeddings) 為其產生嵌入向量 (embeddings)：

```php
$documents = DB::table('documents')
    ->whereVectorSimilarTo('embedding', 'Best wineries in Napa Valley')
    ->limit(10)
    ->get();
```

預設情況下，`whereVectorSimilarTo` 也會按距離對結果進行排序（最相似的排在前面）。您可以透過傳遞 `false` 作為 `order` 引數來停用此排序：

```php
$documents = DB::table('documents')
    ->whereVectorSimilarTo('embedding', $queryEmbedding, minSimilarity: 0.4, order: false)
    ->orderBy('created_at', 'desc')
    ->limit(10)
    ->get();
```

如果您需要更多控制權，可以獨立使用 `selectVectorDistance`、`whereVectorDistanceLessThan` 和 `orderByVectorDistance` 方法：

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

`orderBy` 方法允許您根據給定的欄位對查詢結果進行排序。`orderBy` 方法接受的第一個引數應為您希望排序的欄位，而第二個引數則決定排序的方向，可以是 `asc` 或 `desc`：

```php
$users = DB::table('users')
    ->orderBy('name', 'desc')
    ->get();
```

若要根據多個欄位進行排序，您只需依需求多次呼叫 `orderBy` 即可：

```php
$users = DB::table('users')
    ->orderBy('name', 'desc')
    ->orderBy('email', 'asc')
    ->get();
```

排序方向是可選的，預設為升冪。如果您想以降冪排序，可以為 `orderBy` 方法指定第二個參數，或者直接使用 `orderByDesc`：

```php
$users = DB::table('users')
    ->orderByDesc('verified_at')
    ->get();
```

最後，使用 `->` 運算子，也可以根據 JSON 欄位內的值對結果進行排序：

```php
$corporations = DB::table('corporations')
    ->where('country', 'US')
    ->orderBy('location->state')
    ->get();
```

<a name="latest-oldest"></a>
#### `latest` 與 `oldest` 方法

`latest` 與 `oldest` 方法可讓您輕鬆地根據日期對結果進行排序。預設情況下，結果將根據資料表的 `created_at` 欄位進行排序。或者，您也可以傳入希望排序的欄位名稱：

```php
$user = DB::table('users')
    ->latest()
    ->first();
```

<a name="random-ordering"></a>
#### 隨機排序

`inRandomOrder` 方法可用於隨機排序查詢結果。例如，您可以使用此方法來取得隨機使用者：

```php
$randomUser = DB::table('users')
    ->inRandomOrder()
    ->first();
```

<a name="removing-existing-orderings"></a>
#### 移除現有排序

`reorder` 方法會移除先前套用到查詢的所有 "order by" 子句：

```php
$query = DB::table('users')->orderBy('name');

$unorderedUsers = $query->reorder()->get();
```

您可以在呼叫 `reorder` 方法時傳入欄位與方向，以移除所有現有的 "order by" 子句並向查詢套用全新的排序：

```php
$query = DB::table('users')->orderBy('name');

$usersOrderedByEmail = $query->reorder('email', 'desc')->get();
```

為方便起見，您可以使用 `reorderDesc` 方法以降冪方式重新排序查詢結果：

```php
$query = DB::table('users')->orderBy('name');

$usersOrderedByEmail = $query->reorderDesc('email')->get();
```

<a name="grouping"></a>
### 分組

<a name="groupby-having"></a>
#### `groupBy` 與 `having` 方法

正如您所預期的，`groupBy` 與 `having` 方法可用於對查詢結果進行分組。`having` 方法的簽名與 `where` 方法類似：

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

您可以向 `groupBy` 方法傳遞多個引數，以根據多個欄位進行分組：

```php
$users = DB::table('users')
    ->groupBy('first_name', 'status')
    ->having('account_id', '>', 100)
    ->get();
```

若要建構更進階的 `having` 陳述式，請參見 [havingRaw](#raw-methods) 方法。

<a name="limit-and-offset"></a>
### Limit 與 Offset

您可以使用 `limit` 與 `offset` 方法來限制查詢回傳的結果數量，或跳過查詢中指定數量的結果：

```php
$users = DB::table('users')
    ->offset(10)
    ->limit(5)
    ->get();
```

<a name="conditional-clauses"></a>
## 條件子句

有時您可能希望根據另一個條件將特定的查詢子句套用到查詢中。例如，您可能只希望在傳入的 HTTP 請求中存在給定的輸入值時才套用 `where` 陳述式。您可以使用 `when` 方法來實現這一點：

```php
$role = $request->input('role');

$users = DB::table('users')
    ->when($role, function (Builder $query, string $role) {
        $query->where('role_id', $role);
    })
    ->get();
```

`when` 方法僅在第一個引數為 `true` 時才會執行給定的 Closure。如果第一個引數為 `false`，則不會執行該 Closure。因此，在上面的範例中，傳給 `when` 方法的 Closure 只有在傳入的請求中存在 `role` 欄位且評估為 `true` 時才會被呼叫。

您可以傳遞另一個 Closure 作為 `when` 方法的第三個引數。此 Closure 僅在第一個引數評估為 `false` 時才會執行。為了示範如何使用此功能，我們將用它來設定查詢的預設排序：

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

查詢生成器還提供了一個 `insert` 方法，可用於將記錄新增至資料庫資料表中。`insert` 方法接受一個由欄位名稱與值組成的陣列：

```php
DB::table('users')->insert([
    'email' => 'kayla@example.com',
    'votes' => 0
]);
```

您可以透過傳入陣列的陣列來一次新增多筆記錄。每個陣列代表應該新增至資料表中的一筆記錄：

```php
DB::table('users')->insert([
    ['email' => 'picard@example.com', 'votes' => 0],
    ['email' => 'janeway@example.com', 'votes' => 0],
]);
```

`insertOrIgnore` 方法會在將記錄新增至資料庫時忽略錯誤。使用此方法時，您應該注意重複記錄的錯誤將會被忽略，而且根據資料庫引擎的不同，其他類型的錯誤也可能會被忽略。例如，`insertOrIgnore` 會[繞過 MySQL 的嚴格模式](https://dev.mysql.com/doc/refman/en/sql-mode.html#ignore-effect-on-execution)：

```php
DB::table('users')->insertOrIgnore([
    ['id' => 1, 'email' => 'sisko@example.com'],
    ['id' => 2, 'email' => 'archer@example.com'],
]);
```

`insertUsing` 方法會將新記錄新增至資料表中，同時使用子查詢來決定應該新增的資料：

```php
DB::table('pruned_users')->insertUsing([
    'id', 'name', 'email', 'email_verified_at'
], DB::table('users')->select(
    'id', 'name', 'email', 'email_verified_at'
)->where('updated_at', '<=', now()->minus(months: 1)));
```


<a name="auto-incrementing-ids"></a>
#### 自動遞增 ID

如果資料表擁有自動遞增的 id，可以使用 `insertGetId` 方法來新增記錄並取得該 ID：

```php
$id = DB::table('users')->insertGetId(
    ['email' => 'john@example.com', 'votes' => 0]
);
```

> [!WARNING]
> 當使用 PostgreSQL 時，`insertGetId` 方法預期自動遞增欄位名稱為 `id`。如果您想要從不同的「序列 (sequence)」取得 ID，您可以將欄位名稱作為第二個參數傳遞給 `insertGetId` 方法。


<a name="upserts"></a>
### Upsert

`upsert` 方法會新增不存在的記錄，並使用您指定的新值更新已經存在的記錄。該方法的第一個引數包含要新增或更新的值，第二個引數則列出在相關資料表中唯一識別記錄的欄位。該方法的第三個也是最後一個引數是一個欄位陣列，當資料庫中已存在匹配的記錄時，這些欄位應該被更新：

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

在上方的範例中，Laravel 將嘗試新增兩筆記錄。如果已經存在具有相同 `departure` 與 `destination` 欄位值的記錄，Laravel 將會更新該記錄的 `price` 欄位。

> [!WARNING]
> 除 SQL Server 外的所有資料庫，都要求 `upsert` 方法第二個引數中的欄位必須擁有「主鍵 (primary)」或「唯一 (unique)」索引。此外，MariaDB 與 MySQL 的資料庫驅動程式會忽略 `upsert` 方法的第二個引數，並始終使用資料表的主鍵與唯一索引來檢測已存在的記錄。


<a name="update-statements"></a>
## Update 陳述式

除了向資料庫新增記錄之外，查詢生成器還可以使用 `update` 方法更新現有記錄。`update` 方法與 `insert` 方法類似，接受一個包含欄位與值鍵值對的陣列，用以指定要更新的欄位。`update` 方法會傳回受影響的資料筆數。您可以使用 `where` 子句來限制 `update` 查詢：

```php
$affected = DB::table('users')
    ->where('id', 1)
    ->update(['votes' => 1]);
```


<a name="update-or-insert"></a>
#### 更新或新增

有時您可能想要更新資料庫中的現有記錄，或者在沒有匹配記錄時建立它。在此情境下，可以使用 `updateOrInsert` 方法。`updateOrInsert` 方法接受兩個引數：一個用於尋找記錄的條件陣列，以及一個指定要更新欄位的欄位與值鍵值對陣列。

`updateOrInsert` 方法將嘗試使用第一個引數的欄位與值鍵值對來尋找匹配的資料庫記錄。如果記錄存在，將使用第二個引數中的值進行更新。如果找不到記錄，將使用合併兩個引數後的屬性新增一筆新記錄：

```php
DB::table('users')
    ->updateOrInsert(
        ['email' => 'john@example.com', 'name' => 'John'],
        ['votes' => '2']
    );
```

您也可以向 `updateOrInsert` 方法提供一個閉包，以根據是否存在匹配的記錄，自訂要更新或新增至資料庫中的屬性：

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

更新 JSON 欄位時，您應該使用 `->` 語法來更新 JSON 物件中的適當鍵值。此操作支援 MariaDB 10.3+、MySQL 5.7+ 與 PostgreSQL 9.5+：

```php
$affected = DB::table('users')
    ->where('id', 1)
    ->update(['options->enabled' => true]);
```


<a name="increment-and-decrement"></a>
### 遞增與遞減

查詢生成器還提供了方便的方法來遞增或遞減給定欄位的值。這兩個方法都接受至少一個引數：要修改的欄位。可以提供第二個引數來指定欄位應遞增或遞減的數值：

```php
DB::table('users')->increment('votes');

DB::table('users')->increment('votes', 5);

DB::table('users')->decrement('votes');

DB::table('users')->decrement('votes', 5);
```

如果需要，您也可以在遞增或遞減操作期間指定要更新的額外欄位：

```php
DB::table('users')->increment('votes', 1, ['name' => 'John']);
```

此外，您可以使用 `incrementEach` 與 `decrementEach` 方法一次遞增或遞減多個欄位：

```php
DB::table('users')->incrementEach([
    'votes' => 5,
    'balance' => 100,
]);
```


<a name="delete-statements"></a>
## Delete 陳述式

查詢生成器的 `delete` 方法可用於從資料表中刪除記錄。`delete` 方法會傳回受影響的資料筆數。您可以在呼叫 `delete` 方法之前透過加入「where」子句來限制 `delete` 陳述式：

```php
$deleted = DB::table('users')->delete();

$deleted = DB::table('users')->where('votes', '>', 100)->delete();
```

<a name="pessimistic-locking"></a>
## 悲觀鎖定

查詢生成器也包含了一些函式，可協助你在執行 `select` 陳述式時實現「悲觀鎖定（Pessimistic Locking）」。若要執行帶有「共享鎖（Shared lock）」的陳述式，你可以呼叫 `sharedLock` 方法。共享鎖可以防止被選取的資料列在你的交易提交前被修改：

```php
DB::table('users')
    ->where('votes', '>', 100)
    ->sharedLock()
    ->get();
```

或者，你也可以使用 `lockForUpdate` 方法。「For update」鎖可以防止選取的紀錄被修改，或是被另一個共享鎖選取：

```php
DB::table('users')
    ->where('votes', '>', 100)
    ->lockForUpdate()
    ->get();
```

雖然不是強制的，但建議將悲觀鎖定包裹在[交易](/docs/{{version}}/database#database-transactions)中。這能確保檢索到的資料在整個操作完成前於資料庫中保持不變。若發生失敗，交易會自動復原任何變更並釋放鎖定：

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

若你在整個應用程式中有重複的查詢邏輯，可以使用查詢生成器的 `tap` 與 `pipe` 方法將該邏輯抽離成可重複使用的物件。想像一下你的應用程式中有這兩個不同的查詢：

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

你可能會想將這兩個查詢之間共通的目的地篩選邏輯抽離到一個可重複使用的物件中：

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

接著，你可以使用查詢生成器的 `tap` 方法，將該物件的邏輯套用到查詢中：

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

`tap` 方法永遠會傳回查詢生成器。若你想抽離一個會執行查詢並傳回其他值的物件，則可以使用 `pipe` 方法。

思考以下包含跨應用程式共享[分頁](/docs/{{version}}/pagination)邏輯的查詢物件。與將查詢條件套用到查詢上的 `DestinationFilter` 不同，`Paginate` 物件會執行查詢並傳回分頁器實例：

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

利用查詢生成器的 `pipe` 方法，我們可以借助這個物件來套用共享的分頁邏輯：

```php
$flights = DB::table('flights')
    ->tap(new DestinationFilter($destination))
    ->pipe(new Paginate);
```

<a name="debugging"></a>
## 除錯

在建構查詢時，你可以使用 `dd` 和 `dump` 方法來印出當前的查詢綁定與 SQL。`dd` 方法會顯示除錯資訊並停止執行請求；`dump` 方法則會顯示除錯資訊，但允許請求繼續執行：

```php
DB::table('users')->where('votes', '>', 100)->dd();

DB::table('users')->where('votes', '>', 100)->dump();
```

你可以在查詢上呼叫 `dumpRawSql` 和 `ddRawSql` 方法，以印出已將所有參數綁定正確替換後的 SQL 查詢：

```php
DB::table('users')->where('votes', '>', 100)->dumpRawSql();

DB::table('users')->where('votes', '>', 100)->ddRawSql();
```