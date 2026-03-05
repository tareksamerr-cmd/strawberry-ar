---
title: الاستعلامات (Queries)
---

# الاستعلامات (Queries)

في GraphQL، تستخدم الاستعلامات (queries) لجلب البيانات من الخادم. في Strawberry، يمكنك تحديد البيانات التي يوفرها خادمك عن طريق تعريف أنواع الاستعلامات (query types).

بشكل افتراضي، يتم تجميع جميع الحقول التي تعرضها واجهة برمجة التطبيقات (API) تحت نوع استعلام جذري (root Query type).

إليك كيفية تعريف نوع استعلام جذري في Strawberry:

```python
@strawberry.type
class Query:
    name: str


schema = strawberry.Schema(query=Query)
```

ينشئ هذا مخططًا حيث يحتوي نوع الاستعلام الجذري (Query) على حقل واحد يسمى name.

كما تلاحظ، نحن لا نوفر طريقة لجلب البيانات. للقيام بذلك، نحتاج إلى توفير `resolver`، وهي دالة تعرف كيفية جلب البيانات لحقل معين.

على سبيل المثال، في هذه الحالة يمكن أن يكون لدينا دالة تعيد دائمًا نفس الاسم:

```python
def get_name() -> str:
    return "Strawberry"


@strawberry.type
class Query:
    name: str = strawberry.field(resolver=get_name)


schema = strawberry.Schema(query=Query)
```

لذا الآن، عند طلب حقل name، سيتم استدعاء دالة `get_name`.

بدلاً من ذلك، يمكن الإعلان عن حقل باستخدام مزين (decorator):

```python
@strawberry.type
class Query:
    @strawberry.field
    def name(self) -> str:
        return "Strawberry"
```

يدعم بناء الجملة الخاص بالمزين تحديد `graphql_type` للحالات التي لا يتطابق فيها نوع الإرجاع للدالة مع نوع GraphQL:

```python
class User:
    id: str
    name: str

    def __init__(self, id: str, name: str):
        self.id = id
        self.name = name

@strawberry.type(name="User")
class UserType:
    id: strawberry.ID
    name: str

@strawberry.type
class Query:
    @strawberry.field(graphql_type=UserType)
    def user(self) -> User
        return User(id="ringo", name="Ringo")
```

## الوسائط (Arguments)

يمكن لحقول GraphQL قبول وسائط، عادةً لتصفية أو استرداد كائنات محددة:

```python
FRUITS = [
    "Strawberry",
    "Apple",
    "Orange",
]


@strawberry.type
class Query:
    @strawberry.field
    def fruit(self, startswith: str) -> str | None:
        for fruit in FRUITS:
            if fruit.startswith(startswith):
                return fruit
        return None
```

### أوصاف الوسائط (Argument Descriptions)

استخدم `Annotated` لإعطاء وسيط الحقل وصفًا:

```python
from typing import Annotated
import strawberry


@strawberry.type
class Query:
    @strawberry.field
    def fruit(
        self,
        startswith: Annotated[
            str, strawberry.argument(description="Prefix to filter fruits by.")
        ],
    ) -> str | None: ...
```

### أنواع الوسائط (Argument Types)

كما هو الحال مع الحقول، يدعم `strawberry.argument` تحديد `graphql_type` للحالات التي لا يتطابق فيها نوع الوسيط للدالة مع نوع GraphQL.
يمكن أن يكون هذا مفيدًا للكتابة الثابتة (static typing)، حيث أن أنواع سكالار المخصصة ليست تعليقات توضيحية صالحة للنوع.

```python
BigInt = strawberry.scalar(
    int, name="BigInt", serialize=lambda v: str(v), parse_value=lambda v: int(v)
)


@strawberry.type
class Query:
    @strawberry.field
    def users(
        self,
        ids: Annotated[list[int], strawberry.argument(graphql_type=list[BigInt])],
    ) -> list[User]: ...
```
