---
title: sinon
date: 2018-03-19 17:31:46
tags:
---
원문: https://semaphoreci.com/community/tutorials/best-practices-for-spies-stubs-and-mocks-in-sinon-js

좀 오래된 글이지만 javascript의 test double을 도와주는 sinon.js를 이용해 mock, spy, stub에 대해 잘 설명하고 있는 글이라 번역해 봤다.



Sinon.js는 JavaScript 단위 테스트를 작성할 때 꼭 필요한 라이브러리입니다. 이 기사에서는 Sinon의 다양한 기능에 대한 Best Practice와 일반적인 사용법을 보여줍니다.

## 소개

Ajax, 네트워킹, 타임 아웃, 데이터베이스 또는 기타 종속성을 가진 코드를 테스트하는 것은 어렵습니다. 예를 들어 Ajax 또는 네트워킹을 사용하는 경우 요청에 응답하는 서버가 있어야합니다. 데이터베이스의 경우 테스트를 위한 데이터로 테스트 데이터베이스를 설정해야합니다.

이 모든 것은 테스트 작성 및 실행이 더 어려워집니다. 테스트를 성공적으로 수행 할 수있는 환경을 준비하고 설정하기 위해 추가 작업이 필요하기 때문입니다.

고맙게도 우리는 Sinon.js를 사용하여 위에 언급한 번거로움을 피할 수 있습니다. Sinon.js로 위의 상황을 몇줄의 코드로 간단히 해결할 수 있습니다.

하지만 Sinon을 시작하는 것 자체가 어려울 수 있습니다. spy, stub, mock의 형태로 많은 기능을 사용하지만, 어떤 것을 사용할지 선택하기가 어렵습니다. 또한 sinon을 사용하기 위해서는 어떤 문제를 해결해야 하는지를 명확히 파악해야 합니다.

이 기사에서는 spy, stub, mock의 차이점, 사용시기 및 사용 방법, 삽질을 피하는 데 도움이되는 모범 사례를 제공합니다.

## 예제 함수

쉽게 이해할 수 있도록 예제를 설명하는 간단한 함수가 있습니다.

```js
function setupNewUser(info, callback) {
  var user = {
    name: info.name,
    nameLowercase: info.name.toLowerCase() // 1
  };

  try {
    Database.save(user, callback); // 2
  }
  catch(err) {
    callback(err);
  }
}
```

이 함수는 두 개의 매개 변수, 즉 저장하려는 데이터가있는 객체와 콜백 함수를 인풋값으로 받습니다. info 객체의 데이터를 user 변수에 저장하고 데이터베이스에 저장합니다. 이 튜토리얼의 목적을 위해, save는 Ajax 요청을 보낼 수도 있고, Node.js 코드라면 직접 데이터베이스와 연결할 수도 있지만 구체적인 것은 중요하지 않습니다. 일종의 데이터 저장 작업을한다고 생각하면 될 것 같습니다.

## spy, stub 및 mock

spy, stub 및 mock은 테스트더블(Test Double)로 알려져 있습니다. 스턴트맨이(stunt double) 영화에서 위험한 작업을하는 것과 마찬가지로, 테스트하는데 문제가 될만한 것을 테스트더블로 대체하여 테스트를 쉽게 작성하기 위해 사용합니다.

### 테스트더블은 언제 필요한가요?

테스트더블을 언제 사용하는지 가장 잘 이해하려면 우리가 가질 수 있는 두 가지 다른 유형의 기능을 이해해야합니다. 함수를 두 가지 카테고리로 나눌 수 있습니다.

- 부작용이 없는 함수 (Functions *without* side effects)
- 부작용이 있는 함수 (And functions *with* side effects)

부작용이 없는 함수는 간단합니다. 함수의 결과는 매개 변수에만 의존합니다. 함수는 항상 동일한 매개 변수를 사용하여 동일한 값을 반환합니다. (순수함수의 경우가 부작용이 없는 함수입니다.)

