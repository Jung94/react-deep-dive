# React Deep Dive
리액트 렌더링과 Fiber 아키텍처 분석

## 개요
이 글의 목적은 리액트의 전반적인 동작 방식과 주요 개념인 Fiber 아키텍처에 대한 깊이 있는 이해입니다.

이전에 공부하고 정리했던 리액트 렌더링 과정과 Fiber 아키텍처를 다시 상기하고 재정리합니다. 그리고 바닐라 자바스크립트로 리액트를 직접 구현해 보고자 합니다.

## 목차
[리액트 렌더링은 어떻게 일어나는가?](#리액트-렌더링은-어떻게-일어나는가)  
[JSX](#jsx)  
[Reconciliation](#reconciliation)  
[Batching](#batching)  

## 리액트 렌더링은 어떻게 일어나는가?
리액트에서의 렌더링은 브라우저 렌더링과 다릅니다. 브라우저 렌더링이란 브라우저가 HTML, CSS, JavaScript를 해석하여 우리가 보는 화면에 실제로 나타내는 과정을 말합니다. 단순히 파일을 읽는 것만 아니라 웹 페이지의 구조를 분석하고 스타일과 스크립트를 계산하며, 사용자에게 보이는 픽셀 단위의 화면을 만들어내는 아주 복잡합 과정을 의미합니다.

반면, 리액트에서의 렌더링이란 모든 컴포넌트가 props와 state를 기반으로 어떻게 UI를 구성하고 어떤 DOM 결과를 브라우저에게 제공할 것인지 계산하는 일련의 과정을 의미합니다.
즉 브라우저 렌더링은 화면에 무언가 바로 나타나는 것이지만, 리액트 렌더링은 화면에 무언가 나타나기 전 일련의 과정을 일컫습니다. 여기서 주목해야 할 점은 리액트에서의 렌더링은 DOM 업데이트가 아니라는 점입니다. 즉, 리액트 렌더링은 실질적인 화면 업데이트가 아닙니다.

### 리액트 렌더링의 전체적인 흐름
React 팀은 리액트의 렌더링 과정을 크게 두 단계로 나눴습니다.
1. Render Phase (렌더 단계): 컴포넌트를 렌더링하고 변경 사항을 계산하는 모든 과정이 이루어지는 단계
2. Commit Phase (커밋 단계): 변경 사항을 실제 DOM에 적용하는 단계

### 리액트에서 렌더링이 일어나는 이유
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

> [Reconciliation (조정 과정)](#reconciliation)  
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

## Reconciliation
Reconciliation(재조정)이란 컴포넌트의 상태가 변경되면 기존 가상 DOM 트리와 변경된 가상 DOM 트리를 비교해 어떤 부분을 변경해야 하는지 찾아내기 위한 알고리즘입니다. 이 과정에서 비교 알고리즘(Diffing Algorithm)이 사용되며 성능 최적화를 위해 몇 가지 규칙을 사용합니다.
- 같은 타입의 요소는 그대로 유지
- 다른 타입의 요소는 새로 생성
- key를 활용한 리스트 비교 최적화



## Batching
리액트는 렌더링을 최소한으로 일으키기 위해 작업을 한 번에 묶어서 일괄 처리합니다. Trigger 단계에서 setState 함수는 Queue에 순차적으로 들어갑니다. 비교 알고리즘으로 어떤 DOM이 변경되어야 하는지 알았기 때문에

## Fiber Architecture
### 등장 배경
Fiber 아키텍처는 스택 아키텍처의
### 핵심 기능
리액트 Fiber의 가장 핵심 기능 중 하나는 '비동기 렌더링'입니다. 비동기 렌더링을 통해 리액트는 더 중요한 업데이트를 우선적으로 처리 가능하며, 사용자 인터렉션 등의 중요 작업을 방해 받지 않고 처리할 수 있기 때문입니다.

또 다른 핵심 기능은 '더블 버퍼링'입니다. 더블 버퍼링은 현재 화면에 표시되는 내용과 새로 업데이트될 내용을 별도로 관리함으로써 화면 깜빡임 없이 부드러운 UI 업데이트를 가능하게 합니다. 이를 통해 애플리케이션의 렌더링 성능을 최적화하고 사용자에게 부드러운 인터렉션을 제공합니다.
