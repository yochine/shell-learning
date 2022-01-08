在网上已经有跟多Bean的生命周期的博客，但是很多都是基于比较老的版本了，最近吧整个流程化成了一个流程图。待会儿使用流程图，说明以及代码的形式来说明整个声明周期的流程。**注意因为代码比较多，这里的流程图只画出了大概的流程，具体的可以深入代码**

## **一、获取Bean**

![](https://imgkr2.cn-bj.ufileos.com/1b960367-96f0-4e55-89a3-be0df45899af.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=4zu7v0uDZwTEakkJXGXgvCpz33k%253D&Expires=1604287594)


### **第一阶段获取Bean**

这里的流程图的入口在 `AbstractBeanFactory`类的 `doGetBean`方法，这里可以配合前面的 getBean方法分析文章进行阅读。主要流程就是**1、**先处理Bean 的名称，因为如果以“\&”开头的Bean名称表示获取的是对应的FactoryBean对象；  
**2、**从缓存中获取单例Bean，有则进一步判断这个Bean是不是在创建中，如果是的就等待创建完毕，否则直接返回这个Bean对象  
**3、**如果不存在单例Bean缓存，则先进行循环依赖的解析  
**4、**解析完毕之后先获取父类BeanFactory，获取到了则调用父类的getBean方法，不存在则先合并然后创建Bean

## **二、创建Bean**

### **2.1 创建Bean之前**

![](https://imgkr2.cn-bj.ufileos.com/65da1359-ead7-49d6-8971-40fb9ee6150b.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=lKGUfX4VnGHd0axPkMPXBfguN6o%253D&Expires=1604287650)


### **在真正创建Bean之前逻辑**

这个流程图对应的代码在 `AbstractAutowireCapableBeanFactory`类的 `createBean`方法中。**1、**这里会先获取 `RootBeanDefinition`对象中的Class对象并确保已经关联了要创建的Bean的Class 。整编：微信公众号，搜云库技术团队，ID：souyunku  
**2、**这里会检查3个条件（1）Bean的属性中的 `beforeInstantiationResolved`字段是否为true，默认是false。  
（2）Bean是原生的Bean  
（3）Bean的 `hasInstantiationAwareBeanPostProcessors`属性为true，这个属性在Spring准备刷新容器钱转杯BeanPostProcessors的时候会设置，如果当前Bean实现了 `InstantiationAwareBeanPostProcessor`则这个就会是true。当三个条件都存在的时候，就会调用实现的 `InstantiationAwareBeanPostProcessor`接口的 `postProcessBeforeInstantiation`方法，然后获取返回的Bean，如果返回的Bean不是null还会调用实现的 `BeanPostProcessor`接口的 `postProcessAfterInitialization`方法，这里用代码说明

1.  `protected  Object resolveBeforeInstantiation(String beanName,  RootBeanDefinition mbd)  {`

2.   `Object bean =  null;`

3.   `//条件1`

4.   `if  (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved))  {`

5.   `//条件2跟条件3`

6.   `if  (!mbd.isSynthetic()  && hasInstantiationAwareBeanPostProcessors())  {`

7.   `Class<?> targetType = determineTargetType(beanName, mbd);`

8.   `if  (targetType !=  null)  {`

9.   `//调用实现的postProcessBeforeInstantiation方法`

10.   `bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);`

11.   `if  (bean !=  null)  {`

12.  `//调用实现的postProcessAfterInitialization方法`

13.   `bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);`

14.   `}`

15.   `}`

16.   `}`

17.   `//不满足2或者3的时候就会设置为false`

18.   `mbd.beforeInstantiationResolved =  (bean !=  null);`

19.   `}`

20.   `return bean;`

21.   `}`

**1、**如果上面3个条件其中一个不满足就不会调用实现的方法。默认这里都不会调用的这些 `BeanPostProcessors`的实现方法。然后继续执行后面的 `doCreateBean`方法。

## **2.1 真正的创建Bean，doCreateBean**


![](https://imgkr2.cn-bj.ufileos.com/38a5ff89-ed71-4704-9d25-02f8a92a22e6.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=KdimcuArhTRlyGafxIVZBDQ3z48%253D&Expires=1604287734)


### **doCreateBean方法逻辑**

这个代码的实现还是在 `AbstractAutowireCapableBeanFactory`方法中。流程是**1、**先检查 `instanceWrapper`变量是不是null，这里一般是null，除非当前正在创建的Bean在 `factoryBeanInstanceCache`中存在这个是保存还没创建完成的FactoryBean的集合。  
**2、**调用createBeanInstance方法实例化Bean，这个方法在后面会讲解  
**3、**如果当前 `RootBeanDefinition`对象还没有调用过实现了的 `MergedBeanDefinitionPostProcessor`接口的方法，则会进行调用 。整编：微信公众号，搜云库技术团队，ID：souyunku  
**4、** 当满足以下三点  
（1）是单例Bean  
（2）尝试解析bean之间的循环引用  
（3）bean目前正在创建中  
则会进一步检查是否实现了 `SmartInstantiationAwareBeanPostProcessor`接口如果实现了则调用是实现的 `getEarlyBeanReference`方法 5、 调用 `populateBean`方法进行属性填充，这里后面会讲解 6、 调用 `initializeBean`方法对Bean进行初始化，这里后面会讲解

### **2.1.1 实例化Bean，createBeanInstance**

![](https://imgkr2.cn-bj.ufileos.com/9c641c8d-9296-48d5-911b-3a55405ab60a.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=mwFnC2PO7viOM5YDQI4wfu3mvZw%253D&Expires=1604287782)


### **实例化Bean**

这里的逻辑稍微有一点复杂，这个流程图已经是简化过后的了。简要根据代码说明一下流程

1.   `protected  BeanWrapper createBeanInstance(String beanName,  RootBeanDefinition mbd,  @Nullable  Object[] args)  {`

2.   `//步骤1`

3.   `Class<?> beanClass = resolveBeanClass(mbd, beanName);`

4.    

5.   `if  (beanClass !=  null  &&  !Modifier.isPublic(beanClass.getModifiers())  &&  !mbd.isNonPublicAccessAllowed())  {`

6.   `throw  new  BeanCreationException(mbd.getResourceDescription(), beanName,`

7.   `"Bean class isn't public, and non-public access not allowed: "  + beanClass.getName());`

8.   `}`

9.   `//步骤2`

10.   `Supplier<?> instanceSupplier = mbd.getInstanceSupplier();`

11.   `if  (instanceSupplier !=  null)  {`

12.   `return obtainFromSupplier(instanceSupplier, beanName);`

13.   `}`

14.   `//步骤3`

15.   `if  (mbd.getFactoryMethodName()  !=  null)  {`

16.   `return instantiateUsingFactoryMethod(beanName, mbd, args);`

17.   `}`

18.    

19.    

20.   `boolean resolved =  false;`

21.   `boolean autowireNecessary =  false;`

22.   `if  (args ==  null)  {`

23.   `synchronized  (mbd.constructorArgumentLock)  {`

24.   `if  (mbd.resolvedConstructorOrFactoryMethod !=  null)  {`

25.   `resolved =  true;`

26.   `autowireNecessary = mbd.constructorArgumentsResolved;`

27.   `}`

28.   `}`

29.   `}`

30.   `//步骤4.1`

31.   `if  (resolved)  {`

32.    

33.   `if  (autowireNecessary)  {`

34.   `return autowireConstructor(beanName, mbd,  null,  null);`

35.   `}`

36.   `else  {`

37.    

38.   `return instantiateBean(beanName, mbd);`

39.   `}`

40.   `}`

41.    

42.   `//步骤4.2`

43.   `Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);`

44.   `if  (ctors !=  null  || mbd.getResolvedAutowireMode()  == AUTOWIRE_CONSTRUCTOR ||`

45.   `mbd.hasConstructorArgumentValues()  ||  !ObjectUtils.isEmpty(args))  {`

46.   `return autowireConstructor(beanName, mbd, ctors, args);`

47.   `}`

48.    

49.   `//步骤5`

50.   `ctors = mbd.getPreferredConstructors();`

51.   `if  (ctors !=  null)  {`

52.   `return autowireConstructor(beanName, mbd, ctors,  null);`

53.   `}`

54.    

55.    

56.   `return instantiateBean(beanName, mbd);`

57.   `}`

**1、**先检查Class是否已经关联了，并且对应的修饰符是否是public的  
**2、**如果用户定义了Bean实例化的函数，则调用并返回  
**3、**如果当前Bean实现了 `FactoryBean`接口则调用对应的 `FactoryBean`接口的 `getObject`方法  
**4、**根据getBean时候是否传入构造参数进行处理  
**4.1** 如果没有传入构造参数，则检查是否存在已经缓存的无参构造器，有则使用构造器直接创建，没有就会调用 `instantiateBean`方法先获取实例化的策略默认是 `CglibSubclassingInstantiationStrategy`，然后实例化Bean。最后返回  
**4.2** 如果传入了构造参数，则会先检查是否实现了 `SmartInstantiationAwareBeanPostProcessor`接口，如果实现了会调用 `determineCandidateConstructors`获取返回的候选构造器。整编：微信公众号，搜云库技术团队，ID：souyunku  
**4.3** 检查4个条件是否满足一个  
（1）构造器不为null，  
（2）从RootBeanDefinition中获取到的关联的注入方式是构造器注入（没有构造参数就是setter注入，有则是构造器注入）  
（3）含有构造参数  
（4）getBean方法传入构造参数不是空满足其中一个则会调用返回的候选构造器实例化Bean并返回，如果都不满足，则会根据构造参数选则合适的有参构造器然后实例化Bean并返回**5、**如果上面都没有合适的构造器，则直接使用无参构造器创建并返回Bean。

### **2.1.2 填充Bean，populateBean**


![](https://imgkr2.cn-bj.ufileos.com/c59e8d20-52f9-4b1b-8d39-f0c3eef27696.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=Y3eN7bVE5v1FbrYQduzuIDEggYE%253D&Expires=1604287825)


### **填充Bean**

这里还是根据代码来说一下流程

1.   `protected  void populateBean(String beanName,  RootBeanDefinition mbd,  @Nullable  BeanWrapper bw)  {`

2.   `if  (bw ==  null)  {`

3.   `if  (mbd.hasPropertyValues())  {`

4.   `throw  new  BeanCreationException(`

5.   `mbd.getResourceDescription(), beanName,  "Cannot apply property values to null instance");`

6.   `}`

7.   `else  {`

8.   `// Skip property population phase for null instance.`

9.   `return;`

10.   `}`

11.   `}`

12.    

13.    

14.   `boolean continueWithPropertyPopulation =  true;`

15.   `//步骤1`

16.   `if  (!mbd.isSynthetic()  && hasInstantiationAwareBeanPostProcessors())  {`

17.   `for  (BeanPostProcessor bp : getBeanPostProcessors())  {`

18.   `if  (bp instanceof  InstantiationAwareBeanPostProcessor)  {`

19.   `InstantiationAwareBeanPostProcessor ibp =  (InstantiationAwareBeanPostProcessor) bp;`

20.   `if  (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName))  {`

21.   `continueWithPropertyPopulation =  false;`

22.   `break;`

23.   `}`

24.   `}`

25.   `}`

26.   `}`

27.    

28.   `if  (!continueWithPropertyPopulation)  {`

29.   `return;`

30.   `}`

31.  `//步骤2--------------------`

32.   `PropertyValues pvs =  (mbd.hasPropertyValues()  ? mbd.getPropertyValues()  :  null);`

33.    

34.   `int resolvedAutowireMode = mbd.getResolvedAutowireMode();`

35.   `if  (resolvedAutowireMode == AUTOWIRE_BY_NAME || resolvedAutowireMode == AUTOWIRE_BY_TYPE)  {`

36.   `MutablePropertyValues newPvs =  new  MutablePropertyValues(pvs);`

37.   `// Add property values based on autowire by name if applicable.`

38.   `if  (resolvedAutowireMode == AUTOWIRE_BY_NAME)  {`

39.   `autowireByName(beanName, mbd, bw, newPvs);`

40.   `}`

41.   `// Add property values based on autowire by type if applicable.`

42.   `if  (resolvedAutowireMode == AUTOWIRE_BY_TYPE)  {`

43.   `autowireByType(beanName, mbd, bw, newPvs);`

44.   `}`

45.   `pvs = newPvs;`

46.   `}`

47.    

48.   `boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();`

49.   `boolean needsDepCheck =  (mbd.getDependencyCheck()  !=  AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);`

50.    

51.   `PropertyDescriptor[] filteredPds =  null;`

52.  `//步骤3`

53.   `if  (hasInstAwareBpps)  {`

54.   `if  (pvs ==  null)  {`

55.   `pvs = mbd.getPropertyValues();`

56.   `}`

57.   `for  (BeanPostProcessor bp : getBeanPostProcessors())  {`

58.   `if  (bp instanceof  InstantiationAwareBeanPostProcessor)  {`

59.   `InstantiationAwareBeanPostProcessor ibp =  (InstantiationAwareBeanPostProcessor) bp;`

60.   `PropertyValues pvsToUse = ibp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);`

61.   `if  (pvsToUse ==  null)  {`

62.   `if  (filteredPds ==  null)  {`

63.   `filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);`

64.   `}`

65.   `pvsToUse = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);`

66.   `if  (pvsToUse ==  null)  {`

67.   `return;`

68.   `}`

69.   `}`

70.   `pvs = pvsToUse;`

71.   `}`

72.   `}`

73.   `}`

74.   `if  (needsDepCheck)  {`

75.   `if  (filteredPds ==  null)  {`

76.   `filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);`

77.   `}`

78.   `checkDependencies(beanName, mbd, filteredPds, pvs);`

79.   `}`

80.  `//步骤4`

81.   `if  (pvs !=  null)  {`

82.   `applyPropertyValues(beanName, mbd, bw, pvs);`

83.   `}`

84.   `}`

**1、**检查当前Bean是否实现了 `InstantiationAwareBeanPostProcessor`的 `postProcessAfterInstantiation`方法则调用，并结束Bean的填充。  
**2、**将按照类型跟按照名称注入的Bean分开，如果注入的Bean还没有实例化的这里会实例化，然后放到 `PropertyValues`对象中。  
**3、**如果实现了 `InstantiationAwareBeanPostProcessor`类的 `postProcessProperties`则调用这个方法并获取返回值，如果返回值是null，则有可能是实现了过期的 `postProcessPropertyValues`方法，这里需要进一步调用 `postProcessPropertyValues`方法  
**4、**进行参数填充

### **2.1.3 初始化Bean，initializeBean**

![](https://imgkr2.cn-bj.ufileos.com/7afa9bbb-1afa-4c3e-becb-6954bbe2030c.png?UCloudPublicKey=TOKEN_8d8b72be-579a-4e83-bfd0-5f6ce1546f13&Signature=wnrieMddiyhMsCHzsFgkIUAdl60%253D&Expires=1604287852)

### **初始化Bean**

同时这里根据代码跟流程图来说明**1、**如果Bean实现了 `BeanNameAware`, `BeanClassLoaderAware`, `BeanFactoryAware`则调用对应实现的方法 。整编：微信公众号，搜云库技术团队，ID：souyunku  
**2、**Bean不为null并且bean不是合成的，如果实现了 `BeanPostProcessor`的 `postProcessBeforeInitialization`则会调用实现的 `postProcessBeforeInitialization`方法。在 `ApplicationContextAwareProcessor`类中实现了 `postProcessBeforeInitialization`方法。而这个类会在Spring刷新容器准备 `beanFactory`的时候会加进去，这里就会被调用，而调用里面会检查Bean是不是 `EnvironmentAware`, `EmbeddedValueResolverAware`, `ResourceLoaderAware`, `ApplicationEventPublisherAware`, `MessageSourceAware`, `ApplicationContextAware`的实现类。这里就会调用对应的实现方法。代码如下

1.   `protected  void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory)  {`

2.   `.......`

3.   `beanFactory.addBeanPostProcessor(new  ApplicationContextAwareProcessor(this));`

4.   `.......`

1.   `public  Object postProcessBeforeInitialization(Object bean,  String beanName)  throws  BeansException  {`

2.   `if  (!(bean instanceof  EnvironmentAware  || bean instanceof  EmbeddedValueResolverAware  ||`

3.   `bean instanceof  ResourceLoaderAware  || bean instanceof  ApplicationEventPublisherAware  ||`

4.   `bean instanceof  MessageSourceAware  || bean instanceof  ApplicationContextAware)){`

5.   `return bean;`

6.   `}`

7.    

8.   `AccessControlContext acc =  null;`

9.    

10.   `if  (System.getSecurityManager()  !=  null)  {`

11.   `acc =  this.applicationContext.getBeanFactory().getAccessControlContext();`

12.   `}`

13.    

14.   `if  (acc !=  null)  {`

15.   `AccessController.doPrivileged((PrivilegedAction<Object>)  ()  ->  {`

16.   `invokeAwareInterfaces(bean);`

17.   `return  null;`

18.   `}, acc);`

19.   `}`

20.   `else  {`

21.   `invokeAwareInterfaces(bean);`

22.   `}`

23.    

24.   `return bean;`

25.   `}`

**1、**实例化Bean然后，检查是否实现了 `InitializingBean`的 `afterPropertiesSet`方法，如果实现了就会调用  
**2、**Bean不为null并且bean不是合成的，如果实现了 `BeanPostProcessor`的 `postProcessBeforeInitialization`则会调用实现的 `postProcessAfterInitialization`方法。到此创建Bean 的流程就没了，剩下的就是容器销毁的时候的了

## **三、destory方法跟销毁Bean**

Bean在创建完毕之后会检查用户是否指定了 `destroyMethodName`以及是否实现了 `DestructionAwareBeanPostProcessor`接口的 `requiresDestruction`方法，如果指定了会记录下来保存在 `DisposableBeanAdapter`对象中并保存在bean的 `disposableBeans`属性中。代码在 `AbstractBeanFactory`的 `registerDisposableBeanIfNecessary`中

1.   `protected  void registerDisposableBeanIfNecessary(String beanName,  Object bean,  RootBeanDefinition mbd)  {`

2.   `......`

3.   `registerDisposableBean(beanName,`

4.   `new  DisposableBeanAdapter(bean, beanName, mbd, getBeanPostProcessors(), acc));`

5.   `......`

6.   `}`

1.  `public  DisposableBeanAdapter(Object bean,  String beanName,  RootBeanDefinition beanDefinition,`

2.   `List<BeanPostProcessor> postProcessors,  @Nullable  AccessControlContext acc)  {`

3.   `.......`

4.   `String destroyMethodName = inferDestroyMethodIfNecessary(bean, beanDefinition);`

5.   `if  (destroyMethodName !=  null  &&  !(this.invokeDisposableBean &&  "destroy".equals(destroyMethodName))  &&`

6.   `!beanDefinition.isExternallyManagedDestroyMethod(destroyMethodName))  {`

7.   `......`

8.   `this.destroyMethod = destroyMethod;`

9.   `}`

10.   `this.beanPostProcessors = filterPostProcessors(postProcessors, bean);`

11.   `}`

在销毁Bean的时候最后都会调用 `AbstractAutowireCapableBeanFactory`的 `destroyBean`方法。

1.   `public  void destroyBean(Object existingBean)  {`

2.   `new  DisposableBeanAdapter(existingBean, getBeanPostProcessors(), getAccessControlContext()).destroy();`

3.   `}`

这里是创建一个 `DisposableBeanAdapter`对象，这个对象实现了Runnable接口，在实现的 `run`方法中会调用实现的 `DisposableBean`接口的 `destroy`方法。并且在创建 `DisposableBeanAdapter`对象的时候会根据传入的bean是否实现了 `DisposableBean`接口来设置 `invokeDisposableBean`变量，这个变量表实有没有实现 `DisposableBean`接口

1.   `public  DisposableBeanAdapter(Object bean,  List<BeanPostProcessor> postProcessors,  AccessControlContext acc)  {`

2.   `Assert.notNull(bean,  "Disposable bean must not be null");`

3.   `this.bean = bean;`

4.   `this.beanName = bean.getClass().getName();`

5.   ``//根据传入的bean是否实现了`DisposableBean`接口来设置`invokeDisposableBean`变量``

6.   `this.invokeDisposableBean =  (this.bean instanceof  DisposableBean);`

7.   `this.nonPublicAccessAllowed =  true;`

8.   `this.acc = acc;`

9.   `this.beanPostProcessors = filterPostProcessors(postProcessors, bean);`

10.   `}`

11.    

12.   `public  void destroy()  {`

13.   `......`

14.   `//根据invokeDisposableBean决定是否调用destroy方法`

15.   `if  (this.invokeDisposableBean)  {`

16.   `if  (logger.isTraceEnabled())  {`

17.   `logger.trace("Invoking destroy() on bean with name '"  +  this.beanName +  "'");`

18.   `}`

19.   `try  {`

20.   `if  (System.getSecurityManager()  !=  null)  {`

21.   `AccessController.doPrivileged((PrivilegedExceptionAction<Object>)  ()  ->  {`

22.   `((DisposableBean)  this.bean).destroy();`

23.   `return  null;`

24.   `},  this.acc);`

25.   `}`

26.   `else  {`

27.   `((DisposableBean)  this.bean).destroy();`

28.   `}`

29.   `}`

30.   `catch  (Throwable ex)  {`

31.   `String msg =  "Invocation of destroy method failed on bean with name '"  +  this.beanName +  "'";`

32.   `if  (logger.isDebugEnabled())  {`

33.   `logger.warn(msg, ex);`

34.   `}`

35.   `else  {`

36.   `logger.warn(msg +  ": "  + ex);`

37.   `}`

38.   `}`

39.   `}`

40.  `......`

41.   `}`
