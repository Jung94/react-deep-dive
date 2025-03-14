# React Deep Dive
리액트 렌더링과 파이버 아키텍처 분석

## 개요
이 글의 목적은 리액트의 전반적인 동작 방식과 주요 개념인 파이버 아키텍처에 대한 깊이 있는 이해입니다.

이전에 공부하고 정리했던 리액트 렌더링 과정과 파이버 아키텍처를 다시 상기하고 재정리합니다. 그리고 바닐라 자바스크립트로 리액트를 직접 구현해 보고자 합니다.

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
위 예시는 React 17 이전의 JSX Transform으로 여러 가지 문제점이 존재합니다. 대표적인 문제점으로 React.createElement 코드를 사용하기 위해 `import React from 'react';`를 미리 선언해야 동작한다는 것입니다. 이밖에 여러 문제점으로 인해 현재 createElement는 legacy API가 되었습니다.
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
React 17 이후, 트랜스파일러에 의해 자동으로 `import { jsx as _jsx } from "react/jsx-runtime"` 이 추가됩니다. 그리고 children을 세 번째 인자(17 이전)가 아닌 두 번째 인자로 전달하는 것으로 변경됩니다.

### 자세한 변환 예제
리액트에서는 @babel/plugin-transform-react-jsx 플러그인을 통해 JSX 구문을 자바스크립트가 이해할 수 있는 형태로 변환합니다.
```javascript
// JSX 코드
const A = <A required={true}>Hello World</A>
const B = <>Hello World</>
const C = (
	<div>
		<span>hello world</span>
	</div>
)
```
```javascript
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
아래는 리액트 17, 바벨 7.9.0 이후 버전에서 JSX 코드를 @babel/plugin-transform-react-jsx로 변환한 결과입니다.
```javascript
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
- 옵셔널인 JSXChildren, JSXAttributes, JSXStrings는 이후 인수로 넘겨준다는 것입니다.

차이점
- (React 17 이전) JSX → `React.createElement(…)`
- (React 17 이후) JSX → `jsx(…)` (in react/jsx-runtime)
