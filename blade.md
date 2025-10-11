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
    - [迴圈變數](#the-loop-variable)
    - [條件式 Class](#conditional-classes)
    - [額外屬性](#additional-attributes)
    - [包含子視圖](#including-subviews)
    - [`@once` 指令](#the-once-directive)
    - [原生 PHP](#raw-php)
    - [註解](#comments)
- [組件](#components)
    - [渲染組件](#rendering-components)
    - [索引組件](#index-components)
    - [傳遞資料至組件](#passing-data-to-components)
    - [組件屬性](#component-attributes)
    - [保留關鍵字](#reserved-keywords)
    - [插槽](#slots)
    - [行內組件視圖](#inline-component-views)
    - [動態組件](#dynamic-components)
    - [手動註冊組件](#manually-registering-components)
- [匿名組件](#anonymous-components)
    - [匿名索引組件](#anonymous-index-components)
    - [資料屬性 / Attributes](#data-properties-attributes)
    - [存取父級資料](#accessing-parent-data)
    - [匿名組件路徑](#anonymous-component-paths)
- [建構佈局](#building-layouts)
    - [使用組件的佈局](#layouts-using-components)
    - [使用模板繼承的佈局](#layouts-using-template-inheritance)
- [表單](#forms)
    - [CSRF 欄位](#csrf-field)
    - [Method 欄位](#method-field)
    - [驗證錯誤](#validation-errors)
- [堆疊](#stacks)
- [服務注入](#service-injection)
- [渲染行內 Blade 模板](#rendering-inline-blade-templates)
- [渲染 Blade 片段](#rendering-blade-fragments)
- [擴充 Blade](#extending-blade)
    - [自訂 Echo 處理器](#custom-echo-handlers)
    - [自訂 If 語句](#custom-if-statements)

<a name="introduction"></a>
## 簡介

Blade 是隨 Laravel 框架附帶的一個簡單卻功能強大的模板引擎。與某些 PHP 模板引擎不同，Blade 不會限制您在模板中使用原生 PHP 程式碼。事實上，所有 Blade 模板都會被編譯成原生 PHP 程式碼並快取起來，直到它們被修改為止，這意味著 Blade 幾乎不會為您的應用程式增加額外負擔。Blade 模板檔案使用 `.blade.php` 副檔名，通常儲存於 `resources/views` 目錄中。

Blade 視圖可以透過全域 `view` 輔助函式從路由或控制器回傳。當然，如 [視圖](/docs/{{version}}/views) 說明文件所述，資料可以透過 `view` 輔助函式的第二個參數傳遞給 Blade 視圖：

    Route::get('/', function () {
        return view('greeting', ['name' => 'Finn']);
    });


<a name="supercharging-blade-with-livewire"></a>
### 使用 Livewire 強化 Blade

想將您的 Blade 模板提升到更高層次，並輕鬆建構動態介面嗎？請查看 [Laravel Livewire](https://livewire.laravel.com)。Livewire 允許您編寫 Blade 組件，這些組件透過動態功能來增強，這些功能通常只有透過像 React 或 Vue 這樣的前端框架才能實現，它提供了一種極佳的方法來建構現代、反應式的前端，而無需許多 JavaScript 框架的複雜性、客戶端渲染或建構步驟。


<a name="displaying-data"></a>
## 顯示資料

您可以透過使用大括號包圍變數，來顯示傳遞給 Blade 視圖的資料。例如，給定以下路由：

    Route::get('/', function () {
        return view('welcome', ['name' => 'Samantha']);
    });

您可以像這樣顯示 `name` 變數的內容：

```blade
Hello, {{ $name }}.
```

> [!NOTE]  
> Blade 的 `{{ }}` echo 語句會自動通過 PHP 的 `htmlspecialchars` 函式，以防止 XSS 攻擊。

您不限於顯示傳遞給視圖的變數內容。您也可以輸出任何 PHP 函式的結果。事實上，您可以在 Blade echo 語句內放置任何您希望的 PHP 程式碼：

```blade
The current UNIX timestamp is {{ time() }}.
```


<a name="html-entity-encoding"></a>
### HTML 實體編碼

預設情況下，Blade (以及 Laravel 的 `e` 函式) 會對 HTML 實體進行雙重編碼。如果您想停用雙重編碼，可以在 `AppServiceProvider` 的 `boot` 方法中呼叫 `Blade::withoutDoubleEncoding` 方法：

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


<a name="displaying-unescaped-data"></a>
#### 顯示未逸出資料

預設情況下，Blade 的 `{{ }}` 語句會自動通過 PHP 的 `htmlspecialchars` 函式，以防止 XSS 攻擊。如果您不希望資料被逸出，可以使用以下語法：

```blade
Hello, {!! $name !!}.
```

> [!WARNING]  
> 在輸出應用程式使用者提供的內容時，請務必非常小心。通常您應該使用逸出的雙大括號語法來防止在顯示使用者提供的資料時發生 XSS 攻擊。


<a name="blade-and-javascript-frameworks"></a>
### Blade 與 JavaScript 框架

由於許多 JavaScript 框架也使用「大括號」來表示給定的表達式應在瀏覽器中顯示，因此您可以使用 `@` 符號來告知 Blade 渲染引擎該表達式應保持原樣。例如：

```blade
<h1>Laravel</h1>

Hello, @{{ name }}.
```

在此範例中，`@` 符號將被 Blade 移除；然而，`{{ name }}` 表達式將保持原樣，允許它由您的 JavaScript 框架渲染。

`@` 符號也可以用來逸出 Blade 指令：

```blade
{{-- Blade template --}}
@@if()

<!-- HTML output -->
@if()
```


<a name="rendering-json"></a>
#### 渲染 JSON

有時您可能會將陣列傳遞給視圖，目的是將其渲染為 JSON 以初始化 JavaScript 變數。例如：

```blade
<script>
    var app = <?php echo json_encode($array); ?>;
</script>
```

然而，您可以不必手動呼叫 `json_encode`，而是可以使用 `Illuminate\Support\Js::from` 方法指令。`from` 方法接受與 PHP 的 `json_encode` 函式相同的參數；但是，它會確保產生的 JSON 正確逸出，以便包含在 HTML 引號中。`from` 方法將回傳一個 `JSON.parse` JavaScript 字串語句，該語句會將給定的物件或陣列轉換為有效的 JavaScript 物件：

```blade
<script>
    var app = {{ Illuminate\Support\Js::from($array) }};
</script>
```

Laravel 應用程式骨架的最新版本包含 `Js` Facade，它在您的 Blade 模板中提供了便捷的存取此功能的方式：

```blade
<script>
    var app = {{ Js::from($array) }};
</script>
```

> [!WARNING]  
> 您應該只使用 `Js::from` 方法將現有變數渲染為 JSON。Blade 模板是基於正規表達式，嘗試向指令傳遞複雜的表達式可能會導致意外的失敗。


<a name="the-at-verbatim-directive"></a>
#### `@verbatim` 指令

如果您在模板的大部分內容中顯示 JavaScript 變數，您可以將 HTML 包圍在 `@verbatim` 指令中，這樣您就不必在每個 Blade echo 語句前加上 `@` 符號：

```blade
@verbatim
    <div class="container">
        Hello, {{ name }}.
    </div>
@endverbatim
```

<a name="blade-directives"></a>
## Blade 指令

除了模板繼承和顯示資料外，Blade 也提供了方便的快捷方式來處理常見的 PHP 控制結構，例如條件語句和迴圈。這些快捷方式提供了一種非常簡潔明瞭的方式來使用 PHP 控制結構，同時也與其 PHP 對應物保持一致。


<a name="if-statements"></a>
### If 語句

您可以使用 `@if`、`@elseif`、`@else` 和 `@endif` 指令來建構 `if` 語句。這些指令的功能與其 PHP 對應物完全相同：

```blade
@if (count($records) === 1)
    I have one record!
@elseif (count($records) > 1)
    I have multiple records!
@else
    I don't have any records!
@endif
```

為了方便，Blade 也提供了 `@unless` 指令：

```blade
@unless (Auth::check())
    You are not signed in.
@endunless
```

除了已經討論過的條件指令之外，`@isset` 和 `@empty` 指令也可以用作各自 PHP 函數的方便快捷方式：

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

`@auth` 和 `@guest` 指令可用於快速判斷當前使用者是否已 [認證](/docs/{{version}}/authentication) 或為訪客：

```blade
@auth
    // The user is authenticated...
@endauth

@guest
    // The user is not authenticated...
@endguest
```

如有需要，您可以在使用 `@auth` 和 `@guest` 指令時指定應檢查的認證 Guard：

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

您可以使用 `@production` 指令檢查應用程式是否在 production 環境中執行：

```blade
@production
    // Production specific content...
@endproduction
```

或者，您可以使用 `@env` 指令判斷應用程式是否在特定環境中執行：

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

您可以使用 `@hasSection` 指令判斷模板繼承的區塊 (section) 是否有內容：

```blade
@hasSection('navigation')
    <div class="pull-right">
        @yield('navigation')
    </div>

    <div class="clearfix"></div>
@endif
```

您可以使用 `sectionMissing` 指令判斷某個區塊 (section) 是否沒有內容：

```blade
@sectionMissing('navigation')
    <div class="pull-right">
        @include('default-navigation')
    </div>
@endif
```


<a name="session-directives"></a>
#### Session 指令

`@session` 指令可用於判斷 [session](/docs/{{version}}/session) 值是否存在。如果 session 值存在，則會評估 `@session` 和 `@endsession` 指令內的模板內容。在 `@session` 指令的內容中，您可以印出 `$value` 變數來顯示 session 值：

```blade
@session('status')
    <div class="p-4 bg-green-100">
        {{ $value }}
    </div>
@endsession
```


<a name="switch-statements"></a>
### Switch 語句

Switch 語句可以使用 `@switch`、`@case`、`@break`、`@default` 和 `@endswitch` 指令來建構：

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

除了條件語句之外，Blade 也提供了用於處理 PHP 迴圈結構的簡單指令。同樣地，這些指令的功能與其 PHP 對應物完全相同：

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
> 迭代 `foreach` 迴圈時，您可以使用 [迴圈變數](#the-loop-variable) 來獲取有關迴圈的重要資訊，例如您是否處於迴圈的第一次或最後一次迭代。

使用迴圈時，您也可以使用 `@continue` 和 `@break` 指令來跳過當前迭代或終止迴圈：

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

您也可以在指令宣告中包含 continuation 或 break 條件：

```blade
@foreach ($users as $user)
    @continue($user->type == 1)

    <li>{{ $user->name }}</li>

    @break($user->number == 5)
@endforeach
```


<a name="the-loop-variable"></a>
### 迴圈變數

迭代 `foreach` 迴圈時，`$loop` 變數將在您的迴圈中可用。此變數提供了對一些有用資訊的存取，例如當前迴圈索引以及這是否是迴圈的第一次或最後一次迭代：

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

如果您在巢狀迴圈中，可以透過 `parent` 屬性存取父級迴圈的 `$loop` 變數：

```blade
@foreach ($users as $user)
    @foreach ($user->posts as $post)
        @if ($loop->parent->first)
            This is the first iteration of the parent loop.
        @endif
    @endforeach
@endforeach
```

`$loop` 變數也包含各種其他有用的屬性：

<div class="overflow-auto">

| 屬性               | 說明                                           |
| ------------------ | ------------------------------------------------------ |
| `$loop->index`     | 當前迴圈迭代的索引 (從 0 開始)。                      |
| `$loop->iteration` | 當前迴圈迭代 (從 1 開始)。                         |
| `$loop->remaining` | 迴圈中剩餘的迭代次數。                              |
| `$loop->count`     | 被迭代的陣列中項目的總數。                          |
| `$loop->first`     | 這是否是迴圈的第一次迭代。                          |
| `$loop->last`      | 這是否是迴圈的最後一次迭代。                          |
| `$loop->even`      | 這是否是迴圈的偶數次迭代。                          |
| `$loop->odd`       | 這是否是迴圈的奇數次迭代。                          |
| `$loop->depth`     | 當前迴圈的巢狀層級。                               |
| `$loop->parent`    | 處於巢狀迴圈時，父級的迴圈變數。                  |

</div>


<a name="conditional-classes"></a>
### 條件式 Class

`@class` 指令會條件式地編譯一個 CSS class 字串。該指令接受一個 class 陣列，其中陣列鍵包含您希望添加的一個或多個 class，而值是一個布林表達式。如果陣列元素具有數字鍵，它將始終包含在渲染的 class 列表中：

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

同樣地，`@style` 指令可用於條件式地將行內 CSS 樣式添加到 HTML 元素：

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

為了方便起見，您可以使用 `@checked` 指令來輕鬆指示給定的 HTML 核取方塊輸入是否為「已勾選」(checked)。如果所提供的條件評估為 `true`，此指令將會印出 `checked`：

```blade
<input
    type="checkbox"
    name="active"
    value="active"
    @checked(old('active', $user->active))
/>
```

同樣地，`@selected` 指令可用於指示給定的選取選項是否應為「已選取」(selected)：

```blade
<select name="version">
    @foreach ($product->versions as $version)
        <option value="{{ $version }}" @selected(old('version') == $version)>
            {{ $version }}
        </option>
    @endforeach
</select>
```

此外，`@disabled` 指令可用於指示給定的元素是否應為「已禁用」(disabled)：

```blade
<button type="submit" @disabled($errors->isNotEmpty())>Submit</button>
```

此外，`@readonly` 指令可用於指示給定的元素是否應為「唯讀」(readonly)：

```blade
<input
    type="email"
    name="email"
    value="email@laravel.com"
    @readonly($user->isNotAdmin())
/>
```

再者，`@required` 指令可用於指示給定的元素是否應為「必填」(required)：

```blade
<input
    type="text"
    name="title"
    value="title"
    @required($user->isAdmin())
/>
```


<a name="including-subviews"></a>
### 包含子視圖

> [!NOTE]  
> 儘管您可以自由使用 `@include` 指令，但 Blade [組件](#components)提供了類似的功能，並且比 `@include` 指令提供了一些優點，例如資料和屬性綁定。

Blade 的 `@include` 指令允許您在另一個視圖中包含一個 Blade 視圖。所有父視圖中可用的變數都將在被包含的視圖中可用：

```blade
<div>
    @include('shared.errors')

    <form>
        <!-- Form Contents -->
    </form>
</div>
```

儘管被包含的視圖將繼承父視圖中所有可用的資料，但您也可以傳遞一個額外資料的陣列，使其在被包含的視圖中可用：

```blade
@include('view.name', ['status' => 'complete'])
```

如果您嘗試 `@include` 一個不存在的視圖，Laravel 將會拋出錯誤。如果您想包含一個可能存在或不存在的視圖，您應該使用 `@includeIf` 指令：

```blade
@includeIf('view.name', ['status' => 'complete'])
```

如果您想在給定的布林運算式評估為 `true` 或 `false` 時 `@include` 一個視圖，您可以使用 `@includeWhen` 和 `@includeUnless` 指令：

```blade
@includeWhen($boolean, 'view.name', ['status' => 'complete'])

@includeUnless($boolean, 'view.name', ['status' => 'complete'])
```

若要從給定視圖陣列中包含第一個存在的視圖，您可以使用 `includeFirst` 指令：

```blade
@includeFirst(['custom.admin', 'admin'], ['status' => 'complete'])
```

> [!WARNING]  
> 您應該避免在 Blade 視圖中使用 `__DIR__` 和 `__FILE__` 常數，因為它們會指向快取、編譯後視圖的位置。


<a name="rendering-views-for-collections"></a>
#### 渲染集合的視圖

您可以使用 Blade 的 `@each` 指令將迴圈和包含合併為一行：

```blade
@each('view.name', $jobs, 'job')
```

`@each` 指令的第一個參數是為陣列或集合中的每個元素渲染的視圖。第二個參數是您希望迭代的陣列或集合，而第三個參數是在視圖中分配給當前迭代的變數名稱。因此，例如，如果您正在迭代 `jobs` 陣列，通常您會希望在視圖中將每個 job 作為 `job` 變數存取。當前迭代的陣列鍵將作為 `key` 變數在視圖中可用。

您還可以向 `@each` 指令傳遞第四個參數。此參數決定了如果給定陣列為空時將渲染的視圖。

```blade
@each('view.name', $jobs, 'job', 'view.empty')
```

> [!WARNING]  
> 透過 `@each` 渲染的視圖不會繼承父視圖中的變數。如果子視圖需要這些變數，您應該改用 `@foreach` 和 `@include` 指令。


<a name="the-once-directive"></a>
### `@once` 指令

`@once` 指令允許您定義模板中僅在每個渲染週期中評估一次的部分。這對於使用 [堆疊](#stacks) 將給定的 JavaScript 片段推送到頁面標頭中可能很有用。例如，如果您在迴圈中渲染給定的 [組件](#components)，您可能希望只在第一次渲染該組件時將 JavaScript 推送到標頭：

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


<a name="raw-php"></a>
### 原生 PHP

在某些情況下，將 PHP 程式碼嵌入到視圖中很有用。您可以使用 Blade 的 `@php` 指令在模板中執行一段純 PHP 程式碼區塊：

```blade
@php
    $counter = 1;
@endphp
```

或者，如果您只需要使用 PHP 導入一個類別，您可以使用 `@use` 指令：

```blade
@use('App\Models\Flight')
```

可以為 `@use` 指令提供第二個參數來為導入的類別指定別名：

```php
@use('App\Models\Flight', 'FlightModel')
```


<a name="comments"></a>
### 註解

Blade 還允許您在視圖中定義註解。但是，與 HTML 註解不同，Blade 註解不會包含在應用程式返回的 HTML 中：

```blade
{{-- This comment will not be present in the rendered HTML --}}
```

<a name="components"></a>
## 組件

組件和插槽提供的優勢與區塊、佈局和包含類似；然而，有些人可能會覺得組件和插槽的心智模型更容易理解。編寫組件有兩種方法：類別型組件和匿名組件。

若要建立類別型組件，您可以使用 `make:component` Artisan 命令。為了說明如何使用組件，我們將建立一個簡單的 `Alert` 組件。`make:component` 命令會將組件放置在 `app/View/Components` 目錄中：

```shell
php artisan make:component Alert
```

`make:component` 命令也會為組件建立一個視圖模板。該視圖將會放置在 `resources/views/components` 目錄中。當您為自己的應用程式編寫組件時，組件會自動在 `app/View/Components` 目錄和 `resources/views/components` 目錄中被探索到，因此通常不需要進一步的組件註冊。

您也可以在子目錄中建立組件：

```shell
php artisan make:component Forms/Input
```

上述命令將在 `app/View/Components/Forms` 目錄中建立一個 `Input` 組件，並且該視圖將會放置在 `resources/views/components/forms` 目錄中。

如果您想建立一個匿名組件 (一個只有 Blade 模板而沒有類別的組件)，您可以在呼叫 `make:component` 命令時使用 `--view` 旗標：

```shell
php artisan make:component forms.input --view
```

上述命令將在 `resources/views/components/forms/input.blade.php` 建立一個 Blade 檔案，該檔案可以透過 `<x-forms.input />` 以組件形式渲染。


<a name="manually-registering-package-components"></a>
#### 手動註冊套件組件

當您為自己的應用程式編寫組件時，組件會自動在 `app/View/Components` 目錄和 `resources/views/components` 目錄中被探索到。

然而，如果您正在建構一個使用 Blade 組件的套件，您將需要手動註冊您的組件類別及其 HTML 標籤別名。您通常應該在套件的服務提供者的 `boot` 方法中註冊您的組件：

    use Illuminate\Support\Facades\Blade;

    /**
     * Bootstrap your package's services.
     */
    public function boot(): void
    {
        Blade::component('package-alert', Alert::class);
    }

一旦您的組件已註冊，就可以使用其標籤別名進行渲染：

```blade
<x-package-alert/>
```

或者，您可以使用 `componentNamespace` 方法透過慣例自動載入組件類別。例如，一個 `Nightshade` 套件可能會有 `Calendar` 和 `ColorPicker` 組件，它們位於 `Package\Views\Components` 命名空間中：

    use Illuminate\Support\Facades\Blade;

    /**
     * Bootstrap your package's services.
     */
    public function boot(): void
    {
        Blade::componentNamespace('Nightshade\\Views\\Components', 'nightshade');
    }

這將允許透過其供應商命名空間使用套件組件，並採用 `package-name::` 語法：

```blade
<x-nightshade::calendar />
<x-nightshade::color-picker />
```

Blade 會透過將組件名稱 Pascal-Case 化來自動偵測連結到此組件的類別。也支援使用「點號」表示法的子目錄。


<a name="rendering-components"></a>
### 渲染組件

若要顯示組件，您可以在您的其中一個 Blade 模板中使用 Blade 組件標籤。Blade 組件標籤以字串 `x-` 開頭，後接組件類別的 kebab-case 名稱：

```blade
<x-alert/>

<x-user-profile/>
```

如果組件類別巢狀在 `app/View/Components` 目錄中更深層，您可以使用 `.` 字元來表示目錄巢狀結構。例如，如果我們假設一個組件位於 `app/View/Components/Inputs/Button.php`，我們可以這樣渲染它：

```blade
<x-inputs.button/>
```

如果您想有條件地渲染您的組件，您可以在組件類別上定義一個 `shouldRender` 方法。如果 `shouldRender` 方法回傳 `false`，則組件將不會被渲染：

    use Illuminate\Support\Str;

    /**
     * Whether the component should be rendered
     */
    public function shouldRender(): bool
    {
        return Str::length($this->message) > 0;
    }


<a name="index-components"></a>
### 索引組件

有時候，組件是一個組件群組的一部分，您可能希望將相關組件分組到單一目錄中。例如，想像一個具有以下類別結構的「卡片」組件：

```none
App\Views\Components\Card\Card
App\Views\Components\Card\Header
App\Views\Components\Card\Body
```

由於根 `Card` 組件巢狀在 `Card` 目錄中，您可能會預期需要透過 `<x-card.card>` 來渲染該組件。然而，當組件的檔案名稱與組件的目錄名稱相符時，Laravel 會自動假設該組件是「根」組件，並允許您無須重複目錄名稱即可渲染該組件：

```blade
<x-card>
    <x-card.header>...</x-card.header>
    <x-card.body>...</x-card.body>
</x-card>
```

<a name="passing-data-to-components"></a>
### 傳遞資料至組件

您可以使用 HTML 屬性將資料傳遞給 Blade 組件。硬式編碼(hard-coded)的原生值(primitive values)可以使用簡單的 HTML 屬性字串傳遞給組件。PHP 表達式和變數應透過使用 `:` 字元作為前綴的屬性傳遞給組件：

```blade
<x-alert type="error" :message="$message"/>
```

您應該在組件的 Class 建構子中定義所有組件的資料屬性。組件上的所有 Public 屬性都會自動提供給組件的視圖。無需從組件的 `render` 方法將資料傳遞到視圖：

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

渲染組件時，您可以透過名稱印出組件 Public 變數的內容：

```blade
<div class="alert alert-{{ $type }}">
    {{ $message }}
</div>
```

<a name="casing"></a>
#### 大小寫慣例

組件建構子的參數應使用 `camelCase` 指定，而在 HTML 屬性中引用參數名稱時應使用 `kebab-case`。例如，給定以下組件建構子：

    /**
     * Create the component instance.
     */
    public function __construct(
        public string $alertType,
    ) {}

`$alertType` 參數可以像這樣提供給組件：

```blade
<x-alert alert-type="danger" />
```

<a name="short-attribute-syntax"></a>
#### 短屬性語法

將屬性傳遞給組件時，您也可以使用「短屬性」語法。這通常很方便，因為屬性名稱經常與它們對應的變數名稱相符：

```blade
{{-- Short attribute syntax... --}}
<x-profile :$userId :$name />

{{-- Is equivalent to... --}}
<x-profile :user-id="$userId" :name="$name" />
```

<a name="escaping-attribute-rendering"></a>
#### 跳脫屬性渲染

由於一些 JavaScript 框架（例如 Alpine.js）也使用冒號前綴屬性，因此您可以使用雙冒號 (`::`) 前綴來告知 Blade 該屬性不是 PHP 表達式。例如，給定以下組件：

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
#### 組件方法

除了 Public 變數可供組件模板使用外，組件上的任何 Public 方法也可以被呼叫。例如，假設有一個組件具有 `isSelected` 方法：

    /**
     * Determine if the given option is the currently selected option.
     */
    public function isSelected(string $option): bool
    {
        return $option === $this->selected;
    }

您可以透過呼叫與該方法名稱相符的變數，從組件模板執行此方法：

```blade
<option {{ $isSelected($value) ? 'selected' : '' }} value="{{ $value }}">
    {{ $label }}
</option>
```

<a name="using-attributes-slots-within-component-class"></a>
#### 在組件 Class 中存取屬性與插槽

Blade 組件也允許您在 Class 的 render 方法中存取組件名稱、屬性及插槽。然而，為了存取這些資料，您應該從組件的 `render` 方法返回一個閉包(Closure)：

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

組件的 `render` 方法返回的閉包也可以接收一個 `$data` 陣列作為其唯一參數。這個陣列將包含多個提供組件資訊的元素：

    return function (array $data) {
        // $data['componentName'];
        // $data['attributes'];
        // $data['slot'];

        return '<div {{ $attributes }}>Components content</div>';
    }

> [!WARNING]
> `$data` 陣列中的元素不應直接嵌入到您的 `render` 方法返回的 Blade 字串中，因為這樣做可能會透過惡意屬性內容導致遠端程式碼執行。

`componentName` 等於 HTML 標籤中 `x-` 前綴後使用的名稱。因此，`<x-alert />` 的 `componentName` 將是 `alert`。`attributes` 元素將包含 HTML 標籤上存在的所有屬性。`slot` 元素是一個包含組件插槽內容的 `Illuminate\Support\HtmlString` 實例。

閉包應該返回一個字串。如果返回的字串對應於現有的視圖，該視圖將被渲染；否則，返回的字串將被評估為行內 Blade 視圖。

<a name="additional-dependencies"></a>
#### 額外依賴

如果您的組件需要 Laravel 的 [服務容器](/docs/{{version}}/container) 中的依賴項，您可以將它們列在組件的任何資料屬性之前，它們將自動由容器注入：

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

如果您想防止某些 Public 方法或屬性作為變數暴露給組件模板，您可以將它們添加到組件的 `$except` 陣列屬性中：

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

<a name="component-attributes"></a>
### 組件屬性

我們已經檢視過如何將資料屬性傳遞給組件；然而，有時候您可能需要指定額外的 HTML 屬性，例如 `class`，這些屬性並非組件運作所需的資料。通常，您會希望將這些額外屬性傳遞到組件模板的根元素。舉例來說，假設我們想要渲染一個 `alert` 組件，如下所示：

```blade
<x-alert type="error" :message="$message" class="mt-4"/>
```

所有不屬於組件建構函式的屬性，都將自動添加到組件的「屬性包」中。此屬性包會透過 `$attributes` 變數自動提供給組件。所有屬性都可以透過印出這個變數在組件中渲染：

```blade
<div {{ $attributes }}>
    <!-- Component content -->
</div>
```

> [!WARNING]  
> 使用 `@env` 等指令於組件標籤內目前不支援。例如，`<x-alert :live="@env('production')"/>` 將不會被編譯。

<a name="default-merged-attributes"></a>
#### 預設 / 合併屬性

有時候您可能需要為屬性指定預設值，或將額外的值合併到組件的某些屬性中。為此，您可以使用屬性包的 `merge` 方法。此方法對於定義一組應始終應用於組件的預設 CSS class 特別有用：

```blade
<div {{ $attributes->merge(['class' => 'alert alert-'.$type]) }}>
    {{ $message }}
</div>
```

如果我們假設這個組件是這樣使用的：

```blade
<x-alert type="error" :message="$message" class="mb-4"/>
```

組件最終渲染的 HTML 將會像下面這樣：

```blade
<div class="alert alert-error mb-4">
    <!-- Contents of the $message variable -->
</div>
```

<a name="conditionally-merge-classes"></a>
#### 條件式合併 Class

有時候，您可能會希望在給定條件為 `true` 時合併 class。您可以透過 `class` 方法來實現此目的，該方法接受一個 class 陣列，其中陣列鍵包含您希望添加的一個或多個 class，而值則是一個布林表達式。如果陣列元素具有數值鍵，它將始終包含在渲染後的 class 列表中：

```blade
<div {{ $attributes->class(['p-4', 'bg-red' => $hasError]) }}>
    {{ $message }}
</div>
```

如果您需要將其他屬性合併到組件上，您可以將 `merge` 方法鏈接到 `class` 方法上：

```blade
<button {{ $attributes->class(['p-4'])->merge(['type' => 'button']) }}>
    {{ $slot }}
</button>
```

> [!NOTE]  
> 如果您需要在不應接收合併屬性的其他 HTML 元素上條件式編譯 class，可以使用 [`@class` 指令](#conditional-classes)。

<a name="non-class-attribute-merging"></a>
#### 非 Class 屬性合併

當合併非 `class` 屬性時，提供給 `merge` 方法的值將被視為該屬性的「預設」值。然而，與 `class` 屬性不同，這些屬性不會與注入的屬性值合併。相反地，它們將被覆寫。舉例來說，一個 `button` 組件的實作可能如下所示：

```blade
<button {{ $attributes->merge(['type' => 'button']) }}>
    {{ $slot }}
</button>
```

若要以自訂的 `type` 渲染 button 組件，可以在使用該組件時指定。如果沒有指定 type，則會使用 `button` type：

```blade
<x-button type="submit">
    Submit
</x-button>
```

這個範例中 `button` 組件渲染後的 HTML 將會是：

```blade
<button type="submit">
    Submit
</button>
```

如果您希望除了 `class` 之外的屬性，其預設值和注入值能結合在一起，您可以使用 `prepends` 方法。在這個範例中，`data-controller` 屬性將始終以 `profile-controller` 開頭，並且任何額外注入的 `data-controller` 值將會放在此預設值之後：

```blade
<div {{ $attributes->merge(['data-controller' => $attributes->prepends('profile-controller')]) }}>
    {{ $slot }}
</div>
```

<a name="filtering-attributes"></a>
#### 擷取與過濾屬性

您可以使用 `filter` 方法來過濾屬性。這個方法接受一個閉包，如果您希望將屬性保留在屬性包中，該閉包應回傳 `true`：

```blade
{{ $attributes->filter(fn (string $value, string $key) => $key == 'foo') }}
```

為方便起見，您可以使用 `whereStartsWith` 方法來擷取所有鍵以給定字串開頭的屬性：

```blade
{{ $attributes->whereStartsWith('wire:model') }}
```

反之，`whereDoesntStartWith` 方法可用於排除所有鍵以給定字串開頭的屬性：

```blade
{{ $attributes->whereDoesntStartWith('wire:model') }}
```

使用 `first` 方法，您可以渲染給定屬性包中的第一個屬性：

```blade
{{ $attributes->whereStartsWith('wire:model')->first() }}
```

如果您想檢查組件上是否存在某個屬性，可以使用 `has` 方法。這個方法接受屬性名稱作為其唯一參數，並回傳一個布林值，指示該屬性是否存在：

```blade
@if ($attributes->has('class'))
    <div>Class attribute is present</div>
@endif
```

如果將陣列傳遞給 `has` 方法，該方法將判斷組件上是否具備所有給定的屬性：

```blade
@if ($attributes->has(['name', 'class']))
    <div>All of the attributes are present</div>
@endif
```

`hasAny` 方法可用來判斷組件上是否存在任何給定的屬性：

```blade
@if ($attributes->hasAny(['href', ':href', 'v-bind:href']))
    <div>One of the attributes is present</div>
@endif
```

您可以使用 `get` 方法擷取特定屬性的值：

```blade
{{ $attributes->get('class') }}
```

<a name="reserved-keywords"></a>
### 保留關鍵字

預設情況下，一些關鍵字被 Blade 保留用於內部使用，以便渲染組件。以下關鍵字不能在您的組件中定義為 Public 屬性或方法名稱：

<div class="content-list" markdown="1">

- `data`
- `render`
- `resolveView`
- `shouldRender`
- `view`
- `withAttributes`
- `withName`

</div>

<a name="slots"></a>
### 插槽

您經常需要透過「插槽」向您的組件傳遞額外內容。組件的插槽是透過輸出 `$slot` 變數來渲染的。為了探討這個概念，讓我們想像一個 `alert` 組件具有以下標記：

```blade
<!-- /resources/views/components/alert.blade.php -->

<div class="alert alert-danger">
    {{ $slot }}
</div>
```

我們可以透過向組件注入內容來傳遞內容至 `slot`。

```blade
<x-alert>
    <strong>Whoops!</strong> Something went wrong!
</x-alert>
```

有時，組件可能需要在組件內的不同位置渲染多個不同的插槽。讓我們修改警示組件，以允許注入一個「title」插槽：

```blade
<!-- /resources/views/components/alert.blade.php -->

<span class="alert-title">{{ $title }}</span>

<div class="alert alert-danger">
    {{ $slot }}
</div>
```

您可以使用 `x-slot` 標籤定義命名插槽的內容。任何不在明確的 `x-slot` 標籤內的內容都將以 `$slot` 變數的形式傳遞給組件：

```xml
<x-alert>
    <x-slot:title>
        Server Error
    </x-slot>

    <strong>Whoops!</strong> Something went wrong!
</x-alert>
```

您可以調用插槽的 `isEmpty` 方法來判斷該插槽是否包含內容：

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
#### 作用域插槽

如果您使用過諸如 Vue 的 JavaScript 框架，您可能熟悉「作用域插槽」，它允許您在插槽內存取組件的資料或方法。您可以透過在組件上定義公開方法或屬性，並透過 `$component` 變數在插槽內存取組件，以在 Laravel 中實現類似行為。在此範例中，我們將假設 `x-alert` 組件在其組件類別中定義了一個公開的 `formatAlert` 方法：

```blade
<x-alert>
    <x-slot:title>
        {{ $component->formatAlert('Server Error') }}
    </x-slot>

    <strong>Whoops!</strong> Something went wrong!
</x-alert>
```

<a name="slot-attributes"></a>
#### 插槽屬性

如同 Blade 組件，您可以為插槽分配額外的 [屬性](#component-attributes)，例如 CSS class 名稱：

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

要與插槽屬性互動，您可以存取插槽變數的 `attributes` 屬性。有關如何與屬性互動的更多資訊，請參閱 [組件屬性](#component-attributes) 文件：

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
### 行內組件視圖

對於非常小的組件，同時管理組件類別和組件的視圖模板可能會感到繁瑣。因此，您可以直接從 `render` 方法中回傳組件的標記：

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

<a name="generating-inline-view-components"></a>
#### 生成行內視圖組件

若要創建一個渲染行內視圖的組件，您可以在執行 `make:component` 命令時使用 `inline` 選項：

```shell
php artisan make:component Alert --inline
```

<a name="dynamic-components"></a>
### 動態組件

有時您可能需要渲染一個組件，但直到運行時才知道應該渲染哪個組件。在這種情況下，您可以使用 Laravel 內建的 `dynamic-component` 組件，根據運行時值或變數來渲染組件：

```blade
// $componentName = "secondary-button";

<x-dynamic-component :component="$componentName" class="mt-4" />
```

<a name="manually-registering-components"></a>
### 手動註冊組件

[!WARNING]
以下關於手動註冊組件的文件主要適用於撰寫包含視圖組件的 Laravel 套件的開發人員。如果您不是在撰寫套件，本部分組件文件可能與您無關。

當為您自己的應用程式撰寫組件時，組件會自動在 `app/View/Components` 目錄和 `resources/views/components` 目錄中被發現。

然而，如果您正在建立一個使用 Blade 組件的套件，或將組件放置在非傳統目錄中，您需要手動註冊組件類別及其 HTML 標籤別名，以便 Laravel 知道在哪裡找到該組件。您通常應該在套件的服務提供者的 `boot` 方法中註冊您的組件：

    use Illuminate\Support\Facades\Blade;
    use VendorPackage\View\Components\AlertComponent;

    /**
     * Bootstrap your package's services.
     */
    public function boot(): void
    {
        Blade::component('package-alert', AlertComponent::class);
    }

一旦您的組件已註冊，即可使用其標籤別名來渲染：

```blade
<x-package-alert/>
```

#### 自動載入套件組件

或者，您可以使用 `componentNamespace` 方法，依照慣例自動載入組件類別。例如，一個 `Nightshade` 套件可能擁有位於 `Package\Views\Components` 命名空間內的 `Calendar` 和 `ColorPicker` 組件：

    use Illuminate\Support\Facades\Blade;

    /**
     * Bootstrap your package's services.
     */
    public function boot(): void
    {
        Blade::componentNamespace('Nightshade\\Views\\Components', 'nightshade');
    }

這將允許使用 `package-name::` 語法透過其供應商命名空間來使用套件組件：

```blade
<x-nightshade::calendar />
<x-nightshade::color-picker />
```

Blade 將透過 Pascal-casing (帕斯卡命名法) 組件名稱來自動偵測與此組件連結的類別。也支援使用「點」符號的子目錄。

<a name="anonymous-components"></a>
## 匿名組件

類似於行內組件，匿名組件提供了一種透過單一檔案管理組件的機制。然而，匿名組件只使用一個視圖檔案，並且沒有關聯的類別。要定義一個匿名組件，您只需將 Blade 模板檔案放置在 `resources/views/components` 目錄中。例如，假設您在 `resources/views/components/alert.blade.php` 定義了一個組件，您可以這樣簡單地渲染它：

```blade
<x-alert/>
```

您可以使用 `.` 字元來表示組件是否在 `components` 目錄中更深層次地巢狀。例如，假設組件定義在 `resources/views/components/inputs/button.blade.php`，您可以這樣渲染它：

```blade
<x-inputs.button/>
```

<a name="anonymous-index-components"></a>
### 匿名索引組件

有時候，當一個組件由多個 Blade 模板組成時，您可能會希望將該組件的模板群組到一個單一目錄中。例如，想像一個「手風琴」組件，其目錄結構如下：

```none
/resources/views/components/accordion.blade.php
/resources/views/components/accordion/item.blade.php
```

這種目錄結構允許您像這樣渲染手風琴組件及其項目：

```blade
<x-accordion>
    <x-accordion.item>
        ...
    </x-accordion.item>
</x-accordion>
```

然而，為了透過 `x-accordion` 渲染手風琴組件，我們被迫將「索引」手風琴組件模板放置在 `resources/views/components` 目錄中，而非與其他手風琴相關模板一起巢狀在 `accordion` 目錄中。

幸運的是，Blade 允許您將與組件目錄名稱相符的檔案放置在組件目錄本身中。當此模板存在時，即使它巢狀在一個目錄中，它也可以作為組件的「根」元素被渲染。因此，我們可以繼續使用上述範例中相同的 Blade 語法；然而，我們將像這樣調整我們的目錄結構：

```none
/resources/views/components/accordion/accordion.blade.php
/resources/views/components/accordion/item.blade.php
```

<a name="data-properties-attributes"></a>
### 資料屬性 / Attributes

由於匿名組件沒有任何關聯的類別，您可能會想知道如何區分哪些資料應該作為變數傳遞給組件，以及哪些 Attributes 應該放置在組件的 [Attribute 包](#component-attributes) 中。

您可以使用 `@props` 指令指定哪些 Attributes 應該被視為資料變數，該指令位於組件的 Blade 模板頂部。組件上的所有其他 Attributes 將透過組件的 Attribute 包提供。如果您希望為資料變數設定一個預設值，您可以將變數名稱指定為陣列鍵，預設值指定為陣列值：

```blade
<!-- /resources/views/components/alert.blade.php -->

@props(['type' => 'info', 'message'])

<div {{ $attributes->merge(['class' => 'alert alert-'.$type]) }}>
    {{ $message }}
</div>
```

考慮到上述組件定義，我們可以這樣渲染組件：

```blade
<x-alert type="error" :message="$message" class="mb-4"/>
```

<a name="accessing-parent-data"></a>
### 存取父級資料

有時候您可能希望在子組件中存取父組件的資料。在這些情況下，您可以使用 `@aware` 指令。例如，想像我們正在建立一個複雜的選單組件，該組件由父級 `<x-menu>` 和子級 `<x-menu.item>` 組成：

```blade
<x-menu color="purple">
    <x-menu.item>...</x-menu.item>
    <x-menu.item>...</x-menu.item>
</x-menu>
```

`<x-menu>` 組件可能會有如下實作：

```blade
<!-- /resources/views/components/menu/index.blade.php -->

@props(['color' => 'gray'])

<ul {{ $attributes->merge(['class' => 'bg-'.$color.'-200']) }}>
    {{ $slot }}
</ul>
```

因為 `color` 屬性只傳遞給父級 (`<x-menu>`)，它將不會在 `<x-menu.item>` 內部可用。然而，如果我們使用 `@aware` 指令，我們也可以讓它在 `<x-menu.item>` 內部可用：

```blade
<!-- /resources/views/components/menu/item.blade.php -->

@aware(['color' => 'gray'])

<li {{ $attributes->merge(['class' => 'text-'.$color.'-800']) }}>
    {{ $slot }}
</li>
```

> [!WARNING]  
> `@aware` 指令無法存取未透過 HTML Attributes 明確傳遞給父組件的父級資料。未明確傳遞給父組件的預設 `@props` 值無法透過 `@aware` 指令存取。

<a name="anonymous-component-paths"></a>
### 匿名組件路徑

如前所述，匿名組件通常是透過將 Blade 模板放置在 `resources/views/components` 目錄中來定義的。然而，您偶爾可能希望除了預設路徑之外，還向 Laravel 註冊其他匿名組件路徑。

`anonymousComponentPath` 方法接受匿名組件位置的「路徑」作為其第一個參數，以及一個可選的「命名空間」作為其第二個參數，組件應放置在此命名空間下。通常，此方法應從您的應用程式的 [服務提供者](/docs/{{version}}/providers) 之一的 `boot` 方法中呼叫：

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        Blade::anonymousComponentPath(__DIR__.'/../components');
    }

當組件路徑在沒有指定前綴的情況下註冊時，如上述範例所示，它們也可以在您的 Blade 組件中不帶相應前綴地渲染。例如，如果上述註冊路徑中存在 `panel.blade.php` 組件，它可以這樣渲染：

```blade
<x-panel />
```

前綴「命名空間」可以作為 `anonymousComponentPath` 方法的第二個參數提供：

    Blade::anonymousComponentPath(__DIR__.'/../components', 'dashboard');

提供前綴時，該「命名空間」中的組件可以在渲染組件時，透過在組件名稱前加上組件的命名空間來渲染：

```blade
<x-dashboard::panel />
```

<a name="building-layouts"></a>
## 建構佈局

<a name="layouts-using-components"></a>
### 使用組件的佈局

大多數 Web 應用程式在各個頁面中都維持著相同的通用佈局。如果我們在每個建立的視圖中都必須重複整個佈局的 HTML，那麼維護我們的應用程式將會變得非常繁瑣且困難。幸好，將此佈局定義為單一的 [Blade 組件](#components)，然後在整個應用程式中使用，會非常方便。

<a name="defining-the-layout-component"></a>
#### 定義佈局組件

例如，假設我們正在建構一個「待辦事項」列表應用程式。我們可能會定義一個像這樣子的 `layout` 組件：

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
#### 套用佈局組件

一旦 `layout` 組件定義完成，我們就可以建立一個利用此組件的 Blade 視圖。在此範例中，我們將定義一個簡單的視圖來顯示我們的任務列表：

```blade
<!-- resources/views/tasks.blade.php -->

<x-layout>
    @foreach ($tasks as $task)
        <div>{{ $task }}</div>
    @endforeach
</x-layout>
```

請記住，注入組件的內容將會提供給 `layout` 組件中的預設 `$slot` 變數。您可能已經注意到，如果提供了 `$title` 插槽，我們的 `layout` 也會遵循；否則，將顯示預設標題。我們可以使用 [組件文件](#components) 中討論的標準插槽語法，從任務列表視圖注入自訂標題：

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

現在我們已經定義了佈局和任務列表視圖，我們只需要從路由回傳 `task` 視圖即可：

    use App\Models\Task;

    Route::get('/tasks', function () {
        return view('tasks', ['tasks' => Task::all()]);
    });

<a name="layouts-using-template-inheritance"></a>
### 使用模板繼承的佈局

<a name="defining-a-layout"></a>
#### 定義佈局

佈局也可以透過「模板繼承」來建立。在 [組件](#components) 引入之前，這是建構應用程式的主要方式。

首先，讓我們看一個簡單的範例。首先，我們將檢視一個頁面佈局。由於大多數 Web 應用程式在各個頁面中都維持相同的通用佈局，因此將此佈局定義為單一的 Blade 視圖會很方便：

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

如您所見，此檔案包含典型的 HTML 標記。然而，請注意 `@section` 和 `@yield` 指令。`@section` 指令顧名思義，定義了一個內容區塊，而 `@yield` 指令則用於顯示指定區塊的內容。

現在我們已經為應用程式定義了佈局，接下來定義一個繼承此佈局的子頁面。

<a name="extending-a-layout"></a>
#### 擴充佈局

在定義子視圖時，請使用 `@extends` Blade 指令來指定子視圖應該「繼承」哪個佈局。擴充 Blade 佈局的視圖可以使用 `@section` 指令將內容注入到佈局的區塊中。請記住，如上例所示，這些區塊的內容將使用 `@yield` 在佈局中顯示：

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

在此範例中，`sidebar` 區塊利用 `@@parent` 指令將內容附加（而非覆寫）到佈局的側邊欄。當視圖渲染時，`@@parent` 指令將被佈局的內容替換。

> [!NOTE]  
> 與之前的範例相反，這個 `sidebar` 區塊是以 `@endsection` 結尾，而不是 `@show`。`@endsection` 指令只會定義一個區塊，而 `@show` 則會定義並**立即輸出 (yield)** 該區塊。

`@yield` 指令也接受一個預設值作為其第二個參數。如果要輸出的區塊未定義，則此值將會被渲染：

```blade
@yield('content', 'Default content')
```

<a name="forms"></a>
## 表單

<a name="csrf-field"></a>
### CSRF 欄位

任何時候您在應用程式中定義 HTML 表單時，都應該在表單中包含一個隱藏的 CSRF 權杖欄位，以便 [CSRF 保護](/docs/{{version}}/csrf) 中介層可以驗證請求。您可以使用 `@csrf` Blade 指令來生成權杖欄位：

```blade
<form method="POST" action="/profile">
    @csrf

    ...
</form>
```

<a name="method-field"></a>
### Method 欄位

由於 HTML 表單無法發出 `PUT`、`PATCH` 或 `DELETE` 請求，您需要添加一個隱藏的 `_method` 欄位來偽造這些 HTTP 動詞。`@method` Blade 指令可以為您建立此欄位：

```blade
<form action="/foo/bar" method="POST">
    @method('PUT')

    ...
</form>
```

<a name="validation-errors"></a>
### 驗證錯誤

`@error` 指令可用於快速檢查給定屬性是否存在 [驗證錯誤訊息](/docs/{{version}}/validation#quick-displaying-the-validation-errors)。在 `@error` 指令中，您可以輸出 `$message` 變數來顯示錯誤訊息：

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

由於 `@error` 指令會被編譯成一個「if」語句，因此當屬性沒有錯誤時，您可以使用 `@else` 指令來渲染內容：

```blade
<!-- /resources/views/auth.blade.php -->

<label for="email">Email address</label>

<input
    id="email"
    type="email"
    class="@error('email') is-invalid @else is-valid @enderror"
/>
```

您可以將 [特定錯誤袋的名稱](/docs/{{version}}/validation#named-error-bags) 作為第二個參數傳遞給 `@error` 指令，以在包含多個表單的頁面上檢索驗證錯誤訊息：

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
## 堆疊

Blade 允許您將內容推送到具名堆疊中，這些堆疊可以在另一個視圖或佈局中渲染。這對於指定子視圖所需的任何 JavaScript 函式庫特別有用：

```blade
@push('scripts')
    <script src="/example.js"></script>
@endpush
```

如果您想在給定的布林運算式評估為 `true` 時 `@push` 內容，您可以使用 `@pushIf` 指令：

```blade
@pushIf($shouldPush, 'scripts')
    <script src="/example.js"></script>
@endPushIf
```

您可以根據需要多次推送到堆疊。若要渲染完整的堆疊內容，請將堆疊名稱傳遞給 `@stack` 指令：

```blade
<head>
    <!-- Head Contents -->

    @stack('scripts')
</head>
```

如果您想將內容前置到堆疊的開頭，您應該使用 `@prepend` 指令：

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

`@inject` 指令可用來從 Laravel 的 [service container](/docs/{{version}}/container) 中獲取服務。傳遞給 `@inject` 的第一個參數是服務將被放入的變數名稱，而第二個參數是你希望解析的服務的 class 或介面名稱：

```blade
@inject('metrics', 'App\Services\MetricsService')

<div>
    Monthly Revenue: {{ $metrics->monthlyRevenue() }}.
</div>
```

<a name="rendering-inline-blade-templates"></a>
## 渲染行內 Blade 模板

有時你可能需要將原始的 Blade 模板字串轉換為有效的 HTML。你可以使用 `Blade` Facade 提供的 `render` 方法來實現。`render` 方法接受 Blade 模板字串，以及一個可選的陣列資料來提供給模板：

```php
use Illuminate\Support\Facades\Blade;

return Blade::render('Hello, {{ $name }}', ['name' => 'Julian Bashir']);
```

Laravel 會將行內 Blade 模板寫入 `storage/framework/views` 目錄進行渲染。如果你希望 Laravel 在渲染 Blade 模板後刪除這些臨時檔案，你可以向該方法提供 `deleteCachedView` 參數：

```php
return Blade::render(
    'Hello, {{ $name }}',
    ['name' => 'Julian Bashir'],
    deleteCachedView: true
);
```

<a name="rendering-blade-fragments"></a>
## 渲染 Blade 片段

當使用像 [Turbo](https://turbo.hotwired.dev/) 和 [htmx](https://htmx.org/) 這類前端框架時，你可能偶爾需要在 HTTP 響應中只返回 Blade 模板的一部分。Blade「fragments (片段)」便能達成此目的。要開始使用，請將 Blade 模板的一部分放在 `@fragment` 和 `@endfragment` 指令之間：

```blade
@fragment('user-list')
    <ul>
        @foreach ($users as $user)
            <li>{{ $user->name }}</li>
        @endforeach
    </ul>
@endfragment
```

然後，當渲染使用此模板的視圖時，你可以調用 `fragment` 方法來指定只應將指定的片段包含在輸出的 HTTP 響應中：

```php
return view('dashboard', ['users' => $users])->fragment('user-list');
```

`fragmentIf` 方法允許你根據給定的條件有條件地返回視圖的片段。否則，將返回整個視圖：

```php
return view('dashboard', ['users' => $users])
    ->fragmentIf($request->hasHeader('HX-Request'), 'user-list');
```

`fragments` 和 `fragmentsIf` 方法允許你在響應中返回多個視圖片段。這些片段將會被串聯在一起：

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

Blade 允許你使用 `directive` 方法定義自己的自訂指令。當 Blade 編譯器遇到自訂指令時，它會調用提供的回調函數，並傳入該指令所包含的表達式。

以下範例創建了一個 `@datetime($var)` 指令，用於格式化給定的 `$var`，該變數應為 `DateTime` 實例：

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

如你所見，我們將 `format` 方法鏈接到傳遞給指令的任何表達式上。因此，在此範例中，此指令生成的最終 PHP 將是：

    <?php echo ($var)->format('m/d/Y H:i'); ?>

> [!WARNING]  
> 更新 Blade 指令的邏輯後，你需要刪除所有快取的 Blade 視圖。快取的 Blade 視圖可以使用 `view:clear` Artisan 命令刪除。

<a name="custom-echo-handlers"></a>
### 自訂 Echo 處理器

如果你嘗試使用 Blade「echo」一個物件，該物件的 `__toString` 方法將會被調用。[`__toString`](https://www.php.net/manual/en/language.oop5.magic.php#object.tostring) 方法是 PHP 內建的「魔術方法」之一。然而，有時你可能無法控制給定類別的 `__toString` 方法，例如當你正在互動的類別屬於第三方函式庫時。

在這些情況下，Blade 允許你為該特定類型的物件註冊一個自訂的 echo 處理器。為此，你應該調用 Blade 的 `stringable` 方法。`stringable` 方法接受一個 Closure。這個 Closure 應該型別提示其負責渲染的物件類型。通常，`stringable` 方法應在應用程式 `AppServiceProvider` 類別的 `boot` 方法中調用：

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

一旦定義了你的自訂 echo 處理器，你就可以在 Blade 模板中直接 echo 該物件：

```blade
Cost: {{ $money }}
```

<a name="custom-if-statements"></a>
### 自訂 If 語句

在定義簡單的自訂條件語句時，編寫自訂指令有時會比必要複雜。因此，Blade 提供了一個 `Blade::if` 方法，允許你使用 Closure 快速定義自訂條件指令。例如，讓我們定義一個自訂條件，檢查應用程式設定的預設「disk」。我們可以在 `AppServiceProvider` 的 `boot` 方法中完成此操作：

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

一旦定義了自訂條件，你就可以在你的模板中使用它：

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