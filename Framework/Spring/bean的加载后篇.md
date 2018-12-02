# bean的加载

上一节中 我们讲到了resolveBeforeInstantiation方法 当程序经历过这个方法之后，有两种情况。
1. 如果创建了代理或者说重写了InstantiationAwareBeanPostProcessor的postProcessBeforeInstantiation方法并在该方法中改变了bean，则直接返回就可以了。上一文提到的短路逻辑
2. 进行常规的bean创建doCreateBean

## 创建bean
doCreateBean方法源码如下

```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
    throws BeanCreationException {

  // Instantiate the bean.
  BeanWrapper instanceWrapper = null;
  if (mbd.isSingleton()) {
    instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
  }
  if (instanceWrapper == null) {
    //根据指定的bean使用相应的方式创建新的实例
    instanceWrapper = createBeanInstance(beanName, mbd, args);
  }
  final Object bean = instanceWrapper.getWrappedInstance();
  Class<?> beanType = instanceWrapper.getWrappedClass();
  if (beanType != NullBean.class) {
    mbd.resolvedTargetType = beanType;
  }

  // Allow post-processors to modify the merged bean definition.
  synchronized (mbd.postProcessingLock) {
    if (!mbd.postProcessed) {
      try {
        applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
      }
      catch (Throwable ex) {
        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
            "Post-processing of merged bean definition failed", ex);
      }
      mbd.postProcessed = true;
    }
  }

  //是否提早曝光bean 解决上一篇文章提到的单利模式的循环依赖问题 同时进行循环依赖的检测
  // Eagerly cache singletons to be able to resolve circular references
  // even when triggered by lifecycle interfaces like BeanFactoryAware.
  boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
      isSingletonCurrentlyInCreation(beanName));
      // bean初始化完成前将创建实例的objectFactory加入工厂
  if (earlySingletonExposure) {
    if (logger.isDebugEnabled()) {

    }
    //aop在这里将advice动态植入bean中
    addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
  }

  // Initialize the bean instance.
  Object exposedObject = bean;
  try {
    // 对bean的属性进行填充 如果有其他依赖的bean则递归初始化依赖的bean
    populateBean(beanName, mbd, instanceWrapper);
    exposedObject = initializeBean(beanName, exposedObject, mbd);
  }
  catch (Throwable ex) {

  }

  if (earlySingletonExposure) {
    Object earlySingletonReference = getSingleton(beanName, false);
    if (earlySingletonReference != null) {
      if (exposedObject == bean) {
        exposedObject = earlySingletonReference;
      }
      else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
        String[] dependentBeans = getDependentBeans(beanName);
        Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
        for (String dependentBean : dependentBeans) {
          //检测依赖
          if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
            actualDependentBeans.add(dependentBean);
          }
        }
        //存在循环依赖
        if (!actualDependentBeans.isEmpty()) {
          throw new BeanCurrentlyInCreationException(beanName,

        }
      }
    }
  }

  // Register bean as disposable.
  try {
    // 根据scope注册bean
    registerDisposableBeanIfNecessary(beanName, bean, mbd);
  }
  catch (BeanDefinitionValidationException ex) {
  }

  return exposedObject;
}
```

整个函数的流程：

1. 如果是单例模式的bean先清除缓存
2. 实例化bean将BeanDefinition转化为BeanWrapper
3. `MergedBeanDefinitionPostProcess`的应用进行bean合并后的处理
4. 依赖处理
5. 将bean包含的属性填充到bean的实例中
6. 循环依赖的检查
7. 注册DisposableBean 如果配置了destroy-method在这里进行注册便于在销毁的时候调用
8. 完成创建并返回

### 创建bean的实例

先看一下createBeanInstance的代码

