# 前端

- [介紹](#introduction)
- [使用 PHP](#using-php)
    - [PHP 與 Blade](#php-and-blade)
    - [Livewire](#livewire)
    - [入門套件](#php-starter-kits)
- [使用 React 或 Vue](#using-react-or-vue)
    - [Inertia](#inertia)
    - [入門套件](#inertia-starter-kits)
- [打包資源](#bundling-assets)

<a name="introduction"></a>
## 介紹

Laravel 是一個後端框架，提供了建構現代化網路應用程式所需的所有功能，例如 [路由](/docs/{{version}}/routing)、[驗證](/docs/{{version}}/validation)、[快取](/docs/{{version}}/cache)、[佇列](/docs/{{version}}/queues)、[檔案儲存](/docs/{{version}}/filesystem) 等等。然而，我們認為為開發人員提供一個美觀的全端體驗非常重要，包括建構應用程式前端的強大方法。

使用 Laravel 建構應用程式時，有兩種主要的前端開發方式，您選擇哪種方式取決於您是想利用 PHP 還是使用 Vue 和 React 等 JavaScript 框架來建構前端。我們將在下面討論這兩種選項，以便您能針對應用程式的前端開發做出明智的決定。

<a name="using-php"></a>
## 使用 PHP

<a name="php-and-blade"></a>
### PHP 與 Blade

過去，大多數 PHP 應用程式使用簡單的 HTML 模板，其中穿插著 PHP `echo` 語句，這些語句用於渲染在請求期間從資料庫中檢索到的資料，從而將 HTML 渲染到瀏覽器：

```blade
<div>
    <?php foreach ($users as $user): ?>
        Hello, <?php echo $user->name; ?> <br />
    <?php endforeach; ?>
</div>
```

在 Laravel 中，這種渲染 HTML 的方法仍然可以透過 [視圖](/docs/{{version}}/views) 和 [Blade](/docs/{{version}}/blade) 來實現。Blade 是一種極其輕量級的模板語言，它提供了方便、簡潔的語法來顯示資料、迭代資料等等：

```blade
<div>
    @foreach ($users as $user)
        Hello, {{ $user->name }} <br />
    @endforeach
</div>
```

以這種方式建構應用程式時，表單提交和其他頁面互動通常會從伺服器接收一個全新的 HTML 文件，並且整個頁面會由瀏覽器重新渲染。即使在今天，許多應用程式可能仍然非常適合使用簡單的 Blade 模板來建構其前端。

<a name="growing-expectations"></a>
#### 不斷增長的期望

然而，隨著使用者對網路應用程式的期望日益成熟，許多開發人員發現需要建構更具動態性、互動性更流暢的前端。有鑑於此，一些開發人員選擇開始使用 Vue 和 React 等 JavaScript 框架來建構應用程式的前端。

另一些開發人員則偏好堅持使用他們熟悉的後端語言，並開發出能夠建構現代網路應用程式 UI 的解決方案，同時仍主要利用他們選擇的後端語言。例如，在 [Rails](https://rubyonrails.org/) 生態系統中，這促使了 [Turbo](https://turbo.hotwired.dev/)、[Hotwire](https://hotwired.dev/) 和 [Stimulus](https://stimulus.hotwired.dev/) 等函式庫的誕生。

在 Laravel 生態系統中，主要透過 PHP 建構現代化、動態前端的需求，促成了 [Laravel Livewire](https://livewire.laravel.com) 和 [Alpine.js](https://alpinejs.dev/) 的誕生。

<a name="livewire"></a>
### Livewire

[Laravel Livewire](https://livewire.laravel.com) 是一個用於建構由 Laravel 驅動的前端的框架，它能讓前端感覺像使用 Vue 和 React 等現代 JavaScript 框架建構的前端一樣動態、現代且生動。

使用 Livewire 時，您將建立 Livewire「元件」，這些元件會渲染您 UI 的離散部分，並公開可從應用程式前端調用和互動的方法和資料。例如，一個簡單的「計數器」元件可能如下所示：

```php
<?php

namespace App\Http\Livewire;

use Livewire\Component;

class Counter extends Component
{
    public $count = 0;

    public function increment()
    {
        $this->count++;
    }

    public function render()
    {
        return view('livewire.counter');
    }
}
```

並且，計數器的相應模板將會這樣編寫：

```blade
<div>
    <button wire:click="increment">+</button>
    <h1>{{ $count }}</h1>
</div>
```

如您所見，Livewire 讓您可以編寫新的 HTML 屬性，例如 `wire:click`，這些屬性將您的 Laravel 應用程式的前端和後端連接起來。此外，您可以使用簡單的 Blade 表達式來渲染元件的當前狀態。

對許多人來說，Livewire 徹底改變了 Laravel 的前端開發，讓他們在建構現代化、動態的網路應用程式時，仍能留在 Laravel 的舒適圈內。通常，使用 Livewire 的開發人員也會利用 [Alpine.js](https://alpinejs.dev/)，僅在需要時才將 JavaScript「點綴」到他們的前端，例如為了渲染一個對話視窗。

如果您是 Laravel 的新手，我們建議您先熟悉 [視圖](/docs/{{version}}/views) 和 [Blade](/docs/{{version}}/blade) 的基本用法。然後，查閱官方的 [Laravel Livewire 文件](https://livewire.laravel.com/docs)，了解如何透過互動式 Livewire 元件將您的應用程式提升到新的水平。

<a name="php-starter-kits"></a>
### 入門套件

如果您想使用 PHP 和 Livewire 建構前端，您可以利用我們的 [Livewire 入門套件](/docs/{{version}}/starter-kits) 來加速您應用程式的開發。

<a name="using-react-or-vue"></a>
## 使用 React 或 Vue

儘管可以使用 Laravel 和 Livewire 建構現代前端，但許多開發人員仍然偏好利用 React 或 Vue 等 JavaScript 框架的強大功能。這讓開發人員可以利用透過 NPM 提供的豐富 JavaScript 套件和工具生態系統。

然而，如果沒有額外的工具，將 Laravel 與 React 或 Vue 配對將使我們需要解決各種複雜的問題，例如客戶端路由、資料水合 (data hydration) 和身份驗證。客戶端路由通常透過使用像 [Next](https://nextjs.org/) 和 [Nuxt](https://nuxt.com/) 這樣有主見的 React / Vue 框架來簡化；然而，當將像 Laravel 這樣的後端框架與這些前端框架配對時，資料水合和身份驗證仍然是複雜且繁瑣的問題。

此外，開發人員還需要維護兩個獨立的程式碼儲存庫，通常需要協調兩個儲存庫的維護、發布和部署。儘管這些問題並非無法克服，但我們不認為這是一種高效或愉快的應用程式開發方式。

<a name="inertia"></a>
### Inertia

幸運的是，Laravel 提供了兩全其美的方案。[Inertia](https://inertiajs.com) 彌合了您的 Laravel 應用程式與現代 React 或 Vue 前端之間的鴻溝，讓您可以使用 React 或 Vue 建構完整、現代的前端，同時利用 Laravel 路由和控制器進行路由、資料水合和身份驗證 — 所有這些都在單一程式碼儲存庫中完成。透過這種方法，您可以充分享受 Laravel 和 React / Vue 的全部功能，而不會削弱任何工具的能力。

在您的 Laravel 應用程式中安裝 Inertia 後，您將像往常一樣編寫路由和控制器。然而，您將從控制器返回一個 Inertia 頁面，而不是返回一個 Blade 模板：

```php
<?php

namespace App\Http\Controllers;

use App\Models\User;
use Inertia\Inertia;
use Inertia\Response;

class UserController extends Controller
{
    /**
     * Show the profile for a given user.
     */
    public function show(string $id): Response
    {
        return Inertia::render('users/show', [
            'user' => User::findOrFail($id)
        ]);
    }
}
```

Inertia 頁面對應於一個 React 或 Vue 元件，通常儲存在您應用程式的 `resources/js/pages` 目錄中。透過 `Inertia::render` 方法提供給頁面的資料將用於水合頁面元件的「props」：

```jsx
import Layout from '@/layouts/authenticated';
import { Head } from '@inertiajs/react';

export default function Show({ user }) {
    return (
        <Layout>
            <Head title="Welcome" />
            <h1>Welcome</h1>
            <p>Hello {user.name}, welcome to Inertia.</p>
        </Layout>
    )
}
```

如您所見，Inertia 讓您在建構前端時可以充分利用 React 或 Vue 的強大功能，同時在由 Laravel 驅動的後端和由 JavaScript 驅動的前端之間提供輕量級的橋樑。

#### 伺服器端渲染

如果您擔心因為應用程式需要伺服器端渲染而不敢深入 Inertia，請不用擔心。Inertia 提供了 [伺服器端渲染支援](https://inertiajs.com/server-side-rendering)。而且，當透過 [Laravel Cloud](https://cloud.laravel.com) 或 [Laravel Forge](https://forge.laravel.com) 部署您的應用程式時，確保 Inertia 的伺服器端渲染程序始終運行輕而易舉。

<a name="inertia-starter-kits"></a>
### 入門套件

如果您想使用 Inertia 和 Vue / React 建構前端，您可以利用我們的 [React 或 Vue 應用程式入門套件](/docs/{{version}}/starter-kits) 來加速您應用程式的開發。這些入門套件都使用 Inertia、Vue / React、[Tailwind](https://tailwindcss.com) 和 [Vite](https://vitejs.dev) 來搭建您應用程式的後端和前端身份驗證流程，以便您可以開始建構您的下一個偉大想法。

<a name="bundling-assets"></a>
## 打包資源

無論您選擇使用 Blade 和 Livewire 還是 Vue / React 和 Inertia 來開發前端，您都可能需要將應用程式的 CSS 打包成可供生產使用的資源。當然，如果您選擇使用 Vue 或 React 建構應用程式的前端，您還需要將元件打包成瀏覽器可用的 JavaScript 資源。

預設情況下，Laravel 利用 [Vite](https://vitejs.dev) 來打包您的資源。Vite 提供了閃電般的建構時間和在本地開發期間近乎即時的熱模組替換 (HMR)。在所有新的 Laravel 應用程式中，包括那些使用我們 [入門套件](/docs/{{version}}/starter-kits) 的應用程式，您都會找到一個 `vite.config.js` 檔案，該檔案載入了我們輕量級的 Laravel Vite 外掛，使 Vite 在 Laravel 應用程式中變得易於使用。

開始使用 Laravel 和 Vite 的最快方法是使用 [我們的應用程式入門套件](/docs/{{version}}/starter-kits) 開始應用程式的開發，這些套件透過提供前端和後端身份驗證腳手架來加速您的應用程式。

> [!NOTE]
> 有關在 Laravel 中使用 Vite 的更詳細文件，請參閱我們 [關於打包和編譯資源的專門文件](/docs/{{version}}/vite)。

