# 認證

- [簡介](#introduction)
    - [入門套件](#starter-kits)
    - [資料庫考量](#introduction-database-considerations)
    - [生態系統概覽](#ecosystem-overview)
- [認證快速入門](#authentication-quickstart)
    - [安裝入門套件](#install-a-starter-kit)
    - [取得已認證使用者](#retrieving-the-authenticated-user)
    - [保護路由](#protecting-routes)
    - [登入節流](#login-throttling)
- [手動認證使用者](#authenticating-users)
    - [記住使用者](#remembering-users)
    - [其他認證方法](#other-authentication-methods)
- [HTTP 基本認證](#http-basic-authentication)
    - [無狀態 HTTP 基本認證](#stateless-http-basic-authentication)
- [登出](#logging-out)
    - [使其他裝置上的 Session 失效](#invalidating-sessions-on-other-devices)
- [密碼確認](#password-confirmation)
    - [設定](#password-confirmation-configuration)
    - [路由](#password-confirmation-routing)
    - [保護路由](#password-confirmation-protecting-routes)
- [新增自訂 Guards](#adding-custom-guards)
    - [閉包請求 Guards](#closure-request-guards)
- [新增自訂使用者提供者](#adding-custom-user-providers)
    - [使用者提供者契約](#the-user-provider-contract)
    - [Authenticatable 契約](#the-authenticatable-contract)
- [自動密碼再雜湊](#automatic-password-rehashing)
- [社群認證](/docs/{{version}}/socialite)
- [事件](#events)

<a name="introduction"></a>
## 簡介

許多 Web 應用程式都提供了讓使用者進行應用程式認證與「登入」的方式。在 Web 應用程式中實作此功能可能是一項複雜且潛在風險的任務。因此，Laravel 致力於提供所需的工具，讓你能快速、安全、輕鬆地實作認證。

Laravel 的認證功能核心由「Guard」與「提供者」組成。Guard 定義了使用者如何在每個請求中進行認證。舉例來說，Laravel 內建了一個 `session` Guard，它使用 Session 儲存與 Cookie 來維護狀態。

提供者定義了如何從你的持久化儲存中取得使用者。Laravel 支援使用 [Eloquent](/docs/{{version}}/eloquent) 與資料庫查詢產生器來取得使用者。然而，你可以自由定義應用程式所需的其他提供者。

你的應用程式認證設定檔位於 `config/auth.php`。此檔案包含幾個文件齊全的選項，用於調整 Laravel 認證服務的行為。

> [!NOTE]
> Guard 與提供者不應與「角色 (roles)」和「權限 (permissions)」混淆。要了解更多關於透過權限授權使用者操作的資訊，請參閱 [授權](/docs/{{version}}/authorization) 文件。


<a name="starter-kits"></a>
### 入門套件

想要快速上手嗎？在全新的 Laravel 應用程式中安裝 [Laravel 應用程式入門套件](/docs/{{version}}/starter-kits)。遷移資料庫後，在瀏覽器中導覽至 `/register` 或任何分配給應用程式的其他 URL。這些入門套件將負責為你建構整個認證系統！

**即使你選擇不在最終的 Laravel 應用程式中使用入門套件，安裝 [入門套件](/docs/{{version}}/starter-kits) 也是一個學習如何在實際 Laravel 專案中實作所有 Laravel 認證功能的絕佳機會。** 由於 Laravel 入門套件包含了認證控制器、路由和視圖，你可以檢查這些檔案中的程式碼，以了解 Laravel 的認證功能是如何實作的。


<a name="introduction-database-considerations"></a>
### 資料庫考量

依預設，Laravel 在 `app/Models` 目錄中包含了 `App\Models\User` [Eloquent model](/docs/{{version}}/eloquent)。此 Model 可以與預設的 Eloquent 認證驅動一起使用。

如果你的應用程式未使用 Eloquent，你可以使用 `database` 認證提供者，它會使用 Laravel 查詢產生器。如果你的應用程式正在使用 MongoDB，請查閱 MongoDB 官方的 [Laravel 使用者認證文件](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/user-authentication/)。

在為 `App\Models\User` Model 建構資料庫綱要時，請確保密碼欄位的長度至少為 60 個字元。當然，新的 Laravel 應用程式中包含的 `users` 資料表遷移已經建立了一個超出此長度的欄位。

此外，你應確認你的 `users` (或等效) 資料表包含一個可為空值的字串型 `remember_token` 欄位，長度為 100 個字元。此欄位將用於儲存使用者在登入應用程式時選擇「記住我」選項的 Token。同樣地，新的 Laravel 應用程式中包含的預設 `users` 資料表遷移已經包含了此欄位。

<a name="ecosystem-overview"></a>
### 生態系統概覽

Laravel 提供了幾個與認證相關的套件。在繼續之前，我們將回顧 Laravel 中的一般認證生態系統，並討論每個套件的預期用途。

首先，考慮認證是如何運作的。當使用網路瀏覽器時，使用者會透過登入表單提供其使用者名稱和密碼。如果這些憑證正確，應用程式會將已認證使用者的資訊儲存在使用者的 [session](/docs/{{version}}/session) 中。發給瀏覽器的 cookie 包含 session ID，以便後續對應用程式的請求可以將使用者與正確的 session 關聯起來。在接收到 session cookie 後，應用程式會根據 session ID 檢索 session 資料，注意到認證資訊已儲存在 session 中，並將該使用者視為「已認證」。

當遠端服務需要認證以存取 API 時，通常不會使用 cookie 進行認證，因為沒有網路瀏覽器。相反地，遠端服務會在每個請求中向 API 發送一個 API token。應用程式可以根據有效 API token 的資料表來驗證傳入的 token，並將該請求「認證」為由與該 API token 關聯的使用者所執行。

<a name="laravels-built-in-browser-authentication-services"></a>
#### Laravel 內建的瀏覽器認證服務

Laravel 包含了內建的認證和 session 服務，通常透過 `Auth` 和 `Session` Facades 存取。這些功能為從網路瀏覽器發起的請求提供基於 cookie 的認證。它們提供了允許您驗證使用者憑證和認證使用者的方法。此外，這些服務會自動將適當的認證資料儲存在使用者的 session 中，並發出使用者的 session cookie。本文件中包含了如何使用這些服務的討論。

**應用程式入門套件**

如本文件所討論，您可以手動與這些認證服務互動，以建立您應用程式自己的認證層。然而，為了幫助您更快上手，我們發布了 [免費入門套件](/docs/{{version}}/starter-kits)，這些套件提供了整個認證層強大且現代化的鷹架。

<a name="laravels-api-authentication-services"></a>
#### Laravel 的 API 認證服務

Laravel 提供了兩個可選套件，可協助您管理 API token 並認證使用 API token 發出的請求：[Passport](/docs/{{version}}/passport) 和 [Sanctum](/docs/{{version}}/sanctum)。請注意，這些函式庫與 Laravel 內建的基於 cookie 的認證函式庫並非互斥。這些函式庫主要專注於 API token 認證，而內建的認證服務則專注於基於 cookie 的瀏覽器認證。許多應用程式會同時使用 Laravel 內建的基於 cookie 的認證服務和其中一個 Laravel API 認證套件。

**Passport**

Passport 是一個 OAuth2 認證提供者，提供了多種 OAuth2「授權類型 (grant types)」，允許您發行各種 token。一般而言，這是一個功能強大且複雜的 API 認證套件。然而，大多數應用程式不需要 OAuth2 規範提供的複雜功能，這對使用者和開發人員來說都可能造成困惑。此外，開發人員一直以來都對如何使用像 Passport 這樣的 OAuth2 認證提供者來認證 SPA 應用程式或行動應用程式感到困惑。

**Sanctum**

為了解決 OAuth2 的複雜性和開發人員的困惑，我們著手建立一個更簡單、更精簡的認證套件，該套件可以同時處理來自網路瀏覽器的第一方 Web 請求和透過 token 的 API 請求。這一目標隨著 [Laravel Sanctum](/docs/{{version}}/sanctum) 的發布而實現，它應被視為首選和推薦的認證套件，適用於將提供第一方 Web UI 以及 API 的應用程式，或者將由獨立於後端 Laravel 應用程式的單頁應用程式 (SPA) 提供支援的應用程式，或者提供行動用戶端的應用程式。

Laravel Sanctum 是一個混合式的 Web / API 認證套件，可以管理您應用程式的整個認證過程。這是因為當基於 Sanctum 的應用程式收到請求時，Sanctum 會首先判斷請求是否包含引用已認證 session 的 session cookie。Sanctum 透過呼叫我們前面討論過的 Laravel 內建認證服務來實現這一點。如果請求沒有透過 session cookie 進行認證，Sanctum 將檢查請求中是否有 API token。如果存在 API token，Sanctum 將使用該 token 認證請求。要了解有關此過程的更多資訊，請查閱 Sanctum 的 [「運作方式」](/docs/{{version}}/sanctum#how-it-works) 文件。

<a name="summary-choosing-your-stack"></a>
#### 總結與選擇您的堆疊

總而言之，如果您的應用程式將透過瀏覽器存取，並且您正在建立一個單體式 Laravel 應用程式，您的應用程式將使用 Laravel 內建的認證服務。

接下來，如果您的應用程式提供將由第三方消耗的 API，您將在 [Passport](/docs/{{version}}/passport) 或 [Sanctum](/docs/{{version}}/sanctum) 之間進行選擇，為您的應用程式提供 API token 認證。一般而言，在可能的情況下應優先選擇 Sanctum，因為它是一個簡單、完整的解決方案，適用於 API 認證、SPA 認證和行動認證，包括對「scopes」或「abilities」的支援。

如果您正在建立一個將由 Laravel 後端提供支援的單頁應用程式 (SPA)，您應該使用 [Laravel Sanctum](/docs/{{version}}/sanctum)。使用 Sanctum 時，您需要 [手動實作自己的後端認證路由](#authenticating-users) 或使用 [Laravel Fortify](/docs/{{version}}/fortify) 作為無頭認證後端服務，它提供了註冊、密碼重設、電子郵件驗證等功能的路由和控制器。

當您的應用程式絕對需要 OAuth2 規範提供的所有功能時，可以選擇 Passport。

而且，如果您想快速上手，我們很高興推薦 [我們的應用程式入門套件](/docs/{{version}}/starter-kits) 作為啟動一個已使用我們首選的 Laravel 內建認證服務認證堆疊的 Laravel 新應用程式的快速方式。

<a name="authentication-quickstart"></a>
## 認證快速入門

> [!WARNING]
> 文件中的這個部分討論透過 [Laravel 應用程式入門套件](/docs/{{version}}/starter-kits)認證使用者，此套件包含 UI 鷹架以協助您快速開始。如果您想直接與 Laravel 的認證系統整合，請查看關於[手動認證使用者](#authenticating-users)的文件。

<a name="install-a-starter-kit"></a>
### 安裝入門套件

首先，您應該[安裝一個 Laravel 應用程式入門套件](/docs/{{version}}/starter-kits)。我們的入門套件為將認證整合到您的新的 Laravel 應用程式中提供了精美設計的起點。

<a name="retrieving-the-authenticated-user"></a>
### 取得已認證使用者

從入門套件建立應用程式並允許使用者註冊並認證您的應用程式後，您通常需要與目前已認證的使用者互動。處理傳入請求時，您可以透過 `Auth` Facade 的 `user` 方法存取已認證的使用者：

```php
use Illuminate\Support\Facades\Auth;

// Retrieve the currently authenticated user...
$user = Auth::user();

// Retrieve the currently authenticated user's ID...
$id = Auth::id();
```

或者，一旦使用者已認證，您可以透過 `Illuminate\Http\Request` 實例存取已認證的使用者。請記住，型別提示的類別會自動注入到您的控制器方法中。透過型別提示 `Illuminate\Http\Request` 物件，您可以透過請求的 `user` 方法方便地從應用程式中的任何控制器方法存取已認證的使用者：

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
#### 判斷目前使用者是否已認證

要判斷發出傳入 HTTP 請求的使用者是否已認證，您可以使用 `Auth` Facade 的 `check` 方法。如果使用者已認證，此方法將回傳 `true`：

```php
use Illuminate\Support\Facades\Auth;

if (Auth::check()) {
    // The user is logged in...
}
```

> [!NOTE]
> 即使可以使用 `check` 方法來判斷使用者是否已認證，您通常會使用中介層來驗證使用者是否已認證，然後才允許使用者存取特定路由 / 控制器。要了解更多資訊，請查看關於[保護路由](/docs/{{version}}/authentication#protecting-routes)的文件。

<a name="protecting-routes"></a>
### 保護路由

[路由中介層](/docs/{{version}}/middleware)可用於只允許已認證的使用者存取給定的路由。Laravel 內建 `auth` 中介層，它是 `Illuminate\Auth\Middleware\Authenticate` 類別的[中介層別名](/docs/{{version}}/middleware#middleware-aliases)。由於此中介層已由 Laravel 在內部建立別名，因此您只需將中介層附加到路由定義中即可：

```php
Route::get('/flights', function () {
    // Only authenticated users may access this route...
})->middleware('auth');
```

<a name="redirecting-unauthenticated-users"></a>
#### 重新導向未認證使用者

當 `auth` 中介層偵測到未認證的使用者時，它會將使用者重新導向到 `login` [具名路由](/docs/{{version}}/routing#named-routes)。您可以使用應用程式 `bootstrap/app.php` 檔案中的 `redirectGuestsTo` 方法來修改此行為：

```php
use Illuminate\Http\Request;

->withMiddleware(function (Middleware $middleware): void {
    $middleware->redirectGuestsTo('/login');

    // Using a closure...
    $middleware->redirectGuestsTo(fn (Request $request) => route('login'));
})
```

<a name="redirecting-authenticated-users"></a>
#### 重新導向已認證使用者

當 `guest` 中介層偵測到已認證的使用者時，它會將使用者重新導向到 `dashboard` 或 `home` 具名路由。您可以使用應用程式 `bootstrap/app.php` 檔案中的 `redirectUsersTo` 方法來修改此行為：

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

將 `auth` 中介層附加到路由時，您還可以指定應使用哪個「guard」來認證使用者。指定的 guard 應與您 `auth.php` 設定檔中 `guards` 陣列中的其中一個鍵相對應：

```php
Route::get('/flights', function () {
    // Only authenticated users may access this route...
})->middleware('auth:admin');
```

<a name="login-throttling"></a>
### 登入節流

如果您正在使用我們的[應用程式入門套件](/docs/{{version}}/starter-kits)之一，速率限制將自動應用於登入嘗試。預設情況下，如果使用者在幾次嘗試後未能提供正確的憑證，他們將無法登入一分鐘。節流是針對使用者的使用者名稱 / 電子郵件地址及其 IP 位址進行的。

> [!NOTE]
> 如果您想限制應用程式中其他路由的速率，請查看[速率限制文件](/docs/{{version}}/routing#rate-limiting)。

<a name="authenticating-users"></a>
## 手動認證使用者

您不一定要使用 Laravel [應用程式入門套件](/docs/{{version}}/starter-kits)中包含的認證鷹架。如果您選擇不使用此鷹架，您將需要直接使用 Laravel 認證類別來管理使用者認證。別擔心，這很簡單！

我們將透過 `Auth` [Facade](/docs/{{version}}/facades) 存取 Laravel 的認證服務，因此我們需要確保在類別頂部匯入 `Auth` Facade。接下來，讓我們看看 `attempt` 方法。`attempt` 方法通常用於處理應用程式「登入」表單的認證嘗試。如果認證成功，您應該重新產生使用者的 [Session](/docs/{{version}}/session) 以防止 [Session 劫持](https://en.wikipedia.org/wiki/Session_fixation)：

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

`attempt` 方法接受鍵/值對的陣列作為其第一個參數。陣列中的值將用於在您的資料庫表中尋找使用者。因此，在上面的範例中，使用者將透過 `email` 欄位的值取得。如果找到使用者，資料庫中儲存的雜湊密碼將與透過陣列傳遞給方法的 `password` 值進行比較。您不應該對傳入請求的 `password` 值進行雜湊，因為框架會自動雜湊該值，然後再將其與資料庫中的雜湊密碼進行比較。如果兩個雜湊密碼匹配，將為使用者啟動一個已認證的 Session。

請記住，Laravel 的認證服務將根據您認證 guard 的「provider」設定從資料庫中取得使用者。在預設的 `config/auth.php` 設定檔中，指定了 Eloquent 使用者提供者，並指示其在取得使用者時使用 `App\Models\User` 模型。您可以根據應用程式的需求在設定檔中更改這些值。

如果認證成功，`attempt` 方法將會回傳 `true`。否則，將會回傳 `false`。

Laravel 重新導向器提供的 `intended` 方法將使用者重新導向到他們原本試圖存取的 URL，然後才被認證中介層攔截。如果預期的目的地不可用，可以為此方法提供一個備用 URI。

<a name="specifying-additional-conditions"></a>
#### 指定額外條件

如果您願意，除了使用者的 email 和密碼之外，還可以在認證查詢中添加額外的查詢條件。為此，我們只需將查詢條件添加到傳遞給 `attempt` 方法的陣列中。例如，我們可以驗證使用者是否被標記為「啟用」：

```php
if (Auth::attempt(['email' => $email, 'password' => $password, 'active' => 1])) {
    // Authentication was successful...
}
```

對於複雜的查詢條件，您可以在憑證陣列中提供一個閉包。此閉包將會與查詢實例一同被呼叫，讓您可以根據應用程式的需求自訂查詢：

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
> 在這些範例中，`email` 並非必要選項，它僅作為一個範例。您應該使用資料庫表中對應「使用者名稱」的任何欄位名稱。

`attemptWhen` 方法（它接受一個閉包作為其第二個參數）可用於在實際認證使用者之前，對潛在使用者執行更廣泛的檢查。該閉包會接收潛在使用者，並且應該回傳 `true` 或 `false` 以指示使用者是否可以被認證：

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

透過 `Auth` Facade 的 `guard` 方法，您可以指定在認證使用者時要使用的 guard 實例。這讓您可以透過使用完全獨立的 Authenticatable 模型或使用者資料表，來管理應用程式不同部分的認證。

傳遞給 `guard` 方法的 guard 名稱應該與您 `auth.php` 設定檔中配置的 guard 之一相對應：

```php
if (Auth::guard('admin')->attempt($credentials)) {
    // ...
}
```

<a name="remembering-users"></a>
### 記住使用者

許多網路應用程式在其登入表單上提供「記住我」的核取方塊。如果您想在應用程式中提供「記住我」功能，您可以將布林值作為第二個參數傳遞給 `attempt` 方法。

當此值為 `true` 時，Laravel 將會無限期地保持使用者已認證狀態，或直到他們手動登出為止。您的 `users` 資料表必須包含字串 `remember_token` 欄位，它將用於儲存「記住我」Token。新 Laravel 應用程式中包含的 `users` 資料表遷移已經包含此欄位：

```php
use Illuminate\Support\Facades\Auth;

if (Auth::attempt(['email' => $email, 'password' => $password], $remember)) {
    // The user is being remembered...
}
```

如果您的應用程式提供「記住我」功能，您可以使用 `viaRemember` 方法來判斷目前已認證的使用者是否是透過「記住我」cookie 進行認證的：

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

如果您需要將現有的使用者實例設定為目前已認證的使用者，您可以將使用者實例傳遞給 `Auth` facade 的 `login` 方法。傳入的使用者實例必須是 `Illuminate\Contracts\Auth\Authenticatable` [契約](/docs/{{version}}/contracts)的實作。Laravel 內建的 `App\Models\User` 模型已實作此介面。這種認證方法在您已有有效的使用者實例時非常有用，例如使用者向您的應用程式註冊後：

```php
use Illuminate\Support\Facades\Auth;

Auth::login($user);
```

您可以將布林值作為第二個參數傳遞給 `login` 方法。此值指示是否需要為已認證的 Session 提供「記住我」功能。請記住，這意味著 Session 將會無限期地保持認證狀態，或者直到使用者手動登出應用程式為止：

```php
Auth::login($user, $remember = true);
```

如有需要，您可以在呼叫 `login` 方法之前指定認證 Guard：

```php
Auth::guard('admin')->login($user);
```


<a name="authenticate-a-user-by-id"></a>
#### 透過 ID 認證使用者

若要使用使用者的資料庫紀錄主鍵來認證使用者，您可以使用 `loginUsingId` 方法。此方法接受您希望認證的使用者主鍵：

```php
Auth::loginUsingId(1);
```

您可以將布林值傳遞給 `loginUsingId` 方法的 `remember` 參數。此值指示是否需要為已認證的 Session 提供「記住我」功能。請記住，這意味著 Session 將會無限期地保持認證狀態，或者直到使用者手動登出應用程式為止：

```php
Auth::loginUsingId(1, remember: true);
```


<a name="authenticate-a-user-once"></a>
#### 僅認證一次使用者

您可以使用 `once` 方法來讓應用程式僅針對單一請求認證使用者。呼叫此方法時，不會使用任何 Session 或 Cookie，並且 `Login` 事件將不會被觸發：

```php
if (Auth::once($credentials)) {
    // ...
}
```

<a name="http-basic-authentication"></a>
## HTTP 基本認證

[HTTP 基本認證](https://en.wikipedia.org/wiki/Basic_access_authentication) 提供了一種快速的方式來認證您應用程式的使用者，而無需設置專門的「登入」頁面。要開始使用，請將 `auth.basic` [中介層](/docs/{{version}}/middleware) 附加到路由。Laravel 框架中已包含 `auth.basic` 中介層，因此您無需自行定義：

```php
Route::get('/profile', function () {
    // Only authenticated users may access this route...
})->middleware('auth.basic');
```

一旦將中介層附加到路由，當您在瀏覽器中存取該路由時，系統將自動提示您輸入憑證。預設情況下，`auth.basic` 中介層會假定您 `users` 資料庫表中的 `email` 欄位是使用者的「使用者名稱」。

<a name="a-note-on-fastcgi"></a>
#### 關於 FastCGI 的注意事項

如果您使用 [PHP FastCGI](https://www.php.net/manual/en/install.fpm.php) 和 Apache 來提供您的 Laravel 應用程式服務，HTTP 基本認證可能無法正常運作。為了修正這些問題，您可以將以下幾行加入到應用程式的 `.htaccess` 檔案中：

```apache
RewriteCond %{HTTP:Authorization} ^(.+)$
RewriteRule .* - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization}]
```

<a name="stateless-http-basic-authentication"></a>
### 無狀態 HTTP 基本認證

您也可以使用 HTTP 基本認證，而無需在 Session 中設置使用者識別碼 Cookie。如果您選擇使用 HTTP 認證來驗證對應用程式 API 的請求，這會特別有用。為此，請[定義一個中介層](/docs/{{version}}/middleware)，該中介層會呼叫 `onceBasic` 方法。如果 `onceBasic` 方法未返回任何回應，則請求可能會被進一步傳遞給應用程式：

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

接著，將中介層附加到路由：

```php
Route::get('/api/user', function () {
    // Only authenticated users may access this route...
})->middleware(AuthenticateOnceWithBasicAuth::class);
```

<a name="logging-out"></a>
## 登出

要手動將使用者從您的應用程式登出，您可以使用 `Auth` Facade 提供的 `logout` 方法。這將從使用者的 Session 中移除認證資訊，以確保後續請求不會被認證。

除了呼叫 `logout` 方法外，建議您使使用者的 Session 失效並重新生成其 [CSRF Token](/docs/{{version}}/csrf)。在使用者登出後，您通常會將使用者重新導向到您應用程式的根目錄：

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
### 使其他裝置上的 Session 失效

Laravel 也提供了一種機制，可以使使用者在其他裝置上活躍的 Session 失效並「登出」，而不會使其目前裝置上的 Session 失效。此功能通常在使用者更改或更新密碼時使用，此時您希望使其他裝置上的 Session 失效，同時保持目前裝置的認證狀態。

在開始之前，您應確保在需要接收 Session 認證的路由上包含 `Illuminate\Session\Middleware\AuthenticateSession` 中介層。通常，您應該將此中介層放在路由群組定義中，以便它可以應用於您應用程式的大部分路由。預設情況下，`AuthenticateSession` 中介層可以使用 `auth.session` [中介層別名](/docs/{{version}}/middleware#middleware-aliases) 附加到路由：

```php
Route::middleware(['auth', 'auth.session'])->group(function () {
    Route::get('/', function () {
        // ...
    });
});
```

接著，您可以使用 `Auth` Facade 提供的 `logoutOtherDevices` 方法。此方法要求使用者確認其目前密碼，您的應用程式應透過輸入表單接受此密碼：

```php
use Illuminate\Support\Facades\Auth;

Auth::logoutOtherDevices($currentPassword);
```

當呼叫 `logoutOtherDevices` 方法時，使用者的其他 Session 將完全失效，這表示他們將從所有先前認證的 Guard 中「登出」。

<a name="password-confirmation"></a>
## 密碼確認

在建構應用程式時，您偶爾會遇到某些操作需要使用者在執行該操作之前，或在使用者被重新導向到應用程式的敏感區域之前，先確認其密碼。Laravel 包含了內建的中介層 (middleware) 來簡化此流程。實作此功能需要您定義兩個路由 (route)：一個路由用於顯示要求使用者確認密碼的視圖 (view)，另一個路由則用於確認密碼有效並將使用者重新導向至其預期目的地。

> [!NOTE]
> 以下文件說明如何直接整合 Laravel 的密碼確認功能；然而，如果您想更快開始，[Laravel 應用程式入門套件](/docs/{{version}}/starter-kits)已包含對此功能的支援！

<a name="password-confirmation-configuration"></a>
### 設定

使用者確認密碼後，三小時內不會再被要求確認密碼。然而，您可以透過修改應用程式 `config/auth.php` 設定檔中的 `password_timeout` 設定值，來配置使用者再次被提示輸入密碼前的時間長度。

<a name="password-confirmation-routing"></a>
### 路由

<a name="the-password-confirmation-form"></a>
#### 密碼確認表單

首先，我們將定義一個路由，用於顯示要求使用者確認密碼的視圖：

```php
Route::get('/confirm-password', function () {
    return view('auth.confirm-password');
})->middleware('auth')->name('password.confirm');
```

如您所料，此路由返回的視圖應包含一個帶有 `password` 欄位的表單。此外，請隨意在視圖中加入文字，說明使用者正在進入應用程式的受保護區域，並且必須確認其密碼。

<a name="confirming-the-password"></a>
#### 確認密碼

接著，我們將定義一個路由，用於處理來自「確認密碼」視圖的表單請求。此路由將負責驗證密碼並將使用者重新導向至其預期目的地：

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

在繼續之前，讓我們更詳細地檢視此路由。首先，會判斷請求的 `password` 欄位是否確實與已認證使用者的密碼相符。如果密碼有效，我們需要通知 Laravel 的 session 使用者已確認其密碼。`passwordConfirmed` 方法將在使用者 session 中設定一個時間戳記 (timestamp)，Laravel 可以用它來判斷使用者上次確認密碼的時間。最後，我們可以將使用者重新導向至其預期目的地。

<a name="password-confirmation-protecting-routes"></a>
### 保護路由

您應確保任何執行需要近期密碼確認之操作的路由都分配了 `password.confirm` 中介層。此中介層隨 Laravel 預設安裝而包含，它會自動將使用者的預期目的地儲存在 session 中，以便使用者在確認密碼後可以被重新導向到該位置。在 session 中儲存使用者的預期目的地後，該中介層會將使用者重新導向到 `password.confirm` [命名路由](/docs/{{version}}/routing#named-routes)：

```php
Route::get('/settings', function () {
    // ...
})->middleware(['password.confirm']);

Route::post('/settings', function () {
    // ...
})->middleware(['password.confirm']);
```

<a name="adding-custom-guards"></a>
## 新增自訂 Guards

您可以使用 `Auth` Facade 上的 `extend` 方法來定義自己的認證 Guards。您應該將對 `extend` 方法的呼叫放在 [service provider](/docs/{{version}}/providers) 中。由於 Laravel 已內建 `AppServiceProvider`，我們可以將程式碼放在該 provider 中：

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

正如您在上面的範例中所見，傳遞給 `extend` 方法的回調 (callback) 應返回一個 `Illuminate\Contracts\Auth\Guard` 的實作 (implementation)。此介面 (interface) 包含一些您需要實作的方法，以定義自訂的 guard。一旦您的自訂 guard 被定義後，您可以在 `auth.php` 設定檔的 `guards` 設定中引用該 guard：

```php
'guards' => [
    'api' => [
        'driver' => 'jwt',
        'provider' => 'users',
    ],
],
```

<a name="closure-request-guards"></a>
### 閉包請求 Guards

實作自訂、基於 HTTP 請求的認證系統最簡單的方法是使用 `Auth::viaRequest` 方法。此方法讓您可以使用單一閉包 (closure) 快速定義您的認證流程。

首先，在應用程式 `AppServiceProvider` 的 `boot` 方法中呼叫 `Auth::viaRequest` 方法。`viaRequest` 方法接受一個認證驅動器名稱作為其第一個參數。此名稱可以是任何描述您自訂 guard 的字串。傳遞給方法的第二個參數應該是一個閉包，它接收傳入的 HTTP 請求並返回一個使用者實例 (user instance)，如果認證失敗，則返回 `null`：

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

定義好您的自訂認證驅動器後，您可以在 `auth.php` 設定檔的 `guards` 設定中將其配置為驅動器：

```php
'guards' => [
    'api' => [
        'driver' => 'custom-token',
    ],
],
```

最後，您可以在將認證中介層分配給路由時引用該 guard：

```php
Route::middleware('auth:api')->group(function () {
    // ...
});
```

<a name="adding-custom-user-providers"></a>
## 新增自訂使用者提供者

如果您沒有使用傳統的關聯式資料庫來儲存使用者，您將需要使用您自己的認證使用者提供者來擴展 Laravel。我們將使用 `Auth` Facade 上的 `provider` 方法來定義一個自訂使用者提供者。使用者提供者解析器應回傳 `Illuminate\Contracts\Auth\UserProvider` 的實作：

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

在您使用 `provider` 方法註冊提供者後，您可以在 `auth.php` 設定檔中切換到新的使用者提供者。首先，定義一個使用您新驅動程式的 `provider`：

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
### 使用者提供者契約

`Illuminate\Contracts\Auth\UserProvider` 的實作負責從持久儲存系統（例如 MySQL、MongoDB 等）中獲取 `Illuminate\Contracts\Auth\Authenticatable` 的實作。這兩個介面讓 Laravel 的認證機制能夠持續運作，而無論使用者資料是如何儲存的，也無論使用哪種類型的類別來表示已認證使用者：

讓我們來看看 `Illuminate\Contracts\Auth\UserProvider` 契約：

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

`retrieveById` 函數通常會接收代表使用者的鍵，例如來自 MySQL 資料庫的自動遞增 ID。與該 ID 匹配的 `Authenticatable` 實作應由該方法擷取並回傳。

`retrieveByToken` 函數根據其唯一的 `$identifier` 和「記住我」的 `$token` 來擷取使用者，這些通常儲存在像 `remember_token` 這樣的資料庫欄位中。與前一個方法一樣，應由該方法回傳具有匹配權杖值的 `Authenticatable` 實作。

`updateRememberToken` 方法會使用新的 `$token` 更新 `$user` 實例的 `remember_token`。在成功「記住我」認證嘗試或使用者登出時，會為使用者指派一個新的權杖。

`retrieveByCredentials` 方法接收傳遞給 `Auth::attempt` 方法的憑證陣列，該方法用於嘗試認證應用程式。該方法應接著「查詢」底層持久儲存中與這些憑證匹配的使用者。通常，此方法會執行一個帶有「where」條件的查詢，該條件會搜尋具有與 `$credentials['username']` 值匹配的「使用者名稱」的使用者記錄。該方法應回傳 `Authenticatable` 的實作。**此方法不應嘗試執行任何密碼驗證或認證。**

`validateCredentials` 方法應將給定的 `$user` 與 `$credentials` 進行比較以認證使用者。例如，此方法通常會使用 `Hash::check` 方法來比較 `$user->getAuthPassword()` 的值與 `$credentials['password']` 的值。此方法應回傳 `true` 或 `false`，表示密碼是否有效。

`rehashPasswordIfRequired` 方法應在需要且支援的情況下重新雜湊給定 `$user` 的密碼。例如，此方法通常會使用 `Hash::needsRehash` 方法來判斷 `$credentials['password']` 的值是否需要重新雜湊。如果密碼需要重新雜湊，該方法應使用 `Hash::make` 方法來重新雜湊密碼，並更新使用者在底層持久儲存中的記錄。

<a name="the-authenticatable-contract"></a>
### Authenticatable 契約

現在我們已經探索了 `UserProvider` 上的每個方法，讓我們來看看 `Authenticatable` 契約。請記住，使用者提供者應從 `retrieveById`、`retrieveByToken` 和 `retrieveByCredentials` 方法中回傳此介面的實作：

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

這個介面很簡單。`getAuthIdentifierName` 方法應回傳使用者「主鍵」欄位的名稱，而 `getAuthIdentifier` 方法應回傳使用者「主鍵」的值。在使用 MySQL 後端時，這很可能是指派給使用者記錄的自動遞增主鍵。`getAuthPasswordName` 方法應回傳使用者密碼欄位的名稱。`getAuthPassword` 方法應回傳使用者的雜湊密碼。

此介面允許認證系統與任何「使用者」類別協同工作，無論您使用何種 ORM 或儲存抽象層。預設情況下，Laravel 在 `app/Models` 目錄中包含一個 `App\Models\User` 類別，該類別已實作此介面。

<a name="automatic-password-rehashing"></a>
## 自動密碼再雜湊

Laravel 的預設密碼雜湊演算法是 bcrypt。bcrypt 雜湊的「工作因子」可以透過應用程式的 `config/hashing.php` 設定檔或 `BCRYPT_ROUNDS` 環境變數進行調整。

通常，隨著 CPU / GPU 處理能力的提高，bcrypt 的工作因子應隨著時間增加。如果您增加應用程式的 bcrypt 工作因子，Laravel 將在使用者透過 Laravel 的入門套件或當您透過 `attempt` 方法[手動認證使用者](#authenticating-users)時，優雅地自動重新雜湊使用者密碼。

通常，自動密碼再雜湊不應中斷您的應用程式；但是，您可以透過發布 `hashing` 設定檔來禁用此行為：

```shell
php artisan config:publish hashing
```

一旦設定檔發布，您可以將 `rehash_on_login` 設定值設定為 `false`：

```php
'rehash_on_login' => false,
```

<a name="events"></a>
## 事件

Laravel 在認證過程中會分派各種 [事件](/docs/{{version}}/events)。您可以針對以下任何事件[定義監聽器](/docs/{{version}}/events)：

<div class="overflow-auto">

| 事件名稱                                     |
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