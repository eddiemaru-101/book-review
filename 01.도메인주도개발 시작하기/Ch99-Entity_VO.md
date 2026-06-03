> Ch99 시리즈는 책을 다보기엔 시간이 없어서 개발방향 설계에 필요한 내용만 키워드별로 정리한 내용이다.

# Ch99-엔티티,VO
- Entity, VO를 어떻게 구분하는지 판단 기준
- Entity와 VO가 코드에서 어떻게 관계를 맺는지

## 1단계: 판단 기준

질문 2개로 결정한다.
```
Q1. 이 객체를 ID로 추적해야 하나?
Q2. 상태가 바뀌어야 하나?

둘 다 Yes → Entity
둘 다 No  → Value Object
```



**쇼핑몰 예시로 판단**
```
객체          Q1 ID추적    Q2 상태변경    결론
─────────────────────────────────────────────
주문          Yes          Yes           Entity
주문항목      Yes          No            Entity
고객          Yes          Yes           Entity
배송지        No           No            Value Object
금액          No           No            Value Object
주문상태      No           No            Value Object
```



**판단이 애매한 경우**
```
주소
- 고객 주소록 서비스    → 주소를 ID로 추적, 수정 가능 → Entity
- 주문 배송지           → 그냥 값, 수정하면 새 주소    → Value Object

금액
- 회계 시스템           → 금액 변경 이력 추적 필요     → Entity
- 주문 금액             → 그냥 값                      → Value Object
```
같은 개념이라도 **컨텍스트에 따라 Entity가 되기도 VO가 되기도 한다.**





## 2단계: 코드에서 Entity와 VO가 관계 맺는 방법



**관계 패턴 3가지**
```
1. Entity가 VO를 소유          Order가 Money, Address를 가짐
2. Entity가 Entity를 소유      Order가 OrderItem을 가짐
3. Entity가 다른 Entity를 ID로 참조   Order가 Customer를 ID로 참조
```



**패턴 1 — Entity가 VO를 소유**
```
class Order:
    def __init__(self, customer_id):
        self.order_id = str(uuid.uuid4())
        self._shipping_address = None       # ← VO 소유
        self._total_price = Money(0)        # ← VO 소유

    def set_shipping_address(self, address: Address):
        self._shipping_address = address    # ← VO 교체 (수정 아님)

    def add_item(self, product_id, quantity, price: Money):
        self._items.append(OrderItem(product_id, quantity, price))
        self._total_price = Money(           # ← 새 VO 생성 (기존 VO 수정 아님)
            self._total_price.amount + price.amount
        )
```

VO는 수정하지 않는다. 새 VO를 만들어서 교체한다.
```
# 나쁜 예 - VO 직접 수정
self._shipping_address.city = "부산"        # VO는 불변이어야 함

# 좋은 예 - 새 VO로 교체
self._shipping_address = Address("부산", "해운대로 123")
```



**패턴 2 — Entity가 Entity를 소유**
```
class Order:
    def __init__(self, customer_id):
        self.order_id = str(uuid.uuid4())
        self._items = []                    # ← OrderItem Entity 소유

    def add_item(self, product_id, quantity, price: Money):
        self._items.append(
            OrderItem(product_id, quantity, price)  # ← 내부에서만 생성
        )

    def remove_item(self, item_id):
        self._items = [
            item for item in self._items
            if item.item_id != item_id      # ← item_id로 구별 (Entity니까)
        ]
```

OrderItem은 ID로 구별한다. VO와 다른 점이다.
```
# VO 비교 - 값으로 비교
money_a = Money(5000)
money_b = Money(5000)
money_a == money_b      # True (값이 같으면 같음)

# Entity 비교 - ID로 비교
item_a = OrderItem("prod-001", 2, Money(5000))
item_b = OrderItem("prod-001", 2, Money(5000))
item_a.item_id == item_b.item_id    # False (ID가 다르면 다름)
```



**패턴 3 — Entity가 다른 Entity를 ID로 참조**
```
class Order:
    def __init__(self, customer_id: str):   # ← Customer 객체가 아닌 ID만 받음
        self.order_id = str(uuid.uuid4())
        self.customer_id = customer_id      # ← ID로만 참조

# 나쁜 예 - 객체 직접 참조
class Order:
    def __init__(self, customer: Customer): # ← Customer 객체 직접 참조
        self.customer = customer            # 결합도 생김
```

ID로만 참조하는 이유:
```
객체 직접 참조          ID 참조
──────────────────────────────────────────
Order가 Customer를 앎   Order는 customer_id만 앎
Customer 바뀌면         Customer 바뀌어도
Order도 영향받음        Order 영향 없음
애그리거트 경계 무너짐  애그리거트 경계 유지
```



**전체 관계 한눈에**
```
class Order:                                # Entity (Aggregate Root)
    def __init__(self, customer_id: str):
        self.order_id = str(uuid.uuid4())   # ← 자신의 ID
        self.customer_id = customer_id      # ← 다른 Entity는 ID로만 참조 (패턴 3)
        self._items = []                    # ← 내부 Entity 소유 (패턴 2)
        self._shipping_address = None       # ← VO 소유 (패턴 1)
        self._total_price = Money(0)        # ← VO 소유 (패턴 1)
```



**판단 기준 + 관계 패턴 정리**
```
상황                              패턴
────────────────────────────────────────────────────────
ID 추적 필요, 상태 변경 필요      Entity
값으로 구별, 불변                 Value Object
Entity 안에 VO                   직접 소유, 교체 시 새 객체
Entity 안에 Entity               직접 소유, ID로 구별
Entity가 외부 Entity 참조        ID로만 참조, 객체 직접 참조 안 함
```




**엔지니어 독백**
> Entity와 VO 구분이 처음엔 애매하게 느껴진다. 실무에서 빠르게 판단하는 방법이 있다. "이 객체를 나중에 찾아야 하나?"를 물어봐라.
> 
> 주문은 나중에 "주문번호 001 가져와"라고 찾는다. Entity다. 배송지는 "서울 강남대로 123 가져와"라고 찾지 않는다. 그냥 주문에 딸려온다. VO다.
> 
> 패턴 3이 제일 많이 틀린다. Order 안에 Customer 객체 통째로 넣는 경우가 많다. 그러면 Customer 정보 바꿀 때 Order도 같이 로딩해야 한다. DB 쿼리가 폭발한다. 다른 애그리거트는 무조건 ID로만 참조해라.
