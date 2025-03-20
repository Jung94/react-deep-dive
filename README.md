# React Deep Dive
리액트 렌더링과 Fiber 아키텍처 분석

## 개요
이 글의 목적은 리액트의 전반적인 동작 방식과 주요 개념인 Fiber 아키텍처에 대한 깊이 있는 이해입니다.

이전에 공부하고 정리했던 리액트 렌더링 과정과 Fiber 아키텍처를 다시 상기하고 재정리합니다. 그리고 바닐라 자바스크립트로 리액트를 직접 구현해 보고자 합니다.

## 목차
[리액트 렌더링은 어떻게 일어나는가?](#리액트-렌더링은-어떻게-일어나는가)  
[Fiber Architecture](#fiber-architecture)  
[JSX](#jsx)  
[Batching](#batching)  
[References](#references)  

## 리액트 렌더링은 어떻게 일어나는가?
리액트에서의 렌더링은 브라우저 렌더링과 다릅니다. 브라우저 렌더링이란 브라우저가 HTML, CSS, JavaScript를 해석하여 우리가 보는 화면에 실제로 나타내는 과정을 말합니다. 단순히 파일을 읽는 것만 아니라 웹 페이지의 구조를 분석하고 스타일과 스크립트를 계산하며, 사용자에게 보이는 픽셀 단위의 화면을 만들어내는 아주 복잡합 과정을 의미합니다.

반면, 리액트에서의 렌더링이란 모든 컴포넌트가 props와 state를 기반으로 어떻게 UI를 구성하고 어떤 DOM 결과를 브라우저에게 제공할 것인지 계산하는 일련의 과정을 의미합니다.
즉 브라우저 렌더링은 화면에 무언가 바로 나타나는 것이지만, 리액트 렌더링은 화면에 무언가 나타나기 전 일련의 과정을 일컫습니다. 여기서 주목해야 할 점은 리액트에서의 렌더링은 DOM 업데이트가 아니라는 점입니다. 즉, 리액트 렌더링은 실질적인 화면 업데이트가 아닙니다.

### 리액트 렌더링의 전체적인 흐름
React 팀은 리액트의 렌더링 과정을 크게 두 단계로 나눴습니다.
1. Render Phase (렌더 단계): 컴포넌트를 렌더링하고 변경 사항을 계산하는 모든 과정이 이루어지는 단계
2. Commit Phase (커밋 단계): 변경 사항을 실제 DOM에 적용하는 단계

### 언제 렌더링이 일어날까?
1. 최초 렌더링: 사용자가 처음 애플리케이션에 진입하는 경우
2. 리렌더링: 최초 렌더링 이후 발생하는 모든 렌더링
	- `useState`의 두 번째 배열 요소인 `setter`가 샐행되는 경우
 	- `useReducer`의 두 번째 배열 요소인 `dispatch`가 실행되는 경우
  	- `props`가 변경되는 경우
  		- 컴포넌트의 `key prop`이 변경되는 경우
    			: 리렌더링이 발생하면 current 트리와 workInProgress 트리 사이에서 같은 컴포넌트인지 구별하는 값이 `key`입니다. 만약 `key`가 없다면 단순히 파이버 내부의 `sibling` 인덱스만을 기준으로 판단하게 됩니다.
	- 부모 컴포넌트가 렌더링 되는 경우

### 예제: 상태 업데이트로 인해 렌더링이 발생하는 경우
```javascript
function Counter() {
  const [count, setCount] = React.useState(0);

  return (
    <div>
      <h1 className="title">Hello, React</h1>
      <h2>Count: {count}</h2>
      <button onClick={() => setCount(count + 1)}>Increment</button>
    </div>
  );
}
```
1) 초기 렌더링(Mounting) 과정
2) 사용자가 버튼 클릭 -> `setCount(count + 1)` 실행
3) 리액트가 상태 업데이트를 감지하고, Render Phase 실행
4) Fiber 트리를 생성하고 Reconciliation 수행
5) 변경된 Fiber 노드를 기반으로 Commit Phase에서 실제 DOM 업데이트

### Step 1: 초기 렌더링(Mounting) - Fiber 트리 생성
> JSX -> _jsx() (또는 _jsxs()) -> ReactElement -> Fiber 노드 생성

