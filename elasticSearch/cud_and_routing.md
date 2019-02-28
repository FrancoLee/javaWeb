

## CUD

---

增删改流程：

+ `client`任意选择一个`node`发送请求，被选中的`node`称为`coordinate node(协调节点)`.
+ `coordinate node`根据路由算法把请求转发给对应的`primary shard`.
+ `primary shard`执行请求对应的操作，并将数据同步到自己的`replica shard`.
+ `repliac shard`同步好数据后，`primary shard`告知结果给`coordinate node`.
+ `coordinate node`将结果响应给`client`.

## R

---

读文件流程：

+ `client`任意选择一个`node`发送请求，被选中的`node`称为`coordinate node(协调节点)`.
+ `coordinate node`根据路由算法得知数据在哪个`primary shard`上
+ 通过`round-robin(轮询调度算法)`来选择将请求转发给`primary shard`和`replica shard`中的一个
+ 接收到请求的`shard`将`document`数据返回给`coordinate node`
+ `coordinate node`将`document`返回给`client`.

注：有可能数据在`primary shard`上已经建立好索引，但是`replica shard`上索引还没建立好，此时若`coordinate node`将请求转发给`replica shard`上，会报`document`找不到。

## routing 

---

`document`在存储时和搜索时，会使用路由算法来决定数据在哪个`shard`上。

路由算法：`hash(routing)%number`

`routing`:默认是`document`的`_id`,也可以通过参数来手动指定`PUT /index/type/id?routing=routing_id`。这种做法可以手动的指定某些数据在同一个`shard`上。

`number`:`primary shard`的数量,所以`primary shard`一旦确定后就不可改变