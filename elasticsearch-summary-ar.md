# Elasticsearch — ملخص عملي

## ما هو Elasticsearch؟

**Elasticsearch** هو محرك بحث مخصص للبحث السريع والمرن داخل كميات كبيرة من البيانات.

هو ليس بديلًا عن PostgreSQL أو MySQL، بل يعمل بجانب قاعدة البيانات.

- قاعدة البيانات: المصدر الحقيقي للبيانات `Source of Truth`
- Elasticsearch: نسخة مفهرسة مخصصة للبحث

---

## لماذا لا نبحث مباشرة في قاعدة البيانات؟

البحث في قاعدة البيانات مناسب عندما نبحث باستخدام:

- `id`
- قيمة مطابقة تمامًا
- علاقات بين الجداول
- عمليات الحجز والدفع
- البيانات التي يجب أن تكون دقيقة وفورية

مثال:

```sql
SELECT *
FROM hotels
WHERE city = 'Dubai';
```

لكن البحث النصي باستخدام:

```sql
LIKE '%dubai hotel%'
```

قد يصبح مكلفًا مع ملايين السجلات، كما أنه لا يقدم بسهولة:

- تحمل الأخطاء الإملائية
- ترتيب النتائج حسب الصلة
- Autocomplete
- Synonyms
- Highlighting
- Full-text search متقدم

---

## ماذا يقدم Elasticsearch؟

### 1. Full-Text Search

يبحث داخل النصوص مثل:

- اسم الفندق
- الوصف
- المدينة
- الدولة

### 2. Fuzzy Search

يتحمل الأخطاء الإملائية البسيطة.

مثال:

```text
Dubia
```

قد يعيد:

```text
Dubai
```

### 3. Ranking

يرتب النتائج حسب مدى ارتباطها بعبارة البحث.

### 4. Autocomplete

يعرض اقتراحات أثناء الكتابة.

مثال:

```text
Du
```

النتائج:

```text
Dubai
Dublin
Durban
```

### 5. Filters

يدعم الفلاتر بسرعة، مثل:

- السعر
- التقييم
- المرافق
- الموقع
- نوع الفندق

---

## الفرق بين Database وElasticsearch

| الحالة | Database | Elasticsearch |
|---|---:|---:|
| تخزين البيانات الأساسية | ✅ | ❌ |
| Transactions | ✅ | ❌ |
| الحجز والدفع | ✅ | ❌ |
| العلاقات بين الجداول | ✅ | ❌ |
| البحث النصي المتقدم | محدود | ✅ |
| تحمل الأخطاء الإملائية | محدود | ✅ |
| Ranking | يدوي | ✅ |
| Autocomplete | يحتاج تنفيذ إضافي | ✅ |
| البحث في ملايين السجلات | يعتمد على الاستعلام والفهارس | مناسب جدًا |

---

# الشكل العام للمعمارية

```text
PostgreSQL
   │
   │ إنشاء أو تعديل أو حذف فندق
   ▼
Backend Service
   │
   ▼
Elasticsearch Index
```

وعند البحث:

```text
Mobile / Web
    │
    ▼
Search API
    │
    ▼
Elasticsearch
    │
    ▼
Search Results
```

---

# Implementation باستخدام Node.js

## 1. تشغيل Elasticsearch باستخدام Docker

```yaml
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.15.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
    ports:
      - "9200:9200"
    volumes:
      - elastic_data:/usr/share/elasticsearch/data

volumes:
  elastic_data:
```

تشغيله:

```bash
docker compose up -d
```

يعمل عادة على:

```text
http://localhost:9200
```

---

## 2. تثبيت Elasticsearch Client

```bash
npm install @elastic/elasticsearch
```

إنشاء Client:

```ts
import { Client } from '@elastic/elasticsearch';

export const elasticClient = new Client({
  node: 'http://localhost:9200',
});
```

---

## 3. إنشاء Index للفنادق

يمكن اعتبار الـ `Index` قريبًا من مفهوم الجدول، لكنه ليس Table حرفيًا.

```ts
await elasticClient.indices.create({
  index: 'hotels',
  mappings: {
    properties: {
      id: {
        type: 'integer',
      },
      name: {
        type: 'text',
        fields: {
          keyword: {
            type: 'keyword',
          },
        },
      },
      city: {
        type: 'text',
        fields: {
          keyword: {
            type: 'keyword',
          },
        },
      },
      country: {
        type: 'keyword',
      },
      description: {
        type: 'text',
      },
      rating: {
        type: 'float',
      },
      price: {
        type: 'float',
      },
      amenities: {
        type: 'keyword',
      },
      location: {
        type: 'geo_point',
      },
    },
  },
});
```

---

## الفرق بين `text` و`keyword`

### text

يستخدم للبحث النصي.

مثال:

```text
Luxury hotel near Dubai Marina
```

### keyword

يستخدم للمطابقة الدقيقة والفلاتر والترتيب.

مثال:

