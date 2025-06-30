---
title: "[리액트 렌더링] 불필요한 formState 참조로 인해 페이지 전체가 리렌더링되어 리액트 렌더링 최적화를 통해 스크립트 실행 시간 58% 개선"
layout: single
excerpt: react-hook-form과 렌더링 최적화를 적용하며 스크립트 실행 시간을 개선한 경험을 소개한다.

categories:
  - trouble-shooting
tags: [리액트 렌더링 최적화, react-hook-form, INP]
toc_label: 렌더링 최적화
toc: true
toc_sticky: true
---

<div class="red-box">
  <div>특정 인풋만 입력할 때마다 유효성 검증을 하기 위해 <mark class="mark">trigger를 onChange마다 호출</mark>하였습니다.</div>
  <div>trigger는 Promise를 반환하는 비동기 함수로 무거운 작업이며, 이를 입력할 때마다 호출하면 성능에 문제가 생길 여지가 있었습니다.</div>
  <div>이후 <mark class="mark">모바일 환경을 고려하여 CPU 쓰로틀링을 걸어보니 입력 지연이 발생</mark>하여 UX에 악영향을 미쳐 입력 시 지연이 발생하는 이유를 탐구하기 시작했습니다.</div>
  <div>어떤 동작 때문에 사용자 입력이 화면에 바로 반영되지 않는걸까요?</div>
</div>

## 🔥 결론

1차 개선에는 컴포넌트를 관심사에 따라 나누고 <mark class="mark">렌더링을 유발하는 참조를 분리하여 리액트 렌더링을 최적화</mark> 하였습니다.

|                  -                   |                                                               개선 전                                                               |                                                               1차 개선                                                               |
| :----------------------------------: | :---------------------------------------------------------------------------------------------------------------------------------: | :----------------------------------------------------------------------------------------------------------------------------------: |
| 스크립트 실행 시간<br />(553 -> 230) | <img width="400" alt="개선전_요약" src="https://github.com/user-attachments/assets/5059827b-87da-4236-bb8c-e38656de20bf" /> | <img width="400" alt="1차_개선_요약" src="https://github.com/user-attachments/assets/a38ede5d-c178-4852-9f2d-cd1faa13217c" /> |
|         INP<br/>(168 -> 136)         |   <img width="400" alt="개선전_INP" src="https://github.com/user-attachments/assets/0be43565-e7a9-45c8-bd9d-e9720559d700" />   |   <img width="400" alt="1차_개선_INP" src="https://github.com/user-attachments/assets/09094a16-c981-47f4-92ae-c17cbfad68e8" />   |

## 1차 개선 (리액트 렌더링 최적화)

리액트 렌더링 최적화부터 말씀드리면 단순하게 컴포넌트가 필요한 값만 변할 때 해당 컴포넌트를 렌더링시키는 게 목적입니다.

RHF를 사용하다보니 FormProvider로 form을 내려주고 하위에서는 useFormContext로 값들을 가져오고 있습니다. 이를 통해 props drilling 없이 Context에서 필요한 값만 가지고 올 수 있었습니다.

하지만 RHF에서도 결국 state를 사용하니 불필요한 리렌더링이 발생했습니다.
<br/>
formState도 결국 상태고 리렌더링될 때마다 상태가 변경될텐데, 회원가입 버튼 유효성 검증을 위해 <mark class="mark">formState를 부모에서 참조하고 있어서 입력할 때마다 전체 페이지가 렌더링</mark>되고 있었습니다.

그래서 RHF의 이점을 최대한 살리면서 요구사항을 만족하고 성능을 개선하기 위해 다음과 같은 작업을 진행하였습니다.

<div class="blue-box">  
  <li>부모 요소에서 불필요하게 watch로 값을 구독하는 부분 제거한다.</li>
  <li>회원가입 API에는 사용되지 않더라도 회원가입 진행에 영향을 주는 값이라면 RHF로 관리한다.</li>
  <li>이벤트 시점에 특정 값이 필요할 때는 getValue로 스냅샷을 가져온다.</li>
  <li>useFormContext의 formState 대신 useFormState를 사용한다.</li>
</div>

### 부모 요소에서 불필요하게 watch로 값을 구독하는 부분 제거한다.

유효성 검증에 필요한 값인데 부모 요소에서 폼데이터를 구독하고 있어 불필요하게 페이지 전체가 리렌더링 됩니다.

버튼 컴포넌트를 분리하고 유효성 검증에 필요한 곳만 리렌더링을 발생시킵니다.

**AS-IS**

```tsx
const JoinEmailStep = ({ onNext }: JoinEmailStepProps) => {
  const { formState: { errors } } = useFormContext<ResearcherJoinSchemaType>();

  const oauthEmail = useWatch({ name: 'oauthEmail', control });
  const univEmail = useWatch({ name: 'univEmail', control });
  const contactEmail = useWatch({ name: 'contactEmail', control });

  const isValidForm =
    oauthEmail &&
    univEmail &&
    !errors.contactEmail &&
    !errors.univEmail &&
    isEmailVerified &&
    serviceAgreeCheck.isTermOfService &&
    serviceAgreeCheck.isPrivacy;


  return ( <button disabled={!isValidForm}>다음</button> )
```

**TO-BE**

