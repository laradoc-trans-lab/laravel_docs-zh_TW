# 資料庫：分頁

- [介紹](#introduction)
- [基本用法](#basic-usage)
    - [查詢建構器分頁結果](#paginating-query-builder-results)
    - [Eloquent 分頁結果](#paginating-eloquent-results)
    - [游標分頁](#cursor-pagination)
    - [手動建立 Paginator](#manually-creating-a-paginator)
    - [自訂分頁 URL](#customizing-pagination-urls)
- [顯示分頁結果](#displaying-pagination-results)
    - [調整分頁連結視窗](#adjusting-the-pagination-link-window)
    - [將結果轉換為 JSON](#converting-results-to-json)
- [自訂分頁視圖](#customizing-the-pagination-view)
    - [使用 Bootstrap](#using-bootstrap)
- [Paginator 與 LengthAwarePaginator 實例方法](#paginator-instance-methods)
- [Cursor Paginator 實例方法](#cursor-paginator-instance-methods)

<a name="introduction"></a>
## 介紹

在其他框架中，分頁可能會非常麻煩。我們希望 Laravel 的分頁方法能讓您耳目一新。Laravel 的分頁器整合了 [查詢建構器](/docs/{{version}}/queries) 和 [Eloquent ORM](/docs/{{version}}/eloquent)，提供方便易用、零設定的資料庫記錄分頁功能。

預設情況下，分頁器產生的 HTML 與 [Tailwind CSS 框架](https://tailwindcss.com/) 相容；不過，也提供了 Bootstrap 分頁支援。

<a name="tailwind-jit"></a>
#### Tailwind JIT

如果您正在使用 Laravel 預設的 Tailwind 分頁視圖和 Tailwind JIT 引擎，您應確保應用程式的 `tailwind.config.js` 檔案中的 `content` 鍵引用了 Laravel 的分頁視圖，以避免其 Tailwind 類別被清除：

```js
content: [
    './resources/**/*.blade.php',
    './resources/**/*.js',
    './resources/**/*.vue',
    './vendor/laravel/framework/src/Illuminate/Pagination/resources/views/*.blade.php',
],
```

<a name="basic-usage"></a>
## 基本用法

<a name="paginating-query-builder-results"></a>
### 查詢建構器分頁結果

有多種方法可以對項目進行分頁。最簡單的方法是使用 [查詢建構器](/docs/{{version}}/queries) 或 [Eloquent 查詢](/docs/{{version}}/eloquent) 上的 `paginate` 方法。`paginate` 方法會根據使用者目前正在檢視的頁面，自動設定查詢的「limit」和「offset」。預設情況下，目前頁面是透過 HTTP 請求中 `page` 查詢字串參數的值來偵測。這個值由 Laravel 自動偵測，也會自動插入到 Paginator 產生的連結中。

在此範例中，傳遞給 `paginate` 方法的唯一引數是您希望「每頁」顯示的項目數量。在本例中，讓我們指定每頁顯示 `15` 個項目：

    <?php

    namespace App\Http\Controllers;

    use App\Http\Controllers\Controller;
    use Illuminate\Support\Facades\DB;
    use Illuminate\View\View;

    class UserController extends Controller
    {
        /**
         * Show all application users.
         */
        public function index(): View
        {
            return view('user.index', [
                'users' => DB::table('users')->paginate(15)
            ]);
        }
    }

<a name="simple-pagination"></a>
#### 簡單分頁

`paginate` 方法在從資料庫中檢索記錄之前，會計算查詢所匹配的記錄總數。這樣做的目的是為了讓 Paginator 知道總共有多少頁記錄。然而，如果您不打算在應用程式的 UI 中顯示總頁數，則記錄計數查詢是不必要的。

因此，如果您只需要在應用程式的 UI 中顯示簡單的「下一頁」和「上一頁」連結，您可以使用 `simplePaginate` 方法執行單一且高效的查詢：

    $users = DB::table('users')->simplePaginate(15);

<a name="paginating-eloquent-results"></a>
### Eloquent 分頁結果

您也可以對 [Eloquent](/docs/{{version}}/eloquent) 查詢進行分頁。在此範例中，我們將對 `App\Models\User` 模型進行分頁，並指出我們計畫每頁顯示 15 條記錄。如您所見，其語法與對查詢建構器結果進行分頁幾乎相同：

    use App\Models\User;

    $users = User::paginate(15);

當然，您可以在對查詢設定其他約束（例如 `where` 子句）後呼叫 `paginate` 方法：

    $users = User::where('votes', '>', 100)->paginate(15);

您也可以在使用 Eloquent 模型分頁時使用 `simplePaginate` 方法：

    $users = User::where('votes', '>', 100)->simplePaginate(15);

同樣地，您可以使用 `cursorPaginate` 方法對 Eloquent 模型進行游標分頁：

    $users = User::where('votes', '>', 100)->cursorPaginate(15);

<a name="multiple-paginator-instances-per-page"></a>
#### 每頁多個 Paginator 實例

有時候您可能需要在應用程式的單一畫面中渲染兩個獨立的 Paginator。然而，如果兩個 Paginator 實例都使用 `page` 查詢字串參數來儲存目前頁面，這兩個 Paginator 將會衝突。為了解決這個衝突，您可以透過提供給 `paginate`、`simplePaginate` 和 `cursorPaginate` 方法的第三個引數，傳遞您希望用來儲存 Paginator 目前頁面的查詢字串參數名稱：

    use App\Models\User;

    $users = User::where('votes', '>', 100)->paginate(
        $perPage = 15, $columns = ['*'], $pageName = 'users'
    );

<a name="cursor-pagination"></a>
### 游標分頁

雖然 `paginate` 和 `simplePaginate` 使用 SQL 的「offset」子句建立查詢，但游標分頁是透過建構「where」子句來比較查詢中排序欄位的值，提供 Laravel 所有分頁方法中最有效率的資料庫效能。這種分頁方法特別適用於大型資料集和「無限」捲動的使用者介面。

與基於位移的分頁不同，基於位移的分頁在 Paginator 產生的 URL 查詢字串中包含頁碼，而基於游標的分頁則在查詢字串中放置一個「cursor」字串。該 cursor 是一個編碼字串，包含下一個分頁查詢應該開始分頁的位置以及分頁的方向：

```nothing
http://localhost/users?cursor=eyJpZCI6MTUsIl9wb2ludHNUb05leHRJdGVtcyI6dHJ1ZX0
```

您可以透過查詢建構器提供的 `cursorPaginate` 方法建立一個基於游標的 Paginator 實例。此方法會回傳 `Illuminate\Pagination\CursorPaginator` 的實例：

    $users = DB::table('users')->orderBy('id')->cursorPaginate(15);

一旦您檢索到 Cursor Paginator 實例，您就可以像使用 `paginate` 和 `simplePaginate` 方法時一樣 [顯示分頁結果](#displaying-pagination-results)。有關 Cursor Paginator 提供的實例方法的更多資訊，請查閱 [Cursor Paginator 實例方法文件](#cursor-paginator-instance-methods)。

> [!WARNING]
> 您的查詢必須包含「order by」子句才能利用游標分頁。此外，查詢排序的欄位必須屬於您正在分頁的資料表。

<a name="cursor-vs-offset-pagination"></a>
#### 游標分頁與位移分頁的比較

為了說明位移分頁和游標分頁之間的差異，讓我們檢視一些 SQL 查詢範例。以下兩個查詢都將顯示 `users` 資料表按 `id` 排序的「第二頁」結果：

```sql
# Offset Pagination...
select * from users order by id asc limit 15 offset 15;

# Cursor Pagination...
select * from users where id > 15 order by id asc limit 15;
```

游標分頁查詢比位移分頁具有以下優點：

- 對於大型資料集，如果「order by」欄位已建立索引，游標分頁將提供更好的效能。這是因為「offset」子句會掃描所有先前匹配的資料。
- 對於頻繁寫入的資料集，如果結果最近已新增或刪除，位移分頁可能會跳過記錄或顯示重複項。

然而，游標分頁有以下限制：

- 類似於 `simplePaginate`，游標分頁只能用於顯示「下一頁」和「上一頁」連結，並且不支援產生帶有頁碼的連結。
- 它要求排序基於至少一個唯一欄位或唯一欄位的組合。不支援具有 `null` 值的欄位。
- 只有當「order by」子句中的查詢表達式被別名化並也新增到「select」子句時，才支援這些表達式。
- 不支援帶有參數的查詢表達式。

<a name="manually-creating-a-paginator"></a>
### 手動建立 Paginator

有時您可能希望手動建立一個分頁實例，將您記憶體中已有的項目陣列傳遞給它。您可以根據您的需求，透過建立 `Illuminate\Pagination\Paginator`、`Illuminate\Pagination\LengthAwarePaginator` 或 `Illuminate\Pagination\CursorPaginator` 實例來實現。

`Paginator` 和 `CursorPaginator` 類別不需要知道結果集中的項目總數；然而，由於這個原因，這些類別沒有用於檢索最後一頁索引的方法。`LengthAwarePaginator` 接受幾乎與 `Paginator` 相同的引數；但是，它需要結果集中項目總數的計數。

換句話說，`Paginator` 對應於查詢建構器上的 `simplePaginate` 方法，`CursorPaginator` 對應於 `cursorPaginate` 方法，而 `LengthAwarePaginator` 對應於 `paginate` 方法。

> [!WARNING]
> 手動建立 Paginator 實例時，您應該手動「切割 (slice)」傳遞給 Paginator 的結果陣列。如果您不確定如何操作，請查閱 [array_slice](https://secure.php.net/manual/en/function.array-slice.php) PHP 函數。

<a name="customizing-pagination-urls"></a>
### 自訂分頁 URL

預設情況下，Paginator 所生成的連結將與目前請求的 URI 相符。然而，Paginator 的 `withPath` 方法允許你自訂 Paginator 在生成連結時使用的 URI。例如，如果你希望 Paginator 生成像 `http://example.com/admin/users?page=N` 這樣的連結，你應該將 `/admin/users` 傳遞至 `withPath` 方法：

    use App\Models\User;

    Route::get('/users', function () {
        $users = User::paginate(15);

        $users->withPath('/admin/users');

        // ...
    });


<a name="appending-query-string-values"></a>
#### 附加查詢字串值

你可以使用 `appends` 方法將值附加到分頁連結的查詢字串中。例如，若要將 `sort=votes` 附加到每個分頁連結，你應該像這樣呼叫 `appends`：

    use App\Models\User;

    Route::get('/users', function () {
        $users = User::paginate(15);

        $users->appends(['sort' => 'votes']);

        // ...
    });

如果你想將目前請求的所有查詢字串值附加到分頁連結，可以使用 `withQueryString` 方法：

    $users = User::paginate(15)->withQueryString();


<a name="appending-hash-fragments"></a>
#### 附加 Hash 片段

如果你需要將「hash 片段」附加到 Paginator 生成的 URL，你可以使用 `fragment` 方法。例如，若要將 `#users` 附加到每個分頁連結的末尾，你應該像這樣呼叫 `fragment` 方法：

    $users = User::paginate(15)->fragment('users');

<a name="displaying-pagination-results"></a>
## 顯示分頁結果

當呼叫 `paginate` 方法時，您會收到一個 `Illuminate\Pagination\LengthAwarePaginator` 實例；而呼叫 `simplePaginate` 方法則會回傳一個 `Illuminate\Pagination\Paginator` 實例。最後，呼叫 `cursorPaginate` 方法會回傳一個 `Illuminate\Pagination\CursorPaginator` 實例。

這些物件提供了多種方法來描述結果集。除了這些輔助方法之外，分頁器實例也是迭代器，可以像陣列一樣進行迴圈。因此，一旦您取得結果，就可以使用 [Blade](/docs/{{version}}/blade) 顯示結果並渲染頁面連結：

```blade
<div class="container">
    @foreach ($users as $user)
        {{ $user->name }}
    @endforeach
</div>

{{ $users->links() }}
```

`links` 方法將渲染結果集中其餘頁面的連結。這些連結中的每一個都已經包含正確的 `page` 查詢字串變數。請記住，`links` 方法產生的 HTML 與 [Tailwind CSS framework](https://tailwindcss.com) 相容。

<a name="adjusting-the-pagination-link-window"></a>
### 調整分頁連結視窗

當分頁器顯示分頁連結時，會顯示當前頁碼以及當前頁前後三頁的連結。使用 `onEachSide` 方法，您可以控制在分頁器生成的中間滑動視窗中，當前頁面兩側顯示的額外連結數量：

```blade
{{ $users->onEachSide(5)->links() }}
```

<a name="converting-results-to-json"></a>
### 將結果轉換為 JSON

Laravel 分頁器類別實作了 `Illuminate\Contracts\Support\Jsonable` 介面契約並公開了 `toJson` 方法，因此將分頁結果轉換為 JSON 非常容易。您也可以透過從路由或控制器動作回傳分頁器實例來將其轉換為 JSON：

    use App\Models\User;

    Route::get('/users', function () {
        return User::paginate();
    });

分頁器中的 JSON 將包含元資訊，例如 `total`、`current_page`、`last_page` 等。結果記錄可透過 JSON 陣列中的 `data` 鍵取得。以下是從路由回傳分頁器實例所建立的 JSON 範例：

    {
       "total": 50,
       "per_page": 15,
       "current_page": 1,
       "last_page": 4,
       "first_page_url": "http://laravel.app?page=1",
       "last_page_url": "http://laravel.app?page=4",
       "next_page_url": "http://laravel.app?page=2",
       "prev_page_url": null,
       "path": "http://laravel.app",
       "from": 1,
       "to": 15,
       "data":[
            {
                // Record...
            },
            {
                // Record...
            }
       ]
    }

<a name="customizing-the-pagination-view"></a>
## 自訂分頁視圖

預設情況下，用於顯示分頁連結的視圖與 [Tailwind CSS](https://tailwindcss.com) 框架相容。但是，如果您不使用 Tailwind，則可以自由定義自己的視圖來渲染這些連結。當在分頁器實例上呼叫 `links` 方法時，您可以將視圖名稱作為該方法的第一個參數傳入：

```blade
{{ $paginator->links('view.name') }}

<!-- Passing additional data to the view... -->
{{ $paginator->links('view.name', ['foo' => 'bar']) }}
```

然而，自訂分頁視圖最簡單的方法是使用 `vendor:publish` 指令將它們匯出到您的 `resources/views/vendor` 目錄：

```shell
php artisan vendor:publish --tag=laravel-pagination
```

此指令會將視圖放置在應用程式的 `resources/views/vendor/pagination` 目錄中。此目錄中的 `tailwind.blade.php` 檔案對應於預設分頁視圖。您可以編輯此檔案以修改分頁 HTML。

如果您想指定另一個檔案作為預設分頁視圖，可以在 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫分頁器的 `defaultView` 和 `defaultSimpleView` 方法：

    <?php

    namespace App\Providers;

    use Illuminate\Pagination\Paginator;
    use Illuminate\Support\ServiceProvider;

    class AppServiceProvider extends ServiceProvider
    {
        /**
         * Bootstrap any application services.
         */
        public function boot(): void
        {
            Paginator::defaultView('view-name');

            Paginator::defaultSimpleView('view-name');
        }
    }

<a name="using-bootstrap"></a>
### 使用 Bootstrap

Laravel 包含了使用 [Bootstrap CSS](https://getbootstrap.com/) 建立的分頁視圖。要使用這些視圖而不是預設的 Tailwind 視圖，您可以在 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫分頁器的 `useBootstrapFour` 或 `useBootstrapFive` 方法：

    use Illuminate\Pagination\Paginator;

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        Paginator::useBootstrapFive();
        Paginator::useBootstrapFour();
    }

<a name="paginator-instance-methods"></a>
## Paginator 與 LengthAwarePaginator 實例方法

每個 Paginator 實例都透過以下方法提供額外的分頁資訊：

<div class="overflow-auto">

| 方法 | 描述 |
| --- | --- |
| `$paginator->count()` | 取得當前頁面的項目數量。 |
| `$paginator->currentPage()` | 取得當前頁碼。 |
| `$paginator->firstItem()` | 取得結果集中第一個項目的編號。 |
| `$paginator->getOptions()` | 取得分頁器選項。 |
| `$paginator->getUrlRange($start, $end)` | 建立一系列分頁 URL。 |
| `$paginator->hasPages()` | 判斷是否有足夠項目可分為多頁。 |
| `$paginator->hasMorePages()` | 判斷資料儲存中是否還有更多項目。 |
| `$paginator->items()` | 取得當前頁面的項目。 |
| `$paginator->lastItem()` | 取得結果集中最後一個項目的編號。 |
| `$paginator->lastPage()` | 取得最後一頁的頁碼。(使用 `simplePaginate` 時不可用)。 |
| `$paginator->nextPageUrl()` | 取得下一頁的 URL。 |
| `$paginator->onFirstPage()` | 判斷分頁器是否在第一頁。 |
| `$paginator->perPage()` | 每頁顯示的項目數量。 |
| `$paginator->previousPageUrl()` | 取得上一頁的 URL。 |
| `$paginator->total()` | 判斷資料儲存中符合條件的總項目數量。(使用 `simplePaginate` 時不可用)。 |
| `$paginator->url($page)` | 取得指定頁碼的 URL。 |
| `$paginator->getPageName()` | 取得用於儲存頁面的查詢字串變數。 |
| `$paginator->setPageName($name)` | 設定用於儲存頁面的查詢字串變數。 |
| `$paginator->through($callback)` | 使用回呼函式轉換每個項目。 |

</div>

<a name="cursor-paginator-instance-methods"></a>
## Cursor Paginator 實例方法

每個 cursor Paginator 實例都透過以下方法提供額外的分頁資訊：

<div class="overflow-auto">

| 方法                            | 說明                                          |
| :------------------------------ | :-------------------------------------------- |
| `$paginator->count()`           | 取得目前頁面上的項目數量。                  |
| `$paginator->cursor()`          | 取得目前的 cursor 實例。                      |
| `$paginator->getOptions()`      | 取得 Paginator 選項。                         |
| `$paginator->hasPages()`        | 判斷是否有足夠的項目可分割成多個頁面。      |
| `$paginator->hasMorePages()`    | 判斷資料儲存中是否還有更多項目。            |
| `$paginator->getCursorName()`   | 取得用來儲存 cursor 的查詢字串變數。        |
| `$paginator->items()`           | 取得目前頁面上的項目。                      |
| `$paginator->nextCursor()`      | 取得下一組項目的 cursor 實例。                |
| `$paginator->nextPageUrl()`     | 取得下一頁的 URL。                            |
| `$paginator->onFirstPage()`     | 判斷 Paginator 是否在第一頁。                 |
| `$paginator->onLastPage()`      | 判斷 Paginator 是否在最後一頁。               |
| `$paginator->perPage()`         | 每頁顯示的項目數量。                        |
| `$paginator->previousCursor()`  | 取得前一組項目的 cursor 實例。                |
| `$paginator->previousPageUrl()` | 取得上一頁的 URL。                            |
| `$paginator->setCursorName()`   | 設定用來儲存 cursor 的查詢字串變數。        |
| `$paginator->url($cursor)`      | 取得指定 cursor 實例的 URL。                  |

</div>