---
layout: article
title: "분명 바꿨는데 왜 state가 안 바뀔까"
category: "TIL"
tag: "React"
comment: true
key: 20260202
---

React를 처음 배울 때 useState를 쓰면서 "왜 배열을 직접 수정하면 안 되지?", "왜 spread operator를 써야 하지?" 같은 의문이 생긴다. 그냥 외워서 쓰는 사람도 많은데, 사실 이건 React만의 특별한 규칙이 아니라 JavaScript의 데이터 타입 특성과 React의 렌더링 최적화 전략이 만나서 생긴 결과라고 보면 된다.

## State란 무엇인가

React에서 state는 컴포넌트의 기억이다. 사용자의 입력, 서버에서 받아온 데이터, UI의 현재 상태 같은 것들을 저장하는 공간이다. 그런데 왜 굳이 state를 써야 할까? 걍 일반 변수를 쓰면 안 되나?

```jsx
function Counter() {
  let count = 0;

  const handleClick = () => {
    count = count + 1;
    console.log(count);  // 1, 2, 3... 증가함
  };

  return <button onClick={handleClick}>{count}</button>;  // 항상 0
}
```

버튼을 클릭하면 count 변수는 증가한다. console.log로 찍어보면 1, 2, 3으로 올라가는 게 보인다. 그런데 막상 화면에는 계속 0으로 표시 될건데, 왜 그럴까?

두 가지 문제가 있다.

첫째, count 값을 바꿔도 React는 그 사실을 모른다. 일반 변수가 바뀌었다고 React가 알아서 화면을 다시 그려주지 않는다. React 입장에서는 "화면을 다시 그려야 할 이유"가 없는 것이다.

둘째, 설령 다른 이유로 리렌더링이 발생하더라도 `let count = 0`이 다시 실행되면서 값이 초기화된다. React 컴포넌트는 함수고, 렌더링할 때마다 이 함수가 처음부터 다시 실행되기 때문이다.

state는 이 두 가지 문제를 해결한다. 첫째, state는 컴포넌트가 다시 렌더링되어도 값이 유지된다. React가 내부적으로 값을 기억하고 있기 때문이다. 둘째, state가 변경되면 React에게 "화면을 다시 그려야 해"라고 알려준다. 리렌더링 트리거다.

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  const handleClick = () => {
    setCount(count + 1);  // React에게 변경을 알림
  };

  return <button onClick={handleClick}>{count}</button>;  // 1, 2, 3...
}
```

useState가 반환하는 배열의 첫 번째 요소는 현재 값이고, 두 번째 요소는 그 값을 변경하는 함수다. 이 변경 함수(setter)를 통해서만 state를 바꿔야 React가 변경을 감지할 수 있다.

## useState 동작 원리

React의 state 변경함수는 생각보다 단순한 원리로 동작한다. 새로운 값이 들어오면 기존 값과 비교해서, 같으면 아무것도 안 하고, 다르면 리렌더링을 실행한다. 요게 전부라고 보면된다.

뭐 내부적으로 React는 `Object.is()`라는 비교 함수를 사용한다. 여기서 참고로, 왜 `===`가 아니라 `Object.is()`일까? 대부분의 경우 둘은 같은 결과를 내지만, 두 가지 엣지 케이스에서 다르게 동작한다.

첫 번째는 NaN이다. JavaScript에서 NaN은 자기 자신과도 같지 않은 유일한 값이다.

```javascript
NaN === NaN           // false - 뭔가 이상하다
Object.is(NaN, NaN)   // true - 직관적
```

`===`로 비교하면 NaN은 자기 자신과도 다르다고 판단된다. 이건 IEEE 754 부동소수점 표준을 따른 것인데, 실용적으로는 혼란스럽다. `Object.is()`는 이걸 바로잡아서 NaN끼리는 같다고 판단한다. 만약 React가 `===`를 썼다면, state가 NaN일 때 같은 NaN을 넣어도 매번 리렌더링이 발생했을 것이다.

두 번째는 양수 0과 음수 0이다. JavaScript에는 +0과 -0이 별도로 존재한다. 보통은 신경 쓸 일이 없지만, 수학 연산에서 가끔 -0이 튀어나온다.

```javascript
-0 === +0           // true - 구분 못함
Object.is(-0, +0)   // false - 구분함

