
## 서론

현재 우리는 서버와 유니티를 연결을 했지만  
Unity와 관련된 작업을 해본적이없다  
즉, Unity관련 함수 호출 및 컴포넌트 조작 등등 말하는 것  
  
이번 시간에는 위에 말한것을 해볼것이다.  

---
### 먼저 알아야 하는 것 
  
Unity는 정책을 만들었다 Unity는 ThreadSafety하게 작업을 강제했고 그로인해
Unity가 만들어 둔 MainThread외 다른 Thread에서  
Unity관련 함수나 컴포넌트 조작등등을 할 시  
Crash를 낸다던가 동작이 안되게 끔 처리 해놨다  
  
---
### 구현 순서

그래서 MainThread에서 돌아가는 Update문이나 Coroutine 등  
MainThread의 루틴의 어디선가 호출 해줘야한다는거다  
  
그래서 MainThread에서 호출해주기 위한  
`Packet Queue`를 만들어 주자  
해당 Queue를 이용해서 NetWork에서 온 Packet을 Enqueue후
MainThread어딘가에서 Dequeue해서 사용하는 구조를 만들 것
  
그러기 위해서는 현재 우리는 PacketManager쪽에서 
Recv데이터가 들어오면 바로 Handler안의 함수를 호출하고있는데 이부분을 수정해야 함 
  
왜여??  
여기서 호출을 하면 `Thread Pool` 에서 나온 다른 Thread가 처리를하기에 수정  
  
우리는 MonoBehavior를 상속받는 `NetWorkManager`를 미리 만들어놨고  
해당 클래스 Update문을 이용해서 DeQueue를 할 것

요약
1. Main Thread랑 BackGround(네트워크) Thread랑 소통을 PacketQueue통로를 이용 
2. 메인스레드는 Pop해서 사용하게끔 유도
  
---
  
## PacketQueue
- 간단히 싱글톤 인스턴스로 구현
- Push Pop은 Lock처리 왜?? Push할때도 멀티스레딩 Pop할때도 Push하고있는 스레드가 있을 수 있기에

```csharp
public class PacketQueue  
{  
    public static PacketQueue Instance { get; } = new PacketQueue();  
  
    private Queue<IPacket> _pakcetQueue = new();  
    private object _lock = new();  
  
    public void Push(IPacket packet)  
    {  
        lock (_lock)  
        {  
            _pakcetQueue.Enqueue(packet);  
        }  
    }  
  
    public IPacket Pop()  
    {  
        lock (_lock)  
        {  
            if (_pakcetQueue.Count == 0)  
                return null;  
              
            return _pakcetQueue.Dequeue();  
        }  
    }  
}
```
  
  
  
---
  
## 유니티 관련 함수 호출 예시
```csharp
class PacketHandler  
{  
    public static void S_ChatHandler(PacketSeesion session, IPacket packet)  
    {  
        S_Chat p = packet as S_Chat;
        ServerSession serverSession = session as ServerSession;
        
		GameObject go = GameObject.Find("Player");
		if(go == null)
			Debug.Log("Player not found");
		else
			Debug.Log("Player: found");
    }
}
```
  
  
---
  
## 패킷 매니저 (중요 내용만 작성)
- _makeFunc : Server에서보낸 Data를 Recv하면 Packet생성 후 받아온 데이터 삽입
- _handler : 패킷ID에 맞는 PacketHandler 함수를 호출
- Register : 액션 바인딩
- OnRecvPacket : Data가 Recv되면 외부에서 호출해줌 매개변수의 Action을 이용해서
  Packet Queue에 패킷 집어넣기
- MakePacket : 패킷 생성 함수
- HandlePacket : 패킷ID에 따른 PacketHandler 함수 호출해주는 부분

``` csharp
class PacketManager  
{
	Dictionary<ushort, Func<PacketSeesion, ArraySegment<byte>, IPacket>> _makeFunc = new();  

	Dictionary<ushort, Action<PacketSeesion, IPacket>> _handler = new();  
  
	public void Register()  
	{  
	_makeFunc.Add((ushort)PacketID.S_Chat, MakePacket<S_Chat>);  
    _handler.Add((ushort)PacketID.S_Chat, PacketHandler.S_ChatHandler);
	}
	
	
	public void OnRecvPacket(PacketSeesion session, ArraySegment<byte> buffer, Action<PacketSeesion, IPacket> onRecvCallback = null)  
	{
		Func<PacketSeesion, ArraySegment<byte>, IPacket> func = null;  
		if (_makeFunc.TryGetValue(packetId, out func))  
		{  
		    IPacket packet = func?.Invoke(session, buffer);  
		    if(onRecvCallback != null)  
		        onRecvCallback?.Invoke(session, packet);  
		    else  
		        HandlePacket(session, packet);  
		}
	}
	
	private T MakePacket<T>(PacketSeesion session, ArraySegment<byte> buffer) where T : IPacket, new()  
	{  
	    T pkt = new T();  
	    pkt.Read(buffer);  
  
	    return pkt;  
	}  
  
	public void HandlePacket(PacketSeesion session, IPacket packet)  
	{  
	    Action<PacketSeesion, IPacket> action = null;  
  
	    if (_handler.TryGetValue(packet.Protocol, out action))  
	        action?.Invoke(session, packet);  
	}
}
```
  
---

## 네트워크 매니저 (중요 부분만)
- MainThread에 돌아가는 Update를 사용
	PacketQueue에있는 Packet을 꺼내와서 PacketManager의 HandlePacket 호출

```csharp
public class NetworkManager : MonoBehaviour  
{  
    ServerSession _session = new ServerSession();  

    private void Update()  
    {  
        IPacket packet = PacketQueue.Instance.Pop();  
        if (packet != null)  
        {  
            PacketManager.Instance.HandlePacket(_session, packet);  
        }  
    }  
}
```