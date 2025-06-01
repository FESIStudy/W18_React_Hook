## **1. 리액트 훅의 기본 철학과 설계 원칙**

기존 클래스형 컴포넌트에서는 생명주기를 생명주기 함수로 다루었다. 이후 ES6 문법으로 넘어오면서 함수형 컴포넌트가 나오게 되었고, 이 함수형 컴포넌트에서 생명주기를 다루기 위해 리액트 훅이 나오게 되었다. 

리액트 훅을 사용하지 않던 방식에 비해 컴포넌트로부터 상태 관련 로직을 추상화 할 수 있어 독립적인 테스트와 재사용이 가능해졌고, 여러 생명주기 메서드와 복잡한 상태관리를 좀 더 간편하게 다룰 수 있어 더 작은 단위로 컴포넌트를 나눌 수 있게 되었습니다.

설계 원칙은 크게 2가지가 있습니다. 최상위(Top Level)에서만 호출해야하며, 오직 React 함수 내에서만 훅을 호출해야 합니다. 즉, 반복문, 조건문, 일반 함수에서는 호출이 불가능하며 React 함수 컴포넌트나 Custom Hook에서 호출할 수 있습니다.

---

## **2. 훅의 작동 원리 (렌더링과 메모리 흐름 기반) -** Dispatcher와 Queue

### 2-1. 개념 정리: Dispatcher와 Queue

1. **Dispatcher**
    - 리액트는 훅을 호출하는 시점이 “마운트(처음 렌더)”인지 “업데이트(재렌더)”인지 구분해서 서로 다른 내부 로직을 실행하기 위해, 훅 호출을 담당하는 함수 집합(함수 포인터)을 Dispatcher라고 부른다.
    - 쉽게 말해, 마운트 시에는 “마운트용 useState 로직”을, 재렌더 시에는 “업데이트용 useState 로직”을 호출하도록 연결을 바꿔 주는 역할.
    - 그래서 같은 `useState`라는 이름으로 호출하더라도, 실제로 내부에서 실행되는 코드는 “마운트 단계”와 “업데이트 단계”에서 서로 다르게 작동한다.
2. **Queue (업데이트 큐)**
    - **Queue**는 “이전 렌더에서 호출된 setState 요청(업데이트 액션들)을 순서대로 보관하는 연결 리스트” 같은 자료구조다.
    - `setState` 함수가 호출될 때마다 “업데이트해야 할 정보(액션, 또는 값)”를 이 Queue에 추가(enqueue)한다.
    - 그리고 다음 렌더링이 시작되기 직전에, 이 Queue에 쌓여 있는 모든 액션을 꺼내(process)서(→ dequeue) 최종 새로운 상태값을 계산한 뒤, 그 값을 저장하고 Queue를 비우는 방식으로 동작한다.

### 2-2. 단계별 동작 흐름

### A. 최초 마운트 시 (Mount Phase)

1. **Dispatcher 연결**
    - 컴포넌트가 처음 렌더링될 때, 리액트는 “마운트 전용 Dispatcher”를 훅 호출 엔진에 연결한다.
    - 이 상태에서 `useState(initialValue)`를 호출하면, 내부적으로 다음 작업이 수행된다.
        1. **새로운 훅 노드(hook node)를 생성**
            - 내부 구조로는 `{ memoizedState: initialValue, queue: 빈 연결 리스트 }` 같은 데이터가 만들어진다.
        2. **훅 리스트에 노드 추가**
            - 이 훅 노드가 컴포넌트에 연결된 Fiber의 훅 리스트 끝에 붙는다.
        3. **초기 상태값을 반환**
            - `useState`가 `[state, setState]` 형태로 반환될 때, `state = initialValue` 이다.
            - `setState`는 “이 훅 노드의 queue에 업데이트 액션을 넣도록” 참조가 걸려 있는 함수 형태다.
2. **렌더 마무리**
    - 컴포넌트가 반환한 JSX를 화면에 렌더링하고, 마운트용 Dispatcher는 “업데이트용 Dispatcher”로 교체된다.
    - 즉, 다음번 렌더링부터는 “업데이트 로직”을 실행하는 dispatcher 함수가 훅 호출을 처리하게 된다.

### B. 상태 업데이트 요청 (Calling setState)

1. **setState 호출 → Queue에 액션 저장**
    
    예를 들어, 버튼 클릭 핸들러에서 `setCount(prev => prev + 1)`를 호출하면:
    
    - 내부적으로 “지금 이 훅 노드의 queue”가 연결 리스트 형태로 존재하므로, 거기에 `{ action: prev => prev + 1 }`를 추가한다.
    - 이때 상태 변화가 곧바로 일어나지는 않고, 오직 “queue에 쌓이기만” 함.
    - 그리고 리액트는 해당 컴포넌트를 “다시 렌더링해야 한다”는 스케줄을 등록한다.
