## 리액트의 렌더링

브라우저에서 렌더링이란 **웹 페이지를 화면에 그리는 과정**이다. 만약 리액트 없이 바닐라 자바스크립트만을 사용해 렌더링을 한다면, DOM을 직접 조작해야 한다.

```jsx
const app = document.getElementById('app');

app.innerHTML = `
  <p>새롭게 렌더링되는 app 태그</p>
`;
```

위 코드처럼 DOM을 직접 조작하여 요소의 내용을 변경하거나 새로운 내용을 추가해야 한다.

하지만 리액트를 사용해서 코드를 작성하면 위와 같이 DOM을 직접 조작하지 않는다. **리액트는 이러한 과정을 자동으로 처리하는 라이브러리이기** 때문이다. 그럼 리액트는 언제 렌더링을 할까? 그것은 리액트에서 **state**가 변경될때이다.

```jsx
import React from 'react';

const App = () => {
  const [number, setNumber] = React.useState(0);

  return (
    <>
      <h1>{number}</h1>
      <button
        onClick={() => {
          setNumber(number + 1);
        }}
      >
        +1
      </button>
    </>
  );
};

export default App;
```

위 코드에서 setState 함수를 통해 number 값을 변경하면, 리액트는 자동으로 변경 사항을 감지하고 DOM을 업데이트한다. 이 과정이 **렌더링**이며, 동일한 컴포넌트가 다시 렌더링 되는 것을 **리렌더링**이라고 한다. (이번 글에서는 리렌더링도 렌더링으로 표현하겠다.)

리액트는 컴포넌트 단위로 렌더링이 이루어지며, 컴포넌트는 계층적인 트리 구조를 가진다. 즉, **부모 컴포넌트가 렌더링 되면 모든 자식 컴포넌트들을 재귀적으로 렌더링 한다.**

```jsx
import React from 'react';

const Child1 = () => {
  return <div>Child1</div>;
};

const Child2 = () => {
  return <div>Child2</div>;
};

const App = () => {
  return (
    <div>
      <Child1 />
      <Child2 />
    </div>
  );
};

export default App;
```

