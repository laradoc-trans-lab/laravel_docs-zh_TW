# Laravel Pint

- [簡介](#introduction)
- [安裝](#installation)
- [執行 Pint](#running-pint)
- [設定 Pint](#configuring-pint)
    - [預設集](#presets)
    - [規則](#rules)
    - [排除檔案 / 資料夾](#excluding-files-or-folders)
- [持續整合](#continuous-integration)
    - [GitHub Actions](#running-tests-on-github-actions)

<a name="introduction"></a>
## 簡介

[Laravel Pint](https://github.com/laravel/pint) 是一款為極簡主義者打造、具備主觀風格的 PHP 程式碼風格修復工具。Pint 是基於 [PHP CS Fixer](https://github.com/FriendsOfPHP/PHP-CS-Fixer) 構建的，它能讓你輕鬆確保程式碼風格保持乾淨且一致。

Pint 會自動安裝在所有新的 Laravel 應用程式中，因此你可以立即開始使用。預設情況下，Pint 不需要任何設定，並會遵循 Laravel 的主觀程式碼編寫風格來修復程式碼中的風格問題。


<a name="installation"></a>
## 安裝

Pint 已包含在最近版本的 Laravel 框架中，因此通常不需要手動安裝。但是，對於較舊的應用程式，你可以透過 Composer 安裝 Laravel Pint：

```shell
composer require laravel/pint --dev
```


<a name="running-pint"></a>
## 執行 Pint

你可以透過執行專案中 `vendor/bin` 目錄下的 `pint` 執行檔，來命令 Pint 修復程式碼風格問題：

```shell
./vendor/bin/pint
```

如果你希望 Pint 以平行模式 (parallel mode，實驗性) 執行以提升效能，可以使用 `--parallel` 選項：

```shell
./vendor/bin/pint --parallel
```

平行模式還允許你透過 `--max-processes` 選項來指定要執行的最大進程數。如果未提供此選項，Pint 將使用你機器上所有可用的核心：

```shell
./vendor/bin/pint --parallel --max-processes=4
```

你也可以針對特定的檔案或目錄執行 Pint：

```shell
./vendor/bin/pint app/Models

./vendor/bin/pint app/Models/User.php
```

Pint 將顯示它更新的所有檔案的詳細列表。你可以在執行 Pint 時加入 `-v` 選項，來查看更多關於 Pint 變更的細節：

```shell
./vendor/bin/pint -v
```

如果你只想讓 Pint 檢查程式碼中的風格錯誤，而不實際更改檔案，可以使用 `--test` 選項。如果發現任何程式碼風格錯誤，Pint 將回傳一個非零的結束代碼：

```shell
./vendor/bin/pint --test
```

如果你只想讓 Pint 根據 Git 修改與指定分支不同的檔案，可以使用 `--diff=[branch]` 選項。這可以有效地用於 CI 環境 (如 GitHub Actions)，透過僅檢查新檔案或修改過的檔案來節省時間：

```shell
./vendor/bin/pint --diff=main
```

如果你只想讓 Pint 修改根據 Git 顯示有尚未提交變更的檔案，可以使用 `--dirty` 選項：

```shell
./vendor/bin/pint --dirty
```

如果你希望 Pint 修復任何有風格錯誤的檔案，但若有任何檔案被修復時也以非零的結束代碼結束，可以使用 `--repair` 選項：

```shell
./vendor/bin/pint --repair
```


<a name="configuring-pint"></a>
## 設定 Pint

如前所述，Pint 不需要任何設定。但是，如果你想自訂預設集、規則或檢查的資料夾，可以透過在專案根目錄中建立一個 `pint.json` 檔案來實現：

```json
{
    "preset": "laravel"
}
```

此外，如果你想使用特定目錄下的 `pint.json`，可以在執行 Pint 時提供 `--config` 選項：

```shell
./vendor/bin/pint --config vendor/my-company/coding-style/pint.json
```


<a name="presets"></a>
### 預設集

預設集 (Presets) 定義了一組可用於修復程式碼風格問題的規則。預設情況下，Pint 使用 `laravel` 預設集，它會遵循 Laravel 的主觀程式碼編寫風格來修復問題。不過，你可以透過為 Pint 提供 `--preset` 選項來指定不同的預設集：

```shell
./vendor/bin/pint --preset psr12
```

如果你願意，也可以在專案的 `pint.json` 檔案中設定預設集：

```json
{
    "preset": "psr12"
}
```

Pint 目前支援的預設集有：`laravel`、`per`、`psr12`、`symfony` 與 `empty`。


<a name="rules"></a>
### 規則

規則 (Rules) 是 Pint 用來修復程式碼風格問題的風格指南。如上所述，預設集是預先定義好的規則群組，對於大多數 PHP 專案來說應該都是完美的，因此你通常不需要擔心它們包含的個別規則。

但是，如果你願意，可以在 `pint.json` 檔案中啟用或停用特定規則，或者使用 `empty` 預設集並從頭開始定義規則：

```json
{
    "preset": "laravel",
    "rules": {
        "simplified_null_return": true,
        "array_indentation": false,
        "new_with_parentheses": {
            "anonymous_class": true,
            "named_class": true
        }
    }
}
```

Pint 是建構在 [PHP CS Fixer](https://github.com/FriendsOfPHP/PHP-CS-Fixer) 之上的。因此，你可以使用它的任何規則來修復專案中的程式碼風格問題：[PHP CS Fixer 設定工具](https://mlocati.github.io/php-cs-fixer-configurator)。


<a name="custom-rules"></a>
#### 自訂規則

除了 PHP CS Fixer 規則外，Pint 還提供了以 `Pint/` 為前綴的自訂規則。這些規則預設不啟用，但你可以在 `pint.json` 檔案中啟用它們。


<a name="phpdoc-type-annotations-only"></a>
##### `Pint/phpdoc_type_annotations_only`

此規則會從程式碼中移除所有註解與 docblock 說明文字，僅保留包含 `@` 註釋的行，例如 `@param`、`@return`、`@var`、`@phpstan-type` 等：

```php
/**
 * Get the posts for the user. [tl! remove]
 * [tl! remove]
 * @return HasMany<Post, $this>
 */
public function posts(): HasMany
```

不包含 `@` 註釋的單行註解與區塊註解將被完全移除。如果你想保留特定的註解，可以在其前面加上 `@note`、`@warning` 或 `@todo`：

```php
// @note This comment will be preserved.
```

要啟用此規則，請將其加入到你的 `pint.json` 檔案中：

```json
{
    "preset": "laravel",
    "rules": {
        "Pint/phpdoc_type_annotations_only": true
    }
}
```

> [!NOTE]
> 此規則會自動跳過 `config` 目錄中的檔案，因為設定檔通常依賴註解來進行說明。


<a name="excluding-files-or-folders"></a>
### 排除檔案 / 資料夾

預設情況下，Pint 將檢查專案中除 `vendor` 目錄外的所有 `.php` 檔案。如果你想排除更多資料夾，可以使用 `exclude` 設定選項：

```json
{
    "exclude": [
        "my-specific/folder"
    ]
}
```

如果你想排除所有包含特定名稱模式的檔案，可以使用 `notName` 設定選項：

```json
{
    "notName": [
        "*-my-file.php"
    ]
}
```

如果你想透過提供檔案的確切路徑來排除檔案，可以使用 `notPath` 設定選項：

```json
{
    "notPath": [
        "path/to/excluded-file.php"
    ]
}
```


<a name="continuous-integration"></a>
## 持續整合


<a name="running-tests-on-github-actions"></a>
### GitHub Actions

要使用 Laravel Pint 自動檢查你的專案風格，你可以設定 [GitHub Actions](https://github.com/features/actions)，以便在每次將新程式碼推送到 GitHub 時執行 Pint。首先，請務必在 GitHub 的 **Settings > Actions > General > Workflow permissions** 中授予工作流 (workflows) 「Read and write permissions」。然後，建立一個包含以下內容的 `.github/workflows/lint.yml` 檔案：

```yaml
name: Fix Code Style

on: [push]

jobs:
  lint:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: true
      matrix:
        php: [8.4]

    steps:
      - name: Checkout code
        uses: actions/checkout@v5

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: ${{ matrix.php }}
          tools: pint

      - name: Run Pint
        run: pint

      - name: Commit linted files
        uses: stefanzweifel/git-auto-commit-action@v6
```