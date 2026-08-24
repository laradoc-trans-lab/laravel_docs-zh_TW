# Laravel Head

- [簡介](#introduction)
- [安裝](#installation)
- [快速開始](#quickstart)
- [解析優先權](#resolution-precedence)
- [定義中繼資料](#defining-metadata)
    - [預設值](#defaults)
    - [路由中繼資料](#route-metadata)
    - [執行期中繼資料](#runtime-metadata)
    - [錯誤頁面](#error-pages)
- [Open Graph](#open-graph)
    - [X / Twitter 卡片](#twitter-cards)
- [主題顏色](#theme-colors)
- [應用程式中繼資料與圖示](#app-metadata-and-icons)
- [漸進式 Web 應用程式 (PWA)](#progressive-web-apps)
- [效能與探索](#performance-and-discovery)
- [自訂標籤](#custom-tags)
- [結構化資料 (Schemas)](#schemas)
    - [麵包屑導覽](#breadcrumbs)
    - [常見問題 (FAQs)](#faqs)
    - [自訂結構化資料](#custom-schemas)
- [渲染](#rendering)
    - [Blade](#blade)
    - [Livewire](#livewire)
    - [Inertia](#inertia)

<a name="introduction"></a>
## 簡介

[Laravel Head](https://github.com/laravel/head) 提供了流暢的 API 來管理應用程式的文件 `<head>` 元素，包含標題與 meta 標籤、Open Graph 中繼資料、標準網址 (canonical URLs)、robots 指令、效能提示以及結構化資料。它可與 Blade、Livewire 和 Inertia 搭配運作。


<a name="installation"></a>
## 安裝

你可以使用 Composer 套件管理器安裝 Laravel Head：

```shell
composer require laravel/head
```


<a name="quickstart"></a>
## 快速開始

在服務提供者中註冊全站預設值：

```php
use Laravel\Head\Facades\Head;
use Laravel\Head\HeadBuilder;

Head::defaults(fn (HeadBuilder $head) => $head
    ->title('Laravel', suffix: ' - Laravel')
    ->description('Build something great.'));
```

在執行期設定特定頁面的中繼資料：

```php
Head::title($post->title)
    ->description($post->description);
```

在佈局中渲染解析後的標籤：

```blade
<head>
    @head
</head>
```


<a name="resolution-precedence"></a>
## 解析優先權

頁面中繼資料是由五個層級解析而成，依優先權由低到高排列如下：

1. 頁面預設值
2. 路由群組中繼資料
3. 路由中繼資料
4. 執行期中繼資料
5. 錯誤中繼資料

較高層級會逐欄位替換較低層級的內容。例如，執行期標題會替換路由標題，但不會替換路由描述。接下來的章節將說明如何在各層級設定中繼資料。有關在 Blade、Livewire 和 Inertia 中渲染解析後中繼資料的資訊，請參閱[渲染](#rendering)。

<a name="defining-metadata"></a>
## 定義中繼資料

Laravel Head 允許你透過全站預設值、路由中繼資料、執行期呼叫以及錯誤頁面定義來設定中繼資料。


<a name="defaults"></a>
### 預設值

在服務提供者中註冊頁面預設值：

```php
use Laravel\Head\Enums\OgType;
use Laravel\Head\Facades\Head;
use Laravel\Head\HeadBuilder;

Head::defaults(function (HeadBuilder $head) {
    $head
        ->title('Laravel', suffix: ' - Laravel')
        ->description('Build something great.')
        ->canonical()
        ->og(siteName: 'Laravel', type: OgType::Website)
        ->searchableByRobots()
        ->preconnect('https://fonts.example.com');
});
```

預設值是優先級最低的頁面中繼資料層。如果沒有路由、執行期或錯誤中繼資料設定標題，`Laravel` 會原樣渲染。當較高層級設定頁面標題時，會套用繼承的後綴，因此 `Head::title('About')` 會渲染為 `About - Laravel`。對於需要忽略繼承前綴或後綴的標題，請傳入 `exact: true`。

呼叫 `Head::canonical()` 會使用當前請求的 URL 來渲染標準網址 (Canonical URL)。若要設定明確的 URL，請傳入字串，例如 `Head::canonical('/about')`。標準網址預設會正規化為 `https`；傳入 `forceHttps: false` 則可保留請求通訊協定 scheme。

Robots 指令可以傳入原始字串、`RobotsRule` 列舉項目，或是混合兩種類型的列表。列表會被渲染為以逗號分隔的指令，因此 `Head::robots([RobotsRule::NoIndex, RobotsRule::NoFollow])` 會渲染為 `noindex, nofollow`。

為方便起見，`searchableByRobots` 方法會渲染為 `all`，而 `hiddenFromRobots` 方法則會渲染為 `none`。


<a name="route-metadata"></a>
### 路由中繼資料

你可以直接在路由上定義中繼資料，這對於預先知道中繼資料的半靜態頁面特別有用。


<a name="routes-and-groups"></a>
#### 路由與群組

```php
Route::view('/contact', 'contact')
    ->name('contact')
    ->withHead(
        title: 'Contact Us',
        description: 'Get in touch.',
    );
```

共用的路由中繼資料可以套用在鏈條中任何位置的群組上：

```php
Route::withHead(robots: 'noindex, nofollow')
    ->prefix('admin')
    ->name('admin.')
    ->group(function () {
        Route::get('/dashboard', DashboardController::class)
            ->name('dashboard')
            ->withHead(title: 'Dashboard');
    });
```

你也可以為 resource 與 singleton 路由定義中繼資料：

```php
Route::resource('posts', PostController::class)->withHead(
    robots: 'index, follow',
);

Route::singleton('profile', ProfileController::class)->withHead(
    title: 'Your Profile',
);
```

`withHead` 方法會透過 Laravel 原生的路由中繼資料 API 來儲存純陣列。這等同於呼叫 `metadata` 方法並將屬性巢狀放置於 `head` 鍵之下，因此該中繼資料仍相容於快取的路由。

具名引數刻意限制在 Laravel Head 內建的路由屬性中，讓編輯器與靜態分析工具能捕捉到拼寫錯誤的名稱。自訂標籤建構器所註冊的路由屬性可透過 `extensions` 傳入：

```php
Route::get('/article', ArticleController::class)->withHead(
    title: 'Article',
    extensions: ['readingTime' => 4],
);
```


<a name="supported-properties"></a>
#### 支援的屬性

支援的路由屬性對應到與流暢建構器方法相同的名稱：

| 類別 | 屬性 |
| --- | --- |
| 文件 | `title`, `description`, `canonical`, `robots` |
| 應用程式中繼資料 | `themeColor`, `applicationName`, `colorScheme`, `referrer`, `viewport`, `appleWebAppTitle`, `webAppCapable`, `appleWebAppStatusBarStyle` |
| 社群 | `og`, `ogImage`, `ogVideo`, `ogAudio`, `twitter`, `twitterImage` |
| 效能 | `preload`, `prefetch`, `preconnect`, `dnsPrefetch` |
| 探索 | `alternates`, `feed`, `icon`, `favicon`, `appleTouchIcon`, `appleTouchStartupImage`, `maskIcon`, `manifest` |
| 結構化資料 | `schema` |
| 自訂標籤 | `meta`, `link` |

巢狀選項名稱採用與流暢 API 相同的 `camelCase` 命名，例如 `forceHttps`、`siteName` 和 `secureUrl`。

可重複的屬性，例如 `ogImage`、`preload`、`feed`、`schema`、`icon` 和 `appleTouchStartupImage`，皆可接受單一值或列表。


<a name="runtime-metadata"></a>
### 執行期中繼資料

當某些值在請求到達前無法得知（例如正在瀏覽的文章標題），你可以在執行期進行設定：

```php
use Laravel\Head\Facades\Head;

public function __invoke(Post $post): Response
{
    Head::title($post->title);

    // ...
}
```

透過 `Head` Facade 進行的執行期呼叫會覆蓋依賴請求資料的路由中繼資料。控制器與 Action 是進行這些呼叫最常見的地方：

```php
use App\Models\Post;
use Laravel\Head\Facades\Head;

public function show(Post $post)
{
    Head::title($post->title)
        ->description($post->description);

    return view('posts.show', ['post' => $post]);
}
```

多個執行期呼叫會依照其執行順序進行合併。對於單一值的欄位（如標題、描述、標準網址和 robots 指令），較晚的呼叫具有優先權。可重複的欄位會保留多個項目，但再次新增相同的鍵則會更新先前的項目。以 `ogImage` 方法為例，URL 就是其識別鍵：

```php
Head::ogImage('/images/cover.jpg', alt: 'Draft cover')
    ->ogImage('/images/gallery.jpg', alt: 'Gallery image')
    ->ogImage('/images/cover.jpg', alt: 'Final cover', width: 1200, height: 630);
```

```html
<meta property="og:image" content="/images/cover.jpg">
<meta property="og:image:width" content="1200">
<meta property="og:image:height" content="630">
<meta property="og:image:alt" content="Final cover">
<meta property="og:image" content="/images/gallery.jpg">
<meta property="og:image:alt" content="Gallery image">
```

從預設值繼承的 Open Graph 媒體會作為備用方案。當路由、執行期或錯誤中繼資料定義了相同類型的專屬媒體時，預設媒體會被替換而非合併，因此頁面的 `og:image` 會優先於全站預設圖片。

你可以使用 `when` 和 `unless` 方法流暢地定義條件式中繼資料：

```php
Head::title($post->title)
    ->when($post->isDraft(), fn ($head) => $head->hiddenFromRobots());
```


<a name="error-pages"></a>
### 錯誤頁面

通常，你應該在應用程式 `AppServiceProvider` 類別的 `boot` 方法中註冊錯誤中繼資料：

```php
use Laravel\Head\ErrorPages;
use Laravel\Head\Facades\Head;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Head::errors(function (ErrorPages $errors) {
        $errors->defaults(robots: 'noindex, follow');

        $errors->status(
            404,
            title: 'Page Not Found',
            description: 'The page you are looking for could not be found.',
        );
    });
}
```

`defaults` 和 `status` 方法也接受與 `Head::defaults()` 相同的流暢建構器回呼：

```php
use Laravel\Head\ErrorPages;
use Laravel\Head\Facades\Head;
use Laravel\Head\HeadBuilder;

Head::errors(function (ErrorPages $errors) {
    $errors->status(404, fn (HeadBuilder $head) => $head
        ->title('Page Not Found')
        ->description('The page you are looking for could not be found.'));
});
```

當針對已註冊的錯誤狀態碼渲染回應時，該中繼資料將優先於所有其他層級。

Laravel 在渲染錯誤視圖或執行回應階段掛鉤（如 Inertia 的 `handleExceptionsUsing()` 方法）時，會自動偵測回應狀態碼。如果你在 `$exceptions->render()` 回呼內渲染錯誤回應，請在渲染前呼叫 `Head::status(404)`，以便套用錯誤中繼資料。

<a name="open-graph"></a>
## Open Graph

您可以使用 `og` 方法設定 Open Graph 屬性。可重複的多媒體資源可使用頂層方法新增，這些方法直接接受具名引數：

```php
use Laravel\Head\Enums\ImageType;
use Laravel\Head\Enums\OgType;

Head::og(type: OgType::Article, title: $post->title)
    ->ogImage($post->hero_image_url)
    ->ogImage(
        $post->gallery_image_url,
        alt: $post->gallery_image_alt,
        width: 1200,
        height: 630,
        type: ImageType::Jpeg,
    );
```

在 Open Graph 規範支援的情況下，`ogImage`、`ogVideo` 和 `ogAudio` 方法接受 URL 作為其第一個引數，並可帶入選用的具名引數，例如 `alt`、`width`、`height`、`type` 及 `secureUrl`。

在任何接受圖片 `type` 的 API 位置，您都可以傳入 `ImageType` 列舉作為圖片 MIME 類型，例如 `ImageType::Svg`、`ImageType::Png`、`ImageType::Jpeg` 和 `ImageType::Webp`。

> [!NOTE]
> 文件的 `title` 和 `description` 會自動填補缺失的 `og:title` 和 `og:description` 數值。

若只需要單一 Open Graph 圖片且沒有其他屬性，您可以將 `image` 具名引數傳入 `og` 方法：

```php
Head::og(
    type: OgType::Website,
    title: $page->title,
    description: $page->description,
    image: $page->og_image_url,
);
```

`og(image: ...)` 與 `ogImage(...)` 呼叫會寫入相同的底層圖片清單，因此您可以在呼叫時使用表達力更佳的方式。您可以使用 [`meta`](#custom-tags) 方法來處理自訂的 Open Graph 擴充屬性，例如商品或文章屬性。


<a name="twitter-cards"></a>
### X / Twitter 卡片

若要從 Open Graph 所使用的相同標題、說明與圖片來渲染 X / Twitter 卡片，請在預設值中註冊 `twitter()`：

```php
use Laravel\Head\Enums\TwitterCard;
use Laravel\Head\Facades\Head;
use Laravel\Head\HeadBuilder;

Head::defaults(fn (HeadBuilder $head) => $head->twitter(
    card: TwitterCard::SummaryWithLargeImage,
));
```

接著設定頁面層級的中繼資料：

```php
Head::title('Introducing Laravel Head')
    ->description('A fluent API for Laravel document head metadata.')
    ->ogImage('https://example.com/social.jpg', alt: 'Introducing Laravel Head');
```

這將渲染對應的 Twitter 標籤：

```html
<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:title" content="Introducing Laravel Head">
<meta name="twitter:description" content="A fluent API for Laravel document head metadata.">
<meta name="twitter:image" content="https://example.com/social.jpg">
<meta name="twitter:image:alt" content="Introducing Laravel Head">
```

您可以使用明確的 Twitter 數值自訂個別頁面：

```php
Head::twitter(title: $post->social_title)
    ->twitterImage($post->social_image_url, alt: $post->title);
```

路由中繼資料接受 `twitter` 和 `twitterImage`。


<a name="theme-colors"></a>
## 主題顏色

您可以在全域、個別路由或執行期設定主題顏色：

```php
Head::themeColor('#0f172a');
```

這會渲染 `<meta name="theme-color">` 標籤。針對特定媒體條件的主題顏色，您可以使用 `Media` 列舉：

```php
use Laravel\Head\Enums\Media;

Head::themeColor('#ffffff', media: Media::Light)
    ->themeColor('#111827', media: Media::Dark);
```

`Media` 列舉還包含 `Portrait` 與 `Landscape`。`media` 引數也接受自訂的媒體查詢字串。

路由中繼資料透過相同的 `camelCase` 鍵支援單一主題顏色：

```php
Route::view('/dashboard', 'dashboard')->withHead(
    themeColor: '#0f172a',
);
```


<a name="app-metadata-and-icons"></a>
## 應用程式中繼資料與圖示

Laravel Head 包含用於通用瀏覽器與應用程式中繼資料的方法：

```php
use Laravel\Head\Enums\ImageType;
use Laravel\Head\Enums\Media;

Head::applicationName('Laravel')
    ->colorScheme('light dark')
    ->referrer('strict-origin-when-cross-origin')
    ->viewport('width=device-width, initial-scale=1')
    ->appleWebAppTitle('Laravel')
    ->webAppCapable()
    ->appleWebAppStatusBarStyle('black')
    ->favicon('/favicon.svg', type: ImageType::Svg)
    ->icon('/favicon-32x32.png', type: ImageType::Png, sizes: '32x32')
    ->appleTouchIcon('/apple-touch-icon.png', sizes: '180x180')
    ->appleTouchStartupImage('/launch.png', media: Media::Portrait)
    ->maskIcon('/safari-pinned-tab.svg', color: '#111827')
    ->manifest('/site.webmanifest');
```

`favicon` 方法是 `icon` 方法的別名，並接受相同的 `type`、`sizes` 和 `media` 引數。

路由中繼資料使用相同的名稱：

```php
use Laravel\Head\Enums\ImageType;
use Laravel\Head\Enums\Media;

Route::view('/dashboard', 'dashboard')->withHead(
    applicationName: 'Laravel',
    colorScheme: 'light dark',
    appleWebAppTitle: 'Laravel',
    webAppCapable: true,
    appleWebAppStatusBarStyle: 'black',
    favicon: [
        ['href' => '/favicon.svg', 'type' => ImageType::Svg],
        ['href' => '/favicon-32x32.png', 'type' => ImageType::Png, 'sizes' => '32x32'],
    ],
    appleTouchIcon: ['href' => '/apple-touch-icon.png', 'sizes' => '180x180'],
    appleTouchStartupImage: ['href' => '/launch.png', 'media' => Media::Portrait],
    manifest: '/site.webmanifest',
);
```


<a name="progressive-web-apps"></a>
## 漸進式 Web 應用程式 (PWA)

`pwa` 方法可用於設定可安裝 Web 應用程式所需的常見文件 `<head>` 標籤：

```php
Head::pwa(
    name: 'Laravel',
    manifest: '/site.webmanifest',
    themeColor: '#0f172a',
    appleTouchIcon: '/apple-touch-icon.png',
    appleWebAppStatusBarStyle: 'black',
);
```

這會渲染應用程式名稱、Web 應用程式 manifest 連結以及 iOS 獨立執行中繼資料。若有提供，主題顏色、Apple 狀態列樣式與 Apple touch 圖示也會一併渲染。建立 Web 應用程式 manifest 與註冊 Service Worker 仍由您的應用程式自行負責。

您可以在預設值或執行期中繼資料中使用 `pwa` 方法。路由中繼資料則支援上述的各個個別屬性。


<a name="performance-and-discovery"></a>
## 效能與探索

Laravel Head 可以渲染效能提示、分頁連結、語系替代網址 (Locale Alternates) 以及 Feed 探索：

```php
Head::preload(asset('fonts/inter.woff2'), as: 'font', crossorigin: true)
    ->prefetch(asset('images/next.webp'))
    ->preconnect('https://cdn.example.com')
    ->dnsPrefetch('https://analytics.example.com')
    ->paginate($posts)
    ->alternates([
        'en' => 'https://example.com/en/about',
        'fr' => 'https://example.com/fr/about',
        'x-default' => 'https://example.com/about',
    ])
    ->feed('/feed', title: 'Laravel RSS')
    ->feed('/feed.atom', type: 'atom', title: 'Laravel Atom');
```

對於本機靜態資源，`preloadAsset()` 與 `prefetchAsset()` 會透過 `asset()` 輔助函式解析 URL，並依據副檔名自動偵測 `as` 屬性。字型預先載入會自動加入 `crossorigin`，這即使是同源字型也是 preload 規格所要求的：

```php
Head::preloadAsset('fonts/inter.woff2')
    ->prefetchAsset('images/next.webp');
```

```html
<link rel="preload" href="https://example.com/fonts/inter.woff2" as="font" crossorigin>
<link rel="prefetch" href="https://example.com/images/next.webp" as="image">
```

您可以明確傳入 `as` 來覆寫自動偵測。當無法從副檔名偵測出 `as` 屬性時，`preloadAsset` 方法會擲出例外，因為瀏覽器會忽略沒有此屬性的 preload；而 `prefetchAsset` 方法則只會直接省略該屬性。

<a name="custom-tags"></a>
## 自訂標籤

對於沒有專屬方法的標籤，可以使用 `meta()` 和 `link()`：

```php
Head::meta('format-detection', 'telephone=no')
    ->meta('article:author', $post->author->name)
    ->link('search', '/opensearch.xml', [
        'type' => 'application/opensearchdescription+xml',
        'title' => 'Laravel Search',
    ])
    ->link('me', 'https://social.example.com/@laravel');
```

當瀏覽器僅應在符合條件時套用該標籤時，您可以在 meta 標籤上加入媒體查詢：

```php
use Laravel\Head\Enums\Media;

Head::meta('theme-color', '#ffffff', media: Media::Light)
    ->meta('theme-color', '#111827', media: Media::Dark);
```

`meta` 方法會針對一般的 meta 標籤使用 `name` 屬性。對於通常使用 `property` 屬性的鍵值，例如 Open Graph (`og:`) 或文章中繼資料 (`article:`)，該方法會自動切換：

```php
Head::meta('description', 'About Laravel')
    ->meta('og:title', 'About Laravel');
```

```html
<meta name="description" content="About Laravel">
<meta property="og:title" content="About Laravel">
```

您可以傳入 `property: true` 或 `property: false` 來明確選擇任一屬性。


<a name="schemas"></a>
## 結構化資料 (Schemas)

內建的結構化資料建構器涵蓋了常見的 JSON-LD 類型：

```php
use Laravel\Head\Enums\OfferAvailability;
use Laravel\Head\Facades\Schema;

Head::schema(
    Schema::product()
        ->name($product->name)
        ->offers(
            Schema::offer()
                ->price($product->price)
                ->currency('USD')
                ->availability(OfferAvailability::InStock)
        )
);
```

內建的工廠方法包括 `article`、`blogPosting`、`product`、`offer`、`brand`、`breadcrumbs`、`faq`、`organization`、`person`、`webPage` 和 `webSite`。未知的工廠方法會建立通用的結構化資料物件，因此您仍可表達自訂的 schema.org 類型。

當 JSON-LD 結構化資料無效時，Laravel Head 會在非正式環境中拋出例外，並在正式環境中記錄警告。


<a name="breadcrumbs"></a>
### 麵包屑導覽

麵包屑項目可以逐一新增或批次新增。順序會依照項目新增的先後自動指定位置：

```php
Head::schema(
    Schema::breadcrumbs()->items([
        'Home' => route('home'),
        'Shop' => route('shop.index'),
        'Shoes' => route('shop.category', 'shoes'),
    ])
);
```

您可以使用 `item` 方法來附加單一麵包屑項目：

```php
Schema::breadcrumbs()
    ->item('Home', route('home'))
    ->item('Shop', route('shop.index'));
```


<a name="faqs"></a>
### 常見問題 (FAQs)

FAQ 項目遵循相同的模式。您可以使用 `question` 方法逐一新增，或使用 `questions` 方法批次新增：

```php
Head::schema(
    Schema::faq()->questions([
        'What is Laravel Head?' => 'A fluent API for managing the document head.',
        'Is it free?' => 'Yes, it is open source.',
    ])
);
```


<a name="custom-schemas"></a>
### 自訂結構化資料

您可以明確註冊自訂結構化資料類型：

```php
use DateTimeInterface;
use Laravel\Head\Facades\Schema;
use Laravel\Head\Schema\SchemaObject;
use Laravel\Head\SchemaType;

#[SchemaType('JobPosting')]
class JobPosting extends SchemaObject
{
    public function title(string $title): static
    {
        return $this->set('title', $title);
    }

    public function datePosted(DateTimeInterface|string $date): static
    {
        return $this->date('datePosted', $date);
    }
}

Schema::register(JobPosting::class);

Head::schema(
    Schema::jobPosting()
        ->title('Senior Laravel Developer')
        ->datePosted(now())
);
```

<a name="rendering"></a>
## 渲染

Laravel Head 會將頁面中繼資料解析為目前回應所需的標籤。這些標籤的渲染方式取決於你的應用程式技術棧。

HTML 渲染器支援 `@head` 指令以及 Laravel Head 透過 `head` prop 與 Inertia 共享的已渲染元素。陣列渲染器則支援 `Head::toArray()`，供需要將解析後中繼資料作為結構化資料使用的應用程式使用。


<a name="blade"></a>
### Blade

使用 `@head` 指令在佈局的 `<head>` 中渲染累積的標籤：

```blade
<head>
    <meta charset="utf-8">
    @head
</head>
```

`@head` 指令是同步渲染的，因此你應該在渲染佈局之前定義好頁面中繼資料。


<a name="livewire"></a>
### Livewire

Livewire 應用程式在文件佈局中使用相同的 `@head` 指令：

```blade
<head>
    @head
</head>

<body>
    {{ $slot }}

    @livewireScripts
</body>
```

不需要針對 Livewire 進行特殊設定。Laravel Head 的中繼資料是按請求解析的，且解析器的作用域為請求級別。因此，每次 `wire:navigate` 造訪都會取得一份全新的文件，其 `@head` 輸出會反映目標路由的中繼資料。透過 `wire:navigate` 造訪的頁面會收到相應的路由、執行期與錯誤中繼資料，無需撰寫元件層級的 head 程式碼。


<a name="inertia"></a>
### Inertia

在 Inertia 根範本中使用相同的 `@head` 指令，並與 Inertia 自己的元件並列：

```blade
<html>
<head>
    <meta charset="utf-8">
    @head

    @viteReactRefresh
    @vite(['resources/css/app.css', 'resources/js/app.tsx'])
    <x-inertia::head />
</head>
<body>
    <x-inertia::app />
</body>
</html>
```

安裝 Inertia 後，Laravel Head 會自動在每個頁面物件上的 `head` prop 下，將頁面管理的 head 作為已渲染的元素字串陣列進行共享：

```json
{
    "props": {
        "head": [
            "<title data-inertia=\"title\">Dashboard - Laravel</title>",
            "<meta data-inertia=\"description\" name=\"description\" content=\"Your application overview.\">"
        ]
    }
}
```

在應用程式呼叫 `createInertiaApp()` 的地方啟用 Inertia 的 `serverHead` 選項。此選項自 Inertia 3.5 起提供：

```js
createInertiaApp({
    // ...
    serverHead: true,
});
```

每個由頁面管理的元素都具有穩定的 `data-inertia` 鍵值。`@head` 指令負責渲染初始文件，之後 Inertia 會接管這些元素，並在一般造訪、[即時造訪 (Instant Visits)](https://inertiajs.com/docs/v3/the-basics/instant-visits) 以及上一頁／下一頁導覽期間保持同步。這些元素存在於初始 HTML 回應中，因此檢索器 (Crawlers) 和連結預覽機器人可以在不執行 JavaScript 的情況下讀取它們。無需使用客戶端 `<Head>` 元件。

無論是否使用[伺服器端渲染 (SSR)](https://inertiajs.com/docs/v3/advanced/server-side-rendering)，此機制都能正常運作。如果應用程式有獨立的 SSR 進入點，也請在該處啟用 `serverHead`。Laravel Head 會自動在 `@head` 和 `<x-inertia::head />` 之間對頁面管理的元素進行去重，無論它們的順序為何，同時保留由 JavaScript SSR 產生的其他 head 元素。

> [!NOTE]
> 將 Laravel Head 新增至現有的 Inertia 應用程式時，請從 `resources/js/app.tsx` 和 `resources/js/ssr.tsx` 中移除任何標題回呼，讓 Laravel Head 能夠管理最終的文件標題，並將由 Inertia 的 [`<Head>` 元件](https://inertiajs.com/docs/v3/the-basics/title-and-meta) 管理的標籤移至 Laravel Head 中，避免兩者定義相同的元素。

`head` prop 會從部分重新載入 (Partial Reload) 的回應中省略，因此 Inertia 會保留上一個完整頁面的 head。即時造訪同樣會保留目前的 head，直到背景回應到達為止。如果你的應用程式已使用了 `head` prop，可以在服務提供者中變更其名稱：

```php
use Laravel\Head\Facades\Head;

public function boot(): void
{
    Head::inertia(prop: '_head');
}
```

接著使用 `serverHead: '_head'` 將 Inertia 指向同一個 prop。


<a name="static-inertia-tags"></a>
#### 靜態 Inertia 標籤

大多數標籤應該放在預設值、路由中繼資料或執行期中繼資料中，以便 Laravel Head 能夠為每個頁面解析出正確的值。僅在文件標籤需於第一個 HTML 回應中渲染，且在該連線階段後續都不會被 Inertia 變更時，才使用 Inertia 全域標籤 (Globals)。

在服務提供者中使用 `Head::inertiaGlobals()` 來註冊它們：

```php
use Laravel\Head\Facades\Head;
use Laravel\Head\HeadBuilder;

Head::inertiaGlobals(function (HeadBuilder $head) {
    $head
        ->viewport('width=device-width, initial-scale=1')
        ->colorScheme('light dark')
        ->icon('/favicon.svg', type: 'image/svg+xml')
        ->appleTouchIcon('/apple-touch-icon.png', sizes: '180x180')
        ->manifest('/site.webmanifest');
});
```

Inertia 全域標籤會從 `head` prop 中排除，渲染時不帶 `data-inertia` 歸屬屬性，且在第一次回應後永遠不會被更新。這些全域標籤適用於穩定的瀏覽器提示，例如 viewport、色彩配置、網站圖示 (Favicons)、觸控圖示 (Touch Icons) 與 manifests。如果某個標籤是特定頁面專屬、與 SEO 相關，或者稍後可能會被覆寫，請改將其放在 `defaults`、路由中繼資料或執行期中繼資料中。

需要將解析後的中繼資料作為結構化資料而非已渲染標籤的應用程式，可以呼叫 `Head::toArray()`。回傳的資料包含標題、Open Graph 數值、JSON-LD schemas 以及其他解析後的中繼資料。