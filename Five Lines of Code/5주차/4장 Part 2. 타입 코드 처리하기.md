이번 파트에서 다룰 내용
- 긴 `if` 체인을 분리해서 다음 리팩터링이 가능하도록 만들기
- **메서드 전문화**로 불필요한 일반성 제거하기
- `switch` 를 외부 데이터와 내부 객체를 연결하는 경계에서만 사용하기
- 인터페이스와 컴포지션으로 클래스 중복 다루기
- 필요 없는 코드를 **삭제 후 컴파일하기**로 안전하게 정리하기


# 긴 if 문의 리팩터링
다음 코드는 `if 문은 함수의 시작에만 배치` 규칙을 크게 위반합니다. `while` 안에서 긴 `else if` 체인이 이어지고 있기 때문에, 현재 함수는 입력 순회와 입력 처리라는 두 가지 일을 동시에 하고 있습니다.

```
-- 예제 4.58 변경 전
function updateInputs(player: Player, inputs: Input[]) {
	while (inputs.length > 0) {
		let current = inputs.pop();
		if (current instanceof Left) {
			player.moveHorizontal(-1);
		} else if (current instanceof Right) {
			player.moveHorizontal(1);
		} else if (current instanceof Up) {
			player.jump();
		} else if (current instanceof Down) {
			player.crouch();
		}
		player.updateAnimation();
	}
}
```

이 경우 첫 단계는 체인을 없애는 것이 아니라, 체인을 **독립된 메서드로 추출**하는 것입니다. 그러면 분기 로직이 한곳으로 모이고, 이후 전문화나 클래스 이관을 적용할 수 있습니다.

```
-- 예제 4.59 변경 후
function updateInputs(player: Player, inputs: Input[]) {
	while (inputs.length > 0) {
		let current = inputs.pop();
		handleInput(player, current);
		player.updateAnimation();
	}
}

function handleInput(player: Player, input: Input) {
	if (input instanceof Left) {
		player.moveHorizontal(-1);
	} else if (input instanceof Right) {
		player.moveHorizontal(1);
	} else if (input instanceof Up) {
		player.jump();
	} else if (input instanceof Down) {
		player.crouch();
	}
}
```

핵심은 분기 자체를 당장 제거하는 것이 아니라, **분기를 다루는 위치를 하나로 좁히는 것**입니다. 그래야 다음 단계에서 안전하게 바꿀 수 있습니다.

### 일반성 제거
긴 `if` 체인을 없애기 어려운 이유는 대개 함수가 지나치게 일반적이기 때문입니다. 여러 방향, 여러 타입, 여러 모드를 하나의 메서드가 동시에 처리하면 호출 위치는 늘어나고, 그 메서드는 삭제하기 어려워집니다. 이때 필요한 것이 **일반화의 반대 방향**, 즉 전문화입니다.

### 리팩터링 패턴: 메서드 전문화

#### 설명
이 리팩터링은 많은 프로그래머의 본능에 반하기에 좀 난해합니다. 프로그래머들은 일반화하고 재사용하려는 본능적인 욕구가 있지만 그렇게 하면 책임이 흐려지고 다양한 위치에서 코드를 호출할 수 있기 때문에 문제가 될 수 있습니다. 이 리팩터링 패턴은 이 효과를 반전시킵니다. 좀 더 전문화된 메서드는 더 적은 위치에서 호출되어 필요성이 없어지면 더 빨리 제거할 수 있습니다.

#### 절차
1. 전문화하려는 메서드를 복제합니다.
2. 메서드 중 하나의 이름을 새로 사용할 메서드의 이름으로 변경하고 전문화하려는 매개변수를 제거(또는 교체)합니다.
3. 매개변수 제거에 따라 메서드를 수정해서 오류가 없도록 합니다.
4. 이전의 호출을 새로운 것을 사용하도록 변경합니다.