2. **다음 렌더 전 Dispatcher가 Queue를 처리**
    - 리액트가 실제로 다시 렌더링을 시작하기 직전, “업데이트용 Dispatcher”가 활성화된 상태로 훅을 호출한다.
    - `useState`(업데이트용)가 호출되면, 내부에서 다음을 수행:
        1. **훅 노드의 queue를 확인**
            - queue에 쌓인 모든 액션(함수 혹은 값)들을 순서대로 꺼낸다.
        2. **새 상태값 계산**
            - 예시: queue에 `[prev => prev + 1, prev => prev + 1]` 두 개의 업데이트 액션이 쌓여 있으면,
                1. 첫 번째 액션 적용: `prevState(이전 저장된 값) → +1`
                2. 두 번째 액션 적용: “첫 번째 액션이 반영된 값” → +1
            - 이렇게 queue에 있는 모든 액션을 순서대로 적용해서 최종 state를 구한다.
        3. **훅 노드의 state 업데이트 및 queue 비우기**
            - 계산된 최종 state를 훅 노드의 `memoizedState`(저장 슬롯)에 저장
            - queue는 빈 상태(초기화)로 돌려놓음
        4. **반환**
            - `useState`는 새로 계산된 state 값을 반환하고, 두 번째 자리에는 “이제 또 future update를 enqueue할 수 있는 setState 함수”를 반환
3. **컴포넌트 재실행**
    - 최종적으로 “업데이트된 state”를 가지고 컴포넌트 함수가 다시 실행되고, 그 결과로 반환된 JSX가 화면에 반영된다.

### 2-3. Dispatcher + Queue가 왜 중요한가?

- **Dispatcher**
    1. 훅이 **마운트 시**에 해야 할 초기화 로직과, **업데이트 시**에 해야 할 queue 적용 로직을 분리하기 위해 사용
    2. 마운트 단계에서는 “새 훅 노드 생성 + 초기 상태 저장” 로직이 필요하고, 업데이트 단계에서는 “queue에 쌓인 액션 처리 + 상태 변경” 로직이 필요
    3. 따라서 내부적으로 두 가지 훅 구현 함수를 갖고 있다가(예: `mountState`와 `updateState`), 필요에 따라 dispatcher를 바꿔 가며 호출함
- **Queue**
    1. `setState`가 호출될 때마다 즉시 state를 덮어쓰지 않고, “업데이트 요청을 모아 두는 역할”
    2. 비동기적으로 여러 번 `setState`가 호출돼도(예: 이벤트 핸들러 내에서 `setCount(…)`를 여러 번 호출)
        - 각각을 queue에 쌓아 두었다가, 렌더링 직전 한 번에 차례대로 적용한다
        - 이를 통해 “동일 이벤트 루프 내에서 여러 setState 호출”이 하나의 렌더 싸이클에서 합쳐져 효율적으로 처리됨
    3. 업데이트 로직을 순차적으로 적용하기 때문에, 항상 “이전 값(prev)”을 정확히 계산할 수 있고, 최신 state를 보장함

---

### **3. 클로저 이슈 & stale value 문제 완전 분석**

### 3.1 Stale Closure 문제란?

- **클로저**: 함수가 선언된 시점의 변수(state/props)를 기억.
- React 컴포넌트의 이벤트 핸들러나 `setTimeout` 콜백 등은 선언 시점의 “과거 state”를 클로저로 캡처하기 때문에, 이후 state가 바뀌어도 여전히 예전 값을 참조할 수 있다.
    - 예: `setTimeout(() => setCount(count + 1), 1000)` → `count`가 0 → 1초 뒤 실행 시에도 0을 기반으로 +1 → 의도와 다른 결과 발생.

### 3.2 왜 이런 문제가 발생할까?

- 컴포넌트가 렌더링될 때마다 새로운 state가 저장되지만, 이미 만들어진 콜백(클로저)은 **최초 렌더 시점의 state**를 그대로 갖고 있음
- 따라서 비동기 콜백이 실행될 때 “최신 state”가 아닌 “캡처된 과거 state”로 동작함

### 3.3 해결법

- 함수형 업데이트 사용
    
    ```jsx
    // 클로저가 캡처한 count 값 대신,
    // 최신 state(prev)를 받아서 계산 → stale 방지
    setTimeout(() => {
      setCount(prev => prev + 1);
    }, 1000);
    ```
    