```text
country = UAE
amenities = pool
```

يمكن تعريف الحقل بالصيغة التالية لدعم الحالتين:

```ts
name: {
  type: 'text',
  fields: {
    keyword: {
      type: 'keyword',
    },
  },
}
```

---

# إضافة البيانات إلى Elasticsearch

بعد إنشاء الفندق في PostgreSQL:

```ts
const hotel = await prisma.hotel.create({
  data: {
    name: 'Dubai Marina Hotel',
    city: 'Dubai',
    country: 'UAE',
    description: 'Luxury hotel near Dubai Marina',
    rating: 4.7,
    price: 180,
  },
});
```

نضيف نسخة منه إلى Elasticsearch:

```ts
await elasticClient.index({
  index: 'hotels',
  id: hotel.id.toString(),
  document: {
    id: hotel.id,
    name: hotel.name,
    city: hotel.city,
    country: hotel.country,
    description: hotel.description,
    rating: hotel.rating,
    price: Number(hotel.price),
  },
});
```

يفضل استخدام نفس الـ ID:

```text
PostgreSQL ID = 125
Elasticsearch Document ID = 125
```

---

# تنفيذ Search API

مثال:

```http
GET /api/hotels/search?q=dubai
```

```ts
interface SearchHotelsParams {
  query?: string;
  city?: string;
  minPrice?: number;
  maxPrice?: number;
  minRating?: number;
  amenities?: string[];
}

export async function searchHotels(params: SearchHotelsParams) {
  const filters: Record<string, unknown>[] = [];

  if (params.city) {
    filters.push({
      term: {
        'city.keyword': params.city,
      },
    });
  }

  if (params.minPrice !== undefined || params.maxPrice !== undefined) {
    filters.push({
      range: {
        price: {
          gte: params.minPrice,
          lte: params.maxPrice,
        },
      },
    });
  }

  if (params.minRating !== undefined) {
    filters.push({
      range: {
        rating: {
          gte: params.minRating,
        },
      },
    });
  }

  if (params.amenities?.length) {
    filters.push({
      terms: {
        amenities: params.amenities,
      },
    });
  }

  const result = await elasticClient.search({
    index: 'hotels',
    query: {
      bool: {
        must: params.query
          ? [
              {
                multi_match: {
                  query: params.query,
                  fields: ['name^3', 'city^2', 'description'],
                  fuzziness: 'AUTO',
                },
              },
            ]
          : [
              {
                match_all: {},
              },
            ],
        filter: filters,
      },
    },
    sort: [
      {
        _score: 'desc',
      },
      {
        rating: 'desc',
      },
    ],
  });

  return result.hits.hits.map((hit) => ({
    id: hit._id,
    score: hit._score,
    ...hit._source,
  }));
}
```

---

## معنى Boosting

```ts
fields: ['name^3', 'city^2', 'description']
```

هذا يعني:

- وجود الكلمة في `name` أهم
- وجودها في `city` مهم
- وجودها في `description` أقل أهمية

مثلاً الفندق الذي يحتوي اسمُه على عبارة البحث يظهر قبل فندق يحتوي العبارة فقط داخل الوصف.

---

## معنى Fuzziness

```ts
fuzziness: 'AUTO'
```

تسمح بتحمل الأخطاء الإملائية البسيطة.

مثال:

```text
Dubia
```

قد يجد:

```text
Dubai
```

لا يفضل استخدام Fuzziness على كل الحقول دون دراسة، لأنها قد تزيد تكلفة البحث وتعيد نتائج غير دقيقة.

---

# التحديث والحذف

## التحديث

```ts
await elasticClient.update({
  index: 'hotels',
  id: hotel.id.toString(),
  doc: {
    name: hotel.name,
    price: Number(hotel.price),
    rating: hotel.rating,
  },
});
```

## الحذف

```ts
await elasticClient.delete({
  index: 'hotels',
  id: hotelId.toString(),
});
```

التدفق العام:

```text
Create DB → Index Elasticsearch
Update DB → Update Elasticsearch
Delete DB → Delete Elasticsearch
```

---

# مشكلة المزامنة

في التطبيق البسيط قد نكتب:

```ts
await prisma.hotel.create(...);
await elasticClient.index(...);
```

لكن ماذا يحدث إذا:

```text
نجحت PostgreSQL
وفشل Elasticsearch
```

ستصبح البيانات موجودة في قاعدة البيانات، لكنها غير قابلة للبحث.

---

# الحل الأفضل: Outbox Pattern

داخل نفس Transaction:

```text
1. إنشاء الفندق
2. إنشاء Outbox Event
3. Commit
```

ثم Worker يعالج الحدث:

```text
PostgreSQL
   │
   ├── Hotel
   └── Outbox Event
          │
          ▼
     Background Worker
          │
          ▼
     Elasticsearch
```

مثال Event:

```json
{
  "eventType": "HOTEL_CREATED",
  "aggregateId": 125,
  "payload": {
    "hotelId": 125
  }
}
```

