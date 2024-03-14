# Jmh

记得之前写过一个这样的文档,后面找不着了,现在又来写了. orz

---

### 1. 基础使用

#### 1.1 依赖

```xml
<dependency>
    <groupId>org.openjdk.jmh</groupId>
    <artifactId>jmh-core</artifactId>
    <version>1.23</version>
</dependency>
<dependency>
    <groupId>org.openjdk.jmh</groupId>
    <artifactId>jmh-generator-annprocess</artifactId>
    <version>1.23</version>
    <scope>provided</scope>
</dependency>
```

#### 1.2 测试

```java
package com.dslyy.fd.utils;

import org.openjdk.jmh.annotations.Benchmark;
import org.openjdk.jmh.annotations.BenchmarkMode;
import org.openjdk.jmh.annotations.Fork;
import org.openjdk.jmh.annotations.Measurement;
import org.openjdk.jmh.annotations.Mode;
import org.openjdk.jmh.annotations.OutputTimeUnit;
import org.openjdk.jmh.annotations.Scope;
import org.openjdk.jmh.annotations.State;
import org.openjdk.jmh.annotations.Warmup;
import org.openjdk.jmh.runner.Runner;
import org.openjdk.jmh.runner.RunnerException;
import org.openjdk.jmh.runner.options.Options;
import org.openjdk.jmh.runner.options.OptionsBuilder;

import java.util.concurrent.TimeUnit;

/**
 * @author huapeng.huang
 * @version V1.0
 * @since 2024-03-09 16:03
 */
@BenchmarkMode(Mode.AverageTime)
@State(Scope.Thread)
@Fork(1)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@Warmup(iterations = 3)
@Measurement(iterations = 5)
public class JmhUnit {
    String string = "";
    StringBuilder stringBuilder = new StringBuilder();

    @Benchmark
    public String stringAdd() {
        for (int i = 0; i < 1000; i++) {
            string = string + i;
        }
        return string;
    }

    @Benchmark
    public String stringBuilderAppend() {

        for (int i = 0; i < 1000; i++) {
            stringBuilder.append(i);
        }
        return stringBuilder.toString();
    }

    public static void main(String[] args) throws RunnerException {
        Options opt = new OptionsBuilder()
                .include(JmhUnit.class.getSimpleName())
                .build();
        new Runner(opt).run();
    }
}

```

```
# JMH version: 1.23
# VM version: JDK 1.8.0_201, Java HotSpot(TM) 64-Bit Server VM, 25.201-b09
# VM invoker: /Library/Java/JavaVirtualMachines/jdk1.8.0_201.jdk/Contents/Home/jre/bin/java
# VM options: -javaagent:/Applications/IntelliJ IDEA.app/Contents/lib/idea_rt.jar=56314:/Applications/IntelliJ IDEA.app/Contents/bin -Dfile.encoding=UTF-8
# Warmup: 3 iterations, 10 s each
# Measurement: 5 iterations, 10 s each
# Timeout: 10 min per iteration
# Threads: 1 thread, will synchronize iterations
# Benchmark mode: Average time, time/op
# Benchmark: com.poorman.utils.JmhUnit.stringAdd

# Run progress: 0.00% complete, ETA 00:02:40
# Fork: 1 of 1
# Warmup Iteration   1: 0.445 ms/op
# Warmup Iteration   2: 0.357 ms/op
# Warmup Iteration   3: 0.353 ms/op
Iteration   1: 0.328 ms/op
Iteration   2: 0.345 ms/op
Iteration   3: 0.335 ms/op
Iteration   4: 0.450 ms/op
Iteration   5: 0.421 ms/op


Result "com.poorman.utils.JmhUnit.stringAdd":
  0.376 ±(99.9%) 0.215 ms/op [Average]
  (min, avg, max) = (0.328, 0.376, 0.450), stdev = 0.056
  CI (99.9%): [0.161, 0.591] (assumes normal distribution)


# JMH version: 1.23
# VM version: JDK 1.8.0_201, Java HotSpot(TM) 64-Bit Server VM, 25.201-b09
# VM invoker: /Library/Java/JavaVirtualMachines/jdk1.8.0_201.jdk/Contents/Home/jre/bin/java
# VM options: -javaagent:/Applications/IntelliJ IDEA.app/Contents/lib/idea_rt.jar=56314:/Applications/IntelliJ IDEA.app/Contents/bin -Dfile.encoding=UTF-8
# Warmup: 3 iterations, 10 s each
# Measurement: 5 iterations, 10 s each
# Timeout: 10 min per iteration
# Threads: 1 thread, will synchronize iterations
# Benchmark mode: Average time, time/op
# Benchmark: com.poorman.utils.JmhUnit.stringBuilderAppend

# Run progress: 50.00% complete, ETA 00:01:22
# Fork: 1 of 1
# Warmup Iteration   1: 0.016 ms/op
# Warmup Iteration   2: 0.014 ms/op
# Warmup Iteration   3: 0.014 ms/op
Iteration   1: 0.014 ms/op
Iteration   2: 0.015 ms/op
Iteration   3: 0.015 ms/op
Iteration   4: 0.018 ms/op
Iteration   5: 0.038 ms/op


Result "com.poorman.utils.JmhUnit.stringBuilderAppend":
  0.020 ±(99.9%) 0.039 ms/op [Average]
  (min, avg, max) = (0.014, 0.020, 0.038), stdev = 0.010
  CI (99.9%): [≈ 0, 0.059] (assumes normal distribution)


# Run complete. Total time: 00:02:43

REMEMBER: The numbers below are just data. To gain reusable insights, you need to follow up on
why the numbers are the way they are. Use profilers (see -prof, -lprof), design factorial
experiments, perform baseline and negative tests that provide experimental control, make sure
the benchmarking environment is safe on JVM/OS/HW level, ask for reviews from the domain experts.
Do not assume the numbers tell you what you want them to tell.

Benchmark                    Mode  Cnt  Score   Error  Units
JmhUnit.stringAdd            avgt    5  0.376 ± 0.215  ms/op
JmhUnit.stringBuilderAppend  avgt    5  0.020 ± 0.039  ms/op
```

