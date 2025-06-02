# 리액트 훅이란?

이름 앞에 use로 시작하는 함수를 훅이라고 부름.

사용되는 대표 프리빌트 훅

useState: 컴포넌트 내부의 상태를 관리.
useEffect: 컴포넌트의 렌더링 이후에 어떤 이벤트를 트리거시킬때(데이터 가져오기, 구독 설정, DOM 직접 조작 등) 사용.
useContext: props드릴링을 피하기 위해서, 컴포넌트간에 변수 공유를 쉽게 하려고 사용.
useRef: 특정 DOM 요소에 접근하거나 렌더링 간에 변경되지 않는 값을 만들기 위해서 사용.
useCallback, useMemo: 렌더링 성능 최적화를 위해 사용.

# 리액트 훅을 왜 만들어졌나

기존에는 클래스형 컴포넌트에서만 this.state, setState(), componentDidMount와 같은 방식으로 상태 관리나 생명주기 메서드를 사용할 수 있었다고함.
함수형 컴포넌트에서도 이를 구현하기 위해서 도입함.

-> 함수형 컴포넌트는 왜 쓰는가?
--> this 때문에 복잡한 코드 문제 해결
--> 동일 인풋에 대한 동일 아웃풋이라는 명료함이 있기 때문에 테스트도 편하고, 에러 예측이 편리함.

그리고 훅이 동기적인 순서가 보장되어 있어서 코드 작성을 동기적으로 할 수 있어서 매우 좋음.
-> 저는 fetch 컴포넌트를 모두 커스텀훅으로 만들어서 쓰는데 이 점 때문에 디펜던시를 걸 수 있어서 좋음.

# 커스텀 훅

재사용 가능한 로직을 컴포넌트화 시키기 위해 이름 앞에 use를 붙여서 사용하는 재사용 로직 코드 뭉치

# 렌더링 최적화 훅 비교

기본적으로 세가지 hook은 dependency 관계에 있는 변수 값이 변화하는 것에 연동되어 실행되는데, 구체적인 차이가 무엇인지 비교해보기

## useEffect

```
useEffect(
  () => {
    const subscription = props.source.subscribe();
    return () => {
      subscription.unsubscribe();
    };
  },
  [props.source],
);
```

class component의 life cycle 함수(side effect)를 function component에도 동일하게 사용가능  
최초 컴포넌트 마운트 되는 경우, 컴포넌트 내 레이아웃 배치와 랜더링이 완료된 후에 실행

두 번째 인자(배열)의 요소로 지정하면, 해당 요소 값의 업데이트 되는 경우에만 실행

그러나, state나 props를 dependency로 지정하면, 불필요한 랜더링 발생 가능

## useCallback

```
const memoizedCallback = useCallback(
  () => {
    doSomething(a, b);
  },
  [a, b],
);
```

memoization 된 콜백(함수) 자체를 반환

useCallback(fn, deps)은 useMemo(() => fn, deps)은 동일

의존성이 변경되는 경우, 이전에 기억하고 있던 함수 자체와 비교해서 다른 경우에만 리랜더

## useMemo

```
const memoizedValue = useMemo(() => computeExpensiveValue(a, b), [a, b]);
```

memoization 된 값을 반환

의존성이 변경되는 경우, 이전에 기억하고 있던 리턴 값과 비교해서 다른 경우에만 리랜더

useMemo에서 전달된 함수는 랜더링 중에 실행되므로, 랜더링 중에서 실행하지 않는 함수는 useEffect를 사용할 것
useRef와의 차이는, useRef는 DOM element의 특정 속성 값을 기억한다면, useMemo는 특정 함수의 리턴 값을 기억하는 것

# useTransition

for concurrent rendering

디바운싱으로 지나친 렌더링을 막는 방식은 그 설정값의 설정이 애매함. 유저의 컴퓨터 성능에 따라 불필요한 디바운싱을 막을 수도 있고, 필요한 디바운싱을 막지 못할 수도 있음. 이를 해결하기 위해 useTransition이 나왔다.

https://velog.io/@ktthee/React-18-%EC%97%90-%EC%B6%94%EA%B0%80%EB%90%9C-useDeferredValue-%EB%A5%BC-%EC%8D%A8-%EB%B3%B4%EC%9E%90

```jsx
import './App.css';
import { useState, useTransition } from 'react'; // useTransition import 하기

let a = new Array(10000).fill(0);

function App() {
  let [name, setName] = useState(0);
  let [isPending, startTransition] = useTransition(); // 불러오기

  return (
    <div className="App">
      <input
        onChange={(e) => {
          startTransition(() => {
            // 성능 저하 일으키는 state 변경함수 감싸주기
            setName(e.target.value);
          });
        }}
      ></input>
      {a.map(() => {
        return <div>{name}</div>;
      })}
    </div>
  );
}

export default App;
```

https://velog.io/@brgndy/%EB%A6%AC%EC%95%A1%ED%8A%B8-useTransition-useDeferredValue-%EC%A0%95%EB%A6%AC

useDeferredValue vs useTransition
useDeferredValue와 useTransition hook은 상태변화의 우선순위를 낮게 하는 hook이다. 그렇다면 두 hook의 차이점은 무엇일까?

