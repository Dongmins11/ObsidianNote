
Lock 구현에는 여러 메타가있습니다
그중에 존버메타인 SpinLock계열을 설명하려고합니다


---

# C# SpinLock 구현과 동작 원리

## 1. SpinLock 개념

`SpinLock`은 **잠금이 풀릴 때까지 반복적으로 확인 하는 방식**으로 동기화를 구현하는 Lock 기법이다.  
잠금 해제 전까지 CPU를 계속 사용하며 루프를 도는 방식이라, **짧은 잠금 유지 시간**에는 효율적이지만, 오래 걸리면 성능 저하가 발생한다.

---

## 2. 코드 구현 (주석 포함)

[!NOTE]- 전체코드

```
using System;
using System.Threading;
using System.Threading.Tasks;


namespace ServerCore
{
    class SpinLock
    {
        // 상태
        // true - 잠김 / false - 잠기지 않음
        // volatile: 가시성 보장 (다른 스레드에서 변경된 값 바로 보이게)
        // volatile bool isLocked = false;

        volatile int isLocked = 0; // int로 하면 Interlocked 계열 함수 사용 가능

        // 잠금 획득
        public void Acquire()
        {
            // 옛날 방식
            // while (isLocked) { }  // 뺑뺑이 돌며 기다림
            // isLocked = true;

            // 문제점: while 끝나고 isLocked = true 사이에 다른 스레드가 끼어들 수
            있음 → 원자적 처리 필요

            // --------------------------------------------------------------
            // 해결책 1: Interlocked.Exchange
            // --------------------------------------------------------------
            // int original = Interlocked.Exchange(ref isLocked, 1);
            // if (original == 0) → 아무도 점유 안 함, 내가 가져감

            // --------------------------------------------------------------
            // 해결책 2: Interlocked.CompareExchange (CAS, Compare-And-Swap)
            // --------------------------------------------------------------
            while (true)
            {
                int expected = 0; // 내가 예상한 값 (잠금 안 걸린 상태)
                int desired = 1;  // 내가 원하는 값 (잠금 상태)

                // CompareExchange:
                //  isLocked == expected → isLocked = desired 로 바꾸고 기존
                값 반환
                //  아니면 아무것도 안 하고 기존 값 반환
                if (expected == Interlocked.CompareExchange(ref isLocked, desired, expected))
                    break; // 성공하면 나가기
            }
        }

        // 잠금 해제
        public void Release()
        {
            // 내가 이미 자원을 점유하고 있으니 그냥 0으로 바꿔도 됨
            isLocked = 0;
        }
    }

    class Program
    {
        static int num = 0;
        static SpinLock Lock = new SpinLock();

        static void Thread_1()
        {
            for (int i = 0; i < 1000000; ++i)
            {
                Lock.Acquire();
                num++;
                Lock.Release();
            }
        }

        static void Thread_2()
        {
            for (int i = 0; i < 1000000; ++i)
            {
                Lock.Acquire();
                num--;
                Lock.Release();
            }
        }

        static void Main(string[] args)
        {
            Task t1 = new Task(Thread_1);
            Task t2 = new Task(Thread_2);

            t1.Start();
            t2.Start();

            Task.WaitAll(t1, t2);

            Console.WriteLine(num);
        }
    }
}
```

---


## 3. 핵심 동작 원리

### 3.1 Interlocked.Exchange

- **값 교환 (Swap)** 을 원자적으로 수행.
    
- 기존 값을 반환하기 때문에, 반환값이 0이면 락이 비어있는 상태.
    
```
int original = Interlocked.Exchange(ref isLocked, 1);
if (original == 0)
{
    // 락을 획득함
}
```


---

### 3.2 Interlocked.CompareExchange

- **CAS (Compare-And-Swap)** 기법.
    
- `isLocked` 값이 예상 값(`expected`)이면 새 값(`desired`)으로 바꾸고, 그 전 값을 반환.
    
- 기대한 값과 일치하면 성공, 아니면 실패.
    

```
int expected = 0;
int desired = 1;

if (expected == Interlocked.CompareExchange(ref isLocked, desired, expected))
{
    // 락 획득 성공
}
```



---

## 4. 사용 예시

```
class Program
{
    static int num = 0;
    static SpinLock Lock = new SpinLock();

    static void Thread_1()
    {
        for (int i = 0; i < 1000000; ++i)
        {
            Lock.Acquire();
            num++;
            Lock.Release();
        }
    }

    static void Thread_2()
    {
        for (int i = 0; i < 1000000; ++i)
        {
            Lock.Acquire();
            num--;
            Lock.Release();
        }
    }

    static void Main(string[] args)
    {
        Task t1 = new Task(Thread_1);
        Task t2 = new Task(Thread_2);

        t1.Start();
        t2.Start();

        Task.WaitAll(t1, t2);

        Console.WriteLine(num); // 항상 0 출력
    }
}`
```

---

## 5. Lock 구현 방식 비교

|방식|설명|장점|단점|
|---|---|---|---|
|**SpinLock (존버)**|잠금이 풀릴 때까지 계속 루프|짧은 작업에 효율적, 컨텍스트 스위칭 없음|오래 걸리면 CPU 점유율 급증|
|**Yield (랜덤)**|잠금 실패 시 CPU 점유권 반납|CPU 낭비 줄임|컨텍스트 스위칭 오버헤드|
|**Event (갑질)**|OS에 이벤트 등록 후 신호 받을 때까지 대기|CPU 낭비 거의 없음|이벤트 처리 비용, 구현 복잡|

---

💡 **정리:**  
SpinLock은 간단하고 빠르지만, 긴 작업에는 CPU 낭비가 크므로 주의가 필요하다.  
짧고 빈번한 잠금 해제 상황에서 성능이 좋으며, 장기 대기 상황에서는 `Monitor`, `Mutex` 또는 이벤트 기반 잠금을 사용하는 것이 좋다.