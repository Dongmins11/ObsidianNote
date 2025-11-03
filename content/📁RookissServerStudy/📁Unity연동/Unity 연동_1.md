
## 서론

이제 만든 Server를 Unity와 연동해서 사용해 볼것이다.

어느정도 재사용은가능하지만
모든게 Unity와 호환되는건아니다
Unity만의 정책이있기에 설령 C#문법이라도
자기들의 정책에 어긋나면 과감하게 배제한다

예 : TryWriteByte나 Span같은거는 사용이 불가능하다

Unity는 멀티스레드 환경으로 실행은 할 수 있긴한데
단, 제약사항이 있다 Unity메인스레드가 아니라 BackGround에서 실행이 된다
유니티 게임에서 관련된 Component나 로직관련된게 접근하려하면 Crash를 낸다

그래서 접근을 못하게끔 처리하자

---
## 파일 처리

일단 디버깅을 위해 유니티에다가 코드를 복사해서 넣어보자  
  
일단 클라이언트 만 생각해보자  

정리 품목 :
JobQueue : 유니티에서 코루틴 WaitForsecond같은걸 쓰면된다  
Listener : 서버쪽에서만 쓰던거니 삭제  
PriorityQueue : JobTimer에서 쓰던거니 삭제  
  
사용 목록 : RecvBff, SendBff, Session, ServerSession, (Client)Packet폴더

---

유니티로 가져와보니 이전에 만들었던 GenectePacket에 문제가 생겼다  
  
현재 버전에는 Span계열의 구조체를 사용할 수 있지만  
강의가 2019년도에 나왔다 이 당시에는 Unity가 Span지원을  
안해줬기에 사용이 불가능했다  
그래서 공부할 겸 이걸 수정해보자  

---
### Span을 사용 불가능 할 시  
1. Unsafe코드를 이용해서 직접 포인터를 조작하는 식으로 해도된다
2. 비트연산을 해서 한바이트씩 설정하는 방법이 있다

---
##  Span 수정 부분 
- 주석에 이전버전 나와있음
- 이러한 부분들을 자동화 해놨기에 패턴을 찾았으면 Genecte Format을 수정
### Read메서드
 
```csharp
public void Read(ArraySegment<byte> sagment)  
{  
  ushort count = 0;  

  ReadOnlySpan<byte> s = new ReadOnlySpan<byte>(sagment.Array, sagment.Offset, sagment.Count);  

  count += sizeof(ushort);  
  count += sizeof(ushort);  
	
  // todo 수정버전  
  //ushort chatLen = BitConverter.ToUInt16(s.Slice(count, s.Length - count));  
  ushort chatLen = BitConverter.ToUInt16(sagment.Array, sagment.Offset + count);  
	
count += sizeof(ushort);  
//this.chat = Encoding.Unicode.GetString(s.Slice(count, chatLen));  
this.chat = Encoding.Unicode.GetString(sagment.Array, sagment.Offset + count, chatLen);  
count += chatLen;  
}
```

---

### Write메서드

```csharp
public ArraySegment<byte> Write()  
{  
  ArraySegment<byte> openSegment = SendBufferHelper.Open(4096);  
  ushort count = 0;  
	
  //bool success = true;  

  Span<byte> s = new Span<byte>(openSegment.Array, openSegment.Offset, openSegment.Count);  

  count += sizeof(ushort);  
	
  // Span을 사용 불가능 할 시  
  // 아니면 Unsafe코드를 이용해서 직접 포인터를 조작하는 식으로 해도되ㅣㄴ다  
  // 아니면 비트연산을 해서 한바이트씩 설정하는 방법이 있다  
  //success &= BitConverter.TryWriteBytes(s.Slice(count, s.Length - count), (ushort)PacketID.C_Chat);  
  Array.Copy(BitConverter.GetBytes((ushort)PacketID.C_Chat), 0, openSegment.Array, openSegment.Offset + count, sizeof(ushort));  
  count += sizeof(ushort);  

	
  // Chat  
  ushort chatLen = (ushort)Encoding.Unicode.GetBytes(this.chat, 0, this.chat.Length, openSegment.Array, openSegment.Offset + count + sizeof(ushort));  
//success &= BitConverter.TryWriteBytes(s.Slice(count, s.Length - count), chatLen);  
Array.Copy(BitConverter.GetBytes(chatLen), 0, openSegment.Array, openSegment.Offset + count, sizeof(ushort));  

count += sizeof(ushort);  
count += chatLen;  

// Count  
  //success &= BitConverter.TryWriteBytes(s, count);      Array.Copy(BitConverter.GetBytes(count), 0, openSegment.Array, openSegment.Offset, sizeof(ushort));  

  //if (false == success)  
	  //return null;          return SendBufferHelper.Close(count);  
}
```

---

## Unity 연동

Packet Genector 수정 완료 후  
우리는 이전에 Dummy Client에서 Connector를 이용해서 서버에 접속을 했는데  
그걸 유니티 NetworkManager Script를 만들어서 Start부분에 넣어보자   
이후에는 오브젝트를 하나 만들어서 Play버튼을 누르면 잘 접속하는걸 확인을 할 수 있다
  
```csharp
public class NetworkManager : MonoBehaviour  
{  
    ServerSession _session = new ServerSession();  
      
    void Start()  
    {  
        string host = Dns.GetHostName();  
        IPHostEntry ipHost = Dns.GetHostEntry(host);  
        IPAddress ipAddr = ipHost.AddressList[0];  
        IPEndPoint endPoint = new IPEndPoint(ipAddr, 7777);  
  
        Connector connector = new Connector();  
        connector.Connect(endPoint, () => { return _session; }, 1);  
    }
}
```

---





