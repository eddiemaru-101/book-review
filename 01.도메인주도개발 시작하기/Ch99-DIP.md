# DIP(Dependency Inversion Principle)
> Ch99 시리즈는 책을 다보기엔 시간이 없어서 개발방향 설계에 필요한 내용만 키워드별로 정리한 내용이다. 

<br><br>


## (1)DIP 필요성 

**DIP(Dependency Inversion Principle)는 고수준 모듈이 저수준 모듈에 의존하지 않도록 의존 방향을 역전시키는 원칙이다.**



### 배경 — 왜 나왔나
- 도메인, 인프라스트럭처를 구분짓고 비즈니스 로직을 유지하기 위해서 

레이어드 아키텍처에서 문제가 생겼다.
```
# 문제 상황
domain/order.py
    └── import infrastructure/order_repository  # ← 도메인이 인프라를 직접 의존
```

```
결과:
- DB가 MySQL → PostgreSQL로 바뀌면 domain 코드도 바꿔야 함
- 도메인 테스트할 때 DB가 반드시 필요함
- 도메인이 인프라 기술에 오염됨
```



**고수준 vs 저수준**
```
고수준 모듈    비즈니스 규칙, 정책    Domain, Application
저수준 모듈    기술적 구현 세부사항   Infrastructure (DB, 외부 API)
```



**DIP 적용 전후**
```
적용 전:
Domain ──────────▶ Infrastructure
(고수준)             (저수준)
도메인이 인프라를 직접 앎


적용 후:
Domain ──────────▶ Interface (추상)
                        ▲
Infrastructure ─────────┘
도메인은 인터페이스만 앎
인프라가 인터페이스를 구현
```





## (2) 구현

**DIP를 코드로 구현하는 방법은 인터페이스를 도메인 레이어에 정의하고 인프라가 구현하는 것이다.**


**디렉터리 구조**
```
shop/
└── domain/
│   └── order_aggregate/
│       ├── order.py
│       └── order_repository_interface.py   ← 인터페이스를 도메인에 정의
│
└── infrastructure/
    └── order_repository.py                 ← 인터페이스를 인프라에서 구현
```



**코드**
```
# domain/order_aggregate/order_repository_interface.py
# 도메인 레이어에 인터페이스 정의

from abc import ABC, abstractmethod         # ← 추상 클래스

class OrderRepositoryInterface(ABC):        # ← 인터페이스 역할
    
    @abstractmethod
    def save(self, order: Order):           # ← 어떻게 저장하는지 모름
        pass                                #   저장한다는 사실만 앎

    @abstractmethod
    def find_by_id(self, order_id: str) -> Order:
        pass
```
- `ABC`: 추상 클래스 선언. 이 클래스를 직접 `OrderRepositoryInterface()` 로 쓸 수 없다.
- `@abstractmethod`: 이 메서드를 구현하지 않은 자식 클래스는 인스턴스화 시 에러 발생.
- **핵심**: 도메인은 "저장한다", "찾는다" 는 사실만 안다. MySQL인지 PostgreSQL인지 모른다.



```
# infrastructure/order_repository.py
# 인프라 레이어에서 인터페이스 구현

class OrderRepository(OrderRepositoryInterface):    # ← 인터페이스 구현
    
    def save(self, order: Order):                   # ← 실제 DB 저장 구현
        with transaction():
            db.execute("""
                INSERT INTO orders (
                    order_id, customer_id, status,
                    shipping_city, shipping_street
                ) VALUES (?, ?, ?, ?, ?)
            """,
                order.order_id,
                order.customer_id,
                order._status,
                order._shipping_address.city,
                order._shipping_address.street
            )

    def find_by_id(self, order_id: str) -> Order:
        row = db.query("SELECT * FROM orders WHERE order_id = ?", order_id)
        # DB → 도메인 객체 조립
        order = Order.__new__(Order)
        order.order_id = row["order_id"]
        order._status = row["status"]
        return order
```
- `OrderRepository(OrderRepositoryInterface)`: 인터페이스 상속. `save`, `find_by_id` 구현 안 하면 에러.
- `Order.__new__(Order)`: `__init__` 을 건너뛰고 객체만 생성. DB에서 불러올 때 새 ID가 생성되는 걸 막는다.
- **핵심**: 도메인은 이 파일의 존재를 모른다. 인프라만 도메인을 안다.


