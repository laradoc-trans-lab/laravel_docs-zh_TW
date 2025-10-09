# 資料庫：分頁

- [簡介](#introduction)
- [基本用法](#basic-usage)
    - [對 Query Builder 結果進行分頁](#paginating-query-builder-results)
    - [對 Eloquent 結果進行分頁](#paginating-eloquent-results)
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
## 簡介

在其他框架中，分頁可能非常痛苦。我們希望 Laravel 的分頁方法能帶來一股清新的氣息。Laravel 的 paginator 與 [查詢建構器](/docs/{{version}}/queries) 和 [Eloquent ORM](/docs/{{version}}/eloquent) 整合，提供方便、易於使用的資料庫記錄分頁，且無需任何設定。

預設情況下，由 paginator 產生的 HTML 與 [Tailwind CSS 框架](https://tailwindcss.com/) 相容；不過，也支援 Bootstrap 分頁。


<a name="tailwind"></a>
#### Tailwind

如果您正在使用 Laravel 預設的 Tailwind 分頁視圖並搭配 Tailwind 4.x，您的應用程式的 `resources/css/app.css` 檔案將會自動配置以 `@source` Laravel 的分頁視圖：

```css
@import 'tailwindcss';

@source '../../vendor/laravel/framework/src/Illuminate/Pagination/resources/views/*.blade.php';
```

<a name="basic-usage"></a>
## 基本用法


<a name="paginating-query-builder-results"></a>
### 對 Query Builder 結果進行分頁

有幾種方式可以對項目進行分頁。最簡單的方式是使用 [Query Builder](/docs/{{version}}/queries) 或 [Eloquent 查詢](/docs/{{version}}/eloquent) 上的 `paginate` 方法。`paginate` 方法會自動處理查詢的「limit」和「offset」設定，基於使用者目前正在瀏覽的頁面。預設情況下，目前頁面是透過 HTTP 請求中的 `page` 查詢字串引數值來偵測的。這個值由 Laravel 自動偵測，並自動插入到由分頁器產生的連結中。

在此範例中，傳遞給 `paginate` 方法的唯一引數是您希望「每頁」顯示的項目數量。在此情況下，我們指定每頁顯示 `15` 個項目：

```php
<?php

namespace App\Http\Controllers;

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
```


<a name="simple-pagination"></a>
#### 簡易分頁

`paginate` 方法會在從資料庫檢索記錄之前，計算查詢匹配到的總記錄數。這樣做的目的是讓分頁器能夠知道總共有多少頁記錄。然而，如果您不打算在應用程式的 UI 中顯示總頁數，那麼記錄計數查詢是不必要的。

因此，如果您只需要在應用程式的 UI 中顯示簡單的「下一頁」和「上一頁」連結，您可以使用 `simplePaginate` 方法來執行單一、高效的查詢：

```php
$users = DB::table('users')->simplePaginate(15);
```


<a name="paginating-eloquent-results"></a>
### 對 Eloquent 結果進行分頁

您也可以對 [Eloquent](/docs/{{version}}/eloquent) 查詢進行分頁。在此範例中，我們將對 `App\Models\User` 模型進行分頁，並表示我們計劃每頁顯示 15 筆記錄。正如您所見，語法與對 Query Builder 結果進行分頁幾乎相同：

```php
use App\Models\User;

$users = User::paginate(15);
```

當然，您可以在對查詢設定其他約束條件後呼叫 `paginate` 方法，例如 `where` 子句：

```php
$users = User::where('votes', '>', 100)->paginate(15);
```

您也可以在使用 Eloquent 模型分頁時使用 `simplePaginate` 方法：

```php
$users = User::where('votes', '>', 100)->simplePaginate(15);
```

同樣地，您可以使用 `cursorPaginate` 方法對 Eloquent 模型進行游標分頁：

```php
$users = User::where('votes', '>', 100)->cursorPaginate(15);
```


<a name="multiple-paginator-instances-per-page"></a>
#### 每頁多個 Paginator 實例

有時您可能需要在應用程式渲染的單一螢幕上渲染兩個獨立的分頁器。然而，如果兩個分頁器實例都使用 `page` 查詢字串參數來儲存目前頁面，這兩個分頁器將會衝突。為了解決此衝突，您可以透過傳遞給 `paginate`、`simplePaginate` 和 `cursorPaginate` 方法的第三個引數，來指定您希望用來儲存分頁器目前頁面的查詢字串參數名稱：

```php
use App\Models\User;

$users = User::where('votes', '>', 100)->paginate(
    $perPage = 15, $columns = ['*'], $pageName = 'users'
);
```


<a name="cursor-pagination"></a>
### 游標分頁

儘管 `paginate` 和 `simplePaginate` 使用 SQL 的「offset」子句來建立查詢，游標分頁則是透過建構「where」子句來運作，該子句會比較查詢中排序欄位的值，從而提供 Laravel 所有分頁方法中最有效率的資料庫效能。這種分頁方法特別適用於大型資料集和「無限捲動」的使用者介面。

與基於位移 (offset) 的分頁不同，基於位移的分頁會在分頁器產生的 URL 查詢字串中包含頁碼，基於游標的分頁則是在查詢字串中放置一個「cursor」字串。此游標是一個編碼過的字串，其中包含下一個分頁查詢應該開始分頁的位置以及分頁方向：

```text
http://localhost/users?cursor=eyJpZCI6MTUsIl9wb2ludHNUb05leHRJdGVtcyI6dHJ1ZX0
```

您可以透過 Query Builder 提供的 `cursorPaginate` 方法來建立一個基於游標的分頁器實例。此方法會回傳一個 `Illuminate\Pagination\CursorPaginator` 實例：

```php
$users = DB::table('users')->orderBy('id')->cursorPaginate(15);
```

一旦您獲取了游標分頁器實例，您可以像使用 `paginate` 和 `simplePaginate` 方法時一樣，[顯示分頁結果](#displaying-pagination-results)。有關游標分頁器提供之實例方法的更多資訊，請參閱 [游標分頁器實例方法文件](#cursor-paginator-instance-methods)。

> [!WARNING]
> 您的查詢必須包含「order by」子句才能利用游標分頁的優勢。此外，查詢中排序的欄位必須屬於您正在分頁的資料表。


<a name="cursor-vs-offset-pagination"></a>
#### 游標分頁與位移分頁的比較

為了說明位移分頁與游標分頁之間的差異，讓我們檢視一些 SQL 查詢範例。以下兩個查詢都將顯示依 `id` 排序的 `users` 資料表的「第二頁」結果：

```sql
# Offset Pagination...
select * from users order by id asc limit 15 offset 15;

# Cursor Pagination...
select * from users where id > 15 order by id asc limit 15;
```

游標分頁查詢相較於位移分頁具有以下優勢：

- 對於大型資料集，如果「order by」欄位已建立索引，游標分頁將提供更好的效能。這是因為「offset」子句會掃描所有先前匹配到的資料。
- 對於頻繁寫入的資料集，如果使用者目前檢視頁面中的結果近期有新增或刪除，位移分頁可能會跳過記錄或顯示重複項。

然而，游標分頁有以下限制：

- 與 `simplePaginate` 類似，游標分頁只能用於顯示「下一頁」和「上一頁」連結，不支援產生帶有頁碼的連結。
- 它要求排序基於至少一個唯一欄位或唯一欄位的組合。不支援帶有 `null` 值的欄位。
- 「order by」子句中的查詢表達式僅在其被別名化並同時添加到「select」子句時才受支援。
- 不支援帶有參數的查詢表達式。


<a name="manually-creating-a-paginator"></a>
### 手動建立 Paginator

有時您可能希望手動建立一個分頁實例，並傳遞一個您已在記憶體中擁有的項目陣列給它。您可以透過建立 `Illuminate\Pagination\Paginator`、`Illuminate\Pagination\LengthAwarePaginator` 或 `Illuminate\Pagination\CursorPaginator` 實例來實現，根據您的需求而定。

`Paginator` 和 `CursorPaginator` 類別不需要知道結果集中的總項目數；然而，正因為如此，這些類別沒有用於檢索最後一頁索引的方法。`LengthAwarePaginator` 接受與 `Paginator` 幾乎相同的引數；然而，它需要結果集中總項目數的計數。

換句話說，`Paginator` 對應於 Query Builder 上的 `simplePaginate` 方法，`CursorPaginator` 對應於 `cursorPaginate` 方法，而 `LengthAwarePaginator` 對應於 `paginate` 方法。

> [!WARNING]
> 手動建立分頁器實例時，您應該手動「切割」傳遞給分頁器的結果陣列。如果您不確定如何操作，請查閱 [array_slice](https://secure.php.net/manual/en/function.array-slice.php) PHP 函數。

<a name="customizing-pagination-urls"></a>
### 自訂分頁 URL

預設情況下，Paginator 所生成的連結將與目前請求的 URI 相符。然而，Paginator 的 `withPath` 方法允許您自訂 Paginator 在生成連結時所使用的 URI。例如，如果您希望 Paginator 生成類似 `http://example.com/admin/users?page=N` 的連結，您應該將 `/admin/users` 傳遞給 `withPath` 方法：

```php
use App\Models\User;

Route::get('/users', function () {
    $users = User::paginate(15);

    $users->withPath('/admin/users');

    // ...
});
```


<a name="appending-query-string-values"></a>
#### 附加查詢字串值

您可以使用 `appends` 方法將值附加到分頁連結的查詢字串中。例如，若要將 `sort=votes` 附加到每個分頁連結，您應該呼叫 `appends` 方法，如下所示：

```php
use App\Models\User;

Route::get('/users', function () {
    $users = User::paginate(15);

    $users->appends(['sort' => 'votes']);

    // ...
});
```

如果您希望將目前請求的所有查詢字串值附加到分頁連結中，可以使用 `withQueryString` 方法：

```php
$users = User::paginate(15)->withQueryString();
```


<a name="appending-hash-fragments"></a>
#### 附加雜湊片段

如果您需要將「雜湊片段」附加到 Paginator 生成的 URL 中，可以使用 `fragment` 方法。例如，若要將 `#users` 附加到每個分頁連結的末尾，您應該像這樣呼叫 `fragment` 方法：

```php
$users = User::paginate(15)->fragment('users');
```

<a name="displaying-pagination-results"></a>
## 顯示分頁結果

呼叫 `paginate` 方法時，您將收到一個 `Illuminate\Pagination\LengthAwarePaginator` 實例，而呼叫 `simplePaginate` 方法則會回傳一個 `Illuminate\Pagination\Paginator` 實例。最後，呼叫 `cursorPaginate` 方法會回傳一個 `Illuminate\Pagination\CursorPaginator` 實例。

這些物件提供了多種描述結果集的方法。除了這些輔助方法之外，Paginator 實例也是迭代器，可以像陣列一樣被遍歷。因此，一旦您取得了結果，您就可以使用 [Blade](/docs/{{version}}/blade) 來顯示結果並渲染頁面連結：

```blade
<div class="container">
    @foreach ($users as $user)
        {{ $user->name }}
    @endforeach
</div>

{{ $users->links() }}
```

`links` 方法將渲染結果集中其餘頁面的連結。這些連結都將已經包含正確的 `page` 查詢字串變數。請記住，由 `links` 方法產生的 HTML 與 [Tailwind CSS 框架](https://tailwindcss.com) 相容。

<a name="adjusting-the-pagination-link-window"></a>
### 調整分頁連結視窗

當 Paginator 顯示分頁連結時，會顯示當前頁碼以及當前頁面之前和之後的三個頁面連結。使用 `onEachSide` 方法，您可以控制 Paginator 產生的中間滑動連結視窗中，當前頁面兩側額外顯示多少連結：

```blade
{{ $users->onEachSide(5)->links() }}
```

<a name="converting-results-to-json"></a>
### 將結果轉換為 JSON

Laravel Paginator 類別實作了 `Illuminate\Contracts\Support\Jsonable` 介面契約並暴露了 `toJson` 方法，因此將分頁結果轉換為 JSON 非常容易。您也可以透過從路由或控制器動作回傳 Paginator 實例來將其轉換為 JSON：

```php
use App\Models\User;

Route::get('/users', function () {
    return User::paginate();
});
```

Paginator 的 JSON 將包含 `total`、`current_page`、`last_page` 等中繼資訊。結果記錄則可透過 JSON 陣列中的 `data` 鍵取得。以下是從路由回傳 Paginator 實例所產生的 JSON 範例：

```json
{
   "total": 50,
   "per_page": 15,
   "current_page": 1,
   "last_page": 4,
   "current_page_url": "http://laravel.app?page=1",
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
```

<a name="customizing-the-pagination-view"></a>
## 自訂分頁視圖

預設情況下，用於顯示分頁連結的視圖與 [Tailwind CSS](https://tailwindcss.com) 框架相容。然而，如果您沒有使用 Tailwind，您可以自由定義自己的視圖來渲染這些連結。當在 Paginator 實例上呼叫 `links` 方法時，您可以將視圖名稱作為第一個參數傳遞給該方法：

```blade
{{ $paginator->links('view.name') }}

<!-- Passing additional data to the view... -->
{{ $paginator->links('view.name', ['foo' => 'bar']) }}
```

然而，自訂分頁視圖最簡單的方法是使用 `vendor:publish` 命令將它們匯出到您的 `resources/views/vendor` 目錄：

```shell
php artisan vendor:publish --tag=laravel-pagination
```

此命令將把視圖放置在您應用程式的 `resources/views/vendor/pagination` 目錄中。該目錄下的 `tailwind.blade.php` 檔案對應於預設的分頁視圖。您可以編輯此檔案以修改分頁 HTML。

如果您想指定不同的檔案作為預設分頁視圖，您可以在 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫 Paginator 的 `defaultView` 和 `defaultSimpleView` 方法：

```php
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
```

<a name="using-bootstrap"></a>
### 使用 Bootstrap

Laravel 包含了使用 [Bootstrap CSS](https://getbootstrap.com/) 建構的分頁視圖。若要使用這些視圖而非預設的 Tailwind 視圖，您可以在 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫 Paginator 的 `useBootstrapFour` 或 `useBootstrapFive` 方法：

```php
use Illuminate\Pagination\Paginator;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Paginator::useBootstrapFive();
    Paginator::useBootstrapFour();
}
```

<a name="paginator-instance-methods"></a>
## Paginator 與 LengthAwarePaginator 實例方法

每個 Paginator 實例都透過以下方法提供額外的分頁資訊：

<div class="overflow-auto">

| 方法                                  | 說明                                                                                                  |
| --------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| `$paginator->count()`                   | 取得目前頁面項目數量。                                                                |
| `$paginator->currentPage()`             | 取得目前頁碼。                                                                                 |
| `$paginator->firstItem()`               | 取得結果中第一個項目的序號。                                                      |
| `$paginator->getOptions()`              | 取得 Paginator 選項。                                                                                   |
| `$paginator->getUrlRange($start, $end)` | 建立一系列分頁 URL。                                                                           |
| `$paginator->hasPages()`                | 判斷是否有足夠的項目可分成多個頁面。                                            |
| `$paginator->hasMorePages()`            | 判斷資料儲存區中是否還有更多項目。                                                         |
| `$paginator->items()`                   | 取得目前頁面中的項目。                                                                          |
| `$paginator->lastItem()`                | 取得結果中最後一個項目的序號。                                                       |
| `$paginator->lastPage()`                | 取得最後一個可用頁面的頁碼。(使用 `simplePaginate` 時不可用。)                 |
| `$paginator->nextPageUrl()`             | 取得下一頁的 URL。                                                                               |
| `$paginator->onFirstPage()`             | 判斷 Paginator 是否在第一頁。                                                             |
| `$paginator->onLastPage()`              | 判斷 Paginator 是否在最後一頁。                                                              |
| `$paginator->perPage()`                 | 每頁要顯示的項目數量。                                                                    |
| `$paginator->previousPageUrl()`         | 取得上一頁的 URL。                                                                           |
| `$paginator->total()`                   | 判斷資料儲存區中符合條件的項目總數。(使用 `simplePaginate` 時不可用。) |
| `$paginator->url($page)`                | 取得指定頁碼的 URL。                                                                         |
| `$paginator->getPageName()`             | 取得用於儲存頁面的查詢字串變數。                                                        |
| `$paginator->setPageName($name)`        | 設定用於儲存頁面的查詢字串變數。                                                        |
| `$paginator->through($callback)`        | 使用回呼函式轉換每個項目。                                                                        |

</div>


<a name="cursor-paginator-instance-methods"></a>
## Cursor Paginator 實例方法

每個 cursor paginator 實例都透過以下方法提供額外的分頁資訊：

<div class="overflow-auto">

| 方法                          | 說明                                                       |
| ------------------------------- | ----------------------------------------------------------------- |
| `$paginator->count()`           | 取得目前頁面項目數量。                     |
| `$paginator->cursor()`          | 取得目前 cursor 實例。                                  |
| `$paginator->getOptions()`      | 取得 paginator 選項。                                        |
| `$paginator->hasPages()`        | 判斷是否有足夠的項目可分成多個頁面。 |
| `$paginator->hasMorePages()`    | 判斷資料儲存區中是否還有更多項目。              |
| `$paginator->getCursorName()`   | 取得用於儲存 cursor 的查詢字串變數。           |
| `$paginator->items()`           | 取得目前頁面中的項目。                               |
| `$paginator->nextCursor()`      | 取得下一組項目對應的 cursor 實例。                |
| `$paginator->nextPageUrl()`     | 取得下一頁的 URL。                                    |
| `$paginator->onFirstPage()`     | 判斷 paginator 是否在第一頁。                  |
| `$paginator->onLastPage()`      | 判斷 paginator 是否在最後一頁。                   |
| `$paginator->perPage()`         | 每頁要顯示的項目數量。                         |
| `$paginator->previousCursor()`  | 取得上一組項目對應的 cursor 實例。            |
| `$paginator->previousPageUrl()` | 取得上一頁的 URL。                                |
| `$paginator->setCursorName()`   | 設定用於儲存 cursor 的查詢字串變數。           |
| `$paginator->url($cursor)`      | 取得指定 cursor 實例的 URL。                          |

</div>