## **1. 리액트 훅의 기본 철학과 설계 원칙**

### 훅의 등장 배경

- 클래스 컴포넌트는 재사용과 로직 분리가 어렵고, this 바인딩 등 문법적인 복잡함이 있었다.
- 함수형 컴포넌트가 존재하긴 했었지만, 상태 관리가 어렵고 생명주기 관리 방법이 표준화되어 있지 않았다.
- 이러한 상황에서 **함수형 컴포넌트에서도 상태, 생명주기, 컨텍스트 등을 사용할 수 있게 한 것**이 훅이다.
- 훅은 컴포넌트의 상태와 사이드 이펙트를 함수적 방식(**불변성** 이용)으로 제어한다.

<br/>

### 훅의 설계 원칙

1. **최상위에서만 호출**
    - 훅은 **컴포넌트 렌더링 흐름과 연결된 상태 머신**이다. (훅을 호출한 순서를 기준으로 상태를 연결)
    - 따라서 React가 각 훅의 상태를 렌더링 사이에 올바르게 매칭시키기 위해서는, 항상 컴포넌트의 최상위에서 동일한 순서로 호출되어야 한다.
        
        → 훅은 항상 최상위에서만 호출할 수 있으며(**Top-level only**), 조건문/반복문 안에서 호출할 수 없다.
        
2. **React 함수 내에서만 호출**
    - 훅은 오직 함수형 컴포넌트 또는 커스텀 훅에서만 사용할 수 있다.
    - 훅은 React 렌더링 컨텍스트 안에서 작동하는 상태 머신에 연결되어 있기 때문에, 일반 함수 밖에서 호출하면 상태 관리가 불가능하다.
    

<br/>

## **2. 훅의 작동 원리**

### Dispatcher + hookIndex 구조

- React에서 훅은 컴포넌트 함수 안에서 호출되지만, React는 각 훅이 어떤 상태와 연결되는지 기억해야 한다. 이때 사용하는 것이 Dispatcher와 hookIndex이다.
- **Dispatcher**
    - 현재 렌더링 중인 컴포넌트가 `useState`, `useEffect` 등을 호출할 때 **어떤 내부 훅 구현을 쓸지 결정하는 객체**를 말한다.
        
        ```jsx
        // 예시
        const ReactCurrentDispatcher = {
          current: mountDispatcher // mount 중일 때 사용하는 훅들
        };
        ```
        
        | 상황 | Dispatcher 종류 | 역할 |
        | --- | --- | --- |
        | 최초 렌더링 | `mountDispatcher` | 상태 초기화 |
        | 이후 업데이트 | `updateDispatcher` | 기존 상태 불러오기 |
        | 서버 컴포넌트 등 | `serverDispatcher`, `noopDispatcher` 등 | 상황별 분기 |
- **hookIndex**
    - 현재 컴포넌트에서 몇 번째 훅인지 추적하는 숫자 인덱스이다.
        - 리액트는 이 값을 기준으로 내부 훅 배열에 저장된 상태를 매핑한다.
    
    ```jsx
    let hookIndex = 0;
    const hookStates = []; // 훅 별 상태 저장 배열
    ```
    
    - 작동 예시
        
        ```jsx
        function MyComponent() {
          const [a, setA] = useState(0); // hookIndex = 0
          const [b, setB] = useState(1); // hookIndex = 1
        }
        ```
        
        - `hookIndex = 0` → `hookStates[0]`에 a 상태 저장
        - `hookIndex = 1` → `hookStates[1]`에 b 상태 저장
        - 렌더링이 반복되어도 **순서만 같다면** `hookIndex`에 따라 정확히 같은 상태에 접근할 수 있다.
- 전체 작동 흐름
    
    ```jsx
    function MyComponent() {
      const [count, setCount] = useState(0);
      const [text, setText] = useState("hello");
      ...
    }
    ```
    
    1. React가 함수 컴포넌트 실행
    2. `hookIndex = 0` → `useState(0)` 호출
    3. mount(초기 렌더링)이라면 Dispatcher가 초기 상태를 저장하고 반환
    4. `hookIndex = 1` → `useState("hello")`호출 
    5. 이후 hookIndex 증가, 모든 훅 처리 후 컴포넌트 종료
    6. 다음 렌더링 시 동일한 hookIndex 순서로 상태 재사용

