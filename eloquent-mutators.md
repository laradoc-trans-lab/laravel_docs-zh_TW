# Eloquent: 修改器與型別轉換

- [簡介](#introduction)
- [存取器與修改器](#accessors-and-mutators)
    - [定義存取器](#defining-an-accessor)
    - [定義修改器](#defining-a-mutator)
- [屬性型別轉換](#attribute-casting)
    - [陣列與 JSON 型別轉換](#array-and-json-casting)
    - [二進位型別轉換](#binary-casting)
    - [日期型別轉換](#date-casting)
    - [列舉 (Enum) 型別轉換](#enum-casting)
    - [加密型別轉換](#encrypted-casting)
    - [查詢時型別轉換](#query-time-casting)
- [自定義型別轉換](#custom-casts)
    - [數值物件型別轉換](#value-object-casting)
    - [陣列 / JSON 序列化](#array-json-serialization)
    - [入向型別轉換](#inbound-casting)
    - [型別轉換參數](#cast-parameters)
    - [比較型別轉換值](#comparing-cast-values)
    - [可轉換對象 (Castables)](#castables)

<a name="introduction"></a>
## 簡介

存取器、修改器以及屬性型別轉換讓您在從模型實例（model instances）取得或設定屬性值時，能夠對 Eloquent 屬性值進行轉換。例如，您可以使用 [Laravel encrypter](/docs/{{version}}/encryption) 在值儲存於資料庫時將其加密，然後在透過 Eloquent 模型存取該屬性時自動將其解密。或者，您可能希望將儲存在資料庫中的 JSON 字串在透過 Eloquent 模型存取時轉換為陣列。


<a name="accessors-and-mutators"></a>
## 存取器與修改器


<a name="defining-an-accessor"></a>
### 定義存取器

存取器會在 Eloquent 屬性被存取時對其值進行轉換。要定義存取器，請在模型中建立一個受保護（protected）的方法來代表該可存取的屬性。在適用的情況下，此方法名稱應對應於底層模型屬性或資料庫欄位的「駝峰式命名（camel case）」表示法。

在這個範例中，我們將為 `first_name` 屬性定義一個存取器。當嘗試取得 `first_name` 屬性的值時，Eloquent 會自動呼叫此存取器。所有屬性存取器或修改器方法都必須宣告 `Illuminate\Database\Eloquent\Casts\Attribute` 的回傳型別提示：

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

所有存取器方法都會回傳一個 `Attribute` 實例，該實例定義了屬性將如何被存取，以及（選擇性地）如何被修改。在這個範例中，我們僅定義屬性如何被存取。為了實現這一點，我們向 `Attribute` 類別的建構子提供 `get` 引數。

如您所見，欄位的原始值會被傳遞給存取器，讓您可以操作並回傳該值。要存取存取器的值，您只需存取模型實例上的 `first_name` 屬性即可：

```php
use App\Models\User;

$user = User::find(1);

$firstName = $user->first_name;
```

> [!NOTE]
> 如果您希望將這些計算出的值添加到模型的陣列或 JSON 表示中，[您需要將它們附加 (append) 進去](/docs/{{version}}/eloquent-serialization#appending-values-to-json)。


<a name="building-value-objects-from-multiple-attributes"></a>
#### 從多個屬性構建數值物件

有時候您的存取器可能需要將多個模型屬性轉換為單個「數值物件（value object）」。若要實現此功能，您的 `get` 閉包可以接受第二個引數 `$attributes`，該引數會自動傳遞給閉包，並包含模型目前所有屬性的陣列：

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

當從存取器回傳數值物件時，對該數值物件所做的任何更改在模型儲存之前，都會自動同步回模型。這是因為 Eloquent 會保留存取器回傳的實例，以便每次呼叫存取器時都能回傳相同的實例：

```php
use App\Models\User;

$user = User::find(1);

$user->address->lineOne = 'Updated Address Line 1 Value';
$user->address->lineTwo = 'Updated Address Line 2 Value';

$user->save();
```

然而，您有時可能希望為字串和布林值等原始型別啟用快取，特別是在計算成本較高時。為了實現這一點，您可以在定義存取器時呼叫 `shouldCache` 方法：

```php
protected function hash(): Attribute
{
    return Attribute::make(
        get: fn (string $value) => bcrypt(gzuncompress($value)),
    )->shouldCache();
}
```

如果您想禁用屬性的物件快取行為，可以在定義屬性時呼叫 `withoutObjectCaching` 方法：

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

修改器會在 Eloquent 屬性被設定時對其值進行轉換。要定義修改器，您可以在定義屬性時提供 `set` 引數。讓我們為 `first_name` 屬性定義一個修改器。當我們嘗試設定模型上 `first_name` 屬性的值時，此修改器會被自動呼叫：

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

修改器閉包將接收被設定到屬性上的值，讓您可以操作該值並回傳操作後的結果。要使用我們的修改器，我們只需要設定 Eloquent 模型上的 `first_name` 屬性：

```php
use App\Models\User;

$user = User::find(1);

$user->first_name = 'Sally';
```

在這個範例中，`set` 回呼函數將帶著值 `Sally` 被呼叫。接著修改器會對該名稱套用 `strtolower` 函數，並將結果值設定在模型的內部 `$attributes` 陣列中。


<a name="mutating-multiple-attributes"></a>
#### 修改多個屬性

有時候您的修改器可能需要設定底層模型上的多個屬性。若要實現此功能，您可以從 `set` 閉包回傳一個陣列。陣列中的每個鍵（key）都應對應於模型相關的底層屬性或資料庫欄位：

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

屬性型別轉換提供了與存取器和修改器類似的功能，且不需要你在模型中定義任何額外的方法。相反地，模型的 `casts` 方法提供了一種方便的方式，將屬性轉換為常見的資料型別。

`casts` 方法應該回傳一個陣列，其中鍵 (key) 是被轉換的屬性名稱，而值 (value) 是你希望將該欄位轉換成的型別。支援的型別轉換如下：

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

為了演示屬性型別轉換，讓我們將 `is_admin` 屬性（在資料庫中以整數 (`0` 或 `1`) 儲存）轉換為布林值：

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

定義型別轉換後，當你存取 `is_admin` 屬性時，它將始終被轉換為布林值，即使底層值在資料庫中是以整數儲存：

```php
$user = App\Models\User::find(1);

if ($user->is_admin) {
    // ...
}
```

如果你需要在執行階段新增一個臨時的型別轉換，可以使用 `mergeCasts` 方法。這些型別轉換定義將被添加到模型中已定義的任何型別轉換中：

```php
$user->mergeCasts([
    'is_admin' => 'integer',
    'options' => 'object',
]);
```

> [!WARNING]
> 為 `null` 的屬性將不會被轉換。此外，你不應該定義與關聯 (relationship) 同名的型別轉換（或屬性），也不應該將型別轉換分配給模型的主鍵。


<a name="stringable-casting"></a>
#### Stringable 型別轉換

你可以使用 `Illuminate\Database\Eloquent\Casts\AsStringable` 型別轉換類別，將模型屬性轉換為 [流暢的 Illuminate\Support\Stringable 物件](/docs/{{version}}/strings#fluent-strings-method-list)：

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

`array` 型別轉換在處理儲存為序列化 JSON 的欄位時特別有用。例如，如果您的資料庫具有包含序列化 JSON 的 `JSON` 或 `TEXT` 欄位型別，將 `array` 型別轉換新增至該屬性，便可在您透過 Eloquent 模型存取該屬性時，自動將其反序列化為 PHP 陣列：

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

一旦定義了型別轉換，您就可以存取 `options` 屬性，它會自動從 JSON 反序列化為 PHP 陣列。當您設定 `options` 屬性的值時，提供的陣列將自動被序列化回 JSON 以便儲存：

```php
use App\Models\User;

$user = User::find(1);

$options = $user->options;

$options['key'] = 'value';

$user->options = $options;

$user->save();
```

若要以更簡潔的語法更新 JSON 屬性的單一欄位，您可以[將該屬性設為可大量賦值](/docs/{{version}}/eloquent#mass-assignment-json-columns)，並在呼叫 `update` 方法時使用 `->` 運算子：

```php
$user = User::find(1);

$user->update(['options->key' => 'value']);
```


<a name="json-and-unicode"></a>
#### JSON 與 Unicode

如果您想將陣列屬性儲存為具有未轉義 Unicode 字元的 JSON，可以使用 `json:unicode` 型別轉換：

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

雖然標準的 `array` 型別轉換對於許多應用程式來說已經足夠，但它確實有一些缺點。由於 `array` 型別轉換回傳的是原始型別 (primitive type)，因此無法直接修改陣列的偏移量 (offset)。例如，以下程式碼將會觸發 PHP 錯誤：

```php
$user = User::find(1);

$user->options['key'] = $value;
```

為了解決這個問題，Laravel 提供了 `AsArrayObject` 型別轉換，可將您的 JSON 屬性轉換為 [ArrayObject](https://www.php.net/manual/en/class.arrayobject.php) 類別。此功能是使用 Laravel 的[自定義型別轉換](#custom-casts) 實作的，這讓 Laravel 能智慧地快取並轉換被修改的物件，使得個別的偏移量可以在不觸發 PHP 錯誤的情況下被修改。要使用 `AsArrayObject` 型別轉換，只需將其指派給屬性即可：

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

同樣地，Laravel 提供了 `AsCollection` 型別轉換，可將您的 JSON 屬性轉換為 Laravel [Collection](/docs/{{version}}/collections) 實例：

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

如果您希望 `AsCollection` 型別轉換實例化一個自定義的集合類別而非 Laravel 的基礎集合類別，您可以將集合類別名稱作為型別轉換參數提供：

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

`of` 方法可用於指定集合項目應透過集合的 [mapInto 方法](/docs/{{version}}/collections#method-mapinto) 映射到給定的類別：

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

當將集合映射到物件時，該物件應實作 `Illuminate\Contracts\Support\Arrayable` 與 `JsonSerializable` 介面，以定義其實例應如何被序列化為資料庫中的 JSON：

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


<a name="binary-casting"></a>
### 二進位型別轉換

如果您的 Eloquent 模型除了自動遞增的 ID 欄位外，還有[二進位型別](/docs/{{version}}/migrations#column-method-binary) 的 `uuid` 或 `ulid` 欄位，您可以使用 `AsBinary` 型別轉換，將值自動轉換為其二進位表示形式，反之亦然：

```php
use Illuminate\Database\Eloquent\Casts\AsBinary;

/**
 * Get the attributes that should be cast.
 *
 * @return array<string, string>
 */
protected function casts(): array
{
    return [
        'uuid' => AsBinary::uuid(),
        'ulid' => AsBinary::ulid(),
    ];
}
```

一旦在模型上定義了型別轉換，您就可以將 UUID / ULID 屬性值設定為物件實例或字串。Eloquent 會自動將該值轉換為其二進位表示形式。在取出屬性值時，您將始終收到一個純文字字串值：

```php
use Illuminate\Support\Str;

$user->uuid = Str::uuid();

return $user->uuid;

// "6e8cdeed-2f32-40bd-b109-1e4405be2140"
```

<a name="date-casting"></a>
### 日期型別轉換

預設情況下，Eloquent 會將 `created_at` 與 `updated_at` 欄位轉換為 [Carbon](https://github.com/briannesbitt/Carbon) 的實例，Carbon 擴展了 PHP 的 `DateTime` 類別，並提供了許多實用的方法。您可以在模型的 `casts` 方法中定義額外的日期轉換，以轉換其他的日期屬性。通常，日期應使用 `datetime` 或 `immutable_datetime` 型別轉換類型。

在定義 `date` 或 `datetime` 轉換時，您也可以指定日期的格式。此格式將在 [模型被序列化為陣列或 JSON](/docs/{{version}}/eloquent-serialization) 時被使用：

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

當欄位被轉換為日期時，您可以將對應的模型屬性值設定為 UNIX 時間戳記、日期字串 (`Y-m-d`)、日期時間字串，或是 `DateTime` / `Carbon` 實例。日期值將被正確地轉換並儲存在您的資料庫中。

您可以透過在模型上定義 `serializeDate` 方法，來自定義模型中所有日期的預設序列化格式。此方法不會影響日期儲存在資料庫中的格式：

```php
/**
 * Prepare a date for array / JSON serialization.
 */
protected function serializeDate(DateTimeInterface $date): string
{
    return $date->format('Y-m-d');
}
```

若要指定在將模型日期實際儲存到資料庫時應使用的格式，您應該在模型的 `Table` 屬性中使用 `dateFormat` 參數：

```php
use Illuminate\Database\Eloquent\Attributes\Table;

#[Table(dateFormat: 'U')]
class Flight extends Model
{
    // ...
}
```


<a name="date-casting-and-timezones"></a>
#### 日期型別轉換、序列化與時區

預設情況下，`date` 與 `datetime` 轉換會將日期序列化為 UTC ISO-8601 日期字串 (`YYYY-MM-DDTHH:MM:SS.uuuuuuZ`)，無論您在應用程式的 `timezone` 設定選項中指定了哪個時區。我們強烈建議您始終使用此序列化格式，並將應用程式的日期儲存在 UTC 時區（請勿將應用程式的 `timezone` 設定選項從預設的 `UTC` 值修改）。在整個應用程式中一致地使用 UTC 時區，將能與其他用 PHP 與 JavaScript 編寫的日期處理函式庫提供最高程度的互操作性。

如果對 `date` 或 `datetime` 轉換應用了自定義格式（例如 `datetime:Y-m-d H:i:s`），在日期序列化期間將使用 Carbon 實例的內部時區。通常，這會是您在應用程式的 `timezone` 設定選項中指定的時區。然而，請注意 `timestamp` 類型的欄位（例如 `created_at` 與 `updated_at`）不受此行為影響，無論應用程式的時區設定為何，它們始終以 UTC 格式化。


<a name="enum-casting"></a>
### 列舉 (Enum) 型別轉換

Eloquent 還允許您將屬性值轉換為 PHP [列舉 (Enums)](https://www.php.net/manual/en/language.enumerations.backed.php)。為了實現這一點，您可以在模型的 `casts` 方法中指定您想要轉換的屬性與列舉：

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

一旦您在模型上定義了轉換，當您與該屬性互動時，指定的屬性將自動在列舉之間進行轉換：

```php
if ($server->status == ServerStatus::Provisioned) {
    $server->status = ServerStatus::Ready;

    $server->save();
}
```


<a name="casting-arrays-of-enums"></a>
#### 列舉陣列的型別轉換

有時您可能需要模型在單一欄位中儲存一個列舉值的陣列。為了實現這一點，您可以使用 Laravel 提供的 `AsEnumArrayObject` 或 `AsEnumCollection` 轉換：

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

`encrypted` 轉換將使用 Laravel 內建的 [加密](/docs/{{version}}/encryption) 功能來加密模型的屬性值。此外，`encrypted:array`、`encrypted:collection`、`encrypted:object`、`AsEncryptedArrayObject` 以及 `AsEncryptedCollection` 轉換的運作方式與其未加密的對應項相同；然而，正如您所預期的，儲存在資料庫中的底層值是被加密的。

由於加密後的文字最終長度不可預測，且比純文字長，請確保相關的資料庫欄位為 `TEXT` 型別或更大。此外，由於值在資料庫中是被加密的，您將無法對加密的屬性值進行查詢或搜尋。


<a name="key-rotation"></a>
#### 金鑰輪換

您可能知道，Laravel 使用應用程式 `app` 設定檔中指定的 `key` 設定值來加密字串。通常，此值對應於 `APP_KEY` 環境變數的值。如果您需要輪換應用程式的加密金鑰，您可以 [優雅地進行輪換](/docs/{{version}}/encryption#gracefully-rotating-encryption-keys)。


<a name="query-time-casting"></a>
### 查詢時型別轉換

有時您可能需要在執行查詢時應用型別轉換，例如從資料表中選取原始值時。例如，請考慮以下查詢：

```php
use App\Models\Post;
use App\Models\User;

$users = User::select([
    'users.*',
    'last_posted_at' => Post::selectRaw('MAX(created_at)')
        ->whereColumn('user_id', 'users.id')
])->get();
```

此查詢結果中的 `last_posted_at` 屬性將是一個簡單的字串。如果我們能在執行查詢時對此屬性應用 `datetime` 轉換就太棒了。幸好，我們可以使用 `withCasts` 方法來實現：

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
## 自定義型別轉換

Laravel 提供了多種內建且實用的型別轉換；然而，您偶爾可能需要定義自己的型別轉換。要建立型別轉換，請執行 `make:cast` Artisan 指令。新的型別轉換類別將被放置在您的 `app/Casts` 目錄中：

```shell
php artisan make:cast AsJson
```

所有自定義型別轉換類別都實作了 `CastsAttributes` 介面。實作此介面的類別必須定義 `get` 與 `set` 方法。`get` 方法負責將來自資料庫的原始值轉換為型別轉換值，而 `set` 方法則應將型別轉換值轉換為可儲存在資料庫中的原始值。舉例來說，我們將以自定義型別轉換的方式重新實作內建的 `json` 型別轉換：

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

一旦您定義了自定義型別轉換，您就可以使用其類別名稱將其附加到模型屬性上：

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
### 數值物件型別轉換

您不限於將值轉換為原始型別，您也可以將值轉換為物件。定義將值轉換為物件的自定義型別轉換與轉換為原始型別非常相似；然而，如果您的數值物件包含多個資料庫欄位，`set` 方法必須回傳一個由鍵 / 值對組成的陣列，用來設定模型中可儲存的原始值。如果您的數值物件僅影響單個欄位，您只需回傳可儲存的值即可。

舉例來說，我們將定義一個自定義型別轉換類別，將多個模型值轉換為單個 `Address` 數值物件。我們假設 `Address` 數值物件具有兩個公開屬性：`lineOne` 與 `lineTwo`：

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

當轉換為數值物件時，對數值物件所做的任何更改在模型儲存之前，都會自動同步回模型：

```php
use App\Models\User;

$user = User::find(1);

$user->address->lineOne = 'Updated Address Value';

$user->save();
```

> [!NOTE]
> 如果您計畫將包含數值物件的 Eloquent 模型序列化為 JSON 或陣列，您應該在數值物件上實作 `Illuminate\Contracts\Support\Arrayable` 與 `JsonSerializable` 介面。


<a name="value-object-caching"></a>
#### 數值物件快取

當轉換為數值物件的屬性被解析時，它們會被 Eloquent 快取。因此，如果再次存取該屬性，將會回傳相同的物件實例。

如果您想停用自定義型別轉換類別的物件快取行為，您可以在自定義型別轉換類別中宣告一個公開的 `withoutObjectCaching` 屬性：

```php
class AsAddress implements CastsAttributes
{
    public bool $withoutObjectCaching = true;

    // ...
}
```


<a name="array-json-serialization"></a>
### 陣列 / JSON 序列化

當使用 `toArray` 與 `toJson` 方法將 Eloquent 模型轉換為陣列或 JSON 時，只要您的自定義型別轉換數值物件實作了 `Illuminate\Contracts\Support\Arrayable` 與 `JsonSerializable` 介面，它們通常也會被序列化。然而，當使用第三方函式庫提供的數值物件時，您可能無法將這些介面添加到該物件中。

因此，您可以指定由您的自定義型別轉換類別負責序列化數值物件。若要達成此目的，您的自定義型別轉換類別應實作 `Illuminate\Contracts\Database\Eloquent\SerializesCastableAttributes` 介面。此介面規定您的類別應包含一個 `serialize` 方法，該方法應回傳數值物件的序列化形式：

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
### 入向型別轉換

偶爾，您可能需要編寫一個僅在模型設定值時進行轉換，而在從模型讀取屬性時不執行任何操作的自定義型別轉換類別。

僅限入向的自定義型別轉換應實作 `CastsInboundAttributes` 介面，該介面僅要求定義 `set` 方法。可以透過在執行 `make:cast` Artisan 指令時加上 `--inbound` 選項，來產生僅限入向的型別轉換類別：

```shell
php artisan make:cast AsHash --inbound
```

僅限入向型別轉換的一個典型範例是「雜湊 (hashing)」轉換。例如，我們可以定義一個透過給定演算法對入向值進行雜湊的型別轉換：

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

在將自定義型別轉換附加到模型時，可以使用 `:` 字元將型別轉換參數與類別名稱分開，並使用逗號分隔多個參數。這些參數將被傳遞到型別轉換類別的建構子中：

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

如果您想要定義兩個給定的型別轉換值應如何進行比較，以確定它們是否已被變更，您的自定義型別轉換類別可以實作 `Illuminate\Contracts\Database\Eloquent\ComparesCastableAttributes` 介面。這讓您可以精細地控制 Eloquent 將哪些值視為已變更，並在模型更新時將其儲存至資料庫。

此介面規定您的類別應包含一個 `compare` 方法，如果給定的值被視為相等，則應回傳 `true`：

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
### 可轉換對象 (Castables)

您可能希望允許應用程式的數值物件定義自己的自定義型別轉換類別。您可以選擇將實作了 `Illuminate\Contracts\Database\Eloquent\Castable` 介面的數值物件類別綁定到模型，而非直接綁定自定義型別轉換類別：

```php
use App\ValueObjects\Address;

protected function casts(): array
{
    return [
        'address' => Address::class,
    ];
}
```

實作 `Castable` 介面的物件必須定義一個 `castUsing` 方法，該方法應回傳負責在 `Castable` 類別之間進行型別轉換的自定義轉換器 (caster) 類別名稱：

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

在使用 `Castable` 類別時，您仍然可以在 `casts` 方法定義中提供參數。這些參數將被傳遞給 `castUsing` 方法：

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
#### 可轉換對象與匿名型別轉換類別

透過將「可轉換對象 (castables)」與 PHP 的 [匿名類別 (anonymous classes)](https://www.php.net/manual/en/language.oop5.anonymous.php) 結合，您可以將數值物件及其型別轉換邏輯定義為單一的可轉換對象。若要實現此功能，請在數值物件的 `castUsing` 方法中回傳一個匿名類別。該匿名類別應實作 `CastsAttributes` 介面：

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