```java
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
  // 解析bean class
  Class<?> beanClass = resolveBeanClass(mbd, beanName);

  if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
    throw new BeanCreationException();
  }

  //Spring5.0引入的返回一个supplier对象实例化一个bean
  Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
  if (instanceSupplier != null) {
    return obtainFromSupplier(instanceSupplier, beanName);
  }
// 如果工厂方法不为空则使用工厂方法进行实例化
  if (mbd.getFactoryMethodName() != null)  {
    return instantiateUsingFactoryMethod(beanName, mbd, args);
  }

  // Shortcut when re-creating the same bean...
  boolean resolved = false;
  boolean autowireNecessary = false;
  if (args == null) {
    synchronized (mbd.constructorArgumentLock) {
      if (mbd.resolvedConstructorOrFactoryMethod != null) {
        resolved = true;
        autowireNecessary = mbd.constructorArgumentsResolved;
      }
    }
  }
  // 如果已经解析过则使用解析过的构造函数
  if (resolved) {
    if (autowireNecessary) {
      // 构造函数的自动注入
      return autowireConstructor(beanName, mbd, null, null);
    }
    else {
      // 使用默认构造函数 无参
      return instantiateBean(beanName, mbd);
    }
  }

  // 需要根据参数解析构造函数
  Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
  if (ctors != null ||
      mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_CONSTRUCTOR ||
      mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args))  {
    return autowireConstructor(beanName, mbd, ctors, args);
  }

  // No special handling: simply use no-arg constructor.
  return instantiateBean(beanName, mbd);
}
```
1. 如果有指定的supplier则使用指定的supplier去实例化
2. 如果在RootBeanDefinition中存在factoryMethodName属性，或者再配置文件中配置了factory-method，那么spring会尝试使用instantiateUsingFactoryMethod方法根据RootBeanDefinition中的配置生成实例。
3. 解析构造函数并行构造函数的实例化，因为一个bean可能会有多个构造函数，而每个构造函数的参数不同，spring在根据参数及类型去判断最终会使用哪个构造函数进行实例化。而这个判断过程非常的消耗性能，因此spring采用了缓存机制，如果已经解析过了则不需要充分解析而直接从RootBeanDefinition中的属性resolvedConstructorOrFactoryMethod中去取，否则需要再次解析，并将解析结果添加到RootBeanDefinition的resolvedConstructorOrFactoryMethod中。

#### autowireConstructor方法

