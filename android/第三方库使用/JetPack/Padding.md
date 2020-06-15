Padding 分页组件简介

1，核心类：PagedList

​	PagedList 就是分页列表数据的容器，那为什么要设计这样的 PagedList 数据结构，本质的原因是应为分页数据的操作时异步的，因此 PagedList 可以对分页数据进行异步加载。

2，DataSource 及其工厂

​	简单点说就是用来为 PagedList 容器提供分页数据，那就是数据源 DataSource。

​	没到 padding 被告知需要加载更多数据， DataSource 就会将对应索引的数据交给 pagedList 

DataSource 数据类型

DataSource<Key,Value> 数据源：key 对应加载数据的条件信息，value 对应数据实体类型

- PageKeyedDataSource<Key,Value> ：适用于目标数据根据页面请求数据的场景
- ItemKeyedDataSource<Key,Value> ：适用于目标数据的加载依赖特定 item 的信息
- PositionalDataSource<Key,Value> ：适用于目标数据总数固定，通过特定的位置加载数据

