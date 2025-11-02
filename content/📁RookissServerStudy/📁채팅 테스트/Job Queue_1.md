
## 서론

이번 강의에서는JobQueue를
c#의 람다식을 사용해서 간편하게 만드는 방법
이전 시간에 배웠던 커맨드 디자인 패턴을 사용해서 만드는 방법
두가지를 만들어 보겠다

	그래서 JobQueue가 뭔대?
	쉽게 말해서 내가 행동하는 항목들을 Queue로 들고있는걸 JobQueue라 칭함


	왜 만듬?
	멀티 스레드 방식이다보니 공유자원에 접근할 일이 있다고 가정하면
	각 스레드가 어떠한 일을 수행할 때마다 Lock을 걸고 내가 해야할일을 진행했다
	수행해야할 일이 끝나기 전 까지는 다른 스레드는 그 공간을 들어 갈 수가없는데
	그러면 다른 스레드들은 공간에 들어가기 전까지 기다려야한다
	즉, 멀티스레딩을 하고있지만 비효율적으로 하고있는거다 이럴바엔 
	싱글 스레드를 사용하는것과 다를바 없다

	전제 조건 : 어느 방에 유저가 100명있음 일단 수행작업은 BroadCast를 한다는 조건
	수행 : 게임 룸 입장 ->
	클라 : 데이터 Send() x 100 ->
	서버 : 100명의 유저에게BroadCast() x 100
	스레드가 BroadCast를 다 마칠 때 까지 다른 스레드는 기다려야함
	만약 BroadCast가 길어진다? 그러면 싱글스레드와 다를바 없음

	우리는 그래서 JobQueue라는 시스템을 만들어서 
	BroadCast같이 수행해야하는 작업을 하나의 스레드에게만 일을 시켜서 진행해보자
	즉, JobQueue가 아무일 안할 때 처음 접근하는 스레드가 해당 일만 수행하는 역할을
	주면 어떨까 라는 생각이다.

	 이외 다른 스레드는 Queue안에 수행해야할 작업들을 넣기만하고
	 지금 처리해야하는 스레드는 Queue안에있는 작업만 처리하게끔 하면 보다 깔끔하게
	 멀티스레딩이 가능할 것 같다

---

## JobQueue 클래스 소개

### 인터페이스 
- Push() => 각 스레드가 수행해야 할 작업을 Enqueue를 하게끔 유도

```csharp
public interface IJobQueue
{
	void Push(Action job);
}
```

### 클래스 메서드 소개
- Push
	- 해야할 작업 Enqueue유도
	- 처음 들어온 스레드가 해당 작업을 수행 하게 끔 처리
```csharp
 public void Push(Action job)
 {
	 bool flush = false;

	 lock (_lock)
	 {
		 _jobQeueue.Enqueue(job);

		 if (_flush == false)
			 flush = _flush = true;
	 }
	 
	 if (flush)
		 Flush();
 }
```
- Flush
	- Queue에 일감이있으면 처리해주는 용도
```csharp
private void Flush()
{
	 while (true)
	 {
		 Action action = Pop();
		 if (action == null)
			 return;

		 action?.Invoke();
	 }
}
```
- Pop
	- 먼저 온 순서대로 저장되어있는 일감을 빼내는 용도
	```csharp
private Action Pop()
{
	lock (_lock)
	{
		if (_jobQeueue.Count == 0)
		{
			_flush = false;
			return null;
		}
		else
			return _jobQeueue.Dequeue();
	}
}
```

---

### 전체 코드

```csharp
public class JobQueue : IJobQueue
{
 Queue<Action> _jobQeueue = new();

 object _lock = new object();

 private bool _flush = false;

 public void Push(Action job)
 {
	 bool flush = false;

	 lock (_lock)
	 {
		 _jobQeueue.Enqueue(job);

		 if (_flush == false)
			 flush = _flush = true;
	 }
	 
	 if (flush)
		 Flush();
 }

 private void Flush()
 {
	 while (true)
	 {
		 Action action = Pop();
		 if (action == null)
			 return;

		 action?.Invoke();
	 }
 }

 private Action Pop()
 {
	 lock (_lock)
	 {
		 if (_jobQeueue.Count == 0)
		 {
			 _flush = false;
			 return null;
		 }
		 else
			 return _jobQeueue.Dequeue();
	 }
 }
}
```

---
### 사용 예시

```csharp
class GameRoom : IJobQueue
{
	private JobQueue _jobQueue = new JobQueue();

	public void Push(Action job)
	{
	    _jobQueue.Push(job);
	}
	
	
	public void BroadcastChat(ClientSession session, string chat)
	{
		S_Chat packet = new S_Chat();
		packet.playerId = session.SessionId;
		packet.chat = $"[{session.SessionNickName}] : {chat}";
		ArraySegment<byte> sagment = packet.Write();

		foreach (ClientSession s in _sessions)
			s.Send(sagment);
	}
}
```

---

### 수행해야 할 작업을 요청
```csharp
class PacketHandler
{
    public static void C_ChatHandler(PacketSeesion session, IPacket packet)
    {
	    GameRoom room = clientSession.Room;
	    
		room.Push(() =>
		{
			room.BroadcastChat(clientSession, chatPacket.chat);
		});
    }
}
```


---

## 이외 Lamda식이아닌 클래스로 만들어서 처리 예제

```csharp
interface ITask
{
    public void Execute();
}

class BroadcastTask : ITask
{
    GameRoom _room;
    ClientSession _session;
    string _chat;

    public BroadcastTask(GameRoom room, ClientSession session, string chat)
    {
        _room = room;
        _session = session;
        _chat = chat;
    }

    public void Execute()
    {
        _room.BroadcastChat(_session, _chat);
    }
}


class TaskQueue
{
    Queue<ITask> _queue = new();

    // 여기서 만들어 둔 JobQueue처럼 진행하면 될듯?
}
```




