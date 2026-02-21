# Blade 模板

- [簡介](#introduction)
    - [使用 Livewire 強化 Blade](#supercharging-blade-with-livewire)
- [顯示資料](#displaying-data)
    - [HTML 實體編碼](#html-entity-encoding)
    - [Blade 與 JavaScript 框架](#blade-and-javascript-frameworks)
- [Blade 指令](#blade-directives)
    - [If 語句](#if-statements)
    - [Switch 語句](#switch-statements)
    - [迴圈](#loops)
    - [Loop 變數](#the-loop-variable)
    - [條件式 Class](#conditional-classes)
    - [額外屬性](#additional-attributes)
    - [引入子視圖](#including-subviews)
    - [The `@once` 指令](#the-once-directive)
    - [原生 PHP](#raw-php)
    - [註解](#comments)
- [元件](#components)
    - [渲染元件](#rendering-components)
    - [索引元件](#index-components)
    - [傳遞資料給元件](#passing-data-to-components)
    - [元件屬性](#component-attributes)
    - [保留關鍵字](#reserved-keywords)
    - [插槽 (Slots)](#slots)
    - [行內元件視圖](#inline-component-views)
    - [動態元件](#dynamic-components)
    - [手動註冊元件](#manually-registering-components)
- [匿名元件](#anonymous-components)
    - [匿名索引元件](#anonymous-index-components)
    - [資料屬性 / 特性](#data-properties-attributes)
    - [存取父層資料](#accessing-parent-data)
    - [匿名元件路徑](#anonymous-component-paths)
- [建構佈局](#building-layouts)
    - [使用元件建立佈局](#layouts-using-components)
    - [使用模板繼承建立佈局](#layouts-using-template-inheritance)
- [表單](#forms)
    - [CSRF 欄位](#csrf-field)
    - [方法欄位](#method-field)
    - [驗證錯誤](#validation-errors)
- [堆疊 (Stacks)](#stacks)
- [服務注入](#service-injection)
- [渲染行內 Blade 模板](#rendering-inline-blade-templates)
- [渲染 Blade 片段](#rendering-blade-fragments)
- [擴充 Blade](#extending-blade)
    - [自定義 Echo 處理器](#custom-echo-handlers)
    - [自定義 If 語句](#custom-if-statements)

<a name="introduction"></a>
## 簡介

Blade 是 Laravel 內建的一個簡單但強大的模板引擎。與某些 PHP 模板引擎不同，Blade 並不限制你在模板中使用純 PHP 程式碼。事實上，所有的 Blade 模板都會被編譯成純 PHP 程式碼並快取，直到它們被修改為止，這意味著 Blade 基本上不會為你的應用程式增加任何額外負擔。Blade 模板檔案使用 `.blade.php` 副檔名，通常存放在 `resources/views` 目錄中。

可以使用全域的 `view` 輔助函式從路由或控制器回傳 Blade 視圖。當然，正如 [views](/docs/{{version}}/views) 文件中所提到的，可以透過 `view` 輔助函式的第二個參數將資料傳遞給 Blade 視圖：

```php
Route::get('/', function () {
    return view('greeting', ['name' => 'Finn']);
});
```


<a name="supercharging-blade-with-livewire"></a>
### 使用 Livewire 強化 Blade

想讓你的 Blade 模板更上一層樓並輕鬆構建動態介面嗎？請參考 [Laravel Livewire](https://livewire.laravel.com)。Livewire 讓你可以編寫具有動態功能的 Blade 元件，這些功能通常只能透過 React 或 Vue 等前端框架實現，它提供了一種構建現代化、反應式前端的絕佳方法，且無需處理許多 JavaScript 框架的複雜性、客戶端渲染或建置步驟。


<a name="displaying-data"></a>
## 顯示資料

你可以透過將變數包在花括號中來顯示傳遞給 Blade 視圖的資料。例如，給定以下路由：

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
> Blade 的 `{{ }}` 印出語句會自動通過 PHP 的 `htmlspecialchars` 函式，以防止 XSS 攻擊。

你不僅限於顯示傳遞給視圖的變數內容。你也可以印出任何 PHP 函式的結果。事實上，你可以在 Blade 印出語句中放入任何你想要的 PHP 程式碼：

```blade
The current UNIX timestamp is {{ time() }}.
```


<a name="html-entity-encoding"></a>
### HTML 實體編碼

預設情況下，Blade (以及 Laravel 的 `e` 函式) 會對 HTML 實體進行雙重編碼。如果你想停用雙重編碼，請在 `AppServiceProvider` 的 `boot` 方法中呼叫 `Blade::withoutDoubleEncoding` 方法：

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
#### 顯示未經轉義的資料

預設情況下，Blade `{{ }}` 語句會自動通過 PHP 的 `htmlspecialchars` 函式，以防止 XSS 攻擊。如果你不希望資料被轉義，可以使用以下語法：

```blade
Hello, {!! $name !!}.
```

> [!WARNING]
> 在印出應用程式使用者提供的內容時要非常小心。顯示使用者提供的資料時，通常應使用轉義過的雙花括號語法來防止 XSS 攻擊。


<a name="blade-and-javascript-frameworks"></a>
### Blade 與 JavaScript 框架

由於許多 JavaScript 框架也使用「花括號」來表示應在瀏覽器中顯示的給定表達式，因此你可以使用 `@` 符號來通知 Blade 渲染引擎該表達式應保持原樣。例如：

```blade
<h1>Laravel</h1>

Hello, @{{ name }}.
```

在此範例中，`@` 符號會被 Blade 移除；然而，`{{ name }}` 表達式將保持原樣不被 Blade 引擎處理，從而允許它由你的 JavaScript 框架進行渲染。

`@` 符號也可以用來轉義 Blade 指令：

```blade
{{-- Blade template --}}
@@if()

<!-- HTML output -->
@if()
```


<a name="rendering-json"></a>
#### 渲染 JSON

有時你可能會向視圖傳遞一個陣列，目的是將其渲染為 JSON 以便初始化 JavaScript 變數。例如：

```php
<script>
    var app = <?php echo json_encode($array); ?>;
</script>
```

然而，與其手動呼叫 `json_encode`，你可以改用 `Illuminate\Support\Js::from` 方法。`from` 方法接受與 PHP 的 `json_encode` 函式相同的參數；然而，它會確保產生的 JSON 已針對 HTML 引號內的包含進行了正確的轉義。`from` 方法將回傳一個 `JSON.parse` JavaScript 語句字串，該語句會將給定的物件或陣列轉換為有效的 JavaScript 物件：

```blade
<script>
    var app = {{ Illuminate\Support\Js::from($array) }};
</script>
```

最新版本的 Laravel 應用程式骨架包含一個 `Js` Facade，它在你的 Blade 模板中提供了對此功能的便捷存取：

```blade
<script>
    var app = {{ Js::from($array) }};
</script>
```

> [!WARNING]
> 你應該只使用 `Js::from` 方法來將現有變數渲染為 JSON。Blade 模板是基於正規表示式的，嘗試將複雜表達式傳遞給該指令可能會導致預料之外的失敗。


<a name="the-at-verbatim-directive"></a>
#### The `@verbatim` 指令

如果你在模板的大部分區域中顯示 JavaScript 變數，可以將 HTML 包在 `@verbatim` 指令中，這樣你就不必在每個 Blade 印出語句前都加上 `@` 符號：

```blade
@verbatim
    <div class="container">
        Hello, {{ name }}.
    </div>
@endverbatim
```

<a name="blade-directives"></a>
## Blade 指令

除了模板繼承和顯示資料之外，Blade 還為常見的 PHP 控制結構（例如條件語句和迴圈）提供了方便的捷徑。這些捷徑提供了一種非常乾淨、簡潔的方式來處理 PHP 控制結構，同時也保留了與 PHP 對應部分的相似性。


<a name="if-statements"></a>
### If 語句

您可以使用 `@if`、`@elseif`、`@else` 和 `@endif` 指令來建構 `if` 語句。這些指令的功能與 PHP 的對應部分完全相同：

```blade
@if (count($records) === 1)
    I have one record!
@elseif (count($records) > 1)
    I have multiple records!
@else
    I don't have any records!
@endif
```

為了方便起見，Blade 還提供了一個 `@unless` 指令：

```blade
@unless (Auth::check())
    You are not signed in.
@endunless
```

除了已經討論過的條件指令外，`@isset` 和 `@empty` 指令可以作為其對應 PHP 函式的方便捷徑：

```blade
@isset($records)
    // $records is defined and is not null...
@endisset

@empty($records)
    // $records is "empty"...
@endempty
```


<a name="authentication-directives"></a>
#### 驗證指令

`@auth` 和 `@guest` 指令可用於快速判斷當前使用者是否已[通過驗證](/docs/{{version}}/authentication)或者是訪客：

```blade
@auth
    // The user is authenticated...
@endauth

@guest
    // The user is not authenticated...
@endguest
```

如果需要，您可以在使用 `@auth` 和 `@guest` 指令時指定應檢查的認證守衛 (Guard)：

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

您可以使用 `@production` 指令來檢查應用程式是否正在生產環境 (Production Environment) 中執行：

```blade
@production
    // Production specific content...
@endproduction
```

或者，您可以使用 `@env` 指令來判斷應用程式是否在特定環境中執行：

```blade
@env('staging')
    // The application is running in "staging"...
@endenv

@env(['staging', 'production'])
    // The application is running in "staging" or "production"...
@endenv
```


<a name="section-directives"></a>
#### 區段指令

您可以使用 `@hasSection` 指令來判斷模板繼承的區段 (Section) 是否含有內容：

```blade
@hasSection('navigation')
    <div class="pull-right">
        @yield('navigation')
    </div>

    <div class="clearfix"></div>
@endif
```

您可以使用 `sectionMissing` 指令來判斷區段是否沒有內容：

```blade
@sectionMissing('navigation')
    <div class="pull-right">
        @include('default-navigation')
    </div>
@endif
```


<a name="session-directives"></a>
#### Session 指令

`@session` 指令可用於判斷 [session](/docs/{{version}}/session) 值是否存在。如果 session 值存在，則會評估 `@session` 和 `@endsession` 指令中的模板內容。在 `@session` 指令的內容中，您可以印出 `$value` 變數來顯示 session 的值：

```blade
@session('status')
    <div class="p-4 bg-green-100">
        {{ $value }}
    </div>
@endsession
```


<a name="context-directives"></a>
#### Context 指令

`@context` 指令可用於判斷 [context](/docs/{{version}}/context) 值是否存在。如果 context 值存在，則會評估 `@context` 和 `@endcontext` 指令中的模板內容。在 `@context` 指令的內容中，您可以印出 `$value` 變數來顯示 context 的值：

```blade
@context('canonical')
    <link href="{{ $value }}" rel="canonical">
@endcontext
```


<a name="switch-statements"></a>
### Switch 語句

可以使用 `@switch`、`@case`、`@break`、`@default` 和 `@endswitch` 指令來建構 Switch 語句：

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

除了條件語句之外，Blade 還提供了用於處理 PHP 迴圈結構的簡單指令。同樣地，這些指令中的每一個功能都與其 PHP 對應部分完全相同：

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
> 在 `foreach` 迴圈迭代時，您可以使用 [loop 變數](#the-loop-variable)來獲取有關迴圈的有用資訊，例如您正處於迴圈的第一次還是最後一次迭代。

使用迴圈時，您還可以使用 `@continue` 和 `@break` 指令跳過當前迭代或結束迴圈：

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

您也可以在指令宣告中包含跳過或中斷的條件：

```blade
@foreach ($users as $user)
    @continue($user->type == 1)

    <li>{{ $user->name }}</li>

    @break($user->number == 5)
@endforeach
```


<a name="the-loop-variable"></a>
### Loop 變數

在 `foreach` 迴圈迭代時，您的迴圈內部將可以使用 `$loop` 變數。此變數提供了存取一些有用資訊的能力，例如當前迴圈的索引，以及這是否為迴圈的第一次或最後一次迭代：

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

如果您處於嵌套迴圈中，可以透過 `parent` 屬性存取父層迴圈的 `$loop` 變數：

```blade
@foreach ($users as $user)
    @foreach ($user->posts as $post)
        @if ($loop->parent->first)
            This is the first iteration of the parent loop.
        @endif
    @endforeach
@endforeach
```

`$loop` 變數還包含多種其他有用的屬性：

<div class="overflow-auto">

| 屬性 | 描述 |
| ------------------ | ------------------------------------------------------ |
| `$loop->index`     | 當前迴圈迭代的索引（從 0 開始）。 |
| `$loop->iteration` | 當前迴圈的迭代次數（從 1 開始）。 |
| `$loop->remaining` | 迴圈中剩餘的迭代次數。 |
| `$loop->count`     | 正在迭代的陣列中的項目總數。 |
| `$loop->first`     | 是否為迴圈的第一次迭代。 |
| `$loop->last`      | 是否為迴圈的最後一次迭代。 |
| `$loop->even`      | 是否為迴圈的偶數次迭代。 |
| `$loop->odd`       | 是否為迴圈的奇數次迭代。 |
| `$loop->depth`     | 當前迴圈的嵌套層級。 |
| `$loop->parent`    | 處於嵌套迴圈時，父層的迴圈變數。 |

</div>

<a name="conditional-classes"></a>
### 條件式 Class

`@class` 指令條件式地編譯 CSS class 字串。該指令接受一個 class 陣列，其中陣列的鍵包含您希望新增的一個或多個 class，而值則是一個布林運算式。如果陣列元素具有數字鍵，它將始終包含在渲染後的 class 列表中：

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

同樣地，`@style` 指令可用於條件式地將行內 CSS 樣式新增到 HTML 元素中：

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
### 額外屬性

為了方便起見，您可以使用 `@checked` 指令輕鬆地指示指定的 HTML checkbox 輸入框是否為「checked」。如果提供的條件評估為 `true`，此指令將印出 `checked`：

```blade
<input
    type="checkbox"
    name="active"
    value="active"
    @checked(old('active', $user->active))
/>
```

同樣地，`@selected` 指令可用於指示指定的 select 選項是否應該被「selected」：

```blade
<select name="version">
    @foreach ($product->versions as $version)
        <option value="{{ $version }}" @selected(old('version') == $version)>
            {{ $version }}
        </option>
    @endforeach
</select>
```

此外，`@disabled` 指令可用於指示指定的元素是否應該被「disabled」：

```blade
<button type="submit" @disabled($errors->isNotEmpty())>Submit</button>
```

此外，`@readonly` 指令可用於指示指定的元素是否應該被「readonly」：

```blade
<input
    type="email"
    name="email"
    value="email@laravel.com"
    @readonly($user->isNotAdmin())
/>
```

此外，`@required` 指令可用於指示指定的元素是否應該被「required」：

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
> 雖然您可以隨意使用 `@include` 指令，但 Blade [元件](#components) 提供了類似的功能，並且在資料和屬性綁定等方面比 `@include` 指令更具優勢。

Blade 的 `@include` 指令允許您在一個視圖中引入另一個 Blade 視圖。父視圖中可用的所有變數都將在被引入的視圖中可用：

```blade
<div>
    @include('shared.errors')

    <form>
        <!-- Form Contents -->
    </form>
</div>
```

儘管被引入的視圖將繼承父視圖中可用的所有資料，但您也可以傳遞一個應在被引入視圖中可用的額外資料陣列：

```blade
@include('view.name', ['status' => 'complete'])
```

如果您嘗試 `@include` 一個不存在的視圖，Laravel 將拋出錯誤。如果您想引入一個可能存在也可能不存在的視圖，應該使用 `@includeIf` 指令：

```blade
@includeIf('view.name', ['status' => 'complete'])
```

如果您想在給定的布林運算式評估為 `true` 或 `false` 時 `@include` 視圖，可以使用 `@includeWhen` 和 `@includeUnless` 指令：

```blade
@includeWhen($boolean, 'view.name', ['status' => 'complete'])

@includeUnless($boolean, 'view.name', ['status' => 'complete'])
```

要從給定的視圖陣列中引入第一個存在的視圖，可以使用 `includeFirst` 指令：

```blade
@includeFirst(['custom.admin', 'admin'], ['status' => 'complete'])
```

如果您想引入一個視圖而不繼承父視圖的任何變數，可以使用 `@includeIsolated` 指令。被引入的視圖將只能存取您明確傳遞的變數：

```blade
@includeIsolated('view.name', ['user' => $user])
```

> [!WARNING]
> 您應該避免在 Blade 視圖中使用 `__DIR__` 和 `__FILE__` 常數，因為它們將指向快取的、編譯後的視圖位置。


<a name="rendering-views-for-collections"></a>
#### 為集合渲染視圖

您可以使用 Blade 的 `@each` 指令將迴圈和引入合併為一行：

```blade
@each('view.name', $jobs, 'job')
```

`@each` 指令的第一個參數是為陣列或集合中的每個元素渲染的視圖。第二個參數是您希望迭代的陣列或集合，而第三個參數是在視圖中分配給當前迭代的變數名稱。例如，如果您正在迭代一個 `jobs` 陣列，通常您會希望在視圖中將每個工作作為一個 `job` 變數來存取。當前迭代的陣列鍵將在視圖中作為 `key` 變數可用。

您也可以向 `@each` 指令傳遞第四個參數。此參數定義了在給定陣列為空時將渲染的視圖。

```blade
@each('view.name', $jobs, 'job', 'view.empty')
```

> [!WARNING]
> 透過 `@each` 渲染的視圖不會繼承父視圖的變數。如果子視圖需要這些變數，則應改用 `@foreach` 和 `@include` 指令。


<a name="the-once-directive"></a>
### The `@once` 指令

`@once` 指令允許您定義模板中僅在每個渲染週期評估一次的部分。這對於使用 [堆疊 (Stacks)](#stacks) 將一段特定的 JavaScript 推入頁面頁首非常有用。例如，如果您在迴圈中渲染一個指定的 [元件](#components)，您可能只想在元件第一次渲染時將 JavaScript 推入頁首：

```blade
@once
    @push('scripts')
        <script>
            // Your custom JavaScript...
        </script>
    @endpush
@endonce
```

由於 `@once` 指令經常與 `@push` 或 `@prepend` 指令結合使用，因此為了您的方便，提供了 `@pushOnce` 和 `@prependOnce` 指令：

```blade
@pushOnce('scripts')
    <script>
        // Your custom JavaScript...
    </script>
@endPushOnce
```

如果您從兩個獨立的 Blade 模板推送重複的內容，則應向 `@pushOnce` 指令的第二個參數提供一個唯一的識別碼，以確保內容僅被渲染一次：

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

在某些情況下，將 PHP 程式碼嵌入到視圖中是非常有用的。您可以使用 Blade 的 `@php` 指令在模板中執行一段純 PHP 程式碼：

```blade
@php
    $counter = 1;
@endphp
```

或者，如果您只需要使用 PHP 匯入類別，您可以使用 `@use` 指令：

```blade
@use('App\Models\Flight')
```

可以為 `@use` 指令提供第二個參數，以為匯入的類別設定別名：

```blade
@use('App\Models\Flight', 'FlightModel')
```

如果您在同一個命名空間下有多個類別，您可以將這些類別的匯入進行分組：

```blade
@use('App\Models\{Flight, Airport}')
```

`@use` 指令也支援透過在匯入路徑前加上 `function` 或 `const` 修飾詞來匯入 PHP 函式與常數：

```blade
@use(function App\Helpers\format_currency)
@use(const App\Constants\MAX_ATTEMPTS)
```

就像類別匯入一樣，函式與常數也支援別名：

```blade
@use(function App\Helpers\format_currency, 'formatMoney')
@use(const App\Constants\MAX_ATTEMPTS, 'MAX_TRIES')
```

函式與常數修飾詞也支援分組匯入，讓您能在單個指令中從同一個命名空間匯入多個符號：

```blade
@use(function App\Helpers\{format_currency, format_date})
@use(const App\Constants\{MAX_ATTEMPTS, DEFAULT_TIMEOUT})
```


<a name="comments"></a>
### 註解

Blade 也允許您在視圖中定義註解。然而，與 HTML 註解不同，Blade 註解不會被包含在應用程式回傳的 HTML 中：

```blade
{{-- This comment will not be present in the rendered HTML --}}
```

<a name="components"></a>
## 元件

元件與插槽提供與區塊、佈局以及引入類似的好處；然而，有些人可能會覺得元件與插槽的思維模型更容易理解。編寫元件有兩種方法：類別型元件與匿名元件。

若要建立類別型元件，你可以使用 `make:component` Artisan 指令。為了說明如何使用元件，我們將建立一個簡單的 `Alert` 元件。`make:component` 指令會將元件放置在 `app/View/Components` 目錄中：

```shell
php artisan make:component Alert
```

`make:component` 指令也會為該元件建立一個視圖模板。該視圖將放置在 `resources/views/components` 目錄中。在為你自己的應用程式編寫元件時，元件會自動在 `app/View/Components` 目錄與 `resources/views/components` 目錄中被偵測，因此通常不需要進一步的元件註冊。

你也可以在子目錄中建立元件：

```shell
php artisan make:component Forms/Input
```

上述指令將在 `app/View/Components/Forms` 目錄中建立一個 `Input` 元件，並將視圖放置在 `resources/views/components/forms` 目錄中。


<a name="manually-registering-package-components"></a>
#### 手動註冊套件元件

當為你自己的應用程式編寫元件時，元件會自動在 `app/View/Components` 目錄與 `resources/views/components` 目錄中被偵測。

然而，如果你正在開發一個使用 Blade 元件的套件，則需要手動註冊元件類別及其 HTML 標籤別名。你通常應該在套件的服務提供者的 `boot` 方法中註冊元件：

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

一旦元件註冊完成，就可以使用其標籤別名來渲染：

```blade
<x-package-alert/>
```

或者，你可以使用 `componentNamespace` 方法按照慣例自動載入元件類別。例如，一個 `Nightshade` 套件可能有名為 `Calendar` 和 `ColorPicker` 的元件，它們位於 `Package\Views\Components` 命名空間中：

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

這將允許透過 `package-name::` 語法，使用其供應商命名空間來使用套件元件：

```blade
<x-nightshade::calendar />
<x-nightshade::color-picker />
```

Blade 會透過將元件名稱轉換為 PascalCase 來自動偵測與此元件連結的類別。也支援使用「點」號表示法來指定子目錄。


<a name="rendering-components"></a>
### 渲染元件

若要顯示元件，你可以在 Blade 模板中使用 Blade 元件標籤。Blade 元件標籤以字串 `x-` 開頭，後面接著元件類別的 kebab-case 名稱：

```blade
<x-alert/>

<x-user-profile/>
```

如果元件類別巢狀於 `app/View/Components` 目錄深處，你可以使用 `.` 字元來表示目錄巢狀。例如，假設一個元件位於 `app/View/Components/Inputs/Button.php`，我們可以這樣渲染它：

```blade
<x-inputs.button/>
```

如果你想條件式地渲染元件，可以在元件類別中定義 `shouldRender` 方法。如果 `shouldRender` 方法回傳 `false`，則元件將不會被渲染：

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

有時元件是元件群組的一部分，你可能希望將相關元件分組在單個目錄中。例如，想像一個具有以下類別結構的 "card" 元件：

```text
App\Views\Components\Card\Card
App\Views\Components\Card\Header
App\Views\Components\Card\Body
```

由於根 `Card` 元件巢狀在 `Card` 目錄中，你可能會認為需要透過 `<x-card.card>` 來渲染該元件。然而，當元件的檔案名稱與元件的目錄名稱匹配時，Laravel 會自動假設該元件是「根」元件，並允許你在不重複目錄名稱的情況下渲染該元件：

```blade
<x-card>
    <x-card.header>...</x-card.header>
    <x-card.body>...</x-card.body>
</x-card>
```

<a name="passing-data-to-components"></a>
### 傳遞資料給元件

您可以使用 HTML 屬性將資料傳遞給 Blade 元件。硬編碼的原始值可以使用簡單的 HTML 屬性字串傳遞給元件。PHP 表達式和變數應該透過使用 `:` 字元作為前綴的屬性傳遞給元件：

```blade
<x-alert type="error" :message="$message"/>
```

您應該在元件類別的建構函式中定義所有元件的資料屬性。元件上的所有公開屬性都將自動提供給元件的視圖。不需要從元件的 `render` 方法將資料傳遞到視圖：

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

當您的元件被渲染時，您可以透過名稱印出變數來顯示元件公開變數的內容：

```blade
<div class="alert alert-{{ $type }}">
    {{ $message }}
</div>
```

<a name="casing"></a>
#### 命名規範 (Casing)

元件建構函式的參數應使用 `camelCase` 指定，而在 HTML 屬性中引用參數名稱時應使用 `kebab-case`。例如，給定以下元件建構函式：

```php
/**
 * Create the component instance.
 */
public function __construct(
    public string $alertType,
) {}
```

可以像這樣將 `$alertType` 參數提供給元件：

```blade
<x-alert alert-type="danger" />
```

<a name="short-attribute-syntax"></a>
#### 短屬性語法

將屬性傳遞給元件時，您也可以使用「短屬性」語法。這通常很方便，因為屬性名稱通常與它們對應的變數名稱一致：

```blade
{{-- Short attribute syntax... --}}
<x-profile :$userId :$name />

{{-- Is equivalent to... --}}
<x-profile :user-id="$userId" :name="$name" />
```

<a name="escaping-attribute-rendering"></a>
#### 轉義屬性渲染

由於某些 JavaScript 框架（例如 Alpine.js）也使用冒號前綴的屬性，您可以使用雙冒號 (`::`) 前綴來告知 Blade 該屬性不是 PHP 表達式。例如，給定以下元件：

```blade
<x-button ::class="{ danger: isDeleting }">
    Submit
</x-button>
```

以下 HTML 將由 Blade 渲染：

```blade
<button :class="{ danger: isDeleting }">
    Submit
</button>
```

<a name="component-methods"></a>
#### 元件方法

除了公開變數可用於元件模板外，元件上的任何公開方法也可以被呼叫。例如，假設一個元件有一個 `isSelected` 方法：

```php
/**
 * Determine if the given option is the currently selected option.
 */
public function isSelected(string $option): bool
{
    return $option === $this->selected;
}
```

您可以從元件模板中呼叫與該方法名稱相符的變數來執行此方法：

```blade
<option {{ $isSelected($value) ? 'selected' : '' }} value="{{ $value }}">
    {{ $label }}
</option>
```

<a name="using-attributes-slots-within-component-class"></a>
#### 在元件類別中存取屬性與插槽

Blade 元件也允許您在類別的 render 方法中存取元件名稱、屬性與插槽。然而，為了存取這些資料，您應該從元件的 `render` 方法中回傳一個閉包 (Closure)：

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

由元件的 `render` 方法回傳的閉包也可以接收一個 `$data` 陣列作為其唯一的參數。此陣列將包含幾個提供元件資訊的元素：

```php
return function (array $data) {
    // $data['componentName'];
    // $data['attributes'];
    // $data['slot'];

    return '<div {{ $attributes }}>Components content</div>';
}
```

> [!WARNING]
> `$data` 陣列中的元素絕不應直接嵌入到 `render` 方法回傳的 Blade 字串中，因為這樣做可能會透過惡意的屬性內容導致遠端程式碼執行。

`componentName` 等於在 `x-` 前綴後的 HTML 標籤中所使用的名稱。因此 `<x-alert />` 的 `componentName` 將會是 `alert`。`attributes` 元素將包含 HTML 標籤上存在的所有屬性。`slot` 元素是一個包含元件插槽內容的 `Illuminate\Support\HtmlString` 實例。

該閉包應回傳一個字串。如果回傳的字串對應於現有的視圖，則會渲染該視圖；否則，回傳的字串將被視為行內 Blade 視圖進行評估。

<a name="additional-dependencies"></a>
#### 額外依賴

如果您的元件需要來自 Laravel [服務容器](/docs/{{version}}/container)的依賴項，您可以在元件的任何資料屬性之前列出它們，它們將自動由容器注入：

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

如果您想防止某些公開方法或屬性作為變數暴露給元件模板，您可以將它們新增到元件的 `$except` 陣列屬性中：

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

我們已經研究了如何將資料屬性傳遞給元件；然而，有時您可能需要指定額外的 HTML 屬性（例如 `class`），這些屬性並非元件運作所需的資料。通常，您會希望將這些額外的屬性傳遞到元件模板的根元素。例如，假設我們想要像這樣渲染一個 `alert` 元件：

```blade
<x-alert type="error" :message="$message" class="mt-4"/>
```

所有不屬於元件建構子 (Constructor) 的屬性都將自動加到元件的「屬性包 (Attribute Bag)」中。這個屬性包會透過 `$attributes` 變數自動提供給元件。透過印出此變數，可以在元件內渲染所有的屬性：

```blade
<div {{ $attributes }}>
    <!-- Component content -->
</div>
```

> [!WARNING]
> 目前不支援在元件標籤內使用像是 `@env` 這樣的指令。例如，`<x-alert :live="@env('production')"/>` 將不會被編譯。


<a name="default-merged-attributes"></a>
#### 預設 / 合併屬性

有時您可能需要為屬性指定預設值，或將額外的值合併到元件的某些屬性中。為此，您可以使用屬性包的 `merge` 方法。此方法對於定義一組應始終套用於元件的預設 CSS class 特別有用：

```blade
<div {{ $attributes->merge(['class' => 'alert alert-'.$type]) }}>
    {{ $message }}
</div>
```

假設這個元件是這樣被使用的：

```blade
<x-alert type="error" :message="$message" class="mb-4"/>
```

元件最終渲染出的 HTML 將如下所示：

```blade
<div class="alert alert-error mb-4">
    <!-- Contents of the $message variable -->
</div>
```


<a name="conditionally-merge-classes"></a>
#### 條件式合併 Class

有時您可能希望在給定條件為 `true` 時合併 Class。您可以透過 `class` 方法來達成，該方法接受一個包含 Class 的陣列，其中陣列的鍵名 (Key) 是您想要增加的一個或多個 Class，而鍵值 (Value) 則是一個布林運算式。如果陣列元素具有數字鍵名，則它將始終包含在渲染後的 Class 列表中：

```blade
<div {{ $attributes->class(['p-4', 'bg-red' => $hasError]) }}>
    {{ $message }}
</div>
```

如果您需要將其他屬性合併到元件中，可以將 `merge` 方法鏈接到 `class` 方法之後：

```blade
<button {{ $attributes->class(['p-4'])->merge(['type' => 'button']) }}>
    {{ $slot }}
</button>
```

> [!NOTE]
> 如果您需要在其他不應接收合併屬性的 HTML 元素上條件式編譯 Class，可以使用 [@class 指令](#conditional-classes)。


<a name="non-class-attribute-merging"></a>
#### 非 Class 屬性的合併

合併非 `class` 屬性時，提供給 `merge` 方法的值將被視為屬性的「預設」值。然而，與 `class` 屬性不同，這些屬性不會與注入的屬性值合併，而是會被覆蓋。例如，`button` 元件的實作可能如下所示：

```blade
<button {{ $attributes->merge(['type' => 'button']) }}>
    {{ $slot }}
</button>
```

若要以自定義 `type` 渲染按鈕元件，可以在使用元件時指定。如果未指定類型，則會使用 `button` 類型：

```blade
<x-button type="submit">
    Submit
</x-button>
```

此範例中 `button` 元件渲染出的 HTML 將會是：

```blade
<button type="submit">
    Submit
</button>
```

如果您希望 `class` 以外的屬性同時具有預設值與注入值，可以使用 `prepends` 方法。在此範例中，`data-controller` 屬性將始終以 `profile-controller` 開頭，任何額外注入的 `data-controller` 值都將放在此預設值之後：

```blade
<div {{ $attributes->merge(['data-controller' => $attributes->prepends('profile-controller')]) }}>
    {{ $slot }}
</div>
```


<a name="filtering-attributes"></a>
#### 取得與過濾屬性

您可以使用 `filter` 方法過濾屬性。該方法接受一個閉包 (Closure)，如果您希望保留該屬性在屬性包中，該閉包應回傳 `true`：

```blade
{{ $attributes->filter(fn (string $value, string $key) => $key == 'foo') }}
```

為了方便起見，您可以使用 `whereStartsWith` 方法取得所有鍵名以給定字串開頭的屬性：

```blade
{{ $attributes->whereStartsWith('wire:model') }}
```

相反地，`whereDoesntStartWith` 方法可用於排除所有鍵名以給定字串開頭的屬性：

```blade
{{ $attributes->whereDoesntStartWith('wire:model') }}
```

使用 `first` 方法，您可以渲染給定屬性包中的第一個屬性：

```blade
{{ $attributes->whereStartsWith('wire:model')->first() }}
```

如果您想檢查元件上是否存在某個屬性，可以使用 `has` 方法。該方法僅接受屬性名稱作為其唯一參數，並回傳一個布林值，表示該屬性是否存在：

```blade
@if ($attributes->has('class'))
    <div>Class attribute is present</div>
@endif
```

如果將陣列傳遞給 `has` 方法，該方法將判斷所有指定的屬性是否都存在於元件上：

```blade
@if ($attributes->has(['name', 'class']))
    <div>All of the attributes are present</div>
@endif
```

`hasAny` 方法可用於判斷指定的任一屬性是否存在於元件上：

```blade
@if ($attributes->hasAny(['href', ':href', 'v-bind:href']))
    <div>One of the attributes is present</div>
@endif
```

您可以使用 `get` 方法取得特定屬性的值：

```blade
{{ $attributes->get('class') }}
```

`only` 方法可用於僅取得具有給定鍵名的屬性：

```blade
{{ $attributes->only(['class']) }}
```

`except` 方法可用於取得除了具有給定鍵名的屬性以外的所有屬性：

```blade
{{ $attributes->except(['class']) }}
```


<a name="reserved-keywords"></a>
### 保留關鍵字

預設情況下，某些關鍵字保留給 Blade 內部使用以渲染元件。以下關鍵字不能在您的元件中定義為公用屬性或方法名稱：

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
### 插槽 (Slots)

你經常需要透過「插槽 (Slots)」將額外內容傳遞給元件。元件插槽是透過輸出 `$slot` 變數來渲染的。為了探索這個概念，讓我們想像一個 `alert` 元件具有以下標記：

```blade
<!-- /resources/views/components/alert.blade.php -->

<div class="alert alert-danger">
    {{ $slot }}
</div>
```

我們可以透過將內容注入元件來將內容傳遞給 `slot`：

```blade
<x-alert>
    <strong>Whoops!</strong> Something went wrong!
</x-alert>
```

有時，元件可能需要在元件內的不同位置渲染多個不同的插槽。讓我們修改 alert 元件，以允許注入「title」插槽：

```blade
<!-- /resources/views/components/alert.blade.php -->

<span class="alert-title">{{ $title }}</span>

<div class="alert alert-danger">
    {{ $slot }}
</div>
```

你可以使用 `x-slot` 標籤定義具名插槽的內容。任何不在明確的 `x-slot` 標籤內的內容都將在 `$slot` 變數中傳遞給元件：

```xml
<x-alert>
    <x-slot:title>
        Server Error
    </x-slot>

    <strong>Whoops!</strong> Something went wrong!
</x-alert>
```

你可以呼叫插槽的 `isEmpty` 方法來判斷該插槽是否包含內容：

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

此外，`hasActualContent` 方法可用於判斷插槽是否包含任何不是 HTML 註解的「實際」內容：

```blade
@if ($slot->hasActualContent())
    The scope has non-comment content.
@endif
```


<a name="scoped-slots"></a>
#### 作用域插槽 (Scoped Slots)

如果你使用過 Vue 等 JavaScript 框架，你可能熟悉「作用域插槽 (Scoped Slots)」，它允許你在插槽中存取元件的資料或方法。在 Laravel 中，你可以透過在元件上定義公開方法或屬性，並在插槽中經由 `$component` 變數存取元件來達成類似的行為。在這個範例中，我們假設 `x-alert` 元件在其元件類別中定義了一個公開的 `formatAlert` 方法：

```blade
<x-alert>
    <x-slot:title>
        {{ $component->formatAlert('Server Error') }}
    </x-slot>

    <strong>Whoops!</strong> Something went wrong!
</x-alert>
```


<a name="slot-attributes"></a>
#### 插槽屬性 (Slot Attributes)

就像 Blade 元件一樣，你可以為插槽分配額外的[屬性](#component-attributes)，例如 CSS class 名稱：

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

要與插槽屬性互動，你可以存取插槽變數的 `attributes` 屬性。有關如何與屬性互動的更多資訊，請參閱 [元件屬性 (Component Attributes)](#component-attributes) 的文件：

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

對於非常小的元件，管理元件類別和元件的視圖模板可能會感到繁瑣。因此，你可以直接從 `render` 方法中回傳元件的標記：

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
#### 生成行內視圖元件

要建立渲染行內視圖的元件，可以在執行 `make:component` 指令時使用 `inline` 選項：

```shell
php artisan make:component Alert --inline
```


<a name="dynamic-components"></a>
### 動態元件

有時你可能需要渲染一個元件，但直到執行時才知道應該渲染哪個元件。在這種情況下，你可以使用 Laravel 內建的 `dynamic-component` 元件，根據執行時的值或變數來渲染元件：

```blade
// $componentName = "secondary-button";

<x-dynamic-component :component="$componentName" class="mt-4" />
```


<a name="manually-registering-components"></a>
### 手動註冊元件

> [!WARNING]
> 以下關於手動註冊元件的文件主要適用於編寫包含視圖元件的 Laravel 套件的開發者。如果你不是在編寫套件，則這部分元件文件可能與你無關。

為你自己的應用程式編寫元件時，元件會自動在 `app/View/Components` 目錄和 `resources/views/components` 目錄中被發現。

但是，如果你正在建立一個使用 Blade 元件的套件，或者將元件放在非傳統目錄中，則需要手動註冊你的元件類別及其 HTML 標籤別名，以便 Laravel 知道在哪裡可以找到該元件。你通常應該在套件服務提供者的 `boot` 方法中註冊你的元件：

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

元件註冊後，就可以使用其標籤別名進行渲染：

```blade
<x-package-alert/>
```


#### 自動載入套件元件

或者，你可以使用 `componentNamespace` 方法按照慣例自動載入元件類別。例如，`Nightshade` 套件可能具有位於 `Package\Views\Components` 命名空間內的 `Calendar` 和 `ColorPicker` 元件：

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

這將允許使用 `package-name::` 語法，透過其供應商命名空間來使用套件元件：

```blade
<x-nightshade::calendar />
<x-nightshade::color-picker />
```

Blade 會透過將元件名稱轉換為 PascalCase 來自動偵測連結到此元件的類別。也支援使用「點 (dot)」符號來表示子目錄。

<a name="anonymous-components"></a>
## 匿名元件

類似於行內元件，匿名元件提供了一種透過單個檔案管理元件的機制。然而，匿名元件使用單一視圖檔案且沒有關聯類別。要定義一個匿名元件，您只需要將一個 Blade 模板放置在您的 `resources/views/components` 目錄中。例如，假設您在 `resources/views/components/alert.blade.php` 定義了一個元件，您可以簡單地像這樣渲染它：

```blade
<x-alert/>
```

您可以使用 `.` 字元來表示元件是否巢狀於 `components` 目錄深處。例如，假設元件定義在 `resources/views/components/inputs/button.blade.php`，您可以像這樣渲染它：

```blade
<x-inputs.button/>
```

要透過 Artisan 建立一個匿名元件，您可以在執行 `make:component` 指令時使用 `--view` 旗標：

```shell
php artisan make:component forms.input --view
```

上述指令將在 `resources/views/components/forms/input.blade.php` 建立一個 Blade 檔案，該檔案可以透過 `<x-forms.input />` 作為元件渲染。


<a name="anonymous-index-components"></a>
### 匿名索引元件

有時，當一個元件由許多 Blade 模板組成時，您可能希望將該元件的模板分組到單個目錄中。例如，想像一個具有以下目錄結構的 "accordion" 元件：

```text
/resources/views/components/accordion.blade.php
/resources/views/components/accordion/item.blade.php
```

這種目錄結構允許您像這樣渲染 accordion 元件及其子項：

```blade
<x-accordion>
    <x-accordion.item>
        ...
    </x-accordion.item>
</x-accordion>
```

然而，為了透過 `x-accordion` 渲染 accordion 元件，我們被迫將「索引」的 accordion 元件模板放置在 `resources/views/components` 目錄中，而不是與其他 accordion 相關模板一起巢狀在 `accordion` 目錄中。

幸運的是，Blade 允許您在元件目錄本身中放置一個與該目錄名稱相匹配的檔案。當此模板存在時，它可被渲染為元件的「根」元素，即使它被巢狀在目錄中。因此，我們可以繼續使用上述範例中給出的相同 Blade 語法；但是，我們將調整我們的目錄結構如下：

```text
/resources/views/components/accordion/accordion.blade.php
/resources/views/components/accordion/item.blade.php
```


<a name="data-properties-attributes"></a>
### 資料屬性 / 特性

由於匿名元件沒有任何關聯類別，您可能會想知道如何區分哪些資料應作為變數傳遞給元件，以及哪些屬性應放置在元件的 [屬性包](#component-attributes) 中。

您可以在元件 Blade 模板的頂部使用 `@props` 指令指定哪些屬性應被視為資料變數。元件上的所有其他屬性將可透過元件的屬性包取得。如果您希望為資料變數提供預設值，您可以將變數名稱指定為陣列鍵，並將預設值指定為陣列值：

```blade
<!-- /resources/views/components/alert.blade.php -->

@props(['type' => 'info', 'message'])

<div {{ $attributes->merge(['class' => 'alert alert-'.$type]) }}>
    {{ $message }}
</div>
```

給定上述元件定義，我們可以像這樣渲染元件：

```blade
<x-alert type="error" :message="$message" class="mb-4"/>
```


<a name="accessing-parent-data"></a>
### 存取父層資料

有時您可能想在子元件中存取父元件的資料。在這些情況下，您可以使用 `@aware` 指令。例如，想像我們正在建立一個複雜的選單元件，由父元件 `<x-menu>` 和子元件 `<x-menu.item>` 組成：

```blade
<x-menu color="purple">
    <x-menu.item>...</x-menu.item>
    <x-menu.item>...</x-menu.item>
</x-menu>
```

`<x-menu>` 元件可能有一個如下的實作：

```blade
<!-- /resources/views/components/menu/index.blade.php -->

@props(['color' => 'gray'])

<ul {{ $attributes->merge(['class' => 'bg-'.$color.'-200']) }}>
    {{ $slot }}
</ul>
```

由於 `color` 屬性只被傳遞給父元件 (`<x-menu>`)，它在 `<x-menu.item>` 內部將無法使用。但是，如果我們使用 `@aware` 指令，我們也可以讓它在 `<x-menu.item>` 內部可用：

```blade
<!-- /resources/views/components/menu/item.blade.php -->

@aware(['color' => 'gray'])

<li {{ $attributes->merge(['class' => 'text-'.$color.'-800']) }}>
    {{ $slot }}
</li>
```

> [!WARNING]
> `@aware` 指令無法存取未經由 HTML 屬性明確傳遞給父元件的父層資料。未明確傳遞給父元件的預設 `@props` 值無法被 `@aware` 指令存取。


<a name="anonymous-component-paths"></a>
### 匿名元件路徑

如前所述，匿名元件通常是透過將 Blade 模板放置在您的 `resources/views/components` 目錄中來定義。但是，您偶爾可能希望在預設路徑之外，向 Laravel 註冊其他的匿名元件路徑。

`anonymousComponentPath` 方法接受匿名元件位置的「路徑」作為其第一個參數，並接受一個選用的「命名空間」作為其第二個參數，元件應放置在該命名空間下。通常，應從應用程式其中一個 [服務提供者](/docs/{{version}}/providers) 的 `boot` 方法中呼叫此方法：

```php
/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Blade::anonymousComponentPath(__DIR__.'/../components');
}
```

當元件路徑在註冊時未指定前綴（如上例所示），它們也可以在您的 Blade 元件中渲染，而無需對應的前綴。例如，如果上述註冊的路徑中存在 `panel.blade.php` 元件，則可以像這樣渲染它：

```blade
<x-panel />
```

前綴「命名空間」可以作為 `anonymousComponentPath` 方法的第二個參數提供：

```php
Blade::anonymousComponentPath(__DIR__.'/../components', 'dashboard');
```

當提供了前綴時，該「命名空間」內的元件在渲染時，可以透過在元件名稱前加上元件的命名空間來渲染：

```blade
<x-dashboard::panel />
```

<a name="building-layouts"></a>
## 建構佈局

<a name="layouts-using-components"></a>
### 使用元件建立佈局

大多數網頁應用程式在各個頁面中都維持相同的通用佈局。如果我們必須在建立的每個視圖中重複整個佈局 HTML，那將會非常麻煩且難以維護。幸運的是，將此佈局定義為單個 [Blade 元件](#components)，然後在整個應用程式中使用它非常方便。

<a name="defining-the-layout-component"></a>
#### 定義佈局元件

例如，想像我們正在構建一個「待辦事項 (todo)」清單應用程式。我們可能會定義一個 `layout` 元件，如下所示：

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
#### 套用佈局元件

一旦定義了 `layout` 元件，我們就可以建立一個使用該元件的 Blade 視圖。在此範例中，我們將定義一個顯示任務清單的簡單視圖：

```blade
<!-- resources/views/tasks.blade.php -->

<x-layout>
    @foreach ($tasks as $task)
        <div>{{ $task }}</div>
    @endforeach
</x-layout>
```

請記住，注入到元件中的內容將會提供給 `layout` 元件中預設的 `$slot` 變數。正如您可能已經注意到的，我們的 `layout` 也支援 `$title` 插槽（如果有的話）；否則，會顯示預設標題。我們可以使用 [元件文件](#components) 中討論的標準插槽語法從任務清單視圖中注入自定義標題：

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

現在我們已經定義了佈局和任務清單視圖，我們只需要從路由回傳 `task` 視圖即可：

```php
use App\Models\Task;

Route::get('/tasks', function () {
    return view('tasks', ['tasks' => Task::all()]);
});
```

<a name="layouts-using-template-inheritance"></a>
### 使用模板繼承建立佈局

<a name="defining-a-layout"></a>
#### 定義佈局

佈局也可以透過「模板繼承」來建立。在 [元件 (components)](#components) 推出之前，這是構建應用程式的主要方式。

首先，讓我們看一個簡單的範例。首先，我們將檢查頁面佈局。由於大多數網頁應用程式在各個頁面中維持相同的通用佈局，因此將此佈局定義為單個 Blade 視圖非常方便：

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

如您所見，此檔案包含典型的 HTML 標記。但是，請注意 `@section` 和 `@yield` 指令。正如其名，`@section` 指令定義了一個內容區塊，而 `@yield` 指令則用於顯示指定區塊的內容。

現在我們已經為應用程式定義了一個佈局，讓我們定義一個繼承該佈局的子頁面。

<a name="extending-a-layout"></a>
#### 擴充佈局

定義子視圖時，請使用 `@extends` Blade 指令來指定子視圖應該「繼承」哪個佈局。繼承 Blade 佈局的視圖可以使用 `@section` 指令將內容注入到佈局的區塊中。請記住，如上例所示，這些區塊的內容將使用 `@yield` 顯示在佈局中：

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

在此範例中，`sidebar` 區塊正在利用 `@@parent` 指令將內容附加到佈局的側邊欄（而不是覆寫它）。當視圖渲染時，`@@parent` 指令將被佈局的內容替換。

> [!NOTE]
> 與之前的範例相反，這個 `sidebar` 區塊以 `@endsection` 而非 `@show` 結束。`@endsection` 指令僅定義一個區塊，而 `@show` 則會定義並**立即產生 (yield)** 該區塊。

`@yield` 指令還接受一個預設值作為其第二個參數。如果正在產生的區塊未定義，則會渲染此值：

```blade
@yield('content', 'Default content')
```

<a name="forms"></a>
## 表單

<a name="csrf-field"></a>
### CSRF 欄位

每當您在應用程式中定義 HTML 表單時，都應在表單中包含一個隱藏的 CSRF 權杖 (token) 欄位，以便 [CSRF 保護](/docs/{{version}}/csrf) 中介層 (middleware) 可以驗證請求。您可以使用 `@csrf` Blade 指令來產生權杖欄位：

```blade
<form method="POST" action="/profile">
    @csrf

    ...
</form>
```

<a name="method-field"></a>
### 方法欄位

由於 HTML 表單無法發出 `PUT`、`PATCH` 或 `DELETE` 請求，您需要新增一個隱藏的 `_method` 欄位來模擬這些 HTTP 動詞。`@method` Blade 指令可以為您建立此欄位：

```blade
<form action="/foo/bar" method="POST">
    @method('PUT')

    ...
</form>
```

<a name="validation-errors"></a>
### 驗證錯誤

`@error` 指令可用於快速檢查給定屬性是否存在 [驗證錯誤訊息](/docs/{{version}}/validation#quick-displaying-the-validation-errors)。在 `@error` 指令內，您可以輸出 `$message` 變數以顯示錯誤訊息：

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

由於 `@error` 指令會編譯為 「if」 語句，您可以使用 `@else` 指令在屬性沒有錯誤時渲染內容：

```blade
<!-- /resources/views/auth.blade.php -->

<label for="email">Email address</label>

<input
    id="email"
    type="email"
    class="@error('email') is-invalid @else is-valid @enderror"
/>
```

您可以將 [特定錯誤袋的名稱](/docs/{{version}}/validation#named-error-bags) 作為第二個參數傳遞給 `@error` 指令，以便在包含多個表單的頁面上檢索驗證錯誤訊息：

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
## 堆疊 (Stacks)

Blade 允許您推送到具名堆疊 (Stacks)，這些堆疊可以在其他視圖或佈局中的其他地方進行渲染。這對於指定子視圖所需的任何 JavaScript 函式庫特別有用：

```blade
@push('scripts')
    <script src="/example.js"></script>
@endpush
```

如果您想在給定的布林運算式評估為 `true` 時才 `@push` 內容，可以使用 `@pushIf` 指令：

```blade
@pushIf($shouldPush, 'scripts')
    <script src="/example.js"></script>
@endPushIf
```

您可以根據需要多次推送到同一個堆疊。要渲染完整的堆疊內容，請將堆疊名稱傳遞給 `@stack` 指令：

```blade
<head>
    <!-- Head Contents -->

    @stack('scripts')
</head>
```

如果您想將內容加到堆疊的最前面，應該使用 `@prepend` 指令：

```blade
@push('scripts')
    This will be second...
@endpush

// Later...

@prepend('scripts')
    This will be first...
@endprepend
```

`@hasstack` 指令可用於判斷堆疊是否為空：

```blade
@hasstack('list')
    <ul>
        @stack('list')
    </ul>
@endif
```


<a name="service-injection"></a>
## 服務注入

`@inject` 指令可用於從 Laravel [服務容器](/docs/{{version}}/container)中取得服務。傳遞給 `@inject` 的第一個參數是服務將被放置其中的變數名稱，而第二個參數則是您想要解析的服務類別或介面名稱：

```blade
@inject('metrics', 'App\Services\MetricsService')

<div>
    Monthly Revenue: {{ $metrics->monthlyRevenue() }}.
</div>
```


<a name="rendering-inline-blade-templates"></a>
## 渲染行內 Blade 模板

有時您可能需要將原始的 Blade 模板字串轉換為有效的 HTML。您可以使用 `Blade` Facade 提供的 `render` 方法來達成此目的。`render` 方法接受 Blade 模板字串以及一個可選的資料陣列來提供給模板：

```php
use Illuminate\Support\Facades\Blade;

return Blade::render('Hello, {{ $name }}', ['name' => 'Julian Bashir']);
```

Laravel 透過將行內 Blade 模板寫入 `storage/framework/views` 目錄來進行渲染。如果您希望 Laravel 在渲染 Blade 模板後移除這些暫存檔案，您可以為該方法提供 `deleteCachedView` 參數：

```php
return Blade::render(
    'Hello, {{ $name }}',
    ['name' => 'Julian Bashir'],
    deleteCachedView: true
);
```


<a name="rendering-blade-fragments"></a>
## 渲染 Blade 片段

當使用諸如 [Turbo](https://turbo.hotwired.dev/) 和 [htmx](https://htmx.org/) 等前端框架時，您偶爾可能只需要在 HTTP 回應中回傳 Blade 模板的一部分。Blade「片段 (Fragments)」允許您做到這一點。首先，將 Blade 模板的一部分放在 `@fragment` 和 `@endfragment` 指令中：

```blade
@fragment('user-list')
    <ul>
        @foreach ($users as $user)
            <li>{{ $user->name }}</li>
        @endforeach
    </ul>
@endfragment
```

接著，在渲染使用此模板的視圖時，您可以呼叫 `fragment` 方法來指定只有指定的片段應包含在輸出的 HTTP 回應中：

```php
return view('dashboard', ['users' => $users])->fragment('user-list');
```

`fragmentIf` 方法允許您根據給定條件有條件地回傳視圖片段。否則，將回傳整個視圖：

```php
return view('dashboard', ['users' => $users])
    ->fragmentIf($request->hasHeader('HX-Request'), 'user-list');
```

`fragments` 和 `fragmentsIf` 方法允許您在回應中回傳多個視圖片段。這些片段將會被串接在一起：

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

Blade 允許您使用 `directive` 方法定義自己的自定義指令。當 Blade 編譯器遇到自定義指令時，它將呼叫提供的回呼函式，並傳入該指令所包含的運算式。

以下範例建立了一個 `@datetime($var)` 指令，它會格式化給定的 `$var`，該變數應為 `DateTime` 的實例：

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

如您所見，我們將 `format` 方法串接到傳遞給指令的任何運算式上。因此，在此範例中，此指令生成的最終 PHP 將如下所示：

```php
<?php echo ($var)->format('m/d/Y H:i'); ?>
```

> [!WARNING]
> 更新 Blade 指令的邏輯後，您將需要刪除所有快取的 Blade 視圖。可以使用 `view:clear` Artisan 指令移除快取的 Blade 視圖。


<a name="custom-echo-handlers"></a>
### 自定義 Echo 處理器

如果您嘗試使用 Blade「印出 (Echo)」一個物件，該物件的 `__toString` 方法將被呼叫。[__toString](https://www.php.net/manual/en/language.oop5.magic.php#object.tostring) 方法是 PHP 內建的「魔術方法」之一。然而，有時您可能無法控制給定類別的 `__toString` 方法，例如當您正在互動的類別屬於第三方函式庫時。

在這些情況下， Blade 允許您為該特定類別的物件註冊自定義 Echo 處理器。要達成此目的，您應該呼叫 Blade 的 `stringable` 方法。`stringable` 方法接受一個閉包 (Closure)。此閉包應類型提示 (Type-hint) 其負責渲染的物件類型。通常，`stringable` 方法應在應用程式 `AppServiceProvider` 類別的 `boot` 方法中呼叫：

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

定義好自定義 Echo 處理器後，您就可以簡單地在 Blade 模板中印出該物件：

```blade
Cost: {{ $money }}
```


<a name="custom-if-statements"></a>
### 自定義 If 語句

在定義簡單的自定義條件語句時，撰寫自定義指令有時會比必要的更複雜。因此，Blade 提供了一個 `Blade::if` 方法，讓您可以使用閉包快速定義自定義條件指令。例如，讓我們定義一個自定義條件，用於檢查應用程式設定的預設「磁碟 (Disk)」。我們可以在 `AppServiceProvider` 的 `boot` 方法中執行此操作：

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

定義好自定義條件後，您就可以在模板中使用它：

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