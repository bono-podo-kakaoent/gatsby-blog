---
title: '[javascript] async, await를 사용하여 비동기 javascript를 동기식으로 만들자'
date: 2018-02-27 21:56:31
category: javascript
tags:
  - ES8
  - javascript
  - setTimeout
  - promise
  - callback
  - async & await
  - 이터레이터
  - 제너레이터
---

async, await 는 ES8(ECMAScript2017)의 공식 스펙([링크](https://developer.mozilla.org/ko/docs/Web/JavaScript/Reference/Statements/async_function))으로 비교적 최근에 정의된 문법입니다. `async, await`를 사용하면 비동기 코드를 작성할 때 비교적 쉽고 명확하게 코드를 작성할 수 있습니다. 자바스크립트는 싱글 스레드 프로그래밍언어기 때문에 비동기처리가 필수적입니다. 비동기 처리는 그 결과가 언제 반환될지 알수 없기 때문에 동기식으로 처리하는 기법들이 사용되어야 합니다. 대표적으로 `setTimeout`이 있고 `callback`과 `promise`가 있습니다. 세 가지 모두 비동기 코드를 동기식으로 작성하는데 훌륭한 기법들이지만, 모두 약간의 문제점을 가지고 있습니다. async 와 await 는 이런 문제들을 해결함과 동시에 그 사용법에 있어서도 훨씬 단순해졌습니다. 각각의 방식들을 살펴본 뒤 async, await 를 어떻게 사용하고 어떤 방식으로 구현되어 있는지 알아보도록 하겠습니다.

### setTimeout

setTimeout 은 특정 시간 동안 기다렸다가 이후 첫번째 파라미터의 함수를 실행하는 방식을 사용합니다.

```javascript
let first = 10
let second = 20
let result = 0

function add(x, y) {
  return x + y
}

setTimeout(function() {
  result = add(first, second)
  console.log(result) // 30
}, 1000)
```

위 코드는 1 초 후에 10 과 20 을 더해서 result 에 30 을 할당하는 간단한 setTimeout 예제입니다. setTimeout 함수의 첫번째 파라미터는 실행될 함수이고, 두번째 파라미터는 첫번째 파라미터가 얼마후(ms)에 실행될지를 결정합니다. 여기에 별 문제는 없어 보이지만 비동기에 대한 이해가 부족한 상황에서 더 복잡한 코드를 작성하다가는 큰 문제에 부딪칠수도 있습니다. 조금 수정된 코드입니다.

```javascript
let first = 10
let second = 20
let result = 0

function add(x, y) {
  return x + y
}

setTimeout(function() {
  result = add(first, second)
  console.log(result) // 40
}, 1000)

first = 20
```

이 코드가 동기식으로 처리된다면 result 가 30 이겠지만, 실제로 console 에 찍히는 값은 40 입니다. 어디가 잘못 되었을까요?

자바스크립트는 각각의 `task`를 큐에 적재해두고 순서대로 처리합니다. 이 때 어떤 코드가 새로운 태스크로 적재되지에 대한 이해가 부족하면 위와 같은 실수를 저지를 수 있습니다. 최초의 task 는 스크립트 파일 자체입니다. 이 첫번째 task 내에 `setTimeout은 별도의 task를 생성`하고 첫번째 task 가 종료되길 기다립니다. 첫번째 task 인 스크립트의 실행이 끝나면 비로소 setTimeout 의 함수를 실행할 준비를 합니다. 즉 first 의 값은 초기에 10 이였지만 첫번째 스크립트가 종료되면 20 이 되기때문에 결과적으로 result 는 40 이 됩니다. task 에 대한 이해가 부족하다면 지난번에 번역했던 [Tasks, microtasks, queues and schedules](https://dev-bono.github.io/2018/01/28/tasks-microtasks-queues-and-schedules/)[(원본)](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/)를 참고하세요.

그럼 이 코드를 동기식으로 처리하려면 어떻게 해야할까요?

```javascript
let first = 10
let second = 20
let result = 0

function add(x, y) {
  return x + y
}
function getResult(callback) {
  setTimeout(function() {
    result = add(first, second)
    console.log(result) // 30
    callback()
  }, 1000)
}

getResult(function() {
  first = 20
})
```

위와 같이 callback 함수를 사용하면 비동기 코드를 동기식으로 작성할 수 있습니다. 그렇다면 이제 비동기 코드를 동기식으로 바꾸기 위해 사용하는 callback 이 무엇이고 어떻게 사용하는지 알아보도록 하겠습니다.

### callback

callback 함수란 호출하는 함수(calling function)가 호출되는 함수(called 함수)로 전달하는 함수를 말하며 이때 callback 함수의 제어권은 호출되는 함수에게 있습니다. callback 함수는 setTimeout 함수와 같은 비동기 코드를 동기식으로 처리하기 위해 사용합니다. production 에 사용되는 코드에서는 보통 네트워크 요청 등의 비동기 코드에 많이 사용됩니다.

callback 이 직관적(하나만 사용했을 때)이고 이해가 어렵지는 않지만, 여러개의 callback 을 연달아 사용하게 되면 에러가 발생할 가능성이 높고, 코드의 가독성도 크게 떨어지게 됩니다.

```javascript
// 각 함수는 비동기로 처리되는 로직이라 가정합니다
function goWork(time1, timeStartWork) {
  wakeUp(time1, function (time2) {
    takeSubway(time2, function(time3) {
      takeOffSubway(time3, function(time4) {
        arriveWork(time4, function(arrivalTime) {
          if (arrivalTime > timeStartWork) {
            fire();
          }
        }
      }
    }
  }
}
```

callback 은 비동기 코드를 동기적 만드는데 확실한 방법이긴 하지만 남발하게 되면 가독성이 크게 떨어지고 코드의 복잡성도 크게 증가하게 됩니다. 또한 callback 의 호출에 대한 제어권이 다른 함수들에게 넘어가 버리기 때문에 각 콜백함수가 언제 어떻게 몇번 실행되는지 확신을 할 수 없습니다. 이렇게 코드를 작성하면(물론 잘 하면 상관없지만), 특히 여러명이서 코드를 공유하는 경우라면 결과를 예측하기 어려울 뿐 아니라 코드내에서 에러가 발생할 확률도 높아집니다.

### promise

promise 는 약속입니다. 어떤 작업이 성공했을 때(resolve), promise 객체의 then() 함수에 넘겨진 파라미터(함수)를 단 한번만 호출하겠다는 약속입니다. callback 의 경우 제어권이 호출되는 함수로 넘어가 버리기 때문에 신뢰성이 다소 떨어지지만 promise 는 함수 실행이 성공했을때 then() 함수의 파라미터(함수)가 단 한번만 호출되기 때문에 함수를 호출하는 입장에서 확신을 가지고 코드를 작성할 수 있습니다. 또한 실패했을 경우(reject)에도 catch()함수를 통해서 실패 이후의 작업을 처리할 수 있습니다. 위의 함수(goWork)를 promise 로 바꾸어보겠습니다.

```javascript
function goWork(time1, timeStartWork) {
  return wakeUp(time1).then(function(time2) {
      return tackSubway(time2);
    }).then(function(time3) {
      return takeOffSubway(time3);
    }).then(function(time4) {
      return arriveWork(time4) {
    }).then(function(arrivalTime) {
      if (arrivalTime > timeStartWork) {
        fire();
      }
    });
}
```

callback 보다는 훨씬 덜 복잡해 보입니다. 여기에다 ES6 의 arrow function 문법을 적용하면 훨씬 더 간단해집니다.

```javascript
function goWork(time1, timeStartWork) {
  return wakeUp(time1)
    .then(time2 => tackSubway(time2))
    .then(time3 => takeOffSubway(time3))
    .then(time4 => arriveWork(time4))
    .then(arrivalTime => {
      if (arrivalTime > timeStartWork) {
        fire()
      }
    })
}
```

promise 는 충분히 깔끔하고 완성되어 보이지만, 사실 완전히 만족스럽지 않습니다. C 나 Java 와 같은 절차적 언어에서 사용하듯이 단순하고 직관적이면 더 좋겠다는 생각이 드네요.
그럼 이제 async 와 await 가 등장해야할 시간입니다.

### async & await

async 와 await 는 절차적 언어에서 작성하는 코드와 같이 사용법도 간단하고 이해하기도 쉽습니다. function 키워드 앞에 `async`만 붙여주면 되고 비동기로 처리되는 부분 앞에 `await`만 붙여주면 됩니다. 다만, 몇 가지 주의할 점이 있다면 await 뒷부분이 반드시 promise 를 반환해야 한다는 것과 async function 자체도 promise 를 반환한다는 것입니다. 그럼 사용법을 먼저 살펴보겠습니다.

```javascript
async function goWork(time1, timeStartWork) {
  const time2 = await wakeUp(time1)
  const time3 = await takeSubway(time2)
  const time4 = await takeOffSubway(time3)
  const arrivalTime = await arriveWork(time4)
  if (arrivalTime > timeStartWork) {
    fire()
  }
}
```

우와, promise 에 비해 훨씬 직관적입니다. 사용법도 그다지 어렵지 않고 코드 이해도 훨씬 좋아졌습니다. 그렇다면 간단하게 function 앞에 async 를 붙여주고 호출하는 함수앞에 await 를 붙여준 것만으로 어떻게 비동기 함수들의 동기처리가 가능해진걸까요? async 와 await 가 어떻게 동작하는지 알아보기 위해 babel 의 도움을 좀 받아야겠습니다. async, await 는 ES8 스펙이라 몇몇 브라우저에서는 호환되지 않게 때문에(최신 크롬은 됩니다) 기존 브라우저에서 동작하도록 자바스크립트 코드를 변환해 주어야 합니다. [babel 변환 코드](https://babeljs.io/repl/#?babili=false&browsers=&build=&builtIns=false&code_lz=IYZwngdgxgBAZgV2gFwJYHsIwOboOroBOA1gBRoC2ApgIwA0MlVAyssIcgSQJQwDeAKBgwomEMkapqAJhgBeGMADuwVBJXEqAVQAO5KbW4BuISLESmAZnmKVaxsE3MEAIxVh9M46dERxk6gAWG2VVC0cqAHk4OGc3YA8rb2Fff3ZCVAA3YAAbABUDELsJdKyqLjImQOSYVDgYUlLs_MKAPgCWNg4K3kFhYThUQipSGoBfATGgA&debug=false&forceAllTransforms=false&shippedProposals=false&circleciRepo=&evaluate=false&fileSize=false&lineWrap=true&presets=es2015%2Creact%2Cstage-2&prettier=true&targets=&version=6.26.0&envVersion=).

두 부분으로 나누어 간단히 살펴보겠습니다.
(아래 설명들은 정리가 잘 안되있어 이해하기 어렵습니다. 설명을 잘 못하는거 보니 아직 저도 잘 이해 못하고 있는 부분이 있는것 같네요ㅠㅠ)

```javascript
var goWork = (function() {
  var _ref = _asyncToGenerator(
    regeneratorRuntime.mark(function _callee(
      time1,
      timeStartWork
    ) {
      var time2, time3, time4, arrivalTime;
      return regeneratorRuntime.wrap(
        function _callee$(_context) {
          while (1) {
            switch ((_context.prev = _context.next)) {
              case 0:
                _context.next = 2;
                return wakeUp(time1);
              case 2:
                time2 = _context.sent;
                _context.next = 5;
                return takeSubway(time2);
              ...
              case 13:
              case "end":
                return _context.stop();
            }
          }
        },
        _callee,
        this
      );
    })
  );

  return function goWork(_x, _x2) {
    return _ref.apply(this, arguments);
  };
})();
```

첫번째로 위쪽의 코드는 goWork 를 즉시 실행하는 함수입니다. 즉시실해함수기 때문에 선언과 동시에 함수가 실행됩니다. 이 함수의 결과값은(`var goWork`에 할당되는 값) 맨 아래의 `goWork(_x, _x2)` 함수입니다. 즉, 외부에서 `goWork(...)`를 호출하면 맨 아래의 goWork 함수가 호출되는것입니다.

외부에서 goWork 를 호출하면 `_ref`가 실행되는데 \_ref 는 `_asyncToGenerator` 함수가 실행된 결과(이 또한 함수)가 할당됩니다. `_asyncToGenerator` 함수는 하나의 인자(함수)를 가지는데, 이 인자는 실행되면서 내부적으로 제너레이터가 생성합니다. 이 제너레이터를 생성하고 동작을 처리하기 위해서 babel 의 [regeneratorRuntime](https://babeljs.io/docs/plugins/transform-regenerator/) 플러그인이 이용됩니다.

제너레이터가 생성되어 실행되면 이터레이터가 만들어지고 이터레이터의 next 함수로 yield 구문의 코드를 차례차례 실행하는게 보통의 제너레이터 구조입니다. 여기에서는 만들어진 이터레이터의 next 함수(step('next'))가 호출될 때 마다 context 의 위치를 변경하고(context.next), 해당 위치의 함수를 실행합니다. (실제 await 부분).

```javascript
function _asyncToGenerator(fn) {
  return function() {
    var gen = fn.apply(this, arguments)
    return new Promise(function(resolve, reject) {
      function step(key, arg) {
        try {
          var info = gen[key](arg)
          var value = info.value
        } catch (error) {
          reject(error)
          return
        }
        if (info.done) {
          resolve(value)
        } else {
          return Promise.resolve(value).then(
            function(value) {
              step('next', value)
            },
            function(err) {
              step('throw', err)
            }
          )
        }
      }
      return step('next')
    })
  }
}
```

두번째 부분입니다. 여기서는 우선 위에서 보았던 제너레이터 처리 함수인 `fn`을 실행시켜서 이터레이터(변수명은 gen 이지만 이터레이터가 맞는듯)를 만들어 두고 그 아래에서 `프로미스`를 리턴합니다. `프로미스`가 리턴된다는 것은 goWork() 함수의 리턴 타입이 Promise 가 된다는 것입니다(코드를 잘 따라가보면 goWork()의 결과값이 Promise 를 리턴하는 부분임을 알 수 있습니다). 프로미스 내부에는 `step`이라는 함수를 만들고 이 함수를 `next` 인자와 함께 호출합니다. step 에서는 인자로 받은 `key`를 이용하여 `gen[key](arg)` 형태로 함수를 호출합니다. 이때 key 가 `next`라면 이터레이터의 next()와 동일한 함수가 호출될것이고 반환값(info)은 당연히 `{value: xxx, done: false}`가 될 것입니다. info.done 이 `true`일때는 제너레이터가 종료된것이므로 resolve()를 호출합니다. 그리고 `false`인 경우에는 새로운 프로미스를 생성하고 next 함수를 실행합니다.

위의 부분과 연관지어 설명하자면, step()함수가 한 번 실행될때 마다 하나의 프로미스가 생성되고, 제너레이터의 context 위치가 변경되고 해당부분의 함수를 호출합니다.

async 와 await 는 제너레이터 구조와 거의 동일합니다. 제너레이터에서 함수옆에 붙이는 '\*' 대신 async 를 붙이고, yield 대신 await 를 사용합니다. 다만 await 함수에서는 이터레이터의 next() 호출을 프로미스의 resolve()함수가 담당합니다. 그렇기 때문에 await 함수(프로미스)가 성공(resolve)해야지만 함수의 결과값을 리턴해줄 수 있게됩니다. 이를 간단히 요약해보면 `async,await => 제너레이터,이터레이터 + 프로미스`라 할 수 있겠습니다.

이상 자바스크립트의 비동기 함수들과 동기식 처리의 발전과정에 대해 알아봤습니다. 개인의 공부 및 정리를 목적으로 작성한 글이라 전문성이 결여되어 있으며 다소 이해하기 어려운 설명들이 다수 포함되어 있습니다.

### 참고자료

- https://medium.com/@_bengarrison/javascript-es8-introducing-async-await-functions-7a471ec7de8a
- https://hyunseob.github.io/2015/08/09/async-javascript/
- http://meetup.toast.com/posts/73
- https://jicjjang.github.io/2017/02/05/promise-and-async-await/
- https://medium.com/@jooyunghan/babel%EC%9D%80-generator%EB%A5%BC-%EC%96%B4%EB%96%BB%EA%B2%8C-%EB%B0%94%EA%BE%B8%EB%82%98-c78523645cd7
