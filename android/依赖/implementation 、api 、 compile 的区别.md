

implementation 和 api 的区别

api：跟2.x版本的 compile完全相同

implementation：只能在内部使用此模块，比如我在一个libiary中使用implementation依赖了gson库，然后我的主项目依赖了 libiary，那么，我的主项目就无法访问gson库中的方法。这样的好处是编译速度会加快，推荐使用implementation的方式去依赖，如果你需要提供给外部访问，那么就使用api依赖即可

