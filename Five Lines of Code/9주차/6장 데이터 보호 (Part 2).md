# 6장 데이터 보호 (Part 2)

8주차에서는 getter/setter를 줄이고, 공통 접사를 가진 데이터와 메서드를 클래스로 묶어 **불변속성을 지역화**하는 방법을 다뤘습니다.

이번 파트의 초점은 한 단계 더 나아가 **복잡한 데이터 캡슐화**입니다. 단순히 데이터를 `private`으로 숨기는 수준이 아니라, 객체가 올바른 상태로만 존재하도록 만들고, 특정 순서로 호출해야 한다는 암묵적인 약속을 코드 구조 자체로 제거합니다.

이번 파트에서 다룰 내용
- 순서에 존재하는 불변속성 제거하기
- 리팩터링 패턴: 순서 강제화
- 열거형을 클래스로 대체하는 방법
- 비공개 생성자를 통한 열거
- 숫자를 클래스 인스턴스로 다시 매핑하기
- 데이터 보호 관점에서 보는 6장의 핵심

---

# 복잡한 데이터 캡슐화

캡슐화는 흔히 "필드를 `private`으로 만들고 getter/setter를 둔다" 정도로 이해됩니다. 하지만 이 책에서 말하는 캡슐화는 더 강합니다.

> 캡슐화의 목적은 데이터를 숨기는 것이 아니라, **잘못된 상태가 만들어지기 어렵게 만드는 것**입니다.

예를 들어 어떤 객체가 다음 순서를 반드시 지켜야 한다고 해봅시다.

1. 원본 데이터를 만든다.
2. `transform()`을 호출해 내부 데이터를 변환한다.
3. 변환된 데이터를 사용한다.

이 구조의 문제는 `transform()` 호출이 빠져도 타입 시스템이 막아주지 못한다는 점입니다. 주석이나 문서로 "사용 전에 반드시 transform을 호출하세요"라고 적을 수는 있지만, 그건 설계가 아니라 기억력에 의존하는 약속입니다.

이런 약속을 책에서는 **순서 불변속성(order invariant)** 으로 봅니다.

> 어떤 일이 항상 다른 일보다 먼저 일어나야 한다는 조건이 순서 불변속성입니다.

순서 불변속성이 코드 바깥의 약속으로 남아 있으면 위험합니다. 호출자가 순서를 기억해야 하고, 테스트가 모든 호출 순서를 보장해야 하며, 코드를 읽는 사람도 "이 객체는 이미 초기화된 상태인가?"를 계속 추론해야 합니다.

해결책은 간단합니다. **순서를 지키지 않는 것이 불가능한 구조로 바꾸는 것**입니다.

---

# 순서에 존재하는 불변속성 제거하기

아래 코드는 `GameMap`을 만든 뒤 반드시 `transform()`을 호출해야 올바르게 사용할 수 있는 구조입니다.

```typescript
// Before: 사용 전에 transform()을 호출해야 하는 구조
class GameMap {
  private tiles: number[][];

  constructor(tiles: number[][]) {
    this.tiles = tiles;
  }

  transform(): void {
    this.tiles = this.tiles.map(row => row.map(tile => tile + 1));
  }

  tileAt(x: number, y: number): number {
    return this.tiles[y][x];
  }
}

const map = new GameMap(rawTiles);
map.transform();
const tile = map.tileAt(0, 0);
```

겉보기에는 별문제가 없어 보입니다. 하지만 실제 위험은 호출자가 `transform()`을 잊었을 때 드러납니다.

```typescript
const map = new GameMap(rawTiles);
const tile = map.tileAt(0, 0); // 변환되지 않은 데이터를 읽을 수 있다
```

이 코드는 문법적으로는 맞습니다. 컴파일도 됩니다. 하지만 도메인 규칙은 깨졌습니다.

이 문제는 `transform()`을 생성자 안으로 이동하면 사라집니다.

```typescript
// After: 생성자가 항상 변환을 완료한다
class GameMap {
  private tiles: number[][];

  constructor(rawTiles: number[][]) {
    this.tiles = rawTiles.map(row => row.map(tile => tile + 1));
  }

  tileAt(x: number, y: number): number {
    return this.tiles[y][x];
  }
}

const map = new GameMap(rawTiles);
const tile = map.tileAt(0, 0);
```

