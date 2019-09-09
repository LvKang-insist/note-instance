```

/**
 * 我们在使用 butterknife 的时候一定要加入
 *  apply plugin: 'com.jakewharton.butterknife'
 *  annotationProcessor 'com.jakewharton:butterknife-compiler:8.4.0'
 *  否则 我们的项目会找不到 R2文件
 */
```

在 别的 library 中使用 ButterKnife 时 一点在 当前的 Module中加入 这两句话：

  apply plugin: 'com.jakewharton.butterknife'

annotationProcessor 'com.jakewharton:butterknife-compiler:8.4.0'