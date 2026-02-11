# Apache Kafka — Noldan Professional Darajagacha

## To'liq Kurs O'zbek Tilida

> **Muallif eslatmasi:** Bu kurs backend dasturchi uchun mo'ljallangan. Siz Go, PostgreSQL, Docker va Linux bilasiz deb faraz qilinadi. Barcha misollar Ubuntu va KRaft rejimida (Zookeeper'siz) berilgan.

---

# PHASE 1 — Kafkani Tushunish (Chuqur Konseptlar)

---

## 1-Dars: Kafka Qanday Muammoni Hal Qiladi?

### Muammo

Tasavvur qiling: sizda ERP tizimi bor. Foydalanuvchi buyurtma beradi. Shundan keyin:

1. Buyurtma ma'lumotlar bazasiga yoziladi
2. Omborga xabar yuboriladi
3. Foydalanuvchiga email jo'natiladi
4. SMS xabar yuboriladi
5. Analitika tizimiga ma'lumot uzatiladi

Oddiy yondashuv — **sinxron (synchronous)**:

```
Buyurtma API → DB ga yozish → Ombor API → Email API → SMS API → Analitika API
                                    ↓
                            Agar biri ishlamasa?
                            HAMMASI TO'XTAYDI!
```

**Muammolar:**

| Muammo | Tushuntirish |
|--------|-------------|
| **Tight coupling** | Har bir servis bir-biriga bog'liq. Email servisi ishlamasa, buyurtma ham ishlamaydi |
| **Sekinlik** | Har bir qadamni ketma-ket kutish kerak. 5 ta servis x 200ms = 1 soniya |
| **Nosozlikka chidamsizlik** | Bitta servis tushsa, butun zanjir buziladi |
| **Masshtablash qiyin** | Yukni taqsimlash murakkab |

### Yechim — Event-Driven Architecture (Voqeaga Asoslangan Arxitektura)

```
Buyurtma API → Kafka → [Ombor servisi]
                     → [Email servisi]
                     → [SMS servisi]
                     → [Analitika servisi]
```

Buyurtma API faqat bitta ish qiladi: **Kafkaga voqea (event) yuboradi**. Keyin har bir servis o'zi mustaqil ravishda bu voqeani o'qiydi va ishlaydi.

**Afzalliklari:**

| Afzallik | Tushuntirish |
|----------|-------------|
| **Loose coupling** | Servislar bir-birini bilmaydi. Faqat Kafkadagi voqeani biladi |
| **Tezlik** | Buyurtma API faqat Kafkaga yozadi (1-5ms) va javob qaytaradi |
| **Nosozlikka chidamlilik** | Email servisi tushsa ham, buyurtma ishlaydi. Email servisi qayta ishga tushganda voqealarni o'qib oladi |
| **Masshtablash oson** | Har bir servisni alohida ko'paytirish mumkin |

### Kafka Nima?

**Apache Kafka** — bu taqsimlangan (distributed) voqealar oqimi platformasi (event streaming platform).

Sodda qilib aytganda: **Kafka — bu juda tez, ishonchli, kengaytiriladigan xabar almashish tizimi.**

Kafka quyidagilarni qiladi:

1. **Xabarlarni qabul qiladi** (produce)
2. **Xabarlarni saqlaydi** (store) — diskda, ishonchli
3. **Xabarlarni uzatadi** (consume) — bir nechta iste'molchiga

### Kafka va Oddiy Message Queue (RabbitMQ) Farqi

```
RabbitMQ:
  Producer → Queue → Consumer (o'qilgach o'chadi)

Kafka:
  Producer → Topic → Consumer A (o'qiydi, xabar SAQLANIB QOLADI)
                   → Consumer B (o'sha xabarni yana o'qiydi)
                   → Consumer C (keyinroq kelib o'qiydi)
```

| Xususiyat | RabbitMQ | Kafka |
|-----------|----------|-------|
| Xabar saqlanishi | O'qilgach o'chadi | Muddatgacha saqlanadi |
| Tezlik | O'rtacha | Juda yuqori (millionlab/soniya) |
| Replay | Yo'q | Bor (qayta o'qish mumkin) |
| Tartib | Kafolatlanmaydi | Partition ichida kafolatlanadi |
| Maqsad | Task queue | Event streaming |

### Real Hayotdan Misol

**E-commerce ERP tizimi:**

```
┌─────────────┐
│  Buyurtma   │──→ "order.created" voqeasi
│  Servisi    │
└─────────────┘
        │
        ▼
┌─────────────────────────────────────────┐
│              KAFKA                       │
│  Topic: "orders"                        │
│  ┌──────┬──────┬──────┬──────┐         │
│  │ msg1 │ msg2 │ msg3 │ msg4 │         │
│  └──────┴──────┴──────┴──────┘         │
└─────────────────────────────────────────┘
        │           │           │
        ▼           ▼           ▼
┌──────────┐ ┌──────────┐ ┌──────────┐
│  Ombor   │ │  Email   │ │ Analitika│
│  Servisi │ │  Servisi │ │  Servisi │
└──────────┘ └──────────┘ └──────────┘
```

Har bir servis **mustaqil** ishlaydi. Ombor servisi tushib qolsa, Email va Analitika servislari davom etadi. Ombor servisi qayta ishga tushganda, to'xtalgan joyidan davom etadi.

### Keng Tarqalgan Xatolar

1. **Kafkani oddiy queue deb o'ylash** — Kafka queue emas, event log. Xabarlar o'qilgandan keyin o'chmaydi.
2. **Hamma narsa uchun Kafka ishlatish** — Oddiy request-response uchun HTTP yaxshiroq. Kafka — bu asinxron voqealar uchun.
3. **Kafkasiz boshlanish** — Avval muammoni tushunib, keyin Kafka kerakligini hal qiling.

---

## 2-Dars: Kafka Asosiy Tushunchalari

### Producer (Ishlab Chiqaruvchi)

**Producer** — Kafkaga xabar yuboruvchi dastur.

```
┌──────────┐     xabar     ┌───────────┐
│ Producer │ ─────────────→ │   Kafka   │
│ (Go app) │               │  (Broker) │
└──────────┘               └───────────┘
```

Misol: Buyurtma servisi yangi buyurtma haqida Kafkaga xabar yuboradi.

```json
{
  "event": "order.created",
  "order_id": 12345,
  "user_id": 67,
  "amount": 150000,
  "currency": "UZS",
  "timestamp": "2025-01-15T10:30:00Z"
}
```

### Consumer (Iste'molchi)

**Consumer** — Kafkadan xabar o'qiydigan dastur.

```
┌───────────┐     xabar     ┌──────────┐
│   Kafka   │ ─────────────→ │ Consumer │
│  (Broker) │               │ (Go app) │
└───────────┘               └──────────┘
```

Misol: Email servisi "order.created" voqeasini o'qiydi va foydalanuvchiga email yuboradi.

### Topic (Mavzu)

**Topic** — xabarlarning mantiqiy kategoriyasi. Bu papkaga o'xshaydi.

```
Kafka Broker
├── Topic: "orders"          ← Buyurtma voqealari
├── Topic: "payments"        ← To'lov voqealari
├── Topic: "user-events"     ← Foydalanuvchi voqealari
└── Topic: "notifications"   ← Bildirishnomalar
```

**Qoidalar:**
- Har bir topic nomi unikal bo'lishi kerak
- Topic nomida nuqta (.) yoki pastki chiziq (_) ishlatish mumkin
- Maslahat: `<domen>.<voqea>` formati — `order.created`, `payment.completed`

### Broker (Vositachi)

**Broker** — Kafka serveri. Xabarlarni qabul qiladi, saqlaydi va uzatadi.

```
┌────────────────────────────────────────┐
│            Kafka Cluster               │
│                                        │
│  ┌──────────┐ ┌──────────┐ ┌────────┐│
│  │ Broker 1 │ │ Broker 2 │ │Broker 3││
│  │ (server) │ │ (server) │ │(server)││
│  └──────────┘ └──────────┘ └────────┘│
│                                        │
└────────────────────────────────────────┘
```

**Production muhitda** odatda 3 yoki undan ko'p broker ishlatiladi. Bitta broker tushsa, boshqalari ishlashda davom etadi.

O'rganish uchun bitta broker yetarli.

### Partition (Bo'lim)

**Partition** — bu topic ichidagi parallel bo'lim. Kafka tezligining asosiy sababi.

```
Topic: "orders" (3 ta partition)

Partition 0: [msg1] [msg4] [msg7] [msg10]
Partition 1: [msg2] [msg5] [msg8] [msg11]
Partition 2: [msg3] [msg6] [msg9] [msg12]
```

**Nega partition kerak?**

Bitta partition bilan:
```
1 ta consumer → 1000 xabar/soniya
```

3 ta partition bilan:
```
Consumer 1 → Partition 0 → 1000 xabar/soniya
Consumer 2 → Partition 1 → 1000 xabar/soniya
Consumer 3 → Partition 2 → 1000 xabar/soniya
─────────────────────────────────────────────
Jami: 3000 xabar/soniya
```

**Muhim qoidalar:**
1. Partition ichida xabarlar **tartiblangan** (ordered)
2. Partitionlar orasida tartib **kafolatlanmaydi**
3. Xabar qaysi partitionga tushishini **key** belgilaydi

### Offset

**Offset** — partition ichidagi xabarning tartib raqami. 0 dan boshlanadi.

```
Partition 0:
  Offset: 0    1    2    3    4    5
         [msg] [msg] [msg] [msg] [msg] [msg]
                            ↑
                     Consumer hozir shu yerda
```

**Offset xususiyatlari:**
- Har bir partition uchun alohida
- Faqat oldinga siljiydi (ortga qaytib o'chirib bo'lmaydi)
- Consumer o'zi qayerda turganini offset orqali biladi
- Consumer istalgan offsetdan boshlab o'qishi mumkin (**replay**)

### Hammasini Birga Ko'ramiz

```
                    Kafka Cluster
                    ┌─────────────────────────────────┐
                    │          Broker 1                │
┌──────────┐       │                                  │
│Producer 1│──────→│  Topic: "orders"                 │
│(Buyurtma)│       │  ┌─────────────────────────────┐│       ┌──────────┐
└──────────┘       │  │ Partition 0                  ││──────→│Consumer 1│
                    │  │ [0][1][2][3][4]             ││       │(Ombor)   │
┌──────────┐       │  └─────────────────────────────┘│       └──────────┘
│Producer 2│──────→│  ┌─────────────────────────────┐│
│(To'lov)  │       │  │ Partition 1                  ││──────→┌──────────┐
└──────────┘       │  │ [0][1][2][3]                ││       │Consumer 2│
                    │  └─────────────────────────────┘│       │(Email)   │
                    │  ┌─────────────────────────────┐│       └──────────┘
                    │  │ Partition 2                  ││
                    │  │ [0][1][2]                   ││──────→┌──────────┐
                    │  └─────────────────────────────┘│       │Consumer 3│
                    │                                  │       │(Analitik)│
                    └─────────────────────────────────┘       └──────────┘
```

### Keng Tarqalgan Xatolar

1. **Juda ko'p partition yaratish** — Har bir partition resurs talab qiladi. Boshlang'ich uchun 3-6 ta yetarli.
2. **Offset tushunchasini e'tiborsiz qoldirish** — Offset Kafkaning eng muhim tushunchalaridan biri. Consumer boshqaruvi offset orqali ishlaydi.
3. **Partitionlar orasida tartib kutish** — Faqat bitta partition ichida tartib kafolatlanadi.

---

## 3-Dars: Partitionlar Kafkani Nima Uchun Tez Qiladi?

### Disk Yozish Strategiyasi

Ko'pchilik o'ylaydi: "Diskka yozish sekin". Lekin Kafka diskka **ketma-ket (sequential)** yozadi, bu **tasodifiy (random)** yozishdan 100-1000 marta tez.

```
Tasodifiy yozish (Random I/O):
  Disk boshi: ←→←→←→←→  (har joyga sakraydi)
  Tezlik: ~100 MB/s

Ketma-ket yozish (Sequential I/O):
  Disk boshi: →→→→→→→→  (bir yo'nalishda)
  Tezlik: ~600 MB/s (SSD da 2-3 GB/s)
```

### Page Cache

Kafka OS ning **page cache** dan foydalanadi. Bu degani:

```
Producer → Kafka → OS Page Cache (RAM) → Disk
                        ↓
Consumer ← Kafka ← OS Page Cache (RAM)  ← [Diskdan o'qish shart emas!]
```

Ko'p hollarda consumer xabarni **diskdan emas, RAMdan** o'qiydi, chunki xabar hali page cache da turgan bo'ladi.

### Zero-Copy Transfer

Oddiy dasturda ma'lumot uzatish:
```
Disk → OS Buffer → Application Buffer → Socket Buffer → Network
        (1-copy)      (2-copy)           (3-copy)
```

Kafka **zero-copy** ishlatadi:
```
Disk → OS Buffer → Network
        (1-copy, dastur buferi orqali o'tmaydi)
```

Bu **CPU yukini** va **xotira ishlatishni** keskin kamaytiradi.

### Batch (To'plam) Yuborish

Kafka xabarlarni birma-bir emas, **to'plab** yuboradi:

```
Oddiy yondashuv:
  msg1 → network → Kafka
  msg2 → network → Kafka
  msg3 → network → Kafka
  (3 ta network call)

Kafka yondashuvi:
  [msg1, msg2, msg3] → network → Kafka
  (1 ta network call)
```

### Compression (Siqish)

Kafka xabarlarni siqib yuborishi mumkin:

```
Siqishsiz: 1000 xabar = 1 MB
Siqish bilan (snappy/lz4/zstd): 1000 xabar = 200-400 KB
```

### Partitionlar = Parallellik

```
1 Partition = 1 Consumer = Cheklangan tezlik

                    ┌── Partition 0 ── Consumer 0
                    │
Topic: "orders" ────┼── Partition 1 ── Consumer 1
                    │
                    ├── Partition 2 ── Consumer 2
                    │
                    └── Partition 3 ── Consumer 3

Natija: 4x tezlik!
```

### Raqamlarda

| Metrika | Qiymat |
|---------|--------|
| Yozish tezligi | 1 broker da ~200 MB/s |
| O'qish tezligi | 1 broker da ~300+ MB/s |
| Xabar kechikishi (latency) | 2-5 ms |
| Xabar hajmi | To'plam bilan millionlab/soniya |
| Saqlash | Terabaytlab ma'lumot |

---

## 4-Dars: Kafka Ma'lumotlarni Diskda Qanday Saqlaydi?

### Katalog Tuzilishi

Kafka har bir partition uchun alohida papka yaratadi:

```
/tmp/kraft-combined-logs/    (yoki sozlangan log.dirs)
├── orders-0/                ← Topic "orders", Partition 0
│   ├── 00000000000000000000.log      ← Xabarlar (segment fayl)
│   ├── 00000000000000000000.index    ← Offset indeksi
│   └── 00000000000000000000.timeindex ← Vaqt indeksi
├── orders-1/                ← Topic "orders", Partition 1
│   ├── 00000000000000000000.log
│   ├── 00000000000000000000.index
│   └── 00000000000000000000.timeindex
└── orders-2/                ← Topic "orders", Partition 2
    ├── 00000000000000000000.log
    ├── 00000000000000000000.index
    └── 00000000000000000000.timeindex
```

### Segment Fayllar

Kafka xabarlarni bitta ulkan faylga emas, **segmentlarga** bo'lib yozadi:

