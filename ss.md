# রেফারেন্স শু প্রশ্ন–উত্তর (Q/A) সিস্টেম — বুঝুন ও Postman দিয়ে টেস্ট করুন

তোমার সার্ভারের ঠিকানার পরে বেস URL (Postman এ `{{baseUrl}}` ভেরিয়েবল রাখতে পারো):

**প্রশ্ন ক্যাটালগ (অ্যাডমিন CRUD):**

```text
{{baseUrl}}/v3/reference-shoe/question-category
```

**জুতা (রেফারেন্স শু) + ওই জুতার উত্তর বাঁধা:**

```text
{{baseUrl}}/v3/reference-shoe
```

`question-category` এর সব রাউটে **ADMIN** লগিন লাগে (`verifyUser("ADMIN")`)। সাধারণত:

```http
Authorization: Bearer <তোমার_টোকেন>
```

নিচের JSON ব্লকগুলো Postman এ **Body → raw → JSON** দিয়ে পাঠাবে, ছবি লাগলে **form-data**।

---

## ১) পুরো গল্পটা মাথায় রাখো — "কীভাবে কাজ করে"

### ১.১ ডাটা মডেল — এক নজরে

| টেবিল / ধারণা | সহজ বাংলায় |
|----------------|-------------|
| `question_category` | প্রশ্নের "ফোল্ডার" বা গ্রুপ। চাইলে `parentId` দিয়ে সাব-ক্যাটাগরি। |
| `question` | একটা স্ক্রিন/ধাপ — টাইটেল `text`, বাধ্যতামূলক কিনা `isRequired`, একাধিক উত্তর `isMultiSelect`, ক্রম `order` (বেশি ব্যবহার **রুট** প্রশ্নের জন্য)। |
| `question_option` | ওই প্রশ্নের নিচে একটা উত্তরের অপশন (যেমন "Allrounder")। |
| `question_unlocked_by_option` | **কোন উত্তর বেছে নিলে পরের কোন প্রশ্ন খুলবে** — এটা many-to-many। একই পরের প্রশ্ন খুলতে পারে **অনেক অপশন**; একটা অপশন থেকে পারে **অনেক.follow-up** যেতে। |
| DAG নিয়ম | ফ্লোতে **চক্র (cycle)** চলবে না — A→B→C→A বাছাই করলে সিস্টেম রিজেক্ট করে। **ভাই-ফাটানো (একটা প্রশ্নে অনেক গন্তব্য)** আর **অনেক রাস্তা এক গন্তব্যে** — দুটোই ঠিক আছে। |

### ১.২ রুট প্রশ্ন vs ফলো-আপ

- **রুট প্রশ্ন:** ডাটাবেজে `question_unlocked_by_option` এ তার জন্য **কোনো রো নেই**। মানে কোনো আগের উত্তর ছাড়াই ক্যাটাগরিতে ওটা শুরুতে দেখানো যায়।
- **ফলো-আপ প্রশ্ন:** কমপক্ষে **একটা** আনলক রো আছে। মানে ইউজার আগের প্রশ্নে নির্দিষ্ট এক বা একাধিক অপশন বেছে নিলে পরেই ওটা দেখানোর কথা।

### ১.৩ ক্রম (কে আগে কে পরে)

- **রুট প্রশ্নগুলোর মধ্যে সাজানো:** মূলত `question.order`।
- **একটি নির্দিষ্ট উত্তর বেচে নেওয়ার পর কোন ফলো-আপ আগে:** `question_unlocked_by_option.sortOrder` (ঐ অপশন থেকে ওই প্রশ্নের edge)।

### ১.৪ জুতা ক্রিয়েট করতে `reference_shoe_questions` — এটা **গাছ (tree)**, লিস্ট নয়

প্রোডাক্ট তৈরি করার সময় তুমি যে JSON পাঠাও:

- ওটা **সরল একডাইমেনশন লিস্ট** নয়, **নেস্টেড ট্রি**।
- **শুধু সেরার সারি (top level):** শুধুই **ক্যাটালগের রুট** প্রশ্ন (`unlock_option_ids` খালি)।
- **পরের ধাপ:** অবশ্যই বাবার ভেতরে **`children`** অ্যারেতে। বাবার **`optionIds`** = বাবা প্রশ্নে জুতার জন্য **কোন উত্তরগুলো বেছে নিয়েছ**। **ছেলে প্রশ্নটা** ক্যাটালগে যেন এমন থাকে যে, বাবার বাচা অপশনগুলোর **অন্তত একটা** দিয়ে সেটা আনলক হয়।
- **একই `questionId` এক গাছে দুবার আসা চলবে না**।