---

### 2. 基础知识

Q: 在上面代码里面使用到的注解分别是干啥的呀?

A: 下面我们来看看相关的注解是干啥的.

#### 2.1 BenchmarkMode

配置 Mode 选项，可用于类或者方法上，这个注解的 value 是一个数组，可以把几种 Mode 集合在一起执行，如：`@BenchmarkMode({Mode.SampleTime, Mode.AverageTime})`，还可以设置为 Mode.All，即全部执行一遍。

| 类型           | 备注                                                             |
| -------------- | ---------------------------------------------------------------- |
| Throughput     | 整体吞吐量，每秒执行了多少次调用，单位为 ops/time                |
| AverageTime    | 用的平均时间，每次操作的平均时间，单位为 time/op                 |
| SampleTime     | 随机取样，最后输出取样结果的分布                                 |
| SingleShotTime | 只运行一次，往往同时把 Warmup 次数设为 0，用于测试冷启动时的性能 |
| All            | 上面的所有模式都执行一次                                         |

Q: 这个怎么看确定哪个执行方法性能比较好?

A: 如果<u>采用的是 AverageTime,则 score 的分值越低越好,如果是 Throughput 则越高越好.</u>

#### 2.2 State

通过 State 可以指定一个对象的作用范围，JMH 根据 scope 来进行实例化和共享操作。@State 可以被继承使用，如果父类定义了该注解，子类则无需定义。由于 JMH 允许多线程同时执行测试，不同的选项含义如下：

| 类型            | 备注                                                         |
| --------------- | ------------------------------------------------------------ |
| Scope.Benchmark | 所有测试线程共享一个实例，测试有状态实例在多线程共享下的性能 |
| Scope.Group     | 同一个线程在同一个 group 里共享实例                          |
| Scope.Thread    | 默认的 State，每个测试线程分配一个实例                       |

#### 2.3 OutputTimeUnit

为统计结果的时间单位，可用于类或者方法注解

#### 2.4 Warmup

预热所需要配置的一些基本测试参数，可用于类或者方法上。一般前几次进行程序测试的时候都会比较慢，所以要让程序进行几轮预热，保证测试的准确性。参数如下所示：

| 类型       | 备注                             |
| ---------- | -------------------------------- |
| iterations | 预热的次数                       |
| time       | 每次预热的时间                   |
| timeUnit   | 时间的单位，默认秒               |
| batchSize  | 批处理大小，每次操作调用几次方法 |

Q: 为什么需要预热？

A: 因为 JVM 的 JIT 机制的存在，如果某个函数被调用多次之后，JVM 会尝试将其编译为机器码，从而提高执行速度，所以为了让 benchmark 的结果更加接近真实情况就需要进行预热。

#### 2.5 Measurement

实际调用方法所需要配置的一些基本测试参数，可用于类或者方法上，参数和 @Warmup 相同。

#### 2.6 Thread

@Threads 每个进程中的测试线程，可用于类或者方法上。

#### 2.7 Fork

@Fork 进行 fork 的次数，可用于类或者方法上。如果 fork 数是 2 的话，则 JMH 会 fork 出两个进程来进行测试。

#### 2.8 Param

@Param 指定某项参数的多种情况，特别适合用来测试一个函数在不同的参数输入的情况下的性能，只能作用在字段上，使用该注解必须定义 @State 注解。
