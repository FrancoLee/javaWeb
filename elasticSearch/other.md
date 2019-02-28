## 记一些指令

---

`retry_on_conflict`：设置更新失败再尝试次数，每次失败都会获取最新的版本后再更新。

---

`json`内的参数：

`upsert`:如果要操作的`document`存在就执行`doc`或者`script`的内操作，否知执行`upsert`的指定定的初始化操作。