#### 예제
아래 메서드는 `Direction` 값 하나로 왼쪽 이동과 오른쪽 이동을 모두 처리합니다. 겉으로는 재사용 가능해 보이지만, 실제로는 호출하는 쪽이 늘어날수록 불필요한 일반성이 됩니다.

```
-- 예제 4.60 변경 전
enum Direction {
	LEFT,
	RIGHT
}

function moveHorizontally(player: Player, direction: Direction) {
	if (direction === Direction.LEFT) {
		player.moveHorizontal(-1);
		return;
	}
	player.moveHorizontal(1);
}

class Left implements Input {
	handle(player: Player) {
		moveHorizontally(player, Direction.LEFT);
	}
}

class Right implements Input {
	handle(player: Player) {
		moveHorizontally(player, Direction.RIGHT);
	}
}
```

전문화 후에는 의미가 분명한 메서드만 남습니다. 이제 각 호출 위치는 더 구체적이고, 더 적은 맥락만 이해하면 됩니다.

```
-- 예제 4.61 변경 후
function moveLeft(player: Player) {
	player.moveHorizontal(-1);
}

function moveRight(player: Player) {
	player.moveHorizontal(1);
}

class Left implements Input {
	handle(player: Player) {
		moveLeft(player);
	}
}

class Right implements Input {
	handle(player: Player) {
		moveRight(player);
	}
}
```

이제 `moveHorizontally` 는 더 이상 필요 없어질 가능성이 높고, 나중에 **삭제 후 컴파일하기**로 제거할 수 있습니다.

### switch가 허용되는 유일한 경우
`switch` 가 완전히 금지되는 것은 아닙니다. 예외는 **외부 데이터의 타입 코드를 내부 객체로 매핑하는 경계**입니다. 예를 들어 파일이나 데이터베이스에 저장된 숫자 인덱스를 읽어 와서 내부의 객체로 바꾸는 단계는 분기가 애플리케이션 경계에 머무르므로 허용할 수 있습니다.

```
-- 예제 4.62 변경 전
enum TileCode {
	EMPTY = 0,
	WALL = 1,
	GOAL = 2
}

function stepOn(code: TileCode, player: Player) {
	switch (code) {
		case TileCode.EMPTY:
			return;
		case TileCode.WALL:
			player.stop();
			return;
		case TileCode.GOAL:
			player.win();
			return;
	}
}
```

이 코드는 외부 표현을 내부 로직 전체에 퍼뜨립니다. 대신 경계에서 한 번만 객체로 변환하면, 나머지 애플리케이션은 `Tile` 인터페이스만 알면 됩니다.

```
-- 예제 4.63 변경 후
enum TileCode {
	EMPTY = 0,
	WALL = 1,
	GOAL = 2
}

interface Tile {
	stepOn(player: Player): void;
}

class Empty implements Tile {
	stepOn(player: Player) {}
}

class Wall implements Tile {
	stepOn(player: Player) {
		player.stop();
	}
}

class Goal implements Tile {
	stepOn(player: Player) {
		player.win();
	}
}

function tileFromCode(code: TileCode): Tile {
	switch (code) {
		case TileCode.EMPTY:
			return new Empty();
		case TileCode.WALL:
			return new Wall();
		case TileCode.GOAL:
			return new Goal();
	}
}
```

이제 `switch` 는 외부 포맷을 내부 객체로 바꾸는 지점에만 남습니다. 이것이 `switch` 가 허용되는 사실상 유일한 경우입니다.

> **타입스크립트에서는**
> 열거형은 자바에서와 같이 클래스가 아닌 C#에서와 같은 숫자에 대한 명칭입니다. 따라서 숫자와 열거형 간의 변환이 필요하지 않으며 이전 코드와 같이 열거형 인덱스를 사용하면 됩니다.

### 규칙: switch를 사용하지 말 것

#### 정의
`default` 케이스가 없고 모든 `case` 에 반환 값이 있는 경우가 아니라면 `switch` 를 사용하지 마십시오.

