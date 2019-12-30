

JobService 是 API21 中引入的一个服务。下面看一下官方介绍：

Entry point for the callback from the `JobScheduler`.

This is the base class that handles asynchronous requests that were previously scheduled. You are responsible for overriding `JobService#onStartJob(JobParameters)`, which is where you will implement your job logic.

This service executes each incoming job on a `Handler` running on your application's main thread. This means that you **must** offload your execution logic to another thread/handler/`AsyncTask` of your choosing. Not doing so will result in blocking any future callbacks from the JobManager - specifically `onStopJob(android.app.job.JobParameters)`, which is meant to inform you that the scheduling requirements are no longer being met.

大致是说：

JobService 是 JobScheduler 的回调入口。需要我们覆盖 JobService 中的 onStartJob 方法，这里是实现工作逻辑的地方。因为 JobService 是执行在主线程内的。所以必须提供额外的异步逻辑去执行这些任务。

下面看一下他的 api

-  onStartJob(JobParameters params) ：Job 开始的时候回调，运行在主线程。返回 true 系统会在适当的时候启动它。返回 false 表示已完成工作，不会再启动。
- onStopJob(JobParameters params)：Job 停止的时候调用此方法，当 JobScheduler 发现 job 条件不满足的时候，或者 Job 被抢占取消时候的回调。如果想让 job 在这种意外下重新开始，应该返回 true。
- jobFinished(JobParameters params, boolean wantsReschedule)：被 final 修饰，调用此函数通知JobScheduler作业已完成其工作。调用该方法后则不会回调 onStopJob。但是会回调 onDestory。



从 api 可以看出 JobService 只是执行和结束的入口。那么如何将这个入口告诉系统，就需要用的 JobScheduler 了。

------



