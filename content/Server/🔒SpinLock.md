
Lock 구현에는 여러 메타가있습니다
그중에 존버메타인 SpinLock계열을 설명하려고합니다


---

# C# SpinLock 구현과 동작 원리

## 1. SpinLock 개념

`SpinLock`은 **잠금이 풀릴 때까지 반복적으로 확인 하는 방식**으로 동기화를 구현하는 
Lock 기법이다.  
잠금 해제 전까지 CPU를 계속 사용하며 루프를 도는 방식이라, **짧은 잠금 유지 시간**에는 효율적이지만, 오래 걸리면 성능 저하가 발생한다.

---

## 2. 코드 구현 (주석 포함)

```csharp
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
    
```csharp
int original = Interlocked.Exchange(ref isLocked, 1);
if (original == 0)
{
    // 락을 획득함
}
```


---

### 3.2 Interlocked.CompareExchange

- **CAS (Compare-And-Swap)** 기법.
    
- `isLocked` 값이 예상 값(`expected`)이면 새 값(`desired`)로 바꾸고, 그 전 값을 반환.
    
- 기대한 값과 일치하면 성공, 아니면 실패.
- CompareExchange 반환 값은 isLocked의 desired 삽입 전 값이다
- 결국 expected라면 아무도 건드리지 않았다는 로직
    

```csharp
int expected = 0;
int desired = 1;

if (expected == Interlocked.CompareExchange(ref isLocked, desired, expected))
{
    // 락 획득 성공
}
```



---

## 4. 사용 예시

```csharp
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

## 5. Spinlock Yield

- 자원 점유를 위해 지속적으로 기다리는 SpinLock과 달리
양보를 하거나 잠시 쉬었다가 다시 자원점유를 하려고 시도하는 방법이다

- 장점 : 경합부분에 작업하는 양이많다면 다른스레드가 지속적으로 기다리는 시간을 줄여줌으로
다른 작업을 하는 스레드의 일을 시킬수있다

- 단점 : 컨텍스트 스위칭작업이 빈번하게 발생되면 오버헤드 발생량이 높아짐으로 시간적자원 사용량이 커짐

```
  class SpinLock
  {
      volatile int isLocked = 0;

      //잠금
      public void Acquire()
      {
          while (true)
          {
              int expected = 0; //내가 예상한값
              int desired = 1; //내가 원하는값

              if (expected == Interlocked.CompareExchange(ref isLocked, desired, expected)) 
                  break;
              
              //쉬다 오는것
              //쉬다오는 방법이 세가지가 있다

              //Sleep Ms받는것 
              Thread.Sleep(1); //1 ms만큼 대기하겠다 무조껀 휴식 -> 명식상이렇게되어있지만 스켈줄러에의해 더 긴시간이 될수도있다

              Thread.Sleep(0); // 조건부 양보 -> 나보다 우선순위가 낮은 애들한테는 양보 불가 -> 우선순위가 나보다 같거나 높은 스레드가 없으면 다시 본인한테온다
                               // 우선순위란 : 직원마다 공평하게 우선순위가 같을수도있지만 아닐수도있다
                               // 우선순위가 낮다면 기아형상발생

              Thread.Yield();  // 관대한 양보 : 관대하게 양보할테니, 지금 실행이 가능한 쓰레드가 있다면 실행하세요 
                               // 실행가능한 애가 없으면 본인한테 남은시간 소진

              //장점 : 무한정으로 뺑뺑도는것을 방지
              //단점 : 컨텍스트 스위칭으로인한 오버헤드 발생, 소유권을 뺏길수도있다 

              //컨텍스트 스위칭 : 현재 스레드 상태를 저장하고 다음실행할 스레드 상태를 불러오고 동작시키는것

              //순서
              //1. 현재 스레드 상태 저장
              //  ->레지스터 값, 프로그램 카운터(PC), 스택 포인터(SP), 메모리 매핑 정보 등.

              //2. 다음 실행할 스레드 상태 불러오기
              //  -> 저장돼 있던 레지스터, PC, SP 복원.

              //3. 새 스레드 실행 시작
          }
      }

      //반환
      public void Release()
      {
          isLocked = 0;
      }
  }
```



---
## 6. Lock 구현 방식 비교

|방식|설명|장점|단점|
|---|---|---|---|
|**SpinLock (존버)**|잠금이 풀릴 때까지 계속 루프|짧은 작업에 효율적, 컨텍스트 스위칭 없음|오래 걸리면 CPU 점유율 급증|
|**Yield (랜덤)**|잠금 실패 시 CPU 점유권 반납|CPU 낭비 줄임|컨텍스트 스위칭 오버헤드|
|**Event (갑질)**|OS에 이벤트 등록 후 신호 받을 때까지 대기|CPU 낭비 거의 없음|이벤트 처리 비용, 구현 복잡|

---

💡 **정리:**  
SpinLock은 스레드나 프로세스의 경합 상황이 짧을때 유용하다
왜? ContextSwitching없이 락의 획득과 반환이 빠르게 이어지기 때문이다
반대로 말하면 경합상황이 길어진다면 락의 획득하는 시간이 길어지기에 cpu점유율상승

