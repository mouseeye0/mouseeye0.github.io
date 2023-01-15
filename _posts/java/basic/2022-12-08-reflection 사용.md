---
title: Java Reflection 사용
author: mouseeye
date: 2022-12-08 15:00:00 +0900
categories: [Java]
tags: [Java, Reflection]
---

Reflection을 사용하면 Java Object를 Runtime에 클래스의 메타데이터를 확인 할 수 있고, 조작 또한 가능하다.
사용 가능한 Pattern 들을 정리 해보자.

### 1. 준비 하기

#### Eating Interface
```java
public interface Eating {
    String eats();
}
```

#### Animal Implementation
```java
public abstract class Animal implements Eating {
    public static String CATEGORY = "domestic";
    private String name;
    protected abstract String getSound();

    public Animal(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```

#### Locomotion Interface
```java
public interface Locomotion {
    String getLocomotion();
}
```

#### Final Concrete Class
```java
public class Goat extends Animal implements Locomotion {
  private String dummy = "dummy";

  public Goat(String name) {
    super(name);
  }

  public Goat() {
    super("goat");
  }

  @Override
  protected String getSound() {
    return "bleat";
  }

  @Override
  public String eats() {
    return "grass";
  }

  @Override
  public String getLocomotion() {
    return "walks";
  }
}
```

### 2. Class Names
```java
  @Test
  public void test_classNames() throws ClassNotFoundException {
    Object person = new Goat("goat");
    Class<?> clazz = person.getClass();
    // Class<?> clazz = Class.forName("me.mouseeye.reflection.Goat"); // full class name으로 Class 객체 생성

    assertEquals("Goat", clazz.getSimpleName());
    assertEquals("me.mouseeye.reflection.Goat", clazz.getName());
    assertEquals("me.mouseeye.reflection.Goat", clazz.getCanonicalName());
  }
```
- getSimpleName() : Class 선언에 명시된 Name
- getName() | getCanonicalName() : 패키지명이 포함된 Class Name


### 3. Class Modifier
```java
    @Test
    public void test_classModifier() throws ClassNotFoundException {
        Class<?> goatClazz = Class.forName("me.mouseeye.reflection.Goat");
        Class<?> animalClazz = Class.forName("me.mouseeye.reflection.Animal");

        int goatModifiers = goatClazz.getModifiers();
        int animalModifiers = animalClazz.getModifiers();

        assertTrue(Modifier.isPublic(goatModifiers));
        assertTrue(Modifier.isPublic(animalModifiers));
        assertTrue(Modifier.isAbstract(animalModifiers));
    }
```
- getModifiers() : Modifier 정보 추출 / 추출된 결과값은 int 값을 가지고 있음
- Modifier.isXXX : 추출된 값을 메소드로 검증 가능

### 4. Package Information
```java
    @Test
    public void test_packageName() throws ClassNotFoundException {
        // given
        Class<?> clazz = Class.forName("me.mouseeye.reflection.Goat");

        // when
        String packageName1 = clazz.getPackage().getName();
        String packageName2 = clazz.getPackageName();

        // then
        assertEquals("me.mouseeye.reflection", packageName1);
        assertEquals("me.mouseeye.reflection", packageName2);
    }
```
- getPackage() : package 관련 정보 확인 가능


### 5. Superclass
```java
    @Test
    public void test_subclass() {
        Class<? extends Goat> goatClazz = Goat.class;
        Class<?> superclass = goatClazz.getSuperclass();

        assertEquals("Animal", superclass.getSimpleName());
        assertEquals("Object", superclass.getSuperclass().getSimpleName());
    }
```
- getSuperclass() : 부모 클래스 확인

### 6. Implemented Interfaces
```java
    @Test
    public void test_implemented() {
        Class<Goat> goatClass = Goat.class;
        Class<Animal> animalClass = Animal.class;

        Class<?>[] goatClassInterfaces = goatClass.getInterfaces();
        Class<?>[] animalClassInterfaces = animalClass.getInterfaces();

        assertEquals(1, goatClassInterfaces.length);
        assertEquals(1, animalClassInterfaces.length);

        assertEquals("Locomotion", goatClassInterfaces[0].getSimpleName());
        assertEquals("Eating", animalClassInterfaces[0].getSimpleName());
    }
```
- getInterfaces() : 구현체에서 직접적으로 implements 되어 있는 Interface 배열 반환