이제 `GameMap` 인스턴스를 얻었다는 사실 자체가 "변환이 완료된 맵"이라는 증거가 됩니다. 생성자는 객체의 다른 메서드보다 항상 먼저 실행되기 때문에, `tileAt()`보다 변환이 먼저 일어나는 순서를 별도로 기억할 필요가 없습니다.

이것이 **순서 강제화**입니다.

---

### 리팩터링 패턴: 순서 강제화

#### 설명

객체지향 언어에서 생성자는 객체의 다른 메서드보다 먼저 호출됩니다. 이 성질을 이용하면 특정 작업이 반드시 먼저 일어나도록 강제할 수 있습니다.

순서 강제화의 핵심은 다음과 같습니다.

- "A를 호출한 뒤 B를 호출해야 한다"는 약속을 제거한다.
- A를 생성자 또는 생성 과정 안으로 이동한다.
- 객체 인스턴스 자체가 A가 끝났다는 증거가 되게 한다.
- 외부에서는 B만 호출할 수 있게 만든다.

즉, 순서 강제화는 코드 사용자에게 "순서를 잘 지켜 주세요"라고 말하는 방식이 아닙니다. **순서를 어길 수 있는 API를 없애는 방식**입니다.

#### Before / After

```typescript
// Before: 호출 순서가 외부 책임
class Report {
  private rows: string[] = [];

  load(raw: string): void {
    this.rows = raw.split("\n");
  }

  normalize(): void {
    this.rows = this.rows.map(row => row.trim());
  }

  print(): string {
    return this.rows.join("\n");
  }
}

const report = new Report();
report.load(rawText);
report.normalize();
console.log(report.print());
```

이 코드는 `load()`와 `normalize()`의 호출 순서가 중요합니다. `normalize()`를 빼먹거나, `load()`보다 먼저 호출하면 객체는 잘못된 상태가 됩니다.

```typescript
// After: 생성 과정에서 순서를 강제
class Report {
  private rows: string[];

  constructor(raw: string) {
    this.rows = raw.split("\n").map(row => row.trim());
  }

  print(): string {
    return this.rows.join("\n");
  }
}

const report = new Report(rawText);
console.log(report.print());
```

이제 `Report`는 생성되는 순간부터 출력 가능한 상태입니다. 외부 호출자는 `load()`와 `normalize()`의 순서를 알 필요가 없습니다.

#### 절차

1. 마지막에 실행되어야 하는 메서드 또는 실제로 외부에 노출되어야 하는 메서드를 찾습니다.
2. 그 메서드가 의존하는 사전 작업을 확인합니다.
3. 사전 작업을 생성자 또는 정적 팩토리 메서드로 이동합니다.
4. 사전 작업과 후속 작업이 같은 인자를 공유한다면, 그 인자를 필드로 만들고 메서드 인자에서 제거합니다.
5. 더 이상 외부에서 호출할 필요가 없는 초기화 메서드를 삭제하거나 비공개로 만듭니다.

#### 내부 변형과 외부 변형

순서 강제화에는 크게 두 가지 방식이 있습니다.

```typescript
// 내부 변형: 변환 로직을 새 클래스 내부에 둔다
class NormalizedText {
  private lines: string[];

  constructor(raw: string) {
    this.lines = raw.split("\n").map(line => line.trim());
  }

  print(): string {
    return this.lines.join("\n");
  }
}
```

```typescript
// 외부 변형: 변환은 외부 함수가 수행하고, 결과만 객체에 넣는다
class NormalizedText {
  constructor(private lines: string[]) {}

  print(): string {
    return this.lines.join("\n");
  }
}

function normalize(raw: string): NormalizedText {
  return new NormalizedText(raw.split("\n").map(line => line.trim()));
}
```

둘 다 순서를 강제할 수 있지만, 책은 보통 **내부 변형**을 더 선호합니다. 내부 변형은 변환 로직과 데이터가 같은 클래스 안에 있어 캡슐화가 더 강하고, getter나 public 필드 없이도 상태를 보호할 수 있기 때문입니다.

다만 외부 변형도 쓸모가 있습니다. 생성 과정이 복잡하거나, 실패 가능성이 크거나, 여러 단계를 명시적으로 표현해야 한다면 정적 팩토리나 별도 빌더 함수가 더 읽기 쉬울 수 있습니다.

핵심은 형태가 아닙니다.

> 객체가 만들어진 뒤에는 항상 유효한 상태여야 한다는 점이 핵심입니다.

