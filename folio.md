# Laravel Folio

- [簡介](#introduction)
- [安裝](#installation)
    - [頁面路徑 / URI](#page-paths-uris)
    - [子網域路由](#subdomain-routing)
- [建立路由](#creating-routes)
    - [巢狀路由](#nested-routes)
    - [索引路由](#index-routes)
- [路由參數](#route-parameters)
- [路由模型繫結](#route-model-binding)
    - [軟刪除模型](#soft-deleted-models)
- [渲染掛鉤](#render-hooks)
- [具名路由](#named-routes)
- [Middleware](#middleware)
- [路由快取](#route-caching)

<a name="introduction"></a>
## 簡介

[Laravel Folio](https://github.com/laravel/folio) 是一個強大的基於頁面的路由工具，旨在簡化 Laravel 應用程式中的路由設定。有了 Laravel Folio，建立路由就像在應用程式的 `resources/views/pages` 目錄中建立 Blade 模板一樣輕鬆。

例如，若要建立一個可透過 `/greeting` URL 存取的頁面，只需在應用程式的 `resources/views/pages` 目錄中建立一個 `greeting.blade.php` 檔案：

```php
<div>
    Hello World
</div>
```

<a name="installation"></a>
## 安裝

首先，使用 Composer 套件管理器將 Folio 安裝到您的專案中：

```bash
composer require laravel/folio
```

安裝 Folio 後，您可以執行 `folio:install` Artisan 命令，這將會把 Folio 的 Service Provider 安裝到您的應用程式中。此 Service Provider 會註冊 Folio 搜尋路由 / 頁面的目錄：

```bash
php artisan folio:install
```

<a name="page-paths-uris"></a>
### 頁面路徑 / URI

預設情況下，Folio 會從應用程式的 `resources/views/pages` 目錄提供頁面，但您可以在 Folio Service Provider 的 `boot` 方法中自訂這些目錄。

例如，有時在同一個 Laravel 應用程式中指定多個 Folio 路徑可能會很方便。您可能希望為應用程式的「admin」區域設定一個單獨的 Folio 頁面目錄，同時使用另一個目錄來存放應用程式的其他頁面。

您可以使用 `Folio::path` 和 `Folio::uri` 方法來實現此目的。`path` 方法會註冊一個目錄，Folio 將在路由傳入的 HTTP 請求時掃描該目錄以尋找頁面，而 `uri` 方法則指定該頁面目錄的「基本 URI」：

```php
use Laravel\Folio\Folio;

Folio::path(resource_path('views/pages/guest'))->uri('/');

Folio::path(resource_path('views/pages/admin'))
    ->uri('/admin')
    ->middleware([
        '*' => [
            'auth',
            'verified',

            // ...
        ],
    ]);
```

<a name="subdomain-routing"></a>
### 子網域路由

您也可以根據傳入請求的子網域來路由頁面。例如，您可能希望將來自 `admin.example.com` 的請求路由到與其他 Folio 頁面不同的頁面目錄。您可以透過在呼叫 `Folio::path` 方法後呼叫 `domain` 方法來實現此目的：

```php
use Laravel\Folio\Folio;

Folio::domain('admin.example.com')
    ->path(resource_path('views/pages/admin'));
```

`domain` 方法也允許您將網域或子網域的一部分擷取為參數。這些參數將會被注入到您的頁面模板中：

```php
use Laravel\Folio\Folio;

Folio::domain('{account}.example.com')
    ->path(resource_path('views/pages/admin'));
```

<a name="creating-routes"></a>
## 建立路由

您可以透過將 Blade 模板放置在任何 Folio 掛載的目錄中來建立 Folio 路由。預設情況下，Folio 會掛載 `resources/views/pages` 目錄，但您可以在 Folio Service Provider 的 `boot` 方法中自訂這些目錄。

一旦 Blade 模板被放置在 Folio 掛載的目錄中，您就可以立即透過瀏覽器存取它。例如，放置在 `pages/schedule.blade.php` 中的頁面可以在瀏覽器中透過 `http://example.com/schedule` 存取。

若要快速查看所有 Folio 頁面 / 路由的列表，您可以呼叫 `folio:list` Artisan 命令：

```bash
php artisan folio:list
```

<a name="nested-routes"></a>
### 巢狀路由

您可以透過在 Folio 的目錄中建立一個或多個目錄來建立巢狀路由。例如，若要建立一個可透過 `/user/profile` 存取的頁面，請在 `pages/user` 目錄中建立一個 `profile.blade.php` 模板：

```bash
php artisan folio:page user/profile

# pages/user/profile.blade.php → /user/profile
```

<a name="index-routes"></a>
### 索引路由

有時，您可能希望將給定的頁面設為目錄的「索引」。透過在 Folio 目錄中放置 `index.blade.php` 模板，任何對該目錄根目錄的請求都將路由到該頁面：

```bash
php artisan folio:page index
# pages/index.blade.php → /

php artisan folio:page users/index
# pages/users/index.blade.php → /users
```

<a name="route-parameters"></a>
## 路由參數

通常，您需要將傳入請求 URL 的片段注入到您的頁面中，以便您可以與它們互動。例如，您可能需要存取正在顯示其個人資料的使用者的「ID」。為此，您可以將頁面檔案名稱的片段封裝在方括號中：

```bash
php artisan folio:page "users/[id]"

# pages/users/[id].blade.php → /users/1
```

擷取的片段可以作為變數在您的 Blade 模板中存取：

```html
<div>
    User {{ $id }}
</div>
```

若要擷取多個片段，您可以在封裝的片段前加上三個點 `...`：

```bash
php artisan folio:page "users/[...ids]"

# pages/users/[...ids].blade.php → /users/1/2/3
```

當擷取多個片段時，擷取的片段將作為陣列注入到頁面中：

```html
<ul>
    @foreach ($ids as $id)
        <li>User {{ $id }}</li>
    @endforeach
</ul>
```

<a name="route-model-binding"></a>
## 路由模型繫結

如果您的頁面模板檔案名稱中的萬用字元片段對應到您應用程式的其中一個 Eloquent 模型，Folio 將會自動利用 Laravel 的路由模型繫結功能，並嘗試將解析後的模型實例注入到您的頁面中：

```bash
php artisan folio:page "users/[User]"

# pages/users/[User].blade.php → /users/1
```

擷取的模型可以作為變數在您的 Blade 模板中存取。模型的變數名稱將會轉換為「camel case」：

```html
<div>
    User {{ $user->id }}
</div>
```

#### 自訂鍵值

有時您可能希望使用 `id` 以外的欄位來解析繫結的 Eloquent 模型。為此，您可以在頁面的檔案名稱中指定欄位。例如，檔案名稱為 `[Post:slug].blade.php` 的頁面將會嘗試透過 `slug` 欄位而不是 `id` 欄位來解析繫結的模型。

在 Windows 上，您應該使用 `-` 來分隔模型名稱和鍵值：`[Post-slug].blade.php`。

#### 模型位置

預設情況下，Folio 將會在您應用程式的 `app/Models` 目錄中搜尋您的模型。但是，如果需要，您可以在模板的檔案名稱中指定完整的模型類別名稱：

```bash
php artisan folio:page "users/[.App.Models.User]"

# pages/users/[.App.Models.User].blade.php → /users/1
```

<a name="soft-deleted-models"></a>
### 軟刪除模型

預設情況下，在解析隱式模型繫結時，不會擷取已軟刪除的模型。但是，如果您希望，可以透過在頁面模板中呼叫 `withTrashed` 函數來指示 Folio 擷取軟刪除的模型：

```php
<?php

use function Laravel\Folio\{withTrashed};

withTrashed();

?>

<div>
    User {{ $user->id }}
</div>
```

<a name="render-hooks"></a>
## 渲染掛鉤

預設情況下，Folio 會將頁面 Blade 模板的內容作為對傳入請求的回應。但是，您可以透過在頁面模板中呼叫 `render` 函數來自訂回應。

`render` 函數接受一個閉包，該閉包將會接收 Folio 正在渲染的 `View` 實例，允許您向視圖添加額外資料或自訂整個回應。除了接收 `View` 實例之外，任何額外的路由參數或模型繫結也將提供給 `render` 閉包：

```php
<?php

use App\Models\Post;
use Illuminate\Support\Facades\Auth;
use Illuminate\View\View;

use function Laravel\Folio\render;

render(function (View $view, Post $post) {
    if (! Auth::user()->can('view', $post)) {
        return response('Unauthorized', 403);
    }

    return $view->with('photos', $post->author->photos);
}); ?>

<div>
    {{ $post->content }}
</div>

<div>
    This author has also taken {{ count($photos) }} photos.
</div>
```

<a name="named-routes"></a>
## 具名路由

您可以使用 `name` 函數為給定頁面的路由指定名稱：

```php
<?php

use function Laravel\Folio\name;

name('users.index');
```

就像 Laravel 的具名路由一樣，您可以使用 `route` 函數來產生已分配名稱的 Folio 頁面的 URL：

```php
<a href="{{ route('users.index') }}">
    All Users
</a>
```

如果頁面有參數，您可以直接將其值傳遞給 `route` 函數：

```php
route('users.show', ['user' => $user]);
```

<a name="middleware"></a>
## Middleware

您可以透過在頁面模板中呼叫 `middleware` 函數來將 Middleware 應用於特定頁面：

```php
<?php

use function Laravel\Folio\{middleware};

middleware(['auth', 'verified']);

?>

<div>
    Dashboard
</div>
```

或者，若要將 Middleware 分配給一組頁面，您可以在呼叫 `Folio::path` 方法後鏈接 `middleware` 方法。

若要指定 Middleware 應應用於哪些頁面，Middleware 陣列可以使用其應應用頁面的相應 URL 模式作為鍵。`*` 字元可以用作萬用字元：

```php
use Laravel\Folio\Folio;

Folio::path(resource_path('views/pages'))->middleware([
    'admin/*' => [
        'auth',
        'verified',

        // ...
    ],
]);
```

您可以在 Middleware 陣列中包含閉包來定義行內匿名 Middleware：

```php
use Closure;
use Illuminate\Http\Request;
use Laravel\Folio\Folio;

Folio::path(resource_path('views/pages'))->middleware([
    'admin/*' => [
        'auth',
        'verified',

        function (Request $request, Closure $next) {
            // ...

            return $next($request);
        },
    ],
]);
```

<a name="route-caching"></a>
## 路由快取

使用 Folio 時，您應該始終利用 [Laravel 的路由快取功能](/docs/{{version}}/routing#route-caching)。Folio 會監聽 `route:cache` Artisan 命令，以確保 Folio 頁面定義和路由名稱正確快取，以實現最大效能。
