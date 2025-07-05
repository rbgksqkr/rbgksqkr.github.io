---
title: "[RHF] useWatch가 특정 필드의 변화만 감지할 수 있는 이유"
layout: single
excerpt: useWatch가 특정 필드의 변화만 감지하여 컴포넌트를 분리했을 때 렌더링 최적화를 이룰 수 있는 이유를 설명한다.

categories:
  - react
tags: [리액트 렌더링 최적화, react-hook-form, useWatch, pub/sub]
toc_label: useWatch
toc: true
toc_sticky: true
---

<div class="red-box">
  <div>useWatch를 사용하면 특정 필드(name)만 구독할 수 있어 컴포넌트를 분리하여 렌더링 최적화가 가능합니다.</div>
  <div>하지만 일반적인 useState는 객체 상태를 가질 때 불변성을 유지하기 위해 전체 객체를 setState해주는데, useWatch는 어떻게 <mark class="mark">특정 필드의 변화만 추적하여 리렌더링을 유발할 수 있는지</mark> 궁금증이 생겨 라이브러리를 뜯어보게 되었습니다.</div>
</div>

## 🔥 결론

✅ **useWatch의 개별 useState**

- 각 useWatch는 자신만의 useState를 가짐
- 초기값은 `control._getWatch()` 로 폼의 최신값에서 해당 name 값을 가져옴
- useWatch는 값을 읽기만 함 (폼 데이터 변경 X)
- 폼 데이터 변경은 setValue, onChange 등에서 subject.next를 호출하여 publish

✅ **구독 시스템**

- useWatch는 특정 name 필드를 구독함
- 구독한 name 필드가 변경될 때만 setState(updateValue) 실행
- useWatch를 호출하는 컴포넌트만 리렌더링

✅ **\_subscribe 함수**

- shouldSubscribeByName으로 구독한 name과 변경된 name 비교
- 일치하면 콜백 실행 → setState → 리렌더링

<div class="blue-box">
따라서, callback이 실행되어야 setState가 되는데 name 필드에 대해서만 callback이 실행되도록 처리되어 있어, 지정한 name 필드가 변경되어야만 useWatch 내부 state가 업데이트되면서 리렌더링이 발생합니다.
</div>
---

## 코드 분석

### useWatch

useWatch는 내부적으로 useState로 관리하며, 초기값을 폼에서 최신 데이터를 가져옵니다.

폼 변경을 구독하고 있고, name 필드와 callback을 넘겨주고 있는데, callback이 실행되면 setState가 되는 것을 알 수 있습니다.

```tsx
export function useWatch<TFieldValues extends FieldValues>(
  props?: UseWatchProps<TFieldValues>
) {
  const methods = useFormContext<TFieldValues>();
  const {
    control = methods.control,
    name,
    defaultValue,
    disabled,
    exact,
  } = props || {};
  const _defaultValue = React.useRef(defaultValue);
  const [value, updateValue] = React.useState(
    control._getWatch(
      name as InternalFieldName,
      _defaultValue.current as DeepPartialSkipArrayKey<TFieldValues>
    )
  );

  // 폼 변경 구독
  useIsomorphicLayoutEffect(
    () =>
      control._subscribe({
        name,
        formState: {
          values: true,
        },
        exact,
        callback: (formState) =>
          !disabled &&
          updateValue(
            generateWatchOutput(
              name as InternalFieldName | InternalFieldName[],
              control._names,
              formState.values || control._formValues,
              false,
              _defaultValue.current
            )
          ),
      }),
    [name, control, disabled, exact]
  );

  React.useEffect(() => control._removeUnmounted());

  return value;
}
```

### \_subscribe

subscribe 할 때 구독하고 싶은 필드명(name)을 전달하면 내부적으로 name 필드에 대한 방어코드를 실행합니다.

subject.next를 호출하여 subscribe하고 있는 구독자들에게 notify를 보내는데, next할 때 보내는 name필드가 formState.name에 들어가고, subscribe할 때 주입한 props.name과 비교합니다.

```tsx
_subjects.state.next({
  validatingFields: _formState.validatingFields,
  isValidating: !isEmptyObject(_formState.validatingFields),
});
```

변경된 formState와 구독하고 있는 props의 name이 같을 때만 callback을 실행합니다.

따라서, useWatch는 특정 name에 대해서만 상태 업데이트를 실행해 리렌더링을 발생시키는 것을 알 수 있습니다.

```tsx
const _subscribe: FromSubscribe<TFieldValues> = (props) =>
  _subjects.state.subscribe({
    next: (
      formState: Partial<FormState<TFieldValues>> & {
        name?: InternalFieldName;
        values?: TFieldValues | undefined;
        type?: EventType;
      }
    ) => {
      if (
        // name 필드 방어 코드
        shouldSubscribeByName(props.name, formState.name, props.exact) &&
        shouldRenderFormState(
          formState,
          (props.formState as ReadFormState) || _proxyFormState,
          _setFormState,
          props.reRenderRoot
        )
      ) {
        props.callback({
          values: { ..._formValues } as TFieldValues,
          ..._formState,
          ...formState,
        });
      }
    },
  }).unsubscribe;

const subscribe: UseFormSubscribe<TFieldValues> = (props) => {
  _state.mount = true;
  _proxySubscribeFormState = {
    ..._proxySubscribeFormState,
    ...props.formState,
  };
  return _subscribe({
    ...props,
    formState: _proxySubscribeFormState,
  });
};
```

## 📘 reference

- [react-hook-form - 소스코드](https://github.com/react-hook-form/react-hook-form)
