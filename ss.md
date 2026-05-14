# রেফারেন্স শু প্রশ্ন–উত্তর (Q/A) সিস্টেম — বুঝুন ও Postman দিয়ে টেস্ট করুন

Postman এ **Environment** খুলে দুটো ভেরিয়েবল রাখো (নাম ঠিক এইরকম হলে নিচের ডেমো গুলো একদম কপি–পেস্ট হবে):

| ভেরিয়েবল | উদাহরণ | মন্তব্য |
|-----------|--------|--------|
| `_baseUrl` | `http://localhost:3000/api/` | শেষে `/` দিলে URL জোড়া সহজ (`{{_baseUrl}}v3/...`) |
| `adminToken` | Admin JWT | লগিন API থেকে |

সব **ADMIN** রাউটে:

```http
Authorization: Bearer {{adminToken}}
```

**প্রশ্ন ক্যাটালগ (অ্যাডমিন):**

```text
{{_baseUrl}}v3/reference-shoe/question-category/
```

**জুতা (রেফারেন্স শু):**

```text
{{_baseUrl}}v3/reference-shoe/
```

`_baseUrl` এ যদি শেষে `/` না থাকে, তবে ম্যানুয়ালি `{{_baseUrl}}/v3/...` লিখো।

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
| URL | `{{_baseUrl}}v3/reference-shoe/question-category/create` |
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

`GET` `{{_baseUrl}}v3/reference-shoe/question-category/get-all`

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

`DELETE` `{{_baseUrl}}v3/reference-shoe/question-category/delete-question-category`

```json
{ "categoryId": "clx_question_category_id" }
```

### ২.৪ ক্যাটাগরি আপডেট

`PATCH` `{{_baseUrl}}v3/reference-shoe/question-category/update-question-category/:id` — `multipart`, `name` / `description` / `parentId` / `image`।

---

## ৩) Postman — ক্যাটালগে প্রশ্ন ও অপশন

নিচের ব্লকগুলো **একই প্যাটার্ন**: শিরোনাম → মেথড → পূর্ণ URL → হেডার → বডি → সাধারণ রেসপন্স। Postman এ **New Request** খুলে **Body → raw → JSON** বেছে নিও (যেখানে JSON লিখা আছে)।

**প্রথমে ধরো:** ক্যাটাগরি তুমি আগেই তৈরি করেছ (`create` দিয়ে `categoryId` আছে) — যেমন `Massschafterstellung` বা `Komplettfertigung` এর মতো নাম হলেও **API তে লাগে আসল `id` স্ট্রিং**।

---

### DEMO — ৩.১ নেস্টেড ট্রি দিয়ে প্রশ্ন তৈরি (`create-question`)

ইউজ কেস: একটা রুট প্রশ্ন + অপশনের ভেতর ফলো-আপ প্রশ্ন।

```
POST
{{_baseUrl}}v3/reference-shoe/question-category/create-question
```

**Header:** `Authorization: Bearer {{adminToken}}`  
**Header:** `Content-Type: application/json`

**Body:**

