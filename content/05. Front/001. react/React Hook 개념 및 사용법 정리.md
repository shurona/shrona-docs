---
tags:
  - hook
  - useState
  - useRef
  - useEffect
---
# UseState란
##  개념
React에서 함수형 컴포넌트에서 상태(state)를 관리하기 위해 사용되는 Hook입니다. 이를 통해 컴포넌트 내에서 변수를 선언하고, 해당 변수의 값을 변경하며, 변경 시 컴포넌트를 다시 렌더링하여 화면을 업데이트할 수 있습니다
## 사용법
- `useState()` 함수는 초기 상태 값을 인수로 받아, 현재 상태 값과 해당 상태를 업데이트하는 함수를 반환한다.
```Javascript
const [count, setCount] = useState(0);
```
## 상태 변경 및 렌더링
`setCount()` 함수를 사용하여 `count` 값을 변경하면, React는 변경 사항을 감지하고 해당 컴포넌트를 다시 렌더링하여 화면에 변경된 값을 반영합니다

# UseRef란
## 개념
useRef는 리액트에서 값을 유지하고 DOM 요소에 접근하기 위해 사용되는 훅입니다. useRef는 컴포넌트가 렌더링될 때마다 새로운 값을 생성하는 useState와 달리, 렌더링 전후에 동일한 값을 유지합니다. 또한, useRef는 current 속성을 통해 접근 가능한 객체를 반환하며, 이 객체의 current 값을 변경해도 컴포넌트가 다시 렌더링되지 않습니다
- current라는 키값을 지닌 프로퍼티가 생성되고, 값에 어떤 변경을 줄때도 이 current를 이용해서 한다는 점이다.
- ref의 값은 컴포넌트의 전생애주기를 통해서 관리되기 때문에, 아무리 컴포넌트가 렌더링 되더라도 언마운트가 되기 전까지는 값을 계속해서 유지한다.
## 특징
- 값 유지
	- 컴포넌트가 리렌더링되어도 값이 유지된다.
- Dom 요소 접근
	- 특정 DOM 요소에 직접 접근하여 조작할 수 있다.
- 렌더링을 유발하지 않는다.
	- useRef의 current 값을 변경해도 컴포넌트가 다시 렌더링되지않는다.
- 참조 객체
	- useRef는 { current: value } 형태의 객체를 반환한다.

# useEffect
## 개념
useEffect는 React 함수형 컴포넌트에서 side effect(부수 효과)를 처리하기 위해 사용되는 Hook입니다. 컴포넌트가 렌더링된 후에 특정 작업을 수행하거나, 컴포넌트가 언마운트되기 전에 정리 작업을 수행할 때 사용됩니다. 클래스형 컴포넌트의 componentDidMount, componentDidUpdate, componentWillUnmount의 기능을 통합한 것으로 볼 수 있습니다.

## 사용법
```javascript
useEffect(() => {
  // 실행할 부수 효과
}, [의존성 배열]);
```

- 첫 번째 인수: 실행할 함수 (effect function)
- 두 번째 인수: 의존성 배열 (dependency array)

## 특징
- **컴포넌트 렌더링 후 실행**
	- 컴포넌트가 DOM에 렌더링된 후에 effect가 실행됩니다.
- **의존성 배열 제어**
	- 빈 배열 `[]`: 컴포넌트 마운트 시에만 실행
- 배열 없음: 매 렌더링마다 실행
	- 값이 있는 배열: 해당 값이 변경될 때만 실행
- **정리(cleanup) 함수**
	- effect 함수에서 함수를 반환하면, 컴포넌트 언마운트 시 또는 다음 effect 실행 전에 정리 함수가 실행됩니다.

```javascript
useEffect(() => {
  const timer = setInterval(() => {
    console.log('Timer running');
  }, 1000);

  return () => {
    clearInterval(timer); // 정리 함수
  };
}, []);
```