> Ch99 시리즈는 책을 다보기엔 시간이 없어서 개발방향 설계에 필요한 내용만 키워드별로 정리한 내용이다.

# Ch99-도메인-DB매핑

```
1단계: 도메인 객체 설계        Order, OrderItem, Money, Address 클래스
2단계: DB 테이블 설계          orders, order_items 테이블 구조
3단계: 매핑 구현               Repository에서 변환 코드 작성
4단계: 전체 흐름 연결          ← 아직 안 함
```
# 01. 도메인-DB설계는 각각인 이유
## (1)매핑이 필요한 이유

도메인 객체와 DB 테이블은 생긴 게 다르다.

```
# 도메인 객체
class Order:
    def __init__(self):
        self.order_id = "order-001"
        self._items = [OrderItem(...), OrderItem(...)]  # 객체 리스트
        self._status = "CONFIRMED"
        self._shipping_address = Address("서울시 강남구")  # Value Object
```

```
# DB 테이블
orders
- order_id
- status
- shipping_city      ← Address 객체가 컬럼으로 풀어짐
- shipping_street

order_items          ← 별도 테이블
- item_id
- order_id
- product_id
- quantity
- price
```

객체는 중첩 구조, DB는 테이블 구조다. 이 둘을 연결하는 게 매핑이다.

---

## (2) 작업 흐름

```
1단계: 도메인 객체 설계        "비즈니스 규칙 중심으로 클래스 설계"
       │
2단계: DB 테이블 설계          "어떤 테이블이 필요한가"
       │
3단계: 매핑 결정               "객체의 어느 필드가 어느 컬럼으로 가는가"
       │
4단계: 매핑 구현               "Repository에서 변환 코드 작성"
```






# 02.작업 구현


## 1단계: 도메인 객체 설계 — 비즈니스 규칙 중심으로 클래스를 설계한다.


**예시 도메인 — 쇼핑몰 주문**
요구사항:
```
고객이 상품을 주문하면
- 주문 항목이 1개 이상 있어야 한다
- 배송지 주소가 있어야 한다
- 주문 확정 후에는 항목 추가가 불가능하다
- 금액은 음수가 될 수 없다
```


**개발자의 생각 흐름**
```
요구사항 읽는다
       │
명사 추출 → Entity/VO 후보
       │
동사 추출 → 메서드/규칙 후보
       │
Entity인가 VO인가 결정
       │
클래스 설계
```



**명사/동사 추출**
```
명사                  판단
─────────────────────────────────────────
주문      Order       ID로 구별 → Entity
주문항목  OrderItem   Order에 종속 → Entity
배송지    Address     값으로 구별, 불변 → Value Object
금액      Money       값으로 구별, 불변 → Value Object
```

```
동사                  판단
─────────────────────────────────────────
항목 추가             Order.add_item()
주문 확정             Order.confirm()
배송지 설정           Order.set_shipping_address()
```



**도메인 객체 코드**
```
# Value Object — 값으로 구별, 불변
class Money:
    def __init__(self, amount: int):
        if amount < 0:
            raise Exception("금액은 음수가 될 수 없다")  # ← 비즈니스 규칙
        self.amount = amount

    def __eq__(self, other):
        return self.amount == other.amount              # ← 값으로 비교
```

```
# Value Object — 값으로 구별, 불변
class Address:
    def __init__(self, city: str, street: str):
        if not city or not street:
            raise Exception("주소는 비어있을 수 없다")  # ← 비즈니스 규칙
        self.city = city
        self.street = street

    def __eq__(self, other):
        return self.city == other.city and self.street == other.street
```

```
# Entity — Order에 종속된 내부 객체
class OrderItem:
    def __init__(self, product_id: str, quantity: int, price: Money):
        self.item_id = str(uuid.uuid4())
        self.product_id = product_id
        self.quantity = quantity
        self.price = price                              # ← int 아닌 Money
```

