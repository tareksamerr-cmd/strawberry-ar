---
title: البدء مع Strawberry
---

# البدء مع Strawberry

سيساعدك هذا البرنامج التعليمي على:

- الحصول على فهم أساسي لمبادئ GraphQL
- تعريف مخطط GraphQL باستخدام Strawberry
- تشغيل خادم تطوير Strawberry الذي يتيح لك تنفيذ الاستعلامات (queries) مقابل مخططك

يفترض هذا البرنامج التعليمي أنك على دراية بسطر الأوامر (command line) و Python، وأن لديك إصدارًا حديثًا من Python (3.10+) مثبتًا.

تم بناء Strawberry على أساس وظائف Python لـ [dataclasses](https://realpython.com/python-data-classes/) و [type hints](https://docs.python.org/3/library/typing.html).

## الخطوة 1: إنشاء مشروع جديد وتثبيت Strawberry

لنقم بإنشاء مجلد جديد:

```shell
mkdir strawberry-demo
cd strawberry-demo
```

بعد ذلك نحتاج إلى بيئة افتراضية (virtualenv) جديدة:

```shell
python -m venv virtualenv
```

قم بتنشيط الـ virtualenv ثم قم بتثبيت strawberry بالإضافة إلى خادم التطوير.

```shell
source virtualenv/bin/activate
pip install 'strawberry-graphql[cli]'
```

## الخطوة 2: تعريف المخطط (Schema)

يستخدم كل خادم GraphQL **مخططًا (schema)** لتحديد بنية البيانات التي يمكن للعملاء الاستعلام عنها. في هذا المثال، سنقوم بإنشاء خادم للاستعلام عن مجموعة من الكتب حسب العنوان والمؤلف.

في محرر النصوص المفضل لديك، قم بإنشاء ملف يسمى `schema.py`، بالمحتويات التالية:

```python
import typing
import strawberry


@strawberry.type
class Book:
    title: str
    author: str


@strawberry.type
class Query:
    books: typing.List[Book]
```

سيؤدي هذا إلى إنشاء مخطط GraphQL حيث سيتمكن العملاء من تنفيذ استعلام (query) يسمى `books` والذي سيعيد قائمة من صفر أو أكثر من الكتب.

## الخطوة 3: تعريف مجموعة البيانات الخاصة بك

الآن بعد أن أصبح لدينا بنية مخططنا، يمكننا تعريف البيانات نفسها. يمكن لـ Strawberry العمل مع أي مصدر بيانات (على سبيل المثال قاعدة بيانات، واجهة برمجة تطبيقات REST، ملفات، إلخ). لهذا البرنامج التعليمي، سنستخدم بيانات مبرمجة (hard-coded data).

لنقم بإنشاء دالة تعيد بعض الكتب.

```python
def get_books():
    return [
        Book(
            title="The Great Gatsby",
            author="F. Scott Fitzgerald",
        ),
    ]
```

نظرًا لأن strawberry تستخدم فئات Python لإنشاء المخطط، فهذا يعني أنه يمكننا أيضًا إعادة استخدامها لإنشاء كائنات البيانات.

## الخطوة 4: تعريف محلل (Resolver)

لدينا الآن دالة تعيد بعض الكتب، لكن Strawberry لا تعرف أنه يجب عليها استخدامها عند تنفيذ استعلام. لإصلاح ذلك، نحتاج إلى تحديث استعلامنا لتحديد الـ [resolver](/docs/types/resolvers) لكتبنا. يخبر الـ resolver مكتبة Strawberry بكيفية جلب البيانات المرتبطة بحقل معين.

لنقم بتحديث استعلامنا:

```python
@strawberry.type
class Query:
    books: typing.List[Book] = strawberry.field(resolver=get_books)
```

يسمح لنا استخدام `strawberry.field` بتحديد resolver لحقل معين.

<Note>

لم نكن بحاجة إلى تحديد أي resolver (مُحلِّل) لحقول الكتاب، وذلك لأن strawberry يُضيف قيمة افتراضية لكل حقل، ويُعيد قيمة ذلك الحقل..

</Note>

## الخطوة 5: إنشاء مخططنا وتشغيله

لقد قمنا بتعريف بياناتنا واستعلامنا، والآن ما نحتاج إلى القيام به هو إنشاء مخطط GraphQL وبدء تشغيل الخادم.

لإنشاء المخطط، أضف الكود التالي:

```python
schema = strawberry.Schema(query=Query)
```

ثم قم بتشغيل الأمر التالي:

```shell
strawberry dev schema
```

سيؤدي هذا إلى بدء تشغيل خادم تطوير، ويجب أن ترى الإخراج التالي:

```text
Running strawberry on http://0.0.0.0:8000/graphql 🍓
```

## الخطوة 6: تنفيذ استعلامك الأول

يمكننا الآن تنفيذ استعلامات GraphQL. يأتي Strawberry مع أداة تسمى **GraphiQL**. لفتحها، انتقل إلى [http://0.0.0.0:8000/graphql](http://0.0.0.0:8000/graphql)

يجب أن ترى شيئًا كهذا:

![A view of the GraphiQL interface](./images/index-server.png)

تتضمن واجهة مستخدم GraphiQL ما يلي:

- منطقة نصية (على اليسار) لكتابة الاستعلامات
- زر تشغيل (الزر المثلث في المنتصف) لتنفيذ الاستعلامات
- منطقة نصية (على اليمين) لعرض نتائج الاستعلامات. طرق عرض لفحص المخطط والوثائق التي تم إنشاؤها (عبر علامات التبويب على الجانب الأيمن)

يدعم خادمنا استعلامًا واحدًا يسمى books. لنقم بتنفيذه!

الصق السلسلة التالية في المنطقة اليسرى ثم انقر فوق زر التشغيل:

```graphql
{
  books {
    title
    author
  }
}
```

يجب أن تظهر البيانات المبرمجة على الجانب الأيمن:

![A view of the GraphiQL interface after running a GraphQL query](./images/index-query-example.png)

يسمح GraphQL للعملاء بالاستعلام عن الحقول التي يحتاجونها فقط، امض قدمًا وقم بإزالة `author` من الاستعلام وقم بتشغيله مرة أخرى. يجب أن يعرض الرد الآن فقط عنوان كل كتاب.

## الخطوات التالية

<!--alex ignore retext-equality -->

أحسنت! لقد أنشأت للتو أول واجهة برمجة تطبيقات GraphQL باستخدام Strawberry 🙌!

اطلع على الموارد التالية لمعرفة المزيد حول GraphQL و Strawberry.

- [Schema Basics](./general/schema-basics.md)
- [Resolvers](./types/resolvers.md)
- [Deployment](./operations/deployment.md)

