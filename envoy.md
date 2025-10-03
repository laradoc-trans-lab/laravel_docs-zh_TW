# Laravel Envoy

- [簡介](#introduction)
- [安裝](#installation)
- [編寫任務](#writing-tasks)
    - [定義任務](#defining-tasks)
    - [多個伺服器](#multiple-servers)
    - [設定](#setup)
    - [變數](#variables)
    - [故事](#stories)
    - [掛鉤](#completion-hooks)
- [執行任務](#running-tasks)
    - [確認任務執行](#confirming-task-execution)
- [通知](#notifications)
    - [Slack](#slack)
    - [Discord](#discord)
    - [Telegram](#telegram)
    - [Microsoft Teams](#microsoft-teams)

<a name="introduction"></a>
## 簡介

[Laravel Envoy](https://github.com/laravel/envoy) 是一個用於在遠端伺服器上執行常見任務的工具。透過使用 [Blade](/docs/{{version}}/blade) 風格的語法，您可以輕鬆設定部署、Artisan 指令等任務。目前，Envoy 僅支援 Mac 和 Linux 作業系統。然而，透過使用 [WSL2](https://docs.microsoft.com/en-us/windows/wsl/install-win10)，也可以實現在 Windows 上支援。


<a name="installation"></a>
## 安裝

首先，使用 Composer 套件管理工具將 Envoy 安裝到您的專案中：

```shell
composer require laravel/envoy --dev
```

一旦 Envoy 安裝完成，Envoy 執行檔將可在您的應用程式 `vendor/bin` 目錄中找到：

```shell
php vendor/bin/envoy
```


<a name="writing-tasks"></a>
## 編寫任務


<a name="defining-tasks"></a>
### 定義任務

任務是 Envoy 的基本構成要素。任務定義了當被呼叫時，應在您的遠端伺服器上執行的 Shell 指令。例如，您可以定義一個任務，在應用程式的所有佇列 worker 伺服器上執行 `php artisan queue:restart` 指令。

所有 Envoy 任務都應定義在應用程式根目錄下的 `Envoy.blade.php` 檔案中。以下是一個入門範例：

```blade
@servers(['web' => ['user@192.168.1.1'], 'workers' => ['user@192.168.1.2']])

@task('restart-queues', ['on' => 'workers'])
    cd /home/user/example.com
    php artisan queue:restart
@endtask
```

如您所見，檔案頂部定義了一個 `@servers` 陣列，讓您可以透過任務宣告的 `on` 選項來引用這些伺服器。`@servers` 宣告應始終放在單行。在您的 `@task` 宣告中，您應放置在任務被呼叫時應在伺服器上執行的 Shell 指令。


<a name="local-tasks"></a>
#### 本機任務

您可以透過將伺服器的 IP 位址指定為 `127.0.0.1`，來強制腳本在您的本機電腦上執行：

```blade
@servers(['localhost' => '127.0.0.1'])
```


<a name="importing-envoy-tasks"></a>
#### 匯入 Envoy 任務

使用 `@import` 指令，您可以匯入其他 Envoy 檔案，使其故事 (stories) 和任務 (tasks) 添加到您的檔案中。檔案匯入後，您可以執行其中包含的任務，如同它們定義在您自己的 Envoy 檔案中一樣：

```blade
@import('vendor/package/Envoy.blade.php')
```


<a name="multiple-servers"></a>
### 多個伺服器

Envoy 讓您可以輕鬆地在多個伺服器上執行任務。首先，將額外的伺服器添加到您的 `@servers` 宣告中。每個伺服器都應被分配一個唯一的名稱。一旦您定義了額外的伺服器，您就可以在任務的 `on` 陣列中列出每個伺服器：

```blade
@servers(['web-1' => '192.168.1.1', 'web-2' => '192.168.1.2'])

@task('deploy', ['on' => ['web-1', 'web-2']])
    cd /home/user/example.com
    git pull origin {{ $branch }}
    php artisan migrate --force
@endtask
```


<a name="parallel-execution"></a>
#### 平行執行

預設情況下，任務將在每個伺服器上依序執行。換句話說，一個任務會在第一個伺服器上完成執行後，才會繼續在第二個伺服器上執行。如果您希望在多個伺服器上平行執行任務，請將 `parallel` 選項添加到您的任務宣告中：

```blade
@servers(['web-1' => '192.168.1.1', 'web-2' => '192.168.1.2'])

@task('deploy', ['on' => ['web-1', 'web-2'], 'parallel' => true])
    cd /home/user/example.com
    git pull origin {{ $branch }}
    php artisan migrate --force
@endtask
```


<a name="setup"></a>
### 設定

有時，您可能需要在執行 Envoy 任務之前執行任意的 PHP 程式碼。您可以使用 `@setup` 指令來定義一個 PHP 程式碼區塊，該區塊應在您的任務之前執行：

```php
@setup
    $now = new DateTime;
@endsetup
```

如果您需要在任務執行之前載入其他 PHP 檔案，您可以在 `Envoy.blade.php` 檔案的頂部使用 `@include` 指令：

```blade
@include('vendor/autoload.php')

@task('restart-queues')
    # ...
@endtask
```


<a name="variables"></a>
### 變數

如有需要，您可以在呼叫 Envoy 時，透過在命令列中指定參數來傳遞給 Envoy 任務：

```shell
php vendor/bin/envoy run deploy --branch=master
```

您可以使用 Blade 的 "echo" 語法在任務中存取選項。您也可以在任務中定義 Blade `if` 語句和迴圈。例如，讓我們在執行 `git pull` 指令之前，驗證 `$branch` 變數是否存在：

```blade
@servers(['web' => ['user@192.168.1.1']])

@task('deploy', ['on' => 'web'])
    cd /home/user/example.com

    @if ($branch)
        git pull origin {{ $branch }}
    @endif

    php artisan migrate --force
@endtask
```


<a name="stories"></a>
### 故事

故事 (Stories) 將一系列任務歸納到一個單一、方便的名稱下。例如，一個 `deploy` 故事可以透過在其定義中列出任務名稱來執行 `update-code` 和 `install-dependencies` 任務：

```blade
@servers(['web' => ['user@192.168.1.1']])

@story('deploy')
    update-code
    install-dependencies
@endstory

@task('update-code')
    cd /home/user/example.com
    git pull origin master
@endtask

@task('install-dependencies')
    cd /home/user/example.com
    composer install
@endtask
```

一旦故事編寫完成，您就可以像呼叫任務一樣呼叫它：

```shell
php vendor/bin/envoy run deploy
```


<a name="completion-hooks"></a>
### 掛鉤

當任務和故事執行時，會執行多個掛鉤 (hooks)。Envoy 支援的掛鉤類型有 `@before`、`@after`、`@error`、`@success` 和 `@finished`。這些掛鉤中的所有程式碼都將被解釋為 PHP 並在本機執行，而不是在您的任務互動的遠端伺服器上執行。

您可以定義任意數量的這些掛鉤。它們將按照在您的 Envoy 腳本中出現的順序執行。


<a name="hook-before"></a>
#### `@before`

在每個任務執行之前，所有註冊在您的 Envoy 腳本中的 `@before` 掛鉤都將執行。`@before` 掛鉤會接收將被執行的任務名稱：

```blade
@before
    if ($task === 'deploy') {
        // ...
    }
@endbefore
```


<a name="completion-after"></a>
#### `@after`

在每個任務執行之後，所有註冊在您的 Envoy 腳本中的 `@after` 掛鉤都將執行。`@after` 掛鉤會接收已執行的任務名稱：

```blade
@after
    if ($task === 'deploy') {
        // ...
    }
@endafter
```


<a name="completion-error"></a>
#### `@error`

在每個任務失敗（以大於 `0` 的狀態碼結束）之後，所有註冊在您的 Envoy 腳本中的 `@error` 掛鉤都將執行。`@error` 掛鉤會接收已執行的任務名稱：

```blade
@error
    if ($task === 'deploy') {
        // ...
    }
@enderror
```


<a name="completion-success"></a>
#### `@success`

如果所有任務都已無錯誤執行，則所有註冊在您的 Envoy 腳本中的 `@success` 掛鉤都將執行：

```blade
@success
    // ...
@endsuccess
```


<a name="completion-finished"></a>
#### `@finished`

所有任務執行完畢後（無論退出狀態為何），所有 `@finished` 掛鉤都將執行。`@finished` 掛鉤會接收已完成任務的狀態碼，該狀態碼可能是 `null` 或大於或等於 `0` 的 `integer`：

```blade
@finished
    if ($exitCode > 0) {
        // There were errors in one of the tasks...
    }
@endfinished
```

<a name="running-tasks"></a>
## 執行任務

要執行應用程式 `Envoy.blade.php` 檔案中定義的任務或故事，請執行 Envoy 的 `run` 命令，並傳遞您要執行的任務或故事名稱。Envoy 將會執行該任務並在任務執行時顯示來自遠端伺服器的輸出：

```shell
php vendor/bin/envoy run deploy
```

<a name="confirming-task-execution"></a>
### 確認任務執行

如果您希望在伺服器上執行指定任務之前收到確認提示，您應該在任務宣告中添加 `confirm` 指令。此選項對於破壞性操作特別有用：

```blade
@task('deploy', ['on' => 'web', 'confirm' => true])
    cd /home/user/example.com
    git pull origin {{ $branch }}
    php artisan migrate
@endtask
```

<a name="notifications"></a>
## 通知

<a name="slack"></a>
### Slack

Envoy 支援在每個任務執行後發送通知到 [Slack](https://slack.com)。`@slack` 指令接受一個 Slack webhook URL 以及一個頻道 / 使用者名稱。您可以透過在 Slack 控制面板中建立「Incoming WebHooks」整合來取得您的 webhook URL。

您應該將完整的 webhook URL 作為傳遞給 `@slack` 指令的第一個參數。傳遞給 `@slack` 指令的第二個參數應該是一個頻道名稱 (`#channel`) 或一個使用者名稱 (`@user`)：

```blade
@finished
    @slack('webhook-url', '#bots')
@endfinished
```

依預設，Envoy 通知會向通知頻道發送一條描述已執行任務的訊息。然而，您可以透過向 `@slack` 指令傳遞第三個參數來覆寫此訊息，使用您自己的自訂訊息：

```blade
@finished
    @slack('webhook-url', '#bots', 'Hello, Slack.')
@endfinished
```

<a name="discord"></a>
### Discord

Envoy 也支援在每個任務執行後發送通知到 [Discord](https://discord.com)。`@discord` 指令接受一個 Discord webhook URL 和一條訊息。您可以透過在您的伺服器設定 (Server Settings) 中建立一個「Webhook」並選擇 webhook 應該發佈到的頻道來取得您的 webhook URL。您應該將完整的 Webhook URL 傳遞給 `@discord` 指令：

```blade
@finished
    @discord('discord-webhook-url')
@endfinished
```

<a name="telegram"></a>
### Telegram

Envoy 也支援在每個任務執行後發送通知到 [Telegram](https://telegram.org)。`@telegram` 指令接受一個 Telegram Bot ID 和一個 Chat ID。您可以透過使用 [BotFather](https://t.me/botfather) 建立一個新 Bot 來取得您的 Bot ID。您可以透過使用 [@username_to_id_bot](https://t.me/username_to_id_bot) 取得有效的 Chat ID。您應該將完整的 Bot ID 和 Chat ID 傳遞給 `@telegram` 指令：

```blade
@finished
    @telegram('bot-id','chat-id')
@endfinished
```

<a name="microsoft-teams"></a>
### Microsoft Teams

Envoy 也支援在每個任務執行後發送通知到 [Microsoft Teams](https://www.microsoft.com/en-us/microsoft-teams)。`@microsoftTeams` 指令接受一個 Teams Webhook (必填)、一條訊息、主題顏色 (success、info、warning、error) 以及一個選項陣列。您可以透過建立一個新的 [傳入 Webhook](https://docs.microsoft.com/en-us/microsoftteams/platform/webhooks-and-connectors/how-to/add-incoming-webhook) 來取得您的 Teams Webhook。Teams API 有許多其他屬性可以自訂您的訊息方塊，例如標題 (title)、摘要 (summary) 和區塊 (sections)。您可以在 [Microsoft Teams 文件](https://docs.microsoft.com/en-us/microsoftteams/platform/webhooks-and-connectors/how-to/connectors-using?tabs=cURL#example-of-connector-message) 中找到更多資訊。您應該將完整的 Webhook URL 傳遞給 `@microsoftTeams` 指令：

```blade
@finished
    @microsoftTeams('webhook-url')
@endfinished
```