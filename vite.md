# 資源打包 (Vite)

- [簡介](#introduction)
- [安裝與設定](#installation)
  - [安裝 Node](#installing-node)
  - [安裝 Vite 與 Laravel Plugin](#installing-vite-and-laravel-plugin)
  - [設定 Vite](#configuring-vite)
  - [載入您的腳本與樣式](#loading-your-scripts-and-styles)
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
- [資源預先載入](#asset-prefetching)
- [自訂基礎 URL](#custom-base-urls)
- [環境變數](#environment-variables)
- [在測試中停用 Vite](#disabling-vite-in-tests)
- [伺服器端渲染 (SSR)](#ssr)
- [腳本與樣式標籤屬性](#script-and-style-attributes)
  - [內容安全策略 (CSP) Nonce](#content-security-policy-csp-nonce)
  - [子資源完整性 (SRI)](#subresource-integrity-sri)
  - [任意屬性](#arbitrary-attributes)
- [進階自訂](#advanced-customization)
  - [開發伺服器跨來源資源共用 (CORS)](#cors)
  - [修正開發伺服器 URL](#correcting-dev-server-urls)

<a name="introduction"></a>
## 簡介

[Vite](https://vitejs.dev) 是一個現代化的前端建構工具，它提供極快的開發環境，並將您的程式碼打包以供正式環境使用。當您使用 Laravel 建構應用程式時，通常會使用 Vite 將應用程式的 CSS 與 JavaScript 檔案打包成可供正式環境使用的資源。

Laravel 透過提供官方的 plugin 與 Blade directive，可無縫整合 Vite，以便在開發與正式環境中載入您的資源。

> [!NOTE]  
> 您正在執行 Laravel Mix 嗎？Vite 已在新的 Laravel 安裝中取代了 Laravel Mix。有關 Mix 的說明文件，請造訪 [Laravel Mix](https://laravel-mix.com/) 網站。如果您想切換到 Vite，請參閱我們的 [遷移指南](https://github.com/laravel/vite-plugin/blob/main/UPGRADE.md#migrating-from-laravel-mix-to-vite)。

<a name="vite-or-mix"></a>
#### 選擇 Vite 或 Laravel Mix

在轉換到 Vite 之前，新的 Laravel 應用程式在打包資源時，會使用由 [webpack](https://webpack.js.org/) 驅動的 [Mix](https://laravel-mix.com/)。Vite 專注於在建構豐富的 JavaScript 應用程式時，提供更快、更具生產力的體驗。如果您正在開發單頁應用程式 (SPA)，包括使用 [Inertia](https://inertiajs.com) 等工具開發的應用程式，Vite 將是完美的選擇。

Vite 也適用於傳統的伺服器端渲染應用程式，其中包含少量的 JavaScript「點綴」，包括那些使用 [Livewire](https://livewire.laravel.com) 的應用程式。然而，它缺少 Laravel Mix 支援的一些功能，例如將未直接在 JavaScript 應用程式中引用的任意資源複製到建構中的能力。

<a name="migrating-back-to-mix"></a>
#### 遷移回 Mix

您是否已使用我們的 Vite 鷹架啟動了一個新的 Laravel 應用程式，但需要移回 Laravel Mix 與 webpack？沒問題。請參閱我們的 [從 Vite 遷移到 Mix 的官方指南](https://github.com/laravel/vite-plugin/blob/main/UPGRADE.md#migrating-from-vite-to-laravel-mix)。

<a name="installation"></a>
## 安裝與設定

> [!NOTE]  
> 以下說明文件討論如何手動安裝與設定 Laravel Vite plugin。然而，Laravel 的 [入門套件](/docs/{{version}}/starter-kits) 已包含所有這些鷹架，是開始使用 Laravel 與 Vite 最快的方式。

<a name="installing-node"></a>
### 安裝 Node

在執行 Vite 與 Laravel plugin 之前，您必須確保已安裝 Node.js (16+) 與 NPM：

```sh
node -v
npm -v
```

您可以透過 [Node 官方網站](https://nodejs.org/en/download/) 提供的簡單圖形化安裝程式，輕鬆安裝最新版本的 Node 與 NPM。或者，如果您正在使用 [Laravel Sail](https://laravel.com/docs/{{version}}/sail)，您可以透過 Sail 呼叫 Node 與 NPM：

```sh
./vendor/bin/sail node -v
./vendor/bin/sail npm -v
```

<a name="installing-vite-and-laravel-plugin"></a>
### 安裝 Vite 與 Laravel Plugin

在全新安裝的 Laravel 中，您會在應用程式目錄結構的根目錄中找到一個 `package.json` 檔案。預設的 `package.json` 檔案已包含您開始使用 Vite 與 Laravel plugin 所需的一切。您可以透過 NPM 安裝應用程式的前端依賴項：

```sh
npm install
```

<a name="configuring-vite"></a>
### 設定 Vite

Vite 是透過專案根目錄中的 `vite.config.js` 檔案進行設定的。您可以根據自己的需求自由自訂此檔案，也可以安裝應用程式所需的任何其他 plugin，例如 `@vitejs/plugin-vue` 或 `@vitejs/plugin-react`。

Laravel Vite plugin 要求您指定應用程式的進入點。這些可以是 JavaScript 或 CSS 檔案，並包含預處理語言，例如 TypeScript、JSX、TSX 與 Sass。

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

如果您正在建構 SPA，包括使用 Inertia 建構的應用程式，Vite 在沒有 CSS 進入點的情況下運作最佳：

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

相反地，您應該透過 JavaScript 匯入您的 CSS。通常，這會在應用程式的 `resources/js/app.js` 檔案中完成：

```js
import './bootstrap';
import '../css/app.css'; // [tl! add]
```

Laravel plugin 也支援多個進入點與進階設定選項，例如 [SSR 進入點](#ssr)。

<a name="working-with-a-secure-development-server"></a>
#### 使用安全的開發伺服器

如果您的本機開發 Web 伺服器透過 HTTPS 提供您的應用程式，您可能會遇到連接到 Vite 開發伺服器的問題。

如果您正在使用 [Laravel Herd](https://herd.laravel.com) 並已保護網站，或者您正在使用 [Laravel Valet](/docs/{{version}}/valet) 並已對您的應用程式執行 [secure 命令](/docs/{{version}}/valet#securing-sites)，Laravel Vite plugin 將自動為您偵測並使用產生的 TLS 憑證。

如果您使用與應用程式目錄名稱不符的主機來保護網站，您可以在應用程式的 `vite.config.js` 檔案中手動指定主機：

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

當使用另一個 Web 伺服器時，您應該產生一個受信任的憑證，並手動設定 Vite 以使用產生的憑證：

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

如果您無法為您的系統產生受信任的憑證，您可以安裝並設定 [`@vitejs/plugin-basic-ssl` plugin](https://github.com/vitejs/vite-plugin-basic-ssl)。當使用不受信任的憑證時，您需要在瀏覽器中接受 Vite 開發伺服器的憑證警告，方法是在執行 `npm run dev` 命令時，遵循控制台中的「Local」連結。

<a name="configuring-hmr-in-sail-on-wsl2"></a>
#### 在 WSL2 上的 Sail 中執行開發伺服器

當在 Windows Subsystem for Linux 2 (WSL2) 上的 [Laravel Sail](/docs/{{version}}/sail) 中執行 Vite 開發伺服器時，您應該在 `vite.config.js` 檔案中新增以下設定，以確保瀏覽器可以與開發伺服器通訊：

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

如果您的檔案變更在開發伺服器執行時未反映在瀏覽器中，您可能還需要設定 Vite 的 [`server.watch.usePolling` 選項](https://vitejs.dev/config/server-options.html#server-watch)。

<a name="loading-your-scripts-and-styles"></a>
### 載入您的腳本與樣式

設定好 Vite 進入點後，您現在可以在應用程式根模板的 `<head>` 中新增的 `@vite()` Blade directive 中引用它們：

```blade
<!DOCTYPE html>
<head>
    {{-- ... --}}

    @vite(['resources/css/app.css', 'resources/js/app.js'])
</head>
```

如果您透過 JavaScript 匯入 CSS，則只需包含 JavaScript 進入點：

```blade
<!DOCTYPE html>
<head>
    {{-- ... --}}

    @vite('resources/js/app.js')
</head>
```

`@vite` directive 將自動偵測 Vite 開發伺服器並注入 Vite client 以啟用 Hot Module Replacement。在建構模式下，此 directive 將載入您編譯與版本化的資源，包括任何匯入的 CSS。

如果需要，您也可以在呼叫 `@vite` directive 時指定編譯資源的建構路徑：

```blade
<!doctype html>
<head>
    {{-- Given build path is relative to public path. --}}

    @vite('resources/js/app.js', 'vendor/courier/build')
</head>
```

<a name="inline-assets"></a>
#### 行內資源

有時可能需要包含資源的原始內容，而不是連結到資源的版本化 URL。例如，當將 HTML 內容傳遞給 PDF 產生器時，您可能需要將資源內容直接包含在頁面中。您可以使用 `Vite` facade 提供的 `content` 方法輸出 Vite 資源的內容：

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

有兩種方式可以執行 Vite。您可以透過 `dev` 命令執行開發伺服器，這在本地開發時很有用。開發伺服器將自動偵測您檔案的變更，並立即反映在任何開啟的瀏覽器視窗中。

或者，執行 `build` 命令將對應用程式的資源進行版本化與打包，並準備好部署到正式環境：

```shell
# 執行 Vite 開發伺服器...
npm run dev

# 建構並版本化正式環境的資源...
npm run build
```

如果您正在 WSL2 上的 [Sail](/docs/{{version}}/sail) 中執行開發伺服器，您可能需要一些 [額外的設定](#configuring-hmr-in-sail-on-wsl2) 選項。

<a name="working-with-scripts"></a>
## 使用 JavaScript

<a name="aliases"></a>
### 別名

預設情況下，Laravel plugin 提供了一個常見的別名，可幫助您快速上手並方便地匯入應用程式的資源：

```js
{
    '@' => '/resources/js'
}
```

您可以透過在 `vite.config.js` 設定檔中新增自己的別名來覆寫 `'@'` 別名：

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

如果您想使用 [Vue](https://vuejs.org/) 框架建構前端，那麼您還需要安裝 `@vitejs/plugin-vue` plugin：

```sh
npm install --save-dev @vitejs/plugin-vue
```

然後，您可以將該 plugin 包含在您的 `vite.config.js` 設定檔中。當將 Vue plugin 與 Laravel 搭配使用時，您需要一些額外的選項：

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
                    // Vue plugin 將重新寫入在單一檔案元件中引用的資源 URL，
                    // 以指向 Laravel Web 伺服器。將此設定為 `null` 允許
                    // Laravel plugin 轉而將資源 URL 重新寫入以指向 Vite
                    // 伺服器。
                    base: null,

                    // Vue plugin 將解析絕對 URL 並將其視為磁碟上的檔案絕對路徑。
                    // 將此設定為 `false` 將保持絕對 URL 不變，以便它們可以
                    // 如預期地引用 public 目錄中的資源。
                    includeAbsolute: false,
                },
            },
        }),
    ],
});
```

> [!NOTE]  
> Laravel 的 [入門套件](/docs/{{version}}/starter-kits) 已包含正確的 Laravel、Vue 與 Vite 設定。請查看 [Laravel Breeze](/docs/{{version}}/starter-kits#breeze-and-inertia)，以最快的方式開始使用 Laravel、Vue 與 Vite。

<a name="react"></a>
### React

如果您想使用 [React](https://reactjs.org/) 框架建構前端，那麼您還需要安裝 `@vitejs/plugin-react` plugin：

```sh
npm install --save-dev @vitejs/plugin-react
```

然後，您可以將該 plugin 包含在您的 `vite.config.js` 設定檔中：

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

您需要確保任何包含 JSX 的檔案都具有 `.jsx` 或 `.tsx` 副檔名，並記住，如果需要，請更新您的進入點，如 [上方所示](#configuring-vite)。

您還需要將額外的 `@viteReactRefresh` Blade directive 與您現有的 `@vite` directive 一起包含。

```blade
@viteReactRefresh
@vite('resources/js/app.jsx')
```

`@viteReactRefresh` directive 必須在 `@vite` directive 之前呼叫。

> [!NOTE]  
> Laravel 的 [入門套件](/docs/{{version}}/starter-kits) 已包含正確的 Laravel、React 與 Vite 設定。請查看 [Laravel Breeze](/docs/{{version}}/starter-kits#breeze-and-inertia)，以最快的方式開始使用 Laravel、React 與 Vite。

<a name="inertia"></a>
### Inertia

Laravel Vite plugin 提供了一個方便的 `resolvePageComponent` 函數，可幫助您解析 Inertia 頁面元件。以下是 Vue 3 中使用此輔助函數的範例；但是，您也可以在其他框架（例如 React）中使用此函數：

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

如果您正在將 Vite 的程式碼分割功能與 Inertia 搭配使用，我們建議設定 [資源預先載入](#asset-prefetching)。

> [!NOTE]  
> Laravel 的 [入門套件](/docs/{{version}}/starter-kits) 已包含正確的 Laravel、Inertia 與 Vite 設定。請查看 [Laravel Breeze](/docs/{{version}}/starter-kits#breeze-and-inertia)，以最快的方式開始使用 Laravel、Inertia 與 Vite。

<a name="url-processing"></a>
### URL 處理

當使用 Vite 並在應用程式的 HTML、CSS 或 JS 中引用資源時，有幾個注意事項需要考慮。首先，如果您使用絕對路徑引用資源，Vite 將不會在建構中包含該資源；因此，您應該確保該資源在您的 public 目錄中可用。當使用 [專用的 CSS 進入點](#configuring-vite) 時，您應該避免使用絕對路徑，因為在開發期間，瀏覽器將嘗試從託管 CSS 的 Vite 開發伺服器載入這些路徑，而不是從您的 public 目錄載入。

當引用相對資源路徑時，您應該記住這些路徑是相對於引用它們的檔案。任何透過相對路徑引用的資源都將由 Vite 重新寫入、版本化與打包。

考慮以下專案結構：

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

以下範例演示了 Vite 將如何處理相對與絕對 URL：

```html
<!-- 此資源未由 Vite 處理，也不會包含在建構中 -->
<img src="/taylor.png">

<!-- 此資源將由 Vite 重新寫入、版本化與打包 -->
<img src="../../images/abigail.png">
```

<a name="working-with-stylesheets"></a>
## 使用樣式表

您可以在 [Vite 說明文件](https://vitejs.dev/guide/features.html#css) 中了解更多關於 Vite 的 CSS 支援。如果您正在使用 PostCSS plugin，例如 [Tailwind](https://tailwindcss.com)，您可以在專案根目錄中建立一個 `postcss.config.js` 檔案，Vite 將自動應用它：

```js
export default {
    plugins: {
        tailwindcss: {},
        autoprefixer: {},
    },
};
```

> [!NOTE]  
> Laravel 的 [入門套件](/docs/{{version}}/starter-kits) 已包含正確的 Tailwind、PostCSS 與 Vite 設定。或者，如果您想在不使用我們的入門套件的情況下使用 Tailwind 與 Laravel，請查看 [Tailwind 的 Laravel 安裝指南](https://tailwindcss.com/docs/guides/laravel)。

<a name="working-with-blade-and-routes"></a>
## 使用 Blade 與路由

<a name="blade-processing-static-assets"></a>
### 使用 Vite 處理靜態資源

當在 JavaScript 或 CSS 中引用資源時，Vite 會自動處理並版本化它們。此外，在建構基於 Blade 的應用程式時，Vite 還可以處理並版本化您僅在 Blade 模板中引用的靜態資源。

然而，為了實現這一點，您需要透過將靜態資源匯入應用程式的進入點，讓 Vite 知道您的資源。例如，如果您想處理並版本化儲存在 `resources/images` 中的所有圖片與儲存在 `resources/fonts` 中的所有字體，您應該在應用程式的 `resources/js/app.js` 進入點中新增以下內容：

```js
import.meta.glob([
  '../images/**',
  '../fonts/**',
]);
```

這些資源現在將在執行 `npm run build` 時由 Vite 處理。然後，您可以使用 `Vite::asset` 方法在 Blade 模板中引用這些資源，該方法將返回給定資源的版本化 URL：

```blade
<img src="{{ Vite::asset('resources/images/logo.png') }}">
```

<a name="blade-refreshing-on-save"></a>
### 儲存時重新整理

當您的應用程式是使用傳統的伺服器端渲染與 Blade 建構時，Vite 可以透過在您對應用程式中的視圖檔案進行變更時自動重新整理瀏覽器來改善您的開發工作流程。要開始使用，您只需將 `refresh` 選項指定為 `true`。

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

當 `refresh` 選項為 `true` 時，在以下目錄中儲存檔案將觸發瀏覽器在您執行 `npm run dev` 時執行完整頁面重新整理：

- `app/Livewire/**`
- `app/View/Components/**`
- `lang/**`
- `resources/lang/**`
- `resources/views/**`
- `routes/**`

監控 `routes/**` 目錄很有用，如果您正在利用 [Ziggy](https://github.com/tighten/ziggy) 在應用程式前端產生路由連結。

如果這些預設路徑不符合您的需求，您可以指定自己的監控路徑列表：

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

在底層，Laravel Vite plugin 使用 [`vite-plugin-full-reload`](https://github.com/ElMassimo/vite-plugin-full-reload) 套件，該套件提供了一些進階設定選項來微調此功能的行為。如果您需要這種程度的自訂，您可以提供一個 `config` 定義：

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

在 JavaScript 應用程式中，為經常引用的目錄 [建立別名](#aliases) 是很常見的。但是，您也可以透過在 `Illuminate\Support\Facades\Vite` 類別上使用 `macro` 方法來建立在 Blade 中使用的別名。通常，「macros」應該在 [服務提供者](/docs/{{version}}/providers) 的 `boot` 方法中定義：

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        Vite::macro('image', fn (string $asset) => $this->asset("resources/images/{$asset}"));
    }

一旦定義了 macro，就可以在您的模板中呼叫它。例如，我們可以使用上面定義的 `image` macro 來引用位於 `resources/images/logo.png` 的資源：

```blade
<img src="{{ Vite::image('logo.png') }}" alt="Laravel Logo">
```

<a name="asset-prefetching"></a>
## 資源預先載入

當使用 Vite 的程式碼分割功能建構 SPA 時，所需的資源會在每次頁面導航時被擷取。這種行為可能導致 UI 渲染延遲。如果這對您選擇的前端框架來說是個問題，Laravel 提供了在初始頁面載入時，預先載入應用程式的 JavaScript 與 CSS 資源的能力。

您可以透過在 [服務提供者](/docs/{{version}}/providers) 的 `boot` 方法中呼叫 `Vite::prefetch` 方法，指示 Laravel 預先載入您的資源：

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

在上面的範例中，資源將在每次頁面載入時以最多 `3` 個並行下載進行預先載入。您可以修改並行數以適應應用程式的需求，或者如果應用程式應該一次下載所有資源，則不指定並行限制：

```php
/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Vite::prefetch();
}
```

預設情況下，預先載入將在 [頁面 _load_ 事件](https://developer.mozilla.org/en-US/docs/Web/API/Window/load_event) 觸發時開始。如果您想自訂預先載入何時開始，您可以指定 Vite 將監聽的事件：

```php
/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Vite::prefetch(event: 'vite:prefetch');
}
```

根據上面的程式碼，預先載入現在將在您手動在 `window` 物件上分派 `vite:prefetch` 事件時開始。例如，您可以在頁面載入三秒後開始預先載入：

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

設定資源 URL 後，所有重新寫入的資源 URL 都將以設定的值作為前綴：

```nothing
https://cdn.example.com/build/assets/app.9dce8d17.js
```

請記住，[絕對 URL 不會由 Vite 重新寫入](#url-processing)，因此它們不會被加上前綴。

<a name="environment-variables"></a>
## 環境變數

您可以透過在應用程式的 `.env` 檔案中以 `VITE_` 作為前綴，將環境變數注入到您的 JavaScript 中：

```env
VITE_SENTRY_DSN_PUBLIC=http://example.com
```

您可以透過 `import.meta.env` 物件存取注入的環境變數：

```js
import.meta.env.VITE_SENTRY_DSN_PUBLIC
```

<a name="disabling-vite-in-tests"></a>
## 在測試中停用 Vite

Laravel 的 Vite 整合將嘗試在執行測試時解析您的資源，這要求您執行 Vite 開發伺服器或建構您的資源。

如果您希望在測試期間模擬 Vite，您可以呼叫 `withoutVite` 方法，該方法適用於任何擴展 Laravel `TestCase` 類別的測試：

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

如果您想為所有測試停用 Vite，您可以從基礎 `TestCase` 類別的 `setUp` 方法中呼叫 `withoutVite` 方法：

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

Laravel Vite plugin 讓設定 Vite 的伺服器端渲染變得輕而易舉。要開始使用，請在 `resources/js/ssr.js` 建立一個 SSR 進入點，並透過將設定選項傳遞給 Laravel plugin 來指定該進入點：

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

為了確保您不會忘記重建 SSR 進入點，我們建議擴充應用程式 `package.json` 中的「build」腳本以建立您的 SSR 建構：

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

如果您正在將 [SSR 與 Inertia](https://inertiajs.com/server-side-rendering) 搭配使用，您可以改用 `inertia:start-ssr` Artisan 命令來啟動 SSR 伺服器：

```sh
php artisan inertia:start-ssr
```

> [!NOTE]  
> Laravel 的 [入門套件](/docs/{{version}}/starter-kits) 已包含正確的 Laravel、Inertia SSR 與 Vite 設定。請查看 [Laravel Breeze](/docs/{{version}}/starter-kits#breeze-and-inertia)，以最快的方式開始使用 Laravel、Inertia SSR 與 Vite。

<a name="script-and-style-attributes"></a>
## 腳本與樣式標籤屬性

<a name="content-security-policy-csp-nonce"></a>
### 內容安全策略 (CSP) Nonce

如果您希望在腳本與樣式標籤上包含 [`nonce` 屬性](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/nonce) 作為 [內容安全策略](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP) 的一部分，您可以使用自訂 [middleware](/docs/{{version}}/middleware) 中的 `useCspNonce` 方法來產生或指定 nonce：

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

呼叫 `useCspNonce` 方法後，Laravel 將自動在所有產生的腳本與樣式標籤上包含 `nonce` 屬性。

如果您需要在其他地方指定 nonce，包括 Laravel [入門套件](/docs/{{version}}/starter-kits) 中包含的 [Ziggy `@route` directive](https://github.com/tighten/ziggy#using-routes-with-a-content-security-policy)，您可以使用 `cspNonce` 方法擷取它：

```blade
@routes(nonce: Vite::cspNonce())
```

如果您已經有一個您希望 Laravel 使用的 nonce，您可以將該 nonce 傳遞給 `useCspNonce` 方法：

```php
Vite::useCspNonce($nonce);
```

<a name="subresource-integrity-sri"></a>
### 子資源完整性 (SRI)

如果您的 Vite manifest 包含資源的 `integrity` 雜湊，Laravel 將自動在它產生的任何腳本與樣式標籤上新增 `integrity` 屬性，以強制執行 [子資源完整性](https://developer.mozilla.org/en-US/docs/Web/Security/Subresource_Integrity)。預設情況下，Vite 不會在其 manifest 中包含 `integrity` 雜湊，但您可以透過安裝 [`vite-plugin-manifest-sri`](https://www.npmjs.com/package/vite-plugin-manifest-sri) NPM plugin 來啟用它：

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

如果您需要在腳本與樣式標籤上包含額外的屬性，例如 [`data-turbo-track`](https://turbo.hotwired.dev/handbook/drive#reloading-when-assets-change) 屬性，您可以透過 `useScriptTagAttributes` 與 `useStyleTagAttributes` 方法指定它們。通常，這些方法應該從 [服務提供者](/docs/{{version}}/providers) 中呼叫：

```php
use Illuminate\Support\Facades\Vite;

Vite::useScriptTagAttributes([
    'data-turbo-track' => 'reload', // 為屬性指定一個值...
    'async' => true, // 指定一個沒有值的屬性...
    'integrity' => false, // 排除一個原本會包含的屬性...
]);

Vite::useStyleTagAttributes([
    'data-turbo-track' => 'reload',
]);
```

如果您需要有條件地新增屬性，您可以傳遞一個回呼函數，該函數將接收資源來源路徑、其 URL、其 manifest chunk 與整個 manifest：

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
> 當 Vite 開發伺服器執行時，`$chunk` 與 `$manifest` 參數將為 `null`。

<a name="advanced-customization"></a>
## 進階自訂

開箱即用，Laravel 的 Vite plugin 使用合理的慣例，應該適用於大多數應用程式；然而，有時您可能需要自訂 Vite 的行為。為了啟用額外的自訂選項，我們提供以下方法與選項，這些方法與選項可以用來取代 `@vite` Blade directive：

```blade
<!doctype html>
<head>
    {{-- ... --}}

    {{
        Vite::useHotFile(storage_path('vite.hot')) // 自訂「hot」檔案...
            ->useBuildDirectory('bundle') // 自訂建構目錄...
            ->useManifestFilename('assets.json') // 自訂 manifest 檔案名稱...
            ->withEntryPoints(['resources/js/app.js']) // 指定進入點...
            ->createAssetPathsUsing(function (string $path, ?bool $secure) { // 自訂建構資源的後端路徑產生...
                return "https://cdn.example.com/{$path}";
            })
    }}
</head>
```

在 `vite.config.js` 檔案中，您應該然後指定相同的設定：

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';

export default defineConfig({
    plugins: [
        laravel({
            hotFile: 'storage/vite.hot', // 自訂「hot」檔案...
            buildDirectory: 'bundle', // 自訂建構目錄...
            input: ['resources/js/app.js'], // 指定進入點...
        }),
    ],
    build: {
      manifest: 'assets.json', // 自訂 manifest 檔案名稱...
    },
});
```

<a name="cors"></a>
### 開發伺服器跨來源資源共用 (CORS)

如果您在從 Vite 開發伺服器擷取資源時，在瀏覽器中遇到跨來源資源共用 (CORS) 問題，您可能需要授予您的自訂來源存取開發伺服器的權限。Vite 與 Laravel plugin 結合，允許以下來源，無需任何額外設定：

- `::1`
- `127.0.0.1`
- `localhost`
- `*.test`
- `*.localhost`
- 專案 `.env` 中的 `APP_URL`

允許專案自訂來源最簡單的方法是確保應用程式的 `APP_URL` 環境變數與您在瀏覽器中造訪的來源相符。例如，如果您造訪 `https://my-app.laravel`，您應該更新您的 `.env` 以符合：

```env
APP_URL=https://my-app.laravel
```

如果您需要對來源進行更精細的控制，例如支援多個來源，您應該利用 [Vite 全面且彈性的內建 CORS 伺服器設定](https://vite.dev/config/server-options.html#server-cors)。例如，您可以在專案的 `vite.config.js` 檔案中的 `server.cors.origin` 設定選項中指定多個來源：

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

您還可以包含正規表示式模式，如果您想允許給定頂級網域的所有來源，例如 `*.laravel`，這會很有幫助：

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
                // 支援：SCHEME://DOMAIN.laravel[:PORT] [tl! add]
                /^https?:\/\/.*\.laravel(:\d+)?$/, //[tl! add]
            ], // [tl! add]
        }, // [tl! add]
    }, // [tl! add]
});
```

<a name="correcting-dev-server-urls"></a>
### 修正開發伺服器 URL

Vite 生態系統中的某些 plugin 假設以斜線開頭的 URL 將始終指向 Vite 開發伺服器。然而，由於 Laravel 整合的性質，情況並非如此。

例如，`vite-imagetools` plugin 在 Vite 提供您的資源時，會輸出如下的 URL：

```html
<img src="/@imagetools/f0b2f404b13f052c604e632f2fb60381bf61a520">
```

`vite-imagetools` plugin 預期輸出 URL 將被 Vite 攔截，然後該 plugin 可以處理所有以 `/@imagetools` 開頭的 URL。如果您正在使用預期這種行為的 plugin，您將需要手動修正 URL。您可以在 `vite.config.js` 檔案中使用 `transformOnServe` 選項來執行此操作。

在這個特定的範例中，我們將在產生的程式碼中，將開發伺服器 URL 前置到所有出現的 `/@imagetools`：

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
