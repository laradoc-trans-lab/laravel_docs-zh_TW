# 日誌

- [簡介](#introduction)
- [設定](#configuration)
    - [可用的通道驅動程式](#available-channel-drivers)
    - [通道先決條件](#channel-prerequisites)
    - [記錄棄用警告](#logging-deprecation-warnings)
- [建構日誌堆疊](#building-log-stacks)
- [寫入日誌訊息](#writing-log-messages)
    - [上下文資訊](#contextual-information)
    - [寫入特定通道](#writing-to-specific-channels)
- [Monolog 通道客製化](#monolog-channel-customization)
    - [為通道客製化 Monolog](#customizing-monolog-for-channels)
    - [建立 Monolog 處理器通道](#creating-monolog-handler-channels)
    - [透過工廠建立客製化通道](#creating-custom-channels-via-factories)
- [使用 Pail 追蹤日誌訊息](#tailing-log-messages-using-pail)
    - [安裝](#pail-installation)
    - [用法](#pail-usage)
    - [過濾日誌](#pail-filtering-logs)

<a name="introduction"></a>
## 簡介

為了幫助您更了解應用程式中發生的情況，Laravel 提供了強大的日誌服務，讓您可以將訊息記錄到檔案、系統錯誤日誌，甚至發送到 Slack 以通知您的整個團隊。

Laravel 的日誌功能基於「通道 (channels)」。每個通道代表一種特定的日誌寫入方式。例如，`single` 通道將日誌寫入單一檔案，而 `slack` 通道則將日誌訊息發送到 Slack。日誌訊息可以根據其嚴重程度寫入多個通道。

在底層，Laravel 利用了 [Monolog](https://github.com/Seldaek/monolog) 函式庫，它支援各種強大的日誌處理器 (log handlers)。Laravel 讓這些處理器的設定變得輕而易舉，讓您可以混合搭配它們，以客製化應用程式的日誌處理。

<a name="configuration"></a>
## 設定

所有控制應用程式日誌行為的設定選項都位於 `config/logging.php` 設定檔中。此檔案允許您設定應用程式的日誌通道，因此請務必檢閱每個可用的通道及其選項。我們將在下面檢閱一些常見選項。

預設情況下，Laravel 在記錄訊息時將使用 `stack` 通道。`stack` 通道用於將多個日誌通道聚合到一個單一通道中。有關建構堆疊的更多資訊，請參閱[下面的說明文件](#building-log-stacks)。

<a name="available-channel-drivers"></a>
### 可用的通道驅動程式

每個日誌通道都由一個「驅動程式 (driver)」提供支援。驅動程式決定了日誌訊息實際記錄的方式和位置。每個 Laravel 應用程式都提供以下日誌通道驅動程式。這些驅動程式中的大多數條目已存在於您應用程式的 `config/logging.php` 設定檔
