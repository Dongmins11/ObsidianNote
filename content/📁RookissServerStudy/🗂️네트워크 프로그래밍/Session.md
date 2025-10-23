
# 서론

비유 : 휴대폰을 래핑해서 간단하게 사용할 수 있게만든 클래스

Session에는  
Receive Send DisConnect 기능을 모듈형으로 만들어놓은 클래스다  
연결, 연결해제 및 데이터를 주고 받고 해주는 클래스를 만들것이다  
  
먼저 Receive -> Send -> DisConnect순으로 설명하겠다  
  
우리는 이전시간에 Async계열로 Listener를 만들었다  
이번에 Session도 Async계열로 만들것.  

Connect는 따로 만듬  

---
  
# ✨미리 흝어보면 좋은 클래스 및 구조체

```csharp
1. EndPoint
2. SocketAsyncEventArgs
3. ArraySegment
```


## 1️⃣ EndPoint
네트워크 통신의 주소를 표현하는 추상 클래스다  
즉 데이터를 보내거나 받을 때 어디로 보낼지 어디서 받을지 지정하는 개념이라 보면 됨  

c#에서는 EndPoint 상속 받은 IPEndPoint를 사용  
IPEndPoint는 Ip주소 + 포트 번호로 구성됨  
```csharp
IPEndPoint endPoint = new IPEndPoint(IPAddress.Loopback, 7777);
```

### RemoteEndPoint
원격 측의 주소 정보를 나타내는 프로퍼티  
즉 현재 소켓이 연결되어 있는 상대방(클리어언트, 서버)의 IP+ Port정보를 담고있음  

### 서버 측 사용
```csharp
Socket clientSocket = listener.Accept();
Console.WriteLine($"클라이언트 접속: {clientSocket.RemoteEndPoint}");
```

### 클라 측 사용
```csharp
Socket socket = new Socket(AddressFamily.InterNetwork, SocketType.Stream, ProtocolType.Tcp);
socket.Connect("127.0.0.1", 7777);
Console.WriteLine($"서버에 연결됨: {socket.RemoteEndPoint}");
```

### Aysnc 측 사용 (IOCP 환경)
```csharp
void OnAcceptCompleted(object sender, SocketAsyncEventArgs e)
{
    Socket client = e.AcceptSocket;
    Console.WriteLine($"[접속] {client.RemoteEndPoint}");
}
```

---

## 2️⃣ SocketAsyncEventArgs
비동기 소켓 통신을 위해 설계된 이벤트 인자 클래스  
해당 기능에는 AcceptAsync나 ReceiveAsync, SendAsync등 비동기 이벤트 기반으로 처리가 가능  

### 주요 프로퍼티 설명
```csharp
1. SocketAsyncEventArgs.Buffer : 송수신할 데이터 버퍼
2. SocketAsyncEventArgs.BytesTransferred : 실제 전송된 바이트 수
3. SocketAsyncEventArgs.SocketError : 작업 성공 여부
4. SocketAsyncEventArgs.AcceptSocket : 새로 연결된 소켓 Accept 시
5. SocketAsyncEventArgs.Completed : 비동기 작업 완료 시 발생하는 이벤트
6. SocketAsyncEventArgs.ConnectSocket : 연결된 소켓 Connect 시
```

### 사용 예시

```csharp

SocketAsyncEventArgs _recvArgs = new SocketAsyncEventArgs();
_recvArgs.Completed += new EventHandler<SocketAsyncEventArgs>(OnRecvCompleted)
_recvArgs.SetBuffer(new byte[1024], 0, 1024)

-----------------------------------------------------------------------

private void RegisterRecv()
{
    bool pending = _socket.ReceiveAsync(_recvArgs);
    if (!pending)
        OnRecvCompleted(null, _recvArgs);
}


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

## 3️⃣ ArraySegment
배열의 일부분을 잘라 참조하기 위한 구조체  
즉 배열의 일부 영역을 복사 없이 가리키는 슬라이스 개념  
```csharp
byte[] buffer = new byte[1024];
ArraySegment<byte> segment = new ArraySegment<byte>(buffer, 0, 512);
```
이렇게 하면 `buffer` 배열의 **0~511번 인덱스 영역만 가리키는 구조체**가 만들어짐.  


### 장점
메모리 복사 없이 배열 조각을 전달 가능
버퍼를 효율적으로 전달하기 위해서
Send를 하면 최적화가 필요하다 그래서 ArraySegment이용해서 모아서보내기
최적화를 사용할 수 있다

### 이전 버전

```csharp
byte[] sendBuff = _sendQueue.Dequeue();
_sendArgs.SetBuffer(sendBuff, 0, sendBuff.Length);
```

### 최적화 버전
```csharp
 while (_sendQueue.Count > 0)
 {
     byte[] sendBuff = _sendQueue.Dequeue();
     _pendingList.Add(new ArraySegment<byte>(sendBuff, 0, sendBuff.Length));
 }

 _sendArgs.BufferList = _pendingList;