```java
public BeanWrapper autowireConstructor(final String beanName, final RootBeanDefinition mbd,
    @Nullable Constructor<?>[] chosenCtors, @Nullable final Object[] explicitArgs) {

  BeanWrapperImpl bw = new BeanWrapperImpl();
  this.beanFactory.initBeanWrapper(bw);

  Constructor<?> constructorToUse = null;
  ArgumentsHolder argsHolderToUse = null;
  Object[] argsToUse = null;
  //explicitArgs通过getBean方法传入  
  //如果getBean方法调用的时候指定方法参数那么直接使用  
  if (explicitArgs != null) {
    argsToUse = explicitArgs;
  }
  else {
    // 没传则从配置文件中解析
    Object[] argsToResolve = null;
    synchronized (mbd.constructorArgumentLock) {
      constructorToUse = (Constructor<?>) mbd.resolvedConstructorOrFactoryMethod;
      if (constructorToUse != null && mbd.constructorArgumentsResolved) {
        // Found a cached constructor...
        argsToUse = mbd.resolvedConstructorArguments;
        if (argsToUse == null) {
          argsToResolve = mbd.preparedConstructorArguments;
        }
      }
    }
    if (argsToResolve != null) {
      //解析参数类型，如给定方法的构造函数A(int,int)则通过此方法后就会把配置中（"1","1"）转换为（1，1）
      argsToUse = resolvePreparedArguments(beanName, mbd, bw, constructorToUse, argsToResolve);
    }
  }

  if (constructorToUse == null) {
    // Need to resolve the constructor.
    boolean autowiring = (chosenCtors != null ||
        mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_CONSTRUCTOR);
    ConstructorArgumentValues resolvedValues = null;

    int minNrOfArgs;
    if (explicitArgs != null) {
      minNrOfArgs = explicitArgs.length;
    }
    else {
      // 提取配置文件中配置的构造函数参数
      ConstructorArgumentValues cargs = mbd.getConstructorArgumentValues();
      resolvedValues = new ConstructorArgumentValues();
      // 将参数进行解析然后返回一个个数
      minNrOfArgs = resolveConstructorArguments(beanName, mbd, bw, cargs, resolvedValues);
    }

    // Take specified constructors, if any.
    Constructor<?>[] candidates = chosenCtors;
    if (candidates == null) {
      Class<?> beanClass = mbd.getBeanClass();
      try {
        candidates = (mbd.isNonPublicAccessAllowed() ?
            beanClass.getDeclaredConstructors() : beanClass.getConstructors());
      }
      catch (Throwable ex) {
        throw new BeanCreationException();
      }
    }
    // 将构造函数进行排序 先排public的然后按照参数个数进行降序排序
    AutowireUtils.sortConstructors(candidates);
    int minTypeDiffWeight = Integer.MAX_VALUE;
    Set<Constructor<?>> ambiguousConstructors = null;
    LinkedList<UnsatisfiedDependencyException> causes = null;

    for (Constructor<?> candidate : candidates) {
      Class<?>[] paramTypes = candidate.getParameterTypes();
      // 判断参数的个数
      if (constructorToUse != null && argsToUse.length > paramTypes.length) {
        break;
      }
      if (paramTypes.length < minNrOfArgs) {
        continue;
      }

      ArgumentsHolder argsHolder;
      if (resolvedValues != null) {
        // 根据值构造对应参数类型的参数
        try {
          String[] paramNames = ConstructorPropertiesChecker.evaluate(candidate, paramTypes.length);
          if (paramNames == null) {
            ParameterNameDiscoverer pnd = this.beanFactory.getParameterNameDiscoverer();
            if (pnd != null) {
              // 获取指定构造函数的参数名称
              paramNames = pnd.getParameterNames(candidate);
            }
          }
          // 创建参数的持有者
          argsHolder = createArgumentArray(beanName, mbd, resolvedValues, bw, paramTypes, paramNames,
              getUserDeclaredConstructor(candidate), autowiring);
        }
        catch (UnsatisfiedDependencyException ex) {

        }
      }
      else {
        // Explicit arguments given -> arguments length must match exactly.
        if (paramTypes.length != explicitArgs.length) {
          continue;
        }
        argsHolder = new ArgumentsHolder(explicitArgs);
      }
      // 看一下是否存在不同构造函数的参数为父子类型
      int typeDiffWeight = (mbd.isLenientConstructorResolution() ?
          argsHolder.getTypeDifferenceWeight(paramTypes) : argsHolder.getAssignabilityWeight(paramTypes));
      // Choose this constructor if it represents the closest match.
      if (typeDiffWeight < minTypeDiffWeight) {
        constructorToUse = candidate;
        argsHolderToUse = argsHolder;
        argsToUse = argsHolder.arguments;
        minTypeDiffWeight = typeDiffWeight;
        ambiguousConstructors = null;
      }
      else if (constructorToUse != null && typeDiffWeight == minTypeDiffWeight) {
        if (ambiguousConstructors == null) {
          ambiguousConstructors = new LinkedHashSet<>();
          ambiguousConstructors.add(constructorToUse);
        }
        ambiguousConstructors.add(candidate);
      }
    }

    if (constructorToUse == null) {
      if (causes != null) {
        UnsatisfiedDependencyException ex = causes.removeLast();
        for (Exception cause : causes) {
          this.beanFactory.onSuppressedException(cause);
        }
        throw ex;
      }
      throw new BeanCreationException();
    }
    else if (ambiguousConstructors != null && !mbd.isLenientConstructorResolution()) {
      throw new BeanCreationException();
    }

    if (explicitArgs == null) {
      // 将解析玩的构造函数写入缓存
      argsHolderToUse.storeCache(mbd, constructorToUse);
    }
  }

  try {
    final InstantiationStrategy strategy = beanFactory.getInstantiationStrategy();
    Object beanInstance;

    if (System.getSecurityManager() != null) {
      final Constructor<?> ctorToUse = constructorToUse;
      final Object[] argumentsToUse = argsToUse;
      beanInstance = AccessController.doPrivileged((PrivilegedAction<Object>) () ->
          strategy.instantiate(mbd, beanName, beanFactory, ctorToUse, argumentsToUse),
          beanFactory.getAccessControlContext());
    }
    else {
      beanInstance = strategy.instantiate(mbd, beanName, this.beanFactory, constructorToUse, argsToUse);
    }
    // 将创建的实例加入beanWrapper中
    bw.setBeanInstance(beanInstance);
    return bw;
  }
  catch (Throwable ex) {
    throw new BeanCreationException();
  }
}
```
1 首先是构造函数参数的确定

(1) 根据explicitArgs参数来判断

如果传入的explicitArgs不为空，则可以直接确定参数，因为explicitArgs是在调用bean的时候用户指定的，在BeanFactory类中存在这样的方法：
Object getBean(String name,Object... args) throws BeansException;
在获取bean的时候，用户不但可以指定bean的名称还可以指定bean锁对应类的构造函数或者工厂方法的方法参数，主要用于静态工厂方法调用，而这里是需要给定完全匹配的参数。所以便可以判断如果传入的explicitArgs不为空就可以确定构造函数的参数。

(2) 缓存中获取