```json
{
  "categoryId": "clx_category_id_from_create_category",
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

নোট:

- `categoryId` বাধ্য।
- টপ লেভেল `questions[]` তে `text` বাধ্য। অপশনের ভেতর **`questions`** দিয়ে ফলো-আপ যোগ।
- বড় গাছ একসাথে বাঁধতে টপ আইটেমে `unlock_option_ids` (ইতিমধ্যে থাকা `question_option` id) বা পুরনো একক `parent_question_option_id` চলে।

**Response `201` (আংশিক):**

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

---

### DEMO — ৩.২ গ্রাফ ভ্যালিডেট (`validate-questions-graph` — DB তে লেখে না)

ইউজ কেস: `create-questions-graph` এর আগে চেক করবে সাইকেল / সংযোগ।

```
POST
{{_baseUrl}}v3/reference-shoe/question-category/validate-questions-graph
```

**Header:** `Authorization: Bearer {{adminToken}}`

**Body:** (ঠিক `create-questions-graph` এর মতো ফ্ল্যাট গ্রাফ)

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

**Response `200` ঠিক আছে:**

```json
{
  "success": true,
  "valid": true,
  "message": "OK — acyclic connected DAG",
  "data": { "questionCount": 2, "rootCount": 1 }
}
```

**Response `200` ভুল গ্রাফ (সাইকেল ইত্যাদি):** এখনও HTTP 200, কিন্তু `valid: false`।

```json
{
  "success": true,
  "valid": false,
  "message": "Directed cycle detected in question flow …"
}
```

---

### DEMO — ৩.৩ ফ্ল্যাট গ্রাফ তৈরি (`create-questions-graph`, `nextQuestionIds`)

ইউজ কেস: অস্থায়ী `id` (`q1`, `q2`) + অপশনে `nextQuestionIds` দিয়ে এজ। চক্র চলবে না; অনেক অপশন একই টার্গেটে মিলতে পারে।

```
POST
{{_baseUrl}}v3/reference-shoe/question-category/create-questions-graph
```

**Header:** `Authorization: Bearer {{adminToken}}`  
**Header:** `Content-Type: application/json`

**Body:**

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

**Response `201` (আংশিক):**

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

---

### DEMO — ৩.৪ ক্যাটাগরির সব প্রশ্ন পড়া (`get-question`)

```
GET
{{_baseUrl}}v3/reference-shoe/question-category/get-question?categoryId=clx_category_id
```

**Header:** `Authorization: Bearer {{adminToken}}`

কোয়েরি ঐচ্ছিক:

- `format=nested` বা `view=tree` → গাছ।
- `optionId=...` → ওই অপশনের নিচের সাবট্রি।
- `fixOrders=1` → ধীর, অর্ডার রিপেয়ার।

**Response `200`:** সাধারণত ফ্ল্যাট `data.questions[]`, প্রতিটিতে `unlock_option_ids`, অপশনে `_hasNextQuestion`।

---

### DEMO — ৩.৫ নরমালাইজড পূর্ণ লিস্ট (`get-questions-normalized`)

```
GET
{{_baseUrl}}v3/reference-shoe/question-category/get-questions-normalized?categoryId=clx_category_id
```

**Header:** `Authorization: Bearer {{adminToken}}`

**Response `200` (উদাহরণ):**

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

---

### DEMO — ৩.৬ একটা প্রশ্ন (`get-single-question`)

```
GET
{{_baseUrl}}v3/reference-shoe/question-category/get-single-question/clx_question_id
```

**Header:** `Authorization: Bearer {{adminToken}}`

**Response `200` (উদাহরণ):**

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

---

### DEMO — ৩.৭ অপশন যোগ (`add-options-to-question`)

```
POST
{{_baseUrl}}v3/reference-shoe/question-category/add-options-to-question
```

**Header:** `Authorization: Bearer {{adminToken}}`  
**Header:** `Content-Type: application/json`

**Body:**

```json
{
  "questionId": "clx_question_id",
  "text": "Neue Antwort",
  "objective": null
}
```

**Response `200` (উদাহরণ):**

```json
{
  "success": true,
  "message": "Option added to question",
  "data": {
    "id": "clx_new_option_id",
    "questionId": "clx_question_id",
    "text": "Neue Antwort",
    "objective": null
  }
}
```

---

### DEMO — ৩.৮ অপশন আপডেট (`update-options`)

```
PATCH
{{_baseUrl}}v3/reference-shoe/question-category/update-options
```

**Header:** `Authorization: Bearer {{adminToken}}`  
**Header:** `Content-Type: application/json`

**Body:**

```json
{
  "optionId": "clx_question_option_id",
  "text": "Updated label",
  "nestedQuestionId": "clx_follow_up_question_id",
  "newOrder": 2
}
```

---

### DEMO — ৩.৯ অপশন মুছা (`delete-option`)

```
DELETE
{{_baseUrl}}v3/reference-shoe/question-category/delete-option
```

**Header:** `Authorization: Bearer {{adminToken}}`  
**Header:** `Content-Type: application/json`

**Body:**

```json
{
  "optionIds": ["clx_question_option_id"]
}
```

**Response `200` (উদাহরণ):**

```json
{
  "success": true,
  "message": "Options deleted",
  "data": ["clx_question_option_id"]
}
```

---

### DEMO — ৩.১০ প্রশ্ন আপডেট — ক্রম / টেক্সট (`update-question`)

```
PATCH
{{_baseUrl}}v3/reference-shoe/question-category/update-question
```

**Header:** `Authorization: Bearer {{adminToken}}`  
**Header:** `Content-Type: application/json`

**Body:**

```json
{
  "questionId": "clx_question_id",
  "newOrder": 2,
  "optionId": "clx_context_option_id_optional",
  "text": "optional patch"
}
```

একই প্রশ্নের একাধিক আনলক পথ থাকলে কোন লিস্টে রিঅর্ডার করছ, তা বোঝাতে `optionId` দরকার হতে পারে।

---

### DEMO — ৩.১১ প্রশ্ন মুছা (`delete-question`)

```
DELETE
{{_baseUrl}}v3/reference-shoe/question-category/delete-question
```

**Header:** `Authorization: Bearer {{adminToken}}`  
**Header:** `Content-Type: application/json`

**Body:**

```json
{
  "questionId": "clx_question_id"
}
```

**Response `200` (উদাহরণ):**

```json
{
  "success": true,
  "message": "Question deleted",
  "data": "clx_question_id"
}
```

---

## ৪) Postman — জুতায় Q/A বাঁধা

### ৪.১ ক্রিয়েট

`POST` `{{_baseUrl}}v3/reference-shoe/create` — `multipart` + `images`

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

`GET` `{{_baseUrl}}v3/reference-shoe/get-single/:id`

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

`PATCH` `{{_baseUrl}}v3/reference-shoe/update/:id` — ক্রিয়েটের মতোই প্যাটার্ন।

---

## ৫) Postman এনভায়রনমেন্ট (আবার সংক্ষেপ)

উপরে **সেকশন শুরুতেই** `_baseUrl` ও `adminToken` টেবিল দেওয়া আছে। এখানে শুধু মনে করিয়ে:

| ভেরিয়েবল | উদাহরণ |
|-----------|--------|
| `_baseUrl` | `http://localhost:3000/api/` |
| `adminToken` | Admin JWT |

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