// -0이 나오는 예시
Math.round(-0.1)    // -0
-1 * 0              // -0
```

`===`는 +0과 -0을 같다고 보지만, `Object.is()`는 다르다고 본다. 이게 실제로 문제가 되는 경우는 드물지만, 수치 계산을 다루는 애플리케이션에서는 의미 있는 차이일 수 있다.

React가 `Object.is()`를 선택한 건 "개발자의 직관에 더 맞는 비교"를 하기 위해서다. NaN은 NaN이고, -0과 +0은 엄밀히 다른 값이니까. 물론 일상적인 React 개발에서 이 차이가 체감되는 경우는 거의 없다. 대부분은 숫자, 문자열, 불리언을 다루고, 이 경우 `===`와 `Object.is()`는 완전히 동일하게 동작한다.

여튼 진짜 문제는 배열과 객체다. `Object.is([1,2,3], [1,2,3])`의 결과가 뭘까? 내용이 완전히 같으니 true일 것 같지만, 실제로는 false다. 이걸 이해하려면 JavaScript가 데이터를 메모리에 어떻게 저장하는지 알아야 한다.

## JavaScript의 두 가지 데이터 타입

JavaScript의 데이터 타입은 크게 원시 타입(Primitive Type)과 참조 타입(Reference Type)으로 나뉜다.

원시 타입에는 number, string, boolean, undefined, null, symbol, bigint 이렇게 7가지가 있다. 이 타입들의 특징은 변수에 값 자체가 저장된다는 것이다. `let a = 10`이라고 하면, 메모리 어딘가에 10이라는 값이 직접 들어간다. `let b = a`를 하면 10이라는 값이 복사되어 b만의 공간에 새로 저장된다. 그래서 이후에 b를 수정해도 a는 영향받지 않는다. 완전히 독립적인 두 개의 값이 존재하는 것이다.

참조 타입은 다르게 동작한다. 배열, 객체, 함수 등이 여기에 속한다. 이 타입들은 변수에 값이 아니라 메모리 주소가 저장된다. `let arr1 = [1, 2, 3]`을 실행하면, 실제 배열 데이터는 힙(Heap) 메모리 어딘가에 저장되고, arr1 변수에는 그 위치를 가리키는 주소값만 들어간다. 일종의 화살표라고 생각하면 된다.

이 설계에는 이유가 있다. 배열에 데이터가 100만 개 있다고 생각해보자. 만약 원시 타입처럼 값 자체를 복사한다면, 변수 할당할 때마다 100만 개의 데이터가 통째로 복사되어야 한다. 메모리가 순식간에 터질 것이다. 그래서 실제 데이터는 한 곳에만 두고, 변수에는 그 위치만 저장하는 방식을 택한 것이다. 주소 하나는 고작 8바이트니까.

## 참조 복사의 함정

문제는 여기서 시작된다. `let arr2 = arr1`을 실행하면 무슨 일이 벌어질까? 배열이 복사되는 게 아니라, 주소가 복사된다. arr1과 arr2는 이제 같은 배열을 가리키는 두 개의 화살표가 된다.

```javascript
let arr1 = [1, 2, 3];
let arr2 = arr1;

arr2[0] = 999;

