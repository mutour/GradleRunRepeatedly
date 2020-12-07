# GradleRunRepeatedly
该脚本主要用于重复运行Task

# 前言
最近处理脚本时, 需要用到重复某一个Task运行, 才做过程中发现gradle不支持task的重复运行. 经过内部运行分析发现通过调整Executer可以实现该功能

# gradle版本支持
该脚本时针对4.8.1进行处理的, gradle 6+貌似修改了内部逻辑,暂不支持  
`distributionUrl=https\://services.gradle.org/distributions/gradle-4.8.1-bin.zip`

# gradle调试
命令行运行`./gradlew :createTask -Dorg.gradle.daemon=false -Dorg.gradle.debug=true`
IDEA Edit Configurations添加Remote. host: localhost, port: 5005, 然后Debug即可, 可以断点到gradle源码中, 部分gradle也可以断点


# 以下是相关逻辑

```java

import org.gradle.api.internal.TaskInternal
import org.gradle.api.internal.tasks.TaskExecuter
import org.gradle.api.internal.AbstractTask
import org.gradle.api.internal.tasks.TaskExecutionContext
import org.gradle.api.internal.tasks.TaskStateInternal
import org.gradle.api.internal.tasks.execution.EventFiringTaskExecuter
import org.gradle.api.internal.tasks.execution.CatchExceptionTaskExecuter
import org.gradle.api.internal.tasks.execution.ExecuteAtMostOnceTaskExecuter
import org.gradle.api.internal.tasks.execution.SkipOnlyIfTaskExecuter
import java.lang.reflect.Field

class Wrapper {
    static final class ReflectHelper {
        public static Object getFieldObj(Object obj, String fieldName) {
            try {
                def fieldDelegate = obj.getClass().getDeclaredField(fieldName)
                fieldDelegate.setAccessible(true)
                def delegate = fieldDelegate.get(obj)
                return delegate
            } catch (Exception e) {

            }
            return null
        }

        public static Object setFieldObj(Object instance, String fieldName, Object obj) {
            def fieldDelegate = instance.getClass().getDeclaredField(fieldName)
            fieldDelegate.setAccessible(true)
            def delegate = fieldDelegate.set(instance, obj)
            return delegate
        }

        public static void printFieldValues(Object instance) {
            Field[] fields = instance.getClass().declaredFields
            for (field in fields) {
                field.setAccessible(true)
                Object value = field.get(instance)
                println("filed:" + field + " value:" + value)
            }
        }

        public static void replaceExecuteAtMostOnceTaskExecuter(AbstractTask task) {
            def eventFiringTaskExecuter = task.getExecuter() as EventFiringTaskExecuter
            println eventFiringTaskExecuter
            def catchExceptionTaskExecuter = getFieldObj(eventFiringTaskExecuter, "delegate") as CatchExceptionTaskExecuter
            println catchExceptionTaskExecuter
            def delegate2 = getFieldObj(catchExceptionTaskExecuter, "delegate")
            println delegate2
            if (delegate2 instanceof ExecuteAtMostOnceTaskExecuter) {
                def skipOnlyIftaskExecuter = getFieldObj(delegate2, "executer") as SkipOnlyIfTaskExecuter
                println skipOnlyIftaskExecuter
                ExecuteRepeatedlyTaskExecuter executeRepeatedlyTaskExecuter = new ExecuteRepeatedlyTaskExecuter(skipOnlyIftaskExecuter)
                setFieldObj(catchExceptionTaskExecuter, "delegate", executeRepeatedlyTaskExecuter)
            }
        }
    }

    /**
     * 主要用来替换掉ExecuteAtMostOnceTaskExecuter
     */
    public static class ExecuteRepeatedlyTaskExecuter implements TaskExecuter {
        private static final Logger LOGGER = Logging.getLogger(ExecuteRepeatedlyTaskExecuter.class);
        private final TaskExecuter executer;

        public ExecuteRepeatedlyTaskExecuter(TaskExecuter executer) {
            this.executer = executer;
        }

        @Override
        public void execute(TaskInternal task, TaskStateInternal state, TaskExecutionContext context) {
            try {
                this.executer.execute(task, state, context)
            } catch (Exception e) {
                throw new RuntimeException(e)
            }
        }
    }

    void printlnExecuter(TaskExecuter executer) {
        Object delegate = ReflectHelper.getFieldObj(executer, "delegate")
        Object childDelegate = null
        if (delegate == null) {
            Object executerChild = ReflectHelper.getFieldObj(executer, "executer")
            if (executerChild == null) {
                return
            }
            childDelegate = executerChild
        } else {
            childDelegate = delegate
        }
        println ">>> " + executer + " child: " + childDelegate
        if (childDelegate instanceof TaskExecuter) {
            printlnExecuter(childDelegate)
        }
    }

/**
 * 用来自定义一个任务
 */
    public static class DefaultTaskWrapper extends DefaultTask {
        private DefaultTask originalTask;

        @javax.inject.Inject
        public DefaultTaskWrapper(DefaultTask originalTask) {
            this.originalTask = originalTask
        }
    }

    public static final class ActionWrapper implements Action<Task> {
        private DefaultTask originalTask;

        public ActionWrapper(DefaultTask originalTask) {
            this.originalTask = originalTask
            ReflectHelper.replaceExecuteAtMostOnceTaskExecuter(originalTask)
        }

        @Override
        void execute(Task task) {
            println("$this.${this.originalTask} : execute")
            this.originalTask.execute()
        }
    }
}


task startBuild {
    println "------------------------------------------ startBuild:"
    doFirst {
        println "------------------------------------------ startBuild: start"
    }
    doLast {
        println "------------------------------------------ startBuild: end"
    }
}
task startTask {
    doFirst {
        println "start startTask dependsOn: $dependsOn"
    }
    doLast {
        println ">> end startTask"
    }
}

task createTask {
    println "this is test createTask...."

    doFirst {

    }

    2.times {
        println "start create task: testRunTask${it}"

//        def task = tasks.create(name: "testRunTask__${it}", type: DefaultTaskWrapper.class, constructorArgs: [startBuild], action: new TaskAction(startBuild)) << {
//            println "----- testRunTask${it}:" + dependsOn
//        }

        def task = tasks.create(name: "testRunTask__${it}", action: new Wrapper.ActionWrapper(startBuild)) << {
            println "----- testRunTask${it}:" + dependsOn
        }

//        def task = tasks.create(name: "testRunTask__${it}", dependsOn: startBuild) << {
//            println "----- testRunTask${it}:" + dependsOn
//        }

//        DefaultTaskWrapper abc = new DefaultTaskWrapper("testRunTaskDependsOn${it}", startBuild)
//        def task = tasks.create(name: "testRunTask${it}", dependsOn: startBuild) << {
//            println ">>> testRunTask${it}:" + dependsOn
//        }
        startTask.dependsOn task
    }
    finalizedBy startTask
}


```

