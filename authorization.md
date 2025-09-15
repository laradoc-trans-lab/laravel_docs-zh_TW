# 授權

- [簡介](#introduction)
- [Gates](#gates)
    - [撰寫 Gates](#writing-gates)
    - [透過 Gates 授權動作](#authorizing-actions-via-gates)
    - [Gate 回應](#gate-responses)
    - [攔截 Gate 檢查](#intercepting-gate-checks)
    - [行內授權](#inline-authorization)
- [建立 Policies](#creating-policies)
    - [產生 Policies](#generating-policies)
    - [註冊 Policies](#registering-policies)
- [撰寫 Policies](#writing-policies)
    - [Policy 方法](#policy-methods)
    - [Policy 回應](#policy-responses)
    - [不含 Model 的方法](#methods-without-models)
    - [訪客使用者](#guest-users)
    - [Policy 篩選器](#policy-filters)
- [使用 Policies 授權動作](#authorizing-actions-using-policies)
    - [透過 User Model](#via-the-user-model)
    - [透過 Gate Facade](#via-the-gate-facade)
    - [透過 Middleware](#via-middleware)
    - [透過 Blade 模板](#via-blade-templates)
    - [提供額外 Context](#supplying-additional-context)
- [授權與 Inertia](#authorization-and-inertia)

<a name="introduction"></a>
## 簡介

除了提供內建的 [authentication](/docs/{{version}}/authentication) 服務外，Laravel 也提供了一種簡單的方式來授權使用者對特定資源的動作。舉例來說，即使使用者已通過身份驗證，他們可能仍未被授權更新或刪除應用程式所管理的某些 Eloquent model 或資料庫記錄。Laravel 的授權功能提供了一種簡單、有組織的方式來管理這些類型的授權檢查。

Laravel 提供兩種主要的授權動作方式：[gates](#gates) 和 [policies](#creating-policies)。可以將 gates 和 policies 想像成 routes 和 controllers。Gates 提供了一種簡單、基於閉包的授權方法，而 policies 則像 controllers 一樣，將邏輯圍繞著特定的 model 或資源進行分組。在本文件中，我們將首先探討 gates，然後再研究 policies。

在建構應用程式時，您不需要在僅使用 gates 或僅使用 policies 之間做出選擇。大多數應用程式很可能會包含 gates 和 policies 的混合使用，這完全沒問題！Gates 最適用於與任何 model 或資源無關的動作，例如檢視管理員儀表板。相反地，當您希望授權特定 model 或資源的動作時，則應使用 policies。

<a name="gates"></a>
## Gates

<a name="writing-gates"></a>
### 撰寫 Gates

> [!WARNING]
> Gates 是學習 Laravel 授權功能基礎的好方法；然而，在建構強健的 Laravel 應用程式時，您應該考慮使用 [policies](#creating-policies) 來組織您的授權規則。

Gates 只是閉包，用於判斷使用者是否被授權執行特定動作。通常，gates 是在 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中，使用 `Gate` facade 來定義的。Gates 總是將使用者實例作為第一個參數接收，並且可以選擇性地接收額外參數，例如相關的 Eloquent model。

在這個範例中，我們將定義一個 gate 來判斷使用者是否可以更新給定的 `App\Models\Post` model。這個 gate 將透過比較使用者的 `id` 與建立該文章的使用者的 `user_id` 來實現：

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

如同 controllers，gates 也可以使用類別回呼陣列來定義：

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
### 透過 Gates 授權動作

要使用 gates 授權動作，您應該使用 `Gate` facade 提供的 `allows` 或 `denies` 方法。請注意，您不需要將目前已驗證的使用者傳遞給這些方法。Laravel 會自動處理將使用者傳遞給 gate 閉包。通常，在執行需要授權的動作之前，會在應用程式的 controllers 中呼叫 gate 授權方法：

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

如果您想判斷除了目前已驗證使用者之外的另一個使用者是否被授權執行某個動作，您可以使用 `Gate` facade 上的 `forUser` 方法：

    if (Gate::forUser($user)->allows('update-post', $post)) {
        // The user can update the post...
    }

    if (Gate::forUser($user)->denies('update-post', $post)) {
        // The user can't update the post...
    }

您可以使用 `any` 或 `none` 方法一次授權多個動作：

    if (Gate::any(['update-post', 'delete-post'], $post)) {
        // The user can update or delete the post...
    }

    if (Gate::none(['update-post', 'delete-post'], $post)) {
        // The user can't update or delete the post...
    }

<a name="authorizing-or-throwing-exceptions"></a>
#### 授權或拋出例外

如果您想嘗試授權一個動作，並且在使用者不被允許執行該動作時自動拋出 `Illuminate\Auth\Access\AuthorizationException`，您可以使用 `Gate` facade 的 `authorize` 方法。`AuthorizationException` 的實例會被 Laravel 自動轉換為 403 HTTP 回應：

    Gate::authorize('update-post', $post);

    // The action is authorized...

<a name="gates-supplying-additional-context"></a>
#### 提供額外 Context

用於授權能力的 gate 方法 (`allows`, `denies`, `check`, `any`, `none`, `authorize`, `can`, `cannot`) 和授權 [Blade directives](#via-blade-templates) (` @can`, ` @cannot`, ` @canany`) 可以接收一個陣列作為它們的第二個參數。這些陣列元素會作為參數傳遞給 gate 閉包，並可用於在做出授權決策時提供額外的 context：

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

到目前為止，我們只研究了返回簡單布林值的 gates。然而，有時您可能希望返回更詳細的回應，包括錯誤訊息。為此，您可以從 gate 返回一個 `Illuminate\Auth\Access\Response`：

    use App\Models\User;
    use Illuminate\Auth\Access\Response;
    use Illuminate\Support\Facades\Gate;

    Gate::define('edit-settings', function (User $user) {
        return $user->isAdmin
            ? Response::allow()
            : Response::deny('You must be an administrator.');
    });

即使您從 gate 返回授權回應，`Gate::allows` 方法仍會返回一個簡單的布林值；但是，您可以使用 `Gate::inspect` 方法來取得 gate 返回的完整授權回應：

    $response = Gate::inspect('edit-settings');

    if ($response->allowed()) {
        // The action is authorized...
    } else {
        echo $response->message();
    }

當使用 `Gate::authorize` 方法時，如果動作未被授權，該方法會拋出 `AuthorizationException`，授權回應提供的錯誤訊息將會傳播到 HTTP 回應中：

    Gate::authorize('edit-settings');

    // The action is authorized...

<a name="customizing-gate-response-status"></a>
#### 自訂 HTTP 回應狀態

當透過 Gate 拒絕某個動作時，會返回 `403` HTTP 回應；然而，有時返回替代的 HTTP 狀態碼會很有用。您可以使用 `Illuminate\Auth\Access\Response` 類別上的 `denyWithStatus` 靜態建構子來自訂失敗授權檢查所返回的 HTTP 狀態碼：

    use App\Models\User;
    use Illuminate\Auth\Access\Response;
    use Illuminate\Support\Facades\Gate;

    Gate::define('edit-settings', function (User $user) {
        return $user->isAdmin
            ? Response::allow()
            : Response::denyWithStatus(404);
    });

由於透過 `404` 回應隱藏資源是 Web 應用程式中常見的模式，因此提供了 `denyAsNotFound` 方法以方便使用：

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

如果 `before` 閉包返回非 null 的結果，該結果將被視為授權檢查的結果。

您可以使用 `after` 方法來定義一個閉包，該閉包會在所有其他授權檢查之後執行：

    use App\Models\User;

    Gate::after(function (User $user, string $ability, bool|null $result, mixed $arguments) {
        if ($user->isAdministrator()) {
            return true;
        }
    });

`after` 閉包返回的值不會覆寫授權檢查的結果，除非 gate 或 policy 返回 `null`。

<a name="inline-authorization"></a>
### 行內授權

有時，您可能希望判斷目前已驗證的使用者是否被授權執行特定動作，而無需撰寫與該動作對應的專用 gate。Laravel 允許您透過 `Gate::allowIf` 和 `Gate::denyIf` 方法執行這些「行內」授權檢查。行內授權不會執行任何已定義的「before」或「after」授權 hook：

```php
use App\Models\User;
use Illuminate\Support\Facades\Gate;

Gate::allowIf(fn (User $user) => $user->isAdministrator());

Gate::denyIf(fn (User $user) => $user->banned());
```

如果動作未被授權，或者目前沒有使用者已驗證，Laravel 將會自動拋出 `Illuminate\Auth\Access\AuthorizationException` 例外。`AuthorizationException` 的實例會被 Laravel 的例外處理器自動轉換為 403 HTTP 回應。

<a name="creating-policies"></a>
## 建立 Policies

<a name="generating-policies"></a>
### 產生 Policies

Policies 是將授權邏輯圍繞特定 model 或資源組織起來的類別。例如，如果您的應用程式是一個部落格，您可能會有一個 `App\Models\Post` model 和一個對應的 `App\Policies\PostPolicy` 來授權使用者動作，例如建立或更新文章。

您可以使用 `make:policy` Artisan 命令來產生 policy。產生的 policy 將會放置在 `app/Policies` 目錄中。如果您的應用程式中不存在此目錄，Laravel 會為您建立它：

```shell
php artisan make:policy PostPolicy
```

`make:policy` 命令將會產生一個空的 policy 類別。如果您想產生一個包含與檢視、建立、更新和刪除資源相關的範例 policy 方法的類別，您可以在執行命令時提供 `--model` 選項：

```shell
php artisan make:policy PostPolicy --model=Post
```

<a name="registering-policies"></a>
### 註冊 Policies

<a name="policy-discovery"></a>
#### Policy 探索

預設情況下，只要 model 和 policy 遵循標準的 Laravel 命名慣例，Laravel 就會自動探索 policies。具體來說，policies 必須位於包含您 models 的目錄或其上方的 `Policies` 目錄中。因此，例如，models 可以放置在 `app/Models` 目錄中，而 policies 可以放置在 `app/Policies` 目錄中。在這種情況下，Laravel 將會檢查 `app/Models/Policies`，然後是 `app/Policies` 中的 policies。此外，policy 名稱必須與 model 名稱匹配並帶有 `Policy` 後綴。因此，`User` model 將對應於 `UserPolicy` policy 類別。

如果您想定義自己的 policy 探索邏輯，您可以使用 `Gate::guessPolicyNamesUsing` 方法註冊一個自訂的 policy 探索回呼。通常，這個方法應該在應用程式的 `AppServiceProvider` 的 `boot` 方法中呼叫：

    use Illuminate\Support\Facades\Gate;

    Gate::guessPolicyNamesUsing(function (string $modelClass) {
        // Return the name of the policy class for the given model...
    });

<a name="manually-registering-policies"></a>
#### 手動註冊 Policies

使用 `Gate` facade，您可以在應用程式的 `AppServiceProvider` 的 `boot` 方法中手動註冊 policies 及其對應的 models：

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

一旦 policy 類別被註冊，您可以為其授權的每個動作添加方法。例如，讓我們在 `PostPolicy` 上定義一個 `update` 方法，該方法判斷給定的 `App\Models\User` 是否可以更新給定的 `App\Models\Post` 實例。

`update` 方法將接收 `User` 和 `Post` 實例作為其參數，並應返回 `true` 或 `false`，指示使用者是否被授權更新給定的 `Post`。因此，在這個範例中，我們將驗證使用者的 `id` 是否與文章上的 `user_id` 匹配：

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

您可以根據需要繼續在 policy 上定義額外的方法，以處理其授權的各種動作。例如，您可以定義 `view` 或 `delete` 方法來授權各種與 `Post` 相關的動作，但請記住，您可以自由地為您的 policy 方法命名。

如果您在透過 Artisan console 產生 policy 時使用了 `--model` 選項，它將已經包含 `viewAny`、`view`、`create`、`update`、`delete`、`restore` 和 `forceDelete` 動作的方法。

> [!NOTE]
> 所有 policies 都會透過 Laravel [service container](/docs/{{version}}/container) 解析，允許您在 policy 的建構子中型別提示任何所需的依賴項，以便它們自動注入。

<a name="policy-responses"></a>
### Policy 回應

到目前為止，我們只研究了返回簡單布林值的 policy 方法。然而，有時您可能希望返回更詳細的回應，包括錯誤訊息。為此，您可以從 policy 方法返回一個 `Illuminate\Auth\Access\Response` 實例：

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

當您從 policy 返回授權回應時，`Gate::allows` 方法仍會返回一個簡單的布林值；但是，您可以使用 `Gate::inspect` 方法來取得 gate 返回的完整授權回應：

    use Illuminate\Support\Facades\Gate;

    $response = Gate::inspect('update', $post);

    if ($response->allowed()) {
        // The action is authorized...
    } else {
        echo $response->message();
    }

當使用 `Gate::authorize` 方法時，如果動作未被授權，該方法會拋出 `AuthorizationException`，授權回應提供的錯誤訊息將會傳播到 HTTP 回應中：

    Gate::authorize('update', $post);

    // The action is authorized...

<a name="customizing-policy-response-status"></a>
#### 自訂 HTTP 回應狀態

當透過 policy 方法拒絕某個動作時，會返回 `403` HTTP 回應；然而，有時返回替代的 HTTP 狀態碼會很有用。您可以使用 `Illuminate\Auth\Access\Response` 類別上的 `denyWithStatus` 靜態建構子來自訂失敗授權檢查所返回的 HTTP 狀態碼：

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

由於透過 `404` 回應隱藏資源是 Web 應用程式中常見的模式，因此提供了 `denyAsNotFound` 方法以方便使用：

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

<a name="methods-without-models"></a>
### 不含 Model 的方法

有些 policy 方法只接收目前已驗證使用者的實例。這種情況在授權 `create` 動作時最常見。例如，如果您正在建立一個部落格，您可能希望判斷使用者是否被授權建立任何文章。在這些情況下，您的 policy 方法應該只期望接收一個使用者實例：

    /**
     * Determine if the given user can create posts.
     */
    public function create(User $user): bool
    {
        return $user->role == 'writer';
    }

<a name="guest-users"></a>
### 訪客使用者

預設情況下，如果傳入的 HTTP 請求不是由已驗證使用者發起，所有 gates 和 policies 都會自動返回 `false`。然而，您可以透過宣告「可選」的型別提示或為使用者參數定義提供 `null` 預設值，來允許這些授權檢查通過您的 gates 和 policies：

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

對於某些使用者，您可能希望授權特定 policy 中的所有動作。為此，請在 policy 上定義一個 `before` 方法。`before` 方法將在 policy 上的任何其他方法之前執行，讓您有機會在實際呼叫預期的 policy 方法之前授權該動作。此功能最常用於授權應用程式管理員執行任何動作：

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

如果您想拒絕特定類型使用者的所有授權檢查，您可以從 `before` 方法返回 `false`。如果返回 `null`，授權檢查將會繼續執行 policy 方法。

> [!WARNING]
> 如果 policy 類別不包含與正在檢查的能力名稱匹配的方法，則不會呼叫 policy 類別的 `before` 方法。

<a name="authorizing-actions-using-policies"></a>
## 使用 Policies 授權動作

<a name="via-the-user-model"></a>
### 透過 User Model

您的 Laravel 應用程式中包含的 `App\Models\User` model 包含兩個有用的方法用於授權動作：`can` 和 `cannot`。`can` 和 `cannot` 方法接收您希望授權的動作名稱和相關的 model。例如，讓我們判斷使用者是否被授權更新給定的 `App\Models\Post` model。通常，這會在 controller 方法中完成：

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

如果為給定的 model [註冊了 policy](#registering-policies)，`can` 方法將會自動呼叫適當的 policy 並返回布林結果。如果沒有為 model 註冊 policy，`can` 方法將會嘗試呼叫與給定動作名稱匹配的基於閉包的 Gate。

<a name="user-model-actions-that-dont-require-models"></a>
#### 不需要 Model 的動作

請記住，某些動作可能對應於像 `create` 這樣不需要 model 實例的 policy 方法。在這些情況下，您可以將類別名稱傳遞給 `can` 方法。類別名稱將用於判斷在授權動作時要使用哪個 policy：

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

除了提供給 `App\Models\User` model 的有用方法外，您始終可以透過 `Gate` facade 的 `authorize` 方法授權動作。

與 `can` 方法一樣，此方法接受您希望授權的動作名稱和相關的 model。如果動作未被授權，`authorize` 方法將會拋出 `Illuminate\Auth\Access\AuthorizationException` 例外，Laravel 例外處理器會自動將其轉換為狀態碼為 403 的 HTTP 回應：

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
#### 不需要 Model 的 Controller 動作

如前所述，某些 policy 方法（例如 `create`）不需要 model 實例。在這些情況下，您應該將類別名稱傳遞給 `authorize` 方法。類別名稱將用於判斷在授權動作時要使用哪個 policy：

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
### 透過 Middleware

Laravel 包含一個 middleware，可以在傳入請求到達您的 routes 或 controllers 之前授權動作。預設情況下，`Illuminate\Auth\Middleware\Authorize` middleware 可以使用 `can` [middleware alias](/docs/{{version}}/middleware#middleware-aliases) 附加到 route，該 alias 由 Laravel 自動註冊。讓我們探討一個使用 `can` middleware 授權使用者可以更新文章的範例：

    use App\Models\Post;

    Route::put('/post/{post}', function (Post $post) {
        // The current user may update the post...
    })->middleware('can:update,post');

在這個範例中，我們將兩個參數傳遞給 `can` middleware。第一個是我們希望授權的動作名稱，第二個是我們希望傳遞給 policy 方法的 route 參數。在這種情況下，由於我們使用 [implicit model binding](/docs/{{version}}/routing#implicit-binding)，一個 `App\Models\Post` model 將會傳遞給 policy 方法。如果使用者未被授權執行給定動作，middleware 將會返回狀態碼為 403 的 HTTP 回應。

為方便起見，您也可以使用 `can` 方法將 `can` middleware 附加到您的 route：

    use App\Models\Post;

    Route::put('/post/{post}', function (Post $post) {
        // The current user may update the post...
    })->can('update', 'post');

<a name="middleware-actions-that-dont-require-models"></a>
#### 不需要 Model 的 Middleware 動作

同樣地，某些 policy 方法（例如 `create`）不需要 model 實例。在這些情況下，您可以將類別名稱傳遞給 middleware。類別名稱將用於判斷在授權動作時要使用哪個 policy：

    Route::post('/post', function () {
        // The current user may create posts...
    })->middleware('can:create,App\Models\Post');

在字串 middleware 定義中指定完整的類別名稱可能會變得繁瑣。因此，您可以選擇使用 `can` 方法將 `can` middleware 附加到您的 route：

    use App\Models\Post;

    Route::post('/post', function () {
        // The current user may create posts...
    })->can('create', Post::class);

<a name="via-blade-templates"></a>
### 透過 Blade 模板

在撰寫 Blade 模板時，您可能希望僅在使用者被授權執行特定動作時才顯示頁面的一部分。例如，您可能希望僅在使用者實際可以更新部落格文章時才顯示部落格文章的更新表單。在這種情況下，您可以使用 ` @can` 和 ` @cannot` directives：

```blade
 @can('update', $post)
    <!-- The current user can update the post... -->
 @elsecan('create', App\Models\Post::class)
    <!-- The current user can create new posts... -->
 @else
    <!-- ... -->
 @endcan @cannot('update', $post)
    <!-- The current user cannot update the post... -->
 @elsecannot('create', App\Models\Post::class)
    <!-- The current user cannot create new posts... -->
 @endcannot
```

這些 directives 是撰寫 ` @if` 和 ` @unless` 語句的便捷捷徑。上述的 ` @can` 和 ` @cannot` 語句等同於以下語句：

```blade
 @if (Auth::user()->can('update', $post))
    <!-- The current user can update the post... -->
 @endif @unless (Auth::user()->can('update', $post))
    <!-- The current user cannot update the post... -->
 @endunless
```

您也可以判斷使用者是否被授權執行給定動作陣列中的任何動作。為此，請使用 ` @canany` directive：

```blade
 @canany(['update', 'view', 'delete'], $post)
    <!-- The current user can update, view, or delete the post... -->
 @elsecanany(['create'], \App\Models\Post::class)
    <!-- The current user can create a post... -->
 @endcanany
```

<a name="blade-actions-that-dont-require-models"></a>
#### 不需要 Model 的 Blade 動作

與大多數其他授權方法一樣，如果動作不需要 model 實例，您可以將類別名稱傳遞給 ` @can` 和 ` @cannot` directives：

```blade
 @can('create', App\Models\Post::class)
    <!-- The current user can create posts... -->
 @endcan @cannot('create', App\Models\Post::class)
    <!-- The current user can't create posts... -->
 @endcannot
```

<a name="supplying-additional-context"></a>
### 提供額外 Context

當使用 policies 授權動作時，您可以將一個陣列作為第二個參數傳遞給各種授權函數和輔助函數。陣列中的第一個元素將用於判斷應該呼叫哪個 policy，而陣列的其餘元素則作為參數傳遞給 policy 方法，並可用於在做出授權決策時提供額外的 context。例如，考慮以下包含額外 `$category` 參數的 `PostPolicy` 方法定義：

    /**
     * Determine if the given post can be updated by the user.
     */
    public function update(User $user, Post $post, int $category): bool
    {
        return $user->id === $post->user_id &&
               $user->canUpdateCategory($category);
    }

當嘗試判斷已驗證使用者是否可以更新給定文章時，我們可以這樣呼叫此 policy 方法：

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

儘管授權必須始終在伺服器端處理，但為您的前端應用程式提供授權資料通常會很方便，以便正確呈現應用程式的 UI。Laravel 並未定義將授權資訊公開給 Inertia 驅動的前端所需的慣例。

然而，如果您正在使用 Laravel 的其中一個基於 Inertia 的 [starter kits](/docs/{{version}}/starter-kits)，您的應用程式已經包含一個 `HandleInertiaRequests` middleware。在此 middleware 的 `share` 方法中，您可以返回將提供給應用程式中所有 Inertia 頁面的共享資料。此共享資料可以作為定義使用者授權資訊的便捷位置：

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
