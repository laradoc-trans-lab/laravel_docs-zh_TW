# Blade 樣板

- [簡介](#introduction)
    - [使用 Livewire 強化 Blade](#supercharging-blade-with-livewire)
- [顯示資料](#displaying-data)
    - [HTML 實體編碼](#html-entity-encoding)
    - [Blade 與 JavaScript 框架](#blade-and-javascript-frameworks)
- [Blade 指令](#blade-directives)
    - [If 陳述式](#if-statements)
    - [Switch 陳述式](#switch-statements)
    - [迴圈](#loops)
    - [迴圈變數](#the-loop-variable)
    - [條件式 Class 與樣式](#conditional-classes)
    - [附加屬性](#additional-attributes)
    - [引入子視圖](#including-subviews)
    - [ `@once` 指令](#the-once-directive)
    - [原生 PHP](#raw-php)
    - [註解](#comments)
- [元件](#components)
    - [渲染元件](#rendering-components)
    - [索引元件](#index-components)
    - [傳遞資料至元件](#passing-data-to-components)
    - [元件屬性](#component-attributes)
    - [保留關鍵字](#reserved-keywords)
    - [Slot](#slots)
    - [行內元件視圖](#inline-component-views)
    - [動態元件](#dynamic-components)
    - [手動註冊元件](#manually-registering-components)
- [匿名元件](#anonymous-components)
    - [匿名索引元件](#anonymous-index-components)
    - [資料屬性 / Attributes](#data-properties-attributes)
    - [存取父層資料](#accessing-parent-data)
    - [匿名元件路徑](#anonymous-component-paths)
- [建構版面](#building-layouts)
    - [使用元件的版面](#layouts-using-components)
    - [使用樣板繼承的版面](#layouts-using-template-inheritance)
- [表單](#forms)
    - [CSRF 欄位](#csrf-field)
    - [方法欄位](#method-field)
    - [驗證錯誤](#validation-errors)
- [Stack](#stacks)
- [服務注入](#service-injection)
- [渲染行內 Blade 樣板](#rendering-inline-blade-templates)
- [渲染 Blade 片段](#rendering-blade-fragments)
- [擴充 Blade](#extending-blade)
    - [自訂 Echo 處理常式](#custom-echo-handlers)
    - [自訂 If 陳述式](#custom-if-statements)

<a name="introduction"></a>
## 簡介

Blade 是 Laravel 內建的一個簡單卻又強大的樣板引擎。與某些 PHP 樣板引擎不同，Blade 並不限制你在樣板中使用原生 PHP 程式碼。事實上，所有的 Blade 樣板都會被編譯成原生 PHP 程式碼並快取起來，直到被修改為止。這也代表 Blade 基本上不會對你的應用程式造成任何負擔。Blade 樣板檔案使用 `.blade.php` 副檔名，且通常儲存於 `resources/views` 目錄中。

Blade 視圖可使用全域的 `view` 輔助函式從路由或控制器中回傳。當然，如[視圖](/docs/{{version}}/views)文件中所述，我們也可以使用 `view` 輔助函式的第二個參數來將資料傳遞至 Blade 視圖：

```php
Route::get('/', function () {
    return view('greeting', ['name' => 'Finn']);
});
```

<a name="supercharging-blade-with-livewire"></a>
### 使用 Livewire 強化 Blade

想讓你的 Blade 樣板更上一層樓，並輕鬆地建構動態介面嗎？快來看看 [Laravel Livewire](https://livewire.laravel.com)。Livewire 讓你能撰寫 Blade 元件，並為其擴充動態功能。這些功能通常只有在 React 或 Vue 等前端框架中才可能實現。對於想建構現代化、響應式的前端，卻又不希望處理許多 JavaScript 框架的複雜性、客戶端渲染或建置步驟等問題的開發者來說，Livewire 提供了一個絕佳的方案。

<a name="displaying-data"></a>
## 顯示資料

若要顯示傳入 Blade 視圖的資料，可將變數用大括號包起來。舉例來說，假設有下列路由：

```php
Route::get('/', function () {
    return view('welcome', ['name' => 'Samantha']);
});
```

你可以像這樣顯示 `name` 變數的內容：

```blade
Hello, {{ $name }}.
```

> [!NOTE]
> Blade 的 `{{ }}` echo 陳述式會自動通過 PHP 的 `htmlspecialchars` 函式來處理，以預防 XSS 攻擊。

不僅限於顯示傳入視圖的變數內容，你也可以輸出任何 PHP 函式的結果。事實上，你可以在 Blade 的 echo 陳述式中放入任何你想要的 PHP 程式碼：

```blade
The current UNIX timestamp is {{ time() }}.
```

<a name="html-entity-encoding"></a>
### HTML 實體編碼

預設情況下，Blade（以及 Laravel 的 `e` 函式）會對 HTML 實體進行雙重編碼。若想停用雙重編碼，請在 `AppServiceProvider` 的 `boot` 方法中呼叫 `Blade::withoutDoubleEncoding` 方法：

```php
<?php

namespace App\Providers;

use Illuminate\Support\Facades\Blade;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        Blade::withoutDoubleEncoding();
    }
}
```

<a name="displaying-unescaped-data"></a>
#### 顯示未逸出資料

預設情況下，Blade 的 `{{ }}` 陳述式會自動通過 PHP 的 `htmlspecialchars` 函式來處理，以預防 XSS 攻擊。若不想讓資料被逸出，可使用下列語法：

```blade
Hello, {!! $name !!}.
```

> [!WARNING]
> 在輸出由應用程式使用者提供的內容時，請務必格外小心。顯示使用者提供的資料時，通常應使用逸出的雙大括號語法來預防 XSS 攻擊。

<a name="blade-and-javascript-frameworks"></a>
### Blade 與 JavaScript 框架

由於許多 JavaScript 框架也使用「大括號」來表示應在瀏覽器中顯示的表達式，因此你可以使用 `@` 符號來告知 Blade 渲染引擎該表達式應保持不變。例如：

```blade
<h1>Laravel</h1>

Hello, @{{ name }}.
```

在此範例中，Blade 會移除 `@` 符號；然而，`{{ name }}` 表達式將會被 Blade 引擎保持原樣，從而使其可被你的 JavaScript 框架渲染。

`@` 符號也可用於逸出 Blade 指令：

```blade
{{-- Blade template --}}
@@if()

<!-- HTML output -->
@if()
```

<a name="rendering-json"></a>
#### 渲染 JSON

有時，你可能會將一個陣列傳遞到視圖中，目的是將其渲染為 JSON 來初始化一個 JavaScript 變數。例如：

```php
<script>
    var app = <?php echo json_encode($array); ?>;
</script>
```

不過，除了手動呼叫 `json_encode`，你還可以使用 `Illuminate\Support\Js::from` 方法。`from` 方法接受與 PHP 的 `json_encode` 函式相同的參數；不過，它會確保產生的 JSON 已被正確地逸出，以便包含在 HTML 引號中。`from` 方法會回傳一個字串形式的 `JSON.parse` JavaScript 陳述式，該陳述式會將給定的物件或陣列轉換為一個有效的 JavaScript 物件：

```blade
<script>
    var app = {{ Illuminate\Support\Js::from($array) }};
</script>
```

最新版本的 Laravel 應用程式骨架包含一個 `Js` facade，可在你的 Blade 樣板中方便地存取此功能：

```blade
<script>
    var app = {{ Js::from($array) }};
</script>
```

> [!WARNING]
> 你應該只使用 `Js::from` 方法將現有變數渲染為 JSON。Blade 樣板是基於正規表示式，嘗試將複雜的表達式傳遞給該指令可能會導致非預期的錯誤。

<a name="the-at-verbatim-directive"></a>
#### `@verbatim` 指令

若樣板中有很大一部分都在顯示 JavaScript 變數，可以將 HTML 包在 `@verbatim` 指令中，這樣就不需要為每個 Blade echo 陳述式都加上 `@` 符號前綴了：

```blade
@verbatim
    <div class="container">
        Hello, {{ name }}.
    </div>
@endverbatim
```

<a name="blade-directives"></a>
## Blade 指令

除了樣板繼承與顯示資料外，Blade 還為常見的 PHP 控制結構提供了方便的捷徑，例如條件陳述式與迴圈。這些捷徑提供了一種非常簡潔、扼要的方式來使用 PHP 控制結構，同時也保留了與 PHP 語法相似的風格。

<a name="if-statements"></a>
### If 陳述式

我們可以使用 `@if`、`@elseif`、`@else`、與 `@endif` 指令來建構 `if` 陳述式。這些指令的功能與其對應的 PHP 語法完全相同：

```blade
@if (count($records) === 1)
    I have one record!
@elseif (count($records) > 1)
    I have multiple records!
@else
    I don't have any records!
@endif
```

為了方便，Blade 也提供了一個 `@unless` 指令：

```blade
@unless (Auth::check())
    You are not signed in.
@endunless
```

除了上面已經討論過的條件指令外，`@isset` 與 `@empty` 指令也可用來作為其對應 PHP 函式的方便捷徑：

```blade
@isset($records)
    // $records is defined and is not null...
@endisset

@empty($records)
    // $records is "empty"...
@endempty
```

<a name="authentication-directives"></a>
#### 認證指令

`@auth` 與 `@guest` 指令可用來快速判斷目前的使用者是否已經[登入](/docs/{{version}}/authentication)或為訪客：

```blade
@auth
    // The user is authenticated...
@endauth

@guest
    // The user is not authenticated...
@endguest
```

若有需要，在使用 `@auth` 與 `@guest` 指令時，我們也可以指定要檢查的認證 Guard：

```blade
@auth('admin')
    // The user is authenticated...
@endauth

@guest('admin')
    // The user is not authenticated...
@endguest
```

<a name="environment-directives"></a>
#### 環境指令

我們可以使用 `@production` 指令來檢查應用程式是否正在生產 (Production) 環境中執行：

```blade
@production
    // Production specific content...
@endproduction
```

或者，我們也可以使用 `@env` 指令來判斷應用程式是否正在指定的環境中執行：

```blade
@env('staging')
    // The application is running in "staging"...
@endenv

@env(['staging', 'production'])
    // The application is running in "staging" or "production"...
@endenv
```

<a name="section-directives"></a>
#### Section 指令

我們可以使用 `@hasSection` 指令來判斷某個樣板繼承的 Section 是否有內容：

```blade
@hasSection('navigation')
    <div class="pull-right">
        @yield('navigation')
    </div>

    <div class="clearfix"></div>
@endif
```

我們可以使用 `sectionMissing` 指令來判斷某個 Section 是否沒有內容：

```blade
@sectionMissing('navigation')
    <div class="pull-right">
        @include('default-navigation')
    </div>
@endif
```

<a name="session-directives"></a>
#### Session 指令

`@session` 指令可用來判斷某個 [Session](/docs/{{version}}/session) 值是否存在。若 Session 值存在，則會執行 `@session` 與 `@endsession` 指令中的樣板內容。在 `@session` 指令的內容中，我們可以 Echo `$value` 變數來顯示 Session 的值：

```blade
@session('status')
    <div class="p-4 bg-green-100">
        {{ $value }}
    </div>
@endsession
```

<a name="context-directives"></a>
#### Context 指令

`@context` 指令可用於判斷某個 [Context](/docs/{{version}}/context) 值是否存在。如果 Context 值存在，`@context` 和 `@endcontext` 指令中的樣板內容將會被求值。在 `@context` 指令的內容中，你可以 echo `$value` 變數來顯示 Context 值：

```blade
@context('canonical')
    <link href="{{ $value }}" rel="canonical">
@endcontext
```

<a name="switch-statements"></a>
### Switch 陳述式

Switch 陳述式可以使用 `@switch`、`@case`、`@break`、`@default`、與 `@endswitch` 指令來建構：

```blade
@switch($i)
    @case(1)
        First case...
        @break

    @case(2)
        Second case...
        @break

    @default
        Default case...
@endswitch
```

<a name="loops"></a>
### 迴圈

除了條件陳述式外，Blade 也提供了簡單的指令來使用 PHP 的迴圈結構。同樣的，每個指令的功能都與其對應的 PHP 語法完全相同：

```blade
@for ($i = 0; $i < 10; $i++)
    The current value is {{ $i }}
@endfor

@foreach ($users as $user)
    <p>This is user {{ $user->id }}</p>
@endforeach

@forelse ($users as $user)
    <li>{{ $user->name }}</li>
@empty
    <p>No users</p>
@endforelse

@while (true)
    <p>I'm looping forever.</p>
@endwhile
```

> [!NOTE]
> 在 `foreach` 迴圈中，我們可以使用[迴圈變數](#the-loop-variable)來取得一些關於這個迴圈的有用資訊，例如目前是否為第一個或最後一個迭代。

使用迴圈時，我們也可以使用 `@continue` 與 `@break` 指令來跳過目前的迭代或結束迴圈：

```blade
@foreach ($users as $user)
    @if ($user->type == 1)
        @continue
    @endif

    <li>{{ $user->name }}</li>

    @if ($user->number == 5)
        @break
    @endif
@endforeach
```

我們也可以在指令宣告中包含 continue 或 break 的條件：

```blade
@foreach ($users as $user)
    @continue($user->type == 1)

    <li>{{ $user->name }}</li>

    @break($user->number == 5)
@endforeach
```

<a name="the-loop-variable"></a>
### 迴圈變數

在 `foreach` 迴圈中，`$loop` 變數可用於迴圈內。該變數提供了一些有用的資訊，例如目前的迴圈索引、目前是否為第一個或最後一個迭代等：

```blade
@foreach ($users as $user)
    @if ($loop->first)
        This is the first iteration.
    @endif

    @if ($loop->last)
        This is the last iteration.
    @endif

    <p>This is user {{ $user->id }}</p>
@endforeach
```

若在巢狀的迴圈中，可以透過 `parent` 屬性來存取父層迴圈的 `$loop` 變數：

```blade
@foreach ($users as $user)
    @foreach ($user->posts as $post)
        @if ($loop->parent->first)
            This is the first iteration of the parent loop.
        @endif
    @endforeach
@endforeach
```

`$loop` 變數也包含了其他各種有用的屬性：

<div class="overflow-auto">

| 屬性 | 說明 |
| ------------------ | ------------------------------------------------------ |
| `$loop->index` | 目前迴圈迭代的索引 (從 0 開始)。 |
| `$loop->iteration` | 目前的迴圈迭代 (從 1 開始)。 |
| `$loop->remaining` | 迴圈中剩餘的迭代次數。 |
| `$loop->count` | 正在迭代的陣列中的項目總數。 |
| `$loop->first` | 目前的迭代是否為第一個。 |
| `$loop->last` | 目前的迭代是否為最後一個。 |
| `$loop->even` | 目前的迭代是否為偶數。 |
| `$loop->odd` | 目前的迭代是否為奇數。 |
| `$loop->depth` | 目前迴圈的巢狀層級。 |
| `$loop->parent` | 在巢狀迴圈中，父層迴圈的變數。 |

</div>

<a name="conditional-classes"></a>
### 條件式 Class 與樣式

`@class` 指令可以有條件地編譯 CSS Class 字串。該指令接受一個 Class 陣列，其中陣列的索引鍵包含要加入的一或多個 Class，而值則是一個布林運算式。如果陣列元素是數字索引鍵，它將一律被包含在渲染的 Class 清單中：

```blade
@php
    $isActive = false;
    $hasError = true;
@endphp

<span @class([
    'p-4',
    'font-bold' => $isActive,
    'text-gray-500' => ! $isActive,
    'bg-red' => $hasError,
])></span>

<span class="p-4 text-gray-500 bg-red"></span>
```

同樣地，`@style` 指令可用來有條件地將行內 CSS 樣式新增至 HTML 元素：

```blade
@php
    $isActive = true;
@endphp

<span @style([
    'background-color: red',
    'font-weight: bold' => $isActive,
])></span>

<span style="background-color: red; font-weight: bold;"></span>
```


<a name="additional-attributes"></a>
### 附加屬性

為了方便起見，您可以使用 `@checked` 指令輕鬆地指出某個 HTML 核取方塊輸入是否應為「checked」。如果提供的條件評估為 `true`，此指令將會輸出 `checked`：

```blade
<input
    type="checkbox"
    name="active"
    value="active"
    @checked(old('active', $user->active))
/>
```

同樣地，`@selected` 指令可用來指出某個 select 選項是否應為「selected」：

```blade
<select name="version">
    @foreach ($product->versions as $version)
        <option value="{{ $version }}" @selected(old('version') == $version)>
            {{ $version }}
        </option>
    @endforeach
</select>
```

此外，`@disabled` 指令可用來指出某個元素是否應為「disabled」：

```blade
<button type="submit" @disabled($errors->isNotEmpty())>Submit</button>
```

此外，`@readonly` 指令可用來指出某個元素是否應為「readonly」：

```blade
<input
    type="email"
    name="email"
    value="email@laravel.com"
    @readonly($user->isNotAdmin())
/>
```

此外，`@required` 指令可用來指出某個元素是否應為「required」：

```blade
<input
    type="text"
    name="title"
    value="title"
    @required($user->isAdmin())
/>
```


<a name="including-subviews"></a>
### 引入子視圖

> [!NOTE]
> 雖然您可以自由使用 `@include` 指令，但 Blade [元件](#components) 提供了類似的功能，並提供了一些優於 `@include` 指令的好處，例如資料和屬性綁定。

Blade 的 `@include` 指令可讓您從另一個視圖中引入一個 Blade 視圖。父視圖中可用的所有變數都將在被引入的視圖中可用：

```blade
<div>
    @include('shared.errors')

    <form>
        <!-- Form Contents -->
    </form>
</div>
```

即使被引入的視圖會繼承父視圖中所有可用的資料，您也可以傳遞一個額外的資料陣列，讓這些資料在被引入的視圖中可用：

```blade
@include('view.name', ['status' => 'complete'])
```

如果您嘗試 `@include` 一個不存在的視圖，Laravel 會擲出一個錯誤。如果您想引入一個可能存在也可能不存在的視圖，您應該使用 `@includeIf` 指令：

```blade
@includeIf('view.name', ['status' => 'complete'])
```

如果您想在給定的布林運算式評估為 `true` 或 `false` 時 `@include` 一個視圖，您可以使用 `@includeWhen` 和 `@includeUnless` 指令：

```blade
@includeWhen($boolean, 'view.name', ['status' => 'complete'])

@includeUnless($boolean, 'view.name', ['status' => 'complete'])
```

若要從給定的視圖陣列中引入第一個存在的視圖，您可以使用 `includeFirst` 指令：

```blade
@includeFirst(['custom.admin', 'admin'], ['status' => 'complete'])
```

> [!WARNING]
> 您應避免在 Blade 視圖中使用 `__DIR__` 和 `__FILE__` 常數，因為它們將會指向快取、已編譯視圖的位置。


<a name="rendering-views-for-collections"></a>
#### 為集合渲染視圖

您可以使用 Blade 的 `@each` 指令將迴圈和引入合併為一行：

```blade
@each('view.name', $jobs, 'job')
```

`@each` 指令的第一個參數是要為陣列或集合中的每個元素渲染的視圖。第二個參數是您希望迭代的陣列或集合，而第三個參數是在視圖中分配給目前迭代的變數名稱。因此，舉例來說，如果您正在迭代一個 `jobs` 陣列，您通常會希望在視圖中將每個 job 作為 `job` 變數來存取。目前迭代的陣列索引鍵將作為 `key` 變數在視圖中可用。

您也可以將第四個參數傳遞給 `@each` 指令。此參數決定了當給定陣列為空時將渲染的視圖。

```blade
@each('view.name', $jobs, 'job', 'view.empty')
```

> [!WARNING]
> 透過 `@each` 渲染的視圖不會繼承父視圖的變數。如果子視圖需要這些變數，您應該改用 `@foreach` 和 `@include` 指令。


<a name="the-once-directive"></a>
### `@once` 指令

`@once` 指令可讓您定義樣板的一部分，這部分在每個渲染週期中只會被評估一次。這對於使用 [stacks](#stacks) 將某段 JavaScript 推送到頁面標頭中可能很有用。例如，如果您在迴圈中渲染一個給定的[元件](#components)，您可能希望只在元件第一次渲染時將 JavaScript 推送到標頭：

```blade
@once
    @push('scripts')
        <script>
            // Your custom JavaScript...
        </script>
    @endpush
@endonce
```

由於 `@once` 指令經常與 `@push` 或 `@prepend` 指令一起使用，為了方便起見，`@pushOnce` 和 `@prependOnce` 指令是可用的：

```blade
@pushOnce('scripts')
    <script>
        // Your custom JavaScript...
    </script>
@endPushOnce
```

如果您從兩個不同的 Blade 樣板中推送重複的內容，您應該提供一個唯一的識別碼作為 `@pushOnce` 指令的第二個參數，以確保內容只被渲染一次：

```blade
<!-- pie-chart.blade.php -->
@pushOnce('scripts', 'chart.js')
    <script src="/chart.js"></script>
@endPushOnce

<!-- line-chart.blade.php -->
@pushOnce('scripts', 'chart.js')
    <script src="/chart.js"></script>
@endPushOnce
```


<a name="raw-php"></a>
### 原生 PHP

在某些情況下，將 PHP 程式碼嵌入到視圖中很有用。您可以使用 Blade 的 `@php` 指令在樣板中執行一個原生 PHP 區塊：

```blade
@php
    $counter = 1;
@endphp
```

或者，如果您只需要使用 PHP 來匯入一個類別，您可以使用 `@use` 指令：

```blade
@use('App\Models\Flight')
```

可以提供第二個參數給 `@use` 指令，來為匯入的類別設定別名：

```blade
@use('App\Models\Flight', 'FlightModel')
```

如果您在同一個命名空間中有多個類別，您可以將這些類別的匯入分組：

```blade
@use('App\Models\{Flight, Airport}')
```

`@use` 指令也支援透過在匯入路徑前加上 `function` 或 `const` 修飾詞來匯入 PHP 函式與常數：

```blade
@use(function App\Helpers\format_currency)
@use(const App\Constants\MAX_ATTEMPTS)
```

就像類別匯入一樣，函式和常數也支援別名：

```blade
@use(function App\Helpers\format_currency, 'formatMoney')
@use(const App\Constants\MAX_ATTEMPTS, 'MAX_TRIES')
```

群組匯入也支援 function 和 const 修飾詞，讓您可以在單一指令中從同一個命名空間匯入多個符號：

```blade
@use(function App\Helpers\{format_currency, format_date})
@use(const App\Constants\{MAX_ATTEMPTS, DEFAULT_TIMEOUT})
```

<a name="comments"></a>
### 註解

Blade 也允許你在視圖中定義註解。不過，與 HTML 註解不同的是，Blade 註解不會包含在應用程式回傳的 HTML 中：

```blade
{{-- This comment will not be present in the rendered HTML --}}
```

<a name="components"></a>
## 元件

元件與 Slot 提供的優點與 Section、版面、Include 相似；不過，有些人可能會覺得元件與 Slot 的心智模型更容易理解。撰寫元件的方法有兩種：基於 Class 的元件與匿名元件。

若要建立一個基於 Class 的元件，可使用 `make:component` Artisan 指令。為了說明如何使用元件，我們將建立一個簡單的 `Alert` 元件。`make:component` 指令會將元件放置於 `app/View/Components` 目錄內：

```shell
php artisan make:component Alert
```

`make:component` 指令也會為該元件建立一個視圖樣板。該視圖會被放置於 `resources/views/components` 目錄內。為自己的應用程式撰寫元件時，Laravel 會自動在 `app/View/Components` 目錄與 `resources/views/components` 目錄內探索元件，因此通常不需要額外註冊元件。

也可以在子目錄中建立元件：

```shell
php artisan make:component Forms/Input
```

上述指令會在 `app/View/Components/Forms` 目錄中建立一個 `Input` 元件，且視圖會被放置於 `resources/views/components/forms` 目錄內。

<a name="manually-registering-package-components"></a>
#### 手動註冊套件元件

為自己的應用程式撰寫元件時，Laravel 會自動在 `app/View/Components` 目錄與 `resources/views/components` 目錄中探索元件。

不過，若正在建置一個使用 Blade 元件的套件，則需要手動註冊元件的 Class 與其 HTML 標籤別名。通常應在套件的 Service Provider 中的 `boot` 方法內註冊元件：

```php
use Illuminate\Support\Facades\Blade;

/**
 * Bootstrap your package's services.
 */
public function boot(): void
{
    Blade::component('package-alert', Alert::class);
}
```

元件註冊後，就可以使用其標籤別名來渲染：

```blade
<x-package-alert/>
```

或者，也可以使用 `componentNamespace` 方法來按照慣例自動載入元件 Class。舉例來說，有個 `Nightshade` 套件，其中可能會有 `Calendar` 與 `ColorPicker` 元件，這兩個元件位於 `Package\Views\Components` 命名空間內：

```php
use Illuminate\Support\Facades\Blade;

/**
 * Bootstrap your package's services.
 */
public function boot(): void
{
    Blade::componentNamespace('Nightshade\\Views\\Components', 'nightshade');
}
```

這麼一來，就可以使用 `package-name::` 語法，並透過其提供者 (Vendor) 命名空間來使用套件元件：

```blade
<x-nightshade::calendar />
<x-nightshade::color-picker />
```

Blade 會自動將元件名稱轉為 Pascal 命名法來偵測與該元件連結的 Class。也支援使用「點」號表示法來表示子目錄。

<a name="rendering-components"></a>
### 渲染元件

若要顯示元件，可在 Blade 樣板中使用 Blade 元件標籤。Blade 元件標籤以 `x-` 字串開頭，後面接著元件 Class 的 Kebab 命名法名稱：

```blade
<x-alert/>

<x-user-profile/>
```

若元件 Class 位於 `app/View/Components` 目錄下更深的層級，可使用 `.` 字元來表示目錄的巢狀結構。舉例來說，假設有個元件位於 `app/View/Components/Inputs/Button.php`，則可以像這樣渲染它：

```blade
<x-inputs.button/>
```

若想有條件地渲染元件，可在元件 Class 上定義一個 `shouldRender` 方法。若 `shouldRender` 方法回傳 `false`，則該元件將不會被渲染：

```php
use Illuminate\Support\Str;

/**
 * Whether the component should be rendered
 */
public function shouldRender(): bool
{
    return Str::length($this->message) > 0;
}
```

<a name="index-components"></a>
### 索引元件

有時候，元件會屬於某個元件群組，而我們可能會想將相關的元件都放在同一個目錄下。舉例來說，假設有個「卡片 (Card)」元件，其 Class 結構如下：

```text
App\Views\Components\Card\Card
App\Views\Components\Card\Header
App\Views\Components\Card\Body
```

由於根 `Card` 元件被放在 `Card` 目錄內，你可能會以為需要透過 `<x-card.card>` 來渲染該元件。不過，當元件的檔名與元件目錄的名稱相同時，Laravel 會自動假設該元件為「根」元件，並允許在渲染時省略重複的目錄名稱：

```blade
<x-card>
    <x-card.header>...</x-card.header>
    <x-card.body>...</x-card.body>
</x-card>
```

<a name="passing-data-to-components"></a>
### 傳遞資料至元件

你可以使用 HTML 屬性來傳遞資料到 Blade 元件。寫死的、基本型別的值可以使用簡單的 HTML 屬性字串傳遞給元件。PHP 運算式與變數則應透過使用 `:` 字元作為前綴的屬性來傳遞給元件：

```blade
<x-alert type="error" :message="$message"/>
```

你應在元件的 Class 建構函式中定義所有元件的資料屬性。元件上的所有公開屬性都會自動提供給元件的視圖使用。不需要從元件的 `render` 方法中將資料傳遞給視圖：

```php
<?php

namespace App\View\Components;

use Illuminate\View\Component;
use Illuminate\View\View;

class Alert extends Component
{
    /**
     * Create the component instance.
     */
    public function __construct(
        public string $type,
        public string $message,
    ) {}

    /**
     * Get the view / contents that represent the component.
     */
    public function render(): View
    {
        return view('components.alert');
    }
}
```

當你的元件被渲染時，你可以透過變數名稱來 Echo 元件公開屬性的內容：

```blade
<div class="alert alert-{{ $type }}">
    {{ $message }}
</div>
```

<a name="casing"></a>
#### 大小寫

元件建構函式的引數應使用 `camelCase` 來指定，而在 HTML 屬性中參照引數名稱時，則應使用 `kebab-case`。例如，給定以下元件建構函式：

```php
/**
 * Create the component instance.
 */
public function __construct(
    public string $alertType,
) {}
```

`$alertType` 引數可以像這樣提供給元件：

```blade
<x-alert alert-type="danger" />
```

<a name="short-attribute-syntax"></a>
#### 簡寫屬性語法

將屬性傳遞給元件時，你也可以使用「簡寫屬性」語法。這通常很方便，因為屬性名稱經常與它們對應的變數名稱相符：

```blade
{{-- Short attribute syntax... --}}
<x-profile :$userId :$name />

{{-- Is equivalent to... --}}
<x-profile :user-id="$userId" :name="$name" />
```

<a name="escaping-attribute-rendering"></a>
#### 跳脫屬性渲染

由於某些 JavaScript 框架（如 Alpine.js）也使用冒號前綴的屬性，你可以使用雙冒號（`::`）前綴來告知 Blade 該屬性不是 PHP 運算式。例如，給定以下元件：

```blade
<x-button ::class="{ danger: isDeleting }">
    Submit
</x-button>
```

Blade 將會渲染出以下的 HTML：

```blade
<button :class="{ danger: isDeleting }">
    Submit
</button>
```

<a name="component-methods"></a>
#### 元件方法

除了公開變數可供元件樣板使用外，元件上的任何公開方法都可以被叫用。例如，假設一個元件有一個 `isSelected` 方法：

```php
/**
 * Determine if the given option is the currently selected option.
 */
public function isSelected(string $option): bool
{
    return $option === $this->selected;
}
```

你可以透過叫用與該方法名稱相符的變數，從元件樣板中執行此方法：

```blade
<option {{ $isSelected($value) ? 'selected' : '' }} value="{{ $value }}">
    {{ $label }}
</option>
```

<a name="using-attributes-slots-within-component-class"></a>
#### 在元件 Class 內存取屬性與 Slot

Blade 元件也允許你在類別的 render 方法內存取元件名稱、屬性與 slot。不過，為了存取這些資料，你應從元件的 `render` 方法中回傳一個閉包：

```php
use Closure;

/**
 * Get the view / contents that represent the component.
 */
public function render(): Closure
{
    return function () {
        return '<div {{ $attributes }}>Components content</div>';
    };
}
```

從元件的 `render` 方法回傳的閉包也可以接收一個 `$data` 陣列作為其唯一的引數。此陣列會包含數個元素，提供關於該元件的資訊：

```php
return function (array $data) {
    // $data['componentName'];
    // $data['attributes'];
    // $data['slot'];

    return '<div {{ $attributes }}>Components content</div>';
}
```

> [!WARNING]
> 切勿將 `$data` 陣列中的元素直接嵌入到 `render` 方法回傳的 Blade 字串中，因為這麼做可能因惡意的屬性內容而導致遠端程式碼執行。

`componentName` 等於 HTML 標籤中 `x-` 前綴後的名稱。因此 `<x-alert />` 的 `componentName` 會是 `alert`。`attributes` 元素會包含所有出現在 HTML 標籤上的屬性。`slot` 元素是一個 `Illuminate\Support\HtmlString` 的實體，其中包含元件 slot 的內容。

閉包應回傳一個字串。若回傳的字串對應到一個已存在的視圖，則會渲染該視圖；否則，回傳的字串將被當作行內 Blade 視圖來評估。

<a name="additional-dependencies"></a>
#### 附加的相依性

如果你的元件需要來自 Laravel [服務容器](/docs/{{version}}/container) 的相依性，你可以在任何元件資料屬性之前列出它們，它們將會由容器自動注入：

```php
use App\Services\AlertCreator;

/**
 * Create the component instance.
 */
public function __construct(
    public AlertCreator $creator,
    public string $type,
    public string $message,
) {}
```

<a name="hiding-attributes-and-methods"></a>
#### 隱藏屬性 / 方法

如果你想防止某些公開方法或屬性被當作變數暴露給元件樣板，你可以將它們新增到元件的 `$except` 陣列屬性中：

```php
<?php

namespace App\View\Components;

use Illuminate\View\Component;

class Alert extends Component
{
    /**
     * The properties / methods that should not be exposed to the component template.
     *
     * @var array
     */
    protected $except = ['type'];

    /**
     * Create the component instance.
     */
    public function __construct(
        public string $type,
    ) {}
}
```

<a name="component-attributes"></a>
### 元件屬性

我們已經看過要怎麼傳遞資料屬性給元件了。不過，有時候我們可能需要指定一些額外的 HTML 屬性 (Attribute)，例如 `class`，而這些屬性又不屬於元件運作所需的資料。一般來說，我們會想把這些額外的屬性傳遞到元件樣板的根元素上。舉例來說，假設我們想這樣渲染一個 `alert` 元件：

```blade
<x-alert type="error" :message="$message" class="mt-4"/>
```

所有不屬於元件建構函式的屬性都會被自動加到元件的「屬性包 (Attribute Bag)」中。這個屬性包會透過 `$attributes` 變數自動提供給元件。只要 Echo 這個變數，就能在元件中渲染所有的屬性：

```blade
<div {{ $attributes }}>
    <!-- Component content -->
</div>
```

> [!WARNING]
> 目前不支援在元件標籤內使用 `@env` 這類指令。例如，`<x-alert :live="@env('production')"/>` 將不會被編譯。

<a name="default-merged-attributes"></a>
#### 預設 / 合併屬性

有時候，我們可能需要為屬性指定預設值，或是將額外的值合併到元件的某些屬性中。為此，可以使用屬性包的 `merge` 方法。這個方法在定義一組應永遠套用到某元件上的預設 CSS Class 時特別好用：

```blade
<div {{ $attributes->merge(['class' => 'alert alert-'.$type]) }}>
    {{ $message }}
</div>
```

假設我們這樣使用這個元件：

```blade
<x-alert type="error" :message="$message" class="mb-4"/>
```

該元件最終渲染出來的 HTML 會像這樣：

```blade
<div class="alert alert-error mb-4">
    <!-- Contents of the $message variable -->
</div>
```

<a name="conditionally-merge-classes"></a>
#### 條件式合併 Class

有時候，我們可能會想在某個條件為 `true` 的情況下才合併 Class。這可以透過 `class` 方法來辦到。class 方法可帶入一個 Class 陣列，其中，陣列的鍵為要加入的一或多個 Class，而值則為一個布林表達式。若陣列元素為數字鍵，則該 Class 會被一直包含在渲染出來的 Class 列表中：

```blade
<div {{ $attributes->class(['p-4', 'bg-red' => $hasError]) }}>
    {{ $message }}
</div>
```

若需要合併其他屬性到元件上，可以將 `merge` 方法串在 `class` 方法後面：

```blade
<button {{ $attributes->class(['p-4'])->merge(['type' => 'button']) }}>
    {{ $slot }}
</button>
```

> [!NOTE]
> 若想在其他不應收到合併屬性的 HTML 元素上有條件地編譯 Class，可以使用 [@class 指令](#conditional-classes)。

<a name="non-class-attribute-merging"></a>
#### 合併非 Class 的屬性

當合併非 `class` 的屬性時，提供給 `merge` 方法的值會被視為該屬性的「預設」值。不過，與 `class` 屬性不同，這些屬性不會與傳入的屬性值合併，而是會被覆寫。舉例來說，某個 `button` 元件的實作可能像這樣：

```blade
<button {{ $attributes->merge(['type' => 'button']) }}>
    {{ $slot }}
</button>
```

若要以自訂的 `type` 來渲染按鈕元件，可在使用元件時指定。若未指定類型，則會使用 `button` 類型：

```blade
<x-button type="submit">
    Submit
</x-button>
```

在這個例子中，`button` 元件渲染出來的 HTML 會是：

```blade
<button type="submit">
    Submit
</button>
```

若想讓 `class` 以外的屬性也能將其預設值與傳入值接在一起，可使用 `prepends` 方法。在這個例子中，`data-controller` 屬性將一直以 `profile-controller` 開頭，而任何額外傳入的 `data-controller` 值則會被加到這個預設值後面：

```blade
<div {{ $attributes->merge(['data-controller' => $attributes->prepends('profile-controller')]) }}>
    {{ $slot }}
</div>
```

<a name="filtering-attributes"></a>
#### 取得與篩選屬性

可使用 `filter` 方法來篩選屬性。這個方法可帶入一個閉包，若想在屬性包中保留該屬性，則閉包應回傳 `true`：

```blade
{{ $attributes->filter(fn (string $value, string $key) => $key == 'foo') }}
```

為了方便，可使用 `whereStartsWith` 方法來取得所有鍵 (Key) 為特定字串開頭的屬性：

```blade
{{ $attributes->whereStartsWith('wire:model') }}
```

相反地，`whereDoesntStartWith` 方法可用來排除所有鍵為特定字串開頭的屬性：

```blade
{{ $attributes->whereDoesntStartWith('wire:model') }}
```

使用 `first` 方法，可以渲染給定屬性包中的第一個屬性：

```blade
{{ $attributes->whereStartsWith('wire:model')->first() }}
```

若想檢查元件上是否有某個屬性，可使用 `has` 方法。這個方法只接受屬性名稱作為其唯一的引數，並回傳一個布林值，以表示該屬性是否存在：

```blade
@if ($attributes->has('class'))
    <div>Class attribute is present</div>
@endif
```

若傳遞一個陣列給 `has` 方法，則該方法會判斷所有給定的屬性是否存在於元件上：

```blade
@if ($attributes->has(['name', 'class']))
    <div>All of the attributes are present</div>
@endif
```

`hasAny` 方法可用來判斷給定的屬性中是否有任何一個存在於元件上：

```blade
@if ($attributes->hasAny(['href', ':href', 'v-bind:href']))
    <div>One of the attributes is present</div>
@endif
```

可使用 `get` 方法來取得特定屬性的值：

```blade
{{ $attributes->get('class') }}
```

`only` 方法可用來只取得有給定鍵的屬性：

```blade
{{ $attributes->only(['class']) }}
```

`except` 方法可用來取得除了有給定鍵以外的所有屬性：

```blade
{{ $attributes->except(['class']) }}
```

<a name="reserved-keywords"></a>
### 保留關鍵字

預設情況下，有一些關鍵字被保留給 Blade 內部用於渲染元件。下列關鍵字不能在你的元件中被定義為公開屬性或方法名稱：

<div class="content-list" markdown="1">

- `data`
- `render`
- `resolve`
- `resolveView`
- `shouldRender`
- `view`
- `withAttributes`
- `withName`

</div>

<a name="slots"></a>
### Slot

我們常常需要透過「slot」來傳遞額外內容到元件中。元件的 slot 是透過 echo `$slot` 變數來渲染的。為了探討這個概念，我們來假設有個 `alert` 元件，其標記如下：

```blade
<!-- /resources/views/components/alert.blade.php -->

<div class="alert alert-danger">
    {{ $slot }}
</div>
```

我們可以透過將內容注入到元件中來傳遞內容給 `slot`：

```blade
<x-alert>
    <strong>Whoops!</strong> Something went wrong!
</x-alert>
```

有時候，一個元件可能需要在元件內的不同位置渲染多個不同的 slot。我們來修改一下 alert 元件，讓它能注入一個「title」slot：

```blade
<!-- /resources/views/components/alert.blade.php -->

<span class="alert-title">{{ $title }}</span>

<div class="alert alert-danger">
    {{ $slot }}
</div>
```

我們可以使用 `x-slot` 標籤來定義已命名 slot 的內容。任何不在 `x-slot` 標籤中的內容，都會被傳遞到元件的 `$slot` 變數中：

```xml
<x-alert>
    <x-slot:title>
        Server Error
    </x-slot>

    <strong>Whoops!</strong> Something went wrong!
</x-alert>
```

你可以叫用 slot 的 `isEmpty` 方法來判斷該 slot 是否包含內容：

```blade
<span class="alert-title">{{ $title }}</span>

<div class="alert alert-danger">
    @if ($slot->isEmpty())
        This is default content if the slot is empty.
    @else
        {{ $slot }}
    @endif
</div>
```

此外，`hasActualContent` 方法可用於判斷 slot 是否包含任何不是 HTML 註解的「實際」內容：

```blade
@if ($slot->hasActualContent())
    The scope has non-comment content.
@endif
```

<a name="scoped-slots"></a>
#### 作用域 Slot

如果你曾用過像 Vue 這類的 JavaScript 框架，那你可能對「作用域 slot (scoped slot)」很熟悉。作用域 slot 能讓你在 slot 中存取元件的資料或方法。在 Laravel 中，我們可以透過在元件上定義 public 方法或屬性，並在 slot 中透過 `$component` 變數來存取元件以達成類似的行為。在本例中，我們假設 `x-alert` 元件在其元件 Class 上有定義一個 public 的 `formatAlert` 方法：

```blade
<x-alert>
    <x-slot:title>
        {{ $component->formatAlert('Server Error') }}
    </x-slot>

    <strong>Whoops!</strong> Something went wrong!
</x-alert>
```

<a name="slot-attributes"></a>
#### Slot 屬性

就像 Blade 元件一樣，我們也可以為 slot 指定[附加屬性](#component-attributes)，如 CSS Class 名稱：

```xml
<x-card class="shadow-sm">
    <x-slot:heading class="font-bold">
        Heading
    </x-slot>

    Content

    <x-slot:footer class="text-sm">
        Footer
    </x-slot>
</x-card>
```

若要與 slot 屬性互動，可以存取 slot 變數的 `attributes` 屬性。關於如何與屬性互動的更多資訊，請參考[元件屬性](#component-attributes)的說明文件：

```blade
@props([
    'heading',
    'footer',
])

<div {{ $attributes->class(['border']) }}>
    <h1 {{ $heading->attributes->class(['text-lg']) }}>
        {{ $heading }}
    </h1>

    {{ $slot }}

    <footer {{ $footer->attributes->class(['text-gray-700']) }}>
        {{ $footer }}
    </footer>
</div>
```

<a name="inline-component-views"></a>
### 行內元件視圖

對於非常小的元件來說，同時管理元件 Class 與元件的視圖樣板可能會覺得很麻煩。因此，我們可以從 `render` 方法中直接回傳元件的標記：

```php
/**
 * Get the view / contents that represent the component.
 */
public function render(): string
{
    return <<<'blade'
        <div class="alert alert-danger">
            {{ $slot }}
        </div>
    blade;
}
```

<a name="generating-inline-view-components"></a>
#### 產生行內視圖元件

若要建立一個會渲染行內視圖的元件，可以在執行 `make:component` 指令時使用 `inline` 選項：

```shell
php artisan make:component Alert --inline
```

<a name="dynamic-components"></a>
### 動態元件

有時候，我們可能需要渲染一個元件，但在執行前都還不知道該渲染哪個元件。在這種情況下，我們可以使用 Laravel 內建的 `dynamic-component` 元件，並根據執行期的值或變數來渲染元件：

```blade
// $componentName = "secondary-button";

<x-dynamic-component :component="$componentName" class="mt-4" />
```

<a name="manually-registering-components"></a>
### 手動註冊元件

> [!WARNING]
> 接下來這段關於手動註冊元件的文件主要適用於要撰寫包含視圖元件的 Laravel 套件的開發者。若你不是在撰寫套件，那這部分的元件文件可能與你無關。

為自己的專案撰寫元件時，元件會自動在 `app/View/Components` 目錄與 `resources/views/components` 目錄下被找到。

不過，若你在建立一個使用 Blade 元件的套件，或將元件放在非傳統的目錄下，你就需要手動註冊元件 Class 及其 HTML 標籤別名，好讓 Laravel 知道在哪裡可以找到該元件。一般來說，我們應在套件的 Service Provider 中的 `boot` 方法內註冊元件：

```php
use Illuminate\Support\Facades\Blade;
use VendorPackage\View\Components\AlertComponent;

/**
 * Bootstrap your package's services.
 */
public function boot(): void
{
    Blade::component('package-alert', AlertComponent::class);
}
```

元件註冊好後，就可以使用其標籤別名來渲染：

```blade
<x-package-alert/>
```

#### 自動載入套件元件

或者，我們也可以使用 `componentNamespace` 方法來依照慣例自動載入元件 Class。舉例來說，有個 `Nightshade` 套件，其中可能包含 `Calendar` 與 `ColorPicker` 元件，這些元件都為於 `Package\Views\Components` 命名空間下：

```php
use Illuminate\Support\Facades\Blade;

/**
 * Bootstrap your package's services.
 */
public function boot(): void
{
    Blade::componentNamespace('Nightshade\\Views\\Components', 'nightshade');
}
```

這樣一來，就可以使用 `package-name::` 語法並以 Vendor 命名空間來使用套件元件：

```blade
<x-nightshade::calendar />
<x-nightshade::color-picker />
```

Blade 會自動將元件名稱轉為 PascalCase 來偵測連結到該元件的 Class。也支援使用「點」標記法來表示子目錄。

<a name="anonymous-components"></a>
## 匿名元件

與行內元件類似，匿名元件也提供了一種能以單一檔案來管理元件的機制。不過，匿名元件只使用單一的 View 檔案，且沒有關聯的 Class。若要定義匿名元件，只需要將 Blade 樣板放到 `resources/views/components` 目錄即可。舉例來說，假設我們已在 `resources/views/components/alert.blade.php` 定義了一個元件，則可以直接像這樣渲染該元件：

```blade
<x-alert/>
```

若元件被巢狀地放在 `components` 目錄的更深層，可以使用 `.` 字元來表示。舉例來說，假設元件被定義在 `resources/views/components/inputs/button.blade.php`，則可以像這樣渲染：

```blade
<x-inputs.button/>
```

若要透過 Artisan 來建立匿名元件，可在執行 `make:component` 指令時加上 `--view` 標旗：

```shell
php artisan make:component forms.input --view
```

上述指令會在 `resources/views/components/forms/input.blade.php` 建立一個 Blade 檔案，該檔案可透過 `<x-forms.input />` 來渲染成一個元件。

<a name="anonymous-index-components"></a>
### 匿名索引元件

有時候，當一個元件由多個 Blade 樣板組成時，我們可能會想將給定元件的樣板群組在單一目錄內。舉例來說，假設有一個「accordion (摺疊)」元件，其目錄結構如下：

```text
/resources/views/components/accordion.blade.php
/resources/views/components/accordion/item.blade.php
```

這樣的目錄結構能讓我們像這樣渲染 accordion 元件與其項目：

```blade
<x-accordion>
    <x-accordion.item>
        ...
    </x-accordion.item>
</x-accordion>
```

不過，為了要能透過 `x-accordion` 渲染 accordion 元件，我們必須將「索引」accordion 元件樣板放在 `resources/views/components` 目錄下，而不是跟其他 accordion 相關的樣板一起巢狀地放在 `accordion` 目錄內。

幸好，Blade 允許我們將一個與元件目錄同名的檔案放在該元件目錄中。當有這個樣板時，即使該樣板是巢狀地放在目錄中，也可以被渲染為元件的「根」元素。因此，我們可以繼續使用上述範例中的 Blade 語法；不過，我們會像這樣調整目錄結構：

```text
/resources/views/components/accordion/accordion.blade.php
/resources/views/components/accordion/item.blade.php
```

<a name="data-properties-attributes"></a>
### 資料屬性 / Attributes

由於匿名元件沒有任何關聯的 Class，讀者可能會好奇要怎麼區分哪些資料該以變數的形式傳入元件、又有哪些屬性該被放到元件的[屬性包](#component-attributes)中。

我們可以在元件的 Blade 樣板最上方使用 `@props` 指令來指定哪些屬性該被當作資料變數。元件上所有其他的屬性都會放在元件的屬性包中。若想給資料變數一個預設值，可以將變數的名稱指定為陣列的索引鍵，並將預設值指定為陣列的值：

```blade
<!-- /resources/views/components/alert.blade.php -->

@props(['type' => 'info', 'message'])

<div {{ $attributes->merge(['class' => 'alert alert-'.$type]) }}>
    {{ $message }}
</div>
```

給定上述元件定義後，我們可以像這樣渲染該元件：

```blade
<x-alert type="error" :message="$message" class="mb-4"/>
```

<a name="accessing-parent-data"></a>
### 存取父層資料

有時候，我們可能會想在子元件內存取父層元件的資料。在這種情況下，可使用 `@aware` 指令。舉例來說，假設我們正在建構一個複雜的選單元件，其中包含父層的 `<x-menu>` 與子層的 `<x-menu.item>`：

```blade
<x-menu color="purple">
    <x-menu.item>...</x-menu.item>
    <x-menu.item>...</x-menu.item>
</x-menu>
```

`<x-menu>` 元件的實作可能如下：

```blade
<!-- /resources/views/components/menu/index.blade.php -->

@props(['color' => 'gray'])

<ul {{ $attributes->merge(['class' => 'bg-'.$color.'-200']) }}>
    {{ $slot }}
</ul>
```

因為 `color` Prop (屬性) 只被傳入父層 (`<x-menu>`)，所以 `<x-menu.item>` 中無法使用該屬性。不過，若使用 `@aware` 指令，就可以讓該屬性在 `<x-menu.item>` 中也可用：

```blade
<!-- /resources/views/components/menu/item.blade.php -->

@aware(['color' => 'gray'])

<li {{ $attributes->merge(['class' => 'text-'.$color.'-800']) }}>
    {{ $slot }}
</li>
```

> [!WARNING]
> `@aware` 指令無法存取未明確地透過 HTML 屬性傳入父元件的父層資料。`@aware` 指令無法存取未明確傳入父元件的 `@props` 預設值。

<a name="anonymous-component-paths"></a>
### 匿名元件路徑

如前所述，匿名元件通常是透過將 Blade 樣板檔放到 `resources/views/components` 目錄下來定義的。不過，有時候我們可能會想為 Laravel 註冊除了預設路徑以外的其他匿名元件路徑。

`anonymousComponentPath` 方法的第一個引數是匿名元件位置的「路徑」，而第二個可選引數則是元件所屬的「命名空間」。一般來說，應在應用程式其中一個[服務提供者](/docs/{{version}}/providers)的 `boot` 方法中呼叫此方法：

```php
/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Blade::anonymousComponentPath(__DIR__.'/../components');
}
```

當如上例般註冊元件路徑而未指定前置詞時，也可以在 Blade 元件中不加前置詞地渲染這些元件。舉例來說，若有個 `panel.blade.php` 元件存在於上述註冊的路徑中，則可以像這樣渲染：

```blade
<x-panel />
```

前置「命名空間」可作為 `anonymousComponentPath` 方法的第二個引數傳入：

```php
Blade::anonymousComponentPath(__DIR__.'/../components', 'dashboard');
```

當提供了前置詞時，該「命名空間」內的元件在渲染時可在元件名稱前加上元件的命名空間來渲染：

```blade
<x-dashboard::panel />
```

<a name="building-layouts"></a>
## 建構版面


<a name="layouts-using-components"></a>
### 使用元件的版面

大部分的 Web 應用程式都會在不同頁面間使用相同的通用版面。若我們需要在每個建立的 View 中重複整個版面 HTML，那維護我們的應用程式將會變得非常麻煩且困難。所幸，我們可以很方便地將這個版面定義成單一的 [Blade 元件](#components)，並在整個應用程式中使用它。


<a name="defining-the-layout-component"></a>
#### 定義版面元件

舉例來說，假設我們正在建構一個「待辦事項」清單應用程式。我們可以定義一個 `layout` 元件，看起來像這樣：

```blade
<!-- resources/views/components/layout.blade.php -->

<html>
    <head>
        <title>{{ $title ?? 'Todo Manager' }}</title>
    </head>
    <body>
        <h1>Todos</h1>
        <hr/>
        {{ $slot }}
    </body>
</html>
```


<a name="applying-the-layout-component"></a>
#### 套用版面元件

一旦定義了 `layout` 元件，我們就可以建立一個使用該元件的 Blade View。在這個範例中，我們將定義一個簡單的 View 來顯示我們的任務清單：

```blade
<!-- resources/views/tasks.blade.php -->

<x-layout>
    @foreach ($tasks as $task)
        <div>{{ $task }}</div>
    @endforeach
</x-layout>
```

請記住，注入到元件中的內容將會提供給 `layout` 元件中預設的 `$slot` 變數。你可能已經注意到，我們的 `layout` 也會接受一個 `$title` Slot (若有提供的話)；否則，會顯示一個預設的標題。我們可以像在[元件文件](#components)中討論過的一樣，使用標準的 Slot 語法，從我們的任務清單 View 中注入一個自訂標題：

```blade
<!-- resources/views/tasks.blade.php -->

<x-layout>
    <x-slot:title>
        Custom Title
    </x-slot>

    @foreach ($tasks as $task)
        <div>{{ $task }}</div>
    @endforeach
</x-layout>
```

現在我們已經定義好了版面與任務清單 View，我們只需要從路由回傳 `task` View 即可：

```php
use App\Models\Task;

Route::get('/tasks', function () {
    return view('tasks', ['tasks' => Task::all()]);
});
```


<a name="layouts-using-template-inheritance"></a>
### 使用樣板繼承的版面


<a name="defining-a-layout"></a>
#### 定義版面

版面也可以透過「樣板繼承 (Template Inheritance)」來建立。在[元件](#components)出現前，這是建構應用程式的主要方式。

讓我們先看一個簡單的範例。首先，我們要檢查一下頁面版面。由於大多數 Web 應用程式都會在許多頁面間使用相同的通用版面，因此，將這個版面定義為單一的 Blade View 會很方便：

```blade
<!-- resources/views/layouts/app.blade.php -->

<html>
    <head>
        <title>App Name - @yield('title')</title>
    </head>
    <body>
        @section('sidebar')
            This is the master sidebar.
        @show

        <div class="container">
            @yield('content')
        </div>
    </body>
</html>
```

如你所見，這個檔案包含典型的 HTML 標記。不過，請注意 `@section` 與 `@yield` 指令。`@section` 指令，顧名思義，是用來定義一個內容的區塊 (Section)；而 `@yield` 指令則是用來顯示給定區塊的內容。

現在我們已經為我們的應用程式定義好了一個版面，接著讓我們來定義一個繼承該版面的子頁面。


<a name="extending-a-layout"></a>
#### 擴充版面

在定義子 View 時，請使用 `@extends` Blade 指令來指定該子 View 要「繼承」哪個版面。擴充 Blade 版面的 View 可以使用 `@section` 指令將內容注入版面的區塊中。請記住，如上例所見，這些區塊的內容會使用 `@yield` 顯示在版面中：

```blade
<!-- resources/views/child.blade.php -->

@extends('layouts.app')

@section('title', 'Page Title')

@section('sidebar')
    @@parent

    <p>This is appended to the master sidebar.</p>
@endsection

@section('content')
    <p>This is my body content.</p>
@endsection
```

在這個範例中，`sidebar` 區塊使用了 `@@parent` 指令來將內容附加 (而不是覆寫) 到版面的側邊欄。在渲染 View 時，`@@parent` 指令會被版面的內容給取代。

> [!NOTE]
> 與上一個範例不同，這個 `sidebar` 區塊是以 `@endsection` 而不是 `@show` 結尾。`@endsection` 指令只會定義一個區塊，而 `@show` 則會定義並**立即 Yield** 該區塊。

`@yield` 指令也接受一個預設值作為其第二個參數。若要被 Yield 的區塊未定義，則會渲染這個值：

```blade
@yield('content', 'Default content')
```


<a name="forms"></a>
## 表單


<a name="csrf-field"></a>
### CSRF 欄位

在應用程式中定義 HTML 表單時，都應在表單中包含一個隱藏的 CSRF Token 欄位，這樣[CSRF 保護](/docs/{{version}}/csrf)中介層才能驗證該請求。我們可以使用 `@csrf` Blade 指令來產生 Token 欄位：

```blade
<form method="POST" action="/profile">
    @csrf

    ...
</form>
```


<a name="method-field"></a>
### 方法欄位

由於 HTML 表單不能發出 `PUT`、`PATCH` 或 `DELETE` 請求，因此需要加上一個隱藏的 `_method` 欄位來偽造這些 HTTP 動詞。`@method` Blade 指令可以為我們建立這個欄位：

```blade
<form action="/foo/bar" method="POST">
    @method('PUT')

    ...
</form>
```


<a name="validation-errors"></a>
### 驗證錯誤

`@error` 指令可用來快速檢查給定屬性是否存在[驗證錯誤訊息](/docs/{{version}}/validation#quick-displaying-the-validation-errors)。在 `@error` 指令中，可以 Echo `$message` 變數來顯示錯誤訊息：

```blade
<!-- /resources/views/post/create.blade.php -->

<label for="title">Post Title</label>

<input
    id="title"
    type="text"
    class="@error('title') is-invalid @enderror"
/>

@error('title')
    <div class="alert alert-danger">{{ $message }}</div>
@enderror
```

因為 `@error` 指令會被編譯成一個「if」陳述式，所以我們可以使用 `@else` 指令來在屬性沒有錯誤時渲染內容：

```blade
<!-- /resources/views/auth.blade.php -->

<label for="email">Email address</label>

<input
    id="email"
    type="email"
    class="@error('email') is-invalid @else is-valid @enderror"
/>
```

可以將[特定錯誤包 (Error Bag) 的名稱](/docs/{{version}}/validation#named-error-bags)作為 `@error` 指令的第二個參數傳入，以在包含多個表單的頁面上取得驗證錯誤訊息：

```blade
<!-- /resources/views/auth.blade.php -->

<label for="email">Email address</label>

<input
    id="email"
    type="email"
    class="@error('email', 'login') is-invalid @enderror"
/>

@error('email', 'login')
    <div class="alert alert-danger">{{ $message }}</div>
@enderror
```


<a name="stacks"></a>
## Stack

Blade 可讓開發者將內容推送至具名的 Stack 中，而這些 Stack 可在另一個 View 或版面中的其他地方渲染。這個功能對於指定子 View 所需的 JavaScript 函式庫特別有用：

```blade
@push('scripts')
    <script src="/example.js"></script>
@endpush
```

若想在給定的布林表達式為 `true` 時才 `@push` 內容，可以使用 `@pushIf` 指令：

```blade
@pushIf($shouldPush, 'scripts')
    <script src="/example.js"></script>
@endPushIf
```

可以視需要多次推送至 Stack。若要渲染完整的 Stack 內容，請將 Stack 的名稱傳給 `@stack` 指令：

```blade
<head>
    <!-- Head Contents -->

    @stack('scripts')
</head>
```

若想將內容加到 Stack 的開頭，應使用 `@prepend` 指令：

```blade
@push('scripts')
    This will be second...
@endpush

// Later...

@prepend('scripts')
    This will be first...
@endprepend
```

<a name="service-injection"></a>
## 服務注入

`@inject` 指令可用於從 Laravel [服務容器](/docs/{{version}}/container) 中取得服務。傳遞給 `@inject` 的第一個引數是要置入服務的變數名稱，而第二個引數是您希望解析的服務的類別或介面名稱：

```blade
@inject('metrics', 'App\Services\MetricsService')

<div>
    Monthly Revenue: {{ $metrics->monthlyRevenue() }}.
</div>
```


<a name="rendering-inline-blade-templates"></a>
## 渲染行內 Blade 樣板

有時您可能需要將原始的 Blade 樣板字串轉換為有效的 HTML。您可以使用 `Blade` facade 提供的 `render` 方法來完成此操作。`render` 方法接受 Blade 樣板字串和一個可選的資料陣列以提供給樣板：

```php
use Illuminate\Support\Facades\Blade;

return Blade::render('Hello, {{ $name }}', ['name' => 'Julian Bashir']);
```

Laravel 透過將行內 Blade 樣板寫入 `storage/framework/views` 目錄來渲染它們。如果您希望 Laravel 在渲染 Blade 樣板後移除這些暫存檔案，您可以提供 `deleteCachedView` 引數給該方法：

```php
return Blade::render(
    'Hello, {{ $name }}',
    ['name' => 'Julian Bashir'],
    deleteCachedView: true
);
```


<a name="rendering-blade-fragments"></a>
## 渲染 Blade 片段

在使用如 [Turbo](https://turbo.hotwired.dev/) 和 [htmx](https://htmx.org/) 等前端框架時，您可能偶爾需要在 HTTP 回應中只回傳 Blade 樣板的一部分。Blade 的「片段 (fragments)」功能正能讓您做到這一點。首先，將 Blade 樣板的一部分放在 `@fragment` 和 `@endfragment` 指令中：

```blade
@fragment('user-list')
    <ul>
        @foreach ($users as $user)
            <li>{{ $user->name }}</li>
        @endforeach
    </ul>
@endfragment
```

然後，在渲染使用此樣板的視圖時，您可以呼叫 `fragment` 方法來指定只有指定的片段應包含在送出的 HTTP 回應中：

```php
return view('dashboard', ['users' => $users])->fragment('user-list');
```

`fragmentIf` 方法允許您根據給定條件有條件地回傳視圖的片段。否則，將回傳整個視圖：

```php
return view('dashboard', ['users' => $users])
    ->fragmentIf($request->hasHeader('HX-Request'), 'user-list');
```

`fragments` 和 `fragmentsIf` 方法允許您在回應中回傳多個視圖片段。這些片段將被串接在一起：

```php
view('dashboard', ['users' => $users])
    ->fragments(['user-list', 'comment-list']);

view('dashboard', ['users' => $users])
    ->fragmentsIf(
        $request->hasHeader('HX-Request'),
        ['user-list', 'comment-list']
    );
```


<a name="extending-blade"></a>
## 擴充 Blade

Blade 允許您使用 `directive` 方法定義自己的自訂指令。當 Blade 編譯器遇到自訂指令時，它會呼叫提供的回呼函式，並帶入該指令包含的表達式。

以下範例建立了一個 `@datetime($var)` 指令，用於格式化給定的 `$var`，此變數應為 `DateTime` 的一個實體：

```php
<?php

namespace App\Providers;

use Illuminate\Support\Facades\Blade;
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
        Blade::directive('datetime', function (string $expression) {
            return "<?php echo ($expression)->format('m/d/Y H:i'); ?>";
        });
    }
}
```

如您所見，我們將 `format` 方法鏈式呼叫到傳遞給指令的任何表達式上。因此，在此範例中，此指令產生的最終 PHP 將是：

```php
<?php echo ($var)->format('m/d/Y H:i'); ?>
```

> [!WARNING]
> 更新 Blade 指令的邏輯後，您需要刪除所有快取的 Blade 視圖。快取的 Blade 視圖可以使用 `view:clear` Artisan 指令來移除。


<a name="custom-echo-handlers"></a>
### 自訂 Echo 處理常式

如果您嘗試使用 Blade「echo」一個物件，將會呼叫該物件的 `__toString` 方法。[__toString](https://www.php.net/manual/en/language.oop5.magic.php#object.tostring) 方法是 PHP 的內建「魔術方法」之一。然而，有時您可能無法控制給定類別的 `__toString` 方法，例如當您互動的類別屬於第三方函式庫時。

在這些情況下，Blade 允許您為該特定類型的物件註冊一個自訂的 echo 處理常式。為此，您應該呼叫 Blade 的 `stringable` 方法。`stringable` 方法接受一個閉包。此閉包應對其負責渲染的物件類型進行型別提示。通常，`stringable` 方法應在您應用程式的 `AppServiceProvider` 類別的 `boot` 方法中呼叫：

```php
use Illuminate\Support\Facades\Blade;
use Money\Money;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Blade::stringable(function (Money $money) {
        return $money->formatTo('en_GB');
    });
}
```

一旦定義了您的自訂 echo 處理常式，您就可以簡單地在 Blade 樣板中 echo 該物件：

```blade
Cost: {{ $money }}
```


<a name="custom-if-statements"></a>
### 自訂 If 陳述式

在定義簡單的自訂條件陳述式時，編寫一個自訂指令有時比必要的更複雜。因此，Blade 提供了 `Blade::if` 方法，讓您能使用閉包快速定義自訂條件指令。例如，讓我們定義一個自訂條件，用於檢查應用程式設定的預設「磁碟 (disk)」。我們可以在 `AppServiceProvider` 的 `boot` 方法中執行此操作：

```php
use Illuminate\Support\Facades\Blade;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Blade::if('disk', function (string $value) {
        return config('filesystems.default') === $value;
    });
}
```

一旦定義了自訂條件，您就可以在樣板中使用它：

```blade
@disk('local')
    <!-- The application is using the local disk... -->
@elsedisk('s3')
    <!-- The application is using the s3 disk... -->
@else
    <!-- The application is using some other disk... -->
@enddisk

@unlessdisk('local')
    <!-- The application is not using the local disk... -->
@enddisk
```