确定参数的方法如果直接已经分析过了，也就是说构造函数参数已经记录在缓存中，那可以直接从缓存中读取。这里需要注意，在缓存中存的可能是参数的最终类型也可能是参数的初始类型，所以即使在缓存中得到了参数也需要经过类型转换器的过滤。

(3)配置文件中获取

如果不能根据传入的参数explicitArgs确定构造函数的参数也无法在缓存中得到，那就需要分享配置文件中的配置的构造函数的信息。在spring中配置文件的信息经过转换后都会保存在BeanDefinition中，也就是参数mbd中，因此可以调用mbd.getConstructorArgumentValues()来获取配置的构造函数信息。有了配置中的信息就可以获取对应的参数值信息了。

2 构造函数的确定

经过了第一步后已经确定了构造函数的参数，接下来就是根据参数来锁定对应的构造函数，而匹配的方法就是根据参数个数匹配，所以在匹配之前需要先对构造函数按照public构造函数优先参数数量降序、非public构造函数参数数量降序。这样可以在遍历的情况下迅速判断排在后面的构造函数参数个数是否符合条件。 由于配置文件中并不是唯一限制使用参数位置索引的方式去创建，同样还支持指定参数名称进行设定参数值的情况。这种情况需要先确定构造函数中的参数名称。 获取参数名称有两种方式，一种是通过注解的方式直接获取，另一种是使用spring中提供的工具类ParameterNameDiscoverer来获取。构造函数、参数名称、参数类型、参数值都确定后就可以锁定构造函数及转换对应的参数类型。

3 根据确定的构造函数转换对应的参数类型

主要是spring中提供的类型转换器或者用户提供的自定义类型转换器进行转换

4 构造函数不确定性验证

有时候即使构造函数、参数名称、参数类型、参数值确定后也不一定会直接锁定构造函数，因为不同的构造函数的参数为父子关系，所以spring在最后又做了一次验证

5 根据实例化策略以及得到的构造函数及参数实例化bean。

#### instantiateBean不带参数的构造函数实例化过程
```java
protected BeanWrapper instantiateBean(final String beanName, final RootBeanDefinition mbd) {
  try {
    Object beanInstance;
    final BeanFactory parent = this;
    // 权限
    if (System.getSecurityManager() != null) {

    else {
      // 调用实例化策略进行实例化
      beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, parent);
    }
    BeanWrapper bw = new BeanWrapperImpl(beanInstance);
    initBeanWrapper(bw);
    return bw;
  }
  catch (Throwable ex) {
    throw new BeanCreationException();
  }
}
```

#### 实例化策略
主要看instantiate代码
```java
@Override
public Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner) {
  // Don't override the class with CGLIB if no overrides.
  if (!bd.hasMethodOverrides()) {
    Constructor<?> constructorToUse;
    synchronized (bd.constructorArgumentLock) {
      constructorToUse = (Constructor<?>) bd.resolvedConstructorOrFactoryMethod;
      if (constructorToUse == null) {
        final Class<?> clazz = bd.getBeanClass();
        if (clazz.isInterface()) {
          throw new BeanInstantiationException();
        }
        try {
          // 权限
          if (System.getSecurityManager() != null) {

          }
          else {
            // 反射
            constructorToUse =	clazz.getDeclaredConstructor();
          }
          bd.resolvedConstructorOrFactoryMethod = constructorToUse;
        }
        catch (Throwable ex) {
          throw new BeanInstantiationException();
        }
      }
    }
    return BeanUtils.instantiateClass(constructorToUse);
  }
  else {
    // 如果需要通过动态代理进行替换 将拦截增强器注入进去
    return instantiateWithMethodInjection(bd, beanName, owner);
  }
}
```

