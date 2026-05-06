# 6장 데이터 보호 (Part 1)

이번 장에서는 데이터와 기능에 대한 접근을 제한하는 **캡슐화**에 초점을 맞춰 **불변속성이 지역에만 영향을 주게** 만드는 데 중점을 둡니다.

이번 파트에서 다룰 내용
- getter 없이 캡슐화하기 (getter/setter 사용 금지 규칙)
- 푸시 기반 vs. 풀 기반 아키텍처
- 리팩터링 패턴: getter와 setter 제거하기
- 공통 접사를 사용하지 말 것 (단일 책임 원칙)
- 리팩터링 패턴: 데이터 캡슐화

---

# getter 없이 캡슐화하기

### 규칙: getter와 setter를 사용하지 말 것

#### 정의
**부울(Boolean)이 아닌 필드에 setter나 getter를 사용하지 마십시오.**

여기서 setter/getter란 부울이 아닌 필드를 직접 할당하거나 반환하는 메서드를 의미합니다.

#### 설명
getter와 setter는 흔히 private 필드를 다루기 위한 메서드로 캡슐화와 함께 배웁니다. 그러나 객체의 필드에 대한 getter가 존재하는 순간 **캡슐화는 해제되고 불변속성이 전역적으로** 됩니다.

- 객체를 반환받은 곳은 그 객체를 더 멀리 전달할 수 있고, 우리가 통제할 수 없습니다.
- 객체를 얻은 어느 곳에서나 public 메서드를 호출할 수 있으며, 예상치 못한 방식으로 객체를 수정할 수 있습니다.
- setter도 마찬가지입니다. 이론적으로 setter는 또 다른 간접 레이어를 도입할 수 있지만, 실제로는 새 데이터 구조를 받아들이도록 getter를 수정하게 되고 결국 수신자 측의 코드까지 수정해야 합니다. 이것이 우리가 피하고 싶어 하는 **밀 결합(tight coupling)** 의 형태입니다.

이런 문제는 **가변(mutable) 객체**에서만 발생합니다. 그럼에도 이 규칙이 불변 필드까지 비공개로 만들도록 하는 이유는, 그렇게 해야 **푸시 기반 아키텍처**가 자연스럽게 장려되기 때문입니다. 부울 필드만 예외인 이유는, 부울은 보통 단일 비트의 상태 플래그라 위험성이 적기 때문입니다.

#### 풀 기반 vs 푸시 기반 아키텍처

| 구분 | 풀(pull) 기반 | 푸시(push) 기반 |
| --- | --- | --- |
| 데이터 흐름 | 외부에서 데이터를 **가져와** 중앙에서 연산 | 데이터를 **인자로 전달**, 또는 데이터 옆에서 연산 |
| 결과 구조 | "수동적인 데이터 클래스" + 거대한 "관리자 클래스" | 클래스마다 자신의 기능을 가짐 |
| 결합도 | 데이터 ↔ 관리자, 데이터 ↔ 데이터 사이가 밀 결합 | 클래스 간 결합도가 낮음 |
| 메서드 노출 | getter (데이터 노출) | 서비스성 메서드 (행동 노출) |

푸시 기반 아키텍처에서는 메서드 사용자가 내부 구조를 신경 쓰지 않아도 됩니다.

```typescript
// 예제. 풀 기반 아키텍처 — 외부에서 데이터를 끌어와 처리
class Blog {
  private posts: Post[];
  constructor(posts: Post[]) { this.posts = posts; }
  getPosts(): Post[] { return this.posts; }
}

class Post {
  private title: string;
  private url: string;
  constructor(title: string, url: string) {
    this.title = title;
    this.url = url;
  }
  getTitle(): string { return this.title; }
  getUrl(): string { return this.url; }
}

// 외부 '관리자'가 모든 데이터를 끌어와 작업을 수행
function renderPostList(blog: Blog): string {
  let html = "";
  for (const post of blog.getPosts()) {
    html += `<a href="${post.getUrl()}">${post.getTitle()}</a>`;
  }
  return html;
}
```

```typescript
// 예제. 푸시 기반 아키텍처 — 연산을 데이터 가까이로 이동
class Blog {
  private posts: Post[];
  constructor(posts: Post[]) { this.posts = posts; }

  renderPostList(): string {
    let html = "";
    for (const post of this.posts) {
      html += post.renderLink();
    }
    return html;
  }
}

class Post {
  private title: string;
  private url: string;
  constructor(title: string, url: string) {
    this.title = title;
    this.url = url;
  }

  renderLink(): string {
    return `<a href="${this.url}">${this.title}</a>`;
  }
}
```