```
# Entity (Aggregate Root) — 비즈니스 규칙 소유
class Order:
    def __init__(self, customer_id: str):
        self.order_id = str(uuid.uuid4())
        self.customer_id = customer_id
        self._items = []                                # ← 직접 접근 차단
        self._status = "PENDING"
        self._shipping_address = None

    def add_item(self, product_id: str, quantity: int, price: Money):
        if self._status != "PENDING":
            raise Exception("확정된 주문에 항목 추가 불가")  # ← 비즈니스 규칙
        self._items.append(OrderItem(product_id, quantity, price))

    def set_shipping_address(self, address: Address):
        if self._status != "PENDING":
            raise Exception("확정된 주문의 배송지 변경 불가") # ← 비즈니스 규칙
        self._shipping_address = address

    def confirm(self):
        if not self._items:
            raise Exception("주문 항목 없음")               # ← 비즈니스 규칙
        if not self._shipping_address:
            raise Exception("배송지 없음")                  # ← 비즈니스 규칙
        self._status = "CONFIRMED"
```



### 1단계에서 개발자가 하는 생각
```
"DB를 전혀 생각하지 않는다"
       │
       ├── Address를 어떻게 저장할지 → 2단계에서 생각
       ├── OrderItem 테이블이 필요한지 → 2단계에서 생각
       └── order_id를 PK로 쓸지 → 2단계에서 생각

"지금은 비즈니스 규칙만 생각한다"
       │
       ├── 금액이 음수면 안 된다 → Money에 규칙
       ├── 확정 후 항목 추가 불가 → Order.add_item()에 규칙
       └── 배송지 없으면 확정 불가 → Order.confirm()에 규칙
```




### 종속된다는 의미 

**Order가 `_items` 리스트로 OrderItem을 소유하는 것으로 표현된다.**


**소유 관계 코드**
```
class Order:
    def __init__(self, customer_id: str):
        self.order_id = str(uuid.uuid4())
        self._items = []                    # ← Order가 OrderItem을 소유

    def add_item(self, product_id, quantity, price):
        self._items.append(OrderItem(...))  # ← Order를 통해서만 생성
```



**"종속된다"는 게 코드에서 의미하는 것 3가지**
```
1. OrderItem은 Order 없이 혼자 존재할 수 없다
2. OrderItem 생성은 Order.add_item()을 통해서만 한다
3. OrderItem 저장/삭제는 Order 단위로 한다
```



**외부에서 보면**
```
# OrderItem을 직접 만들지 않는다
item = OrderItem(...)           # ← 이렇게 안 함

# Order를 통해서만 만든다
order = Order(customer_id)
order.add_item(...)             # ← 이렇게만 함
```



**그럼 OrderItem에 order_id가 없어도 되나?**
도메인 객체 레벨에서는 필요 없다. Order가 `_items`로 들고 있으니까.
```
class OrderItem:
    def __init__(self, product_id, quantity, price):
        self.item_id = str(uuid.uuid4())
        self.product_id = product_id        # ← order_id 없음
        self.quantity = quantity
        self.price = price
```
DB 저장할 때는 `order_id`가 FK로 필요하다. 그건 2단계 매핑에서 추가한다.







## 2단계: DB 테이블 설계 — 도메인 객체를 보고 테이블 구조를 결정한다.


**개발자의 생각 흐름**
도메인 객체를 보면서 이런 질문을 던진다.
```
Order 클래스를 보면서:
- order_id, customer_id, status    → orders 테이블 컬럼
- _items (OrderItem 리스트)        → 별도 테이블 필요한가?
- _shipping_address (Address VO)   → 별도 테이블인가, 컬럼으로 풀건가?
```



**결정해야 하는 것 2가지**

**1) OrderItem — 별도 테이블로 분리**
```
이유:
- 주문 1개에 항목이 N개
- 1:N 관계는 별도 테이블이 자연스럽다
```

**2) Address — 컬럼으로 풀기 vs 별도 테이블**
```
선택지 A: orders 테이블에 컬럼으로 풀기
- shipping_city
- shipping_street

선택지 B: addresses 테이블로 분리
- address_id
- city
- street
```
실무에서는 A를 선호한다. Address는 Order 없이 단독으로 조회할 일이 없기 때문이다. 테이블 분리하면 조인 비용만 늘어난다.