- useRef로 최신 값 보관
    
    ```jsx
    const countRef = useRef(count);
    useEffect(() => { countRef.current = count; }, [count]);
    
    setTimeout(() => {
      // 항상 countRef.current(최신) 사용
      setCount(countRef.current + 1);
    }, 1000);
    ```
    
- useCallback 의존성 배열 관리
    
    ```jsx
    // deps에 count를 넣어야 최신 값을 참조하는 함수가 생성됨
    const handler = useCallback(() => {
      setCount(count + 1);
    }, [count]);
    ```
    
- useEffect 사용하기
    
    ```jsx
    useEffect(() => {
      // count가 변경된 후 실행됨
      console.log("변경된 값:", count);
    }, [count]);
    ```
    

### 3-4 useState 직접 구현해보기

```jsx
function useState(initialVal) {
  let _val = initialVal;
  const state = () => _val;
  const setState = (newVal) => {
    _val = newVal;
  };

  return [state, setState];
}

const [count, setCount] = useState(1);

console.log(count()); //1
setCount(2);
console.log(count()); //2
```

---

## **4. 고급 훅 전략: useRef, useLayoutEffect, useReducer**

### **4-1. useRef 심화 활용**

- **값 보존 용도**
    - `useRef`는 렌더 간에 동일한 객체를 유지하므로, 값이 변경돼도 리렌더링을 트리거하지 않는다.
    - 이를 이용해 “이전 렌더의 state/props”를 저장하거나, “비동기 로직이 최신 값을 참조”하도록 만들 수 있다.
    
    ```jsx
    jsx
    복사편집
    const [count, setCount] = useState(0);
    const prevCount = useRef(count);
    useEffect(() => {
      prevCount.current = count; // 렌더 후 즉시 저장 → next 렌더에 이전 값 사용 가능
    }, [count]);
    
    ```
    
    - 위 예시에서 `prevCount.current`는 항상 “직전 렌더의 count”를 가리킨다.
- **비동기 작업 취소·상태 확인**
    - 예: `fetch`나 `setTimeout` 콜백이 실행되기 전, 컴포넌트가 언마운트되었는지 확인
    
    ```jsx
    jsx
    복사편집
    const isMounted = useRef(true);
    useEffect(() => {
      return () => { isMounted.current = false; }; // 언마운트 시 false 설정
    }, []);
    
    useEffect(() => {
      fetch('/api/data').then(res => res.json()).then(data => {
        if (isMounted.current) setData(data); // 마운트된 상태에서만 업데이트
      });
    }, []);
    
    ```
    
    - 이렇게 하면 “언마운트 후 setState”로 인한 경고나 메모리 누수를 방지할 수 있다.

### **4-2. useLayoutEffect vs useEffect**

- **useEffect**
    - **실행 시점**: 렌더 → 브라우저가 화면을 그린 뒤(페인트 후)
    - **용도**: 데이터 페칭, 이벤트 리스너 등록/해제, 타이머 설정, 외부 API 호출 등
    - **특징**: 화면 깜빡임(flicker) 없이 사용자가 먼저 화면을 보고, 백그라운드에서 작업이 처리된다.
    
    ```jsx
    jsx
    복사편집
    useEffect(() => {
      const id = setInterval(() => console.log('tick'), 1000);
      return () => clearInterval(id);
    }, []);
    
    ```
    
- **useLayoutEffect**
    - **실행 시점**: 렌더 → DOM 업데이트 직후(화면 그리기 전)
    - **용도**: DOM 크기/위치 측정, 레이아웃 계산, 스크롤 위치 조정, 포커스 제어 등
    - **특징**: “페인트 전에” 동기적으로 실행되어야 할 작업에 적합. 화면이 그려지기 전에 수정이 끝나므로 깜빡임을 막아준다.
    
    ```jsx
    jsx
    복사편집
    useLayoutEffect(() => {
      const rect = ref.current.getBoundingClientRect();
      // 예: 레이아웃 계산 후 스타일 조정
      ref.current.style.transform = `translateY(${rect.height}px)`;
    }, [dependencies]);
    
    ```
    
- **선택 기준**
    1. **레이아웃 측정·조정이 필요할 때** → `useLayoutEffect`
    2. **단순한 비동기 로직, 데이터 로드, 이벤트 등록 등** → `useEffect`

### **4-3. useReducer 설계 패턴**

