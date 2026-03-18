---
layout: article
title: "함수형 프로그래밍, 생각보다 재밌다"
category: "Wooteco"
tag: "Wooteco"
comment: true
key: 20260319
---

우테코 프론트엔드는 요새 스터디(원정대)에 정신이 팔려있다.

아니 객체지향이면 충분하지 않냐?. 근데 책 읽고 직접 코드를 뜯어서 리팩토링해보니까 이거 졸라 재밌음..
함수형이 얼마나 재밌는지, 미션도 그냥 대충 평일동안 짬내서 만들어둔거 리팩토링도 안하고 제출했다 ㅋㅋㅋㅋㅋ

<!--more-->

> [스터디 레포](https://github.com/woowacourse-study/2026-fp-deep-dive)

문제는, 스터디는 정말 "스터디답게" 각자 속한 원정대에서 딥하게 올인해야하는데, 자꾸 시스템적으로 홍보와 발표에 초점을 두게끔 유도하는 느낌이 난달까.
다같이 거의 밤새가면서 공부하기도하고 주말에도 안쉬고 서로 피드백하면서, 매우 건전(?)하게 하고있는데, 갑자기 다른 백엔드에도 발표해야하고 하니 크루들도 점점 부담을 가지게 되어가고, **"결국 중요하지도 않은 발표준비에 시간을 너무 쏟게된다..."**

아무리봐도 이번주간에는 원정대활동을 하면서 딥하게! 뾰족하게! 공부를 하고있어야하는 꿈만같은 시간인데!, 우리 크루만 봐도 나도 홍보준비하고있고, 다른 크루분들은 웹앱으로 대본 맞추고, 이러는게 너무 내심 짜증이났다...

우리 원정대원들끼리 다른거 준비하지말고 딥하게 공부하라고만 시켜주면, 다같이 죽자살자 하면서 문제만들고 풀고 이러고있었을 엄청난 시간이였을텐데!! 여튼 원정대가 다좋은데 이건 좀 아쉽다.

## 액션, 계산, 데이터

책 기준으로 일단, 함수형 프로그래밍의 시작은 모든 코드를 세 가지로 분류하는 것이다. 액션, 계산, 데이터.

**데이터**는 그냥 값이다. 숫자, 문자열, 객체, 배열 같은 것들. 이벤트에 대한 사실을 기록한 것. 실행되지 않고, 그냥 거기 있을 뿐이다.

```js
const rooms = [
  { id: "ROOM-A", name: "루비룸", capacity: 4, pricePerHour: 1000 },
  { id: "ROOM-B", name: "자바룸", capacity: 8, pricePerHour: 2000 },
];
```

이런 게 데이터다. 아무 동작도 하지 않는다. 그냥 사실을 담고 있을 뿐.

**계산**은 같은 입력을 넣으면 항상 같은 출력이 나오는 함수다. 호출 시점이 언제든, 몇 번을 호출하든 결과가 똑같다.

```js
function calculateFee(pricePerHour, duration) {
  return pricePerHour * duration;
}

calculateFee(1000, 2); // 항상 2000
calculateFee(1000, 2); // 항상 2000
```

이게 왜 좋냐면, 테스트가 졸라 쉽다. 입력 넣고 출력 확인하면 끝이다. 외부 상태 신경 안 써도 된다.

**액션**은 호출 시점이나 횟수에 따라 결과가 달라지는 함수다. 전역 변수를 바꾸거나, 콘솔에 출력하거나, API를 호출하거나. 이런 게 다 액션이다.

```js
let currentMember = null;
let reservations = [];

function registerMember(id, name, point) {
  currentMember = createMember(id, name, "normal", point, 0); // 전역 변수 변경
  reservations = [];                                            // 전역 변수 변경
  console.log(`${name}님이 등록되었습니다`);                      // 외부 출력
}
```

이 함수를 두 번 호출하면 `currentMember`가 매번 달라진다. 전역 상태를 바꾸고 콘솔에 출력하니까.

## 왜 액션이 문제인가

액션이 나쁘다는 건 아니다. 프로그램에서 액션은 반드시 존재한다. 문제는 **액션과 계산이 한 함수에 섞여있을 때**다.

원본 코드의 `makeReservation`을 보면:

```js
function makeReservation(roomId, date, startHour, duration, attendees) {
  var room = null;
  for (var i = 0; i < rooms.length; i++) {      // ← 전역 rooms 참조 (액션)
    if (rooms[i].id === roomId) room = rooms[i];
  }

  var fee = room.pricePerHour * duration;         // ← 이건 계산인데
  var pointRate = gradeConfig[currentMember.grade].pointRate; // ← 전역 참조 (액션)
  var earnedPoints = Math.floor(fee * pointRate / 100);       // ← 이것도 계산인데

  currentMember.points += earnedPoints;           // ← 전역 변수 직접 변이 (액션)
  currentMember.totalUsageHours += duration;      // ← 또 변이 (액션)

  console.log("예약 완료! ...");                   // ← 외부 출력 (액션)
}
```

요금 계산(`pricePerHour * duration`)이 맞는지만 확인하고 싶어도, 이 함수를 호출하면 전역 변수가 바뀌고 콘솔이 찍힌다. 계산만 따로 테스트할 방법이 없다.

분리하면?

```js
// 계산 — 언제 몇 번 호출해도 같은 결과
function calculateFee(pricePerHour, duration) {
  return pricePerHour * duration;
}

function calculateEarnedPoints(fee, gradeConfig, grade) {
  const pointRate = getPointRate(gradeConfig, grade);
  return calculatePercentage(fee, pointRate);
}

// 액션 — 전역 변경 + 출력은 여기서만
function makeReservation(roomId, date, startHour, duration, attendees) {
  const roomInfo = findById(rooms, roomId);
  const fee = calculateFee(roomInfo.pricePerHour, duration);
  const earnedPoints = calculateEarnedPoints(fee, gradeConfig, currentMember.grade);

  currentMember = getUpdateAmountAtKey(currentMember, "points", earnedPoints);
  // ...
  console.log("예약 완료! ...");
}
```

`calculateFee`, `calculateEarnedPoints`는 아무 걱정 없이 테스트할 수 있다. 액션인 `makeReservation`은 여전히 주의가 필요하지만, **신뢰할 수 있는 코드(계산)가 늘어나고 다루기 어려운 코드(액션)가 줄어든다.**

## 불변성과 카피-온-라이트

함수형에서 또 하나 중요한 게 불변성이다. 원본 데이터를 절대 안 건드린다. 바꿀 일이 있으면 복사본을 만들어서 복사본을 수정한다.

```js
// 나쁜 예 — 원본을 직접 변이
function updatePoints(member, points) {
  member.points += points; // 원본이 바뀌어버림
}

// 좋은 예 — 카피-온-라이트
function updatePoints(member, points) {
  const newMember = { ...member }; // 복사
  newMember.points += points;       // 복사본만 수정
  return newMember;                  // 복사본 리턴
}
```

원본을 직접 바꾸면 그 객체를 참조하는 모든 곳에 영향이 간다. 어디서 값이 바뀌었는지 추적하기 어렵다. 카피-온-라이트를 쓰면 "이 함수가 원본을 건드렸나?" 걱정을 안 해도 된다. 리턴된 새 객체를 쓸지 말지는 호출한 쪽이 결정한다.

실제로 스터디에서 스터디룸 예약 시스템을 리팩토링했는데 (문제 진짜 개잘만듬. 유월을 온오프라인에서 몇번을 칭찬하는거지 내가지금?), 예약 취소 로직에서 `reservation.status = "cancelled"` 이렇게 직접 변이하는 코드가 있었다. 카피-온-라이트로 바꾸니까:

```js
const cancelledReservation = getUpdateValueAtKey(targetReservation, "status", "cancelled");
reservations = replaceById(reservations, reservationId, cancelledReservation);
```

원본 예약 객체는 그대로고, 새로운 예약 객체를 만들어서 배열도 새로 만든다. 코드가 좀 길어지긴 하는데, "이 함수 호출했더니 다른 데서 갑자기 값이 바뀌었어" 같은 버그가 원천 차단된다.

## 계층형 설계

함수를 분리하다 보면 자연스럽게 계층이 생긴다.

```
Layer 4 (비즈니스 로직)    makeReservation, cancelReservation — 전체 흐름 조율, 액션
Layer 3 (도메인 관여 계산)  calculateEarnedPoints, calculatePenaltyPoint, getConfirmedReservations
Layer 2 (범용 유틸)        findById, replaceById, filterByKey, calculatePercentage
Layer 1 (JS 문법)         Array.filter, spread 연산자, for, ...
```

아래로 갈수록 재사용성이 높다. `spread 연산자`는 어떤 프로젝트에서든 쓸 수 있고, `findById`는 룸이든 예약이든 아무 배열에나 쓸 수 있다. `calculateEarnedPoints`는 이 도메인을 아는 계산이고, `makeReservation`은 이 시스템에서만 의미 있는 비즈니스 로직이다.

이 구조가 왜 좋냐면, 아래 계층 함수는 위를 모른다. `findById`는 자기가 룸을 찾는 데 쓰이는지 예약을 찾는 데 쓰이는지 모른다. `Array.filter`는 `filterByKey`가 자기를 쓰는지도 모른다. 그래서 안심하고 재사용할 수 있다. 변경이 필요할 때도 영향 범위가 위에서 아래로만 흐르니까 명확하다.

## 암묵적 입력과 출력

함수를 계산으로 만드는 가장 확실한 방법은 **암묵적 입출력을 명시적으로 바꾸는 것**이다.

```js
// 암묵적 입력 — gradeConfig를 전역에서 읽음
function getGrade(hours) {
  if (hours >= gradeConfig.master.minHours) return "master";
  // ...
}

// 명시적 입력 — 인자로 받음
function getGrade(hours, gradeConfig) {
  if (hours >= gradeConfig.master.minHours) return "master";
  // ...
}
```

전역에서 읽으면 `gradeConfig`가 바뀔 때 결과가 달라질 수 있다. 인자로 받으면 같은 입력에 항상 같은 출력이 보장된다. 테스트할 때도 원하는 `gradeConfig`를 넣어볼 수 있어서 편하다.

이게 별거 아닌 것 같은데, 스터디에서 세 팀이 같은 코드를 리팩토링했더니 이 부분에서 가장 많은 논의가 나왔다. "gradeConfig가 사실상 상수인데 굳이 인자로 받아야 하나?" vs "원칙적으로 명시적이 낫다" 이런 논쟁. 결론은 팀마다 달랐는데, 이런 고민 자체가 함수형 사고의 핵심이라고 느꼈다.

## 마무리

솔직히 함수형 프로그래밍이 모든 상황에서 객체지향보다 낫다는 얘기를 하려는 건 아니다. 근데 "이 함수는 액션인가 계산인가?"를 의식하기 시작하면 코드 짜는 방식이 확실히 달라진다. 뭘 분리해야 하는지, 뭘 인자로 받아야 하는지, 뭘 리턴해야 하는지가 명확해진다.

특히 세 팀이 같은 코드를 각자 리팩토링하고 비교한 게 제일 좋았다. 같은 개념을 적용해도 팀마다 구조가 완전히 다르더라. 어떤 팀은 전역 변수를 그대로 뒀고, 어떤 팀은 아예 전역에서 빼서 인자로 흘려보냈고, 어떤 팀은 카피-온-라이트를 적용했고, 어떤 팀은 안 했고. 이 차이를 비교하면서 "왜 저쪽이 더 함수형인가?"를 토론한 게 혼자 공부할 때는 절대 못 얻는 경험이었다. 난 파라디랑 페어프로그래밍을 했는데, 파라디가 내가 고민할동안 부담없게 잘 기다려주면서 진행했었는데, 개인적으로 미션보다 재밌었다. 내가 코딩속도가 매우매우 느리기에 파라디가 답답한게 싫지만 않다면 한번 미션으로 같이 해보고싶긴하다.

파라디, 유월, 루멘, 클라우디, 도넛 모두에게 많이 배웠다... 열심히 하신다 모두

객체지향에 질리셨다면, 한번 해보시길. 생각보다 재밌다.
