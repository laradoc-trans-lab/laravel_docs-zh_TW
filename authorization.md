# 授權

- [介紹](#introduction)
- [Gates](#gates)
    - [撰寫 Gates](#writing-gates)
    - [授權動作](#authorizing-actions-via-gates)
    - [Gate 回應](#gate-responses)
    - [攔截 Gate 檢查](#intercepting-gate-checks)
    - [行內授權](#inline-authorization)
- [建立 Policies](#creating-policies)
    - [產生 Policies](#generating-policies)
    - [註冊 Policies](#registering-policies)
- [撰寫 Policies](#writing-policies)
    - [Policy 方法](#policy-methods)
    - [Policy 回應](#policy-responses)
    - [不帶 Model 的方法](#methods-without-models)
    - [訪客使用者](#guest-users)
    - [Policy 篩選器](#policy-filters)
- [使用 Policies 授權動作](#authorizing-actions-using-policies)
    - [透過 User Model](#via-the-user-model)
    - [透過 `Gate` Facade](#via-the-gate-facade)
    - [透過中介層](#via-middleware)
    - [透過 Blade 模板](#via-blade-templates)
    - [提供額外上下文](#supplying-additional-context)
- [授權與 Inertia](#authorization-and-inertia)

<a name="introduction"></a>
## 介紹

除了提供內建的 [認證](/docs/{{version}}/authentication) 服務之外，Laravel 也提供了一個簡單的方式來針對特定資源授權使用者動作。例如，即使使用者已經通過認證，他們也可能沒有被授權更新或刪除應用程式中由某些 Eloquent 模型或資料庫記錄。Laravel 的授權功能提供了一種簡單、有組織的方式來管理這些類型的授權檢查。

Laravel 提供了兩種主要的動作授權方式：[gates](#gates) 和 [policies](#creating-policies)。可以將 gates 和 policies 想像成路由 (routes) 和控制器 (controllers)。gates 提供了一種簡單、基於閉包 (closure-based) 的授權方法，而 policies 則像控制器一樣，將邏輯圍繞著特定的模型 (model) 或資源 (resource) 進行分組。在本文件中，我們將首先探討 gates，然後再研究 policies。

在建構應用程式時，你不需要在完全使用 gates 或完全使用 policies 之間做出選擇。大多數應用程式很可能會包含 gates 和 policies 的混合使用，這也是完全沒問題的！gates 最適用於與任何模型 (model) 或資源 (resource) 無關的動作，例如查看管理員儀表板。相對地，當你希望授權針對特定模型 (model) 或資源 (resource) 的動作時，就應該使用 policies。

<a name="gates"></a>
## Gates

<a name="writing-gates"></a>
### 撰寫 Gates

> [!WARNING]  
> Gates 是學習 Laravel 授權功能基礎的好方法；然而，當你建構強大的 Laravel 應用程式時，你應該考慮使用 [policies](#creating-policies) 來組織你的授權規則。

Gates 只是判斷使用者是否有權執行指定動作的閉包。通常，Gates 會在 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中，使用 `Gate` facade 來定義。Gates 總是接收一個使用者實例作為第一個參數，並可選擇性地接收額外參數，例如相關的 Eloquent model。

在此範例中，我們將定義一個 gate 來判斷使用者是否可以更新指定的 `App\Models\Post` model。這個 gate 會透過比較使用者的 `id` 與建立該文章的使用者的 `user_id` 來完成這項任務：

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

如同控制器，gates 也可以使用類別回呼陣列來定義：

    use App\Policies\PostPolicy;
    use Illuminate\Support\Facades\Gate;

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        Gate::define('update-post', [PostPolicy::class, 'update']);
    }

<a name="authorizing-actions-via-gates"></a>
### 授權動作

要使用 gates 授權動作，你應該使用 `Gate` facade 提供的 `allows` 或 `denies` 方法。請注意，你不需要將目前已驗證的使用者傳遞給這些方法。Laravel 會自動將使用者傳遞到 gate 閉包中。通常，在執行需要授權的動作之前，會在應用程式的控制器中呼叫 gate 授權方法：

    <?php

    namespace App\Http\Controllers;

    use App\Http\Controllers\Controller;
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

如果你想判斷除了當前已驗證使用者之外的另一位使用者是否有權執行某個動作，你可以使用 `Gate` facade 上的 `forUser` 方法：

    if (Gate::forUser($user)->allows('update-post', $post)) {
        // The user can update the post...
    }

    if (Gate::forUser($user)->denies('update-post', $post)) {
        // The user can't update the post...
    }

你可以使用 `any` 或 `none` 方法同時授權多個動作：

    if (Gate::any(['update-post', 'delete-post'], $post)) {
        // The user can update or delete the post...
    }

    if (Gate::none(['update-post', 'delete-post'], $post)) {
        // The user can't update or delete the post...
    }

<a name="authorizing-or-throwing-exceptions"></a>
#### 授權或拋出例外

如果你想嘗試授權一個動作，並且在使用者不被允許執行該動作時自動拋出 `Illuminate\Auth\Access\AuthorizationException`，你可以使用 `Gate` facade 的 `authorize` 方法。`AuthorizationException` 的實例會被 Laravel 自動轉換為 403 HTTP 回應：

    Gate::authorize('update-post', $post);

    // The action is authorized...

<a name="gates-supplying-additional-context"></a>
#### 提供額外上下文

用於授權能力的 gate 方法 (`allows`、`denies`、`check`、`any`、`none`、`authorize`、`can`、`cannot`) 以及授權 [Blade directives](#via-blade-templates) (`@can`、`@cannot`、`@canany`) 可以接收一個陣列作為它們的第二個參數。這些陣列元素會作為參數傳遞給 gate 閉包，並可用於在進行授權決策時提供額外的上下文：

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

<a name="gate-responses"></a>
### Gate 回應

到目前為止，我們只探討了回傳簡單布林值的 gates。然而，有時你可能希望回傳更詳細的回應，包括錯誤訊息。為此，你可以從你的 gate 中回傳一個 `Illuminate\Auth\Access\Response` 實例：

    use App\Models\User;
    use Illuminate\Auth\Access\Response;
    use Illuminate\Support\Facades\Gate;

    Gate::define('edit-settings', function (User $user) {
        return $user->isAdmin
            ? Response::allow()
            : Response::deny('You must be an administrator.');
    });

即使你從 gate 回傳授權回應，`Gate::allows` 方法仍會回傳一個簡單的布林值；然而，你可以使用 `Gate::inspect` 方法來取得 gate 回傳的完整授權回應：

    $response = Gate::inspect('edit-settings');

    if ($response->allowed()) {
        // The action is authorized...
    } else {
        echo $response->message();
    }

當使用 `Gate::authorize` 方法時，如果動作未被授權，它會拋出 `AuthorizationException`。此時，授權回應提供的錯誤訊息會傳播到 HTTP 回應中：

    Gate::authorize('edit-settings');

    // The action is authorized...

<a name="customizing-gate-response-status"></a>
#### 自訂 HTTP 回應狀態

當動作透過 Gate 被拒絕時，會回傳 `403` HTTP 回應；然而，有時回傳替代的 HTTP 狀態碼會很有用。你可以使用 `Illuminate\Auth\Access\Response` 類別上的 `denyWithStatus` 靜態建構子，來自訂失敗授權檢查所回傳的 HTTP 狀態碼：

    use App\Models\User;
    use Illuminate\Auth\Access\Response;
    use Illuminate\Support\Facades\Gate;

    Gate::define('edit-settings', function (User $user) {
        return $user->isAdmin
            ? Response::allow()
            : Response::denyWithStatus(404);
    });

由於透過 `404` 回應隱藏資源是網頁應用程式的常見模式，因此提供了 `denyAsNotFound` 方法以方便使用：

    use App\Models\User;
    use Illuminate\Auth\Access\Response;
    use Illuminate\Support\Facades\Gate;

    Gate::define('edit-settings', function (User $user) {
        return $user->isAdmin
            ? Response::allow()
            : Response::denyAsNotFound();
    });

<a name="intercepting-gate-checks"></a>
### 攔截 Gate 檢查

有時，您可能希望授予特定使用者所有能力。您可以使用 `before` 方法來定義一個閉包，該閉包會在所有其他授權檢查之前執行：

    use App\Models\User;
    use Illuminate\Support\Facades\Gate;

    Gate::before(function (User $user, string $ability) {
        if ($user->isAdministrator()) {
            return true;
        }
    });

如果 `before` 閉包回傳非 null 的結果，該結果將被視為授權檢查的結果。

您可以使用 `after` 方法來定義一個閉包，該閉包會在所有其他授權檢查之後執行：

    use App\Models\User;

    Gate::after(function (User $user, string $ability, bool|null $result, mixed $arguments) {
        if ($user->isAdministrator()) {
            return true;
        }
    });

`after` 閉包回傳的值不會覆寫授權檢查的結果，除非 Gate 或 Policy 回傳 `null`。

<a name="inline-authorization"></a>
### 行內授權

有時，您可能希望判斷當前已驗證使用者是否有權執行某個動作，而無需撰寫一個對應此動作的專用 Gate。Laravel 允許您透過 `Gate::allowIf` 和 `Gate::denyIf` 方法來執行這些「行內」授權檢查。行內授權不會執行任何已定義的「[before 或 after 授權掛鉤](#intercepting-gate-checks)」：

```php
use App\Models\User;
use Illuminate\Support\Facades\Gate;

Gate::allowIf(fn (User $user) => $user->isAdministrator());

Gate::denyIf(fn (User $user) => $user->banned());
```

如果該動作未被授權，或者目前沒有使用者被驗證，Laravel 將自動拋出 `Illuminate\Auth\Access\AuthorizationException` 例外。`AuthorizationException` 的實例會被 Laravel 的例外處理器自動轉換為 403 HTTP 回應。

<a name="creating-policies"></a>
## 建立 Policies

<a name="generating-policies"></a>
### 產生 Policies

Policies 是圍繞特定 Model 或資源組織授權邏輯的類別。例如，如果你的應用程式是一個部落格，你可能會有一個 `App\Models\Post` Model 以及一個對應的 `App\Policies\PostPolicy`，用來授權使用者建立或更新貼文等動作。

你可以使用 `make:policy` Artisan 命令來產生一個 Policy。產生的 Policy 將會被放置在 `app/Policies` 目錄中。如果此目錄在你的應用程式中不存在，Laravel 會為你建立它：

```shell
php artisan make:policy PostPolicy
```

`make:policy` 命令將會產生一個空的 Policy 類別。如果你想產生一個包含與檢視、建立、更新和刪除資源相關的範例 Policy 方法的類別，你可以在執行命令時提供 `--model` 選項：

```shell
php artisan make:policy PostPolicy --model=Post
```

<a name="registering-policies"></a>
### 註冊 Policies

<a name="policy-discovery"></a>
#### Policy 探索

預設情況下，只要 Model 和 Policy 遵循標準的 Laravel 命名慣例，Laravel 就會自動探索 Policies。具體來說，Policies 必須位於包含你 Models 的目錄中或其上方的 `Policies` 目錄內。因此，舉例來說，Models 可能會被放置在 `app/Models` 目錄中，而 Policies 可能會被放置在 `app/Policies` 目錄中。在這種情況下，Laravel 會先檢查 `app/Models/Policies`，然後是 `app/Policies`。此外，Policy 名稱必須與 Model 名稱相符，並帶有 `Policy` 後綴。所以，一個 `User` Model 會對應到一個 `UserPolicy` Policy 類別。

如果你想定義自己的 Policy 探索邏輯，你可以使用 `Gate::guessPolicyNamesUsing` 方法註冊一個自訂的 Policy 探索回呼。通常，這個方法應該從應用程式的 `AppServiceProvider` 的 `boot` 方法中呼叫：

    use Illuminate\Support\Facades\Gate;

    Gate::guessPolicyNamesUsing(function (string $modelClass) {
        // Return the name of the policy class for the given model...
    });

<a name="manually-registering-policies"></a>
#### 手動註冊 Policies

使用 `Gate` Facade，你可以在應用程式的 `AppServiceProvider` 的 `boot` 方法中手動註冊 Policies 及其對應的 Models：

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

<a name="writing-policies"></a>
## 撰寫 Policies


<a name="policy-methods"></a>
### Policy 方法

一旦 Policy 類別註冊完成，您便可以為其授權的每個動作新增方法。例如，我們來在 `PostPolicy` 上定義一個 `update` 方法，用來判斷給定的 `App\Models\User` 是否能更新給定的 `App\Models\Post` 實例。

`update` 方法會接收一個 `User` 實例和一個 `Post` 實例作為引數，並應回傳 `true` 或 `false`，指示使用者是否被授權更新給定的 `Post`。因此，在此範例中，我們將驗證使用者的 `id` 是否與文章的 `user_id` 相符：

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

您可以根據需要繼續在 Policy 上定義額外的方法，以處理其授權的各種動作。例如，您可以定義 `view` 或 `delete` 方法來授權各種與 `Post` 相關的動作，但請記住，您可以隨意命名您的 Policy 方法。

如果您透過 Artisan 主控台產生 Policy 時使用了 `--model` 選項，它將已包含 `viewAny`、`view`、`create`、`update`、`delete`、`restore` 和 `forceDelete` 動作的方法。

> [!NOTE]  
> 所有 Policy 都透過 Laravel [服務容器](/docs/{{version}}/container)解析，這允許您在 Policy 的建構子中型別提示任何所需的依賴項，讓它們自動注入。


<a name="policy-responses"></a>
### Policy 回應

到目前為止，我們只檢視了回傳簡單布林值的 Policy 方法。然而，有時您可能希望回傳更詳細的回應，包括錯誤訊息。為此，您可以從您的 Policy 方法中回傳 `Illuminate\Auth\Access\Response` 實例：

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

當從您的 Policy 回傳授權回應時，`Gate::allows` 方法仍將回傳簡單的布林值；然而，您可以使用 `Gate::inspect` 方法來取得 Policy 回傳的完整授權回應：

    use Illuminate\Support\Facades\Gate;

    $response = Gate::inspect('update', $post);

    if ($response->allowed()) {
        // The action is authorized...
    } else {
        echo $response->message();
    }

當使用 `Gate::authorize` 方法時，如果動作未經授權，該方法會擲回 `AuthorizationException`，且授權回應提供的錯誤訊息將傳播到 HTTP 回應：

    Gate::authorize('update', $post);

    // The action is authorized...


<a name="customizing-policy-response-status"></a>
#### 自訂 HTTP 回應狀態

當動作透過 Policy 方法被拒絕時，會回傳 `403` HTTP 回應；然而，有時回傳替代的 HTTP 狀態碼會很有用。您可以使用 `Illuminate\Auth\Access\Response` 類別上的 `denyWithStatus` 靜態建構子來自訂因授權檢查失敗而回傳的 HTTP 狀態碼：

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

因為透過 `404` 回應隱藏資源是網頁應用程式中常見的模式，所以提供了 `denyAsNotFound` 方法以方便使用：

    use App\Models\Post;
    use App::Models\User;
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


<a name="methods-without-models"></a>
### 不帶 Model 的方法

某些 Policy 方法僅接收目前已驗證的使用者實例。這種情況在授權 `create` 動作時最為常見。例如，如果您正在建立一個部落格，您可能希望判斷使用者是否被授權建立任何文章。在這些情況下，您的 Policy 方法應該只期望接收一個使用者實例：

    /**
     * Determine if the given user can create posts.
     */
    public function create(User $user): bool
    {
        return $user->role == 'writer';
    }


<a name="guest-users"></a>
### 訪客使用者

依預設，如果傳入的 HTTP 請求不是由已驗證的使用者發起的，所有 Gates 和 Policies 會自動回傳 `false`。但是，您可以透過宣告「可選」型別提示或為使用者引數定義提供 `null` 預設值，來讓這些授權檢查通過到您的 Gates 和 Policies：

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


<a name="policy-filters"></a>
### Policy 篩選器

對於某些使用者，您可能希望授權某個 Policy 中的所有動作。為此，請在 Policy 上定義一個 `before` 方法。`before` 方法將在 Policy 中的所有其他方法之前執行，讓您有機會在實際呼叫預期 Policy 方法之前授權動作。此功能最常用於授權應用程式管理員執行任何動作：

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

如果您想拒絕某類使用者的所有授權檢查，您可以從 `before` 方法回傳 `false`。如果回傳 `null`，則授權檢查將會繼續執行 Policy 方法。

> [!WARNING]  
> 如果 Policy 類別不包含與所檢查能力名稱相符的方法，則不會呼叫 Policy 類別的 `before` 方法。

<a name="authorizing-actions-using-policies"></a>
## 使用 Policies 授權動作

<a name="via-the-user-model"></a>
### 透過 User Model

Laravel 應用程式隨附的 `App\Models\User` Model 包含兩個有用的方法來授權動作：`can` 和 `cannot`。`can` 和 `cannot` 方法會接收您希望授權的動作名稱以及相關的 Model。例如，讓我們判斷使用者是否被授權更新指定的 `App\Models\Post` Model。通常，這會在控制器方法中完成：

    <?php

    namespace App\Http\Controllers;

    use App\Http\Controllers\Controller;
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

如果為指定的 Model [註冊了 Policy](#registering-policies)，`can` 方法將自動呼叫適當的 Policy 並返回布林值結果。如果沒有為 Model 註冊 Policy，`can` 方法將嘗試呼叫與指定動作名稱相符的閉包形式的 Gate。

<a name="user-model-actions-that-dont-require-models"></a>
#### 不帶 Model 的動作

請記住，某些動作可能對應到 `create` 等不需要 Model 實例的 Policy 方法。在這些情況下，您可以將類別名稱傳遞給 `can` 方法。該類別名稱將用於判斷在授權動作時應使用哪個 Policy：

    <?php

    namespace App\Http\Controllers;

    use App\Http\Controllers\Controller;
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

<a name="via-the-gate-facade"></a>
### 透過 `Gate` Facade

除了 `App\Models\User` Model 提供有用的方法外，您還可以透過 `Gate` Facade 的 `authorize` 方法來授權動作。

如同 `can` 方法，此方法接受您希望授權的動作名稱以及相關的 Model。如果動作未經授權，`authorize` 方法將會拋出 `Illuminate\Auth\Access\AuthorizationException` 例外，Laravel 的例外處理器會自動將其轉換為狀態碼為 403 的 HTTP 回應：

    <?php

    namespace App\Http\Controllers;

    use App\Http\Controllers\Controller;
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

<a name="controller-actions-that-dont-require-models"></a>
#### 不帶 Model 的動作

如前所述，某些 Policy 方法 (例如 `create`) 不需要 Model 實例。在這些情況下，您應該將類別名稱傳遞給 `authorize` 方法。該類別名稱將用於判斷在授權動作時應使用哪個 Policy：

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

<a name="via-middleware"></a>
### 透過中介層

Laravel 包含一個中介層，可以在傳入請求到達您的路由或控制器之前授權動作。預設情況下，`Illuminate\Auth\Middleware\Authorize` 中介層可以使用 `can` [中介層別名](/docs/{{version}}/middleware#middleware-aliases)附加到路由，該別名由 Laravel 自動註冊。讓我們探索一個使用 `can` 中介層來授權使用者可以更新文章的範例：

    use App\Models\Post;

    Route::put('/post/{post}', function (Post $post) {
        // The current user may update the post...
    })->middleware('can:update,post');

在此範例中，我們向 `can` 中介層傳遞了兩個引數。第一個是我們希望授權的動作名稱，第二個是我們希望傳遞給 Policy 方法的路由參數。在本例中，由於我們正在使用[隱式 Model 綁定](/docs/{{version}}/routing#implicit-binding)，一個 `App\Models\Post` Model 將被傳遞給 Policy 方法。如果使用者未經授權執行指定動作，中介層將返回狀態碼為 403 的 HTTP 回應。

為了方便，您也可以使用 `can` 方法將 `can` 中介層附加到您的路由：

    use App\Models\Post;

    Route::put('/post/{post}', function (Post $post) {
        // The current user may update the post...
    })->can('update', 'post');

<a name="middleware-actions-that-dont-require-models"></a>
#### 不帶 Model 的動作

同樣，某些 Policy 方法 (例如 `create`) 不需要 Model 實例。在這些情況下，您可以將類別名稱傳遞給中介層。該類別名稱將用於判斷在授權動作時應使用哪個 Policy：

    Route::post('/post', function () {
        // The current user may create posts...
    })->middleware('can:create,App\Models\Post');

在字串中介層定義中指定完整的類別名稱可能會變得繁瑣。因此，您可以選擇使用 `can` 方法將 `can` 中介層附加到您的路由：

    use App\Models\Post;

    Route::post('/post', function () {
        // The current user may create posts...
    })->can('create', Post::class);

<a name="via-blade-templates"></a>
### 透過 Blade 模板

撰寫 Blade 模板時，您可能希望僅在使用者被授權執行某個動作時才顯示頁面的一部分。例如，您可能希望僅在使用者確實能夠更新該文章時，才顯示部落格文章的更新表單。在這種情況下，您可以使用 `@can` 與 `@cannot` 指令：

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

這些指令是方便的捷徑，用於撰寫 `@if` 與 `@unless` 語句。上面顯示的 `@can` 與 `@cannot` 語句等同於以下的語句：

```blade
@if (Auth::user()->can('update', $post))
    <!-- The current user can update the post... -->
@endif

@unless (Auth::user()->can('update', $post))
    <!-- The current user cannot update the post... -->
@endunless
```

您也可以判斷使用者是否被授權執行給定動作陣列中的任何動作。為此，請使用 `@canany` 指令：

```blade
@canany(['update', 'view', 'delete'], $post)
    <!-- The current user can update, view, or delete the post... -->
@elsecanany(['create'], \App\Models\Post::class)
    <!-- The current user can create a post... -->
@endcanany
```

<a name="blade-actions-that-dont-require-models"></a>
#### 不帶 Model 的動作

與大多數其他授權方法一樣，如果該動作不需要 Model 實例，您可以將類別名稱傳遞給 `@can` 與 `@cannot` 指令：

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

使用 Policies 授權動作時，您可以傳遞一個陣列作為第二個參數給各種授權功能與輔助函式。陣列中的第一個元素將用於判斷應呼叫哪個 Policy，而陣列中的其餘元素將作為參數傳遞給 Policy 方法，可用於在做出授權判斷時提供額外上下文。例如，考慮以下 `PostPolicy` 方法定義，其中包含一個額外的 `$category` 參數：

    /**
     * Determine if the given post can be updated by the user.
     */
    public function update(User $user, Post $post, int $category): bool
    {
        return $user->id === $post->user_id &&
               $user->canUpdateCategory($category);
    }

當嘗試判斷已驗證使用者是否可以更新給定文章時，我們可以這樣呼叫此 Policy 方法：

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

<a name="authorization-and-inertia"></a>
## 授權與 Inertia

儘管授權必須始終在伺服器上處理，但為前端應用程式提供授權資料以正確渲染應用程式的 UI 通常會很方便。Laravel 並沒有為將授權資訊暴露給由 Inertia 驅動的前端定義一個必要的慣例。

然而，如果你正在使用 Laravel 基於 Inertia 的 [入門套件](/docs/{{version}}/starter-kits) 之一，你的應用程式中已包含一個 `HandleInertiaRequests` 中介層。在這個中介層的 `share` 方法中，你可以回傳共用資料，這些資料將會提供給應用程式中的所有 Inertia 頁面。這些共用資料可以作為一個便捷的位置來定義使用者的授權資訊：

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