<br/>

### 렌더링 흐름 요약

1. **초기 렌더링 (mount)**
    1. 렌더 트리거
    2. 함수 컴포넌트 실행
    3. 훅 호출 → React 내부 hookIndex 배열에 기록
    4. useState 초기값 설정 → 상태 Dispatcher 등록
    5. 렌더링 결과(VDOM) 계산 → 실제 DOM 반영
2. **상태 업데이트 발생 (re-render)**
    1. setState로 상태 변경
    2. 컴포넌트 재실행
    3. 훅 호출 순서에 맞춰 기존 상태 Dispatcher 재사용
    4. 클로저 내부 값 재생성 → 최신 값으로 렌더링 

<br/>

## **3. 클로저 이슈 & stale value 문제 완전 분석**

### stale closure

- 컴포넌트가 리렌더링되면서 새로운 state를 가지게 되었지만, 이벤트 핸들러 등에 할당된 클로저가 이전 렌더링의 값을 기억하는 것 (클로저가 오래된 값을 참조하는 현상)
    - 컴포넌트 내부에서 어떤 값(예: state)을 참조하는 콜백 함수(예: setTimeout, 이벤트 핸들러 등)가 생성될 때, 그 시점의 값을 클로저로 캡처한다.
    - 이후 상태(state)가 바뀌어도, 이미 생성된 클로저는 "자신이 만들어질 당시의 값"만을 기억한다.
    - 이로 인해, 콜백이 실행될 때 최신 값이 아니라 "오래된 값"을 사용하게 된다. 이 현상을 stale closure라고 부른다.

```jsx
const [count, setCount] = useState(0);

const handleClick = () => {
  console.log(count); // 항상 0으로 찍히는 이슈 가능성
  setCount(count + 1);
};
```

- 해결 방법
    1. 함수형 업데이트 사용
        
        ```jsx
        setCount(prev => prev + 1);
        ```
        
    2. `useRef`를 사용해서 상태 보존
        
        ```jsx
        const latestCount = useRef(count);
        
        useEffect(() => {
          latestCount.current = count;
        }, [count]);
        ```
        

<br/>

## **4. 고급 훅 전략**

### **useRef**

- 단순히 DOM 접근용이 아닌, **렌더링 사이에 값을 유지하는 변수**로도 활용할 수 있다.
- 렌더링을 발생시키지 않으며, **변경해도 컴포넌트는 리렌더되지 않는다.**
- 비동기 로직 중 취소 가능 여부 체크 등에도 활용할 수 있다.
    
    ```jsx
    const mounted = useRef(true);
    
    useEffect(() => {
      return () => {
        mounted.current = false;
      };
    }, []);
    ```
    

<br/>

### **useLayoutEffect vs useEffect**

| 항목 | useLayoutEffect | useEffect |
| --- | --- | --- |
| 실행 시점 | DOM 업데이트 후, paint 전 | 화면 그린 후 |
| 주요 용도 | 레이아웃 계산, 스크롤 위치 복원 | API 호출, 비동기 작업 |
| 주의점 | 사용자 경험 저하 가능 | 일반적인 비동기 작업에 적합 |
- DOM을 조작해야 하는 경우(포커스, 크기 계산 등)는 `useLayoutEffect`를 사용한다.

<br/>

### **useReducer**

- 상태 변경 로직이 복잡하거나 액션이 다양할 때 사용하기 적합하다. (ex. 여러 액션 타임, 중첩 상태)
- 상태 변경을 외부 함수로 분리해서 테스트와 유지보수가 용이해진다.

```jsx
const reducer = (state, action) => {
  switch (action.type) {
    case "INCREMENT":
      return { count: state.count + 1 };
  }
};

const [state, dispatch] = useReducer(reducer, { count: 0 });
```

<br/>