#### 스멜
이 규칙은 **디미터 법칙(Law of Demeter, "낯선 사람에게 말하지 말라")** 에서 유래했습니다. 여기서 낯선 사람이란 직접 접근할 수는 없지만 참조를 얻을 수 있는 객체입니다. 객체지향 언어에서는 보통 getter를 통해 낯선 사람에 접근하게 되므로 이 규칙이 존재합니다.

#### 의도
- 참조를 얻은 객체와 상호작용하면 **객체를 가져오는 방식과 밀 결합**됩니다. 즉 소유자의 내부 구조를 알아야 합니다.
- 필드 소유자는 데이터 구조를 변경할 때 기존 접근 방식을 유지해야 하며, 그렇지 않으면 호출 측 코드가 손상됩니다.
- 푸시 기반 아키텍처에서는 서비스성 메서드를 노출하므로, 사용자는 내부 구조를 알 필요가 없습니다.

---

### 규칙 적용하기

다음과 같이 외부에서 게시물 데이터를 끌어와 사용하는 코드는 디미터 법칙을 위반합니다.

```typescript
// 예제. 변경 전 — getter로 끌어와서 외부에서 처리
class Blog {
  private posts: Post[];
  getPosts(): Post[] { return this.posts; }
}

class Post {
  private title: string;
  private url: string;
  getTitle(): string { return this.title; }
  getUrl(): string { return this.url; }
}

// 외부 함수가 Blog의 내부 구조까지 알아야 한다
function renderPostList(blog: Blog): string {
  let html = "";
  for (const post of blog.getPosts()) {
    html += `<a href="${post.getUrl()}">${post.getTitle()}</a>`;
  }
  return html;
}
```

```typescript
// 예제. 변경 후 — 기능을 데이터 가까이로 이동
class Blog {
  private posts: Post[];
  renderPostList(): string {
    return this.posts.map(p => p.renderLink()).join("");
  }
}

class Post {
  private title: string;
  private url: string;
  renderLink(): string {
    return `<a href="${this.url}">${this.title}</a>`;
  }
}
```

이렇게 하면 `Post`가 자신의 데이터 표현 방식을 변경해도 `Blog`나 외부 코드에 영향을 주지 않습니다. 불변속성은 클래스 내부에 갇히게 됩니다.

---

### 리팩터링 패턴: getter와 setter 제거하기

#### 설명
getter/setter를 제거하기 위해 **기능을 데이터에 더 가깝게 이동**시킵니다. getter와 setter는 거의 대칭이라 같은 절차로 둘 다 제거할 수 있습니다(이하 getter로 설명).

코드를 데이터 가까이 옮기다 보면 보통 getter 하나가 **여러 개의 작은 함수**로 대체됩니다. 이는 getter가 사용되던 호출 컨텍스트의 수에 비례합니다. 새로 만든 메서드들은 데이터 컨텍스트가 아니라 **호출 컨텍스트** 기준으로 이름을 짓는 게 좋습니다 (예: `getTitle` → `renderLink`, `summarize`, `matches` ...).

#### 절차
1. getter/setter가 사용되는 모든 곳에서 컴파일 오류가 나도록 **비공개로 변경**합니다.
2. 클래스로 코드를 이관하면서 오류를 하나씩 해결합니다.
3. getter/setter가 코드 이관 과정에서 인라인화되어 더 이상 사용되지 않으면 **삭제**합니다.

#### 예제

```typescript
// Step 0: 시작 코드
class Post {
  public title: string; // 공개 필드 또는 getter가 있는 상태
}
function renderHeader(post: Post): string {
  return `<h1>${post.title}</h1>`;
}

// Step 1: 비공개로 만들어 오류 유발
class Post {
  private title: string;
}
// renderHeader(post)에서 컴파일 오류 발생

// Step 2: 기능을 Post로 이관
class Post {
  private title: string;
  renderHeader(): string {
    return `<h1>${this.title}</h1>`;
  }
}
function renderHeader(post: Post): string {
  return post.renderHeader(); // 인라인 호출만 남음
}

// Step 3: 더 이상 외부 함수가 필요 없다면 삭제하고 호출지에서 직접 사용
```

### 마지막 getter 삭제

