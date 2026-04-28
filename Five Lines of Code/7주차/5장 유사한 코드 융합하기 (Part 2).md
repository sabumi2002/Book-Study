# 5장 유사한 코드 융합하기 (Part 2)

이번 파트에서 다룰 내용
- 클래스 간의 코드 통합 (전략 패턴)
- 구현체가 하나뿐인 인터페이스를 만들지 말 것
- 유사 함수 통합하기
- 요약


# 클래스 간의 코드 통합

### 클래스 관계를 묘사하기 위한 UML 클래스 다이어그램 소개

아키텍처나 동작 순서 같은 코드의 속성을 전달할 때 UML 클래스 다이어그램을 사용합니다.

- **클래스**: 상자 + 이름 (+ 메서드). 필드는 거의 포함하지 않음.
- **인터페이스**: 클래스와 동일하게 표시하되, 이름 위에 `«interface»` 표기.
- **상속/구현**: 속이 빈 화살표로 표현.
- `public`/`private` 여부를 `+`/`-` 기호로 나타내기도 함.

### 리팩터링 패턴: 전략 패턴의 도입

#### 설명
**전략 패턴**: 다른 클래스를 인스턴스화해서 변형(variance)을 도입하는 개념입니다.

- 전략이 필드를 가지면 **상태 패턴(state pattern)** 이라고도 합니다.
- 인터페이스 없이도 전략 패턴을 적용할 수 있습니다. 인터페이스는 변형이 여러 개 필요할 때 추가합니다(`구현에서 인터페이스 추출` 패턴 활용).
- 전략 클래스는 데이터를 나타내는 게 아니라 행동을 캡슐화하므로, 완성 후 메서드를 추가하기보다 새 클래스를 만드는 것을 선호합니다.
- 변형은 항상 **인터페이스(또는 추상 클래스 아닌 인터페이스)** 로부터 상속받습니다.

#### 절차
1. 분리하려는 코드에 **메서드 추출**을 수행합니다. 통합하려는 대상과 메서드가 동일한지 확인합니다.
2. 새로운 클래스를 만듭니다.
3. 생성자에서 새로운 클래스를 인스턴스화합니다.
4. 추출한 메서드를 새로운 클래스로 옮깁니다.
5. 필드에 종속성이 있으면:
   1. 필드를 새 클래스로 옮기고 접근자(accessor)를 만듭니다.
   2. 원래 클래스에서 새 접근자를 통해 오류를 바로잡습니다.
6. 새 클래스의 나머지 오류에 대해 매개변수를 추가합니다.
7. **메서드 인라인화**로 1단계의 추출을 되돌립니다.

#### 예제

```typescript
// Before: 변형 로직이 GameController 안에 인라인
class GameController {
  handleLeft(player: Player) {
    player.moveHorizontal(-1);
    player.updateAnimation();
  }
  handleRight(player: Player) {
    player.moveHorizontal(1);
    player.updateAnimation();
  }
}

// After: 변형을 전략 클래스로 분리
class LeftHandler {
  handle(player: Player) {
    player.moveHorizontal(-1);
    player.updateAnimation();
  }
}
class RightHandler {
  handle(player: Player) {
    player.moveHorizontal(1);
    player.updateAnimation();
  }
}
```

이렇게 하면 새로운 변형 추가 시 `GameController` 를 수정하지 않고 새 클래스만 만들면 됩니다.


### 규칙: 구현체가 하나뿐인 인터페이스를 만들지 말 것

#### 정의
구현체가 하나뿐인 인터페이스를 사용하지 마십시오.

#### 설명
- 구현 클래스 하나뿐인 인터페이스는 **변형을 전제로 하는데 변형이 없으므로** 오히려 가독성을 방해합니다.
- 구현 클래스를 수정하면 인터페이스도 수정해야 하는 오버헤드가 발생합니다.
- 많은 언어에서 인터페이스를 별도 파일에 작성하므로, 남용하면 파일 수가 두 배로 늘어납니다.

**예외**: 익명 클래스나 익명 내부 클래스(비교자 등)를 사용하는 경우처럼, 구현체가 사실상 여러 개 존재하는 특수한 경우는 괜찮습니다.

#### 스멜
> '추상화는 인지된 복잡성의 감소를 위해 실제의 복잡성의 증가를 허용하는 것이다'