리액트는 JSX를 만나면 _jsx() (또는 _jsxs())를 호출해 ReactElement 객체를 만듭니다.
- JSX 변환 과정
```javascript
const element = <h1 className="title">Hello, React</h1>;
```
⬇ Babel 변환 (React 17+)
```javascript
import { jsx as _jsx } from "react/jsx-runtime";

const element = _jsx("h1", { className: "title", children: "Hello, React" });
```
⬇ ReactElement 변환
```javascript
{
  type: "h1",
  props: {
    className: "title",
    children: "Hello"
  },
  key: undefined,
  ref: null,
  $$typeof: Symbol(react.element)
}
```
이렇게 생성된 ReactElement는 Fiber 아키텍처에 의해 Fiber 노드로 변환됩니다. Fiber 노드는 트리 구조를 가지며, 각 컴포넌트와 DOM 요소를 나타냅니다.
```less
Counter (Function Component)
│
├── div (HTML Element)
│   ├── h1 (HTML Element)
│   ├── h2 (HTML Element)
│   └── button (HTML Element)
```
이렇게 초기 Fiber 트리가 생성되면, Commit Phase에서 실제 DOM에 반영됩니다.

### Step 2: 상태 업데이트(Updating) - Fiber 트리 비교 (Reconciliation)
> 버튼 클릭으로 setCount(count + 1) 실행 → 상태 업데이트 발생 (Trigger)

리액트는 Render Phase를 다시 실행하며 새로운 Fiber 트리를 만듭니다. 다음은 Render Phase에서 수행하는 작업 순서입니다.
1. 새로운 Virtual DOM(Fiber 트리) 생성
2. 이전 Fiber 트리와 새로운 Fiber 트리 비교 (Reconciliation)
3. 변경된 Fiber 노드를 Mark(표시)

- 이전 상태의 Fiber 트리
```less
Counter
│
├── div
│   ├── h1
│   ├── h2 (Count: 0)
│   └── button
```
- 새로운 상태의 Fiber 트리 (`setCount(0 + 1)`)
```less
Counter
│
├── div
│   ├── h1
│   ├── h2 (Count: 1) <- 변경됨 🔥
│   └── button
```

리액트는 새로운 Fiber 트리를 만들고, 이전 트리와 비교하여 변경된 노드(`h2`)를 감지합니다. 이 과정을 Reconciliation이라고 합니다.

