
---
## 서론
 
우리가 이전 시간에는 메인스레드에 Flush작업을 해줬다  
근대 0.25초마다  Sleep을 해줘서 Flush처리를 해줬는데  
  
근대 이게 Sleep 하나만 처리하는게 아니라  
게임이 좀 더 복잡해져서 Room이 하나만 있는게 아니라  
여러개가 있고 Room뿐만 아니라 서버 측 에서 실행해야 할게 많아지면  
Sleep으로만 처리하기에는 무리가 있다  

그래서 우리는 시간관리를 어떻게 해야할지 고민을 해본다  

---
## 여러 구현 방식 중 Tick방식 설명
1. Tick처리 RoomTick이라는 int변수를 둬서 시간 계산 후 If문으로 처리
2. System.Environment.TickCount같이 현재시간 Tick추출 후 RoomTick보다 작다면 처리해주는 방식 
3. 한 while문에 전부 처리해주는 것도 하나의 방법

``` csharp
int roomTick = 0;

while (true)
{
	이전 방식
    Room.Push(() => { Room.Flush(); });
    Thread.Sleep(250);
    
    다른 방식
    int now = System.Environment.TickCount;

    if (roomTick < now)
    {
        Room.Push(() => { Room.Flush();} );

        roomTick = now + 250;
    }
}
```


	- 하지만 이 방식은 어떠한 로직은 다른 Tick에 돌아야하는 로직이 있으므로
	- 그때마다 Tick을 추가해줘야한다
  

---

## 다른 방식 (JobTimer 예약 시스템)
매 프레임마다 틱을 세는것 보다
예약 시스템을 만들어보는건 어떨까?

내부적으로 시간을 재면서 실행할 때가 되면 실행해주는 그런 시스템말이다.  

예 : C# Coroutine같은것  
  
이런 비슷한 시스템을 만들어보자
구현 방법은 여러가지지만 `Priority Queue`를 사용해서 만들어보자

다음으로 실행되어야하는 Tick을 `Priority Queue`에 넣어놔서
실행시간이 가장 임박한 작업을 실행하는 방식으로 진행할 것


### PriorityQueue에 사용될 구조체 선언 
- execTick 실행 되어야 하는 시간
- action 예약한 행동
```csharp
    struct JobTimerElem : IComparable<JobTimerElem>
    {
        public int execTick;
        public Action action;

        public int CompareTo(JobTimerElem other)
        {
            return other.execTick - execTick;
        }
    }
```

### 클래스 (JobTimer)
	메서드 소개
- `Push(Action action, int tickAfeter = 0)` 행동과, 실행해야하는 시간 간격
- `Flush()` 시간 체크 후 작업 실행

 `[변수]`
```csharp
우선순위 큐
PriorityQueue<JobTimerElem> _pq = new();

공유 자원 접근 제어 Lock
object _lock = new();
```

```csharp
public void Push(Action action, int tickAfeter = 0)
{
    JobTimerElem job;
    
    job.execTick = System.Environment.TickCount + tickAfeter;
    job.action = action;

    lock (_lock)
    {
        _pq.Push(job);
    }
}
```
---

```csharp
public void Flush()
{
    while(true)
    {
        int now = System.Environment.TickCount;

        JobTimerElem job;

        lock (_lock)
        {
            if (_pq.Count == 0)
                break;

            job = _pq.Peek();
            // 실행해야하는 Job이 먼 미래다
            // 즉, now보다 크다면 break;
            if (job.execTick > now)
                break;

            // 여기까지왔다면 Job을 실행시켜야한다는거니깐
            // 실행시켜주자
            _pq.Pop();
        }

        job.action?.Invoke();
    }
}
```


---
## 사용 예제

```csharp
static void FlushRoom()
{
   Room.Push(() => { Room.Flush(); });
   JobTimer.Instance.Push(FlushRoom, 250);
}


static void Main(string[] args)
{
	FlushRoom();
	
	while(true)
	{
		JobTimer.Instance.Flush();
	}
}
```

---

개인적으로 느낀 점

어떠한 작업을 Tick별로 하나씩 만들어서 관리하는 것보다
하나의 관리자를 만들어서 함수를 만들어 스케줄 하는 방식이 보다 유지보수에는 편한것같다