```
# application/order_service.py
# 인터페이스에만 의존

class OrderService:
    def __init__(self, order_repository: OrderRepositoryInterface): # ← 인터페이스 타입
        self.order_repository = order_repository    # ← 구현체가 뭔지 모름

    def place_order(self, customer_id, items, shipping_city, shipping_street):
        order = Order(customer_id)
        order.set_shipping_address(Address(shipping_city, shipping_street))
        for item in items:
            order.add_item(item["product_id"], item["quantity"], Money(item["price"]))
        order.confirm()
        self.order_repository.save(order)           # ← 인터페이스 메서드 호출
        return order.order_id
```
- `order_repository: OrderRepositoryInterface`: 타입 힌트가 인터페이스다. 구현체가 뭔지 신경 안 쓴다.
- **핵심**: `OrderService` 는 `OrderRepositoryInterface` 만 안다. DB가 바뀌어도 이 파일은 안 바뀐다.



```
# 실제 사용 - 구현체 주입
order_service = OrderService(OrderRepository())         # MySQL 구현체 주입

# DB 교체 시
order_service = OrderService(PostgreSQLOrderRepository()) # 이 한 줄만 바뀜

# 테스트 시
order_service = OrderService(MockOrderRepository())     # Mock 주입, DB 불필요
```
- **핵심**: `OrderService` 생성할 때 구현체를 밖에서 주입한다. 이걸 의존성 주입(DI) 이라고 한다.
- DIP(방향 역전 원칙) 를 구현하는 수단이 DI(의존성 주입) 다.



**핵심 — 의존 방향**


```
DIP 적용 전:
OrderService ──▶ OrderRepository (구현체)
Application      Infrastructure
고수준이 저수준을 직접 앎


DIP 적용 후:
OrderService ──▶ OrderRepositoryInterface (인터페이스)
Application      Domain
                      ▲
              OrderRepository (구현체)
              Infrastructure
고수준은 인터페이스만 앎
저수준이 인터페이스를 구현
```

```
Application ──▶ Interface ◀── Infrastructure
   (고수준)      (도메인)        (저수준)

import 방향:
order_service.py    → order_repository_interface.py  (도메인만 import)
order_repository.py → order_repository_interface.py  (도메인만 import)
order_repository.py ← order_service.py 절대 없음     (역방향 import 금지)
```


**DIP의 실제 효과**
```
# DB를 MySQL → PostgreSQL로 바꿀 때
class PostgreSQLOrderRepository(OrderRepositoryInterface):  # ← 새 구현체만 추가
    def save(self, order: Order):
        # PostgreSQL 방식으로 저장
        pass

# OrderService 코드는 한 줄도 안 바뀜
order_service = OrderService(PostgreSQLOrderRepository())   # ← 구현체만 교체
```

```
# 테스트할 때 DB 없이 테스트
class MockOrderRepository(OrderRepositoryInterface):        # ← Mock 구현체
    def __init__(self):
        self.saved_orders = []

    def save(self, order: Order):
        self.saved_orders.append(order)                     # ← DB 없이 메모리에 저장

    def find_by_id(self, order_id: str) -> Order:
        return next(o for o in self.saved_orders if o.order_id == order_id)

# 테스트 코드
def test_place_order():
    mock_repo = MockOrderRepository()                       # ← DB 없이 테스트
    order_service = OrderService(mock_repo)
    order_id = order_service.place_order(
        customer_id="cust-001",
        items=[{"product_id": "prod-001", "quantity": 2, "price": 5000}],
        shipping_city="서울",
        shipping_street="강남대로 123"
    )
    assert mock_repo.saved_orders[0].order_id == order_id  # ← DB 없이 검증
```



