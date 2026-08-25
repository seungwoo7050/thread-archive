# Thread: heredoc 의미를 parser에서 입력 경계 복구까지 보존하기

Project: `small-shell` · Branch: `c/minishell` · 문서 번호: 02

## 개요

Heredoc은 `<<EOF`를 알아보는 parser 기능 하나로 끝나지 않는다. 하나의 redirection은 다음 단계를 모두 통과한다.

```text
lexer의 delimiter word
  → parser의 redirection identity와 quote provenance
  → line 전체를 source order로 precollect
  → body의 선택적 변수 확장
  → temporary stream에 완전하게 저장
  → command 실행 시 stdin으로 설치
  → 성공·실패와 무관하게 body와 stream position 정리
```

이 과정에는 서로 다른 두 종류의 정보가 함께 이동한다.

- **정규화된 delimiter text**: 입력 종료 줄과 비교할 실제 문자열
- **quote provenance**: delimiter에 quote 문법이 사용되었는지 나타내는 의미 정보

두 값은 같지 않다. `'EOF'`, `"EOF"`, `E'OF'`는 모두 최종 delimiter text가 `EOF`이지만, body expansion을 막아야 한다는 사실은 원문에서 quote가 사용되었다는 별도 정보에 달려 있다.

이 Thread의 또 다른 중심은 실패 뒤의 입력 위치다. Heredoc 준비가 이미 stdin에서 여러 줄을 소비한 뒤 실패했다면, 메모리만 해제하고 반환해서는 안 된다. 남은 delimiter까지 소비하지 않으면 body였던 줄이 다음 command로 해석된다. 특히 body를 읽는 `shell_read_line` 자체가 실패하고, 같은 heredoc의 delimiter를 찾기 위한 즉시 복구 read도 실패하면 future command boundary를 신뢰할 수 없으므로 shell을 계속 실행하지 않는다.

### 최종 불변 조건

| 대상 | 최종 규칙 |
| --- | --- |
| delimiter text | quote marker를 제거한 별도 owned string으로 비교한다 |
| quote 의미 | lexer가 기록한 `quoted`를 parser가 `heredoc_quoted`로 보존한다 |
| body identity | delimiter 문자열이 아니라 해당 `t_redir *`로 찾는다 |
| 수집 순서 | connector gate와 무관하게 line 전체의 heredoc을 source order로 먼저 읽는다 |
| body expansion | `heredoc_quoted == 0`일 때만 변수와 `$?`를 확장한다 |
| stdin publish | write·flush·seek·fileno가 모두 성공한 temporary stream만 `dup2`한다 |
| 준비 실패 | 현재 heredoc과 뒤에 남은 heredoc 구간을 끝 delimiter까지 소비하려 하며, 정상 read가 계속되는 한 다음 command boundary를 복구한다 |
| body input read failure | EOF와 구분해 status 1로 보고하며, 같은 heredoc의 즉시 delimiter 복구 read도 실패하면 `running = 0`으로 중단한다 |

## 커밋 지도

기존 Thread map은 마지막 read-failure test만 포함하고 그 production contract를 도입한 commit을 빠뜨리고 있었다. Branch history와 commit importance 자료를 대조해 `2d3791748571`을 `c30b39c0bcf8`와 `7e2fdea3affd` 사이에 추가했다. 중요도와 태그는 repository의 기존 분류를 그대로 사용한다.