> [Reconciliation (재조정)](#등장-배경)  
> -> 리액트는 Fiber 트리 비교(Diffing)를 수행하여 다음과 같은 동작을 합니다.
> - 같은 타입의 요소(`div`, `h1`, `button`)는 유지
> - `h2`의 텍스트가 변경됨 -> 새로운 Fiber 노드로 교체
> - 변경된 노드만 "Update"로 표시하고, Commit Phase에서 실제 DOM에 적용

### Step 3: Commit Phase - 변경 사항 적용
Commit Phase에서는 Reconciliation 과정에서 변경된 Fiber 노드를 실제 DOM에 반영합니다.
- `h2` 요소의 textContent를 Count: 1로 업데이트
- 화면을 다시 그림(Repaint)

리액트의 useEffect() 같은 훅은 이 시점에서 실행됩니다.

### 렌더링 과정 요약
`setCount(count + 1)` 실행 시, React의 내부 동작은 다음과 같습니다.
1. Render Phase (렌더 단계)
	- JSX -> `React.createElement()` -> `ReactElement`
 	- ReactElement → Fiber 트리 변환
  	- 기존 Fiber 트리와 새로운 Fiber 트리 비교 (Reconciliation)
   	- 변경된 노드 감지 및 "Update"로 표시
2. Commit Phase (커밋 단계)
	- 번경된 노드를 실제 DOM에 반영
 	- `useEffect()` 실행

* * *

## Fiber Architecture
### 등장 배경
Fiber 아키텍처의 등장 배경을 설명하기 위해서는 Reconciliation(재조정)에 대한 이해가 선행되어야 합니다.

Reconciliation(재조정)이란 컴포넌트의 상태가 변경되면 기존 가상 DOM 트리와 변경된 가상 DOM 트리를 비교해 어떤 부분을 변경해야 하는지 찾아내기 위한 알고리즘입니다. 

React 15까지는 가상 DOM을 변경할 때 전체 컴포넌트 트리를 한 번에 동기적으로 처리하는 Stack Reconciliation(스택 재조정) 방식을 사용했습니다. 이 방식은 몇 가지 문제를 일으켰습니다.

첫 번째, 렌더링이 시작되면 끝날 때까지 중단할 수 없었습니다. 한 번 가상 DOM을 비교(Reconciliation)하기 시작하면, 브라우저가 다른 작업을 처리할 수가 없었습니다. 이 때문에 화면이 버벅거리거나(freezing) 애니메이션이 끊기는 문제가 발생했습니다.

두 번째, 프레임 단위로 나누어 실행할 수 없었습니다. 브라우저는 초당 60프레임으로 동작하는 게 이상적입니다(16.67ms/프레임). 하지만 기존 리액트는 렌더링을 한 번에 끝내야 했기 때문에 무거운 컴포넌트가 많으면 프레임 드롭(frame drop)이 발생했습니다.

마지막으로, 우선순위 기반 업데이트가 불가능했습니다. 사용자 입력(키보드, 마우스)과 같은 중요한 UI 이벤트가 발생해도 리액트가 모든 렌더링을 마칠 때까지 기다려야 했습니다. 다시 말해 긴급한 업데이트를 먼저 수행하는 것이 불가능했습니다.

위 문제점들을 해결하기 위해 리액트팀은 비동기적이고 효율적인 렌더링을 수행할 수 있도록 새로운 아키텍처인 Fiber를 설계했습니다.

### Fiber 아키텍처란?
Fiber는 리액트 팀이 기존의 스택 재조정 알고리즘을 개선하여 더 부드러운 렌더링과 성능 최적화를 목적으로 도입한 새로운 아키텍처입니다. 2017년 React 16에서 처음 도입되었으며, 기존 가상 DOM과 다르게 작업을 작은 단위(Fiber Node)로 나누어 우선순위를 조정하거나 작업을 중단하고 더 중요한 작업을 먼저 처리할 수 있도록 설계되었습니다. 이를 통해 비동기 렌더링, 우선순위 기반 업데이트, 중단 및 재개 기능을 지원합니다.

Fiber는 다음과 같은 작업이 가능합니다.
- 작업을 작은 단위로 분할하고 쪼갠 다음, 우선순위를 매깁니다.
- 이러한 작업을 일시 중지하고 나중에 다시 시작할 수 있습니다.
- 이전에 했던 작업을 다시 재사용하거나 필요하지 않은 경우에는 폐기할 수 있습니다.

이 과정에서 비교 알고리즘(Diffing Algorithm)이 사용되며 성능 최적화를 위해 몇 가지 규칙을 사용합니다.
- 같은 타입의 요소는 그대로 유지
- 다른 타입의 요소는 새로 생성
- key를 활용한 리스트 비교 최적화

중요한 점은 위 모든 작업이 비동기로 일어난다는 점입니다. 사용자의 인터렉션에 따른 동시 다발적인 이벤트와 애니메이션은 다양한 작업을 처리하는 요즘의 웹 애플리케이션에서는 피할 수 없는 문제입니다. 이러한 기존 렌더링 스택의 비효율성을 타파하기 위해 리액트 팀은 기존의 Stack Reconciliation 대신 Fiber라는 개념을 탄생시켰습니다.

### Fiber 렌더링 과정
Fiber 기반의 렌더링은 두 단계, Render 단계와 Commit 단계로 나뉘어 작업이 이루어집니다.
1. Render Phase(렌더 단계)에서는 Reconciliation을 수행하여 새로운 Fiber 트리를 생성합니다. 이 단계는 Concurrency(동시성)를 지원하여 사용자 입력이나 애니메이션 등의 우선순위가 높은 작업을 위한 작업 중지 및 재개가 가능합니다.
	- 렌더링 과정에서 컴포넌트를 실행(render() or return())해 생성한 가상 DOM과 이전 가상 DOM을 비교하는 과정을 거쳐 변경이 필요한 컴포넌트를 체크하는 단계입니다. 여기서 비교하는 것은 크게 세 가지로, type, props, key입니다. 이 세 가지 중 하나라도 변경되었다면 변경이 필요한 컴포넌트로 체크됩니다.
 	- Concurrency(동시성)란 두 개 이상의 테스크를 _동시에 지원_한다는 뜻입니다. 이는 여러 작업에 대해 일시 정지와 재가동을 반복하면서 우선순위를 기반으로 일정 시간 동안 여러 업무를 처리하는 것입니다.
 	- Parallelism(평행성)는 두 개 이상의 테스크를 _동시에 실행_할 수 있는 Concurrency과 비슷해 보이지만 엄연히 다릅니다. Parallelism을 위해서는 물리적으로 여러 개의 스레드를 사용 가능해야 하며, 자바스크립트와 같은 싱글 스레드 환경에서는 Parallelism이 불가능합니다.
  	- 렌더 단계에서 아무런 변경 사항이 감지되지 않았다면 커밋 단계가 생략되면서 DOM 업데이트가 일어나지 않을 수 있습니다.
2. Commit Phase(커밋 단계)에서는 변경된 내용을 실제 DOM에 적용하는 단계입니다. 이 과정은 렌더 단계와 다르게 동기식으로 실행되며 중단할 수 없습니다.

### Fiber 트리
Fiber 트리(Fiber Tree)는 리액트의 렌더링을 관리하는 Singly Linked List(단일 링크드 리스트, 이하 링크드 리스트) 기반 트리 구조입니다. 각 UI 요소는 Fiber 노드로 표현되며, 이를 트리 형태로 구성하여 렌더링을 관리합니다.

- 기존 가상 DOM과의 차이점

| 기존 Virtual DOM (React 15) | Fiber 트리 (React 16+) |
| --- | --- |
| 일반적인 트리 구조 | 링크드 리스트(Linked List) 기반 트리 |
| 재귀 호출로 트리 순회 | 순환(iteration) 방식으로 트리 탐색 |
| 한 번의 업데이트를 모두 수행 | 작업을 나누고 우선순위를 지정 가능 |
| 렌더링 도중 멈출 수 없음 | 중단하고 재개 가능 (비동기 렌더링) |

리액트 팀이 Fiber에서 링크드 리스트를 사용한 이유는 기존의 트리 기반 탐색의 문제점을 보완하기 위해서 입니다. 기존 가상 DOM은 재귀 호출(Recursive Traversal)을 사용해 트리를 탐색했습니다. 구체적인 예로, 트리의 각 노드를 방문할 때마다 자식 노드를 배열로 관리했습니다.

```js
const tree = {
  type: "div",
  children: [
    { type: "h1", children: [] },
    { type: "p", children: [] },
  ],
};
```

재귀 호출은 한 번 호출되면 모든 노드를 방문해야 하기 때문에 렌더링 중단이 불가능합니다. 그리고 재귀 호출 스택이 쌓이면서 메모리 사용량이 증가하게 됩니다.

리액트 팀은 위와 같은 문제점을 해결하기 위해 재귀 호출 대신 반복문(Iteration) 방식을 사용하는 링크드 리스트 형태로 변환했습니다. 체인 형태의 링크드 리스트는 단순히 깊이 우선으로 탐색하는 것이 아닌 각 노드의 child, sibling, return 포인터를 사용하여 child가 있으면 child, child가 없으면 sibling, sibling이 없으면 return... 순으로 다음 Fiber로 이동합니다. 
```js
const fiberNode = {
  type: "div",
  child: h1Fiber,     // 첫 번째 자식
  sibling: pFiber,    // 형제 노드
  return: parentFiber, // 부모 노드
  ...
};
```

각 Fiber는 다음으로 처리해야 할 Fiber를 가리키고 있기 때문에, 아무리 작업이 길어서 중간에 중단되어도 현재 작업 중인 Fiber만 알고 있다면 다시 돌아와 작업을 이어갈 수 있습니다. 그리고 각 Fiber는 Reconciliation 작업 중 변경 사항에 대한 정보(effect)를 가지고 있으며, 이를 DOM에 즉시 반영하지 않고 렌더 단계를 마친 후 커밋 단계에서 한 번에 반영되기 때문에 실제 화면에는 영향을 미치지 않습니다.


요약하자면, 링크드 리스트 기반 트리의 장점은 다음과 같습니다. 
- DFS 탐색 최적화: `child -> sibling -> return` 형태로 순회 가능
- 렌더링 중단 & 재개 가능 → 트리 순회 중 언제든 작업을 멈추고 재개 가능
- 메모리 최적화 → 재귀 호출을 제거하여 스택 오버플로우 방지

<br/>
리액트는 Double Buffering(더블 버퍼링) 기법을 사용해 UI 업데이트 성능을 최적화합니다. 이를 위해 두 개의 Fiber 트리를 유지합니다.
- `current`: 현재 화면에 렌더링된 Fiber 트리
- `workInProgress`: 새롭게 렌더링될 Fiber 트리

다음은 current 트리 구조의 예시입니다.
```js
const App = () => {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>Click {count}</button>;
};
```
<br/>
아래는 위 코드에서 `count = 0`일 때의 current Fiber 트리 구조입니다.

```less
current Fiber Tree
  ├── App (FunctionComponent)
  │   ├── button (HostComponent)
  │       └── "Click 0" (HostText)
```

만약 버튼을 클릭하면 새로운 상태(`count = 1`)로 업데이트가 발생합니다. 그러면 `workInProgress`가 생성되면서 새로운 렌더링 과정이 시작됩니다.
<br/>
workInProgress는 업데이트가 발생했을 때 생성되는 새로운 Fiber 트리로, 작업 중인 상태이자 화면까지 반영되지 않은 Fiber 트리를 의미합니다. 리액트는 현재 트리(current)를 직접 수정하지 않고, current를 복사하여 workInProgress를 새롭게 만들어 변경 사항을 적용합니다. 이후 커밋 단계에서 workInProgress를 새로운 current로 교체합니다.

```less
current Fiber Tree
  ├── App (FunctionComponent)
  │   ├── button (HostComponent)
  │       └── "Click 0" (HostText)

↓ 업데이트 발생 (setCount(1))

workInProgress Fiber Tree
  ├── App (FunctionComponent)
  │   ├── button (HostComponent)
  │       └── "Click 1" (HostText)
```

### Fiber 노드
Fiber 노드(Fiber Node)는 리액트의 렌더링을 관리하는 핵심 단위로, 실제 DOM 노드나 컴포넌트를 나타내는 일반 자바스크립트 객체이며, 아래와 같은 속성을 가집니다.

```js
export type Fiber = {
  // 🔹 기본 정보
  tag: WorkTag;       // 노드 유형 (FunctionComponent, HostComponent 등)
  key: null | string; // key 값 (리스트 최적화)
  elementType: any;   // JSX에서 사용된 컴포넌트 타입
  type: any;          // 실제 렌더링될 타입 (예: div, function component)
  stateNode: any;     // 실제 DOM 노드 or 클래스 컴포넌트 인스턴스

  // 🔹 Fiber 트리 구조 (링크드 리스트 기반)
  child: Fiber | null;     // 첫 번째 자식 (First Child)
  sibling: Fiber | null;   // 형제 노드 (Sibling)
  return: Fiber | null;    // 부모 노드 (Parent)

  // 🔹 이전 렌더 트리와 연결 (더블 버퍼링)
  alternate: Fiber | null; // 이전 Fiber 노드 (이전 상태 저장)

  // 🔹 업데이트 관련 속성
  pendingProps: any; // 업데이트 예정 props
  memoizedProps: any; // 현재 렌더링된 props
  memoizedState: any; // 현재 렌더링된 상태값
  
  updateQueue: any; // 상태 업데이트 큐 (useState, useReducer 등)
  dependencies: Dependencies | null; // 의존성 (Context API, Suspense 등) 

  // 🔹 우선순위 및 스케줄링
  lanes: Lanes;     // 현재 노드의 우선순위 (Concurrent Mode)
  childLanes: Lanes; // 자식 노드의 우선순위 (우선순위 전달)

  // 🔹 변경 사항 및 효과
  flags: Flags; // 변경 사항 (예: Placement, Update, Deletion 등)
  subtreeFlags: Flags; // 하위 트리의 변경 사항 (최적화 목적)
  deletions: Array<Fiber> | null; // 삭제될 노드 목록

  // 🔹 모드 설정 (Concurrent Mode 관련)
  mode: TypeOfMode; // StrictMode, Concurrent Mode 여부
  
  // 🔹 리스트에서 사용되는 index (Fragment 등에서 사용)
  index: number; // 형제 노드에서의 위치
};
```

### Fiber를 활용한 React 기능들
1. Concurrent Rendering (동시성 렌더링)
	- `useTransition`, `startTransition`과 같은 API를 통해 비동기 렌더링이 가능합니다.
 	- 사용자 입력과 무거운 렌더링 작업을 동시에 처리할 수 있습니다.
2. Suspense & Lazy Loading
	- `React.Suspense`를 활용하여 데이터를 가져오면서도 UI가 멈추지 않도록 최적화합니다.
 	- 코드 스플리팅을 통한 `React.lazy()` 사용이 가능합니다.

### 결론: Fiber는 무엇을 해결했나?
1. 비동기 렌더링 지원을 통해 긴 렌더링 중에도 중단 및 재개가 가능합니다.
2. 우선순위 기반 렌더링을 통해 중요 업데이트를 먼저 수행 가능합니다.
3. 더블 버퍼링으로 이전 상태를 저장하고 최적화합니다.
4. Concurrent Mode를 지원하여 더 부드러운 사용자 경험을 제공합니다.
5. Suspense, Lazy Loading 등 최신 리액트 기능을 가능하게 했습니다.

* * *

## JSX
JSX는 JavaScript and Xml의 약자로, 자바스크립트 코드 내부에서 표현하기 까다로웠던 HTML, XML과 유사한 마크업을 작성할 수 있게 도와주는 자바스크립트용 구문 확장입니다. (_jsx 함수를 호출하는 편리한 문법에 불과합니다.)

JSX는 ECMAScript라 불리는 자바스크립트 표준 코드가 아닌 페이스북(현 메타)에서 임의로 만든 새로운 문법입니다. 다시 말해, 자바스크립트 엔진(V8, Deno 등)이나 브라우저(크롬, 웨일 등)에 의해서 실행되거나 표현되도록 만들어진 구문이 아닙니다. 따라서 Babel(자바스크립트 트랜스파일러)과 같은 트랜스파일러를 통해 변환되어야만 자바스크립트 런타임이 이해할 수 있는 의미 있는 자바스크립트 코드로 변환됩니다.

```javascript
const MyApp = () => {
  return <div className="title">Hello World</div>  
};
```
위 코드는 Babel에 의해 createElement 함수로 변환됩니다.
```javascript
const MyApp = () => {
  return React.createElement("div", { className: "title" }, "Hello World")
};
```
위 예시는 React 16 이하의 JSX Transform입니다. React 17 버전부터는 `React.createElement()` 대신 `react/jsx-runtime`에서 제공하는 `_jsx()` / `_jsxs()`을 사용합니다. `_jsx()` / `_jsxs()`을 사용하는 이유는 다음과 같습니다.
1. React를 import할 필요가 없습니다.
	- 기존 JSX 변환(React 16 이하)에서는 `React.createElement()`를 사용하므로 반드시 `import React from "react"`가 필요했습니다.
 	- `_jsx()`를 사용하면 `react/jsx-runtime`을 통해 자동으로 변환되므로 `import React`가 필요가 없습니다.

2. 번들 크기 감소
	- `React.createElement()`는 실행될 때마다 React 객체를 참조했지만, `_jsx()`는 직접 함수를 호출하므로 불필요한 import가 사라져 번들 크기가 줄었습니다.

3. 성능 최적화
	- `React.createElement()`보다 더 가벼운 호출 구조로 렌더링 성능이 향상되었습니다.

```javascript
// 리액트 17, 바벨 7.9.0 이후 버전에서 변환된 코드
import { jsx as _jsx } from "react/jsx-runtime";

function Example() {
 return _jsx("div", {
    className: "title"
    children: "Hello World"
  });
}
```
위처럼 React 17 이후 트랜스파일러에 의해 자동으로 `import { jsx as _jsx } from "react/jsx-runtime"` 이 추가됩니다. 그리고 children을 세 번째 인자(16 이하)가 아닌 두 번째 인자로 전달하는 것으로 변경됩니다.

### 자세한 변환 예제
리액트에서는 `@babel/plugin-transform-react-jsx 플러그인을 통해 JSX 구문을 자바스크립트가 이해할 수 있는 형태로 변환합니다.

```jsx  
// JSX 코드 예시
const A = <A required={true}>Hello World</A>
const B = <>Hello World</>
const C = (
  <div>
    <span>hello world</span>
  </div>
)
```

```JavaScript 
// React 16 이하
'use strict'

var A = React.createElement(
  A,
  {required: true},
  'Hello World'
)
var B = React.createElement(React.Fragment, null, 'Hello World')
var C = React.createElement(
  'div',
  null,
  React.createElement('span', null, 'hello world')
)
```

```javascript
// React 17, 바벨 7.9.0 이후 버전
// JSX 코드를 @babel/plugin-transform-react-jsx로 변환한 결과
'use strict'

var _jsxRuntime = require('custom-jsx-library/jsx-runtime')
var A = (0, _jsxRuntime.jsx)(A, {
  required: true,
  children: 'Hello World'
})
var B = (0, _jsxRuntime.jsx)(_jsxRuntime.Fragment, {
  children: 'Hello World'
})
var C = (0, _jsxRuntime.jsx)('div', {
  children: (0, _jsxRuntime.jsx)('span', {
    children: 'hello world'
  })
})
```
공통점
- JSXElement를 첫 번째 인수로 선언해 요소를 정의합니다.
- 옵셔널인 JSXChildren, JSXAttributes, JSXStrings는 이후 인수로 넘겨줍니다.

차이점
- (React 16 이하) JSX → `React.createElement(…)`
- (React 17 이상) JSX → `jsx(…)` (in react/jsx-runtime)

### _jsx()와 _jsxs()의 차이
리액트 17+에서는 JSX 변환 시 두 가지 새로운 함수가 사용됩니다.

| 함수명 | 사용 상황 |
| --- | --- |
| `_jsx()` | 자식이 하나인 경우 |
| `_jsxs()` | 자식이 여러 개인 경우 |

* * *

## Batching
Batching(배칭)은 업데이트된 여러 개의 상태 값들을 하나로 묶어 일정한 주기로 처리하여 렌더링 횟수를 줄이는 최적화 기법입니다.

리액트는 컴포넌트 상태가 변경될 때마다 렌더링을 수행하려고 합니다. 하지만 여러 개의 상태가 동시에 업데이트되면 각각 렌더링을 실행하는 것이 비효율적이므로, 배칭을 사용해 한 번만 렌더링 하도록 최적화합니다.

React 17 이하에서는 이벤트 핸들러 내부에서만 배칭이 적용되었습니다.
```js
const handleClick = () => {
  setCount(count + 1);
  setText("World");
};
```

하지만 `setTimeout`, `fetch` 등의 비동기 함수에서는 배칭이 동작하지 않았습니다.
```js
setTimeout(() => {
  setCount(count + 1);
  setText("World");
}, 1000);
```

React 18 이후로는 비동기 함수에서도 배칭이 자동으로 적용되었습니다. 즉, `setTimeout` 내부에서도 자동 배칭이 적용되어 한 번만 렌더링이 발생합니다.

배칭으로 인해 상태 업데이트가 즉시 반영되지 않을 때, `flushSync()`를 사용하면 강제로 렌더링을 실행할 수 있습니다.
```js
import { flushSync } from "react-dom";

const handleClick = () => {
  flushSync(() => setCount(count + 1)); // 즉시 렌더링 발생
  flushSync(() => setText("World"));    // 또 즉시 렌더링 발생
};
```

* * *

## References
[[NAVER D2] React 파이버 아키텍처 분석](https://d2.naver.com/helloworld/2690975)  
[[GitHub] React Fiber Architecture (acdlite)](https://github.com/acdlite/react-fiber-architecture)  
[[Mark's Dev Blog] A (Mostly) Complete Guide to React Rendering Behavior](https://blog.isquaredsoftware.com/2020/05/blogged-answers-a-mostly-complete-guide-to-react-rendering-behavior/#final-thoughts)  
[[Medium] What is React Fiber and How It Helps You Build a High-Performing React Applications](https://sunnychopper.medium.com/what-is-react-fiber-and-how-it-helps-you-build-a-high-performing-react-applications-57bceb706ff3)  
[[Medium] React Deep Dive — Fiber (Team QANDA)](https://blog.mathpresso.com/react-deep-dive-fiber-88860f6edbd0)
[[Medium] Concurrency vs. Parallelism — A brief view (Madhavan Nagarajan)](https://medium.com/@itIsMadhavan/concurrency-vs-parallelism-a-brief-review-b337c8dac350)  