#### 설명
`switch` 는 각각 버그로 이어지는 두 가지 '편의성'을 허용하기 때문에 문제가 있습니다. 첫 번째는 `switch` 로 `case` 를 분석할 때 모든 값에 대한 처리를 실행할 필요가 없다는 점입니다. 이를 위해 `switch` 는 `default` 키워드를 지원합니다. `default` 로 중복없이 여러 값을 지정할 수 있습니다. `switch` 를 사용할 경우 무엇을 처리할지와 무엇을 처리하지 않을지는 이제 불변속성입니다. 그러나 기본값이 지정된 다른 경우와 마찬가지로 새로운 값을 추가할 때 이러한 불변속성이 여전히 유효한지 컴파일러를 통해 판단할 수 없게 됩니다. 컴파일러 입장에서는 우리가 새로 추가한 값의 처리를 잊어버린 것인지, 아니면 `default` 에 지정하고자 한 것인지를 구분할 방법이 없습니다.

`switch` 의 또 다른 문제는 `break` 키워드를 만나기 전까지 케이스를 연속해서 실행하는 폴스루(fall-through) 로직이라는 점입니다. `switch` 를 사용할 때 `break` 키워드를 쓰는 것을 누락하고 알아채지 못하기가 쉽습니다.

일반적으로 `switch` 는 멀리하는 것이 좋습니다. 이 규칙의 정의에 명시된 내용을 참고하여 이러한 문제를 고칠 수 있습니다. 기능을 `default` 에 두지 않는 것입니다. 모든 언어가 `default` 생략을 허용하는것은 아닌데, 사용하는 언어가 `default` 의 생략을 허용하지 않으면 `switch` 를 사용하지 말아야 합니다.

모든 케이스에 `return` 을 지정해서 폴스루 문제를 해결합니다. 결과적으로 폴스루 로직이 없어짐으로써 `break` 를 잊어버려 문제가 발생할 여지가 없어집니다.

#### 스멜
`switch` 는 컨텍스트, 즉 값 `X` 를 처리하는 방법에 초점을 맞춥니다. 반대로, 클래스에 기능을 밀어 넣을 때는 데이터, 즉 이 값(객체)이 상황 `X` 를 처리하는 방법에 초점을 맞춥니다. 컨텍스트에 초점을 맞춘다는 것은 데이터에서 불변속성을 더 멀리 위치시켜 불변속성을 전역화하는 것을 의미합니다.

#### 의도
`switch` 를 `else if` 체인 문으로 변환하고 이를 다시 클래스로 만든다는 것입니다. 다시 코드를 클래스로 이관하면서 `if` 문들은 제거됩니다. 이 방법을 통해 결과적으로 기능을 유지하면서 새로운 값을 더 쉽고 안전하게 추가할 수 있게 됩니다.


# 코드 중복 처리
클래스로 책임을 옮기기 시작하면, 이번에는 클래스들 사이의 중복이 눈에 띄기 시작합니다. 이때 가장 먼저 떠오르는 해법이 상속이지만, 이 책은 그 방향을 경계합니다.

### 인터페이스 대신 추상 클래스를 사용할 수는 없을까?
중복이 보이면 추상 클래스를 만들고 공통 코드를 부모로 올리고 싶은 유혹이 강합니다. 하지만 이렇게 하면 코드 재사용과 타입 계층이 강하게 묶입니다.

```
-- 예제 4.64 추상 클래스로 해결하려는 시도
abstract class Input {
	protected consumeEnergy(player: Player) {
		player.energy -= 1;
	}

	abstract handle(player: Player): void;
}

class Left extends Input {
	handle(player: Player) {
		player.moveHorizontal(-1);
		this.consumeEnergy(player);
	}
}

class Right extends Input {
	handle(player: Player) {
		player.moveHorizontal(1);
		this.consumeEnergy(player);
	}
}
```