### 记录创建bean的ObjectFactory
之前doCreate方法中有一段代码
```java
boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
            isSingletonCurrentlyInCreation(beanName));
    if (earlySingletonExposure) {
        if (logger.isDebugEnabled()) {

        }
        //为避免后期循环依赖，可以在bean初始化完成前将创建实例的ObjectFactory加入工厂  
        //依赖处理：在Spring中会有循环依赖的情况，例如，当A中含有B的属性，而B中又含有A的属性时就会  
        //构成一个循环依赖，此时如果A和B都是单例，那么在Spring中的处理方式就是当创建B的时候，涉及  
        //自动注入A的步骤时，并不是直接去再次创建A，而是通过放入缓存中的ObjectFactory来创建实例，  
        //这样就解决了循环依赖的问题。
        addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
    }
```
earlySingletonExposure:提早曝光的单例
mbd.isSingleton()代表的是否单例。
this.allowCircularReferences:是否允许循环依赖，在AbstractRefreshableApplicationContext中提供了设置函数，可以通过硬编码的方式进行设置或者可以通过自定义命名空间进行配置，其中硬编码方式代码如下：
```java
ClassPathXmlApplicationContext bf = new ClassPathXmlApplicationContext("xxx.xml");  
bf.setAllowBeanDefinitionOverriding(false);  
```
isSingletonCurrentlyInCreation(beanName):该bean是否在创建中。在Spring中，会有个专门的属性默认为DefaultSingletonBeanRegistry的singletonsCurrentlyInCreation来记录bean的加载状态，在bean开始创建前会将beanName记录在属性中，在bean创建结束后会将beanName移除。这个状态是在哪里记录的呢？不同scope的记录位置不一样，我们以singleton为例，在singleton下记录属性的函数是在DefaultSingletonBeanRegistry类的public Object getSingleton（String beanName,ObjectFactory singletonFactory）函数的beforeSingletonCreation(beanName)和afterSingletonCreation(beanName)中，在这两段函数中分别this.singletonsCurrentlyInCreation.add(beanName)与this.singletonsCurrentlyInCreation.remove(beanName)来进行状态的记录与移除。

经过以上的分析我们了解变量earlySingletonExposure是否是单例，是否允许循环依赖，是否对应的bean正在创建的条件的综合。当这3个条件都满足时会执行addSingletonFactory操作，那么加入SingletonFactory的作用是什么？又是在什么时候调用的？
我们还是以最简单AB循环为例，类A中含有属性B,而类B中又会含有属性A,那么初始化beanA的过程如下:

1. 创建A
2. addSingletonFactory
3. populateBean 填充属性
4. 创建B
5. addSingletonFactory
6. populateBean 填充属性
7. 结束创建B
8. 结束创建A

在创建A的时候首先会记录类A所对应额beanName，并将beanA的创建工厂加入缓存中，而在对A的属性填充也就是调用pupulateBean方法的时候又会再一次的对B进行递归创建。同样的，因为在B中同样存在A属性，因此在实例化B的populateBean方法中又会再次地初始化B,调用getBean（A）.关键是在这里，我们之前分析过，在这个函数中并不是直接去实例化A,而是先去检测缓存中是否有已经创建好的对应的bean,或者是否已经创建的ObjectFactory，而此时对于A的ObjectFactory我们早已经创建，所以便不会再去向后执行，而是直接调用ObjectFactory去创建A.这里最关键的是ObjectFactory的实现。
addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));

根据分析，基本可以理清Spring处理循环依赖的解决办法，在B中创建依赖A时通过ObjectFactory提供的实例化方法来获取原始A，使B中持有的A仅仅是刚刚初始化并没有填充任何属性的A,而这初始化A的步骤还是刚刚创建A时进行的，但是因为 A与B中的A所表示的属性地址是一样 的所以在A中创建好的属性填充自然可以通过B中的A获取，这样就解决了循环依赖的问题。

### 属性的注入（populateBean的实现）

```java
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
  if (bw == null) {
    if (mbd.hasPropertyValues()) {
      throw new BeanCreationException();
    }
    else {
      // Skip property population phase for null instance.
      return;
    }
  }

  // 给InstantiationAwareBeanPostProcessor代理一个机会改变bean
  boolean continueWithPropertyPopulation = true;

  if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
    for (BeanPostProcessor bp : getBeanPostProcessors()) {
      if (bp instanceof InstantiationAwareBeanPostProcessor) {
        InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
        if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
          continueWithPropertyPopulation = false;
          break;
        }
      }
    }
  }

  if (!continueWithPropertyPopulation) {
    return;
  }

  PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);

  if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME ||
      mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
    MutablePropertyValues newPvs = new MutablePropertyValues(pvs);

    // 根据name注入属性
    if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME) {
      autowireByName(beanName, mbd, bw, newPvs);
    }

    // 根据type
    if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
      autowireByType(beanName, mbd, bw, newPvs);
    }

    pvs = newPvs;
  }
  // 后处理器是否初始化和是否检查依赖
  boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
  boolean needsDepCheck = (mbd.getDependencyCheck() != RootBeanDefinition.DEPENDENCY_CHECK_NONE);

  if (hasInstAwareBpps || needsDepCheck) {
    if (pvs == null) {
      pvs = mbd.getPropertyValues();
    }
    PropertyDescriptor[] filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
    if (hasInstAwareBpps) {
      for (BeanPostProcessor bp : getBeanPostProcessors()) {
        if (bp instanceof InstantiationAwareBeanPostProcessor) {
          InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
          // 对于属性进行后处理
          pvs = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
          if (pvs == null) {
            return;
          }
        }
      }
    }
    if (needsDepCheck) {
      // 检查依赖
      checkDependencies(beanName, mbd, filteredPds, pvs);
    }
  }

  if (pvs != null) {
    // 将相应的属性应用在bean中
    applyPropertyValues(beanName, mbd, bw, pvs);
  }
}

```

