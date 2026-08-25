# Thread: 고정 크기 ContactBook의 교체 실패를 무해하게 만들기

Project: `cpp-foundation` · Branch: `cpp/cpp-foundation`

## 개요

`ContactBook`은 최대 8개의 연락처를 원형 배열에 저장하고, 가득 찬 뒤에는 가장 오래된 항목을 새 연락처로 교체합니다. 정상 경로에서는 slot 하나에 대입한 뒤 `next_`만 전진하면 됩니다. 문제는 `Contact`가 두 `std::string`을 가지므로 교체 대입 중 두 번째 allocation이 실패할 수 있다는 점입니다.

최종 불변 조건은 다음과 같습니다.

- 실패한 `add()`는 교체 대상 slot의 두 field를 모두 보존합니다.
- `size_`와 `next_`도 바뀌지 않아 logical order가 그대로 유지됩니다.
- 다음 성공한 `add()`는 실패 전에 예정되어 있던 동일한 oldest slot을 교체합니다.
- 새 연락처가 완전히 복사된 뒤에만 비예외 `swap()`으로 payload를 공개합니다.

| SHA | 제목 | 중요도 | 태그 | Thread 내 역할 |
| --- | --- | :---: | --- | --- |
| `2f9b934b0825` | `feat(contact): 고정 크기 연락처 저장 순서 보존` | B | `CORE` | capacity 8 ring과 logical oldest-to-newest 조회 구현 |
| `0ad14a57cab6` | `fix(contact): 할당 실패에도 저장 상태 보존` | A | `DEBUG, EXCEPTION, OWNERSHIP` | slot 직접 대입을 local replacement + swap으로 교체 |
| `8930c4d17bc1` | `test(contact): 연락처 교체 실패 회귀 검증` | A | `TEST, EXCEPTION, EDGE` | 모든 관찰된 allocation 실패 뒤 payload·순서·다음 교체 위치 검증 |

## `2f9b934b0825` — `feat(contact): 고정 크기 연락처 저장 순서 보존`

Representation은 고정 배열, 현재 저장 수, 다음 write slot으로 구성됩니다.

```cpp
class ContactBook
{
public:
    enum { capacity = 8 };

    void add(const Contact &contact);
    std::size_t size() const;
    const Contact &at(std::size_t logical_index) const;

private:
    Contact contacts_[capacity];
    std::size_t size_;
    std::size_t next_;
};
```

초기 `add()`는 empty contact를 무시하고 `next_` slot에 직접 대입합니다.

```cpp
void ContactBook::add(const Contact &contact)
{
    if (contact.empty())
        return;
    contacts_[next_] = contact;
    next_ = (next_ + 1) % capacity;
    if (size_ < capacity)
        ++size_;
}
```

Book이 아직 차지 않았다면 physical index 0부터 저장 순서가 곧 logical order입니다. Full 상태에서는 `next_`가 다음에 덮어쓸 oldest slot이므로 조회 시작점도 `next_`입니다.

```cpp
first = size_ == capacity ? next_ : 0;
return contacts_[(first + logical_index) % capacity];
```

예를 들어 A~H로 full이 된 상태에서 `next_ == 0`이고 I, J를 더하면 physical slot 0과 1이 바뀝니다. Logical order는 C, D, E, F, G, H, I, J가 됩니다.

### 정상 순서와 실패 안전성은 별개다

`Contact`는 `name_`, `note_` 두 `std::string`을 갖고 custom copy assignment를 정의하지 않습니다. Compiler-generated assignment는 두 member를 차례로 대입합니다. Name 대입이 성공하고 note 대입의 allocation이 실패하면 target slot은 다음처럼 될 수 있습니다.

```text
name_ = 새 이름
note_ = 이전 note
next_, size_ = 아직 이전 값
```

Metadata를 대입 뒤에 갱신했으므로 ring index는 보존되지만, slot payload는 부분적으로 바뀔 수 있습니다. “Index update를 나중에 한다”만으로 aggregate 전체의 strong guarantee가 되지 않는 이유입니다.

## `0ad14a57cab6` — `fix(contact): 할당 실패에도 저장 상태 보존`

Fix는 세 줄의 순서를 바꿉니다.

```diff
 void ContactBook::add(const Contact &contact)
 {
+    Contact replacement;
+
     if (contact.empty())
         return;
-    contacts_[next_] = contact;
+    replacement = contact;
+    contacts_[next_].swap(replacement);
     next_ = (next_ + 1) % capacity;
```

