# Laravel Mix

- [簡介](#introduction)

<a name="introduction"></a>
## 簡介

> [!WARNING]
> Laravel Mix 是一個不再積極維護的舊版套件。[Vite](/docs/{{version}}/vite) 可作為現代替代方案使用。

[Laravel Mix](https://github.com/laravel-mix/laravel-mix) 是由 [Laracasts](https://laracasts.com) 創辦人 Jeffrey Way 開發的套件，它提供了一個流暢的 API，讓您可以使用多種常見的 CSS 和 JavaScript 預處理器，為您的 Laravel 應用程式定義 [webpack](https://webpack.js.org) 建置步驟。

換句話說，Mix 讓編譯和壓縮應用程式的 CSS 和 JavaScript 檔案變得輕而易舉。透過簡單的方法鏈接，您可以流暢地定義您的資產管線。例如：

```js
mix.js('resources/js/app.js', 'public/js')
    .postCss('resources/css/app.css', 'public/css');
```

如果您曾經對開始使用 webpack 和資產編譯感到困惑和不知所措，那麼您一定會喜歡 Laravel Mix。然而，在開發應用程式時，您並非必須使用它；您可以自由使用任何您喜歡的資產管線工具，甚至完全不使用。

> [!NOTE]
> 在新的 Laravel 安裝中，Vite 已取代 Laravel Mix。有關 Mix 的文件，請造訪 [Laravel Mix 官方網站](https://laravel-mix.com/)。如果您想切換到 Vite，請參閱我們的 [Vite 遷移指南](https://github.com/laravel/vite-plugin/blob/main/UPGRADE.md#migrating-from-laravel-mix-to-vite)。

