# Thread: 브라우저 애플리케이션 셸과 내비게이션 정체성

- 카테고리: `06-browser-application-architecture` — 브라우저 애플리케이션 아키텍처
- Repository: `https://github.com/seungwoo7050/42-archive`
- Branch: `web/ft_transcendence`

## 개요

이 Thread는 `apps/web`을 실행 가능한 Next.js 애플리케이션으로 만들고, 문서 루트·공통 레이아웃·개발 로그인 진입을 구성한 뒤, **현재 사용자에 따라 URL이 바뀌는 프로필 항목의 정체성을 안정화하는 과정**을 다룹니다.

초기 `AppShell`은 프로필 링크를 `/profile/tester`로 고정했습니다. 현재 세션을 읽도록 바뀐 뒤에는 같은 프로필 항목의 URL이 처음에는 `/`, 사용자 확인 뒤에는 `/profile/<handle>`로 변했습니다. 이때 URL을 React key와 항목 정체성으로 함께 사용하면 같은 논리 항목이 세션 해석 전후에 다른 항목으로 취급됩니다. 더 직접적인 문제는 사용자가 확인되기 전에 프로필을 눌렀을 때 로비(`/`)로 이동한다는 점입니다.

최종적으로 이 Thread가 세운 규칙은 다음과 같습니다.

- `RootLayout`은 문서 metadata와 전역 스타일을 소유합니다.
- `AppShell`은 공통 레이아웃, 현재 경로에 따른 active 표시, navigation 항목 정체성을 소유합니다.
- navigation item의 정체성은 바뀔 수 있는 `href`가 아니라 안정적인 논리 ID입니다.
- 현재 사용자가 확인되기 전 프로필 항목은 임시 URL을 가진 링크가 아니라 비활성 항목입니다.
- `HomePage`와 `LoginPanel`은 각각 인증 분기와 개발 로그인 입력·오류 상태를 소유합니다.

## Commit map

| 순서 | SHA | Subject | Importance | Tags | Thread 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `f5c151c7cc7d` | `chore(web): Next.js runtime 경계 구성` | B | PROTOCOL, PERSISTENCE, WEB | web workspace를 Next.js application과 shared package 소비 경계로 구성합니다. |
| 2 | `4071b935cb24` | `chore(web): Tailwind style build 구성` | B | WEB | Tailwind/PostCSS source scan과 style processing 경계를 구성합니다. |
| 3 | `ce174d6b3633` | `feat(web): 한국어 로비 shell 초기화` | B | REALTIME, WEB | 한국어 metadata, design token, base style, focus-visible을 갖는 App Router shell을 만듭니다. |
| 4 | `77f35c72cd7b` | `feat(web): 공통 내비게이션 프레임 구현` | B | WEB, OPERATIONS | responsive sidebar, active route, content width를 `AppShell`이 소유합니다. |
| 5 | `f27199fdcd34` | `feat(web): 개발용 로그인 패널 추가` | B | AUTH, WEB | 개발 로그인 입력과 mutation 상태를 `LoginPanel`에 추가합니다. |
| 6 | `52ddc3acfcce` | `feat(web): 로비 인증 진입 연결` | B | AUTH, WEB | home route가 current session에 따라 LoginPanel과 authenticated lobby를 분기합니다. |
| 7 | `d755b8dae2c1` | `test(e2e): 한국어 내비게이션과 캔버스 흐름 구성` | B | PERSISTENCE, TOURNAMENT, WEB | Playwright desktop/mobile browser 검증 경계를 구성합니다. |
| 8 | `ec000bed0414` | `build(web): production start와 TS cache 정책 구성` | B | WEB | web application에 production start와 package-local test command를 추가하고 type-check cache 생성을 끕니다. |
| 9 | `34eccd6c7150` | `feat(profile): 현재 프로필과 공유 기능 연결` | B | WEB | 현재 session을 읽어 profile navigation target과 profile 공유 기능을 연결합니다. |
| 10 | `a56a4dee9219` | `fix(web): 안정적인 navigation key 사용` | B | WEB | 현재 URL 대신 stable logical ID를 React key로 사용합니다. |
| 11 | `bc8d023b2999` | `fix(web): profile link 전 사용자 식별 대기` | B | WEB | current user가 resolve되기 전에 잘못된 profile link를 만들지 않습니다. |

## 실행 가능한 브라우저 애플리케이션 만들기

### `f5c151c7cc7d` — Next.js runtime 경계

