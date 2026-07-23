# dench-fetch

fetch의 DX를 개선하기 위해 fluent interface 패턴과 메서드 체이닝을 활용한 npm 라이브러리입니다. Native Fetch API를 기반으로, 요청 설정·전송·응답 파싱을 타입 안전한 체인으로 구성합니다.

```ts
const user = await dench("https://api.example.com")
  .get<User>("/users/1")
  .auth("access-token")
  .timeout(3000)
  .toJson();
```

## 주요 기능

1. **체이닝 기반 요청 빌더** — GET·POST·PUT·PATCH·DELETE·HEAD 요청에 인증, 타임아웃, 캐시, 리다이렉트, 쿼리 파라미터를 선언적으로 조합합니다.
2. **요청/응답 타입 분리** — body를 전송하는 요청에는 `sendJson`, `sendForm`, `sendBlob` 등을, HEAD 요청에는 헤더·상태 코드 전용 runner를 제공해 잘못된 응답 파싱을 타입 단계에서 줄입니다.
3. **Fetch 동작 보완** — URL 슬래시 정규화, `AbortController` 기반 취소·타임아웃, 비정상 HTTP 상태의 예외 처리와 에러 콜백을 제공합니다.

## 기술 스택

| 구분 | 기술 |
| --- | --- |
| Language | TypeScript 5.8 |
| HTTP | Native Fetch API, AbortController, URLSearchParams, FormData |
| Build / Package | tsup (ESM·CJS·타입 선언 파일 배포), npm |
| Test / Quality | Vitest, TypeScript typecheck, Husky |
| Automation | GitHub Actions |

## 이슈와 해결

### 1. 체이닝한 쿼리 파라미터가 실제 요청 URL에 누락되는 문제

#### 문제

`.params()`에서 파라미터를 설정해도 요청 실행 단계가 `baseURL + api`만 조합하면 쿼리 문자열이 누락됩니다. 설정 API와 실제 Fetch 호출의 책임이 분리되어 있어, 설정값을 끝까지 전달하는 경로를 보장해야 했습니다.

#### 해결

`paramsConfig`에서 다양한 `URLSearchParams` 입력을 표준 쿼리 문자열로 변환해 config에 저장하고, `runfetch`에서 URL을 완성할 때 해당 값을 결합했습니다. 문자열·객체·`URLSearchParams`·튜플 배열 입력을 테스트로 검증해 입력 형태와 무관하게 동일한 URL을 만들도록 했습니다.

```ts
// src/denchfetch/denchConfigModule.ts
else if (typeof params === "string") {
  params = new URLSearchParams(params);
} else if (typeof params === "object") {
  params = Array.isArray(params)
    ? new URLSearchParams(params)
    : new URLSearchParams(Object.entries(params));
}

if (params) config.params = `?${params.toString()}`;
return { ...config };

// src/denchfetch/denchRunner.ts
return denchfetcher<T>(
  `${config.baseURL}${config.api}${config.params ? config.params : ""}`,
  config,
);
```

#### 성과

- 체이닝 API의 설정값이 최종 네트워크 요청까지 반영되도록 보장했습니다.
- 여러 입력 형식을 단일 `URLSearchParams` 변환 경로로 수렴시켜 URL 직렬화 규칙을 일관되게 유지했습니다.

#### 관련 코드