console.log(arr1);  // [999, 2, 3]
```

arr2를 수정했는데 arr1도 바뀌었다. 둘이 같은 배열을 가리키고 있으니 당연한 결과다. 그리고 `arr1 === arr2`를 비교하면 true가 나온다. 비교 연산자는 참조 타입에 대해 주소를 비교하기 때문이다. 같은 곳을 가리키고 있으니 같다고 판단하는 것이다.

반대로 `[1,2,3] === [1,2,3]`은 false다. 내용은 같지만 각각 새로 만들어진 배열이라 서로 다른 메모리 주소를 가지고 있다. JavaScript 입장에서 이 둘은 완전히 다른 배열이다.

## React에서 문제가 되는 이유

ㅇㅋ 이제 React로 돌아가보자. state로 배열을 관리하고 있다고 하자.

```jsx
const [items, setItems] = useState(['사과', '바나나', '오렌지']);
```

여기서 첫 번째 항목을 '포도'로 바꾸고 싶다. 직관적으로 이렇게 하고 싶을 것이다.

```jsx
items[0] = '포도';
setItems(items);
```

배열 내용을 바꾸고, 그 배열을 setState에 넣었다. 논리적으로는 맞는 것 같다. 하지만 화면은 바뀌지 않는다.

이유는 앞에서 설명한 그대로다. `items[0] = '포도'`는 배열의 내용을 바꿨지만, items가 가리키는 메모리 주소는 그대로다. setItems(items)가 호출되면 React는 기존 state와 새 state를 비교한다. 둘 다 같은 주소를 가리키고 있으니 `Object.is(기존items, 새items)`는 true를 반환한다. React 입장에서는 "바뀐 게 없네"라고 판단하고 리렌더링을 하지 않는다.

내용은 분명히 바뀌었다. 하지만 React는 내용을 일일이 비교하지 않는다. 그러면 너무 느리니까. 대신 주소만 비교해서 빠르게 판단한다. 이게 React의 최적화 전략이다.

## 해결책: 새로운 배열 만들기

React가 변경을 감지하게 하려면 새로운 주소를 가진 배열을 만들어서 넘겨야 한다. 일단은 가장 흔히 쓰는 방법이 spread operator다.

```jsx
const copy = [...items];
copy[0] = '포도';
setItems(copy);
```

`[...items]`는 items의 모든 요소를 꺼내서 새 배열에 담는 문법이다. 새 배열이니 당연히 새로운 메모리 주소를 가진다. 이 배열을 setItems에 넘기면 React는 "오 주소가 다르네. 뭔가 바뀌었구나"라고 판단하고 리렌더링을 실행한다고 보면된다.

## Spread Operator의 동작 원리

참고로 spread operator(`...`)는 ES6에서 추가된 문법으로, 배열이나 객체의 요소를 펼쳐놓는 역할을 한다. `console.log(...[1,2,3])`을 실행하면 `1 2 3`이 출력된다. 배열이라는 껍데기가 벗겨지고 내용물만 나오는 것이다.

이걸 새 배열 리터럴 안에서 사용하면 `[...arr]`처럼 되는데, arr의 모든 요소를 꺼내서 새 배열에 담으라는 뜻이다. 결과적으로 내용은 같지만 완전히 새로운 배열이 만들어진다.

객체에도 같은 원리가 적용된다. `{ ...obj }`는 obj의 모든 속성을 새 객체에 복사한다. 그리고 뒤에 속성을 추가하면 덮어쓰기가 된다.

```jsx
const [user, setUser] = useState({ name: '철수', age: 25 });

