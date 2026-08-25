# Foundations and API Boundaries — Rewritten Development Threads

- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/ft_transcendence`
- Source folder: `development-thread-workbook/01-foundations-and-api-boundaries/completed`

이 폴더는 `01-foundations-and-api-boundaries` 카테고리의 Development Thread를 exact commit diff와 해당 SHA의 source를 기준으로 다시 작성한 결과다.

기존 문서에서 Thread 구분, commit/SHA, 제목, importance, tags, 역할을 출발점으로 사용했다. 반복되는 workbook 표와 미완성 학습자 입력란은 제거하고, 각 Thread가 실제로 설명하는 개발 문제·결정·실패·보장 범위에 맞춰 구조를 다시 잡았다.

## 문서

1. [모노레포 패키지 경계에서 서비스 조립까지](./01-workspace-package-and-composition-boundaries.md)  
   workspace, shared DTO/game contract, DB repository lifecycle, API composition root, Next.js consumer의 책임 분리

2. [Resource API가 실행 가능한 HTTP 계약을 갖기까지](./02-executable-http-contracts-and-resource-api.md)  
   assertion/default 기반 Fastify route에서 shared Zod contract, typed error boundary, input/output validation으로의 retrofit

3. [국소 body 방어에서 모든 JSON 입력의 strict 검증까지](./03-strict-json-request-validation.md)  
   params/query/body complete contract map, 공통 parser, route-wide negative regression

4. [WebSocket client 오류와 내부 처리 실패를 분리한다](./04-websocket-internal-error-containment.md)  
   malformed client event와 repository failure의 분류 분리, raw exception redaction, deterministic failure test

5. [Runtime mode·CORS·network trust를 fail-closed 설정으로 수렴시킨다](./05-runtime-mode-cors-and-network-trust.md)  
   local defaults, browser CORS, explicit mode parsing, guest IP quota, secret·persistent-storage startup requirement

## 작성 기준

- 각 commit은 표시된 exact SHA 또는 그 diff/source에서 확인했다.
- final HEAD의 구현을 과거 commit에 소급하지 않았다.
- 같은 commit의 변경 중 Thread와 무관한 부분은 제외하거나 경계만 명시했다.
- 작은 `B` commit은 맡은 역할과 실제 변경만 설명하고, `A` commit은 실패 원인·결정·보장·비보장 범위를 더 깊게 추적했다.
- repository와 기존 문서가 충돌한 경우 repository를 우선했다. importance와 tags는 변경하지 않았다.
- 여러 Thread를 억지로 선형 연결하지 않고 각자가 다루는 불변 조건을 독립적으로 닫았다.

## 확인 과정에서 바로잡은 대표 사례

- `8ce1199ffd12`는 body 없는 채팅을 허용한 변경이 아니라, 누락된 body를 `{}`로 정규화해 property access 예외 대신 의도한 400 분기로 보내는 변경이다.
- `1140fb868714`은 migration runner 자체를 추가하지 않고 initial SQL을 import 가능한 값으로 노출한다. 실제 실행은 `9277572765e7`의 `ensureSeedData()`에서 연결된다.
- typed HTTP boundary 후 login response와 대표 test는 cookie 중심으로 바뀌지만, `50caaf5c7c49` 시점의 production token reader는 Bearer와 query fallback도 계속 허용한다.
- `f801ccd09cf0`은 `TRUST_PROXY` 값을 parse하지만 그 diff 자체가 Fastify wiring까지 추가하지는 않는다.
- `4b43a284e637`의 `ensureSeedData()`는 listen catch 밖에 있어 초기화 실패 시 repository close를 보장하지 않는다.

## 검증 범위

GitHub에서 지정 브랜치의 문서, exact SHA diff, 필요한 historical source/test를 검사했다. 로컬 checkout을 만들 수 없는 환경이어서 build, typecheck, unit/integration test는 실행하지 않았다. 따라서 문서의 runtime 결과는 source에 존재하는 test intent와 assertion을 설명하며, 실제 통과 결과를 주장하지 않는다.
