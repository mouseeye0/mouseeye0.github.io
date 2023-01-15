---
title: Spring 에서 Custom annotation 사용
author: mouseeye
date: 2022-12-09 15:00:00 +0900
categories: [Spring]
tags: [Spring, BeanPostProcessor]
---

### 1. Case1 @DataAccess annotation
선언된 Entity 객체를 이용해서 자동으로 Data access object 를 만드는 예제

#### `GenericDao` 구현체 오브젝트 생성
```java
import lombok.RequiredArgsConstructor;

import java.util.Collections;
import java.util.List;
import java.util.Optional;

@RequiredArgsConstructor
public class GenericDao<E> {

    private final Class<E> entityClass;
    private String message;

    public List<E> findAll() {
        message = "Would create findAll query from " + entityClass.getSimpleName();
        return Collections.emptyList();
    }

    public Optional<E> persist(E entity) {
        return Optional.empty();
    }

    public final String getMessage() {
        return message;
    }
}
```
#### `@DataAccess` 생성 & 적용
- @DataAccess
```java
import java.lang.annotation.*;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
public @interface DaoAccess {
    Class<?> entity();
}
```

- Person 객체
```java
@AllArgsConstructor
@Data
public class Person {
    private Long id;
    private String name;
}
```

- 실제 사용 패턴 (Person 객체를 사용하는 Bean 생성 및 의존성 주입)
```java
@DaoAccess(entity=Person.class)
private GenericDao<Person> personDao;
```

#### BeanPostProcessor 구현
```java
public class DataAccessAnnotationProcessor implements BeanPostProcessor {

    private ConfigurableListableBeanFactory configurableListableBeanFactory;
    // Bean 정보를 담고 있는 Container, Framework level에서 빈 조작을 하고자 할때 사용

    @Autowired
    public DataAccessAnnotationProcessor(ConfigurableListableBeanFactory configurableListableBeanFactory) {
        this.configurableListableBeanFactory = configurableListableBeanFactory;
    }

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        this.scanDataAccessAnnotation(bean, beanName);
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }

    protected void scanDataAccessAnnotation(Object bean, String beanName) {
        this.configureFieldInjection(bean);
    }

    private void configureFieldInjection(Object bean) {
        Class<?> clazz = bean.getClass();
        ReflectionUtils.FieldCallback fieldCallback =
                new DataAccessFieldCallback(configurableListableBeanFactory, bean);
        ReflectionUtils.doWithFields(clazz, fieldCallback); // target class에 있는 모든 field에 대해서 callback을 호출한다.
    }
}
```

```java
public class DataAccessFieldCallback implements ReflectionUtils.FieldCallback {

    private ConfigurableListableBeanFactory configurableListableBeanFactory;
    private Object bean;

    public DataAccessFieldCallback(ConfigurableListableBeanFactory configurableListableBeanFactory, Object bean) {
        this.configurableListableBeanFactory = configurableListableBeanFactory;
        this.bean = bean;
    }

    @Override
    public void doWith(Field field) throws IllegalArgumentException, IllegalAccessException {
        if (!field.isAnnotationPresent(DataAccess.class)) {
            return;
        }

        ReflectionUtils.makeAccessible(field);
        Type genericType = field.getGenericType();
        Class<?> generic = field.getType();
        Class<?> entity = field.getDeclaredAnnotation(DataAccess.class).entity();

        String beanName = entity.getSimpleName() + generic.getSimpleName();

        Object beanInstance = getBeanInstance(beanName, generic, entity);
        field.set(bean, beanInstance);
    }

    private Object getBeanInstance(String beanName, Class<?> generic, Class<?> entity) {
        Object daoInstance = null;

        if (configurableListableBeanFactory.containsBean(beanName)) {
            daoInstance = configurableListableBeanFactory.getBean(beanName);
        } else {
            Object toRegister;
            try {
                Constructor<?> constructor = generic.getConstructor(Class.class);
                toRegister = constructor.newInstance(entity);
            } catch (NoSuchMethodException | InstantiationException | IllegalAccessException | InvocationTargetException e) {
                throw new RuntimeException(e);
            }

            // 빈 초기화 및 등록
            daoInstance = configurableListableBeanFactory.initializeBean(toRegister, beanName);
            configurableListableBeanFactory.autowireBeanProperties(daoInstance, AutowireCapableBeanFactory.AUTOWIRE_BY_NAME, true);
            configurableListableBeanFactory.registerSingleton(beanName, daoInstance);
        }

        return daoInstance;
    }
}
```


### 정리
ConfigurableListableBeanFactory와 ReflectionUtils 기능을 활용하면, spring application boot time에 bean 객체 변경 가능하다...

### 참고 자료
- https://github.com/eugenp/tutorials/tree/master/spring-core-3
- https://www.baeldung.com/spring-annotation-bean-pre-processor