gradle运行时,会对project内所有task进行归类排序去重, 所以如果想重复运行某一个任务必须创建唯一name的task, 参考上面代码中的tasks.create.
task startBuild是想重复运行的任务, 通过Wrapper.ActionWrapper包裹和ExecuteRepeatedlyTaskExecuter替换ExecuteAtMostOnceTaskExecuter来达到效果
系统使用的是ExecuteAtMostOnceTaskExecuter对任务进行处理, 以下是ExecuteAtMostOnceTaskExecuter源码, if (!state.getExecuted())判断是否已经执行过.
```java
public class ExecuteAtMostOnceTaskExecuter implements TaskExecuter {
    private static final Logger LOGGER = Logging.getLogger(ExecuteAtMostOnceTaskExecuter.class);
    private final TaskExecuter executer;

    public ExecuteAtMostOnceTaskExecuter(TaskExecuter executer) {
        this.executer = executer;
    }

    public void execute(TaskInternal task, TaskStateInternal state, TaskExecutionContext context) {
        if (!state.getExecuted()) {
            LOGGER.debug("Starting to execute {}", task);

            try {
                this.executer.execute(task, state, context);
            } finally {
                LOGGER.debug("Finished executing {}", task);
            }

        }
    }
}
```

### 工程运行结果: 
```gradle
➜  LearnGradle ./gradlew :createTask


> Configure project :
------------------------------------------ startBuild:
this is test createTask....
start create task: testRunTask0
org.gradle.api.internal.tasks.execution.EventFiringTaskExecuter@127e23c0
org.gradle.api.internal.tasks.execution.CatchExceptionTaskExecuter@7ba0909d
org.gradle.api.internal.tasks.execution.ExecuteAtMostOnceTaskExecuter@6fa00c1e
org.gradle.api.internal.tasks.execution.SkipOnlyIfTaskExecuter@306127b6
start create task: testRunTask1
org.gradle.api.internal.tasks.execution.EventFiringTaskExecuter@127e23c0
org.gradle.api.internal.tasks.execution.CatchExceptionTaskExecuter@7ba0909d
Wrapper$ExecuteRepeatedlyTaskExecuter@3ac38b4b

> Task :startBuild
------------------------------------------ startBuild: start
------------------------------------------ startBuild: end

> Task :testRunTask__0
Wrapper$ActionWrapper@19abe28a.task ':startBuild' : execute
----- testRunTasktask ':testRunTask__0':[]

> Task :startBuild
------------------------------------------ startBuild: start
------------------------------------------ startBuild: end

> Task :testRunTask__1
Wrapper$ActionWrapper@736b87ef.task ':startBuild' : execute
----- testRunTasktask ':testRunTask__1':[]

> Task :startTask
start startTask dependsOn: [task ':testRunTask__0', task ':testRunTask__1']
>> end startTask

Deprecated Gradle features were used in this build, making it incompatible with Gradle 5.0.
See https://docs.gradle.org/4.8.1/userguide/command_line_interface.html#sec:command_line_warnings
```