```tsx
const JoinEmailStep = ({ onNext }: JoinEmailStepProps) => {
  return <NextButton />;
};

const NextButton = () => {
  const contactEmail = useWatch({ name: "contactEmail", control });
  const verifiedContactEmail = useWatch({
    name: "verifiedContactEmail",
    control,
  });
  const isTermOfService = useWatch({ name: "isTermOfService", control });
  const isPrivacy = useWatch({ name: "isPrivacy", control });
  const isEmailVerified = useWatch({ name: "isEmailVerified", control });

  const isValidForm =
    Boolean(verifiedContactEmail) &&
    verifiedContactEmail === contactEmail &&
    !errors.contactEmail &&
    !errors.univEmail &&
    isEmailVerified &&
    isTermOfService &&
    isPrivacy;

  return <Button disabled={!isValidForm}>다음</Button>;
};
```

### 회원가입 API에는 사용되지 않더라도 회원가입 진행에 영향을 주는 값이라면 RHF로 관리한다.

이전에는 회원가입 API에 필요한 데이터만 Schema로 관리하고, 그 외에는 클라이언트 상태로 관리했었습니다.

이러다보니 상태를 계속 넘겨줘야 했고, 부모에서 관리될 수 밖에 없어 체크박스를 클릭할 때마다 전체 페이지가 리렌더링되었습니다.

특히 회원가입은 funnel 형식인데, 뒤로 돌아오는 경우 클라이언트 상태는 유지할 수 없어 UX/UI가 의도와 다르게 동작하는 문제가 있었습니다.

회원가입 진행에 영향을 주는 값이라면 RHF로 관리하는 것에 대해 팀원과 논의 후 반영하였고, 실제로 회원가입에 필요한 데이터는 별도의 Schema로 관리하여 처리하였습니다.

**AS-IS**

```tsx
const JoinEmailStep = (...) => {
  const { serviceAgreeCheck, handleAllCheck, handleChangeCheck } = useServiceAgreeCheck({
    onCheckAdConsent: (checked) => setValue('adConsent', checked),
  });

  const isValidForm =
    ...
    serviceAgreeCheck.isTermOfService &&
    serviceAgreeCheck.isPrivacy;

    return (
      <JoinCheckboxContainer
          serviceAgreeCheck={serviceAgreeCheck}
          handleAllCheck={handleAllCheck}
          handleChange={handleChangeCheck}
      />
    )
}
```

**TO-BE**

```tsx
const JoinEmailStep = (...) => {

  return (
    <JoinCheckboxContainer />
    <NextButton />
  )

}

const JoinCheckboxContainer = (...) => {
  const isTermOfService = useWatch({ name: 'isTermOfService', control });
  const isPrivacy = useWatch({ name: 'isPrivacy', control });
}

const NextButton = (...) => {
  const { formState: { errors } } = useFormContext<ResearcherJoinSchemaType>();

  const isTermOfService = useWatch({ name: 'isTermOfService', control });
  const isPrivacy = useWatch({ name: 'isPrivacy', control });


  const isValidForm =
    ...
    isTermOfService && isPrivacy;
}
```

### 이벤트 시점에 특정 값이 필요할 때는 getValue로 스냅샷을 가져온다.

학교 메일 인증이나 특정 시점에 값만 가져오면 되는 경우에는 useWatch로 구독하고 있지 않고 getValue로 스냅샷만 가져옵니다.

불필요한 리렌더링을 발생시키지 않고, 이벤트가 발생한 시점의 필드값을 가져옵니다.

입력값이 변하지 않더라도 불필요한 구독 비용, 메모리 비용을 제거할 수 있습니다.

**AS-IS**

```tsx
const JoinEmailStep = (...) => {
  const oauthEmail = useWatch({ name: 'oauthEmail', control });

  return (
    <JoinInput value={oauthEmail} />
  )
}
```

**TO-BE**

```tsx
const JoinEmailStep = (...) => {
  const oauthEmail = useWatch({ name: 'oauthEmail', control });

  return (
    <JoinInput value={getValues('oauthEmail') || ''} />
  )
}

const NextButton = (...) => {
  const handleNextStep = async () => {
    const isValid = await trigger(['oauthEmail', 'contactEmail']);
    if (isValid) {
      onNext();
    }
  };

    return (
    <Button onClick={handleNextStep}>다음</Button>
  );
}
```

### useFormContext의 formState 대신 useFormState를 사용한다.

formState.errors를 참조했을 때 리렌더링이 발생하는 걸 인지하지 못하고 있었습니다.
<br/>
폼데이터 중 어떤 값이 변경되더라도 페이지 전체가 리렌더링이 되는 문제가 발생했습니다.
<br/>
error 객체 참조를 useFormContext가 아닌 useFormState에서 특정 필드만 참조하여 해결하였습니다.

**AS-IS**

```tsx
const JoinEmailStep = (...) => {
  const { formState: { errors } } = useFormContext<ResearcherJoinSchemaType>();

  const isValidForm =
    ...
    !errors.contactEmail &&
    !errors.univEmail
}
```

**TO-BE**

```tsx
// JoinEmailStep.tsx
const JoinEmailStep = (...) => {
  return (
      <NextButton />
  )
}

// NextButton.tsx
const NextButton = (...) => {
  const { errors } = useFormState({
    control,
    name: ['contactEmail', 'univEmail'],
  });

    const isValidForm =
    ...
    !errors.contactEmail &&
    !errors.univEmail
```

## 📘 reference

- [연구자 schema 리팩토링 및 렌더링 최적화 구조 개선 PR](https://github.com/YAPP-Github/25th-Web-Team-2-FE/pull/160)
- [참여자 schema 리팩토링 및 ContactEmailInput 구현 PR](https://github.com/YAPP-Github/25th-Web-Team-2-FE/pull/165)
- [RHF - formState](https://react-hook-form.com/docs/useform/formstate)
