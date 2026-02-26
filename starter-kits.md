# Starter Kits

- [簡介](#introduction)
- [使用入門套件建立應用程式](#creating-an-application)
- [可用的入門套件](#available-starter-kits)
    - [React](#react)
    - [Svelte](#svelte)
    - [Vue](#vue)
    - [Livewire](#livewire)
- [入門套件客製化](#starter-kit-customization)
    - [React](#react-customization)
    - [Svelte](#svelte-customization)
    - [Vue](#vue-customization)
    - [Livewire](#livewire-customization)
- [身分驗證](#authentication)
    - [啟用與停用功能](#enabling-and-disabling-features)
    - [客製化使用者建立與密碼重設](#customizing-actions)
    - [雙重身分驗證](#two-factor-authentication)
    - [速率限制](#rate-limiting)
- [WorkOS AuthKit 身分驗證](#workos)
- [Inertia SSR](#inertia-ssr)
- [社群維護的入門套件](#community-maintained-starter-kits)
- [常見問題](#faqs)

<a name="introduction"></a>
## 簡介

為了讓您在建立新的 Laravel 應用程式時能有個良好的開端，我們很高興能提供[應用程式入門套件](https://laravel.com/starter-kits)。這些入門套件能讓您快速開始構建下一個 Laravel 應用程式，並包含了註冊與驗證應用程式使用者所需的路由、控制器與視圖。這些入門套件使用 [Laravel Fortify](/docs/{{version}}/fortify) 來提供身分驗證。

雖然我們歡迎您使用這些入門套件，但這並非強制性的。您也可以透過安裝全新的一份 Laravel，自由地從頭開始構建自己的應用程式。無論採用哪種方式，我們相信您都能打造出優秀的作品！


<a name="creating-an-application"></a>
## 使用入門套件建立應用程式

要使用我們的入門套件建立新的 Laravel 應用程式，您應該先[安裝 PHP 與 Laravel CLI 工具](/docs/{{version}}/installation#installing-php)。如果您已經安裝了 PHP 與 Composer，則可以透過 Composer 安裝 Laravel 安裝程式 CLI 工具：

```shell
composer global require laravel/installer
```

然後，使用 Laravel 安裝程式 CLI 建立新的 Laravel 應用程式。Laravel 安裝程式將引導您選擇偏好的入門套件：

```shell
laravel new my-app
```

建立 Laravel 應用程式後，您只需要透過 NPM 安裝其前端依賴項目，並啟動 Laravel 開發伺服器：

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

我們的 React 入門套件為使用 [Inertia](https://inertiajs.com) 建構具有 React 前端的 Laravel 應用程式提供了一個強大且現代的起點。

Inertia 允許您使用傳統的伺服器端路由和控制器來構建現代的單頁面 React 應用程式。這讓您可以同時享受 React 的前端威力，以及 Laravel 驚人的後端生產力和極速的 Vite 編譯。

React 入門套件使用了 React 19、TypeScript、Tailwind 以及 [shadcn/ui](https://ui.shadcn.com) 元件庫。


<a name="svelte"></a>
### Svelte

我們的 Svelte 入門套件為使用 [Inertia](https://inertiajs.com) 建構具有 Svelte 前端的 Laravel 應用程式提供了一個強大且現代的起點。

Inertia 允許您使用傳統的伺服器端路由和控制器來構建現代的單頁面 Svelte 應用程式。這讓您可以同時享受 Svelte 的前端威力，以及 Laravel 驚人的後端生產力和極速的 Vite 編譯。

Svelte 入門套件使用了 Svelte 5、TypeScript、Tailwind 以及 [shadcn-svelte](https://www.shadcn-svelte.com/) 元件庫。


<a name="vue"></a>
### Vue

我們的 Vue 入門套件為使用 [Inertia](https://inertiajs.com) 建構具有 Vue 前端的 Laravel 應用程式提供了一個絕佳的起點。

Inertia 允許您使用傳統的伺服器端路由和控制器來構建現代的單頁面 Vue 應用程式。這讓您可以同時享受 Vue 的前端威力，以及 Laravel 驚人的後端生產力和極速的 Vite 編譯。

Vue 入門套件使用了 Vue Composition API、TypeScript、Tailwind 以及 [shadcn-vue](https://www.shadcn-vue.com/) 元件庫。


<a name="livewire"></a>
### Livewire

我們的 Livewire 入門套件為建構具有 [Laravel Livewire](https://livewire.laravel.com) 前端的 Laravel 應用程式提供了完美的起點。

Livewire 是一種僅使用 PHP 即可建構動態且具響應性前端 UI 的強大方式。對於主要使用 Blade 模板，且正在尋找 React、Svelte 和 Vue 等 JavaScript 驅動 SPA 框架的更簡單替代方案的團隊來說，它非常適合。

Livewire 入門套件使用了 Livewire、Tailwind 以及 [Flux UI](https://fluxui.dev) 元件庫。

<a name="starter-kit-customization"></a>
## 入門套件客製化


<a name="react-customization"></a>
### React

我們的 React 入門套件是使用 Inertia 2、React 19、Tailwind 4 以及 [shadcn/ui](https://ui.shadcn.com) 建構的。與我們所有的入門套件一樣，所有的後端與前端程式碼都存在於您的應用程式中，以實現完整的客製化。

大部分的前端程式碼位於 `resources/js` 目錄中。您可以隨意修改任何程式碼，以客製化應用程式的外觀與行為：

```text
resources/js/
├── components/    # Reusable React components
├── hooks/         # React hooks
├── layouts/       # Application layouts
├── lib/           # Utility functions and configuration
├── pages/         # Page components
└── types/         # TypeScript definitions
```

若要發布額外的 shadcn 元件，請先[尋找您想要發布的元件](https://ui.shadcn.com)。接著，使用 `npx` 發布該元件：

```shell
npx shadcn@latest add switch
```

在此範例中，該命令會將 Switch 元件發布到 `resources/js/components/ui/switch.tsx`。一旦元件發布完成，您就可以在任何頁面中使用它：

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
#### 可用的佈局

React 入門套件包含兩種不同的主要佈局供您選擇：「側邊欄 (sidebar)」佈局與「標頭 (header)」佈局。側邊欄佈局是預設值，但您可以透過修改應用程式 `resources/js/layouts/app-layout.tsx` 檔案頂端匯入的佈局來切換為標頭佈局：

```js
import AppLayoutTemplate from '@/layouts/app/app-sidebar-layout'; // [tl! remove]
import AppLayoutTemplate from '@/layouts/app/app-header-layout'; // [tl! add]
```


<a name="react-sidebar-variants"></a>
#### 側邊欄變體

側邊欄佈局包含三種不同的變體：預設側邊欄變體、「內縮 (inset)」變體以及「懸浮 (floating)」變體。您可以透過修改 `resources/js/components/app-sidebar.tsx` 元件來選擇您最喜歡的變體：

```text
<Sidebar collapsible="icon" variant="sidebar"> [tl! remove]
<Sidebar collapsible="icon" variant="inset"> [tl! add]
```


<a name="react-authentication-page-layout-variants"></a>
#### 身分驗證頁面佈局變體

React 入門套件包含的身分驗證頁面（例如登入頁面與註冊頁面）也提供了三種不同的佈局變體：「簡單 (simple)」、「卡片 (card)」與「分割 (split)」。

若要變更您的身分驗證佈局，請修改應用程式 `resources/js/layouts/auth-layout.tsx` 檔案頂端匯入的佈局：

```js
import AuthLayoutTemplate from '@/layouts/auth/auth-simple-layout'; // [tl! remove]
import AuthLayoutTemplate from '@/layouts/auth/auth-split-layout'; // [tl! add]
```


<a name="svelte-customization"></a>
### Svelte

我們的 Svelte 入門套件是使用 Inertia 2、Svelte 5、Tailwind 以及 [shadcn-svelte](https://www.shadcn-svelte.com/) 建構的。與我們所有的入門套件一樣，所有的後端與前端程式碼都存在於您的應用程式中，以實現完整的客製化。

大部分的前端程式碼位於 `resources/js` 目錄中。您可以隨意修改任何程式碼，以客製化應用程式的外觀與行為：

```text
resources/js/
├── components/    # Reusable Svelte components
├── layouts/       # Application layouts
├── lib/           # Utility functions and configuration and Svelte rune modules
├── pages/         # Page components
└── types/         # TypeScript definitions
```

若要發布額外的 shadcn-svelte 元件，請先[尋找您想要發布的元件](https://www.shadcn-svelte.com)。接著，使用 `npx` 發布該元件：

```shell
npx shadcn-svelte@latest add switch
```

在此範例中，該命令會將 Switch 元件發布到 `resources/js/components/ui/switch/switch.svelte`。一旦元件發布完成，您就可以在任何頁面中使用它：

```svelte
<script lang="ts">
    import { Switch } from '@/components/ui/switch'
</script>

<div>
    <Switch />
</div>
```


<a name="svelte-available-layouts"></a>
#### 可用的佈局

Svelte 入門套件包含兩種不同的主要佈局供您選擇：「側邊欄 (sidebar)」佈局與「標頭 (header)」佈局。側邊欄佈局是預設值，但您可以透過修改應用程式 `resources/js/layouts/AppLayout.svelte` 檔案頂端匯入的佈局來切換為標頭佈局：

```js
import AppLayout from '@/layouts/app/AppSidebarLayout.svelte'; // [tl! remove]
import AppLayout from '@/layouts/app/AppHeaderLayout.svelte'; // [tl! add]
```


<a name="svelte-sidebar-variants"></a>
#### 側邊欄變體

側邊欄佈局包含三種不同的變體：預設側邊欄變體、「內縮 (inset)」變體以及「懸浮 (floating)」變體。您可以透過修改 `resources/js/components/AppSidebar.svelte` 元件來選擇您最喜歡的變體：

```text
<Sidebar collapsible="icon" variant="sidebar"> [tl! remove]
<Sidebar collapsible="icon" variant="inset"> [tl! add]
```


<a name="svelte-authentication-page-layout-variants"></a>
#### 身分驗證頁面佈局變體

Svelte 入門套件包含的身分驗證頁面（例如登入頁面與註冊頁面）也提供了三種不同的佈局變體：「簡單 (simple)」、「卡片 (card)」與「分割 (split)」。

若要變更您的身分驗證佈局，請修改應用程式 `resources/js/layouts/AuthLayout.svelte` 檔案頂端匯入的佈局：

```js
import AuthLayout from '@/layouts/auth/AuthSimpleLayout.svelte'; // [tl! remove]
import AuthLayout from '@/layouts/auth/AuthSplitLayout.svelte'; // [tl! add]
```

<a name="vue-customization"></a>
### Vue

我們的 Vue 入門套件是使用 Inertia 2、Vue 3 Composition API、Tailwind 以及 [shadcn-vue](https://www.shadcn-vue.com/) 構建的。與我們所有的入門套件一樣，所有的後端與前端程式碼都存在於您的應用程式中，以允許完全客製化。

大部分的前端程式碼位於 `resources/js` 目錄中。您可以自由修改任何程式碼，以客製化應用程式的外觀與行為：

```text
resources/js/
├── components/    # Reusable Vue components
├── composables/   # Vue composables / hooks
├── layouts/       # Application layouts
├── lib/           # Utility functions and configuration
├── pages/         # Page components
└── types/         # TypeScript definitions
```

若要發佈額外的 shadcn-vue 元件，請先[尋找您想要發佈的元件](https://www.shadcn-vue.com)。然後，使用 `npx` 發佈該元件：

```shell
npx shadcn-vue@latest add switch
```

在此範例中，該命令將會把 Switch 元件發佈到 `resources/js/components/ui/Switch.vue`。元件發佈後，您可以在任何頁面中使用它：

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
#### 可用的佈局

Vue 入門套件包含兩種不同的主要佈局供您選擇：「側邊欄 (sidebar)」佈局與「頁首 (header)」佈局。側邊欄佈局是預設值，但您可以透過修改應用程式 `resources/js/layouts/AppLayout.vue` 檔案頂部匯入的佈局來切換到頁首佈局：

```js
import AppLayout from '@/layouts/app/AppSidebarLayout.vue'; // [tl! remove]
import AppLayout from '@/layouts/app/AppHeaderLayout.vue'; // [tl! add]
```


<a name="vue-sidebar-variants"></a>
#### 側邊欄變體

側邊欄佈局包含三種不同的變體：預設側邊欄變體、「內嵌 (inset)」變體以及「懸浮 (floating)」變體。您可以透過修改 `resources/js/components/AppSidebar.vue` 元件來選擇您最喜歡的變體：

```text
<Sidebar collapsible="icon" variant="sidebar"> [tl! remove]
<Sidebar collapsible="icon" variant="inset"> [tl! add]
```


<a name="vue-authentication-page-layout-variants"></a>
#### 身分驗證頁面佈局變體

Vue 入門套件隨附的身分驗證頁面（例如登入頁面與註冊頁面）也提供了三種不同的佈局變體：「簡單 (simple)」、「卡片 (card)」與「分割 (split)」。

若要更改您的身分驗證佈局，請修改應用程式 `resources/js/layouts/AuthLayout.vue` 檔案頂部匯入的佈局：

```js
import AuthLayout from '@/layouts/auth/AuthSimpleLayout.vue'; // [tl! remove]
import AuthLayout from '@/layouts/auth/AuthSplitLayout.vue'; // [tl! add]
```


<a name="livewire-customization"></a>
### Livewire

我們的 Livewire 入門套件是使用 Livewire 4、Tailwind 與 [Flux UI](https://fluxui.dev/) 構建的。與我們所有的入門套件一樣，所有的後端與前端程式碼都存在於您的應用程式中，以允許完全客製化。

大部分的前端程式碼位於 `resources/views` 目錄中。您可以自由修改任何程式碼，以客製化應用程式的外觀與行為：

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
#### 可用的佈局

Livewire 入門套件包含兩種不同的主要佈局供您選擇：「側邊欄 (sidebar)」佈局與「頁首 (header)」佈局。側邊欄佈局是預設值，但您可以透過修改應用程式 `resources/views/layouts/app.blade.php` 檔案所使用的佈局來切換到頁首佈局。此外，您應該在主要的 Flux 元件中加入 `container` 屬性：

```blade
<x-layouts::app.header>
    <flux:main container>
        {{ $slot }}
    </flux:main>
</x-layouts::app.header>
```


<a name="livewire-authentication-page-layout-variants"></a>
#### 身分驗證頁面佈局變體

Livewire 入門套件隨附的身分驗證頁面（例如登入頁面與註冊頁面）也提供了三種不同的佈局變體：「簡單 (simple)」、「卡片 (card)」與「分割 (split)」。

若要更改您的身分驗證佈局，請修改應用程式 `resources/views/layouts/auth.blade.php` 檔案所使用的佈局：

```blade
<x-layouts::auth.split>
    {{ $slot }}
</x-layouts::auth.split>
```

<a name="authentication"></a>
## 身分驗證

所有入門套件都使用 [Laravel Fortify](/docs/{{version}}/fortify) 來處理身分驗證。Fortify 提供了登入、註冊、密碼重設、電子郵件驗證等所需的路由、控制器與邏輯。

Fortify 會根據您應用程式中 `config/fortify.php` 設定檔所啟用的功能，自動註冊以下身分驗證路由：

| 路由 | 方法 | 說明 |
| ---------------------------------- | ------ | ----------------------------------- |
| `/login` | `GET` | 顯示登入表單 |
| `/login` | `POST` | 驗證使用者 |
| `/logout` | `POST` | 將使用者登出 |
| `/register` | `GET` | 顯示註冊表單 |
| `/register` | `POST` | 建立新使用者 |
| `/forgot-password` | `GET` | 顯示密碼重設請求表單 |
| `/forgot-password` | `POST` | 發送密碼重設連結 |
| `/reset-password/{token}` | `GET` | 顯示密碼重設表單 |
| `/reset-password` | `POST` | 更新密碼 |
| `/email/verify` | `GET` | 顯示電子郵件驗證通知 |
| `/email/verify/{id}/{hash}` | `GET` | 驗證電子郵件地址 |
| `/email/verification-notification` | `POST` | 重新發送驗證電子郵件 |
| `/user/confirm-password` | `GET` | 顯示密碼確認表單 |
| `/user/confirm-password` | `POST` | 確認密碼 |
| `/two-factor-challenge` | `GET` | 顯示 2FA 挑戰表單 |
| `/two-factor-challenge` | `POST` | 驗證 2FA 代碼 |

`php artisan route:list` 這個 Artisan 指令可用於顯示應用程式中的所有路由。


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

若要停用某個功能，請註解掉或從 `features` 陣列中移除該功能項目。例如，移除 `Features::registration()` 即可停用公開註冊。

當使用 [React](#react)、 [Svelte](#svelte) 或 [Vue](#vue) 入門套件時，您還需要移除前端程式碼中對已停用功能路由的任何引用。例如，如果您停用了電子郵件驗證，則應移除 React、Svelte 或 Vue 元件中對 `verification` 路由的導入 (Import) 與引用。這是必要的，因為這些入門套件使用 Wayfinder 進行型別安全路由，它會在建置 (Build) 時生成路由定義。如果您引用了不再存在的路由，您的應用程式將無法建置。


<a name="customizing-actions"></a>
### 客製化使用者建立與密碼重設

當使用者註冊或重設密碼時，Fortify 會呼叫位於應用程式 `app/Actions/Fortify` 目錄下的 Action 類別：

| 檔案 | 說明 |
| ----------------------------- | ------------------------------------- |
| `CreateNewUser.php` | 驗證並建立新使用者 |
| `ResetUserPassword.php` | 驗證並更新使用者密碼 |
| `PasswordValidationRules.php` | 定義密碼驗證規則 |

例如，若要客製化應用程式的註冊邏輯，您應該編輯 `CreateNewUser` Action：

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
### 雙重身分驗證

入門套件包含內建的雙重身分驗證 (2FA)，允許使用者使用任何相容 TOTP 的驗證器 App 來確保帳號安全。2FA 預設透過您應用程式 `config/fortify.php` 設定檔中的 `Features::twoFactorAuthentication()` 啟用。

`confirm` 選項要求使用者在完全啟用 2FA 之前驗證代碼，而 `confirmPassword` 則要求在啟用或停用 2FA 之前進行密碼確認。更多詳細資訊，請參閱 [Fortify 的雙重身分驗證文件](/docs/{{version}}/fortify#two-factor-authentication)。


<a name="rate-limiting"></a>
### 速率限制

速率限制 (Rate limiting) 可防止暴力破解與重複的登入嘗試癱瘓您的身分驗證端點。您可以在應用程式的 `FortifyServiceProvider` 中客製化 Fortify 的速率限制行為：

```php
use Illuminate\Support\Facades\RateLimiter;
use Illuminate\Cache\RateLimiting\Limit;

RateLimiter::for('login', function ($request) {
    return Limit::perMinute(5)->by($request->email.$request->ip());
});
```

<a name="workos"></a>
## WorkOS AuthKit 身分驗證

預設情況下，React、Svelte、Vue 和 Livewire 入門套件都利用 Laravel 內建的身分驗證系統來提供登入、註冊、密碼重設、電子郵件驗證等功能。此外，我們也為每個入門套件提供了一個由 [WorkOS AuthKit](https://authkit.com) 驅動的變體，它提供了：

<div class="content-list" markdown="1">

- 社群身分驗證 (Google, Microsoft, GitHub, and Apple)
- 通行密鑰 (Passkey) 身分驗證
- 基於電子郵件的「魔術連結驗證 (Magic Auth)」
- SSO

</div>

使用 WorkOS 作為您的身分驗證提供者 [需要一個 WorkOS 帳號](https://workos.com)。WorkOS 為每月活躍使用者達 100 萬的應用程式提供免費的身分驗證服務。

要使用 WorkOS AuthKit 作為您應用程式的身分驗證提供者，請在透過 `laravel new` 建立新的入門套件驅動應用程式時選擇 WorkOS 選項。


### 配置您的 WorkOS 入門套件

在使用 WorkOS 驅動的入門套件建立新應用程式後，您應該在應用程式的 `.env` 檔案中設定 `WORKOS_CLIENT_ID`、`WORKOS_API_KEY` 和 `WORKOS_REDIRECT_URL` 環境變數。這些變數應與您的 WorkOS 儀表板中為該應用程式提供的值相符：

```ini
WORKOS_CLIENT_ID=your-client-id
WORKOS_API_KEY=your-api-key
WORKOS_REDIRECT_URL="${APP_URL}/authenticate"
```

此外，您應該在 WorkOS 儀表板中配置應用程式首頁 URL。當使用者登出您的應用程式後，將會被重新導向至此 URL。


<a name="configuring-authkit-authentication-methods"></a>
#### 配置 AuthKit 身分驗證方法

使用 WorkOS 驅動的入門套件時，我們建議您在應用程式的 WorkOS AuthKit 配置設定中停用「Email + Password」身分驗證，僅允許使用者透過社群身分驗證提供者、通行密鑰、「魔術連結驗證」和 SSO 進行身分驗證。這可以讓您的應用程式完全避免處理使用者密碼。


<a name="configuring-authkit-session-timeouts"></a>
#### 配置 AuthKit 工作階段逾時

此外，我們建議您將 WorkOS AuthKit 的工作階段閒置逾時配置為與您 Laravel 應用程式配置的工作階段逾時閾值一致，通常為兩小時。


<a name="inertia-ssr"></a>
### Inertia SSR

React、Svelte 和 Vue 入門套件皆與 Inertia 的 [伺服器端渲染 (SSR)](https://inertiajs.com/server-side-rendering) 功能相容。要為您的應用程式建置相容 Inertia SSR 的 bundle，請執行 `build:ssr` 命令：

```shell
npm run build:ssr
```

為了方便起見，還提供了一個 `composer dev:ssr` 命令。此命令會在建置相容 SSR 的 bundle 後，啟動 Laravel 開發伺服器和 Inertia SSR 伺服器，讓您可以在本地使用 Inertia 的伺服器端渲染引擎測試您的應用程式：

```shell
composer dev:ssr
```


<a name="community-maintained-starter-kits"></a>
### 社群維護的入門套件

使用 Laravel 安裝程式建立新的 Laravel 應用程式時，您可以將 Packagist 上提供的任何社群維護入門套件提供給 `--using` 旗標：

```shell
laravel new my-app --using=example/starter-kit
```


<a name="creating-starter-kits"></a>
#### 建立入門套件

為了確保您的入門套件可供他人使用，您需要將其發佈到 [Packagist](https://packagist.org)。您的入門套件應在其 `.env.example` 檔案中定義所需的環境變數，且任何必要的安裝後命令都應列在入門套件 `composer.json` 檔案的 `post-create-project-cmd` 陣列中。


<a name="faqs"></a>
### 常見問題


<a name="faq-upgrade"></a>
#### 我該如何升級？

每個入門套件都為您的下一個應用程式提供了一個穩固的起點。由於您擁有程式碼的完全所有權，您可以完全按照自己的構想來調整、客製化並建置您的應用程式。因此，沒有必要更新入門套件本身。


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

註冊後，使用者將收到一封驗證電子郵件。若要在使用者的電子郵件地址驗證之前限制對某些路由的存取，請將 `verified` 中介層加入到路由中：

```php
Route::middleware(['auth', 'verified'])->group(function () {
    Route::get('dashboard', function () {
        return Inertia::render('dashboard');
    })->name('dashboard');
});
```

> [!NOTE]
> 使用 [WorkOS](#workos) 變體的入門套件時，不需要進行電子郵件驗證。


<a name="faq-modify-email-template"></a>
#### 我該如何修改預設的電子郵件範本？

您可能想要客製化預設的電子郵件範本，以更好地符合您應用程式的品牌形象。要修改此範本，您應該使用以下命令將電子郵件視圖發佈到您的應用程式：

```
php artisan vendor:publish --tag=laravel-mail
```

這將在 `resources/views/vendor/mail` 中產生多個檔案。您可以修改其中的任何檔案，以及 `resources/views/vendor/mail/themes/default.css` 檔案，來變更預設電子郵件範本的外觀與樣式。