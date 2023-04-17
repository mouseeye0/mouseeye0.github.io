---
title: Reactive Streams
author: mouseeye
date: 2023-01-19 15:00:00 +0900
categories: [reactor]
tags: [Reactive Streams]
---

### 반복자 패턴
* 다른 자료구조를 가지는 데이터소스에 통일된 방식으로 접근하기 위해 고안된 디자인 패턴
* Set, List 등이 Iterable 인턴페이스를 상속하며 `next()`, `hasNext()`가 있음
```java
public void iterable_test() {
      Iterator<Integer> iterator = List.of(1, 2, 3).iterator();
      while(iterator.hasNext()){
          Integer next=iterator.next();
          System.out.println(next);
      }
}
```
* hasNext()로 존재유무를 확인하고, next()로 ***데이터를 pull***함

### 옵저버 패턴
* Observable, Observer 두개의 구현체를 가지고 있음
* Observable에 Observer를 등록하고 Observable은 등록된 Observer에 ***Event를 push***함
```java
public class Observable {
  private Vector<Observer> obs = new Vector();

  public synchronized void addObserver(Observer o) {}
  public void notifyObservers() {}
  public synchronized void deleteObserver(Observer o) {}
}

public interface Observer {
  void update(Observable var1, Object var2);
}
```
*

### Reactive Streams 란?
* back pressure 기반으로 비동기 논블록킹 스트림을 처리 하는 표준 인터페이스
  * 초기 비동기 처리 기능들은 표준 없이 무분별하게 발전해 왔음
  * 그로인해, 서로 다른 진영의 라이브러리들 끼리 충돌이 일어났으며, 호환이 되지 않았음
  * 이러한 이슈를 해결하기 위해 나온게 `Reactive Streams` 표준!
* Reactive Streams는 옵저버 패턴 + 반복자 패턴의 조합을 활용하여 정의됨.

### 기존 옵저버 패턴의 한계
* 데이터 발행의 완료 발생 처리(Complete) / 에러 발생 처리 (Error) 기능이 없음
* Observable은 Observer의 처리 가능 유무를 알지 못함
  * 데이터를 무한히 발행하여 OOM이나 데이터 유실이 발생할 수 있음

### Reactive Streams Spec - 반복자/옵저버 패턴의 결합
* 반복자/옵저버 패턴을 결합하여 새로운 스펙을 정의함
  * **push-pull 하이브리드 모델** : 구독자가 처리 가능한 데이터 갯수 만큼만 pull 요청을 하고 발행자는 갯수 만큼 데이터를 push함
* https://github.com/reactive-streams/reactive-streams-jvm/tree/master/api/src/main/java/org/reactivestreams

```java
public interface Publisher<T> {
  public void subscribe(Subscriber<? super T> s);
}

public interface Subscriber<T> {
    public void onSubscribe(Subscription s);
    public void onNext(T t);
    public void onError(Throwable t);
    public void onComplete();
}

public interface Subscription {
    public void request(long n);
    public void cancel();
}
```
* https://engineering.linecorp.com/wp-content/uploads/2020/02/reactivestreams1-10.png
* 각 인터페이스의 역할은 다음과 같다.
  * Publisher는 데이터 발행 / Subscliber는 데이터 소비 / Subscription은 두 인터페이스간에 연결
* Subscriber는 Subscription을 통해 실제 소비할 (소비 가능한) 데이터 갯수를 전달한다.
  * Subscription.request(n) / back pressure의 역핧
* 데이터 실행의 흐름은 다음과 같다.
  * onSubscribe onNext* (onError | onComplete)?
  * onSubscribe는 Subscriber가 Pubslisher를 구독할 때 최초 한번
  * onNext는 Publisher가 발행할 데이터 갯수 n번
  * 실행 완료 시 onComplete | 실행 실패 시 onError가 상호 배타적으로 실행된다.

### 구현
```java
    public static class PublisherImpl implements Publisher<Integer> {

        private final Queue<Integer> queue;

        public PublisherImpl() {
            Queue<Integer> blockingQueue = new LinkedBlockingQueue<>();
            blockingQueue.add(1);
            blockingQueue.add(2);
            blockingQueue.add(3);
            this.queue = blockingQueue;
        }

        @Override
        public void subscribe(Subscriber<? super Integer> subscriber) {
            subscriber.onSubscribe(new Subscription() {
                @Override
                public void request(long n) {
                    try {
                        for (int i = 0; i < n; n--) {
                            if (queue.isEmpty()) {
                                subscriber.onComplete();
                            } else {
                                subscriber.onNext(queue.poll());
                            }
                        }
                    } catch (Exception e) {
                        subscriber.onError(e);
                    }
                }

                @Override
                public void cancel() {

                }
            });
        }
    }
```

```java
    public static class SubscriberImpl implements Subscriber<Integer> {

        private final Integer requestWindowSize;

        private Subscription subscription;

        private final AtomicInteger countWindowSize = new AtomicInteger();

        public SubscriberImpl(Integer requestWindowSize) {
            this.requestWindowSize = requestWindowSize;
        }

        @Override
        public void onSubscribe(Subscription s) {
            this.subscription = s;
            s.request(requestWindowSize);
        }

        @Override
        public void onNext(Integer data) {
            System.out.println(data);
            int currentRequestSize = countWindowSize.incrementAndGet();
            if (currentRequestSize == requestWindowSize) {
                System.out.println("--WINDOW END--");
                countWindowSize.setPlain(0);
                subscription.request(requestWindowSize);
            }
        }

        @Override
        public void onError(Throwable t) {
            t.printStackTrace();
        }

        @Override
        public void onComplete() {
            System.out.println("COMPLETED");
        }
    }
```

### 왜 Reactive Streams 여야 하는가? (장점)
* ***프로그램 성능을 향상*** 시키기 위해서
  * ***비동기를 활용하여 병렬 처리***를 하여 컴퓨팅 리소스를 효율적으로 사용할 수 있다.
* 자바 표준 라이브러리에서는 병렬 처리를 위해서 ***Future, CompletableFuture 등 자바 내장 객체***가 제공 된다.
* 하지만, 사용에 몇가지 제약이 있다.
  * 콜백 지옥으로 읽기도 어렵고, 유지보수 하기 힘든 코드를 만들어 내기 쉽다.
  * 블락킹 호출 (Future.get()은 Callable이 데이터를 생성할 때까지 blocking된다.)
* Reactor는 이러한 제약 사항을 해결하였다. (Reactive Streams 스펙을 따르는...)
  * 기존 Future / CompletableFuture 보다 ***비동기 연결 처리를 구성하기 쉽고 가독성이 향상***된다.
  * ***풍부한 연결 연산자를 제공***한다.
  * reactor-netty 프로젝트 지원 : 병목이 발생하는 I/O 연산에 대해서 논블록킹 방식으로 호출 가능
* https://projectreactor.io/docs/core/release/reference/#_from_imperative_to_reactive_programming


### 참고자료
* http://www.reactive-streams.org/
* https://sjh836.tistory.com/182
* https://m.youtube.com/playlist?list=PLOLeoJ50I1kkqC4FuEztT__3xKSfR2fpw
* https://engineering.linecorp.com/ko/blog/reactive-streams-with-armeria-1/