**의존 방향 최종 정리**
```
shop/
└── domain/
│   └── order_aggregate/
│       ├── order.py                        비즈니스 규칙
│       └── order_repository_interface.py   인터페이스 정의
│                ▲
└── infrastructure/
    └── order_repository.py                 인터페이스 구현
            │
            └── import domain (단방향)      인프라 → 도메인만 허용
                                            도메인 → 인프라 절대 금지
```



**엔지니어 독백**
> DIP 처음 배우면 "인터페이스 하나 더 만드는 게 귀찮은데 왜 해요?"라고 한다. 프로젝트 초반엔 맞는 말이다.
> 
> 근데 테스트 코드 짤 때 진짜 빛난다. DIP 없으면 비즈니스 규칙 테스트 하나 돌리려고 DB 띄우고 테이블 만들고 데이터 넣어야 한다. DIP 있으면 Mock 하나 만들어서 DB 없이 순수하게 비즈니스 규칙만 테스트한다.
> 
> 실무에서 테스트 없이 배포하는 팀이 왜 항상 야근하는지 생각해봐라. DIP가 테스트를 쉽게 만들고, 테스트가 야근을 줄인다.






## (3) 파이썬 DIP 트렌드

**DIP는 "구체적인 것에 의존하지 말고 추상적인 것에 의존해라"다.**


### 가장 쉬운 예시 — 콘센트

```
# DIP 없는 경우
class Laptop:
    def __init__(self):
        self.charger = SamsungCharger()     # 삼성 충전기에 직접 의존

laptop = Laptop()
# 애플 충전기로 바꾸려면 Laptop 클래스를 뜯어고쳐야 함
```

```
# DIP 있는 경우
class Laptop:
    def __init__(self, charger: Charger):   # 충전기 인터페이스에만 의존
        self.charger = charger

laptop = Laptop(SamsungCharger())           # 삼성 충전기 꽂기
laptop = Laptop(AppleCharger())             # 애플 충전기로 교체
# Laptop 코드 한 줄도 안 바뀜
```
콘센트 규격(인터페이스)만 맞으면 어떤 충전기든 꽂을 수 있다. Laptop은 충전기 규격만 안다.




### 1세대 — ABC

**배경 — 왜 나왔나**
파이썬 초기엔 인터페이스 개념 자체가 없었다.
```
# 인터페이스 없던 시절
class OrderRepository:
    def save(self, order):
        pass                    # 그냥 빈 메서드

class MySQLRepository(OrderRepository):
    def sav(self, order):       # 오타로 메서드명 틀려도
        db.save(order)          # 런타임 전까지 에러 없음

repo = MySQLRepository()
repo.save(order)                # 여기서 터짐
                                # 실제 서비스에서 터짐
                                # 개발 단계에서 못 잡음
```
팀이 커지면서 "구현 강제"가 필요해졌다. 자바의 `interface` 키워드를 파이썬에서 흉내내기 위해 ABC가 나왔다.



**ABC 코드**
```
from abc import ABC, abstractmethod

class OrderRepositoryInterface(ABC):
    @abstractmethod
    def save(self, order) -> None:          # 구현 안 하면 인스턴스화 시 에러
        pass

    @abstractmethod
    def find_by_id(self, order_id: str):
        pass
```

```
# 메서드 빠뜨리면
class MySQLRepository(OrderRepositoryInterface):
    def save(self, order) -> None:
        db.save(order)
    # find_by_id 구현 안 함

repo = MySQLRepository()
# TypeError: Can't instantiate abstract class MySQLRepository
# with abstract method find_by_id
# → 인스턴스 생성 시점에 에러 잡힘
```