**테이블 설계 결과**
```
orders 테이블
─────────────────────────────
order_id        VARCHAR  PK
customer_id     VARCHAR
status          VARCHAR
shipping_city   VARCHAR  ← Address.city
shipping_street VARCHAR  ← Address.street


order_items 테이블
─────────────────────────────
item_id         VARCHAR  PK
order_id        VARCHAR  FK → orders.order_id
product_id      VARCHAR
quantity        INTEGER
price           INTEGER  ← Money.amount
```



**도메인 객체 → 테이블 매핑 한눈에**
```
도메인 객체                    DB 테이블
────────────────────────────────────────────────────
Order                     →   orders
  order_id                →     order_id (PK)
  customer_id             →     customer_id
  _status                 →     status
  _shipping_address       →     shipping_city
    Address.city          →     shipping_city
    Address.street        →     shipping_street

OrderItem (리스트)         →   order_items (별도 테이블)
  item_id                 →     item_id (PK)
  (없었음)                →     order_id (FK 추가)
  product_id              →     product_id
  quantity                →     quantity
  price                   →     price
    Money.amount          →     price

Money                     →   컬럼으로 풀림 (별도 테이블 없음)
Address                   →   컬럼으로 풀림 (별도 테이블 없음)
```


### 2단계에서 개발자가 하는 생각
```
"도메인 객체 구조를 최대한 유지하되"
       │
       ├── 1:N 관계 → 별도 테이블
       ├── Value Object → 컬럼으로 풀기 (단독 조회 안 하면)
       └── Entity ID → PK

"DB 성능도 같이 고려한다"
       │
       ├── 조인이 많아지면 쿼리 느려짐
       ├── Address 별도 테이블 → 항상 조인 필요 → 비효율
       └── 컬럼으로 풀면 → orders 하나만 조회하면 끝
```










## 3단계: 매핑 구현 — Repository에서 도메인 객체 ↔ DB 테이블 변환 코드를 작성한다.



**개발자의 생각 흐름**

```
도메인 객체를 저장할 때:
Order 객체 → DB 컬럼으로 분해 → INSERT

DB에서 불러올 때:
DB 컬럼 → Order 객체로 조립 → 반환
```



**전체 코드**
```
# infrastructure/order_repository.py

class OrderRepository:

    def save(self, order: Order):                          # 도메인 객체 → DB
        # 1. Order 저장
        db.execute("""
            INSERT INTO orders (
                order_id,
                customer_id,
                status,
                shipping_city,                              -- Address.city
                shipping_street                             -- Address.street
            ) VALUES (?, ?, ?, ?, ?)
        """,
            order.order_id,
            order.customer_id,
            order._status,
            order._shipping_address.city,                  # VO를 컬럼으로 분해
            order._shipping_address.street                 # VO를 컬럼으로 분해
        )

        # 2. OrderItem 저장
        for item in order._items:                        # 리스트를 행으로 분해
            db.execute("""
                INSERT INTO order_items (
                    item_id,
                    order_id,                              -- FK 추가
                    product_id,
                    quantity,
                    price
                ) VALUES (?, ?, ?, ?, ?)
            """,
                item.item_id,
                order.order_id,                          # FK에 order_id 주입
                item.product_id,
                item.quantity,
                item.price.amount                       # Money를 int로 분해
            )


    def find_by_id(self, order_id: str) -> Order:          # DB → 도메인 객체
        # 1. orders 테이블 조회
        row = db.query("""
            SELECT * FROM orders WHERE order_id = ?
        """, order_id)

        # 2. order_items 테이블 조회
        item_rows = db.query("""
            SELECT * FROM order_items WHERE order_id = ?
        """, order_id)

        # 3. DB 컬럼 → 도메인 객체로 조립
        order = Order.__new__(Order)                  # ← 핵심: __init__ 안 씀
        order.order_id = row["order_id"]
        order.customer_id = row["customer_id"]
        order._status = row["status"]
        order._shipping_address = Address(             # 컬럼 → VO로 조립
            city=row["shipping_city"],
            street=row["shipping_street"]
        )
        order._items = [                               # 행 → 객체 리스트로 조립
            OrderItem(
                product_id=item["product_id"],
                quantity=item["quantity"],
                price=Money(item["price"])               # int → Money로 조립
            )
            for item in item_rows
        ]
        return order
```