---

# 열거형을 제거하는 또 다른 방법

이전 장들에서는 `switch`나 `if`로 흩어진 타입별 동작을 제거하기 위해 타입 코드를 클래스로 바꾸는 방법을 다뤘습니다. 그런데 실무에서는 한 가지 제약을 자주 만납니다.

> 열거형(enum)에 메서드를 넣을 수 없는 언어가 있습니다.

예를 들어 TypeScript의 `enum`은 값의 집합을 표현하기에는 편하지만, 각 값마다 서로 다른 동작을 자연스럽게 붙이기 어렵습니다.

```typescript
enum RawTile {
  Air,
  Wall,
  Water
}

function transformTile(tile: RawTile): string {
  switch (tile) {
    case RawTile.Air:
      return " ";
    case RawTile.Wall:
      return "#";
    case RawTile.Water:
      return "~";
  }
}
```

이 코드는 `RawTile`이라는 타입은 있지만, 정작 `RawTile`별 동작은 외부 함수 `transformTile()`에 있습니다. 새 타일이 추가되면 `enum`도 수정하고 `switch`도 수정해야 합니다. 비슷한 `switch`가 여러 곳에 있다면 변경 범위는 더 넓어집니다.

우리가 원하는 구조는 다음에 가깝습니다.

```typescript
RawTile.Wall.transform();
```

즉, 타입별 동작을 타입 자체 옆으로 옮기고 싶습니다. 언어의 `enum`이 이것을 지원하지 않는다면, 클래스로 직접 열거형을 만들 수 있습니다.

---

### 비공개 생성자를 통한 열거

비공개 생성자를 사용하면 클래스 외부에서 새 인스턴스를 만들 수 없습니다. 클래스 내부에서만 미리 정해진 인스턴스를 만들고, 그것을 public 상수로 공개하면 열거형처럼 사용할 수 있습니다.

```typescript
// Before: enum + 외부 switch
enum RawTile {
  Air,
  Wall,
  Water
}

function transformTile(tile: RawTile): string {
  switch (tile) {
    case RawTile.Air:
      return " ";
    case RawTile.Wall:
      return "#";
    case RawTile.Water:
      return "~";
  }
}
```

```typescript
// After: private constructor로 만든 클래스 기반 열거
class RawTile {
  static readonly Air = new RawTile(" ");
  static readonly Wall = new RawTile("#");
  static readonly Water = new RawTile("~");

  private constructor(private readonly symbol: string) {}

  transform(): string {
    return this.symbol;
  }
}

const wall = RawTile.Wall.transform();
```

이 구조의 장점은 분명합니다.

- `RawTile`의 값과 동작이 같은 위치에 있다.
- `transformTile()` 같은 외부 함수가 필요 없다.
- `switch`가 사라진다.
- 생성자가 `private`이므로 정해진 값 외의 `RawTile`을 만들 수 없다.

이 방식은 단순한 상수 목록보다 한 단계 더 강합니다. 각 값이 객체이기 때문에 동작을 가질 수 있고, 값마다 다른 전략을 품을 수도 있습니다.

```typescript
class RawTile {
  static readonly Air = new RawTile(" ", false);
  static readonly Wall = new RawTile("#", true);
  static readonly Water = new RawTile("~", false);

  private constructor(
    private readonly symbol: string,
    private readonly blocksMovement: boolean
  ) {}

  draw(): string {
    return this.symbol;
  }

  canEnter(): boolean {
    return !this.blocksMovement;
  }
}
```

이제 "벽은 이동을 막는다"는 규칙이 `RawTile.Wall` 근처에 머뭅니다. 외부 코드는 `tile.canEnter()`라고 물어볼 뿐, 어떤 값이 벽인지 물어보고 분기하지 않습니다.

#### 주의할 점

이 방식에도 비용은 있습니다.

- 언어가 제공하는 `switch(enum)` 편의성을 잃습니다.
- 직렬화/역직렬화 시 단순 숫자 enum보다 처리가 번거로울 수 있습니다.
- 객체 동일성에 의존하므로 외부에서 임의 인스턴스를 만들 수 없도록 생성자를 반드시 막아야 합니다.

하지만 이 책의 관점에서는 첫 번째 비용이 큰 문제가 아닙니다. 이미 `switch`를 피하라는 규칙이 있기 때문입니다. 오히려 `switch`를 사용할 수 없어진다는 점이 타입별 행동을 객체 안으로 밀어 넣는 데 도움이 됩니다.