তিনটা অবজেক্টের ফ্ল্যাট অ্যারে পাঠালে, সেটা **কেবল তখনই OK** যদি তিনটাই আসলে ক্যাটাগরিতে **রুট** প্রশ্ন হয়। স্টার্ট → ফলো-আপ → আরেকটা ধাপ — এই ফ্লো মানে **নেস্ট** করতে হবে।

**মাল্টিপার্ট ফর্ম (ছবি সহ জুতা):** `reference_shoe_questions` ফিল্ডে পুরো অ্যারেটা **স্ট্রিং হিসেবে JSON** দাও (সার্ভার `JSON.parse` করে)।

---

## ২) Postman — ক্যাটাগরি (তৈরি / দেখা / মুছা / আপডেট)

### ২.১ নতুন ক্যাটাগরি

| | |
|--|--|
| মেথড | `POST` |
| URL | `{{baseUrl}}/v3/reference-shoe/question-category/create` |
| বডি | `form-data` (ছবি চাইলে) |

ফিল্ড: `name` (লাগবে), `description`, `parentId` (সাবক্যাট হলে), `file` কী নামে আপলোড সেটা রাউট অনুযায়ী — এখানে `image`।

উত্তর উদাহরণ `201`:

```json
{
  "success": true,
  "message": "Question category created successfully",
  "data": {
    "id": "clx…",
    "name": "Laufschuhe",
    "description": null,
    "image": "https://…",
    "parentId": null,
    "createdAt": "2026-05-14T…",
    "updatedAt": "2026-05-14T…"
  }
}
```

### ২.২ ক্যাটাগরি লিস্ট

`GET` `{{baseUrl}}/v3/reference-shoe/question-category/get-all`

- কোয়েরি **না** দিলে: শুধু **রুট** ক্যাটাগরি।
- `?categoryId=প্যারেন্ট_আইডি` দিলে: ওই প্যারেন্টের **সরাসরি ছেলে** ক্যাটাগরি।

উত্তর উদাহরণ `200`:

```json
{
  "success": true,
  "message": "Question categories (roots)",
  "data": [
    {
      "id": "clx…",
      "name": "Laufschuhe",
      "_children": 2,
      "_question": 5
    }
  ]
}
```

### ২.৩ ক্যাটাগরি মুছে ফেলা (নিচের সাবট্রি সহ)

`DELETE` `.../delete-question-category`

```json
{ "categoryId": "clx_question_category_id" }
```

### ২.৪ ক্যাটাগরি আপডেট

`PATCH` `.../update-question-category/:id` — `multipart`, `name` / `description` / `parentId` / `image`।

---

## ৩) Postman — ক্যাটালগে প্রশ্ন ও অপশন

### ৩.১ প্রশ্ন তৈরি (নেস্টেড JSON ট্রি)

`POST` `.../create-question` — `Content-Type: application/json`

- `categoryId` (আবশ্যক)
- `questions[]` — প্রতিটিতে `text` (আবশ্যক), `objective`, `isRequired`, `isMultiSelect`
- **ম্যানুয়ালি বড় গাছে একই চাইল্ড বাঁধতে:** টপ লেভেল আইটেমে `unlock_option_ids` (ইতিমধ্যে থাকা `question_option` id গুলো) বা পুরনো স্টাইল `parent_question_option_id` (একটা id)
- প্রতি অপশনে `options[]`: `text`, `objective`, ভেতরে **`questions`** দিয়ে ফলো-আপ (নেস্টেড ভেতরে `unlock_option_ids` পাঠিও না)

**অনুরোধ বডির উদাহরণ:**

```json
{
  "categoryId": "clx_category_id",
  "questions": [
    {
      "text": "Startfrage",
      "isRequired": true,
      "isMultiSelect": false,
      "options": [
        {
          "text": "Allrounder",
          "questions": [
            {
              "text": "Folgefrage Allrounder",
              "options": [{ "text": "Option 1" }]
            }
          ]
        },
        { "text": "Schnell" }
      ]
    }
  ]
}
```

**সফল উত্তর উদাহরণ `201`:** (`follow_up_questions` = পরের ধাপ; `unlock_option_ids` = কোন উত্তর দিয়ে খুলেছে)

