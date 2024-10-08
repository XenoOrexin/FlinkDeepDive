## 事件时间 (event time) 与处理时间 (processing time)


**事件时间（Event Time）**： 事件时间是指每个事件 (event) 在其生成设备上发生的时间。这个时间通常以时间戳的形式记录在消息体中，可以从每条消息中提取。在事件时间语义下，时间的进展取决于数据，而非 flink 集群所在服务器的时间。事件时间程序必须指定如何生成事件时间水位线（Event Time Watermarks），这是一个用来标记事件时间进度的机制。

**处理时间（Processing Time）**： 处理时间是指 flink 集群所在服务器针对每个 event 执行相应操作系统时间。在处理时间语义下，所有的基于时间的操作（例如时间窗口）都将处理器的系统时钟。例如: 如果应用程序在上午 9:15 开始运行，第一个按小时处理时间的窗口将包括在上午 9:15 到 10:00 之间处理的事件，接下来的窗口将包括上午 10:00 到 11:00 之间处理的事件，依此类推。



