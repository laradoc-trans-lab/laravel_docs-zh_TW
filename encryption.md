# 加密

- [簡介](#introduction)
- [設定](#configuration)
    - [優雅地輪換加密金鑰](#gracefully-rotating-encryption-keys)
- [使用加密器](#using-the-encrypter)

<a name="introduction"></a>
## 簡介

Laravel 的加密服務提供了一個簡單、方便的介面，用於透過 OpenSSL 使用 AES-256 和 AES-128 加密來加密和解密文字。所有 Laravel 的加密值都使用訊息驗證碼 (MAC) 進行簽名，因此其底層值在加密後無法被修改或篡改。

<a name="configuration"></a>
## 設定

在使用 Laravel 的加密器之前，你必須在 `config/app.php` 設定檔中設定 `key` 設定選項。此設定值由 `APP_KEY` 環境變數驅動。你應該使用 `php artisan key:generate` 命令來生成此變數的值，因為 `key:generate` 命令將使用 PHP 的安全隨機位元生成器為你的應用程式建立一個加密安全金鑰。通常，`APP_KEY` 環境變數的值會在 [Laravel 安裝](/docs/{{version}}/installation)期間為你生成。

<a name="gracefully-rotating-encryption-keys"></a>
### 優雅地輪換加密金鑰

如果你更改應用程式的加密金鑰，所有已驗證的使用者會話都將從你的應用程式中登出。這是因為每個 Cookie，包括會話 Cookie，都由 Laravel 加密。此外，任何使用你先前加密金鑰加密的資料將無法再解密。

為了緩解這個問題，Laravel 允許你在應用程式的 `APP_PREVIOUS_KEYS` 環境變數中列出你先前的加密金鑰。此變數可以包含一個以逗號分隔的所有先前加密金鑰的列表：

```ini
APP_KEY="base64:J63qRTDLub5NuZvP+kb8YIorGS6qFYHKVo6u7179stY="
APP_PREVIOUS_KEYS="base64:2nLsGFGzyoae2ax3EF2Lyq/hH6QghBGLIq5uL+Gp8/w="
```

當你設定此環境變數時，Laravel 在加密值時將始終使用「目前」的加密金鑰。然而，在解密值時，Laravel 將首先嘗試目前金鑰，如果使用目前金鑰解密失敗，Laravel 將嘗試所有先前的金鑰，直到其中一個金鑰能夠解密該值。

這種優雅解密的方法允許使用者即使在你的加密金鑰輪換後也能不間斷地繼續使用你的應用程式。

<a name="using-the-encrypter"></a>
## 使用加密器

<a name="encrypting-a-value"></a>
#### 加密值

你可以使用 `Crypt` Facade 提供的 `encryptString` 方法來加密值。所有加密值都使用 OpenSSL 和 AES-256-CBC 加密。此外，所有加密值都使用訊息驗證碼 (MAC) 簽名。整合的訊息驗證碼將防止惡意使用者篡改的任何值被解密：

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Crypt;

class DigitalOceanTokenController extends Controller
{
    /**
     * Store a DigitalOcean API token for the user.
     */
    public function store(Request $request): RedirectResponse
    {
        $request->user()->fill([
            'token' => Crypt::encryptString($request->token),
        ])->save();

        return redirect('/secrets');
    }
}
```

<a name="decrypting-a-value"></a>
#### 解密值

你可以使用 `Crypt` Facade 提供的 `decryptString` 方法來解密值。如果值無法正確解密，例如訊息驗證碼無效時，將會拋出 `Illuminate\Contracts\Encryption\DecryptException`：

```php
use Illuminate\Contracts\Encryption\DecryptException;
use Illuminate\Support\Facades\Crypt;

try {
    $decrypted = Crypt::decryptString($encryptedValue);
} catch (DecryptException $e) {
    // ...
}
```

