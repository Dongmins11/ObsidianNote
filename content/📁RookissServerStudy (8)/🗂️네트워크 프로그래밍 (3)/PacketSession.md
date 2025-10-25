
## 서론

다음 시간부터 
Packet을 만들기 위해 사전 작업을 할것
  
  
패킷이 뭔대용?  
쉽게 말해서 컴퓨터 간 데이터를 주고 받을 때 네트워크를 통해서 전송되는 데이터 조각  이라고 생각하면 됨  
  
무슨 작업을 할껀대용?  
TCP와 같은 전송 프로토콜은 데이터를 한번에 보내주는게아니라  
일정부분 잘라서 보내는 경우가있는데  
만약 패킷이 완전체로 오지 않을경우가 있어서 완전체인지 아닌지 구분하는 작업을 해야한다  
  
---
### 패킷 클래스
  
해당 패킷을 구분하기위해 ID를 지정하고  
위에서 말했듯이 TCP는 일정부분 잘라서 송신해주는경우가 있다  
그래서 서버와 약속을해서 어느정도 SIZE를 보낼지 지정해줘야한다  
  
기본 형식  
```csharp
class Packet
{
	public ushort size;
	public ushort packetId;
}
```

## 잠깐!
  
왜 int가 아니라 ushort자료형이에여?  
데이터를 송수신 하는 과정에서 최대한 데이터를 압축하고  
효율적으로 사용하기위해서 ushort를 사용했다  
  
지금은 size와 packetid만 처리해놨지만 앞으로 들어오는 데이터도 최대한 생각해서  
압축을 해주면 좋다  
  
  
왜? 1명의 데이터 처리를 한다면 No상관  
근대 과장된 말로 서버에서 10만명의 유저가 접속했다면?  
  
어떻게든 최적화 처리를해서 내 주변 캐릭터의 데이터만 송수신 한다 쳐도  
어림잡아 1만 유저라고 가정한다면  
  
1명당 송신해야 하는 데이터를 int로 잡아보면 4byte * 10000 = 4만개의 바이트처리가된  다 근대 주변애들 1명 1명도 데이터를 송신해야한다면 그렇다면.......? 오우  

데이터를 ushort로 잡아보면 2byte * 10000 = 2만개의 바이트가 처리된다  
int보다 비용이 싸다 그러므로 short로 처리해둔것  
  
---
  
## 패킷 Session
  
Session을 상속받은 Packet전용 세션을 만든다  
이 친구는 데이터가 조각나서 온걸 합쳐주는 역할을 할것이다  

```csharp
     public static readonly int HeaderSize = 2;

     public sealed override int OnReceive(ArraySegment<byte> buffer)
     {

         int processLen = 0;
         while (true)
         {
             // 최소한 헤더는 파싱할 수 있는지 확인하는 부분
             if (buffer.Count < HeaderSize)
                 break;

             // 패킷이 완전체로 도착했는지 확인
             ushort dataSize = BitConverter.ToUInt16(buffer.Array, buffer.Offset);

             // 데이터가 부분적으로만 왓으니 break;
             if (buffer.Count < dataSize)
                 break;

             // 여기까지 왔으면 패킷 조립 가능
             // 왜?
             // 위에서 buffer.Count < dataSize 데이터 다왔는지 확인하니까

             OnReceivePacket(new ArraySegment<byte>(buffer.Array, buffer.Offset, dataSize));
             
             // 다음부분을 찝어주는 것
             processLen += dataSize;
             buffer = new ArraySegment<byte>(buffer.Array, buffer.Offset + dataSize, buffer.Count - dataSize);
         }

         // 처리한 byte수를 리턴시킴
         return processLen;
     }

     public abstract void OnReceivePacket(ArraySegment<byte> buffer);

```
    
    
## 로직 정리  
  
### 헤더 읽기 
```csharp
public static readonly int HeaderSize = 2;

if (buffer.Count < HeaderSize)
                 break;

```
부분에서 HeaderSize를 넘기지 못하면 break처리가되는데  
현재 short가 2byte라서 그렇다  
buffer에 2byte를 넘기지 못했다는것은 헤더자체도 못받아왔단 뜻  
  
---
   
### 패킷의 완전체 확인 및 조립

```csharp
ushort dataSize = BitConverter.ToUInt16(buffer.Array, buffer.Offset);

 // 데이터가 부분적으로만 왓으니 break;
if (buffer.Count < dataSize)
    break;
    
OnReceivePacket(new ArraySegment<byte>(buffer.Array, buffer.Offset, dataSize));
```

패킷의 헤더가 왔으니 Data의 크기를 얻어올 수 있다  
이걸 이용해서 패킷이 전부 왔는지 안왔는지 확인이 가능하다  
  
---
### 새롭게 PacketSession을 상속받으면 Onrecv안에 처리해줘야할 함수를 호출해줌
  
```csharp
Session의 Recv를 상속 받은 후 밑에서 상속을 못받게 처리
public sealed override int OnReceive(ArraySegment<byte> buffer)

OnReceivePacket 강제로 상속받아서 처리해줘야함
public abstract void OnReceivePacket(ArraySegment<byte> buffer)
```

왜 이렇게 처리??  
OnReceive를 PacketSession만의 패킷 조립방식 강제하고   
Packsession상속받는 친구는 OnReceivePacket를 이용해서 조립된 데이터를 가공을 하게끔 유도 처리   
  
### 상속받은 예시
```csharp
class GameSession : PacketSeesion
{
	public override void OnReceivePacket(ArraySegment<byte> buffer)
	{
	    ushort size = BitConverter.ToUInt16(buffer.Array, buffer.Offset);
	    ushort packetId = BitConverter.ToUInt16(buffer.Array, buffer.Offset + sizeof(short));

	    Console.WriteLine($"Recv PacketId {packetId}, Size {size}");
	}

}
```
  
---
















