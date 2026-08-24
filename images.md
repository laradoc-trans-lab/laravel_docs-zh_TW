# 圖片處理

- [簡介](#introduction)
- [安裝](#installation)
    - [設定](#configuration)
- [讀取圖片](#reading-images)
    - [上傳檔案](#uploaded-files)
    - [儲存檔案](#storage-files)
    - [其他來源](#other-sources)
- [處理圖片](#manipulating-images)
    - [調整圖片尺寸](#resizing-images)
    - [其他轉換](#other-transformations)
- [編碼圖片](#encoding-images)
- [儲存圖片](#storing-images)
- [檢查圖片](#inspecting-images)
- [圖片驅動程式](#image-drivers)
    - [自訂圖片驅動程式](#custom-image-drivers)
    - [自訂轉換](#custom-transformations)

<a name="introduction"></a>
## 簡介

Laravel 提供了一套順暢的圖片處理 API，讓你可以使用整個框架中常見的直覺慣例來調整尺寸、裁切、編碼與儲存圖片。Laravel 的圖片功能由 [Intervention Image](https://image.intervention.io/) 提供支援，並支援 GD 與 Imagick PHP 擴充功能。

當處理上傳的檔案、儲存在 Laravel [檔案系統磁碟](/docs/{{version}}/filesystem)上的檔案、本機檔案、遠端 URL，或是原始圖片位元組時，圖片 API 都非常實用：

```php
use Illuminate\Support\Facades\Image;

$path = Image::fromStorage('avatars/photo.jpg', 'public')
    ->cover(400, 400)
    ->toWebp()
    ->quality(80)
    ->storePublicly('avatars', 'public');
```

> [!WARNING]
> 圖片處理可能會消耗大量的 CPU 與記憶體。建議將大型圖片處理的工作負載交由[佇列任務](/docs/{{version}}/queues)執行，而非在接收上傳的 HTTP 請求期間直接處理。


<a name="installation"></a>
## 安裝

在使用 Laravel 的圖片處理功能之前，請先透過 Composer 安裝 Intervention Image 套件：

```shell
composer require intervention/image:^4.0
```

你也應確保你的 PHP 環境已安裝 GD 或 Imagick 擴充功能，具體取決於你的應用程式將使用哪種驅動程式。


<a name="configuration"></a>
### 設定

Laravel 的圖片設定檔位於 `config/images.php`。如果你的應用程式還沒有 `images` 設定檔，你可以使用 `config:publish` Artisan 命令來發布它：

```shell
php artisan config:publish images
```

圖片設定檔允許你指定應用程式預設的圖片驅動程式。你也可以使用 `IMAGE_DRIVER` 環境變數來指定預設驅動程式。支援的驅動程式有 `gd` 與 `imagick`：

```ini
IMAGE_DRIVER=imagick
```


<a name="reading-images"></a>
## 讀取圖片

`Image` Facade 提供了數個從常見來源讀取圖片的方法。圖片內容採用延遲載入（Lazy Loading），因此在圖片被處理或要求讀取其位元組之前，通常不會讀取來源。


<a name="uploaded-files"></a>
### 上傳檔案

你可以使用 `image` 方法從傳入的請求中取得上傳的圖片。此方法會為上傳的檔案傳回一個 `Illuminate\Image\Image` 執行個體；若檔案不存在則傳回 `null`：

```php
use Illuminate\Http\Request;

Route::post('/avatar', function (Request $request) {
    $request->validate(['avatar' => ['required', 'image']]);

    $path = $request->image('avatar')
        ->cover(400, 400)
        ->toWebp()
        ->storePublicly('avatars', 'public');

    // ...
});
```

另外，你也可以使用 `fromUpload` 方法，從 `Illuminate\Http\UploadedFile` 執行個體建立圖片執行個體：

```php
use Illuminate\Support\Facades\Image;

$image = Image::fromUpload($request->file('avatar'));
```

當從上傳檔案建立圖片時，你可以使用 `file` 方法取得底層的上傳檔案物件：

```php
$file = $image->file();
```


<a name="storage-files"></a>
### 儲存檔案

你可以使用 `fromStorage` 方法，從儲存在應用程式[檔案系統磁碟](/docs/{{version}}/filesystem)上的檔案建立圖片執行個體。第一個引數是檔案路徑，第二個引數則是磁碟名稱：

```php
use Illuminate\Support\Facades\Image;

$image = Image::fromStorage('avatars/photo.jpg', disk: 'public');
```

你也可以直接透過檔案系統磁碟的執行個體，使用 `image` 方法來建立圖片執行個體：

```php
use Illuminate\Support\Facades\Storage;

$image = Storage::disk('public')->image('avatars/photo.jpg');
```


<a name="other-sources"></a>
### 其他來源

`Image` Facade 還包含了從原始位元組、本機檔案路徑、遠端 URL 以及 Base64 編碼字串建立圖片執行個體的方法：

```php
use Illuminate\Support\Facades\Image;

$image = Image::fromBytes($contents);
$image = Image::fromBase64($base64);
$image = Image::fromPath(storage_path('app/avatars/photo.jpg'));
$image = Image::fromUrl('https://example.com/photo.jpg');
```


<a name="manipulating-images"></a>
## 處理圖片

圖片執行個體是不可變的（Immutable）。每個處理方法都會傳回一個新的圖片執行個體，並將轉換操作附加到其處理管道（Pipeline）中，讓你能夠順暢地鏈結呼叫多個方法：

```php
$image = $request->image('avatar')
    ->orient()
    ->cover(400, 400)
    ->sharpen(10);
```

轉換操作會按照加入圖片管道的順序依序處理，且圖片只會在最後編碼一次。


<a name="resizing-images"></a>
### 調整圖片尺寸

`resize` 方法可將圖片調整為指定尺寸。你可以同時提供寬度與高度，或是使用具名引數只提供單一維度：

```php
$image = $image->resize(800, 600);
$image = $image->resize(width: 800);
$image = $image->resize(height: 600);
```

`scale` 方法會依比例按等比例縮小圖片，使其能放入指定尺寸內。此方法絕不會放大圖片尺寸：

```php
$image = $image->scale(800, 600);
$image = $image->scale(width: 800);
$image = $image->scale(height: 600);
```

`cover` 方法會調整並裁切圖片，使其完全涵蓋指定的尺寸：

```php
$image = $image->cover(400, 400);
```

`contain` 方法會調整圖片大小以符合指定尺寸，同時完整保留整張圖片。如有必要，空白處將使用可選的背景顏色填滿：

```php
$image = $image->contain(400, 400);
$image = $image->contain(400, 400, '#ffffff');
$image = $image->contain(400, 400, 'dominant');
```

你可以指定 `dominant` 作為背景顏色，以使用圖片的主導色彩來填滿空白處。

你可以使用 `crop` 方法裁切圖片。前兩個引數是期望的寬度與高度，第三和第四個可選引數則指定裁切點的 `x` 與 `y` 座標：

```php
$image = $image->crop(300, 200);
$image = $image->crop(300, 200, x: 50, y: 25);
```


<a name="other-transformations"></a>
### 其他轉換

Laravel 還提供了多種額外圖片轉換方法：

```php
$image = $image->orient();
$image = $image->rotate(90);
$image = $image->rotate(90, '#ffffff');
$image = $image->rotate(90, 'dominant');
$image = $image->blur(5);
$image = $image->grayscale();
$image = $image->sharpen(10);
$image = $image->flipVertically();
$image = $image->flipHorizontally();
```

`orient` 方法會根據圖片的 EXIF 方向資料旋轉圖片。`rotate` 方法會將圖片依順時針方向旋轉指定角度，並可接收可選的背景顏色。`blur` 與 `sharpen` 方法接受介於 `0` 到 `100` 之間的值。


<a name="conditional-transformations"></a>
#### 條件式轉換

圖片執行個體支援 Laravel 的 `Conditionable` Trait，允許你使用 `when` 和 `unless` 方法根據條件套用轉換：

```php
$image = $request->image('avatar')
    ->when($request->boolean('crop'), fn ($image) => $image->cover(400, 400))
    ->unless($request->boolean('preserve_format'), fn ($image) => $image->toWebp());
```

<a name="encoding-images"></a>
## 編碼圖片

預設情況下，處理後的圖片會使用其原始格式進行編碼。然而，您可以在取得或儲存圖片之前，將圖片轉換為其他支援的格式：

```php
$image = $image->toWebp();
$image = $image->toJpg();
$image = $image->toJpeg();
$image = $image->toPng();
$image = $image->toGif();
$image = $image->toAvif();
$image = $image->toBmp();
```

您可以使用 `quality` 方法來設定輸出品質。品質數值會被限制在 `1` 到 `100` 之間：

```php
$image = $image->toWebp()->quality(80);
```

`optimize` 方法是一個方便的快捷方式，用於將圖片轉換為指定的格式並設定其品質。預設情況下，圖片會被最佳化為品質為 `70` 的 WebP 圖片：

```php
$image = $image->optimize();

$image = $image->optimize(format: 'jpg', quality: 85);
```

您可以用位元組字串、Base64 編碼字串或 Data URI 的形式取得處理後的圖片內容：

```php
$bytes = $image->toBytes();
$base64 = $image->toBase64();
$dataUri = $image->toDataUri();
```

圖片實例也可以轉換為字串以取得 Data URI：

```php
$dataUri = (string) $image;
```


<a name="storing-images"></a>
## 儲存圖片

`store` 方法會將處理後的圖片儲存在應用程式的某個檔案系統磁碟上。就像上傳檔案一樣，Laravel 會產生一個唯一的檔名並回傳儲存路徑。第二個引數可用於指定磁碟：

```php
$path = $request->image('avatar')
    ->cover(400, 400)
    ->store(path: 'avatars');

$path = $request->image('avatar')
    ->cover(400, 400)
    ->store(path: 'avatars', disk: 's3');
```

您可以使用 `storeAs` 方法來指定儲存的檔名：

```php
$path = $request->image('avatar')
    ->cover(400, 400)
    ->storeAs(path: 'avatars', name: 'avatar.jpg', disk: 'public');
```

`storePublicly` 與 `storePubliclyAs` 方法會以 `public` 可見性來儲存圖片：

```php
$path = $request->image('avatar')
    ->cover(400, 400)
    ->storePublicly(path: 'avatars', disk: 'public');

$path = $request->image('avatar')
    ->cover(400, 400)
    ->storePubliclyAs(path: 'avatars', name: 'avatar.webp', disk: 'public');
```

如果無法儲存圖片，這些儲存方法會回傳 `false`。


<a name="inspecting-images"></a>
## 檢查圖片

您可以使用以下方法取得圖片的 MIME 類型、副檔名、尺寸、寬度、高度以及主色調：

```php
$mimeType = $image->mimeType();
$extension = $image->extension();

[$width, $height] = $image->dimensions();
$width = $image->width();
$height = $image->height();

$dominantColor = $image->dominantColor();
```

這些方法都是針對處理後的圖片進行操作。例如，在呼叫 `cover(400, 400)` 之後呼叫 `width`，將會回傳 `400`。


<a name="image-drivers"></a>
## 圖片驅動程式


<a name="custom-image-drivers"></a>
### 自訂圖片驅動程式

Laravel 的圖片管理器（Image Manager）繼承自 Laravel 的基底 `Illuminate\Support\Manager` 類別。這意味著您可以使用圖片管理器與 `Image` Facade 上的 `extend` 方法來註冊自訂的圖片驅動程式。

自訂圖片驅動程式應該實作 `Illuminate\Contracts\Image\Driver` 介面。`process` 方法會接收原始圖片內容以及應套用到圖片上的有序 `Illuminate\Image\ImagePipeline`，並且應該回傳處理後的圖片位元組：

```php
<?php

namespace App\Images;

use Illuminate\Contracts\Image\Driver;
use Illuminate\Image\ImagePipeline;

class VipsDriver implements Driver
{
    /**
     * Process the given image contents with the specified pipeline.
     */
    public function process(string $contents, ImagePipeline $pipeline): string
    {
        // Apply the pipeline's transformations and output options...

        return $contents;
    }

    /**
     * Register a transformation handler.
     */
    public function transformUsing(string $transformation, callable $callback): static
    {
        // Store the handler so it may be applied while processing the pipeline...

        return $this;
    }
}
```

> [!NOTE]
> 若要更深入瞭解如何實作自訂圖片驅動程式，您可以參考框架內建的 `Illuminate\Image\Drivers\InterventionDriver` 類別。

實作好自訂驅動程式後，您可以使用 `Image` Facade 的 `extend` 方法來註冊它。通常，這應該在服務提供者(Service Providers)的 `boot` 方法中完成：

```php
use App\Images\VipsDriver;
use Illuminate\Contracts\Foundation\Application;
use Illuminate\Support\Facades\Image;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Image::extend('vips', function (Application $app) {
        return new VipsDriver;
    });
}
```

註冊驅動程式後，您可以透過 `using` 方法在特定圖片上使用該驅動程式：

```php
$image = $request->image('avatar')
    ->using('vips')
    ->cover(400, 400);
```

您也可以透過應用程式的 `config/images.php` 設定檔中的 `default` 選項，或是 `IMAGE_DRIVER` 環境變數，將自訂驅動程式設定為應用程式的預設圖片驅動程式：

```ini
IMAGE_DRIVER=vips
```


<a name="custom-transformations"></a>
### 自訂轉換

應用程式與套件可以透過建立實作 `Illuminate\Contracts\Image\Transformation` 契約(Contracts)的類別來定義自訂轉換。接著，可以使用 `transform` 方法將自訂轉換加入圖片處理管道（Pipeline）中：

```php
<?php

namespace App\Images\Transformations;

use Illuminate\Contracts\Image\Transformation;

class Pixelate implements Transformation
{
    public function __construct(
        public readonly int $size,
    ) {
        //
    }
}
```

接下來，使用 `Image` Facade 的 `transformUsing` 方法為該轉換與驅動程式註冊一個處理器（Handler）。通常，這應該在服務提供者(Service Providers)的 `boot` 方法中進行：

```php
use App\Images\Transformations\Pixelate;
use Illuminate\Support\Facades\Image;
use Intervention\Image\Interfaces\ImageInterface;

Image::transformUsing('gd', Pixelate::class, function (ImageInterface $image, Pixelate $transformation) {
    return $image->pixelate($transformation->size);
});
```

註冊轉換處理器後，您就可以將該轉換套用到圖片上：

```php
use App\Images\Transformations\Pixelate;

$image = $request->image('avatar')
    ->transform(new Pixelate(12))
    ->store('avatars');
```