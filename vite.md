# 資產打包 (Vite)

- [簡介](#introduction)
- [安裝與設定](#installation)
  - [安裝 Node](#installing-node)
  - [安裝 Vite 與 Laravel 外掛](#installing-vite-and-laravel-plugin)
  - [設定 Vite](#configuring-vite)
  - [載入你的腳本與樣式](#loading-your-scripts-and-styles)
- [執行 Vite](#running-vite)
- [使用 JavaScript](#working-with-scripts)
  - [別名](#aliases)
  - [Vue](#vue)
  - [React](#react)
  - [Inertia](#inertia)
  - [URL 處理](#url-processing)
- [使用樣式表](#working-with-stylesheets)
- [使用 Blade 與路由](#working-with-blade-and-routes)
  - [使用 Vite 處理靜態資產](#blade-processing-static-assets)
  - [儲存時重新整理](#blade-refreshing-on-save)
  - [別名](#blade-aliases)
- [資產預取](#asset-prefetching)
- [自訂基礎 URL](#custom-base-urls)
- [環境變數](#environment-variables)
- [在測試中停用 Vite](#disabling-vite-in-tests)
- [伺服器端渲染 (SSR)](#ssr)
- [腳本與樣式標籤屬性](#script-and-style-attributes)
  - [內容安全策略 (CSP) Nonce](#content-security-policy-csp-nonce)
  - [子資源完整性 (SRI)](#subresource-integrity-sri)
  - [任意屬性](#arbitrary-attributes)
- [進階自訂](#advanced-customization)
  - [開發伺服器跨來源資源共享 (CORS)](#cors)
  - [修正開發伺服器 URL](#correcting-dev-server-urls)

<a name="introduction"></a>
## 簡介

[Vite](https://vitejs.dev) 是一個現代化的前端建構工具，它提供極快的開發環境，並將你的程式碼打包以供生產使用。使用 Laravel 建構應用程式時，你通常會使用 Vite 將應用程式的 CSS 與 JavaScript 檔案打包成可在生產環境中使用的資產。

Laravel 透過提供官方外掛與 Blade 指令，與 Vite 無縫整合，以便在開發與生產環境中載入你的資產。

<a name="installation"></a>
## 安裝與設定

> [!NOTE]
> 以下文件說明如何手動安裝與設定 Laravel Vite 外掛。然而，Laravel 的 [入門套件](/docs/{{version}}/starter-kits) 已包含所有這些 Scaffold，是開始使用 Laravel 與 Vite 的最快方式。

<a name="installing-node"></a>
### 安裝 Node

在執行 Vite 與 Laravel 外掛之前，必須確保已安裝 Node.js (16+) 與 NPM：

```shell
node -v
npm -v
```

你可以透過 [Node 官方網站](https://nodejs.org/en/download/) 提供的簡易圖形化安裝程式，輕鬆安裝最新版本的 Node 與 NPM。或者，如果你正在使用 [Laravel Sail](https://laravel.com/docs/{{version}}/sail)，你可以透過 Sail 叫用 Node 與 NPM：

```shell
./vendor/bin/sail node -v
./vendor/bin/sail npm -v
```

<a name="installing-vite-and-laravel-plugin"></a>
### 安裝 Vite 與 Laravel 外掛

在全新安裝的 Laravel 應用程式中，你會在應用程式目錄結構的根目錄中找到一個 `package.json` 檔案。預設的 `package.json` 檔案已包含開始使用 Vite 與 Laravel 外掛所需的一切。你可以透過 NPM 安裝應用程式的前端依賴套件：

```shell
npm install
```

<a name="configuring-vite"></a>
### 設定 Vite

Vite 透過專案根目錄中的 `vite.config.js` 檔案進行設定。你可以根據需求自訂此檔案，也可以安裝應用程式所需的任何其他外掛，例如 `@vitejs/plugin-vue` 或 `@vitejs/plugin-react`。

Laravel Vite 外掛要求你指定應用程式的進入點。這些可以是 JavaScript 或 CSS 檔案，並包含預處理語言，例如 TypeScript、JSX、TSX 與 Sass。

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';

export default defineConfig({
    plugins: [
        laravel([
            'resources/css/app.css',
            'resources/js/app.js',
        ]),
    ],
});
```

如果你正在建構 SPA (包括使用 Inertia 建構的應用程式)，Vite 在沒有 CSS 進入點的情況下表現最佳：

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';

export default defineConfig({
    plugins: [
        laravel([
            'resources/css/app.css', // [tl! remove]
            'resources/js/app.js',
        ]),
    ],
});
```

相反地，你應該透過 JavaScript 導入你的 CSS。通常，這會在應用程式的 `resources/js/app.js` 檔案中完成：

```js
import './bootstrap';
import '../css/app.css'; // [tl! add]
```

Laravel 外掛也支援多個進入點與進階設定選項，例如 [SSR 進入點](#ssr)。

<a name="working-with-a-secure-development-server"></a>
#### 使用安全的開發伺服器

如果你的本地開發網頁伺服器透過 HTTPS 提供應用程式，你可能會遇到連接到 Vite 開發伺服器的問題。

如果你正在使用 [Laravel Herd](https://herd.laravel.com) 並已保護網站，或者你正在使用 [Laravel Valet](/docs/{{version}}/valet) 並已對你的應用程式執行 [secure 命令](/docs/{{version}}/valet#securing-sites)，Laravel Vite 外掛將自動為你偵測並使用生成的 TLS 憑證。

如果你使用與應用程式目錄名稱不符的主機來保護網站，你可以在應用程式的 `vite.config.js` 檔案中手動指定主機：

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';

export default defineConfig({
    plugins: [
        laravel({
            // ...
            detectTls: 'my-app.test', // [tl! add]
        }),
    ],
});
```

當使用另一個網頁伺服器時，你應該生成一個受信任的憑證，並手動設定 Vite 以使用生成的憑證：

```js
// ...
import fs from 'fs'; // [tl! add]

const host = 'my-app.test'; // [tl! add]

export default defineConfig({
    // ...
    server: { // [tl! add]
        host, // [tl! add]
        hmr: { host }, // [tl! add]
        https: { // [tl! add]
            key: fs.readFileSync(`/path/to/${host}.key`), // [tl! add]
            cert: fs.readFileSync(`/path/to/${host}.crt`), // [tl! add]
        }, // [tl! add]
    }, // [tl! add]
});
```

如果你無法為系統生成受信任的憑證，你可以安裝並設定 [@vitejs/plugin-basic-ssl 外掛](https://github.com/vitejs/vite-plugin-basic-ssl)。當使用不受信任的憑證時，你需要透過在執行 `npm run dev` 命令時，在控制台中點擊「Local」連結，在瀏覽器中接受 Vite 開發伺服器的憑證警告。

<a name="configuring-hmr-in-sail-on-wsl2"></a>
#### 在 WSL2 上於 Sail 中執行開發伺服器

在 Windows Subsystem for Linux 2 (WSL2) 上，於 [Laravel Sail](/docs/{{version}}/sail) 中執行 Vite 開發伺服器時，你應該將以下設定新增到你的 `vite.config.js` 檔案中，以確保瀏覽器可以與開發伺服器通訊：

```js
// ...

export default defineConfig({
    // ...
    server: { // [tl! add:start]
        hmr: {
            host: 'localhost',
        },
    }, // [tl! add:end]
});
```

如果你的檔案變更在開發伺服器執行時未反映在瀏覽器中，你可能還需要設定 Vite 的 [server.watch.usePolling 選項](https://vitejs.dev/config/server-options.html#server-watch)。

<a name="loading-your-scripts-and-styles"></a>
### 載入你的腳本與樣式

設定好 Vite 進入點後，你現在可以在應用程式根模板的 `<head>` 中新增的 `@vite()` Blade 指令中引用它們：

```blade
<!DOCTYPE html>
<head>
    {{-- ... --}}

    @vite(['resources/css/app.css', 'resources/js/app.js'])
</head>
```

如果你透過 JavaScript 導入 CSS，則只需包含 JavaScript 進入點即可：

```blade
<!DOCTYPE html>
<head>
    {{-- ... --}}

    @vite('resources/js/app.js')
</head>
```

`@vite` 指令會自動偵測 Vite 開發伺服器，並注入 Vite 客戶端以啟用熱模組替換 (Hot Module Replacement)。在建構模式下，該指令將載入你編譯並版本化的資產，包括任何導入的 CSS。

如果需要，你也可以在調用 `@vite` 指令時指定已編譯資產的建構路徑：

```blade
<!doctype html>
<head>
    {{-- Given build path is relative to public path. --}}

    @vite('resources/js/app.js', 'vendor/courier/build')
</head>
```

<a name="inline-assets"></a>
#### 行內資產

有時可能需要包含資產的原始內容，而不是連結到資產的版本化 URL。例如，當向 PDF 生成器傳遞 HTML 內容時，你可能需要將資產內容直接包含在頁面中。你可以使用 `Vite` Facade 提供的 `content` 方法，輸出 Vite 資產的內容：

```blade
@use('Illuminate\Support\Facades\Vite')

<!doctype html>
<head>
    {{-- ... --}}

    <style>
        {!! Vite::content('resources/css/app.css') !!}
    </style>
    <script>
        {!! Vite::content('resources/js/app.js') !!}
    </script>
</head>
```

<a name="running-vite"></a>
## 執行 Vite

有兩種方式可以執行 Vite。您可以透過 `dev` 命令執行開發伺服器，這在本地開發時非常有用。開發伺服器會自動偵測您檔案的變更，並立即反映在任何開啟的瀏覽器視窗中。

或者，執行 `build` 命令將會為您應用程式的資產進行版本化並打包，使其準備好部署到生產環境：

```shell

# 執行 Vite 開發伺服器...
npm run dev

```shell
# Build and version the assets for production...
npm run build
```

如果你正在 WSL2 上透過 [Sail](/docs/{{version}}/sail) 執行開發伺服器，你可能需要一些[額外的設定](#configuring-hmr-in-sail-on-wsl2)選項。


<a name="working-with-scripts"></a>
## 使用 JavaScript


<a name="aliases"></a>
### 別名

預設情況下，Laravel 外掛提供了一個通用的別名，幫助你快速上手並方便地匯入應用程式的資產：

```js
{
    '@' => '/resources/js'
}
```

你可以透過在 `vite.config.js` 設定檔中添加自己的別名來覆寫 `'@'` 別名：

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';

export default defineConfig({
    plugins: [
        laravel(['resources/ts/app.tsx']),
    ],
    resolve: {
        alias: {
            '@': '/resources/ts',
        },
    },
});
```


<a name="vue"></a>
### Vue

如果你想使用 [Vue](https://vuejs.org/) 框架來建構你的前端，那麼你還需要安裝 `@vitejs/plugin-vue` 外掛：

```shell
npm install --save-dev @vitejs/plugin-vue
```

然後你可以將此外掛包含在你的 `vite.config.js` 設定檔中。當搭配 Laravel 使用 Vue 外掛時，你還會需要一些額外的選項：

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';
import vue from '@vitejs/plugin-vue';

export default defineConfig({
    plugins: [
        laravel(['resources/js/app.js']),
        vue({
            template: {
                transformAssetUrls: {
                    // The Vue plugin will re-write asset URLs, when referenced
                    // in Single File Components, to point to the Laravel web
                    // server. Setting this to `null` allows the Laravel plugin
                    // to instead re-write asset URLs to point to the Vite
                    // server instead.
                    base: null,

                    // The Vue plugin will parse absolute URLs and treat them
                    // as absolute paths to files on disk. Setting this to
                    // `false` will leave absolute URLs un-touched so they can
                    // reference assets in the public directory as expected.
                    includeAbsolute: false,
                },
            },
        }),
    ],
});
```

> [!NOTE]
> Laravel 的 [入門套件](/docs/{{version}}/starter-kits) 已經包含了適當的 Laravel、Vue 和 Vite 設定。這些入門套件是開始使用 Laravel、Vue 和 Vite 的最快方式。


<a name="react"></a>
### React

如果你想使用 [React](https://reactjs.org/) 框架來建構你的前端，那麼你還需要安裝 `@vitejs/plugin-react` 外掛：

```shell
npm install --save-dev @vitejs/plugin-react
```

然後你可以將此外掛包含在你的 `vite.config.js` 設定檔中：

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';
import react from '@vitejs/plugin-react';

export default defineConfig({
    plugins: [
        laravel(['resources/js/app.jsx']),
        react(),
    ],
});
```

你需要確保任何包含 JSX 的檔案都具有 `.jsx` 或 `.tsx` 副檔名，並記住如有需要，應更新你的進入點，如[上面所示](#configuring-vite)。

你還需要在現有的 `@vite` directive 旁邊包含額外的 `@viteReactRefresh` Blade directive。

```blade
@viteReactRefresh
@vite('resources/js/app.jsx')
```

`@viteReactRefresh` directive 必須在 `@vite` directive 之前呼叫。

> [!NOTE]
> Laravel 的 [入門套件](/docs/{{version}}/starter-kits) 已經包含了適當的 Laravel、React 和 Vite 設定。這些入門套件是開始使用 Laravel、React 和 Vite 的最快方式。


<a name="inertia"></a>
### Inertia

Laravel Vite 外掛提供了一個方便的 `resolvePageComponent` 函式，可幫助你解析你的 Inertia 頁面元件。以下是該輔助函式與 Vue 3 搭配使用的範例；不過，你也可以在其他框架（例如 React）中使用此函式：

```js
import { createApp, h } from 'vue';
import { createInertiaApp } from '@inertiajs/vue3';
import { resolvePageComponent } from 'laravel-vite-plugin/inertia-helpers';

createInertiaApp({
  resolve: (name) => resolvePageComponent(`./Pages/${name}.vue`, import.meta.glob('./Pages/**/*.vue')),
  setup({ el, App, props, plugin }) {
    createApp({ render: () => h(App, props) })
      .use(plugin)
      .mount(el)
  },
});
```

如果你正在搭配 Inertia 使用 Vite 的程式碼分割功能，我們建議設定[資產預取](#asset-prefetching)。

> [!NOTE]
> Laravel 的 [入門套件](/docs/{{version}}/starter-kits) 已經包含了適當的 Laravel、Inertia 和 Vite 設定。這些入門套件是開始使用 Laravel、Inertia 和 Vite 的最快方式。


<a name="url-processing"></a>
### URL 處理

當使用 Vite 並在應用程式的 HTML、CSS 或 JS 中引用資產時，有幾個注意事項需要考量。首先，如果你使用絕對路徑引用資產，Vite 將不會在建構中包含該資產；因此，你應確保資產在你的 public 目錄中可用。當使用[專用的 CSS 進入點](#configuring-vite)時，應避免使用絕對路徑，因為在開發過程中，瀏覽器會嘗試從託管 CSS 的 Vite 開發伺服器載入這些路徑，而非從你的 public 目錄載入。

當引用相對資產路徑時，你應記住這些路徑是相對於它們被引用的檔案。任何透過相對路徑引用的資產都將被 Vite 重寫、版本化並打包。

考慮以下專案結構：

```text
public/
  taylor.png
resources/
  js/
    Pages/
      Welcome.vue
  images/
    abigail.png
```

以下範例展示了 Vite 將如何處理相對和絕對 URL：

```html
<!-- This asset is not handled by Vite and will not be included in the build -->
<img src="/taylor.png">

<!-- This asset will be re-written, versioned, and bundled by Vite -->
<img src="../../images/abigail.png">
```


<a name="working-with-stylesheets"></a>
## 使用樣式表

> [!NOTE]
> Laravel 的 [入門套件](/docs/{{version}}/starter-kits) 已經包含了適當的 Tailwind 和 Vite 設定。或者，如果你想在不使用我們任何入門套件的情況下使用 Tailwind 和 Laravel，請查看 [Tailwind 的 Laravel 安裝指南](https://tailwindcss.com/docs/guides/laravel)。

所有 Laravel 應用程式都已經包含 Tailwind 和一個設定正確的 `vite.config.js` 檔案。因此，你只需要啟動 Vite 開發伺服器，或執行 `dev` Composer 指令，這將同時啟動 Laravel 和 Vite 開發伺服器：

```shell
composer run dev
```

你的應用程式的 CSS 可以放在 `resources/css/app.css` 檔案中。

<a name="working-with-blade-and-routes"></a>
## 使用 Blade 與路由


<a name="blade-processing-static-assets"></a>
### 使用 Vite 處理靜態資產

當你在 JavaScript 或 CSS 中引用資產時，Vite 會自動處理並為它們加上版本。此外，在建構以 Blade 為基礎的應用程式時，Vite 也可以處理並為你僅在 Blade 模板中引用的靜態資產加上版本。

然而，為了實現這一點，你需要將靜態資產導入應用程式的進入點，讓 Vite 知道這些資產。例如，如果你想處理並為儲存在 `resources/images` 中的所有圖片和 `resources/fonts` 中的所有字體加上版本，你應該在應用程式的 `resources/js/app.js` 進入點中加入以下內容：

```js
import.meta.glob([
  '../images/**',
  '../fonts/**',
]);
```

當執行 `npm run build` 時，這些資產將會由 Vite 處理。然後，你可以在 Blade 模板中使用 `Vite::asset` 方法引用這些資產，該方法將會回傳給定資產的版本化 URL：

```blade
<img src="{{ Vite::asset('resources/images/logo.png') }}">
```


<a name="blade-refreshing-on-save"></a>
### 儲存時重新整理

當你的應用程式使用傳統的 Blade 伺服器端渲染建構時，Vite 可以透過在你的應用程式中修改視圖檔案時自動重新整理瀏覽器來改善你的開發工作流程。若要開始，你可以簡單地將 `refresh` 選項設定為 `true`。

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';

export default defineConfig({
    plugins: [
        laravel({
            // ...
            refresh: true,
        }),
    ],
});
```

當 `refresh` 選項為 `true` 時，在你執行 `npm run dev` 期間，儲存以下目錄中的檔案將會觸發瀏覽器執行完整頁面重新整理：

- `app/Livewire/**`
- `app/View/Components/**`
- `lang/**`
- `resources/lang/**`
- `resources/views/**`
- `routes/**`

如果你正在利用 [Ziggy](https://github.com/tighten/ziggy) 在應用程式前端產生路由連結，監控 `routes/**` 目錄會很有用。

如果這些預設路徑不符合你的需求，你可以指定自己的監控路徑列表：

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';

export default defineConfig({
    plugins: [
        laravel({
            // ...
            refresh: ['resources/views/**'],
        }),
    ],
});
```

在底層，Laravel Vite 外掛使用 [vite-plugin-full-reload](https://github.com/ElMassimo/vite-plugin-full-reload) 套件，該套件提供了一些進階設定選項來微調此功能的行為。如果你需要這種程度的自訂，你可以提供 `config` 定義：

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';

export default defineConfig({
    plugins: [
        laravel({
            // ...
            refresh: [{
                paths: ['path/to/watch/**'],
                config: { delay: 300 }
            }],
        }),
    ],
});
```


<a name="blade-aliases"></a>
### 別名

在 JavaScript 應用程式中，為經常引用的目錄建立[別名](#aliases)是很常見的。但是，你也可以使用 `Illuminate\Support\Facades\Vite` 類別上的 `macro` 方法來在 Blade 中建立別名。通常，「macros」應該在[服務提供者](/docs/{{version}}/providers)的 `boot` 方法中定義：

```php
/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Vite::macro('image', fn (string $asset) => $this->asset("resources/images/{$asset}"));
}
```

一旦定義了巨集 (macro)，就可以在你的模板中呼叫它。例如，我們可以利用上方定義的 `image` 巨集來引用位於 `resources/images/logo.png` 的資產：

```blade
<img src="{{ Vite::image('logo.png') }}" alt="Laravel Logo">
```


<a name="asset-prefetching"></a>
## 資產預取

當使用 Vite 的程式碼分割功能建構 SPA 時，所需的資產會在每次頁面導航時被提取。這種行為可能導致 UI 渲染延遲。如果這對你選擇的前端框架來說是個問題，Laravel 提供了在初始頁面載入時預先載入應用程式的 JavaScript 和 CSS 資產的功能。

你可以在[服務提供者](/docs/{{version}}/providers)的 `boot` 方法中呼叫 `Vite::prefetch` 方法，指示 Laravel 預先載入你的資產：

```php
<?php

namespace App\Providers;

use Illuminate\Support\Facades\Vite;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    /**
     * Register any application services.
     */
    public function register(): void
    {
        // ...
    }

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        Vite::prefetch(concurrency: 3);
    }
}
```

在上面的範例中，每次頁面載入時，資產將會以最多 `3` 個併發下載進行預取。你可以修改併發數以符合你的應用程式需求，或者如果應用程式應該一次下載所有資產，則可以指定沒有併發限制：

```php
/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Vite::prefetch();
}
```

預設情況下，預取會在[頁面載入事件](https://developer.mozilla.org/en-US/docs/Web/API/Window/load_event)觸發時開始。如果你想自訂預取何時開始，你可以指定 Vite 將監聽的事件：

```php
/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Vite::prefetch(event: 'vite:prefetch');
}
```

根據上面的程式碼，當你手動在 `window` 物件上分派 `vite:prefetch` 事件時，預取將會開始。例如，你可以讓預取在頁面載入後三秒開始：

```html
<script>
    addEventListener('load', () => setTimeout(() => {
        dispatchEvent(new Event('vite:prefetch'))
    }, 3000))
</script>
```


<a name="custom-base-urls"></a>
## 自訂基礎 URL

如果你的 Vite 編譯資產部署到與你的應用程式獨立的網域，例如透過 CDN，你必須在應用程式的 `.env` 檔案中指定 `ASSET_URL` 環境變數：

```env
ASSET_URL=https://cdn.example.com
```

設定資產 URL 後，所有重新編寫的資產 URL 都將以配置的值為前綴：

```text
https://cdn.example.com/build/assets/app.9dce8d17.js
```

請記住，[絕對 URL 不會被 Vite 重新編寫](#url-processing)，因此它們不會被加上前綴。


<a name="environment-variables"></a>
## 環境變數

你可以在應用程式的 `.env` 檔案中，透過在環境變數前加上 `VITE_` 前綴，將其注入到你的 JavaScript 中：

```env
VITE_SENTRY_DSN_PUBLIC=http://example.com
```

你可以透過 `import.meta.env` 物件存取注入的環境變數：

```js
import.meta.env.VITE_SENTRY_DSN_PUBLIC
```


<a name="disabling-vite-in-tests"></a>
## 在測試中停用 Vite

Laravel 的 Vite 整合會在執行測試時嘗試解析你的資產，這要求你必須運行 Vite 開發伺服器或建構你的資產。

如果你希望在測試期間模擬 Vite，你可以呼叫 `withoutVite` 方法，該方法適用於任何擴展 Laravel 的 `TestCase` 類別的測試：

```php tab=Pest
test('without vite example', function () {
    $this->withoutVite();

    // ...
});
```

```php tab=PHPUnit
use Tests\TestCase;

class ExampleTest extends TestCase
{
    public function test_without_vite_example(): void
    {
        $this->withoutVite();

        // ...
    }
}
```

如果你想為所有測試停用 Vite，你可以在基礎 `TestCase` 類別的 `setUp` 方法中呼叫 `withoutVite` 方法：

```php
<?php

namespace Tests;

use Illuminate\Foundation\Testing\TestCase as BaseTestCase;

abstract class TestCase extends BaseTestCase
{
    protected function setUp(): void// [tl! add:start]
    {
        parent::setUp();

        $this->withoutVite();
    }// [tl! add:end]
}
```

<a name="ssr"></a>
## 伺服器端渲染 (SSR)

Laravel Vite 外掛讓使用 Vite 設定伺服器端渲染變得輕而易舉。要開始使用，請在 `resources/js/ssr.js` 建立一個 SSR 進入點，並透過將設定選項傳遞給 Laravel 外掛來指定該進入點：

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';

export default defineConfig({
    plugins: [
        laravel({
            input: 'resources/js/app.js',
            ssr: 'resources/js/ssr.js',
        }),
    ],
});
```

為確保您不會忘記重新建構 SSR 進入點，我們建議您擴充應用程式 `package.json` 中的 "build" 腳本以建立您的 SSR 建構：

```json
"scripts": {
     "dev": "vite",
     "build": "vite build" // [tl! remove]
     "build": "vite build && vite build --ssr" // [tl! add]
}
```

然後，要建構並啟動 SSR 伺服器，您可以執行以下指令：

```shell
npm run build
node bootstrap/ssr/ssr.js
```

如果您正在使用 [Inertia SSR](https://inertiajs.com/server-side-rendering)，您可以改用 `inertia:start-ssr` Artisan 命令來啟動 SSR 伺服器：

```shell
php artisan inertia:start-ssr
```

> [!NOTE]
> Laravel 的 [入門套件](/docs/{{version}}/starter-kits) 已包含適當的 Laravel、Inertia SSR 和 Vite 設定。這些入門套件是開始使用 Laravel、Inertia SSR 和 Vite 最快的方式。

<a name="script-and-style-attributes"></a>
## 腳本與樣式標籤屬性

<a name="content-security-policy-csp-nonce"></a>
### 內容安全策略 (CSP) Nonce

如果您希望在腳本和樣式標籤中包含 [nonce 屬性](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/nonce) 作為 [內容安全策略 (CSP)](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP) 的一部分，您可以在自訂的 [中介層](/docs/{{version}}/middleware) 中使用 `useCspNonce` 方法來產生或指定一個 nonce：

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Vite;
use Symfony\Component\HttpFoundation\Response;

class AddContentSecurityPolicyHeaders
{
    /**
     * Handle an incoming request.
     *
     * @param  \Closure(\Illuminate\Http\Request): (\Symfony\Component\HttpFoundation\Response)  $next
     */
    public function handle(Request $request, Closure $next): Response
    {
        Vite::useCspNonce();

        return $next($request)->withHeaders([
            'Content-Security-Policy' => "script-src 'nonce-".Vite::cspNonce()."'",
        ]);
    }
}
```

在呼叫 `useCspNonce` 方法後，Laravel 會自動將 `nonce` 屬性包含在所有生成的腳本和樣式標籤中。

如果您需要在其他地方指定 nonce，包括 Laravel [入門套件](/docs/{{version}}/starter-kits) 中包含的 [Ziggy `@route` 指令](https://github.com/tighten/ziggy#using-routes-with-a-content-security-policy)，您可以透過 `cspNonce` 方法來取得它：

```blade
@routes(nonce: Vite::cspNonce())
```

如果您已經有一個 nonce，並希望指示 Laravel 使用它，您可以將該 nonce 傳遞給 `useCspNonce` 方法：

```php
Vite::useCspNonce($nonce);
```

<a name="subresource-integrity-sri"></a>
### 子資源完整性 (SRI)

如果您的 Vite manifest 包含資產的 `integrity` 雜湊，Laravel 將自動在所有生成的腳本和樣式標籤上添加 `integrity` 屬性，以強制執行 [子資源完整性 (SRI)](https://developer.mozilla.org/en-US/docs/Web/Security/Subresource_Integrity)。預設情況下，Vite 不會在其 manifest 中包含 `integrity` 雜湊，但您可以透過安裝 [vite-plugin-manifest-sri](https://www.npmjs.com/package/vite-plugin-manifest-sri) NPM 外掛來啟用它：

```shell
npm install --save-dev vite-plugin-manifest-sri
```

然後，您可以在 `vite.config.js` 檔案中啟用此外掛：

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';
import manifestSRI from 'vite-plugin-manifest-sri';// [tl! add]

export default defineConfig({
    plugins: [
        laravel({
            // ...
        }),
        manifestSRI(),// [tl! add]
    ],
});
```

如果需要，您也可以自訂 manifest 中可以找到 integrity 雜湊的鍵：

```php
use Illuminate\Support\Facades\Vite;

Vite::useIntegrityKey('custom-integrity-key');
```

如果您想完全停用此自動偵測，您可以將 `false` 傳遞給 `useIntegrityKey` 方法：

```php
Vite::useIntegrityKey(false);
```

<a name="arbitrary-attributes"></a>
### 任意屬性

如果您需要在腳本和樣式標籤中包含額外的屬性，例如 [data-turbo-track](https://turbo.hotwired.dev/handbook/drive#reloading-when-assets-change) 屬性，您可以透過 `useScriptTagAttributes` 和 `useStyleTagAttributes` 方法來指定它們。通常，這些方法應該從 [服務提供者](/docs/{{version}}/providers) 中呼叫：

```php
use Illuminate\Support\Facades\Vite;

Vite::useScriptTagAttributes([
    'data-turbo-track' => 'reload', // Specify a value for the attribute...
    'async' => true, // Specify an attribute without a value...
    'integrity' => false, // Exclude an attribute that would otherwise be included...
]);

Vite::useStyleTagAttributes([
    'data-turbo-track' => 'reload',
]);
```

如果您需要有條件地添加屬性，您可以傳遞一個回呼函數，該函數將接收資產的來源路徑、其 URL、其 manifest 區塊以及整個 manifest：

```php
use Illuminate\Support\Facades\Vite;

Vite::useScriptTagAttributes(fn (string $src, string $url, array|null $chunk, array|null $manifest) => [
    'data-turbo-track' => $src === 'resources/js/app.js' ? 'reload' : false,
]);

Vite::useStyleTagAttributes(fn (string $src, string $url, array|null $chunk, array|null $manifest) => [
    'data-turbo-track' => $chunk && $chunk['isEntry'] ? 'reload' : false,
]);
```

> [!WARNING]
> 當 Vite 開發伺服器執行時，`$chunk` 和 `$manifest` 參數將為 `null`。

<a name="advanced-customization"></a>
## 進階自訂

開箱即用時，Laravel 的 Vite 外掛使用合理的慣例，這些慣例應該適用於大多數應用程式；然而，有時你可能需要自訂 Vite 的行為。為了啟用額外的自訂選項，我們提供以下方法和選項，這些方法和選項可以用來取代 `@vite` Blade 指令：

```blade
<!doctype html>
<head>
    {{-- ... --}}

    {{
        Vite::useHotFile(storage_path('vite.hot')) // Customize the "hot" file...
            ->useBuildDirectory('bundle') // Customize the build directory...
            ->useManifestFilename('assets.json') // Customize the manifest filename...
            ->withEntryPoints(['resources/js/app.js']) // Specify the entry points...
            ->createAssetPathsUsing(function (string $path, ?bool $secure) { // Customize the backend path generation for built assets...
                return "https://cdn.example.com/{$path}";
            })
    }}
</head>
```

在 `vite.config.js` 檔案中，你應該接著指定相同的設定：

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';

export default defineConfig({
    plugins: [
        laravel({
            hotFile: 'storage/vite.hot', // Customize the "hot" file...
            buildDirectory: 'bundle', // Customize the build directory...
            input: ['resources/js/app.js'], // Specify the entry points...
        }),
    ],
    build: {
      manifest: 'assets.json', // Customize the manifest filename...
    },
});
```

<a name="cors"></a>
### 開發伺服器跨來源資源共享 (CORS)

如果你在從 Vite 開發伺服器取得資產時，於瀏覽器中遇到跨來源資源共享 (CORS) 問題，你可能需要授予你的自訂來源存取開發伺服器的權限。Vite 結合 Laravel 外掛，無需任何額外設定即可允許以下來源：

- `::1`
- `127.0.0.1`
- `localhost`
- `*.test`
- `*.localhost`
- 專案的 `.env` 檔案中的 `APP_URL`

允許專案自訂來源最簡單的方法是，確保你的應用程式的 `APP_URL` 環境變數與你正在瀏覽器中造訪的來源相符。例如，如果你正在造訪 `https://my-app.laravel`，你應該更新你的 `.env` 以符合：

```env
APP_URL=https://my-app.laravel
```

如果你需要對來源進行更精細的控制，例如支援多個來源，你應該利用 [Vite 全面且彈性的內建 CORS 伺服器設定](https://vite.dev/config/server-options.html#server-cors)。舉例來說，你可以在專案的 `vite.config.js` 檔案中，於 `server.cors.origin` 設定選項中指定多個來源：

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';

export default defineConfig({
    plugins: [
        laravel({
            input: 'resources/js/app.js',
            refresh: true,
        }),
    ],
    server: {  // [tl! add]
        cors: {  // [tl! add]
            origin: [  // [tl! add]
                'https://backend.laravel',  // [tl! add]
                'http://admin.laravel:8566',  // [tl! add]
            ],  // [tl! add]
        },  // [tl! add]
    },  // [tl! add]
});
```

你也可以包含正規表達式模式 (regex patterns)，如果你想允許特定頂級網域 (例如 `*.laravel`) 的所有來源，這會很有幫助：

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';

export default defineConfig({
    plugins: [
        laravel({
            input: 'resources/js/app.js',
            refresh: true,
        }),
    ],
    server: {  // [tl! add]
        cors: {  // [tl! add]
            origin: [ // [tl! add]
                // Supports: SCHEME://DOMAIN.laravel[:PORT] [tl! add]
                /^https?:\/\/.*\.laravel(:\d+)?$/, //[tl! add]
            ], // [tl! add]
        }, // [tl! add]
    }, // [tl! add]
});
```

<a name="correcting-dev-server-urls"></a>
### 修正開發伺服器 URL

Vite 生態系統中的一些外掛假設以斜線開頭的 URL 總是會指向 Vite 開發伺服器。然而，由於 Laravel 整合的特性，情況並非如此。

舉例來說，當 Vite 提供你的資產時，`vite-imagetools` 外掛會輸出如下所示的 URL：

```html
<img src="/@imagetools/f0b2f404b13f052c604e632f2fb60381bf61a520">
```

`vite-imagetools` 外掛預期輸出 URL 會被 Vite 攔截，然後該外掛便可以處理所有以 `/@imagetools` 開頭的 URL。如果你正在使用的外掛預期這種行為，你將需要手動修正這些 URL。你可以在 `vite.config.js` 檔案中，透過使用 `transformOnServe` 選項來完成這項操作。

在這個特定的範例中，我們將在生成的程式碼中，所有出現 `/@imagetools` 的位置前加上開發伺服器的 URL：

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';
import { imagetools } from 'vite-imagetools';

export default defineConfig({
    plugins: [
        laravel({
            // ...
            transformOnServe: (code, devServerUrl) => code.replaceAll('/@imagetools', devServerUrl+'/@imagetools'),
        }),
        imagetools(),
    ],
});
```

現在，當 Vite 提供資產時，它將會輸出指向 Vite 開發伺服器的 URL：

```html
- <img src="/@imagetools/f0b2f404b13f052c604e632f2fb60381bf61a520"><!-- [tl! remove] -->
+ <img src="http://[::1]:5173/@imagetools/f0b2f404b13f052c604e632f2fb60381bf61a520"><!-- [tl! add] -->
```