---

**핵심 코드 2줄**
저장할 때 VO를 컬럼으로 분해:
```
order._shipping_address.city        # Address → city 컬럼
item.price.amount                   # Money → int 컬럼
```

불러올 때 컬럼을 VO로 조립:
```
Address(city=row["shipping_city"], street=row["shipping_street"])
Money(item["price"])
```

**`__new__` 를 쓰는 이유**
```
# 일반적인 방법 - __init__ 호출
order = Order(customer_id)          # __init__이 새 order_id를 생성해버림
                                    # DB에서 불러온 order_id가 덮어씌워짐

# Repository에서 쓰는 방법 - __new__ 호출
order = Order.__new__(Order)        # __init__ 건너뜀
order.order_id = row["order_id"]    # DB에서 불러온 id 직접 주입
```



**전체 흐름 재확인**
```
# 저장
order = Order("cust-001")
order.set_shipping_address(Address("서울", "강남대로"))
order.add_item("product-001", 2, Money(5000))
order.confirm()
order_repository.save(order)        # 도메인 객체 → DB 분해

# 조회
order = order_repository.find_by_id("order-001")
                                    # DB → 도메인 객체 조립
order.confirm()                     # 비즈니스 규칙 그대로 사용 가능
```



**엔지니어 독백**
> 매핑 코드 처음 짜면 "이게 왜 이렇게 복잡해요? 그냥 ORM 쓰면 안 돼요?"라고 한다. 맞는 말이다. SQLAlchemy 같은 ORM 쓰면 이 변환 코드를 상당 부분 자동화할 수 있다.
> 
> 근데 ORM 없이 직접 짜봐야 "도메인 객체랑 DB가 왜 구조가 다른지", "VO가 어떻게 컬럼으로 풀리는지"를 몸으로 이해하게 된다. ORM 먼저 배우면 이 개념이 블랙박스로 남는다.
> 
> 실무에서 ORM 매핑이 꼬이는 버그를 만나면, 결국 이 수동 매핑 개념으로 돌아와서 "ORM이 내부에서 뭘 하는지"를 이해해야 풀린다. 귀찮아도 한 번은 직접 짜봐라.






## 4단계: 전체 흐름 연결 — 실제 HTTP 요청이 들어왔을 때 레이어를 타고 흐르는 전체 코드다.



**흐름 재확인**
```
HTTP POST /orders
       │
       ▼
Presentation   order_router.py       요청 받아서 Application으로 넘김
       │
       ▼
Application    order_service.py      흐름 조율, 도메인 객체 사용
       │
       ▼
Domain         order.py              비즈니스 규칙 실행
       │
       ▼
Infrastructure order_repository.py   DB 저장 (3단계에서 작성한 코드)
```



**전체 코드**
```
# presentation/order_router.py

from fastapi import APIRouter
from application.order_service import OrderService

router = APIRouter()
order_service = OrderService(OrderRepository())

@router.post("/orders")
def place_order(request: dict):                     # HTTP 요청 받음
    order_id = order_service.place_order(
        customer_id=request["customer_id"],
        items=request["items"],
        shipping_city=request["shipping_city"],
        shipping_street=request["shipping_street"]
    )
    return {"order_id": order_id}                   # HTTP 응답 반환
```

```
# application/order_service.py

class OrderService:
    def __init__(self, order_repository: OrderRepository):
        self.order_repository = order_repository    # ← Repository 주입

    def place_order(
        self,
        customer_id: str,
        items: list,
        shipping_city: str,
        shipping_street: str
    ) -> str:

        # 1. 도메인 객체 생성
        order = Order(customer_id)

        # 2. 배송지 설정
        address = Address(city=shipping_city, street=shipping_street)
        order.set_shipping_address(address)

        # 3. 주문 항목 추가
        for item in items:
            order.add_item(
                product_id=item["product_id"],
                quantity=item["quantity"],
                price=Money(item["price"])
            )

        # 4. 주문 확정 (비즈니스 규칙 실행)
        order.confirm()                             # ← 규칙 검증은 여기서

        # 5. 저장
        self.order_repository.save(order)           # ← DB 저장

        return order.order_id
```