// age만 변경
setUser({ ...user, age: 26 });
```

`{ ...user, age: 26 }`은 user의 모든 속성을 새 객체에 복사한 다음, age 속성을 26으로 덮어쓴다. 순서가 중요하다. spread가 먼저 오고 변경할 값이 뒤에 와야 제대로 덮어쓰기가 된다.

배열에서 특정 요소를 변경하는 가장 깔끔한 방법은 map을 쓰는 것이다.

```jsx
setItems(items.map((item, i) => i === 0 ? '포도' : item));
```

map은 항상 새 배열을 반환하기 때문에 spread 없이도 새 참조가 만들어진다. filter도 마찬가지다. 여기서 의문이 생길 수 있다.

"매번 새 배열을 만들면 메모리 낭비 아닌가?"

결론부터 말하면, 대부분의 경우 걱정할 필요 없다.

JavaScript의 가비지 컬렉터는 참조가 없어진 객체를 자동으로 정리한다. 이전 배열은 더 이상 아무도 참조하지 않으니 메모리에서 해제된다. 그리고 현대 JavaScript 엔진은 이런 작은 객체 생성과 해제에 최적화되어 있다. 배열 요소가 수천, 수만 개가 아니라면 성능 차이는 거의 느낄 수 없다.

물론 극단적인 경우는 있다. 60fps로 애니메이션을 돌리면서 매 프레임마다 거대한 배열을 복사한다면 문제가 될 수 있다. 하지만 이건 일반적인 state 관리와는 다른 영역이고, 그런 상황에서는 애초에 state 대신 다른 접근법(Canvas, requestAnimationFrame 등)을 쓰는 게 맞다.

오히려 메모리 걱정된다고 원본을 직접 수정하면 더 큰 문제가 생길 수도 있다. React가 변경을 감지 못해서 UI가 안 바뀌고, 버그를 찾느라 시간을 훨씬 더 쓰게 된다. 성능 최적화는 실제로 문제가 측정됐을 때 하는 거지, 미리 걱정해서 코드를 복잡하게 만들 필요는 없다.

## Shallow Copy의 한계

여기서 한 가지 주의할 점이 있다. spread operator는 얕은 복사(Shallow Copy)만 수행한다. 1단계 깊이만 새로 만들고, 중첩된 객체나 배열은 여전히 같은 참조를 공유한다.

```javascript
const original = {
  name: '철수',
  address: { city: '서울' }
};

const copy = { ...original };
copy.address.city = '부산';

console.log(original.address.city);  // '부산' - 원본도 바뀜!
```

copy는 새 객체지만, copy.address와 original.address는 여전히 같은 객체를 가리킨다. spread가 address 속성의 값(주소)을 그대로 복사했기 때문이다.

React state가 중첩 구조일 때 이게 문제가 된다.

```jsx
const [data, setData] = useState({
  user: {
    settings: { theme: 'dark' }
  }
});

// 틀린 방법
setData({ ...data });  // user는 여전히 같은 참조

// 맞는 방법 - 변경 경로의 모든 객체를 새로 만들어야 함
setData({
  ...data,
  user: {
    ...data.user,
    settings: {
      ...data.user.settings,
      theme: 'light'
    }
  }
});
```

theme 하나 바꾸려고 이 난리를 쳐야 한다. 깊이가 깊어질수록 코드가 지저분해진다. 아 이럴 때는 immer 같은 라이브러리가 유용하다고 한다. immer를 쓰면 마치 직접 수정하는 것처럼 코드를 쓸 수 있고, 내부적으로 알아서 새 객체를 만들어준다.

## Deep Copy가 필요할 때

중첩 구조 전체를 완전히 독립적으로 복사하고 싶다면 깊은 복사(Deep Copy)가 필요하다. 모든 깊이를 재귀적으로 새로 만드는 것이다.

가장 간단한 방법은 `JSON.parse(JSON.stringify(obj))`다. 객체를 JSON 문자열로 변환했다가 다시 파싱하면 완전히 새로운 객체가 만들어진다. 다만 이 방법은 함수, undefined, Symbol 같은 값은 복사하지 못하고, Date 객체는 문자열로 바뀌어버리는 한계가 있다.

모던 브라우저에서는 `structuredClone()`이라는 내장 함수를 쓸 수 있다. JSON 방식의 한계를 대부분 해결했고, Date나 Map, Set 같은 객체도 제대로 복사한다. 다만 함수는 여전히 복사하지 못한다.

실무에서는 lodash의 `cloneDeep` 함수를 많이 쓴다. 검증된 라이브러리라 안정적이고, 대부분의 케이스를 처리한다.

하지만 React state 업데이트에서 deep copy를 남발할 필요는 없다. 변경이 필요한 경로만 새로 만들면 되기 때문이다. 오히려 매번 전체를 deep copy하면 성능상 손해다.

## 함수형 업데이트

setState에는 값을 직접 넘기는 방법 외에 함수를 넘기는 방법도 있다.

```jsx
// 값을 직접 전달
setCount(count + 1);