## **5. 커스텀 훅 설계 패턴과 책임 분리 전략**

### **단일 책임 원칙(SRP)** 기반으로 훅 설계

- 분리 기준
    - **상태 훅**: useToggle, useInput → 단순한 상태 관리
    - **동작 훅**: useClickOutside, useDebounce → 특정 동작을 캡슐화
    - **비즈니스 훅**: useFetch, useForm → 외부 API 또는 복잡한 UI 로직

<br/>

### Composable Hook 패턴

- 내부에서 여러 훅을 조합하여 복잡한 로직을 분리하는 패턴

```jsx
function useSearch(query) {
  const [debouncedQuery] = useDebounce(query, 300);
  const { data } = useFetch(`/search?q=${debouncedQuery}`);
  return data;
}
```

<br/>

## **6. 리액트 훅과 GC, 메모리 누수**

### 메모리 누수 패턴

1. **클로저로 인한 상태 유지 문제**
    - 함수형 컴포넌트에서 클로저가 오래된 상태나 props를 참조하거나, 예상치 못한 방식으로 메모리를 지속적으로 유지하는 경우에 발생할 수 있다.
    - 특히 `setInterval`, `setTimeout`, `addEventlistener`, `async` 내에서 클로저가 이전 상태를 캡쳐할 때 자주 발생한다.
    
    ```jsx
    useEffect(() => {
      const id = setInterval(() => {
        console.log(count); // 오래된 count 참조
      }, 1000);
    }, []);
    ```
    
2. **타이머 (setInterval, setTimeout)의 정리 누락**
    - 타이머는 명시적으로 clear하지 않으면 계속 살아있으며, 컴포넌트가 언마운트되어도 동작을 이어간다.
        
        ```jsx
        useEffect(() => {
          const id = setTimeout(() => {
            // 작업
          }, 5000);
        }, []);
        ```
        
3. **비동기 작업 중 컴포넌트 언마운트**
    - `fetch`, `axios`, `setState` 등을 `async` 함수로 처리하는 경우, 작업 완료 전에 컴포넌트가 언마운트되면 메모리 누수 및 경고가 발생한다.
    
    ```jsx
    useEffect(() => {
      fetchData().then(data => {
        setState(data); // 이미 언마운트되었을 수 있음
      });
    }, []);
    ```
    
4. **이벤트 리스너 제거 누락**
    - `window`, `document`, 외부 DOM 요소 등에 등록한 이벤트 리스너를 해제하지 않으면 계속 남아서 가비지콜렉팅되지 않는다
    
    ```jsx
    useEffect(() => {
      const onResize = () => console.log('resized');
      window.addEventListener('resize', onResize);
    }, []);
    ```
    

<br/>

### 해결 방법

1. **`useEffect`와 clean-up 함수 활용**
    - `useEffect`에서 타이머, 리스너, 비동기 상태 변경 등을 등록했다면, 반드시 return으로 정리(clean-up) 로직을 명시한다.
    
    ```jsx
    useEffect(() => {
      const id = setInterval(() => {
        console.log('interval');
      }, 1000);
    
      return () => {
        clearInterval(id); // 타이머 해제
      };
    }, []);
    ```
    
2. **비동기 작업 취소 or 언마운트 확인**
    - `AbortController`, `isMounted` 플래그 등을 활용해 비동기 작업을 중단 처리해준다.
    
    ```jsx
    useEffect(() => {
      let ignore = false;
    
      const load = async () => {
        const res = await fetch('/data');
        
        if (!ignore) {
          setState(await res.json());
        }
      };
      
      load();
    
      return () => {
        ignore = true; // 언마운트 시 플래그 설정
      };
    }, []);
    ```
    
3. **타이머와 리스너 해제**
    - 등록한 것은 반드시 해제해준다.
    
    ```jsx
    useEffect(() => {
      const handler = () => {};
      window.addEventListener('scroll', handler);
    
      return () => {
        window.removeEventListener('scroll', handler);
      };
    }, []);
    ```
    
4. **`useRef`를 통한 클로저 문제 회피**
    
