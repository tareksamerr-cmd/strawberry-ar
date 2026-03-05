---
title: الطفرات (Mutations)
---

# الطفرات (Mutations)

على عكس الاستعلامات (queries)، تمثل الطفرات (mutations) في GraphQL عمليات تعديل البيانات على جانب الخادم و/أو تتسبب في آثار جانبية على الخادم. على سبيل المثال، يمكنك الحصول على طفرة تنشئ مثيلاً جديدًا في تطبيقك أو طفرة ترسل بريدًا إلكترونيًا. كما هو الحال في الاستعلامات، فإنها تقبل المعلمات ويمكنها إرجاع أي شيء يمكن أن يرجعه حقل عادي، بما في ذلك الأنواع الجديدة وأنواع الكائنات الموجودة. يمكن أن يكون هذا مفيدًا لجلب الحالة الجديدة لكائن بعد التحديث.

دعنا نحسن مشروع الكتب الخاص بنا من [البرنامج التعليمي للبدء](../index.md) وننفذ طفرة من المفترض أن تضيف كتابًا:

```python
import strawberry


# Reader, you can safely ignore Query in this example, it is required by
# strawberry.Schema so it is included here for completeness
@strawberry.type
class Query:
    @strawberry.field
    def hello() -> str:
        return "world"


@strawberry.type
class Mutation:
    @strawberry.mutation
    def add_book(self, title: str, author: str) -> Book:
        print(f"Adding {title} by {author}")

        return Book(title=title, author=author)


schema = strawberry.Schema(query=Query, mutation=Mutation)
```

مثل الاستعلامات، يتم تعريف الطفرات في فئة يتم تمريرها بعد ذلك إلى دالة Schema. هنا نقوم بإنشاء طفرة `addBook` تقبل عنوانًا ومؤلفًا وتعيد نوع `Book`.

سنرسل مستند GraphQL التالي إلى خادمنا لتنفيذ الطفرة:

```graphql
mutation {
  addBook(title: "The Little Prince", author: "Antoine de Saint-Exupéry") {
    title
  }
}
```

طفرة `addBook` هي مثال مبسط. في تطبيق حقيقي، ستحتاج الطفرات غالبًا إلى التعامل مع الأخطاء وإبلاغ تلك الأخطاء مرة أخرى إلى العميل. على سبيل المثال، قد نرغب في إرجاع خطأ إذا كان الكتاب موجودًا بالفعل.