// 함수를 전달
setCount(prev => prev + 1);
```

두 방식의 차이가 뭘까? 결과만 보면 같아 보이지만, 동작 방식이 다르다.

값을 직접 전달하면 그 시점의 count 값을 사용한다. 함수를 전달하면 React가 "현재 state 값"을 인자로 넘겨주고, 그 값을 기반으로 새 값을 계산한다.

이 차이는 연속된 업데이트에서 드러난다.

```jsx
// 문제 상황
const handleClick = () => {
  setCount(count + 1);
  setCount(count + 1);
  setCount(count + 1);
};
```

버튼 한 번 클릭하면 count가 3 증가할 것 같다. 하지만 실제로는 1만 증가한다. 왜 그럴까?

이건 React의 batching 때문이다. React는 성능 최적화를 위해 여러 state 업데이트를 모아서 한 번에 처리한다. 위 코드가 실행될 때 count가 0이었다면, 세 번의 setCount 모두 `setCount(0 + 1)`로 실행된다. 세 번 다 같은 값 1을 설정하는 셈이다.

함수형 업데이트를 쓰면 이 문제가 해결된다.

```jsx
const handleClick = () => {
  setCount(prev => prev + 1);  // 0 -> 1
  setCount(prev => prev + 1);  // 1 -> 2
  setCount(prev => prev + 1);  // 2 -> 3
};
```

React가 각 함수를 순차적으로 실행하면서 이전 결과를 다음 함수의 인자로 넘겨준다. 그래서 의도한 대로 3이 증가한다.

## Batching의 동작 원리

React 18부터는 Automatic Batching이 도입되었다. 이전 버전에서는 이벤트 핸들러 안에서만 batching이 동작했는데, 이제는 setTimeout, Promise, 네이티브 이벤트 핸들러 등 어디서든 batching이 적용된다.

```jsx
setTimeout(() => {
  setCount(c => c + 1);
  setFlag(f => !f);
  // React 18 이전: 여기서 두 번 렌더링됨
  // React 18 이후: 한 번만 렌더링됨
}, 1000);
```

같은 코드인데 React 버전에 따라 동작이 다르다. React 18부터는 setTimeout 안에서도 batching이 적용되어서 한 번만 렌더링된다.

batching은 성능에 중요하다. state가 바뀔 때마다 리렌더링이 발생하면, 여러 state를 연속으로 바꿀 때 불필요한 중간 렌더링이 많아진다. batching은 모든 업데이트가 끝날 때까지 기다렸다가 한 번만 렌더링한다.

만약 batching을 원하지 않고 즉시 렌더링하고 싶다면 `flushSync`를 쓸 수 있다.

```jsx
import { flushSync } from 'react-dom';

