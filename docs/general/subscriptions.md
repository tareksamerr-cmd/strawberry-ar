---
title: الاشتراكات (Subscriptions)
---

# الاشتراكات (Subscriptions)

في GraphQL، يمكنك استخدام الاشتراكات (subscriptions) لبث البيانات من الخادم. لتمكين ذلك باستخدام Strawberry، يجب أن يدعم الخادم الخاص بك واجهة بوابة خادم غير متزامنة (ASGI) ومقابس الويب (websockets) أو استخدام تكامل AIOHTTP.

إليك كيفية تعريف محلل (resolver) قادر على التعامل مع الاشتراكات:

```python
import asyncio
from typing import AsyncGenerator

import strawberry


@strawberry.type
class Query:
    @strawberry.field
    def hello(self) -> str:
        return "world"


@strawberry.type
class Subscription:
    @strawberry.subscription
    async def count(self, target: int = 100) -> AsyncGenerator[int, None]:
        for i in range(target):
            yield i
            await asyncio.sleep(0.5)


schema = strawberry.Schema(query=Query, subscription=Subscription)
```

مثل الاستعلامات (queries) والطفرات (mutations)، يتم تعريف subscriptions في فئة (class) وتمريرها إلى دالة المخطط (Schema). هنا نقوم بإنشاء دالة عد بسيطة تعد من 0 إلى الهدف مع النوم بين كل تكرار للحلقة.

<Note>

