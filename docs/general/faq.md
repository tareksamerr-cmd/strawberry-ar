---
title: الأسئلة الشائعة
faq: true
---

# الأسئلة الشائعة (Frequently Asked Questions)

## كيف يمكنني إخفاء حقل (field) من GraphQL؟

توفر Strawberry نوعاً خاصاً يسمى `Private` يمكن استخدامه لإخفاء الحقول من GraphQL. على سبيل المثال، الكود التالي:

```python
import strawberry


@strawberry.type
class User:
    name: str
    age: int
    password: strawberry.Private[str]


@strawberry.type
class Query:
    @strawberry.field
    def user(self) -> User:
        return User(name="Patrick", age=100, password="This is fake")


schema = strawberry.Schema(query=Query)
```

سينتج عنه المخطط (schema) التالي:

```graphql
type Query {
  user: User!
}

type User {
  name: String!
  age: Int!
}
```

## كيف يمكنني التعامل مع الاستيرادات الدائرية (circular imports)؟

في الحالات التي يكون لديك فيها circular imports، يمكنك استخدام `strawberry.lazy` لحل هذه الاستيرادات الدائرية. على سبيل المثال:

```python
# posts.py
from typing import TYPE_CHECKING, Annotated

import strawberry

if TYPE_CHECKING:
    from .users import User


@strawberry.type
class Post:
    title: str
    author: Annotated["User", strawberry.lazy(".users")]
```

لمزيد من المعلومات، راجع توثيق [Lazy types](./types/lazy.md).

## هل يمكنني إعادة استخدام أنواع الكائنات (Object Types) مع كائنات الإدخال (Input Objects)؟

للأسف لا، لأنه، كما تحدد [مواصفات GraphQL](https://spec.graphql.org/June2018/#sec-Input-Objects)، هناك فرق بين Object Types وأنواع الإدخال (Input types):

> نوع كائن GraphQL (ObjectTypeDefinition) المحدد أعلاه غير مناسب لإعادة الاستخدام هنا، لأن أنواع الكائنات يمكن أن تحتوي على حقول تحدد معاملات (arguments) أو تحتوي على مراجع لواجهات (interfaces) واتحادات (unions)، ولا شيء من هذين الأمرين مناسب للاستخدام كمعامل إدخال (input argument). لهذا السبب، تحتوي كائنات الإدخال على نوع منفصل في النظام.

وهذا ينطبق أيضاً على حقول Input types: يمكنك فقط استخدام Strawberry Input types أو scalar.

راجع توثيق [Input Types](./types/input-types.md) الخاص بنا.

## هل يمكنني استخدام asyncio مع Strawberry و Django؟

نعم، توفر Strawberry عرضاً غير متزامن (async view) يمكن استخدامه مع Django. يمكنك التحقق من [Async Django](./integrations/django.md#async-django) لمزيد من المعلومات.
```