![](https://blog.kakaocdn.net/dn/pGmgm/btsL4w6bv4C/2r3Ore0iy2Kz5FfY0KJH2K/img.png)

App 컴포넌트는 두 개의 Child 컴포넌트를 포함하고 있다. App 컴포넌트가 렌더링이 되면 Child 컴포넌트들은 모두 렌더링이 된다.

![](https://blog.kakaocdn.net/dn/rnwZt/btsL31r7I4X/42DS08r0fB3AWJS4npwDG0/img.png)

이렇게 모든 컴포넌트들이 렌더링이 되고, 렌더링이 된다는 의미는 **컴포넌트에 있는 모든 코드들이 다시 실행**된다는 의미이다.

```jsx
import React from 'react';

const Child1 = ({ value }: { value: number }) => {
  return <div>{value}</div>;
};

const Child2 = () => {
  return <div>Child2</div>;
};

const App = () => {
  const [value, value] = React.useState(0);

  return (
    <div>
      <button onClick={() => setValue(value + 1)}>+1</button>
      <Child1 value={value} />
      <Child2 />
    </div>
  );
};

export default App;
```

![](https://blog.kakaocdn.net/dn/okyE5/btsL4RPJ8C9/FAxejOEC5KIY7lTeTy0cd1/img.png)

이 코드에서 Child1은 value 값을 props로 받기 때문에, App 컴포넌트가 렌더링 될 때 함께 렌더링 되어야 한다. 하지만 Child2는 App의 상태(state)와 무관함에도 불필요하게 다시 렌더링 된다. 만약 Child2가 복잡한 UI를 포함하고 있다면 성능 저하가 발생할 수 있다.

이렇게 **부모 컴포넌트가 렌더링 되어도 자식 컴포넌트가 불필요한 렌더링을 방지할 수 있는 렌더링 최적화**를 할 수 있는 방법이 없을까?? 리액트는 렌더링을 최적화할 수 있는 여러 기능들을 제공하고, 대표적으로 **useMemo, memo, useCallback**과 같은 기능이 있다. 또한 리액트에서는 **리액트 컴파일러**를 개발하여 자동으로 렌더링 최적화를 수행하는 방안을 연구하고 있다.

이번 글에서는 useMemo, memo, useCallback을 활용한 최적화 방법과 리액트 컴파일러에 대해 살펴보겠다.

## useMemo

```jsx
const cachedValue = useMemo(calculateValue, dependencies);
```

useMemo는 **연산의 결과를 캐싱하여 불필요한 연산을 방지하는 훅**이다.

- calculateValue: 값을 계산하는 함수.
- dependencies: 해당 값이 변경될 때만 calculateValue를 다시 실행하는 의존성 배열.

예시를 통해서 useMemo가 어떻게 동작하는지 알아보겠다.

```jsx
import React from 'react';
import './App.css';

const APP = () => {
  const [value, setValue] = React.useState(0);

  const result = value;

  return (
    <div className="container">
      <div className="box">
        <div className="button-box">
          <button onClick={() => setValue(1)}>1</button>
          <button onClick={() => setValue(2)}>2</button>
          <button onClick={() => setValue(3)}>3</button>
        </div>
        <div className="result">계산 결과 : {result}</div>
      </div>
    </div>
  );
};

export default App;
```

![](https://blog.kakaocdn.net/dn/2vIQb/btsL26AQmAs/wGfIj2cPNy2L7peZunrRh1/img.gif)

위 코드는 버튼을 누르면 버튼의 값으로 value를 변경하고, vlaue를 그대로 result로 출력하는 컴포넌트이다.

여기서 value와 result를 두 개로 사용해 보겠다.

```jsx
import React from 'react';
import './App.css';

const heavyWork = (value: number) => {
  const startTime = performance.now();

  while (performance.now() - startTime < 500) {
    // 0.5초간 무거운 작업을 한다.
  }

  return value;
};

const lightWork = (value: number) => {
  return value;
};

const App = () => {
  const [heavyValue, setHeavyValue] = React.useState(0);
  const [lightValue, setLightValue] = React.useState(0);

  const heavyResult = heavyWork(heavyValue);
  const lightResult = lightWork(lightValue);

  return (
    <div className="container">
      <div className="box">
        <p>무거운 연산</p>
        <div className="button-box">
          <button onClick={() => setHeavyValue(1)}>1</button>
          <button onClick={() => setHeavyValue(2)}>2</button>
          <button onClick={() => setHeavyValue(3)}>3</button>
        </div>
        <div className="result">계산 결과 : {heavyResult}</div>
      </div>
      <div className="box">
        <p>가벼운 연산</p>
        <div className="button-box">
          <button onClick={() => setLightValue(1)}>1</button>
          <button onClick={() => setLightValue(2)}>2</button>
          <button onClick={() => setLightValue(3)}>3</button>
        </div>
        <div className="result">계산 결과 : {lightResult}</div>
      </div>
    </div>
  );
};

export default App;
```

![](https://blog.kakaocdn.net/dn/yQZNA/btsL5BZOEOB/4K9q1dAhYlduQvkpK3FFo1/img.png)

무거운 연산을 실행하는 heavyValue와 가벼운 연산을 실행하는 lightValue가 있고, 각 버튼을 누르면 다음과 같이 실행된다.

- 무거운 연산 : heavyValue값이 변경되면 heavyWork 함수를 실행시켜 0.5초 이후에 heavyResult값을 반환
- 가벼운 연산 : lightValue값이 변경되면 lightWork 함수를 실행하고, 곧바로 ligthResult값을 반환

무거운 연산을 실행시키면 값은 어떻게 변할까?

![](https://blog.kakaocdn.net/dn/cDaYYH/btsL4HT7iiL/GgFKD9h7ABT9rcAtEefVC0/img.gif)

예상과 같이 버튼을 누르고 0.5초 뒤에 계산 결과가 바뀐다.

그럼 가벼운 연산을 실행시키면 어떻게 결과가 반영될까?

![](https://blog.kakaocdn.net/dn/W3huD/btsL235h0vR/P0aoSmnNwdPBHdc7RKMoGk/img.gif)

가벼운 연산에 사용되는 ligthWork 함수는 0.5초 지연 없이 값을 바로 반환하지만, 버튼을 누르고 계산 결과가 뒤늦게 반영된다.

이와 같이 가벼운 연산도 오랜 시간이 걸리는 이유는 heavyWork 함수가 같이 실행되기 때문이다.

가벼운 연산을 클릭하면 lightValue state가 변경되기 때문에 App 컴포넌트가 렌더링 된다. 컴포넌트가 렌더링 되면서 모든 코드가 실행되고, 매 렌더링마다 heavyWork와 lightWork 함수가 모두 실행된다. 즉, **lightWork 함수만 실행하면 되는 상황이지만 heavyWork 함수도 불필요하게 실행되고 있는 상황**이다.

이와 같이 비용이 높은 연산을 생략할 수 있게 하는 기능이 useMemo이다.

```jsx
import React from 'react';
import './App.css';

const heavyWork = (value: number) => {
  const startTime = performance.now();

  while (performance.now() - startTime < 500) {
    // 0.5초간 무거운 작업을 한다.
  }

  return value;
};

const lightWork = (value: number) => {
  return value;
};

const App = () => {
  const [heavyValue, setHeavyValue] = React.useState(0);
  const [lightValue, setLightValue] = React.useState(0);

  // heavyWork 함수의 결과를 캐싱해서 사용const heavyResult = React.useMemo(() => heavyWork(heavyValue), [heavyValue])
  const lightResult = lightWork(lightValue);

  return (
    <div className="container">
      <div className="box">
        <p>무거운 연산</p>
        <div className="button-box">
          <button onClick={() => setHeavyValue(1)}>1</button>
          <button onClick={() => setHeavyValue(2)}>2</button>
          <button onClick={() => setHeavyValue(3)}>3</button>
        </div>
        <div className="result">계산 결과 : {heavyResult}</div>
      </div>
      <div className="box">
        <p>가벼운 연산</p>
        <div className="button-box">
          <button onClick={() => setLightValue(1)}>1</button>
          <button onClick={() => setLightValue(2)}>2</button>
          <button onClick={() => setLightValue(3)}>3</button>
        </div>
        <div className="result">계산 결과 : {lightResult}</div>
      </div>
    </div>
  );
};

export default App;
```

useMemo의 첫 번째 인자로 캐싱할 값을 반환하는 함수인 heavyWork 함수로 전달하고, 두 번째 인자로 캐싱된 값의 사용 여부를 결정하는 heavyValue를 전달한다.

이는 heavyValue가 변하지 않으면 heavyWork 함수는 다시 실행하지 않는다는 의미이다.

코드를 수정한 상태에서 무거운 연산과 가벼운 연산을 각각 실행하면 어떻게 변할까?

![](https://blog.kakaocdn.net/dn/nS1g0/btsL24iRfqQ/AIZjPeGvEbThlJuMkBf8c1/img.gif)

무거운 연산은 기존과 같이 결과가 늦게 반영된다. 왜냐하면 useMemo의 의존성 배열에 있는 heavyValue이 계속해서 바뀌기 때문에 첫 인자로 주어진 함수가 계속 재실행된다.

![](https://blog.kakaocdn.net/dn/c9nzTz/btsL4cfT9m6/d6Is7Y6FfcWl3XJ9nSsYl0/img.gif)

반면에 가벼운 연산은 기존과 달리 곧바로 결과가 반영된다!!! 가벼운 연산의 버튼을 누르면 heavyValue는 변하지 않기 때문에 **useMemo가 기존에 계산된 결과를 반환하고, heavyWork 함수를 실행시키지 않는다.** 이렇게 해서 useMemo를 사용해서 렌더링을 최적화할 수 있다.

## memo

```jsx
const MemoizedComponent = memo(Component, arePropsEqual?)
```

memo는 **컴포넌트의 props가 변경되지 않는 경우 렌더링을 생략할 수 있다.**

- Component : 메모이제이션할 컴포넌트
- arePropsEqual(optional) : 컴포넌트의 이전 props와 새로운 props의 두 가지 인수를 받아 props가 변화 여부를 판단하는 함수

여기서 렌더링을 생략할 수 있다고 하는 의미는 props가 변경되지 않아도 해당 컴포넌트 내부의 state나 context의 변경으로 인해 렌더링이 발생할 수 있기 때문이다.

기존의 코드를 컴포넌트로 분리해 보겠다.

```jsx
import React from 'react';
import './App.css';

const HeavyWorkComponent = ({ value }: { value: number }) => {
  const startTime = performance.now();

  while (performance.now() - startTime < 500) {
    // 0.5초간 무거운 작업을 한다.
  }

  return <div className="result">계산 결과 : {value}</div>;
};

const LightWorkComponent = ({ value }: { value: number }) => {
  return <div className="result">계산 결과 : {value}</div>;
};

const App = () => {
  const [heavyValue, setHeavyValue] = React.useState(0);
  const [lightValue, setLightValue] = React.useState(0);

  return (
    <div className="container">
      <div className="box">
        <p>무거운 연산</p>
        <div className="button-box">
          <button onClick={() => setHeavyValue(1)}>1</button>
          <button onClick={() => setHeavyValue(2)}>2</button>
          <button onClick={() => setHeavyValue(3)}>3</button>
        </div>
        <HeavyWorkComponent value={heavyValue} />
      </div>
      <div className="box">
        <p>가벼운 연산</p>
        <div className="button-box">
          <button onClick={() => setLightValue(1)}>1</button>
          <button onClick={() => setLightValue(2)}>2</button>
          <button onClick={() => setLightValue(3)}>3</button>
        </div>
        <LightWorkComponent value={lightValue} />
      </div>
    </div>
  );
};

export default App;
```

HeavyWorkComponent는 HeavyValue를 props로 받아 0.5초의 연산을 실행해서 결과로 출력하고, LightWorkComponent는 lightValue를 props로 받아 곧바로 결과로 출력한다.

이 상태에서 각 버튼을 클릭하면 다음과 같이 동작한다.

![](https://blog.kakaocdn.net/dn/k2Vv8/btsL4XI7CpJ/jecD9Zc3U8jdxrCWkzMxE1/img.gif)

![](https://blog.kakaocdn.net/dn/kQnSJ/btsL5AzSd9f/nxDvdYqaKs2iNbdV48yyq1/img.gif)

최적화가 되지 않은 상태에서 무거운 연산은 당연히 0.5초 이후에 결과가 반영된다. 그리고 가벼운 연산 또한 App 컴포넌트의 state를 변경시켜 렌더링이 발생되고, **App 컴포넌트의 자식 컴포넌트인 HeavyWorkComponent와 LightWorkComponent 모두 렌더링이 발생하게 된다.** 따라서 LightWorkComponent의 결과가 바뀌기 위해서는 HeavyWorkComponent안에 있는 코드가 같이 실행되어야 하고, 이 때문에 0.5초 뒤에 결과가 반영되는 것이다.

useMemo는 연산된 결과를 반환한다면, memo는 메모된(memoized) 컴포넌트를 반환한다.

```jsx
import React from 'react';
import './App.css';

// React.momo를 사용해서 최적화
const HeavyWorkComponent = React.memo(({ value }: { value: number }) => {
  const startTime = performance.now();

  while (performance.now() - startTime < 500) {
    // 0.5초간 무거운 작업을 한다.
  }

  return <div className="result">계산 결과 : {value}</div>;
});

const LightWorkComponent = ({ value }: { value: number }) => {
  return <div className="result">계산 결과 : {value}</div>;
};

const App = () => {
  const [heavyValue, setHeavyValue] = React.useState(0);
  const [lightValue, setLightValue] = React.useState(0);

  return (
    <div className="container">
      <div className="box">
        <p>무거운 연산</p>
        <div className="button-box">
          <button onClick={() => setHeavyValue(1)}>1</button>
          <button onClick={() => setHeavyValue(2)}>2</button>
          <button onClick={() => setHeavyValue(3)}>3</button>
        </div>
        <HeavyWorkComponent value={heavyValue} />
      </div>
      <div className="box">
        <p>가벼운 연산</p>
        <div className="button-box">
          <button onClick={() => setLightValue(1)}>1</button>
          <button onClick={() => setLightValue(2)}>2</button>
          <button onClick={() => setLightValue(3)}>3</button>
        </div>
        <LightWorkComponent value={lightValue} />
      </div>
    </div>
  );
};

export default App;
```

HeavyWorkComponent를 memo로 감싸 props가 변하지 않으면 렌더링이 되지 않게 한다.

이 상태에서 코드를 실행하면 다음과 같이 최적화된 결과를 얻을 수 있다.

![](https://blog.kakaocdn.net/dn/crvRjL/btsL3PFlmh1/p67gZXvKMmKtkWoqPugu8k/img.gif)

![](https://blog.kakaocdn.net/dn/dKpPZH/btsL23Yrf2J/KUggF6YezAGW7zVD4UFpd0/img.gif)

무거운 연산을 클릭하면 HeavyWorkComponent의 props인 heavyValue가 변경되어 전달되므로 렌더링이 다시 이루어진다. 하지만 가벼운 연산은 heavyValue가 변경되지 않기 때문에 **HeavyWorkComponent는 렌더링 하지 않고 LightWorkComponent만 렌더링 한다.**

이렇게 컴포넌트를 memo로 감싸 렌더링을 최적화할 수 있다.

## useCallback

```jsx
const cachedFn = useCallback(fn, dependencies);
```

useCallback은 **함수 정의를 캐싱하는 훅**이다.

- fn: 캐싱할 함수값
- dependencies : 해당 값이 변경될 때만 fn을 다시 정의하는 의존성 배열.

useCallback 또한 렌더링을 최적화하기 위해 사용하는 기능이다. 함수를 useCallback으로 감싸면 의존성 배열이 변하지 않으면 기존에 생성된 함수를 반환한다(함수 호출이 아님)

컴포넌트로 분리한 코드에서 work 함수를 App 컴포넌트에서 제공해 보겠다.

```jsx
import React from 'react';
import './App.css';

const HeavyWorkComponent = React.memo(
  ({ value, work }: { value: number, work: (value: number) => number }) => {
    const result = work(value);

    return <div className="result">계산 결과 : {result}</div>;
  }
);

const LightWorkComponent = ({
  value,
  work,
}: {
  value: number,
  work: (value: number) => number,
}) => {
  const result = work(value);

  return <div className="result">계산 결과 : {result}</div>;
};

const App = () => {
  const [heavyValue, setHeavyValue] = React.useState(0);
  const [lightValue, setLightValue] = React.useState(0);

  const heavyWork = (value: number) => {
    const startTime = performance.now();

    while (performance.now() - startTime < 500) {
      // 0.5초간 무거운 작업을 한다.
    }

    return value;
  };

  const lightWork = (value: number) => {
    return value;
  };

  return (
    <div className="container">
      <div className="box">
        <p>무거운 연산</p>
        <div className="button-box">
          <button onClick={() => setHeavyValue(1)}>1</button>
          <button onClick={() => setHeavyValue(2)}>2</button>
          <button onClick={() => setHeavyValue(3)}>3</button>
        </div>
        <HeavyWorkComponent value={heavyValue} work={heavyWork} />
      </div>
      <div className="box">
        <p>가벼운 연산</p>
        <div className="button-box">
          <button onClick={() => setLightValue(1)}>1</button>
          <button onClick={() => setLightValue(2)}>2</button>
          <button onClick={() => setLightValue(3)}>3</button>
        </div>
        <LightWorkComponent value={lightValue} work={lightWork} />
      </div>
    </div>
  );
};

export default App;
```

heavyWork 함수와 lightWork 함수를 App 컴포넌트 내부에서 정의하고, 이 함수를 각 컴포넌트의 props로 전달했다.

이 상태에서 코드를 실행하면 다음과 같이 동작한다.

![](https://blog.kakaocdn.net/dn/bLiTw4/btsL4ah2erE/WV8F0OGFd7AlCeJE4veYU1/img.gif)

![](https://blog.kakaocdn.net/dn/bY8IY3/btsL3YoE4lW/2lJQdbLKzGCPYMQdBrO4O0/img.gif)

각 컴포넌트를 memo로 감싸 최적화를 함에도 불구하고 가벼운 연산 결과가 0.5초 늦게 반영되고 있다. 이 이유는 App 컴포넌트가 렌더링 되면서 heavyWork 함수와 lightWork 함수가 다시 생성하기 때문이다.

자바스크립트에서 함수는 객체(Function)의 형태이다. 즉, 함수는 하나의 값으로 평가되어 변수에 할당되어 메모리에 로드된다.

![](https://blog.kakaocdn.net/dn/oxLJm/btsL3mp1Nl4/wq7WRyWY3lFkOqla6nTBFk/img.png)

이렇게 함수는 메모리에 로드된다. App 컴포넌트가 처음 실행되면 heavyWork는 0x00001F에 있는 함수이고, ligthWork는 0x003A24에 있는 함수이다. 즉, **함수의 메모리 주소가 달라지면 이는 다른 함수로 평가된다.**

App 컴포넌트가 렌더링 되면 표현식을 다시 실행시켜 함수를 새롭게 생성한다.

![](https://blog.kakaocdn.net/dn/uvmcg/btsL4nIoF9W/ZRzYgKQC6xrR9kKsWy66N1/img.png)

처음 App 컴포넌트가 실행될 때 work 함수와 렌더링 되어 새로 생성된 work 함수는 다른 메모리 주소를 가지고 있기 때문에 다른 함수이다. 그래서 App 컴포넌트가 렌더링 되면 heavyWork 함수가 변경되어 HeavyWorkComponent를 memo로 감싸고 있더라도 리액트는 props가 변경되었다 판단해 HeavyWorkComponent도 다시 렌더링 한다.

이런 경우 useCallback을 사용하여 함수가 새롭게 생성되는 것을 방지할 수 있다.

```jsx
import React from 'react';
import './App.css';

const HeavyWorkComponent = React.memo(
  ({ value, work }: { value: number, work: (value: number) => number }) => {
    const result = work(value);

    return <div className="result">계산 결과 : {result}</div>;
  }
);

const LightWorkComponent = ({
  value,
  work,
}: {
  value: number,
  work: (value: number) => number,
}) => {
  const result = work(value);

  return <div className="result">계산 결과 : {result}</div>;
};

const App = () => {
  const [heavyValue, setHeavyValue] = React.useState(0);
  const [lightValue, setLightValue] = React.useState(0);

  // useCallback을 사용해 함수가 새롭게 생성됨을 방지
  const heavyWork = React.useCallback((value: number) => {
    const startTime = performance.now();

    while (performance.now() - startTime < 500) {
      // 0.5초간 무거운 작업을 한다.
    }

    return value;
  }, []);

  const lightWork = (value: number) => {
    return value;
  };

  return (
    <div className="container">
      <div className="box">
        <p>무거운 연산</p>
        <div className="button-box">
          <button onClick={() => setHeavyValue(1)}>1</button>
          <button onClick={() => setHeavyValue(2)}>2</button>
          <button onClick={() => setHeavyValue(3)}>3</button>
        </div>
        <HeavyWorkComponent value={heavyValue} work={heavyWork} />
      </div>
      <div className="box">
        <p>가벼운 연산</p>
        <div className="button-box">
          <button onClick={() => setLightValue(1)}>1</button>
          <button onClick={() => setLightValue(2)}>2</button>
          <button onClick={() => setLightValue(3)}>3</button>
        </div>
        <LightWorkComponent value={lightValue} work={lightWork} />
      </div>
    </div>
  );
};

export default App;
```

이렇게 heavyWork 함수를 useCallback으로 감싸 App 컴포넌트가 렌더링 되더라도 함수가 생성되는것을 방지하고, 의존성 배열이 비어있기 때문에 함수가 초기 렌더링에 생성된 이후 다시 생성하지 않는다.

![](https://blog.kakaocdn.net/dn/bovRxG/btsL4FosRZI/4ma1hZLzAiYXESgbJjWzK0/img.png)

App 컴포넌트가 렌더링 되면 lightWork 함수만 새롭게 생성하고 heavyWork 함수는 최초에 생성한 함수를 캐싱해서 사용한다. 이렇게 되면 heavyWork 함수는 변화되지 않아 HeavyWorkComponent는 heavyValue만 변경되지 않으면 렌더링이 되지 않는다.

![](https://blog.kakaocdn.net/dn/bVC2Sh/btsL3jGMen0/jO2nKnPQECDv0X0R5kR8W1/img.gif)

이처럼 useCallback은 함수 자체의 실행 성능을 향상하지는 않지만, memo나 useMemo의 의존성 배열에서 불필요한 리렌더링을 줄이는 데 활용할 수 있다. 또한 useCallback은 useEffect가 계속 실행되는 것을 방지하는데도 사용된다.

```jsx
const someFn = () => React.useCallback(() => {
  // do something
}, [...]);

React.useEffect(() => {
  // do something
  const res = someFn();
}, [someFn]);
```

## React Compiler

리액트 컴파일러는 현재 개발 중인 기능으로 memo, useMemo, useCallback 없이 **렌더링을 자동으로 최적화**한다.

리액트에서 렌더링을 최적화하기 위해서는 기존처럼 useMemo, memo, useCallback을 사용해야 한다. 이는 개발자가 리액트에게 렌더링이 돼야 하는 시점을 직접 알려주는 방식이다.

리액트는 **개발자가 상태(state)를 통해 UI를 만들게 하고, 상태의 변경에 따른 렌더링은 자동으로 처리하는 라이브러리**이다. 하지만 최적화를 시키기 위해서는 개발자가 렌더링 시점에 개입을 하고 있는 상황이다. 이것은 리액트가 핵심 아이디어와 맞지 않는 방법이라고 생각한다.

그래서 리액트는 최적화까지 개발자가 개입하지 않고 UI에만 집중할 수 있게 하는 리액트 컴파일러를 준비하고 있다.

프로그래밍 언어에서 컴파일러란 고급 언어(C, Java)를 저급 언어(assembly langauge, machine code)로 번역하는 프로그램이다. 그러면 리액트 컴파일러는 무엇을 컴파일할까? 리액트 컴파일러는 리액트 코드를 최적화된 코드로 변환한다.

[React Compiler Playground](https://playground.react.dev/#N4Igzg9grgTgxgUxALhAgHgBwjALgAgBMEAzAQygBsCSoA7OXASwjvwFkBPAQU0wAoAlPmAAdNvhgJcsNgB5CTAG4A+ABIJKlCPgDqOSoTkB6RaoDc4gL7iQVoA)

해당 사이트에서 리액트 코드를 입력하면 리액트 컴파일러가 최적화한 코드를 보여준다.

```jsx
import React from 'react';

const Child = ({ value }) => {
  return <div>{value}</div>;
};

const App = () => {
  const [number, setNumber] = React.useState(0);

  return (
    <>
      <button
        onClick={() => {
          setNumber(number + 1);
        }}
      >
        +1
      </button>
      <Child value={number} />
    </>
  );
};

export default App;
```

위의 코드를 리액트 컴파일러를 통해 최적화하면 아래와 같이 변환된다.

```jsx
import { c as _c } from 'react/compiler-runtime';
import React from 'react';

const Child = (t0) => {
  const $ = _c(2);
  const { value } = t0;
  let t1;
  if ($[0] !== value) {
    t1 = <div>{value}</div>;
    $[0] = value;
    $[1] = t1;
  } else {
    t1 = $[1];
  }
  return t1;
};

const App = () => {
  const $ = _c(2);
  const [number, setNumber] = React.useState(0);
  let t0;
  if ($[0] !== number) {
    t0 = (
      <>
        <button
          onClick={() => {
            setNumber(number + 1);
          }}
        >
          +1
        </button>
        <Child value={number} />
      </>
    );
    $[0] = number;
    $[1] = t0;
  } else {
    t0 = $[1];
  }
  return t0;
};

export default App;
```

처음에는 꽤 복잡해 보일 수 있겠지만, Child 컴포넌트 부분만 분석을 해보겠다.

1. `const $ = _c(2)` : `_c`는 리액트 컴파일러에서 제공하는 훅인 useMomoChache의 별칭으로 캐싱한 값을 반환
2. `cosnt { value } = t0` : props를 구조 분해 할당
3. `let t1` : 반환할 컴포넌트를 저장하는 `t1` 변수 선언
4. `if ($[0] !== value)` : 캐싱된 어떤 값(아마도 이전 value)가 props로 제공된 value랑 같지 않다면
   1. `t1 = <div>{value}</div>` : 새로운 value를 사용한 컴포넌트 생성해서 `t1`에 저장
   2. `$[0] = value` : `$[0]`에 props 값인 value를 캐싱
   3. `$[1] = t1` : `$[1]`에 새롭게 생성한 컴포넌트를 캐싱
5. `else` : 캐싱된 value(from 4.2)와 props로 제공된 value랑 같지 않다면
   1. `t1 = $[1]` : 캐싱된 컴포넌트(from 4.3)를 t1에 저장
6. `return t0` : 컴포넌트 `t0`를 반환

컴파일된 결과를 보면 Child 컴포넌트에 memo를 씌운 결과와 비슷하게 동작을 한다. 이렇게 리액트 컴파일러는 **개발자가 최적화를 신경 쓰지 않아도 값의 변화를 추적하여 컴포넌트와 계산된 값들을 캐싱하여 불필요한 연산과 렌더링이 발생하지 않도록 한다.**

## 최적화를 적용해야 하는 경우

이번 글에서는 리액트에서 렌더링이 어떤 의미인지, 불필요한 레던더링을 줄일 수 있는 방법들에 대해서 알아보았다. 그럼 모든 연산들에서 대해서 최적화를 시키면 항상 높은 성능을 유지할 수 있을까? **특수한 경우를 제외하면 최적화를 적용하지 않는 게 더 효율적일 수 있다.**

useMemo와 useCallback은 의존성 배열을 통해 값들을 캐싱하고 memo는 props들을 캐싱한다. 그리고 매 실행마다 캐싱된 값이 변경되었는지 확인하는데, 만약 함수가 간단한 기능을 한다면 오히려 캐싱된 값을 확인하는 게 불필요한 연산일 수도 있다.

더 큰 문제점은 코드의 가독성이 저해되는 점이다. 코드를 읽는 입장에서 최적화를 위한 데이터 변경을 계속해서 추적해야 하고, 추후에 기능을 추가하는 경우에도 큰 복잡성을 야기할 수 있다.

그러면 어떤 경우부터 복잡한 연산이라 할 수 있을까? 리액트 문서에서는 수천 개의 개체를 만들거나 반복하는 경우(1ms 이상)인 경우 값을 메모하는것(memoized)이 좋다고 한다.

연산이 복잡해 불가피하게 메모이제이션이 필요한 경우가 발생하지만, 리액트에서 제공하는 원칙들을 사용해서 코드를 설계하면 메모이제이션이 불필요하게 만들 수 있다고 한다.

[useMemo – React
The library for web and native user interfaces](https://ko.react.dev/reference/react/useMemo#should-you-add-usememo-everywhere)

```jsx
import React from 'react';

const Child = ({ value }) => {
  return <div>{value}</div>;
};

const App = () => {
  const [number, setNumber] = React.useState(0);

  return (
    <>
      {/* some component */}
      <button
        onClick={() => {
          setNumber(number + 1);
        }}
      >
        +1
      </button>
      <Child value={number} />
    </>
  );
};

export default App;
```

리액트 컴파일러에 예시로 사용했던 코드를 변경하면 아래처럼 변경할 수 있다. (다른 추가적인 컴포넌트들이 있다고 가정)

```jsx
import React from 'react';

const Child = () => {
  const [number, setNumber] = React.useState(0);

  return (<>
    <button onClick={() => {
      setNumber(number + 1);
    }}>+1</button>
    <div>{number}</div>
  </>)
};

const App = () => {
  return (
    <>
      {/* some components /*}
      <Child />
    </>
  )
};

export default App;
```

자식 컴포넌트의 상태를 부모 컴포넌트로 올리지 않고, 자식 컴포넌트 안에서 처리하면 메모이제이션 없이 불필요한 렌더링을 막을 수 있다. 기존 코드는 number 상태를 변경할 때마다 App 컴포넌트가 렌더링 되어 some components들이 렌더링 되었지만, 개선된 코드에서는 number 상태가 변경하면 Child 컴포넌트만 렌더링 되고 App 컴포넌트의 다른 자식 컴포넌트들은 렌더링 되지 않는다.

## 마무리

이번 글에서는 리액트에서 렌더링이 되는 시점, 그리고 렌더링을 최적화할 수 있는 방법과 리액트 컴파일러에 대해서 알아보았다. 프로젝트를 처음 할 때 아무것도 모르고 모든 함수에 useCallback을 적용해서 코드를 작성한 적이 있는데 이게 상당히 쓸모없는 짓이었다는 거를 깨달았다...

중요한 것은 정말 필요한 경우에만 최적화를 적용해야 하고, 최적화를 적용한 경우 렌더링이 되는 시점을 정확히 파악하고 있어야 추후에 코드를 유지보수할 수 있겠다는 생각이 들었다.

리액트 컴파일러까지 살펴보면 결론은 이제 개발자가 최적화를 시키지 않으니 useMemo, memo, useCallback에 관한 개념은 몰라도 되지 않나?라는 생각이 든다. 하지만 리액트 컴파일러는 2024년 10월에 beta 버전이 출시되었고, stable release가 언제 될지는 아직 모른다.

리액트 컴파일러가 범용적으로 사용되는 시점도 아직 명확하지 않고, 설령 stable이 되었다고 해도 기존 useMemo, memo, useCallback들을 완전히 대체해서 바로 deprecated 시켜버릴지도 모르는 일이다. 리액트 컴파일러가 나와도 기존의 최적화 코드들은 같이 사용해야 할 수도 있다. 그리고 애초에 기존 레거시 코드들이 해당 기능들을 사용하고 있기에 당분간은 계속 이해하고 있어야 하는 내용이다.
