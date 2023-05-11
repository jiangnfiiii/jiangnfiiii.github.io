---
title: skywalking-采样率
date: 2023-05-09 16:51:40
categories: [学习笔记, 链路追踪]
tags:
	- java
	- skywalking
---
思路： 设置默认样本数(NUMBER_OF_SAMPLES=1000), 这里先是以1000 为采样样本总数，然后根据设置采样的采样率设置随机采样点样本，每1000次都按这个采样的样点进行采样

1. 添加默认采样样本总体上变量和采样率开关变量，修改内容 SamplingService类45-47行。

   ```
   //默认采样样本总体上
   private static final int NUMBER_OF_SAMPLES = 1000;
   private volatile boolean proportion = false;
   ```

2. 判断配置是否开启采样率开关，修改内容 SamplingService类77-83行。

   ```
   } else if(Config.Agent.SAMPLE_N_PER_PROPORTION > 0) {
       proportion = true;
       this.resetProportionFactor();
       initCollectionPoints();
       logger.debug(
               "Agent sampling mechanism started. Sample {} traces in 3 seconds.", Config.Agent.SAMPLE_N_PER_PROPORTION);
   }
   ```

3. 判断是否采样方法添加采样率判断方法，修改内容 SamplingService类108-118行。

   ```
   }else if (proportion) {
       int factor = samplingFactorHolder.get();
       if(map.get(factor)) {
           return samplingFactorHolder.compareAndSet(factor, factor + 1);
       }
       //判断是否已经次数已经超过样本数，如果是重置采样次数（采样次数并非样本单位数）
       if(factor==NUMBER_OF_SAMPLES) {
           resetProportionFactor();
       }
   }
   ```



4. 添加重置样本采样次数方法，修改类 SamplingService类135-137行

   ```
   private void resetProportionFactor() {
       samplingFactorHolder = new AtomicInteger(0);
   }
   ```

   

5. 添加初始化样本单位数的采集点

   ```
   public void initCollectionPoints(){
       map = new HashMap<>();
       int count=0;
       int[] result = new int[Config.Agent.SAMPLE_N_PER_PROPORTION];
       while (count< Config.Agent.SAMPLE_N_PER_PROPORTION) {
           int num = (int) (Math.random() * (NUMBER_OF_SAMPLES-1))+1;
           boolean flag =true;
           for (int index= 0; index < Config.Agent.SAMPLE_N_PER_PROPORTION; index++) {
               if(num == result[index]){
                   flag=false;
                   break;
               }
           }
           if(flag){
               map.put(count, true);
               count++;
           }
       }
   }
   ```



### SamplingService.java 