| 순서 | SHA | 제목 | 중요도 | 태그 | 이 Thread에서의 역할 |
| ---: | --- | --- | :---: | --- | --- |
| 1 | `e65591bb66f5` | `feat(heredoc): 구분자 정규화 버퍼 구현` | B | `HEREDOC`, `PRACTICAL` | encoded delimiter에서 quote marker만 제거하는 전용 경로 도입 |
| 2 | `7c9692346824` | `feat(heredoc): 수집 본문 저장소 수명 관리` | A | `HEREDOC`, `ARCH`, `RISK` | body를 owning redirection node identity에 연결 |
| 3 | `fc9c63a03db2` | `feat(heredoc): 구분자별 본문 순차 수집` | A | `HEREDOC`, `CORE`, `INTEGRATION` | line의 모든 heredoc body를 source order로 수집 |
| 4 | `aeb0d6cba9c1` | `feat(heredoc): 인용 여부에 따라 본문 확장` | A | `HEREDOC`, `EXPANSION`, `CORE` | quoted/unquoted delimiter에 따른 body expansion 분기 도입 |
| 5 | `d297bd2e8908` | `feat(redirection): heredoc을 stdin으로 연결` | S | `HEREDOC`, `FD_IO`, `INTEGRATION` | syntax·precollection·lifetime·redirection order·stdin 설치를 product path에 통합 |
| 6 | `854f0f435c82` | `fix(heredoc): 구분자의 인용 상태를 실행 단계까지 보존` | S | `HEREDOC`, `DEBUG`, `RISK` | text heuristic을 버리고 lexical quote provenance를 명시적으로 전달 |
| 7 | `dce9e5c083fa` | `test(heredoc): 이중·부분 인용 구분자 회귀 검증` | A | `TEST`, `HEREDOC`, `EDGE` | double quote와 partial quote의 no-expansion 의미를 고정 |
| 8 | `9afdca85f5a5` | `fix(heredoc): 임시 파일 저장 오류를 전파` | A | `HEREDOC`, `FD_IO`, `FAILURE` | 일부만 저장된 body가 stdin으로 공개되는 경로 차단 |
| 9 | `2fbc4c73af2c` | `test(heredoc): 임시 저장 실패의 데이터 절단 방지 검증` | A | `TEST`, `HEREDOC`, `FAILURE` | flush·seek 실패를 주입해 status와 continuation을 검증 |
| 10 | `c30b39c0bcf8` | `fix(heredoc): 준비 실패 뒤 입력 구분자 경계 복구` | A | `HEREDOC`, `FAILURE`, `RISK` | 실패 뒤 남은 heredoc 구간을 소비해 다음 command boundary를 복구 |
| 11 | `2d3791748571` | `fix(input): EOF와 입력 실패를 구분` | A | `FAILURE`, `SHELL_STATE`, `EDGE` | 입력 API에 explicit failure channel을 추가하고 복구 불가능하면 shell을 중단 |
| 12 | `7e2fdea3affd` | `test(io): read·write와 heredoc 입력 실패 검증` | A | `TEST`, `FAILURE`, `HEREDOC` | one-shot/multiple/persistent read failure에서 복구·계속·중단을 고정 |

## 1. delimiter text와 body identity를 먼저 분리한다

### `e65591bb66f5` — ordinary expansion과 다른 정규화 경로

**중요도** `B` · **태그** `HEREDOC, PRACTICAL`

Lexer는 single-quoted byte 앞에 내부 marker를 넣는다. Heredoc delimiter를 실제 입력 줄과 비교하려면 marker는 제거해야 하지만, `$NAME`이나 `$?`를 ordinary word처럼 확장해서는 안 된다. 이 커밋은 그 둘을 분리한 전용 dequote buffer를 만든다.

핵심 loop는 marker와 뒤 byte를 한 단위로 소비하고 실제 byte만 append한다. marker가 없는 문자는 그대로 복사한다.

```c
if (word[i] == LITERAL_MARK && word[i + 1] != '\0') {
    if (sb_append_char(&out, word[i + 1]) != 0)
        goto fail;
    i += 2;
} else {
    if (sb_append_char(&out, word[i]) != 0)
        goto fail;
    i++;
}
```

예를 들어 parser가 가진 encoded target이 다음과 같다면,

```text
MARK 'E' MARK 'O' MARK 'F' '\0'
```

정규화 결과는 새로 할당된 `"EOF"`다. 원래 parsed target은 성공 전까지 그대로 남고, buffer 생성 중 실패하면 partial output만 해제한다.

이 시점에는 delimiter text만 얻을 수 있다. quote 문법이 single quote였는지 double quote였는지, 일부 fragment만 인용되었는지는 아직 별도 상태로 남지 않는다. 이 결함은 `854f0f435c82`에서 드러난다.

### `7c9692346824` — 같은 문자열보다 redirection node가 더 정확한 key다

**중요도** `A` · **태그** `HEREDOC, ARCH, RISK`

동일한 delimiter text가 여러 번 등장할 수 있으므로 body 저장소를 text-keyed map으로 만들면 안 된다.

```sh
cat <<EOF <<EOF
first
EOF
second
EOF
```

두 heredoc의 delimiter text는 모두 `EOF`지만, 첫 body와 두 번째 body는 서로 다른 redirection이 소유한다. 이 커밋은 execution context에 다음 형태의 entry list를 둔다.

```c
struct heredoc_entry {
    const t_redir          *redir;
    char                   *body;
    struct heredoc_entry   *next;
};
```