이 커밋은 `apps/web`을 이름만 있는 workspace가 아니라 독립적으로 개발·빌드·타입 검사할 수 있는 애플리케이션으로 만듭니다.

```json
{
  "name": "@pong-pong/web",
  "scripts": {
    "dev": "next dev --hostname 0.0.0.0 --port 3000",
    "build": "next build",
    "typecheck": "tsc --noEmit"
  }
}
```

`next.config.mjs`의 핵심은 shared package를 브라우저 build가 직접 transpile하도록 지정한 부분입니다.

```js
const nextConfig = {
  transpilePackages: ["@pong-pong/shared"]
};
```

`tsconfig.json`은 `@/*`를 web source로, `@pong-pong/shared`를 monorepo source로 연결합니다. 이 시점에는 route나 component가 없으므로, 이 커밋이 보장하는 것은 **실행·compile 경계**이지 화면 동작이 아닙니다.

### `4071b935cb24` — style build 경계

PostCSS에 Tailwind와 Autoprefixer를 연결하고, class scan 범위를 `./src/**/*.{ts,tsx}`로 한정합니다. 색상과 shadow token도 이때 정의됩니다.

```ts
const config: Config = {
  content: ["./src/**/*.{ts,tsx}"],
  theme: {
    extend: {
      colors: { /* ink, muted, line, panel, blue, green */ },
      boxShadow: { card: "0 10px 30px rgba(15, 36, 78, 0.08)" }
    }
  }
};
```

이 설정은 이후 component class를 CSS로 생산할 수 있게 하지만, 접근성이나 실제 레이아웃을 아직 검증하지는 않습니다.

### `ce174d6b3633` — 문서 루트와 첫 화면

`RootLayout`이 한국어 문서 metadata, 전역 CSS import, `<html lang="ko">`를 소유합니다.

```tsx
export const metadata: Metadata = {
  title: "퐁퐁",
  description: "실시간 Pong 매칭 프로토타입"
};

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="ko">
      <body>{children}</body>
    </html>
  );
}
```

`globals.css`에는 색상 변수, box sizing, body 기본값, `.card`, `.focus-ring:focus-visible`이 추가됩니다. `HomePage`는 아직 인증이나 server state 없이 정적 소개만 렌더링합니다. 따라서 이 시점의 shell은 브라우저 문서와 시각적 기반이지 애플리케이션 상태 owner는 아닙니다.

## 공통 프레임과 인증 진입

### `77f35c72cd7b` — `AppShell` 도입

`AppShell`은 sidebar, desktop header, content width와 active route 계산을 한 component에 모읍니다. 최초 navigation은 정적 배열이며 프로필 URL도 고정값입니다.

```tsx
const nav = [
  { href: "/", label: "로비", icon: Home },
  { href: "/play", label: "경기", icon: Gamepad2 },
  { href: "/dashboard", label: "대시보드", icon: BarChart3 },
  { href: "/leaderboard", label: "순위표", icon: Trophy },
  { href: "/tournaments", label: "토너먼트", icon: Users },
  { href: "/profile/tester", label: "프로필", icon: UserRound },
  { href: "/admin", label: "관리", icon: Shield }
];
```

active 여부는 현재 pathname과 `href`의 exact/prefix match로 계산하고, React key도 `item.href`를 사용합니다. 정적 URL만 있을 때는 문제가 없지만, 이후 프로필 URL이 세션에 따라 바뀌면서 이 두 역할이 충돌합니다.

이 커밋의 header에는 “서버 준비”, “평균 대기 30초 이하” 같은 고정 문구도 포함됩니다. 이는 공통 프레임을 만드는 단계의 presentation이며 실제 서버 상태를 보장하지 않습니다. 표시 진실성 문제는 resource screen Thread에서 별도로 교정됩니다.

### `f27199fdcd34` — 개발 로그인 입력과 mutation 상태

`LoginPanel`은 handle/display name 입력, 오류 표시, `devLogin` 호출을 소유합니다.

```tsx
onClick={async () => {
  try {
    setError(null);
    onLogin(await devLogin(handle, displayName));
  } catch {
    setError("API 서버에 연결하지 못했습니다.");
  }
}}
```

성공 시 상위 component에 `SessionUser`를 전달하고, 실패 시 local error만 갱신합니다. 별도의 loading/중복 클릭 차단은 아직 없습니다.

### `52ddc3acfcce` — `HomePage`의 인증 분기