**ABC의 한계**

```
# 문제 1 - 구현체가 인터페이스를 반드시 알아야 함
class MySQLRepository(OrderRepositoryInterface):    # 상속 강제
    pass                                            # infrastructure가 domain을 import

# infrastructure/mysql_repository.py
from domain.order_repository_interface import OrderRepositoryInterface  # ← 의존성 생김
```

```
# 문제 2 - 외부 라이브러리 클래스는 상속 불가
# SQLAlchemy가 제공하는 Repository를 쓰고 싶을 때
class SQLAlchemyRepository(OrderRepositoryInterface):   # 상속받아야 함
    pass
# SQLAlchemy 코드를 수정할 수 없으면 쓸 수 없음
```

```
# 문제 3 - 테스트 Mock 만들 때도 상속 필요
class MockRepository(OrderRepositoryInterface):     # 테스트 코드도 상속 강제
    def save(self, order): pass
    def find_by_id(self, order_id): pass
```

**ABC 한계 요약**

```
구현체가 인터페이스 import 필요    → 의존성 발생
외부 라이브러리 클래스 사용 불가   → 확장성 떨어짐
테스트 Mock도 상속 필요            → 테스트 코드 장황
```



**엔지니어 독백**
> ABC 나왔을 때 자바 출신 개발자들이 "이제 파이썬도 인터페이스 있네"라고 좋아했다. 근데 쓰다 보니 문제가 생겼다. DIP를 지키려고 인터페이스를 도메인에 뒀는데, 인프라 구현체가 그 인터페이스를 상속받으려고 도메인을 import해야 했다. 이건 괜찮다. 근데 외부 라이브러리 클래스를 인터페이스에 맞추려면 래퍼 클래스를 하나 더 만들어야 했다. 보일러플레이트가 쌓이기 시작했다.




### 2세대 — Protocol

**배경 — ABC의 한계를 파이썬답게 해결**
파이썬은 원래 덕 타이핑 언어다.
```
# 덕 타이핑
"save 메서드가 있으면 Repository다"
"오리처럼 걸으면 오리다"
```
ABC는 이 철학에 반했다. Protocol은 덕 타이핑을 타입 시스템에 공식적으로 편입시켰다.



**Protocol 코드**
```
from typing import Protocol

class OrderRepositoryInterface(Protocol):
    def save(self, order) -> None: ...      # ... = 구현부 없음
    def find_by_id(self, order_id: str): ...
```

```
# 구현체가 Protocol을 상속받지 않아도 됨
class MySQLRepository:                      # 상속 없음
    def save(self, order) -> None:          # 메서드 시그니처만 맞으면 됨
        db.save(order)

    def find_by_id(self, order_id: str):
        return db.find(order_id)

# infrastructure가 domain을 import할 필요 없음
# 의존성 완전히 제거
```

```
# 외부 라이브러리도 바로 사용 가능
from sqlalchemy_repo import SQLAlchemyOrderRepository   # 외부 라이브러리

# save, find_by_id 메서드만 있으면 바로 사용
order_service = OrderService(SQLAlchemyOrderRepository())   # 래퍼 클래스 불필요
```

```
# 테스트 Mock도 상속 없이
class MockRepository:                       # 상속 없음
    def save(self, order): pass
    def find_by_id(self, order_id): pass

order_service = OrderService(MockRepository())  # 바로 사용
```



**ABC vs Protocol 의존성 비교**
```
ABC:
infrastructure/mysql_repository.py
    └── import domain/order_repository_interface.py  ← 의존성 있음

Protocol:
infrastructure/mysql_repository.py
    └── import 없음                                   ← 의존성 없음
```



**Protocol의 한계**
```
# 문제 - 런타임 체크 불가
repo = MySQLRepository()
isinstance(repo, OrderRepositoryInterface)
# TypeError: Protocols with non-method members don't support issubclass()

# 런타임에 "이 객체가 인터페이스를 구현했는지" 확인 불가
# mypy 없으면 에러를 런타임 전까지 못 잡음
```



