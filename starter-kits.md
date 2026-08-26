# 入門套件

- [簡介](#introduction)
- [使用入門套件建立應用程式](#creating-an-application)
- [可用的入門套件](#available-starter-kits)
    - [React](#react)
    - [Svelte](#svelte)
    - [Vue](#vue)
    - [Livewire](#livewire)
- [入門套件自訂](#starter-kit-customization)
    - [React](#react-customization)
    - [Svelte](#svelte-customization)
    - [Vue](#vue-customization)
    - [Livewire](#livewire-customization)
- [認證](#authentication)
    - [啟用與停用功能](#enabling-and-disabling-features)
    - [自訂使用者建立與密碼重設](#customizing-actions)
    - [雙重驗證](#two-factor-authentication)
    - [速率限制](#rate-limiting)
- [團隊](#teams)
- [WorkOS AuthKit 認證](#workos)
    - [設定您的 WorkOS 入門套件](#configuring-your-workos-starter-kit)
- [Inertia SSR](#inertia-ssr)
- [社群維護的入門套件](#community-maintained-starter-kits)
- [常見問題](#faqs)

<a name="introduction"></a>
## 簡介

為了讓您在建立全新的 Laravel 應用程式時能搶得先機，我們很樂意提供[應用程式入門套件](https://laravel.com/starter-kits)。這些入門套件為您建置下一個 Laravel 應用程式提供了良好的起步，並包含了註冊與認證應用程式使用者所需的路由、控制器與視圖。入門套件使用 [Laravel Fortify](/docs/{{version}}/fortify) 來提供認證功能。

雖然我們非常歡迎您使用這些入門套件，但這並非強制性的。您也可以選擇透過安裝全新的 Laravel，完全從頭開始建立您自己的應用程式。無論選擇哪種方式，我們相信您都能打造出優秀的作品！


<a name="creating-an-application"></a>
## 使用入門套件建立應用程式

若要使用我們的其中一種入門套件建立新的 Laravel 應用程式，您應該先[安裝 PHP 與 Laravel CLI 工具](/docs/{{version}}/installation#installing-php)。如果您已經安裝了 PHP 和 Composer，可以透過 Composer 安裝 Laravel 安裝程式 CLI 工具：

```shell
composer global require laravel/installer
```

接著，使用 Laravel 安裝程式 CLI 建立新的 Laravel 應用程式。Laravel 安裝程式會提示您選擇偏好的入門套件：

```shell
laravel new my-app
```

建立 Laravel 應用程式後，您只需要透過 NPM 安裝其前端相依套件，並啟動 Laravel 開發伺服器：

```shell
cd my-app
npm install && npm run build
composer run dev
```

啟動 Laravel 開發伺服器後，您就可以在網頁瀏覽器中透過 [http://localhost:8000](http://localhost:8000) 存取您的應用程式。


<a name="available-starter-kits"></a>
## 可用的入門套件


<a name="react"></a>
### React

我們的 React 入門套件為使用 [Inertia](https://inertiajs.com) 建置搭配 React 前端的 Laravel 應用程式提供了一個穩健且現代化的起點。

Inertia 允許您使用傳統的伺服器端路由和控制器來建置現代化的單頁 React 應用程式。這讓您既能享受 React 強大的前端能力，又能結合 Laravel 驚人的後端生產力以及超快速的 Vite 編譯。

React 入門套件使用了 React 19、TypeScript、Tailwind 以及 [shadcn/ui](https://ui.shadcn.com) 元件庫。


<a name="svelte"></a>
### Svelte

我們的 Svelte 入門套件為使用 [Inertia](https://inertiajs.com) 建置搭配 Svelte 前端的 Laravel 應用程式提供了一個穩健且現代化的起點。

Inertia 允許您使用傳統的伺服器端路由和控制器來建置現代化的單頁 Svelte 應用程式。這讓您既能享受 Svelte 強大的前端能力，又能結合 Laravel 驚人的後端生產力以及超快速的 Vite 編譯。

Svelte 入門套件使用了 Svelte 5、TypeScript、Tailwind 以及 [shadcn-svelte](https://www.shadcn-svelte.com/) 元件庫。


<a name="vue"></a>
### Vue

我們的 Vue 入門套件為使用 [Inertia](https://inertiajs.com) 建置搭配 Vue 前端的 Laravel 應用程式提供了一個極佳的起點。

Inertia 允許您使用傳統的伺服器端路由和控制器來建置現代化的單頁 Vue 應用程式。這讓您既能享受 Vue 強大的前端能力，又能結合 Laravel 驚人的後端生產力以及超快速的 Vite 編譯。

Vue 入門套件使用了 Vue Composition API、TypeScript、Tailwind 以及 [shadcn-vue](https://www.shadcn-vue.com/) 元件庫。


<a name="livewire"></a>
### Livewire

我們的 Livewire 入門套件為建置搭配 [Laravel Livewire](https://livewire.laravel.com) 前端的 Laravel 應用程式提供了一個完美的起點。

Livewire 是一種強大的方式，僅使用 PHP 即可建立動態且具響應性的前端 UI。它非常適合主要使用 Blade 模板、並希望尋求比 React、Svelte 和 Vue 等 JavaScript 驅動的 SPA 框架更簡單之替代方案的團隊。

Livewire 入門套件使用了 Livewire、Tailwind 以及 [Flux UI](https://fluxui.dev) 元件庫。

<a name="starter-kit-customization"></a>
## 入門套件自訂


<a name="react-customization"></a>
### React

我們的 React 入門套件是使用 Inertia 3、React 19、Tailwind 4 和 [shadcn/ui](https://ui.shadcn.com) 所打造。與我們所有的入門套件一樣，所有的後端與前端程式碼都存在於您的應用程式中，以實現完全的自訂。

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

若要發布額外的 shadcn 元件，請先[找到您想要發布的元件](https://ui.shadcn.com)。然後，使用 `npx` 發布該元件：

```shell
npx shadcn@latest add switch
```

在這個範例中，該命令會將 Switch 元件發布至 `resources/js/components/ui/switch.tsx`。元件發布後，您就可以在任何頁面中使用它：

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

React 入門套件包含兩種不同的主要版面配置供您選擇：「sidebar」版面配置和「header」版面配置。側邊欄版面配置是預設值，但您可以透過修改應用程式 `resources/js/layouts/app-layout.tsx` 檔案頂部所匯入的版面配置，切換為頁首版面配置：

```js
import AppLayoutTemplate from '@/layouts/app/app-sidebar-layout'; // [tl! remove]
import AppLayoutTemplate from '@/layouts/app/app-header-layout'; // [tl! add]
```


<a name="react-sidebar-variants"></a>
#### 側邊欄變體

側邊欄版面配置包含三種不同的變體：預設的 sidebar 變體、「inset」變體以及「floating」變體。您可以透過修改 `resources/js/components/app-sidebar.tsx` 元件來選擇您最喜歡的變體：

```text
<Sidebar collapsible="icon" variant="sidebar"> [tl! remove]
<Sidebar collapsible="icon" variant="inset"> [tl! add]
```


<a name="react-authentication-page-layout-variants"></a>
#### 認證頁面版面配置變體

React 入門套件隨附的認證頁面（例如登入頁面和註冊頁面）也提供三種不同的版面配置變體：「simple」、「card」和「split」。

若要變更您的認證版面配置，請修改應用程式 `resources/js/layouts/auth-layout.tsx` 檔案頂部所匯入的版面配置：

```js
import AuthLayoutTemplate from '@/layouts/auth/auth-simple-layout'; // [tl! remove]
import AuthLayoutTemplate from '@/layouts/auth/auth-split-layout'; // [tl! add]
```


<a name="svelte-customization"></a>
### Svelte

我們的 Svelte 入門套件是使用 Inertia 3、Svelte 5、Tailwind 和 [shadcn-svelte](https://www.shadcn-svelte.com/) 所打造。與我們所有的入門套件一樣，所有的後端與前端程式碼都存在於您的應用程式中，以實現完全的自訂。

大部分的前端程式碼都位於 `resources/js` 目錄中。您可以自由修改任何程式碼，以自訂應用程式的外觀與行為：

```text
resources/js/
├── components/    # Reusable Svelte components
├── layouts/       # Application layouts
├── lib/           # Utility functions and configuration and Svelte rune modules
├── pages/         # Page components
└── types/         # TypeScript definitions
```

若要發布額外的 shadcn-svelte 元件，請先[找到您想要發布的元件](https://www.shadcn-svelte.com)。然後，使用 `npx` 發布該元件：

```shell
npx shadcn-svelte@latest add switch
```

在這個範例中，該命令會將 Switch 元件發布至 `resources/js/components/ui/switch/switch.svelte`。元件發布後，您就可以在任何頁面中使用它：

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

Svelte 入門套件包含兩種不同的主要版面配置供您選擇：「sidebar」版面配置和「header」版面配置。側邊欄版面配置是預設值，但您可以透過修改應用程式 `resources/js/layouts/AppLayout.svelte` 檔案頂部所匯入的版面配置，切換為頁首版面配置：

```js
import AppLayout from '@/layouts/app/AppSidebarLayout.svelte'; // [tl! remove]
import AppLayout from '@/layouts/app/AppHeaderLayout.svelte'; // [tl! add]
```


<a name="svelte-sidebar-variants"></a>
#### 側邊欄變體

側邊欄版面配置包含三種不同的變體：預設的 sidebar 變體、「inset」變體以及「floating」變體。您可以透過修改 `resources/js/components/AppSidebar.svelte` 元件來選擇您最喜歡的變體：

```text
<Sidebar collapsible="icon" variant="sidebar"> [tl! remove]
<Sidebar collapsible="icon" variant="inset"> [tl! add]
```


<a name="svelte-authentication-page-layout-variants"></a>
#### 認證頁面版面配置變體

Svelte 入門套件隨附的認證頁面（例如登入頁面和註冊頁面）也提供三種不同的版面配置變體：「simple」、「card」和「split」。

若要變更您的認證版面配置，請修改應用程式 `resources/js/layouts/AuthLayout.svelte` 檔案頂部所匯入的版面配置：

```js
import AuthLayout from '@/layouts/auth/AuthSimpleLayout.svelte'; // [tl! remove]
import AuthLayout from '@/layouts/auth/AuthSplitLayout.svelte'; // [tl! add]
```

<a name="vue-customization"></a>
### Vue

我們的 Vue 入門套件採用 Inertia 3、Vue 3 Composition API、Tailwind 與 [shadcn-vue](https://www.shadcn-vue.com/) 打造。就像我們所有的入門套件一樣，所有的後端與前端程式碼都存在於您的應用程式中，以便您進行完全自訂。

大部分的前端程式碼位於 `resources/js` 目錄中。您可以自由修改任何程式碼來自訂應用程式的外觀與行為：

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

在此範例中，該命令將發布 Switch 元件至 `resources/js/components/ui/Switch.vue`。元件發布後，您就可以在任何頁面中使用它：

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

Vue 入門套件包含兩種不同的主要版面配置供您選擇："sidebar"（側邊欄）版面配置與 "header"（頁首）版面配置。側邊欄版面配置是預設值，但您可以透過修改應用程式中 `resources/js/layouts/AppLayout.vue` 檔案頂部所匯入的版面配置來切換為頁首版面配置：

```js
import AppLayout from '@/layouts/app/AppSidebarLayout.vue'; // [tl! remove]
import AppLayout from '@/layouts/app/AppHeaderLayout.vue'; // [tl! add]
```


<a name="vue-sidebar-variants"></a>
#### 側邊欄變體

側邊欄版面配置包含三種不同的變體：預設側邊欄變體、"inset" 變體與 "floating" 變體。您可以透過修改 `resources/js/components/AppSidebar.vue` 元件來選擇您最喜歡的變體：

```text
<Sidebar collapsible="icon" variant="sidebar"> [tl! remove]
<Sidebar collapsible="icon" variant="inset"> [tl! add]
```


<a name="vue-authentication-page-layout-variants"></a>
#### 認證頁面版面配置變體

Vue 入門套件中包含的認證頁面（例如登入頁面與註冊頁面）也提供了三種不同的版面配置變體："simple"、"card" 與 "split"。

若要變更您的認證版面配置，請修改應用程式中 `resources/js/layouts/AuthLayout.vue` 檔案頂部所匯入的版面配置：

```js
import AuthLayout from '@/layouts/auth/AuthSimpleLayout.vue'; // [tl! remove]
import AuthLayout from '@/layouts/auth/AuthSplitLayout.vue'; // [tl! add]
```


<a name="livewire-customization"></a>
### Livewire

我們的 Livewire 入門套件採用 Livewire 4、Tailwind 與 [Flux UI](https://fluxui.dev/) 打造。就像我們所有的入門套件一樣，所有的後端與前端程式碼都存在於您的應用程式中，以便您進行完全自訂。

大部分的前端程式碼位於 `resources/views` 目錄中。您可以自由修改任何程式碼來自訂應用程式的外觀與行為：

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

Livewire 入門套件包含兩種不同的主要版面配置供您選擇："sidebar"（側邊欄）版面配置與 "header"（頁首）版面配置。側邊欄版面配置是預設值，但您可以透過修改應用程式中 `resources/views/layouts/app.blade.php` 檔案所使用的版面配置來切換為頁首版面配置。此外，您應該為主要的 Flux 元件新增 `container` 屬性：

```blade
<x-layouts::app.header>
    <flux:main container>
        {{ $slot }}
    </flux:main>
</x-layouts::app.header>
```


<a name="livewire-authentication-page-layout-variants"></a>
#### 認證頁面版面配置變體

Livewire 入門套件中包含的認證頁面（例如登入頁面與註冊頁面）也提供了三種不同的版面配置變體："simple"、"card" 與 "split"。

若要變更您的認證版面配置，請修改應用程式中 `resources/views/layouts/auth.blade.php` 檔案所使用的版面配置：

```blade
<x-layouts::auth.split>
    {{ $slot }}
</x-layouts::auth.split>
```

<a name="authentication"></a>
## 認證

所有入門套件皆使用 [Laravel Fortify](/docs/{{version}}/fortify) 來處理認證。Fortify 提供了用於登入、註冊、密碼重設、Email 驗證等功能的路由、控制器和邏輯。

Fortify 會根據您應用程式的 `config/fortify.php` 設定檔中啟用的功能，自動註冊以下認證路由：

<div class="overflow-auto">

| 路由 | 方法 | 說明 |
| ---------------------------------- | ------ | ----------------------------------- |
| `/login` | `GET` | 顯示登入表單 |
| `/login` | `POST` | 認證使用者 |
| `/logout` | `POST` | 登出使用者 |
| `/register` | `GET` | 顯示註冊表單 |
| `/register` | `POST` | 建立新使用者 |
| `/forgot-password` | `GET` | 顯示密碼重設請求表單 |
| `/forgot-password` | `POST` | 發送密碼重設連結 |
| `/reset-password/{token}` | `GET` | 顯示密碼重設表單 |
| `/reset-password` | `POST` | 更新密碼 |
| `/email/verify` | `GET` | 顯示 Email 驗證提示 |
| `/email/verify/{id}/{hash}` | `GET` | 驗證 Email 地址 |
| `/email/verification-notification` | `POST` | 重新發送驗證 Email |
| `/user/confirm-password` | `GET` | 顯示密碼確認表單 |
| `/user/confirm-password` | `POST` | 確認密碼 |
| `/two-factor-challenge` | `GET` | 顯示雙重驗證挑戰表單 |
| `/two-factor-challenge` | `POST` | 驗證雙重驗證碼 |

</div>

您可以使用 `php artisan route:list` Artisan 指令來顯示應用程式中的所有路由。


<a name="enabling-and-disabling-features"></a>
### 啟用與停用功能

您可以在應用程式的 `config/fortify.php` 設定檔中控制要啟用哪些 Fortify 功能：

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

若要停用某項功能，請註解掉或從 `features` 陣列中移除該功能項目。例如，移除 `Features::registration()` 即可停用公開註冊功能。

當使用 [React](#react)、[Svelte](#svelte) 或 [Vue](#vue) 入門套件時，您還需要從前端程式碼中移除對已停用功能路由的任何引用。例如，如果您停用了 Email 驗證，您應該在 React、Svelte 或 Vue 元件中移除對 `verification` 路由的匯入和引用。這是因為這些入門套件使用 Wayfinder 進行型別安全的路由處理，該工具會在建置時產生路由定義。如果您引用了不再存在的路由，應用程式將無法成功建置。


<a name="customizing-actions"></a>
### 自訂使用者建立與密碼重設

當使用者註冊或重設密碼時，Fortify 會呼叫位於您應用程式 `app/Actions/Fortify` 目錄中的 Action 類別：

<div class="overflow-auto">

| 檔案 | 說明 |
| ----------------------------- | ------------------------------------- |
| `CreateNewUser.php` | 驗證並建立新使用者 |
| `ResetUserPassword.php` | 驗證並更新使用者密碼 |
| `PasswordValidationRules.php` | 定義密碼驗證規則 |

</div>

例如，若要自訂應用程式的註冊邏輯，您可以編輯 `CreateNewUser` Action：

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
### 雙重驗證

入門套件內建了雙重驗證 (2FA) 功能，允許使用者使用任何相容 TOTP 的驗證器應用程式來保護其帳號。2FA 在預設情況下會透過您應用程式 `config/fortify.php` 設定檔中的 `Features::twoFactorAuthentication()` 來啟用。

`confirm` 選項要求使用者在完全啟用 2FA 之前先驗證一次驗證碼，而 `confirmPassword` 則要求在啟用或停用 2FA 之前進行密碼確認。如需更多詳細資訊，請參閱 [Fortify 的雙重驗證說明文件](/docs/{{version}}/fortify#two-factor-authentication)。


<a name="rate-limiting"></a>
### 速率限制

速率限制可以防止暴力破解和重複的登入嘗試癱瘓您的認證端點。您可以在應用程式的 `FortifyServiceProvider` 中自訂 Fortify 的速率限制行為：

```php
use Illuminate\Support\Facades\RateLimiter;
use Illuminate\Cache\RateLimiting\Limit;

RateLimiter::for('login', function ($request) {
    return Limit::perMinute(5)->by($request->email.$request->ip());
});
```


<a name="teams"></a>
## 團隊

React、Svelte、Vue 和 Livewire 入門套件也可以在建立時包含團隊功能支援。當啟用團隊功能時，每個使用者可以屬於一個或多個團隊，並擁有一個當前團隊。在註冊期間，新使用者會自動分配到一個個人團隊。入門套件還包含了用於建立團隊、切換團隊、邀請成員以及更新團隊詳細資訊的團隊管理畫面。

當路由的作用域限定為當前團隊時，當前團隊的代稱 (slug) 會包含在 URL 中。例如，儀表板路由會變成 `/{current_team}/dashboard`，而團隊管理頁面則使用像是 `settings/teams/{team}` 的路由。當使用 `{current_team}` 和 `{team}` 路由參數時，入門套件會自動確保已認證的使用者屬於所請求的團隊，然後才允許存取該路由。

為了讓產生具團隊意識的 URL 更加方便，入門套件為已認證使用者的當前團隊註冊了預設 URL 參數。這使得對 `route('dashboard')` 等輔助函數的呼叫能夠自動包含當前團隊的代稱。當使用者登入、註冊或切換團隊時，入門套件會更新當前團隊並重新整理這些預設 URL 參數，以便產生的連結繼續使用正確的團隊上下文。

在建立或重新命名團隊時，入門套件還能防止使用者選擇預留名稱，避免產生不安全或衝突的路由片段。例如，不能使用與 `settings`、`login` 或 `dashboard` 等路由字首相衝突的名稱。

<a name="workos"></a>
## WorkOS AuthKit 認證

預設情況下，React、Svelte、Vue 和 Livewire 入門套件皆使用 Laravel 內建的認證系統，提供登入、註冊、密碼重設、電子郵件驗證等功能。此外，我們還為每個入門套件提供了由 [WorkOS AuthKit](https://authkit.com) 強力驅動的版本，提供：

<div class="content-list" markdown="1">

- 社群登入認證 (Google、Microsoft、GitHub 與 Apple)
- Passkey 驗證
- 基於電子郵件的「Magic Auth」
- 單一簽入 (SSO)

</div>

使用 WorkOS 作為您的認證提供者[需要一個 WorkOS 帳號](https://workos.com)。WorkOS 為每月活躍使用者少於 100 萬的應用程式提供免費認證。

若要使用 WorkOS AuthKit 作為您應用程式的認證提供者，請在透過 `laravel new` 建立新的入門套件應用程式時選擇 WorkOS 選項。


<a name="configuring-your-workos-starter-kit"></a>
### 設定您的 WorkOS 入門套件

在使用 WorkOS 驅動的入門套件建立新應用程式後，您應該在應用程式的 `.env` 檔案中設定 `WORKOS_CLIENT_ID`、`WORKOS_API_KEY` 和 `WORKOS_REDIRECT_URL` 環境變數。這些變數應該與 WorkOS 主控台中為您的應用程式所提供的數值一致：

```ini
WORKOS_CLIENT_ID=your-client-id
WORKOS_API_KEY=your-api-key
WORKOS_REDIRECT_URL="${APP_URL}/authenticate"
```

此外，您應該在 WorkOS 主控台中設定應用程式的首頁 URL。當使用者從您的應用程式登出後，會被重導向至此 URL。


<a name="configuring-authkit-authentication-methods"></a>
#### 設定 AuthKit 認證方式

當使用 WorkOS 驅動的入門套件時，我們建議您在應用程式的 WorkOS AuthKit 設定中停用「Email + Password」認證，讓使用者僅能透過社群認證提供者、Passkey、「Magic Auth」與 SSO 進行認證。這可以讓您的應用程式完全免去處理使用者密碼的負擔。


<a name="configuring-authkit-session-timeouts"></a>
#### 設定 AuthKit Session 過期時間

此外，我們建議您將 WorkOS AuthKit 的 Session 閒置過期時間設定為與 Laravel 應用程式所設定的 Session 過期時間閾值一致（通常為兩小時）。


<a name="inertia-ssr"></a>
### Inertia SSR

React、Svelte 和 Vue 入門套件皆相容於 Inertia 的[伺服器端渲染 (SSR)](https://inertiajs.com/server-side-rendering) 功能。若要為您的應用程式建置相容於 Inertia SSR 的打包檔，請執行 `build:ssr` 指令：

```shell
npm run build:ssr
```

為了方便起見，我們也提供了 `composer dev:ssr` 指令。此指令會在為您的應用程式建置相容於 SSR 的打包檔後，啟動 Laravel 開發伺服器與 Inertia SSR 伺服器，讓您能夠在本機使用 Inertia 的伺服器端渲染引擎測試您的應用程式：

```shell
composer dev:ssr
```


<a name="community-maintained-starter-kits"></a>
### 社群維護的入門套件

當使用 Laravel 安裝程式建立新的 Laravel 應用程式時，您可以將 Packagist 上可用的任何社群維護入門套件提供給 `--using` 旗標：

```shell
laravel new my-app --using=example/starter-kit
```


<a name="creating-starter-kits"></a>
#### 建立入門套件

若要確保您的入門套件能供其他人使用，您需要將其發布至 [Packagist](https://packagist.org)。您的入門套件應該在其 `.env.example` 檔案中定義所需的環境變數，並且任何必要的安裝後執行指令都應該列於該入門套件 `composer.json` 檔案中的 `post-create-project-cmd` 陣列內。


<a name="faqs"></a>
### 常見問題


<a name="faq-upgrade"></a>
#### 我該如何升級？

每個入門套件都為您的下一個應用程式提供了一個穩固的起點。擁有程式碼的完全掌控權，您可以依照自己的構想去修改、自訂並建置您的應用程式。因此，您不需要去更新入門套件本身。


<a name="faq-enable-email-verification"></a>
#### 我該如何啟用電子郵件驗證？

若要新增電子郵件驗證，可以在 `App/Models/User.php` 模型中取消註解 `MustVerifyEmail` 的匯入，並確保該模型實作了 `MustVerifyEmail` 介面：

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

註冊後，使用者將會收到一封驗證電子郵件。若要在使用者的電子郵件地址驗證之前限制對特定路由的存取，請將 `verified` 中介層新增至這些路由：

```php
Route::middleware(['auth', 'verified'])->group(function () {
    Route::get('dashboard', function () {
        return Inertia::render('dashboard');
    })->name('dashboard');
});
```

> [!NOTE]
> 當使用入門套件的 [WorkOS](#workos) 版本時，不需要進行電子郵件驗證。


<a name="faq-modify-email-template"></a>
#### 我該如何修改預設的電子郵件範本？

您可能希望自訂預設的電子郵件範本，以更好地符合您應用程式的品牌形象。若要修改此範本，您應該透過以下指令將電子郵件視圖發布至您的應用程式：

```shell
php artisan vendor:publish --tag=laravel-mail
```

這將會在 `resources/views/vendor/mail` 中產生數個檔案。您可以修改這些檔案中的任何一個，以及 `resources/views/vendor/mail/themes/default.css` 檔案，來變更預設電子郵件範本的外觀與樣式。