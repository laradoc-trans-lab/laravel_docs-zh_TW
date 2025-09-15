# 加密

- [簡介](#introduction)
- [設定](#configuration)
    - [優雅地輪換加密金鑰](#gracefully-rotating-encryption-keys)
- [使用 Encrypter](#using-the-encrypter)

<a name="introduction"></a>
## 簡介

Laravel 的加密服務提供了一個簡單、方便的介面，可透過 OpenSSL 使用 AES-256 和 AES-128 加密來加密和解密文字。所有 Laravel 的加密值都使用訊息驗證碼 (MAC) 進行簽署，以確保其底層值一旦加密後無法被修改或竄改。

<a name="configuration"></a>
## 設定

在使用 Laravel 的 Encrypter 之前，您必須在 `config/app.php` 設定檔中設定 `key` 設定選項。此設定值由 `APP_KEY` 環境變數驅動。您應該使用 `php artisan key:generate` 命令來產生此變數的值，因為 `key:generate` 命令將使用 PHP 的安全亂數產生器為您的應用程式建立一個加密安全金鑰。通常，`APP_KEY` 環境變數的值會在 [Laravel 安裝](/docs/{{version}}/installation)期間為您產生。

<a name="gracefully-rotating-encryption-keys"></a>
### 優雅地輪換加密金鑰

如果您更改應用程式的加密金鑰，所有已驗證的使用者 Session 都將從您的應用程式中登出。這是因為每個 Cookie，包括 Session Cookie，都由 Laravel 加密。此外，任何使用您先前加密金鑰加密的資料將無法再解密。

為了解決這個問題，Laravel 允許您在應用程式的 `APP_PREVIOUS_KEYS` 環境變數中列出您先前的加密金鑰。此變數可以包含一個以逗號分隔的所有先前加密金鑰列表：

```ini
APP_KEY="base64:J63qRTDLub5NuZvP+kb8YIorGS6qFYHKVo6u7179stY="
APP_PREVIOUS_KEYS="base64:2nLsGFGzyoae2ax3EF2Lyq/hH6QghBGLIq5uL+Gp8/w="
```

當您設定此環境變數時，Laravel 在加密值時將始終使用「目前」的加密金鑰。然而，在解密值時，Laravel 將首先嘗試目前的金鑰，如果使用目前的金鑰解密失敗，Laravel 將嘗試所有先前的金鑰，直到其中一個金鑰能夠解密該值。

這種優雅解密的方法允許使用者即使在您的加密金鑰輪換後也能不間斷地繼續使用您的應用程式。

<a name="using-the-encrypter"></a>
## 使用 Encrypter

<a name="encrypting-a-value"></a>
#### 加密值

您可以使用 `Crypt` Facade 提供的 `encryptString` 方法來加密值。所有加密值都使用 OpenSSL 和 AES-256-CBC 加密演算法進行加密。此外，所有加密值都使用訊息驗證碼 (MAC) 進行簽署。整合的訊息驗證碼將防止惡意使用者竄改的任何值被解密：

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

<a name="decrypting-a-value"></a>
#### 解密值

您可以使用 `Crypt` Facade 提供的 `decryptString` 方法來解密值。如果該值無法正確解密，例如訊息驗證碼無效時，將會拋出 `Illuminate\Contracts\Encryption\DecryptException`：

    use Illuminate\Contracts\Encryption\DecryptException;
    use Illuminate\Support\Facades\Crypt;

    try {
        $decrypted = Crypt::decryptString($encryptedValue);
    } catch (DecryptException $e) {
        // ...
    }

