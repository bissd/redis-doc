创建一个唯一标识为`<groupname>`的新消费组，用于存储在`<key>`上的流。

每个给定流中的消费者组都有唯一的名称。
如果已存在具有相同名称的消费者组，则命令将返回“-BUSYGROUP”错误。

命令的 `<id>` 参数指定了从新群组视角中，流中最新交付条目的 ID。
特殊 ID `$` 是流中最后一个条目的 ID，但你可以用任何有效的 ID 替代它。

例如，如果您希望组的消费者从头开始获取整个流，使用零作为消费者组的起始ID：

    XGROUP CREATE mystream mygroup 0

默认情况下，`XGROUP CREATE`命令要求目标流存在，并在不存在时返回错误。
如果流不存在，可以在`<id>`后使用可选的`MKSTREAM`子命令作为最后一个参数自动创建长度为0的流：

    XGROUP CREATE mystream mygroup $ MKSTREAM

要启用消费者组滞后跟踪，请使用任意 ID 指定可选的 `entries_read` 参数。
任意 ID 是指不是流的第一个条目、最后一个条目或零 ("0-0") ID 的任何 ID。
使用它来查找在任意 ID（不包括它本身）和流的最后一个条目之间有多少条目。
将 `entries_read` 设置为流的 `entries_added` 减去条目数量。
