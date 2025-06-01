## 리액트 훅이 도입된 이유

1. 상태 로직의 분리와 재사용성 증대를 위해
    
    기존에는 상태 관리나 사이드 이펙트를 처리하려면 클래스 컴포넌트를 사용해야 했고, 이를 함수형 컴포넌트에 붙이기 위해서는 고차 컴포넌트나 Render Props 같은 패턴을 사용해야 했는데, 이런 패턴은 계층 구조를 복잡하게 하고 래퍼 지옥(Wrapper Hell)을 유발했다.
    훅을 사용하면 계층 구조를 바꾸지 않고도 상태 로직을 커스텀 훅으로 분리하고 재사용 할 수 있다.
    
2. 복잡해진 컴포넌트의 가독성 개선을 위해
    
    클래스 컴포넌트의 라이프사이클 메서드안에 서로 관련없는 로직이 뒤섞이기 쉽고, 관련 있는 코드가 분산되어 유지보수가 어려웠다.
    훅은 관련 있는 로직끼리 한곳에 모아 코드를 이해하기 쉽고 테스트하기 용이하다.
    
3. 클래스 문법이 주는 학습 장벽과 도구 최적화 한계 극복을 위해
    
    this 바인딩, 생성자 호출 등 클래스의 문법은 초심자가 이해하기 어렵다.
    또한 클래스는 minify나 Hot reload에 제약을 주기도 한다.
    

## 훅은 렌더링 컨텍스트에 바인딩 된 상태머신

- 함수형 컴포넌트를 렌더링 할 때마다 React는 내부적으로 현재 이 컴포넌트를 위해 관리하고 있는 훅들의 목록을 기억
- 같은 컴포넌트가 재렌더링 될 때 호출되는 훅 순서와 개수가 바뀌지 않는 이상 저번 렌더링에서의 훅 상태를 꺼내와서, 이번 렌더링에도 상태를 이어 붙임
- 예를 들어

```tsx
const [count, setCount] = useState(0)
```

위 코드를 호출 시, 이 훅을 위한 노드를 만들고 그 노드 안에 `currentState = 0` 과 `updateQueue = []` 를 저장한다.
이후 `setCount(1)` 이 호출되면 updateQueue에 액션이 쌓이고, 다음 렌더링 때 React 가 이 큐를 순회하며 `currentState` 를 새 값으로 전이 시켜준다.

## 훅의 규칙

1. 최상위에서만 호출하기 (Only call Hooks at the top level)
- 훅을 루프, 조건문, 중첩 함수 안에서 호출 시 매 렌더마다 호출 순서가 달라져 React가 내부 상태 슬롯을 잘못 매핑 할 수 있다.
항상 컴포넌트 함수나 커스텀 훅 최상위 스코프에서만 호출해야 한다.
1. React 함수 컴포넌트 또는 커스텀 훅에서만 호출하기 (Only call Hooks from React functions)
- 일반 JS 함수나 클래스 컴포넌트 등 React 렌더링 외부에서 훅 호출 시 훅의 라이프사이클 관리가 깨진다. 
오직 함수형 컴포넌트나 `use`로 시작하는 커스텀 훅 내부에서만 사용해야 한다.

## **훅의 작동 원리 (렌더링과 메모리 흐름 기반)**

- **렌더 중인 컴포넌트 추적**
    
    React는 현재 렌더링 중인 함수 컴포넌트를 가리키는 내부 포인터를 유지한다. 훅(`useState`, `useEffect` 등)을 호출하면 이 포인터를 통해 “어떤 컴포넌트”에 대한 호출인지 알 수 있다.
    
- **메모리 셀 리스트**
    
    각 컴포넌트 인스턴스(정확히는 Fiber)에, 훅 호출 순서대로 연결된 **메모리 셀(memory cell)** 목록이 있다.
    
    1. 첫 렌더링 시: `useState`를 만나면 새로운 셀을 생성해 초기 상태를 저장
    2. 이후 훅이 호출될 때마다 “다음 셀”로 포인터를 옮겨 가며 데이터를 읽거나 초기화
    3. 렌더가 끝나면 이 훅 목록을 그대로 보존해 두고, 다음 렌더 시 같은 순서로 재사용

## **클로저 이슈 & stale value 문제 완전 분석**

### Stale Value란 무엇인가?

- 직역하면 “오래된 값”이라는 뜻대로, 컴포넌트가 렌더링된 이후로 더 이상 업데이트되지 않은 옛날 값(상태나 프롭스)을 가리킨다.

```tsx
import React, { useState, useEffect } from 'react';

function DelayedLogger() {
  const [count, setCount] = useState(0);

  // count 값이 바뀔 때마다 이 useEffect가 실행
  useEffect(() => {
    setTimeout(() => {
      console.log('[1초 뒤] count =', count);
    }, 1000);
  }, [count]);

  return (
    <div>
      <p>count: {count}</p>
      <button onClick={() => setCount(c => c + 1)}>+1</button>
    </div>
  );
}

```

