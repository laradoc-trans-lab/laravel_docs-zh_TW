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
    - [條件式類別](#conditional-classes)
    - [額外屬性](#additional-attributes)
    - [引入子視圖](#including-subviews)
    - [` @once` 指令](#the-once-directive)
    - [原始 PHP](#raw-php)
    - [註解](#comments)
- [元件](#components)
    - [渲染元件](#rendering-components)
    - [索引元件](#index-components)
    - [傳遞資料給元件](#passing-data-to-components)
    - [元件屬性](#component-attributes)
    - [保留關鍵字](#reserved-keywords)
    - [Slot](#slots)
    - [行內元件視圖](#inline-component-views)
    - [動態元件](#dynamic-components)
    - [手動註冊元件](#manually-registering-components)
- [匿名元件](#anonymous-components)
    - [匿名索引元件](#anonymous-index-components)
    - [資料屬性 / 特性](#data-properties-attributes)
    - [存取父層資料](#accessing-parent-data)
    - [匿名元件路徑](#anonymous-component-paths)
- [建構版面](#building-layouts)
    - [使用元件的版面](#layouts-using-components)
    - [使用模板繼承的版面](#layouts-using-template-inheritance)
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

Blade 是 Laravel 內建的簡單卻強大的模板引擎。與某些 PHP 模板引擎不同，Blade 不會限制您在模板中使用純 PHP 程式碼。事實上，所有 Blade 模板都會被編譯成純 PHP 程式碼並快取起來，直到它們被修改為止，這意味著 Blade 幾乎不會為您的應用程式增加任何額外負擔。Blade 模板檔案使用 `.blade.php` 副檔名，通常儲存在 `resources/views` 目錄中。

Blade 視圖可以使用全域的 `view` 輔助函式從路由或控制器中返回。當然，如 [視圖](/docs/{{version}}/views) 說明文件所述，資料可以使用 `view` 輔助函式的第二個參數傳遞給 Blade 視圖：

    Route::get('/', function () {
        return view('greeting', ['name' => 'Finn']);
    });

<a name="supercharging-blade-with-livewire"></a>
### 使用 Livewire 強化 Blade

想要將您的 Blade 模板提升到新的層次，並輕鬆建構動態介面嗎？請參考 [Laravel Livewire](https://livewire.laravel.com)。Livewire 讓您能夠編寫 Blade 元件，並透過動態功能進行增強，這些功能通常只能透過 React 或 Vue 等前端框架實現，為建構現代、響應式前端提供了一種絕佳的方法，而無需許多 JavaScript 框架的複雜性、客戶端渲染或建構步驟。

<a name="displaying-data"></a>
## 顯示資料

您可以透過將變數包裝在大括號中來顯示傳遞給 Blade 視圖的資料。例如，給定以下路由：

    Route::get('/', function () {
        return view('welcome', ['name' => 'Samantha']);
    });

您可以像這樣顯示 `$name` 變數的內容：

```blade
Hello, {{ $name }}.
```

> [!NOTE]
> Blade 的 `{{ }}` Echo 語句會自動透過 PHP 的 `htmlspecialchars` 函式處理，以防止 XSS 攻擊。

您不限於顯示傳遞給視圖的變數內容。您也可以 Echo 任何 PHP 函式的結果。事實上，您可以將任何您想要的 PHP 程式碼放入 Blade Echo 語句中：

```blade
The current UNIX timestamp is {{ time() }}.
```

<a name="html-entity-encoding"></a>
### HTML 實體編碼

預設情況下，Blade (以及 Laravel 的 `e` 函式) 會對 HTML 實體進行雙重編碼。如果您想禁用雙重編碼，請從 `AppServiceProvider` 的 `boot` 方法中呼叫 `Blade::withoutDoubleEncoding` 方法：

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

預設情況下，Blade 的 `{{ }}` 語句會自動透過 PHP 的 `htmlspecialchars` 函式處理，以防止 XSS 攻擊。如果您不希望您的資料被逸出，您可以使用以下語法：

```blade
Hello, {!! $name !!}.
```

> [!WARNING]
> 在 Echo 應用程式使用者提供的內容時請務必小心。通常，您應該使用逸出的雙大括號語法來防止顯示使用者提供的資料時發生 XSS 攻擊。

<a name="blade-and-javascript-frameworks"></a>
### Blade 與 JavaScript 框架

由於許多 JavaScript 框架也使用「大括號」來表示應在瀏覽器中顯示給定表達式，因此您可以使用 ` @` 符號來告知 Blade 渲染引擎某個表達式應保持不變。例如：

```blade
<h1>Laravel</h1>

Hello, @{{ name }}.
```

在此範例中，` @` 符號將被 Blade 移除；但是，`{{ name }}` 表達式將保持不變，允許其由您的 JavaScript 框架渲染。

` @` 符號也可以用於逸出 Blade 指令：

```blade
{{-- Blade template --}}
 @@if()

<!-- HTML output -->
 @if()
```

<a name="rendering-json"></a>
#### 渲染 JSON

有時您可能會將一個陣列傳遞給您的視圖，目的是將其渲染為 JSON 以初始化 JavaScript 變數。例如：

```blade
<script>
    var app = <?php echo json_encode($array); ?>;
</script>
```

然而，您可以使用 `Illuminate\Support\Js::from` 方法指令，而不是手動呼叫 `json_encode`。`from` 方法接受與 PHP 的 `json_encode` 函式相同的參數；但是，它將確保生成的 JSON 已正確逸出，以便包含在 HTML 引號中。`from` 方法將返回一個字串 `JSON.parse` JavaScript 語句，該語句將給定的物件或陣列轉換為有效的 JavaScript 物件：

```blade
<script>
    var app = {{ Illuminate\Support\Js::from($array) }};
</script>
```

最新版本的 Laravel 應用程式骨架包含一個 `Js` Facade，它在您的 Blade 模板中提供了對此功能的便捷存取：

```blade
<script>
    var app = {{ Js::from($array) }};
</script>
```

> [!WARNING]
> 您應該只使用 `Js::from` 方法將現有變數渲染為 JSON。Blade 模板基於正規表達式，嘗試將複雜表達式傳遞給指令可能會導致意外失敗。

<a name="the-at-verbatim-directive"></a>
#### ` @verbatim` 指令

如果您在模板的很大一部分中顯示 JavaScript 變數，您可以將 HTML 包裝在 ` @verbatim` 指令中，這樣您就不必為每個 Blade Echo 語句加上 ` @` 符號：

```blade
 @verbatim
    <div class="container">
        Hello, {{ name }}.
    </div>
 @endverbatim
```

<a name="blade-directives"></a>
## Blade 指令

除了模板繼承和顯示資料之外，Blade 還為常見的 PHP 控制結構（例如條件語句和迴圈）提供了便捷的捷徑。這些捷徑提供了一種非常簡潔、精確的方式來處理 PHP 控制結構，同時也與其 PHP 對應項保持熟悉。

<a name="if-statements"></a>
### If 語句

您可以使用 ` @if`、` @elseif`、` @else` 和 ` @endif` 指令建構 `if` 語句。這些指令的功能與其 PHP 對應項完全相同：

```blade
 @if (count($records) === 1)
    I have one record!
 @elseif (count($records) > 1)
    I have multiple records!
 @else
    I don't have any records!
 @endif
```

為方便起見，Blade 還提供了 ` @unless` 指令：

```blade
 @unless (Auth::check())
    You are not signed in.
 @endunless
```

除了已經討論過的條件指令之外，` @isset` 和 ` @empty` 指令可以用作其各自 PHP 函式的便捷捷徑：

```blade
 @isset($records)
    // $records is defined and is not null...
 @endisset @empty($records)
    // $records is "empty"...
 @endempty
```

<a name="authentication-directives"></a>
#### 身份驗證指令

` @auth` 和 ` @guest` 指令可用於快速判斷目前使用者是否已 [通過身份驗證](/docs/{{version}}/authentication) 或為訪客：

```blade
 @auth
    // The user is authenticated...
 @endauth @guest
    // The user is not authenticated...
 @endguest
```

如果需要，您可以在使用 ` @auth` 和 ` @guest` 指令時指定應檢查的身份驗證 Guard：

```blade
 @auth('admin')
    // The user is authenticated...
 @endauth @guest('admin')
    // The user is not authenticated...
 @endguest
```

<a name="environment-directives"></a>
#### 環境指令

您可以使用 ` @production` 指令檢查應用程式是否在生產環境中執行：

```blade
 @production
    // Production specific content...
 @endproduction
```

或者，您可以使用 ` @env` 指令判斷應用程式是否在特定環境中執行：

```blade
 @env('staging')
    // The application is running in "staging"...
 @endenv@env(['staging', 'production'])
    // The application is running in "staging" or "production"...
 @endenv
```

<a name="section-directives"></a>
#### 區塊指令

您可以使用 ` @hasSection` 指令判斷模板繼承區塊是否有內容：

```blade
 @hasSection('navigation')
    <div class="pull-right">
        @yield('navigation')
    </div>

    <div class="clearfix"></div>
 @endif
```

您可以使用 `sectionMissing` 指令判斷區塊是否沒有內容：

```blade
 @sectionMissing('navigation')
    <div class="pull-right">
        @include('default-navigation')
    </div>
 @endif
```

<a name="session-directives"></a>
#### Session 指令

` @session` 指令可用於判斷 [Session](/docs/{{version}}/session) 值是否存在。如果 Session 值存在，` @session` 和 ` @endsession` 指令內的模板內容將會被評估。在 ` @session` 指令的內容中，您可以 Echo `$value` 變數來顯示 Session 值：

```blade
 @session('status')
    <div class="p-4 bg-green-100">
        {{ $value }}
    </div>
 @endsession
```

<a name="switch-statements"></a>
### Switch 語句

Switch 語句可以使用 ` @switch`、` @case`、` @break`、` @default` 和 ` @endswitch` 指令建構：

```blade
 @switch($i)
    @case(1)
        First case...
        @break @case(2)
        Second case...
        @break @default
        Default case...
 @endswitch
```

<a name="loops"></a>
### 迴圈

除了條件語句之外，Blade 還為 PHP 的迴圈結構提供了簡單的指令。同樣，這些指令的功能與其 PHP 對應項完全相同：

```blade
 @for ($i = 0; $i < 10; $i++)
    The current value is {{ $i }}
 @endfor @foreach ($users as $user)
    <p>This is user {{ $user->id }}</p>
 @endforeach @forelse ($users as $user)
    <li>{{ $user->name }}</li>
 @empty
    <p>No users</p>
 @endforelse @while (true)
    <p>I'm looping forever.</p>
 @endwhile
```

> [!NOTE]
> 在 `foreach` 迴圈中迭代時，您可以使用 [迴圈變數](#the-loop-variable) 來獲取有關迴圈的寶貴資訊，例如您是否在迴圈的第一次或最後一次迭代中。

使用迴圈時，您也可以使用 ` @continue` 和 ` @break` 指令跳過當前迭代或結束迴圈：

```blade
 @foreach ($users as $user)
    @if ($user->type == 1)
        @continue @endif

    <li>{{ $user->name }}</li>

    @if ($user->number == 5)
        @break @endif @endforeach
```

您也可以在指令宣告中包含繼續或中斷條件：

```blade
 @foreach ($users as $user)
    @continue($user->type == 1)

    <li>{{ $user->name }}</li>

    @break($user->number == 5)
 @endforeach
```

<a name="the-loop-variable"></a>
### 迴圈變數

在 `foreach` 迴圈中迭代時，迴圈內將提供一個 `$loop` 變數。此變數提供對一些有用資訊的存取，例如當前迴圈索引以及這是否是迴圈的第一次或最後一次迭代：

```blade
 @foreach ($users as $user)
    @if ($loop->first)
        This is the first iteration.
    @endif@if ($loop->last)
        This is the last iteration.
    @endif

    <p>This is user {{ $user->id }}</p>
 @endforeach
```

如果您在巢狀迴圈中，您可以透過 `parent` 屬性存取父迴圈的 `$loop` 變數：

```blade
 @foreach ($users as $user)
    @foreach ($user->posts as $post)
        @if ($loop->parent->first)
            This is the first iteration of the parent loop.
        @endif @endforeach @endforeach
```

`$loop` 變數還包含各種其他有用的屬性：

<div class="overflow-auto">

| 屬性           | 描述                                            |
| ------------------ | ------------------------------------------------------ |
| `$loop->index`     | 當前迴圈迭代的索引 (從 0 開始)。 |
| `$loop->iteration` | 當前迴圈迭代 (從 1 開始)。              |
| `$loop->remaining` | 迴圈中剩餘的迭代次數。                  |
| `$loop->count`     | 正在迭代的陣列中的總項目數。 |
| `$loop->first`     | 這是否是迴圈的第一次迭代。  |
| `$loop->last`      | 這是否是迴圈的最後一次迭代。   |
| `$loop->even`      | 這是否是迴圈的偶數次迭代。    |
| `$loop->odd`       | 這是否是迴圈的奇數次迭代。     |
| `$loop->depth`     | 當前迴圈的巢狀層級。                 |
| `$loop->parent`    | 在巢狀迴圈中，父迴圈的變數。     |

</div>

<a name="conditional-classes"></a>
### 條件式類別與樣式

` @class` 指令會條件式地編譯 CSS 類別字串。該指令接受一個類別陣列，其中陣列鍵包含您希望添加的類別，而值是一個布林表達式。如果陣列元素具有數字鍵，它將始終包含在渲染的類別列表中：

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

同樣，` @style` 指令可用於條件式地將行內 CSS 樣式添加到 HTML 元素：

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

為方便起見，您可以使用 ` @checked` 指令輕鬆指示給定的 HTML 複選框輸入是否「已選中」。如果提供的條件評估為 `true`，此指令將 Echo `checked`：

```blade
<input
    type="checkbox"
    name="active"
    value="active"
    @checked(old('active', $user->active))
/>
```

同樣，` @selected` 指令可用於指示給定的選取選項是否應「已選取」：

```blade
<select name="version">
    @foreach ($product->versions as $version)
        <option value="{{ $version }}" @selected(old('version') == $version)>
            {{ $version }}
        </option>
    @endforeach
</select>
```

此外，` @disabled` 指令可用於指示給定元素是否應「禁用」：

```blade
<button type="submit" @disabled($errors->isNotEmpty())>Submit</button>
```

此外，` @readonly` 指令可用於指示給定元素是否應「唯讀」：

```blade
<input
    type="email"
    name="email"
    value="email @laravel.com" @readonly($user->isNotAdmin())
/>
```

此外，` @required` 指令可用於指示給定元素是否應「必填」：

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
> 雖然您可以自由使用 ` @include` 指令，但 Blade [元件](#components) 提供了類似的功能，並提供了一些優於 ` @include` 指令的優點，例如資料和屬性綁定。

Blade 的 ` @include` 指令允許您從另一個視圖中引入 Blade 視圖。所有可供父視圖使用的變數都將可供引入的視圖使用：

```blade
<div>
    @include('shared.errors')

    <form>
        <!-- Form Contents -->
    </form>
</div>
```

即使引入的視圖將繼承父視圖中所有可用的資料，您也可以傳遞一個額外資料陣列，該陣列應可供引入的視圖使用：

```blade
 @include('view.name', ['status' => 'complete'])
```

如果您嘗試 ` @include` 一個不存在的視圖，Laravel 將會拋出錯誤。如果您想引入一個可能存在或不存在的視圖，您應該使用 ` @includeIf` 指令：

```blade
 @includeIf('view.name', ['status' => 'complete'])
```

如果您想在給定布林表達式評估為 `true` 或 `false` 時 ` @include` 一個視圖，您可以使用 ` @includeWhen` 和 ` @includeUnless` 指令：

```blade
 @includeWhen($boolean, 'view.name', ['status' => 'complete'])

 @includeUnless($boolean, 'view.name', ['status' => 'complete'])
```

要從給定視圖陣列中引入第一個存在的視圖，您可以使用 `includeFirst` 指令：

```blade
 @includeFirst(['custom.admin', 'admin'], ['status' => 'complete'])
```

> [!WARNING]
> 透過 ` @each` 渲染的視圖不會繼承父視圖的變數。如果子視圖需要這些變數，您應該改用 ` @foreach` 和 ` @include` 指令。

<a name="rendering-views-for-collections"></a>
#### 渲染集合的視圖

您可以使用 Blade 的 ` @each` 指令將迴圈和引入合併為一行：

```blade
 @each('view.name', $jobs, 'job')
```

` @each` 指令的第一個參數是為陣列或集合中的每個元素渲染的視圖。第二個參數是您希望迭代的陣列或集合，而第三個參數是將在視圖中分配給當前迭代的變數名稱。因此，例如，如果您正在迭代 `jobs` 陣列，通常您會希望在視圖中將每個 job 作為 `job` 變數存取。當前迭代的陣列鍵將在視圖中作為 `key` 變數可用。

您也可以將第四個參數傳遞給 ` @each` 指令。此參數決定了如果給定陣列為空時將渲染的視圖。

```blade
 @each('view.name', $jobs, 'job', 'view.empty')
```

> [!WARNING]
> 透過 ` @each` 渲染的視圖不會繼承父視圖的變數。如果子視圖需要這些變數，您應該改用 ` @foreach` 和 ` @include` 指令。

<a name="the-once-directive"></a>
### ` @once` 指令

` @once` 指令允許您定義模板中只會在每個渲染週期評估一次的部分。這對於使用 [堆疊](#stacks) 將給定的 JavaScript 片段推送到頁面標頭可能很有用。例如，如果您在迴圈中渲染給定的 [元件](#components)，您可能希望只在第一次渲染元件時將 JavaScript 推送到標頭：

```blade
 @once @push('scripts')
        <script>
            // Your custom JavaScript...
        </script>
    @endpush @endonce
```

由於 ` @once` 指令通常與 ` @push` 或 ` @prepend` 指令結合使用，因此為了您的方便，提供了 ` @pushOnce` 和 ` @prependOnce` 指令：

```blade
 @pushOnce('scripts')
    <script>
        // Your custom JavaScript...
    </script>
 @endPushOnce
```

<a name="raw-php"></a>
### 原始 PHP

在某些情況下，將 PHP 程式碼嵌入到您的視圖中很有用。您可以使用 Blade ` @php` 指令在模板中執行一個純 PHP 區塊：

```blade
 @php
    $counter = 1;
 @endphp
```

或者，如果您只需要使用 PHP 引入一個類別，您可以使用 ` @use` 指令：

```blade
 @use('App\Models\Flight')
```

可以為 ` @use` 指令提供第二個參數來為引入的類別設定別名：

```php
 @use('App\Models\Flight', 'FlightModel')
```

<a name="comments"></a>
### 註解

Blade 還允許您在視圖中定義註解。然而，與 HTML 註解不同，Blade 註解不會包含在您的應用程式返回的 HTML 中：

```blade
{{-- This comment will not be present in the rendered HTML --}}
```

<a name="components"></a>
## 元件

元件和 Slot 提供了與區塊、版面和引入類似的優點；然而，有些人可能會發現元件和 Slot 的心智模型更容易理解。編寫元件有兩種方法：基於類別的元件和匿名元件。

要建立基於類別的元件，您可以使用 `make:component` Artisan 命令。為了說明如何使用元件，我們將建立一個簡單的 `Alert` 元件。`make:component` 命令會將元件放置在 `app/View/Components` 目錄中：

```shell
php artisan make:component Alert
```

`make:component` 命令還會為元件建立一個視圖模板。該視圖將放置在 `resources/views/components` 目錄中。在為您自己的應用程式編寫元件時，元件會自動在 `app/View/Components` 目錄和 `resources/views/components` 目錄中被發現，因此通常不需要進一步的元件註冊。

您也可以在子目錄中建立元件：

```shell
php artisan make:component Forms/Input
```

上述命令將在 `app/View/Components/Forms` 目錄中建立一個 `Input` 元件，並且視圖將放置在 `resources/views/components/forms` 目錄中。

如果您想建立一個匿名元件 (只有 Blade 模板而沒有類別的元件)，您可以在呼叫 `make:component` 命令時使用 `--view` 旗標：

```shell
php artisan make:component forms.input --view
```

上述命令將在 `resources/views/components/forms/input.blade.php` 建立一個 Blade 檔案，該檔案可以透過 `<x-forms.input />` 渲染為元件。

<a name="manually-registering-package-components"></a>
#### 手動註冊套件元件

在為您自己的應用程式編寫元件時，元件會自動在 `app/View/Components` 目錄和 `resources/views/components` 目錄中被發現。

然而，如果您正在建構一個使用 Blade 元件的套件，您將需要手動註冊您的元件類別及其 HTML 標籤別名。您通常應該在套件服務提供者的 `boot` 方法中註冊您的元件：

    use Illuminate\Support\Facades\Blade;

    /**
     * Bootstrap your package's services.
     */
    public function boot(): void
    {
        Blade::component('package-alert', Alert::class);
    }

一旦您的元件已註冊，就可以使用其標籤別名進行渲染：

```blade
<x-package-alert/>
```

或者，您可以使用 `componentNamespace` 方法透過慣例自動載入元件類別。例如，`Nightshade` 套件可能包含 `Calendar` 和 `ColorPicker` 元件，這些元件位於 `Package\Views\Components` 命名空間中：

    use Illuminate\Support\Facades\Blade;

    /**
     * Bootstrap your package's services.
     */
    public function boot(): void
    {
        Blade::componentNamespace('Nightshade\\Views\\Components', 'nightshade');
    }

這將允許使用 `package-name::` 語法透過其供應商命名空間使用套件元件：

```blade
<x-nightshade::calendar />
<x-nightshade::color-picker />
```

Blade 將透過將元件名稱轉換為 PascalCase 自動偵測連結到此元件的類別。子目錄也支援使用「點」表示法。

<a name="rendering-components"></a>
### 渲染元件

要顯示元件，您可以在其中一個 Blade 模板中使用 Blade 元件標籤。Blade 元件標籤以字串 `x-` 開頭，後跟元件類別的 kebab-case 名稱：

```blade
<x-alert/>

<x-user-profile/>
```

如果元件類別巢狀在 `app/View/Components` 目錄中更深層，您可以使用 `.` 字元來表示目錄巢狀。例如，如果我們假設元件位於 `app/View/Components/Inputs/Button.php`，我們可以像這樣渲染它：

```blade
<x-inputs.button/>
```

如果您想條件式地渲染您的元件，您可以在元件類別上定義一個 `shouldRender` 方法。如果 `shouldRender` 方法返回 `false`，則元件將不會被渲染：

    use Illuminate\Support\Str;

    /**
     * Whether the component should be rendered
     */
    public function shouldRender(): bool
    {
        return Str::length($this->message) > 0;
    }

<a name="index-components"></a>
### 索引元件

有時元件是元件群組的一部分，您可能希望將相關元件分組到單一目錄中。例如，想像一個具有以下類別結構的「卡片」元件：

```none
App\Views\Components\Card\Card
App\Views\Components\Card\Header
App\Views\Components\Card\Body
```

由於根 `Card` 元件巢狀在 `Card` 目錄中，您可能會預期需要透過 `<x-card.card>` 渲染元件。然而，當元件的檔案名稱與元件目錄的名稱匹配時，Laravel 會自動假設該元件是「根」元件，並允許您在不重複目錄名稱的情況下渲染元件：

```blade
<x-card>
    <x-card.header>...</x-card.header>
    <x-card.body>...</x-card.body>
</x-card>
```

<a name="passing-data-to-components"></a>
### 傳遞資料給元件

您可以透過 HTML 屬性將資料傳遞給 Blade 元件。硬編碼的原始值可以使用簡單的 HTML 屬性字串傳遞給元件。PHP 表達式和變數應透過使用 `:` 字元作為前綴的屬性傳遞給元件：

```blade
<x-alert type="error" :message="$message"/>
```

您應該在其類別建構子中定義所有元件的資料屬性。元件上的所有公共屬性將自動可供元件的視圖使用。無需從元件的 `render` 方法將資料傳遞給視圖：

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

當您的元件被渲染時，您可以透過 Echo 變數名稱來顯示元件公共變數的內容：

```blade
<div class="alert alert-{{ $type }}">
    {{ $message }}
</div>
```

<a name="casing"></a>
#### 命名慣例

元件建構子參數應使用 `camelCase` 指定，而當在 HTML 屬性中引用參數名稱時，應使用 `kebab-case`。例如，給定以下元件建構子：

    /**
     * Create the component instance.
     */
    public function __construct(
        public string $alertType,
    ) {}

`$alertType` 參數可以像這樣提供給元件：

```blade
<x-alert alert-type="danger" />
```

<a name="short-attribute-syntax"></a>
#### 短屬性語法

當將屬性傳遞給元件時，您也可以使用「短屬性」語法。這通常很方便，因為屬性名稱經常與它們對應的變數名稱匹配：

```blade
{{-- Short attribute syntax... --}}
<x-profile :$userId :$name />

{{-- Is equivalent to... --}}
<x-profile :user-id="$userId" :name="$name" />
```

<a name="escaping-attribute-rendering"></a>
#### 逸出屬性渲染

由於某些 JavaScript 框架（例如 Alpine.js）也使用冒號前綴屬性，您可以使用雙冒號 (`::`) 前綴來告知 Blade 該屬性不是 PHP 表達式。例如，給定以下元件：

```blade
<x-button ::class="{ danger: isDeleting }">
    Submit
</x-button>
```

Blade 將渲染以下 HTML：

```blade
<button :class="{ danger: isDeleting }">
    Submit
</button>
```

<a name="component-methods"></a>
#### 元件方法

除了公共變數可供您的元件模板使用之外，元件上的任何公共方法都可以被呼叫。例如，想像一個具有 `isSelected` 方法的元件：

    /**
     * Determine if the given option is the currently selected option.
     */
    public function isSelected(string $option): bool
    {
        return $option === $this->selected;
    }

您可以透過呼叫與方法名稱匹配的變數來從元件模板執行此方法：

```blade
<option {{ $isSelected($value) ? 'selected' : '' }} value="{{ $value }}">
    {{ $label }}
</option>
```

<a name="using-attributes-slots-within-component-class"></a>
#### 在元件類別中存取屬性和 Slot

Blade 元件還允許您在類別的渲染方法中存取元件名稱、屬性和 Slot。但是，為了存取這些資料，您應該從元件的 `render` 方法返回一個閉包：

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

元件的 `render` 方法返回的閉包也可以接收一個 `$data` 陣列作為其唯一參數。此陣列將包含幾個提供有關元件資訊的元素：

    return function (array $data) {
        // $data['componentName'];
        // $data['attributes'];
        // $data['slot'];

        return '<div {{ $attributes }}>Components content</div>';
    }

> [!WARNING]
> `$data` 陣列中的元素不應直接嵌入到您的 `render` 方法返回的 Blade 字串中，因為這樣做可能會透過惡意屬性內容導致遠端程式碼執行。

`componentName` 等於 HTML 標籤中 `x-` 前綴後使用的名稱。因此 `<x-alert />` 的 `componentName` 將是 `alert`。`attributes` 元素將包含 HTML 標籤上存在的所有屬性。`slot` 元素是一個 `Illuminate\Support\HtmlString` 實例，其中包含元件 Slot 的內容。

閉包應返回一個字串。如果返回的字串對應於現有視圖，則該視圖將被渲染；否則，返回的字串將被評估為行內 Blade 視圖。

<a name="additional-dependencies"></a>
#### 額外依賴項

如果您的元件需要 Laravel [服務容器](/docs/{{version}}/container) 中的依賴項，您可以將它們列在元件的任何資料屬性之前，它們將由容器自動注入：

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

如果您想防止某些公共方法或屬性作為變數暴露給您的元件模板，您可以將它們添加到元件上的 `$except` 陣列屬性中：

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
### 元件屬性

我們已經研究了如何將資料屬性傳遞給元件；然而，有時您可能需要指定額外的 HTML 屬性，例如 `class`，這些屬性不是元件功能所需的資料的一部分。通常，您希望將這些額外屬性傳遞給元件模板的根元素。例如，想像我們想這樣渲染一個 `alert` 元件：

```blade
<x-alert type="error" :message="$message" class="mt-4"/>
```

所有不屬於元件建構子的屬性將自動添加到元件的「屬性包」中。此屬性包透過 `$attributes` 變數自動可供元件使用。所有屬性都可以透過 Echo 此變數在元件中渲染：

```blade
<div {{ $attributes }}>
    <!-- Component content -->
</div>
```

> [!WARNING]
> 目前不支援在元件標籤中使用 ` @env` 等指令。例如，`<x-alert :live=" @env('production')"/>` 將不會被編譯。

<a name="default-merged-attributes"></a>
#### 預設 / 合併屬性

有時您可能需要為屬性指定預設值或將額外值合併到某些元件的屬性中。為此，您可以使用屬性包的 `merge` 方法。此方法對於定義一組應始終應用於元件的預設 CSS 類別特別有用：

```blade
<div {{ $attributes->merge(['class' => 'alert alert-'.$type]) }}>
    {{ $message }}
</div>
```

如果我們假設此元件像這樣使用：

```blade
<x-alert type="error" :message="$message" class="mb-4"/>
```

元件的最終渲染 HTML 將如下所示：

```blade
<div class="alert alert-error mb-4">
    <!-- Contents of the $message variable -->
</div>
```

<a name="conditionally-merge-classes"></a>
#### 條件式合併類別

有時您可能希望在給定條件為 `true` 時合併類別。您可以透過 `class` 方法實現此目的，該方法接受一個類別陣列，其中陣列鍵包含您希望添加的類別，而值是一個布林表達式。如果陣列元素具有數字鍵，它將始終包含在渲染的類別列表中：

```blade
<div {{ $attributes->class(['p-4', 'bg-red' => $hasError]) }}>
    {{ $message }}
</div>
```

如果您需要將其他屬性合併到您的元件上，您可以將 `merge` 方法鏈接到 `class` 方法：

```blade
<button {{ $attributes->class(['p-4'])->merge(['type' => 'button']) }}>
    {{ $slot }}
</button>
```

> [!NOTE]
> 如果您需要在不應接收合併屬性的其他 HTML 元素上條件式編譯類別，您可以使用 [` @class` 指令](#conditional-classes)。

<a name="non-class-attribute-merging"></a>
#### 非類別屬性合併

當合併非 `class` 屬性時，提供給 `merge` 方法的值將被視為屬性的「預設」值。然而，與 `class` 屬性不同，這些屬性不會與注入的屬性值合併。相反，它們將被覆寫。例如，`button` 元件的實作可能如下所示：

```blade
<button {{ $attributes->merge(['type' => 'button']) }}>
    {{ $slot }}
</button>
```

要使用自訂 `type` 渲染按鈕元件，可以在使用元件時指定它。如果未指定類型，將使用 `button` 類型：

```blade
<x-button type="submit">
    Submit
</x-button>
```

此範例中 `button` 元件的渲染 HTML 將是：

```blade
<button type="submit">
    Submit
</button>
```

如果您希望除了 `class` 之外的屬性將其預設值和注入值連接在一起，您可以使用 `prepends` 方法。在此範例中，`data-controller` 屬性將始終以 `profile-controller` 開頭，並且任何額外注入的 `data-controller` 值將放置在此預設值之後：

```blade
<div {{ $attributes->merge(['data-controller' => $attributes->prepends('profile-controller')]) }}>
    {{ $slot }}
</div>
```

<a name="filtering-attributes"></a>
#### 擷取和篩選屬性

您可以使用 `filter` 方法篩選屬性。此方法接受一個閉包，如果希望將屬性保留在屬性包中，該閉包應返回 `true`：

```blade
{{ $attributes->filter(fn (string $value, string $key) => $key == 'foo') }}
```

為方便起見，您可以使用 `whereStartsWith` 方法擷取所有鍵以給定字串開頭的屬性：

```blade
{{ $attributes->whereStartsWith('wire:model') }}
```

相反地，`whereDoesntStartWith` 方法可用於排除所有鍵以給定字串開頭的屬性：

```blade
{{ $attributes->whereDoesntStartWith('wire:model') }}
```

使用 `first` 方法，您可以渲染給定屬性包中的第一個屬性：

```blade
{{ $attributes->whereStartsWith('wire:model')->first() }}
```

如果您想檢查元件上是否存在屬性，您可以使用 `has` 方法。此方法接受屬性名稱作為其唯一參數，並返回一個布林值，指示屬性是否存在：

```blade
 @if ($attributes->has('class'))
    <div>Class attribute is present</div>
 @endif
```

如果將陣列傳遞給 `has` 方法，該方法將判斷元件上是否存在所有給定屬性：

```blade
 @if ($attributes->has(['name', 'class']))
    <div>All of the attributes are present</div>
 @endif
```

`hasAny` 方法可用於判斷元件上是否存在任何給定屬性：

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

預設情況下，某些關鍵字保留供 Blade 內部使用，以渲染元件。以下關鍵字不能在您的元件中定義為公共屬性或方法名稱：

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
### Slot

您通常需要透過「Slot」將額外內容傳遞給您的元件。元件 Slot 透過 Echo `$slot` 變數進行渲染。為了探索這個概念，讓我們想像一個 `alert` 元件具有以下標記：

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

有時元件可能需要在元件內的不同位置渲染多個不同的 Slot。讓我們修改我們的警示元件，以允許注入「title」Slot：

```blade
<!-- /resources/views/components/alert.blade.php -->

<span class="alert-title">{{ $title }}</span>

<div class="alert alert-danger">
    {{ $slot }}
</div>
```

您可以使用 `x-slot` 標籤定義具名 Slot 的內容。任何不在明確 `x-slot` 標籤內的內容都將在 `$slot` 變數中傳遞給元件：

```xml
<x-alert>
    <x-slot:title>
        Server Error
    </x-slot>

    <strong>Whoops!</strong> Something went wrong!
</x-alert>
```

您可以呼叫 Slot 的 `isEmpty` 方法來判斷 Slot 是否包含內容：

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

此外，`hasActualContent` 方法可用於判斷 Slot 是否包含任何非 HTML 註解的「實際」內容：

```blade
 @if ($slot->hasActualContent())
    The scope has non-comment content.
 @endif
```

<a name="scoped-slots"></a>
#### 作用域 Slot

如果您使用過 Vue 等 JavaScript 框架，您可能熟悉「作用域 Slot」，它允許您在 Slot 中存取元件的資料或方法。您可以在 Laravel 中透過在元件上定義公共方法或屬性，並透過 `$component` 變數在 Slot 中存取元件來實現類似的行為。在此範例中，我們將假設 `x-alert` 元件在其元件類別上定義了一個公共 `formatAlert` 方法：

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

與 Blade 元件一樣，您可以為 Slot 分配額外 [屬性](#component-attributes)，例如 CSS 類別名稱：

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

要與 Slot 屬性互動，您可以存取 Slot 變數的 `attributes` 屬性。有關如何與屬性互動的更多資訊，請參閱 [元件屬性](#component-attributes) 的說明文件：

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

對於非常小的元件，管理元件類別和元件的視圖模板可能會感到繁瑣。因此，您可以直接從 `render` 方法返回元件的標記：

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
#### 產生行內視圖元件

要建立渲染行內視圖的元件，您可以在執行 `make:component` 命令時使用 `inline` 選項：

```shell
php artisan make:component Alert --inline
```

<a name="dynamic-components"></a>
### 動態元件

有時您可能需要渲染一個元件，但直到執行時才知道應該渲染哪個元件。在這種情況下，您可以使用 Laravel 內建的 `dynamic-component` 元件根據執行時值或變數渲染元件：

```blade
// $componentName = "secondary-button";

<x-dynamic-component :component="$componentName" class="mt-4" />
```

<a name="manually-registering-components"></a>
### 手動註冊元件

> [!WARNING]
> 以下關於手動註冊元件的說明文件主要適用於編寫包含視圖元件的 Laravel 套件的人。如果您沒有編寫套件，則元件說明文件的這部分可能與您無關。

在為您自己的應用程式編寫元件時，元件會自動在 `app/View/Components` 目錄和 `resources/views/components` 目錄中被發現。

然而，如果您正在建構一個使用 Blade 元件的套件，或者將元件放置在非傳統目錄中，您將需要手動註冊您的元件類別及其 HTML 標籤別名，以便 Laravel 知道在哪裡找到元件。您通常應該在套件服務提供者的 `boot` 方法中註冊您的元件：

    use Illuminate\Support\Facades\Blade;
    use VendorPackage\View\Components\AlertComponent;

    /**
     * Bootstrap your package's services.
     */
    public function boot(): void
    {
        Blade::component('package-alert', AlertComponent::class);
    }

一旦您的元件已註冊，就可以使用其標籤別名進行渲染：

```blade
<x-package-alert/>
```

#### 自動載入套件元件

或者，您可以使用 `componentNamespace` 方法透過慣例自動載入元件類別。例如，`Nightshade` 套件可能包含 `Calendar` 和 `ColorPicker` 元件，這些元件位於 `Package\Views\Components` 命名空間中：

    use Illuminate\Support\Facades\Blade;

    /**
     * Bootstrap your package's services.
     */
    public function boot(): void
    {
        Blade::componentNamespace('Nightshade\\Views\\Components', 'nightshade');
    }

這將允許使用 `package-name::` 語法透過其供應商命名空間使用套件元件：

```blade
<x-nightshade::calendar />
<x-nightshade::color-picker />
```

Blade 將透過將元件名稱轉換為 PascalCase 自動偵測連結到此元件的類別。子目錄也支援使用「點」表示法。

<a name="anonymous-components"></a>
## 匿名元件

與行內元件類似，匿名元件提供了一種透過單一檔案管理元件的機制。然而，匿名元件使用單一視圖檔案，並且沒有相關聯的類別。要定義匿名元件，您只需要將 Blade 模板放置在您的 `resources/views/components` 目錄中。例如，假設您已在 `resources/views/components/alert.blade.php` 定義了一個元件，您可以像這樣簡單地渲染它：

```blade
<x-alert/>
```

您可以使用 `.` 字元來指示元件是否巢狀在 `components` 目錄中更深層。例如，假設元件定義在 `resources/views/components/inputs/button.blade.php`，您可以像這樣渲染它：

```blade
<x-inputs.button/>
```

<a name="anonymous-index-components"></a>
### 匿名索引元件

有時，當一個元件由許多 Blade 模板組成時，您可能希望將給定元件的模板分組到單一目錄中。例如，想像一個具有以下目錄結構的「手風琴」元件：

```none
/resources/views/components/accordion.blade.php
/resources/views/components/accordion/item.blade.php
```

此目錄結構允許您像這樣渲染手風琴元件及其項目：

```blade
<x-accordion>
    <x-accordion.item>
        ...
    </x-accordion.item>
</x-accordion>
```

然而，為了透過 `x-accordion` 渲染手風琴元件，我們被迫將「索引」手風琴元件模板放置在 `resources/views/components` 目錄中，而不是將其與其他手風琴相關模板一起巢狀在 `accordion` 目錄中。

幸運的是，Blade 允許您將與元件目錄名稱匹配的檔案放置在元件目錄本身中。當此模板存在時，即使它巢狀在目錄中，它也可以作為元件的「根」元素進行渲染。因此，我們可以繼續使用上面範例中給出的相同 Blade 語法；但是，我們將像這樣調整我們的目錄結構：

```none
/resources/views/components/accordion/accordion.blade.php
/resources/views/components/accordion/item.blade.php
```

<a name="data-properties-attributes"></a>
### 資料屬性 / 特性

由於匿名元件沒有任何相關聯的類別，您可能會想知道如何區分哪些資料應作為變數傳遞給元件，以及哪些屬性應放置在元件的 [屬性包](#component-attributes) 中。

您可以使用元件 Blade 模板頂部的 ` @props` 指令指定哪些屬性應被視為資料變數。元件上的所有其他屬性將透過元件的屬性包可用。如果您希望為資料變數提供預設值，您可以將變數名稱指定為陣列鍵，並將預設值指定為陣列值：

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

有時您可能希望在子元件中存取父元件的資料。在這些情況下，您可以使用 ` @aware` 指令。例如，想像我們正在建構一個複雜的選單元件，由父 `<x-menu>` 和子 `<x-menu.item>` 組成：

```blade
<x-menu color="purple">
    <x-menu.item>...</x-menu.item>
    <x-menu.item>...</x-menu.item>
</x-menu>
```

`<x-menu>` 元件可能具有以下實作：

```blade
<!-- /resources/views/components/menu/index.blade.php -->

 @props(['color' => 'gray'])

<ul {{ $attributes->merge(['class' => 'bg-'.$color.'-200']) }}>
    {{ $slot }}
</ul>
```

因為 `color` 屬性只傳遞給父層 (`<x-menu>`)，所以它在 `<x-menu.item>` 內部將不可用。但是，如果我們使用 ` @aware` 指令，我們也可以讓它在 `<x-menu.item>` 內部可用：

```blade
<!-- /resources/views/components/menu/item.blade.php -->

 @aware(['color' => 'gray'])

<li {{ $attributes->merge(['class' => 'text-'.$color.'-800']) }}>
    {{ $slot }}
</li>
```

> [!WARNING]
> ` @aware` 指令無法存取未透過 HTML 屬性明確傳遞給父元件的父層資料。未明確傳遞給父元件的預設 ` @props` 值無法由 ` @aware` 指令存取。

<a name="anonymous-component-paths"></a>
### 匿名元件路徑

如前所述，匿名元件通常透過將 Blade 模板放置在您的 `resources/views/components` 目錄中來定義。然而，您偶爾可能希望除了預設路徑之外，還向 Laravel 註冊其他匿名元件路徑。

`anonymousComponentPath` 方法接受匿名元件位置的「路徑」作為其第一個參數，以及一個可選的「命名空間」作為其第二個參數，元件應放置在此命名空間下。通常，此方法應從您的應用程式 [服務提供者](/docs/{{version}}/providers) 之一的 `boot` 方法中呼叫：

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        Blade::anonymousComponentPath(__DIR__.'/../components');
    }

當元件路徑在沒有指定前綴的情況下註冊時，如上述範例所示，它們也可以在您的 Blade 元件中無需相應前綴進行渲染。例如，如果 `panel.blade.php` 元件存在於上述註冊的路徑中，它可以像這樣渲染：

```blade
<x-panel />
```

前綴「命名空間」可以作為 `anonymousComponentPath` 方法的第二個參數提供：

    Blade::anonymousComponentPath(__DIR__.'/../components', 'dashboard');

提供前綴後，該「命名空間」內的元件可以在渲染元件時，透過將元件的命名空間作為前綴添加到元件名稱來渲染：

```blade
<x-dashboard::panel />
```

<a name="building-layouts"></a>
## 建構版面

<a name="layouts-using-components"></a>
### 使用元件的版面

大多數 Web 應用程式在不同頁面之間保持相同的通用版面。如果我們必須在我們建立的每個視圖中重複整個版面 HTML，那將會非常繁瑣且難以維護我們的應用程式。幸運的是，將此版面定義為單一 [Blade 元件](#components) 然後在整個應用程式中使用它很方便。

<a name="defining-the-layout-component"></a>
#### 定義版面元件

例如，想像我們正在建構一個「待辦事項」清單應用程式。我們可能會定義一個如下所示的 `layout` 元件：

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
#### 應用版面元件

一旦定義了 `layout` 元件，我們就可以建立一個使用該元件的 Blade 視圖。在此範例中，我們將定義一個顯示任務清單的簡單視圖：

```blade
<!-- resources/views/tasks.blade.php -->

<x-layout>
    @foreach ($tasks as $task)
        <div>{{ $task }}</div>
    @endforeach
</x-layout>
```

請記住，注入到元件中的內容將提供給我們的 `layout` 元件中的預設 `$slot` 變數。您可能已經注意到，如果提供了 `$title` Slot，我們的 `layout` 也會尊重它；否則，將顯示預設標題。我們可以使用 [元件說明文件](#components) 中討論的標準 Slot 語法從任務清單視圖中注入自訂標題：

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

現在我們已經定義了版面和任務清單視圖，我們只需要從路由返回 `task` 視圖：

    use App\Models\Task;

    Route::get('/tasks', function () {
        return view('tasks', ['tasks' => Task::all()]);
    });

<a name="layouts-using-template-inheritance"></a>
### 使用模板繼承的版面

<a name="defining-a-layout"></a>
#### 定義版面

版面也可以透過「模板繼承」建立。這是引入 [元件](#components) 之前建構應用程式的主要方式。

首先，讓我們看一個簡單的範例。首先，我們將檢查頁面版面。由於大多數 Web 應用程式在不同頁面之間保持相同的通用版面，因此將此版面定義為單一 Blade 視圖很方便：

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

如您所見，此檔案包含典型的 HTML 標記。然而，請注意 ` @section` 和 ` @yield` 指令。` @section` 指令，顧名思義，定義了一個內容區塊，而 ` @yield` 指令用於顯示給定區塊的內容。

現在我們已經為我們的應用程式定義了一個版面，讓我們定義一個繼承該版面的子頁面。

<a name="extending-a-layout"></a>
#### 擴充版面

定義子視圖時，使用 ` @extends` Blade 指令指定子視圖應「繼承」哪個版面。擴充 Blade 版面的視圖可以使用 ` @section` 指令將內容注入版面的區塊中。請記住，如上述範例所示，這些區塊的內容將使用 ` @yield` 在版面中顯示：

```blade
<!-- resources/views/child.blade.php -->

 @extends('layouts.app')

 @section('title', 'Page Title')

 @section('sidebar')
    @@parent

    <p>This is appended to the master sidebar.</p>
 @endsection @section('content')
    <p>This is my body content.</p>
 @endsection
```

在此範例中，`sidebar` 區塊正在使用 ` @@parent` 指令將內容附加 (而不是覆寫) 到版面的側邊欄。` @@parent` 指令將在渲染視圖時被版面的內容替換。

> [!NOTE]
> 與上一個範例相反，此 `sidebar` 區塊以 ` @endsection` 結尾，而不是 ` @show`。` @endsection` 指令只會定義一個區塊，而 ` @show` 將定義並**立即產生**該區塊。

` @yield` 指令也接受一個預設值作為其第二個參數。如果正在產生的區塊未定義，則將渲染此值：

```blade
 @yield('content', 'Default content')
```

<a name="forms"></a>
## 表單

<a name="csrf-field"></a>
### CSRF 欄位

每當您在應用程式中定義 HTML 表單時，您都應該在表單中包含一個隱藏的 CSRF Token 欄位，以便 [CSRF 保護](/docs/{{version}}/csrf) Middleware 可以驗證請求。您可以使用 ` @csrf` Blade 指令來產生 Token 欄位：

```blade
<form method="POST" action="/profile">
    @csrf

    ...
</form>
```

<a name="method-field"></a>
### Method 欄位

由於 HTML 表單無法發出 `PUT`、`PATCH` 或 `DELETE` 請求，您將需要添加一個隱藏的 `_method` 欄位來偽造這些 HTTP 動詞。` @method` Blade 指令可以為您建立此欄位：

```blade
<form action="/foo/bar" method="POST">
    @method('PUT')

    ...
</form>
```

<a name="validation-errors"></a>
### 驗證錯誤

` @error` 指令可用於快速檢查給定屬性是否存在 [驗證錯誤訊息](/docs/{{version}}/validation#quick-displaying-the-validation-errors)。在 ` @error` 指令中，您可以 Echo `$message` 變數來顯示錯誤訊息：

```blade
<!-- /resources/views/post/create.blade.php -->

<label for="title">Post Title</label>

<input
    id="title"
    type="text"
    class=" @error('title') is-invalid @enderror"
/>

 @error('title')
    <div class="alert alert-danger">{{ $message }}</div>
 @enderror
```

由於 ` @error` 指令會編譯為「if」語句，因此您可以使用 ` @else` 指令在屬性沒有錯誤時渲染內容：

```blade
<!-- /resources/views/auth.blade.php -->

<label for="email">Email address</label>

<input
    id="email"
    type="email"
    class=" @error('email') is-invalid @else is-valid @enderror"
/>
```

您可以將 [特定錯誤包的名稱](/docs/{{version}}/validation#named-error-bags) 作為第二個參數傳遞給 ` @error` 指令，以擷取包含多個表單的頁面上的驗證錯誤訊息：

```blade
<!-- /resources/views/auth.blade.php -->

<label for="email">Email address</label>

<input
    id="email"
    type="email"
    class=" @error('email', 'login') is-invalid @enderror"
/>

 @error('email', 'login')
    <div class="alert alert-danger">{{ $message }}</div>
 @enderror
```

<a name="stacks"></a>
## 堆疊

Blade 允許您推送到具名堆疊，這些堆疊可以在另一個視圖或版面中的其他位置渲染。這對於指定子視圖所需的任何 JavaScript 函式庫特別有用：

```blade
 @push('scripts')
    <script src="/example.js"></script>
 @endpush
```

如果您想在給定布林表達式評估為 `true` 時 ` @push` 內容，您可以使用 ` @pushIf` 指令：

```blade
 @pushIf($shouldPush, 'scripts')
    <script src="/example.js"></script>
 @endPushIf
```

您可以根據需要多次推送到堆疊。要渲染完整的堆疊內容，請將堆疊的名稱傳遞給 ` @stack` 指令：

```blade
<head>
    <!-- Head Contents -->

    @stack('scripts')
</head>
```

如果您想將內容前置到堆疊的開頭，您應該使用 ` @prepend` 指令：

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

` @inject` 指令可用於從 Laravel [服務容器](/docs/{{version}}/container) 中擷取服務。傳遞給 ` @inject` 的第一個參數是服務將放置到的變數名稱，而第二個參數是您希望解析的服務的類別或介面名稱：

```blade
 @inject('metrics', 'App\Services\MetricsService')

<div>
    Monthly Revenue: {{ $metrics->monthlyRevenue() }}.
</div>
```

<a name="rendering-inline-blade-templates"></a>
## 渲染行內 Blade 模板

有時您可能需要將原始 Blade 模板字串轉換為有效的 HTML。您可以使用 `Blade` Facade 提供的 `render` 方法來實現此目的。`render` 方法接受 Blade 模板字串和一個可選的資料陣列以提供給模板：

```php
use Illuminate\Support\Facades\Blade;

return Blade::render('Hello, {{ $name }}', ['name' => 'Julian Bashir']);
```

Laravel 透過將行內 Blade 模板寫入 `storage/framework/views` 目錄來渲染它們。如果您希望 Laravel 在渲染 Blade 模板後移除這些臨時檔案，您可以將 `deleteCachedView` 參數提供給該方法：

```php
return Blade::render(
    'Hello, {{ $name }}',
    ['name' => 'Julian Bashir'],
    deleteCachedView: true
);
```

<a name="rendering-blade-fragments"></a>
## 渲染 Blade 片段

當使用 [Turbo](https://turbo.hotwired.dev/) 和 [htmx](https://htmx.org/) 等前端框架時，您可能偶爾只需要在 HTTP 回應中返回 Blade 模板的一部分。Blade「片段」允許您做到這一點。首先，將您的 Blade 模板的一部分放置在 ` @fragment` 和 ` @endfragment` 指令中：

```blade
 @fragment('user-list')
    <ul>
        @foreach ($users as $user)
            <li>{{ $user->name }}</li>
        @endforeach
    </ul>
 @endfragment
```

然後，在渲染使用此模板的視圖時，您可以呼叫 `fragment` 方法來指定只有指定的片段應包含在傳出的 HTTP 回應中：

```php
return view('dashboard', ['users' => $users])->fragment('user-list');
```

`fragmentIf` 方法允許您根據給定條件有條件地返回視圖片段。否則，將返回整個視圖：

```php
return view('dashboard', ['users' => $users])
    ->fragmentIf($request->hasHeader('HX-Request'), 'user-list');
```

`fragments` 和 `fragmentsIf` 方法允許您在回應中返回多個視圖片段。這些片段將被連接在一起：

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

Blade 允許您使用 `directive` 方法定義自己的自訂指令。當 Blade 編譯器遇到自訂指令時，它將使用指令包含的表達式呼叫提供的回呼。

以下範例建立了一個 ` @datetime($var)` 指令，該指令格式化給定的 `$var`，該 `$var` 應該是 `DateTime` 的實例：

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

如您所見，我們將 `format` 方法鏈接到傳遞給指令的任何表達式。因此，在此範例中，此指令生成的最終 PHP 將是：

    <?php echo ($var)->format('m/d/Y H:i'); ?>

> [!WARNING]
> 更新 Blade 指令的邏輯後，您需要刪除所有快取的 Blade 視圖。快取的 Blade 視圖可以使用 `view:clear` Artisan 命令移除。

<a name="custom-echo-handlers"></a>
### 自訂 Echo 處理器

如果您嘗試使用 Blade「Echo」一個物件，將會呼叫該物件的 `__toString` 方法。[`__toString`](https://www.php.net/manual/en/language.oop5.magic.php#object.tostring) 方法是 PHP 內建的「魔術方法」之一。然而，有時您可能無法控制給定類別的 `__toString` 方法，例如當您正在互動的類別屬於第三方函式庫時。

在這些情況下，Blade 允許您為該特定類型的物件註冊自訂 Echo 處理器。為此，您應該呼叫 Blade 的 `stringable` 方法。`stringable` 方法接受一個閉包。此閉包應型別提示它負責渲染的物件類型。通常，`stringable` 方法應在您的應用程式 `AppServiceProvider` 類別的 `boot` 方法中呼叫：

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

一旦定義了您的自訂 Echo 處理器，您就可以簡單地在您的 Blade 模板中 Echo 物件：

```blade
Cost: {{ $money }}
```

<a name="custom-if-statements"></a>
### 自訂 If 語句

在定義簡單的自訂條件語句時，編程自訂指令有時比必要更複雜。因此，Blade 提供了一個 `Blade::if` 方法，允許您使用閉包快速定義自訂條件指令。例如，讓我們定義一個自訂條件，檢查應用程式的預設「磁碟」配置。我們可以在 `AppServiceProvider` 的 `boot` 方法中執行此操作：

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

一旦定義了自訂條件，您就可以在模板中使用它：

```blade
 @disk('local')
    <!-- The application is using the local disk... -->
 @elsedisk('s3')
    <!-- The application is using the s3 disk... -->
 @else
    <!-- The application is using some other disk... -->
 @enddisk @unlessdisk('local')
    <!-- The application is not using the local disk... -->
 @enddisk
```