먼저 useDeferredValue는 값을 래핑해서 사용한다. 아래 예시 코드를 보면 count2라는 값을 래핑 해서 count2 값이 바뀌어도 deferredValue는 다른 상태변화가 전부 일어난 후에 바뀌게 된다. 즉, useDeferredValue는 값을 래핑 해서 값의 변화의 우선순위를 낮추도록 한다.

```ts
const deferredValue = useDeferredValue(count2);
```

다음은 useTransition hook이다. useTransition은 useDeferredValue와 다르게 값이 아닌, 함수를 래핑한다. 아래 예시 코드를 보면 () => { setText(e.target.value) } 함수를 래핑하고 있다. 래핑 된 함수의 우선순위를 낮춰서 다른 상태 변경이 전부 일어난 후 해당 함수를 실행하게 된다.

```ts
startTransition(() => {
  setText(e.target.value);
});
```

마치 useMemo는 값을 메모이제이션하고 useCallback은 함수를 메모이제이션 하는 것과 같이 값이냐 함수냐로 구분할 수 있다.

결론

- useDeferredValue는 상태 값에 우선순위를 낮추는 hook

- useTransition은 상태 변화를 일으키는 함수의 우선순위를 낮추는 hook이다

# useLayoutEffect

## useLayoutEffect란?

useLayoutEffect는 useEffect와 비슷하지만, 렌더링 이후에 발생하는 것이 아니라, 렌더링 이전에 발생한다. 즉, useEffect와 비슷하지만, useEffect보다 먼저 발생한다.

## useLayoutEffect와 useEffect의 차이점

useLayoutEffect와 useEffect의 차이점은 렌더링 이전에 발생하는지, 이후에 발생하는지의 차이이다. useLayoutEffect는 렌더링 이전에 발생하고, useEffect는 렌더링 이후에 발생한다.

## useLayoutEffect의 사용법

useLayoutEffect는 useEffect와 사용법이 같다. 다만, useLayoutEffect는 렌더링 이전에 발생한다는 것만 다르다.

https://www.howdy-mj.me/react/useEffect-and-useLayoutEffect

# 소소한 정보들

## useState 비동기처리이다.

하나의 페이지나 컴포넌트 내에도 수많은 상태값이 존재한다. 만약 이 상태 하나하나가 바뀔 때마다 화면을 리렌더링 한다면 문제가 생길수도 있다.

때문에 리액트는 성능의 향상을 위해서 setState를 연속 호출하면 배치 처리하여 한 번에 렌더링하도록 하였다. 아무리 많은 setState가 연속적으로 사용되었어도 배치 처리에 의해서 한 번의 렌더링으로 최신 상태를 유지하는 것이다.

배치란 React가 너 나은 성능을 위해 여러개의 state 업데이트를 하나의 리렌더링으로 묶는 것을 의미한다.
React는 16ms 동안 변경된 상태 값들을 하나로 묶는다. (16ms 단위로 배치를 진행한다.)

참조: https://velog.io/@alstnsrl98/useState%EB%8A%94-%EB%8F%99%EA%B8%B0-%EB%B9%84%EB%8F%99%EA%B8%B0-%EB%8F%99%EA%B8%B0%EC%A0%81-%EC%B2%98%EB%A6%AC

## useRef잘 쓰기.

- useRef를 통해 생성한 변수는 상태값이 변해도 즉시 컴포넌트를 리랜더링시키지 않는다

ref변수를 여러개 생성하기

```jsx
[리팩토링 전]
const firstTab = useRef();
const secondTab = useRef();
const thirdTab = useRef();

.
.
.

<TabContents ref={firstTab}>...</TabContents>
<TabContents ref={secondTab}>...</TabContents>
<TabContents ref={thirdTab}>...</TabContents>

[리팩토링 후]
const tabRef = useRef([]);

.
.
.

<TabContents ref={el => (tabRef.current[0] = el)}>...</TabContents>
<TabContents ref={el => (tabRef.current[1] = el)}>...</TabContents>
<TabContents ref={el => (tabRef.current[2] = el)}>...</TabContents>
```

참고 (https://velog.io/@dosilv/ReactWeb-API-useRef-scrollIntoView)

## useState사용 시 주의할 점

처음 내 생각: `currentHomeTabIndex`를 useMemo로 선언했으니 의존성에 걸린 변수들이 변하면 다시 계산되어 useState의 초기값으로 들어가겠지?
실제: useMemo로 선언한 변수가 변해도 useState의 초기값은 변하지 않는다.
왜? -> 페이지간 이동이 없기에 페이지가 다시 그려지지 않아서 사실상 useState의 초깃값이 유지되어 에러 시 디폴트 값으로 전환이 되는 로직을 타지 않기 떄문

```ts
const currentHomeTabIndex = useMemo(() => (gte(currentTabIndex, tabLabels.length) ? 0 : currentTabIndex), [currentTabIndex, tabLabels.length]);

// 한번 설정된 초깃값이 페이지변환이 이루어지지 않는다면 그대로 유지된다.
const [tabValue, setTabValue] = useState(currentHomeTabIndex);
```

따라서 useEffect를 사용해서 초기값을 설정(디폴트 로직이 들어간)해줘야 한다.

```ts
const [tabValue, setTabValue] = useState(tabLabels.map((item) => item.value).includes(currentTabIndex) ? currentTabIndex : 0);

useEffect(() => {
  if (!tabLabels.map((item) => item.value).includes(currentTabIndex)) {
    setTabValue(0);
  }
}, [currentTabIndex, tabLabels]);
```
