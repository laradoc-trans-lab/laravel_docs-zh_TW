# 授權

- [簡介](#introduction)
- [Gates](#gates)
    - [撰寫 Gates](#writing-gates)
    - [對動作進行授權](#authorizing-actions-via-gates)
    - [Gate 回應](#gate-responses)
    - [攔截 Gate 檢查](#intercepting-gate-checks)
    - [行內授權](#inline-authorization)
- [建立 Policies](#creating-policies)
    - [產生 Policies](#generating-policies)
    - [註冊 Policies](#registering-policies)
- [撰寫 Policies](#writing-policies)
    - [Policy 方法](#policy-methods)
    - [Policy 回應](#policy-responses)
    - [不含模型的方法](#methods-without-models)
    - [訪客使用者](#guest-users)
    - [Policy 篩選器](#policy-filters)
- [使用 Policies 對動作進行授權](#authorizing-actions-using-policies)
    - [透過 User 模型](#via-the-user-model)
    - [透過 `Gate` Facade](#via-the-gate-facade)
    - [透過中介層](#via-middleware)
    - [透過 Blade 模板](#via-blade-templates)
    - [提供額外上下文](#supplying-additional-context)
- [授權與 Inertia](#authorization-and-inertia)

<a name="introduction"></a>
## 簡介

除了提供內建的 [認證](/docs/{{version}}/authentication) 服務外，Laravel 還提供了一種簡單的方式，可用於針對特定資源對使用者動作進行授權。例如，即便使用者已通過認證，他們可能仍未獲授權來更新或刪除由您的應用程式管理的特定 Eloquent 模型或資料庫紀錄。Laravel 的授權功能提供了一種簡單且有組織的方式來管理這類授權檢查。

Laravel 提供了兩種主要的動作授權方式：[gates](#gates) 與 [policies](#creating-policies)。您可以將 gates 與 policies 想像成路由與控制器。Gates 提供了一種簡單的、基於閉包 (closure) 的授權方法；而 policies 則像控制器一樣，將邏輯圍繞在特定的模型或資源周圍。在本文件中，我們將先探討 gates，接著再研究 policies。

在建構應用程式時，您不需要在僅使用 gates 或僅使用 policies 之間做選擇。大多數的應用程式很可能會混合使用 gates 與 policies，而這完全沒有問題！Gates 最適用於與任何模型或資源無關的動作，例如查看管理員儀表板。相比之下，當您希望針對特定模型或資源授權某個動作時，應使用 policies。

<a name="gates"></a>
## Gates


<a name="writing-gates"></a>
### 撰寫 Gates

> [!WARNING]
> Gates 是學習 Laravel 授權功能基礎的好方法；然而，在構建強健的 Laravel 應用程式時，您應該考慮使用 [policies](#creating-policies) 來組織您的授權規則。

Gates 簡單來說就是用來決定使用者是否有權限執行特定動作的閉包。通常，Gates 是使用 `Gate` facade 在 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中定義的。Gates 總是會接收一個使用者實例作為第一個引數，且可以選擇性地接收額外引數，例如相關的 Eloquent 模型。

在此範例中，我們將定義一個 gate 來決定使用者是否可以更新指定的 `App\Models\Post` 模型。此 gate 將透過比較使用者的 `id` 與建立該貼文的使用者的 `user_id` 來達成此目的：

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

與控制器一樣，gates 也可以使用類別回呼陣列來定義：

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
### 對動作進行授權

若要使用 gates 對動作進行授權，您應該使用 `Gate` facade 提供的 `allows` 或 `denies` 方法。請注意，您不需要將目前經過認證的使用者傳遞給這些方法。Laravel 會自動處理將使用者傳遞到 gate 閉包中。通常會在應用程式的控制器中，在執行需要授權的動作之前呼叫 gate 授權方法：

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

如果您想確定目前認證使用者以外的使用者是否有權限執行某個動作，可以使用 `Gate` facade 上的 `forUser` 方法：

```php
if (Gate::forUser($user)->allows('update-post', $post)) {
    // The user can update the post...
}

if (Gate::forUser($user)->denies('update-post', $post)) {
    // The user can't update the post...
}
```

您可以使用 `any` 或 `none` 方法一次對多個動作進行授權：

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

如果您想嘗試對動作進行授權，並且在使用者不被允許執行該動作時自動拋出 `Illuminate\Auth\Access\AuthorizationException`，您可以使用 `Gate` facade 的 `authorize` 方法。`AuthorizationException` 的實例會被 Laravel 自動轉換為 403 HTTP 回應：

```php
Gate::authorize('update-post', $post);

// The action is authorized...
```


<a name="gates-supplying-additional-context"></a>
#### 提供額外上下文

用於授權能力的 gate 方法（`allows`、`denies`、`check`、`any`、`none`、`authorize`、`can`、`cannot`）以及授權 [Blade 指令](#via-blade-templates)（`@can`、`@cannot`、`@canany`）可以接收一個陣列作為其第二個引數。這些陣列元素會作為參數傳遞給 gate 閉包，並可用於在做出授權決定時提供額外的上下文：

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

到目前為止，我們僅探討了回傳簡單布林值的 gates。然而，有時您可能希望回傳更詳細的回應，包括錯誤訊息。為此，您可以從 gate 回傳一個 `Illuminate\Auth\Access\Response`：

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

即使您從 gate 回傳授權回應，`Gate::allows` 方法仍然會回傳一個簡單的布林值；然而，您可以使用 `Gate::inspect` 方法來獲取 gate 回傳的完整授權回應：

```php
$response = Gate::inspect('edit-settings');

if ($response->allowed()) {
    // The action is authorized...
} else {
    echo $response->message();
}
```

當使用 `Gate::authorize` 方法（若動作未獲授權則拋出 `AuthorizationException`）時，授權回應提供的錯誤訊息將會傳遞到 HTTP 回應中：

```php
Gate::authorize('edit-settings');

// The action is authorized...
```


<a name="customizing-gate-response-status"></a>
#### 自訂 HTTP 回應狀態

當動作透過 Gate 被拒絕時，會回傳 `403` HTTP 回應；然而，有時回傳另一個 HTTP 狀態碼會很有用。您可以使用 `Illuminate\Auth\Access\Response` 類別上的 `denyWithStatus` 靜態建構子，來自訂授權檢查失敗時回傳的 HTTP 狀態碼：

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
### 攔截 Gate 檢查

有時，您可能希望將所有能力授予特定使用者。您可以使用 `before` 方法來定義一個在所有其他授權檢查之前執行的閉包：

```php
use App\Models\User;
use Illuminate\Support\Facades\Gate;

Gate::before(function (User $user, string $ability) {
    if ($user->isAdministrator()) {
        return true;
    }
});
```

如果 `before` 閉包回傳一個非 null 的結果，該結果將被視為授權檢查的結果。

您可以使用 `after` 方法來定義一個在所有其他授權檢查之後執行的閉包：

```php
use App\Models\User;

Gate::after(function (User $user, string $ability, bool|null $result, mixed $arguments) {
    if ($user->isAdministrator()) {
        return true;
    }
});
```

`after` 閉包回傳的值不會覆蓋授權檢查的結果，除非 gate 或 policy 回傳了 `null`。

<a name="inline-authorization"></a>
### 行內授權

有時候，您可能希望在不撰寫對應於該動作的專屬 gate 的情況下，判斷目前已認證的使用者是否有權限執行給定動作。Laravel 允許您透過 `Gate::allowIf` 與 `Gate::denyIf` 方法來執行這類「行內」授權檢查。行內授權不會執行任何已定義的 ["before" 或 "after" 授權鉤子](#intercepting-gate-checks)：

```php
use App\Models\User;
use Illuminate\Support\Facades\Gate;

Gate::allowIf(fn (User $user) => $user->isAdministrator());

Gate::denyIf(fn (User $user) => $user->banned());
```

如果該動作未獲授權，或者目前沒有任何已認證的使用者，Laravel 將會自動拋出 `Illuminate\Auth\Access\AuthorizationException` 異常。`AuthorizationException` 的實例會由 Laravel 的異常處理器自動轉換為 403 HTTP 回應。

<a name="creating-policies"></a>
## 建立 Policies


<a name="generating-policies"></a>
### 產生 Policies

Policies 是用來圍繞特定模型或資源組織授權邏輯的類別。例如，如果您的應用程式是一個部落格，您可能會有一個 `App\Models\Post` 模型以及對應的 `App\Policies\PostPolicy` 來對使用者的動作（例如建立或更新文章）進行授權。

您可以使用 `make:policy` Artisan 指令來產生 policy。產生的 policy 將會被放置在 `app/Policies` 目錄中。如果您的應用程式中不存在此目錄，Laravel 會為您建立：

```shell
php artisan make:policy PostPolicy
```

`make:policy` 指令會產生一個空的 policy 類別。如果您想要產生一個包含與查看、建立、更新和刪除資源相關的範例 policy 方法的類別，您可以在執行指令時提供 `--model` 選項：

```shell
php artisan make:policy PostPolicy --model=Post
```


<a name="registering-policies"></a>
### 註冊 Policies


<a name="policy-discovery"></a>
#### Policy 自動發現

預設情況下，只要模型和 policy 遵循標準的 Laravel 命名慣例，Laravel 就會自動發現 policies。具體來說，policies 必須位於包含模型的目錄之中或之上的一個 `Policies` 目錄中。例如，模型可以放置在 `app/Models` 目錄中，而 policies 則可以放置在 `app/Policies` 目錄中。在這種情況下，Laravel 會先檢查 `app/Models/Policies`，然後檢查 `app/Policies` 中的 policies。此外，policy 的名稱必須與模型名稱匹配且具有 `Policy` 後綴。因此，`User` 模型將對應到 `UserPolicy` policy 類別。

如果您想要定義自己的 policy 發現邏輯，可以使用 `Gate::guessPolicyNamesUsing` 方法註冊自定義的 policy 發現回呼函數。通常，此方法應該在應用程式 `AppServiceProvider` 的 `boot` 方法中被呼叫：

```php
use Illuminate\Support\Facades\Gate;

Gate::guessPolicyNamesUsing(function (string $modelClass) {
    // Return the name of the policy class for the given model...
});
```


<a name="manually-registering-policies"></a>
#### 手動註冊 Policies

您可以使用 `Gate` facade 在應用程式 `AppServiceProvider` 的 `boot` 方法中手動註冊 policies 及其對應的模型：

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

或者，您可以在模型類別上放置 `UsePolicy` 屬性，以告知 Laravel 該模型對應的 policy：

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

一旦 Policy 類別被註冊後，您就可以為其授權的每個動作添加方法。例如，讓我們在 `PostPolicy` 中定義一個 `update` 方法，用來決定特定的 `App\Models\User` 是否能更新特定的 `App\Models\Post` 實例。

`update` 方法將接收一個 `User` 和一個 `Post` 實例作為其引數，並應回傳 `true` 或 `false` 以表示該使用者是否獲權更新該 `Post`。因此，在此範例中，我們將驗證使用者的 `id` 是否與文章上的 `user_id` 相同：

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

您可以根據授權各種動作的需求，繼續在 Policy 中定義額外的方法。例如，您可以定義 `view` 或 `delete` 方法來授權各種與 `Post` 相關的動作，但請記得，您可以隨意為 Policy 方法命名。

如果您在透過 Artisan 主控台產生 Policy 時使用了 `--model` 選項，它將已經包含 `viewAny`、`view`、`create`、`update`、`delete`、`restore` 和 `forceDelete` 動作的方法。

> [!NOTE]
> 所有 Policy 都是透過 Laravel [服務容器](/docs/{{version}}/container) 解析的，這允許您在 Policy 的建構子中型別提示任何需要的依賴項，以便讓它們被自動注入。


<a name="policy-responses"></a>
### Policy 回應

到目前為止，我們只討論了回傳簡單布林值的 Policy 方法。然而，有時您可能希望回傳更詳細的回應，包括錯誤訊息。若要實現此功能，您可以從 Policy 方法中回傳一個 `Illuminate\Auth\Access\Response` 實例：

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

當您從 Policy 回傳授權回應時，`Gate::allows` 方法仍將回傳簡單的布林值；然而，您可以使用 `Gate::inspect` 方法來獲取 Gate 回傳的完整授權回應：

```php
use Illuminate\Support\Facades\Gate;

$response = Gate::inspect('update', $post);

if ($response->allowed()) {
    // The action is authorized...
} else {
    echo $response->message();
}
```

當使用 `Gate::authorize` 方法（若動作未獲授權則會拋出 `AuthorizationException`）時，授權回應中提供的錯誤訊息將會傳遞到 HTTP 回應中：

```php
Gate::authorize('update', $post);

// The action is authorized...
```


<a name="customizing-policy-response-status"></a>
#### 自訂 HTTP 回應狀態

當動作透過 Policy 方法被拒絕時，會回傳 `403` HTTP 回應；然而，有時回傳另一個 HTTP 狀態碼會更有用。您可以使用 `Illuminate\Auth\Access\Response` 類別上的 `denyWithStatus` 靜態建構子，來自訂授權檢查失敗時回傳的 HTTP 狀態碼：

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

由於透過 `404` 回應來隱藏資源是 Web 應用程式中非常常見的模式，因此為了方便起見，提供了 `denyAsNotFound` 方法：

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
### 不含模型的方法

某些 Policy 方法僅接收目前已認證使用者的實例。這種情況在授權 `create` 動作時最為常見。例如，如果您正在建立一個部落格，您可能希望決定使用者是否獲權建立任何文章。在這些情況下，您的 Policy 方法應該只預期接收一個使用者實例：

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

預設情況下，如果傳入的 HTTP 請求不是由已認證的使用者發起的，所有 Gates 和 Policies 都會自動回傳 `false`。然而，您可以透過宣告「可選 (optional)」的型別提示，或為使用者引數定義提供 `null` 預設值，來允許這些授權檢查進入您的 Gates 和 Policies：

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

對於某些使用者，您可能希望授權給定 Policy 中的所有動作。要實現此功能，請在 Policy 上定義一個 `before` 方法。`before` 方法將在 Policy 上的任何其他方法執行之前執行，讓您在實際呼叫預定的 Policy 方法之前有機會授權該動作。此功能最常用於授權應用程式管理員執行任何動作：

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

如果您希望拒絕特定類型使用者的所有授權檢查，則可以從 `before` 方法回傳 `false`。如果回傳 `null`，則授權檢查將會進入對應的 Policy 方法。

> [!WARNING]
> 如果 Policy 類別中不包含名稱與被檢查能力名稱相匹配的方法，則不會呼叫該類別的 `before` 方法。

<a name="authorizing-actions-using-policies"></a>
## 使用 Policies 對動作進行授權


<a name="via-the-user-model"></a>
### 透過 User 模型

隨 Laravel 應用程式提供的 `App\Models\User` 模型包含兩個用於授權動作的實用方法：`can` 與 `cannot`。`can` 與 `cannot` 方法接收您想要授權的動作名稱以及相關的模型。例如，讓我們來判斷使用者是否有權限更新特定的 `App\Models\Post` 模型。通常，這會在控制器方法中完成：

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

如果該模型已[註冊 Policy](#registering-policies)，`can` 方法會自動呼叫對應的 Policy 並回傳布林值結果。如果該模型沒有註冊 Policy，`can` 方法將嘗試呼叫與該動作名稱相匹配的閉包式 Gate。


<a name="user-model-actions-that-dont-require-models"></a>
#### 不需要模型的動作

請記得，某些動作可能對應到不需要模型實例的 Policy 方法，例如 `create`。在這種情況下，您可以將類別名稱傳遞給 `can` 方法。類別名稱將用於在授權動作時決定使用哪一個 Policy：

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

除了 `App\Models\User` 模型提供的方法外，您也可以隨時透過 `Gate` facade 的 `authorize` 方法來對動作進行授權。

與 `can` 方法類似，此方法接收您想要授權的動作名稱以及相關模型。如果動作未獲授權，`authorize` 方法將拋出 `Illuminate\Auth\Access\AuthorizationException` 異常，Laravel 的異常處理程序會自動將其轉換為狀態碼 403 的 HTTP 回應：

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
#### 不需要模型的動作

如前所述，某些 Policy 方法（如 `create`）不需要模型實例。在這種情況下，您應該將類別名稱傳遞給 `authorize` 方法。類別名稱將用於在授權動作時決定使用哪一個 Policy：

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
### 透過中介層

Laravel 包含一個中介層，可以在傳入的請求到達您的路由或控制器之前對動作進行授權。預設情況下，`Illuminate\Auth\Middleware\Authorize` 中介層可以使用 `can` [中介層別名](/docs/{{version}}/middleware#middleware-aliases) 附加到路由上，該別名由 Laravel 自動註冊。讓我們看一個使用 `can` 中介層來授權使用者可以更新文章的範例：

```php
use App\Models\Post;

Route::put('/post/{post}', function (Post $post) {
    // The current user may update the post...
})->middleware('can:update,post');
```

在這個範例中，我們向 `can` 中介層傳遞了兩個引數。第一個是我們想要授權的動作名稱，第二個是我們想要傳遞給 Policy 方法的路由參數。在這種情況下，由於我們使用了[隱含模型綁定](/docs/{{version}}/routing#implicit-binding)，`App\Models\Post` 模型將被傳遞到 Policy 方法。如果使用者未獲權限執行該動作，中介層將回傳狀態碼 403 的 HTTP 回應。

為了方便起見，您也可以使用 `can` 方法將 `can` 中介層附加到路由：

```php
use App\Models\Post;

Route::put('/post/{post}', function (Post $post) {
    // The current user may update the post...
})->can('update', 'post');
```

如果您使用的是[控制器中介層屬性](/docs/{{version}}/controllers#middleware-attributes)，可以透過 `Authorize` 屬性來套用 `can` 中介層：

```php
use Illuminate\Routing\Attributes\Controllers\Authorize;

#[Authorize('update', 'post')]
public function update(Post $post)
{
    // The current user may update the post...
}
```


<a name="middleware-actions-that-dont-require-models"></a>
#### 不需要模型的動作

同樣地，某些 Policy 方法（如 `create`）不需要模型實例。在這種情況下，您可以將類別名稱傳遞給中介層。類別名稱將用於在授權動作時決定使用哪一個 Policy：

```php
Route::post('/post', function () {
    // The current user may create posts...
})->middleware('can:create,App\Models\Post');
```

在字串中介層定義中指定完整的類別名稱可能會很繁瑣。因此，您可以選擇使用 `can` 方法將 `can` 中介層附加到路由：

```php
use App\Models\Post;

Route::post('/post', function () {
    // The current user may create posts...
})->can('create', Post::class);
```

<a name="via-blade-templates"></a>
### 透過 Blade 模板

在撰寫 Blade 模板時，您可能希望僅在使用者獲授權執行特定動作時，才顯示頁面的部分內容。例如，您可能希望僅在使用者確實可以更新文章時，才顯示部落格文章的更新表單。在這種情況下，您可以使用 `@can` 和 `@cannot` 指令：

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

這些指令是撰寫 `@if` 和 `@unless` 敘述的便捷快捷方式。上述的 `@can` 和 `@cannot` 敘述等同於以下敘述：

```blade
@if (Auth::user()->can('update', $post))
    <!-- The current user can update the post... -->
@endif

@unless (Auth::user()->can('update', $post))
    <!-- The current user cannot update the post... -->
@endunless
```

您也可以判斷使用者是否獲授權執行給定動作陣列中的任何一項動作。若要實現此功能，請使用 `@canany` 指令：

```blade
@canany(['update', 'view', 'delete'], $post)
    <!-- The current user can update, view, or delete the post... -->
@elsecanany(['create'], \App\Models\Post::class)
    <!-- The current user can create a post... -->
@endcanany
```


<a name="blade-actions-that-dont-require-models"></a>
#### 不含模型要求的動作

與大多數其他授權方法一樣，如果該動作不需要模型實例，您可以將類別名稱傳遞給 `@can` 和 `@cannot` 指令：

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

使用 Policies 對動作進行授權時，您可以將陣列作為各種授權函式與輔助函式的第二個引數。陣列中的第一個元素將用於決定應該調用哪個 Policy，而其餘的陣列元素則會作為參數傳遞給 Policy 方法，並可用於在做出授權決定時提供額外的上下文。例如，請參考以下包含額外 `$category` 參數的 `PostPolicy` 方法定義：

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

當嘗試判斷已認證的使用者是否可以更新給定文章時，我們可以像這樣調用此 Policy 方法：

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

雖然授權必須始終在伺服器端處理，但為了正確地渲染應用程式的 UI，將授權資料提供給前端應用程式通常會比較方便。Laravel 並未定義將授權資訊暴露給由 Inertia 驅動的前端之必要慣例。

然而，如果您使用的是 Laravel 基於 Inertia 的 [入門套件](/docs/{{version}}/starter-kits)，您的應用程式已經包含了一個 `HandleInertiaRequests` 中介層。在此中介層的 `share` 方法中，您可以回傳將提供給應用程式中所有 Inertia 頁面的共用資料。這些共用資料可以作為定義使用者授權資訊的便利位置：

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