- **언제 사용?**
    1. 상태 객체에 여러 속성이 있고, “하나의 액션”이 여러 속성을 동시에 변경할 때
    2. 상태 변경 로직을 “action(type) + payload” 형태로 분리하여 관리하고 싶을 때
    3. 외부에 reducer 함수를 두어 **단위 테스트**나 **재사용**이 필요할 때
- **기본 구조**
    
    ```tsx
    tsx
    복사편집
    // 1) 초기 상태
    interface State { count: number; text: string; }
    const initialState: State = { count: 0, text: '' };
    
    // 2) Action 타입 정의
    type Action =
      | { type: 'INCREMENT' }
      | { type: 'SET_TEXT'; payload: string };
    
    // 3) reducer 작성
    function reducer(state: State, action: Action): State {
      switch (action.type) {
        case 'INCREMENT':
          return { ...state, count: state.count + 1 };
        case 'SET_TEXT':
          return { ...state, text: action.payload };
        default:
          return state;
      }
    }
    
    // 4) 컴포넌트에서 useReducer 사용
    function MyComponent() {
      const [state, dispatch] = useReducer(reducer, initialState);
      return (
        <><p>Count: {state.count}</p>
          <p>Text: {state.text}</p>
          <button onClick={() => dispatch({ type: 'INCREMENT' })}>+</button>
          <input onChange={e => dispatch({ type: 'SET_TEXT', payload: e.target.value })} />
        </>
      );
    }
    
    ```
    
    - `dispatch`를 호출하면, 리액트는 `reducer`를 실행해 “새로운 state”를 계산하고, 그에 따라 컴포넌트를 재렌더링한다.
- **장점**
    1. **로직 분리**: 복잡한 상태 변경 로직을 컴포넌트 외부의 순수 함수로 분리 → 코드 가독성↑
    2. **테스트 용이성**: 단순한 함수이므로 “주어진 state + action → 예상 state” 형태로 단위 테스트 가능
    3. **여러 action 관리**: `switch-case` 구조로 다양한 액션을 깔끔하게 처리 가능
- **주의사항**
    - 상태가 단순(예: boolean 토글, 단순 카운터)할 때는 `useState`가 더 가볍고 직관적
    - `useReducer`로 가독성이 오히려 떨어질 정도로 간단한 경우는 피하는 것이 좋음

---

## **5. 면접 질문 리스트 & 예상 답안 정리**

- **useEffect는 언제 실행되며 어떤 순서로 클린업이 되나요?**
    
    `useEffect`는 컴포넌트가 렌더링된 후, 브라우저가 화면에 그리기를 완료한 시점에 호출됩니다. 의존성 배열(deps)을 비워 두면 마운트 직후 한 번만 실행되고, deps에 값이 있을 경우 해당 값이 변경될 때마다 실행됩니다. 그리고 다음 이펙트가 실행되기 전, 또는 컴포넌트가 언마운트되기 직전 **이전 이펙트의 반환(clean-up) 함수**가 먼저 호출됩니다. 정리하자면 “렌더 → 브라우저 페인트 → 이펙트 실행 → 다음 이펙트 실행 직전 또는 언마운트 직전 클린업 실행” 순서로 동작합니다.
    
- **상태가 stale하게 유지되는 이유는 무엇인가요?**
    
    React 함수형 컴포넌트 내부에서 정의된 함수(이벤트 핸들러, 타이머 콜백 등)는 **함수가 생성될 당시의 state/props 값을 클로저**로 캡처하기 때문에, 이후 state가 바뀌어도 기존 클로저에는 여전히 “캡처된 과거 값”이 남아 있게 됩니다. 예를 들어 `setTimeout(() => setCount(count + 1), 1000)`처럼 작성했을 때, 콜백 내부의 `count`는 타이머를 설정한 시점의 값이기 때문에 실행 시점의 최신 값을 반영하지 않아 stale 상태가 됩니다. 이를 방지하려면 함수형 업데이트 또는 ref를 사용해 항상 최신 상태를 참조해야 합니다.
    
- **useRef와 useState의 차이는 무엇인가요?**
    
    `useState`는 상태가 변경될 때마다 컴포넌트를 **재렌더링**하도록 트리거하기 위해 사용되며, 리액트가 업데이트 스케줄을 관리합니다. 반면 `useRef`는 `.current` 프로퍼티에 값을 저장하지만, 이 값을 변경해도 **렌더링이 일어나지 않습니다**. 따라서 `useRef`는 “값을 렌더 사이에 유지(메모리 보존)하되, 화면 갱신이 필요 없는 경우”에 유용합니다. 예로 이전 렌더 값을 기억하거나, 비동기 콜백에서 최신 값을 확인할 때 사용하며, DOM 노드 참조에도 활용됩니다.
    