정적 home route가 client component가 되고, mount 시 `getMe()`와 `getLobby()`를 요청합니다. `me`가 없으면 `LoginPanel`, 있으면 authenticated content를 렌더링합니다.

```tsx
const [me, setMe] = useState<SessionUser | null>(null);
const [players, setPlayers] = useState<PublicUser[]>(sampleUsers);
const [chat, setChat] = useState<ChatMessage[]>(sampleChat);

useEffect(() => {
  getMe().then(setMe);
  getLobby().then((lobby) => {
    setPlayers(lobby.onlinePlayers);
    setChat(lobby.chat);
    if (lobby.me) setMe(lobby.me);
  });
}, []);
```

여기에는 두 가지 중요한 한계가 있습니다.

1. `null` 하나가 “아직 확인 중”과 “로그인되지 않음”을 함께 표현하므로 요청이 끝나기 전에도 로그인 패널이 보입니다.
2. players/chat가 sample data로 시작하므로 request 실패가 실제 데이터처럼 남을 수 있습니다.

이 Thread는 인증 진입 경계만 다룹니다. sample fallback 제거와 loading/error/empty 상태 분리는 Thread 03의 책임입니다.

## 첫 브라우저 검증과 production 실행

### `d755b8dae2c1` — Playwright 경계

root script와 Makefile에 Playwright 실행 경로를 추가하고 desktop Chrome과 Pixel 7 project를 정의합니다. 테스트는 두 흐름을 고정합니다.

- 개발 로그인 후 대시보드·순위표·토너먼트 heading까지 이동할 수 있는가
- `/play`의 canvas에 alpha가 0이 아닌 pixel이 하나 이상 그려지는가

canvas 검사는 실제 pixel buffer를 읽습니다.

```ts
const data = ctx.getImageData(0, 0, canvas.width, canvas.height).data;
for (let index = 3; index < data.length; index += 4) {
  if (data[index] !== 0) return true;
}
```

이 테스트는 navigation wiring과 canvas paint를 확인하지만, API 응답이 실제 server data인지 sample fallback인지 구분하지 않습니다. 이 작업 환경에서는 Playwright를 실행하지 않았으므로 source에 테스트가 존재한다는 사실만 확인했습니다.

### `ec000bed0414` — production command와 type-check cache 정책

web package에 `next start`와 package-local Vitest command를 추가하고 TypeScript incremental cache 생성을 끕니다.

```diff
+"start": "next start --hostname 0.0.0.0 --port 3000",
+"test": "vitest run --passWithNoTests"
```

```diff
-"incremental": true
+"incremental": false
```

이는 production process가 어떤 command로 web server를 띄우는지와 CI/type-check가 package-local cache artifact에 의존하지 않도록 하는 build 계약입니다. navigation 동작 자체를 바꾸지는 않습니다.

## 프로필 URL과 navigation identity 분리

### `34eccd6c7150` — 현재 사용자 기반 프로필 target

프로필 page에는 현재 URL을 clipboard에 복사하는 실제 공유 action이 연결됩니다. 같은 커밋에서 `AppShell`은 `getMe()`를 호출해 profile target을 현재 사용자 handle로 바꿉니다.

```tsx
const [me, setMe] = useState<SessionUser | null>(null);
const profileHref = me ? `/profile/${me.handle}` : "/";

useEffect(() => {
  getMe().then(setMe).catch(() => setMe(null));
}, []);
```

```tsx
{ href: profileHref, label: "프로필", icon: UserRound, matchPrefix: "/profile" }
```

active 계산에는 `matchPrefix: "/profile"`이 추가되어 다른 사용자의 공개 profile route에서도 프로필 항목을 활성화합니다. 그러나 이 구현은 `href`에 세 가지 책임을 동시에 부여합니다.

- 클릭 destination
- React list key
- 항목의 논리적 identity

세션을 읽기 전 `profileHref`는 `/`이므로 프로필 항목은 로비 항목과 같은 destination/key를 가집니다. 세션이 resolve되면 key가 `/profile/<handle>`로 바뀌어 React가 같은 항목을 교체된 element로 취급합니다.

프로필 공유는 현재 origin과 route handle을 조합한 URL을 clipboard에 기록하고 성공·실패 message를 표시합니다. clipboard 권한이나 secure context 실패에 대한 별도 recovery는 제공하지 않습니다.

### `a56a4dee9219` — URL이 아닌 논리 ID를 key로 사용

