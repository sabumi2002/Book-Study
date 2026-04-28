# 5장 유사한 코드 융합하기 (Part 1)

이번 파트에서 다룰 내용
- 유사 클래스 통합하기
- 단순한 조건 통합하기 (if 문 결합)
- 복잡한 조건 통합하기 (조건 산술)


# 유사한 클래스 통합하기

메서드가 상수를 반환할 때 그것을 **상수 메서드**라고 합니다. 두 클래스가 상수 메서드 집합만 다를 경우, 그 집합을 **기준(basis)** 이라 하며 두 클래스를 하나로 합칠 수 있습니다. 클래스를 합치는 것은 분수 더하기와 유사합니다. 분모를 같게 만든 뒤 더하듯이, 상수 메서드 이외의 모든 것을 동일하게 만든 뒤 통합합니다.

X개의 클래스를 통합하려면 최대 (X-1)개의 접점을 가진 기준이 필요합니다. 클래스 수가 줄어든다는 것은 구조를 더 잘 발견했다는 의미입니다.

### 리팩터링 패턴: 유사 클래스 통합

#### 설명
상수 메서드 집합(기준)만 다른 두 개 이상의 클래스를 생성자 매개변수로 통합합니다.

#### 절차
1. 두 클래스의 기준(다른 상수 메서드들)을 파악합니다.
2. 기준을 제외한 모든 메서드가 동일해질 때까지 리팩터링합니다.
3. 한 클래스에 기준 값들을 생성자 매개변수로 추가합니다.
4. 다른 클래스의 인스턴스화를 첫 번째 클래스로 교체하여 해당 상수 값을 전달합니다.
5. 이제 빈 클래스를 삭제합니다.

#### 예제

```typescript
// Before: 두 클래스는 getDeltaX()의 반환값만 다름
class Left {
  getDeltaX() { return -1; }
  handle(player: Player) { player.moveHorizontal(this.getDeltaX()); }
}
class Right {
  getDeltaX() { return 1; }
  handle(player: Player) { player.moveHorizontal(this.getDeltaX()); }
}

// After: 기준(deltaX)을 생성자 매개변수로 통합
class Horizontal {
  constructor(private deltaX: number) {}
  getDeltaX() { return this.deltaX; }
  handle(player: Player) { player.moveHorizontal(this.getDeltaX()); }
}
const left = new Horizontal(-1);
const right = new Horizontal(1);
```


# 단순한 조건 통합하기

코드를 완전히 이해하지 않아도 이 리팩터링을 수행할 수 있습니다. 이는 리팩터링 비용을 크게 줄여줍니다.

### 리팩터링 패턴: if 문 결합

#### 설명
내용이 동일한 연속된 if 문을 `||` 로 결합해 중복을 제거합니다. 두 조건의 관계를 명시적으로 드러낸다는 점에서 유용합니다.

#### 절차
1. 동일한 본문을 가진 연속된 두 if 문을 찾습니다.
2. 두 조건을 `||` 로 결합합니다.
3. 빈 if 문을 제거합니다.

#### 예제

```typescript
// Before: 본문이 동일한 연속 if 문
if (input instanceof Left) { player.updateUI(); }
if (input instanceof Right) { player.updateUI(); }

// After: 조건 결합
if (input instanceof Left || input instanceof Right) { player.updateUI(); }
```


# 복잡한 조건 통합하기

### 조건을 위한 산술 규칙 사용

조건 표현식이 무엇을 하는지 몰라도 수학 법칙으로 조작할 수 있습니다.

| 조건 연산자 | 산술 유사체 |
|---|---|
| `\|\|` | `+` (더하기) |
| `&&` | `×` (곱하기) |

모든 정규 산술 법칙(교환법칙, 결합법칙, 분배법칙, 드모르간)이 적용됩니다.
단, 이 규칙을 사용하려면 조건에 **부수적인 동작이 없어야** 합니다.

### 규칙: 순수 조건 사용

#### 정의
조건(if/while/for)은 항상 순수 조건이어야 합니다.

#### 설명
**순수(pure)** 란 조건에 부수적인 동작(변수 할당, 예외 발생, I/O)이 없음을 의미합니다.

부수적인 동작이 있으면:
- 산술 규칙을 적용할 수 없습니다.
- 조건에서 부수적인 동작을 예상하지 않기 때문에 버그를 찾기 어렵습니다.

**해결책:** 부수적인 동작의 반환과 분리합니다.
```typescript
// Bad: readLine()이 포인터를 전진시키는 부수 동작을 가짐
while ((line = readLine()) !== null) { ... }

// Good: 반환과 부수 동작 분리
while (hasMoreLines()) {
  let line = readLine();
  ...
}
```
제어 불가능한 구현이라면 **캐시**를 활용해 분리합니다.

#### 스멜
**명령에서 질의 분리(Separate queries from commands)**: void 메서드에서만 부수 동작을 허용합니다. 부수 동작을 하거나 무언가를 반환하거나 둘 중 하나만 합니다.

#### 의도
데이터를 가져오는 것과 변경하는 것을 분리해서 코드를 더 예측 가능하게 만들고, 부수 동작을 격리해 관리하기 쉽게 합니다.

### 조건 산술 적용

```typescript
// Before: 중복된 서브 표현식
if ((a && b) || (a && c)) { ... }

// 분배법칙 적용: a × (b + c) → a && (b || c)
if (a && (b || c)) { ... }
```

```typescript
// Before: 드모르간 법칙 적용
if (!a && !b) { ... }

// !(a || b)
if (!(a || b)) { ... }
```

조건을 수학 방정식으로 변환 → 단순화 → 코드로 복원하는 연습은 복잡한 조건의 버그를 찾는 데도 유용합니다.
