# 快速入門套件

- [簡介](#introduction)
- [使用快速入門套件建立應用程式](#creating-an-application)
- [可用的快速入門套件](#available-starter-kits)
    - [React](#react)
    - [Vue](#vue)
    - [Livewire](#livewire)
- [快速入門套件客製化](#starter-kit-customization)
    - [React](#react-customization)
    - [Vue](#vue-customization)
    - [Livewire](#livewire-customization)
- [身份驗證](#authentication)
    - [啟用與停用功能](#enabling-and-disabling-features)
    - [客製化使用者建立與密碼重設](#customizing-actions)
    - [雙重身份驗證](#two-factor-authentication)
    - [頻率限制](#rate-limiting)
- [WorkOS AuthKit 身份驗證](#workos)
- [Inertia SSR](#inertia-ssr)
- [社群維護的快速入門套件](#community-maintained-starter-kits)
- [常見問題](#faqs)

<a name="introduction"></a>
## 簡介

為了讓您在建立新的 Laravel 應用程式時能搶先一步，我們很高興提供[應用程式快速入門套件](https://laravel.com/starter-kits)。這些快速入門套件讓您能夠搶先開始建立您的下一個 Laravel 應用程式，其中包含您註冊與驗證應用程式使用者所需的路由、控制器與視圖。這些快速入門套件使用 [Laravel Fortify](/docs/{{version}}/fortify) 來提供身份驗證功能。

雖然歡迎您使用這些快速入門套件，但它們並非強制性。您可以透過簡單地安裝全新的 Laravel 應用程式，從頭開始建立自己的應用程式。無論哪種方式，我們相信您都能建立出色的作品！

<a name="creating-an-application"></a>
## 使用快速入門套件建立應用程式

若要使用我們的快速入門套件之一建立新的 Laravel 應用程式，您應該首先[安裝 PHP 和 Laravel CLI 工具](/docs/{{version}}/installation#installing-php)。若您已安裝 PHP 和 Composer，則可透過 Composer 安裝 Laravel 安裝器 CLI 工具：

```shell
composer global require laravel/installer
```

接著，使用 Laravel 安裝器 CLI 建立一個新的 Laravel 應用程式。Laravel 安裝器將提示您選擇首選的快速入門套件：

```shell
laravel new my-app
```

建立您的 Laravel 應用程式後，您只需要透過 NPM 安裝其前端依賴項，並啟動 Laravel 開發伺服器：

```shell
cd my-app
npm install && npm run build
composer run dev
```

啟動 Laravel 開發伺服器後，您的應用程式將可在瀏覽器中透過 [http://localhost:8000](http://localhost:8000) 存取。

<a name="available-starter-kits"></a>
## 可用的快速入門套件

<a name="react"></a>
### React

我們的 React 快速入門套件為使用 [Inertia](https://inertiajs.com) 搭配 React 前端來建構 Laravel 應用程式，提供了強健且現代化的起點。

Inertia 讓您能夠使用傳統的伺服器端路由與控制器來建構現代化的單頁式 React 應用程式。這讓您可以享受 React 的前端強大功能，並結合 Laravel 令人難以置信的後端生產力，以及閃電般快速的 Vite 編譯。

React 快速入門套件使用 React 19、TypeScript、Tailwind 以及 [shadcn/ui](https://ui.shadcn.com) 組件函式庫。

<a name="vue"></a>
### Vue

我們的 Vue 快速入門套件為使用 [Inertia](https://inertiajs.com) 搭配 Vue 前端來建構 Laravel 應用程式，提供了絕佳的起點。

Inertia 讓您能夠使用傳統的伺服器端路由與控制器來建構現代化的單頁式 Vue 應用程式。這讓您可以享受 Vue 的前端強大功能，並結合 Laravel 令人難以置信的後端生產力，以及閃電般快速的 Vite 編譯。

Vue 快速入門套件使用 Vue Composition API、TypeScript、Tailwind 以及 [shadcn-vue](https://www.shadcn-vue.com/) 組件函式庫。

<a name="livewire"></a>
### Livewire

我們的 Livewire 快速入門套件為使用 [Laravel Livewire](https://livewire.laravel.com) 前端來建構 Laravel 應用程式，提供了完美的起點。

Livewire 是一種強大的方式，僅使用 PHP 即可建構動態、響應式的前端使用者介面 (UI)。對於主要使用 Blade 模板，並正在尋找比 React 和 Vue 等 JavaScript 驅動的 SPA 框架更簡單替代方案的團隊來說，它是一個絕佳的選擇。

Livewire 快速入門套件使用 Livewire、Tailwind 以及 [Flux UI](https://fluxui.dev) 組件函式庫。

<a name="starter-kit-customization"></a>
## 快速入門套件客製化


<a name="react-customization"></a>
### React

我們的 React 快速入門套件使用 Inertia 2, React 19, Tailwind 4, 和 [shadcn/ui](https://ui.shadcn.com) 建構。與所有我們的快速入門套件一樣，所有的後端與前端程式碼都存在於您的應用程式中，以實現完全客製化。

大部分前端程式碼位於 `resources/js` 目錄中。您可以自由修改任何程式碼，以客製化您應用程式的外觀與行為：

```text
resources/js/
├── components/    # Reusable React components
├── hooks/         # React hooks
├── layouts/       # Application layouts
├── lib/           # Utility functions and configuration
├── pages/         # Page components
└── types/         # TypeScript definitions
```

若要發佈額外的 shadcn 元件，請先[找到您要發佈的元件](https://ui.shadcn.com)。然後，使用 `npx` 發佈該元件：

```shell
npx shadcn@latest add switch
```

在此範例中，該指令會將 Switch 元件發佈到 `resources/js/components/ui/switch.tsx`。一旦元件發佈成功，您就可以在任何頁面中使用它：

```jsx
import { Switch } from "@/components/ui/switch"

const MyPage = () => {
  return (
    <div>
      <Switch />
    </div>
  );
};

export default MyPage;
```


<a name="react-available-layouts"></a>
#### 可用佈局

React 快速入門套件包含兩種主要佈局供您選擇：「側邊欄」佈局和「標頭」佈局。側邊欄佈局是預設值，但您可以透過修改應用程式 `resources/js/layouts/app-layout.tsx` 檔案頂部匯入的佈局來切換到標頭佈局：

```js
import AppLayoutTemplate from '@/layouts/app/app-sidebar-layout'; // [tl! remove]
import AppLayoutTemplate from '@/layouts/app/app-header-layout'; // [tl! add]
```


<a name="react-sidebar-variants"></a>
#### 側邊欄變體

側邊欄佈局包含三種不同的變體：預設側邊欄變體、「內嵌 (inset)」變體和「浮動 (floating)」變體。您可以透過修改 `resources/js/components/app-sidebar.tsx` 元件來選擇您最喜歡的變體：

```text
<Sidebar collapsible="icon" variant="sidebar"> [tl! remove]
<Sidebar collapsible="icon" variant="inset"> [tl! add]
```


<a name="react-authentication-page-layout-variants"></a>
#### 身份驗證頁面佈局變體

React 快速入門套件中包含的身份驗證頁面，例如登入頁面和註冊頁面，也提供三種不同的佈局變體：「簡潔 (simple)」、「卡片 (card)」和「分割 (split)」。

若要更改您的身份驗證佈局，請修改應用程式 `resources/js/layouts/auth-layout.tsx` 檔案頂部匯入的佈局：

```js
import AuthLayoutTemplate from '@/layouts/auth/auth-simple-layout'; // [tl! remove]
import AuthLayoutTemplate from '@/layouts/auth/auth-split-layout'; // [tl! add]
```


<a name="vue-customization"></a>
### Vue

我們的 Vue 快速入門套件使用 Inertia 2, Vue 3 Composition API, Tailwind, 和 [shadcn-vue](https://www.shadcn-vue.com/) 建構。與所有我們的快速入門套件一樣，所有的後端與前端程式碼都存在於您的應用程式中，以實現完全客製化。

大部分前端程式碼位於 `resources/js` 目錄中。您可以自由修改任何程式碼，以客製化您應用程式的外觀與行為：

```text
resources/js/
├── components/    # Reusable Vue components
├── composables/   # Vue composables / hooks
├── layouts/       # Application layouts
├── lib/           # Utility functions and configuration
├── pages/         # Page components
└── types/         # TypeScript definitions
```

若要發佈額外的 shadcn-vue 元件，請先[找到您要發佈的元件](https://www.shadcn-vue.com)。然後，使用 `npx` 發佈該元件：

```shell
npx shadcn-vue@latest add switch
```

在此範例中，該指令會將 Switch 元件發佈到 `resources/js/components/ui/Switch.vue`。一旦元件發佈成功，您就可以在任何頁面中使用它：

```vue
<script setup lang="ts">
import { Switch } from '@/Components/ui/switch'
</script>

<template>
    <div>
        <Switch />
    </div>
</template>
```


<a name="vue-available-layouts"></a>
#### 可用佈局

Vue 快速入門套件包含兩種主要佈局供您選擇：「側邊欄」佈局和「標頭」佈局。側邊欄佈局是預設值，但您可以透過修改應用程式 `resources/js/layouts/AppLayout.vue` 檔案頂部匯入的佈局來切換到標頭佈局：

```js
import AppLayout from '@/layouts/app/AppSidebarLayout.vue'; // [tl! remove]
import AppLayout from '@/layouts/app/AppHeaderLayout.vue'; // [tl! add]
```


<a name="vue-sidebar-variants"></a>
#### 側邊欄變體

側邊欄佈局包含三種不同的變體：預設側邊欄變體、「內嵌 (inset)」變體和「浮動 (floating)」變體。您可以透過修改 `resources/js/components/AppSidebar.vue` 元件來選擇您最喜歡的變體：

```text
<Sidebar collapsible="icon" variant="sidebar"> [tl! remove]
<Sidebar collapsible="icon" variant="inset"> [tl! add]
```


<a name="vue-authentication-page-layout-variants"></a>
#### 身份驗證頁面佈局變體

Vue 快速入門套件中包含的身份驗證頁面，例如登入頁面和註冊頁面，也提供三種不同的佈局變體：「簡潔 (simple)」、「卡片 (card)」和「分割 (split)」。

若要更改您的身份驗證佈局，請修改應用程式 `resources/js/layouts/AuthLayout.vue` 檔案頂部匯入的佈局：

```js
import AuthLayout from '@/layouts/auth/AuthSimpleLayout.vue'; // [tl! remove]
import AuthLayout from '@/layouts/auth/AuthSplitLayout.vue'; // [tl! add]
```


<a name="livewire-customization"></a>
### Livewire

我們的 Livewire 快速入門套件使用 Livewire 3, Tailwind, 和 [Flux UI](https://fluxui.dev/) 建構。與所有我們的快速入門套件一樣，所有的後端與前端程式碼都存在於您的應用程式中，以實現完全客製化。


#### Livewire 與 Volt

大部分前端程式碼位於 `resources/views` 目錄中。您可以自由修改任何程式碼，以客製化您應用程式的外觀與行為：

```text
resources/views
├── components            # Reusable Livewire components
├── flux                  # Customized Flux components
├── livewire              # Livewire pages
├── partials              # Reusable Blade partials
├── dashboard.blade.php   # Authenticated user dashboard
├── welcome.blade.php     # Guest user welcome page
```


#### 傳統 Livewire 元件

前端程式碼位於 `resources/views` 目錄中，而 `app/Livewire` 目錄包含 Livewire 元件相對應的後端邏輯。


<a name="livewire-available-layouts"></a>
#### 可用佈局

Livewire 快速入門套件包含兩種主要佈局供您選擇：「側邊欄」佈局和「標頭」佈局。側邊欄佈局是預設值，但您可以透過修改應用程式 `resources/views/components/layouts/app.blade.php` 檔案所使用的佈局來切換到標頭佈局。此外，您應該為主 Flux 元件添加 `container` 屬性：

```blade
<x-layouts.app.header>
    <flux:main container>
        {{ $slot }}
    </flux:main>
</x-layouts.app.header>
```


<a name="livewire-authentication-page-layout-variants"></a>
#### 身份驗證頁面佈局變體

Livewire 快速入門套件中包含的身份驗證頁面，例如登入頁面和註冊頁面，也提供三種不同的佈局變體：「簡潔 (simple)」、「卡片 (card)」和「分割 (split)」。

若要更改您的身份驗證佈局，請修改應用程式 `resources/views/components/layouts/auth.blade.php` 檔案所使用的佈局：

```blade
<x-layouts.auth.split>
    {{ $slot }}
</x-layouts.auth.split>
```

<a name="authentication"></a>
## 身份驗證

所有快速入門套件都使用 [Laravel Fortify](/docs/{{version}}/fortify) 來處理身份驗證。Fortify 提供了登入、註冊、密碼重設、Email 驗證等所需的路由、控制器和邏輯。

Fortify 會根據您應用程式的 `config/fortify.php` 設定檔中啟用的功能，自動註冊以下身份驗證路由：

| 路由                              | 方法 | 描述                         |
| ---------------------------------- | ------ | ----------------------------------- |
| `/login`                           | `GET`    | 顯示登入表單                  |
| `/login`                           | `POST`   | 驗證使用者                   |
| `/logout`                          | `POST`   | 登出使用者                        |
| `/register`                        | `GET`    | 顯示註冊表單           |
| `/register`                        | `POST`   | 建立新使用者                     |
| `/forgot-password`                 | `GET`    | 顯示密碼重設請求表單 |
| `/forgot-password`                 | `POST`   | 傳送密碼重設連結            |
| `/reset-password/{token}`          | `GET`    | 顯示密碼重設表單         |
| `/reset-password`                  | `POST`   | 更新密碼                     |
| `/email/verify`                    | `GET`    | 顯示 Email 驗證通知   |
| `/email/verify/{id}/{hash}`        | `GET`    | 驗證 Email 位址                |
| `/email/verification-notification` | `POST`   | 重新傳送驗證 Email           |
| `/user/confirm-password`           | `GET`    | 顯示密碼確認表單  |
| `/user/confirm-password`           | `POST`   | 確認密碼                    |
| `/two-factor-challenge`            | `GET`    | 顯示 2FA 挑戰表單          |
| `/two-factor-challenge`            | `POST`   | 驗證 2FA 程式碼                     |

`php artisan route:list` Artisan 指令可用於顯示應用程式中的所有路由。


<a name="enabling-and-disabling-features"></a>
### 啟用與停用功能

您可以透過應用程式的 `config/fortify.php` 設定檔來控制哪些 Fortify 功能已啟用：

```php
use Laravel\Fortify\Features;

'features' => [
    Features::registration(),
    Features::resetPasswords(),
    Features::emailVerification(),
    Features::twoFactorAuthentication([
        'confirm' => true,
        'confirmPassword' => true,
    ]),
],
```

若要停用某項功能，請將 `features` 陣列中的該功能條目註解掉或移除。例如，移除 `Features::registration()` 以停用公開註冊。

使用 [React](#react) 或 [Vue](#vue) 快速入門套件時，您還需要從前端程式碼中移除對已停用功能路由的任何引用。例如，如果您停用 Email 驗證，您應該移除 Vue 或 React 元件中對 `verification` 路由的導入和引用。這是必要的，因為這些快速入門套件使用 Wayfinder 進行型別安全的路由，它會在建構時產生路由定義。如果您引用的路由不再存在，您的應用程式將無法建構。


<a name="customizing-actions"></a>
### 客製化使用者建立與密碼重設

當使用者註冊或重設密碼時，Fortify 會呼叫位於您應用程式 `app/Actions/Fortify` 目錄中的 Action Class：

| 檔案                          | 描述                           |
| ----------------------------- | ------------------------------------- |
| `CreateNewUser.php`           | 驗證並建立新使用者       |
| `ResetUserPassword.php`       | 驗證並更新使用者密碼  |
| `PasswordValidationRules.php` | 定義密碼驗證規則     |

例如，若要客製化應用程式的註冊邏輯，您應該編輯 `CreateNewUser` action：

```php
public function create(array $input): User
{
    Validator::make($input, [
        'name' => ['required', 'string', 'max:255'],
        'email' => ['required', 'email', 'max:255', 'unique:users'],
        'phone' => ['required', 'string', 'max:20'], // [tl! add]
        'password' => $this->passwordRules(),
    ])->validate();

    return User::create([
        'name' => $input['name'],
        'email' => $input['email'],
        'phone' => $input['phone'], // [tl! add]
        'password' => Hash::make($input['password']),
    ]);
}
```


<a name="two-factor-authentication"></a>
### 雙重身份驗證

快速入門套件包含內建的雙重身份驗證 (2FA)，允許使用者使用任何相容於 TOTP 的身份驗證器應用程式來保護其帳戶。2FA 預設透過您應用程式的 `config/fortify.php` 設定檔中的 `Features::twoFactorAuthentication()` 啟用。

`confirm` 選項要求使用者在 2FA 完全啟用之前驗證程式碼，而 `confirmPassword` 選項則要求在啟用或停用 2FA 之前確認密碼。更多詳情，請參閱 [Fortify 的雙重身份驗證文件](/docs/{{version}}/fortify#two-factor-authentication)。


<a name="rate-limiting"></a>
### 頻率限制

頻率限制可防止暴力破解和重複的登入嘗試使您的身份驗證端點不堪重負。您可以在應用程式的 `FortifyServiceProvider` 中客製化 Fortify 的頻率限制行為：

```php
use Illuminate\Support\Facades\RateLimiter;
use Illuminate\Cache\RateLimiting\Limit;

RateLimiter::for('login', function ($request) {
    return Limit::perMinute(5)->by($request->email.$request->ip());
});
```

<a name="workos"></a>
## WorkOS AuthKit 身份驗證

預設情況下，React、Vue 與 Livewire 快速入門套件都利用 Laravel 內建的身份驗證系統來提供登入、註冊、密碼重設、電子郵件驗證等功能。此外，我們還提供由 [WorkOS AuthKit](https://authkit.com) 支援的快速入門套件變體，提供以下功能：

<div class="content-list" markdown="1">

- 社交身份驗證 (Google、Microsoft、GitHub 與 Apple)
- 通行金鑰身份驗證
- 基於電子郵件的「Magic Auth」
- SSO

</div>

使用 WorkOS 作為您的身份驗證提供者需要一個 [WorkOS 帳號](https://workos.com)。WorkOS 為每月活躍用戶高達一百萬的應用程式提供免費身份驗證。

若要使用 WorkOS AuthKit 作為您應用程式的身份驗證提供者，請透過 `laravel new` 建立您的新快速入門套件應用程式時，選擇 WorkOS 選項。

### 設定您的 WorkOS 快速入門套件

在使用 WorkOS 支援的快速入門套件建立新應用程式後，您應該在應用程式的 `.env` 檔案中設定 `WORKOS_CLIENT_ID`、`WORKOS_API_KEY` 和 `WORKOS_REDIRECT_URL` 環境變數。這些變數應與 WorkOS 儀表板中為您的應用程式提供的值相符：

```ini
WORKOS_CLIENT_ID=your-client-id
WORKOS_API_KEY=your-api-key
WORKOS_REDIRECT_URL="${APP_URL}/authenticate"
```

此外，您應該在 WorkOS 儀表板中設定應用程式首頁 URL。這是使用者在登出應用程式後會被重新導向的 URL。

<a name="configuring-authkit-authentication-methods"></a>
#### 設定 AuthKit 身份驗證方法

當使用 WorkOS 支援的快速入門套件時，我們建議您在應用程式的 WorkOS AuthKit 設定中停用「電子郵件 + 密碼」身份驗證，讓使用者只能透過社交身份驗證提供者、通行金鑰、「Magic Auth」和 SSO 進行身份驗證。這讓您的應用程式可以完全避免處理使用者密碼。

<a name="configuring-authkit-session-timeouts"></a>
#### 設定 AuthKit 工作階段逾時

此外，我們建議您將 WorkOS AuthKit 工作階段閒置逾時設定為與您的 Laravel 應用程式配置的工作階段逾時門檻相符，通常為兩小時。

<a name="inertia-ssr"></a>
### Inertia SSR

React 和 Vue 快速入門套件與 Inertia 的 [伺服器端渲染](https://inertiajs.com/server-side-rendering) 功能相容。為了為您的應用程式建構一個 Inertia SSR 相容的套件，請執行 `build:ssr` 指令：

```shell
npm run build:ssr
```

為了方便，也提供了 `composer dev:ssr` 指令。此指令將在為您的應用程式建構一個 SSR 相容的套件後，啟動 Laravel 開發伺服器和 Inertia SSR 伺服器，讓您可以使用 Inertia 的伺服器端渲染引擎在本地測試您的應用程式：

```shell
composer dev:ssr
```

<a name="community-maintained-starter-kits"></a>
### 社群維護的快速入門套件

當使用 Laravel 安裝程式建立新的 Laravel 應用程式時，您可以將 Packagist 上任何可用的社群維護快速入門套件提供到 `--using` 旗標：

```shell
laravel new my-app --using=example/starter-kit
```

<a name="creating-starter-kits"></a>
#### 建立快速入門套件

為了確保您的快速入門套件可供他人使用，您需要將其發佈到 [Packagist](https://packagist.org)。您的快速入門套件應在其 `.env.example` 檔案中定義其所需的環境變數，並且任何必要的安裝後指令應列在快速入門套件的 `composer.json` 檔案中的 `post-create-project-cmd` 陣列中。

<a name="faqs"></a>
### 常見問題

<a name="faq-upgrade"></a>
#### 如何升級？

每個快速入門套件都為您的下一個應用程式提供了堅實的起點。憑藉對程式碼的完全所有權，您可以依照您的設想調整、客製化並建構您的應用程式。然而，無需更新快速入門套件本身。

<a name="faq-enable-email-verification"></a>
#### 如何啟用電子郵件驗證？

您可以透過取消註解 `App/Models/User.php` 模型中的 `MustVerifyEmail` 匯入，並確保該模型實作了 `MustVerifyEmail` 介面來添加電子郵件驗證：

```php
<?php

namespace App\Models;

use Illuminate\Contracts\Auth\MustVerifyEmail;
// ...

class User extends Authenticatable implements MustVerifyEmail
{
    // ...
}
```

註冊後，使用者將收到一封驗證電子郵件。為了限制對某些路由的存取，直到使用者的電子郵件地址被驗證，請將 `verified` 中介層加入到路由中：

```php
Route::middleware(['auth', 'verified'])->group(function () {
    Route::get('dashboard', function () {
        return Inertia::render('dashboard');
    })->name('dashboard');
});
```

> [!NOTE]
> 當使用快速入門套件的 [WorkOS](#workos) 變體時，電子郵件驗證不是必需的。

<a name="faq-modify-email-template"></a>
#### 如何修改預設電子郵件範本？

您可能希望客製化預設電子郵件範本，以更好地符合您應用程式的品牌。若要修改此範本，您應該使用以下指令將電子郵件視圖發佈到您的應用程式：

```
php artisan vendor:publish --tag=laravel-mail
```

這將在 `resources/views/vendor/mail` 中生成幾個檔案。您可以修改這些檔案中的任何一個以及 `resources/views/vendor/mail/themes/default.css` 檔案，以更改預設電子郵件範本的外觀和樣式。