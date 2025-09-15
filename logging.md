# 日誌

- [簡介](#introduction)
- [設定](#configuration)
    - [可用的 Channel 驅動](#available-channel-drivers)
    - [Channel 的先決條件](#channel-prerequisites)
    - [記錄棄用警告](#logging-deprecation-warnings)
- [建立日誌堆疊](#building-log-stacks)
- [寫入日誌訊息](#writing-log-messages)
    - [上下文資訊](#contextual-information)
    - [寫入到特定 Channel](#writing-to-specific-channels)
- [Monolog Channel 自訂](#monolog-channel-customization)
    - [為 Channel 自訂 Monolog](#customizing-monolog-for-channels)
    - [建立 Monolog Handler Channel](#creating-monolog-handler-channels)
    - [透過 Factory 建立自訂 Channel](#creating-custom-channels-via-factories)
- [使用 Pail 追蹤日誌訊息](#tailing-log-messages-using-pail)
    - [安裝](#pail-installation)
    - [使用方式](#pail-usage)
    - [過濾日誌](#pail-filtering-logs)

<a name="introduction"></a>
## 簡介

為了幫助您更了解應用程式中發生的情況，Laravel 提供了強大的日誌服務，讓您可以將訊息記錄到檔案、系統錯誤日誌，甚至發送到 Slack 以通知整個團隊。

Laravel 的日誌功能基於「Channel」。每個 Channel 代表一種特定的日誌資訊寫入方式。例如，`single` Channel 會將日誌寫入單一檔案，而 `slack` Channel 則會將日誌訊息發送到 Slack。日誌訊息可以根據其嚴重程度寫入多個 Channel。

在底層，Laravel 利用了 [Monolog](https://github.com/Seldaek/monolog) 函式庫，該函式庫支援各種強大的日誌 Handler。Laravel 讓設定這些 Handler 變得輕而易舉，讓您可以混合搭配它們來自訂應用程式的日誌處理。

<a name="configuration"></a>
## 設定

所有控制應用程式日誌行為的設定選項都位於 `config/logging.php` 設定檔中。此檔案允許您設定應用程式的日誌 Channel，因此請務必檢閱每個可用的 Channel 及其選項。我們將在下面檢閱一些常見選項。

預設情況下，Laravel 在記錄訊息時會使用 `stack` Channel。`stack` Channel 用於將多個日誌 Channel 聚合到單一 Channel 中。有關建立堆疊的更多資訊，請參閱[下面的說明文件](#building-log-stacks)。

<a name="available-channel-drivers"></a>
### 可用的 Channel 驅動

每個日誌 Channel 都由一個「驅動 (driver)」提供支援。驅動決定了日誌訊息實際記錄的方式和位置。以下日誌 Channel 驅動在每個 Laravel 應用程式中都可用。這些驅動中的大多數條目已存在於應用程式的 `config/logging.php
