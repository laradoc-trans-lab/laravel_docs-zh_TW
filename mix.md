# Laravel Mix

- [介紹](#introduction)

<a name="introduction"></a>
## 介紹

[Laravel Mix](https://github.com/laravel-mix/laravel-mix) 是由 [Laracasts](https://laracasts.com) 創作者 Jeffrey Way 開發的一個套件，它提供了一個流暢的 API，用於為您的 Laravel 應用程式定義 [webpack](https://webpack.js.org) 建置步驟，並使用多種常見的 CSS 和 JavaScript 預處理器。

換句話說，Mix 讓編譯和壓縮您的應用程式的 CSS 和 JavaScript 檔案變得輕而易舉。透過簡單的方法鏈接 (method chaining)，您可以流暢地定義您的資產管線。例如：

```js
mix.js('resources/js/app.js', 'public/js')
    .postCss('resources/css/app.css', 'public/css');
```

如果您曾經對開始使用 webpack 和資產編譯感到困惑和不知所措，您會愛上 Laravel Mix。然而，在開發應用程式時，您並非強制要求使用它；您可以自由使用任何您想要的資產管線工具，甚至完全不使用。

> [!NOTE]  
> 在新的 Laravel 安裝中，Vite 已取代 Laravel Mix。如需 Mix 文件，請造訪[Laravel Mix 官方](https://laravel-mix.com/)網站。如果您想切換到 Vite，請參閱我們的 [Vite 遷移指南](https://github.com/laravel/vite-plugin/blob/main/UPGRADE.md#migrating-from-laravel-mix-to-vite)。