- **커스텀 훅을 어떻게 구조화하고 분리하나요?**
    
    커스텀 훅 설계 시 단일 책임 원칙(SRP)을 적용해, 하나의 훅은 한 가지 역할만 담당하도록 분리합니다. 예컨대 `useInput`은 입력값 관리용, `useToggle`은 불리언 토글용으로 만들고, 비동기 데이터 패칭은 `useFetch` 훅으로 따로 분리하는 식입니다. 내부에서는 여러 기본 훅(useState, useEffect 등)을 조합해 구현하며, **재사용 가능한 API**(예: 훅이 반환하는 값과 핸들러 인터페이스)가 직관적이어야 합니다. 또한 로직이 복잡할 경우 훅 내에서 상태 관리 로직을 분리하고, 반환값만 알맞게 노출해 “결합도를 낮추고 응집도를 높이는” 구조로 만드는 것이 좋습니다.
    
- **useLayoutEffect는 언제 사용하고, 어떤 경우 주의해야 하나요?**
    
    `useLayoutEffect`는 브라우저가 **화면을 그리기(페인트) 전**에 동기적으로 실행되므로, “DOM 크기나 레이아웃을 측정하고 바로 수정해야 할 때” 사용합니다. 예컨대 모달이 열릴 때 스크롤 위치를 즉시 조정하거나, 렌더 직후 요소 크기를 읽어 스타일을 변경해야 할 경우에 유용합니다. 하지만 동기적으로 작동하므로 **렌더링을 블로킹**할 수 있고, 잘못 사용하면 첫 화면이 느리게 보일 수 있으니 “레이아웃 측정/변경이 반드시 필요한 경우에만” 제한적으로 사용해야 합니다. 일반적인 비동기 작업이나 데이터 패칭에는 `useEffect`를 쓰는 편이 성능 및 사용자 경험 면에서 안전합니다.
    
- **useCallback과 memo가 항상 성능을 높이나요?**
    
    `useCallback`과 `React.memo`는 “함수를 재사용하거나 컴포넌트 리렌더를 방지”하기 위해 사용되지만, **항상 성능을 높여주지는 않습니다**. 두 훅 모두 내부적으로 의존성 검사나 props 비교 과정을 수행하기 때문에, 비교 비용이 실제 렌더링 비용보다 커지면 오히려 성능 저하를 유발할 수 있습니다. 따라서 **컴포넌트가 자주 리렌더링되고, 자식으로 전달되는 함수/객체가 크거나 복잡할 때**만 사용을 고려하는 것이 좋습니다. 간단한 컴포넌트나 빈번히 변경되는 값에는 불필요한 오버헤드가 생길 수 있음을 항상 염두에 두어야 합니다.
    

---

## 참고자료

https://ko.legacy.reactjs.org/docs/hooks-intro.html

https://ingg.dev/hook-work/

[https://velog.io/@boyeon_jeong/Hook의-동작원리-파헤쳐보기React-코드-까보기-03-dispatch-함수](https://velog.io/@boyeon_jeong/Hook%EC%9D%98-%EB%8F%99%EC%9E%91%EC%9B%90%EB%A6%AC-%ED%8C%8C%ED%97%A4%EC%B3%90%EB%B3%B4%EA%B8%B0React-%EC%BD%94%EB%93%9C-%EA%B9%8C%EB%B3%B4%EA%B8%B0-03-dispatch-%ED%95%A8%EC%88%98)

[https://velog.io/@njt6419/React-Hooks-파헤치기-1-useState-hooks-구현채#참고-자료](https://velog.io/@njt6419/React-Hooks-%ED%8C%8C%ED%97%A4%EC%B9%98%EA%B8%B0-1-useState-hooks-%EA%B5%AC%ED%98%84%EC%B1%84#%EC%B0%B8%EA%B3%A0-%EC%9E%90%EB%A3%8C) (추천!!!!)

[https://velog.io/@imnotmoon/React-Hook이-동작하는-방식](https://velog.io/@imnotmoon/React-Hook%EC%9D%B4-%EB%8F%99%EC%9E%91%ED%95%98%EB%8A%94-%EB%B0%A9%EC%8B%9D)

[https://velog.io/@alsgud8311/React-Fiber-아키텍처-딥다이브](https://velog.io/@alsgud8311/React-Fiber-%EC%95%84%ED%82%A4%ED%85%8D%EC%B2%98-%EB%94%A5%EB%8B%A4%EC%9D%B4%EB%B8%8C)

https://handhand.tistory.com/m/264

https://jellajellaangela.tistory.com/45