<br/>


## **7. 훅 활용 실전 시나리오**

- 입력 지연 처리: `useDebounce`, `useDeferredValue` → 성능 개선
- 컴포넌트 간 상태 공유: `useContext` + `useReducer`
- 요청 취소: `AbortController` + `useEffect cleanup`
- 무한 루프 방지: `useEffect`의 deps에 불필요한 상태 넣지 않기

<br/>

## **8. 면접 질문 리스트**

- 왜 useState를 조건문에서 쓰면 안 되나요?
    - React의 훅은 실행 순서를 기준으로 내부적으로 상태를 매핑하기 때문에, 조건문 안에서 훅을 쓰면 매 렌더링마다 호출 순서가 달라질 수 있습니다. 그렇게 되면 React가 어떤 상태가 어떤 훅에 대응되는지 추적할 수 없기 때문에 에러가 발생할 수 있습니다. 그래서 항상 컴포넌트의 최상단에서, 조건문이나 반복문 밖에서 훅을 호출해야 합니다.
- useEffect는 언제 실행되며 어떤 순서로 클린업이 되나요?
    - useEffect는 컴포넌트가 화면에 렌더링된 이후에 실행됩니다. 그리고 의존성 배열에 변화가 생기면 먼저 이전 effect의 클린업 함수가 실행되고, 그 다음에 새 effect가 실행됩니다. 컴포넌트가 ㅇ너마운트될 때도 마지막 클린업이 호출됩니다.
- 상태가 stale하게 유지되는 이유는 무엇인가요?
    - stale 상태는 보통 클로저 때문입니다. 함수가 실행될 당시의 상태를 기억하고 있어서, 이후 상태가 바뀌어도 여전히 이전 값을 참조하는 것입니다. 예를 들어 타이머 안에서 setState를 할 때 최신 상태가 반영되지 않는 경우가 대표적입니다. 이럴 때는 상태 변경 함수에 이전 값을 기반으로 처리하는 방식이나, useRef를 써서 최신 값을 직접 추적하는 방법으로 해결할 수 있습니다.
- useRef와 useState의 차이는 무엇인가요?
    - useState는 값이 바뀌면 컴포넌트가 리렌더링되지만, useRef는 값이 바뀌어도 리렌더링이 안 됩니다. 그래서 useRef는 DOM에 접근할 때나 렌더링과 무관한 데이터를 유지할 때 씁니다.
- 커스텀 훅을 어떻게 구조화하고 분리하나요?
- useLayoutEffect는 언제 사용하고, 어떤 경우 주의해야 하나요?
    - useLayoutEffect는 화면에 그리기 전에 동기적으로 실행됩니다. 그래서 DOM의 크기나 위치를 계산하거나 스타일을 바꾸는 작업이 있을 때 사용하는 것이 적절합니다. 그런데 이 훅은 렌더링을 막기 때문에 성능에 민감한 환경에서는 조심해서 써야 합니다. 꼭 필요한 경우가 아니면 일반적인 useEffect로도 충분한 경우가 많습니다.
- useCallback과 memo가 항상 성능을 높이나요?
    - 항상 그렇진 않습니다. 이 둘은 리렌더링을 최적화할 때 도움은 되지만, 사용 자체가 비용이 들기 때문에 오히려 성능이 나빠질 수도 있습니다.
        
        그래서 저는 정말 자식 컴포넌트가 props로 함수를 받고 있고, 불필요한 리렌더링이 발생한다는 게 확인됐을 때만 사용하는 편이에요.
        
- 어떤 기준으로 커스텀 훅을 분리하시나요?
    - 반복되는 로직이 생길 때, 혹은 어떤 기능이 다른 기능과 명확하게 구분될 수 있을 때 커스텀 훅으로 분리합니다. 예를 들어 폼 상태 관리, 뷰포트 감지, 로딩 상태 제어 같은 것들이 있습니다. 기준은 ‘이 로직이 다른 곳에서도 쓸 수 있는가’와 ‘이 훅이 하나의 책임만 가지고 있는가’입니다.