1. 先进行属性是否为空的判断
2. 通过调用InstantiationAwareBeanPostProcessor的postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)方法来控制程序是否继续进行属性填充
3 根据注入类型（byName/byType）提取依赖的bean，并统一存入PropertyValues中
4 应用InstantiationAwareBeanPostProcessor的postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName)方法，对属性获取完毕填充前的再次处理
5 将所有的PropertyValues中的属性填充至BeanWrapper中

其中最终要的方法是 autowireByName ByType

#### autowireByName

```java
    //在传入的参数bw中，找出已经加载的bean,并递归实例化，进而加入到pvs中。
    protected void autowireByName(  
            String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {  
        //寻找bw中需要依赖注入的属性  
        String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);  
        for (String propertyName : propertyNames) {  
            if (containsBean(propertyName)) {  
                //递归初始化相关的bean  
                Object bean = getBean(propertyName);  
                pvs.add(propertyName, bean);  
                //注册依赖  
                registerDependentBean(propertyName, beanName);  
                if (logger.isDebugEnabled()) {  
                    logger.debug();  
                }  
            }  
            else {  
                if (logger.isTraceEnabled()) {  
                    logger.trace();  
                }  
            }  
        }  
    }

```

#### autowireByType

```java
protected void autowireByType(
			String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {

		TypeConverter converter = getCustomTypeConverter();
		if (converter == null) {
			converter = bw;
		}

		Set<String> autowiredBeanNames = new LinkedHashSet<>(4);
		String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
		for (String propertyName : propertyNames) {
			try {
				PropertyDescriptor pd = bw.getPropertyDescriptor(propertyName);

				if (Object.class != pd.getPropertyType()) {
          // 找到set方法
					MethodParameter methodParam = BeanUtils.getWriteMethodParameter(pd);
					boolean eager = !PriorityOrdered.class.isInstance(bw.getWrappedInstance());
					DependencyDescriptor desc = new AutowireByTypeDependencyDescriptor(methodParam, eager);
          //解析指定beanName的属性所匹配的值，并把解析到的属性名称存储在autowiredBeanNames中，  
					Object autowiredArgument = resolveDependency(desc, beanName, autowiredBeanNames, converter);
					if (autowiredArgument != null) {
						pvs.add(propertyName, autowiredArgument);
					}
					for (String autowiredBeanName : autowiredBeanNames) {
            // 注册依赖
						registerDependentBean(autowiredBeanName, beanName);
						if (logger.isDebugEnabled()) {
							logger.debug();
						}
					}
					autowiredBeanNames.clear();
				}
			}
			catch (BeansException ex) {
			}
		}
	}
```

寻找类型匹配的逻辑实现是封装在了resolveDependency函数中，其实现主要是寻找类型的匹配执行顺序时，首先尝试使用解析器解析，如果解析器没有成功解析，可能是默认的解析器没有做任何处理，或者是使用了自定义解析器。
虽说对于不同类型的处理方式不一致，但是大致的思路还是相似的，以数组类型举例

1. 根据属性类型找到beanFactory中所有类型的匹配bean，  
2. 如果autowire的require属性为true而找到的匹配项却为空则只能抛出异常
3. 过转换器将bean的值转换为对应的type类型  converter.convertIfNecessary(matchingBeans.values(), type)

#### applyPropertyValues

已经完成了对所有注入属性的获取，但是获取的属性是以PropertyValues形式存在的，还并没有应用到已经实例化的bean中，这一工作是在applyPropertyValues中

