---
tags:
  - reactContext
---
# React Context API 가이드

## Context란?
React Context는 **컴포넌트 트리를 통해 데이터를 전역적으로 공유**할 수 있게 해주는 React의 내장 기능입니다. Props drilling(prop을 여러 레벨에 걸쳐 전달하는 것) 없이 깊은 레벨의 컴포넌트들이 데이터에 접근할 수 있도록 해줍니다.
## 왜 Context를 사용하나요?
### 문제: Props Drilling
```tsx
// ❌ Props Drilling의 예시
function App() {
  const user = { id: 1, name: "홍길동" };
  return <Header user={user} />;
}

function Header({ user }) {
  return <Navigation user={user} />;
}

function Navigation({ user }) {
  return <UserProfile user={user} />;
}

function UserProfile({ user }) {
  return <div>{user.name}</div>;
}
```
### 해결: Context 사용
```tsx
// ✅ Context를 사용한 해결
const UserContext = createContext();

function App() {
  const user = { id: 1, name: "홍길동" };
  return (
    <UserContext.Provider value={user}>
      <Header />
    </UserContext.Provider>
  );
}

function UserProfile() {
  const user = useContext(UserContext);
  return <div>{user.name}</div>;
}
```

## Context vs Props의 비교

| 항목 | Props | Context |
|------|-------|---------|
| **데이터 전달** | 부모 → 자식으로만 | 어느 깊이든 직접 접근 |
| **코드 복잡성** | Props drilling 발생 | 깔끔한 코드 |
| **성능** | 필요한 컴포넌트만 리렌더 | Provider 하위 모든 컴포넌트 리렌더 가능 |
| **사용 케이스** | 지역적 데이터 | 전역적 데이터 (테마, 인증, 소켓 등) |

## 주의사항 및 Best Practices

### 1. Context 분리
```tsx
// ✅ 관심사별로 Context 분리
<SessionProvider>      {/* 인증 관련 */}
  <SocketProvider>     {/* 소켓 관련 */}
    <ThemeProvider>    {/* 테마 관련 */}
      {children}
    </ThemeProvider>
  </SocketProvider>
</SessionProvider>
```

### 2. 불필요한 리렌더링 방지
```tsx
// ✅ 값이 변경될 때만 새 객체 생성
const value = useMemo(() => ({
  client: clientRef.current,
  isConnected,
  sendMessage,
  subscribe,
  unsubscribe,
}), [isConnected]);
```

### 3. 에러 처리
```tsx
// ✅ Provider 외부 사용 시 명확한 에러
if (!context) {
  throw new Error('useSocket must be used within a SocketProvider');
}
```

### 4. TypeScript 활용
```tsx
// ✅ 타입 안정성 확보
interface SocketContextType {
  // 모든 메서드와 속성에 타입 지정
}
```

## 결론

React Context는 **전역 상태 관리**를 위한 강력한 도구입니다. 특히 소켓 연결처럼 애플리케이션 전체에서 공유되어야 하는 상태와 기능들을 관리할 때 매우 유용합니다.

프로젝트에서는 Context를 통해:
- 🔗 **소켓 연결 상태 전역 관리**
- 📡 **실시간 통신 기능 제공**  
- 🧹 **코드 복잡성 감소**
- 🔒 **타입 안정성 확보**
