## 1. 들어가며


경합조건에서는  `lock` 키워드로 막아버리면 된다라고 생각하지만
게임에서는 성능이 생명이라 **모든 접근을 막는 건 낭비**일 수 있다.  

예를 들어 **여러 스레드가 동시에 읽기만** 하는 경우는 안전한데,
굳이 한 줄씩 줄 세우게 할 필요가 없다.

그래서 등장하는 게 **Read-Write Lock**이다.

- **Read는 동시에 여러 명 OK**
    
- **Write는 오직 한 명만, 독점!**
    

멀티스레드로직이있는데 Read로직이 99.999%,  Write로직이 0.0001%로 발생한다 가정한다
그러면 Read만 주구장창하는데 굳이 lock을 계속해서 쓸필요가 없다 성능의 이점을 챙기려면
write할때만 read도 lock에 걸리는 방식으로 하면 성능적으로 이득을 보게된다

즉, **Read-Write Lock은 “읽을 땐 자유롭게, 쓸 땐 독점적으로”**라는 규칙으로 성능과 안전성을 모두 챙길 수 있는 도구다.

---

## 2. Read-Write Lock 기본 개념

- **Read Lock**:  
    여러 스레드가 동시에 **읽기만** 하는 것은 안전하므로, 여러 개가 동시에 들어올 수 있습니다.
    
- **Write Lock**:  
    데이터를 수정할 때는 **단 한 스레드만** 접근할 수 있도록 보장해야 합니다.
    
- **제약**:
    
    - 쓰는 동안에는 아무도 읽거나 쓸 수 없음.
        
    - 읽는 중에는 다른 읽기 스레드는 허용, 하지만 쓰기는 불가능.
        
    - 재귀적 락(Recursive Lock) 지원 여부는 구현에 따라 다름.
        

---

## 3. 구현 아이디어

### 3.1 Flag 구조

락 상태를 나타내기 위해 **32비트 int** 하나를 사용합니다.

`[Unused(1bit)] [WriteThreadId(15bit)] [ReadCount(16bit)]`

- **ReadCount (하위 16비트)** : 현재 몇 개의 Reader가 접근 중인지 저장
    
- **WriteThreadId (중간 15비트)** : WriteLock을 잡은 Thread ID 저장
    
- **Unused (상위 1비트)** : 예비 비트(추후 기능 확장용)
    

```

//변수 소개

        const int EMPTY_FLAG = 0x00000000;
        const int WRITE_MASK = 0x7FFF0000;
        const int READ_MASK = 0x0000FFFF;
        const int MAX_SPIN_COUNT = 5000;

        // [Unused(1 bit)]    [WriteThreadId(15 bit)]    [ReadCount(16 bit)]

        // ReadCount -> 우리가 ReadLock획득하면 여러애들이 Read할수있으니 카운팅하는거고
        // Write -> 한번에 한 스레드만 획득할수있다 그친구가 누구인지 id저장
        int flag = EMPTY_FLAG;

        //플래그가 필요없는이유 
        //누군가가 write를 잡았다는건 다른애는없다 오직 한놈만 잡는것
        int writeCount = 0;

```


👉 장점: int 하나로 관리할 수 있어 `Interlocked.CompareExchange` 같은 **원자적 연산**을 활용할 수 있습니다.

### 3.2 Write Lock

```csharp
public void WriteLock()
{
    // 재귀 락
    // 동일 쓰레드가 writeLock을 이미 획득하고 있는지 확인
    int lockThreadId = (flag & WRITE_MASK) >> 16;
    if (Thread.CurrentThread.ManagedThreadId == lockThreadId)
    {
        writeCount++;
        return;
    }


    //----------------------------------------

    //아무도 WriteLock or ReadLock을 획득하고 있지 않을 때, 경합해서 소유권을 얻는다.

    // Thread.CurrentThread.ManagedThreadId == 본인 스레드 아이디
    // << 16 해주는 이유 WRTIE_MASK가 16비트이후부터 값을쓰기에 밀어주는것
    // & 연산을 통해서 WRITE_MASK 값이 아닌것을 다 밀어준다 혹시 모르니

    // 내 스레드 아이디만 비트에 들어가길 원하니 desired가 된것
    int desired = (Thread.CurrentThread.ManagedThreadId << 16) & WRITE_MASK;
    while(true)
    {
        for(int i =0; i < MAX_SPIN_COUNT; i++)
        {
            if (EMPTY_FLAG == Interlocked.CompareExchange(ref flag, desired, EMPTY_FLAG))
            {
                writeCount = 1;
                return;
            }
        }

        Thread.Yield();
    }
}
```

- **재귀 락 허용**: 동일 스레드가 여러 번 WriteLock을 잡을 수 있음.
    
- 다른 스레드가 쓰고 있거나 읽고 있으면 실패 → 스핀 후 재시도.

---


### 3.3 Read Lock