lookup은 문자열 비교가 아니라 pointer identity를 사용한다.

```c
while (entry != NULL) {
    if (entry->redir == redir)
        return entry->body;
    entry = entry->next;
}
```

이 선택은 parsed tree의 lifetime과 직접 연결된다.

- `t_redir` node는 command line 실행이 끝날 때까지 주소가 안정적이다.
- entry는 body 문자열을 소유하지만 redirection 자체는 빌리지 않는다.
- 따라서 **heredoc entry list를 먼저 해제한 뒤 parsed tree를 해제**해야 한다.

Body 생성이 실패하거나 entry allocation이 실패하면 아직 list에 연결되지 않은 body는 local owner가 정리한다. 성공한 entry만 `ctx->heredocs`에 publish된다.

## 2. 실행 여부보다 먼저 source order대로 입력을 소비한다

### `fc9c63a03db2` — connector gate보다 앞선 precollection

**중요도** `A` · **태그** `HEREDOC, CORE, INTEGRATION`

Heredoc body는 command 실행 중에 즉석으로 읽을 수 없다. 같은 입력 stream에 command line 다음으로 body가 배치되므로, line의 parsed structure를 얻은 직후 모든 heredoc을 source order대로 먼저 소비해야 한다.

```text
pipeline list order
  → 각 pipeline의 command order
     → 각 command의 redirection order
        → REDIR_HEREDOC마다 body read
```

이 순서는 connector gating보다 앞선다. 다음 입력에서 `false && cat <<EOF`의 `cat`은 실행되지 않더라도 `body`와 종료 delimiter는 입력 stream에서 제거되어야 한다.

```sh
false && cat <<EOF
body
EOF
echo after
```

실행 여부를 먼저 판단하고 skipped pipeline의 heredoc을 읽지 않으면 `body` 또는 `EOF`가 다음 top-level command line으로 흘러간다. 따라서 heredoc precollection은 runtime short-circuit와 독립된 **source consumption 의무**다.

수집 loop는 delimiter에 도달할 때까지 line과 newline을 body buffer에 append하고, 정상 EOF에서는 warning을 남긴 뒤 지금까지의 body를 사용한다. 이 SHA의 input API는 아직 EOF와 실제 read failure를 구분하지 못한다.

### `aeb0d6cba9c1` — body expansion policy를 추가하지만 provenance는 아직 불완전하다

**중요도** `A` · **태그** `HEREDOC, EXPANSION, CORE`

Unquoted delimiter의 body는 변수와 `$?`를 확장하고, quoted delimiter의 body는 literal text로 남겨야 한다. 이 커밋은 body line을 append할 때 두 경로를 나눈다.

```text
quoted delimiter   → line을 그대로 append
unquoted delimiter → shell state로 line을 expand한 뒤 append
```

하지만 quoted 여부를 encoded delimiter text에 literal marker가 있는지로 추론한다. Single quote는 marker를 남기므로 동작하지만, 이 프로젝트의 double-quote encoding은 내부 문자를 marker 없이 보존한다. 따라서 다음 두 입력의 최종 delimiter text는 같고, heuristic도 둘을 구별하지 못한다.

```sh
cat <<EOF
$HOME
EOF

cat <<"EOF"
$HOME
EOF
```

첫 body는 확장되어야 하고 두 번째 body는 literal `$HOME`이어야 한다. Text만 보고 quote provenance를 복원하려는 시도 자체가 부족한 것이다.

## 3. heredoc을 실제 redirection lifecycle에 통합한다

### `d297bd2e8908` — syntax에서 stdin 설치까지 하나의 수명으로 연결

**중요도** `S` · **태그** `HEREDOC, FD_IO, INTEGRATION`

이 커밋은 앞선 helper들을 실제 shell path에 연결한다. Diff에는 다른 formatting 변경도 섞여 있지만, 이 Thread와 직접 관련된 변화는 다음과 같다.

1. lexer가 `<<`를 `TOK_HEREDOC`으로 만든다.
2. parser가 target word를 가진 `REDIR_HEREDOC` node를 만든다.
3. `shell_process_line`이 실행 전에 `exec_prepare_heredocs`를 호출한다.
4. body entry는 같은 command line의 `exec_context`가 소유한다.
5. redirection 적용 시 해당 `t_redir *`로 body를 찾는다.
6. body를 temporary stream에 쓰고 처음으로 되감은 뒤, 그 descriptor를 stdin에 `dup2`한다.
7. 실행이 끝나면 entry list를 parsed tree보다 먼저 해제한다.

