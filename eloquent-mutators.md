# Eloquent：修改器 & 型別轉換

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
    - [值物件 (Value Object) 型別轉換](#value-object-casting)
    - [陣列 / JSON 序列化](#array-json-serialization)
    - [入站型別轉換](#inbound-casting)
    - [轉換參數](#cast-parameters)
    - [比較轉換值](#comparing-cast-values)
    - [可轉換類別 (Castables)](#castables)

<a name="introduction"></a>
## 簡介

存取器 (Accessor)、修改器 (Mutator) 與屬性型別轉換 (Attribute Casting) 讓您在取得或設定模型實例上的 Eloquent 屬性值時對其進行轉換。例如，您可能希望使用 [Laravel 加密器](/docs/{{version}}/encryption) 在資料儲存在資料庫時對其進行加密，然後在透過 Eloquent 模型存取該屬性時自動將其解密。或者，您可能希望在透過 Eloquent 模型存取儲存在資料庫中的 JSON 字串時，將其轉換為陣列。


<a name="accessors-and-mutators"></a>
## 存取器與修改器


<a name="defining-an-accessor"></a>
### 定義存取器

存取器在存取時會轉換 Eloquent 屬性值。若要定義存取器，請在模型中建立一個受保護 (protected) 的方法來代表可存取的屬性。該方法名稱應對應於底層真實模型屬性或資料庫欄位的「小駝峰式 (camel case)」命名。

在此範例中，我們將為 `first_name` 屬性定義一個存取器。當嘗試取得 `first_name` 屬性的值時，Eloquent 將自動呼叫該存取器。所有的屬性存取器與修改器方法都必須宣告傳回型別提示為 `Illuminate\Database\Eloquent\Casts\Attribute`：

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

所有存取器方法都會傳回一個 `Attribute` 實例，該實例定義了屬性將如何被存取，以及（可選的）如何被修改。在此範例中，我們僅定義了屬性的存取方式。為此，我們向 `Attribute` 類別建構子提供了 `get` 引數。

如您所見，資料欄位的原始值會傳遞給存取器，讓您可以操作並傳回該值。若要存取存取器的值，您只需存取模型實例上的 `first_name` 屬性即可：

```php
use App\Models\User;

$user = User::find(1);

$firstName = $user->first_name;
```

> [!NOTE]
> 如果您希望將這些計算後的值新增到模型的陣列或 JSON 表現形式中，[您需要將它們附加 (append)](/docs/{{version}}/eloquent-serialization#appending-values-to-json) 上去。


<a name="building-value-objects-from-multiple-attributes"></a>
#### 從多個屬性建立值物件

有時您的存取器可能需要將多個模型屬性轉換為單一的「值物件 (value object)」。為此，您的 `get` 閉包 (closure) 可以接受第二個引數 `$attributes`，該引數會自動提供給閉包，並包含一個模型所有目前屬性的陣列：

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

從存取器傳回值物件時，在模型儲存之前，對該值物件所做的任何變更都將自動同步回模型中。這是可能的，因為 Eloquent 會保留存取器傳回的實例，以便在每次呼叫存取器時都能傳回相同的實例：

```php
use App\Models\User;

$user = User::find(1);

$user->address->lineOne = 'Updated Address Line 1 Value';
$user->address->lineTwo = 'Updated Address Line 2 Value';

$user->save();
```

然而，有時您可能希望為字串和布林值等純量值 (primitive values) 啟用快取，特別是在它們的運算開銷較大時。若要實現此目的，您可以在定義存取器時呼叫 `shouldCache` 方法：

```php
protected function hash(): Attribute
{
    return Attribute::make(
        get: fn (string $value) => bcrypt(gzuncompress($value)),
    )->shouldCache();
}
```

如果您想停用屬性的物件快取行為，可以在定義屬性時呼叫 `withoutObjectCaching` 方法：

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

修改器在設定時會轉換 Eloquent 屬性值。若要定義修改器，您可以在定義屬性時提供 `set` 引數。讓我們為 `first_name` 屬性定義一個修改器。當我們嘗試在模型上設定 `first_name` 屬性的值時，系統會自動呼叫此修改器：

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

修改器閉包將接收正在為該屬性設定的值，讓您可以操作該值並傳回操作後的值。若要使用我們的修改器，我們只需在 Eloquent 模型上設定 `first_name` 屬性即可：

```php
use App\Models\User;

$user = User::find(1);

$user->first_name = 'Sally';
```

在此範例中，將以值 `Sally` 呼叫 `set` 回呼函式。接著，修改器將對該名稱套用 `strtolower` 函式，並將得到的結果值設定在模型內部的 `$attributes` 陣列中。


<a name="mutating-multiple-attributes"></a>
#### 修改多個屬性

有時您的修改器可能需要在底層模型上設定多個屬性。為此，您可以從 `set` 閉包中傳回一個陣列。陣列中的每個鍵 (key) 應與模型相關聯的底層屬性或資料庫欄位相互對應：

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

屬性型別轉換提供了與存取器和修改器類似的功能，且不需要在 Model 中定義任何額外的方法。相反地，您的 Model 的 `casts` 方法提供了一種將屬性轉換為常用資料型別的便捷方式。

`casts` 方法應回傳一個陣列，其鍵 (Key) 是要轉換的屬性名稱，而值 (Value) 是您希望該欄位轉換成的型別。支援的轉換型別有：

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

為了示範屬性型別轉換，讓我們將儲存在資料庫中為整數 (`0` 或 `1`) 的 `is_admin` 屬性轉換為布林值：

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

定義轉換後，當您存取 `is_admin` 屬性時，它將始終被轉換為布林值，即使底層值在資料庫中是以整數儲存的：

```php
$user = App\Models\User::find(1);

if ($user->is_admin) {
    // ...
}
```

如果您需要在執行期間新增臨時的轉換，可以使用 `mergeCasts` 方法。這些轉換定義將被添加到 Model 上已定義的任何轉換中：

```php
$user->mergeCasts([
    'is_admin' => 'integer',
    'options' => 'object',
]);
```

> [!WARNING]
> 值為 `null` 的屬性將不會被轉換。此外，您不應該定義與關聯名稱相同的轉換（或屬性），也不應該為 Model 的主鍵指派轉換。

<a name="stringable-casting"></a>
#### Stringable 型別轉換

您可以使用 `Illuminate\Database\Eloquent\Casts\AsStringable` 轉換類別將 Model 屬性轉換為 [流暢的 (Fluent) Illuminate\Support\Stringable 物件](/docs/{{version}}/strings#fluent-strings-method-list)：

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

`array` 轉換在處理以序列化 JSON 儲存的欄位時特別有用。例如，如果你的資料庫有一個包含序列化 JSON 的 `JSON` 或 `TEXT` 欄位類型，在你的 Eloquent 模型中為該屬性加上 `array` 轉換，當你存取它時，它將會自動將該屬性反序列化為 PHP 陣列：

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

一旦定義了轉換，你就可以存取 `options` 屬性，它會自動從 JSON 反序列化為 PHP 陣列。當你設定 `options` 屬性的值時，給定的陣列會自動被序列化回 JSON 以供儲存：

```php
use App\Models\User;

$user = User::find(1);

$options = $user->options;

$options['key'] = 'value';

$user->options = $options;

$user->save();
```

若要以更簡潔的語法更新 JSON 屬性的單個欄位，你可以[將該屬性設為可大量指派](/docs/{{version}}/eloquent#mass-assignment-json-columns)，並在呼叫 `update` 方法時使用 `->` 運算子：

```php
$user = User::find(1);

$user->update(['options->key' => 'value']);
```


<a name="json-and-unicode"></a>
#### JSON 與 Unicode

如果你想將陣列屬性儲存為包含未轉義 (Unescaped) Unicode 字元的 JSON，可以使用 `json:unicode` 轉換：

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

雖然標準的 `array` 轉換對於許多應用程式來說已經足夠，但它確實有一些缺點。由於 `array` 轉換回傳的是原始型別，因此無法直接修改陣列的偏移量 (Offset)。例如，以下程式碼會觸發 PHP 錯誤：

```php
$user = User::find(1);

$user->options['key'] = $value;
```

為了達成這個目的，Laravel 提供了 `AsArrayObject` 轉換，它會將你的 JSON 屬性轉換為 [ArrayObject](https://www.php.net/manual/en/class.arrayobject.php) 類別。此功能是使用 Laravel 的[自定義型別轉換](#custom-casts)實作的，這讓 Laravel 能夠智慧地快取並轉換修改後的物件，以便在不觸發 PHP 錯誤的情況下修改個別偏移量。要使用 `AsArrayObject` 轉換，只需將其指派給屬性即可：

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

同樣地，Laravel 提供了一個 `AsCollection` 轉換，它會將你的 JSON 屬性轉換為 Laravel [Collection](/docs/{{version}}/collections) 實例：

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

如果你希望 `AsCollection` 轉換實例化自定義集合類別，而不是 Laravel 的基礎集合類別，你可以提供集合類別名稱作為轉換參數：

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

`of` 方法可用於指示應透過集合的 [mapInto 方法](/docs/{{version}}/collections#method-mapinto) 將集合項目映射到給定的類別中：

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

當將集合映射到物件時，該物件應實作 `Illuminate\Contracts\Support\Arrayable` 和 `JsonSerializable` 介面，以定義其實例應如何序列化為 JSON 並儲存到資料庫中：

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

如果你的 Eloquent 模型除了模型的自動遞增 ID 欄位之外，還有一個 [二進位類型](/docs/{{version}}/migrations#column-method-binary) 的 `uuid` 或 `ulid` 欄位，你可以使用 `AsBinary` 轉換來自動在其二進位表示形式之間進行轉換：

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

一旦在模型上定義了轉換，你就可以將 UUID / ULID 屬性值設定為物件實例或字串。Eloquent 會自動將該值轉換為其二進位表示形式。擷取屬性值時，你將始終收到純文字字串值：

```php
use Illuminate\Support\Str;

$user->uuid = Str::uuid();

return $user->uuid;

// "6e8cdeed-2f32-40bd-b109-1e4405be2140"
```

<a name="date-casting"></a>
### 日期型別轉換

預設情況下，Eloquent 會將 `created_at` 與 `updated_at` 欄位轉換為 [Carbon](https://github.com/briannesbitt/Carbon) 的實例，它擴充了 PHP 的 `DateTime` 類別並提供各種有用的方法。您可以在 Model 的 `casts` 方法中定義額外的日期轉換，藉此轉換其他的日期屬性。通常，日期應該使用 `datetime` 或 `immutable_datetime` 轉換類型。

當定義 `date` 或 `datetime` 轉換時，您也可以指定日期的格式。當 [Model 被序列化為陣列或 JSON](/docs/{{version}}/eloquent-serialization) 時，將會使用此格式：

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

當欄位被轉換為日期時，您可以將相對應的 Model 屬性值設定為 UNIX 時間戳記、日期字串 (`Y-m-d`)、日期時間字串，或是 `DateTime` / `Carbon` 實例。該日期的值將會被正確地轉換並儲存到您的資料庫中。

您可以透過在 Model 中定義 `serializeDate` 方法來自定義所有 Model 日期的預設序列化格式。此方法不會影響日期在資料庫中儲存的格式：

```php
/**
 * Prepare a date for array / JSON serialization.
 */
protected function serializeDate(DateTimeInterface $date): string
{
    return $date->format('Y-m-d');
}
```

若要指定實際在資料庫中儲存 Model 日期時應使用的格式，您應該在 Model 中定義 `$dateFormat` 屬性：

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

預設情況下，`date` 與 `datetime` 轉換會將日期序列化為 UTC ISO-8601 日期字串 (`YYYY-MM-DDTHH:MM:SS.uuuuuuZ`)，不論應用程式的 `timezone` 設定選項中指定了哪個時區。強烈建議您始終使用此序列化格式，並將應用程式的日期以 UTC 時區儲存，即不要將應用程式的 `timezone` 設定選項從預設值 `UTC` 更改。在整個應用程式中一致地使用 UTC 時區，將提供與其他使用 PHP 和 JavaScript 編寫的日期處理函式庫的最大互操作性。

如果對 `date` 或 `datetime` 轉換套用了自定義格式，例如 `datetime:Y-m-d H:i:s`，則在日期序列化期間將使用 Carbon 實例的內部時區。通常，這會是應用程式 `timezone` 設定選項中指定的時區。然而，重要的是要注意，像 `created_at` 與 `updated_at` 這樣的 `timestamp` 欄位不受此行為影響，並且始終以 UTC 格式化，而不論應用程式的時區設定為何。


<a name="enum-casting"></a>
### 列舉 (Enum) 型別轉換

Eloquent 也允許您將屬性值轉換為 PHP [列舉 (Enums)](https://www.php.net/manual/en/language.enumerations.backed.php)。要實現這一點，您可以在 Model 的 `casts` 方法中指定您想要轉換的屬性與列舉：

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

一旦在 Model 上定義了轉換，當您與該屬性互動時，指定的屬性將自動在列舉之間來回轉換：

```php
if ($server->status == ServerStatus::Provisioned) {
    $server->status = ServerStatus::Ready;

    $server->save();
}
```


<a name="casting-arrays-of-enums"></a>
#### 轉換列舉陣列

有時您可能需要 Model 在單一欄位中儲存列舉值的陣列。要實現這一點，您可以使用 Laravel 提供的 `AsEnumArrayObject` 或 `AsEnumCollection` 轉換：

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

`encrypted` 轉換將使用 Laravel 內建的[加密](/docs/{{version}}/encryption)功能來加密 Model 的屬性值。此外，`encrypted:array`、`encrypted:collection`、`encrypted:object`、`AsEncryptedArrayObject` 與 `AsEncryptedCollection` 轉換的作用與它們未加密的對應項相似；然而，如您所料，當儲存在資料庫中時，底層的值是被加密的。

由於加密文本的最終長度是不可預測的，且比其純文本對應項更長，請確保相關的資料庫欄位是 `TEXT` 類型或更大。此外，由於值在資料庫中是加密的，您將無法查詢或搜尋加密的屬性值。


<a name="key-rotation"></a>
#### 密鑰輪替

如您所知，Laravel 使用應用程式 `app` 設定檔中指定的 `key` 設定值來加密字串。通常，此值對應於 `APP_KEY` 環境變數的值。如果您需要輪替應用程式的加密金鑰，您可以[優雅地執行此操作](/docs/{{version}}/encryption#gracefully-rotating-encryption-keys)。


<a name="query-time-casting"></a>
### 查詢時型別轉換

有時您可能需要在執行查詢時套用轉換，例如從資料表中選取原始值時。例如，考慮以下查詢：

```php
use App\Models\Post;
use App\Models\User;

$users = User::select([
    'users.*',
    'last_posted_at' => Post::selectRaw('MAX(created_at)')
        ->whereColumn('user_id', 'users.id')
])->get();
```

此查詢結果中的 `last_posted_at` 屬性將是一個簡單的字串。如果我們能在執行查詢時對此屬性套用 `datetime` 轉換，那就太好了。幸運的是，我們可以使用 `withCasts` 方法來實現：

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

Laravel 擁有多種內建且實用的型別轉換類型；然而，有時您可能需要定義自己的型別轉換類型。要建立型別轉換，請執行 `make:cast` Artisan 指令。新的型別轉換類別將放置在您的 `app/Casts` 目錄中：

```shell
php artisan make:cast AsJson
```

所有自定義型別轉換類別都實作了 `CastsAttributes` 介面。實作此介面的類別必須定義 `get` 和 `set` 方法。`get` 方法負責將資料庫中的原始值轉換為轉換後的值，而 `set` 方法應將轉換後的值轉換為可以儲存在資料庫中的原始值。舉例來說，我們將自定義型別轉換重新實作內建的 `json` 型別轉換：

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

定義好自定義型別轉換類型後，您可以使用類別名稱將其附加到 Model 屬性：

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
### 值物件 (Value Object) 型別轉換

您不限於將值轉換為原始類型。您也可以將值轉換為物件。定義將值轉換為物件的自定義型別轉換與轉換為原始類型非常相似；然而，如果您的值物件涵蓋多個資料庫欄位，則 `set` 方法必須返回一個鍵值對 (Key / Value Pairs) 陣列，這些鍵值對將用於在 Model 上設定原始且可儲存的值。如果您的值物件僅影響單個欄位，則只需返回可儲存的值。

作為範例，我們將定義一個自定義型別轉換類別，將多個 Model 值轉換為單個 `Address` 值物件。我們假設 `Address` 值物件具有兩個公開屬性：`lineOne` 和 `lineTwo`：

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

當轉換為值物件時，對值物件所做的任何更改都將在 Model 儲存之前自動同步回 Model：

```php
use App\Models\User;

$user = User::find(1);

$user->address->lineOne = 'Updated Address Value';

$user->save();
```

> [!NOTE]
> 如果您計畫將包含值物件的 Eloquent Model 序列化為 JSON 或陣列，您應該在值物件上實作 `Illuminate\Contracts\Support\Arrayable` 和 `JsonSerializable` 介面。


<a name="value-object-caching"></a>
#### 值物件快取 (Value Object Caching)

當轉換為值物件的屬性被解析時，它們會由 Eloquent 快取。因此，如果再次存取該屬性，將返回相同的物件實例。

如果您想停用自定義型別轉換類別的物件快取行為，可以在自定義型別轉換類別中宣告一個公開的 `withoutObjectCaching` 屬性：

```php
class AsAddress implements CastsAttributes
{
    public bool $withoutObjectCaching = true;

    // ...
}
```


<a name="array-json-serialization"></a>
### 陣列 / JSON 序列化

當使用 `toArray` 和 `toJson` 方法將 Eloquent Model 轉換為陣列或 JSON 時，只要您的自定義型別轉換值物件實作了 `Illuminate\Contracts\Support\Arrayable` 和 `JsonSerializable` 介面，它們通常也會被序列化。然而，當使用第三方函式庫提供的值物件時，您可能無法將這些介面新增到物件中。

因此，您可以指定由自定義型別轉換類別負責序列化該值物件。為此，您的自定義型別轉換類別應實作 `Illuminate\Contracts\Database\Eloquent\SerializesCastableAttributes` 介面。此介面規定您的類別應包含一個 `serialize` 方法，該方法應返回值物件的序列化形式：

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

有時，您可能需要編寫一個僅轉換設定到 Model 的值，且在從 Model 檢索屬性時不執行任何操作的自定義型別轉換類別。

僅限入站的自定義型別轉換應實作 `CastsInboundAttributes` 介面，這僅需要定義 `set` 方法。可以使用帶有 `--inbound` 選項的 `make:cast` Artisan 指令來生成僅限入站的型別轉換類別：

```shell
php artisan make:cast AsHash --inbound
```

僅限入站型別轉換的一個經典範例是「雜湊 (Hashing)」型別轉換。例如，我們可以定義一個型別轉換，透過指定的演算法對入站值進行雜湊處理：

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
### 轉換參數

將自定義型別轉換附加到 Model 時，可以透過使用 `:` 字元將參數與類別名稱分隔開來，並使用逗號分隔多個參數。這些參數將傳遞給型別轉換類別的建構子：

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
### 比較轉換值

如果你想定義兩個給定的轉換值應如何比較以確定它們是否已更改，你的自定義型別轉換類別可以實作 `Illuminate\Contracts\Database\Eloquent\ComparesCastableAttributes` 介面。這讓你可以細粒度地控制 Eloquent 認為哪些值已被更改，並在更新模型時將其儲存到資料庫中。

此介面規定你的類別應包含一個 `compare` 方法，如果給定的值被視為相等，則該方法應回傳 `true`：

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
### 可轉換類別 (Castables)

你可能希望允許應用程式的值物件 (Value Object) 定義自己的自定義型別轉換類別。與其將自定義型別轉換類別附加到你的模型中，你也可以附加一個實作了 `Illuminate\Contracts\Database\Eloquent\Castable` 介面的值物件類別：

```php
use App\ValueObjects\Address;

protected function casts(): array
{
    return [
        'address' => Address::class,
    ];
}
```

實作 `Castable` 介面的物件必須定義一個 `castUsing` 方法，該方法回傳負責在 `Castable` 類別之間進行轉換的自定義轉換器類別名稱：

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

使用 `Castable` 類別時，你仍然可以在 `casts` 方法定義中提供參數。這些參數將被傳遞給 `castUsing` 方法：

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
#### 可轉換類別 (Castables) 與匿名轉換類別

透過將「可轉換類別 (Castables)」與 PHP 的 [匿名類別 (Anonymous Classes)](https://www.php.net/manual/en/language.oop5.anonymous.php) 相結合，你可以將值物件及其轉換邏輯定義為單個可轉換物件。為此，請從值物件的 `castUsing` 方法回傳一個匿名類別。該匿名類別應實作 `CastsAttributes` 介面：

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