```json
{
  "success": true,
  "message": "Question and options created",
  "data": [
    {
      "id": "clx…",
      "categoryId": "clx…",
      "unlock_option_ids": [],
      "order": 1,
      "text": "Startfrage",
      "options": [
        {
          "id": "clx_opt…",
          "follow_up_questions": [
            {
              "id": "clx…",
              "unlock_option_ids": ["clx_opt…"],
              "text": "Folgefrage Allrounder",
              "options": [{ "id": "…", "follow_up_questions": [] }]
            }
          ]
        }
      ]
    }
  ]
}
```

### ৩.২ গ্রাফ চেক (ডাটাবেজে লেখে না — দ্রুত)

`POST` `.../validate-questions-graph`

বডি **ঠিক `create-questions-graph` এর মতো**। HTTP সাধারণত **২০০**; দেখো `valid: true/false`।

- `valid: true` → পরে নিরাপদে `create-questions-graph` চালাতে পারো।
- `valid: false` → `message` এ কারণ (সাইকেল, ডিসকানেক্ট, ইত্যাদি)।

**টেস্ট বডি উদাহরণ:**

```json
{
  "categoryId": "clx_category_id_optional",
  "questions": [
    {
      "id": "q1",
      "text": "Q1",
      "options": [{ "text": "A", "nextQuestionIds": ["q2"] }]
    },
    {
      "id": "q2",
      "text": "Q2",
      "options": [{ "text": "B", "nextQuestionIds": [] }]
    }
  ]
}
```

**ঠিক আছে → `200`:**

```json
{
  "success": true,
  "valid": true,
  "message": "OK — acyclic connected DAG",
  "data": { "questionCount": 2, "rootCount": 1 }
}
```

**ভুল → এখনও `200`, তবে `valid: false`:**

```json
{
  "success": true,
  "valid": false,
  "message": "Directed cycle detected in question flow …"
}
```

### ৩.৩ ফ্ল্যাট গ্রাফ তৈরি (`nextQuestionIds`)

`POST` `.../create-questions-graph`

প্রতিটা নোডের জন্য তুমি অস্থায়ী `id` দিও (যেমন `"q1"`, `"q2"`); অপশনে `nextQuestionIds: ["q2"]` দিয়ে এজ বাঁধো। চক্র চলবে না। একাধিক অপশন একই `q3` এ আসতে পারে (ছবির মতো many-to-one)।

**বডি উদাহরণ:**

```json
{
  "categoryId": "clx_category_id",
  "questions": [
    {
      "id": "q1",
      "text": "Root",
      "order": 1,
      "isRequired": true,
      "isMultiSelect": false,
      "options": [{ "text": "Go to Q2", "nextQuestionIds": ["q2"] }]
    },
    {
      "id": "q2",
      "text": "Next",
      "options": [{ "text": "End", "nextQuestionIds": [] }]
    }
  ]
}
```

**উত্তর `201`:**

```json
{
  "success": true,
  "message": "Created graph (2 questions)",
  "data": {
    "questionTempIdToId": { "q1": "clx_real…", "q2": "clx_real…" },
    "trees": [{ "id": "clx_real…", "options": [{ "follow_up_questions": [] }] }]
  }
}
```

### ৩.৪ প্রশ্ন লোড

`GET` `.../get-question?categoryId=...`

- সাধারণত ফ্ল্যাট `data.questions[]`, প্রতিটিতে `unlock_option_ids`, অপশনে `_hasNextQuestion`।
- `format=nested` বা `view=tree` → গাছের আকার।
- `optionId=...` → ওই উত্তরের নিচের সাবট্রি।
- `fixOrders=1` → ধীর, DB তে ক্রম ঠিক করে।

সম্পূর্ণ নরমালাইজড লিস্টের জন্য আলাদা এন্ডপয়েন্ট: `GET` `.../get-questions-normalized?categoryId=...`

**নরমালাইজড উত্তরের উদাহরণ `200`:**

```json
{
  "success": true,
  "message": "Success",
  "data": {
    "questions": [
      {
        "id": "clx…",
        "categoryId": "clx…",
        "unlock_option_ids": [],
        "order": 1,
        "text": "Startfrage",
        "isRequired": true,
        "isMultiSelect": false,
        "options": [
          {
            "id": "clx…",
            "questionId": "clx…",
            "text": "Allrounder",
            "_hasNextQuestion": 1
          }
        ]
      }
    ]
  }
}
```

### ৩.৫ একটা প্রশ্ন

`GET` `.../get-single-question/:id`

**উত্তর উদাহরণ `200`:**

```json
{
  "success": true,
  "message": "Success",
  "data": {
    "id": "clx…",
    "unlock_option_ids": ["clx_option…"],
    "order": 2,
    "text": "Folgefrage",
    "options": []
  }
}
```

