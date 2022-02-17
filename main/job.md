# versea AppSwitcher Job

AppSwitcher 中需要一个核心逻辑，job 和 job 执行序列的设计。

由于加载和 mount 应用的过程中可能又发生 reroute，然后 job 执行序列需要取消。