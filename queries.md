# 資料庫：查詢產生器

- [簡介](#introduction)
- [執行資料庫查詢](#running-database-queries)
    - [分批處理結果](#chunking-results)
    - [惰性串流處理結果](#streaming-results-lazily)
    - [聚合函數](#aggregates)
- [Select 語句](#select-statements)
- [原生表達式](#raw-expressions)
- [Join 語句](#joins)
- [Union 語句](#unions)
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
- [排序、分組、限制與偏移](#ordering-grouping-limit-and-offset)
    - [排序](#ordering)
    - [分組](#grouping)
    - [限制與偏移](#limit-and-offset)
- [條件式子句](#conditional-clauses)
- [Insert 語句](#insert-statements)
    - [Upserts](#upserts)
- [Update 語句](#update-statements)
    - [更新 JSON 欄位](#updating-json-columns)
    - [遞增與遞減](#increment-and-decrement)
- [Delete 語句](#delete-statements)
- [悲觀鎖定](#pessimistic-locking)
- [除錯](#debugging)

<a name="introduction"></a>
## 簡介

Laravel 的資料庫查詢產生器提供了一個方便、流暢的介面來建立與執行資料庫查詢。它可以用於執行應用程式中的大多數資料庫操作，並且與所有 Laravel 支援的資料庫系統完美配合。

Laravel 查詢產生器使用 PDO 參數綁定來保護您的應用程式免受 SQL 注入攻擊。傳遞給查詢產生器作為查詢綁定的字串無需清理或消毒。

> [!WARNING]
> PDO 不支援綁定欄位名稱。因此，您絕不應允許使用者輸入來決定查詢中引用的欄位名稱，包括「order by」欄位。

<a name="running-database-queries"></a>
## 執行資料庫查詢

<a name="retrieving-all-rows-from-a-table"></a>
#### 從資料表中取得所有資料列

您可以使用 `DB` Facade 提供的 `table` 方法來開始查詢。`table` 方法會回傳指定資料表的流暢查詢產生器實例，讓您可以將更多限制條件鏈接到查詢中，然後最終使用 `get` 方法取得查詢結果：

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

`get` 方法會回傳一個 `Illuminate\Support\Collection` 實例，其中包含查詢結果，每個結果都是 PHP `stdClass` 物件的實例。您可以透過將欄位作為物件的屬性來存取每個欄位的值：

    use Illuminate\Support\Facades\DB;

    $users = DB::table('users')->get();

    foreach ($users as $user) {
        echo $user->name;
    }

> [!NOTE]
> Laravel Collection 提供了各種功能強大的方法來映射與縮減資料。有關 Laravel Collection 的更多資訊，請查閱 [Collection 說明文件](/docs/{{version}}/collections)。

<a name="retrieving-a-single-row-column-from-a-table"></a>
#### 從資料表中取得單一資料列/欄位

如果您只需要從資料庫表中取得單一資料列，可以使用 `DB` Facade 的 `first` 方法。此方法將回傳一個單一的 `stdClass` 物件：

    $user = DB::table('users')->where('name', 'John')->first();

    return $user->email;

如果您想從資料庫表中取得單一資料列，但如果找不到匹配的資料列則拋出 `Illuminate\Database\RecordNotFoundException`，您可以使用 `firstOrFail` 方法。如果未捕獲 `RecordNotFoundException`，則會自動向客戶端回傳 404 HTTP 回應：

    $user = DB::table('users')->where('name', 'John')->firstOrFail();

如果您不需要整個資料列，可以使用 `value` 方法從記錄中提取單一值。此方法將直接回傳欄位的值：

    $email = DB::table('users')->where('name', 'John')->value('email');

若要依據 `id` 欄位值取得單一資料列，請使用 `find` 方法：

    $user = DB::table('users')->find(3);

<a name="retrieving-a-list-of-column-values"></a>
#### 取得欄位值列表

如果您想取得一個包含單一欄位值的 `Illuminate\Support\Collection` 實例，可以使用 `pluck` 方法。在此範例中，我們將取得一個使用者標題的 Collection：

    use Illuminate\Support\Facades\DB;

    $titles = DB::table('users')->pluck('title');

    foreach ($titles as $title) {
        echo $title;
    }

您可以透過向 `pluck` 方法提供第二個參數來指定結果 Collection 應使用哪個欄位作為其鍵：

    $titles = DB::table('users')->pluck('title', 'name');

    foreach ($titles as $name => $title) {
        echo $title;
    }

<a name="chunking-results"></a>
### 分批處理結果

如果您需要處理數千條資料庫記錄，請考慮使用 `DB` Facade 提供的 `chunk` 方法。此方法一次取得一小批結果，並將每批結果傳遞給一個閉包進行處理。例如，讓我們以每次 100 條記錄的分批方式取得整個 `users` 資料表：

    use Illuminate\Support\Collection;
    use Illuminate\Support\Facades\DB;

    DB::table('users')->orderBy('id')->chunk(100, function (Collection $users) {
        foreach ($users as $user) {
            // ...
        }
    });

您可以透過從閉包中回傳 `false` 來停止處理後續的批次：

    DB::table('users')->orderBy('id')->chunk(100, function (Collection $users) {
        // Process the records...

        return false;
    });

如果您在分批處理結果時更新資料庫記錄，您的分批結果可能會以意想不到的方式改變。如果您計劃在分批處理時更新取得的記錄，最好始終使用 `chunkById` 方法。此方法將根據記錄的主鍵自動分頁結果：

    DB::table('users')->where('active', false)
        ->chunkById(100, function (Collection $users) {
            foreach ($users as $user) {
                DB::table('users')
                    ->where('id', $user->id)
                    ->update(['active' => true]);
            }
        });

由於 `chunkById` 和 `lazyById` 方法會將其自己的「where」條件新增到正在執行的查詢中，因此您通常應該在閉包中[邏輯分組](#logical-grouping)您自己的條件：

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
> 在 `chunk` 回呼中更新或刪除記錄時，主鍵或外鍵的任何變更都可能影響分批查詢。這可能會導致記錄未包含在分批結果中。

<a name="streaming-results-lazily"></a>
### 惰性串流處理結果

`lazy` 方法的工作方式與[ `chunk` 方法](#chunking-results)類似，它會分批執行查詢。然而，`lazy()` 方法不是將每個批次傳遞給回呼，而是回傳一個 [`LazyCollection`](/docs/{{version}}/collections#lazy-collections)，讓您可以將結果作為單一串流進行互動：

```php
use Illuminate\Support\Facades\DB;

DB::table('users')->orderBy('id')->lazy()->each(function (object $user) {
    // ...
});
```

再次強調，如果您計劃在迭代記錄時更新它們，最好使用 `lazyById` 或 `lazyByIdDesc` 方法。這些方法將根據記錄的主鍵自動分頁結果：

```php
DB::table('users')->where('active', false)
    ->lazyById()->each(function (object $user) {
        DB::table('users')
            ->where('id', $user->id)
            ->update(['active' => true]);
    });
```

> [!WARNING]
> 在迭代記錄時更新或刪除記錄時，主鍵或外鍵的任何變更都可能影響分批查詢。這可能會導致記錄未包含在結果中。

<a name="aggregates"></a>
### 聚合函數

查詢產生器還提供了各種方法來取得聚合值，例如 `count`、`max`、`min`、`avg` 和 `sum`。您可以在建構查詢後呼叫這些方法中的任何一個：

    use Illuminate\Support\Facades\DB;

    $users = DB::table('users')->count();

    $price = DB::table('orders')->max('price');

當然，您可以將這些方法與其他子句結合使用，以微調聚合值的計算方式：

    $price = DB::table('orders')
        ->where('finalized', 1)
        ->avg('price');

<a name="determining-if-records-exist"></a>
#### 判斷記錄是否存在

您可以使用 `exists` 和 `doesntExist` 方法來判斷是否存在符合查詢條件的記錄，而不是使用 `count` 方法：

    if (DB::table('orders')->where('finalized', 1)->exists()) {
        // ...
    }

    if (DB::table('orders')->where('finalized', 1)->doesntExist()) {
        // ...
    }

<a name="select-statements"></a>
## Select 語句

<a name="specifying-a-select-clause"></a>
#### 指定 Select 子句

您可能不總是想從資料庫表中選取所有欄位。使用 `select` 方法，您可以為查詢指定自訂的「select」子句：

    use Illuminate\Support\Facades\DB;

    $users = DB::table('users')
        ->select('name', 'email as user_email')
        ->get();

`distinct` 方法允許您強制查詢回傳不同的結果：

    $users = DB::table('users')->distinct()->get();

如果您已經有一個查詢產生器實例，並且希望向其現有的 select 子句新增一個欄位，您可以使用 `addSelect` 方法：

    $query = DB::table('users')->select('name');

    $users = $query->addSelect('age')->get();

<a name="raw-expressions"></a>
## 原生表達式

有時您可能需要在查詢中插入任意字串。若要建立原生字串表達式，您可以使用 `DB` Facade 提供的 `raw` 方法：

    $users = DB::table('users')
        ->select(DB::raw('count(*) as user_count, status'))
        ->where('status', '<>', 1)
        ->groupBy('status')
        ->get();

> [!WARNING]
> 原生語句將作為字串注入到查詢中，因此您應該非常小心，避免產生 SQL 注入漏洞。

<a name="raw-methods"></a>
### 原生方法

除了使用 `DB::raw` 方法，您還可以使用以下方法將原生表達式插入到查詢的各個部分。**請記住，Laravel 無法保證任何使用原生表達式的查詢都能免受 SQL 注入漏洞的侵害。**

<a name="selectraw"></a>
#### `selectRaw`

`selectRaw` 方法可以用來取代 `addSelect(DB::raw(/* ... */))`。此方法接受一個可選的綁定陣列作為其第二個參數：

    $orders = DB::table('orders')
        ->selectRaw('price * ? as price_with_tax', [1.0825])
        ->get();

<a name="whereraw-orwhereraw"></a>
#### `whereRaw / orWhereRaw`

`whereRaw` 和 `orWhereRaw` 方法可用於將原生「where」子句注入到查詢中。這些方法接受一個可選的綁定陣列作為其第二個參數：

    $orders = DB::table('orders')
        ->whereRaw('price > IF(state = "TX", ?, 100)', [200])
        ->get();

<a name="havingraw-orhavingraw"></a>
#### `havingRaw / orHavingRaw`

`havingRaw` 和 `orHavingRaw` 方法可用於提供原生字串作為「having」子句的值。這些方法接受一個可選的綁定陣列作為其第二個參數：

    $orders = DB::table('orders')
        ->select('department', DB::raw('SUM(price) as total_sales'))
        ->groupBy('department')
        ->havingRaw('SUM(price) > ?', [2500])
        ->get();

<a name="orderbyraw"></a>
#### `orderByRaw`

`orderByRaw` 方法可用於提供原生字串作為「order by」子句的值：

    $orders = DB::table('orders')
        ->orderByRaw('updated_at - created_at DESC')
        ->get();

<a name="groupbyraw"></a>
### `groupByRaw`

`groupByRaw` 方法可用於提供原生字串作為 `group by` 子句的值：

    $orders = DB::table('orders')
        ->select('city', 'state')
        ->groupByRaw('city, state')
        ->get();

<a name="joins"></a>
## Joins

<a name="inner-join-clause"></a>
#### Inner Join 子句

查詢產生器也可用於向查詢新增 Join 子句。若要執行基本的「inner join」，您可以在查詢產生器實例上使用 `join` 方法。傳遞給 `join` 方法的第一個參數是您需要 Join 的資料表名稱，而其餘參數則指定 Join 的欄位限制。您甚至可以在單一查詢中 Join 多個資料表：

    use Illuminate\Support\Facades\DB;

    $users = DB::table('users')
        ->join('contacts', 'users.id', '=', 'contacts.user_id')
        ->join('orders', 'users.id', '=', 'orders.user_id')
        ->select('users.*', 'contacts.phone', 'orders.price')
        ->get();

<a name="left-join-right-join-clause"></a>
#### Left Join / Right Join 子句

如果您想執行「left join」或「right join」而不是「inner join」，請使用 `leftJoin` 或 `rightJoin` 方法。這些方法與 `join` 方法具有相同的簽名：

    $users = DB::table('users')
        ->leftJoin('posts', 'users.id', '=', 'posts.user_id')
        ->get();

    $users = DB::table('users')
        ->rightJoin('posts', 'users.id', '=', 'posts.user_id')
        ->get();

<a name="cross-join-clause"></a>
#### Cross Join 子句

您可以使用 `crossJoin` 方法執行「cross join」。Cross Join 會在第一個資料表與 Join 的資料表之間產生笛卡爾積：

    $sizes = DB::table('sizes')
        ->crossJoin('colors')
        ->get();

<a name="advanced-join-clauses"></a>
#### 進階 Join 子句

您還可以指定更進階的 Join 子句。首先，將閉包作為第二個參數傳遞給 `join` 方法。該閉包將接收一個 `Illuminate\Database\Query\JoinClause` 實例，允許您指定「join」子句上的限制：

    DB::table('users')
        ->join('contacts', function (JoinClause $join) {
            $join->on('users.id', '=', 'contacts.user_id')->orOn(/* ... */);
        })
        ->get();

如果您想在 Join 上使用「where」子句，您可以使用 `JoinClause` 實例提供的 `where` 和 `orWhere` 方法。這些方法不是比較兩個欄位，而是將欄位與值進行比較：

    DB::table('users')
        ->join('contacts', function (JoinClause $join) {
            $join->on('users.id', '=', 'contacts.user_id')
                ->where('contacts.user_id', '>', 5);
        })
        ->get();

<a name="subquery-joins"></a>
#### 子查詢 Join

您可以使用 `joinSub`、`leftJoinSub` 和 `rightJoinSub` 方法將查詢 Join 到子查詢。這些方法中的每一個都接收三個參數：子查詢、其資料表別名以及定義相關欄位的閉包。在此範例中，我們將取得一個使用者集合，其中每個使用者記錄還包含該使用者最近發布的部落格文章的 `created_at` 時間戳記：

    $latestPosts = DB::table('posts')
        ->select('user_id', DB::raw('MAX(created_at) as last_post_created_at'))
        ->where('is_published', true)
        ->groupBy('user_id');

    $users = DB::table('users')
        ->joinSub($latestPosts, 'latest_posts', function (JoinClause $join) {
            $join->on('users.id', '=', 'latest_posts.user_id');
        })->get();

<a name="lateral-joins"></a>
#### Lateral Joins

> [!WARNING]
> Lateral Joins 目前支援 PostgreSQL、MySQL >= 8.0.14 和 SQL Server。

您可以使用 `joinLateral` 和 `leftJoinLateral` 方法與子查詢執行「lateral join」。這些方法中的每一個都接收兩個參數：子查詢及其資料表別名。Join 條件應在給定子查詢的 `where` 子句中指定。Lateral Joins 會針對每一列進行評估，並且可以引用子查詢外部的欄位。

在此範例中，我們將取得一個使用者集合以及該使用者最近的三篇部落格文章。每個使用者在結果集中最多可以產生三列：每篇最近的部落格文章一列。Join 條件在子查詢中透過 `whereColumn` 子句指定，引用當前使用者列：

    $latestPosts = DB::table('posts')
        ->select('id as post_id', 'title as post_title', 'created_at as post_created_at')
        ->whereColumn('user_id', 'users.id')
        ->orderBy('created_at', 'desc')
        ->limit(3);

    $users = DB::table('users')
        ->joinLateral($latestPosts, 'latest_posts')
        ->get();

<a name="unions"></a>
## Unions

查詢產生器還提供了一個方便的方法來「union」兩個或多個查詢。例如，您可以建立一個初始查詢，並使用 `union` 方法將其與更多查詢進行 union：

    use Illuminate\Support\Facades\DB;

    $first = DB::table('users')
        ->whereNull('first_name');

    $users = DB::table('users')
        ->whereNull('last_name')
        ->union($first)
        ->get();

除了 `union` 方法，查詢產生器還提供了 `unionAll` 方法。使用 `unionAll` 方法組合的查詢將不會移除重複的結果。`unionAll` 方法與 `union` 方法具有相同的簽名。

<a name="basic-where-clauses"></a>
## 基本 Where 子句

<a name="where-clauses"></a>
### Where 子句

您可以使用查詢產生器的 `where` 方法向查詢新增「where」子句。對 `where` 方法最基本的呼叫需要三個參數。第一個參數是欄位名稱。第二個參數是運算子，可以是資料庫支援的任何運算子。第三個參數是與欄位值進行比較的數值。

例如，以下查詢取得 `votes` 欄位值等於 `100` 且 `age` 欄位值大於 `35` 的使用者：

    $users = DB::table('users')
        ->where('votes', '=', 100)
        ->where('age', '>', 35)
        ->get();

為方便起見，如果您想驗證欄位是否 `=` 於給定值，您可以將該值作為第二個參數傳遞給 `where` 方法。Laravel 將假定您想使用 `=` 運算子：

    $users = DB::table('users')->where('votes', 100)->get();

如前所述，您可以使用資料庫系統支援的任何運算子：

    $users = DB::table('users')
        ->where('votes', '>=', 100)
        ->get();

    $users = DB::table('users')
        ->where('votes', '<>', 100)
        ->get();

    $users = DB::table('users')
        ->where('name', 'like', 'T%')
        ->get();

您也可以向 `where` 函數傳遞一個條件陣列。陣列的每個元素都應該是一個包含通常傳遞給 `where` 方法的三個參數的陣列：

    $users = DB::table('users')->where([
        ['status', '=', '1'],
        ['subscribed', '<>', '1'],
    ])->get();

> [!WARNING]
> PDO 不支援綁定欄位名稱。因此，您絕不應允許使用者輸入來決定查詢中引用的欄位名稱，包括「order by」欄位。

> [!WARNING]
> MySQL 和 MariaDB 在字串與數字比較中會自動將字串轉換為整數。在此過程中，非數字字串會轉換為 `0`，這可能導致意外結果。例如，如果您的資料表有一個 `secret` 欄位，其值為 `aaa`，並且您執行 `User::where('secret', 0)`，則會回傳該列。為避免這種情況，請確保所有值在使用於查詢之前都已轉換為其適當的類型。

<a name="or-where-clauses"></a>
### Or Where 子句

當鏈接呼叫查詢產生器的 `where` 方法時，「where」子句將使用 `and` 運算子連接在一起。但是，您可以使用 `orWhere` 方法使用 `or` 運算子將子句連接到查詢。`orWhere` 方法接受與 `where` 方法相同的參數：

    $users = DB::table('users')
        ->where('votes', '>', 100)
        ->orWhere('name', 'John')
        ->get();

如果您需要在括號中分組「or」條件，您可以將閉包作為第一個參數傳遞給 `orWhere` 方法：

    $users = DB::table('users')
        ->where('votes', '>', 100)
        ->orWhere(function (Builder $query) {
            $query->where('name', 'Abigail')
                ->where('votes', '>', 50);
            })
        ->get();

上面的範例將產生以下 SQL：

```sql
select * from users where votes > 100 or (name = 'Abigail' and votes > 50)
```

> [!WARNING]
> 您應該始終將 `orWhere` 呼叫分組，以避免在應用全域範圍時出現意外行為。

<a name="where-not-clauses"></a>
### Where Not 子句

`whereNot` 和 `orWhereNot` 方法可用於否定給定的一組查詢限制。例如，以下查詢排除正在清倉或價格低於十元的產品：

    $products = DB::table('products')
        ->whereNot(function (Builder $query) {
            $query->where('clearance', true)
                ->orWhere('price', '<', 10);
            })
        ->get();

<a name="where-any-all-none-clauses"></a>
### Where Any / All / None 子句

有時您可能需要將相同的查詢限制應用於多個欄位。例如，您可能只想取得給定列表中任何欄位 `LIKE` 給定值的記錄。您可以使用 `whereAny` 方法來實現此目的：

    $users = DB::table('users')
        ->where('active', true)
        ->whereAny([
            'name',
            'email',
            'phone',
        ], 'like', 'Example%')
        ->get();

上面的查詢將產生以下 SQL：

```sql
SELECT *
FROM users
WHERE active = true AND (
    name LIKE 'Example%' OR
    email LIKE 'Example%' OR
    phone LIKE 'Example%'
)
```

同樣，`whereAll` 方法可用於取得所有給定欄位都符合給定限制的記錄：

    $posts = DB::table('posts')
        ->where('published', true)
        ->whereAll([
            'title',
            'content',
        ], 'like', '%Laravel%')
        ->get();

上面的查詢將產生以下 SQL：

```sql
SELECT *
FROM posts
WHERE published = true AND (
    title LIKE '%Laravel%' AND
    content LIKE '%Laravel%'
)
```

`whereNone` 方法可用於取得所有給定欄位都不符合給定限制的記錄：

    $posts = DB::table('albums')
        ->where('published', true)
        ->whereNone([
            'title',
            'lyrics',
            'tags',
        ], 'like', '%explicit%')
        ->get();

上面的查詢將產生以下 SQL：

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

    $users = DB::table('users')
        ->where('preferences->dining->meal', 'salad')
        ->get();

您可以使用 `whereJsonContains` 查詢 JSON 陣列：

    $users = DB::table('users')
        ->whereJsonContains('options->languages', 'en')
        ->get();

如果您的應用程式使用 MariaDB、MySQL 或 PostgreSQL 資料庫，您可以將值陣列傳遞給 `whereJsonContains` 方法：

    $users = DB::table('users')
        ->whereJsonContains('options->languages', ['en', 'de'])
        ->get();

您可以使用 `whereJsonLength` 方法依據其長度查詢 JSON 陣列：

    $users = DB::table('users')
        ->whereJsonLength('options->languages', 0)
        ->get();

    $users = DB::table('users')
        ->whereJsonLength('options->languages', '>', 1)
        ->get();

<a name="additional-where-clauses"></a>
### 額外的 Where 子句

**whereLike / orWhereLike / whereNotLike / orWhereNotLike**

`whereLike` 方法允許您向查詢新增「LIKE」子句以進行模式匹配。這些方法提供了一種與資料庫無關的字串匹配查詢方式，並能夠切換大小寫敏感度。預設情況下，字串匹配不區分大小寫：

    $users = DB::table('users')
        ->whereLike('name', '%John%')
        ->get();

您可以透過 `caseSensitive` 參數啟用大小寫敏感搜尋：

    $users = DB::table('users')
        ->whereLike('name', '%John%', caseSensitive: true)
        ->get();

`orWhereLike` 方法允許您新增帶有 LIKE 條件的「or」子句：

    $users = DB::table('users')
        ->where('votes', '>', 100)
        ->orWhereLike('name', '%John%')
        ->get();

`whereNotLike` 方法允許您向查詢新增「NOT LIKE」子句：

    $users = DB::table('users')
        ->whereNotLike('name', '%John%')
        ->get();

同樣，您可以使用 `orWhereNotLike` 新增帶有 NOT LIKE 條件的「or」子句：

    $users = DB::table('users')
        ->where('votes', '>', 100)
        ->orWhereNotLike('name', '%John%')
        ->get();

> [!WARNING]
> `whereLike` 大小寫敏感搜尋選項目前不支援 SQL Server。

**whereIn / whereNotIn / orWhereIn / orWhereNotIn**

`whereIn` 方法驗證給定欄位的值是否包含在給定陣列中：

    $users = DB::table('users')
        ->whereIn('id', [1, 2, 3])
        ->get();

`whereNotIn` 方法驗證給定欄位的值是否不包含在給定陣列中：

    $users = DB::table('users')
        ->whereNotIn('id', [1, 2, 3])
        ->get();

您也可以將查詢物件作為 `whereIn` 方法的第二個參數提供：

    $activeUsers = DB::table('users')->select('id')->where('is_active', 1);

    $users = DB::table('comments')
        ->whereIn('user_id', $activeUsers)
        ->get();

上面的範例將產生以下 SQL：

```sql
select * from comments where user_id in (
    select id
    from users
    where is_active = 1
)
```

> [!WARNING]
> 如果您要向查詢新增大量整數綁定陣列，可以使用 `whereIntegerInRaw` 或 `whereIntegerNotInRaw` 方法來大幅減少記憶體使用量。

**whereBetween / orWhereBetween**

`whereBetween` 方法驗證欄位的值是否介於兩個值之間：

    $users = DB::table('users')
        ->whereBetween('votes', [1, 100])
        ->get();

**whereNotBetween / orWhereNotBetween**

`whereNotBetween` 方法驗證欄位的值是否在兩個值之外：

    $users = DB::table('users')
        ->whereNotBetween('votes', [1, 100])
        ->get();

**whereBetweenColumns / whereNotBetweenColumns / orWhereBetweenColumns / orWhereNotBetweenColumns**

`whereBetweenColumns` 方法驗證欄位的值是否介於同一資料表列中兩個欄位的值之間：

    $patients = DB::table('patients')
        ->whereBetweenColumns('weight', ['minimum_allowed_weight', 'maximum_allowed_weight'])
        ->get();

`whereNotBetweenColumns` 方法驗證欄位的值是否在同一資料表列中兩個欄位的值之外：

    $patients = DB::table('patients')
        ->whereNotBetweenColumns('weight', ['minimum_allowed_weight', 'maximum_allowed_weight'])
        ->get();

**whereNull / whereNotNull / orWhereNull / orWhereNotNull**

`whereNull` 方法驗證給定欄位的值是否為 `NULL`：

    $users = DB::table('users')
        ->whereNull('updated_at')
        ->get();

`whereNotNull` 方法驗證欄位的值是否不為 `NULL`：

    $users = DB::table('users')
        ->whereNotNull('updated_at')
        ->get();

**whereDate / whereMonth / whereDay / whereYear / whereTime**

`whereDate` 方法可用於將欄位的值與日期進行比較：

    $users = DB::table('users')
        ->whereDate('created_at', '2016-12-31')
        ->get();

`whereMonth` 方法可用於將欄位的值與特定月份進行比較：

    $users = DB::table('users')
        ->whereMonth('created_at', '12')
        ->get();

`whereDay` 方法可用於將欄位的值與月份的特定日期進行比較：

    $users = DB::table('users')
        ->whereDay('created_at', '31')
        ->get();

`whereYear` 方法可用於將欄位的值與特定年份進行比較：

    $users = DB::table('users')
        ->whereYear('created_at', '2016')
        ->get();

`whereTime` 方法可用於將欄位的值與特定時間進行比較：

    $users = DB::table('users')
        ->whereTime('created_at', '=', '11:20:45')
        ->get();

**wherePast / whereFuture / whereToday / whereBeforeToday / whereAfterToday**

`wherePast` 和 `whereFuture` 方法可用於判斷欄位的值是在過去還是未來：

    $invoices = DB::table('invoices')
        ->wherePast('due_at')
        ->get();

    $invoices = DB::table('invoices')
        ->whereFuture('due_at')
        ->get();

`whereNowOrPast` 和 `whereNowOrFuture` 方法可用於判斷欄位的值是在過去還是未來，包括當前日期和時間：

    $invoices = DB::table('invoices')
        ->whereNowOrPast('due_at')
        ->get();

    $invoices = DB::table('invoices')
        ->whereNowOrFuture('due_at')
        ->get();

`whereToday`、`whereBeforeToday` 和 `whereAfterToday` 方法可用於判斷欄位的值分別是今天、今天之前或今天之後：

    $invoices = DB::table('invoices')
        ->whereToday('due_at')
        ->get();

    $invoices = DB::table('invoices')
        ->whereBeforeToday('due_at')
        ->get();

    $invoices = DB::table('invoices')
        ->whereAfterToday('due_at')
        ->get();

同樣，`whereTodayOrBefore` 和 `whereTodayOrAfter` 方法可用於判斷欄位的值是在今天之前還是今天之後，包括今天的日期：

    $invoices = DB::table('invoices')
        ->whereTodayOrBefore('due_at')
        ->get();

    $invoices = DB::table('invoices')
        ->whereTodayOrAfter('due_at')
        ->get();

**whereColumn / orWhereColumn**

`whereColumn` 方法可用於驗證兩個欄位是否相等：

    $users = DB::table('users')
        ->whereColumn('first_name', 'last_name')
        ->get();

您也可以將比較運算子傳遞給 `whereColumn` 方法：

    $users = DB::table('users')
        ->whereColumn('updated_at', '>', 'created_at')
        ->get();

您也可以將欄位比較陣列傳遞給 `whereColumn` 方法。這些條件將使用 `and` 運算子連接：

    $users = DB::table('users')
        ->whereColumn([
            ['first_name', '=', 'last_name'],
            ['updated_at', '>', 'created_at'],
        ])->get();

<a name="logical-grouping"></a>
### 邏輯分組

有時您可能需要將幾個「where」子句分組在括號中，以實現查詢所需的邏輯分組。事實上，您通常應該始終將對 `orWhere` 方法的呼叫分組在括號中，以避免意外的查詢行為。為此，您可以將閉包傳遞給 `where` 方法：

    $users = DB::table('users')
        ->where('name', '=', 'John')
        ->where(function (Builder $query) {
            $query->where('votes', '>', 100)
                ->orWhere('title', '=', 'Admin');
        })
        ->get();

如您所見，將閉包傳遞給 `where` 方法會指示查詢產生器開始一個限制組。該閉包將接收一個查詢產生器實例，您可以使用它來設定應包含在括號組中的限制。上面的範例將產生以下 SQL：

```sql
select * from users where name = 'John' and (votes > 100 or title = 'Admin')
```

> [!WARNING]
> 您應該始終將 `orWhere` 呼叫分組，以避免在應用全域範圍時出現意外行為。

<a name="advanced-where-clauses"></a>
## 進階 Where 子句

<a name="where-exists-clauses"></a>
### Where Exists 子句

`whereExists` 方法允許您編寫「where exists」SQL 子句。`whereExists` 方法接受一個閉包，該閉包將接收一個查詢產生器實例，允許您定義應放置在「exists」子句內的查詢：

    $users = DB::table('users')
        ->whereExists(function (Builder $query) {
            $query->select(DB::raw(1))
                ->from('orders')
                ->whereColumn('orders.user_id', 'users.id');
        })
        ->get();

或者，您可以向 `whereExists` 方法提供查詢物件而不是閉包：

    $orders = DB::table('orders')
        ->select(DB::raw(1))
        ->whereColumn('orders.user_id', 'users.id');

    $users = DB::table('users')
        ->whereExists($orders)
        ->get();

上面兩個範例都將產生以下 SQL：

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

有時您可能需要建構一個「where」子句，該子句將子查詢的結果與給定值進行比較。您可以透過將閉包和值傳遞給 `where` 方法來實現此目的。例如，以下查詢將取得所有具有給定類型「membership」的最近使用者：

    use App\Models\User;
    use Illuminate\Database\Query\Builder;

    $users = User::where(function (Builder $query) {
        $query->select('type')
            ->from('membership')
            ->whereColumn('membership.user_id', 'users.id')
            ->orderByDesc('membership.start_date')
            ->limit(1);
    }, 'Pro')->get();

或者，您可能需要建構一個「where」子句，該子句將欄位與子查詢的結果進行比較。您可以透過將欄位、運算子和閉包傳遞給 `where` 方法來實現此目的。例如，以下查詢將取得所有金額小於平均值的收入記錄：

    use App\Models\Income;
    use Illuminate\Database\Query\Builder;

    $incomes = Income::where('amount', '<', function (Builder $query) {
        $query->selectRaw('avg(i.amount)')->from('incomes as i');
    })->get();

<a name="full-text-where-clauses"></a>
### 全文檢索 Where 子句

> [!WARNING]
> 全文檢索 Where 子句目前支援 MariaDB、MySQL 和 PostgreSQL。

`whereFullText` 和 `orWhereFullText` 方法可用於向查詢新增全文檢索「where」子句，用於具有[全文檢索索引](/docs/{{version}}/migrations#available-index-types)的欄位。這些方法將由 Laravel 轉換為底層資料庫系統的適當 SQL。例如，對於使用 MariaDB 或 MySQL 的應用程式，將產生 `MATCH AGAINST` 子句：

    $users = DB::table('users')
        ->whereFullText('bio', 'web developer')
        ->get();

<a name="ordering-grouping-limit-and-offset"></a>
## 排序、分組、限制與偏移

<a name="ordering"></a>
### 排序

<a name="orderby"></a>
#### `orderBy` 方法

`orderBy` 方法允許您依據給定欄位對查詢結果進行排序。`orderBy` 方法接受的第一個參數應該是您希望排序的欄位，而第二個參數決定排序方向，可以是 `asc` 或 `desc`：

    $users = DB::table('users')
        ->orderBy('name', 'desc')
        ->get();

若要依據多個欄位排序，您可以簡單地呼叫 `orderBy` 任意多次：

    $users = DB::table('users')
        ->orderBy('name', 'desc')
        ->orderBy('email', 'asc')
        ->get();

<a name="latest-oldest"></a>
#### `latest` 與 `oldest` 方法

`latest` 和 `oldest` 方法允許您輕鬆地依日期排序結果。預設情況下，結果將依資料表的 `created_at` 欄位排序。或者，您可以傳遞您希望排序的欄位名稱：

    $user = DB::table('users')
        ->latest()
        ->first();

<a name="random-ordering"></a>
#### 隨機排序

`inRandomOrder` 方法可用於隨機排序查詢結果。例如，您可以使用此方法來取得隨機使用者：

    $randomUser = DB::table('users')
        ->inRandomOrder()
        ->first();

<a name="removing-existing-orderings"></a>
#### 移除現有排序

`reorder` 方法會移除先前已應用於查詢的所有「order by」子句：

    $query = DB::table('users')->orderBy('name');

    $unorderedUsers = $query->reorder()->get();

您可以在呼叫 `reorder` 方法時傳遞欄位和方向，以移除所有現有的「order by」子句並將全新的順序應用於查詢：

    $query = DB::table('users')->orderBy('name');

    $usersOrderedByEmail = $query->reorder('email', 'desc')->get();

<a name="grouping"></a>
### 分組

<a name="groupby-having"></a>
#### `groupBy` 與 `having` 方法

正如您所預期的，`groupBy` 和 `having` 方法可用於分組查詢結果。`having` 方法的簽名與 `where` 方法類似：

    $users = DB::table('users')
        ->groupBy('account_id')
        ->having('account_id', '>', 100)
        ->get();

您可以使用 `havingBetween` 方法篩選給定範圍內的結果：

    $report = DB::table('orders')
        ->selectRaw('count(id) as number_of_orders, customer_id')
        ->groupBy('customer_id')
        ->havingBetween('number_of_orders', [5, 15])
        ->get();

您可以向 `groupBy` 方法傳遞多個參數以依多個欄位分組：

    $users = DB::table('users')
        ->groupBy('first_name', 'status')
        ->having('account_id', '>', 100)
        ->get();

若要建構更進階的 `having` 語句，請參閱 [`havingRaw`](#raw-methods) 方法。

<a name="limit-and-offset"></a>
### 限制與偏移

<a name="skip-take"></a>
#### `skip` 與 `take` 方法

您可以使用 `skip` 和 `take` 方法來限制查詢回傳的結果數量，或跳過查詢中給定數量的結果：

    $users = DB::table('users')->skip(10)->take(5)->get();

或者，您可以使用 `limit` 和 `offset` 方法。這些方法在功能上分別等同於 `take` 和 `skip` 方法：

    $users = DB::table('users')
        ->offset(10)
        ->limit(5)
        ->get();

<a name="conditional-clauses"></a>
## 條件式子句

有時您可能希望某些查詢子句根據另一個條件應用於查詢。例如，您可能只想在傳入的 HTTP 請求中存在給定輸入值時才應用 `where` 語句。您可以使用 `when` 方法來實現此目的：

    $role = $request->input('role');

    $users = DB::table('users')
        ->when($role, function (Builder $query, string $role) {
            $query->where('role_id', $role);
        })
        ->get();

`when` 方法僅在第一個參數為 `true` 時執行給定的閉包。如果第一個參數為 `false`，則閉包將不會執行。因此，在上面的範例中，傳遞給 `when` 方法的閉包僅在傳入請求中存在 `role` 欄位且其評估結果為 `true` 時才會被呼叫。

您可以將另一個閉包作為第三個參數傳遞給 `when` 方法。此閉包僅在第一個參數評估結果為 `false` 時執行。為了說明如何使用此功能，我們將使用它來設定查詢的預設排序：

    $sortByVotes = $request->boolean('sort_by_votes');

    $users = DB::table('users')
        ->when($sortByVotes, function (Builder $query, bool $sortByVotes) {
            $query->orderBy('votes');
        }, function (Builder $query) {
            $query->orderBy('name');
        })
        ->get();

<a name="insert-statements"></a>
## Insert 語句

查詢產生器還提供了一個 `insert` 方法，可用於將記錄插入到資料庫表中。`insert` 方法接受一個欄位名稱和值陣列：

    DB::table('users')->insert([
        'email' => 'kayla @example.com',
        'votes' => 0
    ]);

您可以透過傳遞一個陣列的陣列來一次插入多條記錄。每個陣列都代表應插入到資料表中的一條記錄：

    DB::table('users')->insert([
        ['email' => 'picard @example.com', 'votes' => 0],
        ['email' => 'janeway @example.com', 'votes' => 0],
    ]);

`insertOrIgnore` 方法將在將記錄插入資料庫時忽略錯誤。使用此方法時，您應該注意重複記錄錯誤將被忽略，並且根據資料庫引擎，其他類型的錯誤也可能被忽略。例如，`insertOrIgnore` 將[繞過 MySQL 的嚴格模式](https://dev.mysql.com/doc/refman/en/sql-mode.html#ignore-effect-on-execution)：

    DB::table('users')->insertOrIgnore([
        ['id' => 1, 'email' => 'sisko @example.com'],
        ['id' => 2, 'email' => 'archer @example.com'],
    ]);

`insertUsing` 方法將使用子查詢來確定應插入的資料，從而將新記錄插入到資料表中：

    DB::table('pruned_users')->insertUsing([
        'id', 'name', 'email', 'email_verified_at'
    ], DB::table('users')->select(
        'id', 'name', 'email', 'email_verified_at'
    )->where('updated_at', '<=', now()->subMonth()));

<a name="auto-incrementing-ids"></a>
#### 自動遞增 ID

如果資料表具有自動遞增 ID，請使用 `insertGetId` 方法插入記錄然後取得 ID：

    $id = DB::table('users')->insertGetId(
        ['email' => 'john @example.com', 'votes' => 0]
    );

> [!WARNING]
> 使用 PostgreSQL 時，`insertGetId` 方法預期自動遞增欄位名為 `id`。如果您想從不同的「sequence」取得 ID，您可以將欄位名稱作為第二個參數傳遞給 `insertGetId` 方法。

<a name="upserts"></a>
### Upserts

`upsert` 方法將插入不存在的記錄，並使用您可以指定的新值更新已存在的記錄。該方法的第一個參數包含要插入或更新的值，而第二個參數列出唯一識別相關資料表中記錄的欄位。該方法的第三個也是最後一個參數是一個欄位陣列，如果資料庫中已存在匹配的記錄，則應更新這些欄位：

    DB::table('flights')->upsert(
        [
            ['departure' => 'Oakland', 'destination' => 'San Diego', 'price' => 99],
            ['departure' => 'Chicago', 'destination' => 'New York', 'price' => 150]
        ],
        ['departure', 'destination'],
        ['price']
    );

在上面的範例中，Laravel 將嘗試插入兩條記錄。如果已存在具有相同 `departure` 和 `destination` 欄位值的記錄，Laravel 將更新該記錄的 `price` 欄位。

> [!WARNING]
> 除 SQL Server 外，所有資料庫都要求 `upsert` 方法的第二個參數中的欄位具有「primary」或「unique」索引。此外，MariaDB 和 MySQL 資料庫驅動程式會忽略 `upsert` 方法的第二個參數，並始終使用資料表的「primary」和「unique」索引來偵測現有記錄。

<a name="update-statements"></a>
## Update 語句

除了將記錄插入資料庫外，查詢產生器還可以使用 `update` 方法更新現有記錄。`update` 方法與 `insert` 方法一樣，接受一個欄位和值對陣列，指示要更新的欄位。`update` 方法回傳受影響的列數。您可以使用 `where` 子句限制 `update` 查詢：

    $affected = DB::table('users')
        ->where('id', 1)
        ->update(['votes' => 1]);

<a name="update-or-insert"></a>
#### 更新或插入

有時您可能希望更新資料庫中的現有記錄，或者如果不存在匹配的記錄則建立它。在這種情況下，可以使用 `updateOrInsert` 方法。`updateOrInsert` 方法接受兩個參數：一個用於查找記錄的條件陣列，以及一個欄位和值對陣列，指示要更新的欄位。

`updateOrInsert` 方法將嘗試使用第一個參數的欄位和值對來定位匹配的資料庫記錄。如果記錄存在，它將使用第二個參數中的值進行更新。如果找不到記錄，則將插入一個新記錄，其中包含兩個參數的合併屬性：

    DB::table('users')
        ->updateOrInsert(
            ['email' => 'john @example.com', 'name' => 'John'],
            ['votes' => '2']
        );

您可以向 `updateOrInsert` 方法提供一個閉包，以根據匹配記錄的存在來自訂更新或插入到資料庫中的屬性：

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

    $affected = DB::table('users')
        ->where('id', 1)
        ->update(['options->enabled' => true]);

<a name="increment-and-decrement"></a>
### 遞增與遞減

查詢產生器還提供了方便的方法來遞增或遞減給定欄位的值。這兩個方法都至少接受一個參數：要修改的欄位。可以提供第二個參數來指定欄位應遞增或遞減的數量：

    DB::table('users')->increment('votes');

    DB::table('users')->increment('votes', 5);

    DB::table('users')->decrement('votes');

    DB::table('users')->decrement('votes', 5);

如果需要，您還可以指定在遞增或遞減操作期間要更新的其他欄位：

    DB::table('users')->increment('votes', 1, ['name' => 'John']);

此外，您可以使用 `incrementEach` 和 `decrementEach` 方法一次遞增或遞減多個欄位：

    DB::table('users')->incrementEach([
        'votes' => 5,
        'balance' => 100,
    ]);

<a name="delete-statements"></a>
## Delete 語句

查詢產生器的 `delete` 方法可用於從資料表中刪除記錄。`delete` 方法回傳受影響的列數。您可以透過在呼叫 `delete` 方法之前新增「where」子句來限制 `delete` 語句：

    $deleted = DB::table('users')->delete();

    $deleted = DB::table('users')->where('votes', '>', 100)->delete();

<a name="pessimistic-locking"></a>
## 悲觀鎖定

查詢產生器還包含一些函數，可幫助您在執行 `select` 語句時實現「悲觀鎖定」。若要執行帶有「共享鎖定」的語句，您可以呼叫 `sharedLock` 方法。共享鎖定會阻止選定的資料列被修改，直到您的交易提交：

    DB::table('users')
        ->where('votes', '>', 100)
        ->sharedLock()
        ->get();

或者，您可以使用 `lockForUpdate` 方法。「for update」鎖定會阻止選定的記錄被修改或被另一個共享鎖定選取：

    DB::table('users')
        ->where('votes', '>', 100)
        ->lockForUpdate()
        ->get();

雖然不是強制性的，但建議將悲觀鎖定包裝在[交易](/docs/{{version}}/database#database-transactions)中。這可確保在整個操作完成之前，從資料庫中取得的資料保持不變。如果發生故障，交易將回滾任何變更並自動釋放鎖定：

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

<a name="debugging"></a>
## 除錯

您可以在建構查詢時使用 `dd` 和 `dump` 方法來傾印當前的查詢綁定和 SQL。`dd` 方法將顯示除錯資訊然後停止執行請求。`dump` 方法將顯示除錯資訊但允許請求繼續執行：

    DB::table('users')->where('votes', '>', 100)->dd();

    DB::table('users')->where('votes', '>', 100)->dump();

`dumpRawSql` 和 `ddRawSql` 方法可以在查詢上呼叫，以傾印查詢的 SQL，並正確替換所有參數綁定：

    DB::table('users')->where('votes', '>', 100)->dumpRawSql();

    DB::table('users')->where('votes', '>', 100)->ddRawSql();
