# 授權

- [簡介](#introduction)
- [閘道 (Gates)](#gates)
    - [撰寫閘道](#writing-gates)
    - [透過閘道授權動作](#authorizing-actions-via-gates)
    - [閘道回應](#gate-responses)
    - [攔截閘道檢查](#intercepting-gate-checks)
    - [行內授權](#inline-authorization)
- [建立策略 (Policies)](#creating-policies)
    - [產生策略](#generating-policies)
    - [註冊策略](#registering-policies)
- [撰寫策略](#writing-policies)
    - [策略方法](#policy-methods)
    - [策略回應](#policy-responses)
    - [無模型的方法](#methods-without-models)
    - [訪客使用者](#guest-users)
    - [策略篩選器](#policy-filters)
- [使用策略授權動作](#authorizing-actions-using-policies)
    - [透過 User 模型](#via-the-user-model)
    - [透過 Gate Facade](#via-the-gate-facade)
    - [透過 Middleware](#via-middleware)
    - [透過 Blade 模板](#via-blade-templates)
    - [提供額外上下文](#supplying-additional-context)
- [授權與 Inertia](#authorization-and-inertia)

<a name="introduction"></a>
## 簡介

除了提供內建的 [authentication](/docs/{{version}}/authentication) 服務外，Laravel 也提供了一種簡單的方式來授權使用者對給定資源的動作。例如，即使使用者已通過身份驗證，他們也可能無權更新或刪除應用程式管理的某些 Eloquent 模型或資料庫記錄。Laravel 的授權功能提供了一種簡單、有組織的方式來管理這些類型的授權檢查。

Laravel 提供了兩種主要的授權動作方式：[gates](#gates) 和 [policies](#creating-policies)。可以將 gates 和 policies 想像成路由和控制器。Gates 提供了一種簡單、基於閉包的授權方法，而 policies 則像控制器一樣，將邏輯圍繞著特定的模型或資源進行分組。在本文件中，我們將首先探討 gates，然後再研究 policies。

在建構應用程式時，您不需要在專門使用 gates 或專門使用 policies 之間做選擇。大多數應用程式很可能包含 gates 和 policies 的混合使用，這完全沒問題！Gates 最適用於與任何模型或資源無關的動作，例如查看管理員儀表板。相反地，當您希望授權特定模型或資源的動作時，則應使用 policies。

<a name="gates"></a>
## 閘道 (Gates)

<a name="writing-gates"></a>
### 撰寫閘道

> [!WARNING]
> Gates 是學習 Laravel 授權功能基礎的好方法；然而，在建構強大的 Laravel 應用程式時，您應該考慮使用 [policies](#creating-policies) 來組織您的授權規則。

Gates 只是決定使用者是否有權執行給定動作的閉包。通常，gates 是在 `App\\Providers\\AppServiceProvider` 類別的 `boot` 方法中，使用 `Gate` facade 來定義的。Gates 總是將使用者實例作為其第一個引數，並且可以選擇性地接收額外的引數，例如相關的 Eloquent 模型。

在這個範例中，我們將定義一個 gate 來判斷使用者是否可以更新給定的 `App\\Models\\Post` 模型。該 gate 將透過比較使用者的 `id` 與建立該文章的使用者的 `user_id` 來完成此操作：

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

像控制器一樣，gates 也可以使用類別回呼陣列來定義：

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
### 授權動作

要使用 gates 授權動作，您應該使用 `Gate` facade 提供的 `allows` 或 `denies` 方法。請注意，您不需要將當前已驗證的使用者傳遞給這些方法。Laravel 會自動將使用者傳遞給 gate 閉包。通常，在執行需要授權的動作之前，在應用程式的控制器中呼叫 gate 授權方法：

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

如果您想判斷除了當前已驗證使用者之外的另一個使用者是否有權執行某個動作，您可以使用 `Gate` facade 上的 `forUser` 方法：

```php
if (Gate::forUser($user)->allows('update-post', $post)) {
    // The user can update the post...
}

if (Gate::forUser($user)->denies('update-post', $post)) {
    // The user can't update the post...
}
```

您可以使用 `any` 或 `none` 方法一次授權多個動作：

```php
if (Gate::any(['update-post', 'delete-post'], $post)) {
    // The user can update or delete the post...
}

if (Gate::none(['update-post', 'delete-post'], $post)) {
    // The user can't update or delete the post...
}
```

<a name="authorizing-or-throwing-exceptions"></a>
#### 授權或拋出例外

如果您想嘗試授權一個動作，並且如果使用者不允許執行給定動作時自動拋出 `Illuminate\\Auth\\Access\\AuthorizationException`，您可以使用 `Gate` facade 的 `authorize` 方法。`AuthorizationException` 的實例會被 Laravel 自動轉換為 403 HTTP 回應：

```php
Gate::authorize('update-post', $post);

// The action is authorized...
```

<a name="gates-supplying-additional-context"></a>
#### 提供額外上下文

用於授權能力的 gate 方法 (`allows`, `denies`, `check`, `any`, `none`, `authorize`, `can`, `cannot`) 和授權 [Blade 指令](#via-blade-templates) (`@can`, `@cannot`, `@canany`) 可以接收一個陣列作為它們的第二個引數。這些陣列元素會作為參數傳遞給 gate 閉包，並可用於在做出授權決策時提供額外上下文：

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
### 閘道回應

到目前為止，我們只研究了返回簡單布林值的 gates。然而，有時您可能希望返回更詳細的回應，包括錯誤訊息。為此，您可以從 gate 返回一個 `Illuminate\\Auth\\Access\\Response`：

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

即使您從 gate 返回授權回應，`Gate::allows` 方法仍將返回一個簡單的布林值；但是，您可以使用 `Gate::inspect` 方法來獲取 gate 返回的完整授權回應：

```php
$response = Gate::inspect('edit-settings');

if ($response->allowed()) {
    // The action is authorized...
} else {
    echo $response->message();
}
```

當使用 `Gate::authorize` 方法時，如果動作未經授權，該方法會拋出 `AuthorizationException`，授權回應提供的錯誤訊息將會傳播到 HTTP 回應：

```php
Gate::authorize('edit-settings');

// The action is authorized...
```

<a name="customizing-gate-response-status"></a>
#### 自訂 HTTP 回應狀態

當透過 Gate 拒絕動作時，會返回 `403` HTTP 回應；然而，有時返回替代的 HTTP 狀態碼會很有用。您可以使用 `Illuminate\\Auth\\Access\\Response` 類別上的 `denyWithStatus` 靜態建構函式來自訂失敗授權檢查返回的 HTTP 狀態碼：

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

由於透過 `404` 回應隱藏資源是 Web 應用程式中非常常見的模式，因此提供了 `denyAsNotFound` 方法以方便使用：

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
### 攔截閘道檢查

有時，您可能希望授予特定使用者所有能力。您可以使用 `before` 方法來定義一個閉包，該閉包在所有其他授權檢查之前執行：

```php
use App\Models\User;
use Illuminate\Support\Facades\Gate;

Gate::before(function (User $user, string $ability) {
    if ($user->isAdministrator()) {
        return true;
    }
});
```

如果 `before` 閉包返回非 null 結果，則該結果將被視為授權檢查的結果。

您可以使用 `after` 方法來定義一個閉包，該閉包在所有其他授權檢查之後執行：

```php
use App\Models\User;

Gate::after(function (User $user, string $ability, bool|null $result, mixed $arguments) {
    if ($user->isAdministrator()) {
        return true;
    }
});
```

`after` 閉包返回的值不會覆蓋授權檢查的結果，除非 gate 或 policy 返回 `null`。

<a name="inline-authorization"></a>
### 行內授權

有時，您可能希望判斷當前已驗證的使用者是否有權執行給定動作，而無需撰寫與該動作對應的專用 gate。Laravel 允許您透過 `Gate::allowIf` 和 `Gate::denyIf` 方法執行這些「行內」授權檢查。行內授權不會執行任何已定義的 [「before」或「after」授權掛鉤](#intercepting-gate-checks)：

```php
use App\Models\User;
use Illuminate\Support\Facades\Gate;

Gate::allowIf(fn (User $user) => $user->isAdministrator());

Gate::denyIf(fn (User $user) => $user->banned());
```

如果動作未經授權或目前沒有使用者通過身份驗證，Laravel 將自動拋出 `Illuminate\\Auth\\Access\\AuthorizationException` 例外。`AuthorizationException` 的實例會被 Laravel 的例外處理器自動轉換為 403 HTTP 回應。

<a name="creating-policies"></a>
## 建立策略 (Policies)

<a name="generating-policies"></a>
### 產生策略

Policies 是將授權邏輯圍繞特定模型或資源組織起來的類別。例如，如果您的應用程式是一個部落格，您可能有一個 `App\\Models\\Post` 模型和一個對應的 `App\\Policies\\PostPolicy` 來授權使用者動作，例如建立或更新文章。

您可以使用 `make:policy` Artisan 命令產生策略。產生的策略將放置在 `app/Policies` 目錄中。如果您的應用程式中不存在此目錄，Laravel 將為您建立它：

```shell
php artisan make:policy PostPolicy
```

`make:policy` 命令將產生一個空的策略類別。如果您想產生一個包含與查看、建立、更新和刪除資源相關的範例策略方法的類別，您可以在執行命令時提供 `--model` 選項：

```shell
php artisan make:policy PostPolicy --model=Post
```

<a name="registering-policies"></a>
### 註冊策略

<a name="policy-discovery"></a>
#### 策略發現

預設情況下，只要模型和策略遵循標準的 Laravel 命名慣例，Laravel 就會自動發現策略。具體來說，策略必須位於包含模型的目錄或其上方的 `Policies` 目錄中。因此，例如，模型可以放置在 `app/Models` 目錄中，而策略可以放置在 `app/Policies` 目錄中。在這種情況下，Laravel 將在 `app/Models/Policies` 然後是 `app/Policies` 中檢查策略。此外，策略名稱必須與模型名稱匹配並帶有 `Policy` 後綴。因此，`User` 模型將對應於 `UserPolicy` 策略類別。

如果您想定義自己的策略發現邏輯，您可以使用 `Gate::guessPolicyNamesUsing` 方法註冊自訂策略發現回呼。通常，此方法應從應用程式的 `AppServiceProvider` 的 `boot` 方法中呼叫：

```php
use Illuminate\Support\Facades\Gate;

Gate::guessPolicyNamesUsing(function (string $modelClass) {
    // Return the name of the policy class for the given model...
});
```

<a name="manually-registering-policies"></a>
#### 手動註冊策略

使用 `Gate` facade，您可以在應用程式的 `AppServiceProvider` 的 `boot` 方法中手動註冊策略及其對應的模型：

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

或者，您可以在模型類別上放置 `UsePolicy` 屬性，以告知 Laravel 該模型對應的策略：

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
## 撰寫策略

<a name="policy-methods"></a>
### 策略方法

一旦策略類別註冊完成，您可以為其授權的每個動作新增方法。例如，讓我們在 `PostPolicy` 上定義一個 `update` 方法，該方法判斷給定的 `App\\Models\\User` 是否可以更新給定的 `App\\Models\\Post` 實例。

`update` 方法將接收 `User` 和 `Post` 實例作為其引數，並且應該返回 `true` 或 `false`，表示使用者是否有權更新給定的 `Post`。因此，在此範例中，我們將驗證使用者的 `id` 是否與文章的 `user_id` 匹配：

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

您可以根據需要繼續在策略上定義其他方法，以處理其授權的各種動作。例如，您可以定義 `view` 或 `delete` 方法來授權各種與 `Post` 相關的動作，但請記住，您可以自由地為您的策略方法命名。

如果您在透過 Artisan console 產生策略時使用了 `--model` 選項，它將已經包含 `viewAny`、`view`、`create`、`update`、`delete`、`restore` 和 `forceDelete` 動作的方法。

> [!NOTE]
> 所有策略都透過 Laravel [service container](/docs/{{version}}/container) 解析，允許您在策略的建構函式中型別提示任何所需的依賴項，以便它們自動注入。

<a name="policy-responses"></a>
### 策略回應

到目前為止，我們只研究了返回簡單布林值的策略方法。然而，有時您可能希望返回更詳細的回應，包括錯誤訊息。為此，您可以從策略方法返回一個 `Illuminate\\Auth\\Access\\Response` 實例：

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

當您從策略返回授權回應時，`Gate::allows` 方法仍將返回一個簡單的布林值；但是，您可以使用 `Gate::inspect` 方法來獲取 gate 返回的完整授權回應：

```php
use Illuminate\Support\Facades\Gate;

$response = Gate::inspect('update', $post);

if ($response->allowed()) {
    // The action is authorized...
} else {
    echo $response->message();
}
```

當使用 `Gate::authorize` 方法時，如果動作未經授權，該方法會拋出 `AuthorizationException`，授權回應提供的錯誤訊息將會傳播到 HTTP 回應：

```php
Gate::authorize('update', $post);

// The action is authorized...
```

<a name="customizing-policy-response-status"></a>
#### 自訂 HTTP 回應狀態

當透過策略方法拒絕動作時，會返回 `403` HTTP 回應；然而，有時返回替代的 HTTP 狀態碼會很有用。您可以使用 `Illuminate\\Auth\\Access\\Response` 類別上的 `denyWithStatus` 靜態建構函式來自訂失敗授權檢查返回的 HTTP 狀態碼：

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

由於透過 `404` 回應隱藏資源是 Web 應用程式中非常常見的模式，因此提供了 `denyAsNotFound` 方法以方便使用：

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
### 無模型的方法

某些策略方法只接收當前已驗證使用者的實例。這種情況在授權 `create` 動作時最常見。例如，如果您正在建立一個部落格，您可能希望判斷使用者是否有權建立任何文章。在這些情況下，您的策略方法應該只期望接收一個使用者實例：

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

預設情況下，如果傳入的 HTTP 請求不是由已驗證使用者發起的，所有 gates 和 policies 都會自動返回 `false`。但是，您可以透過宣告「可選」的型別提示或為使用者引數定義提供 `null` 預設值，來允許這些授權檢查通過您的 gates 和 policies：

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
### 策略篩選器

對於某些使用者，您可能希望授權給定策略中的所有動作。為此，請在策略上定義一個 `before` 方法。`before` 方法將在策略上的任何其他方法之前執行，讓您有機會在實際呼叫預期的策略方法之前授權動作。此功能最常用於授權應用程式管理員執行任何動作：

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

如果您想拒絕特定類型使用者的所有授權檢查，則可以從 `before` 方法返回 `false`。如果返回 `null`，則授權檢查將會傳遞給策略方法。

> [!WARNING]
> 如果策略類別不包含與正在檢查的能力名稱匹配的方法，則不會呼叫策略類別的 `before` 方法。

<a name="authorizing-actions-using-policies"></a>
## 使用策略授權動作

<a name="via-the-user-model"></a>
### 透過 User 模型

Laravel 應用程式中包含的 `App\\Models\\User` 模型包含兩個有用的授權動作方法：`can` 和 `cannot`。`can` 和 `cannot` 方法接收您希望授權的動作名稱和相關模型。例如，讓我們判斷使用者是否有權更新給定的 `App\\Models\\Post` 模型。通常，這將在控制器方法中完成：

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

如果為給定模型 [註冊了策略](#registering-policies)，`can` 方法將自動呼叫適當的策略並返回布林結果。如果沒有為模型註冊策略，`can` 方法將嘗試呼叫與給定動作名稱匹配的基於閉包的 Gate。

<a name="user-model-actions-that-dont-require-models"></a>
#### 不需要模型的動作

請記住，某些動作可能對應於不需要模型實例的策略方法，例如 `create`。在這些情況下，您可以將類別名稱傳遞給 `can` 方法。類別名稱將用於確定在授權動作時使用哪個策略：

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

除了提供給 `App\\Models\\User` 模型的有用方法外，您始終可以透過 `Gate` facade 的 `authorize` 方法授權動作。

與 `can` 方法一樣，此方法接受您希望授權的動作名稱和相關模型。如果動作未經授權，`authorize` 方法將拋出 `Illuminate\\Auth\\Access\\AuthorizationException` 例外，Laravel 例外處理器會自動將其轉換為狀態碼為 403 的 HTTP 回應：

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
#### 不需要模型的控制器動作

如前所述，某些策略方法（例如 `create`）不需要模型實例。在這些情況下，您應該將類別名稱傳遞給 `authorize` 方法。類別名稱將用於確定在授權動作時使用哪個策略：

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

Laravel 包含一個 Middleware，可以在傳入請求甚至到達您的路由或控制器之前授權動作。預設情況下，`Illuminate\\Auth\\Middleware\\Authorize` Middleware 可以使用 `can` [middleware alias](/docs/{{version}}/middleware#middleware-aliases) 附加到路由，該別名由 Laravel 自動註冊。讓我們探討一個使用 `can` Middleware 授權使用者可以更新文章的範例：

```php
use App\Models\Post;

Route::put('/post/{post}', function (Post $post) {
    // The current user may update the post...
})->middleware('can:update,post');
```

在此範例中，我們將兩個引數傳遞給 `can` Middleware。第一個是我們希望授權的動作名稱，第二個是我們希望傳遞給策略方法的路由參數。在這種情況下，由於我們使用 [implicit model binding](/docs/{{version}}/routing#implicit-binding)，`App\\Models\\Post` 模型將傳遞給策略方法。如果使用者未經授權執行給定動作，Middleware 將返回狀態碼為 403 的 HTTP 回應。

為方便起見，您也可以使用 `can` 方法將 `can` Middleware 附加到您的路由：

```php
use App\Models\Post;

Route::put('/post/{post}', function (Post $post) {
    // The current user may update the post...
})->can('update', 'post');
```

<a name="middleware-actions-that-dont-require-models"></a>
#### 不需要模型的 Middleware 動作

同樣，某些策略方法（例如 `create`）不需要模型實例。在這些情況下，您可以將類別名稱傳遞給 Middleware。類別名稱將用於確定在授權動作時使用哪個策略：

```php
Route::post('/post', function () {
    // The current user may create posts...
})->middleware('can:create,App\\Models\\Post');
```

在字串 Middleware 定義中指定整個類別名稱可能會變得繁瑣。因此，您可以選擇使用 `can` 方法將 `can` Middleware 附加到您的路由：

```php
use App\Models\Post;

Route::post('/post', function () {
    // The current user may create posts...
})->can('create', Post::class);
```

<a name="via-blade-templates"></a>
### 透過 Blade 模板

在撰寫 Blade 模板時，您可能希望僅在使用者有權執行給定動作時才顯示頁面的一部分。例如，您可能希望僅在使用者可以實際更新文章時才顯示部落格文章的更新表單。在這種情況下，您可以使用 `@can` 和 `@cannot` 指令：

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

這些指令是撰寫 `@if` 和 `@unless` 語句的便捷捷徑。上面的 `@can` 和 `@cannot` 語句等同於以下語句：

```blade
@if (Auth::user()->can('update', $post))
    <!-- The current user can update the post... -->
@endif

@unless (Auth::user()->can('update', $post))
    <!-- The current user cannot update the post... -->
@endunless
```

您還可以判斷使用者是否有權執行給定動作陣列中的任何動作。為此，請使用 `@canany` 指令：

```blade
@canany(['update', 'view', 'delete'], $post)
    <!-- The current user can update, view, or delete the post... -->
@elsecanany(['create'], \App\Models\Post::class)
    <!-- The current user can create a post... -->
@endcanany
```

<a name="blade-actions-that-dont-require-models"></a>
#### 不需要模型的 Blade 動作

像大多數其他授權方法一樣，如果動作不需要模型實例，您可以將類別名稱傳遞給 `@can` 和 `@cannot` 指令：

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

當使用策略授權動作時，您可以將一個陣列作為第二個引數傳遞給各種授權函數和輔助函數。陣列中的第一個元素將用於確定應呼叫哪個策略，而陣列的其餘元素將作為參數傳遞給策略方法，並可用於在做出授權決策時提供額外上下文。例如，考慮以下包含額外 `$category` 參數的 `PostPolicy` 方法定義：

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

當嘗試判斷已驗證使用者是否可以更新給定文章時，我們可以這樣呼叫此策略方法：

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

儘管授權必須始終在伺服器端處理，但為您的前端應用程式提供授權資料以正確呈現應用程式的 UI 通常會很方便。Laravel 沒有定義一種強制性的慣例來將授權資訊公開給由 Inertia 驅動的前端。

然而，如果您正在使用 Laravel 的其中一個基於 Inertia 的 [starter kits](/docs/{{version}}/starter-kits)，您的應用程式已經包含一個 `HandleInertiaRequests` Middleware。在此 Middleware 的 `share` 方法中，您可以返回將提供給應用程式中所有 Inertia 頁面的共享資料。此共享資料可以作為定義使用者授權資訊的便捷位置：

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