```java
protected void applyPropertyValues(String beanName, BeanDefinition mbd, BeanWrapper bw, PropertyValues pvs) {
		if (pvs.isEmpty()) {
			return;
		}

		if (System.getSecurityManager() != null && bw instanceof BeanWrapperImpl) {
			((BeanWrapperImpl) bw).setSecurityContext(getAccessControlContext());
		}

		MutablePropertyValues mpvs = null;
		List<PropertyValue> original;

		if (pvs instanceof MutablePropertyValues) {
			mpvs = (MutablePropertyValues) pvs;
			if (mpvs.isConverted()) {
				try {
          // 如果已经转换过 则直接set
					bw.setPropertyValues(mpvs);
					return;
				}
				catch (BeansException ex) {
					throw new BeanCreationException();
				}
			}
			original = mpvs.getPropertyValueList();
		}
		else {
      //如果pvs并不是使用MutablePropertyValues封装的类型，那么直接使用原始的属性获取方法  
			original = Arrays.asList(pvs.getPropertyValues());
		}

		TypeConverter converter = getCustomTypeConverter();
		if (converter == null) {
			converter = bw;
		}
    //获取对应的解析器  
		BeanDefinitionValueResolver valueResolver = new BeanDefinitionValueResolver(this, beanName, mbd, converter);

		// 深拷贝 遍历属性，将属性转换为对应类的对应属性的类型
		List<PropertyValue> deepCopy = new ArrayList<>(original.size());
		boolean resolveNecessary = false;
		for (PropertyValue pv : original) {
			if (pv.isConverted()) {
				deepCopy.add(pv);
			}
			else {
				String propertyName = pv.getName();
				Object originalValue = pv.getValue();
				Object resolvedValue = valueResolver.resolveValueIfNecessary(pv, originalValue);
				Object convertedValue = resolvedValue;
				boolean convertible = bw.isWritableProperty(propertyName) &&
						!PropertyAccessorUtils.isNestedOrIndexedProperty(propertyName);
				if (convertible) {
					convertedValue = convertForProperty(resolvedValue, propertyName, bw, converter);
				}

				if (resolvedValue == originalValue) {
					if (convertible) {
						pv.setConvertedValue(convertedValue);
					}
					deepCopy.add(pv);
				}
				else if (convertible && originalValue instanceof TypedStringValue &&
						!((TypedStringValue) originalValue).isDynamic() &&
						!(convertedValue instanceof Collection || ObjectUtils.isArray(convertedValue))) {
					pv.setConvertedValue(convertedValue);
					deepCopy.add(pv);
				}
				else {
					resolveNecessary = true;
					deepCopy.add(new PropertyValue(pv, convertedValue));
				}
			}
		}
		if (mpvs != null && !resolveNecessary) {
			mpvs.setConverted();
		}

		// Set our (possibly massaged) deep copy.
		try {
			bw.setPropertyValues(new MutablePropertyValues(deepCopy));
		}
		catch (BeansException ex) {
		}
	}
```

### 初始化bean

Spring在配置bean的时候有一个init-method属性，这个属性的作用是在bean实例化前调用init-method指定的方法来根据用户业务进行相应的实例化，首先看下这个方法的执行位置，Spring中程序已经执行过bean的实例化，并进行了属性填充，而就在这时将会调用用户设定的初始化方法
在populateBean方法下面有一个initializeBean(beanName, exposedObject, mbd)方法，这个就是用来执行用户设定的初始化操作。

```java
protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {  
        if (System.getSecurityManager() != null) {  
            // 权限
        }  
        else {  
            //对特殊的bean处理：Aware,BeanClassLoaderAware,BeanFactoryAware  
            invokeAwareMethods(beanName, bean);  
        }  

        Object wrappedBean = bean;  
        if (mbd == null || !mbd.isSynthetic()) {  
            //应用后处理器  
            wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);  
        }  

        try {  
            //激活用户自定义的init方法  
            invokeInitMethods(beanName, wrappedBean, mbd);  
        }  
        catch (Throwable ex) {  
            throw new BeanCreationException();  
        }  

        if (mbd == null || !mbd.isSynthetic()) {  
            //后处理器应用  
            wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);  
        }  
        return wrappedBean;  
    }  
```

这个方法主要实现了进行用户设定的初始化方法的调用

#### 激活aware方法

Spring中提供了一些Aware接口，比如BeanFactoryAware,ApplicationContextAware,ResourceLoaderAware,ServletContextAware等，实现这些Aware接口的bean在被初始化后，可以取得一些相对应的资源，例如实现BeanFactoryAware的bean在初始化之后，Spring容器将会注入BeanFactory实例，而实现ApplicationContextAware的bean，在bean被初始化后，将会被注入ApplicationContext实例等。我们先通过示例方法了解下Aware的使用。

