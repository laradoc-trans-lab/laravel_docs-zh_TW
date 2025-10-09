# 資源打包 (Vite)

- [簡介](#introduction)
- [安裝與設定](#installation)
  - [安裝 Node](#installing-node)
  - [安裝 Vite 與 Laravel 外掛](#installing-vite-and-laravel-plugin)
  - [設定 Vite](#configuring-vite)
  - [載入您的指令碼與樣式](#loading-your-scripts-and-styles)
- [執行 Vite](#running-vite)
- [使用 JavaScript](#working-with-scripts)
  - [別名](#aliases)
  - [Vue](#vue)
  - [React](#react)
  - [Inertia](#inertia)
  - [URL 處理](#url-processing)
- [使用樣式表](#working-with-stylesheets)
- [使用 Blade 與路由](#working-with-blade-and-routes)
  - [使用 Vite 處理靜態資源](#blade-processing-static-assets)
  - [儲存時重新整理](#blade-refreshing-on-save)
  - [別名](#blade-aliases)
- [資源預先擷取](#asset-prefetching)
- [自訂基礎 URL](#custom-base-urls)
- [環境變數](#environment-variables)
- [在測試中停用 Vite](#disabling-vite-in-tests)
- [伺服器端渲染 (SSR)](#ssr)
- [指令碼與樣式標籤屬性](#script-and-style-attributes)
  - [內容安全策略 (CSP) Nonce](#content-security-policy-csp-nonce)
  - [子資源完整性 (SRI)](#subresource-integrity-sri)
  - [任意屬性](#arbitrary-attributes)
- [進階自訂](#advanced-customization)
  - [開發伺服器跨來源資源共用 (CORS)](#cors)
  - [更正開發伺服器 URL](#correcting-dev-server-urls)

<a name="introduction"></a>
## 簡介

[Vite](https://vitejs.dev) 是一個現代化的前端建構工具，它提供了極快的開發環境並將您的程式碼打包以用於正式環境。當使用 Laravel 建構應用程式時，您通常會使用 Vite 將應用程式的 CSS 和 JavaScript 檔案打包成可供正式環境使用的資源。

Laravel 透過提供官方外掛和 Blade 指令來無縫整合 Vite，以在開發和正式環境中載入您的資源。

<a name="installation"></a>
## 安裝與設定

> [!NOTE]
> 以下文件說明了如何手動安裝和設定 Laravel Vite 外掛。然而，Laravel 的 [starter kits](/docs/{{version}}/starter-kits) 已包含所有這些基礎設定，是使用 Laravel 和 Vite 最快的方式。

<a name="installing-node"></a>
### 安裝 Node

在執行 Vite 和 Laravel 外掛之前，您必須確保已安裝 Node.js (16+) 和 NPM：

```shell
node -v
npm -v
```

您可以從 [Node 官方網站](https://nodejs.org/en/download/) 使用簡單的圖形安裝程式輕鬆安裝最新版本的 Node 和 NPM。或者，如果您正在使用 [Laravel Sail](https://laravel.com/docs/{{version}}/sail)，您可以透過 Sail 呼叫 Node 和 NPM：

```shell
./vendor/bin/sail node -v
./vendor/bin/sail npm -v
```

<a name="installing-vite-and-laravel-plugin"></a>
### 安裝 Vite 與 Laravel 外掛

在全新安裝的 Laravel 中，您會在應用程式目錄結構的根目錄中找到一個 `package.json` 檔案。預設的 `package.json` 檔案已包含開始使用 Vite 和 Laravel 外掛所需的一切。您可以透過 NPM 安裝應用程式的前端依賴項：

```shell
npm install
```

<a name="configuring-vite"></a>
### 設定 Vite

Vite 是透過專案根目錄中的 `vite.config.js` 檔案進行設定的。您可以根據需要自由自訂此檔案，並且還可以安裝應用程式所需的任何其他外掛，例如 `@vitejs/plugin-vue` 或 `@vitejs/plugin-react`。

Laravel Vite 外掛要求您指定應用程式的進入點。這些可以是 JavaScript 或 CSS 檔案，並包括 TypeScript、JSX、TSX 和 Sass 等預處理語言。

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

如果您正在建構 SPA，包括使用 Inertia 建構的應用程式，Vite 最好不要有 CSS 進入點：

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

相反地，您應該透過 JavaScript 匯入 CSS。通常，這會在應用程式的 `resources/js/app.js` 檔案中完成：

```js
import './bootstrap';
import '../css/app.css'; // [tl! add]
```

Laravel 外掛還支援多個進入點和進階設定選項，例如 [SSR 進入點](#ssr)。

<a name="working-with-a-secure-development-server"></a>
#### 使用安全的開發伺服器

如果您的本機開發網頁伺服器是透過 HTTPS 提供您的應用程式，您可能會遇到連接到 Vite 開發伺服器的問題。

如果您正在使用 [Laravel Herd](https://herd.laravel.com) 並已保護網站安全，或者您正在使用 [Laravel Valet](/docs/{{version}}/valet) 並已對您的應用程式執行 [secure 命令](/docs/{{version}}/valet#securing-sites)，則 Laravel Vite 外掛會自動偵測並為您使用產生的 TLS 憑證。

如果您使用與應用程式目錄名稱不符的主機來保護網站安全，您可以手動在應用程式的 `vite.config.js` 檔案中指定主機：

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

當使用其他網頁伺服器時，您應該產生一個受信任的憑證，並手動設定 Vite 使用產生的憑證：

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

如果您無法為您的系統產生受信任的憑證，您可以安裝並設定 [@vitejs/plugin-basic-ssl 外掛](https://github.com/vitejs/vite-plugin-basic-ssl)。當使用不受信任的憑證時，您需要在執行 `npm run dev` 命令時，在瀏覽器中點擊主控台中的「Local」連結，接受 Vite 開發伺服器的憑證警告。

<a name="configuring-hmr-in-sail-on-wsl2"></a>
#### 在 WSL2 上的 Sail 中執行開發伺服器

當在 Windows Sub系統 Linux 2 (WSL2) 上於 [Laravel Sail](/docs/{{version}}/sail) 中執行 Vite 開發伺服器時，您應該在 `vite.config.js` 檔案中加入以下設定，以確保瀏覽器可以與開發伺服器溝通：

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

如果您的檔案變更在開發伺服器執行時未反映在瀏覽器中，您可能還需要設定 Vite 的 [server.watch.usePolling option](https://vitejs.dev/config/server-options.html#server-watch)。

<a name="loading-your-scripts-and-styles"></a>
### 載入您的指令碼與樣式

設定好您的 Vite 進入點後，您現在可以將它們參考到您應用程式根範本的 `<head>` 中新增的 `@vite()` Blade 指令中：

```blade
<!DOCTYPE html>
<head>
    {{-- ... --}}

    @vite(['resources/css/app.css', 'resources/js/app.js'])
</head>
```

如果您是透過 JavaScript 匯入 CSS，則只需包含 JavaScript 進入點：

```blade
<!DOCTYPE html>
<head>
    {{-- ... --}}

    @vite('resources/js/app.js')
</head>
```

`@vite` 指令會自動偵測 Vite 開發伺服器並注入 Vite client 以啟用 Hot Module Replacement。在建構模式中，此指令將載入您的已編譯和版本化資源，包括任何匯入的 CSS。

如果需要，您也可以在呼叫 `@vite` 指令時指定已編譯資源的建構路徑：

```blade
<!doctype html>
<head>
    {{-- Given build path is relative to public path. --}}

    @vite('resources/js/app.js', 'vendor/courier/build')
</head>
```

<a name="inline-assets"></a>
#### 行內資源

有時可能需要包含資源的原始內容，而不是連結到資源的版本化 URL。例如，當將 HTML 內容傳遞給 PDF 產生器時，您可能需要將資源內容直接包含到您的頁面中。您可以使用 `Vite` facade 提供的 `content` 方法輸出 Vite 資源的內容：

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

您可以透過兩種方式執行 Vite。您可以透過 `dev` 指令執行開發伺服器，這對於本機開發非常有用。開發伺服器會自動偵測您檔案的變更，並即時反映在任何開啟的瀏覽器視窗中。

或者，執行 `build` 指令將會對您應用程式的資產進行版本化並打包，使其準備好部署到生產環境：

```shell
# Run the Vite development server...
npm run dev

# Build and version the assets for production...
npm run build
```

如果您在 WSL2 的 [Sail](/docs/{{version}}/sail) 中執行開發伺服器，您可能需要一些[額外設定](#configuring-hmr-in-sail-on-wsl2)選項。


<a name="working-with-scripts"></a>
## 使用 JavaScript


<a name="aliases"></a>
### 別名

預設情況下，Laravel 外掛提供了一個常用別名，可幫助您快速上手並方便地匯入應用程式的資產：

```js
{
    '@' => '/resources/js'
}
```

您可以將自己的別名新增到 `vite.config.js` 設定檔中，以覆寫 `'@'` 別名：

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

如果您想使用 [Vue](https://vuejs.org/) 框架建構前端，那麼您還需要安裝 `@vitejs/plugin-vue` 外掛：

```shell
npm install --save-dev @vitejs/plugin-vue
```

接著，您可以將該外掛包含在您的 `vite.config.js` 設定檔中。將 Vue 外掛與 Laravel 搭配使用時，您將需要一些額外的選項：

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
> Laravel 的[入門套件](/docs/{{version}}/starter-kits)已經包含了適當的 Laravel、Vue 和 Vite 設定。這些入門套件提供了最快的方式來開始使用 Laravel、Vue 和 Vite。


<a name="react"></a>
### React

如果您想使用 [React](https://reactjs.org/) 框架建構前端，那麼您還需要安裝 `@vitejs/plugin-react` 外掛：

```shell
npm install --save-dev @vitejs/plugin-react
```

接著，您可以將該外掛包含在您的 `vite.config.js` 設定檔中：

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

您需要確保任何包含 JSX 的檔案都具有 `.jsx` 或 `.tsx` 副檔名，並記得視需要更新您的進入點，如同[上方所示](#configuring-vite)。

您還需要將額外的 `@viteReactRefresh` Blade 指令與您現有的 `@vite` 指令一起包含進來。

```blade
@viteReactRefresh
@vite('resources/js/app.jsx')
```

`@viteReactRefresh` 指令必須在 `@vite` 指令之前呼叫。

> [!NOTE]
> Laravel 的[入門套件](/docs/{{version}}/starter-kits)已經包含了適當的 Laravel、React 和 Vite 設定。這些入門套件提供了最快的方式來開始使用 Laravel、React 和 Vite。


<a name="inertia"></a>
### Inertia

Laravel Vite 外掛提供了一個方便的 `resolvePageComponent` 函式，以幫助您解析 Inertia 頁面元件。以下是該輔助函式與 Vue 3 搭配使用的範例；然而，您也可以在其他框架（例如 React）中使用該函式：

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

如果您正在將 Vite 的程式碼分割功能與 Inertia 搭配使用，我們建議設定[資產預先擷取](#asset-prefetching)。

> [!NOTE]
> Laravel 的[入門套件](/docs/{{version}}/starter-kits)已經包含了適當的 Laravel、Inertia 和 Vite 設定。這些入門套件提供了最快的方式來開始使用 Laravel、Inertia 和 Vite。


<a name="url-processing"></a>
### URL 處理

當使用 Vite 並在您應用程式的 HTML、CSS 或 JS 中引用資產時，有幾個注意事項需要考慮。首先，如果您使用絕對路徑引用資產，Vite 將不會在建構中包含該資產；因此，您應確保該資產在您的 public 目錄中可用。當使用[專用 CSS 進入點](#configuring-vite)時，應避免使用絕對路徑，因為在開發期間，瀏覽器會嘗試從託管 CSS 的 Vite 開發伺服器載入這些路徑，而不是從您的 public 目錄載入。

當引用相對資產路徑時，您應該記住這些路徑是相對於它們被引用的檔案。任何透過相對路徑引用的資產都將由 Vite 重新寫入、版本化並打包。

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

以下範例示範 Vite 將如何處理相對和絕對 URL：

```html
<!-- This asset is not handled by Vite and will not be included in the build -->
<img src="/taylor.png">

<!-- This asset will be re-written, versioned, and bundled by Vite -->
<img src="../../images/abigail.png">
```


<a name="working-with-stylesheets"></a>
## 使用樣式表

> [!NOTE]
> Laravel 的[入門套件](/docs/{{version}}/starter-kits)已經包含了適當的 Tailwind 和 Vite 設定。或者，如果您想在不使用我們的入門套件的情況下使用 Tailwind 和 Laravel，請查閱 [Tailwind 的 Laravel 安裝指南](https://tailwindcss.com/docs/guides/laravel)。

所有 Laravel 應用程式都已經包含了 Tailwind 和一個正確設定的 `vite.config.js` 檔案。因此，您只需啟動 Vite 開發伺服器或執行 `dev` Composer 指令，這將會同時啟動 Laravel 和 Vite 開發伺服器：

```shell
composer run dev
```

您的應用程式 CSS 可以放在 `resources/css/app.css` 檔案中。

<a name="working-with-blade-and-routes"></a>
## 使用 Blade 與路由


<a name="blade-processing-static-assets"></a>
### 使用 Vite 處理靜態資源

當您在 JavaScript 或 CSS 中引用資源時，Vite 會自動處理並版本化這些資源。此外，在使用 Blade 構建應用程式時，Vite 也可以處理並版本化您僅在 Blade 模板中引用的靜態資源。

然而，為了實現這一點，您需要將靜態資源導入到應用程式的進入點，以讓 Vite 感知這些資源。例如，如果您想處理和版本化儲存在 `resources/images` 中的所有圖片以及儲存在 `resources/fonts` 中的所有字體，您應該在應用程式的 `resources/js/app.js` 進入點中添加以下內容：

```js
import.meta.glob([
  '../images/**',
  '../fonts/**',
]);
```

這些資源現在將在執行 `npm run build` 時由 Vite 處理。然後，您可以在 Blade 模板中使用 `Vite::asset` 方法引用這些資源，該方法將返回給定資源的版本化 URL：

```blade
<img src="{{ Vite::asset('resources/images/logo.png') }}">
```


<a name="blade-refreshing-on-save"></a>
### 儲存時重新整理

當您的應用程式使用傳統的伺服器端渲染 (SSR) 搭配 Blade 構建時，Vite 可以透過在您更改應用程式中的視圖檔案時自動重新整理瀏覽器來改進您的開發工作流程。要開始使用，您只需將 `refresh` 選項指定為 `true`。

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

當 `refresh` 選項為 `true` 時，在以下目錄中儲存檔案將觸發瀏覽器執行完整頁面重新整理，而您正在執行 `npm run dev`：

- `app/Livewire/**`
- `app/View/Components/**`
- `lang/**`
- `resources/lang/**`
- `resources/views/**`
- `routes/**`

監看 `routes/**` 目錄非常有用，特別是當您使用 [Ziggy](https://github.com/tighten/ziggy) 在應用程式前端生成路由連結時。

如果這些預設路徑不符合您的需求，您可以指定自己的監看路徑列表：

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

在底層，Laravel Vite 外掛程式使用 [vite-plugin-full-reload](https://github.com/ElMassimo/vite-plugin-full-reload) 套件，該套件提供一些進階配置選項，可微調此功能的行為。如果您需要此級別的自訂，您可以提供 `config` 定義：

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

在 JavaScript 應用程式中，為經常引用的目錄[建立別名](#aliases)是很常見的。但是，您也可以透過在 `Illuminate\Support\Facades\Vite` 類別上使用 `macro` 方法來建立用於 Blade 的別名。通常，「巨集 (macros)」應在 [服務提供者](/docs/{{version}}/providers) 的 `boot` 方法中定義：

```php
/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Vite::macro('image', fn (string $asset) => $this->asset("resources/images/{$asset}"));
}
```

一旦巨集被定義，它就可以在您的模板中調用。例如，我們可以使用上面定義的 `image` 巨集來引用位於 `resources/images/logo.png` 的資源：

```blade
<img src="{{ Vite::image('logo.png') }}" alt="Laravel Logo">
```


<a name="asset-prefetching"></a>
## 資源預先擷取

當使用 Vite 的程式碼分割功能構建單頁應用程式 (SPA) 時，所需的資源會在每次頁面導航時被擷取。這種行為可能導致使用者介面 (UI) 渲染延遲。如果這對您選擇的前端框架來說是個問題，Laravel 提供了在初始頁面載入時預先擷取應用程式 JavaScript 和 CSS 資源的功能。

您可以透過在 [服務提供者](/docs/{{version}}/providers) 的 `boot` 方法中調用 `Vite::prefetch` 方法，指示 Laravel 積極預先擷取您的資源：

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

在上面的範例中，資源將在每次頁面載入時以最多 `3` 個併發下載進行預先擷取。您可以修改併發數以符合應用程式的需求，或者如果不限制應用程式一次下載所有資源，則不指定併發限制：

```php
/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Vite::prefetch();
}
```

依預設，預先擷取將在 [頁面載入 (page _load_) 事件](https://developer.mozilla.org/en-US/docs/Web/API/Window/load_event) 觸發時開始。如果您想自訂預先擷取的開始時機，您可以指定一個 Vite 將監聽的事件：

```php
/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Vite::prefetch(event: 'vite:prefetch');
}
```

根據上面的程式碼，當您手動在 `window` 物件上分發 `vite:prefetch` 事件時，預先擷取將開始。例如，您可以在頁面載入三秒後開始預先擷取：

```html
<script>
    addEventListener('load', () => setTimeout(() => {
        dispatchEvent(new Event('vite:prefetch'))
    }, 3000))
</script>
```


<a name="custom-base-urls"></a>
## 自訂基礎 URL

如果您的 Vite 編譯資源部署到與您的應用程式不同的網域，例如透過 CDN，您必須在應用程式的 `.env` 檔案中指定 `ASSET_URL` 環境變數：

```env
ASSET_URL=https://cdn.example.com
```

設定資源 URL 後，所有重寫的資源 URL 都將以設定的值為前綴：

```text
https://cdn.example.com/build/assets/app.9dce8d17.js
```

請記住，[Vite 不會重寫絕對 URL](#url-processing)，因此它們將不會被加上前綴。


<a name="environment-variables"></a>
## 環境變數

您可以透過在應用程式的 `.env` 檔案中，將環境變數以 `VITE_` 為前綴，將其注入到您的 JavaScript 中：

```env
VITE_SENTRY_DSN_PUBLIC=http://example.com
```

您可以透過 `import.meta.env` 物件存取注入的環境變數：

```js
import.meta.env.VITE_SENTRY_DSN_PUBLIC
```


<a name="disabling-vite-in-tests"></a>
## 在測試中停用 Vite

Laravel 的 Vite 整合將在執行測試時嘗試解析您的資源，這要求您執行 Vite 開發伺服器或建構您的資源。

如果您希望在測試期間模擬 Vite，您可以調用 `withoutVite` 方法，該方法適用於任何擴展 Laravel `TestCase` 類別的測試：

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

如果您想為所有測試停用 Vite，您可以在基礎 `TestCase` 類別的 `setUp` 方法中調用 `withoutVite` 方法：

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

為了確保您不會忘記重新建構 SSR 進入點，我們建議強化您應用程式 `package.json` 中的 "build" 指令碼，以建立您的 SSR 建構：

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

如果您正在使用 [Inertia 進行 SSR](https://inertiajs.com/server-side-rendering)，您可以改用 `inertia:start-ssr` Artisan 指令來啟動 SSR 伺服器：

```shell
php artisan inertia:start-ssr
```

> [!NOTE]
> Laravel 的 [入門套件](/docs/{{version}}/starter-kits) 已經包含了適當的 Laravel、Inertia SSR 與 Vite 設定。這些入門套件是開始使用 Laravel、Inertia SSR 與 Vite 的最快方式。

<a name="script-and-style-attributes"></a>
## 指令碼與樣式標籤屬性

<a name="content-security-policy-csp-nonce"></a>
### 內容安全策略 (CSP) Nonce

如果您希望在指令碼與樣式標籤上包含一個 [nonce 屬性](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/nonce) 作為 [內容安全策略](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP) 的一部分，您可以使用 `useCspNonce` 方法在自訂 [中介層](/docs/{{version}}/middleware) 中產生或指定 nonce：

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

呼叫 `useCspNonce` 方法後，Laravel 將自動在所有產生的指令碼與樣式標籤上包含 `nonce` 屬性。

如果您需要在其他地方指定 nonce，包括 Laravel [入門套件](/docs/{{version}}/starter-kits) 中包含的 [Ziggy `@route` 指令](https://github.com/tighten/ziggy#using-routes-with-a-content-security-policy)，您可以透過 `cspNonce` 方法擷取它：

```blade
@routes(nonce: Vite::cspNonce())
```

如果您已經有一個您想指示 Laravel 使用的 nonce，您可以將該 nonce 傳遞給 `useCspNonce` 方法：

```php
Vite::useCspNonce($nonce);
```

<a name="subresource-integrity-sri"></a>
### 子資源完整性 (SRI)

如果您的 Vite manifest 包含您資源的 `integrity` 雜湊，Laravel 將自動在它產生的任何指令碼與樣式標籤上新增 `integrity` 屬性，以強制執行 [子資源完整性](https://developer.mozilla.org/en-US/docs/Web/Security/Subresource_Integrity)。預設情況下，Vite 不會在其 manifest 中包含 `integrity` 雜湊，但您可以透過安裝 [vite-plugin-manifest-sri](https://www.npmjs.com/package/vite-plugin-manifest-sri) NPM 外掛來啟用它：

```shell
npm install --save-dev vite-plugin-manifest-sri
```

然後您可以在 `vite.config.js` 檔案中啟用意外外掛：

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

如果需要，您還可以自訂可以找到完整性雜湊的 manifest 鍵：

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

如果您需要在指令碼與樣式標籤上包含額外屬性，例如 [data-turbo-track](https://turbo.hotwired.dev/handbook/drive#reloading-when-assets-change) 屬性，您可以透過 `useScriptTagAttributes` 與 `useStyleTagAttributes` 方法來指定它們。通常，這些方法應從 [服務提供者](/docs/{{version}}/providers) 中呼叫：

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

如果您需要有條件地新增屬性，您可以傳遞一個回呼，該回呼將接收資源來源路徑、其 URL、其 manifest chunk 以及整個 manifest：

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
> `$chunk` 和 `$manifest` 引數在 Vite 開發伺服器執行時將為 `null`。

<a name="advanced-customization"></a>
## 進階自訂

開箱即用，Laravel 的 Vite 外掛使用合理的慣例，應適用於大多數應用程式；然而，有時您可能需要自訂 Vite 的行為。為了啟用額外的自訂選項，我們提供以下方法和選項，可用來取代 `@vite` Blade 指令：

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

在 `vite.config.js` 檔案中，您應該指定相同的設定：

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
### 開發伺服器跨來源資源共用 (CORS)

如果您在瀏覽器中從 Vite 開發伺服器擷取資源時遇到跨來源資源共用 (CORS) 問題，您可能需要授予您的自訂來源存取開發伺服器的權限。Vite 結合 Laravel 外掛，允許以下來源而無需任何額外設定：

- `::1`
- `127.0.0.1`
- `localhost`
- `*.test`
- `*.localhost`
- 專案 `.env` 中的 `APP_URL`

為您的專案允許自訂來源的最簡單方式是，確保您的應用程式 `APP_URL` 環境變數與您在瀏覽器中造訪的來源相符。例如，如果您造訪 `https://my-app.laravel`，您應該更新您的 `.env` 以符合：

```env
APP_URL=https://my-app.laravel
```

如果您需要對來源進行更細緻的控制，例如支援多個來源，您應該利用 [Vite 全面且彈性的內建 CORS 伺服器設定](https://vite.dev/config/server-options.html#server-cors)。例如，您可以在專案的 `vite.config.js` 檔案中的 `server.cors.origin` 設定選項中指定多個來源：

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

您也可以包含正規表達式模式，如果您想允許特定頂級網域的所有來源（例如 `*.laravel`），這會很有幫助：

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
### 更正開發伺服器 URL

Vite 生態系統中的某些外掛假設以斜線開頭的 URL 將始終指向 Vite 開發伺服器。然而，由於 Laravel 整合的特性，情況並非如此。

例如，當 Vite 提供您的資源時，`vite-imagetools` 外掛會輸出如下的 URL：

```html
<img src="/@imagetools/f0b2f404b13f052c604e632f2fb60381bf61a520">
```

`vite-imagetools` 外掛預期輸出 URL 將被 Vite 攔截，然後該外掛便可處理所有以 `/@imagetools` 開頭的 URL。如果您使用的外掛預期這種行為，您將需要手動更正這些 URL。您可以在 `vite.config.js` 檔案中透過使用 `transformOnServe` 選項來執行此操作。

在這個特定範例中，我們將在產生的程式碼中，所有出現的 `/@imagetools` 前加上開發伺服器 URL：

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

現在，當 Vite 提供資源時，它將輸出指向 Vite 開發伺服器的 URL：

```html
- <img src="/@imagetools/f0b2f404b13f052c604e632f2fb60381bf61a520"><!-- [tl! remove] -->
+ <img src="http://[::1]:5173/@imagetools/f0b2f404b13f052c604e632f2fb60381bf61a520"><!-- [tl! add] -->
```