# Eloquent: Mutators & Casting

- [簡介](#introduction)
- [存取器與修改器](#accessors-and-mutators)
    - [定義存取器](#defining-an-accessor)
    - [定義修改器](#defining-a-mutator)
- [屬性型別轉換](#attribute-casting)
    - [陣列與 JSON 型別轉換](#array-and-json-casting)
    - [日期型別轉換](#date-casting)
    - [Enum 型別轉換](#enum-casting)
    - [加密型別轉換](#encrypted-casting)
    - [查詢時型別轉換](#query-time-casting)
- [自訂型別轉換](#custom-casts)
    - [值物件型別轉換](#value-object-casting)
    - [陣列 / JSON 序列化](#array-json-serialization)
    - [入站型別轉換](#inbound-casting)
    - [型別轉換參數](#cast-parameters)
    - [比較型別轉換值](#comparing-cast-values)
    - [可型別轉換](#castables)

<a name="introduction"></a>
## 簡介

存取器 (Accessors)、修改器 (Mutators) 和屬性型別轉換 (Attribute Casting) 允許你在 Eloquent 模型實例上取得或設定屬性值時對其進行轉換。例如，你可能希望使用 [Laravel 加密器](/docs/{{version}}/encryption) 在資料庫中儲存值時對其進行加密，然後在 Eloquent 模型上存取該屬性時自動解密。或者，你可能希望將資料庫中儲存的 JSON 字串在透過 Eloquent 模型存取時轉換為陣列。

<a name="accessors-and-mutators"></a>
## 存取器與修改器

<a name="defining-an-accessor"></a>
### 定義存取器

存取器在 Eloquent 屬性值被存取時對其進行轉換。要定義存取器，請在你的模型上建立一個受保護的方法來表示可存取的屬性。此方法名稱應與適用時的真實底層模型屬性 / 資料庫欄位的「駝峰式命名」表示法相對應。

在此範例中，我們將為 `first_name` 屬性定義一個存取器。當嘗試取得 `first_name` 屬性的值時，Eloquent 將自動呼叫此存取器。所有屬性存取器 / 修改器方法都必須宣告 `Illuminate\Database\Eloquent\Casts\Attribute` 的回傳型別提示：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Casts\Attribute;
use Illuminate\Database\Eloquent\Model;

class User extends Model
{
    /**
     * Get the user's first name.
     */
    protected function firstName(): Attribute
    {
        return Attribute::make(
            get: fn (string $value) => ucfirst($value),
        );
    }
}
```

所有存取器方法都回傳一個 `Attribute` 實例，該實例定義了屬性將如何被存取，以及可選地，如何被修改。在此範例中，我們只定義了屬性將如何被存取。為此，我們向 `Attribute` 類別建構函式提供了 `get` 引數。

如你所見，欄位的原始值會傳遞給存取器，允許你操作並回傳該值。要存取存取器的值，你只需在模型實例上存取 `first_name` 屬性即可：

```php
use App\Models\User;

$user = User::find(1);

$firstName = $user->first_name;
```

> [!NOTE]
> 如果你希望這些計算值被新增到模型的陣列 / JSON 表示法中，[你需要將它們附加](/docs/{{version}}/eloquent-serialization#appending-values-to-json)。

<a name="building-value-objects-from-multiple-attributes"></a>
#### 從多個屬性建立值物件

有時你的存取器可能需要將多個模型屬性轉換為單一的「值物件」。為此，你的 `get` 閉包可以接受第二個引數 `$attributes`，該引數將自動提供給閉包，並包含模型所有當前屬性的陣列：

```php
use App\Support\Address;
use Illuminate\Database\Eloquent\Casts\Attribute;

/**
 * Interact with the user's address.
 */
protected function address(): Attribute
{
    return Attribute::make(
        get: fn (mixed $value, array $attributes) => new Address(
            $attributes['address_line_one'],
            $attributes['address_line_two'],
        ),
    );
}
```

<a name="accessor-caching"></a>
#### 存取器快取

當從存取器回傳值物件時，對值物件所做的任何更改都將在模型儲存之前自動同步回模型。這是因為 Eloquent 會保留存取器回傳的實例，以便每次呼叫存取器時都能回傳相同的實例：

```php
use App\Models\User;

$user = User::find(1);

$user->address->lineOne = 'Updated Address Line 1 Value';
$user->address->lineTwo = 'Updated Address Line 2 Value';

$user->save();
```

但是，你可能希望有時為字串和布林值等基本值啟用快取，特別是當它們計算密集時。為此，你可以在定義存取器時呼叫 `shouldCache` 方法：

```php
protected function hash(): Attribute
{
    return Attribute::make(
        get: fn (string $value) => bcrypt(gzuncompress($value)),
    )->shouldCache();
}
```

如果你想禁用屬性的物件快取行為，你可以在定義屬性時呼叫 `withoutObjectCaching` 方法：

```php
/**
 * Interact with the user's address.
 */
protected function address(): Attribute
{
    return Attribute::make(
        get: fn (mixed $value, array $attributes) => new Address(
            $attributes['address_line_one'],
            $attributes['address_line_two'],
        ),
    )->withoutObjectCaching();
}
```

<a name="defining-a-mutator"></a>
### 定義修改器

修改器在 Eloquent 屬性值被設定時對其進行轉換。要定義修改器，你可以在定義屬性時提供 `set` 引數。讓我們為 `first_name` 屬性定義一個修改器。當我們嘗試在模型上設定 `first_name` 屬性的值時，此修改器將自動被呼叫：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Casts\Attribute;
use Illuminate\Database\Eloquent\Model;

class User extends Model
{
    /**
     * Interact with the user's first name.
     */
    protected function firstName(): Attribute
    {
        return Attribute::make(
            get: fn (string $value) => ucfirst($value),
            set: fn (string $value) => strtolower($value),
        );
    }
}
```

修改器閉包將接收正在設定到屬性上的值，允許你操作該值並回傳操作後的值。要使用我們的修改器，我們只需在 Eloquent 模型上設定 `first_name` 屬性即可：

```php
use App\Models\User;

$user = User::find(1);

$user->first_name = 'Sally';
```

在此範例中，`set` 回呼將以值 `Sally` 呼叫。然後修改器將對名稱應用 `strtolower` 函數，並將其結果值設定在模型的內部 `$attributes` 陣列中。

<a name="mutating-multiple-attributes"></a>
#### 修改多個屬性

有時你的修改器可能需要在底層模型上設定多個屬性。為此，你可以從 `set` 閉包回傳一個陣列。陣列中的每個鍵都應與與模型關聯的底層屬性 / 資料庫欄位相對應：

```php
use App\Support\Address;
use Illuminate\Database\Eloquent\Casts\Attribute;

/**
 * Interact with the user's address.
 */
protected function address(): Attribute
{
    return Attribute::make(
        get: fn (mixed $value, array $attributes) => new Address(
            $attributes['address_line_one'],
            $attributes['address_line_two'],
        ),
        set: fn (Address $value) => [
            'address_line_one' => $value->lineOne,
            'address_line_two' => $value->lineTwo,
        ],
    );
}
```

<a name="attribute-casting"></a>
## 屬性型別轉換

屬性型別轉換提供了與存取器和修改器類似的功能，而無需你在模型上定義任何額外的方法。相反，模型的 `casts` 方法提供了一種將屬性轉換為常見資料型別的便捷方式。

`casts` 方法應回傳一個陣列，其中鍵是要進行型別轉換的屬性名稱，值是你希望將欄位轉換為的型別。支援的型別轉換型別有：

<div class="content-list" markdown="1">

- `array`
- `AsFluent::class`
- `AsStringable::class`
- `AsUri::class`
- `boolean`
- `collection`
- `date`
- `datetime`
- `immutable_date`
- `immutable_datetime`
- <code>decimal:&lt;precision&gt;</code>
- `double`
- `encrypted`
- `encrypted:array`
- `encrypted:collection`
- `encrypted:object`
- `float`
- `hashed`
- `integer`
- `object`
- `real`
- `string`
- `timestamp`

</div>

為了演示屬性型別轉換，讓我們將 `is_admin` 屬性進行型別轉換，該屬性在我們的資料庫中儲存為整數 (`0` 或 `1`)，轉換為布林值：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class User extends Model
{
    /**
     * Get the attributes that should be cast.
     *
     * @return array<string, string>
     */
    protected function casts(): array
    {
        return [
            'is_admin' => 'boolean',
        ];
    }
}
```

定義型別轉換後，當你存取 `is_admin` 屬性時，它將始終被轉換為布林值，即使底層值在資料庫中儲存為整數：

```php
$user = App\Models\User::find(1);

if ($user->is_admin) {
    // ...
}
```

如果你需要在執行時新增一個新的臨時型別轉換，你可以使用 `mergeCasts` 方法。這些型別轉換定義將新增到模型上已定義的任何型別轉換中：

```php
$user->mergeCasts([
    'is_admin' => 'integer',
    'options' => 'object',
]);
```

> [!WARNING]
> `null` 的屬性將不會被型別轉換。此外，你不應定義與關聯關係同名的型別轉換（或屬性），也不應將型別轉換分配給模型的主鍵。

<a name="stringable-casting"></a>
#### 可字串化型別轉換

你可以使用 `Illuminate\Database\Eloquent\Casts\AsStringable` 型別轉換類別將模型屬性轉換為 [流暢的 Illuminate\Support\Stringable 物件](/docs/{{version}}/strings#fluent-strings-method-list)：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Casts\AsStringable;
use Illuminate\Database\Eloquent\Model;

class User extends Model
{
    /**
     * Get the attributes that should be cast.
     *
     * @return array<string, string>
     */
    protected function casts(): array
    {
        return [
            'directory' => AsStringable::class,
        ];
    }
}
```

<a name="array-and-json-casting"></a>
### 陣列與 JSON 型別轉換

`array` 型別轉換在處理儲存為序列化 JSON 的欄位時特別有用。例如，如果你的資料庫有一個包含序列化 JSON 的 `JSON` 或 `TEXT` 欄位型別，將 `array` 型別轉換新增到該屬性將在你從 Eloquent 模型存取它時自動將該屬性反序列化為 PHP 陣列：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class User extends Model
{
    /**
     * Get the attributes that should be cast.
     *
     * @return array<string, string>
     */
    protected function casts(): array
    {
        return [
            'options' => 'array',
        ];
    }
}
```

定義型別轉換後，你可以存取 `options` 屬性，它將自動從 JSON 反序列化為 PHP 陣列。當你設定 `options` 屬性的值時，給定的陣列將自動序列化回 JSON 以供儲存：

```php
use App\Models\User;

$user = User::find(1);

$options = $user->options;

$options['key'] = 'value';

$user->options = $options;

$user->save();
```

要以更簡潔的語法更新 JSON 屬性的單個欄位，你可以[將屬性設定為可大量賦值](/docs/{{version}}/eloquent#mass-assignment-json-columns) 並在呼叫 `update` 方法時使用 `->` 運算子：

```php
$user = User::find(1);

$user->update(['options->key' => 'value']);
```

<a name="json-and-unicode"></a>
#### JSON 與 Unicode

如果你想將陣列屬性儲存為帶有未逸出 Unicode 字元的 JSON，你可以使用 `json:unicode` 型別轉換：

```php
/**
 * Get the attributes that should be cast.
 *
 * @return array<string, string>
 */
protected function casts(): array
{
    return [
        'options' => 'json:unicode',
    ];
}
```

<a name="array-object-and-collection-casting"></a>
#### 陣列物件與集合型別轉換

儘管標準的 `array` 型別轉換對於許多應用程式來說已經足夠，但它也有一些缺點。由於 `array` 型別轉換回傳的是基本型別，因此無法直接修改陣列的偏移量。例如，以下程式碼將觸發 PHP 錯誤：

```php
$user = User::find(1);

$user->options['key'] = $value;
```

為了解決這個問題，Laravel 提供了一個 `AsArrayObject` 型別轉換，它將你的 JSON 屬性轉換為 [ArrayObject](https://www.php.net/manual/en/class.arrayobject.php) 類別。此功能是透過 Laravel 的[自訂型別轉換](#custom-casts)實作的，它允許 Laravel 智慧地快取和轉換變異的物件，以便可以修改單個偏移量而不會觸發 PHP 錯誤。要使用 `AsArrayObject` 型別轉換，只需將其分配給屬性即可：

```php
use Illuminate\Database\Eloquent\Casts\AsArrayObject;

/**
 * Get the attributes that should be cast.
 *
 * @return array<string, string>
 */
protected function casts(): array
{
    return [
        'options' => AsArrayObject::class,
    ];
}
```

同樣，Laravel 提供了一個 `AsCollection` 型別轉換，它將你的 JSON 屬性轉換為 Laravel [Collection](/docs/{{version}}/collections) 實例：

```php
use Illuminate\Database\Eloquent\Casts\AsCollection;

/**
 * Get the attributes that should be cast.
 *
 * @return array<string, string>
 */
protected function casts(): array
{
    return [
        'options' => AsCollection::class,
    ];
}
```

如果你希望 `AsCollection` 型別轉換實例化自訂集合類別而不是 Laravel 的基本集合類別，你可以將集合類別名稱作為型別轉換引數提供：

```php
use App\Collections\OptionCollection;
use Illuminate\Database\Eloquent\Casts\AsCollection;

/**
 * Get the attributes that should be cast.
 *
 * @return array<string, string>
 */
protected function casts(): array
{
    return [
        'options' => AsCollection::using(OptionCollection::class),
    ];
}
```

`of` 方法可用於指示集合項目應透過集合的 [mapInto 方法](/docs/{{version}}/collections#method-mapinto) 映射到給定的類別中：

```php
use App\ValueObjects\Option;
use Illuminate\Database\Eloquent\Casts\AsCollection;

/**
 * Get the attributes that should be cast.
 *
 * @return array<string, string>
 */
protected function casts(): array
{
    return [
        'options' => AsCollection::of(Option::class)
    ];
}
```

將集合映射到物件時，物件應實作 `Illuminate\Contracts\Support\Arrayable` 和 `JsonSerializable` 介面，以定義其實例應如何序列化為資料庫中的 JSON：

```php
<?php

namespace App\ValueObjects;

use Illuminate\Contracts\Support\Arrayable;
use JsonSerializable;

class Option implements Arrayable, JsonSerializable
{
    public string $name;
    public mixed $value;
    public bool $isLocked;

    /**
     * Create a new Option instance.
     */
    public function __construct(array $data)
    {
        $this->name = $data['name'];
        $this->value = $data['value'];
        $this->isLocked = $data['is_locked'];
    }

    /**
     * Get the instance as an array.
     *
     * @return array{name: string, data: string, is_locked: bool}
     */
    public function toArray(): array
    {
        return [
            'name' => $this->name,
            'value' => $this->value,
            'is_locked' => $this->isLocked,
        ];
    }

    /**
     * Specify the data which should be serialized to JSON.
     *
     * @return array{name: string, data: string, is_locked: bool}
     */
    public function jsonSerialize(): array
    {
        return $this->toArray();
    }
}
```

<a name="date-casting"></a>
### 日期型別轉換

預設情況下，Eloquent 會將 `created_at` 和 `updated_at` 欄位轉換為 [Carbon](https://github.com/briannesbitt/Carbon) 實例，該實例擴展了 PHP `DateTime` 類別並提供了各種有用的方法。你可以透過在模型的 `casts` 方法中定義額外的日期型別轉換來轉換額外的日期屬性。通常，日期應使用 `datetime` 或 `immutable_datetime` 型別轉換型別進行型別轉換。

定義 `date` 或 `datetime` 型別轉換時，你還可以指定日期的格式。當[模型序列化為陣列或 JSON](/docs/{{version}}/eloquent-serialization) 時，將使用此格式：

```php
/**
 * Get the attributes that should be cast.
 *
 * @return array<string, string>
 */
protected function casts(): array
{
    return [
        'created_at' => 'datetime:Y-m-d',
    ];
}
```

當欄位被轉換為日期時，你可以將對應的模型屬性值設定為 UNIX 時間戳記、日期字串 (`Y-m-d`)、日期時間字串或 `DateTime` / `Carbon` 實例。日期的值將被正確轉換並儲存在你的資料庫中。

你可以透過在模型上定義 `serializeDate` 方法來自訂所有模型日期的預設序列化格式。此方法不會影響你的日期在資料庫中儲存時的格式：

```php
/**
 * Prepare a date for array / JSON serialization.
 */
protected function serializeDate(DateTimeInterface $date): string
{
    return $date->format('Y-m-d');
}
```

要指定在資料庫中實際儲存模型日期時應使用的格式，你應該在模型上定義 `$dateFormat` 屬性：

```php
/**
 * The storage format of the model's date columns.
 *
 * @var string
 */
protected $dateFormat = 'U';
```

<a name="date-casting-and-timezones"></a>
#### 日期型別轉換、序列化與時區

預設情況下，`date` 和 `datetime` 型別轉換會將日期序列化為 UTC ISO-8601 日期字串 (`YYYY-MM-DDTHH:MM:SS.uuuuuuZ`)，無論你的應用程式 `timezone` 設定選項中指定的時區為何。強烈建議你始終使用此序列化格式，並透過不更改應用程式 `timezone` 設定選項的預設 `UTC` 值來將應用程式的日期儲存在 UTC 時區。在整個應用程式中一致地使用 UTC 時區將提供與用 PHP 和 JavaScript 編寫的其他日期操作函式庫的最大互通性。

如果自訂格式應用於 `date` 或 `datetime` 型別轉換，例如 `datetime:Y-m-d H:i:s`，則 Carbon 實例的內部時區將在日期序列化期間使用。通常，這將是你的應用程式 `timezone` 設定選項中指定的時區。但是，重要的是要注意，`timestamp` 欄位（例如 `created_at` 和 `updated_at`）不受此行為的限制，並且始終以 UTC 格式化，無論應用程式的時區設定如何。

<a name="enum-casting"></a>
### Enum 型別轉換

Eloquent 還允許你將屬性值轉換為 PHP [Enums](https://www.php.net/manual/en/language.enumerations.backed.php)。為此，你可以在模型的 `casts` 方法中指定要轉換的屬性和 Enum：

```php
use App\Enums\ServerStatus;

/**
 * Get the attributes that should be cast.
 *
 * @return array<string, string>
 */
protected function casts(): array
{
    return [
        'status' => ServerStatus::class,
    ];
}
```

一旦你在模型上定義了型別轉換，當你與屬性互動時，指定的屬性將自動在 Enum 之間進行型別轉換：

```php
if ($server->status == ServerStatus::Provisioned) {
    $server->status = ServerStatus::Ready;

    $server->save();
}
```

<a name="casting-arrays-of-enums"></a>
#### Enum 陣列型別轉換

有時你可能需要模型在單一欄位中儲存 Enum 值陣列。為此，你可以使用 Laravel 提供的 `AsEnumArrayObject` 或 `AsEnumCollection` 型別轉換：

```php
use App\Enums\ServerStatus;
use Illuminate\Database\Eloquent\Casts\AsEnumCollection;

/**
 * Get the attributes that should be cast.
 *
 * @return array<string, string>
 */
protected function casts(): array
{
    return [
        'statuses' => AsEnumCollection::of(ServerStatus::class),
    ];
}
```

<a name="encrypted-casting"></a>
### 加密型別轉換

`encrypted` 型別轉換將使用 Laravel 內建的[加密](/docs/{{version}}/encryption)功能加密模型的屬性值。此外，`encrypted:array`、`encrypted:collection`、`encrypted:object`、`AsEncryptedArrayObject` 和 `AsEncryptedCollection` 型別轉換的功能與其未加密的對應項相同；但是，正如你所預期的，底層值在儲存到資料庫時會被加密。

由於加密文字的最終長度不可預測且比其純文字對應項長，請確保相關的資料庫欄位是 `TEXT` 型別或更大。此外，由於值在資料庫中是加密的，因此你將無法查詢或搜尋加密的屬性值。

<a name="key-rotation"></a>
#### 金鑰輪替

如你所知，Laravel 使用應用程式 `app` 設定檔中指定的 `key` 設定值來加密字串。通常，此值對應於 `APP_KEY` 環境變數的值。如果你需要輪替應用程式的加密金鑰，你需要使用新金鑰手動重新加密你的加密屬性。

<a name="query-time-casting"></a>
### 查詢時型別轉換

有時你可能需要在執行查詢時應用型別轉換，例如從表格中選取原始值時。例如，考慮以下查詢：

```php
use App\Models\Post;
use App\Models\User;

$users = User::select([
    'users.*',
    'last_posted_at' => Post::selectRaw('MAX(created_at)')
        ->whereColumn('user_id', 'users.id')
])->get();
```

此查詢結果中的 `last_posted_at` 屬性將是一個簡單的字串。如果我們可以在執行查詢時將 `datetime` 型別轉換應用於此屬性，那將會很棒。幸運的是，我們可以使用 `withCasts` 方法來實現這一點：

```php
$users = User::select([
    'users.*',
    'last_posted_at' => Post::selectRaw('MAX(created_at)')
        ->whereColumn('user_id', 'users.id')
])->withCasts([
    'last_posted_at' => 'datetime'
])->get();
```

<a name="custom-casts"></a>
## 自訂型別轉換

Laravel 有各種內建的、有用的型別轉換型別；但是，你可能偶爾需要定義自己的型別轉換型別。要建立型別轉換，請執行 `make:cast` Artisan 命令。新的型別轉換類別將放置在你的 `app/Casts` 目錄中：

```shell
php artisan make:cast AsJson
```

所有自訂型別轉換類別都實作 `CastsAttributes` 介面。實作此介面的類別必須定義 `get` 和 `set` 方法。`get` 方法負責將資料庫中的原始值轉換為型別轉換值，而 `set` 方法應將型別轉換值轉換為可以儲存在資料庫中的原始值。作為範例，我們將重新實作內建的 `json` 型別轉換型別作為自訂型別轉換型別：

```php
<?php

namespace App\Casts;

use Illuminate\Contracts\Database\Eloquent\CastsAttributes;
use Illuminate\Database\Eloquent\Model;

class AsJson implements CastsAttributes
{
    /**
     * Cast the given value.
     *
     * @param  array<string, mixed>  $attributes
     * @return array<string, mixed>
     */
    public function get(
        Model $model,
        string $key,
        mixed $value,
        array $attributes,
    ): array {
        return json_decode($value, true);
    }

    /**
     * Prepare the given value for storage.
     *
     * @param  array<string, mixed>  $attributes
     */
    public function set(
        Model $model,
        string $key,
        mixed $value,
        array $attributes,
    ): string {
        return json_encode($value);
    }
}
```

一旦你定義了自訂型別轉換型別，你就可以使用其類別名稱將其附加到模型屬性：

```php
<?php

namespace App\Models;

use App\Casts\AsJson;
use Illuminate\Database\Eloquent\Model;

class User extends Model
{
    /**
     * Get the attributes that should be cast.
     *
     * @return array<string, string>
     */
    protected function casts(): array
    {
        return [
            'options' => AsJson::class,
        ];
    }
}
```

<a name="value-object-casting"></a>
### 值物件型別轉換

你不僅限於將值轉換為基本型別。你還可以將值轉換為物件。定義將值轉換為物件的自訂型別轉換與轉換為基本型別非常相似；但是，如果你的值物件包含多個資料庫欄位，則 `set` 方法必須回傳鍵 / 值對的陣列，這些鍵 / 值對將用於在模型上設定原始、可儲存的值。如果你的值物件只影響單個欄位，你應該只回傳可儲存的值。

作為範例，我們將定義一個自訂型別轉換類別，該類別將多個模型值轉換為單個 `Address` 值物件。我們假設 `Address` 值物件有兩個公共屬性：`lineOne` 和 `lineTwo`：

```php
<?php

namespace App\Casts;

use App\ValueObjects\Address;
use Illuminate\Contracts\Database\Eloquent\CastsAttributes;
use Illuminate\Database\Eloquent\Model;
use InvalidArgumentException;

class AsAddress implements CastsAttributes
{
    /**
     * Cast the given value.
     *
     * @param  array<string, mixed>  $attributes
     */
    public function get(
        Model $model,
        string $key,
        mixed $value,
        array $attributes,
    ): Address {
        return new Address(
            $attributes['address_line_one'],
            $attributes['address_line_two']
        );
    }

    /**
     * Prepare the given value for storage.
     *
     * @param  array<string, mixed>  $attributes
     * @return array<string, string>
     */
    public function set(
        Model $model,
        string $key,
        mixed $value,
        array $attributes,
    ): array {
        if (! $value instanceof Address) {
            throw new InvalidArgumentException('The given value is not an Address instance.');
        }

        return [
            'address_line_one' => $value->lineOne,
            'address_line_two' => $value->lineTwo,
        ];
    }
}
```

當轉換為值物件時，對值物件所做的任何更改都將在模型儲存之前自動同步回模型：

```php
use App\Models\User;

$user = User::find(1);

$user->address->lineOne = 'Updated Address Value';

$user->save();
```

> [!NOTE]
> 如果你打算將包含值物件的 Eloquent 模型序列化為 JSON 或陣列，你應該在值物件上實作 `Illuminate\Contracts\Support\Arrayable` 和 `JsonSerializable` 介面。

<a name="value-object-caching"></a>
#### 值物件快取

當轉換為值物件的屬性被解析時，它們會被 Eloquent 快取。因此，如果再次存取該屬性，將回傳相同的物件實例。

如果你想禁用自訂型別轉換類別的物件快取行為，你可以在自訂型別轉換類別上宣告一個公共的 `withoutObjectCaching` 屬性：

```php
class AsAddress implements CastsAttributes
{
    public bool $withoutObjectCaching = true;

    // ...
}
```

<a name="array-json-serialization"></a>
### 陣列 / JSON 序列化

當 Eloquent 模型使用 `toArray` 和 `toJson` 方法轉換為陣列或 JSON 時，你的自訂型別轉換值物件通常也會被序列化，只要它們實作 `Illuminate\Contracts\Support\Arrayable` 和 `JsonSerializable` 介面。但是，當使用第三方函式庫提供的值物件時，你可能無法將這些介面新增到物件中。

因此，你可以指定你的自訂型別轉換類別將負責序列化值物件。為此，你的自訂型別轉換類別應實作 `Illuminate\Contracts\Database\Eloquent\SerializesCastableAttributes` 介面。此介面規定你的類別應包含一個 `serialize` 方法，該方法應回傳值物件的序列化形式：

```php
/**
 * Get the serialized representation of the value.
 *
 * @param  array<string, mixed>  $attributes
 */
public function serialize(
    Model $model,
    string $key,
    mixed $value,
    array $attributes,
): string {
    return (string) $value;
}
```

<a name="inbound-casting"></a>
### 入站型別轉換

有時，你可能需要編寫一個自訂型別轉換類別，該類別只轉換正在設定到模型上的值，並且在從模型中取得屬性時不執行任何操作。

僅入站的自訂型別轉換應實作 `CastsInboundAttributes` 介面，該介面只要求定義一個 `set` 方法。`make:cast` Artisan 命令可以使用 `--inbound` 選項呼叫，以生成一個僅入站的型別轉換類別：

```shell
php artisan make:cast AsHash --inbound
```

僅入站型別轉換的經典範例是「雜湊」型別轉換。例如，我們可以定義一個型別轉換，透過給定的演算法雜湊入站值：

```php
<?php

namespace App\Casts;

use Illuminate\Contracts\Database\Eloquent\CastsInboundAttributes;
use Illuminate\Database\Eloquent\Model;

class AsHash implements CastsInboundAttributes
{
    /**
     * Create a new cast class instance.
     */
    public function __construct(
        protected string|null $algorithm = null,
    ) {}

    /**
     * Prepare the given value for storage.
     *
     * @param  array<string, mixed>  $attributes
     */
    public function set(
        Model $model,
        string $key,
        mixed $value,
        array $attributes,
    ): string {
        return is_null($this->algorithm)
            ? bcrypt($value)
            : hash($this->algorithm, $value);
    }
}
```

<a name="cast-parameters"></a>
### 型別轉換參數

將自訂型別轉換附加到模型時，可以透過使用 `:` 字元將其與類別名稱分開並用逗號分隔多個參數來指定型別轉換參數。參數將傳遞給型別轉換類別的建構函式：

```php
/**
 * Get the attributes that should be cast.
 *
 * @return array<string, string>
 */
protected function casts(): array
{
    return [
        'secret' => AsHash::class.':sha256',
    ];
}
```

<a name="comparing-cast-values"></a>
### 比較型別轉換值

如果你想定義如何比較兩個給定的型別轉換值以確定它們是否已更改，你的自訂型別轉換類別可以實作 `Illuminate\Contracts\Database\Eloquent\ComparesCastableAttributes` 介面。這允許你精細控制 Eloquent 認為已更改的值，從而在模型更新時將其儲存到資料庫。

此介面規定你的類別應包含一個 `compare` 方法，如果給定值被認為相等，則該方法應回傳 `true`：

```php
/**
 * Determine if the given values are equal.
 *
 * @param  \Illuminate\Database\Eloquent\Model  $model
 * @param  string  $key
 * @param  mixed  $firstValue
 * @param  mixed  $secondValue
 * @return bool
 */
public function compare(
    Model $model,
    string $key,
    mixed $firstValue,
    mixed $secondValue
): bool {
    return $firstValue === $secondValue;
}
```

<a name="castables"></a>
### 可型別轉換

你可能希望允許應用程式的值物件定義自己的自訂型別轉換類別。你可以將實作 `Illuminate\Contracts\Database\Eloquent\Castable` 介面的值物件類別附加到模型，而不是將自訂型別轉換類別附加到模型：

```php
use App\ValueObjects\Address;

protected function casts(): array
{
    return [
        'address' => Address::class,
    ];
}
```

實作 `Castable` 介面的物件必須定義一個 `castUsing` 方法，該方法回傳負責將 `Castable` 類別進行型別轉換的自訂型別轉換類別的類別名稱：

```php
<?php

namespace App\ValueObjects;

use Illuminate\Contracts\Database\Eloquent\Castable;
use App\Casts\AsAddress;

class Address implements Castable
{
    /**
     * Get the name of the caster class to use when casting from / to this cast target.
     *
     * @param  array<string, mixed>  $arguments
     */
    public static function castUsing(array $arguments): string
    {
        return AsAddress::class;
    }
}
```

使用 `Castable` 類別時，你仍然可以在 `casts` 方法定義中提供引數。引數將傳遞給 `castUsing` 方法：

```php
use App\ValueObjects\Address;

protected function casts(): array
{
    return [
        'address' => Address::class.':argument',
    ];
}
```

<a name="anonymous-cast-classes"></a>
#### 可型別轉換與匿名型別轉換類別

透過將「可型別轉換」與 PHP 的[匿名類別](https://www.php.net/manual/en/language.oop5.anonymous.php)結合，你可以將值物件及其型別轉換邏輯定義為單個可型別轉換物件。為此，從值物件的 `castUsing` 方法回傳一個匿名類別。匿名類別應實作 `CastsAttributes` 介面：

```php
<?php

namespace App\ValueObjects;

use Illuminate\Contracts\Database\Eloquent\Castable;
use Illuminate\Contracts\Database\Eloquent\CastsAttributes;

class Address implements Castable
{
    // ...

    /**
     * Get the caster class to use when casting from / to this cast target.
     *
     * @param  array<string, mixed>  $arguments
     */
    public static function castUsing(array $arguments): CastsAttributes
    {
        return new class implements CastsAttributes
        {
            public function get(
                Model $model,
                string $key,
                mixed $value,
                array $attributes,
            ): Address {
                return new Address(
                    $attributes['address_line_one'],
                    $attributes['address_line_two']
                );
            }

            public function set(
                Model $model,
                string $key,
                mixed $value,
                array $attributes,
            ): array {
                return [
                    'address_line_one' => $value->lineOne,
                    'address_line_two' => $value->lineTwo,
                ];
            }
        };
    }
}
```

