- **git 提交规范**

  feat： 新增 feature

  fix: 修复 bug

  docs: 仅仅修改了文档，比如 README, CHANGELOG, CONTRIBUTE等等

  style: 仅仅修改了空格、格式缩进、逗号等等，不改变代码逻辑

  refactor: 代码重构，没有加新功能或者修复 bug

  perf: 优化相关，比如提升性能、体验

  test: 测试用例，包括单元测试、集成测试等

  chore: 改变构建流程、或者增加依赖库、工具等

  revert: 回滚到上一个版本

- Git tag

  Git 中的 Tag 指向一次 commit 的 id，通常用来给开发分支做一个标记，例如标记一个版本号。

  tag 主要用于发布版本的管理，一个版本发布后，可以为当前的 commit id 打上 v 1.0.1 等这样的标签。

  tag 对应的是某次 commit 节点，是一个点，不可以动的。而 branche 对应一些列的 commit ，是很多点连成的一根线，有一个 head 指针，是可以依靠 head 指针移动的。

  所以，两者的区别也觉得了使用方式，改动代码用 branch，不该动只查看用 tag。

  > tag 和 branch 的相互配合使用，有时候起到了非常方便的效果，例如：已经发布了 v1.0，v2.0，v3.0 版本，这个时候需要在 v2.0 的基础上加个新功能，作为 v4.0 ，这可时候就可以检出 v2.0的代码作为一个 branch，然后作为开发分支。

  

  - 添加 tag

    ```text
    git tag -a v1.0-beta -m "v1.0 beta版本发布上线"
    ```

    也可以不使用 -a 和 -m，如下：

     

    ```
    git tag v1.0
    ```

    给指定的 commit 打上 tag

    ```
    git tag -a v0.1.1 9fbc3d0
    ```

    打 tag 也可以在之前的版本上打，你需要知道某个提交对象的校验，通过 git log 获取。

  - 查看所有的 tab 版本

    ```
    git tag ：查看tag列表
    git tag --list ：查看tag列表
    git tag -l ：同理查看tag列表
    ```

  - 切换 标签

    ```
    git checkout v1.0
    ```

  - 删除 tag

    ```
    git tag -d tagName //删除本地标签
    ```

    ```
    git push origin :refs/tags/ # 删除一个远程标签
    ```

  - 推送到远端

    ```
    git push origin v1.0 //推送指定的 tag
    git push origin --tags  //推送全部未推送的 tag
    ```

  - 同步远端

    ```
    git pull --rebase origin master//先同步
    git push -u origin master //再上传
    ```

    

  

  

- 退出编辑器

  :wq 保存退出

  :q! 不保存退出