const handleClick = () => {
  flushSync(() => {
    setCount(c => c + 1);
  });
  // 여기서 DOM이 이미 업데이트됨

  flushSync(() => {
    setFlag(f => !f);
  });
  // 여기서 또 DOM 업데이트됨
};
```

하지만 flushSync는 성능상 좋지 않으니 꼭 필요한 경우에만 써야 한다고...

## setState 직후에 값을 찍으면?

batching과 관련해서 자주 하는 실수가 있다. setState 직후에 state 값을 확인하려고 console.log를 찍는 것이다.

```jsx
const handleClick = () => {
  setCount(count + 1);
  console.log(count);  // 여전히 이전 값!
};
```

count가 0일 때 버튼을 클릭하면 console.log는 1이 아니라 0을 출력한다. 에.. 아니 분명히 setCount를 호출했는데 왜?

이건 setState가 비동기적으로 동작하기 때문이다. 정확히 말하면, setState는 "state를 이 값으로 바꿔줘"라는 요청을 큐에 넣는 것이지, 그 자리에서 즉시 바꾸는 게 아니다. React는 이 요청들을 모아서(batching) 나중에 한꺼번에 처리한다.

그래서 setState 호출 직후의 시점에서 count 변수는 아직 이전 값을 가리키고 있다. 새 값은 다음 렌더링에서야 반영된다.

만약 변경된 값을 바로 사용해야 한다면, 변수에 저장해두면 된다.

```jsx
const handleClick = () => {
  const newCount = count + 1;
  setCount(newCount);
  console.log(newCount);  // 새 값 출력

  // 서버에 보내거나, 다른 로직에 사용
  sendToServer(newCount);
};
```

또는 useEffect를 써서 state가 바뀐 후에 반응할 수 있다.

```jsx
useEffect(() => {
  console.log('count가 바뀜:', count);
}, [count]);
```

"setState가 동기적으로 동작했으면 좋겠다"고 생각할 수 있지만, 그러면 앞서 말한 batching의 이점을 잃는다. 매번 즉시 리렌더링하면 성능이 떨어진다. 비동기 동작은 불편해 보여도 성능을 위한 설계다.

## Closure와 Stale State

React를 쓰다 보면 "아 분명 state를 바꿨는데 왜 옛날 값이 나오지? 같은 상황을 만난다. 대표적인 케이스가 setTimeout이나 setInterval 안에서 state를 참조할 때 그렇다.

```jsx
function Timer() {
  const [count, setCount] = useState(0);

  const handleStart = () => {
    setInterval(() => {
      console.log(count);  // 항상 0이 출력됨
      setCount(count + 1); // 항상 0 + 1 = 1로 설정됨
    }, 1000);
  };

  return <button onClick={handleStart}>Start</button>;
}
```

버튼을 클릭하면 1초마다 count가 증가해야 할 것 같지만, 실제로는 count가 계속 1에서 멈춘다. console.log도 계속 0을 출력한다.

이건 JavaScript의 클로저(Closure) 때문이다. setInterval에 전달된 함수는 handleStart가 실행될 때의 count 값(0)을 "기억"한다. 이후에 state가 바뀌어도 그 함수가 기억하는 count는 여전히 0이다. 이걸 stale closure 또는 stale state 문제라고 부른다.

해결책은 함수형 업데이트를 쓰는 것이다.

```jsx
const handleStart = () => {
  setInterval(() => {
    setCount(prev => prev + 1);  // 항상 최신 값 기준으로 계산
  }, 1000);
};
```

함수형 업데이트를 쓰면 React가 현재 state 값을 인자로 넘겨주기 때문에 클로저 문제를 피할 수 있다.

만약 state 값을 읽어야 하는 경우(console.log 등)에는 useRef를 활용할 수 있다.

```jsx
function Timer() {
  const [count, setCount] = useState(0);
  const countRef = useRef(count);

  useEffect(() => {
    countRef.current = count;  // state가 바뀔 때마다 ref 업데이트
  }, [count]);

  const handleStart = () => {
    setInterval(() => {
      console.log(countRef.current);  // 최신 값 출력
      setCount(prev => prev + 1);
    }, 1000);
  };

  return <button onClick={handleStart}>Start</button>;
}
```

useRef가 반환하는 객체는 컴포넌트 생애 동안 동일한 참조를 유지한다. 그래서 ref.current를 통해 항상 최신 값에 접근할 수 있다.

## 왜 React는 이렇게 설계했을까

React가 굳이 이런 방식을 택한 이유가 있다. 내용을 일일이 비교하는 건 비용이 크다. 객체에 속성이 100개 있으면 100번 비교해야 하고, 중첩 구조면 재귀적으로 들어가야 한다. 반면 참조(주소) 비교는 딱 한 번이면 끝난다. 음 O(n)과 O(1)의 차이?

또한 원본을 직접 수정하지 않고 새 복사본을 만드는 패턴, 이른바 불변성(Immutability)을 지키면 여러 이점이 있다. 이전 상태가 그대로 보존되니 디버깅이 쉽고, Redux DevTools 같은 도구에서 시간 여행 디버깅도 가능해진다. 상태가 언제 바뀌는지 명확해서 버그를 추적하기도 쉽다.

batching과 함수형 업데이트도 마찬가지다. 매번 즉시 렌더링하면 성능이 떨어지고, 클로저 문제도 함수형 프로그래밍 패러다임에서 자연스럽게 해결된다. React의 설계 철학 전체가 "예측 가능하고 효율적인 UI 업데이트"를 향해 있다.

처음에는 귀찮게 느껴질 수 있다. 배열 하나 수정하려고 spread를 쓰고 새 배열을 만들어야 하고, setTimeout에서 값이 안 바뀌는 것도 이해해야 한다. 뭐 그래도 이런 원리를 알고 나면 "왜 state가 안 바뀌냐", "왜 옛날 값이 나오냐" 같은 문제를 겪을 일이 거의 없어진다. 그런 문제의 대부분은 참조와 클로저를 이해하지 못해서 생기는 것이기 때문이다.

## 아 근데 Vue는 그냥 되던데?

React를 쓰다가 Vue를 접하면, 또는 그 반대면 의아할 수 있다. Vue에서는 배열을 직접 수정해도 화면이 잘 바뀐다.

```javascript
// Vue 3
const items = ref(['사과', '바나나', '오렌지']);

