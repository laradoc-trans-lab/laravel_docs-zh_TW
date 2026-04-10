# 資料庫：查詢產生器 (Query Builder)

- [簡介](#introduction)
- [執行資料庫查詢](#running-database-queries)
    - [分塊查詢結果](#chunking-results)
    - [延遲串流結果](#streaming-results-lazily)
    - [聚合 (Aggregates)](#aggregates)
- [Select 語句](#select-statements)
- [原生運算式 (Raw Expressions)](#raw-expressions)
- [連接 (Joins)](#joins)
- [聯合 (Unions)](#unions)
- [基礎 Where 子句](#basic-where-clauses)
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
- [排序、分組、限制與偏移](#ordering-grouping-limit-and-offset)
    - [排序](#ordering)
    - [分組](#grouping)
    - [限制與偏移](#limit-and-offset)
- [條件子句](#conditional-clauses)
- [Insert 語句](#insert-statements)
    - [更新或插入 (Upserts)](#upserts)
- [Update 語句](#update-statements)
    - [更新 JSON 欄位](#updating-json-columns)
    - [遞增與遞減](#increment-and-decrement)
- [Delete 語句](#delete-statements)
- [悲觀鎖定 (Pessimistic Locking)](#pessimistic-locking)
- [可重複使用的查詢元件](#reusable-query-components)
- [除錯](#debugging)

<a name="introduction"></a>
## 簡介

Laravel 的資料庫查詢產生器提供了一個方便且流暢的介面，用於建立與執行資料庫查詢。它可以用於執行應用程式中的大多數資料庫操作，並能完美地與 Laravel 支援的所有資料庫系統搭配使用。

Laravel 查詢產生器使用 PDO 參數綁定，以保護您的應用程式免受 SQL 插入攻擊。您不需要對傳遞給查詢產生器作為查詢綁定的字串進行清理或過濾。

> [!WARNING]
> PDO 不支援綁定欄位名稱。因此，您絕對不應該讓使用者輸入來決定查詢中參照的欄位名稱，包括 "order by" 欄位。

<a name="running-database-queries"></a>
## 執行資料庫查詢


<a name="retrieving-all-rows-from-a-table"></a>
#### 從資料表中取得所有資料列

您可以使用 `DB` Facade 提供的 `table` 方法來開始查詢。`table` 方法會為指定的資料表回傳一個流暢的查詢產生器實例，讓您可以在查詢中串接更多的約束條件，最後使用 `get` 方法取得查詢結果：

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

`get` 方法會回傳一個包含查詢結果的 `Illuminate\Support\Collection` 實例，其中每個結果都是 PHP `stdClass` 物件的實例。您可以透過存取物件的屬性來取得每個欄位的值：

```php
use Illuminate\Support\Facades\DB;

$users = DB::table('users')->get();

foreach ($users as $user) {
    echo $user->name;
}
```

> [!NOTE]
> Laravel collection 提供了多種極其強大的方法來對資料進行映射 (Mapping) 和縮減 (Reducing)。如需更多關於 Laravel collection 的資訊，請參考 [collection 說明文件](/docs/{{version}}/collections)。


<a name="retrieving-a-single-row-column-from-a-table"></a>
#### 從資料表中取得單一資料列 / 欄位

如果您只需要從資料庫資料表中取得單一資料列，可以使用 `DB` Facade 的 `first` 方法。此方法將回傳單個 `stdClass` 物件：

```php
$user = DB::table('users')->where('name', 'John')->first();

return $user->email;
```

如果您想從資料庫資料表中取得單一資料列，但在找不到相符資料列時拋出 `Illuminate\Database\RecordNotFoundException`，可以使用 `firstOrFail` 方法。如果未捕捉到 `RecordNotFoundException`，則會自動向用戶端回傳 404 HTTP 回應：

```php
$user = DB::table('users')->where('name', 'John')->firstOrFail();
```

如果您不需要一整行資料，可以使用 `value` 方法從紀錄中提取單個值。此方法將直接回傳該欄位的值：

```php
$email = DB::table('users')->where('name', 'John')->value('email');
```

若要透過 `id` 欄位值取得單一資料列，請使用 `find` 方法：

```php
$user = DB::table('users')->find(3);
```


<a name="retrieving-a-list-of-column-values"></a>
#### 取得欄位值列表

如果您想取得一個包含單一欄位值的 `Illuminate\Support\Collection` 實例，可以使用 `pluck` 方法。在以下範例中，我們將取得使用者職稱的集合：

```php
use Illuminate\Support\Facades\DB;

$titles = DB::table('users')->pluck('title');

foreach ($titles as $title) {
    echo $title;
}
```

您可以透過為 `pluck` 方法提供第二個參數，來指定結果集合應使用的鍵值欄位：

```php
$titles = DB::table('users')->pluck('title', 'name');

foreach ($titles as $name => $title) {
    echo $title;
}
```


<a name="chunking-results"></a>
### 分塊查詢結果

如果您需要處理數千條資料庫紀錄，請考慮使用 `DB` Facade 提供的 `chunk` 方法。此方法一次取得一小塊結果，並將每個區塊傳入閉包 (Closure) 進行處理。例如，讓我們一次以 100 筆紀錄為單位，取得整個 `users` 資料表：

```php
use Illuminate\Support\Collection;
use Illuminate\Support\Facades\DB;

DB::table('users')->orderBy('id')->chunk(100, function (Collection $users) {
    foreach ($users as $user) {
        // ...
    }
});
```

您可以從閉包中回傳 `false` 來停止處理後續的區塊：

```php
DB::table('users')->orderBy('id')->chunk(100, function (Collection $users) {
    // Process the records...

    return false;
});
```

如果您在分塊查詢結果時更新資料庫紀錄，分塊結果可能會以非預期的方式改變。如果您打算在分塊時更新取得的紀錄，最好改用 `chunkById` 方法。此方法會根據紀錄的主鍵自動對結果進行分頁：

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

由於 `chunkById` 和 `lazyById` 方法會對執行的查詢加入自己的 "where" 條件，因此您通常應該在閉包內[邏輯分組](#logical-grouping)您自己的條件：

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
> 當在分塊回呼中更新或刪除紀錄時，對主鍵或外鍵的任何變動都可能影響分塊查詢。這可能會導致紀錄未包含在分塊結果中。


<a name="streaming-results-lazily"></a>
### 延遲串流結果

`lazy` 方法與[分塊方法](#chunking-results)類似，因為它也是以分塊方式執行查詢。但是，`lazy()` 方法不會將每個區塊傳遞給回呼，而是回傳一個 [LazyCollection](/docs/{{version}}/collections#lazy-collections)，這讓您可以像處理單一串流一樣與結果互動：

```php
use Illuminate\Support\Facades\DB;

DB::table('users')->orderBy('id')->lazy()->each(function (object $user) {
    // ...
});
```

同樣地，如果您打算在迭代紀錄時對其進行更新，最好改用 `lazyById` 或 `lazyByIdDesc` 方法。這些方法會根據紀錄的主鍵自動對結果進行分頁：

```php
DB::table('users')->where('active', false)
    ->lazyById()->each(function (object $user) {
        DB::table('users')
            ->where('id', $user->id)
            ->update(['active' => true]);
    });
```

> [!WARNING]
> 當在迭代紀錄時更新或刪除紀錄，對主鍵或外鍵的任何變動都可能影響分塊查詢。這可能會導致紀錄未包含在結果中。


<a name="aggregates"></a>
### 聚合 (Aggregates)

查詢產生器還提供了多種方法來取得聚合值，如 `count`、`max`、`min`、`avg` 和 `sum`。您可以在建構查詢後呼叫其中的任何方法：

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

除了使用 `count` 方法來判斷是否存在符合查詢條件的紀錄外，您還可以使用 `exists` 和 `doesntExist` 方法：

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

您可能並不總是想從資料庫資料表中選取所有欄位。使用 `select` 方法，您可以為查詢指定自定義的「select」子句：

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

如果您已經有一個查詢產生器實例，且希望在現有的 select 子句中加入一個欄位，您可以使用 `addSelect` 方法：

```php
$query = DB::table('users')->select('name');

$users = $query->addSelect('age')->get();
```


<a name="raw-expressions"></a>
## 原生運算式 (Raw Expressions)

有時您可能需要在查詢中插入一段任意字串。要建立一個原生字串運算式，您可以使用 `DB` Facade 提供的 `raw` 方法：

```php
$users = DB::table('users')
    ->select(DB::raw('count(*) as user_count, status'))
    ->where('status', '<>', 1)
    ->groupBy('status')
    ->get();
```

> [!WARNING]
> 原生語句將會以字串形式注入查詢中，因此您應該非常小心，以避免產生 SQL 注入攻擊 (SQL injection) 的漏洞。


<a name="raw-methods"></a>
### 原生方法

除了使用 `DB::raw` 方法之外，您也可以使用以下方法將原生運算式插入查詢的各個部分。**請記住，Laravel 無法保證任何使用原生運算式的查詢都能防禦 SQL 注入攻擊。**


<a name="selectraw"></a>
#### `selectRaw`

`selectRaw` 方法可以用來代替 `addSelect(DB::raw(/* ... */))`。此方法接受一個選用的綁定陣列作為其第二個參數：

```php
$orders = DB::table('orders')
    ->selectRaw('price * ? as price_with_tax', [1.0825])
    ->get();
```


<a name="whereraw-orwhereraw"></a>
#### `whereRaw / orWhereRaw`

`whereRaw` 與 `orWhereRaw` 方法可用於將原生的「where」子句注入您的查詢。這些方法接受一個選用的綁定陣列作為其第二個參數：

```php
$orders = DB::table('orders')
    ->whereRaw('price > IF(state = "TX", ?, 100)', [200])
    ->get();
```


<a name="havingraw-orhavingraw"></a>
#### `havingRaw / orHavingRaw`

`havingRaw` 與 `orHavingRaw` 方法可用於提供一個原生字串作為「having」子句的值。這些方法接受一個選用的綁定陣列作為其第二個參數：

```php
$orders = DB::table('orders')
    ->select('department', DB::raw('SUM(price) as total_sales'))
    ->groupBy('department')
    ->havingRaw('SUM(price) > ?', [2500])
    ->get();
```


<a name="orderbyraw"></a>
#### `orderByRaw`

`orderByRaw` 方法可用於提供一個原生字串作為「order by」子句的值：

```php
$orders = DB::table('orders')
    ->orderByRaw('updated_at - created_at DESC')
    ->get();
```


<a name="groupbyraw"></a>
### `groupByRaw`

`groupByRaw` 方法可用於提供一個原生字串作為 `group by` 子句的值：

```php
$orders = DB::table('orders')
    ->select('city', 'state')
    ->groupByRaw('city, state')
    ->get();
```


<a name="joins"></a>
## 連接 (Joins)


<a name="inner-join-clause"></a>
#### 內部連接 (Inner Join) 子句

查詢產生器也可以用來在您的查詢中加入連接 (join) 子句。要執行基本的「內部連接 (inner join)」，您可以在查詢產生器實例上使用 `join` 方法。傳遞給 `join` 方法的第一個參數是您需要連接的資料表名稱，其餘參數則指定連接的欄位條件。您甚至可以在單次查詢中連接多個資料表：

```php
use Illuminate\Support\Facades\DB;

$users = DB::table('users')
    ->join('contacts', 'users.id', '=', 'contacts.user_id')
    ->join('orders', 'users.id', '=', 'orders.user_id')
    ->select('users.*', 'contacts.phone', 'orders.price')
    ->get();
```


<a name="left-join-right-join-clause"></a>
#### 左連接 (Left Join) / 右連接 (Right Join) 子句

如果您想執行「左連接 (left join)」或「右連接 (right join)」而非「內部連接」，請使用 `leftJoin` 或 `rightJoin` 方法。這些方法與 `join` 方法具有相同的參數定義：

```php
$users = DB::table('users')
    ->leftJoin('posts', 'users.id', '=', 'posts.user_id')
    ->get();

$users = DB::table('users')
    ->rightJoin('posts', 'users.id', '=', 'posts.user_id')
    ->get();
```


<a name="cross-join-clause"></a>
#### 交叉連接 (Cross Join) 子句

您可以使用 `crossJoin` 方法來執行「交叉連接」。交叉連接會在第一個資料表與被連接的資料表之間產生笛卡兒積 (cartesian product)：

```php
$sizes = DB::table('sizes')
    ->crossJoin('colors')
    ->get();
```


<a name="advanced-join-clauses"></a>
#### 進階連接子句

您也可以指定更進階的連接子句。首先，將一個閉包 (closure) 作為 `join` 方法的第二個參數傳遞。該閉包將接收一個 `Illuminate\Database\Query\JoinClause` 實例，這允許您在「join」子句上指定條件約束：

```php
DB::table('users')
    ->join('contacts', function (JoinClause $join) {
        $join->on('users.id', '=', 'contacts.user_id')->orOn(/* ... */);
    })
    ->get();
```

如果您想在連接上使用「where」子句，您可以使用 `JoinClause` 實例提供的 `where` 和 `orWhere` 方法。這些方法會將欄位與值進行比較，而不是比較兩個欄位：

```php
DB::table('users')
    ->join('contacts', function (JoinClause $join) {
        $join->on('users.id', '=', 'contacts.user_id')
            ->where('contacts.user_id', '>', 5);
    })
    ->get();
```


<a name="subquery-joins"></a>
#### 子查詢連接

您可以使用 `joinSub`、`leftJoinSub` 和 `rightJoinSub` 方法將查詢與子查詢進行連接。這些方法中的每一個都接受三個參數：子查詢、其資料表別名，以及一個定義相關欄位的閉包。在此範例中，我們將檢索一組使用者，其中每個使用者紀錄還包含該使用者最近發布的部落格文章的 `created_at` 時間戳記：

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
#### 橫向連接 (Lateral Joins)

> [!WARNING]
> 目前 PostgreSQL、MySQL >= 8.0.14 以及 SQL Server 支援橫向連接 (Lateral joins)。

您可以使用 `joinLateral` 和 `leftJoinLateral` 方法對子查詢執行「橫向連接 (lateral join)」。這些方法中的每一個都接受兩個參數：子查詢及其資料表別名。連接條件應在給定子查詢的 `where` 子句中指定。橫向連接會針對每一列進行評估，並且可以引用子查詢外部的欄位。

在此範例中，我們將檢索使用者集合以及該使用者的三篇最新部落格文章。每個使用者在結果集中最多可以產生三列：分別對應於他們最新部落格文章中的每一篇。連接條件是透過子查詢中的 `whereColumn` 子句指定的，該子句引用了當前的使用者列：

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
## 聯合 (Unions)

查詢產生器還提供了一個方便的方法，可以將兩個或多個查詢「聯合 (Union)」在一起。例如，您可以先建立一個初始查詢，然後使用 `union` 方法將其與更多查詢進行聯合：

```php
use Illuminate\Support\Facades\DB;

$usersWithoutFirstName = DB::table('users')
    ->whereNull('first_name');

$users = DB::table('users')
    ->whereNull('last_name')
    ->union($usersWithoutFirstName)
    ->get();
```

除了 `union` 方法之外，查詢產生器還提供了 `unionAll` 方法。使用 `unionAll` 方法結合的查詢不會移除重複的結果。`unionAll` 方法與 `union` 方法具有相同的方法簽章。

<a name="basic-where-clauses"></a>
## 基礎 Where 子句


<a name="where-clauses"></a>
### Where 子句

您可以使用查詢產生器的 `where` 方法在查詢中加入「where」子句。最基本的 `where` 方法呼叫需要三個參數。第一個參數是欄位名稱。第二個參數是運算子，可以是資料庫支援的任何運算子。第三個參數是要與欄位值進行比較的值。

例如，以下查詢會取得 `votes` 欄位等於 `100` 且 `age` 欄位大於 `35` 的使用者：

```php
$users = DB::table('users')
    ->where('votes', '=', 100)
    ->where('age', '>', 35)
    ->get();
```

為了方便起見，如果您想驗證欄位是否「等於」(`=`) 給定值，您可以將該值作為第二個參數傳遞給 `where` 方法。Laravel 會假設您想使用 `=` 運算子：

```php
$users = DB::table('users')->where('votes', 100)->get();
```

您也可以向 `where` 方法提供一個關聯陣列，以便快速地對多個欄位進行查詢：

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

您也可以將條件陣列傳遞給 `where` 函式。陣列中的每個元素應該也是一個陣列，並包含通常傳遞給 `where` 方法的三個參數：

```php
$users = DB::table('users')->where([
    ['status', '=', '1'],
    ['subscribed', '<>', '1'],
])->get();
```

> [!WARNING]
> PDO 不支援綁定欄位名稱。因此，您絕不應該允許使用者輸入來決定查詢所參考的欄位名稱，包括「order by」欄位。

> [!WARNING]
> MySQL 與 MariaDB 在字串與數字的比較中會自動將字串進行型別轉換為整數。在此過程中，非數值字串會被轉換為 `0`，這可能會導致意想不到的結果。例如，如果您的資料表有一個值為 `aaa` 的 `secret` 欄位，而您執行 `User::where('secret', 0)`，該行紀錄將會被回傳。為了避免這種情況，請確保在查詢中使用所有值之前，已將其轉換為正確的型別。


<a name="or-where-clauses"></a>
### Or Where 子句

當串接查詢產生器的 `where` 方法呼叫時，「where」子句將使用 `and` 運算子連接。但是，您可以使用 `orWhere` 方法使用 `or` 運算子將子句加入到查詢中。`orWhere` 方法接受與 `where` 方法相同的參數：

```php
$users = DB::table('users')
    ->where('votes', '>', 100)
    ->orWhere('name', 'John')
    ->get();
```

如果您需要在括號內將一個「or」條件分組，您可以將一個閉包 (Closure) 作為第一個參數傳遞給 `orWhere` 方法：

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

上面的範例將產生以下 SQL：

```sql
select * from users where votes > 100 or (name = 'Abigail' and votes > 50)
```

> [!WARNING]
> 您應該始終對 `orWhere` 呼叫進行分組，以避免在套用全域範圍 (Global Scopes) 時出現預期之外的行為。


<a name="where-not-clauses"></a>
### Where Not 子句

`whereNot` 和 `orWhereNot` 方法可用於否定給定的查詢條件組。例如，以下查詢排除了正在出清或價格小於十個單位的產品：

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

有時您可能需要將相同的查詢條件套用到多個欄位。例如，您可能想取得給定清單中任何欄位都 `LIKE` 給定值的所有紀錄。您可以使用 `whereAny` 方法來達成此目的：

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

同樣地，`whereAll` 方法可用於取得所有給定欄位都符合給定條件的紀錄：

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

`whereNone` 方法可用於取得所有給定欄位都不符合給定條件的紀錄：

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

Laravel 也支援在提供 JSON 欄位類型支援的資料庫上查詢 JSON 欄位類型。目前這包括 MariaDB 10.3+、MySQL 8.0+、PostgreSQL 12.0+、SQL Server 2017+ 和 SQLite 3.39.0+。要查詢 JSON 欄位，請使用 `->` 運算子：

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

如果您的應用程式使用 MariaDB、MySQL 或 PostgreSQL 資料庫，您可以向 `whereJsonContains` 和 `whereJsonDoesntContain` 方法傳遞一個值陣列：

```php
$users = DB::table('users')
    ->whereJsonContains('options->languages', ['en', 'de'])
    ->get();

$users = DB::table('users')
    ->whereJsonDoesntContain('options->languages', ['en', 'de'])
    ->get();
```

此外，您可以使用 `whereJsonContainsKey` 或 `whereJsonDoesntContainKey` 方法來取得包含或不包含特定 JSON 鍵值的結果：

```php
$users = DB::table('users')
    ->whereJsonContainsKey('preferences->dietary_requirements')
    ->get();

$users = DB::table('users')
    ->whereJsonDoesntContainKey('preferences->dietary_requirements')
    ->get();
```

最後，您可以使用 `whereJsonLength` 方法來根據 JSON 陣列的長度進行查詢：

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

`whereLike` 方法允許您在查詢中加入「LIKE」子句以進行模式比對。這些方法提供了一種與資料庫無關的方式來執行字串比對查詢，並具有切換大小寫敏感度的能力。預設情況下，字串比對是不區分大小寫的：

```php
$users = DB::table('users')
    ->whereLike('name', '%John%')
    ->get();
```

您可以透過 `caseSensitive` 參數來啟用區分大小寫的搜尋：

```php
$users = DB::table('users')
    ->whereLike('name', '%John%', caseSensitive: true)
    ->get();
```

`orWhereLike` 方法允許您加入一個帶有 LIKE 條件的「or」子句：

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

同樣地，您可以使用 `orWhereNotLike` 來加入一個帶有 NOT LIKE 條件的「or」子句：

```php
$users = DB::table('users')
    ->where('votes', '>', 100)
    ->orWhereNotLike('name', '%John%')
    ->get();
```

> [!WARNING]
> `whereLike` 的區分大小寫搜尋選項目前在 SQL Server 上不受支援。

**whereIn / whereNotIn / orWhereIn / orWhereNotIn**

`whereIn` 方法驗證給定欄位的值是否包含在給定的陣列中：

```php
$users = DB::table('users')
    ->whereIn('id', [1, 2, 3])
    ->get();
```

`whereNotIn` 方法驗證給定欄位的值不包含在給定的陣列中：

```php
$users = DB::table('users')
    ->whereNotIn('id', [1, 2, 3])
    ->get();
```

您也可以提供一個查詢物件作為 `whereIn` 方法的第二個參數：

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
> 如果您要在查詢中加入大量的整數綁定陣列，可以使用 `whereIntegerInRaw` 或 `whereIntegerNotInRaw` 方法來大幅減少記憶體使用量。

**whereBetween / orWhereBetween**

`whereBetween` 方法驗證欄位的值介於兩個值之間：

```php
$users = DB::table('users')
    ->whereBetween('votes', [1, 100])
    ->get();
```

**whereNotBetween / orWhereNotBetween**

`whereNotBetween` 方法驗證欄位的值位在兩個值之外：

```php
$users = DB::table('users')
    ->whereNotBetween('votes', [1, 100])
    ->get();
```

**whereBetweenColumns / whereNotBetweenColumns / orWhereBetweenColumns / orWhereNotBetweenColumns**

`whereBetweenColumns` 方法驗證欄位的值是否介於同一個資料表資料列中兩個欄位的值之間：

```php
$patients = DB::table('patients')
    ->whereBetweenColumns('weight', ['minimum_allowed_weight', 'maximum_allowed_weight'])
    ->get();
```

`whereNotBetweenColumns` 方法驗證欄位的值是否位在同一個資料表資料列中兩個欄位的值之外：

```php
$patients = DB::table('patients')
    ->whereNotBetweenColumns('weight', ['minimum_allowed_weight', 'maximum_allowed_weight'])
    ->get();
```

**whereValueBetween / whereValueNotBetween / orWhereValueBetween / orWhereValueNotBetween**

`whereValueBetween` 方法驗證給定的值是否介於同一個資料表資料列中兩個相同型別欄位的值之間：

```php
$products = DB::table('products')
    ->whereValueBetween(100, ['min_price', 'max_price'])
    ->get();
```

`whereValueNotBetween` 方法驗證一個值是否位在同一個資料表資料列中兩個欄位的值之外：

```php
$products = DB::table('products')
    ->whereValueNotBetween(100, ['min_price', 'max_price'])
    ->get();
```

**whereNull / whereNotNull / orWhereNull / orWhereNotNull**

`whereNull` 方法驗證給定欄位的值是否為 `NULL`：

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

`whereDay` 方法可用於將欄位的值與一個月中的特定日期進行比較：

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

`wherePast` 與 `whereFuture` 方法可用於判斷欄位的值是在過去還是未來：

```php
$invoices = DB::table('invoices')
    ->wherePast('due_at')
    ->get();

$invoices = DB::table('invoices')
    ->whereFuture('due_at')
    ->get();
```

`whereNowOrPast` 與 `whereNowOrFuture` 方法可用於判斷欄位的值是否在過去或未來，包含目前的日期與時間在內：

```php
$invoices = DB::table('invoices')
    ->whereNowOrPast('due_at')
    ->get();

$invoices = DB::table('invoices')
    ->whereNowOrFuture('due_at')
    ->get();
```

`whereToday`、`whereBeforeToday` 與 `whereAfterToday` 方法分別可用於判斷欄位的值是否為今天、今天之前或今天之後：

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

同樣地，`whereTodayOrBefore` 與 `whereTodayOrAfter` 方法可用於判斷欄位的值是否在今天之前或今天之後，包含今天的日期在內：

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

您也可以傳遞比較運算子給 `whereColumn` 方法：

```php
$users = DB::table('users')
    ->whereColumn('updated_at', '>', 'created_at')
    ->get();
```

您也可以傳遞一個欄位比較的陣列給 `whereColumn` 方法。這些條件將使用 `and` 運算子串接：

```php
$users = DB::table('users')
    ->whereColumn([
        ['first_name', '=', 'last_name'],
        ['updated_at', '>', 'created_at'],
    ])->get();
```

<a name="logical-grouping"></a>
### 邏輯分組

有時您可能需要將多個 "where" 子句分組在括號內，以實現查詢所需的邏輯分組。事實上，通常您應該始終將 `orWhere` 方法的呼叫分組在括號中，以避免非預期的查詢行為。要達成此目的，您可以將閉包 (Closure) 傳遞給 `where` 方法：

```php
$users = DB::table('users')
    ->where('name', '=', 'John')
    ->where(function (Builder $query) {
        $query->where('votes', '>', 100)
            ->orWhere('title', '=', 'Admin');
    })
    ->get();
```

如您所見，將閉包傳遞到 `where` 方法中會指示查詢產生器開始一個限制條件分組。該閉包將接收一個查詢產生器實例，您可以使用它來設定應包含在括號群組內的限制條件。上述範例將產生以下 SQL：

```sql
select * from users where name = 'John' and (votes > 100 or title = 'Admin')
```

> [!WARNING]
> 您應該始終對 `orWhere` 呼叫進行分組，以避免在套用全域範圍 (Global Scopes) 時出現非預期的行為。

<a name="advanced-where-clauses"></a>
## 進階 Where 子句


<a name="where-exists-clauses"></a>
### Where Exists 子句

`whereExists` 方法允許您編寫 "where exists" SQL 子句。`whereExists` 方法接受一個閉包，該閉包將接收一個查詢產生器實例，允許您定義應放置在 "exists" 子句內部的查詢：

```php
$users = DB::table('users')
    ->whereExists(function (Builder $query) {
        $query->select(DB::raw(1))
            ->from('orders')
            ->whereColumn('orders.user_id', 'users.id');
    })
    ->get();
```

或者，您可以向 `whereExists` 方法提供一個查詢物件來代替閉包：

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

有時您可能需要建構一個 "where" 子句，將子查詢的結果與給定值進行比較。您可以透過將閉包和值傳遞給 `where` 方法來實現此目的。例如，以下查詢將檢索所有具有給定類型近期 "membership" 的使用者：

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

或者，您可能需要建構一個將欄位與子查詢結果進行比較的 "where" 子句。您可以透過將欄位、運算器和閉包傳遞給 `where` 方法來實現此目的。例如，以下查詢將檢索所有金額小於平均值的收入紀錄：

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
> 目前 MariaDB、MySQL 和 PostgreSQL 支援全文檢索 where 子句。

`whereFullText` 和 `orWhereFullText` 方法可用於為具有 [全文索引](/docs/{{version}}/migrations#available-index-types) 的欄位新增全文檢索 "where" 子句。這些方法將由 Laravel 轉換為底層資料庫系統適用的 SQL。例如，在使用 MariaDB 或 MySQL 的應用程式中，將會生成 `MATCH AGAINST` 子句：

```php
$users = DB::table('users')
    ->whereFullText('bio', 'web developer')
    ->get();
```


<a name="vector-similarity-clauses"></a>
### 向量相似度子句

> [!NOTE]
> 目前僅在使用 `pgvector` 擴充功能的 PostgreSQL 連線支援向量相似度子句。有關定義向量欄位和索引的資訊，請參閱 [遷移文件](/docs/{{version}}/migrations#available-column-types)。

`whereVectorSimilarTo` 方法根據與給定向量的餘弦相似度 (Cosine Similarity) 過濾結果，並按相關性對結果進行排序。`minSimilarity` 閾值應為 `0.0` 到 `1.0` 之間的值，其中 `1.0` 代表完全相同：

```php
$documents = DB::table('documents')
    ->whereVectorSimilarTo('embedding', $queryEmbedding, minSimilarity: 0.4)
    ->limit(10)
    ->get();
```

當向量參數給定為純字串時，Laravel 將使用 [Laravel AI SDK](/docs/{{version}}/ai-sdk#embeddings) 自動為其生成嵌入 (Embeddings)：

```php
$documents = DB::table('documents')
    ->whereVectorSimilarTo('embedding', 'Best wineries in Napa Valley')
    ->limit(10)
    ->get();
```

預設情況下，`whereVectorSimilarTo` 也會按距離進行排序（最相似的排在前面）。您可以透過將 `false` 作為 `order` 參數傳遞來停用此排序：

```php
$documents = DB::table('documents')
    ->whereVectorSimilarTo('embedding', $queryEmbedding, minSimilarity: 0.4, order: false)
    ->orderBy('created_at', 'desc')
    ->limit(10)
    ->get();
```

如果您需要更多控制，可以獨立使用 `selectVectorDistance`、`whereVectorDistanceLessThan` 和 `orderByVectorDistance` 方法：

```php
$documents = DB::table('documents')
    ->select('*')
    ->selectVectorDistance('embedding', $queryEmbedding, as: 'distance')
    ->whereVectorDistanceLessThan('embedding', $queryEmbedding, maxDistance: 0.3)
    ->orderByVectorDistance('embedding', $queryEmbedding)
    ->limit(10)
    ->get();
```

使用 PostgreSQL 時，必須在建立 `vector` 欄位之前載入 `pgvector` 擴充功能：

```php
Schema::ensureVectorExtensionExists();
```

<a name="ordering-grouping-limit-and-offset"></a>
## 排序、分組、限制與偏移

<a name="ordering"></a>
### 排序

<a name="orderby"></a>
#### `orderBy` 方法

`orderBy` 方法允許您根據給定的欄位對查詢結果進行排序。`orderBy` 方法接受的第一個參數應為您希望排序的欄位，而第二個參數決定排序的方向，可以是 `asc` 或 `desc`：

```php
$users = DB::table('users')
    ->orderBy('name', 'desc')
    ->get();
```

若要根據多個欄位排序，您只需根據需要多次呼叫 `orderBy`：

```php
$users = DB::table('users')
    ->orderBy('name', 'desc')
    ->orderBy('email', 'asc')
    ->get();
```

排序方向是選填的，預設為遞增排序。如果您想以遞減順序排序，可以為 `orderBy` 方法指定第二個參數，或者直接使用 `orderByDesc`：

```php
$users = DB::table('users')
    ->orderByDesc('verified_at')
    ->get();
```

最後，使用 `->` 運算子，可以根據 JSON 欄位中的值對結果進行排序：

```php
$corporations = DB::table('corporations')
    ->where('country', 'US')
    ->orderBy('location->state')
    ->get();
```

<a name="latest-oldest"></a>
#### `latest` 與 `oldest` 方法

`latest` 與 `oldest` 方法讓您可以輕鬆地根據日期對結果進行排序。預設情況下，結果將根據資料表的 `created_at` 欄位排序。或者，您可以傳入想要排序的欄位名稱：

```php
$user = DB::table('users')
    ->latest()
    ->first();
```

<a name="random-ordering"></a>
#### 隨機排序

`inRandomOrder` 方法可用於隨機排序查詢結果。例如，您可以使用此方法取得一位隨機使用者：

```php
$randomUser = DB::table('users')
    ->inRandomOrder()
    ->first();
```

<a name="removing-existing-orderings"></a>
#### 移除現有的排序

`reorder` 方法會移除先前應用於查詢的所有 "order by" 子句：

```php
$query = DB::table('users')->orderBy('name');

$unorderedUsers = $query->reorder()->get();
```

您可以在呼叫 `reorder` 方法時傳入欄位與方向，以移除所有現有的 "order by" 子句並為查詢應用全新的排序：

```php
$query = DB::table('users')->orderBy('name');

$usersOrderedByEmail = $query->reorder('email', 'desc')->get();
```

為了方便起見，您可以使用 `reorderDesc` 方法以遞減順序對查詢結果進行重新排序：

```php
$query = DB::table('users')->orderBy('name');

$usersOrderedByEmail = $query->reorderDesc('email')->get();
```

<a name="grouping"></a>
### 分組

<a name="groupby-having"></a>
#### `groupBy` 與 `having` 方法

如您所料，`groupBy` 與 `having` 方法可用於對查詢結果進行分組。`having` 方法的簽章與 `where` 方法相似：

```php
$users = DB::table('users')
    ->groupBy('account_id')
    ->having('account_id', '>', 100)
    ->get();
```

您可以使用 `havingBetween` 方法在給定範圍內過濾結果：

```php
$report = DB::table('orders')
    ->selectRaw('count(id) as number_of_orders, customer_id')
    ->groupBy('customer_id')
    ->havingBetween('number_of_orders', [5, 15])
    ->get();
```

您可以向 `groupBy` 方法傳遞多個參數，以便根據多個欄位進行分組：

```php
$users = DB::table('users')
    ->groupBy('first_name', 'status')
    ->having('account_id', '>', 100)
    ->get();
```

若要建立更進階的 `having` 語句，請參閱 [havingRaw](#raw-methods) 方法。

<a name="limit-and-offset"></a>
### 限制與偏移

您可以使用 `limit` 與 `offset` 方法來限制查詢回傳的結果數量，或在查詢中跳過給定數量的結果：

```php
$users = DB::table('users')
    ->offset(10)
    ->limit(5)
    ->get();
```

<a name="conditional-clauses"></a>
## 條件子句

有時您可能希望根據另一個條件將某些查詢子句應用於查詢。例如，只有在傳入的 HTTP 請求中存在給定的輸入值時，您才想應用 `where` 語句。您可以使用 `when` 方法來達成此目的：

```php
$role = $request->input('role');

$users = DB::table('users')
    ->when($role, function (Builder $query, string $role) {
        $query->where('role_id', $role);
    })
    ->get();
```

`when` 方法僅在第一個參數為 `true` 時執行給定的閉包。如果第一個參數為 `false`，則閉包將不會執行。因此，在上面的範例中，只有當傳入請求中存在 `role` 欄位且其評估為 `true` 時，傳給 `when` 方法的閉包才會被呼叫。

您可以傳遞另一個閉包作為 `when` 方法的第三個參數。該閉包僅在第一個參數評估為 `false` 時執行。為了說明如何使用此功能，我們將使用它來設定查詢的預設排序：

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

查詢產生器也提供了一個 `insert` 方法，可用於將紀錄插入資料庫資料表中。`insert` 方法接受一個包含欄位名稱與值的陣列：

```php
DB::table('users')->insert([
    'email' => 'kayla@example.com',
    'votes' => 0
]);
```

您可以透過傳遞陣列的陣列來一次插入多條紀錄。每個陣列代表應插入資料表的一條紀錄：

```php
DB::table('users')->insert([
    ['email' => 'picard@example.com', 'votes' => 0],
    ['email' => 'janeway@example.com', 'votes' => 0],
]);
```

`insertOrIgnore` 方法在將紀錄插入資料庫時會忽略錯誤。使用此方法時，您應該注意重複紀錄的錯誤將被忽略，且根據資料庫引擎的不同，其他類型的錯誤也可能被忽略。例如，`insertOrIgnore` 會 [繞過 MySQL 的嚴格模式 (Strict Mode)](https://dev.mysql.com/doc/refman/en/sql-mode.html#ignore-effect-on-execution)：

```php
DB::table('users')->insertOrIgnore([
    ['id' => 1, 'email' => 'sisko@example.com'],
    ['id' => 2, 'email' => 'archer@example.com'],
]);
```

`insertUsing` 方法會將新紀錄插入資料表中，同時使用子查詢來決定應插入的資料：

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
> 當使用 PostgreSQL 時，`insertGetId` 方法預期自動遞增欄位的名稱為 `id`。如果您想從不同的「序列 (Sequence)」中取得 ID，可以將欄位名稱作為第二個參數傳遞給 `insertGetId` 方法。


<a name="upserts"></a>
### 更新或插入 (Upserts)

`upsert` 方法將插入不存在的紀錄，並使用您指定的新值更新已存在的紀錄。該方法的第一個參數包含要插入或更新的值，而第二個參數列出在相關資料表中唯一識別紀錄的欄位。該方法的第三個也是最後一個參數是一個陣列，包含如果資料庫中已存在匹配紀錄時應更新的欄位：

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

在上面的範例中，Laravel 將嘗試插入兩條紀錄。如果已經存在具有相同 `departure` 和 `destination` 欄位值的紀錄，Laravel 將更新該紀錄的 `price` 欄位。

> [!WARNING]
> 除了 SQL Server 之外的所有資料庫都要求 `upsert` 方法第二個參數中的欄位具有「主鍵 (Primary)」或「唯一 (Unique)」索引。此外，MariaDB 和 MySQL 資料庫驅動程式會忽略 `upsert` 方法的第二個參數，並始終使用資料表的「主鍵」和「唯一」索引來偵測現有紀錄。


<a name="update-statements"></a>
## Update 語句

除了向資料庫插入紀錄外，查詢產生器還可以使用 `update` 方法更新現有紀錄。`update` 方法與 `insert` 方法一樣，接受一個由欄位與值組成的陣列，指示要更新的欄位。`update` 方法會返回受影響的列數。您可以使用 `where` 子句來限制 `update` 查詢：

```php
$affected = DB::table('users')
    ->where('id', 1)
    ->update(['votes' => 1]);
```


<a name="update-or-insert"></a>
#### 更新或插入 (Update or Insert)

有時您可能想更新資料庫中的現有紀錄，或者如果沒有匹配的紀錄則建立它。在這種情況下，可以使用 `updateOrInsert` 方法。`updateOrInsert` 方法接受兩個參數：一個用於尋找紀錄的條件陣列，以及一個指示要更新的欄位的欄位與值陣列。

`updateOrInsert` 方法將嘗試使用第一個參數的欄位與值來定位匹配的資料庫紀錄。如果紀錄存在，則會使用第二個參數中的值進行更新。如果找不到紀錄，則會插入一條新紀錄，其中包含兩個參數合併後的屬性：

```php
DB::table('users')
    ->updateOrInsert(
        ['email' => 'john@example.com', 'name' => 'John'],
        ['votes' => '2']
    );
```

您可以為 `updateOrInsert` 方法提供一個閉包 (Closure)，以根據是否存在匹配紀錄來自定義更新或插入資料庫的屬性：

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

更新 JSON 欄位時，您應該使用 `->` 語法來更新 JSON 物件中的相應鍵 (Key)。MariaDB 10.3+、MySQL 5.7+ 和 PostgreSQL 9.5+ 支援此操作：

```php
$affected = DB::table('users')
    ->where('id', 1)
    ->update(['options->enabled' => true]);
```


<a name="increment-and-decrement"></a>
### 遞增與遞減

查詢產生器還提供了遞增或遞減給定欄位值的便捷方法。這兩個方法都接受至少一個參數：要修改的欄位。可以提供第二個參數來指定欄位應遞增或遞減的量：

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
## Delete 語句

查詢產生器的 `delete` 方法可以用於從資料表中刪除紀錄。`delete` 方法返回受影響的列數。您可以在呼叫 `delete` 方法之前透過新增「where」子句來限制 `delete` 語句：

```php
$deleted = DB::table('users')->delete();

$deleted = DB::table('users')->where('votes', '>', 100)->delete();
```

<a name="pessimistic-locking"></a>
## 悲觀鎖定 (Pessimistic Locking)

查詢產生器還包含了一些函式，可以幫助您在執行 `select` 語句時實現「悲觀鎖定」。若要執行帶有「共用鎖 (shared lock)」的語句，您可以呼叫 `sharedLock` 方法。共用鎖可防止選取的資料列在您的交易提交之前被修改：

```php
DB::table('users')
    ->where('votes', '>', 100)
    ->sharedLock()
    ->get();
```

或者，您可以使用 `lockForUpdate` 方法。「更新鎖 (for update lock)」可防止選取的紀錄被修改，或被另一個共用鎖選取：

```php
DB::table('users')
    ->where('votes', '>', 100)
    ->lockForUpdate()
    ->get();
```

雖然不是強制性的，但建議將悲觀鎖定包裝在 [交易](/docs/{{version}}/database#database-transactions) 中。這可確保在整個操作完成之前，資料庫中擷取的資料保持不變。如果發生失敗，交易將自動回滾任何更改並釋放鎖定：

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

如果在整個應用程式中重複使用相同的查詢邏輯，您可以使用查詢產生器的 `tap` 和 `pipe` 方法將邏輯擷取到可重複使用的物件中。假設您的應用程式中有這兩個不同的查詢：

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

您可能想將查詢之間共同的目的地過濾邏輯擷取到一個可重複使用的物件中：

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

接著，您可以使用查詢產生器的 `tap` 方法將該物件的邏輯套用到查詢中：

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
#### 查詢管線 (Query Pipes)

`tap` 方法始終會回傳查詢產生器。如果您想擷取一個會執行查詢並回傳另一個值的物件，則可以使用 `pipe` 方法代替。

考慮以下查詢物件，它包含了整個應用程式中通用的 [分頁 (pagination)](/docs/{{version}}/pagination) 邏輯。與套用查詢條件的 `DestinationFilter` 不同，`Paginate` 物件會執行查詢並回傳一個分頁器實例：

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

藉由使用查詢產生器的 `pipe` 方法，我們可以利用這個物件來套用我們共用的分頁邏輯：

```php
$flights = DB::table('flights')
    ->tap(new DestinationFilter($destination))
    ->pipe(new Paginate);
```


<a name="debugging"></a>
## 除錯

您可以在建立查詢時使用 `dd` 和 `dump` 方法來傾印目前的查詢綁定 (bindings) 與 SQL。`dd` 方法會顯示除錯資訊並停止執行請求。`dump` 方法則會顯示除錯資訊，但允許請求繼續執行：

```php
DB::table('users')->where('votes', '>', 100)->dd();

DB::table('users')->where('votes', '>', 100)->dump();
```

可以在查詢上呼叫 `dumpRawSql` 和 `ddRawSql` 方法，以傾印已將所有參數綁定正確代入後的查詢 SQL：

```php
DB::table('users')->where('votes', '>', 100)->dumpRawSql();

DB::table('users')->where('votes', '>', 100)->ddRawSql();
```