---

### 숫자를 클래스에 다시 매핑하기

실제 게임 맵이나 파일 포맷에서는 타일이 숫자로 저장되는 경우가 많습니다.

```typescript
const rawMap = [
  [0, 1, 1],
  [0, 0, 2],
  [1, 0, 0]
];
```

여기서 `0`, `1`, `2`는 각각 `Air`, `Wall`, `Water`를 의미한다고 해봅시다. 문제는 숫자 자체에는 의미도 동작도 없다는 점입니다.

```typescript
// Before: 숫자 코드가 여러 곳으로 퍼지기 쉬운 구조
function drawTile(tile: number): string {
  if (tile === 0) return " ";
  if (tile === 1) return "#";
  if (tile === 2) return "~";
  throw new Error("Unknown tile");
}
```

숫자 코드는 외부 입력 경계에서는 피할 수 없을 수 있습니다. 하지만 내부 모델까지 숫자로 유지할 필요는 없습니다. 입력을 받는 순간 숫자를 객체로 바꾸면 됩니다.

```typescript
class RawTile {
  static readonly Air = new RawTile(" ");
  static readonly Wall = new RawTile("#");
  static readonly Water = new RawTile("~");

  private static readonly values = [
    RawTile.Air,
    RawTile.Wall,
    RawTile.Water
  ];

  private constructor(private readonly symbol: string) {}

  static fromNumber(value: number): RawTile {
    const tile = RawTile.values[value];
    if (tile === undefined) {
      throw new Error(`Unknown tile: ${value}`);
    }
    return tile;
  }

  draw(): string {
    return this.symbol;
  }
}

const tiles = rawMap.map(row => row.map(RawTile.fromNumber));
```

이제 시스템 내부에서는 `number`가 아니라 `RawTile`을 사용합니다.

```typescript
class GameMap {
  private tiles: RawTile[][];

  constructor(rawMap: number[][]) {
    this.tiles = rawMap.map(row => row.map(RawTile.fromNumber));
  }

  draw(): string {
    return this.tiles
      .map(row => row.map(tile => tile.draw()).join(""))
      .join("\n");
  }
}
```

여기서 중요한 점은 숫자 처리가 **경계에 갇혔다**는 것입니다. `0`이 공기인지, `1`이 벽인지 아는 코드는 `RawTile.fromNumber()` 하나뿐입니다. 나머지 코드는 `RawTile`의 메서드만 사용합니다.

이 구조는 8주차에서 다룬 푸시 기반 아키텍처와도 연결됩니다.

- 나쁜 구조: 숫자를 꺼내 외부에서 의미를 해석한다.
- 좋은 구조: 숫자를 객체로 바꾼 뒤 객체에게 행동을 요청한다.

---

# 실무 관점에서 보기

이번 파트의 예제는 게임 타일처럼 단순하지만, 실무 코드에서도 같은 문제가 반복됩니다.

### 1. 초기화 메서드가 따로 있는 객체

```typescript
const client = new ApiClient();
client.configure(config);
client.connect();
client.request("/users");
```

이런 구조는 `configure()`와 `connect()`의 순서를 외부에 떠넘깁니다. 가능하다면 생성자나 정적 팩토리로 유효한 객체만 만들게 하는 편이 낫습니다.

```typescript
const client = ApiClient.connect(config);
client.request("/users");
```

### 2. 상태 코드가 서비스 계층에 퍼진 도메인

```typescript
if (order.status === "PAID") {
  // ...
}
```

이런 코드가 여러 곳에 반복되면 상태 코드가 도메인 전체로 새어 나간 것입니다. 상태별 행동을 `OrderStatus` 객체나 `Order`의 의미 있는 메서드로 옮길 수 있는지 봐야 합니다.

```typescript
if (order.canCancel()) {
  order.cancel();
}
```

또는 더 나아가 다음처럼 상태 전이를 객체 내부에 숨길 수 있습니다.

```typescript
order.cancel();
```

이 관점은 llm-wiki의 JPA 베스트 프랙티스와도 맞닿아 있습니다. 엔티티를 단순 DTO처럼 열어두고 setter로 상태를 바꾸면 비즈니스 규칙이 서비스 계층으로 퍼집니다. 반대로 `order.cancel(reason)`, `order.pay()`처럼 의미 있는 메서드를 제공하면 상태 전이 규칙이 엔티티 내부에 캡슐화됩니다.