이 구조는 당장은 편하지만, 모든 입력 타입이 하나의 클래스 계층에 묶이고 단일 상속 제약까지 떠안게 됩니다.

### 규칙: 인터페이스에서만 상속받을 것

#### 정의
코드 재사용을 위해 클래스를 상속하지 말고, 타입을 표현하기 위해 인터페이스만 상속받으십시오.

#### 설명
상속은 공통 코드를 재사용하는 가장 손쉬운 도구처럼 보이지만, 동시에 불변속성을 부모 클래스 전체에 퍼뜨립니다. 반면 인터페이스는 "무엇을 할 수 있는가"만 표현하고, 구현은 독립적으로 유지할 수 있습니다.

#### 예제
인터페이스를 사용하면 타입만 공유하고, 공통 로직은 별도 함수나 조합 가능한 객체로 다룰 수 있습니다.

```
-- 예제 4.65 인터페이스로 분리
interface Input {
	handle(player: Player): void;
}

function consumeEnergy(player: Player) {
	player.energy -= 1;
}

class Left implements Input {
	handle(player: Player) {
		player.moveHorizontal(-1);
		consumeEnergy(player);
	}
}

class Right implements Input {
	handle(player: Player) {
		player.moveHorizontal(1);
		consumeEnergy(player);
	}
}
```

### 클래스에 있는 코드의 중복은 다 무엇일까?
클래스들 사이에 같은 줄이 보인다고 해서 항상 같은 책임이 중복된 것은 아닙니다. 중복이 정말 같은 개념이라면, 부모 클래스로 올리기보다 **별도 객체나 함수로 추출해서 조합**하는 편이 더 안전합니다.

```
-- 예제 4.66 변경 전
class Left implements Input {
	handle(player: Player) {
		player.moveHorizontal(-1);
		player.energy -= 1;
	}
}

class Right implements Input {
	handle(player: Player) {
		player.moveHorizontal(1);
		player.energy -= 1;
	}
}
```

```
-- 예제 4.67 변경 후
class HorizontalMove {
	constructor(private deltaX: number) {}

	apply(player: Player) {
		player.moveHorizontal(this.deltaX);
		player.energy -= 1;
	}
}

class Left implements Input {
	private move = new HorizontalMove(-1);

	handle(player: Player) {
		this.move.apply(player);
	}
}

class Right implements Input {
	private move = new HorizontalMove(1);

	handle(player: Player) {
		this.move.apply(player);
	}
}
```

이렇게 하면 중복을 제거하면서도 `Left` 와 `Right` 의 타입 계층은 독립적으로 유지할 수 있습니다. 즉, **중복 제거에는 상속보다 컴포지션이 기본값**입니다.


# 복잡한 if 체인 구문 리팩터링
긴 `if` 체인을 추출하고, 메서드를 전문화하고, 타입 코드를 클래스로 바꿨다면 마지막 남은 문제는 여러 축의 조건이 한곳에 섞여 있는 경우입니다. 이런 복잡한 체인은 각 축을 담당하는 객체에 책임을 나눠주는 방식으로 풀어야 합니다.

```
-- 예제 4.68 변경 전
function collide(player: Player, tile: Tile, input: Input) {
	if (input instanceof Left && tile instanceof Empty) {
		player.moveLeft();
	} else if (input instanceof Left && tile instanceof Box) {
		player.pushLeft();
	} else if (input instanceof Right && tile instanceof Empty) {
		player.moveRight();
	} else if (input instanceof Right && tile instanceof Box) {
		player.pushRight();
	}
}
```

이 코드는 입력의 종류와 타일의 종류라는 두 개의 축이 한 메서드에 겹쳐 있습니다. 이를 객체로 분산하면 각 객체는 자신이 알아야 할 축만 담당합니다.

