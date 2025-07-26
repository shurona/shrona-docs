---
tags:
  - Stale-Closure
---
# 스테일 클로저 (Stale Closure) 문제
## 스테일 클로저란?
- **클로저가 생성된 시점의 변수 값을 "기억"하고 있어서, 최신 값이 아닌 예전 값을 참조하는 현상**
```tsx
const [messages, setMessages] = useState<ChatLogResponseDto[]>([]);

useEffect(() => {
  // 🔍 이 시점에서 messages = []
  
  const handleMessage = (message: ChatLogResponseDto) => {
    // ❌ 클로저가 생성된 시점의 messages([])를 계속 참조
    console.log('Current messages:', messages); // 항상 []
    const updated = [...messages, message]; // 항상 [message]만 됨
    setMessages(updated);
  };
  
  subscribe(`/topic/room/${room}`, handleMessage);
}, [selectedChatRoom]); // messages가 의존성에 없음!
```
## 스테일 클로저 발생 조건
1. **useEffect의 의존성 배열에 상태가 없음**
2. **콜백 함수가 외부 상태를 참조**
```
•	setInterval(() => { ... }) 안에서 count를 참조하고 있죠.
•	하지만 이 콜백은 useEffect 실행 시점의 클로저 내부에 만들어진 함수이기 때문에,
	그 시점의 count 값만 기억합니다.
•	이후 setCount()로 값이 바뀌어도, 이 setInterval 내부의 함수는 그걸 모릅니다.
```

3. **상태가 변경되어도 effect가 재실행되지 않음**

```tsx
// 스테일 클로저 예시
function Counter() {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    const timer = setInterval(() => {
      // ❌ 항상 초기값 0을 참조
      console.log('Count:', count); // 항상 0
      setCount(count + 1); // 항상 0 + 1 = 1
    }, 1000);
    
    return () => clearInterval(timer);
  }, []); // count가 의존성에 없음! 1번의 예시를 의미한다.
  
  return <div>{count}</div>;
}
```

### 위의 문제 출력값
```text
렌더링 1회 (count = 0)
→ useEffect 실행 (setInterval 시작, count = 0 캡처됨)
→ 1초 후 setCount(0 + 1) → count = 1
→ 리렌더링
→ 2초 후 setCount(0 + 1) → 여전히 count = 1
→ 계속 반복...
```

## 해결 방법들
### useRef 사용 (현재 적용된 해결책)
```tsx
const [messages, setMessages] = useState<ChatLogResponseDto[]>([]);
const messagesRef = useRef<ChatLogResponseDto[]>([]);

// ref와 state 동기화
useEffect(() => {
  messagesRef.current = messages;
}, [messages]);

useEffect(() => {
  subscribe(`/topic/room/${room}`, (message: ChatLogResponseDto) => {
    // ✅ ref는 항상 최신 값을 참조
    const prev = messagesRef.current;
    const updated = [...prev, message];
    messagesRef.current = updated;
    setMessages(updated);
  });
}, [selectedChatRoom, isConnected]);
```
**장점:**
- ref는 렌더링과 무관하게 항상 최신 값 유지
- 성능이 좋음 (리렌더링 없음)
**단점:**
- 코드가 복잡해짐
- ref와 state 동기화 필요

### 함수형 업데이트 사용
```tsx
useEffect(() => {
  subscribe(`/topic/room/${room}`, (message: ChatLogResponseDto) => {
    // ✅ 함수형 업데이트: 최신 상태를 매개변수로 받음
    setMessages(prevMessages => [...prevMessages, message]);
  });
}, [selectedChatRoom, isConnected]);
```
**장점:**
- 간단하고 직관적
- React 권장 패턴
**단점:**
- 상태를 즉시 읽을 수 없음 (비동기)

### 의존성 배열에 상태 추가
```tsx
useEffect(() => {
  subscribe(`/topic/room/${room}`, (message: ChatLogResponseDto) => {
    const updated = [...messages, message];
    setMessages(updated);
  });
  
  return () => {
    unsubscribe(`/topic/room/${room}`);
  };
}, [selectedChatRoom, isConnected, messages]); // ✅ messages 추가
```
**장점:**
- 항상 최신 상태 참조
**단점:**
- effect가 자주 재실행됨 (성능 문제)
- 무한 루프 가능성
## 참고 자료
- [React 공식 문서 - State Updates](https://react.dev/learn/queueing-a-series-of-state-updates)
- [React 공식 문서 - useRef](https://react.dev/reference/react/useRef)
- [JavaScript 클로저 이해하기](https://developer.mozilla.org/ko/docs/Web/JavaScript/Closures)