마지막 getter를 제거할 때 흔히 마주치는 상황은 **getter의 결과가 여러 곳에서 서로 다른 방식으로 가공**되고 있다는 것입니다. 이 경우 한 번에 모두 푸시 기반으로 옮기지 못할 수 있습니다.

해결 전략:
- 호출 컨텍스트별로 **별도의 메서드**를 도입한다 (`renderLink`, `renderHeader`, `summarize` 등).
- 호출 컨텍스트가 정말 일회성·외부 시스템 연동(예: 로깅, 직렬화)이라면 **전용 메서드**를 만들고 getter는 그 안으로 한정시킨다.
- 끝까지 남는 마지막 getter가 있다면, 그것은 **진짜 외부 인터페이스**일 가능성이 높습니다. 다만 그 시점에는 **불변(immutable) 값**을 반환하도록 만드는 것이 좋습니다.

목표는 단순히 "getter를 0개로 만들기"가 아니라, **객체 외부의 코드가 객체의 내부 표현에 의존하지 않도록 만드는 것**입니다.

---

# 간단한 데이터 캡슐화하기

다시 한번 코드가 모든 규칙을 준수합니다. 따라서 새로운 규칙을 도입합니다.

### 규칙: 공통 접사를 사용하지 말 것

#### 정의
**코드에는 공통 접두사나 접미사가 있는 메서드나 변수가 없어야 합니다.**

#### 설명
사용자 이름의 경우 `username`, 타이머를 시작하는 동작의 경우 `startTimer` 처럼 컨텍스트를 암시하기 위해 접사를 붙일 때가 많습니다. 코드를 읽기 쉽게 만들지만, 여러 요소가 동일한 접사를 가질 때는 그 요소들이 **긴밀히 묶여 있다는 신호**입니다. 이 구조를 더 잘 표현하는 방법은 바로 **클래스**입니다.

클래스를 사용해 메서드/변수를 그룹화하면 다음과 같은 장점이 있습니다.
- **외부 인터페이스를 완전하게 통제**할 수 있다 (도우미 메서드를 숨겨 전역 범위 오염 방지).
- 다섯 줄 제한 규칙으로 메서드 수가 늘어나는 상황에서 특히 중요하다.
- 모든 메서드가 모든 곳에서 안전하게 호출될 수 있는 것은 아니다 → 클래스가 호출 가능한 시점을 보장한다.
- 가장 중요한 것은 **데이터가 숨겨져서 불변속성이 클래스 내에서 관리**된다는 점이다 → **지역 불변속성**.

#### 스멜
이 규칙은 **단일 책임 원칙(SRP)** 에서 파생되었습니다. 앞서 다룬 "메서드는 한 가지 작업만 해야 한다"가 메서드 단위였다면, 이 규칙은 **클래스 단위의 SRP**입니다.

#### 의도
- 단일 책임으로 클래스를 설계하려면 원칙이 필요한데, 공통 접사가 그 **하위 책임을 식별**하는 단서를 줍니다.
- 공통 접사가 암시하는 구조 = 그 메서드/변수가 **공통 접사의 책임을 공유**한다는 의미.
- 따라서 그 메서드/변수는 **그 공통 책임을 전담하는 별도 클래스**에 있어야 합니다.
- 시간이 지나며 클래스가 비대해질 때, 어떤 책임을 어디로 옮길지 판단하는 가이드가 됩니다.

---

### 규칙 적용하기

```typescript
// 변경 전 — 공통 접사(user, timer)가 동일 클래스 안에 흩어져 있음
class App {
  private userName: string;
  private userEmail: string;
  private userAge: number;

  private timerStart: number;
  private timerDuration: number;

  validateUserName(): boolean { /* ... */ }
  validateUserEmail(): boolean { /* ... */ }
  startTimer(): void { /* ... */ }
  stopTimer(): void { /* ... */ }
  isTimerExpired(): boolean { /* ... */ }
}
```

```typescript
// 변경 후 — 접사를 클래스로 승격, 책임을 분리
class User {
  private name: string;
  private email: string;
  private age: number;

  validateName(): boolean { /* ... */ }
  validateEmail(): boolean { /* ... */ }
}

class Timer {
  private start: number;
  private duration: number;

  start(): void { /* ... */ }
  stop(): void { /* ... */ }
  isExpired(): boolean { /* ... */ }
}

class App {
  private user: User;
  private timer: Timer;
}
```

각 책임이 별도 클래스로 분리되면서 외부 인터페이스가 명확해지고, 도우미 메서드는 클래스 내부에 숨겨집니다.

---

### 리팩터링 패턴: 데이터 캡슐화