부작용이 있는 함수는 어떤 개체의 상태, 현재 시간, 데이터베이스 호출 또는 일종의 상태를 유지하는 다른 메커니즘과 같은 외부의 것에 의존하는 함수로 정의 할 수 있습니다. 이러한 함수의 결과는 매개 변수 외에도 다양한 요인에 의해 영향을 받을 수 있습니다.

예제 함수를 살펴보면 toLowerCase와 Database.save라는 두 함수가 호출됩니다.(위 코드예제에서 1번과 2번) toLowerCase는 부작용이 없습니다. toLowerCase의 결과는 문자열의 값에만 의존합니다. 그러나 Database.save는 부작용이 있습니다. 이전에 언급했듯이, 일종의 저장 작업을 수행하므로 Database.save의 결과는 해당 동작의 영향을받습니다.

setupNewUser를 테스트하려면 Database.save에 부작용이 있으므로 테스트더블을 사용해야합니다. 즉, 함수에 부작용이있을 때 테스트더블이 필요하다고 말할 수 있습니다.

부작용이있는 함수 외에도 테스트에서 문제를 일으키는 함수에서 테스트더블이 필요할 수도 있습니다. 일반적인 경우는 함수가 계산을하거나 다른 작업을 수행하여 속도가 느리고 테스트가 느려지는 경우입니다. 그러나 우리는 주로 부작용이있는 함수를 다루기 위해 테스트더블을 필요로합니다.

## 언제 spy를 사용할까요?

이름에서 알 수 있듯이 **spy는 함수 호출에 대한 정보를 얻는 데 사용** 됩니다. 예를 들어, **spy는 함수가 얼마나 많이 호출되었는지, 각각 어떤 인자를 가지고 호출되었는지, 어떤 값을 리턴하는지, 어떤 에러를 던지는지** 등을 정의할 수 있습니다.

따라서 spy는 테스트의 목적이 어떤 일이 발생했는지(어떤함수가 호출되었는지)를 확인할 때 사용하기에 적합한 선택입니다. Sinon의 assertion과 결합하여 여러 가지 결과를 확인할 수 있습니다.

spy와 관련된 가장 일반적인 시나리오는 다음과 같습니다.

- 함수가 얼마나 많이 호출되었는지 확인
- 함수에 어떤 인수가 전달되었는지 확인

sinon.assert.callCount, sinon.assert.calledOnce, sinon.assert.notCalled 등을 사용하여 **함수가 호출 된 횟수를 확인할 수 있습니다.** 예를 들어 save 함수가 호출되었음을 확인하는 방법은 다음과 같습니다.

```js
it('should call save once', function() {
  var save = sinon.spy(Database, 'save');

  setupNewUser({ name: 'test' }, function() { });

  save.restore();
  sinon.assert.calledOnce(save);
});
```

sinon.assert.calledWith를 사용하여 **함수에 전달 된 인수를 확인**하거나 spy.lastCall 또는 spy.getCall ()을 사용하여 직접 호출에 액세스 할 수 있습니다. 예를 들어 앞서 언급 한 Database의 save 함수가 올바른 매개 변수를 받는지 확인하려면 아래와 같이 사용합니다.

```js
it('should pass object with correct values to save', function() {
  var save = sinon.spy(Database, 'save');
  var info = { name: 'test' };
  var expectedUser = {
    name: info.name,
    nameLowercase: info.name.toLowerCase()
  };

  setupNewUser(info, function() { });

  save.restore();
  sinon.assert.calledWith(save, expectedUser);
});
```

spy로 확인할 수 있는 것은 이것 뿐만이 아닙니다 - Sinon은 다양한 다른 것들을 검사하는데 사용할 수있는 많은 다른 assertion을 제공합니다. stub에서도 동일한 assertion을 사용할 수 있습니다.

**함수를 spy하면 함수의 동작에는 영향을 미치지 않습니다**. 함수의 동작 방식을 변경하려면 stub이 필요합니다.

## 언제 stub을 사용할까요?