```text
parse complete
  ↓
prepare all heredocs
  ↓
connector-gated execution
  ↓
free heredoc entries
  ↓
free parsed pipelines
```

### redirection 순서와 “마지막 입력 redirection이 이긴다”

Heredoc도 command의 ordered redirection list에 들어간다. 따라서 ordinary input redirection과 동일하게 source order대로 적용된다.

```sh
cat < file <<EOF
body
EOF
```

마지막 `<<EOF`가 stdin을 덮는다. 반대로 `cat <<EOF < file`이면 마지막 `< file`이 heredoc stdin을 덮는다. Body는 line 전체를 위해 이미 precollect되었지만, 실제 stdin binding은 command의 redirection order를 따른다.

### temporary stream의 publish 지점

Body는 먼저 private temporary stream에 저장된다. `dup2`가 성공한 순간에만 command stdin으로 공개된다. 이 단계의 정확한 실패 원자성은 아직 완성되지 않아 `9afdca85f5a5`가 보강한다.

## 4. text heuristic 대신 provenance를 직접 전달한다

### `854f0f435c82` — root cause fix

**중요도** `S` · **태그** `HEREDOC, DEBUG, RISK`

#### 문제

`aeb0d6cba9c1`은 encoded delimiter에 literal marker가 있으면 quoted라고 판단했다. 이 표현은 single quote만 감지하며 double quote나 partial quote를 놓친다.

```sh
cat <<"EOF"     # marker가 없어 unquoted로 오판 가능
cat <<E'OF'      # marker는 일부 fragment에만 존재
```

#### 원인

최종 text에서 source-level syntax provenance를 재구성하려 했다. Dequote가 끝난 문자열에는 quote delimiter가 이미 사라져 있으므로, 모든 quote 형태를 안정적으로 복원할 수 없다.

#### 결정

Lexer가 word를 읽을 때 quote delimiter를 한 번이라도 통과했는지 `quoted`에 기록하고, parser가 heredoc target token의 값을 redirection node에 복사한다.

```c
/* token */
int quoted;

/* redirection */
int heredoc_quoted;
```

```c
node->heredoc_quoted =
    (type == REDIR_HEREDOC && target_quoted);
```

실행 단계는 encoded text를 다시 검사하지 않고 이 field를 직접 읽는다.

```c
quoted = redir->heredoc_quoted;
```

이 변경 뒤 delimiter normalization과 body expansion policy가 명확히 분리된다.

| 입력 delimiter | normalized text | `heredoc_quoted` | body `$NAME` |
| --- | --- | ---: | --- |
| `EOF` | `EOF` | 0 | 확장 |
| `'EOF'` | `EOF` | 1 | literal |
| `"EOF"` | `EOF` | 1 | literal |
| `E'OF'` | `EOF` | 1 | literal |

이 fix는 Thread 01의 일반 word marker contract를 없애지 않는다. 일반 expansion에는 문자 단위 marker가 필요하고, heredoc policy에는 word 전체의 quote provenance가 필요하다. 서로 다른 질문이므로 서로 다른 표현을 유지한다.

### `dce9e5c083fa` — single quote로는 잡히지 않던 회귀를 고정

**중요도** `A` · **태그** `TEST, HEREDOC, EDGE`

Test는 double-quoted delimiter와 partially quoted delimiter의 body에 변수 형태의 text를 넣고, 확장되지 않은 literal 값이 출력되는지 확인한다.

대표 경계는 다음과 같다.

```sh
cat <<"EOF"
$HEREDOC_VALUE
EOF
```

```sh
cat <<E'OF'
$HEREDOC_VALUE
EOF
```

Ordinary unquoted case도 함께 있어 “모든 heredoc expansion을 꺼 버리는” 잘못된 fix를 막는다. 이 test가 고정하는 것은 quote provenance에 따른 expansion 분기다. 같은 delimiter가 여러 번 등장하는 identity 문제나 temporary stream 실패는 별도 commit들이 담당한다.

## 5. 일부만 저장된 body를 stdin으로 공개하지 않는다

### `9afdca85f5a5` — temporary stream을 작은 transaction으로 만든다

**중요도** `A` · **태그** `HEREDOC, FD_IO, FAILURE`