#### 설명
변수와 메서드를 캡슐화하면 다음을 얻습니다.
- **접근 지점을 제한**해 구조가 명확해진다.
- 메서드명을 단순화할 수 있다 (`validateUserName` → `User.validateName`).
- **응집력**이 명확해지고, 더 좋은(=작은) 클래스가 늘어난다. 사람들은 보통 클래스를 만드는 데 너무 소극적이다.
- 변수를 캡슐화할 때 가장 큰 이익이 발생한다: 데이터에 대한 **속성/가정**은 접근 위치가 많을수록 유지보수가 어려워지는데, 범위를 클래스 내부로 제한하면 **불변속성 검증이 클래스 내부 코드만 보면 충분**해진다.

> 변수 없이 공통 접사를 가진 메서드만 존재하는 경우에도 이 패턴은 유용합니다. 다만 절차를 적용하기 전에 **메서드를 먼저 클래스로 옮겨야** 합니다.

#### 절차
1. 공통 접사를 가진 변수와 메서드를 식별한다.
2. 새 클래스를 만들고, 변수들을 **이동(move) 후 비공개**로 만든다.
3. 해당 변수에 접근하던 메서드들을 새 클래스로 이동하면서, 접사를 **이름에서 제거**한다 (`validateUserName` → `validateName`).
4. 외부에서 변수에 직접 접근하던 코드는 새 클래스의 메서드 호출로 대체한다.
5. 컴파일/테스트를 통해 누락된 호출이 없는지 확인한다.
6. 더 이상 사용하지 않는 헬퍼/필드는 삭제한다.

#### 예제

```typescript
// Before
class Game {
  private playerX: number;
  private playerY: number;
  private playerHealth: number;

  movePlayerLeft() { this.playerX--; }
  movePlayerRight() { this.playerX++; }
  damagePlayer(amount: number) { this.playerHealth -= amount; }
  isPlayerAlive() { return this.playerHealth > 0; }
}
```

```typescript
// After — 'player' 접사를 클래스로 승격
class Player {
  private x: number;
  private y: number;
  private health: number;

  moveLeft()  { this.x--; }
  moveRight() { this.x++; }
  damage(amount: number) { this.health -= amount; }
  isAlive() { return this.health > 0; }
}

class Game {
  private player: Player;
}
```

`Player`의 불변속성(`health >= 0`, 좌표 범위 등)은 이제 **`Player` 내부에서만 검증**하면 충분해집니다. `Game`은 이 디테일을 알 필요가 없습니다.

---

# 핵심 요약

> **6장 Part 1의 한 줄 요약**: 데이터를 숨기고, 행동을 데이터 옆으로 옮겨라. 그러면 불변속성이 지역화된다.

### 1. getter/setter 금지 (부울 제외)
- getter/setter는 **불변속성을 전역으로 만든다**.
- 디미터 법칙 위반: 호출자가 객체 내부 구조에 의존하게 된다.
- 대안은 **푸시 기반 아키텍처**: 데이터를 끌어오지 말고, 행동을 데이터 가까이로 보내라.

### 2. 풀 기반 vs 푸시 기반
- **풀**: 수동적 데이터 클래스 + 거대한 관리자 클래스 → 밀 결합.
- **푸시**: 모든 클래스가 자신의 기능을 갖는다 → 결합도 ↓.

### 3. getter/setter 제거 절차
1. 비공개로 바꿔 컴파일 오류를 일으킨다.
2. 오류 지점의 로직을 클래스 안으로 이관한다.
3. 인라인화되어 사라진 getter/setter를 삭제한다.

### 4. 공통 접사 = 새 클래스의 신호
- `userName`, `userEmail`, `validateUserName`, `startTimer`, `stopTimer` ...
- 같은 접사를 공유한다 = 같은 책임을 공유한다 = **별도 클래스로 분리해야 한다**.
- 단일 책임 원칙(SRP)의 클래스 버전.

### 5. 데이터 캡슐화의 본질
- 캡슐화의 가장 큰 가치는 "접근 제어"가 아니라 **불변속성의 지역화**.
- 클래스 내부에서만 데이터에 접근할 수 있으면, 불변속성 검증은 그 클래스 안만 보면 된다.
- → 코드 변경 시 영향 범위를 좁히는 가장 강력한 도구.

### 마음에 새길 한 문장
> "데이터를 노출하지 말고 행동을 노출하라. getter는 캡슐화의 성공이 아니라 실패의 흔적이다."
