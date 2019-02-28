## 乐观锁

---

`_version`：每次修改和删除数据时都会使得`_version+=1`，当删除数据的时候并不是马上在物理上删除数据，而是标记数据为`deleted`,这时再插入这条数据时，`_version`并不会从`1`开始，而是删除前的`_version+1`.

基于版本号进行并发控制时，可以加上参数`_version=number`,

例：

`PUT /index/type/id?version=version_number`:当`_version`为给定的`version_number`时才执行`PUT`操作。

`PUT /index/type/id?version=version_number&version_type=external`：当`_version`小于给定的`version_number`时执行`PUT`操作，用于使用非`ES`自带的`version_number`来进行并发控制时。