# 入門套件

- [簡介](#introduction)
- [使用入門套件建立應用程式](#creating-an-application)
- [可用的入門套件](#available-starter-kits)
    - [React](#react)
    - [Svelte](#svelte)
    - [Vue](#vue)
    - [Livewire](#livewire)
- [自訂入門套件](#starter-kit-customization)
    - [React](#react-customization)
    - [Svelte](#svelte-customization)
    - [Vue](#vue-customization)
    - [Livewire](#livewire-customization)
- [認證](#authentication)
    - [啟用與停用功能](#enabling-and-disabling-features)
    - [自訂使用者建立與密碼重設](#customizing-actions)
    - [雙重認證](#two-factor-authentication)
    - [速率限制](#rate-limiting)
- [團隊](#teams)
- [WorkOS AuthKit 認證](#workos)
- [Inertia SSR](#inertia-ssr)
- [社群維護的入門套件](#community-maintained-starter-kits)
- [常見問題](#faqs)

<a name="introduction"></a>
## 簡介

為了讓您在建置新的 Laravel 應用程式時能有個良好的起點，我們很高興能提供 [應用程式入門套件 (Application starter kits)](https://laravel.com/starter-kits)。這些入門套件能讓您快速開始建置下一個 Laravel 應用程式，並包含了註冊與認證應用程式使用者所需的路由、控制器與視圖。這些入門套件使用 [Laravel Fortify](/docs/{{version}}/fortify) 來提供認證功能。

雖然非常歡迎您使用這些入門套件，但它們並非必要。您大可藉由直接安裝全新的 Laravel 來從頭開始建置您自己的應用程式。不論選擇哪種方式，我們相信您都將建置出棒極了的東西！


<a name="creating-an-application"></a>
## 使用入門套件建立應用程式

要使用我們的其中一款入門套件來建立新的 Laravel 應用程式，您應該先 [安裝 PHP 與 Laravel CLI 工具](/docs/{{version}}/installation#installing-php)。如果您已經安裝了 PHP 與 Composer，您可以透過 Composer 安裝 Laravel 安裝器 CLI 工具：

```shell
composer global require laravel/installer
```

接著，使用 Laravel 安裝器 CLI 建立新的 Laravel 應用程式。Laravel 安裝器會提示您選擇您偏好的入門套件：

```shell
laravel new my-app
```

建立 Laravel 應用程式後，您只需要透過 NPM 安裝其前端依賴項目，並啟動 Laravel 開發伺服器：

```shell
cd my-app
npm install && npm run build
composer run dev
```

一旦啟動了 Laravel 開發伺服器，您就可以在瀏覽器中透過 [http://localhost:8000](http://localhost:8000) 存取您的應用程式。


<a name="available-starter-kits"></a>
## 可用的入門套件


<a name="react"></a>
### React

我們的 React 入門套件提供了一個強大且現代的起點，讓您能使用 [Inertia](https://inertiajs.com) 來建置具有 React 前端的 Laravel 應用程式。

Inertia 允許您使用傳統的伺服器端路由與控制器來建置現代的單頁 (Single-page) React 應用程式。這讓您能同時享受 React 的強大前端實力、Laravel 令人難以置信的後端生產力，以及超快速的 Vite 編譯。

React 入門套件使用了 React 19、TypeScript、Tailwind 以及 [shadcn/ui](https://ui.shadcn.com) 元件庫。


<a name="svelte"></a>
### Svelte

我們的 Svelte 入門套件提供了一個強大且現代的起點，讓您能使用 [Inertia](https://inertiajs.com) 來建置具有 Svelte 前端的 Laravel 應用程式。

Inertia 允許您使用傳統的伺服器端路由與控制器來建置現代的單頁 Svelte 應用程式。這讓您能同時享受 Svelte 的強大前端實力、Laravel 令人難以置信的後端生產力，以及超快速的 Vite 編譯。

Svelte 入門套件使用了 Svelte 5、TypeScript、Tailwind 以及 [shadcn-svelte](https://www.shadcn-svelte.com/) 元件庫。


<a name="vue"></a>
### Vue

我們的 Vue 入門套件提供了一個極佳的起點，讓您能使用 [Inertia](https://inertiajs.com) 來建置具有 Vue 前端的 Laravel 應用程式。

Inertia 允許您使用傳統的伺服器端路由與控制器來建置現代的單頁 Vue 應用程式。這讓您能同時享受 Vue 的強大前端實力、Laravel 令人難以置信的後端生產力，以及超快速的 Vite 編譯。

Vue 入門套件使用了 Vue Composition API、TypeScript、Tailwind 以及 [shadcn-vue](https://www.shadcn-vue.com/) 元件庫。


<a name="livewire"></a>
### Livewire

我們的 Livewire 入門套件提供了一個完美的起點，讓您能建置具有 [Laravel Livewire](https://livewire.laravel.com) 前端的 Laravel 應用程式。

Livewire 是一種強大的方式，讓您能僅使用 PHP 來建置動態、響應式的前端 UI。它非常適合主要使用 Blade 範本且正在尋找比 React、Svelte 和 Vue 等 JavaScript 驅動之 SPA 框架更簡單之替代方案的團隊。

Livewire 入門套件使用了 Livewire、Tailwind 以及 [Flux UI](https://fluxui.dev) 元件庫。

<a name="starter-kit-customization"></a>
## 自訂入門套件


<a name="react-customization"></a>
### React

我們的 React 入門套件是使用 Inertia 3、React 19、Tailwind 4 以及 [shadcn/ui](https://ui.shadcn.com) 建構的。與我們所有的入門套件一樣，所有的後端與前端程式碼都存在於您的應用程式中，以便進行完整的自訂。

大部分的前端程式碼都位於 `resources/js` 目錄中。您可以自由修改任何程式碼，以自訂應用程式的外觀與行為：

```text
resources/js/
├── components/    # Reusable React components
├── hooks/         # React hooks
├── layouts/       # Application layouts
├── lib/           # Utility functions and configuration
├── pages/         # Page components
└── types/         # TypeScript definitions
```

若要發布額外的 shadcn 元件，首先請[尋找您想要發布的元件](https://ui.shadcn.com)。接著，使用 `npx` 來發布元件：

```shell
npx shadcn@latest add switch
```

在此範例中，該指令會將 Switch 元件發布至 `resources/js/components/ui/switch.tsx`。元件發布後，您就可以在任何頁面中使用它：

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
#### 可用的版面配置

React 入門套件包含兩種不同的主要版面配置供您選擇：一個是 "sidebar" 版面配置，另一個是 "header" 版面配置。預設為 sidebar 版面配置，但您可以透過修改應用程式 `resources/js/layouts/app-layout.tsx` 檔案頂部匯入的版面配置來切換至 header 版面配置：

```js
import AppLayoutTemplate from '@/layouts/app/app-sidebar-layout'; // [tl! remove]
import AppLayoutTemplate from '@/layouts/app/app-header-layout'; // [tl! add]
```


<a name="react-sidebar-variants"></a>
#### Sidebar 樣式變體

sidebar 版面配置包含三種不同的樣式變體：預設的 sidebar 變體、"inset" 變體以及 "floating" 變體。您可以透過修改 `resources/js/components/app-sidebar.tsx` 元件來選擇您最喜歡的變體：

```text
<Sidebar collapsible="icon" variant="sidebar"> [tl! remove]
<Sidebar collapsible="icon" variant="inset"> [tl! add]
```


<a name="react-authentication-page-layout-variants"></a>
#### 認證頁面版面配置變體

React 入門套件中包含的認證頁面（例如登入頁面與註冊頁面）也提供了三種不同的版面配置變體："simple"、"card" 以及 "split"。

若要變更您的認證版面配置，請修改應用程式 `resources/js/layouts/auth-layout.tsx` 檔案頂部匯入的版面配置：

```js
import AuthLayoutTemplate from '@/layouts/auth/auth-simple-layout'; // [tl! remove]
import AuthLayoutTemplate from '@/layouts/auth/auth-split-layout'; // [tl! add]
```


<a name="svelte-customization"></a>
### Svelte

我們的 Svelte 入門套件是使用 Inertia 3、Svelte 5、Tailwind 以及 [shadcn-svelte](https://www.shadcn-svelte.com/) 建構的。與我們所有的入門套件一樣，所有的後端與前端程式碼都存在於您的應用程式中，以便進行完整的自訂。

大部分的前端程式碼都位於 `resources/js` 目錄中。您可以自由修改任何程式碼，以自訂應用程式的外觀與行為：

```text
resources/js/
├── components/    # Reusable Svelte components
├── layouts/       # Application layouts
├── lib/           # Utility functions and configuration and Svelte rune modules
├── pages/         # Page components
└── types/         # TypeScript definitions
```

若要發布額外的 shadcn-svelte 元件，首先請[尋找您想要發布的元件](https://www.shadcn-svelte.com)。接著，使用 `npx` 來發布元件：

```shell
npx shadcn-svelte@latest add switch
```

在此範例中，該指令會將 Switch 元件發布至 `resources/js/components/ui/switch/switch.svelte`。元件發布後，您就可以在任何頁面中使用它：

```svelte
<script lang="ts">
    import { Switch } from '@/components/ui/switch'
</script>

<div>
    <Switch />
</div>
```


<a name="svelte-available-layouts"></a>
#### 可用的版面配置

Svelte 入門套件包含兩種不同的主要版面配置供您選擇：一個是 "sidebar" 版面配置，另一個是 "header" 版面配置。預設為 sidebar 版面配置，但您可以透過修改應用程式 `resources/js/layouts/AppLayout.svelte` 檔案頂部匯入的版面配置來切換至 header 版面配置：

```js
import AppLayout from '@/layouts/app/AppSidebarLayout.svelte'; // [tl! remove]
import AppLayout from '@/layouts/app/AppHeaderLayout.svelte'; // [tl! add]
```


<a name="svelte-sidebar-variants"></a>
#### Sidebar 樣式變體

sidebar 版面配置包含三種不同的樣式變體：預設的 sidebar 變體、"inset" 變體以及 "floating" 變體。您可以透過修改 `resources/js/components/AppSidebar.svelte` 元件來選擇您最喜歡的變體：

```text
<Sidebar collapsible="icon" variant="sidebar"> [tl! remove]
<Sidebar collapsible="icon" variant="inset"> [tl! add]
```


<a name="svelte-authentication-page-layout-variants"></a>
#### 認證頁面版面配置變體

Svelte 入門套件中包含的認證頁面（例如登入頁面與註冊頁面）也提供了三種不同的版面配置變體："simple"、"card" 以及 "split"。

若要變更您的認證版面配置，請修改應用程式 `resources/js/layouts/AuthLayout.svelte` 檔案頂部匯入的版面配置：

```js
import AuthLayout from '@/layouts/auth/AuthSimpleLayout.svelte'; // [tl! remove]
import AuthLayout from '@/layouts/auth/AuthSplitLayout.svelte'; // [tl! add]
```

<a name="vue-customization"></a>
### Vue

我們的 Vue 入門套件是使用 Inertia 3、Vue 3 Composition API、Tailwind 和 [shadcn-vue](https://www.shadcn-vue.com/) 所構建。如同我們所有的入門套件一樣，所有的後端與前端程式碼都存在於您的應用程式中，以便進行完整的自訂。

大部分的前端程式碼都位於 `resources/js` 目錄中。您可以自由修改任何程式碼，以自訂應用程式的外觀和行為：

```text
resources/js/
├── components/    # Reusable Vue components
├── composables/   # Vue composables / hooks
├── layouts/       # Application layouts
├── lib/           # Utility functions and configuration
├── pages/         # Page components
└── types/         # TypeScript definitions
```

若要發布額外的 shadcn-vue 元件，請先[尋找您想要發布的元件](https://www.shadcn-vue.com)。接著，使用 `npx` 來發布該元件：

```shell
npx shadcn-vue@latest add switch
```

在此範例中，該指令會將 Switch 元件發布至 `resources/js/components/ui/Switch.vue`。一旦元件發布完成，您就可以在任何頁面中使用它：

```vue
<script setup lang="ts">
import { Switch } from '@/components/ui/switch'
</script>

<template>
    <div>
        <Switch />
    </div>
</template>
```


<a name="vue-available-layouts"></a>
#### 可用的版面配置

Vue 入門套件提供了兩種不同的主要版面配置供您選擇：「側邊欄 (sidebar)」版面配置與「標頭 (header)」版面配置。預設為側邊欄版面配置，但您可以透過修改應用程式中 `resources/js/layouts/AppLayout.vue` 檔案頂部所匯入的版面配置，來切換為標頭版面配置：

```js
import AppLayout from '@/layouts/app/AppSidebarLayout.vue'; // [tl! remove]
import AppLayout from '@/layouts/app/AppHeaderLayout.vue'; // [tl! add]
```


<a name="vue-sidebar-variants"></a>
#### 側邊欄變體

側邊欄版面配置包含三種不同的變體：預設側邊欄變體、「內縮 (inset)」變體以及「懸浮 (floating)」變體。您可以透過修改 `resources/js/components/AppSidebar.vue` 元件來選擇最喜歡的變體：

```text
<Sidebar collapsible="icon" variant="sidebar"> [tl! remove]
<Sidebar collapsible="icon" variant="inset"> [tl! add]
```


<a name="vue-authentication-page-layout-variants"></a>
#### 認證頁面版面配置變體

Vue 入門套件中附帶的認證頁面（例如登入頁面和註冊頁面）也提供了三種不同的版面配置變體：「簡單 (simple)」、「卡片 (card)」和「分割 (split)」。

若要變更您的認證版面配置，請修改應用程式中 `resources/js/layouts/AuthLayout.vue` 檔案頂部所匯入的版面配置：

```js
import AuthLayout from '@/layouts/auth/AuthSimpleLayout.vue'; // [tl! remove]
import AuthLayout from '@/layouts/auth/AuthSplitLayout.vue'; // [tl! add]
```


<a name="livewire-customization"></a>
### Livewire

我們的 Livewire 入門套件是使用 Livewire 4、Tailwind 和 [Flux UI](https://fluxui.dev/) 構建而成。如同我們所有的入門套件一樣，所有的後端與前端程式碼都存在於您的應用程式中，以便進行完整的自訂。

大部分的前端程式碼都位於 `resources/views` 目錄中。您可以自由修改任何程式碼，以自訂應用程式的外觀和行為：

```text
resources/views
├── components            # Reusable components
├── flux                  # Customized Flux components
├── layouts               # Application layouts
├── pages                 # Livewire pages
├── partials              # Reusable Blade partials
├── dashboard.blade.php   # Authenticated user dashboard
├── welcome.blade.php     # Guest user welcome page
```


<a name="livewire-available-layouts"></a>
#### 可用的版面配置

Livewire 入門套件提供了兩種不同的主要版面配置供您選擇：「側邊欄 (sidebar)」版面配置與「標頭 (header)」版面配置。預設為側邊欄版面配置，但您可以透過修改應用程式中 `resources/views/layouts/app.blade.php` 檔案所使用的版面配置，來切換為標頭版面配置。此外，您應該在主要的 Flux 元件上新增 `container` 屬性：

```blade
<x-layouts::app.header>
    <flux:main container>
        {{ $slot }}
    </flux:main>
</x-layouts::app.header>
```


<a name="livewire-authentication-page-layout-variants"></a>
#### 認證頁面版面配置變體

Livewire 入門套件中附帶的認證頁面（例如登入頁面和註冊頁面）也提供了三種不同的版面配置變體：「簡單 (simple)」、「卡片 (card)」和「分割 (split)」。

若要變更您的認證版面配置，請修改應用程式中 `resources/views/layouts/auth.blade.php` 檔案所使用的版面配置：

```blade
<x-layouts::auth.split>
    {{ $slot }}
</x-layouts::auth.split>
```

<a name="authentication"></a>
## 認證

所有入門套件都使用 [Laravel Fortify](/docs/{{version}}/fortify) 來處理認證。Fortify 提供了可用於登入、註冊、密碼重設、電子郵件驗證等功能的路由、控制器與邏輯。

Fortify 會根據您應用程式的 `config/fortify.php` 設定檔中啟用的功能，自動註冊以下認證路由：

| Route                              | Method | Description                         |
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
| `/email/verify`                    | `GET`    | 顯示電子郵件驗證提示   |
| `/email/verify/{id}/{hash}`        | `GET`    | 驗證電子郵件地址                |
| `/email/verification-notification` | `POST`   | 重新傳送驗證電子郵件           |
| `/user/confirm-password`           | `GET`    | 顯示密碼確認表單  |
| `/user/confirm-password`           | `POST`   | 確認密碼                    |
| `/two-factor-challenge`            | `GET`    | 顯示雙重認證 (2FA) 挑戰表單          |
| `/two-factor-challenge`            | `POST`   | 驗證雙重認證 (2FA) 代碼                     |

您可以使用 `php artisan route:list` Artisan 命令來顯示應用程式中的所有路由。


<a name="enabling-and-disabling-features"></a>
### 啟用與停用功能

您可以透過應用程式的 `config/fortify.php` 設定檔來控制啟用哪些 Fortify 功能：

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

若要停用某個功能，請在 `features` 陣列中註解或移除該功能的項目。例如，移除 `Features::registration()` 即可停用公開註冊功能。

使用 [React](#react)、[Svelte](#svelte) 或 [Vue](#vue) 入門套件時，您還需要從前端程式碼中移除對已停用功能路由的所有引用。例如，如果您停用了電子郵件驗證，則應移除 React、Svelte 或 Vue 元件中對 `verification` 路由的匯入和引用。這是必要的，因為這些入門套件使用 Wayfinder 進行型別安全的路由，它會在建置時生成路由定義。如果您引用了已不存在的路由，您的應用程式將會建置失敗。


<a name="customizing-actions"></a>
### 自訂使用者建立與密碼重設

當使用者註冊或重設密碼時，Fortify 會調用位於應用程式 `app/Actions/Fortify` 目錄中的 Action 類別：

| 檔案 | 說明 |
| ----------------------------- | ------------------------------------- |
| `CreateNewUser.php`           | 驗證並建立新使用者 |
| `ResetUserPassword.php`       | 驗證並更新使用者密碼 |
| `PasswordValidationRules.php` | 定義密碼驗證規則 |

例如，若要自訂您應用程式的註冊邏輯，您應該編輯 `CreateNewUser` Action：

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
### 雙重認證

入門套件內建了雙重認證 (2FA) 功能，允許使用者使用任何與 TOTP 相容的驗證器應用程式來保護其帳號。在您應用程式的 `config/fortify.php` 設定檔中，2FA 預設透過 `Features::twoFactorAuthentication()` 啟用。

`confirm` 選項要求使用者在完全啟用 2FA 之前必須先驗證一組驗證碼，而 `confirmPassword` 則要求在啟用或停用 2FA 之前必須先確認密碼。如需更多詳細資訊，請參閱 [Fortify 的雙重認證說明文件](/docs/{{version}}/fortify#two-factor-authentication)。


<a name="rate-limiting"></a>
### 速率限制

速率限制可防止暴力破解與重複的登入嘗試癱瘓您的認證端點。您可以在應用程式的 `FortifyServiceProvider` 中自訂 Fortify 的速率限制行為：

```php
use Illuminate\Support\Facades\RateLimiter;
use Illuminate\Cache\RateLimiting\Limit;

RateLimiter::for('login', function ($request) {
    return Limit::perMinute(5)->by($request->email.$request->ip());
});
```


<a name="teams"></a>
## 團隊

React、Svelte、Vue 與 Livewire 入門套件也可以在建立時包含團隊支援。當啟用團隊功能時，每個使用者都會屬於一個或多個團隊，並擁有一個「目前團隊」。在註冊期間，新使用者會自動獲得一個個人團隊。入門套件還包含了團隊管理畫面，用於建立團隊、切換團隊、邀請成員以及更新團隊詳細資訊。

當路由的作用域限制在目前團隊時，目前團隊的代稱 (slug) 會被包含在 URL 中。例如，儀表板路由會變成 `/{current_team}/dashboard`，而團隊管理頁面則使用像是 `settings/teams/{team}` 的路由。當使用 `{current_team}` 與 `{team}` 路由參數時，入門套件會自動確保已驗證的使用者確實屬於所請求的團隊，然後才允許存取該路由。

為了讓生成含有團隊資訊的 URL 更加方便，入門套件會為已驗證使用者的目前團隊註冊預設 URL 參數。這使得調用像是 `route('dashboard')` 的輔助函式時，會自動包含目前團隊的代稱 (slug)。當使用者登入、註冊或切換團隊時，入門套件會更新目前團隊並重新整理這些預設 URL 參數，以便生成的連結能持續使用正確的團隊上下文。

在建立或重新命名團隊時，入門套件也會防止使用者選擇保留名稱，以避免產生不安全或衝突的路由段。例如，無法使用會與 `settings`、`login` 或 `dashboard` 等路由前綴產生衝突的名稱。

<a name="workos"></a>
## WorkOS AuthKit 認證

預設情況下，React、Svelte、Vue 和 Livewire 入門套件都利用 Laravel 內建的認證系統來提供登入、註冊、密碼重設、電子郵件驗證等功能。此外，我們還提供了由 [WorkOS AuthKit](https://authkit.com) 支援的各個入門套件版本，其提供：

<div class="content-list" markdown="1">

- 社群認證 (Google、Microsoft、GitHub 和 Apple)
- 通行鑰 (Passkey) 認證
- 基於電子郵件的「Magic Auth」
- 單一登入 (SSO)

</div>

使用 WorkOS 作為您的認證提供者[需要一個 WorkOS 帳號](https://workos.com)。WorkOS 為每月活躍用戶不超過 100 萬的應用程式提供免費認證。

要使用 WorkOS AuthKit 作為您應用程式的認證提供者，請在透過 `laravel new` 建立新的入門套件應用程式時，選擇 WorkOS 選項。


### 設定您的 WorkOS 入門套件

在使用 WorkOS 支援的入門套件建立新的應用程式之後，您應該在應用程式的 `.env` 檔案中設定 `WORKOS_CLIENT_ID`、`WORKOS_API_KEY` 和 `WORKOS_REDIRECT_URL` 環境變數。這些變數應該與 WorkOS 儀表板中為您的應用程式提供的值相匹配：

```ini
WORKOS_CLIENT_ID=your-client-id
WORKOS_API_KEY=your-api-key
WORKOS_REDIRECT_URL="${APP_URL}/authenticate"
```

此外，您應該在 WorkOS 儀表板中設定應用程式的首頁 URL。此 URL 是使用者登出應用程式後將被重導向的地方。


<a name="configuring-authkit-authentication-methods"></a>
#### 設定 AuthKit 認證方法

使用 WorkOS 支援的入門套件時，我們建議您在應用程式的 WorkOS AuthKit 設定中停用「Email + Password（電子郵件 + 密碼）」認證，僅允許使用者透過社群認證提供者、通行鑰 (passkeys)、「Magic Auth」和 SSO 進行認證。這可以讓您的應用程式完全免於處理使用者密碼。


<a name="configuring-authkit-session-timeouts"></a>
#### 設定 AuthKit 工作階段逾時

此外，我們建議您設定 WorkOS AuthKit 的工作階段閒置逾時，以匹配您 Laravel 應用程式所設定的工作階段逾時限制（通常為兩小時）。


<a name="inertia-ssr"></a>
### Inertia SSR

React、Svelte 和 Vue 入門套件與 Inertia 的[伺服器端渲染 (server-side rendering)](https://inertiajs.com/server-side-rendering) 功能相容。要為您的應用程式建置與 Inertia SSR 相容的套件，請執行 `build:ssr` 命令：

```shell
npm run build:ssr
```

為了方便起見，也提供了一個 `composer dev:ssr` 命令。該命令將在為您的應用程式建置 SSR 相容套件後，啟動 Laravel 開發伺服器和 Inertia SSR 伺服器，讓您可以使用 Inertia 的伺服器端渲染引擎在本地端測試您的應用程式：

```shell
composer dev:ssr
```


<a name="community-maintained-starter-kits"></a>
### 社群維護的入門套件

使用 Laravel 安裝程式建立新的 Laravel 應用程式時，您可以將 Packagist 上可用的任何社群維護入門套件提供給 `--using` 旗標：

```shell
laravel new my-app --using=example/starter-kit
```


<a name="creating-starter-kits"></a>
#### 建立入門套件

為了確保您的入門套件可供他人使用，您需要將其發布到 [Packagist](https://packagist.org)。您的入門套件應該在它的 `.env.example` 檔案中定義其所需的環境變數，並且任何必要的安裝後命令都應該列在入門套件 `composer.json` 檔案中的 `post-create-project-cmd` 陣列中。


<a name="faqs"></a>
### 常見問題


<a name="faq-upgrade"></a>
#### 我該如何升級？

每個入門套件都為您的下一個應用程式提供了一個穩固的起點。擁有了程式碼的完整所有權，您可以完全按照自己的想法來調整、自訂和建置您的應用程式。因此，不需要升級入門套件本身。


<a name="faq-enable-email-verification"></a>
#### 我該如何啟用電子郵件驗證？

可以透過取消註解 `App/Models/User.php` 模型中的 `MustVerifyEmail` 匯入，並確保該模型實作了 `MustVerifyEmail` 介面來加入電子郵件驗證：

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

註冊後，使用者將收到一封驗證電子郵件。若要在使用者的電子郵件地址驗證之前限制對某些路由的存取，請將 `verified` 中介層加入到這些路由中：

```php
Route::middleware(['auth', 'verified'])->group(function () {
    Route::get('dashboard', function () {
        return Inertia::render('dashboard');
    })->name('dashboard');
});
```

> [!NOTE]
> 使用 [WorkOS](#workos) 版本的入門套件時，不需要進行電子郵件驗證。


<a name="faq-modify-email-template"></a>
#### 我該如何修改預設的電子郵件範本？

您可能想要自訂預設的電子郵件範本，以更符合您應用程式的品牌形象。若要修改此範本，您應該使用以下命令將電子郵件視圖發布到您的應用程式中：

```
php artisan vendor:publish --tag=laravel-mail
```

這將在 `resources/views/vendor/mail` 中生成數個檔案。您可以修改這些檔案中的任何一個，以及 `resources/views/vendor/mail/themes/default.css` 檔案，來變更預設電子郵件範本的外觀與樣式。