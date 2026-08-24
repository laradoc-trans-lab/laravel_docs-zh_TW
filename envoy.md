# Laravel Envoy

- [簡介](#introduction)
- [安裝](#installation)
- [撰寫任務](#writing-tasks)
    - [定義任務](#defining-tasks)
    - [多台伺服器](#multiple-servers)
    - [前置設定](#setup)
    - [變數](#variables)
    - [故事 (Stories)](#stories)
    - [掛鉤 (Hooks)](#completion-hooks)
- [執行任務](#running-tasks)
    - [確認任務執行](#confirming-task-execution)
- [通知](#notifications)
    - [Slack](#slack)
    - [Discord](#discord)
    - [Telegram](#telegram)
    - [Microsoft Teams](#microsoft-teams)

<a name="introduction"></a>
## 簡介

[Laravel Envoy](https://github.com/laravel/envoy) 是一個用於在遠端伺服器上執行常見任務的工具。使用 [Blade](/docs/{{version}}/blade) 風格的語法，您可以輕鬆設定部署任務、Artisan 指令等功能。目前 Envoy 僅支援 Mac 和 Linux 作業系統。不過，您也可以透過 [WSL2](https://docs.microsoft.com/en-us/windows/wsl/install-win10) 在 Windows 上運行。


<a name="installation"></a>
## 安裝

首先，使用 Composer 套件管理器將 Envoy 安裝到您的專案中：

```shell
composer require laravel/envoy --dev
```

安裝 Envoy 後，Envoy 執行檔將會位於應用程式的 `vendor/bin` 目錄中：

```shell
php vendor/bin/envoy
```


<a name="writing-tasks"></a>
## 撰寫任務


<a name="defining-tasks"></a>
### 定義任務

任務是 Envoy 的基本建構區塊。任務定義了當呼叫該任務時，應在遠端伺服器上執行的 Shell 指令。例如，您可以定義一個任務，在應用程式的所有佇列 Worker 伺服器上執行 `php artisan queue:restart` 指令。

所有的 Envoy 任務都應該定義在應用程式根目錄下的 `Envoy.blade.php` 檔案中。以下是一個入門範例：

```blade
@servers(['web' => ['user@192.168.1.1'], 'workers' => ['user@192.168.1.2']])

@task('restart-queues', ['on' => 'workers'])
    cd /home/user/example.com
    php artisan queue:restart
@endtask
```

如您所見，檔案頂端定義了一個 `@servers` 陣列，允許您透過任務宣告中的 `on` 選項來參照這些伺服器。`@servers` 宣告應始終寫在同一行。在 `@task` 宣告中，您應該放置呼叫該任務時要在伺服器上執行的 Shell 指令。


<a name="local-tasks"></a>
#### 本地任務

您可以透過將伺服器的 IP 位址指定為 `127.0.0.1`，強制指令稿在本機電腦上執行：

```blade
@servers(['localhost' => '127.0.0.1'])
```


<a name="importing-envoy-tasks"></a>
#### 匯入 Envoy 任務

使用 `@import` 指令，您可以匯入其他 Envoy 檔案，將它們的故事與任務新增到您的檔案中。檔案匯入後，您可以執行其中包含的任務，就像它們定義在您自己的 Envoy 檔案中一樣：

```blade
@import('vendor/package/Envoy.blade.php')
```


<a name="multiple-servers"></a>
### 多台伺服器

Envoy 讓您可以輕鬆跨多台伺服器執行任務。首先，將其他伺服器新增至您的 `@servers` 宣告中。每台伺服器都應該分配一個唯一的名稱。定義好其他伺服器後，您可以在任務的 `on` 陣列中列出各台伺服器：

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

預設情況下，任務將在每台伺服器上依序執行。換句話說，任務會在第一台伺服器上執行完成後，才繼續在第二台伺服器上執行。如果您希望跨多台伺服器平行執行任務，請在任務宣告中新增 `parallel` 選項：

```blade
@servers(['web-1' => '192.168.1.1', 'web-2' => '192.168.1.2'])

@task('deploy', ['on' => ['web-1', 'web-2'], 'parallel' => true])
    cd /home/user/example.com
    git pull origin {{ $branch }}
    php artisan migrate --force
@endtask
```


<a name="setup"></a>
### 前置設定

有時，您可能需要在執行 Envoy 任務之前執行任意 PHP 程式碼。您可以使用 `@setup` 指令來定義在任務執行前應執行的 PHP 程式碼區塊：

```php
@setup
    $now = new DateTime;
@endsetup
```

如果您在執行任務前需要載入其他 PHP 檔案，可以在 `Envoy.blade.php` 檔案的頂端使用 `@include` 指令：

```blade
@include('vendor/autoload.php')

@task('restart-queues')
    # ...
@endtask
```


<a name="variables"></a>
### 變數

如果需要，您可以在呼叫 Envoy 時，透過命令列指定引數傳遞給 Envoy 任務：

```shell
php vendor/bin/envoy run deploy --branch=master
```

您可以在任務中使用 Blade 的「輸出 (echo)」語法來存取這些選項。您也可以在任務中定義 Blade 的 `if` 陳述式和迴圈。例如，讓我們在執行 `git pull` 指令之前驗證 `$branch` 變數是否存在：

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
### 故事 (Stories)

故事將一組任務整合在一個方便的單一名稱下。例如，`deploy` 故事可以透過在其定義中列出任務名稱，來執行 `update-code` 和 `install-dependencies` 任務：

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

撰寫好故事後，您可以像呼叫任務一樣來呼叫它：

```shell
php vendor/bin/envoy run deploy
```


<a name="completion-hooks"></a>
### 掛鉤 (Hooks)

當任務和故事執行時，會觸發許多掛鉤。Envoy 支援的掛鉤類型有 `@before`、`@after`、`@error`、`@success` 和 `@finished`。這些掛鉤中的所有程式碼都會被解析為 PHP 並在本地執行，而不是在任務所互動的遠端伺服器上執行。

您可以根據需要定義任意數量的掛鉤。它們將按照在 Envoy 指令稿中出現的順序執行。


<a name="hook-before"></a>
#### `@before`

在每個任務執行之前，Envoy 指令稿中註冊的所有 `@before` 掛鉤都會執行。`@before` 掛鉤會接收即將執行的任務名稱：

```blade
@before
    if ($task === 'deploy') {
        // ...
    }
@endbefore
```


<a name="completion-after"></a>
#### `@after`

在每個任務執行之後，Envoy 指令稿中註冊的所有 `@after` 掛鉤都會執行。`@after` 掛鉤會接收剛剛執行的任務名稱：

```blade
@after
    if ($task === 'deploy') {
        // ...
    }
@endafter
```


<a name="completion-error"></a>
#### `@error`

在每個任務執行失敗（結束狀態碼大於 `0`）之後，Envoy 指令稿中註冊的所有 `@error` 掛鉤都會執行。`@error` 掛鉤會接收剛剛執行的任務名稱：

```blade
@error
    if ($task === 'deploy') {
        // ...
    }
@enderror
```


<a name="completion-success"></a>
#### `@success`

如果所有任務執行都沒有發生錯誤，Envoy 指令稿中註冊的所有 `@success` 掛鉤都會執行：

```blade
@success
    // ...
@endsuccess
```


<a name="completion-finished"></a>
#### `@finished`

在所有任務都執行完畢後（無論結束狀態為何），所有的 `@finished` 掛鉤都會被執行。`@finished` 掛鉤會接收已完成任務的狀態碼，該狀態碼可能是 `null` 或是大於等於 `0` 的 `integer`：

```blade
@finished
    if ($exitCode > 0) {
        // There were errors in one of the tasks...
    }
@endfinished
```

<a name="running-tasks"></a>
## 執行任務

若要執行在應用程式的 `Envoy.blade.php` 檔案中所定義的任務或故事 (Story)，請執行 Envoy 的 `run` 指令，並傳入您想要執行的任務或故事名稱。Envoy 將會執行該任務，並在任務執行時顯示來自遠端伺服器的輸出：

```shell
php vendor/bin/envoy run deploy
```


<a name="confirming-task-execution"></a>
### 確認任務執行

若您希望在伺服器上執行指定任務之前先收到確認提示，可以在任務宣告中加入 `confirm` 指令。這個選項對於具破壞性的操作特別有用：

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

Envoy 支援在每個任務執行完成後發送通知到 [Slack](https://slack.com)。`@slack` 指令接受一個 Slack Hook 網址以及頻道／使用者名稱。您可以透過在 Slack 控制面板中建立「Incoming WebHooks」整合來取得您的 Webhook 網址。

您應該將完整的 Webhook 網址作為第一個引數傳入 `@slack` 指令。傳入 `@slack` 指令的第二個引數應該是頻道名稱 (`#channel`) 或使用者名稱 (`@user`)：

```blade
@finished
    @slack('webhook-url', '#bots')
@endfinished
```

預設情況下，Envoy 通知會向通知頻道發送一條描述已執行任務的訊息。不過，您可以透過傳入第三個引數給 `@slack` 指令，用自訂訊息覆寫此訊息：

```blade
@finished
    @slack('webhook-url', '#bots', 'Hello, Slack.')
@endfinished
```


<a name="discord"></a>
### Discord

Envoy 也支援在每個任務執行完成後發送通知到 [Discord](https://discord.com)。`@discord` 指令接受一個 Discord Hook 網址與一則訊息。您可以透過在伺服器設定中建立「Webhook」並選擇 Webhook 要發布到哪個頻道來取得 Webhook 網址。您應該將完整的 Webhook 網址傳入 `@discord` 指令：

```blade
@finished
    @discord('discord-webhook-url')
@endfinished
```


<a name="telegram"></a>
### Telegram

Envoy 也支援在每個任務執行完成後發送通知到 [Telegram](https://telegram.org)。`@telegram` 指令接受一個 Telegram Bot ID 與一個 Chat ID。您可以透過使用 [BotFather](https://t.me/botfather) 建立新的機器人來取得 Bot ID。您可以使用 [@username_to_id_bot](https://t.me/username_to_id_bot) 取得有效的 Chat ID。您應該將完整的 Bot ID 與 Chat ID 傳入 `@telegram` 指令：

```blade
@finished
    @telegram('bot-id','chat-id')
@endfinished
```


<a name="microsoft-teams"></a>
### Microsoft Teams

Envoy 也支援在每個任務執行完成後發送通知到 [Microsoft Teams](https://www.microsoft.com/en-us/microsoft-teams)。`@microsoftTeams` 指令接受一個 Teams Webhook (必填)、一則訊息、主題顏色 (success、info、warning、error) 以及一個選項陣列。您可以透過建立新的 [傳入 Webhook](https://docs.microsoft.com/en-us/microsoftteams/platform/webhooks-and-connectors/how-to/add-incoming-webhook) 來取得 Teams Webhook。Teams API 還有許多其他屬性可以自訂您的訊息方塊，例如標題 (title)、摘要 (summary) 和區段 (sections)。您可以在 [Microsoft Teams 文件](https://docs.microsoft.com/en-us/microsoftteams/platform/webhooks-and-connectors/how-to/connectors-using?tabs=cURL#example-of-connector-message) 中找到更多資訊。您應該將完整的 Webhook 網址傳入 `@microsoftTeams` 指令：

```blade
@finished
    @microsoftTeams('webhook-url')
@endfinished
```