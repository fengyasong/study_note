# 在springboot开启定时任务

1. 基于注解（@Scheduled）

2. 基于接口（SchedulingConfigurer，重写configureTasks方法）

    ```
    @Configuration      //1.主要用于标记配置类，兼备Component的效果。
    @EnableScheduling   // 2.开启定时任务
    @EnableAsync        //开启异步
    public class ScheduleTask implements SchedulingConfigurer {

        /**
        * 创建线程池
        */
        @Bean
        public Executor executor() {
            ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
            executor.setThreadNamePrefix("test-schedule");
            executor.setMaxPoolSize(20);
            executor.setCorePoolSize(10);
            executor.setQueueCapacity(5);// 缓存队列
            executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
            return executor;
        }

        
        @Async
        @Scheduled(cron = "0/5 * * * * ?")
        public void configureTasks() {
            System.err.println("执行定时任务1: " + LocalDateTime.now());
        }

        /**
        * 执行定时任务.
        */        
        @Async
        @Override
        public void configureTasks(ScheduledTaskRegistrar taskRegistrar) {
            taskRegistrar.addTriggerTask(
                    //1.添加任务内容(Runnable)
                    () -> {
                        System.out.println("执行动态定时任务: ");
                    },
                    //2.设置执行周期(Trigger)
                    triggerContext -> {
                        //执行周期
                        String cron = "*/6 * * * * ?";
                        //返回执行周期(Date)
                        return new CronTrigger(cron).nextExecutionTime(triggerContext);
                    }
            );
        }
    }
    ```

@Scheduled 参数可以接受两种定时的设置，一种是我们常用的cron="*/6 * * * * ?",一种是 fixedRate = 6000，两种都表示每隔六秒打印一下内容。

+ fixedRate 说明
    - @Scheduled(fixedRate = 6000) ：上一次开始执行时间点之后6秒再执行
    - @Scheduled(fixedDelay = 6000) ：上一次执行完毕时间点之后6秒再执行
    - @Scheduled(initialDelay=1000, fixedRate=6000) ：第一次延迟1秒后执行，之后按 fixedRate 的规则每6秒执行一次

    ```
    /**
        *  Cron表达式参数分别表示：
        * 秒（0~59） 例如0/5表示每5秒
        * 分（0~59）
        * 时（0~23）
        * 月的某天（0~31） 需计算
        * 月（0~11）
        * 周几（ 可填1-7 或 SUN/MON/TUE/WED/THU/FRI/SAT）
        *
        * 0 0 1 * * ?	每天1点0分0秒执行一次
        */
    ```