이전 경로는 body를 temporary stream에 쓴 뒤 `fflush`와 `fseek`의 결과를 충분히 확인하지 않았다. 다음 문제가 생길 수 있다.

- `fwrite` 또는 write가 중간에 실패했는데 일부 body만 남는다.
- buffered bytes의 `fflush`가 실패했는데 성공한 것처럼 진행한다.
- rewind `fseek`가 실패해 stdin이 body 시작이 아닌 위치를 가리킨다.
- `fileno`가 유효 descriptor를 주지 못했는데 `dup2`를 시도한다.

Fix는 공개 순서를 다음처럼 고정한다.

```text
create temp stream
  → write complete body
  → fflush 성공
  → fseek(..., 0, SEEK_SET) 성공
  → fileno 성공
  → dup2(fd, STDIN_FILENO)
  → close stream
```

어느 단계든 실패하면 diagnostic과 status 1을 반환하고 `dup2`에 도달하지 않는다. 따라서 command는 truncated 또는 잘못 positioned된 stdin을 관찰하지 않는다.

이 결정은 body 저장을 “best effort” 출력이 아니라 **command input을 publish하기 전의 검증 단계**로 다룬다.

### `2fbc4c73af2c` — flush와 seek를 결정적으로 실패시킨다

**중요도** `A` · **태그** `TEST, HEREDOC, FAILURE`

Runtime wrapper에 `fflush`와 `fseek` failure seam을 추가하고, 각각 선택한 호출 순번에서 실패하도록 한다. Regression은 다음 두 가지를 함께 본다.

- 실패한 heredoc command의 status는 `1`이다.
- 입력 stream 경계는 손상되지 않아 뒤의 `echo after`가 정상 실행된다.

즉 “오류를 반환했다”만으로는 충분하지 않다. Truncated body가 실행되지 않았고, 실패가 다음 command까지 오염시키지 않았음을 같은 process에서 확인한다.

이 commit은 temporary storage의 failure만 검증한다. Raw input read failure는 아직 deterministic하게 구분할 수 없으며 `2d3791748571`과 `7e2fdea3affd`가 담당한다.

## 6. 준비 실패 뒤에도 다음 command boundary를 지킨다

### `c30b39c0bcf8` — 메모리 cleanup만으로는 복구가 끝나지 않는다

**중요도** `A` · **태그** `HEREDOC, FAILURE, RISK`

Heredoc preparation은 side effect를 가진다. Body line을 읽는 순간 stdin position이 앞으로 이동한다. Delimiter normalization, body buffer growth, expansion, entry allocation 중 하나가 실패하면 현재 redirection object를 정리하는 것만으로는 부족하다.

예를 들어 다음 입력에서 body append가 `first`를 읽은 뒤 실패했다고 하자.

```sh
cat <<ONE <<TWO
first
ONE
second
TWO
echo after
```

그 자리에서 command processing을 끝내면 남아 있는 `ONE`, `second`, `TWO`가 top-level command reader로 넘어간다. Data가 code로 재해석되는 입력 경계 손상이다.

Fix는 두 수준의 drain을 추가한다.

1. 현재 heredoc 준비가 중간에 실패하면 해당 delimiter까지 `discard_heredoc`으로 소비한다.
2. `exec_prepare_heredocs`는 failed flag를 유지하면서 뒤의 모든 heredoc도 각 delimiter까지 소비한다.

Delimiter matching은 allocation 없이 encoded target과 input line을 비교할 수 있도록 literal marker를 건너뛴다. 이미 allocation failure가 발생한 상황에서 복구를 위해 또 delimiter string을 할당하면 같은 실패를 반복할 수 있기 때문이다.

```c
static int delimiter_matches(const char *line, const char *encoded)
{
    /* encoded의 LITERAL_MARK는 건너뛰고 실제 byte만 line과 비교 */
}
```

이 SHA에서 `discard_heredoc`은 EOF와 read error를 아직 같은 `NULL`로 본다. “끝까지 버렸다”와 “버리는 도중 I/O가 실패했다”를 구별할 수 없으므로, 복구 가능성 판단은 다음 commit에서 완성된다.

### `2d3791748571` — `NULL` 하나로 EOF와 실패를 표현하지 않는다

**중요도** `A` · **태그** `FAILURE, SHELL_STATE, EDGE`

이 commit은 기존 Thread map에 없었지만 `7e2fdea3affd`의 heredoc read-failure regression을 설명하는 production change다.

