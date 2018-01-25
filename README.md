# TypeScript에서의 React와 Redux 가이드
이 가이드는 타입스크립트를 사용해서 React & Redux 앱을 작성하는 포괄적인 가이드입니다.

### 타입스크립트2.2 로드맵 (https://github.com/Microsoft/TypeScript/wiki/Roadmap)

### 목표:
- 완벽하고 안전하게 any타입에서 벗어나기.
- [Type Inference](https://www.typescriptlang.org/docs/handbook/type-inference.html)을 이용해서 수동으로 입력하는 주석의 양 최소화
- [Generics](https://github.com/piotrwitek/react-redux-typescript) 및 [Advanced Types](https://www.typescriptlang.org/docs/handbook/generics.html) 기능을 사용하여 [심플한 유틸리티 기능](https://github.com/piotrwitek/react-redux-typescript)으로 boilerplate를 제작.

### 목차
- [React](#react)
  - [Stateless Component](#stateless-component)
  - [Class Component](#class-component)
  - [Higher-Order Component](#higher-order-component)
  - [Redux Connected Component](#redux-connected-component)
- [Redux](#redux)
  - [Actions](#actions)
  - [Reducers](#reducers)
  - [Store types](#store-types)
  - [Create Store](#create-store)
- [Ecosystem](#ecosystem)
  - [Async Flow with "redux-observable"](#async-flow-with-redux-observable)
  - [Selectors with "reselect"](#selectors-with-reselect)
  - Forms with "formstate" WIP
  - Styles with "typestyle" WIP
- [Extras](#extras)
  - [tsconfig.json](#tsconfigjson)
  - [tslint.json](#tslintjson)
  - [Default and Named Module Exports](#default-and-named-module-exports)
  - [Vendor Types Augumentation](#vendor-types-augmentation)
- [FAQ](#faq)
- [Project Examples](#project-examples)

---

# React

---

## Stateless Component
- state가 없는 component 샘플 코드 (멈청한 컴포넌트 글래스)
```tsx
import * as React from 'react';

type Props = {
  className?: string,
  style?: React.CSSProperties,
};

const MyComponent: React.StatelessComponent<Props> = (props) => {
  const { children, ...restProps } = props;
  return (
    <div {...restProps} >
      {children}
    </div>
  );
};

export default MyComponent;
```

---

## Class Component
- class component 샘플 코드 (일반적인 컴포넌트 클래스)
```tsx
import * as React from 'react';

type Props = {
  className?: string,
  style?: React.CSSProperties,
  initialCount?: number,
};

type State = {
  counter: number,
};

class MyComponent extends React.Component<Props, State> {
  // Property Initializers를 이용하여 Props의 기본값을 지정하는 부분
  static defaultProps: Partial<Props> = {
    className: 'default-class',
  };

  // Property Initializers를 이용하여 State의 기본값을 지정하는 부분
  state: State = {
    counter: this.props.initialCount || 0,
  };

  // 라이프 사이클 메소드는 일반적인 인스턴스 메소드로 선언되어야하며 아무 문제 없습니다.
  componentDidMount() {
    // tslint:disable-next-line:no-console
    console.log('Mounted!');
  }

  // 화살표 함수가있는 클래스 필드를 사용하는 핸들러
  increaseCounter = () => {
    this.setState({
      counter: this.state.counter + 1
    });
  };

  render() {
    const { children, initialCount, ...restProps } = this.props;

    return (
      <div {...restProps} onClick={this.increaseCounter} >
        Clicks: {this.state.counter}
        <hr />
        {children}
      </div>
    );
  }
}

export default MyComponent;
```

---

## Higher-Order Component
- 새로운 Component를 반환하는 Component랩핑 또는 decorate를 생성합니다.
- 새로운 Component는 'HOC'의 Props와 함께 확장된 입력 composition을 통해 props인터페이스를 상속받습니다.
- Type Inference을 이용해서 결과적인 Props 인터페이스를 자동으로 계산합니다.
- decorate Props를 필터링 하고 관련된 Props 만 랩핑된 Component에 전달합니다.
- stateless functional 또는 규칙적인 Component 포함해서 제작합니다.

```tsx
// controls/button.tsx
import * as React from 'react';
import { Button } from 'antd';

type Props = {
  className?: string,
  autoFocus?: boolean,
  htmlType?: typeof Button.prototype.props.htmlType,
  type?: typeof Button.prototype.props.type,
};

const ButtonControl: React.StatelessComponent<Props> = (props) => {
  const { children, ...restProps } = props;

  return (
    <Button {...restProps} >
      {children}
    </Button>
  );
};

export default ButtonControl;
```

```tsx
// decorators/with-form-item.tsx
import * as React from 'react';
import { Form } from 'antd';
const FormItem = Form.Item;

type BaseProps = {
};

type HOCProps = FormItemProps & {
  error?: string;
};

type FormItemProps = {
  label?: typeof FormItem.prototype.props.label;
  labelCol?: typeof FormItem.prototype.props.labelCol;
  wrapperCol?: typeof FormItem.prototype.props.wrapperCol;
  required?: typeof FormItem.prototype.props.required;
  help?: typeof FormItem.prototype.props.help;
  validateStatus?: typeof FormItem.prototype.props.validateStatus;
  colon?: typeof FormItem.prototype.props.colon;
};

export function withFormItem<WrappedComponentProps extends BaseProps>(
  WrappedComponent:
    React.StatelessComponent<WrappedComponentProps> | React.ComponentClass<WrappedComponentProps>,
) {
  const HOC: React.StatelessComponent<HOCProps & WrappedComponentProps> =
    (props) => {
      const {
        label, labelCol, wrapperCol, required, help, validateStatus, colon,
        error, ...passThroughProps,
      } = props as HOCProps;

      // 빈 decorate props를 필터링
      const formItemProps: FormItemProps = Object.entries({
        label, labelCol, wrapperCol, required, help, validateStatus, colon,
      }).reduce((definedProps: any, [key, value]) => {
        if (value !== undefined) { definedProps[key] = value; }
        return definedProps;
      }, {});

      // 조건에 따라 추가적 props을 주입(injecting)
      if (error) {
        formItemProps.help = error;
        formItemProps.validateStatus = 'error';
      }

      return (
        <FormItem {...formItemProps} hasFeedback={true} >
          <WrappedComponent {...passThroughProps as any} />
        </FormItem>
      );
    };

  return HOC;
}
```

```tsx
// components/consumer-component.tsx
...
import { Button, Input } from '../controls';
import { withFormItem, withFieldState } from '../decorators';

// decorator를 사용해서 좀 더 전문화된 component를 만들수 있습니다.
const ButtonField = withFormItem(Button);

// function composition를 활용하면 여러개의 decorator를 생성할수 있습니다.
const InputFieldWithState = withFormItem(withFieldState(Input));

// 향상된 component는 HOC가 적용된 컴포넌트에서 props type을 상속받습니다.
<ButtonField type="primary" htmlType="submit" wrapperCol={{ offset: 4, span: 12 }} autoFocus={true} >
  Next Step
</ButtonField>
...
<InputFieldWithState {...formFieldLayout} label="Type" required={true} autoFocus={true}
  fieldState={configurationTypeFieldState} error={configurationTypeFieldState.error}
/>
...

// 필요에 따라 ramda 또는 lodash와 같은 기능적 라이브러리를 사용할 수 있습니다.
const InputFieldWithState = compose(withFormItem, withFieldStateInput)(Input);
// 참고 : compose 함수는 type declarations이 필요하거나 type inference를 잃을 수 있습니다
```

---

## Redux Connected Component
> 참고: connect함수의 type inference는 완전한 형태의 안전성을 제공하지 않고 자동으로 [`Higher-Order Component`] (#higher-order-component) 위의 예에서와 같이 소품 인터페이스를 결과 계산하는 타입 추론을 활용하지 않을 것입니다..

> 위의 문제는 내가 해결해보려고 한 문제입니다.

- 이 해결책은 type inference을 사용하여`mapStateToProps` 함수로부터 Props 타입을 얻습니다.
- `connect` 도우미 함수로부터 주입 된 Props 유형을 선언하고 유지하기위한 수작업을 최소화합니다.
- TypeScript가 아직 이 기능을 지원하지 않기 때문에 `returntypeof()` 헬퍼 함수를 사용합니다. (https://github.com/piotrwitek/react-redux-typescript#returntypeof-polyfill)
- Real project example:  https://github.com/piotrwitek/react-redux-typescript-starter-kit/blob/ef2cf6b5a2e71c55e18ed1e250b8f7cadea8f965/src/containers/currency-converter-container/index.tsx

```tsx
import { returntypeof } from 'react-redux-typescript';

import { RootState } from '../../store/types';
import { increaseCounter, changeBaseCurrency } from '../../store/action-creators';
import { getCurrencies } from '../../store/state/currency-rates/selectors';

const mapStateToProps = (rootState: RootState) => ({
  counter: rootState.counter,
  baseCurrency: rootState.baseCurrency,
  currencies: getCurrencies(rootState),
});
const dispatchToProps = {
  increaseCounter: increaseCounter,
  changeBaseCurrency: changeBaseCurrency,
};

// mapStateToProps 및 dispatchToProps에서 유추 된 Props types
const stateProps = returntypeof(mapStateToProps);
type Props = typeof stateProps & typeof dispatchToProps;

class CurrencyConverterContainer extends React.Component<Props, {}> {
  handleInputBlur = (ev: React.FocusEvent<HTMLInputElement>) => {
    const intValue = parseInt(ev.currentTarget.value, 10);
    this.props.increaseCounter(intValue); // number
  }

  handleSelectChange = (ev: React.ChangeEvent<HTMLSelectElement>) => {
    this.props.changeBaseCurrency(ev.target.value); // string
  }

  render() {
    const { counter, baseCurrency, currencies } = this.props; // number, string, string[]

    return (
      <section>
        <input type="text" value={counter} onBlur={handleInputBlur} ... />
        <select value={baseCurrency} onChange={handleSelectChange} ... >
          {currencies.map(currency =>
            <option key={currency}>{currency}</option>
          )}
        </select>
        ...
      </section>
    );
  }
}

export default connect(mapStateToProps, dispatchToProps)(CurrencyConverterContainer);
```


---

# Redux

---

## Actions

- ### KISS Style (Keep it short and simple)
이 전근법에서 저는 KISS 원칙을 준수했고, 많은 TypeScript Redux 가이드에서 흔히 볼 수있는 독점적 인 추상화를 벗어나, 익숙한 자바스크립트 사용에 최대한 가깝지만 static types의 이점을 얻으려 했습니다. 그 방법으로는,

- 전통적인 const 타입을 사용
- 표준에 가까운 JS 사용
- 표준 boilerplate
- `redux-saga` 또는`redux-observable`과 같은 다른 모듈에서 재사용하기 위해 Action Types과 Action Creators를 내 보내야합니다.

**DEMO:** [TypeScript Playground](https://www.typescriptlang.org/play/index.html#src=declare%20const%20store%3A%20any%3B%0D%0A%0D%0A%2F%2F%20Action%20Types%0D%0Aexport%20const%20INCREASE_COUNTER%20%3D%20'INCREASE_COUNTER'%3B%0D%0Aexport%20const%20CHANGE_BASE_CURRENCY%20%3D%20'CHANGE_BASE_CURRENCY'%3B%0D%0A%0D%0A%2F%2F%20Action%20Creators%0D%0Aexport%20const%20actionCreators%20%3D%20%7B%0D%0A%20%20increaseCounter%3A%20()%20%3D%3E%20(%7B%0D%0A%20%20%20%20type%3A%20INCREASE_COUNTER%20as%20typeof%20INCREASE_COUNTER%2C%0D%0A%20%20%7D)%2C%0D%0A%20%20changeBaseCurrency%3A%20(payload%3A%20string)%20%3D%3E%20(%7B%0D%0A%20%20%20%20type%3A%20CHANGE_BASE_CURRENCY%20as%20typeof%20CHANGE_BASE_CURRENCY%2C%20payload%2C%0D%0A%20%20%7D)%2C%0D%0A%7D%0D%0A%0D%0A%2F%2F%20Examples%0D%0Astore.dispatch(actionCreators.increaseCounter(4))%3B%20%2F%2F%20Error%3A%20Supplied%20parameters%20do%20not%20match%20any%20signature%20of%20call%20target.%20%0D%0Astore.dispatch(actionCreators.increaseCounter())%3B%20%2F%2F%20OK%20%3D%3E%20%7B%20type%3A%20%22INCREASE_COUNTER%22%20%7D%0D%0A%0D%0Astore.dispatch(actionCreators.changeBaseCurrency())%3B%20%2F%2F%20Error%3A%20Supplied%20parameters%20do%20not%20match%20any%20signature%20of%20call%20target.%20%0D%0Astore.dispatch(actionCreators.changeBaseCurrency('USD'))%3B%20%2F%2F%20OK%20%3D%3E%20%7B%20type%3A%20%22CHANGE_BASE_CURRENCY%22%2C%20payload%3A%20'USD'%20%7D)

```ts
// Action Types
export const INCREASE_COUNTER = 'INCREASE_COUNTER';
export const CHANGE_BASE_CURRENCY = 'CHANGE_BASE_CURRENCY';

// Action Creators
export const actionCreators = {
  increaseCounter: () => ({
    type: INCREASE_COUNTER as typeof INCREASE_COUNTER,
  }),
  changeBaseCurrency: (payload: string) => ({
    type: CHANGE_BASE_CURRENCY as typeof CHANGE_BASE_CURRENCY, payload,
  }),
}

// Examples
store.dispatch(actionCreators.increaseCounter(4)); // Error: 제공하는 매개 변수가 대상의 인수와 일치하지 않음.
store.dispatch(actionCreators.increaseCounter()); // OK => { type: "INCREASE_COUNTER" }

store.dispatch(actionCreators.changeBaseCurrency()); // Error: 제공하는 매개 변수가 대상의 인수와 일치하지 않음.
store.dispatch(actionCreators.changeBaseCurrency('USD')); // OK => { type: "CHANGE_BASE_CURRENCY", payload: 'USD' }
```

- ### DRY Style
다른 대안으로 DRY 원칙, 입력 된 액션 제작자의 생성을 자동화하는 간단한 도우미 팩토리 함수를 소개합니다.
여기서 장점은 일부 상용구 및 코드 반복을 줄일 수 있다는 것입니다. 액션 생성자를 다른 모듈에서 재사용하는 것이 더 쉽습니다.

- 팩토리 함수를 이용해서 입력 된 액션 제작자를 자동으로 생성
- KISS원칙 보다 적은 상용구와 코드 반복
- redux-saga또는 같은 다른 모듈에서 재사용하기 쉽다 redux-observable(액션 생성자는 타입 프로퍼티를 가지며, 타입 상수는 중복된다)

**DEMO:** WIP

```ts
import { createActionCreator } from 'react-redux-typescript';

// Action Creators
export const actionCreators = {
  increaseCounter: createActionCreator('INCREASE_COUNTER'), // { type: "INCREASE_COUNTER" }
  changeBaseCurrency: createActionCreator('CHANGE_BASE_CURRENCY', (payload: string) => payload), // { type: "CHANGE_BASE_CURRENCY", payload: string }
  showNotification: createActionCreator('SHOW_NOTIFICATION', (payload: string, meta?: { type: string }) => payload),
};

// Examples
store.dispatch(actionCreators.increaseCounter(4)); // Error: 제공하는 매개 변수가 대상의 인수와 일치하지 않음.
store.dispatch(actionCreators.increaseCounter()); // OK: { type: "INCREASE_COUNTER" }
actionCreators.increaseCounter.type === "INCREASE_COUNTER" // true

store.dispatch(actionCreators.changeBaseCurrency()); // Error: 제공하는 매개 변수가 대상의 인수와 일치하지 않음.
store.dispatch(actionCreators.changeBaseCurrency('USD')); // OK: { type: "CHANGE_BASE_CURRENCY", payload: 'USD' }
actionCreators.changeBaseCurrency.type === "CHANGE_BASE_CURRENCY" // true

store.dispatch(actionCreators.showNotification()); // Error: 제공하는 매개 변수가 대상의 인수와 일치하지 않음.
store.dispatch(actionCreators.showNotification('Hello!')); // OK: { type: "SHOW_NOTIFICATION", payload: 'Hello!' }
store.dispatch(actionCreators.showNotification('Hello!', { type: 'warning' })); // OK: { type: "SHOW_NOTIFICATION", payload: 'Hello!', meta: { type: 'warning' } }
actionCreators.showNotification.type === "SHOW_NOTIFICATION" // true
```

---

## Reducers
관련 TypeScript 문서 참조:
- [Discriminated Union types](https://www.typescriptlang.org/docs/handbook/advanced-types.html)
- [Mapped types](https://www.typescriptlang.org/docs/handbook/advanced-types.html) like `Readonly` & `Partial`

### Declare reducer `State` 유형을 선언 하여 컴파일 타임 불변성 확보

**DEMO:** [TypeScript Playground](https://www.typescriptlang.org/play/index.html#src=export%20type%20State%20%3D%20%7B%0D%0A%20%20readonly%20counter%3A%20number%2C%0D%0A%20%20readonly%20baseCurrency%3A%20string%2C%0D%0A%7D%3B%0D%0A%0D%0Aexport%20type%20StateAlternative%20%3D%20Readonly%3C%7B%0D%0A%20%20counter%3A%20number%2C%0D%0A%20%20baseCurrency%3A%20string%2C%0D%0A%7D%3E%3B%0D%0A%0D%0Aexport%20const%20initialState%3A%20State%20%3D%20%7B%0D%0A%20%20counter%3A%200%2C%0D%0A%20%20baseCurrency%3A%20'EUR'%2C%0D%0A%7D%3B%0D%0A%0D%0AinitialState.counter%20%3D%203%3B)

```ts
// 1a. readonly를 사용한 타입으로 state, props을 불편으로 표시하고 컴파일러를 이용해 모든 변형에 대해 보호합니다.
export type State = {
  readonly counter: number,
  readonly baseCurrency: string,
};

// 1b. 원하는 경우 'Readonly'맵핑된 타입을 사용할 수 있습니다.
export type StateAlternative = Readonly<{
  counter: number,
  baseCurrency: string,
}>;

// 2. 초기 state 선언 -> Note: 수정 자만 초기화 할 수 있습니다. (일기 전용 속성)
export const initialState: State = {
  counter: 0,
  baseCurrency: 'EUR',
};

initialState.counter = 3; // Error: 읽기 전용 속성이므로 'counter'에 할당할 수 없습니다.
```

### switch style reducer
- 전통적인 const 타입을 사용
- 단일 prop 업데이트 또는 간단한 state objects에 적합

**DEMO:** [TypeScript Playground](https://www.typescriptlang.org/play/index.html#src=export%20const%20INCREASE_COUNTER%20%3D%20'INCREASE_COUNTER'%3B%0D%0Aexport%20const%20CHANGE_BASE_CURRENCY%20%3D%20'CHANGE_BASE_CURRENCY'%3B%0D%0A%0D%0Atype%20Action%20%3D%20%7B%20type%3A%20typeof%20INCREASE_COUNTER%20%7D%0D%0A%7C%20%7B%20type%3A%20typeof%20CHANGE_BASE_CURRENCY%2C%20payload%3A%20string%20%7D%3B%0D%0A%0D%0Aexport%20type%20State%20%3D%20%7B%0D%0A%20%20readonly%20counter%3A%20number%2C%0D%0A%20%20readonly%20baseCurrency%3A%20string%2C%0D%0A%7D%3B%0D%0A%0D%0Aexport%20const%20initialState%3A%20State%20%3D%20%7B%0D%0A%20%20counter%3A%200%2C%0D%0A%20%20baseCurrency%3A%20'EUR'%2C%0D%0A%7D%3B%0D%0A%0D%0Aexport%20default%20function%20reducer(state%3A%20State%20%3D%20initialState%2C%20action%3A%20Action)%3A%20State%20%7B%0D%0A%20%20switch%20(action.type)%20%7B%0D%0A%20%20%20%20case%20INCREASE_COUNTER%3A%0D%0A%20%20%20%20%20%20return%20%7B%0D%0A%20%20%20%20%20%20%20%20...state%2C%20counter%3A%20state.counter%20%2B%201%2C%20%2F%2F%20no%20payload%0D%0A%20%20%20%20%20%20%7D%3B%0D%0A%20%20%20%20case%20CHANGE_BASE_CURRENCY%3A%0D%0A%20%20%20%20%20%20return%20%7B%0D%0A%20%20%20%20%20%20%20%20...state%2C%20baseCurrency%3A%20action.payload%2C%20%2F%2F%20payload%3A%20string%0D%0A%20%20%20%20%20%20%7D%3B%0D%0A%0D%0A%20%20%20%20default%3A%20return%20state%3B%0D%0A%20%20%7D%0D%0A%7D)

```ts
import { Action } from '../../types';

export default function reducer(state: State = initialState, action: Action): State {
  switch (action.type) {
    case INCREASE_COUNTER:
      return {
        ...state, counter: state.counter + 1, // no payload
      };
    case CHANGE_BASE_CURRENCY:
      return {
        ...state, baseCurrency: action.payload, // payload: string
      };

    default: return state;
  }
}
```

### if's style reducer
> 헬퍼 팩토리 함수에서`actionCreator`에 정적 static`type` 속성 사용하기
- if의 "block scope"는 복잡한 상태 업데이트 로직에서 로컬변수를 사용할 수 있게 해줍나다.
- actionCreator의 static type속성을 on으로 사용하면 액션 유형 상수를 제거 할 수 있으므로 액션 을 쉽게 만들 수 있고 reducer, sagas, epics 등의 액션 유형을 확인하기 위해 다시 사용할 수 있습니다.

**DEMO:** WIP

```ts
import { Action } from '../../types';

// Reducer
export default function reducer(state: State = initialState, action: Action): State {
  switch (action.type) {
    case actionCreators.increaseCounter.type:
      return {
        ...state, counter: state.counter + 1, // no payload
      };
    case actionCreators.changeBaseCurrency.type:
      return {
        ...state, baseCurrency: action.payload, // payload: string
      };

    default: return state;
  }
}
```

### Spread operation는 일치하지 않는 props을 방지하는 정확한 유형을 확인 후 적용
`partialState` 객체를 사용하면 spread operation 중에 보호된 객체에 병합할 수 있습니다. - 이렇게 하면 객체가 초기회 되거나 일치하지 않는 속성을 가지는 것을 방지하고 state interface와 정홧히 일치 합니다.

**WARNING:** 이 해결법은 spread operation 중에 TypeScript 컴파일러가 불필요하거나 불일치하는 props으로부터 보호 해주지 않기 때문에 현재로서는 필수입니다.

**PS:** There is an [Exact Type proposal](https://github.com/Microsoft/TypeScript/issues/12936) to improve this behaviour

**DEMO:** [TypeScript Playground](https://www.typescriptlang.org/play/index.html#src=export%20const%20INCREASE_COUNTER%20%3D%20'INCREASE_COUNTER'%3B%0D%0Aexport%20const%20CHANGE_BASE_CURRENCY%20%3D%20'CHANGE_BASE_CURRENCY'%3B%0D%0A%0D%0Atype%20Action%20%3D%20%7B%20type%3A%20typeof%20INCREASE_COUNTER%20%7D%0D%0A%7C%20%7B%20type%3A%20typeof%20CHANGE_BASE_CURRENCY%2C%20payload%3A%20string%20%7D%3B%0D%0A%0D%0Aexport%20type%20State%20%3D%20%7B%0D%0A%20%20readonly%20counter%3A%20number%2C%0D%0A%20%20readonly%20baseCurrency%3A%20string%2C%0D%0A%7D%3B%0D%0A%0D%0Aexport%20const%20initialState%3A%20State%20%3D%20%7B%0D%0A%20%20counter%3A%200%2C%0D%0A%20%20baseCurrency%3A%20'EUR'%2C%0D%0A%7D%3B%0D%0A%0D%0Aexport%20function%20badReducer(state%3A%20State%20%3D%20initialState%2C%20action%3A%20Action)%3A%20State%20%7B%0D%0A%20%20if%20(action.type%20%3D%3D%3D%20INCREASE_COUNTER)%20%7B%0D%0A%20%20%20%20return%20%7B%0D%0A%20%20%20%20%20%20...state%2C%0D%0A%20%20%20%20%20%20counterTypoError%3A%20state.counter%20%2B%201%2C%20%2F%2F%20OK%0D%0A%20%20%20%20%7D%3B%20%2F%2F%20it's%20a%20bug!%20but%20the%20compiler%20will%20not%20find%20it%20%0D%0A%20%20%7D%0D%0A%7D%0D%0A%0D%0Aexport%20function%20goodReducer(state%3A%20State%20%3D%20initialState%2C%20action%3A%20Action)%3A%20State%20%7B%0D%0A%20%20let%20partialState%3A%20Partial%3CState%3E%20%7C%20undefined%3B%0D%0A%0D%0A%20%20if%20(action.type%20%3D%3D%3D%20INCREASE_COUNTER)%20%7B%0D%0A%20%20%20%20partialState%20%3D%20%7B%0D%0A%20%20%20%20%20%20counterTypoError%3A%20state.counter%20%2B%201%2C%20%2F%2F%20Error%3A%20Object%20literal%20may%20only%20specify%20known%20properties%2C%20and%20'counterTypoError'%20does%20not%20exist%20in%20type%20'Partial%3CState%3E'.%20%0D%0A%20%20%20%20%7D%3B%20%2F%2F%20now%20it's%20showing%20a%20typo%20error%20correctly%20%0D%0A%20%20%7D%0D%0A%20%20if%20(action.type%20%3D%3D%3D%20CHANGE_BASE_CURRENCY)%20%7B%0D%0A%20%20%20%20partialState%20%3D%20%7B%20%2F%2F%20Error%3A%20Types%20of%20property%20'baseCurrency'%20are%20incompatible.%20Type%20'number'%20is%20not%20assignable%20to%20type%20'string'.%0D%0A%20%20%20%20%20%20baseCurrency%3A%205%2C%0D%0A%20%20%20%20%7D%3B%20%2F%2F%20type%20errors%20also%20works%20fine%20%0D%0A%20%20%7D%0D%0A%0D%0A%20%20return%20partialState%20!%3D%20null%20%3F%20%7B%20...state%2C%20...partialState%20%7D%20%3A%20state%3B%0D%0A%7D)

```ts
import { Action } from '../../types';

// BAD
export function badReducer(state: State = initialState, action: Action): State {
  if (action.type === INCREASE_COUNTER) {
    return {
      ...state,
      counterTypoError: state.counter + 1, // counterTypoError가 정의 되지 않아도 오류를 내보내지 않습니다.
    }; // 컴파일 오류가 예상되고, state와 정확히 일치하는것을 보장하지 못합니다/
  }
}

// GOOD
export function goodReducer(state: State = initialState, action: Action): State {
  let partialState: Partial<State> | undefined;

  if (action.type === INCREASE_COUNTER) {
    partialState = {
      counterTypoError: state.counter + 1, // counterTypoError가 정의되어 있지 않다면 state에 정의된 속성만 리터럴 할수 잇게 방어 할수 있습니다.
    }; // 이제 에러를 올바르게 표현합니다.
  }
  if (action.type === CHANGE_BASE_CURRENCY) {
    partialState = { // 속성의 오류를 잘 잡아 냅니다. 'string'속성에 'number'를 지정할수 없습니다.
      baseCurrency: 5,
    }; // 타입 오류를 잘 잡아냅니다.
  }

  return partialState != null ? { ...state, ...partialState } : state;
}
```

---

## Store Types

- ### `Action` - 정적인 타입의 전역 action 타입
- redux, redux-sagas, redux-observables와 같은 redux 액션을 다루는 레이어로 가져와야합니다.
```ts
import { returntypeof } from 'react-redux-typescript';
import * as actionCreators from './action-creators';
const actions = Object.values(actionCreators).map(returntypeof);

export type Action = typeof actions[number];
```

- ### `RootState` - 정적으로 정형화된 전역 state 구조
- Redux connect기능에 유형안전성을 제공하는 연결된 구성 요소를 가져와야 합니다.

```ts
import {
  reducer as currencyRatesReducer, State as CurrencyRatesState,
} from './state/currency-rates/reducer';
import {
  reducer as currencyConverterReducer, State as CurrencyConverterState,
} from './state/currency-converter/reducer';

export type RootState = {
  currencyRates: CurrencyRatesState;
  currencyConverter: CurrencyConverterState;
};
```

---

## Create Store

- creating store - use `RootState` (in `combineReducers` and when providing preloaded state object) to set-up **state object type guard** to leverage strongly typed Store instance
```ts
import { combineReducers, createStore } from 'redux';
import { RootState } from '../types';

const rootReducer = combineReducers<RootState>({
  currencyRates: currencyRatesReducer,
  currencyConverter: currencyConverterReducer,
});

// rehydrating state on app start: implement here...
const recoverState = (): RootState => ({} as RootState);

export const store = createStore(
  rootReducer,
  recoverState(),
);
```

- composing enhancers - example of setting up `redux-observable` middleware
```ts
declare var window: Window & { devToolsExtension: any, __REDUX_DEVTOOLS_EXTENSION_COMPOSE__: any };
import { createStore, compose, applyMiddleware } from 'redux';
import { combineEpics, createEpicMiddleware } from 'redux-observable';

import { epics as currencyConverterEpics } from './currency-converter/epics';

const rootEpic = combineEpics(
  currencyConverterEpics,
);
const epicMiddleware = createEpicMiddleware(rootEpic);
const composeEnhancers = window.__REDUX_DEVTOOLS_EXTENSION_COMPOSE__ || compose;

// store singleton instance
export const store = createStore(
  rootReducer,
  recoverState(),
  composeEnhancers(applyMiddleware(epicMiddleware)),
);
```

---

# Ecosystem

---

## Async Flow with "redux-observable"

```ts
import 'rxjs/add/operator/map';
import { combineEpics, Epic } from 'redux-observable';

import { RootState, Action } from '../types'; // check store section
import { actionCreators } from '../reducer';
import { convertValueWithBaseRateToTargetRate } from './utils';
import * as currencyConverterSelectors from './selectors';
import * as currencyRatesSelectors from '../currency-rates/selectors';

const recalculateTargetValueOnCurrencyChange: Epic<Action, RootState> = (action$, store) =>
  action$.ofType(
    actionCreators.changeBaseCurrency.type,
    actionCreators.changeTargetCurrency.type,
  ).map((action: any) => {
    const value = convertValueWithBaseRateToTargetRate(
      currencyConverterSelectors.getBaseValue(store.getState()),
      currencyRatesSelectors.getBaseCurrencyRate(store.getState()),
      currencyRatesSelectors.getTargetCurrencyRate(store.getState()),
    );
    return actionCreators.recalculateTargetValue(value);
  });

const recalculateTargetValueOnBaseValueChange: Epic<Action, RootState> = (action$, store) =>
  action$.ofType(
    actionCreators.changeBaseValue.type,
  ).map((action: any) => {
    const value = convertValueWithBaseRateToTargetRate(
      action.payload,
      currencyRatesSelectors.getBaseCurrencyRate(store.getState()),
      currencyRatesSelectors.getTargetCurrencyRate(store.getState()),
    );
    return actionCreators.recalculateTargetValue(value);
  });

const recalculateBaseValueOnTargetValueChange: Epic<Action, RootState> = (action$, store) =>
  action$.ofType(
    actionCreators.changeTargetValue.type,
  ).map((action: any) => {
    const value = convertValueWithBaseRateToTargetRate(
      action.payload,
      currencyRatesSelectors.getTargetCurrencyRate(store.getState()),
      currencyRatesSelectors.getBaseCurrencyRate(store.getState()),
    );
    return actionCreators.recalculateBaseValue(value);
  });

export const epics = combineEpics(
  recalculateTargetValueOnCurrencyChange,
  recalculateTargetValueOnBaseValueChange,
  recalculateBaseValueOnTargetValueChange,
);
```

---

## Selectors with "reselect"

```ts
import { createSelector } from 'reselect';
import { RootState } from '../types';

const getCurrencyConverter = (state: RootState) => state.currencyConverter;
const getCurrencyRates = (state: RootState) => state.currencyRates;

export const getCurrencies = createSelector(
  getCurrencyRates,
  (currencyRates) => {
    return Object.keys(currencyRates.rates).concat(currencyRates.base);
  },
);

export const getBaseCurrencyRate = createSelector(
  getCurrencyConverter, getCurrencyRates,
  (currencyConverter, currencyRates) => {
    const selectedBase = currencyConverter.baseCurrency;
    return selectedBase === currencyRates.base
      ? 1 : currencyRates.rates[selectedBase];
  },
);

export const getTargetCurrencyRate = createSelector(
  getCurrencyConverter, getCurrencyRates,
  (currencyConverter, currencyRates) => {
    return currencyRates.rates[currencyConverter.targetCurrency];
  },
);
```

---

# Extras

### tsconfig.json
> Recommended setup for best benefits from type-checking, with support for JSX and ES2016 features.
> Add [`tslib`](https://www.npmjs.com/package/tslib) to minimize bundle size: `npm i tslib` -  this will externalize helper functions generated by transpiler and otherwise inlined in your modules
```js
{
  "compilerOptions": {
    "baseUrl": "src/", // enables relative imports to root
    "outDir": "out/", // target for compiled files
    "allowSyntheticDefaultImports": true, // no errors on commonjs default import
    "allowJs": true, // include js files
    "checkJs": true, // typecheck js files
    "declaration": false, // don't emit declarations
    "emitDecoratorMetadata": true,
    "experimentalDecorators": true,
    "forceConsistentCasingInFileNames": true,
    "importHelpers": true, // importing helper functions from tslib
    "noEmitHelpers": true, // disable emitting inline helper functions
    "jsx": "react", // process JSX
    "lib": [
      "dom",
      "es2016",
      "es2017.object"
    ],
    "target": "es5",
    "module": "es2015",
    "moduleResolution": "node",
    "noEmitOnError": true,
    "noFallthroughCasesInSwitch": true,
    "noImplicitAny": true,
    "noImplicitReturns": true,
    "noImplicitThis": true,
    "strictNullChecks": true,
    "pretty": true,
    "removeComments": true,
    "sourceMap": true
  },
  "include": [
    "src/**/*"
  ],
  "exclude": [
    "node_modules"
  ]
}
```

### tslint.json
> Recommended setup is to extend build-in preset `tslint:latest` (for all rules use `tslint:all`).
> Add tslint react rules: `npm i -D tslint-react` https://github.com/palantir/tslint-react
> Amended some extended defaults for more flexibility.
```json
{
  "extends": ["tslint:latest", "tslint-react"],
  "rules": {
    "arrow-parens": false,
    "arrow-return-shorthand": [false],
    "comment-format": [true, "check-space"],
    "import-blacklist": [true, "rxjs"],
    "interface-over-type-literal": false,
    "member-access": false,
    "member-ordering": [true, {"order": "statics-first"}],
    "newline-before-return": false,
    "no-any": false,
    "no-inferrable-types": [true],
    "no-import-side-effect": [true, {"ignore-module": "^rxjs/"}],
    "no-invalid-this": [true, "check-function-in-method"],
    "no-null-keyword": false,
    "no-require-imports": false,
    "no-switch-case-fall-through": true,
    "no-trailing-whitespace": true,
    "no-unused-variable": [true, "react"],
    "object-literal-sort-keys": false,
    "only-arrow-functions": [true, "allow-declarations"],
    "ordered-imports": [false],
    "prefer-method-signature": false,
    "prefer-template": [true, "allow-single-concat"],
    "quotemark": [true, "single", "jsx-double"],
    "triple-equals": [true, "allow-null-check"],
    "typedef": [true,"parameter", "property-declaration", "member-variable-declaration"],
    "variable-name": [true, "ban-keywords", "check-format", "allow-pascal-case"]
  }
}
```

### Default and Named Module Exports
> Most flexible solution is to use module folder pattern, because you can leverage both named and default import when you see fit.
Using this solution you'll achieve better encapsulation for internal structure/naming refactoring without breaking your consumer code:
```ts
// 1. in `components/` folder create component file (`select.tsx`) with default export:

// components/select.tsx
const Select: React.StatelessComponent<Props> = (props) => {
...
export default Select;

// 2. in `components/` folder create `index.ts` file handling named imports:

// components/index.ts
export { default as Select } from './select';
...

// 3. now you can import your components in both ways like this:

// containers/container.tsx
import { Select } from '../components';
or
import Select from '../components/select';
...
```

### Vendor Types Augmentation
> Augmenting missing autoFocus Prop on `Input` and `Button` components in `antd` npm package (https://ant.design/).
```ts
declare module '../node_modules/antd/lib/input/Input' {
  export interface InputProps {
    autoFocus?: boolean;
  }
}

declare module '../node_modules/antd/lib/button/Button' {
  export interface ButtonProps {
    autoFocus?: boolean;
  }
}
```

---

# FAQ

### - how to install react & redux types?
```
// react
npm i -D @types/react @types/react-dom @types/react-redux

// redux has types included in it's npm package - don't install from @types
```

### - should I still use React.PropTypes in TS?
> No. In TypeScript it is unnecessary, when declaring Props and State types (refer to React components examples) you will get completely free intellisense and compile-time safety with static type checking, this way you'll be safe from runtime errors, not waste time on debugging and get an elegant way of describing component external API in your source code.

### - how to best initialize class instance or static properties?
> Prefered modern style is to use class Property Initializers
```tsx
class MyComponent extends React.Component<Props, State> {
  // default props using Property Initializers
  static defaultProps: Props = {
    className: 'default-class',
    initialCount: 0,
  };

  // initial state using Property Initializers
  state: State = {
    counter: this.props.initialCount,
  };
  ...
}
```

### - how to best declare component handler functions?
> Prefered modern style is to use Class Fields with arrow functions
```tsx
class MyComponent extends React.Component<Props, State> {
// handlers using Class Fields with arrow functions
  increaseCounter = () => { this.setState({ counter: this.state.counter + 1}); };
  ...
}
```

### - differences between `interface` declarations and `type` aliases
> From practical point of view `interface` types will use it's identity when showing compiler errors, while `type` aliases will be always unwinded to show all the nested types it consists of. This can be too noisy when reading compiler errors and I like to leverage this distinction to hide some not important type details in reported type errors
Related `ts-lint` rule: https://palantir.github.io/tslint/rules/interface-over-type-literal/

---

# Project Examples

https://github.com/piotrwitek/react-redux-typescript-starter-kit
https://github.com/piotrwitek/react-redux-typescript-webpack-starter
