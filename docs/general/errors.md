---
title: الأخطاء
---

# الأخطاء في Strawberry

تحتوي Strawberry على أخطاء مدمجة للأوقات التي يحدث فيها خطأ أثناء إنشاء
واستخدام المخطط (schema).

كما توفر معالج استثناءات مخصص (custom exception handler) لتحسين طريقة طباعة الأخطاء
ولتسهيل العثور على مصدر الاستثناء، على سبيل المثال الكود التالي:

```python
import strawberry


@strawberry.type
class Query:
    @strawberry.field
    def hello_world(self):
        return "Hello there!"


schema = strawberry.Schema(query=Query)
```

سيظهر الاستثناء التالي في سطر الأوامر:

```text

  error: Missing annotation for field `hello_world`

       @ demo.py:7

     6 |     @strawberry.field
  ❱  7 |     def hello_world(self):
                 ^^^^^^^^^^^ resolver missing annotation
     8 |         return "Hello there!"


  To fix this error you can add an annotation, like so `def hello_world(...) -> str:`

  Read more about this error on https://errors.strawberry.rocks/missing-return-annotation

```

هذه الأخطاء ممكنة فقط عند تثبيت rich و libcst. يمكنك
تثبيت Strawberry مع تفعيل الأخطاء عن طريق تشغيل:

```shell
pip install "strawberry-graphql[cli]"
```

إذا أردت تعطيل الأخطاء، يمكنك ذلك عن طريق تعيين
متغير البيئة STRAWBERRY_DISABLE_RICH_ERRORS إلى 1.

```
