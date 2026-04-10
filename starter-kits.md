# 快速啟動套件

- [簡介](#introduction)
- [使用快速啟動套件建立應用程式](#creating-an-application)
- [可用的快速啟動套件](#available-starter-kits)
    - [React](#react)
    - [Svelte](#svelte)
    - [Vue](#vue)
    - [Livewire](#livewire)
- [快速啟動套件自定義](#starter-kit-customization)
    - [React](#react-customization)
    - [Svelte](#svelte-customization)
    - [Vue](#vue-customization)
    - [Livewire](#livewire-customization)
- [認證](#authentication)
    - [啟用與停用功能](#enabling-and-disabling-features)
    - [自定義使用者建立與密碼重設](#customizing-actions)
    - [雙因子認證](#two-factor-authentication)
    - [速率限制](#rate-limiting)
- [團隊](#teams)
- [WorkOS AuthKit 認證](#workos)
- [Inertia SSR](#inertia-ssr)
- [社群維護的快速啟動套件](#community-maintained-starter-kits)
- [常見問題](#faqs)

<a name="introduction"></a>
## 簡介

為了讓您在建立新的 Laravel 應用程式時能快速上手，我們很高興提供 [應用程式快速啟動套件 (application starter kits)](https://laravel.com/starter-kits)。這些快速啟動套件能讓您在建立下一個 Laravel 應用程式時搶佔先機，其中包含了您註冊與認證應用程式使用者所需的路由、控制器與視圖。這些快速啟動套件使用 [Laravel Fortify](/docs/{{version}}/fortify) 來提供認證功能。

雖然歡迎您使用這些快速啟動套件，但它們並非強制要求。您可以透過簡單地安裝一個全新的 Laravel 複本，從零開始建立您自己的應用程式。無論採取哪種方式，我們相信您都會打造出很棒的作品！


<a name="creating-an-application"></a>
## 使用快速啟動套件建立應用程式

要使用我們的快速啟動套件之一來建立新的 Laravel 應用程式，您應該先 [安裝 PHP 與 Laravel CLI 工具](/docs/{{version}}/installation#installing-php)。如果您已經安裝了 PHP 與 Composer，可以透過 Composer 安裝 Laravel 安裝程式 CLI 工具：

```shell
composer global require laravel/installer
```

接著，使用 Laravel 安裝程式 CLI 建立新的 Laravel 應用程式。安裝程式會提示您選擇偏好的快速啟動套件：

```shell
laravel new my-app
```

建立 Laravel 應用程式後，您只需要透過 NPM 安裝其前端依賴並啟動 Laravel 開發伺服器：

```shell
cd my-app
npm install && npm run build
composer run dev
```

一旦啟動了 Laravel 開發伺服器，您就可以在網頁瀏覽器中透過 [http://localhost:8000](http://localhost:8000) 存取您的應用程式。


<a name="available-starter-kits"></a>
## 可用的快速啟動套件


<a name="react"></a>
### React

我們的 React 快速啟動套件為使用 [Inertia](https://inertiajs.com) 建立具有 React 前端的 Laravel 應用程式提供了一個強大且現代的起點。

Inertia 允許您使用傳統的伺服器端路由與控制器來構建現代的單頁 React 應用程式。這讓您能同時享受 React 的前端強大功能，以及 Laravel 驚人的後端開發生產力與極速的 Vite 編譯。

React 快速啟動套件使用了 React 19、TypeScript、Tailwind 以及 [shadcn/ui](https://ui.shadcn.com) 元件庫。


<a name="svelte"></a>
### Svelte

我們的 Svelte 快速啟動套件為使用 [Inertia](https://inertiajs.com) 建立具有 Svelte 前端的 Laravel 應用程式提供了一個強大且現代的起點。

Inertia 允許您使用傳統的伺服器端路由與控制器來構建現代的單頁 Svelte 應用程式。這讓您能同時享受 Svelte 的前端強大功能，以及 Laravel 驚人的後端開發生產力與極速的 Vite 編譯。

Svelte 快速啟動套件使用了 Svelte 5、TypeScript、Tailwind 以及 [shadcn-svelte](https://www.shadcn-svelte.com/) 元件庫。


<a name="vue"></a>
### Vue

我們的 Vue 快速啟動套件為使用 [Inertia](https://inertiajs.com) 建立具有 Vue 前端的 Laravel 應用程式提供了一個絕佳的起點。

Inertia 允許您使用傳統的伺服器端路由與控制器來構建現代的單頁 Vue 應用程式。這讓您能同時享受 Vue 的前端強大功能，以及 Laravel 驚人的後端開發生產力與極速的 Vite 編譯。

Vue 快速啟動套件使用了 Vue Composition API、TypeScript、Tailwind 以及 [shadcn-vue](https://www.shadcn-vue.com/) 元件庫。


<a name="livewire"></a>
### Livewire

我們的 Livewire 快速啟動套件為建立具有 [Laravel Livewire](https://livewire.laravel.com) 前端的 Laravel 應用程式提供了完美的起點。

Livewire 是一種僅使用 PHP 即可構建動態、響應式前端 UI 的強大方式。對於主要使用 Blade 模板並在尋找比 React、Svelte 和 Vue 等 JavaScript 驅動的 SPA 框架更簡單替代方案的團隊來說，這是一個非常適合的選擇。

Livewire 快速啟動套件使用了 Livewire、Tailwind 以及 [Flux UI](https://fluxui.dev) 元件庫。

<a name="starter-kit-customization"></a>
## 快速啟動套件自定義


<a name="react-customization"></a>
### React

我們的 React 快速啟動套件是使用 Inertia 2, React 19, Tailwind 4 和 [shadcn/ui](https://ui.shadcn.com) 構建的。與我們所有的快速啟動套件一樣，所有的後端和前端程式碼都存在於您的應用程式中，以便進行完全的自定義。

大部分的前端程式碼位於 `resources/js` 目錄中。您可以隨意修改任何程式碼以自定義應用程式的外觀和行為：

```text
resources/js/
├── components/    # Reusable React components
├── hooks/         # React hooks
├── layouts/       # Application layouts
├── lib/           # Utility functions and configuration
├── pages/         # Page components
└── types/         # TypeScript definitions
```

要發布額外的 shadcn 元件，請先 [尋找您想要發布的元件](https://ui.shadcn.com)。然後，使用 `npx` 發布該元件：

```shell
npx shadcn@latest add switch
```

在此範例中，該指令將把 Switch 元件發布到 `resources/js/components/ui/switch.tsx`。元件發布後，您可以在任何頁面中使用它：

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

React 快速啟動套件包含兩種不同的主要佈局供您選擇：「側邊欄 (sidebar)」佈局和「頁首 (header)」佈局。側邊欄佈局是預設選項，但您可以透過修改應用程式 `resources/js/layouts/app-layout.tsx` 檔案頂部導入的佈局來切換到頁首佈局：

```js
import AppLayoutTemplate from '@/layouts/app/app-sidebar-layout'; // [tl! remove]
import AppLayoutTemplate from '@/layouts/app/app-header-layout'; // [tl! add]
```


<a name="react-sidebar-variants"></a>
#### 側邊欄變體

側邊欄佈局包含三種不同的變體：預設側邊欄變體、「內嵌 (inset)」變體和「懸浮 (floating)」變體。您可以透過修改 `resources/js/components/app-sidebar.tsx` 元件來選擇您最喜歡的變體：

```text
<Sidebar collapsible="icon" variant="sidebar"> [tl! remove]
<Sidebar collapsible="icon" variant="inset"> [tl! add]
```


<a name="react-authentication-page-layout-variants"></a>
#### 認證頁面佈局變體

React 快速啟動套件包含的認證頁面（例如登入頁面和註冊頁面）也提供三種不同的佈局變體：「簡單 (simple)」、「卡片 (card)」和「分割 (split)」。

要更改您的認證佈局，請修改應用程式 `resources/js/layouts/auth-layout.tsx` 檔案頂部導入的佈局：

```js
import AuthLayoutTemplate from '@/layouts/auth/auth-simple-layout'; // [tl! remove]
import AuthLayoutTemplate from '@/layouts/auth/auth-split-layout'; // [tl! add]
```


<a name="svelte-customization"></a>
### Svelte

我們的 Svelte 快速啟動套件是使用 Inertia 2, Svelte 5, Tailwind 和 [shadcn-svelte](https://www.shadcn-svelte.com/) 構建的。與我們所有的快速啟動套件一樣，所有的後端和前端程式碼都存在於您的應用程式中，以便進行完全的自定義。

大部分的前端程式碼位於 `resources/js` 目錄中。您可以隨意修改任何程式碼以自定義應用程式的外觀和行為：

```text
resources/js/
├── components/    # Reusable Svelte components
├── layouts/       # Application layouts
├── lib/           # Utility functions and configuration and Svelte rune modules
├── pages/         # Page components
└── types/         # TypeScript definitions
```

要發布額外的 shadcn-svelte 元件，請先 [尋找您想要發布的元件](https://www.shadcn-svelte.com)。然後，使用 `npx` 發布該元件：

```shell
npx shadcn-svelte@latest add switch
```

在此範例中，該指令將把 Switch 元件發布到 `resources/js/components/ui/switch/switch.svelte`。元件發布後，您可以在任何頁面中使用它：

```svelte
<script lang="ts">
    import { Switch } from '@/components/ui/switch'
</script>

<div>
    <Switch />
</div>
```


<a name="svelte-available-layouts"></a>
#### 可用佈局

Svelte 快速啟動套件包含兩種不同的主要佈局供您選擇：「側邊欄 (sidebar)」佈局和「頁首 (header)」佈局。側邊欄佈局是預設選項，但您可以透過修改應用程式 `resources/js/layouts/AppLayout.svelte` 檔案頂部導入的佈局來切換到頁首佈局：

```js
import AppLayout from '@/layouts/app/AppSidebarLayout.svelte'; // [tl! remove]
import AppLayout from '@/layouts/app/AppHeaderLayout.svelte'; // [tl! add]
```


<a name="svelte-sidebar-variants"></a>
#### 側邊欄變體

側邊欄佈局包含三種不同的變體：預設側邊欄變體、「內嵌 (inset)」變體和「懸浮 (floating)」變體。您可以透過修改 `resources/js/components/AppSidebar.svelte` 元件來選擇您最喜歡的變體：

```text
<Sidebar collapsible="icon" variant="sidebar"> [tl! remove]
<Sidebar collapsible="icon" variant="inset"> [tl! add]
```


<a name="svelte-authentication-page-layout-variants"></a>
#### 認證頁面佈局變體

Svelte 快速啟動套件包含的認證頁面（例如登入頁面和註冊頁面）也提供三種不同的佈局變體：「簡單 (simple)」、「卡片 (card)」和「分割 (split)」。

要更改您的認證佈局，請修改應用程式 `resources/js/layouts/AuthLayout.svelte` 檔案頂部導入的佈局：

```js
import AuthLayout from '@/layouts/auth/AuthSimpleLayout.svelte'; // [tl! remove]
import AuthLayout from '@/layouts/auth/AuthSplitLayout.svelte'; // [tl! add]
```

<a name="vue-customization"></a>
### Vue

我們的 Vue 快速啟動套件是使用 Inertia 2, Vue 3 Composition API, Tailwind, 與 [shadcn-vue](https://www.shadcn-vue.com/) 構建而成。與我們所有的快速啟動套件一樣，所有的後端與前端程式碼都存在於您的應用程式中，以便您可以進行完全的自定義。

大部分的前端程式碼位於 `resources/js` 目錄中。您可以隨意修改任何程式碼來自定義應用程式的外觀與行為：

```text
resources/js/
├── components/    # Reusable Vue components
├── composables/   # Vue composables / hooks
├── layouts/       # Application layouts
├── lib/           # Utility functions and configuration
├── pages/         # Page components
└── types/         # TypeScript definitions
```

若要發布額外的 shadcn-vue 元件，請先 [尋找您想要發布的元件](https://www.shadcn-vue.com)。接著，使用 `npx` 發布該元件：

```shell
npx shadcn-vue@latest add switch
```

在此範例中，該指令會將 Switch 元件發布到 `resources/js/components/ui/Switch.vue`。一旦元件被發布，您就可以在任何頁面中使用它：

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

Vue 快速啟動套件提供兩種不同的主要佈局供您選擇：「側邊欄 (sidebar)」佈局與「頁首 (header)」佈局。側邊欄佈局為預設選項，但您可以透過修改應用程式 `resources/js/layouts/AppLayout.vue` 檔案頂端所匯入的佈局來切換至頁首佈局：

```js
import AppLayout from '@/layouts/app/AppSidebarLayout.vue'; // [tl! remove]
import AppLayout from '@/layouts/app/AppHeaderLayout.vue'; // [tl! add]
```


<a name="vue-sidebar-variants"></a>
#### 側邊欄變體

側邊欄佈局包含三種不同的變體：預設側邊欄變體、「內嵌 (inset)」變體與「浮動 (floating)」變體。您可以透過修改 `resources/js/components/AppSidebar.vue` 元件來選擇您最喜歡的變體：

```text
<Sidebar collapsible="icon" variant="sidebar"> [tl! remove]
<Sidebar collapsible="icon" variant="inset"> [tl! add]
```


<a name="vue-authentication-page-layout-variants"></a>
#### 認證頁面佈局變體

Vue 快速啟動套件中包含的認證頁面（例如登入頁面與註冊頁面）同樣提供三種不同的佈局變體：「簡單 (simple)」、「卡片 (card)」與「分割 (split)」。

若要更改您的認證佈局，請修改應用程式 `resources/js/layouts/AuthLayout.vue` 檔案頂端所匯入的佈局：

```js
import AuthLayout from '@/layouts/auth/AuthSimpleLayout.vue'; // [tl! remove]
import AuthLayout from '@/layouts/auth/AuthSplitLayout.vue'; // [tl! add]
```


<a name="livewire-customization"></a>
### Livewire

我們的 Livewire 快速啟動套件是使用 Livewire 4, Tailwind, 與 [Flux UI](https://fluxui.dev/) 構建而成。與我們所有的快速啟動套件一樣，所有的後端與前端程式碼都存在於您的應用程式中，以便您可以進行完全的自定義。

大部分的前端程式碼位於 `resources/views` 目錄中。您可以隨意修改任何程式碼來自定義應用程式的外觀與行為：

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

Livewire 快速啟動套件提供兩種不同的主要佈局供您選擇：「側邊欄 (sidebar)」佈局與「頁首 (header)」佈局。側邊欄佈局為預設選項，但您可以透過修改應用程式 `resources/views/layouts/app.blade.php` 檔案所使用的佈局來切換至頁首佈局。此外，您應該將 `container` 屬性添加到主 Flux 元件中：

```blade
<x-layouts::app.header>
    <flux:main container>
        {{ $slot }}
    </flux:main>
</x-layouts::app.header>
```


<a name="livewire-authentication-page-layout-variants"></a>
#### 認證頁面佈局變體

Livewire 快速啟動套件中包含的認證頁面（例如登入頁面與註冊頁面）同樣提供三種不同的佈局變體：「簡單 (simple)」、「卡片 (card)」與「分割 (split)」。

若要更改您的認證佈局，請修改應用程式 `resources/views/layouts/auth.blade.php` 檔案所使用的佈局：

```blade
<x-layouts::auth.split>
    {{ $slot }}
</x-layouts::auth.split>
```

<a name="authentication"></a>
## 認證

所有快速啟動套件都使用 [Laravel Fortify](/docs/{{version}}/fortify) 來處理認證。Fortify 提供了登入、註冊、密碼重設、電子郵件驗證等所需的路由、控制器與邏輯。

Fortify 會根據您應用程式中 `config/fortify.php` 設定檔所啟用的功能，自動註冊以下認證路由：

| 路由                              | 方法 | 描述                               |
| ---------------------------------- | ------ | ----------------------------------- |
| `/login`                           | `GET`    | 顯示登入表單                  |
| `/login`                           | `POST`   | 認證使用者                   |
| `/logout`                          | `POST`   | 使用者登出                        |
| `/register`                        | `GET`    | 顯示註冊表單           |
| `/register`                        | `POST`   | 建立新使用者                     |
| `/forgot-password`                 | `GET`    | 顯示密碼重設請求表單 |
| `/forgot-password`                 | `POST`   | 發送密碼重設連結            |
| `/reset-password/{token}`          | `GET`    | 顯示密碼重設表單         |
| `/reset-password`                  | `POST`   | 更新密碼                     |
| `/email/verify`                    | `GET`    | 顯示電子郵件驗證通知   |
| `/email/verify/{id}/{hash}`        | `GET`    | 驗證電子郵件地址                |
| `/email/verification-notification` | `POST`   | 重新發送驗證電子郵件           |
| `/user/confirm-password`           | `GET`    | 顯示密碼確認表單  |
| `/user/confirm-password`           | `POST`   | 確認密碼                    |
| `/two-factor-challenge`            | `GET`    | 顯示 2FA 驗證碼表單          |
| `/two-factor-challenge`            | `POST`   | 驗證 2FA 驗證碼                     |

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

若要停用某項功能，請將該功能項從 `features` 陣列中註解掉或移除。例如，移除 `Features::registration()` 以停用公開註冊。

當使用 [React](#react)、[Svelte](#svelte) 或 [Vue](#vue) 快速啟動套件時，您還需要在前端程式碼中移除對已停用功能路由的任何引用。例如，如果您停用了電子郵件驗證，則應移除 React、Svelte 或 Vue 元件中對 `verification` 路由的 imports 與引用。這是必要的，因為這些快速啟動套件使用 Wayfinder 進行型別安全的路由 (type-safe routing)，它會在建構時產生路由定義。如果您引用了不再存在的路由，您的應用程式將無法順利建構。


<a name="customizing-actions"></a>
### 自定義使用者建立與密碼重設

當使用者註冊或重設密碼時，Fortify 會調用位於應用程式 `app/Actions/Fortify` 目錄下的 action 類別：

| 檔案                          | 描述                           |
| ----------------------------- | ------------------------------------- |
| `CreateNewUser.php`           | 驗證並建立新使用者       |
| `ResetUserPassword.php`       | 驗證並更新使用者密碼  |
| `PasswordValidationRules.php` | 定義密碼驗證規則     |

例如，若要自定義應用程式的註冊邏輯，您應該編輯 `CreateNewUser` action：

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
### 雙因子認證

快速啟動套件內建了雙因子認證 (2FA)，允許使用者使用任何相容於 TOTP 的認證應用程式來保護其帳戶。2FA 預設透過應用程式 `config/fortify.php` 設定檔中的 `Features::twoFactorAuthentication()` 啟用。

`confirm` 選項要求使用者在完全啟用 2FA 之前必須驗證驗證碼，而 `confirmPassword` 則要求在啟用或停用 2FA 之前必須確認密碼。詳細資訊請參閱 [Fortify 的雙因子認證文件](/docs/{{version}}/fortify#two-factor-authentication)。


<a name="rate-limiting"></a>
### 速率限制

速率限制可防止暴力破解以及重複的登入嘗試對您的認證端點造成壓力。您可以在應用程式的 `FortifyServiceProvider` 中自定義 Fortify 的速率限制行為：

```php
use Illuminate\Support\Facades\RateLimiter;
use Illuminate\Cache\RateLimiting\Limit;

RateLimiter::for('login', function ($request) {
    return Limit::perMinute(5)->by($request->email.$request->ip());
});
```


<a name="teams"></a>
## 團隊

React、Svelte、Vue 與 Livewire 快速啟動套件在產生時也可以包含團隊支援。當啟用團隊功能時，每個使用者都屬於一個或多個團隊，並有一個目前團隊。在註冊期間，新使用者會自動被分配一個個人團隊。快速啟動套件還包含了建立團隊、切換團隊、邀請成員以及更新團隊詳細資訊的團隊管理畫面。

當路由被限制在目前團隊範圍內時，目前團隊的 slug 會包含在 URL 中。例如，儀表板路由會變成 `/{current_team}/dashboard`，而團隊管理頁面則使用如 `settings/teams/{team}` 這樣的路由。當使用 `{current_team}` 與 `{team}` 路由參數時，快速啟動套件會自動確保已認證的使用者屬於所請求的團隊，之後才允許存取該路由。

為了讓產生具備團隊感知 (team-aware) 的 URL 更加方便，快速啟動套件為已認證使用者的目前團隊註冊了 URL 預設值。這使得呼叫如 `route('dashboard')` 的輔助函式時會自動包含目前團隊的 slug。當使用者登入、註冊或切換團隊時，快速啟動套件會更新目前團隊並重新整理這些 URL 預設值，以便產生的連結能持續使用正確的團隊上下文。

在建立或重新命名團隊時，快速啟動套件還會防止使用者選擇可能導致不安全或衝突路由段的保留名稱。例如，不能使用會與 `settings`、`login` 或 `dashboard` 等路由前綴衝突的名稱。

<a name="workos"></a>
## WorkOS AuthKit 認證

預設情況下，React、Svelte、Vue 與 Livewire 的快速啟動套件都利用 Laravel 內建的認證系統來提供登入、註冊、密碼重設、電子郵件驗證等功能。此外，我們也為每個快速啟動套件提供了一個由 [WorkOS AuthKit](https://authkit.com) 驅動的變體，其提供：

<div class="content-list" markdown="1">

- 社交認證 (Google, Microsoft, GitHub 與 Apple)
- 通行金鑰 (Passkey) 認證
- 基於電子郵件的 "Magic Auth"
- SSO

</div>

使用 WorkOS 作為您的認證提供者 [需要 WorkOS 帳號](https://workos.com)。WorkOS 對於每月活躍使用者數達 100 萬人以下的應用程式提供免費認證。

若要將 WorkOS AuthKit 作為您應用程式的認證提供者，請在透過 `laravel new` 建立新的快速啟動套件應用程式時選擇 WorkOS 選項。


### 設定您的 WorkOS 快速啟動套件

在使用 WorkOS 驅動的快速啟動套件建立新應用程式後，您應該在應用程式的 `.env` 檔案中設定 `WORKOS_CLIENT_ID`、`WORKOS_API_KEY` 與 `WORKOS_REDIRECT_URL` 環境變數。這些變數應與 WorkOS 控制面板中為您的應用程式所提供的數值一致：

```ini
WORKOS_CLIENT_ID=your-client-id
WORKOS_API_KEY=your-api-key
WORKOS_REDIRECT_URL="${APP_URL}/authenticate"
```

此外，您應該在 WorkOS 控制面板中設定應用程式的首頁 URL。使用者在登出您的應用程式後將被重新導向至此 URL。


<a name="configuring-authkit-authentication-methods"></a>
#### 設定 AuthKit 認證方法

當使用 WorkOS 驅動的快速啟動套件時，我們建議您在應用程式的 WorkOS AuthKit 設定中停用 "Email + Password" 認證，讓使用者僅能透過社交認證提供者、通行金鑰 (passkeys)、"Magic Auth" 與 SSO 進行認證。這能讓您的應用程式完全避免處理使用者密碼。


<a name="configuring-authkit-session-timeouts"></a>
#### 設定 AuthKit 工作階段逾時

此外，我們建議您將 WorkOS AuthKit 的工作階段不活動逾時 (session inactivity timeout) 設定為與 Laravel 應用程式設定的工作階段逾時閾值一致，通常為兩小時。


<a name="inertia-ssr"></a>
### Inertia SSR

React、Svelte 與 Vue 的快速啟動套件與 Inertia 的 [伺服器端渲染 (server-side rendering)](https://inertiajs.com/server-side-rendering) 功能相容。若要為您的應用程式建立相容於 Inertia SSR 的打包檔，請執行 `build:ssr` 指令：

```shell
npm run build:ssr
```

為了方便起見，還提供了 `composer dev:ssr` 指令。此指令會在為您的應用程式建立相容於 SSR 的打包檔後，啟動 Laravel 開發伺服器與 Inertia SSR 伺服器，讓您可以使用 Inertia 的伺服器端渲染引擎在本地測試您的應用程式：

```shell
composer dev:ssr
```


<a name="community-maintained-starter-kits"></a>
### 社群維護的快速啟動套件

當使用 Laravel 安裝程式建立新的 Laravel 應用程式時，您可以將 Packagist 上任何社群維護的快速啟動套件提供給 `--using` 參數：

```shell
laravel new my-app --using=example/starter-kit
```


<a name="creating-starter-kits"></a>
#### 建立快速啟動套件

若要確保您的快速啟動套件能被他人使用，您需要將其發佈到 [Packagist](https://packagist.org)。您的快速啟動套件應在 `.env.example` 檔案中定義所需的環境變數，且任何必要的安裝後指令應列在快速啟動套件 `composer.json` 檔案的 `post-create-project-cmd` 陣列中。


<a name="faqs"></a>
### 常見問題


<a name="faq-upgrade"></a>
#### 我該如何升級？

每個快速啟動套件都為您的下一個應用程式提供了一個堅實的起點。由於您完全擁有程式碼的所有權，您可以根據自己的構想對應用程式進行調整、自定義與建構。然而，並沒有必要更新快速啟動套件本身。


<a name="faq-enable-email-verification"></a>
#### 我該如何啟用電子郵件驗證？

您可以透過取消 `App/Models/User.php` 模型中 `MustVerifyEmail` 匯入的註解，並確保該模型實作了 `MustVerifyEmail` 介面來加入電子郵件驗證：

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

註冊後，使用者將收到一封驗證電子郵件。若要限制某些路由的存取，直到使用者的電子郵件地址通過驗證為止，請將 `verified` 中介層添加到路由中：

```php
Route::middleware(['auth', 'verified'])->group(function () {
    Route::get('dashboard', function () {
        return Inertia::render('dashboard');
    })->name('dashboard');
});
```

> [!NOTE]
> 當使用 [WorkOS](#workos) 變體的快速啟動套件時，不需要進行電子郵件驗證。


<a name="faq-modify-email-template"></a>
#### 我該如何修改預設的電子郵件範本？

您可能想要自定義預設的電子郵件範本，以更好地符合您應用程式的品牌形象。若要修改此範本，您應該使用以下指令將電子郵件視圖發佈到您的應用程式：

```
php artisan vendor:publish --tag=laravel-mail
```

這將在 `resources/views/vendor/mail` 中產生數個檔案。您可以修改其中任何檔案以及 `resources/views/vendor/mail/themes/default.css` 檔案，以更改預設電子郵件範本的外觀。