---
title: أساسيات المخطط (Schema)
---

# أساسيات المخطط (Schema)

تستخدم خوادم GraphQL **مخططًا (schema)** لوصف شكل البيانات. يحدد المخطط تسلسلاً هرميًا من **الأنواع (types)** مع حقول يتم ملؤها من مخازن البيانات. يحدد المخطط أيضًا بالضبط ما هي الاستعلامات (queries) والطفرات (mutations) المتاحة للعملاء لتنفيذها.

يصف هذا الدليل لبنات البناء الأساسية للمخطط وكيفية استخدام Strawberry لإنشاء واحد.

## لغة تعريف المخطط (SDL)

هناك نهجان لإنشاء مخطط لخادم GraphQL. أحدهما يسمى "المخطط أولاً" (schema-first) والآخر يسمى "الكود أولاً" (code-first). تدعم Strawberry مخططات "الكود أولاً" *فقط*. قبل الغوص في "الكود أولاً"، دعنا نشرح أولاً ما هي لغة تعريف المخطط.

يعمل نهج "المخطط أولاً" باستخدام لغة تعريف المخطط الخاصة بـ GraphQL، والتي تم تضمينها في مواصفات GraphQL.

إليك مثال على مخطط تم تعريفه باستخدام SDL:

```graphql
type Book {
  title: String!
  author: Author!
}

type Author {
  name: String!
  books: [Book!]!
}
```

يحدد المخطط جميع الأنواع والعلاقات بينها. بهذا نتمكن من تمكين مطوري العملاء من رؤية البيانات المتاحة بالضبط وطلب مجموعة فرعية محددة من تلك البيانات.

<Note>

تحدد علامة `!` أن الحقل غير قابل للإلغاء (non-nullable).

</Note>

لاحظ أن المخطط لا يحدد كيفية الحصول على البيانات. يأتي ذلك لاحقًا عند تعريف المحللات (resolvers).

## نهج الكود أولاً (Code first approach)

كما ذكرنا، تستخدم Strawberry نهج "الكود أولاً". سيبدو المخطط السابق بهذا الشكل في Strawberry:

```python
import typing
import strawberry


@strawberry.type
class Book:
    title: str
    author: "Author"


@strawberry.type
class Author:
    name: str
    books: typing.List[Book]
```

كما ترى، الكود يطابق المخطط تقريبًا واحدًا لواحد، بفضل ميزة تلميحات النوع (type hints) في Python.

لاحظ أننا هنا أيضًا لا نحدد كيفية جلب البيانات، سيتم شرح ذلك في قسم المحللات (resolvers).

## الأنواع المدعومة (Supported types)

يدعم GraphQL بضعة أنواع مختلفة:

- أنواع سكالار (Scalar types)
- أنواع الكائنات (Object types)
- نوع الاستعلام (Query type)
- نوع الطفرة (Mutation type)
- أنواع الإدخال (Input types)

## أنواع سكالار (Scalar types)

<!--alex ignore-->

أنواع سكالار تشبه أنواع Python الأولية. إليك قائمة بأنواع سكالار الافتراضية في GraphQL:

- Int، عدد صحيح موقع 32 بت، يطابق int في Python
- Float، قيمة نقطة عائمة مزدوجة الدقة موقعة، يطابق float في Python
- String، يطابق str في Python
- Boolean، صحيح أو خطأ، يطابق bool في Python
- ID، معرف فريد يستخدم عادةً لإعادة جلب كائن أو كمفتاح لذاكرة التخزين المؤقت. يتم تسلسله كسلسلة نصية ومتاح كـ `strawberry.ID(“value”)`
- `UUID`، قيمة [UUID](https://docs.python.org/3/library/uuid.html#uuid.UUID) يتم تسلسلها كسلسلة نصية

<Note>

تتضمن Strawberry أيضًا دعمًا لكائنات التاريخ (date)، الوقت (time) والتاريخ والوقت (datetime)، وهي غير مدرجة رسميًا في مواصفات GraphQL، ولكن عادةً ما تكون مطلوبة في معظم الخوادم. يتم تسلسلها بتنسيق ISO-8601.

</Note>

<!--alex ignore-->

تعمل هذه الأوليات لغالبية حالات الاستخدام، ولكن يمكنك أيضًا تحديد [أنواع سكالار الخاصة بك](/docs/types/scalars#custom-scalars).

## أنواع الكائنات (Object types)

معظم الأنواع التي تحددها في مخطط GraphQL هي أنواع كائنات. يحتوي نوع الكائن على مجموعة من الحقول، يمكن أن يكون كل منها إما نوع سكالار أو نوع كائن آخر.

يمكن لأنواع الكائنات أن تشير إلى بعضها البعض، كما كان لدينا في مخططنا سابقًا:

```python
import typing
import strawberry


@strawberry.type
class Book:
    title: str
    author: "Author"


@strawberry.type
class Author:
    name: str
    books: typing.List[Book]
```

## توفير البيانات للحقول (Providing data to fields)

في المخطط أعلاه، يحتوي `Book` على حقل `author` ويحتوي `Author` على حقل `books` ومع ذلك لا نعرف كيف يمكن مطابقة بياناتنا لتحقيق بنية المخطط الموعود.

لتحقيق ذلك، نقدم مفهوم [_المحلل (resolver)_](../types/resolvers.md) الذي يوفر بعض البيانات لحقل من خلال دالة.

استمرارًا مع هذا المثال للكتب والمؤلفين، يمكن تعريف المحللات لتوفير قيم للحقول:

```python
def get_author_for_book(root) -> "Author":
    return Author(name="Michael Crichton")


@strawberry.type
class Book:
    title: str
    author: "Author" = strawberry.field(resolver=get_author_for_book)


def get_books_for_author(root) -> typing.List[Book]:
    return [Book(title="Jurassic Park")]


@strawberry.type
class Author:
    name: str
    books: typing.List[Book] = strawberry.field(resolver=get_books_for_author)


def get_authors(root) -> typing.List[Author]:
    return [Author(name="Michael Crichton")]


@strawberry.type
class Query:
    authors: typing.List[Author] = strawberry.field(resolver=get_authors)
    books: typing.List[Book] = strawberry.field(resolver=get_books_for_author)
```

توفر هذه الدوال لـ `strawberry.field` القدرة على تقديم البيانات لاستعلام GraphQL عند الطلب وهي العمود الفقري لجميع واجهات برمجة تطبيقات GraphQL.

هذا المثال بسيط لأن البيانات التي يتم حلها ثابتة تمامًا. ومع ذلك، عند بناء واجهات برمجة تطبيقات أكثر تعقيدًا، يمكن كتابة هذه المحللات لمطابقة البيانات من قواعد البيانات، على سبيل المثال إجراء استعلامات SQL باستخدام SQLAlchemy، وواجهات برمجة تطبيقات أخرى، على سبيل المثال إجراء طلبات HTTP باستخدام aiohttp.

لمزيد من المعلومات والتفاصيل حول الطرق المختلفة لكتابة المحللات، راجع [قسم المحللات](../types/resolvers.md).

## نوع الاستعلام (Query type)

يحدد نوع `Query` بالضبط ما هي استعلامات GraphQL (أي عمليات القراءة) التي يمكن للعملاء تنفيذها مقابل بياناتك. إنه يشبه نوع الكائن، لكن اسمه دائمًا هو `Query`.

يحدد كل حقل من نوع `Query` الاسم ونوع الإرجاع لاستعلام مدعوم مختلف. قد يشبه نوع `Query` لمخطط المثال الخاص بنا ما يلي:

```python
@strawberry.type
class Query:
    books: typing.List[Book]
    authors: typing.List[Author]
```

يحدد نوع الاستعلام هذا استعلامين متاحين: books و authors. يعيد كل استعلام قائمة من النوع المقابل.

مع واجهة برمجة تطبيقات قائمة على REST، من المحتمل أن يتم إرجاع books و authors بواسطة نقاط نهاية مختلفة (على سبيل المثال، /api/books و /api/authors). تمكن مرونة GraphQL العملاء من الاستعلام عن كلا الموردين بطلب واحد.

### هيكلة الاستعلام (Structuring a query)

عندما يبني عملاؤك استعلامات لتنفيذها مقابل رسم البيانات الخاص بك، فإن تلك الاستعلامات تطابق شكل أنواع الكائنات التي تحددها في مخططك.

بناءً على مخطط المثال الخاص بنا حتى الآن، يمكن للعميل تنفيذ الاستعلام التالي، الذي يطلب كلاً من قائمة بجميع عناوين الكتب وقائمة بجميع أسماء المؤلفين:

```graphql
query {
  books {
    title
  }

  authors {
    name
  }
}
```

سيستجيب خادمنا بعد ذلك للاستعلام بنتائج تطابق بنية الاستعلام، مثل هذا:

```json
{
  "data": {
    "books": [{ "title": "Jurassic Park" }],
    "authors": [{ "name": "Michael Crichton" }]
  }
}
```

على الرغم من أنه قد يكون مفيدًا في بعض الحالات جلب هاتين القائمتين المنفصلتين، إلا أن العميل سيفضل على الأرجح جلب قائمة واحدة من الكتب، حيث يتم تضمين مؤلف كل كتاب في النتيجة.

نظرًا لأن نوع Book في مخططنا يحتوي على حقل author من نوع Author، يمكن للعميل هيكلة استعلامه مثل هذا:

```graphql
query {
  books {
    title
    author {
      name
    }
  }
}
```

ومرة أخرى، سيستجيب خادمنا بنتائج تطابق بنية الاستعلام:

```json
{
  "data": {
    "books": [
      { "title": "Jurassic Park", "author": { "name": "Michael Crichton" } }
    ]
  }
}
```

## نوع الطفرة (Mutation type)

نوع `Mutation` مشابه في البنية والغرض لنوع Query. بينما يحدد نوع Query عمليات القراءة المدعومة لبياناتك، يحدد نوع `Mutation` عمليات الكتابة المدعومة.

يحدد كل حقل من نوع `Mutation` التوقيع ونوع الإرجاع لطفرة مختلفة. قد يشبه نوع `Mutation` لمخطط المثال الخاص بنا ما يلي:

```python
@strawberry.type
class Mutation:
    @strawberry.mutation
    def add_book(self, title: str, author: str) -> Book: ...
```

يحدد نوع الطفرة هذا طفرة واحدة متاحة، `addBook`. تقبل الطفرة وسيطين (title و author) وتعيد كائن Book تم إنشاؤه حديثًا. كما تتوقع، يتوافق كائن Book هذا مع البنية التي حددناها في مخططنا.

<Note>

تقوم Strawberry بتحويل أسماء الحقول من snake case إلى camel case افتراضيًا. يمكن تغيير ذلك عن طريق تحديد [تكوين `StrawberryConfig` مخصص على المخطط](../types/schema-configurations.md)

</Note>

### هيكلة الطفرة (Structuring a mutation)

مثل الاستعلامات، تطابق الطفرات بنية تعريفات الأنواع في مخططك. تقوم الطفرة التالية بإنشاء Book جديد وتطلب حقولاً معينة من الكائن الذي تم إنشاؤه كقيمة إرجاع:

```graphql
mutation {
  addBook(title: "Fox in Socks", author: "Dr. Seuss") {
    title
    author {
      name
    }
  }
}
```

كما هو الحال مع الاستعلامات، سيستجيب خادمنا لهذه الطفرة بنتيجة تطابق بنية الطفرة، مثل هذا:

```json
{
  "data": {
    "addBook": {
      "title": "Fox in Socks",
      "author": {
        "name": "Dr. Seuss"
      }
    }
  }
}
```

## أنواع الإدخال (Input types)

أنواع الإدخال هي أنواع كائنات خاصة تسمح لك بتمرير كائنات كوسائط للاستعلامات والطفرات (بدلاً من تمرير أنواع سكالار فقط). تساعد أنواع الإدخال في الحفاظ على نظافة تواقيع العمليات.

فكر في طفرتنا السابقة لإضافة كتاب:

```python
@strawberry.type
class Mutation:
    @strawberry.mutation
    def add_book(self, title: str, author: str) -> Book: ...
```

بدلاً من قبول وسيطين، يمكن لهذه الطفرة قبول نوع إدخال واحد يتضمن كل هذه الحقول. هذا مفيد بشكل إضافي إذا قررنا قبول وسيط إضافي في المستقبل، مثل تاريخ النشر.

تعريف نوع الإدخال مشابه لنوع الكائن، ولكنه يستخدم الكلمة المفتاحية input:

```python
@strawberry.input
class AddBookInput:
    title: str
    author: str


@strawberry.type
class Mutation:
    @strawberry.mutation
    def add_book(self, book: AddBookInput) -> Book: ...
```

لا يسهل هذا تمرير نوع AddBookInput داخل مخططنا فحسب، بل يوفر أيضًا أساسًا لتعليق الحقول بالأوصاف التي يتم كشفها تلقائيًا بواسطة الأدوات التي تدعم GraphQL:

```python
@strawberry.input
class AddBookInput:
    title: str = strawberry.field(description="The title of the book")
    author: str = strawberry.field(description="The name of the author")
```

يمكن أن تكون أنواع الإدخال مفيدة أحيانًا عندما تتطلب عمليات متعددة نفس المجموعة بالضبط من المعلومات، ولكن يجب عليك إعادة استخدامها باعتدال. قد تتباعد العمليات في النهاية في مجموعات الوسائط المطلوبة الخاصة بها.

## المزيد (More)

إذا كنت تريد معرفة المزيد عن تصميم المخطط، فتأكد من اتباع [الوثائق المقدمة من Apollo](https://www.apollographql.com/docs/apollo-server/schema/schema/#growing-with-a-schema).