يمكن استخدام:

- BullMQ
- RabbitMQ
- Kafka
- AWS SQS
- Cron Worker بسيط

مثال Worker:

```ts
const hotel = await prisma.hotel.findUnique({
  where: {
    id: event.aggregateId,
  },
});

if (!hotel) {
  return;
}

await elasticClient.index({
  index: 'hotels',
  id: hotel.id.toString(),
  document: {
    id: hotel.id,
    name: hotel.name,
    city: hotel.city,
    country: hotel.country,
    price: Number(hotel.price),
    rating: hotel.rating,
  },
});
```

يفضل أن تكون عملية الـ Worker قابلة لإعادة المحاولة وIdempotent.

---

# إعادة بناء الـ Index

قد تحتاج إلى إعادة إنشاء الـ Index عند:

- تغيير الـ Mapping
- إضافة Analyzer جديد
- حدوث مشكلة في المزامنة
- إنشاء Elasticsearch جديد
- فقدان بعض البيانات

نقرأ البيانات من PostgreSQL ونرسلها على دفعات:

```ts
const hotels = await prisma.hotel.findMany({
  take: 1000,
});

const operations = hotels.flatMap((hotel) => [
  {
    index: {
      _index: 'hotels',
      _id: hotel.id.toString(),
    },
  },
  {
    id: hotel.id,
    name: hotel.name,
    city: hotel.city,
    country: hotel.country,
    price: Number(hotel.price),
    rating: hotel.rating,
  },
]);

await elasticClient.bulk({
  operations,
});
```

هذه العملية تسمى:

```text
Reindexing
```

لا يجب تحميل ملايين السجلات مرة واحدة، بل استخدام Pagination أو Cursor ومعالجة البيانات على دفعات.

---

# استخدام Elasticsearch في Booking System

يستخدم Elasticsearch عادةً في:

- البحث عن الفنادق
- البحث حسب المدينة والدولة
- البحث داخل الاسم والوصف
- الفلاتر
- Autocomplete
- Suggestions
- ترتيب النتائج
- البحث الجغرافي
- تحمل الأخطاء الإملائية

لكن لا يفضل اعتباره المصدر النهائي لـ:

- توفر الغرف الحالي
- السعر النهائي
- عدد الغرف المتبقية
- إنشاء الحجز
- الدفع
- العمليات المالية
- الـ Transactions

السبب أن Elasticsearch يعمل غالبًا بـ:

```text
Eventual Consistency
```

أي أن البيانات قد تتأخر قليلًا عن PostgreSQL.

---

# التدفق الصحيح في نظام حجز

```text
1. المستخدم يبحث في Elasticsearch
2. Elasticsearch يعيد الفنادق المناسبة
3. المستخدم يختار فندقًا
4. Backend يتحقق من التوفر الحقيقي
5. Backend يحسب السعر النهائي
6. الحجز ينفذ داخل PostgreSQL
```

Elasticsearch مسؤول عن:

```text
Discovery
```

أما قاعدة البيانات وخدمة الحجز فمسؤولتان عن:

```text
Validation
Confirmation
Booking
Payment
```

---

# Architecture مقترحة

```text
                         ┌──────────────────┐
                         │ Mobile / Web App │
                         └─────────┬────────┘
                                   │
                  ┌────────────────┴───────────────┐
                  │                                │
           Create Booking                    Search Hotels
                  │                                │
                  ▼                                ▼
          Booking Service                   Search Service
                  │                                │
                  ▼                                ▼
            PostgreSQL                      Elasticsearch
                  │
                  ▼
             Outbox Table
                  │
                  ▼
             Background Worker
                  │
                  ▼
            Elasticsearch
```

---

# خطة Implementation تدريجية

## المرحلة الأولى

```text
1. تشغيل Elasticsearch باستخدام Docker
2. إنشاء hotels index
3. إدخال بيانات تجريبية
4. إنشاء GET /hotels/search
5. تجربة multi_match
6. تجربة filters
7. تجربة fuzziness
```

## المرحلة الثانية

```text
1. مزامنة Create
2. مزامنة Update
3. مزامنة Delete
4. إضافة Bulk Reindex Script
```

## المرحلة الثالثة

```text
1. إضافة Outbox Pattern
2. إضافة Background Worker
3. إضافة Retry Mechanism
4. إضافة Dead Letter Queue
5. مراقبة فشل المزامنة
```

---

# الخلاصة

- PostgreSQL هو المصدر الحقيقي للبيانات.
- Elasticsearch نسخة مخصصة للبحث.
- البحث يتم من Elasticsearch.
- الحجز والدفع والتحقق النهائي يتم من PostgreSQL.
- في البداية يمكن المزامنة مباشرة.
- في النظام الحقيقي يفضل استخدام Outbox Pattern وBackground Worker.
- يجب توفير Reindex Script لإعادة بناء البيانات عند الحاجة.