``` csharp
public void ReadLock()
{
    //@@write@@가 이미 잡고있을때만 readCount를 늘리는것
    int lockThreadId = (flag & WRITE_MASK) >> 16;
    if (Thread.CurrentThread.ManagedThreadId == lockThreadId)
    {
        Interlocked.Increment(ref flag);
        return;
    }


    // 아무도 WriteLock을 획득하고 있지 않으면, ReadCount를 1 늘린다
    while (true)
    {
        for(int i =0; i < MAX_SPIN_COUNT; i++)
        {
            int expected = (flag & READ_MASK);
            if (Interlocked.CompareExchange(ref flag, expected + 1, expected) == expected)
                return;

        }

        Thread.Yield();
    }
}




```


- 핵심: `expected = flag & READ_MASK`
    
    - **라이터가 있으면 상위 15비트가 채워져 있어 `flag != expected` → CAS 실패 → 읽기 차단**
        
    - **라이터가 없으면 `flag == expected` → CAS 성공 → ReadCount +1**
        
- 여러 Reader가 동시에 들어와도 CAS로 **원자적 증가** 보장.
    

---

### 3.4 Unlock

``` csharp
public void WriteUnLock()
{
    int lockCount = --writeCount;
    if (lockCount == 0)
        Interlocked.Exchange(ref flag, EMPTY_FLAG);
}

public void ReadUnLock()
{
    Interlocked.Decrement(ref flag);
}

```


- Write는 재귀 횟수를 체크하고 0이 되면 완전히 해제.
    
- Read는 단순히 카운트 감소.

---
## 4. 테스트 코드

``` csharp
static volatile int count = 0;
static Lock _lock = new Lock();

static void Main(string[] args)
{
    Task t1 = new Task(() =>
    {
        for (int i = 0; i < 100000; ++i)
        {
            _lock.ReadLock();
            count++;
            _lock.ReadUnLock();
        }
    });

    Task t2 = new Task(() =>
    {
        for (int i = 0; i < 100000; ++i)
        {
            _lock.ReadLock();
            count--;
            _lock.ReadUnLock();
        }
    });

    t1.Start();
    t2.Start();

    Task.WaitAll(t1, t2);

    Console.WriteLine(count);
}

```


---

## 5. 정리

- **일반 락** : 화장실 열쇠 1개. 누구든 1명만 들어가고, 나올 때까지 대기.
    
- **Read-Write 락** :
    
    - 일반인은 **읽기** → 여러 명 동시에 화장실 구경 가능
        
    - VIP는 **쓰기** → VIP가 들어가면 문을 걸어 잠그고 혼자만 사용
        
    - VIP가 없을 때는 마치 락이 없는 것처럼 자유롭게 입장

---

# 6. 헷갈릴 수 있는 flag & READ_MASK 정리

## 👾자 생각해보자 flag & READ_MASK를 한다치자
1. 만약 누군가 WRITE를 해서 앞쪽 0x7FFF0000부분이 변경된 상태라고 가정
2. 그럼 현재 flag는 0x32CF0000이 되어있다고 가정
3. 현재 연산하고있는 부분은 READ_MASK의 부분 즉 0x0000FFFF이부분을 연산중이다
4. WRITE가있는상태  expected = 0x32CF0000 & 0000FFFF 을하면 값이 어떻게나올까?
5. expected = 0x00000000이 나온다 (여기까지는 의도된게 맞다 RAED_MASK만 체크할거이기에)
6. !!여기서 중요!!
7. CompareExchange 여기서는 expected(0x00000000이) == flag(0x32CF0000)가 같은지 물어보는것
8. flag가 0x0000000이냐? 아니다! write가 락을 잡고있어서 있어서 0x32CF0000다!
9. 그래서 실패를 하게된다!
	!! 결론: WRITE하는동안 RAED를 못하게 하려는것

## 👾다음단계 READ경합 조건
1. RAED하려는 두 스레드의 경합 상태를 원활하게 정리하고 카운팅 방식
2. int expected = (flag & READ_MASK) 내가 원하는값을 계산한다
3. 다른스레드로 컨텍스트 스위칭을 한다
4. 도중에 다른스레드도 (flag & READ_MASK)연산을 하게된다
5. 그럼 첫번째 스레드의 expected값과 두번째 스레드의 expected 값이 같다
6. 그럼 두번째 스레드가 먼저 CompareExchange 한다치면 두번째 스레드는 통과한다
7. 다음에 첫번째 스레드로 컨텍스트 스위칭을한다.
8. 근대 내가 예상했던 flag가 expected 값과 다르다 그러면 한번 더 순회를 하게된다
9. 다시 돌아와서 (flag & READ_MASK); 하게되고 컨텍스트 스위칭을하게되도 문제가없다
10. 왜냐? 지금 기다리고있는 스레드가 없기때문이다.
11. 그래서 첫번째 스레드도 성공하게되면서 락을 얻게된다