#### 입력 API 변경

```diff
-char *shell_read_line(const char *prompt, int interactive);
+char *shell_read_line(const char *prompt, int interactive, int *failed);
```

반환값 `NULL`은 여전히 “line 없음”을 뜻하지만, `*failed`가 의미를 분리한다.

| 반환 | `failed` | 의미 |
| --- | ---: | --- |
| non-NULL | 0 | 정상 line |
| NULL | 0 | 정상 EOF |
| NULL | 1 | read 또는 allocation failure |

Non-interactive reader는 stdio의 `fgetc` 대신 runtime-wrapped `read`를 사용한다. `EINTR`은 retry하고, 다른 negative result는 `failed = 1`로 전파한다. Buffer doubling 전에는 `SIZE_MAX` overflow도 검사한다.

```c
count = shell_read(STDIN_FILENO, &ch, 1);
if (count < 0 && errno == EINTR)
    continue;
if (count < 0) {
    free(line);
    *failed = 1;
    return NULL;
}
```

#### heredoc에서의 세 갈래

```text
normal delimiter → body publish
normal EOF       → warning, collected prefix 사용
read failure     → diagnostic, current delimiter까지 recovery 시도
                    ├─ recovery 성공: command status 1, shell 계속 가능
                    └─ recovery도 실패: stream boundary 불명 → running = 0
```

```c
if (line == NULL && input_failed) {
    fprintf(stderr, "small-shell: heredoc input: %s\n", strerror(errno));
    if (discard_heredoc(redir->target, interactive) != 0)
        ctx->shell->running = 0;
    sb_free(&body);
    return 1;
}
```

복구용 `discard_heredoc`도 성공/실패를 반환한다. 다만 이 SHA에서 `running = 0`을 설정하는 곳은 `read_heredoc`이 body read failure를 받은 뒤 같은 delimiter를 즉시 복구하는 호출이 다시 실패한 branch다. Delimiter normalization·body buffer·body append 실패 뒤 호출되는 discard는 반환값을 버리고, 이미 failed state에서 뒤 heredoc을 버리다 실패한 경우도 `failed`만 유지한다. 따라서 “모든 drain 실패가 shell 중단으로 이어진다”는 더 넓은 보장은 이 commit에서 성립하지 않는다.

Top-level `shell_loop` 역시 EOF와 input failure를 분리해, 후자에는 diagnostic과 status 1을 남긴다. 다만 top-level command line 자체를 읽지 못한 경우에는 안전하게 계속할 다음 line이 없으므로 loop를 종료한다.

### `7e2fdea3affd` — one-shot recovery와 persistent failure를 분리한다

**중요도** `A` · **태그** `TEST, FAILURE, HEREDOC`

이 commit의 production 변경은 `shell_read`/`shell_write` wrapper에 test-only failure injection을 추가하는 것이다. 같은 diff에 일반 output failure case도 있지만, 이 Thread에는 heredoc input과 직접 연결된 사례만 포함한다.

#### 단일 heredoc read failure

선택한 read call 한 번만 실패시킨다. `read_heredoc`은 오류를 보고하고 delimiter까지 다시 읽어 경계를 복구한다. 기대 결과는 다음과 같다.

```text
failed heredoc status = 1
following echo $?     = 1
following echo after  = 정상 출력
```

#### 여러 heredoc 중 read failure

두 heredoc의 일부를 이미 소비한 위치에서 failure를 주입한다. 현재 delimiter와 뒤의 delimiter를 모두 drain해야 `echo after`가 command로 남는다. 이 case는 `c30b39c0bcf8`의 failed flag와 source-order discard를 함께 통과한다.

#### persistent read failure

`SMALL_SHELL_FAIL_READ_REPEAT`로 선택한 call 이후 모든 read를 실패시킨다. 최초 body read뿐 아니라 recovery read도 실패하므로 command boundary를 복구할 수 없다.

```sh
cat <<EOF
body
EOF
echo never
```

기대 결과는 exit status `1`, stdout empty이며 `echo never`가 실행되지 않는 것이다. 이는 `2d3791748571`의 `shell->running = 0` 분기를 직접 겨냥한다.

### 테스트가 증명하는 것과 남는 범위

증명하는 것:

- 선택된 one-shot read failure에서 delimiter boundary를 복구하고 다음 command를 실행한다.
- 여러 heredoc에서도 남은 body 구간을 source order대로 소비한다.
- recovery read까지 반복 실패하면 residual input을 실행하지 않고 shell을 중단한다.