نوع الإرجاع لـ `count` هو `AsyncGenerator` حيث يكون الوسيط العام (generic argument) الأول هو النوع الفعلي للاستجابة، وفي معظم الحالات يجب ترك الوسيط الثاني كـ `None` (المزيد حول كتابة المولدات (Generator typing) [هنا](https://docs.python.org/3/library/typing.html#typing.AsyncGenerator)).

</Note>

سنرسل مستند GraphQL التالي إلى خادمنا للاشتراك في تدفق البيانات هذا:

```graphql
subscription {
  count(target: 5)
}
```

في هذا المثال، تبدو البيانات بهذا الشكل أثناء مرورها عبر websocket:

![عرض للبيانات التي تم تمريرها عبر websocket](../images/subscriptions-count-websocket.png)

هذا مثال قصير جداً لما هو ممكن. كما هو الحال مع queries و mutations، يمكن للاشتراك إرجاع أي نوع GraphQL، وليس فقط القيم القياسية (scalars) كما هو موضح هنا.

## توثيق الاشتراكات (Authenticating Subscriptions)

دون الخوض في تفاصيل [السبب](https://github.com/websockets/ws/issues/467)، لا يمكن تعيين رؤوس (headers) مخصصة على طلبات websocket التي تنشأ في المتصفحات. لذلك، عند إجراء أي طلبات GraphQL تعتمد على اتصال websocket، يكون التوثيق القائم على الرؤوس (header-based authentication) مستحيلاً.

حلول GraphQL الشائعة الأخرى، مثل Apollo على سبيل المثال، تنفذ وظائف لتمرير المعلومات من العميل إلى الخادم عند نقطة تهيئة اتصال websocket. بهذه الطريقة، يمكن تمرير المعلومات ذات الصلة بتهيئة اتصال websocket وعمر الاتصال بشكل عام إلى الخادم قبل بث أي بيانات من قبل الخادم. على هذا النحو، لا يقتصر الأمر على بيانات اعتماد التوثيق فقط!

يتبع تنفيذ Strawberry تنفيذ Apollo، والذي يحتوي على توثيق لتنفيذات [العميل](https://www.apollographql.com/docs/react/data/subscriptions/#5-authenticate-over-websocket-optional) و [الخادم](https://www.apollographql.com/docs/apollo-server/data/subscriptions/#operation-context)، من خلال قراءة محتويات رسالة اتصال websocket الأولية في كائن `info.context`.

مع استخدام Apollo-client كمثال لكيفية إرسال معلومات الاتصال الأولية هذه، يتم تعريف `ws-link` كـ:

```javascript
import { GraphQLWsLink } from "@apollo/client/link/subscriptions";
import { createClient } from "graphql-ws";

const wsLink = new GraphQLWsLink(
  createClient({
    url: "ws://localhost:4000/subscriptions",
    connectionParams: {
      authToken: "Bearer I_AM_A_VALID_AUTH_TOKEN",
    },
  }),
);
```

وبعد ذلك، عند إنشاء طلب Susbcription واتصال websocket الأساسي، يقوم Strawberry بحقن كائن `connectionParams` هذا كما يلي:

```python
import asyncio
from typing import AsyncGenerator

import strawberry

from .auth import authenticate_token


@strawberry.type
class Query:
    @strawberry.field
    def hello(self) -> str:
        return "world"


@strawberry.type
class Subscription:
    @strawberry.subscription
    async def count(
        self, info: strawberry.Info, target: int = 100
    ) -> AsyncGenerator[int, None]:
        connection_params: dict = info.context.get("connection_params")
        token: str = connection_params.get(
            "authToken"
        )  # يساوي "Bearer I_AM_A_VALID_AUTH_TOKEN"
        if not authenticate_token(token):
            raise Exception("Forbidden!")
        for i in range(target):
            yield i
            await asyncio.sleep(0.5)


schema = strawberry.Schema(query=Query, subscription=Subscription)
```

يتوقع Strawberry أن يكون كائن `connection_params` من أي نوع، لذا فإن العميل حر في إرسال أي كائن JSON صالح كرسالة أولية لاتصال websocket، والذي يتم تجريده كـ `connectionParams` في Apollo-client، وسيتم حقنه بنجاح في كائن `info.context`. الأمر متروك لك بعد ذلك للتعامل معه بشكل صحيح!

## أنماط الاشتراك المتقدمة (Advanced Subscription Patterns)

عادةً ما يقوم اشتراك GraphQL ببث شيء أكثر إثارة للاهتمام. مع وضع ذلك في الاعتبار، يمكن لدالة الاشتراك الخاصة بك إرجاع أحد:

- `AsyncIterator` (مكرر غير متزامن)، أو
- `AsyncGenerator` (مولد غير متزامن)

كلا هذين النوعين موثقان في [PEP-525][pep-525]. أي شيء يتم إنتاجه (yielded) من هذه الأنواع من resolvers سيتم شحنه عبر websocket. يجب توخي الحذر لضمان توافق القيم المعادة مع Schema الخاص بـ GraphQL.

فائدة AsyncGenerator، مقارنة بالمكرر (iterator)، هي أنه يمكن تقسيم منطق العمل المعقد إلى وحدة منفصلة داخل قاعدة الكود الخاصة بك. مما يسمح لك بالحفاظ على منطق resolver موجزاً.

المثال التالي مشابه للمثال أعلاه، باستثناء أنه يعيد AsyncGenerator إلى خادم ASGI المسؤول عن بث نتائج الاشتراك حتى يخرج المولد (Generator).

```python
import strawberry
import asyncio
import asyncio.subprocess as subprocess
from asyncio import streams
from typing import Any, AsyncGenerator, AsyncIterator, Coroutine, Optional


async def wait_for_call(coro: Coroutine[Any, Any, bytes]) -> Optional[bytes]:
    """
    wait_for_call تستدعي الكوروتين المزود في كتلة wait_for.

    هذا يخفف من الحالات التي لا ينتج فيها الكوروتين حتى يكمل مهمته.
    في هذه الحالة، قراءة سطر من StreamReader؛ إذا لم تكن هناك أحرف سطر `\n` في التدفق، فلن تخرج الدالة أبداً.
    """
    try:
        return await asyncio.wait_for(coro(), timeout=0.1)
    except asyncio.TimeoutError:
        pass


async def lines(stream: streams.StreamReader) -> AsyncIterator[str]:
    """
    lines تقرأ جميع الأسطر من التدفق المزود، وتفك تشفيرها كسلاسل UTF-8.
    """
    while True:
        b = await wait_for_call(stream.readline)
        if b:
            yield b.decode("UTF-8").rstrip()
        else:
            break


async def exec_proc(target: int) -> subprocess.Process:
    """
    exec_proc تبدأ عملية فرعية وتعيد المقبض الخاص بها.
    """
    return await asyncio.create_subprocess_exec(
        "/bin/bash",
        "-c",
        f"for ((i = 0 ; i < {target} ; i++)); do echo $i; sleep 0.2; done",
        stdout=subprocess.PIPE,
    )


async def tail(proc: subprocess.Process) -> AsyncGenerator[str, None]:
    """
    tail تقرأ من stdout حتى تنتهي العملية
    """
    # ملاحظة: حالات السباق (race conditions) ممكنة هنا لأننا في عملية فرعية.
    # في هذه الحالة، يمكن أن تنتهي العملية بين مسند الحلقة واستدعاء قراءة سطر من stdout.
    # هذا مثال جيد على سبب حاجتك لأن تكون دفاعياً باستخدام asyncio.wait_for في wait_for_call().
    while proc.returncode is None:
        async for l in lines(proc.stdout):
            yield l
    else:
        # قراءة أي شيء متبقٍ على الأنبوب بعد انتهاء العملية
        async for l in lines(proc.stdout):
            yield l


@strawberry.type
class Query:
    @strawberry.field
    def hello() -> str:
        return "world"


@strawberry.type
class Subscription:
    @strawberry.subscription
    async def run_command(self, target: int = 100) -> AsyncGenerator[str, None]:
        proc = await exec_proc(target)
        return tail(proc)


schema = strawberry.Schema(query=Query, subscription=Subscription)
```

[pep-525]: https://www.python.org/dev/peps/pep-0525/

## إلغاء الاشتراكات (Unsubscribing subscriptions)

في GraphQL، من الممكن إلغاء الاشتراك من اشتراك ما. يدعم Strawberry هذا السلوك، ويتم ذلك باستخدام كتلة `try...except`.

في Apollo-client، يمكن تحقيق إغلاق الاشتراك كما يلي:

```javascript
const client = useApolloClient();
const subscriber = client.subscribe({query: ...}).subscribe({...})
// ...
// تم الانتهاء من الاشتراك. الآن قم بإلغاء الاشتراك
subscriber.unsubscribe();
```

يمكن لـ Strawberry التقاط متى يقوم المشترك بإلغاء الاشتراك باستخدام استثناء `asyncio.CancelledError`.

```python
import asyncio
from typing import AsyncGenerator
from uuid import uuid4

import strawberry

# تتبع المشتركين النشطين
event_messages = {}


@strawberry.type
class Subscription:
    @strawberry.subscription
    async def message(self) -> AsyncGenerator[int, None]:
        try:
            subscription_id = uuid4()

            event_messages[subscription_id] = []

            while True:
                if len(event_messages[subscription_id]) > 0:
                    yield event_messages[subscription_id]
                    event_messages[subscription_id].clear()

                await asyncio.sleep(1)
        except asyncio.CancelledError:
            # التوقف عن الاستماع للأحداث
            del event_messages[subscription_id]
```

## بروتوكولات GraphQL عبر WebSocket (GraphQL over WebSocket protocols)

يدعم Strawberry كلاً من بروتوكول [graphql-ws](https://github.com/apollographql/subscriptions-transport-ws) القديم وبروتوكول [graphql-transport-ws](https://github.com/enisdenjo/graphql-ws) الأحدث الموصى به.

<Note>

يسمى مستودع بروتوكولات `graphql-transport-ws` بـ `graphql-ws`. ومع ذلك، فإن `graphql-ws` هو أيضاً اسم البروتوكول القديم. يشير هذا التوثيق دائماً إلى أسماء البروتوكولات.

</Note>

لاحظ أن البروتوكول الفرعي `graphql-ws` مدعوم بشكل أساسي للتوافق مع الإصدارات السابقة. اقرأ [إعلان بروتوكولات graphql-ws-transport](https://the-guild.dev/blog/graphql-over-websockets) لمعرفة المزيد حول سبب تفضيل البروتوكول الأحدث.

يسمح لك Strawberry باختيار البروتوكولات التي تريد قبولها. يمكن تكوين جميع التكاملات التي تدعم subscriptions بقائمة من `subscription_protocols` لقبولها. بشكل افتراضي، يتم قبول جميع البروتوكولات.

### AIOHTTP

```python
from strawberry.aiohttp.views import GraphQLView
from strawberry.subscriptions import GRAPHQL_TRANSPORT_WS_PROTOCOL, GRAPHQL_WS_PROTOCOL
from api.schema import schema

view = GraphQLView(
    schema, subscription_protocols=[GRAPHQL_TRANSPORT_WS_PROTOCOL, GRAPHQL_WS_PROTOCOL]
)
```

### ASGI

```python
from strawberry.asgi import GraphQL
from strawberry.subscriptions import GRAPHQL_TRANSPORT_WS_PROTOCOL, GRAPHQL_WS_PROTOCOL
from api.schema import schema

app = GraphQL(
    schema,
    subscription_protocols=[
        GRAPHQL_TRANSPORT_WS_PROTOCOL,
        GRAPHQL_WS_PROTOCOL,
    ],
)
```

### Django + Channels

```python
import os

from django.core.asgi import get_asgi_application
from strawberry.channels import GraphQLProtocolTypeRouter

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "mysite.settings")
django_asgi_app = get_asgi_application()

# استورد مخطط Strawberry الخاص بك بعد إنشاء تطبيق django ASGI
# هذا يضمن استدعاء django.setup() قبل استيراد أي نماذج ORM للمخطط.
from mysite.graphql import schema

application = GraphQLProtocolTypeRouter(
    schema,
    django_application=django_asgi_app,
)
```

ملاحظة: تحقق من صفحة [تكامل Channels](../integrations/channels.md) لمزيد من المعلومات المتعلقة بها.

### FastAPI

```python
from strawberry.fastapi import GraphQLRouter
from strawberry.subscriptions import GRAPHQL_TRANSPORT_WS_PROTOCOL, GRAPHQL_WS_PROTOCOL
from fastapi import FastAPI
from api.schema import schema

graphql_router = GraphQLRouter(
    schema,
    subscription_protocols=[
        GRAPHQL_TRANSPORT_WS_PROTOCOL,
        GRAPHQL_WS_PROTOCOL,
    ],
)

app = FastAPI()
app.include_router(graphql_router, prefix="/graphql")
```

### Quart

```python
from strawberry.quart.views import GraphQLView
from strawberry.subscriptions import GRAPHQL_TRANSPORT_WS_PROTOCOL, GRAPHQL_WS_PROTOCOL
from quart import Quart
from api.schema import schema

view = GraphQLView.as_view(
    "graphql_view",
    schema=schema,
    subscription_protocols=[
        GRAPHQL_TRANSPORT_WS_PROTOCOL,
        GRAPHQL_WS_PROTOCOL,
    ],
)

app = Quart(__name__)
app.add_url_rule(
    "/graphql",
    view_func=view,
    methods=["GET"],
    websocket=True,
)
```

## عمليات النتيجة الواحدة (Single result operations)

بالإضافة إلى _عمليات البث_ (أي subscriptions)، يدعم بروتوكول `graphql-transport-ws` ما يسمى بـ _عمليات النتيجة الواحدة_ (أي queries و mutations).

هذا يمكن العملاء من استخدام بروتوكول واحد واتصال واحد لـ queries و mutations و subscriptions. ألقِ نظرة على [مستودع البروتوكول](https://github.com/enisdenjo/graphql-ws) لمعرفة كيفية إعداد عميل graphql الذي تختاره بشكل صحيح.

يدعم Strawberry عمليات النتيجة الواحدة بشكل مباشر عند تمكين بروتوكول `graphql-transport-ws`. عمليات النتيجة الواحدة هي queries و mutations عادية، لذا لا داعي لتعديل أي resolvers.
