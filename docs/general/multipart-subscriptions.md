---
title: الاشتراكات متعددة الأجزاء
---

# الاشتراكات متعددة الأجزاء

تدعم Strawberry الاشتراكات (subscriptions) عبر استجابات متعددة الأجزاء (multipart responses). هذا
[بروتوكول بديل](https://www.apollographql.com/docs/router/executing-operations/subscription-multipart-protocol/)
تم إنشاؤه بواسطة [Apollo](https://www.apollographql.com/) لدعم الاشتراكات
عبر HTTP، وهو مدعوم افتراضيًا بواسطة Apollo Client..

# يدعم

ندعم الاشتراكات متعددة الأجزاء بشكل افتراضي في مكتبات HTTP التالية:

- Django (فقط في العرض غير المتزامن)
- ASGI
- Litestar
- FastAPI
- AioHTTP
- Quart

# الاستخدام

يتم تفعيل الاشتراكات متعددة الأجزاء تلقائيًا عند استخدام الاشتراك، لذلك لا يلزم أي تكوين إضافي.
