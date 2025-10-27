
### Serialization이란 뭘까?
  
한글로는 `직렬화` 라는 뜻을 지닌다  
  
직렬화는 객체나 데이터 구조를 Byte Stream같은 전송 및 저장에 적합한 연속적인  
형태로 변환하는 과정  
  
### 어디다가써요?
데이터를 파일이나 네트워크를 통해 다른 시스템이나 나중에 다시 사용하기 위해
저장할 수 있다.  
  
반대로 바이트 스트림을 원래 객체로 복원하는 과정을 `역 직렬화(DeSerialization)`라 부른다  
  
우리는 이번 강의에서 진행할 것은  
여러 직렬화의 방식중 몇 가지만 다룰 것이고  
직렬화를 통해 네트워크에 Send 및 Recv를 진행 할 예정  
  
직렬화 관련 강의가  
총 1~4장의 파트로 나뉘어져있지만 계속 개선을 하는 강의 방향이므로  
중요부 내용만 설명할 예정

---

## 부모 클래스 소개

- 현재 클래스의 변수의 선언 이유를 소개할것  

```csharp
public abstract class  Packet
{
    public ushort size;
    public ushort packetId;

    public abstract ArraySegment<byte> Write();
    public abstract void Read(ArraySegment<byte> sagment);
}
```

우리는 Server와 패킷을 주고 받으면서 데이터를 보내고 확인을 할 것이다  
  
### Size
```cshap
public ushort size;
```

이전에 TCP에 알아봤을 때 특징 중 하나는  
`데이터를 조각내서` 보낼 경우도 있다고 했다  
받는 입장에서는 패킷이 완전체로 왔는지 조각나서 왔는지 확인을 할 방도가 없다  
  
그래서 Packet의 size를 넣어줌으로 패킷이 완전체로 왔는지 확인을 할 수 있게된것  
  
### PacketId
```csharp
public ushort packetId;
``` 

송 수신할때 하나의 약속을 통해 이 Id는 어떤것에 해당하는것이니  
이렇게 읽어들여라 라고 해주는 Id값  


### 메서드

```csharp
public abstract ArraySegment<byte> Write();
public abstract void Read(ArraySegment<byte> sagment); 
```

상속받은 패킷의 데이터를 어떻게 쓰고 읽을지를 강제성을 부여해주기 위해서 선언  

---
## 자식 클래스 설명 (Primitive Type만 소개)

Primitive Type (long)예시 데이터 송 수신  
가변적 데이터 송 수신은 설명X
```csharp
public long playerId;
```

---

### Write
1. 송신 측에서 Send보낼 데이터 버퍼 생성
2. count값 설정 우리가 보낸 byte갯수를 알고있어야지 수신측에서 알 수 있음  
   | 또한 우리가 만든 버퍼의 크기가 문제없는지 확인도 가능
3. success를 통해 버퍼에 데이터를 잘 쌓았는지 확인
4. BitConverter의 TryWriteByte를 사용하기 위해  
   Span으로 우리가 사용 할 데이터의 크기를 찝어줌  
5. TryWriteBtye를 사용해 데이터 삽입
6. 문제 없다면 Send
  
Span MSDN : https://learn.microsoft.com/ko-kr/dotnet/api/system.span-1?view=net-8.0    
TryByWrite MSDN : https://learn.microsoft.com/ko-kr/dotnet/api/system.bitconverter.trywritebytes?view=net-9.0  

```csharp
ArraySegment<byte> openSegment = SendBufferHelper.Open(4096);

        ushort count = 0;
        bool success = true;

        Span<byte> sagment = new Span<byte>(openSegment.Array, openSegment.Offset, openSegment.Count);

        count += sizeof(ushort);
        
        success &= BitConverter.TryWriteBytes(sagment.Slice(count, sagment.Length - count), this.packetId);
        count += sizeof(ushort);

        success &= BitConverter.TryWriteBytes(sagment.Slice(count, sagment.Length - count), this.playerId);
        count += sizeof(long);
        
        success &= BitConverter.TryWriteBytes(sagment, count);
        
        if (false == success)
		    return null;

		return SendBufferHelper.Close(count);
```

---

### Read
- Buff가 도착했다는것은 데이터가 전부 왔다는 뜻
- 이걸 가지고 하나하나 데이터를 까봐야함

1.  BitConverter를 이용한 데이터를 가져오기
2. 각 데이터 고유의 크기 만큼 함수를 결정 (예 :  ToUInt16, ToInt64 등)
3. count값으로 어디서부터 읽어야 할지 정해줌