### 7. Constructors
```java
    @Test
    public void test_constructor() throws ClassNotFoundException, NoSuchMethodException, InvocationTargetException, InstantiationException, IllegalAccessException {
        Class<?> goatClass = Class.forName("me.mouseeye.reflection.Goat");

        Constructor<?>[] constructors = goatClass.getConstructors();
        assertEquals(2, constructors.length);

        Constructor<?> defaultConstructor = goatClass.getConstructor();
        Constructor<?> stringConstructor = goatClass.getConstructor(String.class);

        Goat goatWithDefaultConstructor = (Goat) defaultConstructor.newInstance();
        Goat goatWithStringConstructor = (Goat) stringConstructor.newInstance("string goat");

        assertEquals("goat", goatWithDefaultConstructor.getName());
        assertEquals("string goat", goatWithStringConstructor.getName());
    }
```
- getConstructor(Class<?> parameters...) : 매개변수로 선언된 타입, 순서가 일치하는 생성자 객체 반환

### 8. Methods
```java
    @Test
public void givenClass_whenGetsMethods_thenCorrect() throws Exception {
  // getDeclaredMethods()
  Class<?> animalClass = Animal.class;
        Method[] methods = animalClass.getDeclaredMethods();
          List<String> actualMethods = Arrays.stream(methods).map(Method::getName).toList();

  assertEquals(3, actualMethods.size());
  assertTrue(
  actualMethods.containsAll(Arrays.asList("getName", "setName", "getSound"))
  );


  // getMethods();
  Class<?> gaotClass = Goat.class;
        Method[] goatMethods = gaotClass.getMethods();
          List<String> goatMethodNames = Arrays.stream(goatMethods).map(Method::getName).toList();

  assertTrue(
  goatMethodNames.containsAll(List.of("toString", "equals", "hashCode", "eats", "getName"))
  );

  // Method invoke
  Class<?> goatClass = Goat.class;
        Goat bird = (Goat) goatClass.getConstructor().newInstance();
          Method setName = goatClass.getMethod("setName", String.class);
  Method getName = goatClass.getMethod("getName");
  String name = (String) getName.invoke(bird);

  assertEquals("goat", name);
  assertEquals("goat", bird.getName());

  setName.invoke(bird, "setGoat");
  String name2 = (String) getName.invoke(bird);

  assertEquals("setGoat", name2);
  assertEquals("setGoat", bird.getName());
  }
```
- getDeclaredMethods() :  자신의 public, protected, default method만 반환
- getMethods() : 자신과 부모 클래스 포함 public method 반환
- method.invoke(Object obj, Object.. params) : 메소드 호출


### 9. Fields
```java
    @Test
    public void test_field() throws Exception {
        // getFields()
        Class<?> goatClazz = Class.forName("me.mouseeye.reflection.Goat");

        Field[] fields = goatClazz.getFields();
        assertEquals(1, fields.length);
        assertEquals("CATEGORY", fields[0].getName());

        // getDeclaredFields()
        Field[] declaredFields = goatClazz.getDeclaredFields();
        assertEquals(1, declaredFields.length);
        assertEquals("dummy", declaredFields[0].getName());


        // get value from field object
        Goat goat = Goat.class.getConstructor().newInstance();
        Field dummy = goatClazz.getDeclaredField("dummy");
        dummy.setAccessible(true); // private field의 경우 setAccessible(true) 호출
        assertEquals("dummy", dummy.get(goat));

        // get value from static field
        Field category = goatClazz.getField("CATEGORY");
        assertEquals("domestic", category.get(null)); // static field 호출
    }
```
- getFields() : 자신과 부모 클래스 포함 public field 반환
- getDeclaredFields() : 자신의 public, protected, default, private field 반환
- field.get(Object obj)
  - field 객체에 실제 인스턴스를 get method로 호출 하면, field의 값을 얻을 수 있다.
  - 이때 private field의 경우 setAccessible(true) 호출이 선행되어야 한다.
  - static field의 경우 get method 호출 시에 obj에 null을 넘겨주면 된다.

### 참고 자료
- https://www.baeldung.com/java-reflection