（1）定义普通bean，如下代码：

```java
public class HelloBean {
    public void say()
    {
        System.out.println("Hello");
    }
}
```

（2）定义beanFactoryAware类型的bean

```java
public class MyBeanAware implements BeanFactoryAware {
    private BeanFactory beanFactory;
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        this.beanFactory = beanFactory;
    }
    public void testAware()
    {
        //通过hello这个bean id从beanFactory获取实例  
        HelloBean hello = (HelloBean)beanFactory.getBean("hello");
        hello.say();
    }
}
```

（3）进行测试

```java
public class Test {
    public static void main(String[] args) {
        ApplicationContext ctx = new ClassPathXmlApplicationContext("applicationContext.xml");
        MyBeanAware test = (MyBeanAware)ctx.getBean("myBeanAware");
        test.testAware();
    }
}
```

其中applicationContext.xml文件内容

<?xml version="1.0" encoding="UTF-8" ?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="myBeanAware" class="com.yhl.myspring.demo.aware.MyBeanAware">
    </bean>
    <bean id="hello" class="com.yhl.myspring.demo.aware.HelloBean">
    </bean>
</beans>

上面的方法我们获取到Spring中BeanFactory，并且可以根据BeanFactory获取所有的bean,以及进行相关设置。还有其他Aware的使用都是大同小异，看一下Spring的实现方式：

```java
private void invokeAwareMethods(final String beanName, final Object bean) {  
        if (bean instanceof Aware) {  
            if (bean instanceof BeanNameAware) {  
                ((BeanNameAware) bean).setBeanName(beanName);  
            }  
            if (bean instanceof BeanClassLoaderAware) {  
                ((BeanClassLoaderAware) bean).setBeanClassLoader(getBeanClassLoader());  
            }  
            if (bean instanceof BeanFactoryAware) {  
                ((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);  
            }  
        }  
```

#### 处理器的应用
BeanPostPrecessor我给用户充足的权限去更改或者扩展Spring,而除了BeanPostProcessor外还有很多其他的PostProcessor，当然大部分都以此为基础，集成自BeanPostProcessor。BeanPostProcessor在调用用户自定义初始化方法前或者调用自定义初始化方法后分别会调用BeanPostProcessor的postProcessBeforeInitialization和postProcessAfterinitialization方法，使用户可以根据自己的业务需求就行相应的处理。

#### 激活自定义的init方法 常用的读取配置文件的实现
客户定制的初始化方法除了我们熟知的使用配置init-method外，还有使自定义的bean实现InitializingBean接口，并在afterPropertiesSet中实现自己的初始化业务逻辑。
init-method与afterPropertiesSet都是在初始化bean时执行，执行顺序是afterPropertiesSet先执行，而init-method后执行。
在invokeInitMethods方法中就实现了这两个步骤的初始化调用。

```java
protected void invokeInitMethods(String beanName, final Object bean, RootBeanDefinition mbd)  
            throws Throwable {  

        boolean isInitializingBean = (bean instanceof InitializingBean);  
        if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {  
            if (logger.isDebugEnabled()) {  
                logger.debug("Invoking afterPropertiesSet() on bean with name '" + beanName + "'");  
            }  
            if (System.getSecurityManager() != null) {  
                try {  
                    AccessController.doPrivileged(new PrivilegedExceptionAction<Object>() {  
                        public Object run() throws Exception {  
                            ((InitializingBean) bean).afterPropertiesSet();  
                            return null;  
                        }  
                    }, getAccessControlContext());  
                }  
                catch (PrivilegedActionException pae) {  
                    throw pae.getException();  
                }  
            }  
            else {  
                //属性初始化后的处理  
                ((InitializingBean) bean).afterPropertiesSet();  
            }  
        }  

        if (mbd != null) {  
            String initMethodName = mbd.getInitMethodName();  
            if (initMethodName != null && !(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&  
                    !mbd.isExternallyManagedInitMethod(initMethodName)) {  
                //调用自定义初始化方法  
                invokeCustomInitMethod(beanName, bean, mbd);  
            }  
        }  
    }  
```

### 注册DisponableBean

Spring中不仅提供了对于初始化方法的扩展入口，同样也提供了销毁方法的扩展入口，对于销毁方法的扩展，除了配置属性destroy-method方法外，还可以注册后处理器DestructionAwareBeanPostProcessor来统一处理bean的销毁方法
