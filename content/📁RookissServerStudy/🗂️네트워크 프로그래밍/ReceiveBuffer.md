
오늘 우리는 ReceiveBuffer를 처리해줄 것

이전까지는 Receive에 SetBuffer를 할 때byte[1024]를 대충 감잡아서 때려넣어놨는데
```csharp
_recvArgs.SetBuffer(new byte[1024], 0, 1024);
```

지난시간에 TCP개론 시간에 잠깐 얘기해봤지만  
클라에서 100Byte를 보냇다고해서 무조껀 100Byte전부 넘어오는게 아니다  
TCP는 버퍼의 상황을 봐서 분할해서 보낼 수도 있다  
  
즉 이전 시간에  Register -> CallBack -> Completed가 됐는데  
Completed 단계에서 넘어온 값을 한번에 처리 해주고 있엇는데  
이제 TCP의 특성으로 인해서 데이터가 전부 왔는지 확인해서   
데이터를 처리해주는 식으로 변경해야한다  
  
## 변경전
```csharp
private void OnRecvCompleted(object sender, SocketAsyncEventArgs args)
{
    if (args.BytesTransferred > 0 && args.SocketError == SocketError.Success)
    {
        try
        {
            OnReceive(new ArraySegment<byte>(args.Buffer, args.Offset, args.BytesTransferred));
            
            RegisterRecv();
        }
        catch (Exception ex)
        {
            Console.WriteLine($"OnRecvCompleted Failed {ex.Message}");
        }
    }
    else
    {
        Disconnect();
    }
}
```

---

## RecvBuffer 역할
  
네트워크 프로그래밍에서 소켓으로부터 받은 데이터를 임시로 저장하고 관리하는 버퍼  가 필요함 우리는 이전에 임시로 해둔거임  
전부 Session에서 처리를 해도 되지만 강사님은 따로 모듈화해서 처리해뒀다고 함
  
이 클래스는 OS에서 들어온 Recv데이터 와  
게임/서버 로직에서 처리하는 데이터(Read)의 경계를  
ReadPos와 WritePos로 나눠서 관리한다  
  
즉,   
ReadPos : 콘텐츠가 어디까지 읽었는지  
WritePos : OS가 어디까지 버퍼를 썼는지  
나타내는것이다  
  
### 사용되는 변수
```csharp
private ArraySegment<byte> _buffer;
private int _readPos;
private int _writePos;
```

_buffer는 왜 byte배열이 아니에요?_  
  
byte[] 배열을 사용해도 되지만  
나중에 엄청 큰 바이트 배열을 부분적으로 잘라서 쓰고싶을때도있으니  
ArraySegment 변수 사용하는걸로함  


---
## 동작 원리 (read / write 포인터 흐름)

배열 표시는 [ ] 로 진행하겠음
### 기본 개념

`[rw][ ][ ][ ][ ][ ][ ][ ][ ][ ]`
- 처음엔 r=w=0, 즉 아무 데이터도 안들어있는거

### 클라가 5바이트를 보냄

`[r][ ][ ][ ][ ][w][ ][ ][ ][ ]`
- recv 성공 -> _writePos 가 5 증가.
- 아직 콘텐츠단에서 처리 안 했으니 _readPos 는 그대로.

### 콘텐츠에서 패킷을 처리 완료

`[ ][ ][ ][ ][rw][ ][ ][ ][ ][ ]`
- 콘텐츠단이 5바이트 소비 완료 -> _readPos = _writePos = 5.

---

## 부분적 데이터가 넘어오거나 데이터를 배열을 초과하는 문제를 해결해보자

### 문제 상황 예시

- 2바이트짜리 패킷을 클라이언트가 두 개 연속 보냄 (총 4바이트)
- 근데 첫 recv에선 3바이트만 옴

`[r][ ][ ][w][ ][ ][ ][ ][ ][ ]   (유효 데이터 3바이트)`

- 콘텐츠는 “2바이트짜리 패킷” 하나만 처리 가능함 -> 첫 2바이트 소비
- 남은 1바이트는 다음 패킷이 도착하기 전까진 처리 불가  
  readPos = 2, _writePos = 3  대기

이 상태가 누적되면:
- r이 계속 증가하고
- w도 뒤로 밀리면서 뒤쪽 여유 공간이 점점 줄어듦


### 해결 방안 Clean 처리

이때, 남은 데이터(write - read) 를 앞으로 복사해서  
read / write를 처음으로 돌려주는 것이 Clean() 함수의 역할이다.

```csharp
public void Clean() 
{
    int dataSize = DataSize;
	if (dataSize == 0)     
	{
	    _readPos = 0;
	    _writePos = 0;
	}
	else
	{
	   Array.Copy(_buffer.Array, _buffer.Offset + _readPos,                    _buffer.Array, _buffer.Offset, dataSize);
	   
		_readPos = 0;
		_writePos = dataSize; 
	 } 
}
```

