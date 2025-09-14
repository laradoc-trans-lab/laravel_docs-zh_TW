# 提示 (Prompts)

- [簡介](#introduction)
- [安裝](#installation)
- [可用的提示](#available-prompts)
    - [文字輸入 (Text)](#text)
    - [多行文字輸入 (Textarea)](#textarea)
    - [密碼輸入 (Password)](#password)
    - [確認 (Confirm)](#confirm)
    - [單選 (Select)](#select)
    - [多選 (Multi-select)](#multiselect)
    - [建議 (Suggest)](#suggest)
    - [搜尋 (Search)](#search)
    - [多重搜尋 (Multi-search)](#multisearch)
    - [暫停 (Pause)](#pause)
- [驗證前轉換輸入](#transforming-input-before-validation)
- [表單 (Forms)](#forms)
- [資訊訊息](#informational-messages)
- [表格 (Tables)](#tables)
- [旋轉指示器 (Spin)](#spin)
- [進度條 (Progress Bar)](#progress)
- [清除終端機](#clear)
- [終端機考量](#terminal-considerations)
- [不支援的環境與備用方案](#fallbacks)
- [測試](#testing)

<a name="introduction"></a>
## 簡介

[Laravel Prompts](https://github.com/laravel/prompts) 是一個 PHP 套件，用於為您的命令列應用程式新增美觀且使用者友善的表單，並具備類似瀏覽器的功能，包括預留文字和驗證。

<img src="https://laravel.com/img/docs/prompts-example.png">

Laravel Prompts 非常適合在您的 [Artisan 命令列指令](/docs/{{version}}/artisan#writing-commands) 中接受使用者輸入，但它也可以用於任何命令列 PHP 專案。

> [!NOTE]
> Laravel Prompts 支援 macOS、Linux 和 Windows (搭配 WSL)。欲了解更多資訊，請參閱我們關於 [不支援的環境與備用方案](#fallbacks) 的說明文件。

<a name="installation"></a>
## 安裝

Laravel Prompts 已包含在最新版的 Laravel 中。

Laravel Prompts 也可以透過 Composer 套件管理器安裝到您的其他 PHP 專案中：

```shell
composer require laravel/prompts
```

<a name="available-prompts"></a>
## 可用的提示

<a name="text"></a>
### 文字輸入 (Text)

`text` 函式會向使用者提示給定的問題，接受他們的輸入，然後回傳該輸入：

```php
use function Laravel\Prompts\text;

$name = text('What is your name?');
```

您也可以包含預留文字、預設值和資訊提示：

```php
$name = text(
    label: 'What is your name?',
    placeholder: 'E.g. Taylor Otwell',
    default: $user?->name,
    hint: 'This will be displayed on your profile.'
);
```

<a name="text-required"></a>
#### 必填值

如果您要求輸入值，可以傳遞 `required` 引數：

```php
$name = text(
    label: 'What is your name?',
    required: true
);
```

如果您想自訂驗證訊息，也可以傳遞一個字串：

```php
$name = text(
    label: 'What is your name?',
    required: 'Your name is required.'
);
```

<a name="text-validation"></a>
#### 額外驗證

最後，如果您想執行額外的驗證邏輯，可以將一個閉包傳遞給 `validate` 引數：

```php
$name = text(
    label: 'What is your name?',
    validate: fn (string $value) => match (true) {
        strlen($value) < 3 => 'The name must be at least 3 characters.',
        strlen($value) > 255 => 'The name must not exceed 255 characters.',
        default => null
    }
);
```

該閉包將接收已輸入的值，並可回傳錯誤訊息，如果驗證通過則回傳 `null`。

或者，您可以利用 Laravel [Validator](/docs/{{version}}/validation) 的強大功能。為此，請提供一個包含屬性名稱和所需驗證規則的陣列給 `validate` 引數：

```php
$name = text(
    label: 'What is your name?',
    validate: ['name' => 'required|max:255|unique:users']
);
```

<a name="textarea"></a>
### 多行文字輸入 (Textarea)

`textarea` 函式會向使用者提示給定的問題，透過多行文字區域接受他們的輸入，然後回傳該輸入：

```php
use function Laravel\Prompts\textarea;

$story = textarea('Tell me a story.');
```

您也可以包含預留文字、預設值和資訊提示：

```php
$story = textarea(
    label: 'Tell me a story.',
    placeholder: 'This is a story about...',
    hint: 'This will be displayed on your profile.'
);
```

<a name="textarea-required"></a>
#### 必填值

如果您要求輸入值，可以傳遞 `required` 引數：

```php
$story = textarea(
    label: 'Tell me a story.',
    required: true
);
```

如果您想自訂驗證訊息，也可以傳遞一個字串：

```php
$story = textarea(
    label: 'Tell me a story.',
    required: 'A story is required.'
);
```

<a name="textarea-validation"></a>
#### 額外驗證

最後，如果您想執行額外的驗證邏輯，可以將一個閉包傳遞給 `validate` 引數：

```php
$story = textarea(
    label: 'Tell me a story.',
    validate: fn (string $value) => match (true) {
        strlen($value) < 250 => 'The story must be at least 250 characters.',
        strlen($value) > 10000 => 'The story must not exceed 10,000 characters.',
        default => null
    }
);
```

該閉包將接收已輸入的值，並可回傳錯誤訊息，如果驗證通過則回傳 `null`。

或者，您可以利用 Laravel [Validator](/docs/{{version}}/validation) 的強大功能。為此，請提供一個包含屬性名稱和所需驗證規則的陣列給 `validate` 引數：

```php
$story = textarea(
    label: 'Tell me a story.',
    validate: ['story' => 'required|max:10000']
);
```

<a name="password"></a>
### 密碼輸入 (Password)

`password` 函式與 `text` 函式類似，但使用者在控制台中輸入時，其輸入將被遮蔽。這在要求敏感資訊（例如密碼）時很有用：

```php
use function Laravel\Prompts\password;

$password = password('What is your password?');
```

您也可以包含預留文字和資訊提示：

```php
$password = password(
    label: 'What is your password?',
    placeholder: 'password',
    hint: 'Minimum 8 characters.'
);
```

<a name="password-required"></a>
#### 必填值

如果您要求輸入值，可以傳遞 `required` 引數：

```php
$password = password(
    label: 'What is your password?',
    required: true
);
```

如果您想自訂驗證訊息，也可以傳遞一個字串：

```php
$password = password(
    label: 'What is your password?',
    required: 'The password is required.'
);
```

<a name="password-validation"></a>
#### 額外驗證

最後，如果您想執行額外的驗證邏輯，可以將一個閉包傳遞給 `validate` 引數：

```php
$password = password(
    label: 'What is your password?',
    validate: fn (string $value) => match (true) {
        strlen($value) < 8 => 'The password must be at least 8 characters.',
        default => null
    }
);
```

該閉包將接收已輸入的值，並可回傳錯誤訊息，如果驗證通過則回傳 `null`。

或者，您可以利用 Laravel [Validator](/docs/{{version}}/validation) 的強大功能。為此，請提供一個包含屬性名稱和所需驗證規則的陣列給 `validate` 引數：

```php
$password = password(
    label: 'What is your password?',
    validate: ['password' => 'min:8']
);
```

<a name="confirm"></a>
### 確認 (Confirm)

如果您需要向使用者詢問「是或否」的確認，可以使用 `confirm` 函式。使用者可以使用方向鍵或按下 `y` 或 `n` 來選擇他們的回答。此函式將回傳 `true` 或 `false`。

```php
use function Laravel\Prompts\confirm;

$confirmed = confirm('Do you accept the terms?');
```

您也可以包含預設值、自訂的「是」和「否」標籤文字，以及資訊提示：

```php
$confirmed = confirm(
    label: 'Do you accept the terms?',
    default: false,
    yes: 'I accept',
    no: 'I decline',
    hint: 'The terms must be accepted to continue.'
);
```

<a name="confirm-required"></a>
#### 要求「是」

如有必要，您可以透過傳遞 `required` 引數來要求使用者選擇「是」：

```php
$confirmed = confirm(
    label: 'Do you accept the terms?',
    required: true
);
```

如果您想自訂驗證訊息，也可以傳遞一個字串：

```php
$confirmed = confirm(
    label: 'Do you accept the terms?',
    required: 'You must accept the terms to continue.'
);
```

<a name="select"></a>
### 單選 (Select)

如果您需要使用者從預定義的選項集中進行選擇，可以使用 `select` 函式：

```php
use function Laravel\Prompts\select;

$role = select(
    label: 'What role should the user have?',
    options: ['Member', 'Contributor', 'Owner']
);
```

您也可以指定預設選項和資訊提示：

```php
$role = select(
    label: 'What role should the user have?',
    options: ['Member', 'Contributor', 'Owner'],
    default: 'Owner',
    hint: 'The role may be changed at any time.'
);
```

您也可以將關聯陣列傳遞給 `options` 引數，以便回傳選定的鍵而不是其值：

```php
$role = select(
    label: 'What role should the user have?',
    options: [
        'member' => 'Member',
        'contributor' => 'Contributor',
        'owner' => 'Owner',
    ],
    default: 'owner'
);
```

最多會顯示五個選項，然後列表才會開始捲動。您可以透過傳遞 `scroll` 引數來自訂此行為：

```php
$role = select(
    label: 'Which category would you like to assign?',
    options: Category::pluck('name', 'id'),
    scroll: 10
);
```

<a name="select-validation"></a>
#### 額外驗證

與其他提示函式不同，`select` 函式不接受 `required` 引數，因為不可能不選擇任何東西。但是，如果您需要呈現一個選項但阻止其被選中，您可以將一個閉包傳遞給 `validate` 引數：

```php
$role = select(
    label: 'What role should the user have?',
    options: [
        'member' => 'Member',
        'contributor' => 'Contributor',
        'owner' => 'Owner',
    ],
    validate: fn (string $value) =>
        $value === 'owner' && User::where('role', 'owner')->exists()
            ? 'An owner already exists.'
            : null
);
```

如果 `options` 引數是關聯陣列，則閉包將接收選定的鍵，否則將接收選定的值。閉包可以回傳錯誤訊息，如果驗證通過則回傳 `null`。

<a name="multiselect"></a>
### 多選 (Multi-select)

如果您需要使用者能夠選擇多個選項，可以使用 `multiselect` 函式：

```php
use function Laravel\Prompts\multiselect;

$permissions = multiselect(
    label: 'What permissions should be assigned?',
    options: ['Read', 'Create', 'Update', 'Delete']
);
```

您也可以指定預設選項和資訊提示：

```php
use function Laravel\Prompts\multiselect;

$permissions = multiselect(
    label: 'What permissions should be assigned?',
    options: ['Read', 'Create', 'Update', 'Delete'],
    default: ['Read', 'Create'],
    hint: 'Permissions may be updated at any time.'
);
```

您也可以將關聯陣列傳遞給 `options` 引數，以便回傳選定選項的鍵而不是其值：

```php
$permissions = multiselect(
    label: 'What permissions should be assigned?',
    options: [
        'read' => 'Read',
        'create' => 'Create',
        'update' => 'Update',
        'delete' => 'Delete',
    ],
    default: ['read', 'create']
);
```

最多會顯示五個選項，然後列表才會開始捲動。您可以透過傳遞 `scroll` 引數來自訂此行為：

```php
$categories = multiselect(
    label: 'What categories should be assigned?',
    options: Category::pluck('name', 'id'),
    scroll: 10
);
```

<a name="multiselect-required"></a>
#### 要求值

預設情況下，使用者可以選擇零個或多個選項。您可以傳遞 `required` 引數來強制選擇一個或多個選項：

```php
$categories = multiselect(
    label: 'What categories should be assigned?',
    options: Category::pluck('name', 'id'),
    required: true
);
```

如果您想自訂驗證訊息，可以向 `required` 引數提供一個字串：

```php
$categories = multiselect(
    label: 'What categories should be assigned?',
    options: Category::pluck('name', 'id'),
    required: 'You must select at least one category'
);
```

<a name="multiselect-validation"></a>
#### 額外驗證

如果您需要呈現一個選項但阻止其被選中，您可以將一個閉包傳遞給 `validate` 引數：

```php
$permissions = multiselect(
    label: 'What permissions should the user have?',
    options: [
        'read' => 'Read',
        'create' => 'Create',
        'update' => 'Update',
        'delete' => 'Delete',
    ],
    validate: fn (array $values) => ! in_array('read', $values)
        ? 'All users require the read permission.'
        : null
);
```

如果 `options` 引數是關聯陣列，則閉包將接收選定的鍵，否則將接收選定的值。閉包可以回傳錯誤訊息，如果驗證通過則回傳 `null`。

<a name="suggest"></a>
### 建議 (Suggest)

`suggest` 函式可用於為可能的選項提供自動完成功能。使用者仍然可以提供任何答案，無論自動完成提示如何：

```php
use function Laravel\Prompts\suggest;

$name = suggest('What is your name?', ['Taylor', 'Dayle']);
```

或者，您可以將一個閉包作為第二個引數傳遞給 `suggest` 函式。每次使用者輸入字元時都會呼叫該閉包。該閉包應接受一個包含使用者目前輸入的字串參數，並回傳一個用於自動完成的選項陣列：

```php
$name = suggest(
    label: 'What is your name?',
    options: fn ($value) => collect(['Taylor', 'Dayle'])
        ->filter(fn ($name) => Str::contains($name, $value, ignoreCase: true))
)
```

您也可以包含預留文字、預設值和資訊提示：

```php
$name = suggest(
    label: 'What is your name?',
    options: ['Taylor', 'Dayle'],
    placeholder: 'E.g. Taylor',
    default: $user?->name,
    hint: 'This will be displayed on your profile.'
);
```

<a name="suggest-required"></a>
#### 必填值

如果您要求輸入值，可以傳遞 `required` 引數：

```php
$name = suggest(
    label: 'What is your name?',
    options: ['Taylor', 'Dayle'],
    required: true
);
```

如果您想自訂驗證訊息，也可以傳遞一個字串：

```php
$name = suggest(
    label: 'What is your name?',
    options: ['Taylor', 'Dayle'],
    required: 'Your name is required.'
);
```

<a name="suggest-validation"></a>
#### 額外驗證

最後，如果您想執行額外的驗證邏輯，可以將一個閉包傳遞給 `validate` 引數：

```php
$name = suggest(
    label: 'What is your name?',
    options: ['Taylor', 'Dayle'],
    validate: fn (string $value) => match (true) {
        strlen($value) < 3 => 'The name must be at least 3 characters.',
        strlen($value) > 255 => 'The name must not exceed 255 characters.',
        default => null
    }
);
```

該閉包將接收已輸入的值，並可回傳錯誤訊息，如果驗證通過則回傳 `null`。

或者，您可以利用 Laravel [Validator](/docs/{{version}}/validation) 的強大功能。為此，請提供一個包含屬性名稱和所需驗證規則的陣列給 `validate` 引數：

```php
$name = suggest(
    label: 'What is your name?',
    options: ['Taylor', 'Dayle'],
    validate: ['name' => 'required|min:3|max:255']
);
```

<a name="search"></a>
### 搜尋 (Search)

如果您有很多選項供使用者選擇，`search` 函式允許使用者輸入搜尋查詢來篩選結果，然後使用方向鍵選擇選項：

```php
use function Laravel\Prompts\search;

$id = search(
    label: 'Search for the user that should receive the mail',
    options: fn (string $value) => strlen($value) > 0
        ? User::whereLike('name', "%{$value}%")->pluck('name', 'id')->all()
        : []
);
```

該閉包將接收使用者目前輸入的文字，並且必須回傳一個選項陣列。如果您回傳一個關聯陣列，則會回傳選定選項的鍵，否則會回傳其值。

當篩選一個您打算回傳值的陣列時，您應該使用 `array_values` 函式或 `values` Collection 方法來確保陣列不會變成關聯陣列：

```php
$names = collect(['Taylor', 'Abigail']);

$selected = search(
    label: 'Search for the user that should receive the mail',
    options: fn (string $value) => $names
        ->filter(fn ($name) => Str::contains($name, $value, ignoreCase: true))
        ->values()
        ->all(),
);
```

您也可以包含預留文字和資訊提示：

```php
$id = search(
    label: 'Search for the user that should receive the mail',
    placeholder: 'E.g. Taylor Otwell',
    options: fn (string $value) => strlen($value) > 0
        ? User::whereLike('name', "%{$value}%")->pluck('name', 'id')->all()
        : [],
    hint: 'The user will receive an email immediately.'
);
```

最多會顯示五個選項，然後列表才會開始捲動。您可以透過傳遞 `scroll` 引數來自訂此行為：

```php
$id = search(
    label: 'Search for the user that should receive the mail',
    options: fn (string $value) => strlen($value) > 0
        ? User::whereLike('name', "%{$value}%")->pluck('name', 'id')->all()
        : [],
    scroll: 10
);
```

<a name="search-validation"></a>
#### 額外驗證

如果您想執行額外的驗證邏輯，可以將一個閉包傳遞給 `validate` 引數：

```php
$id = search(
    label: 'Search for the user that should receive the mail',
    options: fn (string $value) => strlen($value) > 0
        ? User::whereLike('name', "%{$value}%")->pluck('name', 'id')->all()
        : [],
    validate: function (int|string $value) {
        $user = User::findOrFail($value);

        if ($user->opted_out) {
            return 'This user has opted-out of receiving mail.';
        }
    }
);
```

如果 `options` 閉包回傳關聯陣列，則閉包將接收選定的鍵，否則將接收選定的值。閉包可以回傳錯誤訊息，如果驗證通過則回傳 `null`。

<a name="multisearch"></a>
### 多重搜尋 (Multi-search)

如果您有很多可搜尋的選項，並且需要使用者能夠選擇多個項目，`multisearch` 函式允許使用者輸入搜尋查詢來篩選結果，然後使用方向鍵和空白鍵選擇選項：

```php
use function Laravel\Prompts\multisearch;

$ids = multisearch(
    'Search for the users that should receive the mail',
    fn (string $value) => strlen($value) > 0
        ? User::whereLike('name', "%{$value}%")->pluck('name', 'id')->all()
        : []
);
```

該閉包將接收使用者目前輸入的文字，並且必須回傳一個選項陣列。如果您回傳一個關聯陣列，則會回傳選定選項的鍵；否則，會回傳其值。

當篩選一個您打算回傳值的陣列時，您應該使用 `array_values` 函式或 `values` Collection 方法來確保陣列不會變成關聯陣列：

```php
$names = collect(['Taylor', 'Abigail']);

$selected = multisearch(
    label: 'Search for the users that should receive the mail',
    options: fn (string $value) => $names
        ->filter(fn ($name) => Str::contains($name, $value, ignoreCase: true))
        ->values()
        ->all(),
);
```

您也可以包含預留文字和資訊提示：

```php
$ids = multisearch(
    label: 'Search for the users that should receive the mail',
    placeholder: 'E.g. Taylor Otwell',
    options: fn (string $value) => strlen($value) > 0
        ? User::whereLike('name', "%{$value}%")->pluck('name', 'id')->all()
        : [],
    hint: 'The user will receive an email immediately.'
);
```

最多會顯示五個選項，然後列表才會開始捲動。您可以透過提供 `scroll` 引數來自訂此行為：

```php
$ids = multisearch(
    label: 'Search for the users that should receive the mail',
    options: fn (string $value) => strlen($value) > 0
        ? User::whereLike('name', "%{$value}%")->pluck('name', 'id')->all()
        : [],
    scroll: 10
);
```

<a name="multisearch-required"></a>
#### 必填值

預設情況下，使用者可以選擇零個或多個選項。您可以傳遞 `required` 引數來強制選擇一個或多個選項：

```php
$ids = multisearch(
    label: 'Search for the users that should receive the mail',
    options: fn (string $value) => strlen($value) > 0
        ? User::whereLike('name', "%{$value}%")->pluck('name', 'id')->all()
        : [],
    required: true
);
```

如果您想自訂驗證訊息，也可以向 `required` 引數提供一個字串：

```php
$ids = multisearch(
    label: 'Search for the users that should receive the mail',
    options: fn (string $value) => strlen($value) > 0
        ? User::whereLike('name', "%{$value}%")->pluck('name', 'id')->all()
        : [],
    required: 'You must select at least one user.'
);
```

<a name="multisearch-validation"></a>
#### 額外驗證

如果您想執行額外的驗證邏輯，可以將一個閉包傳遞給 `validate` 引數：

```php
$ids = multisearch(
    label: 'Search for the users that should receive the mail',
    options: fn (string $value) => strlen($value) > 0
        ? User::whereLike('name', "%{$value}%")->pluck('name', 'id')->all()
        : [],
    validate: function (array $values) {
        $optedOut = User::whereLike('name', '%a%')->findMany($values);

        if ($optedOut->isNotEmpty()) {
            return $optedOut->pluck('name')->join(', ', ', and ').' have opted out.';
        }
    }
);
```

如果 `options` 閉包回傳關聯陣列，則閉包將接收選定的鍵；否則，將接收選定的值。閉包可以回傳錯誤訊息，如果驗證通過則回傳 `null`。

<a name="pause"></a>
### 暫停 (Pause)

`pause` 函式可用於向使用者顯示資訊文字，並等待他們按下 Enter / Return 鍵確認繼續：

```php
use function Laravel\Prompts\pause;

pause('Press ENTER to continue.');
```

<a name="transforming-input-before-validation"></a>
## 驗證前轉換輸入

有時您可能希望在驗證發生之前轉換提示輸入。例如，您可能希望從任何提供的字串中移除空白字元。為了實現這一點，許多提示函式提供了 `transform` 引數，它接受一個閉包：

```php
$name = text(
    label: 'What is your name?',
    transform: fn (string $value) => trim($value),
    validate: fn (string $value) => match (true) {
        strlen($value) < 3 => 'The name must be at least 3 characters.',
        strlen($value) > 255 => 'The name must not exceed 255 characters.',
        default => null
    }
);
```

<a name="forms"></a>
## 表單 (Forms)

通常，您會有多個提示依序顯示以收集資訊，然後執行額外的動作。您可以使用 `form` 函式為使用者建立一組分組的提示以供完成：

```php
use function Laravel\Prompts\form;

$responses = form()
    ->text('What is your name?', required: true)
    ->password('What is your password?', validate: ['password' => 'min:8'])
    ->confirm('Do you accept the terms?')
    ->submit();
```

`submit` 方法將回傳一個數值索引陣列，其中包含表單提示的所有回應。但是，您可以透過 `name` 引數為每個提示提供一個名稱。當提供名稱時，可以透過該名稱存取命名提示的回應：

```php
use App\Models\User;
use function Laravel\Prompts\form;

$responses = form()
    ->text('What is your name?', required: true, name: 'name')
    ->password(
        label: 'What is your password?',
        validate: ['password' => 'min:8'],
        name: 'password'
    )
    ->confirm('Do you accept the terms?')
    ->submit();

User::create([
    'name' => $responses['name'],
    'password' => $responses['password'],
]);
```

使用 `form` 函式的主要好處是使用者可以使用 `CTRL + U` 返回表單中的先前提示。這允許使用者修正錯誤或更改選擇，而無需取消並重新開始整個表單。

如果您需要對表單中的提示進行更細緻的控制，您可以呼叫 `add` 方法而不是直接呼叫其中一個提示函式。`add` 方法會傳遞使用者提供的所有先前回應：

```php
use function Laravel\Prompts\form;
use function Laravel\Prompts\outro;
use function Laravel\Prompts\text;

$responses = form()
    ->text('What is your name?', required: true, name: 'name')
    ->add(function ($responses) {
        return text("How old are you, {$responses['name']}?");
    }, name: 'age')
    ->submit();

outro("Your name is {$responses['name']} and you are {$responses['age']} years old.");
```

<a name="informational-messages"></a>
## 資訊訊息

`note`、`info`、`warning`、`error` 和 `alert` 函式可用於顯示資訊訊息：

```php
use function Laravel\Prompts\info;

info('Package installed successfully.');
```

<a name="tables"></a>
### 表格 (Tables)

`table` 函式可以輕鬆顯示多行多列的資料。您只需提供欄位名稱和表格資料：

```php
use function Laravel\Prompts\table;

table(
    headers: ['Name', 'Email'],
    rows: User::all(['name', 'email'])->toArray()
);
```

<a name="spin"></a>
## 旋轉指示器 (Spin)

`spin` 函式在執行指定的回呼時，會顯示一個旋轉指示器和一個可選訊息。它用於指示正在進行的程序，並在完成後回傳回呼的結果：

```php
use function Laravel\Prompts\spin;

$response = spin(
    callback: fn () => Http::get('http://example.com'),
    message: 'Fetching response...'
);
```

> [!WARNING]
> `spin` 函式需要 [PCNTL](https://www.php.net/manual/en/book.pcntl.php) PHP 擴充功能才能使旋轉指示器動畫化。當此擴充功能不可用時，將顯示旋轉指示器的靜態版本。

<a name="progress"></a>
## 進度條 (Progress Bars)

對於長時間執行的任務，顯示一個進度條來告知使用者任務完成度會很有幫助。使用 `progress` 函式，Laravel 將顯示一個進度條，並在每次迭代給定可迭代值時推進其進度：

```php
use function Laravel\Prompts\progress;

$users = progress(
    label: 'Updating users',
    steps: User::all(),
    callback: fn ($user) => $this->performTask($user)
);
```

`progress` 函式的作用類似於 `map` 函式，它將回傳一個陣列，其中包含每次回呼迭代的回傳值。

回呼也可以接受 `Laravel\Prompts\Progress` 實例，允許您在每次迭代時修改標籤和提示：

```php
$users = progress(
    label: 'Updating users',
    steps: User::all(),
    callback: function ($user, $progress) {
        $progress
            ->label("Updating {$user->name}")
            ->hint("Created on {$user->created_at}");

        return $this->performTask($user);
    },
    hint: 'This may take some time.'
);
```

有時，您可能需要更手動地控制進度條的推進方式。首先，定義程序將迭代的總步數。然後，在處理每個項目後，透過 `advance` 方法推進進度條：

```php
$progress = progress(label: 'Updating users', steps: 10);

$users = User::all();

$progress->start();

foreach ($users as $user) {
    $this->performTask($user);

    $progress->advance();
}

$progress->finish();
```

<a name="clear"></a>
## 清除終端機

`clear` 函式可用於清除使用者的終端機：

```php
use function Laravel\Prompts\clear;

clear();
```

<a name="terminal-considerations"></a>
## 終端機考量

<a name="terminal-width"></a>
#### 終端機寬度

如果任何標籤、選項或驗證訊息的長度超過使用者終端機的「欄位」數量，它將自動截斷以適應。如果您的使用者可能使用較窄的終端機，請考慮最小化這些字串的長度。通常安全的最小長度是 74 個字元，以支援 80 個字元的終端機。

<a name="terminal-height"></a>
#### 終端機高度

對於任何接受 `scroll` 引數的提示，配置的值將自動縮小以適應使用者終端機的高度，包括驗證訊息的空間。

<a name="fallbacks"></a>
## 不支援的環境與備用方案

Laravel Prompts 支援 macOS、Linux 和 Windows (搭配 WSL)。由於 Windows 版 PHP 的限制，目前無法在 WSL 之外的 Windows 上使用 Laravel Prompts。

因此，Laravel Prompts 支援回退到替代實作，例如 [Symfony Console Question Helper](https://symfony.com/doc/current/components/console/helpers/questionhelper.html)。

> [!NOTE]
> 當您將 Laravel Prompts 與 Laravel 框架一起使用時，每個提示的備用方案都已為您配置，並將在不支援的環境中自動啟用。

<a name="fallback-conditions"></a>
#### 備用條件

如果您沒有使用 Laravel 或需要自訂何時使用備用行為，您可以將一個布林值傳遞給 `Prompt` 類別上的 `fallbackWhen` 靜態方法：

```php
use Laravel\Prompts\Prompt;

Prompt::fallbackWhen(
    ! $input->isInteractive() || windows_os() || app()->runningUnitTests()
);
```

<a name="fallback-behavior"></a>
#### 備用行為

如果您沒有使用 Laravel 或需要自訂備用行為，您可以將一個閉包傳遞給每個提示類別上的 `fallbackUsing` 靜態方法：

```php
use Laravel\Prompts\TextPrompt;
use Symfony\Component\Console\Question\Question;
use Symfony\Component\Console\Style\SymfonyStyle;

TextPrompt::fallbackUsing(function (TextPrompt $prompt) use ($input, $output) {
    $question = (new Question($prompt->label, $prompt->default ?: null))
        ->setValidator(function ($answer) use ($prompt) {
            if ($prompt->required && $answer === null) {
                throw new \RuntimeException(
                    is_string($prompt->required) ? $prompt->required : 'Required.'
                );
            }

            if ($prompt->validate) {
                $error = ($prompt->validate)($answer ?? '');

                if ($error) {
                    throw new \RuntimeException($error);
                }
            }

            return $answer;
        });

    return (new SymfonyStyle($input, $output))
        ->askQuestion($question);
});
```

備用方案必須為每個提示類別單獨配置。閉包將接收提示類別的實例，並且必須回傳提示的適當類型。

<a name="testing"></a>
## 測試

Laravel 提供了多種方法來測試您的命令是否顯示預期的 Prompt 訊息：

```php tab=Pest
test('report generation', function () {
    $this->artisan('report:generate')
        ->expectsPromptsInfo('Welcome to the application!')
        ->expectsPromptsWarning('This action cannot be undone')
        ->expectsPromptsError('Something went wrong')
        ->expectsPromptsAlert('Important notice!')
        ->expectsPromptsIntro('Starting process...')
        ->expectsPromptsOutro('Process completed!')
        ->expectsPromptsTable(
            headers: ['Name', 'Email'],
            rows: [
                ['Taylor Otwell', 'taylor@example.com'],
                ['Jason Beggs', 'jason@example.com'],
            ]
        )
        ->assertExitCode(0);
});
```

```php tab=PHPUnit
public function test_report_generation(): void
{
    $this->artisan('report:generate')
        ->expectsPromptsInfo('Welcome to the application!')
        ->expectsPromptsWarning('This action cannot be undone')
        ->expectsPromptsError('Something went wrong')
        ->expectsPromptsAlert('Important notice!')
        ->expectsPromptsIntro('Starting process...')
        ->expectsPromptsOutro('Process completed!')
        ->expectsPromptsTable(
            headers: ['Name', 'Email'],
            rows: [
                ['Taylor Otwell', 'taylor@example.com'],
                ['Jason Beggs', 'jason@example.com'],
            ]
        )
        ->assertExitCode(0);
}
```