`Contact::swap()`은 두 string의 `swap()`을 호출하고 `throw()`로 선언되어 있습니다.

```cpp
void Contact::swap(Contact &other) throw()
{
    name_.swap(other.name_);
    note_.swap(other.note_);
}
```

새 순서는 다음과 같습니다.

```text
empty 여부 확인
  -> local replacement에 contact 복사   # name/note allocation 가능
  -> target slot과 replacement swap      # 비예외 payload commit
  -> next_ 전진
  -> 필요하면 size_ 증가
  -> replacement destructor가 이전 slot payload 정리
```

`replacement = contact`가 name 복사 뒤 note 복사에서 실패해도 손상되는 것은 곧 소멸할 local object뿐입니다. Target slot과 ring metadata에는 아직 접근하지 않았습니다.

Swap 성공 뒤에는 replacement가 target slot의 이전 payload를 소유합니다. Scope 종료 시 이전 string storage가 자동으로 해제되므로 성공 경로의 old-value cleanup도 별도 branch가 필요 없습니다.

이 commit은 payload commit과 metadata commit을 모두 실패 가능한 작업 뒤에 둡니다. Swap 이후 `size_t` 산술과 비교에는 예외 지점이 없으므로 payload만 바뀌고 index는 이전 상태인 중간 결과도 외부로 남지 않습니다.

## `8930c4d17bc1` — `test(contact): 연락처 교체 실패 회귀 검증`

Failure binary는 먼저 `one`부터 `eight`까지 넣어 book을 full로 만듭니다. Replacement는 최대 허용 길이인 32-byte name과 64-byte note를 사용해 두 string 모두 실제 allocation을 일으키도록 합니다.

정상 `add()`에서 발생한 allocation 수를 측정하고, 1번부터 마지막 allocation까지 각각 `std::bad_alloc`을 주입합니다.

각 실패 뒤 `checkSeed()`는 다음을 전부 검사합니다.

- `size() == 8`
- logical index 0~7의 name이 `one`~`eight`
- 모든 note가 `seed`
- live allocation block 수가 시도 전 baseline과 같음

이것만으로도 payload와 logical order 보존을 확인하지만, test는 한 단계 더 진행합니다. 같은 book에서 failure injection을 끈 뒤 replacement를 다시 성공시킵니다.

```text
실패 직후 expected next_ == 0
성공한 add 뒤 logical order:
  two, three, four, five, six, seven, eight, replacement
```

즉 다음 성공에서 `one`이 제거되고 `two`가 oldest가 되어야 합니다. 만약 실패 경로가 `next_`만 전진시켰다면 다른 slot이 교체되어 이 assertion이 깨집니다. Test는 private `next_`를 보지 않고 public logical order로 metadata 보존을 관찰합니다.

할당 실패의 exception type도 `std::bad_alloc` 그대로여야 하며 unexpected exception은 허용하지 않습니다. 각 반복과 전체 scope 뒤 live block baseline을 비교해 local partial `Contact`와 이전 slot storage가 모두 회수되는지도 확인합니다.

## 최종 상태

| 단계 | 실패 가능 | Public book 변화 |
| --- | --- | --- |
| Empty contact 확인 | 아니오 | 없음 |
| Local `replacement = contact` | 예 | 없음 |
| Slot `swap(replacement)` | 아니오 (`throw()`) | 새 payload 공개 |
| `next_`/`size_` 갱신 | 아니오 | logical order commit |
| Local replacement 소멸 | 아니오 | 이전 slot storage 해제 |

```text
before full book: [one ... eight], next=one
failed add:      [one ... eight], next=one
next success:    [two ... eight, replacement], next=two
```

이 Thread의 핵심은 원형 배열 계산이 아니라, **여러 allocation을 포함한 element replacement를 먼저 local value로 완성하고 slot과 metadata를 함께 commit 가능한 상태로 만든 것**입니다.

## Thread 경계와 검증 범위

- Capacity는 고정 8이며 resize나 concurrent access는 다루지 않습니다.
- Empty/invalid `Contact`를 만드는 field validation 자체는 이 Thread의 대상이 아닙니다.
- Test는 global `new`로 관찰된 allocation 지점을 sweep하지만 arbitrary allocator나 asynchronous failure는 다루지 않습니다.
- 각 exact SHA의 diff와 `Contact`/`ContactBook` source 및 failure test를 확인했습니다. 이 환경에서는 binary를 실행하지 않았습니다.
