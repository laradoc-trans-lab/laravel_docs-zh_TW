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

[Laravel Pint](https://github.com/laravel/pint) 是一個為極簡主義者設計的 PHP 程式碼風格修正工具。Pint 建構於 [PHP CS Fixer](https://github.com/FriendsOfPHP/PHP-CS-Fixer) 之上，讓您能輕鬆確保程式碼風格保持簡潔與一致。

Pint 會自動安裝在所有新的 Laravel 應用程式中，因此您可以立即開始使用。預設情況下，Pint 無需任何設定，並會遵循 Laravel 獨特的程式碼風格，修正您程式碼中的風格問題。

<a name="installation"></a>
## 安裝

Pint 已包含在 Laravel 框架的最新版本中，因此通常無需額外安裝。然而，對於較舊的應用程式，您可以透過 Composer 安裝 Laravel Pint：

```shell
composer require laravel/pint --dev
```

<a name="running-pint"></a>
## 執行 Pint

您可以透過呼叫專案 `vendor/bin` 目錄中可用的 `pint` 二進位檔，指示 Pint 修正程式碼風格問題：

```shell
./vendor/bin/pint
```

如果您希望 Pint 以平行模式（實驗性）執行以提升效能，可以使用 `--parallel` 選項：

```shell
./vendor/bin/pint --parallel
```

平行模式也允許您透過 `--max-processes` 選項指定要執行的最大處理程序數量。如果未提供此選項，Pint 將使用您機器上所有可用的核心：

```shell
./vendor/bin/pint --parallel --max-processes=4
```

您也可以在特定檔案或目錄上執行 Pint：

```shell
./vendor/bin/pint app/Models

./vendor/bin/pint app/Models/User.php
```

Pint 將顯示所有更新檔案的詳細列表。您可以在呼叫 Pint 時提供 `-v` 選項，以查看 Pint 變更的更多細節：

```shell
./vendor/bin/pint -v
```

如果您希望 Pint 僅檢查程式碼中的風格錯誤，而不實際修改檔案，可以使用 `--test` 選項。如果發現任何程式碼風格錯誤，Pint 將返回非零的結束代碼：

```shell
./vendor/bin/pint --test
```

如果您希望 Pint 僅修改根據 Git 與指定分支不同的檔案，可以使用 `--diff=[branch]` 選項。這可以在您的 CI 環境（如 GitHub Actions）中有效使用，透過僅檢查新增或修改的檔案來節省時間：

```shell
./vendor/bin/pint --diff=main
```

如果您希望 Pint 僅修改根據 Git 具有未提交變更的檔案，可以使用 `--dirty` 選項：

```shell
./vendor/bin/pint --dirty
```

如果您希望 Pint 修正任何具有程式碼風格錯誤的檔案，但如果修正了任何錯誤也以非零結束代碼退出，可以使用 `--repair` 選項：

```shell
./vendor/bin/pint --repair
```

<a name="configuring-pint"></a>
## 設定 Pint

如前所述，Pint 無需任何設定。但是，如果您希望自訂預設集、規則或檢查的資料夾，可以在專案的根目錄中建立一個 `pint.json` 檔案：

```json
{
    "preset": "laravel"
}
```

此外，如果您希望使用特定目錄中的 `pint.json`，可以在呼叫 Pint 時提供 `--config` 選項：

```shell
./vendor/bin/pint --config vendor/my-company/coding-style/pint.json
```

<a name="presets"></a>
### 預設集

預設集定義了一組可用於修正程式碼風格問題的規則。預設情況下，Pint 使用 `laravel` 預設集，它遵循 Laravel 獨特的程式碼風格來修正問題。但是，您可以透過向 Pint 提供 `--preset` 選項來指定不同的預設集：

```shell
./vendor/bin/pint --preset psr12
```

如果您願意，也可以在專案的 `pint.json` 檔案中設定預設集：

```json
{
    "preset": "psr12"
}
```

Pint 目前支援的預設集有：`laravel`、`per`、`psr12`、`symfony` 和 `empty`。

<a name="rules"></a>
### 規則

規則是 Pint 將用於修正程式碼風格問題的風格指南。如上所述，預設集是預定義的規則組，對於大多數 PHP 專案來說應該是完美的，因此您通常無需擔心它們包含的個別規則。

但是，如果您願意，可以在 `pint.json` 檔案中啟用或禁用特定規則，或者使用 `empty` 預設集並從頭開始定義規則：

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

Pint 建構於 [PHP CS Fixer](https://github.com/FriendsOfPHP/PHP-CS-Fixer) 之上。因此，您可以使用其任何規則來修正專案中的程式碼風格問題：[PHP CS Fixer Configurator](https://mlocati.github.io/php-cs-fixer-configurator)。

<a name="excluding-files-or-folders"></a>
### 排除檔案 / 資料夾

預設情況下，Pint 將檢查專案中除了 `vendor` 目錄之外的所有 `.php` 檔案。如果您希望排除更多資料夾，可以使用 `exclude` 設定選項：

```json
{
    "exclude": [
        "my-specific/folder"
    ]
}
```

如果您希望排除所有包含給定名稱模式的檔案，可以使用 `notName` 設定選項：

```json
{
    "notName": [
        "*-my-file.php"
    ]
}
```

如果您希望透過提供檔案的確切路徑來排除檔案，可以使用 `notPath` 設定選項：

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

為了使用 Laravel Pint 自動化專案的程式碼檢查，您可以設定 [GitHub Actions](https://github.com/features/actions)，以便在每次有新程式碼推送到 GitHub 時執行 Pint。首先，請務必在 GitHub 的 **Settings > Actions > General > Workflow permissions** 中，授予工作流程「讀寫權限」。然後，建立一個 `.github/workflows/lint.yml` 檔案，內容如下：

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
        uses: actions/checkout@v4

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: ${{ matrix.php }}
          extensions: json, dom, curl, libxml, mbstring
          coverage: none

      - name: Install Pint
        run: composer global require laravel/pint

      - name: Run Pint
        run: pint

      - name: Commit linted files
        uses: stefanzweifel/git-auto-commit-action@v5
```

