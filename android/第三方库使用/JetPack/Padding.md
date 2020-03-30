DataSource 类型

DataSource<Key,Value> 数据源：key 对应加载数据的条件信息，value 对应数据实体类型

- PageKeyedDataSource<Key,Value> ：适用于目标数据根据页面请求数据的场景
- ItemKeyedDataSource<Key,Value> ：适用于目标数据的加载依赖特定 item 的信息
- PositionalDataSource<Key,Value> ：适用于目标数据总数固定，通过特定的位置加载数据