```

---


## Receive

### Receive를 호출하기 전 이벤트 설정
- Async계열을 사용할 거 이기에 콜백이 필요함
```csharp
_recvArgs.Completed += new EventHandler<SocketAsyncEventArgs>(OnRecvCompleted);

//버퍼 셋팅
_recvArgs.SetBuffer(new byte[1024], 0, 1024);
```

### Receive등록 시켜놓는 함수
- 처음 Start나 Init부분에 한번만 호출해서 한개의 루프만 돌리게끔 처리 해야함
```csharp
private void RegisterRecv()
{
    bool pending = _socket.ReceiveAsync(_recvArgs);
    if (!pending)
        OnRecvCompleted(null, _recvArgs);
}
```


### Receive 데이터가 왔을 때 호출되는 함수
- 받는 데이터가 있고 해당 데이터를 문제없이 받았으면  
  다시 RegisterRecv를 돌려서 루프를 만들어줌  
```csharp
 private void OnRecvCompleted(object sender, SocketAsyncEventArgs args)
 {
     //BytesTransferred 내가 데이터를 얼마나 받앗는지
     //0이 오는경우도있다

     if (args.BytesTransferred > 0 && args.SocketError == SocketError.Success)
     {
         //Offset 어디서부터 시작하는지
         //BytesTransferred 몇 바이트를 받았는지
         try
         {
             OnReceive(new ArraySegment<byte>(args.Buffer, args.Offset, args.BytesTransferred));

             //string recvData = Encoding.UTF8.GetString(args.Buffer, args.Offset, args.BytesTransferred);
             //Console.WriteLine($"[From Client] {recvData}");

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

## Send

Send는 Receive랑 비슷하지만 다르게 짜야한다  
왜냐? Receive는 커널에서 호출해주는 이벤트에 반응하는  
이벤트 성이라 한개의 루프만 돌리면되지만  

Send는 내가 직접 원하는 시점에 호출해야한다
즉, 여러 번 Async가 호출될 수 도 있는데 성능에 문제가 생길 수 있으므로
Queue나 ArraySagment를 사용해서 한번에 모아서 처리해주는 방식으로
해주면 일단은! 최적화에 도움이 된다

또한 데이터가 바뀔수 있는 RaceCondition상황이 있기에
Lock을 사용해서 처리해준다

###  Send를 호출하기 전 이벤트 설정
- Async계열을 사용할 거 이기에 콜백이 필요함
```csharp
_sendArgs.Completed += new EventHandler<SocketAsyncEventArgs>(OnSendCompleted);
```

### Send함수 (Lock 처리)
```csharp
 public void Send(byte[] sendBuff)
 {
     lock (_lock)
     {
         _sendQueue.Enqueue(sendBuff);

         //1) _pendingList 대기중인 데이터가 없다고 판별하니 RegisterSend
         //2) 만약 예약중인 데이터가 있으면_sendQueue만 넣고 나옴
         if (_pendingList.Count == 0)
             RegisterSend();
     }
 }
```

### Send등록 시켜놓는 함수
- Queue와 ArraySegment를 사용해서 한번에 데이터를 모아서 보내는 방식으로 처리  
```csharp
private void RegisterSend()
{
    while (_sendQueue.Count > 0)
    {
        byte[] sendBuff = _sendQueue.Dequeue();
        _pendingList.Add(new ArraySegment<byte>(sendBuff, 0, sendBuff.Length));
    }

    _sendArgs.BufferList = _pendingList;

    bool pending = _socket.SendAsync(_sendArgs);
    if (pending == false)
        OnSendCompleted(null, _sendArgs);
}
```


### Send 데이터가 왔을 때 호출되는 함수
```csharp
 private void OnSendCompleted(object sender, SocketAsyncEventArgs args)
 {
     lock (_lock)
     {
         if (args.BytesTransferred > 0 && args.SocketError == SocketError.Success)
         {
             try
             {
                 _sendArgs.BufferList = null;
                 _pendingList.Clear();
                 
                 OnSend(_sendArgs.BytesTransferred);
                 
                 if (_sendQueue.Count > 0)
                     RegisterSend();
             }
             catch (Exception ex)
             {
                 Console.WriteLine($"OnSendCompleted Failed {ex.Message}");
             }
         }
         else
         {
             Disconnect();
         }
     }
 }
```

---
## DisConnected

작업을 종료 할때 호출함
연속으로 호출되면 문제가 발생하기에
Interlocked계열을 사용해서
내가 만든 변수가 변경이 되었다면 DisConnected가 된거이기에
연속으로 호출을 막아줌

```csharp
public void Disconnect()
{
    int desired = 1;
    if (desired == Interlocked.Exchange(ref _disconnected, desired))
        return;

    OnDisConnected(_socket.RemoteEndPoint);
    _socket.Shutdown(SocketShutdown.Both);
    _socket.Close();
}

```


---