```
-- 예제 4.69 변경 후
interface Input {
	handle(tile: Tile, player: Player): void;
}

class Left implements Input {
	handle(tile: Tile, player: Player) {
		tile.handleLeft(player);
	}
}

class Right implements Input {
	handle(tile: Tile, player: Player) {
		tile.handleRight(player);
	}
}

interface Tile {
	handleLeft(player: Player): void;
	handleRight(player: Player): void;
}

class Empty implements Tile {
	handleLeft(player: Player) {
		player.moveLeft();
	}

	handleRight(player: Player) {
		player.moveRight();
	}
}

class Box implements Tile {
	handleLeft(player: Player) {
		player.pushLeft();
	}

	handleRight(player: Player) {
		player.pushRight();
	}
}
```

이 버전에서는 새 입력 타입이나 새 타일 타입을 추가할 때 기존 거대한 `if` 체인을 다시 열지 않아도 됩니다.


# 필요 없는 코드 제거하기
전문화, 추출, 클래스 이관을 거치고 나면 예전의 일반 함수나 보조 메서드는 종종 더 이상 필요하지 않게 됩니다. 이때 안전하게 정리하는 방법이 **삭제 후 컴파일하기**입니다.

### 리팩터링 패턴: 삭제 후 컴파일하기

#### 설명
이 리팩터링 패턴은 "정말 더 이상 필요 없는가?"를 추측하지 않고, 실제로 삭제한 뒤 컴파일러가 알려주는 오류를 따라가며 정리하는 방법입니다. 삭제 대상이 작고 컴파일 피드백이 빠를수록 특히 강력합니다.

#### 절차
1. 더 이상 필요 없어 보이는 메서드나 클래스를 삭제합니다.
2. 컴파일합니다.
3. 남아 있는 호출 위치가 있으면 새 구조에 맞게 고칩니다.
4. 다시 컴파일하고, 오류가 없어질 때까지 반복합니다.

#### 예제
메서드 전문화 이후에는 예전의 일반 메서드가 단순 위임만 남는 경우가 많습니다.

```
-- 예제 4.70 변경 전
enum Direction {
	LEFT,
	RIGHT
}

function moveHorizontally(player: Player, direction: Direction) {
	if (direction === Direction.LEFT) {
		player.moveHorizontal(-1);
		return;
	}
	player.moveHorizontal(1);
}

function moveLeft(player: Player) {
	moveHorizontally(player, Direction.LEFT);
}

function moveRight(player: Player) {
	moveHorizontally(player, Direction.RIGHT);
}
```

모든 호출부가 `moveLeft`, `moveRight` 로 바뀌었다면, 이제 `moveHorizontally` 는 지워도 됩니다. 컴파일러가 남아 있는 호출이 있다면 바로 알려줄 것입니다.

```
-- 예제 4.71 변경 후
function moveLeft(player: Player) {
	player.moveHorizontal(-1);
}

function moveRight(player: Player) {
	player.moveHorizontal(1);
}

// 삭제된 moveHorizontally
```

이 패턴의 장점은 "혹시 어딘가에서 아직 쓰고 있지 않을까?"라는 막연한 불안을 컴파일 오류라는 구체적인 신호로 바꿔준다는 데 있습니다.


# 요약
- 긴 `if` 체인은 먼저 한 메서드로 모은 뒤, 메서드 전문화와 클래스 이관으로 단계적으로 줄여나갑니다.
- `메서드 전문화`는 재사용성을 줄이는 것이 아니라, 삭제 가능성과 이해 가능성을 높이는 리팩터링입니다.
- `switch` 는 외부의 타입 코드를 내부 객체로 바꾸는 경계에서만 제한적으로 허용할 수 있습니다.
- 클래스 중복은 추상 클래스보다 인터페이스와 컴포지션으로 다루는 편이 결합을 낮춥니다.
- 구조 변경이 끝난 뒤에는 `삭제 후 컴파일하기`로 남은 구 버전을 안전하게 제거합니다.
