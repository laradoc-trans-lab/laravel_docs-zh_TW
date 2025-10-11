# 資源打包 (Vite)

- [簡介](#introduction)
- [安裝與設定](#installation)
  - [安裝 Node](#installing-node)
  - [安裝 Vite 與 Laravel Plugin](#installing-vite-and-laravel-plugin)
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
  - [使用 Vite 處理靜態資源](#blade-processing-static-assets)
  - [儲存時重新整理](#blade-refreshing-on-save)
  - [別名](#blade-aliases)
- [資源預載](#asset-prefetching)
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

[Vite](https://vitejs.dev) 是一個現代化的前端建構工具，它提供了極速的開發環境，並將您的程式碼打包以供生產環境使用。當使用 Laravel 建構應用程式時，您通常會使用 Vite 將應用程式的 CSS 和 JavaScript 檔案打包成準備好用於生產的資源。

Laravel 透過提供官方外掛和 Blade 指令，與 Vite 無縫整合，以便在開發和生產環境中載入您的資源。

> [!NOTE]  
> 您正在執行 Laravel Mix 嗎？在新的 Laravel 安裝中，Vite 已取代 Laravel Mix。有關 Mix 的文件，請造訪 [Laravel Mix](https://laravel-mix.com/) 網站。如果您想切換到 Vite，請參閱我們的[遷移指南](https://github.com/laravel/vite-plugin/blob/main/UPGRADE.md#migrating-from-laravel-mix-to-vite)。


<a name="vite-or-mix"></a>
#### 選擇 Vite 或 Laravel Mix

在轉換到 Vite 之前，新的 Laravel 應用程式在打包資源時，是使用由 [webpack](https://webpack.js.org/) 驅動的 [Mix](https://laravel-mix.com/)。Vite 專注於在建構豐富的 JavaScript 應用程式時提供更快、更高效的體驗。如果您正在開發單頁應用程式 (SPA)，包括使用 [Inertia](https://inertiajs.com) 等工具開發的應用程式，Vite 將是完美契合的選擇。

Vite 也適用於傳統的伺服器端渲染應用程式，即便其中帶有輕量級 JavaScript，包括那些使用 [Livewire](https://livewire.laravel.com) 的應用程式。然而，它缺少 Laravel Mix 支援的一些功能，例如將未直接引用到 JavaScript 應用程式中的任意資源複製到建構輸出的功能。


<a name="migrating-back-to-mix"></a>
#### 遷移回 Mix

您是否已使用我們的 Vite 骨架開始了一個新的 Laravel 應用程式，但需要切換回 Laravel Mix 和 webpack？沒問題。請查閱我們的[從 Vite 遷移到 Mix 的官方指南](https://github.com/laravel/vite-plugin/blob/main/UPGRADE.md#migrating-from-vite-to-laravel-mix)。

<a name="installation"></a>
## 安裝與設定

> [!NOTE]  
> 以下文件討論如何手動安裝與配置 Laravel Vite plugin。然而，Laravel 的 [入門套件](/docs/{{version}}/starter-kits) 已經包含了所有這些腳手架，是使用 Laravel 與 Vite 最快的方式。

<a name="installing-node"></a>
### 安裝 Node

在執行 Vite 與 Laravel plugin 之前，你必須確保 Node.js (16+) 和 NPM 已安裝：

```sh
node -v
npm -v
```

你可以從 [官方 Node 網站](https://nodejs.org/en/download/) 透過簡易的圖形安裝程式輕鬆安裝最新版本的 Node 與 NPM。或者，如果你正在使用 [Laravel Sail](https://laravel.com/docs/{{version}}/sail)，你可以透過 Sail 呼叫 Node 與 NPM：

```sh
./vendor/bin/sail node -v
./vendor/bin/sail npm -v
```

<a name="installing-vite-and-laravel-plugin"></a>
### 安裝 Vite 與 Laravel Plugin

在全新安裝的 Laravel 應用程式中，你會在應用程式目錄結構的根目錄找到一個 `package.json` 檔案。預設的 `package.json` 檔案已經包含了開始使用 Vite 與 Laravel plugin 所需的一切。你可以透過 NPM 安裝應用程式的前端依賴套件：

```sh
npm install
```

<a name="configuring-vite"></a>
### 設定 Vite

Vite 是透過位於專案根目錄的 `vite.config.js` 檔案來設定。你可以根據你的需求自訂此檔案，並且還可以安裝應用程式所需的任何其他 plugin，例如 `@vitejs/plugin-vue` 或 `@vitejs/plugin-react`。

Laravel Vite plugin 要求你指定應用程式的進入點。這些可以是 JavaScript 或 CSS 檔案，並且包含預處理語言，例如 TypeScript、JSX、TSX 與 Sass。

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

如果你正在建構一個 SPA，包括使用 Inertia 建構的應用程式，Vite 在沒有 CSS 進入點的情況下運作最佳：

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

相反地，你應該透過 JavaScript 引入你的 CSS。通常，這會在應用程式的 `resources/js/app.js` 檔案中完成：

```js
import './bootstrap';
import '../css/app.css'; // [tl! add]
```

Laravel plugin 也支援多個進入點以及進階設定選項，例如 [SSR 進入點](#ssr)。

<a name="working-with-a-secure-development-server"></a>
#### 使用安全的開發伺服器

如果你的本機開發網頁伺服器透過 HTTPS 提供應用程式服務，你在連接到 Vite 開發伺服器時可能會遇到問題。

如果你正在使用 [Laravel Herd](https://herd.laravel.com) 並已保護網站，或者你正在使用 [Laravel Valet](/docs/{{version}}/valet) 並已對你的應用程式執行 [secure 指令](/docs/{{version}}/valet#securing-sites)，則 Laravel Vite plugin 將會自動偵測並使用為你產生的 TLS 憑證。

如果你使用不符合應用程式目錄名稱的主機保護網站，你可以在應用程式的 `vite.config.js` 檔案中手動指定主機：

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

當使用另一個網頁伺服器時，你應該產生受信任的憑證，並手動設定 Vite 使用產生的憑證：

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

如果你無法為你的系統產生受信任的憑證，你可以安裝並設定 [`@vitejs/plugin-basic-ssl` plugin](https://github.com/vitejs/vite-plugin-basic-ssl)。當使用不受信任的憑證時，你需要在瀏覽器中接受 Vite 開發伺服器的憑證警告，方法是在執行 `npm run dev` 指令時依照控制台中「Local」連結的指示。

<a name="configuring-hmr-in-sail-on-wsl2"></a>
#### 在 WSL2 上的 Sail 中執行開發伺服器

當在 Windows Subsystem for Linux 2 (WSL2) 上於 [Laravel Sail](/docs/{{version}}/sail) 內執行 Vite 開發伺服器時，你應該將以下設定新增到你的 `vite.config.js` 檔案中，以確保瀏覽器能與開發伺服器通訊：

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

如果你的檔案變更未在瀏覽器中顯示，而開發伺服器正在執行，你可能還需要設定 Vite 的 [`server.watch.usePolling` 選項](https://vitejs.dev/config/server-options.html#server-watch)。

<a name="loading-your-scripts-and-styles"></a>
### 載入你的腳本與樣式

設定好 Vite 進入點後，你現在可以在應用程式根範本的 `<head>` 中，透過 `@vite()` Blade directive 引用它們：

```blade
<!DOCTYPE html>
<head>
    {{-- ... --}}

    @vite(['resources/css/app.css', 'resources/js/app.js'])
</head>
```

如果你透過 JavaScript 引入你的 CSS，你只需要包含 JavaScript 進入點即可：

```blade
<!DOCTYPE html>
<head>
    {{-- ... --}}

    @vite('resources/js/app.js')
</head>
```

`@vite` directive 將會自動偵測 Vite 開發伺服器，並注入 Vite client 以啟用 Hot Module Replacement (熱模組替換)。在建構模式下，此 directive 將會載入你編譯並版本化的資源，包括任何引入的 CSS。

如果需要，你也可以在呼叫 `@vite` directive 時指定編譯後資源的建構路徑：

```blade
<!doctype html>
<head>
    {{-- Given build path is relative to public path. --}}

    @vite('resources/js/app.js', 'vendor/courier/build')
</head>
```

<a name="inline-assets"></a>
#### 行內資源

有時候，可能需要將資源的原始內容直接包含在頁面中，而不是連結到資源的版本化 URL。例如，當將 HTML 內容傳遞給 PDF 產生器時，你可能需要將資源內容直接包含到你的頁面中。你可以使用 `Vite` facade 提供的 `content` 方法輸出 Vite 資源的內容：

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

有兩種方法可以執行 Vite。你可以透過 `dev` 指令執行開發伺服器，這在本地開發時非常有用。開發伺服器會自動偵測你檔案的變更，並立即反映到任何開啟的瀏覽器視窗中。

或者，執行 `build` 指令會為你的應用程式資源進行版本化和打包，並讓它們準備好部署到生產環境：

```shell
# Run the Vite development server...
npm run dev

# Build and version the assets for production...
npm run build
```

如果你正在 WSL2 上的 [Sail](/docs/{{version}}/sail) 中執行開發伺服器，你可能需要一些[額外設定](#configuring-hmr-in-sail-on-wsl2)選項。

<a name="working-with-scripts"></a>
## 使用 JavaScript


<a name="aliases"></a>
### 別名

預設情況下，Laravel plugin 提供了一個常見的別名，可幫助您快速入門並方便地引入應用程式的資源：

```js
{
    '@' => '/resources/js'
}
```

您可以透過在 `vite.config.js` 設定檔中加入自己的別名來覆寫 `'@'` 別名：

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

如果您希望使用 [Vue](https://vuejs.org/) 框架來建構前端，那麼您也需要安裝 `@vitejs/plugin-vue` plugin：

```sh
npm install --save-dev @vitejs/plugin-vue
```

接著，您可以在 `vite.config.js` 設定檔中包含此 plugin。在使用 Laravel 的 Vue plugin 時，您還需要一些額外的選項：

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
> Laravel 的 [入門套件](/docs/{{version}}/starter-kits) 已經包含了適當的 Laravel、Vue 與 Vite 設定。請查看 [Laravel Breeze](/docs/{{version}}/starter-kits#breeze-and-inertia)，這是使用 Laravel、Vue 與 Vite 快速入門的最佳方式。


<a name="react"></a>
### React

如果您希望使用 [React](https://reactjs.org/) 框架來建構前端，那麼您也需要安裝 `@vitejs/plugin-react` plugin：

```sh
npm install --save-dev @vitejs/plugin-react
```

接著，您可以在 `vite.config.js` 設定檔中包含此 plugin：

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

您需要確保任何包含 JSX 的檔案都帶有 `.jsx` 或 `.tsx` 副檔名，並記得視需要更新您的進入點，如 [上方所示](#configuring-vite)。

您還需要將額外的 `@viteReactRefresh` Blade directive 與您現有的 `@vite` directive 一起包含進來。

```blade
@viteReactRefresh
@vite('resources/js/app.jsx')
```

`@viteReactRefresh` directive 必須在 `@vite` directive 之前呼叫。

> [!NOTE]  
> Laravel 的 [入門套件](/docs/{{version}}/starter-kits) 已經包含了適當的 Laravel、React 與 Vite 設定。請查看 [Laravel Breeze](/docs/{{version}}/starter-kits#breeze-and-inertia)，這是使用 Laravel、React 與 Vite 快速入門的最佳方式。


<a name="inertia"></a>
### Inertia

Laravel Vite plugin 提供了一個方便的 `resolvePageComponent` 函數，可幫助您解析 Inertia 頁面元件。以下是 Vue 3 中使用此 helper 的範例；不過，您也可以在其他框架（如 React）中利用此函數：

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

如果您正將 Vite 的程式碼分割功能與 Inertia 搭配使用，我們建議設定 [資源預載](#asset-prefetching)。

> [!NOTE]  
> Laravel 的 [入門套件](/docs/{{version}}/starter-kits) 已經包含了適當的 Laravel、Inertia 與 Vite 設定。請查看 [Laravel Breeze](/docs/{{version}}/starter-kits#breeze-and-inertia)，這是使用 Laravel、Inertia 與 Vite 快速入門的最佳方式。


<a name="url-processing"></a>
### URL 處理

當使用 Vite 並在應用程式的 HTML、CSS 或 JS 中引用資源時，有幾點需要考慮。首先，如果您使用絕對路徑引用資源，Vite 將不會在建構中包含該資源；因此，您應確保該資源在您的公共目錄中可用。在使用 [專用的 CSS 進入點](#configuring-vite) 時，應避免使用絕對路徑，因為在開發期間，瀏覽器會嘗試從託管 CSS 的 Vite 開發伺服器載入這些路徑，而不是從您的公共目錄載入。

當引用相對資源路徑時，您應記住這些路徑是相對於它們被引用的檔案。任何透過相對路徑引用的資源都將由 Vite 重新寫入、版本化並打包。

請考慮以下專案結構：

```nothing
public/
  taylor.png
resources/
  js/
    Pages/
      Welcome.vue
  images/
    abigail.png
```

以下範例展示了 Vite 如何處理相對與絕對 URL：

```html
<!-- This asset is not handled by Vite and will not be included in the build -->
<img src="/taylor.png">

<!-- This asset will be re-written, versioned, and bundled by Vite -->
<img src="../../images/abigail.png">
```


<a name="working-with-stylesheets"></a>
## 使用樣式表

您可以在 [Vite 文件](https://vitejs.dev/guide/features.html#css) 中了解更多關於 Vite 的 CSS 支援。如果您正在使用 PostCSS plugins (例如 [Tailwind](https://tailwindcss.com))，您可以在專案的根目錄中建立一個 `postcss.config.js` 檔案，Vite 將會自動應用它：

```js
export default {
    plugins: {
        tailwindcss: {},
        autoprefixer: {},
    },
};
```

> [!NOTE]  
> Laravel 的 [入門套件](/docs/{{version}}/starter-kits) 已經包含了適當的 Tailwind、PostCSS 與 Vite 設定。或者，如果您希望在不使用我們的入門套件的情況下使用 Tailwind 與 Laravel，請查看 [Tailwind 的 Laravel 安裝指南](https://tailwindcss.com/docs/guides/laravel)。

<a name="working-with-blade-and-routes"></a>
## 使用 Blade 與路由


<a name="blade-processing-static-assets"></a>
### 使用 Vite 處理靜態資源

當在 JavaScript 或 CSS 中引用資源時，Vite 會自動處理並進行版本控制。此外，在建構基於 Blade 的應用程式時，Vite 也能處理並版本控制僅在 Blade 模板中引用的靜態資源。

然而，為了實現這一點，你需要將靜態資源匯入應用程式的進入點，讓 Vite 知道它們的存在。舉例來說，如果你想處理並版本控制儲存在 `resources/images` 中的所有圖片，以及儲存在 `resources/fonts` 中的所有字型，你應該在應用程式的 `resources/js/app.js` 進入點中加入以下內容：

```js
import.meta.glob([
  '../images/**',
  '../fonts/**',
]);
```

這些資源現在將在執行 `npm run build` 時由 Vite 處理。然後，你可以在 Blade 模板中使用 `Vite::asset` 方法來引用這些資源，它會返回指定資源的版本化 URL：

```blade
<img src="{{ Vite::asset('resources/images/logo.png') }}">
```


<a name="blade-refreshing-on-save"></a>
### 儲存時重新整理

當你的應用程式使用傳統伺服器端渲染 (搭配 Blade) 建構時，Vite 可以透過在您修改應用程式中的視圖檔案時自動重新整理瀏覽器來改善你的開發工作流程。要開始使用，你只需將 `refresh` 選項設定為 `true`。

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

當 `refresh` 選項為 `true` 時，在執行 `npm run dev` 期間，儲存以下目錄中的檔案將觸發瀏覽器執行整頁重新整理：

- `app/Livewire/**`
- `app/View/Components/**`
- `lang/**`
- `resources/lang/**`
- `resources/views/**`
- `routes/**`

監控 `routes/**` 目錄非常有用，尤其當您利用 [Ziggy](https://github.com/tighten/ziggy) 在應用程式前端生成路由連結時。

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

在底層，Laravel Vite plugin 使用了 [`vite-plugin-full-reload`](https://github.com/ElMassimo/vite-plugin-full-reload) 套件，該套件提供了一些進階配置選項，可以微調此功能的行為。如果你需要這種程度的自訂，可以提供一個 `config` 定義：

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

在 JavaScript 應用程式中，為常用目錄 [建立別名](#aliases) 是很常見的。但是，你也可以透過在 `Illuminate\Support\Facades\Vite` 類別上使用 `macro` 方法，來建立在 Blade 中使用的別名。通常，「巨集 (macro)」應該在 [服務提供者](/docs/{{version}}/providers) 的 `boot` 方法中定義：

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        Vite::macro('image', fn (string $asset) => $this->asset("resources/images/{$asset}"));
    }

一旦定義了巨集，就可以在你的模板中呼叫它。例如，我們可以使用上面定義的 `image` 巨集來引用位於 `resources/images/logo.png` 的資源：

```blade
<img src="{{ Vite::image('logo.png') }}" alt="Laravel Logo">
```


<a name="asset-prefetching"></a>
## 資源預載

當使用 Vite 的程式碼分割功能建構 SPA 時，所需的資源會在每次頁面導航時被取用。這種行為可能導致 UI 渲染延遲。如果這對你選擇的前端框架造成問題，Laravel 提供了在初始頁面載入時預先載入 (eagerly prefetch) 應用程式 JavaScript 和 CSS 資源的功能。

你可以透過在 [服務提供者](/docs/{{version}}/providers) 的 `boot` 方法中呼叫 `Vite::prefetch` 方法，來指示 Laravel 預先載入你的資源：

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

在上面的範例中，資源將在每次頁面載入時，以最多 `3` 個併發下載的方式進行預載。你可以修改併發數以符合你的應用程式需求，或者如果應用程式應該一次下載所有資源，則不指定併發限制：

```php
/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Vite::prefetch();
}
```

預設情況下，預載會在 [頁面載入事件](https://developer.mozilla.org/en-US/docs/Web/API/Window/load_event) 觸發時開始。如果你想自訂預載何時開始，可以指定一個 Vite 將監聽的事件：

```php
/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Vite::prefetch(event: 'vite:prefetch');
}
```

根據上述程式碼，當你在 `window` 物件上（手動） dispatch `vite:prefetch` 事件時，預載就會開始。例如，你可以讓預載在頁面載入三秒後開始：

```html
<script>
    addEventListener('load', () => setTimeout(() => {
        dispatchEvent(new Event('vite:prefetch'))
    }, 3000))
</script>
```


<a name="custom-base-urls"></a>
## 自訂基礎 URL

如果你的 Vite 編譯資源部署在與應用程式不同的網域上，例如透過 CDN，你必須在應用程式的 `.env` 檔案中指定 `ASSET_URL` 環境變數：

```env
ASSET_URL=https://cdn.example.com
```

設定資源 URL 後，所有重新編寫的資源 URL 都將以配置的值作為前綴：

```nothing
https://cdn.example.com/build/assets/app.9dce8d17.js
```

請記住，[絕對 URL 不會被 Vite 重寫](#url-processing)，因此它們不會被加上前綴。


<a name="environment-variables"></a>
## 環境變數

你可以透過在應用程式的 `.env` 檔案中，將環境變數以 `VITE_` 作為前綴，來將它們注入到 JavaScript 中：

```env
VITE_SENTRY_DSN_PUBLIC=http://example.com
```

你可以透過 `import.meta.env` 物件來存取注入的環境變數：

```js
import.meta.env.VITE_SENTRY_DSN_PUBLIC
```

<a name="disabling-vite-in-tests"></a>
## 在測試中停用 Vite

Laravel 的 Vite 整合將在執行測試時嘗試解析您的資產，這要求您運行 Vite 開發伺服器或建構您的資產。

如果您希望在測試期間模擬 Vite，可以呼叫 `withoutVite` 方法，此方法適用於任何繼承 Laravel `TestCase` 類別的測試。

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

如果您想為所有測試停用 Vite，可以在基礎 `TestCase` 類別的 `setUp` 方法中呼叫 `withoutVite` 方法：

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

Laravel Vite plugin 讓使用 Vite 設定伺服器端渲染變得輕而易舉。首先，在 `resources/js/ssr.js` 建立一個 SSR 進入點，並透過向 Laravel plugin 傳遞設定選項來指定該進入點：

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

為確保您不會忘記重新建構 SSR 進入點，我們建議在您的應用程式 `package.json` 中增強「build」腳本以建立您的 SSR 建構：

```json
"scripts": {
     "dev": "vite",
     "build": "vite build" // [tl! remove]
     "build": "vite build && vite build --ssr" // [tl! add]
}
```

然後，要建構並啟動 SSR 伺服器，您可以執行以下命令：

```sh
npm run build
node bootstrap/ssr/ssr.js
```

如果您正在使用 [Inertia 的 SSR](https://inertiajs.com/server-side-rendering)，您可以改用 `inertia:start-ssr` Artisan 命令來啟動 SSR 伺服器：

```sh
php artisan inertia:start-ssr
```

> [!NOTE]  
> Laravel 的 [入門套件](/docs/{{version}}/starter-kits) 已經包含適當的 Laravel、Inertia SSR 和 Vite 設定。查看 [Laravel Breeze](/docs/{{version}}/starter-kits#breeze-and-inertia) 以最快的方式開始使用 Laravel、Inertia SSR 和 Vite。

<a name="script-and-style-attributes"></a>
## 腳本與樣式標籤屬性

<a name="content-security-policy-csp-nonce"></a>
### 內容安全策略 (CSP) Nonce

如果您希望在腳本和樣式標籤中包含 [`nonce` 屬性](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/nonce) 作為 [內容安全策略](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP) 的一部分，您可以在自訂的 [中介層](/docs/{{version}}/middleware) 中使用 `useCspNonce` 方法來產生或指定一個 nonce：

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

呼叫 `useCspNonce` 方法後，Laravel 將自動在所有產生的腳本和樣式標籤上包含 `nonce` 屬性。

如果您需要在其他地方指定 nonce，包括 Laravel [入門套件](/docs/{{version}}/starter-kits) 中包含的 [Ziggy `@route` directive](https://github.com/tighten/ziggy#using-routes-with-a-content-security-policy)，您可以使用 `cspNonce` 方法來擷取它：

```blade
@routes(nonce: Vite::cspNonce())
```

如果您已經有一個 nonce 並且希望 Laravel 使用它，您可以將該 nonce 傳遞給 `useCspNonce` 方法：

```php
Vite::useCspNonce($nonce);
```

<a name="subresource-integrity-sri"></a>
### 子資源完整性 (SRI)

如果您的 Vite manifest 包含資產的 `integrity` 雜湊，Laravel 將自動在它產生的任何腳本和樣式標籤上添加 `integrity` 屬性，以強制執行 [子資源完整性](https://developer.mozilla.org/en-US/docs/Web/Security/Subresource_Integrity)。預設情況下，Vite 不會在其 manifest 中包含 `integrity` 雜湊，但您可以透過安裝 [`vite-plugin-manifest-sri`](https://www.npmjs.com/package/vite-plugin-manifest-sri) NPM plugin 來啟用它：

```shell
npm install --save-dev vite-plugin-manifest-sri
```

然後，您可以在 `vite.config.js` 檔案中啟用此 plugin：

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

如果需要，您還可以自訂 manifest 中可以找到 integrity 雜湊的鍵：

```php
use Illuminate\Support\Facades\Vite;

Vite::useIntegrityKey('custom-integrity-key');
```

如果您想完全停用此自動偵測功能，可以將 `false` 傳遞給 `useIntegrityKey` 方法：

```php
Vite::useIntegrityKey(false);
```

<a name="arbitrary-attributes"></a>
### 任意屬性

如果您需要在腳本和樣式標籤中包含額外的屬性，例如 [`data-turbo-track`](https://turbo.hotwired.dev/handbook/drive#reloading-when-assets-change) 屬性，可以透過 `useScriptTagAttributes` 和 `useStyleTagAttributes` 方法來指定它們。通常，這些方法應該從 [服務提供者](/docs/{{version}}/providers) 中呼叫：

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

如果您需要有條件地添加屬性，可以傳遞一個回調函數，該函數將接收資產的來源路徑、其 URL、其 manifest 區塊以及整個 manifest：

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
> 當 Vite 開發伺服器運行時，`$chunk` 和 `$manifest` 參數將為 `null`。

<a name="advanced-customization"></a>
## 進階自訂

開箱即用，Laravel 的 Vite Plugin 採用了合理的慣例，應能適用於大多數應用程式；然而，有時您可能需要自訂 Vite 的行為。為了啟用額外的自訂選項，我們提供了以下方法和選項，可用於取代 `@vite` Blade 指令：

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

在 `vite.config.js` 檔案中，您應該接著指定相同的設定：

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

如果您在瀏覽器中從 Vite 開發伺服器擷取資源時遇到跨來源資源共享 (CORS) 問題，您可能需要授予您的自訂來源存取開發伺服器的權限。Vite 結合 Laravel plugin 允許以下來源，無需額外設定：

- `::1`
- `127.0.0.1`
- `localhost`
- `*.test`
- `*.localhost`
- `APP_URL` in the project's `.env`

允許專案使用自訂來源最簡單的方法是，確保您應用程式的 `APP_URL` 環境變數與您在瀏覽器中存取的來源相符。例如，如果您正在存取 `https://my-app.laravel`，您應該更新您的 `.env` 以符合：

```env
APP_URL=https://my-app.laravel
```

如果您需要對來源進行更精細的控制，例如支援多個來源，您應該利用 [Vite 全面且靈活的內建 CORS 伺服器設定](https://vite.dev/config/server-options.html#server-cors)。例如，您可以在專案的 `vite.config.js` 檔案中，在 `server.cors.origin` 設定選項中指定多個來源：

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

您也可以包含 regex 模式，如果您希望允許特定頂級網域的所有來源（例如 `*.laravel`），這會很有幫助：

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

Vite 生態系統中的某些 plugin 假設以斜線開頭的 URL 總是會指向 Vite 開發伺服器。然而，由於 Laravel 整合的特性，情況並非如此。

例如，當 Vite 提供您的資源時，`vite-imagetools` plugin 會輸出如下所示的 URL：

```html
<img src="/@imagetools/f0b2f404b13f052c604e632f2fb60381bf61a520">
```

`vite-imagetools` plugin 預期輸出 URL 會被 Vite 攔截，然後該 plugin 就可以處理所有以 `/@imagetools` 開頭的 URL。如果您使用的 plugin 預期這種行為，您將需要手動修正這些 URL。您可以在 `vite.config.js` 檔案中使用 `transformOnServe` 選項來執行此操作。

在這個特定範例中，我們將在生成程式碼中所有出現 `/@imagetools` 的地方預先加入開發伺服器 URL：

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