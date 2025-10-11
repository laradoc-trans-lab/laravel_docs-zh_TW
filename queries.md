# 資料庫：查詢產生器

- [介紹](#introduction)
- [執行資料庫查詢](#running-database-queries)
    - [分塊處理結果](#chunking-results)
    - [惰性串流處理結果](#streaming-results-lazily)
    - [聚合](#aggregates)
- [Select 語句](#select-statements)
- [原始表達式](#raw-expressions)
- [連接](#joins)
- [聯集](#unions)
- [基本 Where 條件](#basic-where-clauses)
    - [Where 條件](#where-clauses)
    - [Or Where 條件](#or-where-clauses)
    - [Where Not 條件](#where-not-clauses)
    - [Where Any / All / None 條件](#where-any-all-none-clauses)
    - [JSON Where 條件](#json-where-clauses)
    - [其他 Where 條件](#additional-where-clauses)
    - [邏輯分組](#logical-grouping)
- [進階 Where 條件](#advanced-where-clauses)
    - [Where Exists 條件](#where-exists-clauses)
    - [子查詢 Where 條件](#subquery-where-clauses)
    - [全文檢索 Where 條件](#full-text-where-clauses)
- [排序、分組、限制與偏移](#ordering-grouping-limit-and-offset)
    - [排序](#ordering)
    - [分組](#grouping)
    - [限制與偏移](#limit-and-offset)
- [條件式子句](#conditional-clauses)
- [Insert 語句](#insert-statements)
    - [新增或更新 (Upserts)](#upserts)
- [Update 語句](#update-statements)
    - [更新 JSON 欄位](#updating-json-columns)
    - [遞增與遞減](#increment-and-decrement)
- [Delete 語句](#delete-statements)
- [悲觀鎖定](#pessimistic-locking)
- [除錯](#debugging)

<a name="introduction"></a>
## 介紹

Laravel 的資料庫查詢產生器提供了一個方便、流暢的介面，用於建立和執行資料庫查詢。它可以用來執行應用程式中的大部分資料庫操作，並能與所有 Laravel 支援的資料庫系統完美運作。

Laravel 查詢產生器使用 PDO 參數綁定來保護你的應用程式免受 SQL 隱碼攻擊。對於傳遞給查詢產生器作為查詢綁定的字串，無需進行清理或消毒。

> [!WARNING]  
> PDO 不支援綁定欄位名稱。因此，你絕不應該允許使用者輸入來決定你的查詢所參考的欄位名稱，包括「order by」欄位。

<a name="running-database-queries"></a>
## 執行資料庫查詢

<a name="retrieving-all-rows-from-a-table"></a>
#### 從資料表中擷取所有列

您可以使用 `DB` Facade 提供的 `table` 方法來開始查詢。`table` 方法會針對指定資料表回傳一個流暢的查詢產生器實例，讓您能夠將更多條件鏈接到查詢上，然後最終使用 `get` 方法來擷取查詢結果：

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

`get` 方法會回傳一個 `Illuminate\Support\Collection` 實例，其中包含查詢的結果，每個結果都是一個 PHP `stdClass` 物件的實例。您可以透過存取物件的屬性來存取每個欄位的值：

    use Illuminate\Support\Facades\DB;

    $users = DB::table('users')->get();

    foreach ($users as $user) {
        echo $user->name;
    }

> [!NOTE]
> Laravel Collection 提供各種極其強大的方法來對資料進行映射與縮減。有關 Laravel Collection 的更多資訊，請查閱 [Collection 文件](/docs/{{version}}/collections)。

<a name="retrieving-a-single-row-column-from-a-table"></a>
#### 從資料表中擷取單一列 / 欄位

如果您只需要從資料庫中擷取單一列，您可以使用 `DB` Facade 的 `first` 方法。此方法會回傳一個單一的 `stdClass` 物件：

    $user = DB::table('users')->where('name', 'John')->first();

    return $user->email;

如果您想從資料庫中擷取單一列，但在找不到符合的列時拋出 `Illuminate\Database\RecordNotFoundException`，您可以使用 `firstOrFail` 方法。如果 `RecordNotFoundException` 未被捕捉，則會自動向用戶端傳送 404 HTTP 回應：

    $user = DB::table('users')->where('name', 'John')->firstOrFail();

如果您不需要完整的列，您可以使用 `value` 方法從記錄中提取單一值。此方法會直接回傳欄位的值：

    $email = DB::table('users')->where('name', 'John')->value('email');

要透過其 `id` 欄位值來擷取單一列，請使用 `find` 方法：

    $user = DB::table('users')->find(3);

<a name="retrieving-a-list-of-column-values"></a>
#### 擷取欄位值的列表

如果您想擷取一個包含單一欄位值的 `Illuminate\Support\Collection` 實例，您可以使用 `pluck` 方法。在這個範例中，我們將擷取使用者標題的 Collection：

    use Illuminate\Support\Facades\DB;

    $titles = DB::table('users')->pluck('title');

    foreach ($titles as $title) {
        echo $title;
    }

您可以透過向 `pluck` 方法提供第二個參數，來指定結果 Collection 應使用的欄位作為其鍵：

    $titles = DB::table('users')->pluck('title', 'name');

    foreach ($titles as $name => $title) {
        echo $title;
    }

<a name="chunking-results"></a>
### 分塊處理結果

如果您需要處理數千條資料庫記錄，請考慮使用 `DB` Facade 提供的 `chunk` 方法。此方法會一次擷取一小塊結果，並將每塊結果傳遞給一個閉包進行處理。例如，讓我們以每次 100 條記錄的分塊方式來擷取整個 `users` 資料表：

    use Illuminate\Support\Collection;
    use Illuminate\Support\Facades\DB;

    DB::table('users')->orderBy('id')->chunk(100, function (Collection $users) {
        foreach ($users as $user) {
            // ...
        }
    });

您可以透過從閉包中回傳 `false` 來停止處理後續的分塊：

    DB::table('users')->orderBy('id')->chunk(100, function (Collection $users) {
        // Process the records...

        return false;
    });

如果您在分塊處理結果時更新資料庫記錄，您的分塊結果可能會以意想不到的方式改變。如果您打算在分塊處理時更新擷取的記錄，最好使用 `chunkById` 方法。此方法會根據記錄的主鍵自動對結果進行分頁：

    DB::table('users')->where('active', false)
        ->chunkById(100, function (Collection $users) {
            foreach ($users as $user) {
                DB::table('users')
                    ->where('id', $user->id)
                    ->update(['active' => true]);
            }
        });

由於 `chunkById` 和 `lazyById` 方法會將自己的「where」條件添加到正在執行的查詢中，您通常應該在閉包中 [邏輯分組](#logical-grouping) 您自己的條件：

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
> 在分塊回呼中更新或刪除記錄時，對主鍵或外鍵的任何更改都可能會影響分塊查詢。這可能會導致某些記錄未包含在分塊結果中。

<a name="streaming-results-lazily"></a>
### 惰性串流處理結果

`lazy` 方法的運作方式與 [the `chunk` method](#chunking-results) 類似，它會分塊執行查詢。然而，`lazy()` 方法不是將每個分塊傳遞給回呼，而是回傳一個 [`LazyCollection`](/docs/{{version}}/collections#lazy-collections)，讓您可以將結果作為單一串流來互動：

```php
use Illuminate\Support\Facades\DB;

DB::table('users')->orderBy('id')->lazy()->each(function (object $user) {
    // ...
});
```

再一次，如果您打算在迭代記錄時更新擷取的記錄，最好使用 `lazyById` 或 `lazyByIdDesc` 方法。這些方法會根據記錄的主鍵自動對結果進行分頁：

```php
DB::table('users')->where('active', false)
    ->lazyById()->each(function (object $user) {
        DB::table('users')
            ->where('id', $user->id)
            ->update(['active' => true]);
    });
```

> [!WARNING]
> 在迭代記錄時更新或刪除記錄時，對主鍵或外鍵的任何更改都可能會影響分塊查詢。這可能會導致某些記錄未包含在結果中。

<a name="aggregates"></a>
### 聚合

查詢產生器還提供了多種方法，用於擷取 `count`、`max`、`min`、`avg` 和 `sum` 等聚合值。您可以在建構查詢後呼叫任何這些方法：

    use Illuminate\Support\Facades\DB;

    $users = DB::table('users')->count();

    $price = DB::table('orders')->max('price');

當然，您可以將這些方法與其他子句結合使用，以微調聚合值的計算方式：

    $price = DB::table('orders')
        ->where('finalized', 1)
        ->avg('price');

<a name="determining-if-records-exist"></a>
#### 判斷記錄是否存在

您可以使用 `exists` 和 `doesntExist` 方法來判斷是否存在符合您查詢條件的記錄，而不是使用 `count` 方法：

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

您可能不總是希望從資料庫表格中選取所有欄位。使用 `select` 方法，您可以為查詢指定一個自訂的「select」子句：

    use Illuminate\Support\Facades\DB;

    $users = DB::table('users')
        ->select('name', 'email as user_email')
        ->get();

`distinct` 方法可讓您強制查詢傳回不同的結果：

    $users = DB::table('users')->distinct()->get();

如果您已經有一個查詢產生器實例，並且希望將欄位新增到其現有的 select 子句中，您可以使用 `addSelect` 方法：

    $query = DB::table('users')->select('name');

    $users = $query->addSelect('age')->get();

<a name="raw-expressions"></a>
## 原始表達式

有時候您可能需要在查詢中插入任意字串。若要建立原始字串表達式，您可以使用 `DB` Facade 提供的 `raw` 方法：

    $users = DB::table('users')
        ->select(DB::raw('count(*) as user_count, status'))
        ->where('status', '<>', 1)
        ->groupBy('status')
        ->get();

> [!WARNING]  
> 原始語句將作為字串注入到查詢中，因此您應該非常小心，避免建立 SQL 注入漏洞。

<a name="raw-methods"></a>
### 原始方法

除了使用 `DB::raw` 方法之外，您還可以使用以下方法將原始表達式插入到查詢的各個部分。**請記住，Laravel 無法保證任何使用原始表達式的查詢都能防止 SQL 注入漏洞。**

<a name="selectraw"></a>
#### `selectRaw`

`selectRaw` 方法可以取代 `addSelect(DB::raw(/* ... */))`。此方法接受一個選用的繫結陣列作為其第二個參數：

    $orders = DB::table('orders')
        ->selectRaw('price * ? as price_with_tax', [1.0825])
        ->get();

<a name="whereraw-orwhereraw"></a>
#### `whereRaw / orWhereRaw`

`whereRaw` 和 `orWhereRaw` 方法可用於將原始的「where」子句注入到查詢中。這些方法接受一個選用的繫結陣列作為其第二個參數：

    $orders = DB::table('orders')
        ->whereRaw('price > IF(state = "TX", ?, 100)', [200])
        ->get();

<a name="havingraw-orhavingraw"></a>
#### `havingRaw / orHavingRaw`

`havingRaw` 和 `orHavingRaw` 方法可用於提供一個原始字串作為「having」子句的值。這些方法接受一個選用的繫結陣列作為其第二個參數：

    $orders = DB::table('orders')
        ->select('department', DB::raw('SUM(price) as total_sales'))
        ->groupBy('department')
        ->havingRaw('SUM(price) > ?', [2500])
        ->get();

<a name="orderbyraw"></a>
#### `orderByRaw`

`orderByRaw` 方法可用於提供一個原始字串作為「order by」子句的值：

    $orders = DB::table('orders')
        ->orderByRaw('updated_at - created_at DESC')
        ->get();

<a name="groupbyraw"></a>
### `groupByRaw`

`groupByRaw` 方法可用於提供一個原始字串作為 `group by` 子句的值：

    $orders = DB::table('orders')
        ->select('city', 'state')
        ->groupByRaw('city, state')
        ->get();

<a name="joins"></a>
## 連接

<a name="inner-join-clause"></a>
#### 內連接 (Inner Join) 子句

查詢產生器也可用於為您的查詢新增連接子句。若要執行基本的「內連接 (inner join)」，您可以在查詢產生器實例上使用 `join` 方法。傳遞給 `join` 方法的第一個參數是您需要連接的表格名稱，而其餘參數則指定連接的欄位條件。您甚至可以在單一查詢中連接多個表格：

    use Illuminate\Support\Facades\DB;

    $users = DB::table('users')
        ->join('contacts', 'users.id', '=', 'contacts.user_id')
        ->join('orders', 'users.id', '=', 'orders.user_id')
        ->select('users.*', 'contacts.phone', 'orders.price')
        ->get();

<a name="left-join-right-join-clause"></a>
#### 左連接 (Left Join) / 右連接 (Right Join) 子句

如果您想執行「左連接 (left join)」或「右連接 (right join)」而不是「內連接 (inner join)」，請使用 `leftJoin` 或 `rightJoin` 方法。這些方法與 `join` 方法具有相同的簽名：

    $users = DB::table('users')
        ->leftJoin('posts', 'users.id', '=', 'posts.user_id')
        ->get();

    $users = DB::table('users')
        ->rightJoin('posts', 'users.id', '=', 'posts.user_id')
        ->get();

<a name="cross-join-clause"></a>
#### 交叉連接 (Cross Join) 子句

您可以使用 `crossJoin` 方法執行「交叉連接 (cross join)」。交叉連接會在第一個表格與連接的表格之間產生笛卡兒積：

    $sizes = DB::table('sizes')
        ->crossJoin('colors')
        ->get();

<a name="advanced-join-clauses"></a>
#### 進階連接子句

您也可以指定更進階的連接子句。若要開始，請將閉包作為第二個參數傳遞給 `join` 方法。該閉包將接收一個 `Illuminate\Database\Query\JoinClause` 實例，允許您指定「join」子句上的約束：

    DB::table('users')
        ->join('contacts', function (JoinClause $join) {
            $join->on('users.id', '=', 'contacts.user_id')->orOn(/* ... */);
        })
        ->get();

如果您希望在連接上使用「where」子句，您可以使用 `JoinClause` 實例提供的 `where` 和 `orWhere` 方法。這些方法不會比較兩個欄位，而是將欄位與值進行比較：

    DB::table('users')
        ->join('contacts', function (JoinClause $join) {
            $join->on('users.id', '=', 'contacts.user_id')
                ->where('contacts.user_id', '>', 5);
        })
        ->get();

<a name="subquery-joins"></a>
#### 子查詢連接

您可以使用 `joinSub`、`leftJoinSub` 和 `rightJoinSub` 方法將查詢連接到子查詢。這些方法都接收三個參數：子查詢、其表格別名，以及定義相關欄位的閉包。在此範例中，我們將檢索使用者集合，其中每個使用者記錄還包含使用者最近發布的部落格文章的 `created_at` 時間戳：

    $latestPosts = DB::table('posts')
        ->select('user_id', DB::raw('MAX(created_at) as last_post_created_at'))
        ->where('is_published', true)
        ->groupBy('user_id');

    $users = DB::table('users')
        ->joinSub($latestPosts, 'latest_posts', function (JoinClause $join) {
            $join->on('users.id', '=', 'latest_posts.user_id');
        })->get();

<a name="lateral-joins"></a>
#### 側向連接 (Lateral Joins)

> [!WARNING]  
> 側向連接目前支援 PostgreSQL、MySQL >= 8.0.14 和 SQL Server。

您可以使用 `joinLateral` 和 `leftJoinLateral` 方法執行帶有子查詢的「側向連接 (lateral join)」。這些方法各接收兩個參數：子查詢及其表格別名。連接條件應在給定子查詢的 `where` 子句中指定。側向連接會針對每一行進行評估，並且可以引用子查詢外部的欄位。

在此範例中，我們將檢索使用者集合以及該使用者的三篇最新部落格文章。每個使用者在結果集中最多可產生三行：每篇最新部落格文章一行。連接條件在子查詢中透過 `whereColumn` 子句指定，引用目前的使用者行：

    $latestPosts = DB::table('posts')
        ->select('id as post_id', 'title as post_title', 'created_at as post_created_at')
        ->whereColumn('user_id', 'users.id')
        ->orderBy('created_at', 'desc')
        ->limit(3);

    $users = DB::table('users')
        ->joinLateral($latestPosts, 'latest_posts')
        ->get();

<a name="unions"></a>
## 聯集

查詢產生器也提供了一個便捷的方法來將一個或多個查詢進行「聯集」(union) 操作。例如，您可以建立一個初始查詢，然後使用 `union` 方法將其與更多查詢進行聯集：

    use Illuminate\Support\Facades\DB;

    $first = DB::table('users')
        ->whereNull('first_name');

    $users = DB::table('users')
        ->whereNull('last_name')
        ->union($first)
        ->get();

除了 `union` 方法之外，查詢產生器還提供了 `unionAll` 方法。使用 `unionAll` 方法組合的查詢將不會移除重複的結果。`unionAll` 方法與 `union` 方法具有相同的方法簽章。

<a name="basic-where-clauses"></a>
## 基本 Where 條件


<a name="where-clauses"></a>
### Where 條件

您可以使用查詢產生器的 `where` 方法將 "where" 條件加入查詢中。對 `where` 方法最基本的呼叫需要三個參數。第一個參數是欄位名稱。第二個參數是運算子，可以是資料庫支援的任何運算子。第三個參數是用來與欄位值比較的值。

舉例來說，以下查詢會取得 `votes` 欄位值等於 `100` 且 `age` 欄位值大於 `35` 的使用者：

    $users = DB::table('users')
        ->where('votes', '=', 100)
        ->where('age', '>', 35)
        ->get();

為了方便起見，如果您想驗證某個欄位是否 `=` 某個給定值，您可以將該值作為 `where` 方法的第二個參數傳入。Laravel 會假定您想使用 `=` 運算子：

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

您也可以傳遞條件陣列給 `where` 函式。陣列中的每個元素都應是包含通常傳遞給 `where` 方法的三個參數的陣列：

    $users = DB::table('users')->where([
        ['status', '=', '1'],
        ['subscribed', '<>', '1'],
    ])->get();

> [!WARNING]  
> PDO 不支援綁定欄位名稱。因此，您絕不應允許使用者輸入來決定查詢中引用的欄位名稱，包括「order by」欄位。

> [!WARNING]
> MySQL 和 MariaDB 在字串與數字比較時會自動將字串轉換為整數。在此過程中，非數值字串會被轉換為 `0`，這可能導致意外結果。例如，如果您的資料表有一個 `secret` 欄位，其值為 `aaa`，並且您執行 `User::where('secret', 0)`，該行將會被返回。為避免這種情況，請確保所有值在查詢中使用前都已轉換為其適當的類型。


<a name="or-where-clauses"></a>
### Or Where 條件

當查詢產生器的 `where` 方法呼叫鏈接在一起時，這些 "where" 條件將會使用 `and` 運算子連接。然而，您可以使用 `orWhere` 方法來使用 `or` 運算子將一個條件加入查詢中。`orWhere` 方法接受與 `where` 方法相同的參數：

    $users = DB::table('users')
        ->where('votes', '>', 100)
        ->orWhere('name', 'John')
        ->get();

如果您需要將 "or" 條件分組在括號內，您可以將閉包作為 `orWhere` 方法的第一個參數傳入：

    $users = DB::table('users')
        ->where('votes', '>', 100)
        ->orWhere(function (Builder $query) {
            $query->where('name', 'Abigail')
                ->where('votes', '>', 50);
            })
        ->get();

上述範例將產生以下 SQL：

```sql
select * from users where votes > 100 or (name = 'Abigail' and votes > 50)
```

> [!WARNING]  
> 您應始終將 `orWhere` 呼叫分組，以避免在套用全域範圍時發生非預期行為。


<a name="where-not-clauses"></a>
### Where Not 條件

`whereNot` 和 `orWhereNot` 方法可用於否定給定的查詢條件組。舉例來說，以下查詢會排除正在促銷中或價格低於十的商品：

    $products = DB::table('products')
        ->whereNot(function (Builder $query) {
            $query->where('clearance', true)
                ->orWhere('price', '<', 10);
            })
        ->get();


<a name="where-any-all-none-clauses"></a>
### Where Any / All / None 條件

有時候，您可能需要將相同的查詢條件套用至多個欄位。例如，您可能希望擷取在指定清單中任何欄位 `LIKE` 給定值的所有記錄。您可以使用 `whereAny` 方法來實現此目的：

    $users = DB::table('users')
        ->where('active', true)
        ->whereAny([
            'name',
            'email',
            'phone',
        ], 'like', 'Example%')
        ->get();

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

同樣地，`whereAll` 方法可用於擷取所有給定欄位都符合指定條件的記錄：

    $posts = DB::table('posts')
        ->where('published', true)
        ->whereAll([
            'title',
            'content',
        ], 'like', '%Laravel%')
        ->get();

上述查詢將產生以下 SQL：

```sql
SELECT *
FROM posts
WHERE published = true AND (
    title LIKE '%Laravel%' AND
    content LIKE '%Laravel%'
)
```

`whereNone` 方法可用於擷取所有給定欄位都不符合指定條件的記錄：

    $posts = DB::table('albums')
        ->where('published', true)
        ->whereNone([
            'title',
            'lyrics',
            'tags',
        ], 'like', '%explicit%')
        ->get();

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
### JSON Where 條件

Laravel 也支援在提供 JSON 欄位類型支援的資料庫上查詢 JSON 欄位類型。目前，這包括 MariaDB 10.3+、MySQL 8.0+、PostgreSQL 12.0+、SQL Server 2017+ 和 SQLite 3.39.0+。要查詢 JSON 欄位，請使用 `->` 運算子：

    $users = DB::table('users')
        ->where('preferences->dining->meal', 'salad')
        ->get();

您可以使用 `whereJsonContains` 來查詢 JSON 陣列：

    $users = DB::table('users')
        ->whereJsonContains('options->languages', 'en')
        ->get();

如果您的應用程式使用 MariaDB、MySQL 或 PostgreSQL 資料庫，您可以傳遞一個值陣列給 `whereJsonContains` 方法：

    $users = DB::table('users')
        ->whereJsonContains('options->languages', ['en', 'de'])
        ->get();

您可以使用 `whereJsonLength` 方法，依 JSON 陣列的長度進行查詢：

    $users = DB::table('users')
        ->whereJsonLength('options->languages', 0)
        ->get();

    $users = DB::table('users')
        ->whereJsonLength('options->languages', '>', 1)
        ->get();

<a name="additional-where-clauses"></a>
### 其他 Where 條件

**whereLike / orWhereLike / whereNotLike / orWhereNotLike**

`whereLike` 方法允許您在查詢中添加 "LIKE" 條件以進行模式匹配。這些方法提供了一種與資料庫無關的方式來執行字串匹配查詢，並能夠切換大小寫敏感性。預設情況下，字串匹配不區分大小寫：

    $users = DB::table('users')
        ->whereLike('name', '%John%')
        ->get();

您可以透過 `caseSensitive` 參數啟用區分大小寫的搜尋：

    $users = DB::table('users')
        ->whereLike('name', '%John%', caseSensitive: true)
        ->get();

`orWhereLike` 方法允許您添加一個帶有 LIKE 條件的 "or" 子句：

    $users = DB::table('users')
        ->where('votes', '>', 100)
        ->orWhereLike('name', '%John%')
        ->get();

`whereNotLike` 方法允許您在查詢中添加 "NOT LIKE" 條件：

    $users = DB::table('users')
        ->whereNotLike('name', '%John%')
        ->get();

同樣地，您可以使用 `orWhereNotLike` 添加一個帶有 NOT LIKE 條件的 "or" 子句：

    $users = DB::table('users')
        ->where('votes', '>', 100)
        ->orWhereNotLike('name', '%John%')
        ->get();

> [!WARNING]
> `whereLike` 的大小寫敏感搜尋選項目前不支援 SQL Server。

**whereIn / whereNotIn / orWhereIn / orWhereNotIn**

`whereIn` 方法會驗證給定欄位的值是否包含在給定陣列中：

    $users = DB::table('users')
        ->whereIn('id', [1, 2, 3])
        ->get();

`whereNotIn` 方法會驗證給定欄位的值是否不包含在給定陣列中：

    $users = DB::table('users')
        ->whereNotIn('id', [1, 2, 3])
        ->get();

您也可以將查詢物件作為 `whereIn` 方法的第二個參數：

    $activeUsers = DB::table('users')->select('id')->where('is_active', 1);

    $users = DB::table('comments')
        ->whereIn('user_id', $activeUsers)
        ->get();

上述範例將產生以下 SQL：

```sql
select * from comments where user_id in (
    select id
    from users
    where is_active = 1
)
```

> [!WARNING]
> 如果您正在為查詢添加大量整數綁定，可以使用 `whereIntegerInRaw` 或 `whereIntegerNotInRaw` 方法，這可以大大減少記憶體使用量。

**whereBetween / orWhereBetween**

`whereBetween` 方法會驗證欄位的值是否介於兩個值之間：

    $users = DB::table('users')
        ->whereBetween('votes', [1, 100])
        ->get();

**whereNotBetween / orWhereNotBetween**

`whereNotBetween` 方法會驗證欄位的值是否不介於兩個值之間：

    $users = DB::table('users')
        ->whereNotBetween('votes', [1, 100])
        ->get();

**whereBetweenColumns / whereNotBetweenColumns / orWhereBetweenColumns / orWhereNotBetweenColumns**

`whereBetweenColumns` 方法會驗證欄位的值是否介於同一表格列中兩個欄位的值之間：

    $patients = DB::table('patients')
        ->whereBetweenColumns('weight', ['minimum_allowed_weight', 'maximum_allowed_weight'])
        ->get();

`whereNotBetweenColumns` 方法會驗證欄位的值是否不介於同一表格列中兩個欄位的值之間：

    $patients = DB::table('patients')
        ->whereNotBetweenColumns('weight', ['minimum_allowed_weight', 'maximum_allowed_weight'])
        ->get();

**whereNull / whereNotNull / orWhereNull / orWhereNotNull**

`whereNull` 方法會驗證給定欄位的值是否為 `NULL`：

    $users = DB::table('users')
        ->whereNull('updated_at')
        ->get();

`whereNotNull` 方法會驗證欄位的值是否不為 `NULL`：

    $users = DB::table('users')
        ->whereNotNull('updated_at')
        ->get();

**whereDate / whereMonth / whereDay / whereYear / whereTime**

`whereDate` 方法可以用來比較欄位的值與日期：

    $users = DB::table('users')
        ->whereDate('created_at', '2016-12-31')
        ->get();

`whereMonth` 方法可以用來比較欄位的值與特定月份：

    $users = DB::table('users')
        ->whereMonth('created_at', '12')
        ->get();

`whereDay` 方法可以用來比較欄位的值與特定日期：

    $users = DB::table('users')
        ->whereDay('created_at', '31')
        ->get();

`whereYear` 方法可以用來比較欄位的值與特定年份：

    $users = DB::table('users')
        ->whereYear('created_at', '2016')
        ->get();

`whereTime` 方法可以用來比較欄位的值與特定時間：

    $users = DB::table('users')
        ->whereTime('created_at', '=', '11:20:45')
        ->get();

**wherePast / whereFuture / whereToday / whereBeforeToday / whereAfterToday**

`wherePast` 和 `whereFuture` 方法可以用來判斷欄位的值是在過去還是未來：

    $invoices = DB::table('invoices')
        ->wherePast('due_at')
        ->get();

    $invoices = DB::table('invoices')
        ->whereFuture('due_at')
        ->get();

`whereNowOrPast` 和 `whereNowOrFuture` 方法可以用來判斷欄位的值是在過去還是未來，包含當前的日期和時間：

    $invoices = DB::table('invoices')
        ->whereNowOrPast('due_at')
        ->get();

    $invoices = DB::table('invoices')
        ->whereNowOrFuture('due_at')
        ->get();

`whereToday`、`whereBeforeToday` 和 `whereAfterToday` 方法可以用來判斷欄位的值分別是今天、今天之前或今天之後：

    $invoices = DB::table('invoices')
        ->whereToday('due_at')
        ->get();

    $invoices = DB::table('invoices')
        ->whereBeforeToday('due_at')
        ->get();

    $invoices = DB::table('invoices')
        ->whereAfterToday('due_at')
        ->get();

同樣地，`whereTodayOrBefore` 和 `whereTodayOrAfter` 方法可以用來判斷欄位的值是在今天或今天之前，或是在今天或今天之後，包含今天：

    $invoices = DB::table('invoices')
        ->whereTodayOrBefore('due_at')
        ->get();

    $invoices = DB::table('invoices')
        ->whereTodayOrAfter('due_at')
        ->get();

**whereColumn / orWhereColumn**

`whereColumn` 方法可以用來驗證兩個欄位是否相等：

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

有時候，您可能需要將多個「where」條件分組到括號內，以實現查詢所需的邏輯分組。事實上，您通常應該總是將 `orWhere` 方法的呼叫分組到括號中，以避免非預期的查詢行為。為此，您可以將一個閉包傳遞給 `where` 方法：

    $users = DB::table('users')
        ->where('name', '=', 'John')
        ->where(function (Builder $query) {
            $query->where('votes', '>', 100)
                ->orWhere('title', '=', 'Admin');
        })
        ->get();

如您所見，將閉包傳遞給 `where` 方法會指示查詢產生器開始一個約束分組。此閉包將接收一個查詢產生器實例，您可以使用它來設定應包含在括號分組內的約束。上述範例將產生以下 SQL：

```sql
select * from users where name = 'John' and (votes > 100 or title = 'Admin')
```

> [!WARNING]  
> 您應該總是對 `orWhere` 呼叫進行分組，以避免在套用全域範圍時發生非預期的行為。

<a name="advanced-where-clauses"></a>
## 進階 Where 條件

<a name="where-exists-clauses"></a>
### Where Exists 條件

`whereExists` 方法允許您撰寫「where exists」SQL 子句。`whereExists` 方法接受一個閉包，該閉包會接收一個查詢產生器實例，讓您可以定義應放置在「exists」子句內的查詢：

    $users = DB::table('users')
        ->whereExists(function (Builder $query) {
            $query->select(DB::raw(1))
                ->from('orders')
                ->whereColumn('orders.user_id', 'users.id');
        })
        ->get();

或者，您可以將一個查詢物件提供給 `whereExists` 方法，而非一個閉包：

    $orders = DB::table('orders')
        ->select(DB::raw(1))
        ->whereColumn('orders.user_id', 'users.id');

    $users = DB::table('users')
        ->whereExists($orders)
        ->get();

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
### 子查詢 Where 條件

有時您可能需要建構一個「where」子句，將子查詢的結果與給定值進行比較。您可以透過將閉包和一個值傳遞給 `where` 方法來實現此目的。例如，以下查詢將會檢索所有具有指定類型「membership」的近期使用者；

    use App\Models\User;
    use Illuminate\Database\Query\Builder;

    $users = User::where(function (Builder $query) {
        $query->select('type')
            ->from('membership')
            ->whereColumn('membership.user_id', 'users.id')
            ->orderByDesc('membership.start_date')
            ->limit(1);
    }, 'Pro')->get();

或者，您可能需要建構一個「where」子句，將欄位與子查詢的結果進行比較。您可以透過將欄位、運算子和閉包傳遞給 `where` 方法來實現此目的。例如，以下查詢將會檢索所有金額低於平均值的收入記錄；

    use App\Models\Income;
    use Illuminate\Database\Query\Builder;

    $incomes = Income::where('amount', '<', function (Builder $query) {
        $query->selectRaw('avg(i.amount)')->from('incomes as i');
    })->get();

<a name="full-text-where-clauses"></a>
### 全文檢索 Where 條件

> [!WARNING]  
> MariaDB、MySQL 和 PostgreSQL 目前支援全文檢索的 Where 條件。

`whereFullText` 和 `orWhereFullText` 方法可用於為具有 [全文檢索索引](/docs/{{version}}/migrations#available-index-types) 的欄位查詢新增全文檢索「where」子句。這些方法將由 Laravel 轉換為底層資料庫系統適用的 SQL。例如，對於使用 MariaDB 或 MySQL 的應用程式，將會產生一個 `MATCH AGAINST` 子句：

    $users = DB::table('users')
        ->whereFullText('bio', 'web developer')
        ->get();

<a name="ordering-grouping-limit-and-offset"></a>
## 排序、分組、限制與偏移

<a name="ordering"></a>
### 排序

<a name="orderby"></a>
#### `orderBy` 方法

`orderBy` 方法允許您依據給定欄位來排序查詢結果。`orderBy` 方法接受的第一個引數應該是您希望排序的欄位，而第二個引數則決定排序方向，可以是 `asc` 或 `desc`：

    $users = DB::table('users')
        ->orderBy('name', 'desc')
        ->get();

若要依據多個欄位排序，您可以視需要多次呼叫 `orderBy`：

    $users = DB::table('users')
        ->orderBy('name', 'desc')
        ->orderBy('email', 'asc')
        ->get();

<a name="latest-oldest"></a>
#### `latest` 與 `oldest` 方法

`latest` 與 `oldest` 方法讓您可以輕鬆地依日期排序結果。預設情況下，結果將會依據資料表的 `created_at` 欄位排序。或者，您可以傳遞您希望排序的欄位名稱：

    $user = DB::table('users')
        ->latest()
        ->first();

<a name="random-ordering"></a>
#### 隨機排序

`inRandomOrder` 方法可用於隨機排序查詢結果。例如，您可以使用此方法來取得一個隨機使用者：

    $randomUser = DB::table('users')
        ->inRandomOrder()
        ->first();

<a name="removing-existing-orderings"></a>
#### 移除現有排序

`reorder` 方法會移除所有先前套用至查詢的「order by」子句：

    $query = DB::table('users')->orderBy('name');

    $unorderedUsers = $query->reorder()->get();

在呼叫 `reorder` 方法時，您可以傳遞一個欄位和方向，以移除所有現有的「order by」子句並對查詢套用全新的排序：

    $query = DB::table('users')->orderBy('name');

    $usersOrderedByEmail = $query->reorder('email', 'desc')->get();

<a name="grouping"></a>
### 分組

<a name="groupby-having"></a>
#### `groupBy` 與 `having` 方法

如您所預期，`groupBy` 與 `having` 方法可用於分組查詢結果。`having` 方法的簽章與 `where` 方法相似：

    $users = DB::table('users')
        ->groupBy('account_id')
        ->having('account_id', '>', 100)
        ->get();

您可以使用 `havingBetween` 方法來篩選指定範圍內的結果：

    $report = DB::table('orders')
        ->selectRaw('count(id) as number_of_orders, customer_id')
        ->groupBy('customer_id')
        ->havingBetween('number_of_orders', [5, 15])
        ->get();

您可以傳遞多個引數給 `groupBy` 方法，以依據多個欄位進行分組：

    $users = DB::table('users')
        ->groupBy('first_name', 'status')
        ->having('account_id', '>', 100)
        ->get();

若要建構更進階的 `having` 語句，請參閱 [`havingRaw`](#raw-methods) 方法。

<a name="limit-and-offset"></a>
### 限制與偏移

<a name="skip-take"></a>
#### `skip` 與 `take` 方法

您可以使用 `skip` 與 `take` 方法來限制查詢返回的結果數量，或在查詢中跳過指定數量的結果：

    $users = DB::table('users')->skip(10)->take(5)->get();

或者，您可以使用 `limit` 與 `offset` 方法。這些方法在功能上分別與 `take` 和 `skip` 方法等效：

    $users = DB::table('users')
        ->offset(10)
        ->limit(5)
        ->get();

<a name="conditional-clauses"></a>
## 條件式子句

有時您可能希望根據另一個條件將某些查詢子句套用至查詢。例如，您可能只希望在傳入的 HTTP 請求中存在給定輸入值時才套用 `where` 語句。您可以使用 `when` 方法來實現此目的：

    $role = $request->input('role');

    $users = DB::table('users')
        ->when($role, function (Builder $query, string $role) {
            $query->where('role_id', $role);
        })
        ->get();

`when` 方法僅在第一個引數為 `true` 時執行給定的閉包。如果第一個引數為 `false`，則閉包將不會執行。因此，在上面的範例中，傳遞給 `when` 方法的閉包只有在傳入請求中存在 `role` 欄位且其值為 `true` 時才會被呼叫。

您可以傳遞另一個閉包作為 `when` 方法的第三個引數。此閉包只有在第一個引數為 `false` 時才會執行。為了說明此功能如何使用，我們將使用它來設定查詢的預設排序：

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

查詢產生器也提供一個 `insert` 方法，可用來將記錄插入資料庫表格中。`insert` 方法接受一個包含欄位名稱與值的陣列：

    DB::table('users')->insert([
        'email' => 'kayla@example.com',
        'votes' => 0
    ]);

您可以傳遞一個陣列的陣列，一次插入多筆記錄。每個陣列代表一筆應插入表格的記錄：

    DB::table('users')->insert([
        ['email' => 'picard@example.com', 'votes' => 0],
        ['email' => 'janeway@example.com', 'votes' => 0],
    ]);

`insertOrIgnore` 方法會在將記錄插入資料庫時忽略錯誤。使用此方法時，您應該注意重複記錄的錯誤將會被忽略，而其他類型的錯誤也可能根據資料庫引擎的不同而被忽略。例如，`insertOrIgnore` 將會[繞過 MySQL 的嚴格模式](https://dev.mysql.com/doc/refman/en/sql-mode.html#ignore-effect-on-execution)：

    DB::table('users')->insertOrIgnore([
        ['id' => 1, 'email' => 'sisko@example.com'],
        ['id' => 2, 'email' => 'archer@example.com'],
    ]);

`insertUsing` 方法會在將新記錄插入表格時，使用子查詢來決定應插入的資料：

    DB::table('pruned_users')->insertUsing([
        'id', 'name', 'email', 'email_verified_at'
    ], DB::table('users')->select(
        'id', 'name', 'email', 'email_verified_at'
    )->where('updated_at', '<=', now()->subMonth()));


<a name="auto-incrementing-ids"></a>
#### 自動遞增 ID

如果表格有自動遞增的 `id`，可以使用 `insertGetId` 方法插入一筆記錄並取得該 ID：

    $id = DB::table('users')->insertGetId(
        ['email' => 'john@example.com', 'votes' => 0]
    );

> [!WARNING]  
> 當使用 PostgreSQL 時，`insertGetId` 方法預期自動遞增的欄位名稱為 `id`。如果您想從不同的「sequence」中取得 ID，可以將欄位名稱作為 `insertGetId` 方法的第二個參數傳入。


<a name="upserts"></a>
### 新增或更新 (Upserts)

`upsert` 方法將會插入不存在的記錄，並使用您可能指定的新的值來更新已存在的記錄。此方法的第一個參數包含要插入或更新的值，第二個參數則列出在相關表格中唯一識別記錄的欄位。此方法的第三個也是最後一個參數是一個欄位陣列，如果資料庫中已存在匹配的記錄，則這些欄位將會被更新：

    DB::table('flights')->upsert(
        [
            ['departure' => 'Oakland', 'destination' => 'San Diego', 'price' => 99],
            ['departure' => 'Chicago', 'destination' => 'New York', 'price' => 150]
        ],
        ['departure', 'destination'],
        ['price']
    );

在上面的範例中，Laravel 將會嘗試插入兩筆記錄。如果已存在具有相同 `departure` 與 `destination` 欄位值的記錄，Laravel 將會更新該記錄的 `price` 欄位。

> [!WARNING]  
> 除了 SQL Server 之外的所有資料庫，都要求 `upsert` 方法第二個參數中的欄位必須具有「primary」或「unique」索引。此外，MariaDB 與 MySQL 資料庫驅動程式會忽略 `upsert` 方法的第二個參數，並始終使用表格的「primary」與「unique」索引來偵測現有記錄。


<a name="update-statements"></a>
## Update 語句

除了將記錄插入資料庫之外，查詢產生器還可以使用 `update` 方法更新現有記錄。`update` 方法與 `insert` 方法一樣，接受一個包含欄位與值配對的陣列，指示要更新的欄位。`update` 方法會回傳受影響的列數。您可以使用 `where` 條件來約束 `update` 查詢：

    $affected = DB::table('users')
        ->where('id', 1)
        ->update(['votes' => 1]);


<a name="update-or-insert"></a>
#### 更新或插入

有時您可能想更新資料庫中的現有記錄，如果沒有匹配的記錄則建立它。在這種情況下，可以使用 `updateOrInsert` 方法。`updateOrInsert` 方法接受兩個參數：一個用於尋找記錄的條件陣列，以及一個包含要更新欄位與值配對的陣列。

`updateOrInsert` 方法將會嘗試使用第一個參數的欄位與值配對來定位匹配的資料庫記錄。如果記錄存在，它將會以第二個參數中的值進行更新。如果找不到記錄，將會插入一個新記錄，其中包含兩個參數合併後的屬性：

    DB::table('users')
        ->updateOrInsert(
            ['email' => 'john@example.com', 'name' => 'John'],
            ['votes' => '2']
        );

您可以為 `updateOrInsert` 方法提供一個閉包，以根據匹配記錄的存在與否，來自訂更新或插入資料庫中的屬性：

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

當更新 JSON 欄位時，您應該使用 `->` 語法來更新 JSON 物件中相應的鍵。此操作支援 MariaDB 10.3+、MySQL 5.7+ 和 PostgreSQL 9.5+：

    $affected = DB::table('users')
        ->where('id', 1)
        ->update(['options->enabled' => true]);


<a name="increment-and-decrement"></a>
### 遞增與遞減

查詢產生器也提供方便的方法來遞增或遞減指定欄位的值。這兩種方法都至少接受一個參數：要修改的欄位。可以提供第二個參數來指定欄位應 incremented 或 decremented 的量：

    DB::table('users')->increment('votes');

    DB::table('users')->increment('votes', 5);

    DB::table('users')->decrement('votes');

    DB::table('users')->decrement('votes', 5);

如果需要，您也可以在遞增或遞減操作期間指定要更新的其他欄位：

    DB::table('users')->increment('votes', 1, ['name' => 'John']);

此外，您還可以使用 `incrementEach` 和 `decrementEach` 方法一次遞增或遞減多個欄位：

    DB::table('users')->incrementEach([
        'votes' => 5,
        'balance' => 100,
    ]);


<a name="delete-statements"></a>
## Delete 語句

查詢產生器的 `delete` 方法可用來從表格中刪除記錄。`delete` 方法會回傳受影響的列數。您可以透過在呼叫 `delete` 方法之前加入 "where" 條件來約束 `delete` 語句：

    $deleted = DB::table('users')->delete();

    $deleted = DB::table('users')->where('votes', '>', 100)->delete();

<a name="pessimistic-locking"></a>
## 悲觀鎖定

查詢產生器也包含了一些功能，可協助您在執行 `select` 語句時實現「悲觀鎖定」。若要使用「共用鎖定」來執行語句，您可以呼叫 `sharedLock` 方法。「共用鎖定」會防止選定的資料列在您的交易提交之前被修改：

    DB::table('users')
        ->where('votes', '>', 100)
        ->sharedLock()
        ->get();

或者，您可以使用 `lockForUpdate` 方法。「更新鎖定」會防止選定的記錄被修改，或被另一個共用鎖定選取：

    DB::table('users')
        ->where('votes', '>', 100)
        ->lockForUpdate()
        ->get();

雖然不是強制性的，但建議將悲觀鎖定包裝在 [交易](/docs/{{version}}/database#database-transactions) 中。這確保了在整個操作完成之前，擷取到的資料在資料庫中保持不變。如果發生失敗，交易將自動回滾所有變更並釋放鎖定：

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

您可以在建構查詢時使用 `dd` 和 `dump` 方法來傾印目前的查詢綁定和 SQL。`dd` 方法將顯示除錯資訊，然後停止執行請求。`dump` 方法將顯示除錯資訊，但允許請求繼續執行：

    DB::table('users')->where('votes', '>', 100)->dd();

    DB::table('users')->where('votes', '>', 100)->dump();

`dumpRawSql` 和 `ddRawSql` 方法可以在查詢上呼叫，以傾印所有參數綁定已正確替換的查詢 SQL：

    DB::table('users')->where('votes', '>', 100)->dumpRawSql();

    DB::table('users')->where('votes', '>', 100)->ddRawSql();