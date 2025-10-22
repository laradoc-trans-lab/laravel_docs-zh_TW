# 資料庫：分頁

- [簡介](#introduction)
- [基本用法](#basic-usage)
    - [查詢產生器結果的分頁](#paginating-query-builder-results)
    - [Eloquent 結果的分頁](#paginating-eloquent-results)
    - [游標分頁](#cursor-pagination)
    - [手動建立 Paginator](#manually-creating-a-paginator)
    - [自訂分頁 URL](#customizing-pagination-urls)
- [顯示分頁結果](#displaying-pagination-results)
    - [調整分頁連結視窗](#adjusting-the-pagination-link-window)
    - [將結果轉換為 JSON](#converting-results-to-json)
- [自訂分頁視圖](#customizing-the-pagination-view)
    - [使用 Bootstrap](#using-bootstrap)
- [Paginator 與 LengthAwarePaginator 實體方法](#paginator-instance-methods)
- [CursorPaginator 實體方法](#cursor-paginator-instance-methods)

<a name="introduction"></a>
## 簡介

在其他框架中，分頁可能會非常麻煩。我們希望 Laravel 的分頁方法能帶來耳目一新的體驗。Laravel 的 Paginator 已整合至 [query builder](/docs/{{version}}/queries) 和 [Eloquent ORM](/docs/{{version}}/eloquent)，並以零配置的方式提供方便且易用的資料庫記錄分頁功能。

預設情況下，Paginator 產生的 HTML 與 [Tailwind CSS 框架](https://tailwindcss.com/) 相容；不過，也支援 Bootstrap 分頁。

<a name="tailwind"></a>
#### Tailwind

如果您正在使用 Laravel 預設的 Tailwind 分頁視圖並搭配 Tailwind 4.x，您的應用程式的 `resources/css/app.css` 檔案將會已正確配置為 `@source` Laravel 的分頁視圖：

```css
@import 'tailwindcss';

@source '../../vendor/laravel/framework/src/Illuminate/Pagination/resources/views/*.blade.php';
```

<a name="basic-usage"></a>
## 基本用法


<a name="paginating-query-builder-results"></a>
### 查詢產生器結果的分頁

項目分頁有多種方式。最簡單的方式是在[查詢產生器](/docs/{{version}}/queries)或 [Eloquent 查詢](/docs/{{version}}/eloquent)上使用 `paginate` 方法。`paginate` 方法會根據使用者目前正在檢視的頁面，自動處理查詢的「limit」和「offset」設定。預設情況下，目前的頁面是透過 HTTP 請求中的 `page` 查詢字串引數值來偵測的。此值由 Laravel 自動偵測，並會自動插入由 Paginator 產生的連結中。

在此範例中，傳遞給 `paginate` 方法的唯一引數是您希望「每頁」顯示的項目數量。在此情況下，讓我們指定希望每頁顯示 `15` 個項目：

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
#### 簡單分頁

`paginate` 方法會在從資料庫中擷取記錄之前，計算查詢匹配到的總記錄數。這樣做的目的是讓 Paginator 知道總共有多少頁記錄。然而，如果您不打算在應用程式的 UI 中顯示總頁數，那麼記錄計數查詢就是不必要的。

因此，如果您的應用程式 UI 只需顯示簡單的「下一頁」和「上一頁」連結，您可以使用 `simplePaginate` 方法執行單一且高效的查詢：

```php
$users = DB::table('users')->simplePaginate(15);
```


<a name="paginating-eloquent-results"></a>
### Eloquent 結果的分頁

您也可以對 [Eloquent](/docs/{{version}}/eloquent) 查詢進行分頁。在此範例中，我們將對 `App\Models\User` 模型進行分頁，並指出我們計劃每頁顯示 15 條記錄。如您所見，其語法與對查詢產生器結果進行分頁幾乎相同：

```php
use App\Models\User;

$users = User::paginate(15);
```

當然，您可以在設定查詢的其他限制（例如 `where` 語句）之後呼叫 `paginate` 方法：

```php
$users = User::where('votes', '>', 100)->paginate(15);
```

對 Eloquent 模型進行分頁時，您也可以使用 `simplePaginate` 方法：

```php
$users = User::where('votes', '>', 100)->simplePaginate(15);
```

同樣地，您可以使用 `cursorPaginate` 方法對 Eloquent 模型進行游標分頁：

```php
$users = User::where('votes', '>', 100)->cursorPaginate(15);
```


<a name="multiple-paginator-instances-per-page"></a>
#### 每頁多個 Paginator 實例

有時您可能需要在應用程式渲染的單一螢幕上渲染兩個獨立的 Paginator。然而，如果兩個 Paginator 實例都使用 `page` 查詢字串參數來儲存目前頁面，這兩個 Paginator 將會衝突。為了解決這個衝突，您可以透過提供給 `paginate`、`simplePaginate` 和 `cursorPaginate` 方法的第三個引數，傳遞您希望用來儲存 Paginator 目前頁面的查詢字串參數名稱：

```php
use App\Models\User;

$users = User::where('votes', '>', 100)->paginate(
    $perPage = 15, $columns = ['*'], $pageName = 'users'
);
```


<a name="cursor-pagination"></a>
### 游標分頁

雖然 `paginate` 和 `simplePaginate` 使用 SQL「offset」語句建立查詢，但游標分頁是透過建構「where」語句來比較查詢中包含的排序欄位值，提供了所有 Laravel 分頁方法中最有效率的資料庫性能。這種分頁方法特別適合大型資料集和「無限滾動」使用者介面。

與基於偏移量的分頁不同（後者在 Paginator 產生的 URL 查詢字串中包含頁碼），基於游標的分頁會將一個「cursor」字串放置在查詢字串中。該游標是一個編碼字串，包含下一個分頁查詢應該開始分頁的位置以及分頁的方向：

```text
http://localhost/users?cursor=eyJpZCI6MTUsIl9wb2ludHNUb05leHRJdGVtcyI6dHJ1ZX0
```

您可以透過查詢產生器提供的 `cursorPaginate` 方法建立一個基於游標的 Paginator 實例。此方法會回傳一個 `Illuminate\Pagination\CursorPaginator` 實例：

```php
$users = DB::table('users')->orderBy('id')->cursorPaginate(15);
```

一旦您擷取到一個游標 Paginator 實例，您可以像通常使用 `paginate` 和 `simplePaginate` 方法時一樣，[顯示分頁結果](#displaying-pagination-results)。有關游標 Paginator 提供的實例方法的更多資訊，請查閱[游標 Paginator 實例方法文件](#cursor-paginator-instance-methods)。

> [!WARNING]
> 您的查詢必須包含「order by」語句才能利用游標分頁。此外，查詢所排序的欄位必須屬於您正在分頁的資料表。


<a name="cursor-vs-offset-pagination"></a>
#### 游標分頁與偏移量分頁

為了說明偏移量分頁與游標分頁之間的差異，我們來看一些 SQL 查詢範例。以下兩個查詢都會顯示依 `id` 排序的 `users` 資料表「第二頁」的結果：

```sql
# Offset Pagination...
select * from users order by id asc limit 15 offset 15;

# Cursor Pagination...
select * from users where id > 15 order by id asc limit 15;
```

游標分頁查詢相較於偏移量分頁，具有以下優勢：

- 對於大型資料集，如果「order by」欄位已建立索引，游標分頁將提供更好的性能。這是因為「offset」語句會掃描所有先前匹配的資料。
- 對於頻繁寫入的資料集，如果使用者目前檢視的頁面最近新增或刪除了結果，偏移量分頁可能會跳過記錄或顯示重複項目。

然而，游標分頁有以下限制：

- 就像 `simplePaginate` 一樣，游標分頁只能用於顯示「下一頁」和「上一頁」連結，並且不支援產生帶有頁碼的連結。
- 它要求排序基於至少一個唯一欄位或唯一欄位的組合。不支援具有 `null` 值的欄位。
- 「order by」語句中的查詢表達式僅在它們被別名化並同時添加到「select」語句中時才受支援。
- 不支援帶有參數的查詢表達式。


<a name="manually-creating-a-paginator"></a>
### 手動建立 Paginator

有時您可能希望手動建立一個分頁實例，將您記憶體中已有的項目陣列傳遞給它。您可以根據您的需求，透過建立 `Illuminate\Pagination\Paginator`、`Illuminate\Pagination\LengthAwarePaginator` 或 `Illuminate\Pagination\CursorPaginator` 實例來實現。

`Paginator` 和 `CursorPaginator` 類別不需要知道結果集中的項目總數；然而，正因為如此，這些類別沒有用於擷取最後一頁索引的方法。`LengthAwarePaginator` 接受與 `Paginator` 幾乎相同的引數；然而，它需要結果集中項目總數的計數。

換句話說，`Paginator` 對應於查詢產生器上的 `simplePaginate` 方法，`CursorPaginator` 對應於 `cursorPaginate` 方法，而 `LengthAwarePaginator` 則對應於 `paginate` 方法。

> [!WARNING]
> 手動建立 Paginator 實例時，您應該手動「slice」傳遞給 Paginator 的結果陣列。如果您不確定如何操作，請查閱 [array_slice](https://secure.php.net/manual/en/function.array-slice.php) PHP 函式。

<a name="customizing-pagination-urls"></a>
### 自訂分頁 URL

預設情況下，分頁器生成的連結會與當前請求的 URI 相符。然而，分頁器的 `withPath` 方法允許您在生成連結時，自訂分頁器使用的 URI。例如，如果您希望分頁器生成類似 `http://example.com/admin/users?page=N` 的連結，您應該將 `/admin/users` 傳遞給 `withPath` 方法：

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

您可以使用 `appends` 方法將值附加到分頁連結的查詢字串中。例如，若要將 `sort=votes` 附加到每個分頁連結，您應該執行以下對 `appends` 的呼叫：

```php
use App\Models\User;

Route::get('/users', function () {
    $users = User::paginate(15);

    $users->appends(['sort' => 'votes']);

    // ...
});
```

如果您希望將當前請求的所有查詢字串值附加到分頁連結中，您可以使用 `withQueryString` 方法：

```php
$users = User::paginate(15)->withQueryString();
```


<a name="appending-hash-fragments"></a>
#### 附加雜湊片段

如果您需要將「雜湊片段」附加到分頁器生成的 URL 中，您可以使用 `fragment` 方法。例如，若要將 `#users` 附加到每個分頁連結的末尾，您應該如此呼叫 `fragment` 方法：

```php
$users = User::paginate(15)->fragment('users');
```

<a name="displaying-pagination-results"></a>
## 顯示分頁結果

當呼叫 `paginate` 方法時，您將會收到一個 `Illuminate\Pagination\LengthAwarePaginator` 實例，而呼叫 `simplePaginate` 方法則會回傳一個 `Illuminate\Pagination\Paginator` 實例。最後，呼叫 `cursorPaginate` 方法會回傳一個 `Illuminate\Pagination\CursorPaginator` 實例。

這些物件提供了數個方法來描述結果集。除了這些輔助方法外，Paginator 實例也是迭代器，可以像陣列一樣進行迴圈。因此，一旦您取得了結果，您就可以使用 [Blade](/docs/{{version}}/blade) 來顯示結果並渲染頁面連結：

```blade
<div class="container">
    @foreach ($users as $user)
        {{ $user->name }}
    @endforeach
</div>

{{ $users->links() }}
```

`links` 方法將會渲染結果集中其餘頁面的連結。這些連結中的每一個都已經包含正確的 `page` 查詢字串變數。請記住，`links` 方法所產生的 HTML 與 [Tailwind CSS 框架](https://tailwindcss.com) 相容。

<a name="adjusting-the-pagination-link-window"></a>
### 調整分頁連結視窗

當 Paginator 顯示分頁連結時，會顯示目前的頁碼，以及目前頁碼前後三頁的連結。您可以使用 `onEachSide` 方法來控制在 Paginator 產生的中間滑動視窗內，目前頁面兩側顯示額外連結的數量：

```blade
{{ $users->onEachSide(5)->links() }}
```

<a name="converting-results-to-json"></a>
### 將結果轉換為 JSON

Laravel Paginator 類別實作了 `Illuminate\Contracts\Support\Jsonable` 介面契約並公開了 `toJson` 方法，因此將您的分頁結果轉換為 JSON 非常容易。您也可以透過從路由或控制器動作中回傳 Paginator 實例來將其轉換為 JSON：

```php
use App\Models\User;

Route::get('/users', function () {
    return User::paginate();
});
```

來自 Paginator 的 JSON 將包含中繼資訊，例如 `total`、`current_page`、`last_page` 等。結果記錄可透過 JSON 陣列中的 `data` 鍵取得。以下是從路由回傳 Paginator 實例所建立的 JSON 範例：

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

預設情況下，用於顯示分頁連結的視圖與 [Tailwind CSS](https://tailwindcss.com) 框架相容。然而，如果您沒有使用 Tailwind，您可以自由定義自己的視圖來渲染這些連結。當在 Paginator 實例上呼叫 `links` 方法時，您可以將視圖名稱作為該方法的第一個參數傳遞：

```blade
{{ $paginator->links('view.name') }}

<!-- Passing additional data to the view... -->
{{ $paginator->links('view.name', ['foo' => 'bar']) }}
```

然而，自訂分頁視圖最簡單的方法是使用 `vendor:publish` 指令將它們匯出到您的 `resources/views/vendor` 目錄中：

```shell
php artisan vendor:publish --tag=laravel-pagination
```

這個指令會將視圖放置在您應用程式的 `resources/views/vendor/pagination` 目錄中。該目錄下的 `tailwind.blade.php` 檔案對應著預設的分頁視圖。您可以編輯此檔案以修改分頁 HTML。

如果您想指定不同的檔案作為預設分頁視圖，您可以在您的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫 Paginator 的 `defaultView` 和 `defaultSimpleView` 方法：

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

Laravel 包含了使用 [Bootstrap CSS](https://getbootstrap.com/) 建構的分頁視圖。要使用這些視圖而不是預設的 Tailwind 視圖，您可以在您的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫 Paginator 的 `useBootstrapFour` 或 `useBootstrapFive` 方法：

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
## Paginator 與 LengthAwarePaginator 實體方法

每個 Paginator 實體都透過以下方法提供額外的分頁資訊：

<div class="overflow-auto">

| Method                                  | Description                                                                                                  |
| --------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| `$paginator->count()`                   | 取得目前頁面的項目數量。                                                                                   |
| `$paginator->currentPage()`             | 取得目前頁碼。                                                                                               |
| `$paginator->firstItem()`               | 取得結果集中第一個項目的編號。                                                                               |
| `$paginator->getOptions()`              | 取得 Paginator 的選項。                                                                                      |
| `$paginator->getUrlRange($start, $end)` | 建立一個分頁 URL 範圍。                                                                                      |
| `$paginator->hasPages()`                | 判斷是否有足夠的項目可以分成多個頁面。                                                                       |
| `$paginator->hasMorePages()`            | 判斷資料儲存區中是否還有更多項目。                                                                           |
| `$paginator->items()`                   | 取得目前頁面的項目。                                                                                         |
| `$paginator->lastItem()`                | 取得結果集中最後一個項目的編號。                                                                             |
| `$paginator->lastPage()`                | 取得最後一個可用頁面的頁碼。(使用 `simplePaginate` 時不可用)。                                               |
| `$paginator->nextPageUrl()`             | 取得下一頁的 URL。                                                                                           |
| `$paginator->onFirstPage()`             | 判斷 Paginator 是否位於第一頁。                                                                              |
| `$paginator->onLastPage()`              | 判斷 Paginator 是否位於最後一頁。                                                                            |
| `$paginator->perPage()`                 | 每頁要顯示的項目數量。                                                                                       |
| `$paginator->previousPageUrl()`         | 取得上一頁的 URL。                                                                                           |
| `$paginator->total()`                   | 判斷資料儲存區中符合條件的項目總數。(使用 `simplePaginate` 時不可用)。                                       |
| `$paginator->url($page)`                | 取得給定頁碼的 URL。                                                                                         |
| `$paginator->getPageName()`             | 取得用於儲存頁面的查詢字串變數。                                                                             |
| `$paginator->setPageName($name)`        | 設定用於儲存頁面的查詢字串變數。                                                                             |
| `$paginator->through($callback)`        | 使用回呼函式轉換每個項目。                                                                                   |

</div>


<a name="cursor-paginator-instance-methods"></a>
## CursorPaginator 實體方法

每個 Cursor Paginator 實體都透過以下方法提供額外的分頁資訊：

<div class="overflow-auto">

| Method                          | Description                                                       |
| ------------------------------- | ----------------------------------------------------------------- |
| `$paginator->count()`           | 取得目前頁面的項目數量。                                          |
| `$paginator->cursor()`          | 取得目前的 Cursor 實體。                                          |
| `$paginator->getOptions()`      | 取得 Paginator 的選項。                                           |
| `$paginator->hasPages()`        | 判斷是否有足夠的項目可以分成多個頁面。                            |
| `$paginator->hasMorePages()`    | 判斷資料儲存區中是否還有更多項目。                                |
| `$paginator->getCursorName()`   | 取得用於儲存 Cursor 的查詢字串變數。                              |
| `$paginator->items()`           | 取得目前頁面的項目。                                              |
| `$paginator->nextCursor()`      | 取得下一組項目的 Cursor 實體。                                    |
| `$paginator->nextPageUrl()`     | 取得下一頁的 URL。                                                |
| `$paginator->onFirstPage()`     | 判斷 Paginator 是否位於第一頁。                                   |
| `$paginator->onLastPage()`      | 判斷 Paginator 是否位於最後一頁。                                 |
| `$paginator->perPage()`         | 每頁要顯示的項目數量。                                            |
| `$paginator->previousCursor()`  | 取得上一組項目的 Cursor 實體。                                    |
| `$paginator->previousPageUrl()` | 取得上一頁的 URL。                                                |
| `$paginator->setCursorName()`   | 設定用於儲存 Cursor 的查詢字串變數。                              |
| `$paginator->url($cursor)`      | 取得給定 Cursor 實體的 URL。                                      |

</div>