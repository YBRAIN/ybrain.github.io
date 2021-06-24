---
publish: true
title: "RxJava를 이용한 뇌파 데이터 스트리밍"
author: Sangwoo Maeng
categories:
  - Jekyll
tags: 
  - YBRAIN
  - MINDD SCAN
  - RxJava
  - Kotlin
  - JavaFx  
  - 뇌파
  - EEG  
--- 


![](/assets/images/mindd_scan.jpg)

> 이 글은 독자의 ReactiveX에 대한 기본적인 이해를 가정합니다.

MINDD SCAN은 와이브레인에서 개발한 Desktop용 정량뇌파측정용 의료기기 소프트웨어입니다.
올해 초에 Major 업데이트로 출시한 MINDD SCAN은 사실 거의 모든 부분에서 RxJava를 활용했습니다.
그 중 뇌파(EEG) 스트리밍 처리에서 RxJava를 유용하게 활용한 사례를 간단히 소개합니다.

> **EEG란** 뇌파신호기술을 말하는 영어약자입니다.  
> 무려 electroencephalography 라고 하네요..ㅎㄷㄷ(의학영어 울렁증)  
> 뜯어보면 아래와 같습니다.
>
> - electro (전기적)
> - en (내부의)
> - cephalo (머리의, 두뇌의)
> - graphy (표현)
>
> 한마디로 **머리 내부의 전기적 표현**이군요.

## 뇌파 데이터의 흐름
**뇌파 측정기**에서 올라오는 뇌파데이터는 아래 그림처럼 다양한 경로를 통해 실시간으로 전달되어 처리됩니다.

<div class="mermaid"> 
graph LR;
    a[뇌파측정기];
    b[FFT 필터];
    c[수신율 계산];
    d[신호분석기 by RPC];
    e[녹화기];
    f[연결끊김 감지];
    g[뇌파 View];

    a-->b;
    a-->c;
    b-->d;
    b-->e;
    b-->g;
    c-->f;
    f-->g;
    d-->g;
    e-->g;
</div>
 
  
위 실시간 데이터 흐름에서 한가지 중요한 점은 결국 뇌파 파형을 그려주는 View에 전달된다는 점입니다.
즉, UI Thread에서 모든 일을 처리했다가는 뇌파 파형이 엄청나게 뚝뚝 끊겨 그려지게 되겠죠.
따라서 위 그림의 각 처리기를 원하는 Thread에서 처리할 수 있도록 RxJava의 `observeOn()` 을 활용해 처리했습니다.

> 만약 RxJava를 안썼다면 직접 Thread띄우고, Task queue 처리하느라 많은 노력과 시간이 들었을거에요. 생각만해도 끔찍합니다;;

## 뇌파 측정기
그림에서 **뇌파측정기**는 실제 뇌파측정용 모듈에서 들어오는 신호를 받아 Rx의 Observable 로 스트리밍해줍니다.
실제 기기 연결상태에 따른 분기처리를 생략하기 위해 Hot-observable로 구성했습니다.
RxJava에서는 `Subject`라고 부르죠. 의도치 않은 종료를 방지하도록 `Subject`를 특별처리한 Jake Wharton 형님의 RxRelay를 사용했습니다.
아래 코드는 모든 뇌파신호의 시초가되는 **뇌파측정기** 부분입니다.

```kotlin
fun observeSignals(): Observable<SignalChunk> = signalRelay.hide()        
```

## 수신율 계산 처리기
뇌파측정기는 무선으로 뇌파신호를 전송하기 때문에 무선환경에 따라 Packet-loss 가 발생할 수 있습니다.  
(녹화시에는 뇌파측정기의 자체 메모리에도 저장하므로 나중에 복구하게 됩니다.)

이에 따라 수신율을 화면에 표시해주게 되는데요. 아래와 같은 로직으로 계산합니다.
1. 1초동안 수신한 뇌파샘플의 묶음인 Chunk를 수집
2. 뇌파 측정기의 Sampling rate 대비 몇 개의 샘플을 받았는지 퍼센트로 계산 
3. 급격한 수신율 변화를 방지하기 위해 10초 이동평균으로 계산

RxJava의 `buffer()` 와 `map` 함수를 사용하면 아래와 같이 간단히 파이프라인으로 처리할 수 있습니다.