증명하지 않는 것:

- 모든 possible errno와 모든 read call 위치
- interactive Readline 내부의 실패 주입
- 무한한 heredoc 크기나 외부 signal 조합 전체

## failure → decision → evidence 정리

| 실패 또는 잘못된 가정 | root cause | 수정 결정 | 회귀 증거 |
| --- | --- | --- | --- |
| same text delimiter의 body가 섞일 수 있음 | text를 identity로 사용 | `t_redir *`를 entry key로 사용 | multiple heredoc path |
| `"EOF"` body가 확장됨 | encoded text에서 quote를 재추론 | lexer `quoted` → parser `heredoc_quoted` | `dce9e5c083fa` double/partial quote |
| 일부 body만 저장된 stream이 stdin이 됨 | flush/seek 결과를 무시 | write·flush·seek·fileno 성공 뒤에만 `dup2` | `2fbc4c73af2c` fault cases |
| 준비 실패 뒤 body line이 command로 실행됨 | memory cleanup만 하고 stdin position을 방치 | 현재·후속 delimiter까지 drain | allocation sweep, multiple heredoc read fault |
| EOF warning과 read error가 같은 결과로 처리됨 | `NULL` 하나뿐인 input API | explicit `failed` channel | `7e2fdea3affd` read cases |
| body read failure 직후 같은 delimiter를 찾는 recovery read도 실패 | future command boundary를 신뢰할 근거 없음 | 해당 branch에서 `running = 0` | persistent read failure |

## ownership과 publish 지점

| 자원 | 준비 중 owner | publish 지점 | 정상 종료 | 실패 종료 |
| --- | --- | --- | --- | --- |
| normalized delimiter | `read_heredoc` local | `redir->target` 교체 | parsed tree가 해제 | local 또는 redir cleanup |
| body buffer | `read_heredoc` local | successful entry 생성 | entry destructor가 해제 | `sb_free` |
| heredoc entry | `exec_context` | list tail 연결 | pipeline tree보다 먼저 해제 | publish 전이면 local cleanup |
| temporary stream | redirection helper | 모든 storage/position check 뒤 stdin `dup2` | stream close, duplicated stdin은 command lifecycle | `dup2` 전 close하고 status 1 |
| stdin position | input subsystem | delimiter 소비가 완료된 시점 | next command boundary 유지 | drain 시도; body read failure 직후의 즉시 recovery도 실패한 branch에서는 shell 중단 |

## 최종 흐름

```text
[tokenize / parse]
  - TOK_HEREDOC + target word
  - target quote provenance를 heredoc_quoted에 복사
  ↓
[exec_prepare_heredocs: line 전체, source order]
  - encoded target을 dequote하여 delimiter text 생성
  - delimiter까지 body line 수집
  - quoted? literal append : variable/status expansion
  - (t_redir*, body) entry publish
  - 실패 시 현재/남은 delimiter 구간 drain
  - body read failure 뒤 즉시 delimiter 복구 read도 실패하면 running=0
  ↓
[connector gate 후 selected pipeline 실행]
  ↓
[ordered redirection 적용]
  - owning t_redir*로 body lookup
  - temp stream write → flush → rewind → fileno
  - 모두 성공한 경우에만 dup2(stdin)
  ↓
[command 완료]
  - heredoc entries 해제
  - parsed pipelines 해제
```

## 이 Thread의 경계

- Token·command·pipeline·connector의 일반 표현과 지연 expansion은 Thread 01에 속한다.
- Temporary stream의 `dup2`, parent stdio 복원 등 일반 descriptor ownership은 Thread 03에서 더 넓게 다룬다.
- Heredoc allocation failure가 command-level failure가 되는 전역 정책은 Thread 04이며, 이 문서는 그 실패 뒤 **입력 위치 복구**에 집중한다.
- Body와 delimiter를 조립하는 growable buffer의 점근적 개선은 Thread 05에 속한다.
- 같은 `7e2fdea3affd`에 포함된 일반 stdout write failure case는 heredoc input boundary와 무관하므로 여기서 제외했다.

### 검증 범위

표시된 각 SHA의 diff와 해당 시점 source를 `c/minishell` branch에서 확인했다. Repository를 로컬 checkout할 수 없어 build와 test suite는 다시 실행하지 않았으며, 위 test 결과는 source가 요구하는 assertion으로만 기술했다.
