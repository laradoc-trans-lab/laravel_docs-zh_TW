# 授權

- [簡介](#introduction)
- [Gates](#gates)
    - [撰寫 Gates](#writing-gates)
    - [授權操作](#authorizing-actions-via-gates)
    - [Gate 回應](#gate-responses)
    - [攔截 Gate 檢查](#intercepting-gate-checks)
    - [行內授權](#inline-authorization)
- [建立 Policies](#creating-policies)
    - [產生 Policies](#generating-policies)
    - [註冊 Policies](#registering-policies)
- [撰寫 Policies](#writing-policies)
    - [Policy 方法](#policy-methods)
    - [Policy 回應](#policy-responses)
    - [不需 Model 的方法](#methods-without-models)
    - [訪客使用者](#guest-users)
    - [Policy 篩選器](#policy-filters)
- [使用 Policies 授權操作](#authorizing-actions-using-policies)
    - [透過 User Model](#via-the-user-model)
    - [透過 `Gate` Facade](#via-the-gate-facade)
    - [透過 Middleware](#via-middleware)
    - [透過 Blade 模板](#via-blade-templates)
    - [提供額外上下文](#supplying-additional-context)
- [授權與 Inertia](#authorization-and-inertia)

<a name="introduction"></a>
## 簡介

除了提供內建的 [認證](/docs/{{version}}/authentication) 服務之外，Laravel 也提供了一種簡單的方式來授權使用者對特定資源的操作。例如，即使使用者已通過認證，他們也可能無權更新或刪除應用程式所管理的某些 Eloquent Model 或資料庫記錄。Laravel 的授權功能提供了一種簡單、有組織的方式來管理這些類型的授權檢查。

Laravel 提供了兩種主要的授權方式：[Gates](#gates) 和 [Policies](#creating-policies)。可以將 Gates 和 Policies 想像成路由和控制器。Gates 提供了一種簡單、基於閉包的授權方法，而 Policies 則像控制器一樣，將邏輯圍繞特定的 Model 或資源進行分組。在這份文件中，我們將先探討 Gates，然後再研究 Policies。

在建構應用程式時，您不需要在單獨使用 Gates 或單獨使用 Policies 之間做出選擇。大多數應用程式很可能會同時包含 Gates 和 Policies 的組合，這是完全可以接受的！Gates 最適用於與任何 Model 或資源無關的操作，例如檢視管理員儀表板。相反地，當您希望授權特定 Model 或資源的操作時，應使用 Policies。

<a name="gates"></a>
## Gates

<a name="writing-gates"></a>
### 撰寫 Gates

> [!WARNING]
> Gates 是學習 Laravel 授權功能基礎的好方法；然而，在建構強大的 Laravel 應用程式時，您應該考慮使用 [policies](#creating-policies) 來組織您的授權規則。

Gates 就是決定使用者是否有權限執行某項指定操作的 Closure。通常，Gates 會使用 `Gate` Facade 在 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中定義。Gates 始終會將使用者實例作為第一個引數，並可選地接收額外引數，例如相關的 Eloquent model。

在此範例中，我們將定義一個 Gate 來判斷使用者是否可以更新指定的 `App\Models\Post` model。Gate 將透過比較使用者的 `id` 與建立該文章的使用者的 `user_id` 來實現此功能：

```php
use App\Models\Post;
use App\Models\User;
use Illuminate\Support\Facades\Gate;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Gate::define('update-post', function (User $user, Post $post) {
        return $user->id === $post->user_id;
    });
}
```

如同控制器，Gates 也可以使用類別回呼陣列來定義：

```php
use App\Policies\PostPolicy;
use Illuminate\Support\Facades\Gate;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Gate::define('update-post', [PostPolicy::class, 'update']);
}
```

<a name="authorizing-actions-via-gates"></a>
### 授權操作

要使用 Gates 授權操作，您應該使用 `Gate` Facade 提供的 `allows` 或 `denies` 方法。請注意，您不需要將當前認證的使用者傳遞給這些方法。Laravel 會自動處理將使用者傳遞給 Gate Closure。通常會在您的應用程式控制器中呼叫 Gate 授權方法，然後再執行需要授權的操作：

```php
<?php

namespace App\Http\Controllers;

use App\Models\Post;
use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Gate;

class PostController extends Controller
{
    /**
     * Update the given post.
     */
    public function update(Request $request, Post $post): RedirectResponse
    {
        if (! Gate::allows('update-post', $post)) {
            abort(403);
        }

        // Update the post...

        return redirect('/posts');
    }
}
```

如果您想判斷除了當前認證使用者之外的另一個使用者是否有權限執行某個操作，您可以使用 `Gate` Facade 上的 `forUser` 方法：

```php
if (Gate::forUser($user)->allows('update-post', $post)) {
    // The user can update the post...
}

if (Gate::forUser($user)->denies('update-post', $post)) {
    // The user can't update the post...
}
```

您可以使用 `any` 或 `none` 方法一次授權多個操作：

```php
if (Gate::any(['update-post', 'delete-post'], $post)) {
    // The user can update or delete the post...
}

if (Gate::none(['update-post', 'delete-post'], $post)) {
    // The user can't update or delete the post...
}
```

<a name="authorizing-or-throwing-exceptions"></a>
#### 授權或拋出異常

如果您想嘗試授權某個操作，並且如果使用者不允許執行該操作時自動拋出 `Illuminate\Auth\Access\AuthorizationException`，您可以使用 `Gate` Facade 的 `authorize` 方法。`AuthorizationException` 的實例會被 Laravel 自動轉換為 403 HTTP 回應：

```php
Gate::authorize('update-post', $post);

// The action is authorized...
```

<a name="gates-supplying-additional-context"></a>
#### 提供額外上下文

用於授權能力的 Gate 方法 (`allows`、`denies`、`check`、`any`、`none`、`authorize`、`can`、`cannot`) 和授權 [Blade directives](#via-blade-templates) (`@can`、`@cannot`、`@canany`) 可以接收一個陣列作為它們的第二個引數。這些陣列元素會作為參數傳遞給 Gate Closure，並可用於在進行授權決策時提供額外的上下文：

```php
use App\Models\Category;
use App\Models\User;
use Illuminate\Support\Facades\Gate;

Gate::define('create-post', function (User $user, Category $category, bool $pinned) {
    if (! $user->canPublishToGroup($category->group)) {
        return false;
    } elseif ($pinned && ! $user->canPinPosts()) {
        return false;
    }

    return true;
});

if (Gate::check('create-post', [$category, $pinned])) {
    // The user can create the post...
}
```

<a name="gate-responses"></a>
### Gate 回應

到目前為止，我們只檢視了回傳簡單布林值的 Gates。然而，有時您可能希望回傳更詳細的回應，包括錯誤訊息。為此，您可以從 Gate 回傳一個 `Illuminate\Auth\Access\Response` 實例：

```php
use App\Models\User;
use Illuminate\Auth\Access\Response;
use Illuminate\Support\Facades\Gate;

Gate::define('edit-settings', function (User $user) {
    return $user->isAdmin
        ? Response::allow()
        : Response::deny('You must be an administrator.');
});
```

即使您從 Gate 回傳授權回應，`Gate::allows` 方法仍會回傳一個簡單的布林值；然而，您可以使用 `Gate::inspect` 方法來取得 Gate 回傳的完整授權回應：

```php
$response = Gate::inspect('edit-settings');

if ($response->allowed()) {
    // The action is authorized...
} else {
    echo $response->message();
}
```

當使用 `Gate::authorize` 方法時，如果操作未被授權，它會拋出 `AuthorizationException`，而授權回應提供的錯誤訊息將會傳播到 HTTP 回應：

```php
Gate::authorize('edit-settings');

// The action is authorized...
```

<a name="customizing-gate-response-status"></a>
#### 客製化 HTTP 回應狀態

當透過 Gate 拒絕某個操作時，會回傳 `403` HTTP 回應；然而，有時回傳替代的 HTTP 狀態碼會很有用。您可以使用 `Illuminate\Auth\Access\Response` 類別上的 `denyWithStatus` 靜態建構子來自訂失敗授權檢查所回傳的 HTTP 狀態碼：

```php
use App\Models\User;
use Illuminate\Auth\Access\Response;
use Illuminate\Support\Facades\Gate;

Gate::define('edit-settings', function (User $user) {
    return $user->isAdmin
        ? Response::allow()
        : Response::denyWithStatus(404);
});
```

由於透過 `404` 回應隱藏資源是網頁應用程式的常見模式，因此提供了 `denyAsNotFound` 方法以方便使用：

```php
use App\Models\User;
use Illuminate\Auth\Access\Response;
use Illuminate\Support\Facades\Gate;

Gate::define('edit-settings', function (User $user) {
    return $user->isAdmin
        ? Response::allow()
        : Response::denyAsNotFound();
});
```

<a name="intercepting-gate-checks"></a>
### 攔截 Gate 檢查

有時，您可能希望授予特定使用者所有權限。您可以使用 `before` 方法來定義一個 Closure，該 Closure 在所有其他授權檢查之前執行：

```php
use App\Models\User;
use Illuminate\Support\Facades\Gate;

Gate::before(function (User $user, string $ability) {
    if ($user->isAdministrator()) {
        return true;
    }
});
```

如果 `before` Closure 回傳一個非空結果，該結果將被視為授權檢查的結果。

您可以使用 `after` 方法定義一個 Closure，該 Closure 將在所有其他授權檢查之後執行：

```php
use App\Models\User;

Gate::after(function (User $user, string $ability, bool|null $result, mixed $arguments) {
    if ($user->isAdministrator()) {
        return true;
    }
});
```

`after` Closure 回傳的值不會覆寫授權檢查的結果，除非 Gate 或 Policy 回傳 `null`。

<a name="inline-authorization"></a>
### 行內授權

有時，您可能會希望判斷目前已驗證的使用者是否被授權執行某個動作，而無需撰寫一個與該動作對應的專用 Gate。Laravel 允許您透過 `Gate::allowIf` 和 `Gate::denyIf` 方法來執行這些類型的「行內」授權檢查。行內授權不會執行任何已定義的「[前置 (before)」或「後置 (after)」授權鉤子](#intercepting-gate-checks)：

```php
use App\Models\User;
use Illuminate\Support\Facades\Gate;

Gate::allowIf(fn (User $user) => $user->isAdministrator());

Gate::denyIf(fn (User $user) => $user->banned());
```

如果該動作未被授權，或目前沒有使用者被驗證，Laravel 將會自動拋出一個 `Illuminate\Auth\Access\AuthorizationException` 例外。`AuthorizationException` 的實例會透過 Laravel 的例外處理器自動轉換為一個 403 HTTP 回應。

<a name="creating-policies"></a>
## 建立 Policies

<a name="generating-policies"></a>
### 產生 Policies

Policies 是圍繞特定 model 或資源組織授權邏輯的類別。例如，如果您的應用程式是一個部落格，您可能會有一個 `App\Models\Post` model 和一個對應的 `App\Policies\PostPolicy` 來授權使用者操作，例如建立或更新貼文。

您可以使用 `make:policy` Artisan 指令來產生一個 policy。產生的 policy 將會被放置在 `app/Policies` 目錄中。如果您的應用程式中不存在此目錄，Laravel 會為您建立它：

```shell
php artisan make:policy PostPolicy
```

`make:policy` 指令將會產生一個空的 policy 類別。如果您想產生一個包含與檢視、建立、更新和刪除資源相關的範例 policy 方法的類別，您可以在執行指令時提供 `--model` 選項：

```shell
php artisan make:policy PostPolicy --model=Post
```

<a name="registering-policies"></a>
### 註冊 Policies

<a name="policy-discovery"></a>
#### Policy 發現

預設情況下，只要 model 和 policy 遵循標準的 Laravel 命名慣例，Laravel 就會自動發現 policies。具體來說，policies 必須位於包含您 model 的目錄或其上層目錄中的 `Policies` 目錄內。例如，model 可以放置在 `app/Models` 目錄中，而 policies 則可以放置在 `app/Policies` 目錄中。在這種情況下，Laravel 會先在 `app/Models/Policies` 中檢查 policies，然後在 `app/Policies` 中檢查。此外，policy 名稱必須與 model 名稱匹配，並帶有 `Policy` 尾碼。因此，一個 `User` model 將對應於一個 `UserPolicy` policy 類別。

如果您想定義自己的 policy 發現邏輯，您可以使用 `Gate::guessPolicyNamesUsing` 方法註冊一個自訂的 policy 發現回呼。通常，這個方法應該在您應用程式的 `AppServiceProvider` 的 `boot` 方法中被呼叫：

```php
use Illuminate\Support\Facades\Gate;

Gate::guessPolicyNamesUsing(function (string $modelClass) {
    // Return the name of the policy class for the given model...
});
```

<a name="manually-registering-policies"></a>
#### 手動註冊 Policies

使用 `Gate` Facade，您可以在應用程式的 `AppServiceProvider` 的 `boot` 方法中手動註冊 policies 及其對應的 models：

```php
use App\Models\Order;
use App\Policies\OrderPolicy;
use Illuminate\Support\Facades\Gate;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Gate::policy(Order::class, OrderPolicy::class);
}
```

或者，您可以將 `UsePolicy` attribute 放置在 model 類別上，以告知 Laravel 該 model 對應的 policy：

```php
<?php

namespace App\Models;

use App\Policies\OrderPolicy;
use Illuminate\Database\Eloquent\Attributes\UsePolicy;
use Illuminate\Database\Eloquent\Model;

#[UsePolicy(OrderPolicy::class)]
class Order extends Model
{
    //
}
```

<a name="writing-policies"></a>
## 撰寫 Policies

<a name="policy-methods"></a>
### Policy 方法

一旦 Policy 類別已註冊，您便可以為其授權的每個操作新增方法。例如，讓我們在 `PostPolicy` 上定義一個 `update` 方法，該方法判斷指定的 `App\Models\User` 是否可以更新指定的 `App\Models\Post` 實例。

`update` 方法會接收一個 `User` 實例和一個 `Post` 實例作為其參數，並應回傳 `true` 或 `false`，指示該使用者是否被授權更新指定的 `Post`。因此，在此範例中，我們將驗證使用者的 `id` 是否與貼文的 `user_id` 相符：

```php
<?php

namespace App\Policies;

use App\Models\Post;
use App\Models\User;

class PostPolicy
{
    /**
     * Determine if the given post can be updated by the user.
     */
    public function update(User $user, Post $post): bool
    {
        return $user->id === $post->user_id;
    }
}
```

您可以根據需要繼續在 Policy 上定義其他方法，以處理其授權的各種操作。例如，您可以定義 `view` 或 `delete` 方法來授權各種 `Post` 相關操作，但請記住，您可以隨意命名您的 Policy 方法。

如果您在使用 Artisan 命令列產生 Policy 時使用了 `--model` 選項，它將已經包含 `viewAny`、`view`、`create`、`update`、`delete`、`restore` 和 `forceDelete` 操作的方法。

> [!NOTE]
> 所有 Policies 都透過 Laravel [服務容器](/docs/{{version}}/container)解析，讓您可以在 Policy 的建構函式中型別提示任何所需的依賴，以讓它們自動注入。

<a name="policy-responses"></a>
### Policy 回應

到目前為止，我們只檢視了回傳簡單布林值的 Policy 方法。然而，有時您可能希望回傳更詳細的回應，包括錯誤訊息。為此，您可以從您的 Policy 方法中回傳一個 `Illuminate\Auth\Access\Response` 實例：

```php
use App\Models\Post;
use App\Models\User;
use Illuminate\Auth\Access\Response;

/**
 * Determine if the given post can be updated by the user.
 */
public function update(User $user, Post $post): Response
{
    return $user->id === $post->user_id
        ? Response::allow()
        : Response::deny('You do not own this post.');
}
```

當從您的 Policy 回傳授權回應時，`Gate::allows` 方法仍會回傳一個簡單的布林值；但是，您可以使用 `Gate::inspect` 方法來取得 Policy 所回傳的完整授權回應：

```php
use Illuminate\Support\Facades\Gate;

$response = Gate::inspect('update', $post);

if ($response->allowed()) {
    // The action is authorized...
} else {
    echo $response->message();
}
```

當使用 `Gate::authorize` 方法時，如果操作未經授權，該方法會拋出 `AuthorizationException` 例外。此時，授權回應所提供的錯誤訊息將會傳播到 HTTP 回應中：

```php
Gate::authorize('update', $post);

// The action is authorized...
```

<a name="customizing-policy-response-status"></a>
#### 自訂 HTTP 回應狀態

當操作經由 Policy 方法拒絕時，會回傳 `403` HTTP 回應；然而，有時回傳替代的 HTTP 狀態碼會很有用。您可以使用 `Illuminate\Auth\Access\Response` 類別上的 `denyWithStatus` 靜態建構函式，來自訂失敗授權檢查所回傳的 HTTP 狀態碼：

```php
use App\Models\Post;
use App\Models\User;
use Illuminate\Auth\Access\Response;

/**
 * Determine if the given post can be updated by the user.
 */
public function update(User $user, Post $post): Response
{
    return $user->id === $post->user_id
        ? Response::allow()
        : Response::denyWithStatus(404);
}
```

由於透過 `404` 回應隱藏資源是網頁應用程式中常見的模式，因此提供了 `denyAsNotFound` 方法以方便使用：

```php
use App\Models\Post;
use App\Models\User;
use Illuminate\Auth\Access\Response;

/**
 * Determine if the given post can be updated by the user.
 */
public function update(User $user, Post $post): Response
{
    return $user->id === $post->user_id
        ? Response::allow()
        : Response::denyAsNotFound();
}
```

<a name="methods-without-models"></a>
### 不需 Model 的方法

有些 Policy 方法只接收目前已認證使用者的實例。這種情況在授權 `create` 操作時最為常見。例如，如果您正在建立一個部落格，您可能希望判斷使用者是否被授權建立任何貼文。在這些情況下，您的 Policy 方法應只期望接收一個使用者實例：

```php
/**
 * Determine if the given user can create posts.
 */
public function create(User $user): bool
{
    return $user->role == 'writer';
}
```

<a name="guest-users"></a>
### 訪客使用者

預設情況下，如果傳入的 HTTP 請求不是由已認證使用者發起的，則所有 Gates 和 Policies 都會自動回傳 `false`。但是，您可以透過宣告「選用」的型別提示或為使用者參數定義提供 `null` 預設值，來允許這些授權檢查通過您的 Gates 和 Policies：

```php
<?php

namespace App\Policies;

use App\Models\Post;
use App\Models\User;

class PostPolicy
{
    /**
     * Determine if the given post can be updated by the user.
     */
    public function update(?User $user, Post $post): bool
    {
        return $user?->id === $post->user_id;
    }
}
```

<a name="policy-filters"></a>
### Policy 篩選器

對於某些使用者，您可能希望授權某個 Policy 中的所有操作。為此，請在 Policy 上定義一個 `before` 方法。`before` 方法將在 Policy 上的所有其他方法之前執行，讓您有機會在實際呼叫預期 Policy 方法之前授權該操作。此功能最常用於授權應用程式管理員執行任何操作：

```php
use App\Models\User;

/**
 * Perform pre-authorization checks.
 */
public function before(User $user, string $ability): bool|null
{
    if ($user->isAdministrator()) {
        return true;
    }

    return null;
}
```

如果您想拒絕特定類型使用者的所有授權檢查，則可以從 `before` 方法回傳 `false`。如果回傳 `null`，授權檢查將會繼續執行 Policy 方法。

> [!WARNING]
> 如果 Policy 類別不包含與所檢查能力名稱相符的方法，則不會呼叫 Policy 類別的 `before` 方法。

<a name="authorizing-actions-using-policies"></a>
## 使用 Policies 授權操作

<a name="via-the-user-model"></a>
### 透過 User Model

您的 Laravel 應用程式隨附的 `App\Models\User` model 包含兩個實用的方法，用於授權操作：`can` 和 `cannot`。`can` 和 `cannot` 方法接收您希望授權的操作名稱以及相關的 model。例如，讓我們判斷使用者是否被授權更新指定的 `App\Models\Post` model。通常，這會在控制器方法中完成：

```php
<?php

namespace App\Http\Controllers;

use App\Models\Post;
use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;

class PostController extends Controller
{
    /**
     * Update the given post.
     */
    public function update(Request $request, Post $post): RedirectResponse
    {
        if ($request->user()->cannot('update', $post)) {
            abort(403);
        }

        // Update the post...

        return redirect('/posts');
    }
}
```

如果已為指定的 model [註冊 Policy](#registering-policies)，`can` 方法將自動呼叫適當的 Policy 並返回布林值結果。如果沒有為該 model 註冊 Policy，`can` 方法將嘗試呼叫與指定操作名稱相符的基於閉包的 Gate。

<a name="user-model-actions-that-dont-require-models"></a>
#### 不需 Model 的操作

請記住，某些操作可能對應到不需要 model 實例的 Policy 方法，例如 `create`。在這些情況下，您可以將類別名稱傳遞給 `can` 方法。該類別名稱將用於確定授權操作時要使用的 Policy：

```php
<?php

namespace App\Http\Controllers;

use App\Models\Post;
use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;

class PostController extends Controller
{
    /**
     * Create a post.
     */
    public function store(Request $request): RedirectResponse
    {
        if ($request->user()->cannot('create', Post::class)) {
            abort(403);
        }

        // Create the post...

        return redirect('/posts');
    }
}
```

<a name="via-the-gate-facade"></a>
### 透過 `Gate` Facade

除了提供給 `App\Models\User` model 的實用方法之外，您還可以透過 `Gate` Facade 的 `authorize` 方法來授權操作。

與 `can` 方法類似，此方法接受您希望授權的操作名稱和相關的 model。如果操作未經授權，`authorize` 方法將拋出 `Illuminate\Auth\Access\AuthorizationException` 異常，Laravel 的異常處理器會自動將其轉換為狀態碼為 403 的 HTTP 回應：

```php
<?php

namespace App\Http\Controllers;

use App\Models\Post;
use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Gate;

class PostController extends Controller
{
    /**
     * Update the given blog post.
     *
     * @throws \Illuminate\Auth\Access\AuthorizationException
     */
    public function update(Request $request, Post $post): RedirectResponse
    {
        Gate::authorize('update', $post);

        // The current user can update the blog post...

        return redirect('/posts');
    }
}
```

<a name="controller-actions-that-dont-require-models"></a>
#### 不需 Model 的操作

如前所述，某些 Policy 方法（例如 `create`）不需要 model 實例。在這些情況下，您應該將類別名稱傳遞給 `authorize` 方法。該類別名稱將用於確定授權操作時要使用的 Policy：

```php
use App\Models\Post;
use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Gate;

/**
 * Create a new blog post.
 *
 * @throws \Illuminate\Auth\Access\AuthorizationException
 */
public function create(Request $request): RedirectResponse
{
    Gate::authorize('create', Post::class);

    // The current user can create blog posts...

    return redirect('/posts');
}
```

<a name="via-middleware"></a>
### 透過 Middleware

Laravel 包含一個中介層，可以在傳入的請求到達您的路由或控制器之前授權操作。預設情況下，`Illuminate\Auth\Middleware\Authorize` 中介層可以使用 `can` [中介層別名](/docs/{{version}}/middleware#middleware-aliases) 附加到路由，該別名由 Laravel 自動註冊。讓我們看一個使用 `can` 中介層來授權使用者可以更新貼文的範例：

```php
use App\Models\Post;

Route::put('/post/{post}', function (Post $post) {
    // The current user may update the post...
})->middleware('can:update,post');
```

在此範例中，我們將兩個參數傳遞給 `can` 中介層。第一個是我們希望授權的操作名稱，第二個是我們希望傳遞給 Policy 方法的路由參數。在本例中，由於我們使用 [隱式 Model 綁定](/docs/{{version}}/routing#implicit-binding)，`App\Models\Post` model 將被傳遞給 Policy 方法。如果使用者未被授權執行指定操作，中介層將返回狀態碼為 403 的 HTTP 回應。

為了方便起見，您也可以使用 `can` 方法將 `can` 中介層附加到您的路由：

```php
use App\Models\Post;

Route::put('/post/{post}', function (Post $post) {
    // The current user may update the post...
})->can('update', 'post');
```

<a name="middleware-actions-that-dont-require-models"></a>
#### 不需 Model 的操作

同樣地，某些 Policy 方法（例如 `create`）不需要 model 實例。在這些情況下，您可以將類別名稱傳遞給中介層。該類別名稱將用於確定授權操作時要使用的 Policy：

```php
Route::post('/post', function () {
    // The current user may create posts...
})->middleware('can:create,App\Models\Post');
```

在字串中介層定義中指定完整的類別名稱可能會變得繁瑣。因此，您可以選擇使用 `can` 方法將 `can` 中介層附加到您的路由：

```php
use App\Models\Post;

Route::post('/post', function () {
    // The current user may create posts...
})->can('create', Post::class);
```

<a name="via-blade-templates"></a>
### 透過 Blade 模板

撰寫 Blade 模板時，您可能希望僅在使用者被授權執行指定動作時才顯示頁面的一部分。例如，您可能希望僅在使用者確實可以更新部落格文章時才顯示其更新表單。在這種情況下，您可以使用 `@can` 與 `@cannot` 指令：

```blade
@can('update', $post)
    <!-- The current user can update the post... -->
@elsecan('create', App\Models\Post::class)
    <!-- The current user can create new posts... -->
@else
    <!-- ... -->
@endcan

@cannot('update', $post)
    <!-- The current user cannot update the post... -->
@elsecannot('create', App\Models\Post::class)
    <!-- The current user cannot create new posts... -->
@endcannot
```

這些指令是撰寫 `@if` 與 `@unless` 語句的方便捷徑。上述的 `@can` 與 `@cannot` 語句等同於以下語句：

```blade
@if (Auth::user()->can('update', $post))
    <!-- The current user can update the post... -->
@endif

@unless (Auth::user()->can('update', $post))
    <!-- The current user cannot update the post... -->
@endunless
```

您也可以判斷使用者是否被授權執行指定操作陣列中的任何動作。為此，請使用 `@canany` 指令：

```blade
@canany(['update', 'view', 'delete'], $post)
    <!-- The current user can update, view, or delete the post... -->
@elsecanany(['create'], \App\Models\Post::class)
    <!-- The current user can create a post... -->
@endcanany
```

<a name="blade-actions-that-dont-require-models"></a>
#### Blade 不需 Model 的操作

與大多數其他授權方法一樣，如果動作不需 model 實例，您可以將類別名稱傳遞給 `@can` 與 `@cannot` 指令：

```blade
@can('create', App\Models\Post::class)
    <!-- The current user can create posts... -->
@endcan

@cannot('create', App\Models\Post::class)
    <!-- The current user can't create posts... -->
@endcannot
```

<a name="supplying-additional-context"></a>
### 提供額外上下文

使用 Policies 授權操作時，您可以將陣列作為第二個引數傳遞給各種授權函式與輔助函式。陣列中的第一個元素將用於決定要呼叫哪個 Policy，而陣列其餘的元素則作為參數傳遞給 Policy 方法，並可用於進行授權判斷時的額外上下文。例如，考慮以下包含額外 `$category` 參數的 `PostPolicy` 方法定義：

```php
/**
 * Determine if the given post can be updated by the user.
 */
public function update(User $user, Post $post, int $category): bool
{
    return $user->id === $post->user_id &&
           $user->canUpdateCategory($category);
}
```

當嘗試判斷已驗證的使用者是否可以更新指定的文章時，我們可以這樣呼叫這個 Policy 方法：

```php
/**
 * Update the given blog post.
 *
 * @throws \Illuminate\Auth\Access\AuthorizationException
 */
public function update(Request $request, Post $post): RedirectResponse
{
    Gate::authorize('update', [$post, $request->category]);

    // The current user can update the blog post...

    return redirect('/posts');
}
```

<a name="authorization-and-inertia"></a>
## 授權與 Inertia

儘管授權必須始終在伺服器端處理，但為前端應用程式提供授權資料以正確渲染應用程式的 UI 通常會很方便。Laravel 並沒有定義一個強制性的慣例來將授權資訊暴露給由 Inertia 驅動的前端。

然而，如果您使用 Laravel 基於 Inertia 的 [入門套件](/docs/{{version}}/starter-kits) 之一，您的應用程式已包含一個 `HandleInertiaRequests` middleware。在這個 middleware 的 `share` 方法中，您可以回傳將提供給應用程式中所有 Inertia 頁面的共享資料。此共享資料可以作為一個便捷的位置來為使用者定義授權資訊：

```php
<?php

namespace App\Http\Middleware;

use App\Models\Post;
use Illuminate\Http\Request;
use Inertia\Middleware;

class HandleInertiaRequests extends Middleware
{
    // ...

    /**
     * Define the props that are shared by default.
     *
     * @return array<string, mixed>
     */
    public function share(Request $request)
    {
        return [
            ...parent::share($request),
            'auth' => [
                'user' => $request->user(),
                'permissions' => [
                    'post' => [
                        'create' => $request->user()->can('create', Post::class),
                    ],
                ],
            ],
        ];
    }
}
```