1. 처음 렌더링 → `count = 0`
2. “+1”을 눌러서 `count`가 `1`이 되면, `useEffect`가 실행되고, 내부의 `setTimeout`이 등록된다. 이때 클로저가 “`count`는 1이다”라고 기억한다.
3. 만약 1초 이내에 다시 “+1”을 눌러서 `count`를 `2`로 올린다.
4. 1초가 지나면 첫 번째 타이머 콜백이 동작하면서 “`count = 1`”을 찍는다.
하지만 실제 컴포넌트의 현재 `count`는 `2`다. 
즉, 타이머 콜백 안의 `count`는 “이미 오래전 렌더링 시점(1)이 캡처된 stale value”가 된 것이다.

이처럼 “타이밍 차이”나 “비동기 처리”가 있으면, 함수가 참조한 값이 최신 상태를 반영하지 못할 수 있는데, 그것을 **Stale Value**라고 부른다.

### Stale Closure란 무엇인가?

- JS의 클로저 특성상, 함수가 정의될 때의 스코프(변수들)를 기억하게 되는데, React에서는 이 함수가 렌더 시점에 만들어지면서 특정 시점의 상태나 변수를 캡처한다.
- 만약 그 함수(클로저)를 나중에 실행했을 때, 함수가 참조하고 있는 상태 값이 이미 더 이상 최신이 아니라면, 그 함수는 “Stale Closure” 상태이다.

```tsx
import React, { useState, useEffect } from 'react';

function TimerComponent() {
  const [seconds, setSeconds] = useState(0);

  // 1. 렌더 시마다 새로운 함수가 만들어지고,
  //    그 함수는 그 순간의 'seconds'를 캡처한다.
  const handleAlert = () => {
    alert('현재 seconds 값은 ' + seconds + ' 입니다.');
  };

  useEffect(() => {
    // 컴포넌트가 마운트될 때 한 번만 등록
    document.addEventListener('click', handleAlert);
    return () => {
      document.removeEventListener('click', handleAlert);
    };
  }, []); // 의존성 배열이 비어 있어서 딱 한 번만 등록됨

  // 2. 타이머를 돌려서 1초마다 seconds를 1씩 올린다.
  useEffect(() => {
    const id = setInterval(() => {
      setSeconds(s => s + 1);
    }, 1000);
    return () => clearInterval(id);
  }, []);

  return <div>seconds: {seconds}</div>;
}
```

1. 첫 렌더(초기 `seconds = 0`) 시점에 `handleAlert` 함수가 생성되고, 그 안에서 `seconds`는 `0`으로 캡처된다.
2. `useEffect`의 빈 배열(`[]`) 덕분에, 이 `handleAlert`는 컴포넌트가 마운트된 직후에 **단 한 번**만 `document`의 클릭 이벤트 리스너로 등록된다.
3. 이후 1초마다 `seconds` 값은 1, 2, 3… 계속 변해 가지만, `handleAlert`는 **처음 생성되었을 때 캡처한 `seconds = 0`*를 기억하고 있다.
4. 사용자가 클릭을 하면, 알림창에는 항상 “현재 seconds 값은 0 입니다.”가 뜨게 된다.
    - 클릭하는 순간의 **실제 최신 `seconds`*가 아니라, 클로저가 생성된 시점의 “Stale Value(=0)”를 보여 주는 것이다.

즉, 이벤트 핸들러나 타이머 콜백처럼 “한 번 등록해 놓고 나중에 실행되는 함수”들이, 내부에서 사용하는 상태나 변수를 처음 캡처된 시점 그대로 기억하다 보니, 실제 상태가 바뀌어도 업데이트된 값을 반영하지 못할 때 이걸 “Stale Closure 문제”라고 부른다.

### Stale Value/Closure를 방지하는 대표적인 패턴

1. 의존성 배열(Dependency Array)을 정확히 명시하기
    - 위 예시처럼 `useEffect`에서 비어 있는 배열 `[]`로 한 번만 함수를 등록하면, 그 함수가 캡처한 상태는 전혀 바뀌지 않는다.
    - 만약 클릭할 때마다 최신 `seconds`를 보고 싶다면, `useEffect` 안에서 이벤트 리스너를 등록할 때 `seconds`를 의존성으로 넣어서, `seconds`가 바뀔 때마다 새 클로저를 등록하도록 한다.
2. Ref를 활용해 최신 값을 보관하기
    - 상태를 직접 클로저에 캡처시키는 대신, `useRef`로 “항상 최신 값을 담아두는 그릇”을 만들고, 클로저 안에서는 상태가 아니라 `ref.current`를 참조하게 하는 방식
3. useCallback/useMemo로 함수를 메모이제이션하기
    - 외부 의존성을 최소화하여 클로저가 캡처하는 값의 범위를 좁혀 버리는 방법
    - `useCallback`쓰면, 의존성 배열에 명시된 값이 바뀔 때에만 새로 클로저가 만들어집니다