```csharp
--- 외부 처리 ---
ushort count = 0;

ushort size = BitConverter.ToUInt16(buffer.Array, buffer.Offset + count);
count += 2;

ushort packetId = BitConverter.ToUInt16(buffer.Array, buffer.Offset + count);
count += 2;


--- 내부 처리 ---
public override void Read(ArraySegment<byte> sagment)
{
    ushort count = 0;

    ReadOnlySpan<byte> s = new ReadOnlySpan<byte>(sagment.Array, sagment.Offset, sagment.Count);

    count += sizeof(ushort);
    count += sizeof(ushort);

    this.playerId = BitConverter.ToInt64(s.Slice(count, s.Length - count));
    count += sizeof(long);
}

--- 나눔 처리이유 --
고유의 Read방식을 처리할 거 이기에 공용된 데이터는 밖에서 꺼내서 처리


-- 이외 개개인의 별도 처리 --

switch ((PacketID)packetId)
{
    case PacketID.PlayerInfoReq:
        {
            PlayerInfoReq req = new PlayerInfoReq();
            req.Read(buffer);
            Console.WriteLine($"PlayerInfoReq {req.playerId}");
        }
        break;
    case PacketID.PlayerInfoOk:
        break;
}

Console.WriteLine($"Recv PacketId {packetId}, Size {size}");
```


---
## 전체 코드
```csharp
class PlayerInfoReq : Packet
{
    public long playerId;
    public string name;
	
    public struct SkillInfo
    {
        public int id;
        public short level;
        public float duration;

        public bool Write(Span<byte> s, ref ushort count)
        {
            bool success = true;
            success &= BitConverter.TryWriteBytes(s.Slice(count, s.Length - count), this.id);
            count += sizeof(int);

            success &= BitConverter.TryWriteBytes(s.Slice(count, s.Length - count), this.level);
            count += sizeof(short);

            success &= BitConverter.TryWriteBytes(s.Slice(count, s.Length - count), this.duration);
            count += sizeof(float);

            return success;
        }

        public void Read(ReadOnlySpan<byte> s, ref ushort count)
        {
            this.id = BitConverter.ToInt32(s.Slice(count, s.Length - count));
            count += sizeof(int);
            this.level = BitConverter.ToInt16(s.Slice(count, s.Length - count));
            count += sizeof(short);
            this.duration = BitConverter.ToSingle(s.Slice(count, s.Length - count));
            count += sizeof(float);
        }
    }

	public List<SkillInfo> skills = new List<SkillInfo>();
    

    public PlayerInfoReq()
    {
        this.packetId = (ushort)PacketID.PlayerInfoReq;
    }

    public override void Read(ArraySegment<byte> sagment)
    {
        ushort count = 0;

        ReadOnlySpan<byte> s = new ReadOnlySpan<byte>(sagment.Array, sagment.Offset, sagment.Count);

        count += sizeof(ushort);
        count += sizeof(ushort);

        this.playerId = BitConverter.ToInt64(s.Slice(count, s.Length - count));
        count += sizeof(long);

        ushort nameLen = BitConverter.ToUInt16(s.Slice(count, s.Length - count));
        count += sizeof(ushort);

        this.name = Encoding.Unicode.GetString(s.Slice(count, nameLen));
        count += nameLen;

        ushort skillLen = BitConverter.ToUInt16(s.Slice(count, s.Length - count));
        count += sizeof(ushort);

        this.skills.Clear();

        for(ushort i =0; i < skillLen; ++i)
        {
            SkillInfo info = new SkillInfo();
            info.Read(s, ref count);
            this.skills.Add(info);
        }
    }

    public override ArraySegment<byte> Write()
    {
        ArraySegment<byte> openSegment = SendBufferHelper.Open(4096);

        ushort count = 0;
        bool success = true;

        Span<byte> sagment = new Span<byte>(openSegment.Array, openSegment.Offset, openSegment.Count);

        count += sizeof(ushort);
        
        success &= BitConverter.TryWriteBytes(sagment.Slice(count, sagment.Length - count), this.packetId);
        count += sizeof(ushort);

        success &= BitConverter.TryWriteBytes(sagment.Slice(count, sagment.Length - count), this.playerId);
        count += sizeof(long);


        ushort nameLen = (ushort)Encoding.Unicode.GetBytes(this.name, 0, this.name.Length, openSegment.Array, openSegment.Offset + count + sizeof(ushort));
        success &= BitConverter.TryWriteBytes(sagment.Slice(count, sagment.Length - count), nameLen);
        count += sizeof(ushort);
        count += nameLen;


        success &= BitConverter.TryWriteBytes(sagment.Slice(count, sagment.Length - count), (ushort)skills.Count);
        count += sizeof(ushort);

        foreach(SkillInfo skill in skills)
            success &= skill.Write(sagment, ref count);

        success &= BitConverter.TryWriteBytes(sagment, count);

        if (false == success)
            return null;

        return SendBufferHelper.Close(count);
    }
}

public enum PacketID
{
    PlayerInfoReq = 1,
    PlayerInfoOk = 2,
}
```