يمكنك الاطلاع على وثائقنا حول [التعامل مع الأخطاء](/docs/guides/errors#expected-errors) لمعرفة كيفية إرجاع اتحاد من الأنواع من طفرة.

## الطفرات بدون بيانات مُرجعة (Mutations without returned data)

من الممكن أيضًا كتابة طفرة لا تعيد أي شيء.

يتم تعيين هذا إلى `Void` GraphQL scalar، ويعيد دائمًا `null`.

<CodeGrid>
```python
@strawberry.type
class Mutation:
    @strawberry.mutation
    def restart() -> None:
        print(f"Restarting the server")
```

```graphql
type Mutation {
  restart: Void
}
```

</CodeGrid>

<Note>

الطفرات ذات النتيجة الفارغة تتعارض مع [هذا الدليل الذي أنشأه المجتمع حول أفضل ممارسات GQL](https://graphql-rules.com/rules/mutation-payload).

</Note>

## امتداد طفرة الإدخال (The input mutation extension)

من المفيد عادةً استخدام نمط تعريف طفرة تستقبل وسيطًا واحدًا من [نوع الإدخال (input type)](../types/input-types) يسمى `input`.

توفر Strawberry امتداد [`InputMutationExtension`](../extensions/input-mutation.md)، وهو [امتداد حقل](../guides/field-extensions.md) ينشئ تلقائيًا نوع إدخال لك، تكون سماته هي نفسها وسائط المحلل (resolver).

على سبيل المثال، لنفترض أننا نريد أن تكون الطفرة المعرفة في القسم أعلاه طفرة إدخال. يمكننا إضافة `InputMutationExtension` إلى الحقل بهذا الشكل:

```python
from strawberry.field_extensions import InputMutationExtension


@strawberry.type
class Mutation:
    @strawberry.mutation(extensions=[InputMutationExtension()])
    def update_fruit_weight(
        self,
        info: strawberry.Info,
        id: strawberry.ID,
        weight: Annotated[
            float,
            strawberry.argument(description="The fruit\"s new weight in grams"),
        ],
    ) -> Fruit:
        fruit = ...  # retrieve the fruit with the given ID
        fruit.weight = weight
        ...  # maybe save the fruit in the database
        return fruit
```

سيؤدي ذلك إلى إنشاء مخطط مثل هذا:

```graphql
input UpdateFruitWeightInput {
  id: ID!

  """
  The fruit\"s new weight in grams
  """
  weight: Float!
}

type Mutation {
  updateFruitWeight(input: UpdateFruitWeightInput!): Fruit!
}
```

## الطفرات المتداخلة (Nested mutations)

لتجنب أن يصبح الرسم البياني كبيرًا جدًا ولتحسين إمكانية الاكتشاف، قد يكون من المفيد تجميع الطفرات في مساحة اسم (namespace)، كما هو موضح في [دليل Apollo حول تحديد مساحات الأسماء حسب فصل الاهتمامات](https://www.apollographql.com/docs/technotes/TN0012-namespacing-by-separation-of-concern/).

```graphql
type Mutation {
  fruit: FruitMutations!
}

type FruitMutations {
  add(input: AddFruitInput): Fruit!
  updateWeight(input: UpdateFruitWeightInput!): Fruit!
}
```

نظرًا لأن جميع عمليات GraphQL هي حقول، يمكننا تعريف نوع `FruitMutation` وإضافة حقول طفرة إليه كما يمكننا إضافة حقول طفرة إلى نوع `Mutation` الجذري.

```python
import strawberry


@strawberry.type
class FruitMutations:
    @strawberry.mutation
    def add(self, info, input: AddFruitInput) -> Fruit: ...

    @strawberry.mutation
    def update_weight(self, info, input: UpdateFruitWeightInput) -> Fruit: ...


@strawberry.type
class Mutation:
    @strawberry.field
    def fruit(self) -> FruitMutations:
        return FruitMutations()
```

<Note>

يتم حل الحقول في نوع `Mutation` الجذري بشكل تسلسلي. تقدم أنواع مساحات الأسماء إمكانية حل الطفرات بشكل غير متزامن ومتوازٍ لأن حقول الطفرة التي تعدل البيانات لم تعد على المستوى الجذري.

لضمان التنفيذ التسلسلي عند استخدام أنواع مساحات الأسماء، يجب على العملاء استخدام `aliases` لتحديد حقل الطفرة الجذري لكل طفرة. في المثال التالي، بمجرد اكتمال تنفيذ `addFruit`، يبدأ `updateFruitWeight`.

```graphql
mutation (
  $addFruitInput: AddFruitInput!
  $updateFruitWeightInput: UpdateFruitWeightInput!
) {
  addFruit: fruit {
    add(input: $addFruitInput) {
      id
    }
  }

  updateFruitWeight: fruit {
    updateWeight(input: $updateFruitWeightInput) {
      id
    }
  }
}
```

لمزيد من التفاصيل، راجع [دليل Apollo حول مساحات الأسماء للطفرات التسلسلية](https://www.apollographql.com/docs/technotes/TN0012-namespacing-by-separation-of-concern/#namespaces-for-serial-mutations)
و [دليل Rapid API التفاعلي لاستعلامات GraphQL: Aliases and Variables](https://rapidapi.com/guides/graphql-aliases-variables).

</Note>