**엔지니어 독백**
> Protocol이 나왔을 때 제일 좋았던 건 외부 라이브러리 클래스를 래퍼 없이 바로 쓸 수 있게 된 거다. ABC 쓸 때는 SQLAlchemy Repository 하나 쓰려고 래퍼 클래스 만들고 상속받고 메서드 위임하는 보일러플레이트를 매번 짰다. Protocol로 넘어오면서 그게 다 사라졌다.
> 
> 근데 팀에 mypy 안 쓰는 사람이 있으면 Protocol은 그냥 주석이다. 에러를 런타임에만 잡으니까. ABC는 인스턴스 만드는 순간 터지는데, Protocol은 mypy 돌려야 잡힌다. 팀 전체가 mypy를 CI에 넣기 전까지는 ABC가 더 안전할 수도 있다.





### 3세대 — Protocol + runtime_checkable

**배경 — Protocol의 런타임 체크 한계 해결**

Protocol은 정적 타입 체커(mypy)에서만 검증됐다. 런타임에 `isinstance` 체크가 필요한 상황이 생겼다.

```
# 런타임 체크가 필요한 상황
def process(repo):                          # 타입 힌트 없는 레거시 코드
    if isinstance(repo, OrderRepositoryInterface):  # 런타임 체크 필요
        repo.save(order)
    else:
        raise Exception("잘못된 Repository")
```

---

**runtime_checkable 코드**

```
from typing import Protocol, runtime_checkable

@runtime_checkable
class OrderRepositoryInterface(Protocol):
    def save(self, order) -> None: ...
    def find_by_id(self, order_id: str): ...
```

```
repo = MySQLRepository()
isinstance(repo, OrderRepositoryInterface)  # True  ← 런타임 체크 가능

class BrokenRepository:
    def save(self, order): pass
    # find_by_id 없음

isinstance(BrokenRepository(), OrderRepositoryInterface)    # False
```

```
# 주의 - 메서드 존재만 체크, 시그니처는 체크 안 함
class WrongRepository:
    def save(self, wrong_param) -> None:    # 파라미터 달라도
        pass
    def find_by_id(self, order_id: str):
        pass

isinstance(WrongRepository(), OrderRepositoryInterface)
# True  ← 메서드 이름만 맞으면 True
# 시그니처 검증은 여전히 mypy 필요
```

---

**3세대가 추가한 것**

```
ABC                런타임 강제, 상속 필요
Protocol           정적 검증, 상속 불필요, 런타임 체크 불가
Protocol+checkable 정적 검증, 상속 불필요, 런타임 체크 가능
```

---

**실무 선택 기준**

```
상황                                    선택
────────────────────────────────────────────────────────────────
자바 백그라운드 팀, mypy 없음           ABC
파이썬 네이티브 팀, mypy CI 적용        Protocol
레거시 코드와 혼용, 런타임 체크 필요    Protocol + runtime_checkable
```

---

**엔지니어 독백**

> runtime_checkable이 나온 배경이 재밌다. 실무에서 레거시 코드랑 새 코드를 섞어 쓸 때, 타입 힌트 없는 옛날 코드에서 isinstance 체크를 해야 하는 상황이 생겼다. Protocol만으론 부족했던 거다.
> 
> 근데 runtime_checkable도 완벽하지 않다. 메서드 시그니처까지 체크 안 한다는 게 함정이다. 파라미터 이름 틀려도 True가 나온다. 결국 완전한 검증은 mypy가 해야 한다.
> 
> 결론은 이거다. 어떤 방식을 쓰든 mypy를 CI에 넣어라. Protocol이든 ABC든 mypy 없으면 반쪽짜리다. 타입 힌트 열심히 달아봤자 mypy 안 돌리면 그냥 문서 주석이랑 다를 게 없다.