---

## 데이터를 읽고 쓸때 포인터를 이동하는데 그거에 대한 기능 모음

| 함수             | 역할                   |
| -------------- | -------------------- |
| `DataSize`     | 처리되지 않은 데이터의 길이      |
| `FreeSize`     | 버퍼의 남은 공간            |
| `OnRead()`     | 처리 완료 후 read 포인터 이동  |
| `OnWrite()`    | 수신 완료 후 write 포인터 이동 |
| `ReadSegment`  | 콘텐츠단에서 읽을 구간         |
| `WriteSegment` | 다음 recv에서 쓸 구간       |

```csharp

        //유효범위 얼마나 데이터가 쌓여있는지 (처리가 안된 데이터)
        public int DataSize { get { return _writePos - _readPos; } }

        //버퍼의 남은 공간
        public int FreeSize { get { return _buffer.Count - _writePos; } }


        //유효 범위의 세그먼트라해서
        //어디부터 데이터를 콘텐츠 단에넘겨준건지에 대한 Segment
        public ArraySegment<byte> ReadSegment
        {
            //데이터 배열
            //어디부터 데이터를 넘겨줄지
            //데이터의 크기는 얼만큼 보내줄지
            get { return new ArraySegment<byte>(_buffer.Array, _buffer.Offset + _readPos, DataSize); }
        }

        // 다음에 Recv할 때 어디서부터 유효범위인지 나타내는것
        // 예 : [ ][ ][ ][ ][r][ ][w][ ][ ][ ] 여기서 [w][ ][ ][ ] 이부분이 사용할 수 있는 범위
        public ArraySegment<byte> WriteSegment
        {
            get { return new ArraySegment<byte>(_buffer.Array, _buffer.Offset + _writePos, FreeSize); }
        }

        //우리가 콘텐츠 코드에서 데이터를 가공해서 처리할껀대
        //성공적으로 처리를 했으면 OnRead를 호출해서 커서위치를 바꿔줄것
        public bool OnRead(int numOfBytes)
        {
            // numOfBytes 왜 받음?
            // 범위를 넘어설 수 있기에 호출한 애가 numOfBytes 만큼 처리했다는걸 보내줌
            // 근대 데이터 유효범위 이상을 처리하면 이상하게된거니까 false처리
            if (numOfBytes > DataSize)
                return false;

            _readPos += numOfBytes;

            return true;
        }

        // 클라에서 서버쪽으로 데이터를 쏴줘서 서버에서 Recv를 했을 때 _writePos를 이동시키기 위한 함수
        public bool OnWrite(int numOfBytes)
        {
            if(numOfBytes > FreeSize)
                return false;

            _writePos += numOfBytes;

            return true;
        }
```

---

## 사용 흐름 


1. Seesion에 recvBuff생성 
2. RigesterRecv 부분에서 Clean처리 후 OS에서 Recv데이터를 쌓아줄 버퍼를 셋 (Write버퍼)
3. RecvCompleted 부분에서 Write커서이동 
4. 컨텐츠에서 처리할 수 있도록 데이터 넘김
5. 컨텐츠 단에서 처리한 데이터 만큼 Read커서이동


```csharp
1. Seesion에 recvBuff생성 
RecvBuffer _recvBuffer = new RecvBuffer(1024);



2.RigesterRecv 부분에서 Clean처리 후 OS에서 Recv데이터를 쌓아줄 버퍼를 셋 (Write버퍼)

private void RegisterRecv()
{
    _recvBuffer.Clean();
    ArraySegment<byte> segment = _recvBuffer.WriteSegment;
    _recvArgs.SetBuffer(segment.Array, segment.Offset, segment.Count);

    bool pending = _socket.ReceiveAsync(_recvArgs);
    if (!pending)
        OnRecvCompleted(null, _recvArgs);
}




private void OnRecvCompleted(object sender, SocketAsyncEventArgs args)
{           
3. RecvCompleted 부분에서 Write커서이동 

 if(_recvBuffer.OnWrite(args.BytesTransferred) == false)
 {
     Disconnect();
     return;
 }

4.컨텐츠에서 처리할 수 있도록 데이터 넘김

 int processLength = OnReceive(_recvBuffer.ReadSegment);
if (processLength < 0 || _recvBuffer.DataSize < processLength)
{
    Disconnect();
    return;
}

5.컨텐츠 단에서 처리한 데이터 만큼 Read커서이동

if(_recvBuffer.OnRead(processLength) == false)
{
    Disconnect();
    return;
} 
}
```

---

## 요약

- RecvBuffer는 네트워크에서 조각나서 들어오는 데이터를 **안전하게 누적, 처리, 정리**하는 버퍼 클래스다.  

- 읽기/쓰기 포인터를 분리함으로써 데이터 손실 없이 패킷 단위 처리를 구현할 수 있다.