```
# domain/order_aggregate/order.py
# 1단계에서 작성한 코드 그대로

class Order:
    def __init__(self, customer_id: str):
        self.order_id = str(uuid.uuid4())
        self.customer_id = customer_id
        self._items = []
        self._status = "PENDING"
        self._shipping_address = None

    def add_item(self, product_id, quantity, price: Money):
        if self._status != "PENDING":
            raise Exception("확정된 주문에 항목 추가 불가")
        self._items.append(OrderItem(product_id, quantity, price))

    def set_shipping_address(self, address: Address):
        if self._status != "PENDING":
            raise Exception("확정된 주문의 배송지 변경 불가")
        self._shipping_address = address

    def confirm(self):
        if not self._items:
            raise Exception("주문 항목 없음")
        if not self._shipping_address:
            raise Exception("배송지 없음")
        self._status = "CONFIRMED"
```

```
# infrastructure/order_repository.py
# 3단계에서 작성한 코드 그대로

class OrderRepository:
    def save(self, order: Order):
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
            for item in order._items:
                db.execute("""
                    INSERT INTO order_items (
                        item_id, order_id, product_id, quantity, price
                    ) VALUES (?, ?, ?, ?, ?)
                """,
                    item.item_id,
                    order.order_id,
                    item.product_id,
                    item.quantity,
                    item.price.amount
                )
```



**실제 요청 데이터로 흐름 추적**
```
# HTTP 요청
POST /orders
{
    "customer_id": "cust-001",
    "shipping_city": "서울",
    "shipping_street": "강남대로 123",
    "items": [
        {"product_id": "prod-001", "quantity": 2, "price": 5000},
        {"product_id": "prod-002", "quantity": 1, "price": 3000}
    ]
}
```

```
1. order_router.py
   request 받음
   order_service.place_order() 호출

2. order_service.py
   Order("cust-001") 생성          order_id = "uuid-xxx" 자동 생성
   address = Address("서울", "강남대로 123")
   order.set_shipping_address(address)
   order.add_item("prod-001", 2, Money(5000))
   order.add_item("prod-002", 1, Money(3000))
   order.confirm()                  규칙 검증 통과
   order_repository.save(order)

3. order_repository.py
   orders 테이블 INSERT
   ┌─────────────────────────────────────────────┐
   │ order_id    │ cust-001 │ CONFIRMED │ 서울 │ 강남대로 123 │
   └─────────────────────────────────────────────┘

   order_items 테이블 INSERT
   ┌─────────────────────────────────────┐
   │ item_id │ uuid-xxx │ prod-001 │ 2 │ 5000 │
   │ item_id │ uuid-xxx │ prod-002 │ 1 │ 3000 │
   └─────────────────────────────────────┘

4. order_router.py
   {"order_id": "uuid-xxx"} 반환
```



**각 레이어 책임 재확인**
```
레이어              책임                        하지 않는 것
────────────────────────────────────────────────────────────────
Presentation        HTTP 요청/응답 변환          비즈니스 규칙
Application         흐름 조율                    비즈니스 규칙
Domain              비즈니스 규칙 실행           DB 저장 방법
Infrastructure      DB 저장/조회                 비즈니스 규칙
```



**엔지니어 독백**
> 이 흐름을 처음 보면 "코드가 너무 많은 거 아니에요? 그냥 service에서 다 하면 안 돼요?"라고 한다.
> 
> 작은 서비스에서는 맞는 말이다. 근데 비즈니스가 복잡해지면 이 구조가 진짜 빛난다. 주문 확정 규칙이 바뀌면 `order.py`만 보면 된다. DB가 MySQL에서 PostgreSQL로 바뀌면 `order_repository.py`만 바꾸면 된다. 테스트할 때 Repository를 Mock으로 교체하면 DB 없이 비즈니스 규칙만 테스트할 수 있다.
> 
> 레이어 분리의 진짜 가치는 "변경이 일어났을 때 어디만 바꾸면 되는지"가 명확한 거다. 이게 없으면 버그 하나 고치려다 다른 버그 3개 만드는 상황이 반복된다.