```kotlin 
 fun startSignalRxRateCalculation() = observeSignals()
        // 1초 동안 패킷을 모음        
        .buffer(
            1,
            TimeUnit.SECONDS,            
            Schedulers.newThread()
        )
        
        // 패킷안에 든 샘플 수를 SampleRate으로 나눠 Percentage로 계산
        .map { receivedSignals ->
            val numOfSamples = receivedSignals.size * ScanDeviceInfo.signalChunkSize
            return@map (numOfSamples * 100 / ScanDeviceInfo.eegSamplingRateHz)
                .coerceAtMost(100)
        }        
        
         //  10개씩 모으되 1개씩 건너뜀. 1~10, 2~11, 3~12 와 같이 묶어 들어옴
        .buffer(10, 1)  
        
         // 10개 리스트에 대한 평균
        .map { it.average().toInt() }
        
        // 신호 수신율을 출력하는 Relay로 전달
        .subscribe { signalRxRateRelay.accept(it) }  
```

## FFT필터 처리기
뇌파 측정이란 결국 두뇌에서 나오는 아주 미세한 전기신호를 증폭해 기록하는 것입니다.  
전기신호를 증폭하다보니 우리 주변의 전자기기에서 나오는 60Hz의 전력 잡음이 뇌에서 나오는 신호보다 훨씬 강력합니다.
따라서 60Hz의 주파수 성분을 제거해야 하는데 이때 FFT(Fast Fourier Transform)를 이용한 필터를 사용합니다.

문제는 신호분석기, 녹화기, 뇌파View 이렇게 세가지 처리기가 필터링된 신호를 사용하지만 사용하는 시점은 제 각각입니다.
각 처리기마다 따로 필터링을 하자니 CPU 비용이 많이 들어갑니다.
해결 방법은 **FFT필터 처리기**를 하나만 돌리는 것입니다.
위의 세가지 처리기 중 하나 이상 동작한다면 **FFT필터 처리기**가 동작해 필터된 데이터를 공유받고
그렇지 않다면 **FFT필터 처리기**가 멈췄으면 좋겠습니다.

RxJava의 `ConnectableObservable`이 정확이 이런 동작을 할 수 있습니다.
`Observable`의 `publish()`함수를 이용해 `ConnectableObservable`로 변환하고
`refCount()`를 이용해 하나 이상의 Subscriber가 연결되면 동작한 후 모두 연결해제되면 Dispose 되도록 합니다. 

일단 `ConnectableObservable`의 기본적인 동작을 시험해 보기 위해 아래 코드를 실행해 봤습니다.
```kotlin
import io.reactivex.Observable
import io.reactivex.schedulers.Schedulers
import java.util.concurrent.TimeUnit

// 1초 마다 숫자 만들어내는 Observable
val connectable= Observable.interval(0, 1, TimeUnit.SECONDS)
    .doOnSubscribe { println("doOnSubscribe") }
    .doOnDispose { println("doOnDispose") }
    .publish()
    .refCount()

fun main() {
    // 첫 번째 Subscription. 3초동안 3개를 받고 Dispose 한다. 
    connectable
        .observeOn(Schedulers.newThread())
        .take(3)
        .subscribe { println("A:$it") }

    // 첫 번째와 두 번째 Subscription이 동시에 일어날 때와 아닐때를 각각 비교해보기 
    Thread.sleep(1000)
    //Thread.sleep(4000)

    // 두 번째 Subscription. 
    connectable
        .observeOn(Schedulers.newThread())
        .take(3)
        .subscribe { println("B:$it") }

    Thread.sleep(5000)
}
```

결과는 아래와 같습니다.
```
doOnSubscribe
A:0
A:1
A:2
B:2
B:3
B:4
doOnDispose
```

두 개의 Subscription이 동시에 진행되어도 원래의 Observable은 계속 유지됨을 확인할 수 있습니다.
중간에 sleep time을 바꿔서 어떤 결과가 나오는지 실험해보세요.

이제 **FFT필터 처리기**를 `ConnectableObservable` 이용해 구현하고 필요한 곳에서 `filteredEegSignal`을 Subscribe하면 됩니다.

```kotlin
val filteredEegSignal = observeSignals()
        .map { filterSignals(it, eegFilter) }        
        .publish()
        .refCount()        
```

## 마무리
조금 과장하면 제 개발인생은 ReactiveX를 알기 전과 후로 나뉘는 것 같습니다.
손으로 구현하기 번거로운 코드들을 한 줄로 깔끔하게 처리할 수 있다는 점이 개인적으로 RxJava의 가장 큰 매력입니다. 
포스팅에 잘못된 점이 있다면 언제든 지적해주세요.
곧 댓글 서비스 밑에 달 예정입니다.