stub은 타겟 함수를 대체한다는 점을 제외하면 spy와 같습니다. 또한 값을 반환하거나 예외를 발생시키는 등의 행위를 포함 할 수 있습니다. 입력한 매개변수에 따라 다른 callback을 호출하는 것도 가능합니다.

stub에는 몇 가지 일반적인 용도가 있습니다.

- 코드를 사용하여 문제가 있는(테스트가 어려운) **코드 조각을 대체** 할 수 있습니다.
- 오류 처리와 같이 트리거하지 않는 코드 경로를 트리거하는 데 사용할 수 있습니다.
- 비동기 코드를 쉽게 테스트 할 수 있습니다.

**stub은 문제가되는 코드, 즉 테스트를 어렵게 만드는 코드를 대체하는 데 사용할 수 있습니다.** stub은 네트워크 연결, 데이터베이스 또는 기타 비자바스크립트 시스템과 같은 외부적인 요인에 의해 발생합니다. 이 문제는 종종 수동 설정이 필요하다는 것을 의미합니다. 예를 들어, 테스트를 실행하기 전에 데이터베이스에 테스트 데이터를 넣어주어야 하기 때문에 실행 및 쓰기가 더 복잡해집니다.

문제가 되는 코드 조각을 대신 stub을 사용하면 이러한 문제를 완전히 피할 수 있습니다. 이전 예제에서는 Database.save를 사용합니다. Database.save는 테스트를 실행하기 전에 데이터베이스를 설정하지 않으면 문제가 될 수 있습니다. 따라서 spy 대신 stub을 사용하는 것이 좋습니다.

```js
it('should pass object with correct values to save', function() {
  var save = sinon.stub(Database, 'save');
  var info = { name: 'test' };
  var expectedUser = {
    name: info.name,
    nameLowercase: info.name.toLowerCase()
  };

  setupNewUser(info, function() { });

  save.restore();
  sinon.assert.calledWith(save, expectedUser);
});
```

데이터베이스 관련 기능을 stub으로 대체함으로써 더 이상 실제 데이터베이스를 테스트하지 않아도됩니다. 테스트하기 어려운 코드를 포함하는 거의 모든 상황에서 유사한 접근법을 사용할 수 있습니다.(spy와 유사하지만 spy를 사용할 경우, 실제 Database.save가 실행이되고, stub을 사용할 경우 Database.save가 실행되지 않음)

**stub은 다른 코드 경로를 트리거하는 데 사용될 수도 있습니다.** 테스트하는 코드에서 다른 함수를 호출하는 경우 비정상적인 조건에서 어떻게 작동하는지 테스트해야합니다. 일반적으로 오류가있는 경우가 많습니다. stub을 사용하여 코드에서 오류를 트리거 할 수 있습니다.(아래 예제코드의 1번)

```js
it('should pass the error into the callback if save fails', function() {
  var expectedError = new Error('oops');
  var save = sinon.stub(Database, 'save');
  save.throws(expectedError); // 1
  var callback = sinon.spy();

  setupNewUser({ name: 'foo' }, callback);

  save.restore();
  sinon.assert.calledWith(callback, expectedError);
});
```

**마지막으로 비동기 코드 테스트를 단순화하기 위해 stub을 사용할 수 있습니다.** 비동기 함수를  stubbing하면 즉시 콜백을 호출하여 테스트를 동기시키고 비동기 테스트 처리의 필요를 제거 할 수 있습니다.

```js
it('should pass the database result into the callback', function() {
  var expectedResult = { success: true };
  var save = sinon.stub(Database, 'save');
  save.yields(null, expectedResult);
  var callback = sinon.spy();

  setupNewUser({ name: 'foo' }, callback);

  save.restore();
  sinon.assert.calledWith(callback, null, expectedResult);
});
```