```
Partition 0 papkasi:
  00000000000000000000.log   ← 0-dan 999-gacha offset (1 GB to'lgach)
  00000000000000001000.log   ← 1000-dan 1999-gacha offset
  00000000000000002000.log   ← 2000-dan ... gacha offset (hozirgi aktiv)
```

**Segment foydalari:**
- Eski ma'lumotlarni o'chirish oson (butun segment faylni o'chirish)
- Disk bo'shashini boshqarish oson
- Indekslash samarali

### .log Fayl Formati

Har bir xabar quyidagi formatda saqlanadi:

```
┌─────────────────────────────────────────────┐
│ Offset: 0                                   │
│ Timestamp: 1705312200000                    │
│ Key: "user-67"                              │
│ Value: {"event":"order.created","id":123}   │
│ Headers: [{"source":"order-service"}]       │
├─────────────────────────────────────────────┤
│ Offset: 1                                   │
│ Timestamp: 1705312201000                    │
│ Key: "user-42"                              │
│ Value: {"event":"order.created","id":124}   │
│ Headers: []                                 │
├─────────────────────────────────────────────┤
│ ...                                         │
└─────────────────────────────────────────────┘
```

### .index Fayl

Index fayl offsetni fayldagi fizik pozitsiyaga (byte offset) bog'laydi:

```
.index fayl:
  Offset 0   → Fayl pozitsiyasi: 0
  Offset 100 → Fayl pozitsiyasi: 15360
  Offset 200 → Fayl pozitsiyasi: 30720
  ...
```

**Qidiruv jarayoni (offset 150 ni topish):**
1. Index dan 100 → 15360 topiladi (eng yaqin past qiymat)
2. 15360 pozitsiyadan boshlab 150-gacha ketma-ket o'qiladi

Bu **O(1)** murakkablikda ishlaydi (deyarli bir zumda).

### .timeindex Fayl

Vaqt bo'yicha qidirish uchun:

```
.timeindex fayl:
  Timestamp 1705312200000 → Offset 0
  Timestamp 1705312260000 → Offset 100
  Timestamp 1705312320000 → Offset 200
```

Bu "menga soat 10:30 dan keyingi xabarlarni ber" degan so'rovlarni tez bajarishga yordam beradi.

### Retention (Saqlash Muddati)

Kafka xabarlarni **abadiy** saqlamaydi. Ikki xil saqlash siyosati bor:

**1. Vaqt bo'yicha (time-based):**
```properties
# Standart: 7 kun
log.retention.hours=168

# 30 kun saqlash
log.retention.hours=720

# 1 yil saqlash
log.retention.hours=8760
```

**2. Hajm bo'yicha (size-based):**
```properties
# Standart: cheksiz (-1)
log.retention.bytes=-1

# Har bir partition uchun 1 GB
log.retention.bytes=1073741824
```

**3. Compact (Ixchamlash):**
```
Har bir key uchun faqat oxirgi qiymat saqlanadi:

Oldin:
  key=user1, value={"name":"Ali"}     ← o'chiriladi
  key=user2, value={"name":"Vali"}    ← o'chiriladi
  key=user1, value={"name":"Alisher"} ← saqlanadi (oxirgisi)
  key=user2, value={"name":"Valisher"}← saqlanadi (oxirgisi)
```

---

## 5-Dars: Event-Driven Architecture — Real ERP Misoli

### Klassik Monolitik Yondashuv

```
┌────────────────────────────────────────────────────┐
│                    ERP MONOLITH                     │
│                                                    │
│  Buyurtma → DB yozish → Ombor yangilash →          │
│  → To'lov → Hisob-faktura → Email → SMS            │
│                                                    │
│  Hammasi BITTA dasturda, BITTA bazada              │
└────────────────────────────────────────────────────┘

Muammolar:
- Bitta qism buzilsa, hammasi to'xtaydi
- Deploy qilish qo'rqinchli (butun tizimni qayta ishga tushirish)
- Turli jamoa bitta kodda ishlaydi
- Masshtablash mumkin emas
```

### Event-Driven Microservices Yondashuvi

```
┌──────────────┐     order.created     ┌─────────────┐
│   Buyurtma   │──────────────────────→│             │
│   Servisi    │                       │             │
└──────────────┘                       │             │
                                       │             │
┌──────────────┐     payment.completed │             │     ┌──────────────┐
│    To'lov    │──────────────────────→│    KAFKA    │────→│    Ombor     │
│   Servisi    │                       │             │     │   Servisi    │
└──────────────┘                       │             │     └──────────────┘
                                       │             │
┌──────────────┐                       │             │     ┌──────────────┐
│   Hisobot    │←──────────────────────│             │────→│    Email     │
│   Servisi    │                       │             │     │   Servisi    │
└──────────────┘                       │             │     └──────────────┘
                                       │             │
                                       │             │     ┌──────────────┐
                                       │             │────→│  Analitika   │
                                       │             │     │   Servisi    │
                                       └─────────────┘     └──────────────┘
```

### Real ERP Voqealar Oqimi

**1. Buyurtma yaratildi:**
```json
Topic: "orders"
{
  "event_type": "order.created",
  "order_id": "ORD-2025-001",
  "user_id": 67,
  "items": [
    {"product_id": "P100", "name": "Noutbuk", "qty": 1, "price": 8500000},
    {"product_id": "P205", "name": "Sichqoncha", "qty": 2, "price": 150000}
  ],
  "total": 8800000,
  "currency": "UZS",
  "timestamp": "2025-01-15T10:30:00Z"
}
```

**2. Ombor servisi bu voqeani o'qiydi va mahsulotni zaxiradan chiqaradi:**
```json
Topic: "inventory"
{
  "event_type": "stock.reserved",
  "order_id": "ORD-2025-001",
  "items": [
    {"product_id": "P100", "reserved": 1, "remaining": 45},
    {"product_id": "P205", "reserved": 2, "remaining": 230}
  ],
  "timestamp": "2025-01-15T10:30:01Z"
}
```

**3. To'lov servisi to'lovni amalga oshiradi:**
```json
Topic: "payments"
{
  "event_type": "payment.completed",
  "order_id": "ORD-2025-001",
  "amount": 8800000,
  "method": "payme",
  "transaction_id": "TXN-98765",
  "timestamp": "2025-01-15T10:30:05Z"
}
```

**4. Email servisi tasdiqlash xati yuboradi:**
```json
Topic: "notifications"
{
  "event_type": "email.sent",
  "order_id": "ORD-2025-001",
  "to": "user@example.com",
  "template": "order_confirmation",
  "timestamp": "2025-01-15T10:30:06Z"
}
```

### Voqealar Zanjiri Diagrammasi

```
Vaqt →

order.created ─────┬──→ stock.reserved ──→ stock.updated
                   │
                   ├──→ payment.initiated ──→ payment.completed
                   │                              │
                   │                              ├──→ invoice.generated
                   │                              │
                   │                              └──→ email.sent (receipt)
                   │
                   ├──→ email.sent (confirmation)
                   │
                   └──→ analytics.recorded
```

### Keng Tarqalgan Xatolar

1. **Juda ko'p topic yaratish** — Har bir kichik voqea uchun alohida topic kerak emas. `orders` topicda `order.created`, `order.updated`, `order.cancelled` barchasi bo'lishi mumkin.
2. **Voqeani juda katta qilish** — Voqeada faqat kerakli ma'lumot bo'lsin. To'liq ob'ektni yuborish shart emas.
3. **Sinxron mantiqni asinxron qilish** — Foydalanuvchi javob kutayotgan bo'lsa (masalan, login), Kafka ishlatmang. HTTP yaxshiroq.

---

## 6-Dars: Consumer Group va Masshtablash

### Consumer Group Nima?

**Consumer Group** — bu bitta mantiqiy iste'molchini tashkil qiluvchi bir nechta consumer instansiyasi.

```
Oddiy consumer (groupsiz):
  Har bir consumer BARCHA xabarlarni o'qiydi

  Consumer A → [msg1, msg2, msg3, msg4, msg5]
  Consumer B → [msg1, msg2, msg3, msg4, msg5]  ← dublikat!

Consumer Group:
  Group ichida xabarlar TAQSIMLANADI

  Group: "order-processor"
  Consumer A → [msg1, msg3, msg5]     ← Partition 0
  Consumer B → [msg2, msg4]           ← Partition 1
```

### Consumer Group Ishlash Prinsipi

```
Topic: "orders" (3 partition)

Consumer Group: "inventory-service"
┌────────────────────────────────────────────────┐
│                                                │
│  Partition 0 ───→ Consumer Instance 1          │
│  Partition 1 ───→ Consumer Instance 2          │
│  Partition 2 ───→ Consumer Instance 3          │
│                                                │
└────────────────────────────────────────────────┘

Consumer Group: "email-service"
┌────────────────────────────────────────────────┐
│                                                │
│  Partition 0 ─┐                                │
│  Partition 1 ─┼──→ Consumer Instance 1         │
│  Partition 2 ─┘    (bitta instance hammani     │
│                     o'qiydi)                   │
└────────────────────────────────────────────────┘
```

**Muhim qoidalar:**

1. **Bitta partition faqat bitta consumer ga tegishli** (group ichida)
2. **Bitta consumer bir nechta partition o'qishi mumkin**
3. **Consumerlar soni > Partitionlar soni** bo'lsa, ortiqcha consumer bo'sh turadi

### Masshtablash Misollari

**Holat 1: 3 partition, 1 consumer**
```
  P0 ─┐
  P1 ─┼──→ Consumer 1  (hammani o'zi o'qiydi, sekin)
  P2 ─┘
```

**Holat 2: 3 partition, 2 consumer**
```
  P0 ────→ Consumer 1  (2 ta partition)
  P1 ────→ Consumer 1
  P2 ────→ Consumer 2  (1 ta partition)
```

**Holat 3: 3 partition, 3 consumer (IDEAL)**
```
  P0 ────→ Consumer 1
  P1 ────→ Consumer 2
  P2 ────→ Consumer 3
```

**Holat 4: 3 partition, 4 consumer**
```
  P0 ────→ Consumer 1
  P1 ────→ Consumer 2
  P2 ────→ Consumer 3
  (hech narsa) Consumer 4  ← BO'SH TURADI!
```

### Ko'p Consumer Group

Turli servislar bitta topicdan mustaqil o'qishi mumkin:

```
Topic: "orders" (3 partition)

Group: "inventory-service"   → P0→C1, P1→C2, P2→C3
Group: "email-service"       → P0→C1, P1→C1, P2→C1
Group: "analytics-service"   → P0→C1, P1→C2, P2→C2

Har bir group o'z offsetini alohida saqlaydi!
```

### Offset Boshqaruvi

```
Topic: "orders", Partition 0

Xabarlar:  [0] [1] [2] [3] [4] [5] [6] [7] [8] [9]

Group "inventory": offset = 7  (7 tagacha o'qigan)
Group "email":     offset = 9  (hammasini o'qigan)
Group "analytics": offset = 3  (sekin ishlayapti)
```

Har bir group o'z tezligida ishlaydi. Biri sekin bo'lsa, boshqalariga ta'sir qilmaydi.

### Rebalancing (Qayta Taqsimlash)

Consumer qo'shilganda yoki tushganda, Kafka partitionlarni **qayta taqsimlaydi**:

```
Oldin (2 consumer):
  P0, P1 → Consumer 1
  P2     → Consumer 2

Consumer 3 qo'shildi:
  REBALANCING...

Keyin (3 consumer):
  P0 → Consumer 1
  P1 → Consumer 2
  P2 → Consumer 3
```

```
Oldin (3 consumer):
  P0 → Consumer 1
  P1 → Consumer 2
  P2 → Consumer 3

Consumer 2 tushdi:
  REBALANCING...

Keyin (2 consumer):
  P0     → Consumer 1
  P1, P2 → Consumer 3
```

### Keng Tarqalgan Xatolar

1. **Consumerlar sonini partitionlar sonidan ko'p qilish** — Ortiqcha consumerlar bo'sh turadi, resurs isrof.
2. **Group ID ni noto'g'ri qo'yish** — Turli servislar bir xil group ID ishlatsa, xabarlar taqsimlanib ketadi (har biri faqat bir qismini oladi).
3. **Rebalancing ni hisobga olmaslik** — Rebalancing vaqtida consumer to'xtaydi. Bu production da muammo bo'lishi mumkin.

### Phase 1 Xulosa

```
┌─────────────────────────────────────────────────────────┐
│                    KAFKA ARXITEKTURASI                   │
│                                                         │
│  ┌──────────┐                          ┌──────────────┐│
│  │Producer 1│──┐                   ┌──→│Consumer Grp 1││
│  └──────────┘  │                   │   │  (Ombor)     ││
│                │   ┌───────────┐   │   └──────────────┘│
│  ┌──────────┐  ├──→│  BROKER   │───┤                   │
│  │Producer 2│──┤   │           │   │   ┌──────────────┐│
│  └──────────┘  │   │ Topic     │   ├──→│Consumer Grp 2││
│                │   │  ├─Part 0 │   │   │  (Email)     ││
│  ┌──────────┐  │   │  ├─Part 1 │   │   └──────────────┘│
│  │Producer 3│──┘   │  └─Part 2 │   │                   │
│  └──────────┘      └───────────┘   │   ┌──────────────┐│
│                                    └──→│Consumer Grp 3││
│  Xabar Key orqali                      │  (Analitika) ││
│  partitionga tushadi                   └──────────────┘│
│                                                         │
│  Offset: Har bir group alohida offset saqlaydi         │
│  Retention: Xabarlar muddatgacha diskda saqlanadi      │
└─────────────────────────────────────────────────────────┘
```

---

# PHASE 2 — Kafkani Terminalda Ishlatish (Ubuntu Amaliyot)

---

## 7-Dars: Java O'rnatish

Kafka Java da yozilgan, shuning uchun Java kerak.

### Java Versiyasini Tekshirish

```bash
java -version
```

Agar Java o'rnatilmagan bo'lsa:

### Java 17 O'rnatish (LTS)

```bash
# Paket ro'yxatini yangilash
sudo apt update

# Java 17 JDK o'rnatish
sudo apt install -y openjdk-17-jdk

# Tekshirish
java -version
# Natija: openjdk version "17.0.x" ...

# JAVA_HOME ni sozlash
echo 'export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64' >> ~/.bashrc
echo 'export PATH=$PATH:$JAVA_HOME/bin' >> ~/.bashrc
source ~/.bashrc

# Tekshirish
echo $JAVA_HOME
# Natija: /usr/lib/jvm/java-17-openjdk-amd64
```

---

## 8-Dars: Kafka O'rnatish va KRaft Rejimini Sozlash

### KRaft Nima?

Ilgari Kafka ishlashi uchun **Zookeeper** kerak edi — bu alohida servis bo'lib, Kafka klaster metadatasini boshqarar edi. Bu murakkablik qo'shar edi.

**KRaft (Kafka Raft)** — Kafka 3.3+ dan boshlab Zookeeper o'rniga Kafkaning o'zi metadata boshqaradi.

```
Eski usul:
  Zookeeper (alohida servis) ←→ Kafka Broker

Yangi usul (KRaft):
  Kafka Broker (o'zi hammani boshqaradi)
```

### Kafka Yuklab Olish va O'rnatish

```bash
# Uy papkasiga o'tish
cd ~

# Kafka 3.7.0 yuklab olish (yoki eng oxirgi versiya)
wget https://downloads.apache.org/kafka/3.7.0/kafka_2.13-3.7.0.tgz

# Arxivni ochish
tar -xzf kafka_2.13-3.7.0.tgz

# Papkani ko'chirish
sudo mv kafka_2.13-3.7.0 /opt/kafka

# PATH ga qo'shish
echo 'export KAFKA_HOME=/opt/kafka' >> ~/.bashrc
echo 'export PATH=$PATH:$KAFKA_HOME/bin' >> ~/.bashrc
source ~/.bashrc

# Tekshirish
kafka-server-start.sh --version
```

### Kafka Papka Tuzilishi

```
/opt/kafka/
├── bin/                          ← Buyruqlar (scriptlar)
│   ├── kafka-server-start.sh     ← Kafka serverni ishga tushirish
│   ├── kafka-topics.sh           ← Topiclarni boshqarish
│   ├── kafka-console-producer.sh ← Terminaldan xabar yuborish
│   ├── kafka-console-consumer.sh ← Terminaldan xabar o'qish
│   ├── kafka-storage.sh          ← KRaft storage boshqaruvi
│   └── ...
├── config/                       ← Sozlamalar
│   ├── kraft/
│   │   └── server.properties     ← KRaft rejim sozlamalari
│   └── server.properties         ← Oddiy sozlamalar (Zookeeper uchun)
├── libs/                         ← Java kutubxonalar
└── logs/                         ← Log fayllar
```

### KRaft Konfiguratsiyasi

```bash
# KRaft konfiguratsiya faylini tahrirlash
nano /opt/kafka/config/kraft/server.properties
```

Muhim sozlamalar:

```properties
# Unikal klaster ID — keyinroq generatsiya qilamiz
# process.roles=broker,controller — bu server ham broker, ham controller

# Broker va controller roli
process.roles=broker,controller

# Node ID (har bir broker uchun unikal)
node.id=1

# Controller quorum — klasterdagi controllerlar ro'yxati
controller.quorum.voters=1@localhost:9093

# Tinglash manzillari
listeners=PLAINTEXT://:9092,CONTROLLER://:9093

# E'lon qilinadigan manzillar
advertised.listeners=PLAINTEXT://localhost:9092

# Controller listener nomi
controller.listener.names=CONTROLLER

# Ma'lumotlar saqlanadigan papka
log.dirs=/tmp/kraft-combined-logs

# Standart partition soni (yangi topiclar uchun)
num.partitions=3

# Standart replikatsiya faktori
default.replication.factor=1

# Segment hajmi
log.segment.bytes=1073741824

# Saqlash muddati (7 kun)
log.retention.hours=168
```

### KRaft Storage ni Formatlash

```bash
# Unikal klaster ID generatsiya qilish
KAFKA_CLUSTER_ID="$(kafka-storage.sh random-uuid)"

# Generatsiya qilingan ID ni ko'rish
echo $KAFKA_CLUSTER_ID
# Natija: MkU3OEVBNTcwNTJENDM2Qk (shunga o'xshash)

# Storage ni formatlash
kafka-storage.sh format \
  -t $KAFKA_CLUSTER_ID \
  -c /opt/kafka/config/kraft/server.properties

# Natija: Formatting /tmp/kraft-combined-logs with metadata.version 3.7-IV4
```

### Kafka ni Ishga Tushirish

```bash
# Foreground da ishga tushirish (loglarni ko'rish uchun)
kafka-server-start.sh /opt/kafka/config/kraft/server.properties

# YOKI background da ishga tushirish
kafka-server-start.sh -daemon /opt/kafka/config/kraft/server.properties

# Kafka ishlayotganini tekshirish
# Yangi terminal oching:
kafka-broker-api-versions.sh --bootstrap-server localhost:9092 | head -1
# Natija: localhost:9092 (id: 1 rack: null) -> (...)
```

### Kafka ni To'xtatish

```bash
kafka-server-stop.sh
```

### Keng Tarqalgan Muammolar

| Muammo | Yechim |
|--------|--------|
| `java: command not found` | Java o'rnatilmagan. 7-darsga qarang |
| Port 9092 band | `lsof -i :9092` bilan tekshirib, jarayonni to'xtating |
| `Not a valid cluster id` | Storage ni qayta formatlang |
| Xotira yetishmovchiligi | `KAFKA_HEAP_OPTS="-Xmx512M -Xms512M"` o'rnating |

---

## 9-Dars: Topic Yaratish va Boshqarish

### Topic Yaratish

```bash
# "orders" topicini 3 ta partition bilan yaratish
kafka-topics.sh --create \
  --topic orders \
  --partitions 3 \
  --replication-factor 1 \
  --bootstrap-server localhost:9092

# Natija: Created topic orders.
```

### Barcha Topiclarni Ko'rish

```bash
kafka-topics.sh --list \
  --bootstrap-server localhost:9092

# Natija:
# orders
```

### Topic Haqida Ma'lumot

```bash
kafka-topics.sh --describe \
  --topic orders \
  --bootstrap-server localhost:9092

# Natija:
# Topic: orders   TopicId: abc123   PartitionCount: 3   ReplicationFactor: 1
#   Topic: orders   Partition: 0   Leader: 1   Replicas: 1   Isr: 1
#   Topic: orders   Partition: 1   Leader: 1   Replicas: 1   Isr: 1
#   Topic: orders   Partition: 2   Leader: 1   Replicas: 1   Isr: 1
```

**Tushuntirish:**
- `PartitionCount: 3` — 3 ta partition
- `Leader: 1` — Broker 1 bu partitionning lideri
- `Replicas: 1` — Faqat 1 ta nusxa (bitta broker)
- `Isr: 1` — In-Sync Replicas (sinxronlashgan nusxalar)

### Partition Sonini Ko'paytirish

```bash
# 3 dan 6 ga ko'paytirish
kafka-topics.sh --alter \
  --topic orders \
  --partitions 6 \
  --bootstrap-server localhost:9092

# Tekshirish
kafka-topics.sh --describe \
  --topic orders \
  --bootstrap-server localhost:9092
```

> **Diqqat:** Partition sonini KAMAYTIRISH mumkin emas! Faqat ko'paytirish mumkin.

### Topic O'chirish

```bash
kafka-topics.sh --delete \
  --topic orders \
  --bootstrap-server localhost:9092
```

### Ko'p Ishlatiladigan Topiclarni Yaratish (Amaliyot)

```bash
# Buyurtmalar
kafka-topics.sh --create --topic orders \
  --partitions 3 --replication-factor 1 \
  --bootstrap-server localhost:9092

# To'lovlar
kafka-topics.sh --create --topic payments \
  --partitions 3 --replication-factor 1 \
  --bootstrap-server localhost:9092

# Bildirishnomalar
kafka-topics.sh --create --topic notifications \
  --partitions 2 --replication-factor 1 \
  --bootstrap-server localhost:9092

# Foydalanuvchi voqealari
kafka-topics.sh --create --topic user-events \
  --partitions 3 --replication-factor 1 \
  --bootstrap-server localhost:9092

# Barcha topiclarni ko'rish
kafka-topics.sh --list --bootstrap-server localhost:9092
```

---

## 10-Dars: Xabar Yuborish va O'qish (Terminal)

### Console Producer — Xabar Yuborish

```bash
# "orders" topicga xabar yuborish
kafka-console-producer.sh \
  --topic orders \
  --bootstrap-server localhost:9092

# Kursor paydo bo'ladi, xabar yozing:
>Birinchi buyurtma
>Ikkinchi buyurtma
>{"order_id": 1, "product": "Noutbuk", "amount": 8500000}
>{"order_id": 2, "product": "Telefon", "amount": 3200000}
# Ctrl+C bilan chiqish
```

### Console Consumer — Xabar O'qish

```bash
# "orders" topicdan xabar o'qish (faqat yangi xabarlar)
kafka-console-consumer.sh \
  --topic orders \
  --bootstrap-server localhost:9092

# Boshidan o'qish (barcha xabarlar)
kafka-console-consumer.sh \
  --topic orders \
  --from-beginning \
  --bootstrap-server localhost:9092

# Natija:
# Birinchi buyurtma
# Ikkinchi buyurtma
# {"order_id": 1, "product": "Noutbuk", "amount": 8500000}
# {"order_id": 2, "product": "Telefon", "amount": 3200000}
```

### Key Bilan Xabar Yuborish

Key xabarning qaysi partitionga tushishini belgilaydi. Bir xil key doimo bir xil partitionga tushadi.

```bash
# Key bilan producer (key:value formatda)
kafka-console-producer.sh \
  --topic orders \
  --property "parse.key=true" \
  --property "key.separator=:" \
  --bootstrap-server localhost:9092

# Xabar yozing (key:value):
>user-1:{"order_id":1, "user":"Ali"}
>user-2:{"order_id":2, "user":"Vali"}
>user-1:{"order_id":3, "user":"Ali"}
# user-1 ga tegishli xabarlar DOIMO bitta partitionga tushadi
```

### Key ni Ko'rsatib O'qish

```bash
kafka-console-consumer.sh \
  --topic orders \
  --from-beginning \
  --property "print.key=true" \
  --property "print.partition=true" \
  --property "print.offset=true" \
  --bootstrap-server localhost:9092

# Natija:
# Partition:0  Offset:0  null    Birinchi buyurtma
# Partition:2  Offset:0  null    Ikkinchi buyurtma
# Partition:1  Offset:0  user-1  {"order_id":1, "user":"Ali"}
# Partition:0  Offset:1  user-2  {"order_id":2, "user":"Vali"}
# Partition:1  Offset:1  user-1  {"order_id":3, "user":"Ali"}
```

E'tibor bering: `user-1` keyga ega xabarlar doimo **Partition 1** ga tushdi.

### Consumer Group Bilan O'qish

```bash
# 1-terminal: Consumer 1
kafka-console-consumer.sh \
  --topic orders \
  --group order-processor \
  --bootstrap-server localhost:9092

# 2-terminal: Consumer 2
kafka-console-consumer.sh \
  --topic orders \
  --group order-processor \
  --bootstrap-server localhost:9092

# 3-terminal: Xabar yuborish
kafka-console-producer.sh \
  --topic orders \
  --bootstrap-server localhost:9092

# Xabar yozing — consumer lar orasida taqsimlanadi
>test1
>test2
>test3
>test4
>test5
>test6
```

Ikkala consumer ham xabarlarni **taqsimlab** oladi (har biri o'z partitionlaridan).

### Consumer Group Holatini Ko'rish

```bash
# Barcha consumer grouplarni ko'rish
kafka-consumer-groups.sh --list \
  --bootstrap-server localhost:9092

# "order-processor" group haqida batafsil
kafka-consumer-groups.sh --describe \
  --group order-processor \
  --bootstrap-server localhost:9092

# Natija:
# GROUP           TOPIC    PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
# order-processor orders   0          4               4               0
# order-processor orders   1          3               3               0
# order-processor orders   2          2               2               0
```

**Ustunlar tushuntirishi:**
- `CURRENT-OFFSET` — Consumer hozir qayerda turgan
- `LOG-END-OFFSET` — Partitiondagi oxirgi xabar offseti
- `LAG` — O'qilmagan xabarlar soni (LOG-END - CURRENT). **0 bo'lsa yaxshi!**

---

## 11-Dars: Partition va Offsetlarni Tekshirish

### Partition Ichidagi Xabarlarni Ko'rish

```bash
# Faqat Partition 0 dan o'qish
kafka-console-consumer.sh \
  --topic orders \
  --partition 0 \
  --from-beginning \
  --bootstrap-server localhost:9092
```

### Ma'lum Offsetdan Boshlab O'qish

```bash
# Partition 0 da offset 2 dan boshlab o'qish
kafka-console-consumer.sh \
  --topic orders \
  --partition 0 \
  --offset 2 \
  --bootstrap-server localhost:9092
```

### Topic Offsetlarini Ko'rish

```bash
# Eng erta (earliest) offsetlar
kafka-run-class.sh kafka.tools.GetOffsetShell \
  --topic orders \
  --time -2 \
  --broker-list localhost:9092

# Eng so'nggi (latest) offsetlar
kafka-run-class.sh kafka.tools.GetOffsetShell \
  --topic orders \
  --time -1 \
  --broker-list localhost:9092
```

### Consumer Group Offsetini Qayta O'rnatish (Reset)

```bash
# DIQQAT: Avval consumer larni to'xtating!

# Boshiga qaytarish
kafka-consumer-groups.sh --reset-offsets \
  --group order-processor \
  --topic orders \
  --to-earliest \
  --execute \
  --bootstrap-server localhost:9092

# Ma'lum offsetga o'rnatish
kafka-consumer-groups.sh --reset-offsets \
  --group order-processor \
  --topic orders \
  --to-offset 5 \
  --execute \
  --bootstrap-server localhost:9092

# Ma'lum vaqtga o'rnatish
kafka-consumer-groups.sh --reset-offsets \
  --group order-processor \
  --topic orders \
  --to-datetime 2025-01-15T00:00:00.000 \
  --execute \
  --bootstrap-server localhost:9092
```

### Diskdagi Ma'lumotlarni Ko'rish

```bash
# Log papkasini ko'rish
ls -la /tmp/kraft-combined-logs/

# orders topicining partition papkalarini ko'rish
ls -la /tmp/kraft-combined-logs/orders-*/

# Segment fayllarni ko'rish
ls -la /tmp/kraft-combined-logs/orders-0/

# Segment faylni dump qilish (o'qish)
kafka-dump-log.sh \
  --files /tmp/kraft-combined-logs/orders-0/00000000000000000000.log \
  --print-data-log

# Natija:
# Dumping /tmp/kraft-combined-logs/orders-0/00000000000000000000.log
# Log starting offset: 0
# baseOffset: 0 ... payload: Birinchi buyurtma
# baseOffset: 1 ... payload: {"order_id": 1, ...}
```

### Foydali Kafka Buyruqlari Jadvali

| Buyruq | Vazifasi |
|--------|---------|
| `kafka-topics.sh --create` | Yangi topic yaratish |
| `kafka-topics.sh --list` | Barcha topiclarni ko'rsatish |
| `kafka-topics.sh --describe` | Topic haqida batafsil ma'lumot |
| `kafka-topics.sh --delete` | Topic o'chirish |
| `kafka-topics.sh --alter` | Topic sozlamalarini o'zgartirish |
| `kafka-console-producer.sh` | Terminaldan xabar yuborish |
| `kafka-console-consumer.sh` | Terminaldan xabar o'qish |
| `kafka-consumer-groups.sh --list` | Consumer grouplarni ko'rsatish |
| `kafka-consumer-groups.sh --describe` | Group haqida batafsil ma'lumot |
| `kafka-consumer-groups.sh --reset-offsets` | Offsetni qayta o'rnatish |
| `kafka-dump-log.sh` | Segment fayllarni o'qish |

---

# PHASE 3 — Kafka Arxitektura Ustasi

---

## 12-Dars: Consumer Grouplar Chuqurroq

### Offset Commit Strategiyalari

Consumer xabar o'qigandan keyin, "men buni o'qidim" deb Kafkaga xabar berishi kerak. Bunga **offset commit** deyiladi.

**1. Auto Commit (Avtomatik):**
```properties
enable.auto.commit=true
auto.commit.interval.ms=5000  # Har 5 soniyada
```

```
Xabarlar: [0] [1] [2] [3] [4] [5] [6] [7]
                              ↑
                    Avtomatik commit (5 soniyada bir)

Muammo: Agar Consumer 4-xabarni qayta ishlab,
5-soniya o'tib commit bo'lsa, lekin keyin crash bo'lsa:
→ 5,6,7 xabarlar YO'QOLADI (commit bo'lgan, lekin ishlanmagan)
```

**2. Manual Commit (Qo'lda):**
```
Xabarlar: [0] [1] [2] [3] [4] [5] [6] [7]

Har bir xabar ishlanganidan KEYIN commit:
  O'qi → Ishlat → Commit → O'qi → Ishlat → Commit

Afzallik: Xabar yo'qolmaydi
Kamchilik: Sekinroq
```

**3. Batch Commit:**
```
Xabarlar: [0] [1] [2] [3] [4] [5] [6] [7]

10 ta xabar ishlanganidan keyin commit:
  O'qi [0-9] → Ishlat → Commit offset=9

Afzallik: Tez va ishonchli
Kamchilik: Crash bo'lsa, 10 tagacha xabar qayta ishlanishi mumkin
```

### Consumer Heartbeat va Session

Consumer Kafkaga "men tirikman" degan signal yuboradi:

```
Consumer ──heartbeat──→ Kafka (har 3 soniyada)
Consumer ──heartbeat──→ Kafka
Consumer ──heartbeat──→ Kafka
Consumer ──────X────── Kafka (10 soniya javob yo'q)
                            ↓
                    "Consumer o'ldi" deb hisoblanadi
                            ↓
                    REBALANCING boshlanadi
```

**Sozlamalar:**
```properties
# Heartbeat intervali
heartbeat.interval.ms=3000

# Session timeout — bu vaqt ichida heartbeat kelmasa, consumer o'lgan hisoblanadi
session.timeout.ms=45000

# Xabarni qayta ishlash uchun max vaqt
max.poll.interval.ms=300000  # 5 daqiqa
```

### Poll Loop

Consumer xabarlarni **poll** qilib oladi:

```
while (true) {
    records = consumer.poll(timeout=1000ms)
    for record in records:
        process(record)
    consumer.commit()
}
```

**max.poll.records** — bir poll da nechta xabar olinadi:
```properties
max.poll.records=500  # Bir poll da max 500 ta xabar
```

Agar xabarni ishlash uzoq davom etsa va `max.poll.interval.ms` vaqti o'tsa, Kafka consumerni "o'lgan" deb hisoblaydi va rebalancing qiladi.

---

## 13-Dars: Rebalancing Chuqurroq

### Rebalancing Qachon Bo'ladi?

1. **Yangi consumer qo'shilganda**
2. **Consumer tushganda (crash/shutdown)**
3. **Yangi partition qo'shilganda**
4. **Consumer subscription o'zgarganda**

### Rebalancing Jarayoni

```
1. Trigger (consumer qo'shildi/tushdi)
        ↓
2. Barcha consumerlar STOP (xabar o'qishni to'xtatadi)
        ↓
3. Group Coordinator partitionlarni qayta taqsimlaydi
        ↓
4. Har bir consumer yangi partition tayinlanadi
        ↓
5. Consumerlar RESUME (xabar o'qishni davom ettiradi)
```

**Muammo:** Rebalancing vaqtida (2-5 soniya) HECH KIM xabar o'qimaydi!

### Partition Assignment Strategiyalari

**1. Range (Standart):**
```
Topiclar: orders (P0,P1,P2), payments (P0,P1,P2)
Consumerlar: C0, C1

C0 → orders-P0, orders-P1, payments-P0, payments-P1
C1 → orders-P2, payments-P2

Muammo: C0 ga ko'proq partition tushdi (notekis)
```

**2. RoundRobin:**
```
C0 → orders-P0, orders-P2, payments-P1
C1 → orders-P1, payments-P0, payments-P2

Afzallik: Tekisroq taqsimlash
```

**3. Sticky:**
```
Rebalancing da imkon qadar MAVJUD tayinlashni saqlaydi.

Oldin:
  C0 → P0, P1
  C1 → P2

C2 qo'shildi:
  C0 → P0       (P1 olib tashlandi)
  C1 → P2       (o'zgarmadi!)
  C2 → P1       (C0 dan olindi)

Afzallik: Minimal o'zgarish, tez rebalancing
```

**4. CooperativeSticky (Tavsiya etiladi):**
```
Oddiy rebalancing:
  STOP all → Reassign → RESUME all

Cooperative rebalancing:
  Faqat o'zgargan partitionlar STOP → Reassign → RESUME
  Qolgan partitionlar ishlashda davom etadi!

Afzallik: Downtime deyarli yo'q
```

```properties
# CooperativeSticky ni yoqish
partition.assignment.strategy=org.apache.kafka.clients.consumer.CooperativeStickyAssignor
```

### Static Group Membership

Har safar consumer qayta ishga tushganda rebalancing bo'lmasligini istasangiz:

```properties
# Har bir consumerga unikal ID berish
group.instance.id=consumer-host-1
```

Bu bilan consumer tushib qayta turganda, Kafka uni "yangi" consumer deb emas, "qaytgan" consumer deb taniydi va rebalancing qilmaydi.

---

## 14-Dars: Xabar Tartibi (Message Ordering)

### Partition Ichida Tartib

**KAFOLAT:** Bitta partition ichida xabarlar **doimo tartibda** bo'ladi.

```
Partition 0:
  [Offset 0: order.created]
  [Offset 1: order.paid]
  [Offset 2: order.shipped]

Bu tartib HECH QACHON buzilmaydi.
```

### Partitionlar Orasida Tartib

**KAFOLAT YO'Q:** Turli partitionlarda tartib kafolatlanmaydi.

```
Partition 0: [order.created (user-1)]    t=10:00:01
Partition 1: [order.created (user-2)]    t=10:00:00

Consumer P0 ni P1 dan KEYIN o'qishi mumkin,
natijada user-2 ning buyurtmasi user-1 nikidan keyin ko'rinadi,
garchi aslida avvalroq bo'lgan bo'lsa ham.
```

### Tartibni Qanday Ta'minlash?

**Usul 1: Key Ishlatish**

Bir xil keyga ega xabarlar doimo bitta partitionga tushadi:

```
Key: "user-67"
  → order.created  → Partition 2
  → order.paid     → Partition 2  (bir xil partition!)
  → order.shipped  → Partition 2  (bir xil partition!)

Natija: user-67 ning barcha voqealari TARTIBDA
```

**Usul 2: Bitta Partition**

```bash
kafka-topics.sh --create \
  --topic critical-events \
  --partitions 1 \
  --replication-factor 1 \
  --bootstrap-server localhost:9092
```

Bitta partition = to'liq tartib kafolati, LEKIN masshtablanmaydi.

**Usul 3: Idempotent Producer**

```
max.in.flight.requests.per.connection=5
enable.idempotence=true
```

Idempotent producer bilan, hatto retry bo'lsa ham tartib buzilmaydi (bitta partition ichida).

### Key Tanlash Strategiyasi

| Holat | Tavsiya etilgan Key |
|-------|-------------------|
| Buyurtma voqealari | `order_id` |
| Foydalanuvchi voqealari | `user_id` |
| To'lov voqealari | `payment_id` yoki `order_id` |
| Mahsulot yangilanishlari | `product_id` |
| Sessiya voqealari | `session_id` |

```
Key: "order-123"
  order.created  ──→ Partition X
  order.updated  ──→ Partition X  (doimo bir xil)
  order.cancelled──→ Partition X  (doimo bir xil)
```

---

## 15-Dars: Replikatsiya (Replication)

### Nega Replikatsiya Kerak?

Bitta broker tushsa, undagi barcha ma'lumotlar yo'qoladi:

```
Replikatsiyasiz:
  Broker 1: [orders-P0: msg1, msg2, msg3]
  Broker 1 TUSHDI → msg1, msg2, msg3 YO'QOLDI!

Replikatsiya bilan:
  Broker 1: [orders-P0: msg1, msg2, msg3]  ← Leader
  Broker 2: [orders-P0: msg1, msg2, msg3]  ← Follower (nusxa)
  Broker 1 TUSHDI → Broker 2 LEADER bo'ladi → Ma'lumot SAQLANIB QOLADI!
```

### Leader va Follower

Har bir partitionning **bitta Leader** va **bir nechta Follower** lari bor:

```
Topic: "orders", Partition 0, Replication Factor = 3

  Broker 1: [P0 - LEADER]     ← Producer shu yerga yozadi
  Broker 2: [P0 - Follower]   ← Leader dan nusxa oladi
  Broker 3: [P0 - Follower]   ← Leader dan nusxa oladi

Producer → Leader → Follower 1, Follower 2 (parallel nusxalash)
Consumer ← Leader (standart holda leader dan o'qiydi)
```

### ISR (In-Sync Replicas)

**ISR** — Leader bilan sinxronlashgan followerlar ro'yxati.

```
Leader:    [msg1] [msg2] [msg3] [msg4] [msg5]
Follower1: [msg1] [msg2] [msg3] [msg4] [msg5]  ← ISR da (sinxron)
Follower2: [msg1] [msg2] [msg3]                 ← ISR dan CHIQDI (orqada)

ISR = {Leader, Follower1}
```

Follower orqada qolsa (sozlangan vaqtdan ko'p), ISR dan chiqariladi.

```properties
# Follower bu vaqtda sinxronlanmasa, ISR dan chiqariladi
replica.lag.time.max.ms=30000  # 30 soniya
```

### Acks (Acknowledgment) Sozlamalari

Producer xabar yuborganda, qanday kafolat olishni tanlashi mumkin:

**acks=0 — "Yuborish va unutish"**
```
Producer → Broker (javob KUTMAYDI)

Tezlik: ⚡⚡⚡ Eng tez
Ishonchlilik: ❌ Xabar yo'qolishi mumkin
Foydalanish: Log, metrika (yo'qolsa ham bo'ladi)
```

**acks=1 — "Leader tasdiqlashi" (Standart)**
```
Producer → Leader (yozdi) → Producer ga "OK"
           ↓
        Follower larga KEYIN nusxalanadi

Tezlik: ⚡⚡ Tez
Ishonchlilik: ⚠️ Leader tushsa, nusxalanmagan xabarlar yo'qoladi
Foydalanish: Ko'p hollarda yetarli
```

**acks=all (-1) — "Barcha ISR tasdiqlashi"**
```
Producer → Leader (yozdi) → Follower 1 (yozdi) → Follower 2 (yozdi)
                                                        ↓
                                              Producer ga "OK"

Tezlik: ⚡ Eng sekin
Ishonchlilik: ✅ Eng ishonchli
Foydalanish: To'lov, buyurtma (yo'qolmasligi kerak)
```

### min.insync.replicas

`acks=all` bilan birga ishlatiladi:

```properties
min.insync.replicas=2  # Kamida 2 ta ISR bo'lishi kerak
```

```
Replication Factor = 3, min.insync.replicas = 2

Holat 1: 3 broker ishlayapti
  ISR = {Leader, F1, F2} → Yozish MUVAFFAQIYATLI

Holat 2: 1 broker tushdi
  ISR = {Leader, F1} → Yozish MUVAFFAQIYATLI (2 >= 2)

Holat 3: 2 broker tushdi
  ISR = {Leader} → Yozish RAD ETILADI (1 < 2)
  Bu ma'lumot yo'qolishini oldini oladi
```

### Tavsiya Etilgan Production Sozlamalari

```
Replication Factor = 3
min.insync.replicas = 2
acks = all

Bu bilan:
- 1 ta broker tushsa → Tizim ishlaydi, ma'lumot yo'qolmaydi
- 2 ta broker tushsa → Yozish to'xtaydi (lekin ma'lumot saqlanadi)
```

---

## 16-Dars: Nosozlikka Chidamlilik (Fault Tolerance)

### Broker Tushganda Nima Bo'ladi?

```
OLDIN:
  Broker 1: orders-P0 (Leader), orders-P1 (Follower)
  Broker 2: orders-P0 (Follower), orders-P1 (Leader)
  Broker 3: orders-P0 (Follower), orders-P1 (Follower)

Broker 2 TUSHDI:

KEYIN:
  Broker 1: orders-P0 (Leader), orders-P1 (YANGI Leader!)
  Broker 3: orders-P0 (Follower), orders-P1 (Follower)

→ Follower lardan biri avtomatik Leader bo'ladi
→ Client lar yangi Leader ga yo'naltiriladi
→ Foydalanuvchi hech narsa sezmaydi
```

### Unclean Leader Election

Barcha ISR dagi brokerlar tushsa nima bo'ladi?

```properties
# Standart: false (ma'lumot yo'qolmasin)
unclean.leader.election.enable=false

# Agar true bo'lsa:
# ISR da bo'lmagan (orqada qolgan) follower Leader bo'lishi mumkin
# Bu ma'lumot YO'QOLISHIGA olib keladi, lekin tizim ishlashda davom etadi
```

**Qoidalar:**
- `false` → Ma'lumot muhimroq (to'lov, buyurtma)
- `true` → Availability muhimroq (log, metrika)

### Controller va KRaft

KRaft rejimida **controller** Kafka klasterini boshqaradi:

```
┌────────────────────────────────────────────┐
│            KRaft Cluster                   │
│                                            │
│  Broker 1 (Controller + Broker)            │
│  Broker 2 (Controller + Broker)            │
│  Broker 3 (Controller + Broker)            │
│                                            │
│  Controller quorum: 3 ta controller        │
│  Raft protokoli bilan leader tanlaydi      │
│                                            │
│  1 ta controller tushsa → Yangi leader     │
│  2 ta controller tushsa → Klaster to'xtaydi│
└────────────────────────────────────────────┘
```

---

## 17-Dars: Retention Siyosatlari

### Time-Based Retention (Vaqt Bo'yicha)

```properties
# Topic darajasida
# 7 kun (standart)
log.retention.hours=168

# 30 kun
log.retention.hours=720

# 1 soat
log.retention.hours=1

# Aniqroq sozlash
log.retention.ms=86400000  # 1 kun (millisekundda)
```

### Size-Based Retention (Hajm Bo'yicha)

```properties
# Har bir partition uchun max hajm
log.retention.bytes=1073741824  # 1 GB

# Cheksiz (standart)
log.retention.bytes=-1
```

### Topic Darajasida O'zgartirish

```bash
# "orders" topic uchun 30 kun saqlash
kafka-configs.sh --alter \
  --entity-type topics \
  --entity-name orders \
  --add-config retention.ms=2592000000 \
  --bootstrap-server localhost:9092

# Tekshirish
kafka-configs.sh --describe \
  --entity-type topics \
  --entity-name orders \
  --bootstrap-server localhost:9092
```

### Log Compaction

**Compaction** — har bir key uchun faqat **oxirgi qiymatni** saqlaydi.

```
OLDIN (oddiy retention):
  key=user-1, value={"name":"Ali"}          offset=0
  key=user-2, value={"name":"Vali"}         offset=1
  key=user-1, value={"name":"Ali Updated"}  offset=2
  key=user-3, value={"name":"Soli"}         offset=3
  key=user-2, value={"name":"Vali Updated"} offset=4

KEYIN (compaction):
  key=user-1, value={"name":"Ali Updated"}  offset=2  (oxirgisi)
  key=user-3, value={"name":"Soli"}         offset=3  (yagona)
  key=user-2, value={"name":"Vali Updated"} offset=4  (oxirgisi)
```

```bash
# Compaction yoqish
kafka-configs.sh --alter \
  --entity-type topics \
  --entity-name user-profiles \
  --add-config cleanup.policy=compact \
  --bootstrap-server localhost:9092
```

**Compaction qachon ishlatiladi:**
- Foydalanuvchi profillari (oxirgi holat kerak)
- Konfiguratsiya o'zgarishlari
- Balans holati

**Compaction + Delete:**
```bash
# Ikkalasi ham — muddat o'tganini o'chirish VA compaction
kafka-configs.sh --alter \
  --entity-type topics \
  --entity-name user-profiles \
  --add-config cleanup.policy=compact,delete \
  --bootstrap-server localhost:9092
```

---

## 18-Dars: Idempotentlik va Yetkazish Kafolatlari

### Muammo: Duplikat Xabarlar

```
Producer → Broker: msg1 yubordi
Broker: msg1 ni yozdi
Broker → Producer: "OK" javob yuborayapti...
NETWORK XATOSI! Producer javob olmadi.

Producer: "Javob kelmadi, qayta yuboraman"
Producer → Broker: msg1 ni QAYTA yubordi
Broker: msg1 ni YANA yozdi

Natija: msg1 IKKI MARTA yozildi! (duplikat)
```

### Idempotent Producer

```properties
enable.idempotence=true  # Kafka 3.0+ da standart true
```

Kafka har bir producerga unikal **Producer ID (PID)** va har bir xabarga **Sequence Number** beradi:

```
Producer (PID=1):
  msg1 (seq=0) → Broker: YOZDI
  msg1 (seq=0) → Broker: "Bu seq=0 allaqachon bor, TASHLAYAPMAN" (duplikat!)
  msg2 (seq=1) → Broker: YOZDI
```

### Yetkazish Kafolatlari

**1. At Most Once (Ko'pi bilan bir marta)**
```
Producer → Broker (acks=0, retry yo'q)

Xabar: 0 yoki 1 marta yetkaziladi
Muammo: Xabar yo'qolishi MUMKIN
Foydalanish: Log, metrika
```

**2. At Least Once (Kamida bir marta)**
```
Producer → Broker (acks=1 yoki all, retry bor)

Xabar: 1 yoki UNDAN KO'P marta yetkaziladi
Muammo: Duplikatlar bo'lishi MUMKIN
Foydalanish: Ko'p hollarda yetarli
```

**3. Exactly Once (Aniq bir marta)**
```
Producer → Broker (idempotent + transactional)

Xabar: ANIQ 1 marta yetkaziladi
Muammo: Eng sekin
Foydalanish: To'lov, moliyaviy operatsiyalar
```

### Exactly Once Semantics (EOS) Sozlamalari

**Producer tomonda:**
```properties
enable.idempotence=true
transactional.id=my-transactional-producer
acks=all
retries=2147483647  # Integer.MAX_VALUE
max.in.flight.requests.per.connection=5
```

**Consumer tomonda:**
```properties
isolation.level=read_committed
# Faqat commit bo'lgan tranzaksiya xabarlarini o'qiydi
```

### Tranzaksiya Misoli (Pseudocode)

```
producer.initTransactions()

try:
    producer.beginTransaction()

    producer.send("orders", msg1)
    producer.send("payments", msg2)
    producer.send("notifications", msg3)

    producer.commitTransaction()
    # Uchala xabar ham BIRGALIKDA yoziladi yoki HECH BIRI yozilmaydi

catch Exception:
    producer.abortTransaction()
    # Hech narsa yozilmaydi
```

### Amaliy Tavsiyalar

| Holat | Tavsiya |
|-------|---------|
| Loglar, metrika | At Most Once (`acks=0`) |
| Oddiy voqealar | At Least Once (`acks=all` + idempotent) |
| To'lov, moliya | Exactly Once (transactional) |
| Consumer tomoni | Idempotent consumer yozing (bir xil xabarni ikki marta qabul qilsa ham muammo bo'lmasin) |

### Idempotent Consumer Qanday Yoziladi?

```
YOMON:
  consume(msg) {
    balance += msg.amount  // Ikki marta kelsa, ikki marta qo'shiladi!
  }

YAXSHI:
  consume(msg) {
    if (already_processed(msg.id)) {
      return  // Allaqachon ishlangan, o'tkazib yuboramiz
    }
    balance += msg.amount
    mark_as_processed(msg.id)
  }
```

Odatda **bazada** `processed_events` jadvali yaratiladi va har bir event ID saqlanadi.

---

# PHASE 4 — Golang + Kafka

---

## 19-Dars: Go da Kafka Producer Yozish

### Kutubxona Tanlash

Go uchun eng mashhur Kafka kutubxonalari:

| Kutubxona | Xususiyat |
|-----------|-----------|
| `confluent-kafka-go` | librdkafka asosida, eng tez, C bog'lanishi kerak |
| `segmentio/kafka-go` | Pure Go, o'rnatish oson, yetarlicha tez |
| `IBM/sarama` | Eski, keng tarqalgan, murakkab API |

Biz **`segmentio/kafka-go`** ishlatamiz — o'rnatish oson, pure Go, production uchun yetarli.

### Loyiha Yaratish

```bash
# Loyiha papkasi
mkdir -p ~/kafka-course/producer
cd ~/kafka-course/producer

# Go module yaratish
go mod init kafka-producer

# kafka-go kutubxonasini o'rnatish
go get github.com/segmentio/kafka-go
```

### Oddiy Producer

```go
// ~/kafka-course/producer/main.go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"time"

	"github.com/segmentio/kafka-go"
)

// Order — buyurtma strukturasi
type Order struct {
	OrderID   string    `json:"order_id"`
	UserID    int       `json:"user_id"`
	Product   string    `json:"product"`
	Amount    int       `json:"amount"`
	Currency  string    `json:"currency"`
	CreatedAt time.Time `json:"created_at"`
}

func main() {
	// Kafka writer (producer) yaratish
	writer := &kafka.Writer{
		Addr:         kafka.TCP("localhost:9092"),
		Topic:        "orders",
		Balancer:     &kafka.LeastBytes{}, // Partition tanlash strategiyasi
		BatchTimeout: 10 * time.Millisecond,
		RequiredAcks: kafka.RequireAll, // acks=all
	}
	defer writer.Close()

	// Buyurtma yaratish
	order := Order{
		OrderID:   "ORD-2025-001",
		UserID:    67,
		Product:   "Noutbuk Lenovo ThinkPad",
		Amount:    8500000,
		Currency:  "UZS",
		CreatedAt: time.Now(),
	}

	// JSON ga aylantirish
	value, err := json.Marshal(order)
	if err != nil {
		log.Fatalf("JSON marshal xatosi: %v", err)
	}

	// Kafkaga yuborish
	err = writer.WriteMessages(context.Background(),
		kafka.Message{
			Key:   []byte(order.OrderID), // Key — shu buyurtmaning barcha voqealari bitta partitionga tushadi
			Value: value,
		},
	)

	if err != nil {
		log.Fatalf("Xabar yuborishda xato: %v", err)
	}

	fmt.Printf("Xabar yuborildi: %s\n", order.OrderID)
}
```

### Ishga Tushirish

```bash
cd ~/kafka-course/producer
go run main.go

# Natija: Xabar yuborildi: ORD-2025-001
```

### Tekshirish (Terminalda)

```bash
kafka-console-consumer.sh \
  --topic orders \
  --from-beginning \
  --property print.key=true \
  --bootstrap-server localhost:9092

# Natija:
# ORD-2025-001  {"order_id":"ORD-2025-001","user_id":67,...}
```

### Ko'p Xabar Yuborish (Batch)

```go
// batch_producer.go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"time"

	"github.com/segmentio/kafka-go"
)

type Order struct {
	OrderID   string    `json:"order_id"`
	UserID    int       `json:"user_id"`
	Product   string    `json:"product"`
	Amount    int       `json:"amount"`
	CreatedAt time.Time `json:"created_at"`
}

func main() {
	writer := &kafka.Writer{
		Addr:         kafka.TCP("localhost:9092"),
		Topic:        "orders",
		Balancer:     &kafka.Hash{}, // Key bo'yicha partition tanlash
		BatchSize:    100,            // 100 tagacha xabarni birgalikda yuborish
		BatchTimeout: 10 * time.Millisecond,
		RequiredAcks: kafka.RequireAll,
	}
	defer writer.Close()

	// 10 ta buyurtma yaratish va yuborish
	orders := []Order{
		{OrderID: "ORD-001", UserID: 1, Product: "Noutbuk", Amount: 8500000},
		{OrderID: "ORD-002", UserID: 2, Product: "Telefon", Amount: 3200000},
		{OrderID: "ORD-003", UserID: 1, Product: "Sichqoncha", Amount: 150000},
		{OrderID: "ORD-004", UserID: 3, Product: "Klaviatura", Amount: 450000},
		{OrderID: "ORD-005", UserID: 2, Product: "Monitor", Amount: 2800000},
		{OrderID: "ORD-006", UserID: 4, Product: "Printer", Amount: 1500000},
		{OrderID: "ORD-007", UserID: 1, Product: "USB kabel", Amount: 25000},
		{OrderID: "ORD-008", UserID: 5, Product: "SSD 512GB", Amount: 600000},
		{OrderID: "ORD-009", UserID: 3, Product: "RAM 16GB", Amount: 750000},
		{OrderID: "ORD-010", UserID: 4, Product: "Sumka", Amount: 200000},
	}

	messages := make([]kafka.Message, len(orders))
	for i, order := range orders {
		order.CreatedAt = time.Now()
		value, _ := json.Marshal(order)
		messages[i] = kafka.Message{
			Key:   []byte(fmt.Sprintf("user-%d", order.UserID)),
			Value: value,
		}
	}

	// Batch yuborish
	start := time.Now()
	err := writer.WriteMessages(context.Background(), messages...)
	if err != nil {
		log.Fatalf("Batch yuborishda xato: %v", err)
	}

	fmt.Printf("%d ta xabar yuborildi, vaqt: %v\n", len(messages), time.Since(start))
}
```

---

## 20-Dars: Go da Kafka Consumer Yozish

### Oddiy Consumer

```go
// ~/kafka-course/consumer/main.go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"os"
	"os/signal"
	"syscall"

	"github.com/segmentio/kafka-go"
)

type Order struct {
	OrderID   string `json:"order_id"`
	UserID    int    `json:"user_id"`
	Product   string `json:"product"`
	Amount    int    `json:"amount"`
	Currency  string `json:"currency"`
	CreatedAt string `json:"created_at"`
}

func main() {
	// Kafka reader (consumer) yaratish
	reader := kafka.NewReader(kafka.ReaderConfig{
		Brokers:  []string{"localhost:9092"},
		Topic:    "orders",
		GroupID:  "order-processor", // Consumer Group
		MinBytes: 1,                 // Min batch hajmi
		MaxBytes: 10e6,              // Max batch hajmi (10 MB)
	})
	defer reader.Close()

	// Graceful shutdown
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	sigChan := make(chan os.Signal, 1)
	signal.Notify(sigChan, syscall.SIGINT, syscall.SIGTERM)

	go func() {
		<-sigChan
		fmt.Println("\nTo'xtatilmoqda...")
		cancel()
	}()

	fmt.Println("Consumer ishga tushdi. Xabarlar kutilmoqda...")

	// Xabarlarni o'qish loop
	for {
		msg, err := reader.ReadMessage(ctx)
		if err != nil {
			if ctx.Err() != nil {
				break // Graceful shutdown
			}
			log.Printf("O'qishda xato: %v", err)
			continue
		}

		// JSON ni parse qilish
		var order Order
		if err := json.Unmarshal(msg.Value, &order); err != nil {
			log.Printf("JSON parse xatosi: %v", err)
			continue
		}

		fmt.Printf("Yangi buyurtma qabul qilindi:\n")
		fmt.Printf("  Partition: %d, Offset: %d\n", msg.Partition, msg.Offset)
		fmt.Printf("  Key: %s\n", string(msg.Key))
		fmt.Printf("  OrderID: %s\n", order.OrderID)
		fmt.Printf("  Mahsulot: %s\n", order.Product)
		fmt.Printf("  Narx: %d UZS\n", order.Amount)
		fmt.Println("---")
	}

	fmt.Println("Consumer to'xtadi.")
}
```

### Ishga Tushirish

```bash
# 1-terminal: Consumer ishga tushirish
cd ~/kafka-course/consumer
go mod init kafka-consumer
go get github.com/segmentio/kafka-go
go run main.go

# 2-terminal: Producer orqali xabar yuborish
cd ~/kafka-course/producer
go run main.go

# 1-terminalda natija:
# Yangi buyurtma qabul qilindi:
#   Partition: 2, Offset: 0
#   Key: ORD-2025-001
#   OrderID: ORD-2025-001
#   Mahsulot: Noutbuk Lenovo ThinkPad
#   Narx: 8500000 UZS
# ---
```

### Manual Commit Bilan Consumer

```go
// manual_commit_consumer.go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"os"
	"os/signal"
	"syscall"

	"github.com/segmentio/kafka-go"
)

type Order struct {
	OrderID string `json:"order_id"`
	UserID  int    `json:"user_id"`
	Product string `json:"product"`
	Amount  int    `json:"amount"`
}

func processOrder(order Order) error {
	// Bu yerda haqiqiy ish bajariladi:
	// - Bazaga yozish
	// - Tashqi API ga so'rov
	// - va h.k.
	fmt.Printf("Ishlanmoqda: %s - %s (%d UZS)\n",
		order.OrderID, order.Product, order.Amount)
	return nil
}

func main() {
	reader := kafka.NewReader(kafka.ReaderConfig{
		Brokers:  []string{"localhost:9092"},
		Topic:    "orders",
		GroupID:  "order-processor-manual",
		MinBytes: 1,
		MaxBytes: 10e6,
		// Auto commit O'CHIRILGAN — biz o'zimiz commit qilamiz
	})
	defer reader.Close()

	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	sigChan := make(chan os.Signal, 1)
	signal.Notify(sigChan, syscall.SIGINT, syscall.SIGTERM)

	go func() {
		<-sigChan
		cancel()
	}()

	fmt.Println("Manual commit consumer ishga tushdi...")

	for {
		// FetchMessage — xabarni oladi, lekin COMMIT QILMAYDI
		msg, err := reader.FetchMessage(ctx)
		if err != nil {
			if ctx.Err() != nil {
				break
			}
			log.Printf("Fetch xatosi: %v", err)
			continue
		}

		var order Order
		if err := json.Unmarshal(msg.Value, &order); err != nil {
			log.Printf("Parse xatosi: %v", err)
			// Noto'g'ri format — commit qilib o'tkazib yuboramiz
			reader.CommitMessages(ctx, msg)
			continue
		}

		// Xabarni qayta ishlash
		if err := processOrder(order); err != nil {
			log.Printf("Ishlashda xato: %v (qayta uriniladi)", err)
			// COMMIT QILMAYMIZ — xabar qayta o'qiladi
			continue
		}

		// Muvaffaqiyatli ishlanganidan KEYIN commit
		if err := reader.CommitMessages(ctx, msg); err != nil {
			log.Printf("Commit xatosi: %v", err)
		}
	}
}
```

---

## 21-Dars: Retry va Xatolarni Boshqarish

### Retry Strategiyalari

```go
// retry.go
package main

import (
	"fmt"
	"math"
	"time"
)

// ExponentialBackoff — har safar kechikish ikki baravar oshadi
func ExponentialBackoff(attempt int, baseDelay time.Duration, maxDelay time.Duration) time.Duration {
	delay := time.Duration(math.Pow(2, float64(attempt))) * baseDelay
	if delay > maxDelay {
		delay = maxDelay
	}
	return delay
}

// Retry — berilgan funksiyani retry qiladi
func Retry(maxAttempts int, fn func() error) error {
	var lastErr error

	for attempt := 0; attempt < maxAttempts; attempt++ {
		if err := fn(); err != nil {
			lastErr = err
			delay := ExponentialBackoff(attempt, 100*time.Millisecond, 30*time.Second)
			fmt.Printf("Urinish %d muvaffaqiyatsiz: %v. %v dan keyin qayta uriniladi\n",
				attempt+1, err, delay)
			time.Sleep(delay)
			continue
		}
		return nil // Muvaffaqiyat
	}

	return fmt.Errorf("%d urinishdan keyin xato: %w", maxAttempts, lastErr)
}
```

### Dead Letter Queue (DLQ)

Bir necha marta urinib bo'lmaydigan xabarlarni alohida topicga yuborish:

```go
// dlq_consumer.go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"time"

	"github.com/segmentio/kafka-go"
)

type Order struct {
	OrderID string `json:"order_id"`
	UserID  int    `json:"user_id"`
	Product string `json:"product"`
	Amount  int    `json:"amount"`
}

type DLQMessage struct {
	OriginalTopic     string `json:"original_topic"`
	OriginalPartition int    `json:"original_partition"`
	OriginalOffset    int64  `json:"original_offset"`
	Error             string `json:"error"`
	Payload           string `json:"payload"`
	Attempts          int    `json:"attempts"`
	FailedAt          string `json:"failed_at"`
}

func main() {
	// Asosiy consumer
	reader := kafka.NewReader(kafka.ReaderConfig{
		Brokers: []string{"localhost:9092"},
		Topic:   "orders",
		GroupID: "order-processor-dlq",
	})
	defer reader.Close()

	// DLQ writer
	dlqWriter := &kafka.Writer{
		Addr:  kafka.TCP("localhost:9092"),
		Topic: "orders-dlq", // Dead Letter Queue topic
	}
	defer dlqWriter.Close()

	ctx := context.Background()
	maxRetries := 3

	for {
		msg, err := reader.FetchMessage(ctx)
		if err != nil {
			log.Printf("Fetch xatosi: %v", err)
			continue
		}

		var order Order
		if err := json.Unmarshal(msg.Value, &order); err != nil {
			// Parse bo'lmaydi — DLQ ga yuboramiz
			sendToDLQ(ctx, dlqWriter, msg, err, 0)
			reader.CommitMessages(ctx, msg)
			continue
		}

		// Retry bilan ishlash
		var lastErr error
		success := false
		for attempt := 1; attempt <= maxRetries; attempt++ {
			if err := processOrder(order); err != nil {
				lastErr = err
				log.Printf("Urinish %d/%d: %v", attempt, maxRetries, err)
				time.Sleep(time.Duration(attempt) * time.Second)
				continue
			}
			success = true
			break
		}

		if !success {
			// Barcha urinishlar muvaffaqiyatsiz — DLQ ga
			log.Printf("DLQ ga yuborilmoqda: %s", order.OrderID)
			sendToDLQ(ctx, dlqWriter, msg, lastErr, maxRetries)
		}

		reader.CommitMessages(ctx, msg)
	}
}

func processOrder(order Order) error {
	// Buyurtmani qayta ishlash logikasi
	fmt.Printf("Ishlanmoqda: %s\n", order.OrderID)
	return nil
}

func sendToDLQ(ctx context.Context, writer *kafka.Writer, msg kafka.Message, err error, attempts int) {
	dlqMsg := DLQMessage{
		OriginalTopic:     msg.Topic,
		OriginalPartition: msg.Partition,
		OriginalOffset:    msg.Offset,
		Error:             err.Error(),
		Payload:           string(msg.Value),
		Attempts:          attempts,
		FailedAt:          time.Now().Format(time.RFC3339),
	}

	value, _ := json.Marshal(dlqMsg)
	writer.WriteMessages(ctx, kafka.Message{
		Key:   msg.Key,
		Value: value,
	})
}
```

---

## 22-Dars: Real Microservice Misoli — Auth → Kafka → Email/SMS

### Arxitektura

```
┌──────────────┐     user.registered     ┌───────────┐
│  Auth        │────────────────────────→│           │
│  Service     │     user.logged_in      │   KAFKA   │
│  (:8080)     │────────────────────────→│           │
└──────────────┘                         │  Topics:  │
                                         │  - users  │
       HTTP                              └───────────┘
    ┌──────┐                                │     │
    │Client│                                │     │
    └──────┘                                ▼     ▼
                                    ┌─────────┐ ┌──────────┐
                                    │  Email  │ │   SMS    │
                                    │ Service │ │  Service │
                                    └─────────┘ └──────────┘
```

### Umumiy Strukturalar (shared/events.go)

```go
// ~/kafka-course/microservices/shared/events.go
package shared

import "time"

// EventType — voqea turlari
type EventType string

const (
	UserRegistered EventType = "user.registered"
	UserLoggedIn   EventType = "user.logged_in"
	UserUpdated    EventType = "user.updated"
)

// UserEvent — foydalanuvchi voqeasi
type UserEvent struct {
	EventType EventType `json:"event_type"`
	UserID    int       `json:"user_id"`
	Email     string    `json:"email"`
	Phone     string    `json:"phone"`
	Name      string    `json:"name"`
	Timestamp time.Time `json:"timestamp"`
	Metadata  map[string]string `json:"metadata,omitempty"`
}
```

### Auth Service (Producer)

```go
// ~/kafka-course/microservices/auth-service/main.go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	"time"

	"github.com/segmentio/kafka-go"
)

// UserEvent strukturasi
type UserEvent struct {
	EventType string            `json:"event_type"`
	UserID    int               `json:"user_id"`
	Email     string            `json:"email"`
	Phone     string            `json:"phone"`
	Name      string            `json:"name"`
	Timestamp time.Time         `json:"timestamp"`
	Metadata  map[string]string `json:"metadata,omitempty"`
}

// RegisterRequest — ro'yxatdan o'tish so'rovi
type RegisterRequest struct {
	Email    string `json:"email"`
	Phone    string `json:"phone"`
	Name     string `json:"name"`
	Password string `json:"password"`
}

var kafkaWriter *kafka.Writer
var userIDCounter = 0

func main() {
	// Kafka writer yaratish
	kafkaWriter = &kafka.Writer{
		Addr:         kafka.TCP("localhost:9092"),
		Topic:        "users",
		Balancer:     &kafka.Hash{},
		RequiredAcks: kafka.RequireAll,
		BatchTimeout: 10 * time.Millisecond,
	}
	defer kafkaWriter.Close()

	// HTTP endpointlar
	http.HandleFunc("/register", handleRegister)
	http.HandleFunc("/login", handleLogin)

	fmt.Println("Auth Service ishga tushdi: http://localhost:8080")
	log.Fatal(http.ListenAndServe(":8080", nil))
}

func handleRegister(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodPost {
		http.Error(w, "Faqat POST", http.StatusMethodNotAllowed)
		return
	}

	var req RegisterRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		http.Error(w, "Noto'g'ri JSON", http.StatusBadRequest)
		return
	}

	// Foydalanuvchini bazaga yozish (soddalashtirilgan)
	userIDCounter++
	userID := userIDCounter
	log.Printf("Yangi foydalanuvchi ro'yxatdan o'tdi: %s (ID: %d)", req.Name, userID)

	// Kafkaga voqea yuborish
	event := UserEvent{
		EventType: "user.registered",
		UserID:    userID,
		Email:     req.Email,
		Phone:     req.Phone,
		Name:      req.Name,
		Timestamp: time.Now(),
		Metadata: map[string]string{
			"ip":         r.RemoteAddr,
			"user_agent": r.UserAgent(),
		},
	}

	if err := publishEvent(event); err != nil {
		log.Printf("Kafka xatosi: %v", err)
		// Xabar yuborilmasa ham, foydalanuvchi ro'yxatdan o'tgan
		// Email keyinroq yuboriladi (yoki manually)
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(map[string]interface{}{
		"user_id": userID,
		"message": "Ro'yxatdan o'tish muvaffaqiyatli",
	})
}

func handleLogin(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodPost {
		http.Error(w, "Faqat POST", http.StatusMethodNotAllowed)
		return
	}

	// Login logikasi (soddalashtirilgan)
	userID := 1
	log.Printf("Foydalanuvchi tizimga kirdi: ID %d", userID)

	event := UserEvent{
		EventType: "user.logged_in",
		UserID:    userID,
		Timestamp: time.Now(),
		Metadata: map[string]string{
			"ip": r.RemoteAddr,
		},
	}

	publishEvent(event)

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(map[string]string{
		"token":   "jwt-token-example",
		"message": "Tizimga kirish muvaffaqiyatli",
	})
}

func publishEvent(event UserEvent) error {
	value, err := json.Marshal(event)
	if err != nil {
		return fmt.Errorf("marshal xatosi: %w", err)
	}

	return kafkaWriter.WriteMessages(context.Background(),
		kafka.Message{
			Key:   []byte(fmt.Sprintf("user-%d", event.UserID)),
			Value: value,
		},
	)
}
```

### Email Service (Consumer)

```go
// ~/kafka-course/microservices/email-service/main.go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"os"
	"os/signal"
	"syscall"

	"github.com/segmentio/kafka-go"
)

type UserEvent struct {
	EventType string `json:"event_type"`
	UserID    int    `json:"user_id"`
	Email     string `json:"email"`
	Name      string `json:"name"`
}

func main() {
	reader := kafka.NewReader(kafka.ReaderConfig{
		Brokers: []string{"localhost:9092"},
		Topic:   "users",
		GroupID: "email-service",
	})
	defer reader.Close()

	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	sigChan := make(chan os.Signal, 1)
	signal.Notify(sigChan, syscall.SIGINT, syscall.SIGTERM)
	go func() {
		<-sigChan
		cancel()
	}()

	fmt.Println("Email Service ishga tushdi...")

	for {
		msg, err := reader.FetchMessage(ctx)
		if err != nil {
			if ctx.Err() != nil {
				break
			}
			continue
		}

		var event UserEvent
		if err := json.Unmarshal(msg.Value, &event); err != nil {
			log.Printf("Parse xatosi: %v", err)
			reader.CommitMessages(ctx, msg)
			continue
		}

		switch event.EventType {
		case "user.registered":
			sendWelcomeEmail(event)
		case "user.logged_in":
			sendLoginNotification(event)
		default:
			log.Printf("Noma'lum voqea turi: %s", event.EventType)
		}

		reader.CommitMessages(ctx, msg)
	}
}

func sendWelcomeEmail(event UserEvent) {
	// Haqiqiy email yuborish logikasi (SMTP, SendGrid va h.k.)
	fmt.Printf("[EMAIL] Xush kelibsiz! → %s (%s)\n", event.Name, event.Email)
	fmt.Printf("  Mavzu: Xush kelibsiz, %s!\n", event.Name)
	fmt.Printf("  Matn: Siz muvaffaqiyatli ro'yxatdan o'tdingiz.\n")
	fmt.Println()
}

func sendLoginNotification(event UserEvent) {
	fmt.Printf("[EMAIL] Tizimga kirish → User ID: %d\n", event.UserID)
	fmt.Printf("  Mavzu: Yangi kirish aniqlandi\n")
	fmt.Println()
}
```

### SMS Service (Consumer)

```go
// ~/kafka-course/microservices/sms-service/main.go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"os"
	"os/signal"
	"syscall"

	"github.com/segmentio/kafka-go"
)

type UserEvent struct {
	EventType string `json:"event_type"`
	UserID    int    `json:"user_id"`
	Phone     string `json:"phone"`
	Name      string `json:"name"`
}

func main() {
	reader := kafka.NewReader(kafka.ReaderConfig{
		Brokers: []string{"localhost:9092"},
		Topic:   "users",
		GroupID: "sms-service", // BOSHQA group — email-service bilan mustaqil
	})
	defer reader.Close()

	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	sigChan := make(chan os.Signal, 1)
	signal.Notify(sigChan, syscall.SIGINT, syscall.SIGTERM)
	go func() {
		<-sigChan
		cancel()
	}()

	fmt.Println("SMS Service ishga tushdi...")

	for {
		msg, err := reader.FetchMessage(ctx)
		if err != nil {
			if ctx.Err() != nil {
				break
			}
			continue
		}

		var event UserEvent
		if err := json.Unmarshal(msg.Value, &event); err != nil {
			reader.CommitMessages(ctx, msg)
			continue
		}

		switch event.EventType {
		case "user.registered":
			sendWelcomeSMS(event)
		default:
			// Faqat registratsiya uchun SMS yuboramiz
		}

		reader.CommitMessages(ctx, msg)
	}
}

func sendWelcomeSMS(event UserEvent) {
	// Haqiqiy SMS yuborish (Eskiz, Playmobile API va h.k.)
	fmt.Printf("[SMS] → %s: Xush kelibsiz, %s! Ro'yxatdan muvaffaqiyatli o'tdingiz.\n",
		event.Phone, event.Name)
}
```

### Barcha Servislarni Ishga Tushirish

```bash
# Terminal 1: Auth Service
cd ~/kafka-course/microservices/auth-service
go run main.go

# Terminal 2: Email Service
cd ~/kafka-course/microservices/email-service
go run main.go

# Terminal 3: SMS Service
cd ~/kafka-course/microservices/sms-service
go run main.go

# Terminal 4: Test so'rov yuborish
curl -X POST http://localhost:8080/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "ali@example.com",
    "phone": "+998901234567",
    "name": "Ali Valiyev",
    "password": "secret123"
  }'

# Auth Service logi:
#   Yangi foydalanuvchi ro'yxatdan o'tdi: Ali Valiyev (ID: 1)

# Email Service logi:
#   [EMAIL] Xush kelibsiz! → Ali Valiyev (ali@example.com)
#     Mavzu: Xush kelibsiz, Ali Valiyev!
#     Matn: Siz muvaffaqiyatli ro'yxatdan o'tdingiz.

# SMS Service logi:
#   [SMS] → +998901234567: Xush kelibsiz, Ali Valiyev! ...
```

---

# PHASE 5 — Production Kafka

---

## 23-Dars: Kafka ni Systemd Service Sifatida Ishga Tushirish

### Systemd Service Fayl Yaratish

```bash
sudo nano /etc/systemd/system/kafka.service
```

```ini
[Unit]
Description=Apache Kafka Server (KRaft Mode)
Documentation=https://kafka.apache.org
After=network.target

[Service]
Type=simple
User=kafka
Group=kafka
Environment="JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64"
Environment="KAFKA_HEAP_OPTS=-Xmx1G -Xms1G"
ExecStart=/opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/kraft/server.properties
ExecStop=/opt/kafka/bin/kafka-server-stop.sh
Restart=on-failure
RestartSec=10
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

### Kafka Foydalanuvchisi Yaratish

```bash
# kafka foydalanuvchisi yaratish
sudo useradd -r -s /bin/false kafka

# Papkalarga ruxsat berish
sudo chown -R kafka:kafka /opt/kafka
sudo chown -R kafka:kafka /tmp/kraft-combined-logs
# Yoki production uchun alohida papka:
sudo mkdir -p /var/lib/kafka/data
sudo chown -R kafka:kafka /var/lib/kafka
```

### Service ni Boshqarish

```bash
# Systemd ni qayta yuklash
sudo systemctl daemon-reload

# Kafka ni ishga tushirish
sudo systemctl start kafka

# Holat tekshirish
sudo systemctl status kafka

# Boot da avtomatik ishga tushish
sudo systemctl enable kafka

# To'xtatish
sudo systemctl stop kafka

# Qayta ishga tushirish
sudo systemctl restart kafka

# Loglarni ko'rish
sudo journalctl -u kafka -f
```

### Production Konfiguratsiya Tavsiyalari

```properties
# /opt/kafka/config/kraft/server.properties

# === ASOSIY ===
process.roles=broker,controller
node.id=1
controller.quorum.voters=1@broker1:9093,2@broker2:9093,3@broker3:9093

# === NETWORK ===
listeners=PLAINTEXT://:9092,CONTROLLER://:9093
advertised.listeners=PLAINTEXT://broker1.example.com:9092
num.network.threads=8
num.io.threads=16
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600

# === SAQLASH ===
log.dirs=/var/lib/kafka/data
num.partitions=6
default.replication.factor=3
min.insync.replicas=2
log.retention.hours=168
log.segment.bytes=1073741824
log.retention.check.interval.ms=300000

# === PERFORMANCE ===
num.replica.fetchers=4
replica.lag.time.max.ms=30000
unclean.leader.election.enable=false
auto.create.topics.enable=false  # Production da avtomatik topic yaratishni O'CHIRING

# === XOTIRA ===
# KAFKA_HEAP_OPTS orqali (service faylda)
# Tavsiya: Umumiy RAM ning 1/3 qismi (max 6 GB)
# Qolgan RAM → OS page cache uchun
```

---

## 24-Dars: Monitoring (Kuzatuv)

### Muhim Metrikalar

| Metrika | Tavsif | Ogohlantirish Chegarasi |
|---------|--------|------------------------|
| **UnderReplicatedPartitions** | ISR da bo'lmagan partitionlar | > 0 |
| **OfflinePartitionsCount** | Leader yo'q partitionlar | > 0 |
| **ActiveControllerCount** | Aktiv controller soni | != 1 |
| **Consumer Lag** | O'qilmagan xabarlar soni | Servisga qarab |
| **RequestsPerSec** | Soniyada so'rovlar | Trendi kuzating |
| **BytesInPerSec** | Kiritish tezligi | Tarmoq chegarasiga yaqin |
| **BytesOutPerSec** | Chiqarish tezligi | Tarmoq chegarasiga yaqin |

### Consumer Lag Monitoring

```bash
# Consumer lag ni tekshirish
kafka-consumer-groups.sh --describe \
  --group order-processor \
  --bootstrap-server localhost:9092

# Natija:
# GROUP           TOPIC    PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
# order-processor orders   0          1000            1050            50
# order-processor orders   1          980             1020            40
# order-processor orders   2          990             990             0

# LAG = LOG-END-OFFSET - CURRENT-OFFSET
# LAG oshib borayotgan bo'lsa → Consumer sekin ishlamoqda!
```

### JMX Orqali Metrikalar

```bash
# Kafka ni JMX bilan ishga tushirish
export JMX_PORT=9999
kafka-server-start.sh /opt/kafka/config/kraft/server.properties
```

### Prometheus + Grafana Bilan Monitoring

**1. JMX Exporter O'rnatish:**

```bash
# JMX Exporter yuklab olish
wget https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.20.0/jmx_prometheus_javaagent-0.20.0.jar \
  -O /opt/kafka/jmx_prometheus_javaagent.jar
```

**2. JMX Konfiguratsiya:**

```yaml
# /opt/kafka/jmx-exporter-config.yml
lowercaseOutputName: true
rules:
  - pattern: kafka.server<type=(.+), name=(.+)><>Value
    name: kafka_server_$1_$2
    type: GAUGE
  - pattern: kafka.server<type=(.+), name=(.+), topic=(.+)><>Value
    name: kafka_server_$1_$2
    type: GAUGE
    labels:
      topic: $3
  - pattern: kafka.controller<type=(.+), name=(.+)><>Value
    name: kafka_controller_$1_$2
    type: GAUGE
```

**3. Kafka ni JMX Exporter Bilan Ishga Tushirish:**

```bash
export KAFKA_OPTS="-javaagent:/opt/kafka/jmx_prometheus_javaagent.jar=7071:/opt/kafka/jmx-exporter-config.yml"
kafka-server-start.sh /opt/kafka/config/kraft/server.properties
```

**4. Metrikalarni Tekshirish:**

```bash
curl http://localhost:7071/metrics | grep kafka
```

### Go da Consumer Lag Monitoring

```go
// monitor.go — Consumer lag ni Go da tekshirish
package main

import (
	"context"
	"fmt"
	"log"
	"time"

	"github.com/segmentio/kafka-go"
)

func checkConsumerLag(brokers []string, topic string, groupID string) {
	client := &kafka.Client{
		Addr: kafka.TCP(brokers...),
	}

	// Topic partition ma'lumotlarini olish
	partitions, err := client.Metadata(context.Background(), &kafka.MetadataRequest{
		Topics: []string{topic},
	})
	if err != nil {
		log.Printf("Metadata xatosi: %v", err)
		return
	}

	for _, t := range partitions.Topics {
		for _, p := range t.Partitions {
			// Har bir partition uchun latest offset
			conn, _ := kafka.DialLeader(context.Background(), "tcp",
				brokers[0], topic, p.ID)
			lastOffset, _ := conn.ReadLastOffset()
			conn.Close()

			fmt.Printf("Topic: %s, Partition: %d, Latest Offset: %d\n",
				topic, p.ID, lastOffset)
		}
	}
}

func main() {
	brokers := []string{"localhost:9092"}
	ticker := time.NewTicker(30 * time.Second)

	for range ticker.C {
		fmt.Println("=== Lag Tekshiruv ===")
		checkConsumerLag(brokers, "orders", "order-processor")
		fmt.Println()
	}
}
```

---

## 25-Dars: Topic Dizayn Best Practices

### Topic Nomlash

```
Yaxshi:
  orders.created
  payments.completed
  user.registered
  inventory.updated

Yomon:
  topic1
  my-topic
  data
  events
```

**Tavsiya etilgan format:**
```
<domen>.<voqea>

Misollar:
  order.created
  order.updated
  order.cancelled
  payment.initiated
  payment.completed
  payment.failed
  user.registered
  user.updated
  notification.email.sent
  notification.sms.sent
```

### Topic vs Event Type

**Yondashuv 1: Har bir voqea alohida topic**
```
Topic: order-created
Topic: order-updated
Topic: order-cancelled

Afzallik: Har bir consumerni alohida subscribe qilish oson
Kamchilik: Juda ko'p topic, boshqarish qiyin
```

**Yondashuv 2: Bitta topic, voqea turi ichida (TAVSIYA)**
```
Topic: orders
  → {"event_type": "order.created", ...}
  → {"event_type": "order.updated", ...}
  → {"event_type": "order.cancelled", ...}

Afzallik: Kamroq topic, tartib saqlanadi (bir xil key bilan)
Kamchilik: Consumer keraksiz voqealarni ham o'qiydi
```

### Partition Soni Tanlash

```
Qoidalar:

1. Partition soni ≥ Consumer soni (max parallellik)
2. Ko'proq partition = ko'proq parallellik, LEKIN ko'proq resurs
3. Boshlash uchun: 3-6 partition
4. Yuqori yuklanish uchun: 12-30 partition
5. Partition sonini KAMAYTIRISH mumkin EMAS

Hisoblash formulasi:
  Target throughput: 100 MB/s
  Bitta partition throughput: ~10 MB/s
  Partition soni: 100 / 10 = 10 ta

Amaliy tavsiya:
  Kichik tizim:   3 partition
  O'rta tizim:    6 partition
  Katta tizim:    12-30 partition
  Juda katta:     50+ partition
```

### Key Strategiyasi

```
Maqsad: Bitta entity (buyurtma, foydalanuvchi) ning barcha voqealari
        bitta partitionda bo'lsin (tartib uchun)

Buyurtma voqealari → Key: order_id
  order.created (order-123)  → P2
  order.paid (order-123)     → P2 (bir xil partition!)
  order.shipped (order-123)  → P2 (bir xil partition!)

Foydalanuvchi voqealari → Key: user_id
  user.registered (user-67)  → P1
  user.updated (user-67)     → P1

DIQQAT: Key null bo'lsa, xabar tasodifiy partitionga tushadi!
```

### Schema Evolution (Sxema O'zgarishi)

```json
// V1 — boshlang'ich
{
  "event_type": "order.created",
  "order_id": "ORD-001",
  "amount": 100000
}

// V2 — yangi maydon qo'shildi (backward compatible)
{
  "event_type": "order.created",
  "order_id": "ORD-001",
  "amount": 100000,
  "currency": "UZS"     // ← Yangi maydon (ixtiyoriy)
}

// V3 — yana yangi maydon (backward compatible)
{
  "event_type": "order.created",
  "order_id": "ORD-001",
  "amount": 100000,
  "currency": "UZS",
  "version": 3           // ← Versiya raqami qo'shish foydali
}
```

**Qoidalar:**
1. Yangi maydon qo'shish — OK (consumer bilmasa e'tiborsiz qoldiradi)
2. Mavjud maydonni O'CHIRISH — XAVFLI (eski consumerlar buziladi)
3. Maydon turini O'ZGARTIRISH — XAVFLI
4. Versiya maydoni qo'shish — Tavsiya etiladi

---

## 26-Dars: Docker Compose Bilan Kafka Cluster

### Bitta Broker (Development Uchun)

```yaml
# ~/kafka-course/docker/docker-compose.yml
version: '3.8'

services:
  kafka:
    image: apache/kafka:3.7.0
    container_name: kafka
    ports:
      - "9092:9092"
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_LOG_DIRS: /var/lib/kafka/data
      CLUSTER_ID: MkU3OEVBNTcwNTJENDM2Qk
    volumes:
      - kafka-data:/var/lib/kafka/data

volumes:
  kafka-data:
```

```bash
# Ishga tushirish
cd ~/kafka-course/docker
docker compose up -d

# Tekshirish
docker compose logs kafka

# Topic yaratish
docker exec kafka /opt/kafka/bin/kafka-topics.sh \
  --create --topic orders --partitions 3 \
  --bootstrap-server localhost:9092

# To'xtatish
docker compose down
```

### 3 Broker Cluster (Production-Like)

```yaml
# ~/kafka-course/docker/docker-compose-cluster.yml
version: '3.8'

services:
  kafka1:
    image: apache/kafka:3.7.0
    container_name: kafka1
    ports:
      - "9092:9092"
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka1:9093,2@kafka2:9093,3@kafka3:9093
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 3
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 2
      KAFKA_DEFAULT_REPLICATION_FACTOR: 3
      KAFKA_MIN_INSYNC_REPLICAS: 2
      KAFKA_LOG_DIRS: /var/lib/kafka/data
      CLUSTER_ID: MkU3OEVBNTcwNTJENDM2Qk
    volumes:
      - kafka1-data:/var/lib/kafka/data
    networks:
      - kafka-net

  kafka2:
    image: apache/kafka:3.7.0
    container_name: kafka2
    ports:
      - "9093:9092"
    environment:
      KAFKA_NODE_ID: 2
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9093
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka1:9093,2@kafka2:9093,3@kafka3:9093
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 3
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 2
      KAFKA_DEFAULT_REPLICATION_FACTOR: 3
      KAFKA_MIN_INSYNC_REPLICAS: 2
      KAFKA_LOG_DIRS: /var/lib/kafka/data
      CLUSTER_ID: MkU3OEVBNTcwNTJENDM2Qk
    volumes:
      - kafka2-data:/var/lib/kafka/data
    networks:
      - kafka-net

  kafka3:
    image: apache/kafka:3.7.0
    container_name: kafka3
    ports:
      - "9094:9092"
    environment:
      KAFKA_NODE_ID: 3
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9094
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka1:9093,2@kafka2:9093,3@kafka3:9093
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 3
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 2
      KAFKA_DEFAULT_REPLICATION_FACTOR: 3
      KAFKA_MIN_INSYNC_REPLICAS: 2
      KAFKA_LOG_DIRS: /var/lib/kafka/data
      CLUSTER_ID: MkU3OEVBNTcwNTJENDM2Qk
    volumes:
      - kafka3-data:/var/lib/kafka/data
    networks:
      - kafka-net

  # Kafka UI — klasterni brauzerda boshqarish
  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    container_name: kafka-ui
    ports:
      - "8080:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local-cluster
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka1:9092,kafka2:9092,kafka3:9092
    depends_on:
      - kafka1
      - kafka2
      - kafka3
    networks:
      - kafka-net

volumes:
  kafka1-data:
  kafka2-data:
  kafka3-data:

networks:
  kafka-net:
    driver: bridge
```

### Cluster Ni Ishga Tushirish va Tekshirish

```bash
# Ishga tushirish
docker compose -f docker-compose-cluster.yml up -d

# Barcha konteynerlarni ko'rish
docker compose -f docker-compose-cluster.yml ps

# Topic yaratish (replication-factor=3)
docker exec kafka1 /opt/kafka/bin/kafka-topics.sh \
  --create --topic orders \
  --partitions 6 \
  --replication-factor 3 \
  --bootstrap-server kafka1:9092

# Topic ma'lumotini ko'rish
docker exec kafka1 /opt/kafka/bin/kafka-topics.sh \
  --describe --topic orders \
  --bootstrap-server kafka1:9092

# Natija:
# Topic: orders  PartitionCount: 6  ReplicationFactor: 3
#   Partition: 0  Leader: 1  Replicas: 1,2,3  Isr: 1,2,3
#   Partition: 1  Leader: 2  Replicas: 2,3,1  Isr: 2,3,1
#   Partition: 2  Leader: 3  Replicas: 3,1,2  Isr: 3,1,2
#   ...

# Kafka UI ni brauzerda ochish
# http://localhost:8080
```

### Broker Tushish Testini O'tkazish

```bash
# Broker 2 ni to'xtatish
docker stop kafka2

# Topic holatini tekshirish
docker exec kafka1 /opt/kafka/bin/kafka-topics.sh \
  --describe --topic orders \
  --bootstrap-server kafka1:9092

# Partition 1 da Leader o'zgardi! (2 → 1 yoki 3)
# Isr dan 2 chiqdi

# Broker 2 ni qayta ishga tushirish
docker start kafka2

# Biroz kutib, tekshirish — ISR qayta tiklanadi
docker exec kafka1 /opt/kafka/bin/kafka-topics.sh \
  --describe --topic orders \
  --bootstrap-server kafka1:9092
```

---

# PHASE 6 — Advanced Mavzular

---

## 27-Dars: Kafka + ClickHouse (Analitika)

### Arxitektura

```
Servislar → Kafka → ClickHouse → Grafana/Metabase

Misol:
  Buyurtma servisi → orders topic → ClickHouse jadvali → Dashboard
```

### ClickHouse Kafka Engine

ClickHouse bevosita Kafkadan ma'lumot o'qiy oladi:

```sql
-- 1. Kafka Engine jadvali (bufer)
CREATE TABLE orders_kafka (
    order_id String,
    user_id UInt32,
    product String,
    amount UInt64,
    currency String,
    created_at DateTime
) ENGINE = Kafka()
SETTINGS
    kafka_broker_list = 'localhost:9092',
    kafka_topic_list = 'orders',
    kafka_group_name = 'clickhouse-consumer',
    kafka_format = 'JSONEachRow',
    kafka_num_consumers = 3;

-- 2. Asosiy jadval (saqlash)
CREATE TABLE orders (
    order_id String,
    user_id UInt32,
    product String,
    amount UInt64,
    currency String,
    created_at DateTime,
    inserted_at DateTime DEFAULT now()
) ENGINE = MergeTree()
ORDER BY (created_at, order_id);

-- 3. Materialized View (avtomatik ko'chirish)
CREATE MATERIALIZED VIEW orders_mv TO orders AS
SELECT * FROM orders_kafka;

-- Endi Kafkaga xabar kelsa, avtomatik ClickHouse ga tushadi!

-- 4. So'rovlar
-- Kunlik buyurtmalar
SELECT
    toDate(created_at) AS kun,
    count() AS buyurtmalar_soni,
    sum(amount) AS umumiy_summa
FROM orders
GROUP BY kun
ORDER BY kun DESC;

-- Eng ko'p sotilgan mahsulotlar
SELECT
    product,
    count() AS soni,
    sum(amount) AS umumiy
FROM orders
GROUP BY product
ORDER BY soni DESC
LIMIT 10;
```

---

## 28-Dars: Kafka Logging Uchun

### Markazlashtirilgan Log Tizimi

```
┌────────────┐
│ Service A  │──log──┐
└────────────┘       │
                     ▼
┌────────────┐   ┌───────┐   ┌──────────────┐
│ Service B  │──→│ KAFKA │──→│ Elasticsearch│──→ Kibana
└────────────┘   │ logs  │   └──────────────┘
                 │ topic │
┌────────────┐   └───────┘   ┌──────────────┐
│ Service C  │──log──┘   └──→│  S3/MinIO    │ (arxiv)
└────────────┘               └──────────────┘
```

### Go da Kafka Logger

```go
// kafka_logger.go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"os"
	"time"

	"github.com/segmentio/kafka-go"
)

type LogLevel string

const (
	INFO  LogLevel = "INFO"
	WARN  LogLevel = "WARN"
	ERROR LogLevel = "ERROR"
	DEBUG LogLevel = "DEBUG"
)

type LogEntry struct {
	Timestamp   time.Time         `json:"timestamp"`
	Level       LogLevel          `json:"level"`
	Service     string            `json:"service"`
	Message     string            `json:"message"`
	TraceID     string            `json:"trace_id,omitempty"`
	Error       string            `json:"error,omitempty"`
	Metadata    map[string]string `json:"metadata,omitempty"`
	Hostname    string            `json:"hostname"`
}

type KafkaLogger struct {
	writer  *kafka.Writer
	service string
	host    string
}

func NewKafkaLogger(brokers []string, service string) *KafkaLogger {
	host, _ := os.Hostname()
	return &KafkaLogger{
		writer: &kafka.Writer{
			Addr:         kafka.TCP(brokers...),
			Topic:        "logs",
			Balancer:     &kafka.LeastBytes{},
			BatchTimeout: 50 * time.Millisecond,
			RequiredAcks: kafka.RequireOne, // Loglar uchun acks=1 yetarli
			Async:        true,              // Asinxron yuborish (tez)
		},
		service: service,
		host:    host,
	}
}

func (l *KafkaLogger) log(level LogLevel, msg string, meta map[string]string) {
	entry := LogEntry{
		Timestamp: time.Now().UTC(),
		Level:     level,
		Service:   l.service,
		Message:   msg,
		Metadata:  meta,
		Hostname:  l.host,
	}

	value, _ := json.Marshal(entry)
	l.writer.WriteMessages(context.Background(),
		kafka.Message{
			Key:   []byte(l.service),
			Value: value,
		},
	)
}

func (l *KafkaLogger) Info(msg string, meta ...map[string]string) {
	var m map[string]string
	if len(meta) > 0 {
		m = meta[0]
	}
	l.log(INFO, msg, m)
}

func (l *KafkaLogger) Error(msg string, err error, meta ...map[string]string) {
	var m map[string]string
	if len(meta) > 0 {
		m = meta[0]
	}
	entry := LogEntry{
		Timestamp: time.Now().UTC(),
		Level:     ERROR,
		Service:   l.service,
		Message:   msg,
		Error:     err.Error(),
		Metadata:  m,
		Hostname:  l.host,
	}
	value, _ := json.Marshal(entry)
	l.writer.WriteMessages(context.Background(),
		kafka.Message{
			Key:   []byte(l.service),
			Value: value,
		},
	)
}

func (l *KafkaLogger) Close() {
	l.writer.Close()
}

func main() {
	logger := NewKafkaLogger([]string{"localhost:9092"}, "order-service")
	defer logger.Close()

	logger.Info("Servis ishga tushdi")
	logger.Info("Buyurtma qabul qilindi", map[string]string{
		"order_id": "ORD-001",
		"user_id":  "67",
	})
	logger.Error("Bazaga yozishda xato", fmt.Errorf("connection refused"), map[string]string{
		"order_id": "ORD-002",
	})

	time.Sleep(time.Second) // Async yuborishni kutish
}
```

---

## 29-Dars: Event Sourcing

### Event Sourcing Nima?

Odatda biz **holatni** (state) saqlaymiz:
```
Hisob balansi: 500,000 UZS
```

Event Sourcing da **voqealarni** saqlaymiz:
```
1. account.created     → balans: 0
2. money.deposited     → +1,000,000
3. money.withdrawn     → -300,000
4. money.withdrawn     → -200,000
                         ─────────
                         = 500,000  (voqealardan hisoblangan)
```

### Kafka Bilan Event Sourcing

```
┌──────────┐    voqealar    ┌───────┐    replay    ┌──────────┐
│  Buyruq   │──────────────→│ KAFKA │─────────────→│  Read    │
│ (Command) │               │       │              │  Model   │
└──────────┘               │       │              │ (Query)  │
                            └───────┘              └──────────┘
                                │
                                ▼
                          Voqealar abadiy
                          saqlanadi (log
                          compaction bilan)
```

### Go da Event Sourcing Misoli

```go
// event_sourcing.go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"time"

	"github.com/segmentio/kafka-go"
)

// Event turlari
type EventType string

const (
	AccountCreated  EventType = "account.created"
	MoneyDeposited  EventType = "money.deposited"
	MoneyWithdrawn  EventType = "money.withdrawn"
)

// Event strukturasi
type AccountEvent struct {
	EventType EventType `json:"event_type"`
	AccountID string    `json:"account_id"`
	Amount    int64     `json:"amount,omitempty"`
	Balance   int64     `json:"balance,omitempty"`
	Timestamp time.Time `json:"timestamp"`
	Version   int       `json:"version"`
}

// Account holati (voqealardan quriladi)
type Account struct {
	ID      string
	Balance int64
	Version int
}

// Voqealardan holatni qayta qurish
func ReplayEvents(events []AccountEvent) *Account {
	account := &Account{}

	for _, event := range events {
		switch event.EventType {
		case AccountCreated:
			account.ID = event.AccountID
			account.Balance = 0
			account.Version = event.Version

		case MoneyDeposited:
			account.Balance += event.Amount
			account.Version = event.Version

		case MoneyWithdrawn:
			account.Balance -= event.Amount
			account.Version = event.Version
		}
	}

	return account
}

func main() {
	writer := &kafka.Writer{
		Addr:  kafka.TCP("localhost:9092"),
		Topic: "accounts",
	}
	defer writer.Close()

	accountID := "ACC-001"

	// Voqealar ketma-ketligi
	events := []AccountEvent{
		{EventType: AccountCreated, AccountID: accountID, Timestamp: time.Now(), Version: 1},
		{EventType: MoneyDeposited, AccountID: accountID, Amount: 1000000, Timestamp: time.Now(), Version: 2},
		{EventType: MoneyWithdrawn, AccountID: accountID, Amount: 300000, Timestamp: time.Now(), Version: 3},
		{EventType: MoneyDeposited, AccountID: accountID, Amount: 500000, Timestamp: time.Now(), Version: 4},
		{EventType: MoneyWithdrawn, AccountID: accountID, Amount: 200000, Timestamp: time.Now(), Version: 5},
	}

	// Kafkaga yuborish
	for _, event := range events {
		value, _ := json.Marshal(event)
		writer.WriteMessages(context.Background(), kafka.Message{
			Key:   []byte(accountID),
			Value: value,
		})
	}

	// Voqealardan holatni qayta qurish
	account := ReplayEvents(events)
	fmt.Printf("Hisob: %s\n", account.ID)
	fmt.Printf("Balans: %d UZS\n", account.Balance) // 1,000,000
	fmt.Printf("Versiya: %d\n", account.Version)

	// Natija: Balans: 1000000 UZS
	// (1,000,000 + 500,000 - 300,000 - 200,000 = 1,000,000)
}
```

### Event Sourcing Afzalliklari

| Afzallik | Tushuntirish |
|----------|-------------|
| **To'liq tarix** | Hamma o'zgarish saqlanadi, audit uchun ajoyib |
| **Qayta qurish** | Istalgan vaqt nuqtasiga "qaytish" mumkin |
| **Debug oson** | Nima bo'lganini aniq ko'rish mumkin |
| **Replay** | Yangi servisni eski voqealardan to'ldirish |

### Event Sourcing Kamchiliklari

| Kamchilik | Tushuntirish |
|-----------|-------------|
| **Murakkablik** | Oddiy CRUD dan murakkabroq |
| **So'rovlar** | "Hozirgi holat" uchun barcha voqealarni replay qilish kerak |
| **Disk hajmi** | Ko'p voqea ko'p joy egallaydi |
| **Schema evolution** | Eski voqealar yangi formatga mos kelmasligi mumkin |

---

## 30-Dars: Real-World Xatolar va Ulardan Qochish

### Xato 1: Auto-Create Topics

```
MUAMMO: Producer noto'g'ri topic nomiga xabar yuboradi,
Kafka avtomatik yangi topic yaratadi.

orders → OK
ordres → Kafka yangi "ordres" topic yaratdi! (typo)

YECHIM:
auto.create.topics.enable=false
```

### Xato 2: Partition Soni Noto'g'ri

```
MUAMMO: 3 partition yaratib, 10 ta consumer ishga tushirish
Natija: 7 ta consumer bo'sh turadi

YECHIM: Partition soni ≥ Consumer soni
```

### Xato 3: Katta Xabarlar

```
MUAMMO: 10 MB lik xabar yuborish
Standart limit: 1 MB

YECHIM:
1. Xabarni kichiklashtiring (faqat kerakli ma'lumotni yuboring)
2. Katta fayllarni S3/MinIO ga yuklang, Kafkaga faqat URL yuboring
3. Agar shart bo'lsa:
   message.max.bytes=10485760 (broker)
   max.request.size=10485760 (producer)
   max.partition.fetch.bytes=10485760 (consumer)
```

### Xato 4: Consumer Lag O'sib Ketishi

```
MUAMMO: Consumer xabarlarni qayta ishlash sekin, lag oshib boradi

TEKSHIRISH:
kafka-consumer-groups.sh --describe --group my-group \
  --bootstrap-server localhost:9092

YECHIMLAR:
1. Consumer sonini ko'paytiring (partition sonigacha)
2. Partition sonini ko'paytiring
3. Consumer kodini optimizatsiya qiling
4. Batch processing ishlatish
5. Asinxron qayta ishlash
```

### Xato 5: Offset Commit Strategiyasi

```
MUAMMO: Auto-commit + sekin qayta ishlash = xabar yo'qolishi

Consumer offset=100 da turadi
Auto-commit 100 ni commit qildi
Consumer hali 95-99 ni ishlayapti
Consumer crash bo'ldi
Qayta ishga tushdi — offset 100 dan boshlaydi
95-99 YOQOLDI!

YECHIM: Manual commit ishlatish
```

### Xato 6: Replication Factor = 1

```
MUAMMO: Bitta broker tushsa, ma'lumot yo'qoladi

YECHIM:
Production da DOIMO:
  replication.factor = 3
  min.insync.replicas = 2
```

### Xato 7: Key Ishlatmaslik

```
MUAMMO: Bir xil buyurtmaning voqealari turli partitionlarga tushadi
  order.created (order-123)  → P0
  order.paid (order-123)     → P2  (boshqa partition!)
  order.shipped (order-123)  → P1  (boshqa partition!)

Consumer tartibsiz o'qiydi: shipped → created → paid 😱

YECHIM: Doimo key ishlatish
  Key: "order-123" → Barcha voqealar P0 ga tushadi
```

### Xato 8: Monitoring Yo'q

```
MUAMMO: Kafka ishlayaptimi bilmaysiz, muammo paydo bo'lganda
sezmay qolasiz

YECHIM: Monitoring o'rnating
  - Consumer lag kuzating
  - Under-replicated partitions
  - Disk hajmi
  - Broker holati
  - Ogohlantirish (alerting) sozlang
```

### Production Checklist

```
☐ auto.create.topics.enable = false
☐ replication.factor = 3
☐ min.insync.replicas = 2
☐ acks = all (muhim topiclar uchun)
☐ enable.idempotence = true
☐ unclean.leader.election.enable = false
☐ Consumer group ID to'g'ri nomlangan
☐ Key strategiyasi aniqlangan
☐ DLQ (Dead Letter Queue) mavjud
☐ Consumer lag monitoring ishlayapti
☐ Disk hajmi kuzatilmoqda
☐ Backup/DR rejasi bor
☐ Topic retention sozlangan
☐ Schema versiyalash mavjud
☐ Graceful shutdown implemented
```

---

# XULOSA

```
┌─────────────────────────────────────────────────────────────┐
│                 KAFKA BILIM XARITASI                        │
│                                                             │
│  Phase 1: Asosiy tushunchalar                               │
│    ✓ Producer, Consumer, Topic, Partition, Broker, Offset  │
│    ✓ Event-Driven Architecture                              │
│    ✓ Consumer Groups                                        │
│                                                             │
│  Phase 2: Terminal amaliyot                                 │
│    ✓ Java + Kafka o'rnatish (KRaft)                        │
│    ✓ Topic yaratish/boshqarish                             │
│    ✓ Xabar yuborish/o'qish                                │
│    ✓ Partition va offsetlarni tekshirish                    │
│                                                             │
│  Phase 3: Arxitektura ustasi                                │
│    ✓ Consumer group chuqur                                 │
│    ✓ Rebalancing                                           │
│    ✓ Replikatsiya va fault tolerance                       │
│    ✓ Exactly once semantics                                │
│                                                             │
│  Phase 4: Go + Kafka                                        │
│    ✓ Producer/Consumer Go da                               │
│    ✓ Retry va DLQ                                          │
│    ✓ Real microservice (Auth→Kafka→Email/SMS)              │
│                                                             │
│  Phase 5: Production                                        │
│    ✓ Systemd service                                       │
│    ✓ Monitoring                                            │
│    ✓ Topic dizayn                                          │
│    ✓ Docker Compose cluster                                │
│                                                             │
│  Phase 6: Advanced                                          │
│    ✓ ClickHouse analitika                                  │
│    ✓ Markazlashtirilgan logging                            │
│    ✓ Event Sourcing                                        │
│    ✓ Real-world xatolar                                    │
│                                                             │
│  Keyingi qadamlar:                                          │
│    → Kafka Streams                                         │
│    → Kafka Connect                                         │
│    → Schema Registry (Avro/Protobuf)                       │
│    → Kafka bilan gRPC                                      │
│    → Multi-datacenter replikatsiya (MirrorMaker)           │
└─────────────────────────────────────────────────────────────┘
```

---

> **Muallif eslatmasi:** Bu kurs sizga Kafkaning asosiy tushunchalaridan production darajasigacha yo'l ko'rsatadi. Har bir mavzuni amalda sinab ko'ring. Faqat o'qish yetarli emas — terminal oching, kod yozing, xatolardan o'rganing. Omad!
