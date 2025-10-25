## 🧩 RecvBuffer의 역할

네트워크 프로그래밍에서 **소켓으로부터 받은 데이터를 임시로 저장하고 관리하는 버퍼**가 필요하다.  
`RecvBuffer`는 바로 그 역할을 수행하는 클래스다.

이 클래스는 OS에서 들어온 데이터(`recv`)와  
게임/서버 로직에서 처리하는 데이터(`read`)의 경계를 **r(read), w(write)** 포인터로 나누어 관리한다.

즉,

- **_readPos** : 콘텐츠(프로토콜)가 어디까지 읽었는가
    
- **_writePos** : OS(`Socket.ReceiveAsync`)가 어디까지 써놨는가
    

를 나타낸다.

---

## 📊 구조 개념 요약

|이름|설명|
|---|---|
|`_buffer`|실질적인 데이터가 저장되는 메모리 영역 (`ArraySegment<byte>`)|
|`_readPos`|현재까지 처리(읽기) 완료한 위치|
|`_writePos`|현재까지 수신(쓰기) 완료한 위치|
|`DataSize`|아직 처리되지 않은 데이터의 길이 (`_writePos - _readPos`)|
|`FreeSize`|추가로 쓸 수 있는 남은 공간 (`_buffer.Count - _writePos`)|

---

## 📦 ArraySegment를 사용하는 이유

단순히 `byte[]` 배열만 써도 되지만,  
`ArraySegment<byte>`를 사용하면 **대용량 버퍼를 부분적으로 재사용**하기 쉽다.

예를 들어:

- 10KB짜리 큰 버퍼 중에서 `[2000~4000]` 영역만 `recv` 용도로 쓰고 싶을 때,
    
- 전체 배열을 복사하지 않고 **세그먼트(부분)**만 넘길 수 있다.
    

이 구조는 서버에서 **수백, 수천 클라이언트**를 동시에 처리할 때 메모리 복사를 최소화하는 데 큰 도움이 된다.

---

## 📈 동작 원리 (r/w 포인터 흐름)

### 1️⃣ 기본 개념

`[rw][ ][ ][ ][ ][ ][ ][ ][ ][ ]`

- 처음엔 `r=w=0`, 즉 아무 데이터도 없음.
    

### 2️⃣ 클라가 5바이트를 보냄

`[r][ ][ ][ ][ ][w][ ][ ][ ][ ]`

- `recv` 성공 → `_writePos`가 5 증가.
    
- 아직 콘텐츠단에서 처리 안 했으니 `_readPos`는 그대로.
    

### 3️⃣ 콘텐츠에서 패킷을 처리 완료

`[ ][ ][ ][ ][rw][ ][ ][ ][ ][ ]`

- 콘텐츠단이 5바이트 소비 완료 → `_readPos = _writePos = 5`.
    

---

## 🌀 부분 수신 및 복사(Compaction)

### 문제 상황 예시

- 2바이트짜리 패킷을 클라이언트가 두 개 연속 보냄 (총 4바이트)
    
- 근데 첫 `recv`에선 3바이트만 옴
    

`[r][ ][ ][w][ ][ ][ ][ ][ ][ ]   (유효 데이터 3바이트)`

- 콘텐츠는 “2바이트짜리 패킷” 하나만 처리 가능함 → 첫 2바이트 소비
    
- 남은 1바이트는 다음 패킷이 도착하기 전까진 처리 불가  
    → `_readPos = 2, _writePos = 3`  
    → 대기
    

이 상태가 누적되면:

- r이 계속 증가하고
    
- w도 뒤로 밀리면서 **뒤쪽 여유 공간이 점점 줄어듦**
    

이때, 남은 데이터(`w-r`)를 앞으로 복사해서  
r/w를 처음으로 돌려주는 것이 **`Clean()` 함수의 역할**이다.

---

## 🧹 Clean() 함수 작동 방식

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

|케이스|동작|
|---|---|
|`DataSize == 0`|남은 데이터가 없으므로 커서만 리셋|
|`DataSize > 0`|남은 데이터를 앞으로 복사하고 r/w 재정렬|

결과적으로,  
버퍼 뒤쪽 여유 공간을 확보해주고  
메모리 낭비 없이 `recv`를 계속 수행할 수 있다.

---

## 🧮 OnRead / OnWrite 함수

### ▶ OnRead(int numOfBytes)

콘텐츠에서 패킷을 성공적으로 처리했을 때 호출  
→ **r 포인터 이동**

`public bool OnRead(int numOfBytes) {     if (numOfBytes > DataSize)         return false;  // 처리 가능한 데이터보다 많이 읽으려 함 (에러)     _readPos += numOfBytes;     return true; }`

---

### ▶ OnWrite(int numOfBytes)

소켓에서 데이터를 수신했을 때 호출  
→ **w 포인터 이동**

`public bool OnWrite(int numOfBytes) {     if (numOfBytes > FreeSize)         return false;  // 버퍼 범위 초과 (비정상)     _writePos += numOfBytes;     return true; }`

---

## 📑 ReadSegment / WriteSegment

`SocketAsyncEventArgs`와 연동될 때 핵심 포인트.

### ReadSegment

- 아직 처리되지 않은 데이터 구간 `[r, w)`
    
- 콘텐츠단에서 읽을 때 사용
    

### WriteSegment

- 다음 수신 시 쓸 수 있는 구간 `[w, end)`
    
- `Socket.ReceiveAsync()` 호출 시 인자로 전달됨
    

`public ArraySegment<byte> ReadSegment =>     new ArraySegment<byte>(_buffer.Array, _buffer.Offset + _readPos, DataSize);  public ArraySegment<byte> WriteSegment =>     new ArraySegment<byte>(_buffer.Array, _buffer.Offset + _writePos, FreeSize);`

---

## ⚙️ RecvBuffer의 전체 데이터 흐름 요약

1. **Socket.ReceiveAsync() 호출**
    
    - → `WriteSegment`로 버퍼 포인터 전달
        
2. **데이터 수신 완료 이벤트 발생**
    
    - → `OnWrite()`로 w포인터 이동
        
3. **콘텐츠(프로토콜)에서 패킷 파싱**
    
    - 패킷 단위로 읽고 처리 → `OnRead()` 호출
        
4. **r/w가 멀어지면** 버퍼 공간 부족 → `Clean()` 호출로 앞으로 복사
    

---

## 📘 마무리 요약

|함수|역할|
|---|---|
|`DataSize`|처리되지 않은 데이터의 길이|
|`FreeSize`|버퍼의 남은 공간|
|`OnRead()`|처리 완료 후 read 포인터 이동|
|`OnWrite()`|수신 완료 후 write 포인터 이동|
|`Clean()`|남은 데이터 앞으로 복사(버퍼 정리)|
|`ReadSegment`|콘텐츠단에서 읽을 구간|
|`WriteSegment`|다음 recv에서 쓸 구간|

---

## ✅ 한 줄 요약

> `RecvBuffer`는 네트워크에서 “조각나서 들어오는 데이터”를 **안전하게 누적, 처리, 정리**하는 버퍼 클래스다.  
> 읽기/쓰기 포인터를 분리함으로써 데이터 손실 없이 패킷 단위 처리를 구현할 수 있다.