stub은 다양하게 설정가능하고 이것보다 훨씬 [더 많은 기능](http://sinonjs.org/releases/v4.1.3/stubs/)을 가지고 있지만 대부분 위에 설명한 기본적인 범주에서 사용합니다.

## 언제 mock을 사용할까요?

mock 객체를 사용할 때는 조심해야합니다. mock은 spy와 stub이 할 수 있는 모든 것을 할 수 있기 때문에 spy와 stub이 필요한 경우라도 무작정 mock을 사용하기 쉽습니다. 그러나 mock은 테스트 코드를 지나치게 구체적으로 만들기 때문에 깨지기 쉬운 코드를 만들기 쉽습니다. 깨지기 쉬운 코드는 코드가 변경될 때 의도치 않게 깨지는 코드를 말합니다.

mock은 stub이 필요할 때 사용하지만 stub보다 더 구체적인 검증이 필요할 때 사용합니다.

예를 들어, mock을 사용하여 더 구체적으로 데이터베이스에 저장하는 시나리오를 검증할 수 있는 방법은 아래와 같습니다.

```js
it('should pass object with correct values to save only once', function() {
  var info = { name: 'test' };
  var expectedUser = {
    name: info.name,
    nameLowercase: info.name.toLowerCase()
  };
  var database = sinon.mock(Database);
  database.expects('save').once().withArgs(expectedUser);

  setupNewUser(info, function() { });

  database.verify();
  database.restore();
});
```

mock으로 우리가 원하는 결과(expectation)를 정의합니다. 일반적으로 expectation은 assert 함수 호출의 형태로 끝납니다. mock을 사용하여 우리는 mocking된 함수에서 직접 값을 정의하고 마지막으로 verify를 호출합니다.

이 테스트에서는 once와 withArgs를 사용하여 호출 수와 주어진 인수를 검사하는 mock을 정의합니다. 만약에 stub을 사용한다면 여러 조건을 검사 할 때 코드 스멜이 될 수 있는 여러개의 assertion이 필요합니다.

mock에 대한 여러 조건을 선언 할 때 이러한 편의성 때문에 선외로 나가기는 쉽습니다. 우리는 mock을 사용하여 조건을 필요 이상으로 쉽게 만들 수 있기 때문에 테스트를 이해하기 어렵고 깨지기 쉽게 만들 수 있습니다. 또한 여러개의 assertion을 피하는 이유 중 하나이기 때문에 mock 객체를 사용할 때는 이것을 명심하십시오.

## Best Practice와 팁

spy, stub 및 mock과 관련된 일반적인 문제를 피하기 위해 Best Practice를 살펴 보도록 하겠습니다.

### 가능하면 sinon.test를 사용하십시오.

spy, stub 또는 mock 객체를 사용할 때 테스트 함수를 sinon.test로 래핑하십시오. 이를 통해 Sinon의 자동 초기화 기능을 사용할 수 있습니다. 이 기능이 없으면 테스트더블이 초기화되기 전에 테스트가 실패하면 연쇄 실패가 발생할 수 있습니다. 초기 실패로 인한 테스트 실패가 더 많습니다. 계단식 오류는 문제의 실제 소스를 쉽게 가려 낼 수 있으므로 가능한 경우 문제를 피하고 싶습니다.

sinon.test를 사용하면 연쇄 실패 사례가 제거됩니다. 앞서 작성한 테스트 중 하나는 다음과 같습니다.

```js
it('should call save once', function() {
  var save = sinon.spy(Database, 'save');

  setupNewUser({ name: 'test' }, function() { });

  save.restore();
  sinon.assert.calledOnce(save);
});
```

setupNewUser가 이 테스트에서 예외를 던지면 스파이가 초기화되지 않을 것이며 다음 테스트에서 큰 피해를 입힐 것입니다.

다음과 같이 sinon.test를 사용하면이 문제를 피할 수 있습니다.

```js
it('should call save once', sinon.test(function() {
  var save = this.spy(Database, 'save');

  setupNewUser({ name: 'test' }, function() { });

  sinon.assert.calledOnce(save);
}));
```

세 가지 차이점에 유의하십시오. 첫 번째 줄에서는 테스트 함수를 sinon.test로 래핑합니다. 두 번째 줄에서는 sinon.spy 대신 this.spy를 사용합니다. 마지막으로 자동으로 정리되기 때문에 save.restore 호출을 제거했습니다.

세 가지 테스트 더블 모두를 사용하여이 메커니즘을 사용할 수 있습니다.

- sinon.spy 는 this.spy
- sinon.stub 는 this.stub
- sinon.mock 는 this.mock

## sinon.test로 비동기 테스트

sinon.test를 사용할 때 비동기 테스트를 위해 fake timer를 비활성화해야 할 수 있습니다. 이것은 mocha의 비동기 테스트를 sinon.test와 함께 사용할 때 발생할 수있는 혼란의 원인입니다.

모카와 비동기 테스트를하기 위해 테스트 함수에 추가 매개 변수를 추가 할 수 있습니다.

```js
it('should do something async', function(done) {
})
```

sinon.test와 결합하면, 이것은 깨질 수 있습니다:

```js
it('should do something async', sinon.test(function(done) {
})
```

이것들을 조합하면 명백한 이유없이 테스트가 실패하고 테스트 시간 초과에 대한 메시지가 표시 될 수 있습니다. 이것은 Sinon의 fake timer 때문에 발생합니다. 이 타이머는 기본적으로 sinon.test로 래핑 된 테스트에 사용할 수 있으므로 사용하지 않도록 설정해야합니다.

이 문제는 테스트 코드 또는 테스트와 함께로드 된 구성 파일의 어딘가에서 sinon.config를 변경하여 해결할 수 있습니다.

```js
sinon.config = {
  useFakeTimers: false
};
```

sinon.config는 sinon.test와 같은 일부 함수의 기본 동작을 제어합니다. 또한 다른 옵션도 있습니다.

### beforeEach에 공유 스텁 만들기

모든 테스트에서 특정 기능을 stub으로 대체해야하는 경우 beforeEach에서 스텁하는 것이 좋습니다. 예를 들어, 모든 테스트에서 Database.save에 대한 Test Double을 사용 했으므로 다음을 수행 할 수 있습니다.

```js
describe('Something', function() {
  var save;
  beforeEach(function() {
    save = sinon.stub(Database, 'save');
  });

  afterEach(function() {
    save.restore();
  });

  it('should do something', function() {
    //you can use the stub in tests by accessing the variable
    save.yields('something');
  });
});
```

또한 afterEach를 추가하고 스텁을 정리하십시오. 그것 없이는 stub가 제 자리에 남아있을 수 있으며 다른 테스트에서 문제를 일으킬 수 있습니다.

### 설정된 함수 호출 또는 값의 순서 확인

특정 함수가 순서대로 호출되는지 확인해야하는 경우, spy 또는 stub을 sinon.assert.callOrder와 함께 사용할 수 있습니다.

```js
var a = sinon.spy();
var b = sinon.spy();

a();
b();

sinon.assert.callOrder(a, b);
```

함수가 호출되기 전에 특정 값이 설정되어 있는지 확인해야하는 경우 stub의 세 번째 매개 변수를 사용하여 스텁에 assertion을 삽입 할 수 있습니다.

```js
var object = { };
var expectedValue = 'something';
var func = sinon.stub(example, 'func', function() {
  assert.equal(object.value, expectedValue);
});

doSomethingWithObject(object);

sinon.assert.calledOnce(func);
```

stub 내의 assertion은 stub 된 함수가 호출되기 전에 값이 올바르게 설정되도록 합니다. stub이 호출되는지 확인하려면 sinon.assert.calledOnce 검사도 포함해야합니다. 그것 없이는 stub이 호출되지 않을 때 테스트가 실패하지 않습니다.

## 결론

Sinon은 강력한 도구이며, 이 자습서에 설명 된 방법을 따르면 개발자가 사용하는 가장 일반적인 문제를 피할 수 있습니다. 기억해야 할 가장 중요한 점은 sinon.test를 사용하는 것입니다. 그렇지 않으면 연쇄오류가 좌절의 큰 원인이 될 수 있습니다.



## 참고

sinon.js 공식 홈페이지 : [http://sinonjs.org/](http://sinonjs.org/)