인터페이스는 상용구(boilerplate)의 대표적인 원인입니다. 항상 바람직하다고 배웠기 때문에 특히 위험하며, 불필요한 일반화를 유발합니다.

#### 의도
불필요한 코드 상용구를 제한합니다. 인터페이스는 변형이 실제로 필요할 때 추가합니다.


### 리팩터링 패턴: 구현에서 인터페이스 추출

#### 설명
인터페이스가 필요해지는 시점(변형이 추가될 때)까지 인터페이스 생성을 미룰 수 있어 유용합니다.

#### 절차
1. 추출할 클래스와 동일한 이름으로 새 인터페이스를 만듭니다.
2. 기존 클래스의 이름을 변경하고 새 인터페이스를 구현하게 합니다.
3. 컴파일하고 오류를 검토합니다.
   1. `new` 때문에 오류가 발생하면 인스턴스화 부분을 새로운 클래스 이름으로 변경합니다.
   2. 그렇지 않으면 오류를 일으키는 메서드를 인터페이스에 추가합니다.

#### 예제

```typescript
// Before: 클래스만 존재
class RightHandler {
  handle(player: Player) { player.moveHorizontal(1); }
}

// After: 두 번째 변형(LeftHandler)이 필요해진 시점에 인터페이스 추출
interface InputHandler {           // 1. 동일 이름으로 인터페이스 생성
  handle(player: Player): void;
}
class RightHandlerImpl implements InputHandler {  // 2. 기존 클래스 이름 변경
  handle(player: Player) { player.moveHorizontal(1); }
}
class LeftHandlerImpl implements InputHandler {   // 3. 새 변형 추가
  handle(player: Player) { player.moveHorizontal(-1); }
}
```


# 유사 함수 통합하기

구조는 같지만 세부 동작만 다른 두 함수는 전략 패턴(또는 함수형 언어의 고차 함수)으로 통합할 수 있습니다.

```typescript
// Before: 구조가 동일하고 동작만 다른 두 함수
function drawRedCircle(x: number, y: number) {
  setColor('red');
  drawCircle(x, y);
}
function drawBlueCircle(x: number, y: number) {
  setColor('blue');
  drawCircle(x, y);
}

// After: 전략(color)을 매개변수로 분리
function drawColoredCircle(x: number, y: number, color: string) {
  setColor(color);
  drawCircle(x, y);
}
```

함수 차이가 단순 값이 아닌 **동작** 일 경우, 전략 객체나 고차 함수로 파라미터화합니다.

```typescript
// Before: 처리 로직만 다른 두 반복문
function sumValues(arr: number[]) {
  let result = 0;
  for (const n of arr) result += n;
  return result;
}
function productValues(arr: number[]) {
  let result = 1;
  for (const n of arr) result *= n;
  return result;
}

// After: 동작을 함수로 추상화 (전략)
function reduceValues(arr: number[], combine: (a: number, b: number) => number, initial: number) {
  let result = initial;
  for (const n of arr) result = combine(result, n);
  return result;
}
const sum = (arr: number[]) => reduceValues(arr, (a, b) => a + b, 0);
const product = (arr: number[]) => reduceValues(arr, (a, b) => a * b, 1);
```


# 유사한 코드 통합하기 요약

| 상황 | 패턴 |
|---|---|
| 상수 메서드만 다른 두 클래스 | **유사 클래스 통합** — 생성자 매개변수로 통합 |
| 본문이 같은 연속 if 문 | **if 문 결합** — `\|\|` 로 조건 결합 |
| 복잡한 중첩 조건 | **조건 산술** — `\|\|`=`+`, `&&`=`×`, 드모르간 적용 |
| 조건에 부수 동작 | **순수 조건 사용** — 명령/질의 분리 |
| 클래스 간 변형 로직 | **전략 패턴 도입** — 변형을 별도 클래스로 분리 |
| 구현체 하나짜리 인터페이스 | **제거** — 변형이 필요할 때 **인터페이스 추출** |
| 구조 같고 동작만 다른 함수 | **유사 함수 통합** — 전략 / 고차 함수로 파라미터화 |

**핵심 원칙**: 유사한 코드를 발견하면 공통 구조를 드러내는 것이 목표입니다. 클래스나 함수를 통합할수록 코드에 숨겨진 구조가 명확해지고, 이후 변경 비용이 줄어듭니다.