## useRef 심화 활용

`useRef`는 흔히 DOM 노드 접근 용도로만 알려져 있지만, 그 외에도 “렌더 간에 값을 보존하면서도 렌더를 트리거하지 않는 저장소(storage)”로서 활용할 수 있다.

1. **렌더 간 값 보존**
    - `useRef`가 반환하는 객체(`{ current: 초기값 }`)는 컴포넌트가 **언마운트될 때까지** 동일한 참조를 유지한다.
    - 이 점을 활용해, “이전 렌더의 상태값”이나 “컴포넌트가 재렌더되더라도 초기화되지 않아야 할 값”을 저장할 수 있다.
    - 예를 들어 아래 코드처럼, 현재 `count`가 변경될 때마다 이전 값을 `prevCount.current`에 기록하면, 다음 렌더에서 손쉽게 ‘직전 값’을 꺼내 쓸 수 있다.
        
        ```tsx
        function Counter({ count }) {
          const prevCount = useRef(count);
        
          useEffect(() => {
            prevCount.current = count;
          }, [count]);
        
          return (
            <div>
              <p>현재 카운트: {count}</p>
              <p>직전 카운트: {prevCount.current}</p>
            </div>
          );
        }
        ```
        
2. **타이머 ID, 외부 객체 저장** 
    - `setTimeout`이나 `setInterval`의 ID, WebSocket 인스턴스, 외부 라이브러리에서 생성된 객체(instance)를 `state` 대신 `ref.current`에 보관할 수 있다.
    - 이렇게 하면 “타이머 ID가 바뀐다고 매 렌더마다 컴포넌트가 다시 렌더된다” 같은 불필요한 렌더링을 피할 수 있다.
        
        ```tsx
        function IntervalLogger() {
          const countRef = useRef(0);
          const intervalIdRef = useRef(null);
        
          useEffect(() => {
            intervalIdRef.current = setInterval(() => {
              countRef.current += 1;
              console.log('카운터:', countRef.current);
            }, 1000);
            return () => clearInterval(intervalIdRef.current);
          }, []);
        
          return <p>콘솔 확인</p>;
        }
        ```
        

`countRef.current`는 **변경돼도 컴포넌트를 리렌더링하지 않으면서** 값을 누적해 주기 때문에, 단순 로그 등 “화면 UI 변화가 필요 없는” 내부 로직 상태를 추적할 때 유용하다.

## useLayoutEffect vs useEffect

**실행 시점 차이**

- `useEffect`
    - **커밋(commit) 단계 후**: 브라우저가 레이아웃을 계산하고 화면을 페인트한 뒤에 실행
    - 따라서 UI가 먼저 시각적으로 업데이트되고, 그다음에 비동기 작업, 구독, 타이머 등록 등을 안전하게 처리
- `useLayoutEffect`
    - **DOM 업데이트 직후, 브라우저가 페인트하기 전에** 실행
    - React가 “가상 DOM(diffing) → 실 DOM 업데이트”를 끝낸 뒤, 브라우저가 화면에 그리기하기 바로 직전에 동기적으로 실행됨
    - 이 시점에서는 브라우저가 아직 화면을 렌더링하지 않았기 때문에, DOM을 읽거나 치수를 계산하거나, 레이아웃을 조정해도 유저에게 깜빡임이 일어나지 않음

일반적인 사이드 이펙트(ex. 데이터 페칭, 타이머 등록 등): 브라우저 페인트 이후 실행되어도 문제가 없다면, 오히려 성능과 유연성을 위해 useEffect를 쓰는 것이 좋다.

레이아웃 계산이나 즉시 DOM 변형이 필요한 시각적 작업(ex. 툴팁 위치 조정, 스크롤 보정, 포커싱 제어 등): 브라우저가 화면을 그리기 전에 미리 값을 계산·적용해야 깜빡임이 발생하지 않으므로, useLayoutEffect를 사용해야 한다.

## 면접 예상 질문 리스트 & 답변

- useState를 조건문에서 쓰면 안되는 이유?
    - 훅은 내부적으로 “호출 순서(call order)”에 따라 컴포넌트의 각 훅 호출을 고유한 슬롯(state slot)에 매핑해 두는데, 조건문 안에서 `useState`를 호출하면 렌더링마다 이 순서가 달라질 수 있어 문제가 발생함.
    - 이처럼 훅은 **반드시 동일한 순서**로 호출되어야만, React가 렌더링 간에 “어떤 훅이 어떤 상태를 갖고 있던 슬롯(slot)인지”를 올바르게 인식할 수 있음. 따라서 조건문, 루프, 중첩 함수 안에서 훅을 호출하면 호출 순서가 바뀔 위험이 있어 훅 매핑이 깨지는 것이고, 이를 방지하기 위해 공식 문서에서는 다음과 같이 명시함.
        - “Don’t call Hooks inside loops, conditions, or nested functions. Instead, always use Hooks at the top level of your React function…”