- [src/denchfetch/denchConfigModule.ts](https://github.com/luckydeench/dench/blob/master/src/denchfetch/denchConfigModule.ts): 쿼리 파라미터 직렬화
- [src/denchfetch/denchRunner.ts](https://github.com/luckydeench/dench/blob/master/src/denchfetch/denchRunner.ts): 최종 요청 URL 조합
- [tests/denchConfigModule.test.ts](https://github.com/luckydeench/dench/blob/master/tests/denchConfigModule.test.ts): 입력 형태별 파라미터 직렬화 테스트

### 2. HEAD 요청에 body 파서가 노출되는 타입 설계 문제

#### 문제

일반 요청과 HEAD 요청이 같은 runner를 공유하면, body가 없는 HEAD 응답에도 `toJson()`과 `toFormData()`가 노출됩니다. 이는 HTTP 의미와 맞지 않을 뿐 아니라 사용자가 실행 시점에 잘못된 파서를 호출하게 만드는 API였습니다.

#### 해결

공통 `toResponse`는 `DenchBaseRunner`로 분리하고, 일반 요청에는 body 파서를 가진 `DenchRunner`, HEAD 요청에는 헤더·상태 코드만 제공하는 `DenchHeadRunner`를 정의했습니다. `createHeadBuilder`에서 일반 runner의 body 파서를 제외하고 HEAD 전용 runner를 조합했습니다.

```ts
export interface DenchHeadRunner extends DenchBaseRunner<never> {
  toHeadResponse: () => Promise<Headers>;
  toStatus: () => Promise<number>;
}

const createHeadBuilder = (config: DenchConfig): DenchHeadBuilder => {
  const { toJson, toFormData, ...builder } = createBuilder(config);
  return { ...builder as DenchHeadBuilder, ...createHeadRunner(config) };
};
```

#### 성과

- HTTP 메서드의 응답 특성을 API 표면과 타입에 반영했습니다.
- HEAD 빌더에서 `toJson`·`toFormData`가 노출되지 않는 것을 테스트로 고정해 회귀를 방지했습니다.

#### 관련 코드

- [src/types/denchRunner.ts](https://github.com/luckydeench/dench/blob/master/src/types/denchRunner.ts): 일반/HEAD runner 타입 분리
- [src/types/denchBuilder.ts](https://github.com/luckydeench/dench/blob/master/src/types/denchBuilder.ts): `DenchHeadBuilder` 계약
- [src/denchfetch/dench.ts](https://github.com/luckydeench/dench/blob/master/src/denchfetch/dench.ts): HEAD 전용 빌더 조합
- [tests/denchInterface.test.ts](https://github.com/luckydeench/dench/blob/master/tests/denchInterface.test.ts): HEAD API 노출 범위 테스트

### 3. HTTP 메서드별 빌더 구현 중복으로 인한 확장 비용

#### 문제

GET 계열과 body 전송 계열이 각각 인증·취소·타임아웃·URL 정규화·응답 처리 체인을 구현하면, 공통 옵션을 추가하거나 수정할 때 여러 빌더에 같은 변경을 반복해야 합니다. 메서드별 기능 차이도 흐려져 유지보수 비용이 커집니다.

#### 해결

공통 체인은 `createBuilder`에 집중하고, body 전송이 필요한 메서드에만 `createCreateBuilder`가 `send*` 메서드를 확장하도록 구성했습니다. 설정 함수는 변경된 config를 다음 빌더로 전달해 체이닝 단계가 누적되도록 했습니다.

```ts
const createCreateBuilder = <T>(config: DenchConfig): DenchCreateBuilder<T> => ({
  ...createBuilder<T>(config) as DenchCreateBuilder<T>,
  sendJson: (data?: unknown) => createCreateBuilder<T>(sendJsonConfig(config, data)),
  sendForm: (data?: FormData) => createCreateBuilder<T>(sendFormConfig(config, data)),
  sendBlob: (data?: Blob) => createCreateBuilder<T>(sendBlobConfig(config, data)),
  sendUrlEncoded: (data?: DenchURLSearchParams) =>
    createCreateBuilder<T>(sendUrlEncodedConfig(config, data)),
});

export function timeoutConfig(config: DenchConfig, ms: number): DenchConfig {
  return { ...config, timeout: ms };
}
```

#### 성과

- 공통 체인 로직을 한 곳으로 모아 옵션 추가·수정의 변경 범위를 축소했습니다.
- body 전송 기능을 POST·PUT·PATCH에만 제공해 HTTP 메서드별 사용 가능한 기능을 명확히 했습니다.
- 빌더 복사 시 config·options·headers를 분리 복사하는 테스트를 추가해 재사용 시 설정 오염을 방지했습니다.

#### 관련 코드

- [src/denchfetch/dench.ts](https://github.com/luckydeench/dench/blob/master/src/denchfetch/dench.ts): 공통/생성 요청 빌더 구성
- [src/denchfetch/denchConfigModule.ts](https://github.com/luckydeench/dench/blob/master/src/denchfetch/denchConfigModule.ts): 요청 옵션별 config 변환
- [src/types/denchBuilder.ts](https://github.com/luckydeench/dench/blob/master/src/types/denchBuilder.ts): 체이닝 인터페이스와 메서드별 타입 제한
- [tests/denchUtils.test.ts](https://github.com/luckydeench/dench/blob/master/tests/denchUtils.test.ts): 빌더 `copy()`의 깊은 복사 검증
