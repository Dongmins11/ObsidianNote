
## Listener 서론

이번 시간에는 식당의 문지기를 따로 
책임분리를 시킬것이다

또한 블로킹함수를 논블로킹함수로 대체하고
콜백함수를 이용해서 지속적인 루프문을 돌릴것.

---

### Why?  
블로킹으로 하면 어떤 작업을 할 때마다 해당 작업이 완료 되기 전 까지  
다른 작업을 하지못한다  
  
### 나는 블로킹 함수를 꼭 써야해!
그렇다면 소켓 마다 1개의 스레드가 필요함  
만약 10만명의 클라가 접속한다면 10만개의 스레드가 필요하다는 소리  
터무니없는 소리다 생각을 잠깐만 해봐도 ContextSwitching이 미친듯이 발생한다는것...  
  
그래서 논 블로킹을 사용해서 흐름을 유지한 채 요청을 미리 해두고  
비동기 방식 콜백으로 데이터를 받아와서 처리를해주면 보다 적은비용으로   
많은 처리가 가능하다는것  

---

## 전체 로직

```csharp
 class Listener
 {
     Socket _listenSocket;
     Action<Socket> _onAcceptHandler;

     public void Init(IPEndPoint endPoint, Action<Socket> onAcceptHandler)
     {
         _listenSocket = new Socket(endPoint.AddressFamily, SocketType.Stream, ProtocolType.Tcp);
         _onAcceptHandler += onAcceptHandler;

         _listenSocket.Bind(endPoint);

         _listenSocket.Listen(10);

         SocketAsyncEventArgs args = new SocketAsyncEventArgs();
         args.Completed += new EventHandler<SocketAsyncEventArgs>(OnAcceptCompleted);
         //왜 초기화 하는시점에 바로 한번에 처리하냐?
         //이벤트를 한번에 같이 등록시켜줘서
         //args를 재사용 하기위해서 그럼 그뒤에는
         //OnAcceptCompeted 호출됨
         RegisterAccept(args);
     }


     //당장 완료하는게아니라 등록을 한 것
     private void RegisterAccept(SocketAsyncEventArgs args)
     {
         //이 코드는 이벤트 재사용을 위해 
         //클리어 처리해주는 것
         //클리어처리를 안해주면 기존 AcceptSocket이 남아있게된다
         args.AcceptSocket = null;

         bool pending = _listenSocket.AcceptAsync(args);
         //펜딩 여부 (대기가있는지없는지 여부)
         //소켓이 진짜 아예 할게없어서 바로 리턴을 때려주면
         //처리해줄려고 펜딩여부를 둔것
         if (pending == false)
             OnAcceptCompleted(null, args);

         //근대왜 pending여부를 두냐? 바로 이벤트연결해서 
         //pending이 false가나오면 바로 콜백이 호출될텐대?
         // -> 답변 ms가 그렇게 만듬
     }

     private void OnAcceptCompleted(object sender, SocketAsyncEventArgs args)
     {
         //소켓에 간간히 에러가 나와서 처리해줌
         if (args.SocketError == SocketError.Success)
         {
             _onAcceptHandler.Invoke(args.AcceptSocket);
         }
         else
             Console.WriteLine(args.SocketError.ToString());


         //다음 턴을 위해 args를 등록해주는 것
         //전에 우리 코드는 while문으로 accecpt블로킹함수를썻는데
         //비동기는 계속 호출을 해주기위해서는 따로 
         //우리가 register를 해줘야한다
         //코드의 흐름을 따라가보면 
         //init -> 이벤트 등록 -> RegisterAccept -> OnAcceptCompeted -> RegisterAccept ...n
         //init한번으로 while처럼 반복되게 처리한 것 
         RegisterAccept(args);
     }
```


---

## 동작 순서

유저 코드
   ↓
Socket.AcceptAsync(SocketAsyncEventArgs)
   ↓
Windows 커널 (IOCP 등록)
   ↓
클라이언트 접속 발생
   ↓
IOCP 완료 큐에 등록
   ↓
.NET이 Completed 이벤트 발생
   ↓
args.Completed → OnAcceptCompleted 실행

---

## 문뜻 생각난 정리

서버코어부분에서  
이런식으로 while문을 돌리는 메인 스레드가있는데  
```csharp
  static void Main(string[] args)
  {
      string host = Dns.GetHostName();
      IPHostEntry ipHost = Dns.GetHostEntry(host);
      IPAddress ipAddr = ipHost.AddressList[0];
      IPEndPoint endPoint = new IPEndPoint(ipAddr, 7777);

      _listener.Init(endPoint, OnAcceptHandler);
      Console.WriteLine("Listening...");

      while(true)
      {
          ;
      }

  }
```


밑에 로직은  
OnAcceptCompleted -> RegisterAccept 돌리면서 뺑뺑이를 돌고있다  
실행시키면 두개가 따로 도는걸 확인 할 수잇는데  

```csharp
      SocketAsyncEventArgs args = new SocketAsyncEventArgs();
         args.Completed += new EventHandler<SocketAsyncEventArgs>(OnAcceptCompleted);
         RegisterAccept(args);
         
     private void RegisterAccept(SocketAsyncEventArgs args)
     {
         args.AcceptSocket = null;
         bool pending = _listenSocket.AcceptAsync(args);
  
         if (pending == false)
             OnAcceptCompleted(null, args);
     }

     private void OnAcceptCompleted(object sender, SocketAsyncEventArgs args)
     {
         //소켓에 간간히 에러가 나와서 처리해줌
         if (args.SocketError == SocketError.Success)
         {
             _onAcceptHandler.Invoke(args.AcceptSocket);
         }
         else
             Console.WriteLine(args.SocketError.ToString());
         RegisterAccept(args);
     }
```


찾아보니 스레드풀에서 하나를 꺼내와서 작업을 돌려주는걸 확인할 수 있다
![[Pasted image 20251023181847.png]]


## 즉!!
두개의 스레드가 돌아간다는 뜻은
메인스레드 와 작업자 스레드가 공용 데이터를 건드릴 수 있는
RaceCondition상태가 되기에
이제는 머리에 염두해두고 작업을 진행해야한다
동기화 문제를 생각하면서 코딩을 하자!

---