items.value[0] = '포도';  // 그냥 바꿔도 됨!
items.value.push('딸기'); // 이것도 됨!
```

Vue는 Proxy를 사용한다. ref()나 reactive()로 감싼 객체는 JavaScript Proxy로 래핑되어서, 속성에 접근하거나 수정할 때마다 Vue가 그걸 가로챈다. `items.value[0] = '포도'`를 실행하면 Proxy의 set 트랩이 호출되고, Vue는 "아, 이 데이터가 바뀌었구나"를 자동으로 감지한다.

React는 다른 철학을 택했다. 데이터를 감싸서 변경을 추적하는 대신, 개발자가 명시적으로 "이게 바뀌었어"라고 알려주는 방식이다. 그래서 setState를 호출해야 하고, 새 참조를 만들어야 한다.

어느 쪽이 더 좋다고 말하기는 어렵다. Vue 방식은 직관적이고 편하다. 그냥 값을 바꾸면 되니까. 하지만 "마법"이 많아서 내부에서 뭐가 일어나는지 파악하기 어려울 수 있고, Proxy 오버헤드도 있다.

React 방식은 장황하고 불편하다. 하지만 데이터 흐름이 명시적이라 디버깅이 쉽고, 순수 JavaScript 객체를 그대로 쓰니까 예측 가능하다. 그리고 불변성을 강제하면서 얻는 이점들(시간 여행 디버깅, 쉬운 변경 감지 등)도 있다.

결국 트레이드오프다. React를 쓴다면 이 방식에 익숙해지는 게 맞고, 불편하다면 immer 같은 도구로 보완할 수 있다. 프레임워크마다 철학이 다르고, 그 철학에 맞는 패턴을 익히는 게 중요하다.

---

글이 꽤 길어졌는데, 결국 핵심은 React는 state가 바뀌었는지를 참조 비교로 판단한다는 것이다. 그래서 객체나 배열은 내용을 바꾸는 게 아니라 새로 만들어야 한다. 이 원리만 이해하면 대부분의 state 관련 버그는 피할 수 있다.