```java
/*
 * Licensed to the Apache Software Foundation (ASF) under one or more
 * contributor license agreements.  See the NOTICE file distributed with
 * this work for additional information regarding copyright ownership.
 * The ASF licenses this file to You under the Apache License, Version 2.0
 * (the "License"); you may not use this file except in compliance with
 * the License.  You may obtain a copy of the License at
 *
 *     http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 *
 */

package org.apache.skywalking.apm.agent.core.sampling;

import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

import org.apache.skywalking.apm.agent.core.boot.BootService;
import org.apache.skywalking.apm.agent.core.boot.DefaultImplementor;
import org.apache.skywalking.apm.agent.core.boot.DefaultNamedThreadFactory;
import org.apache.skywalking.apm.agent.core.conf.Config;
import org.apache.skywalking.apm.agent.core.context.trace.TraceSegment;
import org.apache.skywalking.apm.agent.core.logging.api.ILog;
import org.apache.skywalking.apm.agent.core.logging.api.LogManager;
import org.apache.skywalking.apm.util.RunnableWithExceptionProtection;

/**
 * The <code>SamplingService</code> take charge of how to sample the {@link TraceSegment}. Every {@link TraceSegment}s
 * have been traced, but, considering CPU cost of serialization/deserialization, and network bandwidth, the agent do NOT
 * send all of them to collector, if SAMPLING is on.
 * <p>
 * By default, SAMPLING is on, and  {@link Config.Agent#SAMPLE_N_PER_3_SECS }
 */
@DefaultImplementor
public class SamplingService implements BootService {
    private static final ILog logger = LogManager.getLogger(SamplingService.class);
    //默认采样样本总体上
    private static final int NUMBER_OF_SAMPLES = 1000;
    private volatile boolean per3secs = false;
    private volatile boolean proportion = false;
    private volatile AtomicInteger samplingFactorHolder;
    private volatile Map<Integer, Boolean> map = new HashMap<>();
    private volatile ScheduledFuture<?> scheduledFuture;

    @Override
    public void prepare() {

    }

    @Override
    public void boot() {
        if (scheduledFuture != null) {
            /*
             * If {@link #boot()} invokes twice, mostly in test cases,
             * cancel the old one.
             */
            scheduledFuture.cancel(true);
        }
        if (Config.Agent.SAMPLE_N_PER_3_SECS > 0) {
            per3secs = true;
            this.resetSamplingFactor();
            ScheduledExecutorService service = Executors.newSingleThreadScheduledExecutor(
                new DefaultNamedThreadFactory("SamplingService"));
            scheduledFuture = service.scheduleAtFixedRate(new RunnableWithExceptionProtection(
                this::resetSamplingFactor, t -> logger.error("unexpected exception.", t)), 0, 3, TimeUnit.SECONDS);
            logger.debug(
                "Agent sampling mechanism started. Sample {} traces in 3 seconds.", Config.Agent.SAMPLE_N_PER_3_SECS);
        } else if(Config.Agent.SAMPLE_N_PER_PROPORTION > 0) {
            proportion = true;
            this.resetProportionFactor();
            initCollectionPoints();
            logger.debug(
                    "Agent sampling mechanism started. Sample {} traces in 3 seconds.", Config.Agent.SAMPLE_N_PER_PROPORTION);
        }
    }

    @Override
    public void onComplete() {

    }

    @Override
    public void shutdown() {
        if (scheduledFuture != null) {
            scheduledFuture.cancel(true);
        }
    }

    /**
     * @return true, if sampling mechanism is on, and getDefault the sampling factor successfully.
     */
    public boolean trySampling() {
        if (per3secs) {
            int factor = samplingFactorHolder.get();
            if (factor < Config.Agent.SAMPLE_N_PER_3_SECS) {
                return samplingFactorHolder.compareAndSet(factor, factor + 1);
            } else {
                return false;
            }
        }else if (proportion) {
            int factor = samplingFactorHolder.get();
            if(map.get(factor)) {
                return samplingFactorHolder.compareAndSet(factor, factor + 1);
            }
            //判断是否已经次数已经超过样本数，如果是重置采样次数（采样次数并非样本单位数）
            if(factor==NUMBER_OF_SAMPLES) {
                resetProportionFactor();
            }
        }
        return true;
    }

    /**
     * Increase the sampling factor by force, to avoid sampling too many traces. If many distributed traces require
     * sampled, the trace beginning at local, has less chance to be sampled.
     */
    public void forceSampled() {
        if (per3secs||proportion) {
            samplingFactorHolder.incrementAndGet();
        }
    }

    private void resetSamplingFactor() {
        samplingFactorHolder = new AtomicInteger(0);
    }
    private void resetProportionFactor() {
        samplingFactorHolder = new AtomicInteger(0);
    }

    /**
     * 初始化样本单位数的采集点
     */
    public void initCollectionPoints(){
        map = new HashMap<>();
        int count=0;
        int[] result = new int[Config.Agent.SAMPLE_N_PER_PROPORTION];
        while (count< Config.Agent.SAMPLE_N_PER_PROPORTION) {
            int num = (int) (Math.random() * (NUMBER_OF_SAMPLES-1))+1;
            boolean flag =true;
            for (int index= 0; index < Config.Agent.SAMPLE_N_PER_PROPORTION; index++) {
                if(num == result[index]){
                    flag=false;
                    break;
                }
            }
            if(flag){
                map.put(count, true);
                count++;
            }
        }
    }


}

```



### skywalking 调用链开关设置 

思路：在jvm环境变量设置是否开启调用链开关，然后在其代码增强地方加if-else判断。

1. 修改内容 InstMethodsInter类77-89行 

   ```java
   //回去jvm 环境变量值sw-collection,并设置默认值为1
   String swCollection = System.getProperty("sw-collection", "1");
   logger.info("skyWalking-enabled  [{}].", swCollection);
   try {
       //判断jvm 设置的环境变量值是否为1， 如果1: 执行调用链设置，如果不是这跳过
       if(swCollection.equals("1"))    {
           logger.info("skyWalking interceptor beforeMethod");
           interceptor.beforeMethod(targetObject, method, allArguments, method.getParameterTypes(), result);
       }else {
           logger.info("skyWalking not interceptor beforeMethod");
       }
   } catch (Throwable t) {
       logger.error(t, "class[{}] before method[{}] intercept failure", obj.getClass(), method.getName());
   }
   ```

   

2. 添加方法调用后置处理内容 InstMethodsInter类107-113行

   

   ```java
   try {
       //判断jvm 设置的环境变量值是否为1， 如果1: 执行调用链设置，如果不是这跳过
       if(swCollection.equals("1")) {
           ret = interceptor.afterMethod(targetObject, method, allArguments, method.getParameterTypes(), ret);
       }
   } catch (Throwable t) {
       logger.error(t, "class[{}] after method[{}] intercept failure", obj.getClass(), method.getName());
   }
   ```







### InstMethodsInter.java

