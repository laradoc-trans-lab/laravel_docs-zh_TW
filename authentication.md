# 認證

- [簡介](#introduction)
    - [入門套件](#starter-kits)
    - [資料庫考量](#introduction-database-considerations)
    - [生態系統概覽](#ecosystem-overview)
- [認證快速入門](#authentication-quickstart)
    - [安裝入門套件](#install-a-starter-kit)
    - [取得已認證的使用者](#retrieving-the-authenticated-user)
    - [保護路由](#protecting-routes)
    - [登入嘗試限制](#login-throttling)
- [手動認證使用者](#authenticating-users)
    - [記住使用者](#remembering-users)
    - [其他認證方法](#other-authentication-methods)
- [HTTP 基本認證](#http-basic-authentication)
    - [無狀態 HTTP 基本認證](#stateless-http-basic-authentication)
- [登出](#logging-out)
    - [讓其他裝置的 Session 失效](#invalidating-sessions-on-other-devices)
- [密碼確認](#password-confirmation)
    - [設定](#password-confirmation-configuration)
    - [路由](#password-confirmation-routing)
    - [保護路由](#password-confirmation-protecting-routes)
- [新增自訂 Guard](#adding-custom-guards)
    - [Closure 請求 Guard](#closure-request-guards)
- [新增自訂使用者提供者](#adding-custom-user-providers)
    - [User Provider 契約(Contracts)](#the-user-provider-contract)
    - [Authenticatable 契約(Contracts)](#the-authenticatable-contract)
- [自動密碼重新雜湊](#automatic-password-rehashing)
- [社群登入認證](/docs/{{version}}/socialite)
- [事件](#events)

<a name="introduction"></a>
## 簡介

許多 Web 應用程式都提供了一種讓使用者認證並「登入」應用程式的方法。在 Web 應用程式中實作此功能可能是一項複雜且潛在高風險的工作。因此，Laravel 致力於為您提供快速、安全且輕鬆實作認證所需的工具。

從核心來看，Laravel 的認證機制是由「guard」與「provider」所組成。Guard 定義了使用者在每次請求中如何被認證。例如，Laravel 內建了一個 `session` guard，它會使用 session 儲存空間和 cookie 來維護狀態。

Provider 則定義了如何從您的持久化儲存空間中擷取使用者。Laravel 內建支援使用 [Eloquent](/docs/{{version}}/eloquent) 和資料庫查詢建立器 (Query Builder) 來擷取使用者。然而，您可以根據應用程式的需求自由定義額外的 provider。

您的應用程式認證設定檔位於 `config/auth.php`。該檔案包含許多說明完善的選項，可用於微調 Laravel 認證服務的行為。

> [!NOTE]
> Guard 與 provider 不應與「角色 (roles)」和「權限 (permissions)」混淆。若要瞭解更多透過權限授權使用者操作的資訊，請參閱[授權](/docs/{{version}}/authorization)文件。

<a name="starter-kits"></a>
### 入門套件

想要快速開始嗎？在全新的 Laravel 應用程式中安裝 [Laravel 應用程式入門套件](/docs/{{version}}/starter-kits)。在執行資料庫遷移後，將您的瀏覽器導向至 `/register` 或任何指派給您應用程式的 URL。入門套件將會為您建置整個認證系統的骨架！

**即使您選擇不在最終的 Laravel 應用程式中使用入門套件，安裝[入門套件](/docs/{{version}}/starter-kits)也是一個在真實的 Laravel 專案中學習如何實作所有 Laravel 認證功能的絕佳機會。**由於 Laravel 入門套件已經為您包含了認證控制器、路由和檢視 (views)，您可以檢視這些檔案中的程式碼，以瞭解如何實作 Laravel 的認證功能。

<a name="introduction-database-considerations"></a>
### 資料庫考量

預設情況下，Laravel 在您的 `app/Models` 目錄中包含了一個 `App\Models\User` [Eloquent 模型](/docs/{{version}}/eloquent)。此模型可搭配預設的 Eloquent 認證驅動程式使用。

如果您的應用程式沒有使用 Eloquent，您可以使用搭配 Laravel 查詢建立器的 `database` 認證提供者。如果您的應用程式使用的是 MongoDB，請參考 MongoDB 的官方 [Laravel 使用者認證文件](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/user-authentication/)。

為 `App\Models\User` 模型建立資料庫結構 (Schema) 時，請確保密碼欄位長度至少為 60 個字元。當然，全新 Laravel 應用程式中包含的 `users` 資料表遷移檔已經建立了一個超過此長度的欄位。

此外，您應該驗證您的 `users`（或同等功能）資料表包含一個長度為 100 個字元、可為 null 的字串型態 `remember_token` 欄位。此欄位將用於儲存選擇「記住我」選項登入應用程式的使用者令牌。同樣地，全新的 Laravel 應用程式所包含的預設 `users` 資料表遷移檔已經包含了此欄位。

<a name="ecosystem-overview"></a>
### 生態系統概覽

Laravel 提供數個與認證相關的套件。在繼續之前，我們將回顧 Laravel 中的整體認證生態系統，並討論每個套件預期的用途。

首先，思考一下認證是如何運作的。使用網頁瀏覽器時，使用者會透過登入表單提供其使用者名稱與密碼。如果這些憑證正確，應用程式將在使用者的 [session](/docs/{{version}}/session) 中儲存有關已認證使用者的資訊。發送到瀏覽器的 Cookie 包含 Session ID，以便對應用程式的後續請求可以將使用者與正確的 Session 關聯起來。收到 Session Cookie 後，應用程式將根據 Session ID 檢索 Session 資料，注意到認證資訊已儲存在 Session 中，並將該使用者視為「已認證」。

當遠端服務需要認證以存取 API 時，通常不會使用 Cookie 進行認證，因為沒有網頁瀏覽器。相反地，遠端服務會在每次請求時發送一個 API token 給 API。應用程式可以對照有效的 API token 資料表來驗證傳入的 token，並將該請求「認證」為是由與該 API token 關聯的使用者所執行的。

<a name="laravels-built-in-browser-authentication-services"></a>
#### Laravel 內建的瀏覽器認證服務

Laravel 包含內建的認證與 Session 服務，通常透過 `Auth` 及 `Session` Facade 進行存取。這些功能為從網頁瀏覽器發起的請求提供基於 Cookie 的認證。它們提供了允許您驗證使用者憑證並認證使用者的方法。此外，這些服務會自動將適當的認證資料儲存在使用者的 Session 中，並發行使用者的 Session Cookie。本文件中包含了如何使用這些服務的討論。

**應用程式入門套件**

如本文件所述，您可以手動與這些認證服務互動，建立您應用程式專屬的認證層。然而，為了協助您更快上手，我們發布了[免費的入門套件](/docs/{{version}}/starter-kits)，為整個認證層提供強大且現代化的基架 (Scaffolding)。

<a name="laravels-api-authentication-services"></a>
#### Laravel 的 API 認證服務

Laravel 提供兩個可選的套件來協助您管理 API token 並認證使用 API token 發起的請求：[Passport](/docs/{{version}}/passport) 與 [Sanctum](/docs/{{version}}/sanctum)。請注意，這些函式庫與 Laravel 內建基於 Cookie 的認證函式庫並不是互斥的。這些函式庫主要專注於 API token 認證，而內建的認證服務則專注於基於 Cookie 的瀏覽器認證。許多應用程式會同時使用 Laravel 內建基於 Cookie 的認證服務以及 Laravel 的其中一個 API 認證套件。

**Passport**

Passport 是一個 OAuth2 認證提供者，提供各種 OAuth2「授權類型 (grant types)」，允許您發行各種類型的 token。總體來說，這是一個用於 API 認證強大且複雜的套件。然而，大多數應用程式並不需要 OAuth2 規格所提供的複雜功能，這可能會讓使用者和開發人員感到困惑。此外，開發人員過去對於如何使用像 Passport 這樣的 OAuth2 認證提供者來認證 SPA 應用程式或行動裝置應用程式常常感到困惑。

**Sanctum**

為了應對 OAuth2 的複雜性與開發人員的困惑，我們著手建構一個更簡單、更精簡的認證套件，它可以同時處理來自網頁瀏覽器的第一方 Web 請求以及透過 token 的 API 請求。這個目標隨著 [Laravel Sanctum](/docs/{{version}}/sanctum) 的發布而實現，對於除了 API 之外還提供第一方 Web UI、或是由獨立於 Laravel 後端應用程式之外的單頁應用程式 (SPA) 所驅動、或者有提供行動客戶端的應用程式來說，Sanctum 應被視為首選且推薦的認證套件。

Laravel Sanctum 是一個混合 Web / API 認證套件，可以管理您應用程式的整個認證過程。這是可行的，因為當基於 Sanctum 的應用程式收到請求時，Sanctum 會首先判斷請求是否包含引用已認證 Session 的 Session Cookie。Sanctum 透過呼叫我們前面討論過的 Laravel 內建認證服務來實現這一點。如果請求不是透過 Session Cookie 進行認證，Sanctum 將檢查請求中是否包含 API token。如果存在 API token，Sanctum 將使用該 token 來認證請求。要深入瞭解此流程，請參閱 Sanctum 的[「運作原理」](/docs/{{version}}/sanctum#how-it-works)文件。

<a name="summary-choosing-your-stack"></a>
#### 總結與選擇您的技術堆疊

總結來說，如果您的應用程式將使用瀏覽器存取，且您正在建構單體 (Monolithic) Laravel 應用程式，您的應用程式將使用 Laravel 內建的認證服務。

接下來，如果您的應用程式提供了供第三方使用的 API，您將在 [Passport](/docs/{{version}}/passport) 或 [Sanctum](/docs/{{version}}/sanctum) 之間進行選擇，以防為您的應用程式提供 API token 認證。一般來說，在可能的情況下應優先選擇 Sanctum，因為它是一個用於 API 認證、SPA 認證與行動裝置認證的簡單且完整的解決方案，包含對「作用域 (scopes)」或「功能 (abilities)」的支援。

如果您正在建構一個由 Laravel 後端驅動的單頁應用程式 (SPA)，您應該使用 [Laravel Sanctum](/docs/{{version}}/sanctum)。使用 Sanctum 時，您需要[手動實現自己的後端認證路由](#authenticating-users)，或是利用 [Laravel Fortify](/docs/{{version}}/fortify) 作為 Headless 認證後端服務，由它提供註冊、密碼重設、電子郵件驗證等功能的路由與控制器。

當您的應用程式絕對需要 OAuth2 規格所提供的所有功能時，可以選擇 Passport。此外，如果您正在建構一個將由 AI 用戶端存取的 [MCP 伺服器](/docs/{{version}}/mcp)，您應該使用 Passport，因為 MCP 用戶端通常預期[使用 OAuth 進行認證](/docs/{{version}}/mcp#oauth)。

而且，如果您想快速上手，我們很高興向您推薦[我們的應用程式入門套件](/docs/{{version}}/starter-kits)，這是啟動新 Laravel 應用程式的快速方式，該應用程式已經使用了我們偏好的認證堆疊——Laravel 內建的認證服務。

<a name="authentication-quickstart"></a>
## 認證快速入門

> [!WARNING]
> 本文件的此部分討論如何透過 [Laravel 應用程式入門套件](/docs/{{version}}/starter-kits) 認證使用者，其中包括 UI 骨架以協助您快速開始。如果您想直接與 Laravel 的認證系統整合，請參考[手動認證使用者](#authenticating-users)的說明文件。


<a name="install-a-starter-kit"></a>
### 安裝入門套件

首先，您應該[安裝 Laravel 應用程式入門套件](/docs/{{version}}/starter-kits)。我們的入門套件提供了精心設計的起點，能將認證功能整合至您全新的 Laravel 應用程式中。


<a name="retrieving-the-authenticated-user"></a>
### 取得已認證的使用者

使用入門套件建立應用程式並讓使用者能夠在您的應用程式中註冊與認證後，您經常需要與當前已認證的使用者進行互動。在處理傳入的請求時，您可以透過 `Auth` Facade 的 `user` 方法存取已認證的使用者：

```php
use Illuminate\Support\Facades\Auth;

// Retrieve the currently authenticated user...
$user = Auth::user();

// Retrieve the currently authenticated user's ID...
$id = Auth::id();
```

另外，使用者通過認證後，您也可以透過 `Illuminate\Http\Request` 實例存取已認證的使用者。請記住，型態提示 (type-hinted) 的類別將會自動注入到您的控制器方法中。透過型態提示 `Illuminate\Http\Request` 物件，您可以透過請求的 `user` 方法，從應用程式中的任何控制器方法方便地存取已認證的使用者：

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;

class FlightController extends Controller
{
    /**
     * Update the flight information for an existing flight.
     */
    public function update(Request $request): RedirectResponse
    {
        $user = $request->user();

        // ...

        return redirect('/flights');
    }
}
```


<a name="determining-if-the-current-user-is-authenticated"></a>
#### 確認當前使用者是否已通過認證

若要確認發送傳入 HTTP 請求的使用者是否已通過認證，您可以使用 `Auth` Facade 的 `check` 方法。如果使用者已通過認證，該方法將會回傳 `true`：

```php
use Illuminate\Support\Facades\Auth;

if (Auth::check()) {
    // The user is logged in...
}
```

> [!NOTE]
> 雖然可以使用 `check` 方法來確認使用者是否已通過認證，但您通常會使用中介層在允許使用者存取特定路由 / 控制器之前，先驗證該使用者是否已通過認證。若要瞭解更多資訊，請參考[保護路由](/docs/{{version}}/authentication#protecting-routes)的說明文件。


<a name="protecting-routes"></a>
### 保護路由

[路由中介層](/docs/{{version}}/middleware)可用於僅允許已認證的使用者存取指定的路由。Laravel 內建了 `auth` 中介層，這是 `Illuminate\Auth\Middleware\Authenticate` 類別的[中介層別名](/docs/{{version}}/middleware#middleware-aliases)。由於此中介層已在 Laravel 內部設定別名，您只需要將該中介層附加到路由定義上即可：

```php
Route::get('/flights', function () {
    // Only authenticated users may access this route...
})->middleware('auth');
```


<a name="redirecting-unauthenticated-users"></a>
#### 重新導向未認證的使用者

當 `auth` 中介層偵測到未認證的使用者時，它會將使用者重新導向至 `login` [具名路由](/docs/{{version}}/routing#named-routes)。您可以在應用程式的 `bootstrap/app.php` 檔案中使用 `redirectGuestsTo` 方法來修改此行為：

```php
use Illuminate\Http\Request;

->withMiddleware(function (Middleware $middleware): void {
    $middleware->redirectGuestsTo('/login');

    // Using a closure...
    $middleware->redirectGuestsTo(fn (Request $request) => route('login'));
})
```


<a name="redirecting-authenticated-users"></a>
#### 重新導向已認證的使用者

當 `guest` 中介層偵測到已認證的使用者時，它會將使用者重新導向至 `dashboard` 或 `home` 具名路由。您可以在應用程式的 `bootstrap/app.php` 檔案中使用 `redirectUsersTo` 方法來修改此行為：

```php
use Illuminate\Http\Request;

->withMiddleware(function (Middleware $middleware): void {
    $middleware->redirectUsersTo('/panel');

    // Using a closure...
    $middleware->redirectUsersTo(fn (Request $request) => route('panel'));
})
```


<a name="specifying-a-guard"></a>
#### 指定 Guard

將 `auth` 中介層附加到路由時，您也可以指定應用於認證使用者的「guard」。所指定的 guard 應該對應到 `auth.php` 設定檔中 `guards` 陣列的其中一個鍵值：

```php
Route::get('/flights', function () {
    // Only authenticated users may access this route...
})->middleware('auth:admin');
```


<a name="login-throttling"></a>
### 登入嘗試限制

如果您使用的是我們的 [應用程式入門套件](/docs/{{version}}/starter-kits) 之一，系統將會自動對登入嘗試套用速率限制 (Rate limiting)。預設情況下，如果使用者在多次嘗試後無法提供正確的憑證，他們將在一分鐘內無法登入。此限制是依據使用者的使用者名稱 / Email 地址以及其 IP 地址來分別計算的。

> [!NOTE]
> 如果您想對應用程式中的其他路由進行速率限制，請參考[速率限制說明文件](/docs/{{version}}/routing#rate-limiting)。

<a name="authenticating-users"></a>
## 手動認證使用者

您並不一定要使用 Laravel [應用程式入門套件](/docs/{{version}}/starter-kits) 所提供的認證鷹架 (Scaffolding)。若選擇不使用這些鷹架，您就需要直接使用 Laravel 的認證類別來管理使用者認證。別擔心，這非常簡單！

我們將透過 `Auth` [Facade](/docs/{{version}}/facades) 來存取 Laravel 的認證服務，因此需要確保在類別頂端匯入 `Auth` Facade。接著，我們來看看 `attempt` 方法。`attempt` 方法通常用於處理來自應用程式「登入」表單的認證嘗試。若認證成功，您應該重新產生使用者的 [Session](/docs/{{version}}/session)，以防止 [Session 固定攻擊 (Session fixation)](https://en.wikipedia.org/wiki/Session_fixation)：

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Http\RedirectResponse;
use Illuminate\Support\Facades\Auth;

class LoginController extends Controller
{
    /**
     * Handle an authentication attempt.
     */
    public function authenticate(Request $request): RedirectResponse
    {
        $credentials = $request->validate([
            'email' => ['required', 'email'],
            'password' => ['required'],
        ]);

        if (Auth::attempt($credentials)) {
            $request->session()->regenerate();

            return redirect()->intended('dashboard');
        }

        return back()->withErrors([
            'email' => 'The provided credentials do not match our records.',
        ])->onlyInput('email');
    }
}
```

`attempt` 方法的第一個引數接收一個鍵值對 (Key / Value) 陣列。陣列中的值將用於在資料庫資料表中尋找使用者。因此，在上面的範例中，使用者將透過 `email` 欄位的值來檢索。如果找到使用者，儲存在資料庫中的雜湊密碼將與透過陣列傳遞給該方法的 `password` 值進行比對。您不應該對傳入請求的 `password` 值進行雜湊，因為框架會在將該值與資料庫中的雜湊密碼比對之前自動進行雜湊處理。若兩個雜湊密碼比對符合，將會為使用者啟動一個已認證的 Session。

請記住，Laravel 的認證服務會根據您認證 Guard 的「Provider」設定來從資料庫檢索使用者。在預設的 `config/auth.php` 設定檔中，指定了 Eloquent 使用者 Provider，並指示其在檢索使用者時使用 `App\Models\User` 模型。您可以根據應用程式的需求在設定檔中更改這些值。

若認證成功，`attempt` 方法將傳回 `true`。否則，將傳回 `false`。

Laravel 重導向器 (Redirector) 提供的 `intended` 方法會將使用者重導向至他們在被認證中介層攔截前嘗試存取的 URL。如果預期的目的地不可用，可以向此方法提供備用 URI。


<a name="specifying-additional-conditions"></a>
#### 指定額外條件

如果需要，除了使用者的 Email 與密碼之外，您還可以向認證查詢加入額外的查詢條件。為實現這一點，我們只需將查詢條件加入到傳遞給 `attempt` 方法的陣列中即可。例如，我們可以驗證使用者是否被標記為「active」：

```php
if (Auth::attempt(['email' => $email, 'password' => $password, 'active' => 1])) {
    // Authentication was successful...
}
```

對於複雜的查詢條件，您可以在憑證陣列中提供一個 Closure。這個 Closure 將會以查詢實例作為引數被呼叫，讓您可以根據應用程式的需求來自訂查詢：

```php
use Illuminate\Database\Eloquent\Builder;

if (Auth::attempt([
    'email' => $email,
    'password' => $password,
    fn (Builder $query) => $query->has('activeSubscription'),
])) {
    // Authentication was successful...
}
```

> [!WARNING]
> 在這些範例中，`email` 並非必要選項，它僅僅作為範例使用。您應該使用資料庫資料表中對應於「使用者名稱」的任何欄位名稱。

`attemptWhen` 方法接受 Closure 作為其第二個引數，可用於在實際認證使用者之前對潛在使用者進行更廣泛的檢視。該 Closure 接收潛在使用者實例，並應傳回 `true` 或 `false` 以指示是否可以認證該使用者：

```php
if (Auth::attemptWhen([
    'email' => $email,
    'password' => $password,
], function (User $user) {
    return $user->isNotBanned();
})) {
    // Authentication was successful...
}
```


<a name="accessing-specific-guard-instances"></a>
#### 存取特定的 Guard 實例

透過 `Auth` Facade 的 `guard` 方法，您可以指定在認證使用者時想要使用的 Guard 實例。這允許您使用完全獨立的可認證 (Authenticatable) 模型或使用者資料表來管理應用程式中不同部分的認證。

傳遞給 `guard` 方法的 Guard 名稱應對應於 `auth.php` 設定檔中設定的其中一個 Guard：

```php
if (Auth::guard('admin')->attempt($credentials)) {
    // ...
}
```


<a name="remembering-users"></a>
### 記住使用者

許多 Web 應用程式在其登入表單上提供「記住我」核取方塊。如果您想在應用程式中提供「記住我」功能，可以傳遞一個布林值作為 `attempt` 方法的第二個引數。

當此值為 `true` 時，Laravel 將無限期保持使用者的認證狀態，直到他們手動登出。您的 `users` 資料表必須包含字串型態的 `remember_token` 欄位，該欄位將用於儲存「記住我」令牌。新 Laravel 應用程式包含的 `users` 資料表遷移 (Migration) 已經包含此欄位：

```php
use Illuminate\Support\Facades\Auth;

if (Auth::attempt(['email' => $email, 'password' => $password], $remember)) {
    // The user is being remembered...
}
```

如果您的應用程式提供「記住我」功能，您可以使用 `viaRemember` 方法來確定目前已認證的使用者是否是使用「記住我」Cookie 進行認證的：

```php
use Illuminate\Support\Facades\Auth;

if (Auth::viaRemember()) {
    // ...
}
```

<a name="other-authentication-methods"></a>
### 其他認證方法

<a name="authenticate-a-user-instance"></a>
#### 認證使用者實例

如果您需要將現有的使用者實例設定為當前已認證的使用者，可以將使用者實例傳遞給 `Auth` Facade 的 `login` 方法。給定的使用者實例必須實現 `Illuminate\Contracts\Auth\Authenticatable` [契約(Contracts)](/docs/{{version}}/contracts)。Laravel 隨附的 `App\Models\User` 模型已經實作了這個介面。這種認證方法在您已經擁有有效的使用者實例時非常有用，例如使用者剛在您的應用程式中完成註冊之後：

```php
use Illuminate\Support\Facades\Auth;

Auth::login($user);
```

您可以傳遞一個布林值作為 `login` 方法的第二個引數。此值指出該認證 Session 是否需要「記住我」的功能。請記住，這意味著該 Session 將保持認證狀態，直到使用者手動登出應用程式為止：

```php
Auth::login($user, $remember = true);
```

如果需要，您可以在呼叫 `login` 方法之前指定認證 guard：

```php
Auth::guard('admin')->login($user);
```

<a name="authenticate-a-user-by-id"></a>
#### 透過 ID 認證使用者

要使用使用者資料庫紀錄的主鍵來進行認證，您可以使用 `loginUsingId` 方法。此方法接受您想要認證的使用者主鍵：

```php
Auth::loginUsingId(1);
```

您可以傳遞布林值給 `loginUsingId` 方法的 `remember` 引數。此值指出該認證 Session 是否需要「記住我」的功能。請記住，這意味著該 Session 將保持認證狀態，直到使用者手動登出應用程式為止：

```php
Auth::loginUsingId(1, remember: true);
```

<a name="authenticate-a-user-once"></a>
#### 單次認證使用者

您可以使用 `once` 方法為單一請求在應用程式中認證使用者。呼叫此方法時不會使用任何 Session 或 Cookie，並且不會發送 `Login` 事件：

```php
if (Auth::once($credentials)) {
    // ...
}
```

<a name="http-basic-authentication"></a>
## HTTP 基本認證

[HTTP Basic Authentication](https://en.wikipedia.org/wiki/Basic_access_authentication) 提供了一種快速為應用程式使用者進行認證的方法，無需設定專屬的「登入」頁面。若要開始使用，請將 `auth.basic` [中介層](/docs/{{version}}/middleware) 附加到路由上。`auth.basic` 中介層已包含在 Laravel 框架中，因此您無需自行定義：

```php
Route::get('/profile', function () {
    // Only authenticated users may access this route...
})->middleware('auth.basic');
```

中介層附加到路由後，當您在瀏覽器中存取該路由時，系統將會自動提示您輸入憑證。預設情況下，`auth.basic` 中介層會假設您 `users` 資料庫表單中的 `email` 欄位是使用者的「使用者名稱」。


<a name="a-note-on-fastcgi"></a>
#### 關於 FastCGI 的注意事項

如果您使用 [PHP FastCGI](https://www.php.net/manual/en/install.fpm.php) 和 Apache 來運行您的 Laravel 應用程式，HTTP 基本認證可能無法正常運作。若要修正這些問題，可以將以下幾行程式碼新增到應用程式的 `.htaccess` 檔案中：

```apache
RewriteCond %{HTTP:Authorization} ^(.+)$
RewriteRule .* - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization}]
```


<a name="stateless-http-basic-authentication"></a>
### 無狀態 HTTP 基本認證

您也可以使用 HTTP 基本認證，而不在 Session 中設定使用者識別碼 Cookie。這在您選擇使用 HTTP 認證來驗證發送到應用程式 API 的請求時特別有用。若要達成此目的，請[定義一個中介層](/docs/{{version}}/middleware)來呼叫 `onceBasic` 方法。如果 `onceBasic` 方法沒有回傳任何回應，則請求可以進一步傳遞到應用程式中：

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use Symfony\Component\HttpFoundation\Response;

class AuthenticateOnceWithBasicAuth
{
    /**
     * Handle an incoming request.
     *
     * @param  \Closure(\Illuminate\Http\Request): (\Symfony\Component\HttpFoundation\Response)  $next
     */
    public function handle(Request $request, Closure $next): Response
    {
        return Auth::onceBasic() ?: $next($request);
    }

}
```

接著，將該中介層附加到路由：

```php
Route::get('/api/user', function () {
    // Only authenticated users may access this route...
})->middleware(AuthenticateOnceWithBasicAuth::class);
```


<a name="logging-out"></a>
## 登出

若要手動讓使用者從您的應用程式登出，您可以使用 `Auth` Facade 提供的 `logout` 方法。這將從使用者的 Session 中移除認證資訊，使得後續的請求不再具備認證狀態。

除了呼叫 `logout` 方法之外，建議您讓使用者的 Session 失效並重新產生其 [CSRF token](/docs/{{version}}/csrf)。在讓使用者登出後，通常會將使用者重新導向至應用程式的根目錄：

```php
use Illuminate\Http\Request;
use Illuminate\Http\RedirectResponse;
use Illuminate\Support\Facades\Auth;

/**
 * Log the user out of the application.
 */
public function logout(Request $request): RedirectResponse
{
    Auth::logout();

    $request->session()->invalidate();

    $request->session()->regenerateToken();

    return redirect('/');
}
```


<a name="invalidating-sessions-on-other-devices"></a>
### 讓其他裝置的 Session 失效

Laravel 還提供了一種機制，可讓使用者在其他裝置上處於活動狀態的 Session 失效並使其「登出」，同時不會讓他們當前裝置上的 Session 失效。當使用者更改或更新密碼，且您希望在保持當前裝置認證狀態的同時讓其他裝置上的 Session 失效時，通常會使用此功能。

在開始之前，您應該確保 `Illuminate\Session\Middleware\AuthenticateSession` 中介層已包含在應該接收 Session 認證的路由中。通常，您應該將此中介層放在路由群組定義上，以便它可以套用到應用程式的大部分路由。預設情況下，可以使用 `auth.session` [中介層別名](/docs/{{version}}/middleware#middleware-aliases)將 `AuthenticateSession` 中介層附加到路由：

```php
Route::middleware(['auth', 'auth.session'])->group(function () {
    Route::get('/', function () {
        // ...
    });
});
```

然後，您可以使用 `Auth` Facade 提供的 `logoutOtherDevices` 方法。此方法需要使用者確認其當前密碼，您的應用程式應透過輸入表單接收該密碼：

```php
use Illuminate\Support\Facades\Auth;

Auth::logoutOtherDevices($currentPassword);
```

當呼叫 `logoutOtherDevices` 方法時，使用者的其他 Session 將會完全失效，這意味著他們將從之前通過認證的所有 Guard 中「登出」。

<a name="password-confirmation"></a>
## 密碼確認

在建置應用程式時，你偶爾會有一些操作需要使用者在執行該操作或被重定向至應用程式敏感區域之前先確認其密碼。Laravel 內建的中介層讓這個過程變得非常簡單。實作此功能需要你定義兩個路由：一個路由用於顯示要求使用者確認密碼的視圖，另一個路由則用於確認密碼是否有效並將使用者重定向至其原本想去的目的地。

> [!NOTE]
> 以下說明將探討如何直接整合 Laravel 的密碼確認功能；不過，如果你想更快速地開始，[Laravel 應用程式入門套件](/docs/{{version}}/starter-kits) 已經支援了此功能！


<a name="password-confirmation-configuration"></a>
### 設定

在確認密碼後，系統在三個小時內不會再次要求使用者確認密碼。不過，你可以透過修改應用程式 `config/auth.php` 設定檔中的 `password_timeout` 設定值，來調整再次提示使用者輸入密碼的時間間隔。


<a name="password-confirmation-routing"></a>
### 路由


<a name="the-password-confirmation-form"></a>
#### 密碼確認表單

首先，我們將定義一個路由來顯示要求使用者確認密碼的視圖：

```php
Route::get('/confirm-password', function () {
    return view('auth.confirm-password');
})->middleware('auth')->name('password.confirm');
```

正如你所預期的，該路由返回的視圖應該包含一個帶有 `password` 欄位的表單。此外，你可以在視圖中加入說明文字，向使用者解釋他們正在進入應用程式的受保護區域，因此必須確認其密碼。


<a name="confirming-the-password"></a>
#### 確認密碼

接下來，我們將定義一個路由來處理來自「確認密碼」視圖的表單請求。這個路由將負責驗證密碼，並將使用者重定向至其原本想去的目的地：

```php
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Hash;

Route::post('/confirm-password', function (Request $request) {
    if (! Hash::check($request->password, $request->user()->password)) {
        return back()->withErrors([
            'password' => ['The provided password does not match our records.']
        ]);
    }

    $request->session()->passwordConfirmed();

    return redirect()->intended();
})->middleware(['auth', 'throttle:6,1']);
```

在繼續之前，讓我們更詳細地檢視這個路由。首先，確認請求中的 `password` 欄位確實與已認證使用者的密碼符合。如果密碼有效，我們需要通知 Laravel 的 Session 使用者已經確認了其密碼。`passwordConfirmed` 方法會在使用者的 Session 中設定一個時間戳記，供 Laravel 用來判斷使用者最後一次確認密碼的時間。最後，我們可以將使用者重定向至其原本想去的目的地。


<a name="password-confirmation-protecting-routes"></a>
### 保護路由

你應該確保任何執行需要近期進行密碼確認操作的路由都指派了 `password.confirm` 中介層。此中介層包含在 Laravel 的預設安裝中，它會自動將使用者原本想去的目的地儲存在 Session 中，以便使用者在確認密碼後可以被重定向至該位置。在 Session 中儲存使用者原本想去的目的地後，該中介層會將使用者重定向至 `password.confirm` [具名路由](/docs/{{version}}/routing#named-routes)：

```php
Route::get('/settings', function () {
    // ...
})->middleware(['password.confirm']);

Route::post('/settings', function () {
    // ...
})->middleware(['password.confirm']);
```


<a name="adding-custom-guards"></a>
## 新增自訂 Guard

你可以使用 `Auth` Facade 上的 `extend` 方法來定義你自己的認證 Guard。你應該將呼叫 `extend` 方法的程式碼放在[服務提供者(Service Providers)](/docs/{{version}}/providers)中。由於 Laravel 已經附帶了一個 `AppServiceProvider`，我們可以將程式碼放置於該提供者中：

```php
<?php

namespace App\Providers;

use App\Services\Auth\JwtGuard;
use Illuminate\Contracts\Foundation\Application;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    // ...

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        Auth::extend('jwt', function (Application $app, string $name, array $config) {
            // Return an instance of Illuminate\Contracts\Auth\Guard...

            return new JwtGuard(Auth::createUserProvider($config['provider']));
        });
    }
}
```

如上面的範例所示，傳遞給 `extend` 方法的回呼函式應回傳 `Illuminate\Contracts\Auth\Guard` 的實作。該介面包含幾個你需要實作的方法以定義自訂的 Guard。一旦定義了自訂 Guard，你就可以在 `auth.php` 設定檔的 `guards` 設定中引用該 Guard：

```php
'guards' => [
    'api' => [
        'driver' => 'jwt',
        'provider' => 'users',
    ],
],
```


<a name="closure-request-guards"></a>
### Closure 請求 Guard

實作基於 HTTP 請求的自訂認證系統最簡單的方法是使用 `Auth::viaRequest` 方法。這個方法允許你使用單個 Closure 來快速定義認證流程。

首先，在你應用程式的 `AppServiceProvider` 的 `boot` 方法中呼叫 `Auth::viaRequest` 方法。`viaRequest` 方法的第一個引數接受一個認證驅動名稱，這個名稱可以是任何用來描述自訂 Guard 的字串。傳遞給該方法的第二個引數應為一個 Closure，它接收傳入的 HTTP 請求並回傳一個使用者實例；如果認證失敗，則回傳 `null`：

```php
use App\Models\User;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Auth::viaRequest('custom-token', function (Request $request) {
        return User::where('token', (string) $request->token)->first();
    });
}
```

一旦定義了自訂認證驅動，你就可以在 `auth.php` 設定檔的 `guards` 設定中將其設定為驅動：

```php
'guards' => [
    'api' => [
        'driver' => 'custom-token',
    ],
],
```

最後，在將認證中介層指派給路由時，你可以引用該 Guard：

```php
Route::middleware('auth:api')->group(function () {
    // ...
});
```

<a name="adding-custom-user-providers"></a>
## 新增自訂使用者提供者

若您未採用傳統的關聯式資料庫來儲存使用者，則需要使用自訂的認證使用者提供者來擴充 Laravel。我們可以使用 `Auth` Facade 上的 `provider` 方法來定義自訂的使用者提供者。使用者提供者的解析器應回傳 `Illuminate\Contracts\Auth\UserProvider` 的實作：

```php
<?php

namespace App\Providers;

use App\Extensions\MongoUserProvider;
use Illuminate\Contracts\Foundation\Application;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    // ...

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        Auth::provider('mongo', function (Application $app, array $config) {
            // Return an instance of Illuminate\Contracts\Auth\UserProvider...

            return new MongoUserProvider($app->make('mongo.connection'));
        });
    }
}
```

在使用 `provider` 方法註冊提供者之後，您可以在 `auth.php` 設定檔中切換至新的使用者提供者。首先，定義一個使用新驅動程式的 `provider`：

```php
'providers' => [
    'users' => [
        'driver' => 'mongo',
    ],
],
```

最後，您可以在 `guards` 設定中參考此提供者：

```php
'guards' => [
    'web' => [
        'driver' => 'session',
        'provider' => 'users',
    ],
],
```


<a name="the-user-provider-contract"></a>
### User Provider 契約(Contracts)

`Illuminate\Contracts\Auth\UserProvider` 的實作負責從持久化儲存系統（例如 MySQL、MongoDB 等）中取得 `Illuminate\Contracts\Auth\Authenticatable` 的實作。這兩個介面允許 Laravel 的認證機制維持運作，無論使用者資料如何儲存，或使用何種類別來表示已認證的使用者：

讓我們來看看 `Illuminate\Contracts\Auth\UserProvider` 契約(Contracts)：

```php
<?php

namespace Illuminate\Contracts\Auth;

interface UserProvider
{
    public function retrieveById($identifier);
    public function retrieveByToken($identifier, $token);
    public function updateRememberToken(Authenticatable $user, $token);
    public function retrieveByCredentials(array $credentials);
    public function validateCredentials(Authenticatable $user, array $credentials);
    public function rehashPasswordIfRequired(Authenticatable $user, array $credentials, bool $force = false);
}
```

`retrieveById` 函式通常接收代表使用者的鍵值，例如 MySQL 資料庫的自動遞增 ID。與該 ID 匹配的 `Authenticatable` 實作應由此方法取得並回傳。

`retrieveByToken` 函式透過使用者唯一的 `$identifier` 以及「記住我」`$token`（通常儲存在如 `remember_token` 等資料庫欄位中）來取得使用者。與上一個方法相同，此方法應回傳具有匹配 token 值的 `Authenticatable` 實作。

`updateRememberToken` 方法會使用新的 `$token` 來更新 `$user` 實例的 `remember_token`。當使用者成功進行「記住我」認證或登出時，會為使用者指派一個新的 token。

`retrieveByCredentials` 方法接收在嘗試對應用程式進行認證時傳遞給 `Auth::attempt` 方法的憑證陣列。接著，該方法應該在底層持久化儲存中「查詢」符合這些憑證的使用者。通常，此方法會執行帶有 "where" 條件的查詢，搜尋 "username" 與 `$credentials['username']` 的值匹配的使用者紀錄。該方法應回傳 `Authenticatable` 的實作。**此方法不應嘗試進行任何密碼驗證或認證。**

`validateCredentials` 方法應該比較給定的 `$user` 與 `$credentials` 以認證使用者。例如，此方法通常會使用 `Hash::check` 方法來將 `$user->getAuthPassword()` 的值與 `$credentials['password']` 的值進行比較。此方法應回傳 `true` 或 `false`，以表示密碼是否有效。

`rehashPasswordIfRequired` 方法應該在需要且支援的情況下，對給定 `$user` 的密碼重新進行雜湊。例如，此方法通常會使用 `Hash::needsRehash` 方法來判斷 `$credentials['password']` 的值是否需要重新雜湊。如果密碼需要重新雜湊，該方法應使用 `Hash::make` 方法來重新雜湊密碼，並更新底層持久化儲存中的使用者紀錄。


<a name="the-authenticatable-contract"></a>
### Authenticatable 契約(Contracts)

在我們探索了 `UserProvider` 上的每個方法之後，讓我們來看看 `Authenticatable` 契約(Contracts)。請記住，使用者提供者應從 `retrieveById`、`retrieveByToken` 和 `retrieveByCredentials` 方法回傳此介面的實作：

```php
<?php

namespace Illuminate\Contracts\Auth;

interface Authenticatable
{
    public function getAuthIdentifierName();
    public function getAuthIdentifier();
    public function getAuthPasswordName();
    public function getAuthPassword();
    public function getRememberToken();
    public function setRememberToken($value);
    public function getRememberTokenName();
}
```

這個介面非常簡單。`getAuthIdentifierName` 方法應回傳使用者的 "primary key"（主鍵）欄位名稱，而 `getAuthIdentifier` 方法應回傳使用者的 "primary key"。當使用 MySQL 後端時，這通常是指指派給使用者紀錄的自動遞增主鍵。`getAuthPasswordName` 方法應回傳使用者密碼欄位的名稱。`getAuthPassword` 方法則應回傳使用者經雜湊後的密碼。

無論您使用的是何種 ORM 或儲存抽象層，此介面皆可讓認證系統與任何「使用者」類別搭配運作。預設情況下，Laravel 在 `app/Models` 目錄中包含了一個實作此介面的 `App\Models\User` 類別。


<a name="automatic-password-rehashing"></a>
## 自動密碼重新雜湊

Laravel 預設的密碼雜湊演算法為 bcrypt。Bcrypt 雜湊的「工作因子 (Work factor)」可透過應用程式的 `config/hashing.php` 設定檔或 `BCRYPT_ROUNDS` 環境變數進行調整。

通常，隨著 CPU / GPU 算力的提升，bcrypt 的工作因子應隨時間適度提高。如果您提高了應用程式的 bcrypt 工作因子，當使用者透過 Laravel 的入門套件或當您透過 `attempt` 方法[手動認證使用者](#authenticating-users)進行認證時，Laravel 會順暢且自動地重新雜湊使用者密碼。

通常，自動密碼重新雜湊不應該干擾您的應用程式；然而，您可以透過發布 `hashing` 設定檔來停用此行為：

```shell
php artisan config:publish hashing
```

發布設定檔後，您可以將 `rehash_on_login` 設定值設為 `false`：

```php
'rehash_on_login' => false,
```

<a name="events"></a>
## 事件

Laravel 在認證過程中會派發各種[事件](/docs/{{version}}/events)。您可以為以下任何事件[定義監聽器](/docs/{{version}}/events)：

<div class="overflow-auto">

| 事件名稱 |
| ---------------------------------------------- |
| `Illuminate\Auth\Events\Registered`            |
| `Illuminate\Auth\Events\Attempting`            |
| `Illuminate\Auth\Events\Authenticated`         |
| `Illuminate\Auth\Events\Login`                 |
| `Illuminate\Auth\Events\Failed`                |
| `Illuminate\Auth\Events\Validated`             |
| `Illuminate\Auth\Events\Verified`              |
| `Illuminate\Auth\Events\Logout`                |
| `Illuminate\Auth\Events\CurrentDeviceLogout`   |
| `Illuminate\Auth\Events\OtherDeviceLogout`     |
| `Illuminate\Auth\Events\Lockout`               |
| `Illuminate\Auth\Events\PasswordReset`         |
| `Illuminate\Auth\Events\PasswordResetLinkSent` |

</div>