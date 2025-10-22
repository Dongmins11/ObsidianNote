
## 1. 개념

### AutoResetEvent

- **문이 자동으로 잠기는 톨게이트**
    
- 스레드가 `WaitOne()`으로 입장 시도 → 입장 성공 후 **자동으로 문이 닫힘**
    
- 한 번에 **하나의 스레드만** 통과 가능
    
- `Set()` 호출 시 문을 열어줌 (`flag = true`)
    
- 스레드가 지나가면 자동으로 다시 잠김 (`flag = false`)
    

### ManualResetEvent

- **문이 수동으로 잠기는 방식**
    
- `Set()`으로 한 번 문을 열면, **닫을 때까지(`Reset()` 호출)** 여러 스레드가 동시에 들어올 수 있음
    
- 로딩 작업이 끝난 뒤, 대기 중이던 스레드를 한 번에 입장시킬 때 유용
    

---

## 2. SpinLock과 Event 차이

|항목|SpinLock|Auto/ManualResetEvent|
|---|---|---|
|**동작 위치**|유저 모드(while 루프)|커널 모드(이벤트 객체)|
|**대기 방식**|계속 CPU 점유(바쁜 대기)|OS에서 스레드를 대기 상태로 전환|
|**응답 속도**|매우 빠름(컨텍스트 스위칭 없음)|커널 호출로 인해 상대적으로 느림|
|**사용 예시**|아주 짧은 락 대기|긴 작업 대기, 스레드 간 시그널 전달|

---

## 3. 코드 예제

```csharp 
using System;
using System.Threading;
using System.Threading.Tasks;

namespace ServerCore
{
    class Lock
    {
        // 톨게이트: true = 문 열림 / false = 문 닫힘
        AutoResetEvent available = new AutoResetEvent(true);
        // ManualResetEvent available = new ManualResetEvent(true);

        public void Acquire()
        {
            // 입장 시도 (문이 닫혀 있으면 대기)
            available.WaitOne();
        }

        public void Release()
        {
            // 문을 열어줌
            available.Set();
        }
    }

    class Program_Yield
    {
        static int num = 0;
        static Lock Lock = new Lock();

        static void Thread_1()
        {
            for (int i = 0; i < 10000; ++i)
            {
                Lock.Acquire();
                num++;
                Lock.Release();
            }
        }

        static void Thread_2()
        {
            for (int i = 0; i < 10000; ++i)
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

## 4. 톨게이트 비유로 이해하기

- **AutoResetEvent**  
    → 톨게이트를 한 명이 지나가면 **자동으로 문이 잠김**  
    → 다음 사람이 지나가려면 톨게이트 관리자가 `Set()`으로 다시 열어줘야 함
    
- **ManualResetEvent**  
    → 톨게이트를 열면 **여러 명이 동시에** 지나갈 수 있음  
    → `Reset()` 호출 전까지 계속 열려 있음
    

---

## 5. 사용 시 주의점

- `AutoResetEvent`는 짧은 임계 구역 보호에 적합
    
- `ManualResetEvent`는 다수의 스레드를 한 번에 깨워야 할 때 유리
    
- SpinLock 대비 커널 모드 전환 비용이 있기 때문에, **긴 대기**나 **대량의 스레드 제어**가 필요한 경우에 사용
    

---

💡 **정리:**

- SpinLock → 짧고 빠른 락, CPU 점유
    
- Auto/ManualResetEvent → 커널에서 대기, CPU 점유율 낮음, 상대적으로 느림
    
- 상황에 맞게 선택하는 것이 중요!