### ৩.৬ অপশন যোগ

`POST` `.../add-options-to-question`

```json
{
  "questionId": "clx_question_id",
  "text": "Neue Antwort",
  "objective": null
}
```

### ৩.৭ অপশন আপডেট (টেক্সট / ফলো-আপ রিঅর্ডার)

`PATCH` `.../update-options`

```json
{
  "optionId": "clx_question_option_id",
  "text": "Updated label",
  "nestedQuestionId": "clx_follow_up_question_id",
  "newOrder": 2
}
```

### ৩.৮ অপশন মুছা

`DELETE` `.../delete-option` — বডিতে `optionIds` অ্যারে।

### ৩.৯ প্রশ্ন আপডেট (ক্রম / টেক্সট)

`PATCH` `.../update-question`

```json
{
  "questionId": "clx_question_id",
  "newOrder": 2,
  "optionId": "clx_context_option_id_optional",
  "text": "optional patch"
}
```

একই প্রশ্নের **একাধিক `unlock_option_ids`** থাকলে কোন লিস্টে রিঅর্ডার করছ সেটা বোঝাতে **`optionId`** দরকার।

### ৩.১০ প্রশ্ন মুছা

`DELETE` `.../delete-question` — `{ "questionId": "..." }`

---

## ৪) Postman — জুতায় Q/A বাঁধা

### ৪.১ ক্রিয়েট

`POST` `{{baseUrl}}/v3/reference-shoe/create` — `multipart` + `images`

`reference_shoe_questions` = ট্রি, ফর্মে স্ট্রিং JSON:

```json
[
  {
    "questionId": "clx_root_question_id",
    "optionIds": ["clx_selected_option_id"],
    "children": [
      {
        "questionId": "clx_next_question_id",
        "optionIds": ["clx_another_option_id"],
        "children": []
      }
    ]
  }
]
```

নিয়ম মনে রাখো:

- গাছ জুড়ে প্রতি `questionId` **একবার**।
- শুধু **রুট** টপ লেভেলে।
- ছেলের ক্যাটালগ আনলক সেট **বাবার `optionIds` এর সাথে মিল খায়**।

### ৪.২ একটা জুতা দেখা (`mcq` সহ)

`GET` `{{baseUrl}}/v3/reference-shoe/get-single/:id`

`data.mcq` = উত্তর দেওয়া গাছ; নোডে `unlock_option_ids`, `selectedOptionIds`, `children`।

**উত্তরের অংশবিশেষ `200`:**

```json
{
  "success": true,
  "data": {
    "id": "clx…",
    "name": "Shoe A",
    "question_category_id": "clx…",
    "question_category": { "id": "clx…", "name": "…" },
    "mcq": [
      {
        "reference_shoe_question_id": "clx…",
        "id": "clx_question_id",
        "order": 1,
        "unlock_option_ids": [],
        "text": "…",
        "selectedOptionIds": ["clx…"],
        "options": [{ "id": "clx…", "text": "…", "isCorrect": true }],
        "children": []
      }
    ]
  }
}
```

### ৪.৩ আপডেট

`PATCH` `{{baseUrl}}/v3/reference-shoe/update/:id` — ক্রিয়েটের মতোই প্যাটার্ন।

---

## ৫) Postman এনভায়রনমেন্ট দ্রুত সেটআপ

| ভেরিয়েবল | উদাহরণ |
|-----------|--------|
| `baseUrl` | `http://localhost:3000` |
| `adminToken` | JWT |

হেডার:

```http
Authorization: Bearer {{adminToken}}
```

---

## ৬) সমস্যা হলে (খুব সাধারণ)

| মেসেজ / লক্ষণ | সাধারণত মানে |
|----------------|---------------|
| only category-root … top level | ফলো-আপটা টাপ লেভেলে দিয়েছ — `children` এ ঢুকাও। |
| must be unlocked by one of the parent's … | ছেলে প্রশ্ন ক্যাটালগে বাবার বাচা `optionIds` দিয়ে আনলক হয় না — গ্রাফ বা উত্তর মিলাও। |
| Duplicate questionId in tree | একই প্রশ্ন আইডি দুবার ব্যবহার। |
| Directed cycle detected | `create-questions-graph` / ভ্যালিডেট — `nextQuestionIds` এ লুপ কেটে দাও। |

---

এই ডক `module/v3/reference_shoe/question_category/` ও `reference_shoe.controllers.ts` এর বর্তমান কোড মিলিয়ে লেখা — `question_unlocked_by_option` জUNCTION + DAG ভ্যালিডেশন অনুযায়ী।