```java
/*
 * Licensed to the Apache Software Foundation (ASF) under one or more
 * contributor license agreements.  See the NOTICE file distributed with
 * this work for additional information regarding copyright ownership.
 * The ASF licenses this file to You under the Apache License, Version 2.0
 * (the "License"); you may not use this file except in compliance with
 * the License.  You may obtain a copy of the License at
 *
 *     http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 *
 */

package org.apache.skywalking.apm.agent.core.plugin.interceptor.enhance;

import java.lang.reflect.Method;
import java.util.concurrent.Callable;
import net.bytebuddy.implementation.bind.annotation.AllArguments;
import net.bytebuddy.implementation.bind.annotation.Origin;
import net.bytebuddy.implementation.bind.annotation.RuntimeType;
import net.bytebuddy.implementation.bind.annotation.SuperCall;
import net.bytebuddy.implementation.bind.annotation.This;
import org.apache.skywalking.apm.agent.core.logging.api.LogManager;
import org.apache.skywalking.apm.agent.core.plugin.PluginException;
import org.apache.skywalking.apm.agent.core.plugin.loader.InterceptorInstanceLoader;
import org.apache.skywalking.apm.agent.core.logging.api.ILog;
import org.apache.skywalking.apm.util.StringUtil;

/**
 * The actual byte-buddy's interceptor to intercept class instance methods. In this class, it provide a bridge between
 * byte-buddy and sky-walking plugin.
 */
public class InstMethodsInter {
    private static final ILog logger = LogManager.getLogger(InstMethodsInter.class);

    /**
     * An {@link InstanceMethodsAroundInterceptor} This name should only stay in {@link String}, the real {@link Class}
     * type will trigger classloader failure. If you want to know more, please check on books about Classloader or
     * Classloader appointment mechanism.
     */
    private InstanceMethodsAroundInterceptor interceptor;

    /**
     * @param instanceMethodsAroundInterceptorClassName class full name.
     */
    public InstMethodsInter(String instanceMethodsAroundInterceptorClassName, ClassLoader classLoader) {
        try {
            interceptor = InterceptorInstanceLoader.load(instanceMethodsAroundInterceptorClassName, classLoader);
        } catch (Throwable t) {
            throw new PluginException("Can't create InstanceMethodsAroundInterceptor.", t);
        }
    }

    /**
     * Intercept the target instance method.
     *
     * @param obj          target class instance.
     * @param allArguments all method arguments
     * @param method       method description.
     * @param zuper        the origin call ref.
     * @return the return value of target instance method.
     * @throws Exception only throw exception because of zuper.call() or unexpected exception in sky-walking ( This is a
     *                   bug, if anything triggers this condition ).
     */
    @RuntimeType
    public Object intercept(@This Object obj, @AllArguments Object[] allArguments, @SuperCall Callable<?> zuper,
        @Origin Method method) throws Throwable {
        EnhancedInstance targetObject = (EnhancedInstance) obj;

        MethodInterceptResult result = new MethodInterceptResult();
        //回去jvm 环境变量值sw-collection,并设置默认值为1
        String swCollection = System.getProperty("sw-collection", "1");
        logger.info("skyWalking-enabled  [{}].", swCollection);
        try {
            //判断jvm 设置的环境变量值是否为1， 如果1: 执行调用链设置，如果不是这跳过
            if(swCollection.equals("1"))    {
                logger.info("skyWalking interceptor beforeMethod");
                interceptor.beforeMethod(targetObject, method, allArguments, method.getParameterTypes(), result);
            }else {
                logger.info("skyWalking not interceptor beforeMethod");
            }
        } catch (Throwable t) {
            logger.error(t, "class[{}] before method[{}] intercept failure", obj.getClass(), method.getName());
        }

        Object ret = null;
        try {
            if (!result.isContinue()) {
                ret = result._ret();
            } else {
                ret = zuper.call();
            }
        } catch (Throwable t) {
            try {
                interceptor.handleMethodException(targetObject, method, allArguments, method.getParameterTypes(), t);
            } catch (Throwable t2) {
                logger.error(t2, "class[{}] handle method[{}] exception failure", obj.getClass(), method.getName());
            }
            throw t;
        } finally {
            try {
                //判断jvm 设置的环境变量值是否为1， 如果1: 执行调用链设置，如果不是这跳过
                if(swCollection.equals("1")) {
                    ret = interceptor.afterMethod(targetObject, method, allArguments, method.getParameterTypes(), ret);
                }
            } catch (Throwable t) {
                logger.error(t, "class[{}] after method[{}] intercept failure", obj.getClass(), method.getName());
            }
        }
        return ret;
    }



}

```





### 动态配置使用说明 

#### 动态开关

 设置jvm变量 sw-collection 变量值 0：关闭，1：开启（默认值为0），使用方法在启动-Dsw-collection=0

如果在运行中需要变更采样开关 调用 System.setProperty(“sw-collection”, “0”) 方法设置调用链开关;

#### 采样率设置

 设置jvm变量sample-per-proportion 变量值为1-100整数，使用方法在启动-Dsample-per-proportion =60，

如果在运行中需要变更采样开关 调用 System.setProperty(“sample-per-proportion ”, “50”) 方法设置调用链开关;



​	