각 navigation item에 안정적인 `id`를 추가하고 key를 `href`에서 `id`로 바꿉니다.

```diff
-{ href: profileHref, label: "프로필", icon: UserRound, matchPrefix: "/profile" }
+{ id: "profile", href: profileHref, label: "프로필", icon: UserRound, matchPrefix: "/profile" }
```

```diff
-key={item.href}
+key={item.id}
```

프로필 destination이 세션에 따라 변해도 `profile`이라는 항목은 같은 element로 유지됩니다. 다른 항목에도 명시적 ID를 부여해 동일 규칙을 적용합니다.

다만 클릭 target 문제는 남아 있습니다. `me`가 아직 없을 때 프로필의 `href`는 여전히 `/`이므로, 안정적인 key만으로는 잘못된 이동을 막지 못합니다.

### `bc8d023b2999` — 사용자 식별 전에는 링크를 만들지 않음

마지막 fix는 프로필 item과 현재 사용자 사이의 precondition을 rendering branch로 표현합니다.

```tsx
if (item.id === "profile" && !me) {
  return (
    <span key={item.id} aria-disabled="true" className={className}>
      <Icon size={19} />
      {item.label}
    </span>
  );
}

return <Link key={item.id} href={item.href} className={className}>...</Link>;
```

이제 세션 해석 전 프로필 항목은 같은 visual slot과 stable key를 유지하지만 navigation capability는 갖지 않습니다. `/` 같은 임시 destination을 사용자에게 노출하지 않고, 사용자 식별이 끝난 뒤에만 실제 profile link가 만들어집니다.

이 구현은 `me === null`이 “loading”인지 “unauthenticated/error”인지 구분하지 않습니다. request가 영구 실패한 경우 프로필 항목은 계속 비활성 상태로 남습니다. session query의 상태 모델과 cache ownership은 Thread 06에서 다룹니다.

## 최종 책임 경계

| 대상 | 이 Thread 마지막 SHA의 owner | 보장 | 아직 이 Thread 밖인 것 |
| --- | --- | --- | --- |
| 문서 metadata·전역 CSS | `RootLayout` | 한국어 document root와 공통 style import | route별 server state |
| 공통 navigation/layout | `AppShell` | pathname active 표시, stable item ID, profile precondition | React Query 기반 session 공유, guest capability 정책 |
| 인증 진입 분기 | `HomePage` | `me` 유무에 따른 login/authenticated branch | loading/error/empty의 완전한 구분 |
| 개발 로그인 입력 | `LoginPanel` | 입력값, API 호출, 오류 message | submit pending·중복 요청 제어 |
| profile 공유 | `ProfilePage` | 현재 profile URL clipboard copy와 결과 message | clipboard 권한 recovery |
| browser smoke | Playwright suite | 주요 route 이동과 canvas pixel 존재 | 실제 API 성공·sample fallback 구분, realtime correctness |
| production 실행 | web package scripts | `next start`, package-local test command | deployment orchestration 전체 |

## 최종 흐름

```text
[Next.js/Tailwind runtime 설정]
    -> [RootLayout: metadata + globals.css]
    -> [HomePage: session 확인]
       ├─ me 없음 -> LoginPanel
       └─ me 있음 -> authenticated content / AppShell

[AppShell render]
    -> [pathname으로 active 계산]
    -> [navigation item은 stable id로 식별]
    -> [profile item]
       ├─ me 미확정/없음 -> aria-disabled span
       └─ me 있음 -> /profile/<handle> Link
```

이 Thread의 핵심 결과는 단순히 sidebar를 만든 것이 아닙니다. **navigation element의 정체성, 클릭 destination, 현재 사용자라는 선행 조건을 서로 다른 값과 branch로 분리했다는 것**이 최종 설계 결정입니다.

## 범위

이 문서는 다음 항목을 의도적으로 다루지 않습니다.

- HTTP adapter의 cookie credential·runtime schema·AbortSignal — Thread 02
- sample fallback 제거와 server-backed loading/error/empty state — Thread 03
- guest mode navigation 제한 — Thread 08
- `AppShell`의 독립 `getMe()`를 shared query cache로 이전하는 작업 — Thread 06
- realtime play connection과 canvas authority — Thread 04·05·07

모든 설명은 표시된 exact SHA의 diff와 당시 source에 한정했습니다. 이 환경에서는 repository를 로컬 checkout하거나 build·Playwright를 실행하지 않았습니다.