### 3. 외부 입력 형식이 내부 모델을 지배하는 경우

API, DB, 파일 포맷은 문자열과 숫자 코드를 많이 사용합니다. 하지만 그 형식을 내부까지 그대로 끌고 들어오면, 내부 코드 전체가 외부 표현에 결합됩니다.

좋은 기준은 다음과 같습니다.

> 외부 표현은 경계에서만 다루고, 내부에서는 의미 있는 타입으로 바꾼다.

숫자 타일을 `RawTile.fromNumber()`에서 객체로 바꾸는 방식이 바로 그 예입니다.

---

# 6장 전체 흐름으로 다시 보기

6장의 주제는 "데이터 보호"입니다. 하지만 여기서 보호한다는 말은 단순히 외부 접근을 막는다는 뜻이 아닙니다.

6장이 말하는 데이터 보호는 세 가지 층위로 볼 수 있습니다.

### 1. getter/setter를 줄여 데이터 노출을 막는다

getter는 객체 내부 데이터를 외부로 끌어내는 통로입니다. 데이터가 밖으로 나가면 그 데이터를 해석하는 규칙도 밖으로 퍼집니다.

그래서 8주차에서는 getter/setter를 제거하고, 행동을 데이터 가까이로 옮기는 푸시 기반 구조를 다뤘습니다.

### 2. 공통 접사를 클래스로 묶어 책임을 분리한다

`playerX`, `playerY`, `playerHealth`, `movePlayerLeft()`처럼 같은 접사가 반복되면 사실상 하나의 개념이 흩어져 있다는 신호입니다.

이를 `Player` 클래스로 묶으면 데이터와 행동이 같은 곳에 모이고, `Player`의 불변속성은 `Player` 내부에서만 관리하면 됩니다.

### 3. 순서와 타입의 불변속성을 구조로 강제한다

이번 파트에서는 여기서 더 나아갔습니다.

- 반드시 먼저 호출해야 하는 메서드는 생성자나 팩토리로 옮긴다.
- 숫자나 enum으로 표현된 타입 코드는 객체로 바꾼다.
- 타입별 행동은 외부 분기가 아니라 타입 객체의 메서드로 이동한다.

결국 목표는 하나입니다.

> 잘못된 사용법을 문서로 금지하지 말고, 코드 구조상 불가능하게 만든다.

---

# 핵심 요약

> **6장 Part 2의 한 줄 요약**: 호출 순서와 타입 코드에 숨어 있는 불변속성을 객체 구조 안으로 밀어 넣어라.

### 1. 순서 불변속성은 위험하다

- "A를 호출한 뒤 B를 호출해야 한다"는 규칙은 쉽게 깨진다.
- 호출자가 순서를 기억해야 한다면 캡슐화가 부족한 것이다.
- 생성자나 정적 팩토리를 사용해 유효한 객체만 만들게 해야 한다.

### 2. 순서 강제화의 핵심

- 생성자는 다른 메서드보다 먼저 실행된다.
- 생성이 끝난 객체는 필요한 초기화가 완료되었다는 증거가 된다.
- 외부에 초기화 메서드를 노출하지 않을수록 잘못된 상태가 줄어든다.

### 3. enum에 행동을 붙일 수 없다면 클래스로 열거한다

- `private constructor` + `static readonly` 인스턴스로 enum처럼 사용할 수 있다.
- 값과 행동을 같은 클래스 안에 둘 수 있다.
- `switch`가 사라지고 타입별 행동이 객체 내부로 이동한다.

### 4. 숫자 코드는 경계에서 객체로 바꾼다

- 외부 입력은 숫자나 문자열일 수 있다.
- 내부 모델까지 숫자 코드로 유지하면 의미 해석 코드가 퍼진다.
- `fromNumber()`, `fromString()` 같은 변환 지점에서 의미 있는 타입으로 바꿔야 한다.

### 5. 데이터 보호의 본질

- 데이터 보호는 접근 제한보다 넓은 개념이다.
- 중요한 것은 불변속성의 범위를 줄이는 것이다.
- 좋은 캡슐화는 객체가 항상 올바른 상태로 존재하게 만든다.

### 마음에 새길 한 문장

> "좋은 API는 올바른 사용법을 쉽게 만드는 데서 멈추지 않고, 잘못된 사용법을 불가능하게 만든다."
