# FeetF1rst API Guide

This guide documents all available API endpoints, core request fields, and how to use them.

Base URL (local): `http://127.0.0.1:8000`

## Authentication

- Most endpoints require JWT access token.
- Add this header for protected routes:
  - `Authorization: Bearer <access_token>`
- Token refresh endpoint:
  - `POST /api/token/refresh/`

Example:

```bash
curl -X GET "http://127.0.0.1:8000/api/products/" \
  -H "Authorization: Bearer <access_token>"
```

---

## Accounts API (`/api/users/`)

### `GET /api/users/`
- Auth: required
- Returns current authenticated user data list.

### `POST /api/users/signup/`
- Auth: public
- Body (common):
  - `email` (required)
  - `password` (required)
  - `name`, `phone`, `role`, `date_of_birth`, etc. (optional)

```bash
curl -X POST "http://127.0.0.1:8000/api/users/signup/" \
  -H "Content-Type: application/json" \
  -d '{
    "email":"user@example.com",
    "password":"Pass@1234",
    "name":"Test User"
  }'
```

### `POST /api/users/login/`
- Auth: public
- Body:
  - `email` (required)
  - `password` (required)
- Response includes `access` and `refresh` tokens.

### `POST /api/users/logout/`
- Auth: token user
- Body:
  - `refresh` (required)

### `PUT/PATCH/DELETE /api/users/update/`
- Auth: required
- Update current user profile.

### `POST /api/users/get-otp/`
- Auth: public
- Body:
  - `email` (required)
  - `task` (optional)

### `POST /api/users/verify-otp/`
- Auth: public
- Body:
  - `email` (required)
  - `otp_code` (required)
- Returns JWT tokens when valid.

### `POST /api/users/reset-password/`
- Auth: required
- Body:
  - `email` (required)
  - `new_password` (required)

### `POST /api/users/change-password/`
- Auth: required
- Body:
  - `old_password` (required)
  - `new_password` (required)
  - `confirm_password` (required)

### `GET/POST /api/users/addresses/`
- Auth: required
- POST fields:
  - `first_name`, `street_address`, `postal_code`, `city`, `phone_number`, `country` (required)
  - `last_name`, `address_line2`, `comments` (optional)

### `GET/PUT/PATCH/DELETE /api/users/addresses/me/`
- Auth: required
- Manage logged-in user's address.

### `POST /api/users/deletion-request/`
- Auth: required
- Body:
  - `reason` (JSON/list, optional)

### `POST /api/users/google/callback/`
- Auth: public
- Body:
  - `access_token` (Google token, required)

### `GET /api/users/paertners/`
- Auth: required
- Returns partner list with location/contact fields.

### `POST /api/users/verify-access/`
- Auth: public
- Body:
  - `access_token` (required)
  - `refresh_token` (optional fallback)

---

## Products API (`/api/products/`)

### `GET /api/products/`
- Auth: required
- Query params (optional):
  - `brandName`, `sub_category`, `gender`, `size_id`, `color_id`, `partner_id`, `match`
- Returns product listing (partner-aware).

### `GET /api/products/count/`
- Auth: required
- Optional query:
  - `sub_category`, `gender`

### `GET /api/products/<id>/`
- Auth: required
- Returns full product detail with images, features, sizes, qna, match data.

### `GET/POST /api/products/footscans/`
- Auth: required
- POST body:
  - `left_length`, `right_length`, `left_width`, `right_width` (required)
  - `left_arch_index`, `right_arch_index`, `left_heel_angle`, `right_heel_angle` (optional)

### `GET/PATCH /api/products/favorites/`
- Auth: required
- PATCH body:
  - `product_id` (required)
  - `action` = `add` or `remove` (required)

### `POST /api/products/qna-match/`
- Auth: required
- Body example:
```json
{
  "sub_category": "running-shoes",
  "questions": [
    { "question": "key_or_label", "answers": ["answer-1", "answer-2"] }
  ]
}
```

### `GET /api/products/suggestions/<product_id>/`
- Auth: required
- Returns similar/recommended products.

### Partner-only endpoints

These require partner-authenticated JWT.

### `GET /api/products/all/`
- List all catalog products for partner workflow.

### `GET /api/products/partner/`
- List partner's product variants/dashboard data.

### `GET /api/products/partner/<product_id>/`
- Single partner variant info.

### `PATCH /api/products/partner/<product_id>/<action>/`
- Actions: `add`, `update`, `del`/`remove`
- Common body fields:
  - `price`, `buy_price`, `eanc`
  - `warehouse_id`
  - `color` or `color_id`
  - `sizes` (list or dict format)
  - `online`, `local`, `is_active`

### `POST /api/products/partner/upload/`
- Multipart upload for partner stock file.
- Body:
  - `file` (required: csv/xls/xlsx)
  - `warehouse_id` (optional)

### `POST /api/products/partner/add-local/`
- Add local-only product variant.
- Typical body:
  - `name`, `brand`, `color`
  - optional `price`, `buy_price`, `warehouse`, `sizes`, `eanc`, `online`, `local`

### `GET/POST/PATCH/DELETE /api/products/partner/accessories/`
- Manage partner accessories.
- POST: create
- PATCH: update by `id`
- DELETE: delete by `id`

---

## Surveys API (`/api/surveys/`)

### `POST /api/surveys/onboarding-surveys/`
- Auth: required
- Body:
  - `discovery_question` (JSON list)
  - `interests` (JSON list)
  - `gender` (`man` or `woman`)
  - `foot_problems` (optional)

### `GET /api/surveys/onboarding-surveys/me/`
- Auth: required
- Get current user's survey.

---

## Contact API

### `POST /api/contactus/`
- Auth: required
- Body:
  - `name` (required)
  - `email` (required)
  - `subject` (required)
  - `message` (required)

---

## Brands API

### `GET /api/brands/`
- Auth: required
- Returns brand list (`name`, `image`).

---

## Other API routes (from core)

### FAQ / News
- `GET /api/faq/` (auth required)
- `GET /api/news/` (auth required)

### Orders / Dashboard
- `POST /api/create-order/` (auth required)
- `GET /api/dashboard/` (partner auth)
- `GET /api/orders/` (partner auth)
- `GET /api/orders/info/` (partner auth)
- `PUT/PATCH /api/orders/<pk>/` (partner auth)
- `GET /api/users/orders/` (auth required)

### Warehouse
- `GET/POST /api/warehouse/` (partner auth)
- `GET/PUT/PATCH/DELETE /api/warehouse/<pk>/` (partner auth)

### Finance
- `GET /api/partner/income/` (partner auth)
- `GET /api/partner/finance/` (partner auth)

### Cart
- `GET/POST /api/cart/` (auth required)
- `PATCH/DELETE /api/cart/<pk>/` (auth required)
- `POST /api/cart/clear/` (auth required)

### Payments/Webhook
- `POST /api/stripe-webhook/` (public webhook endpoint)
- `POST /api/token/refresh/` (JWT refresh)

---

## Quick usage flow

1. Signup: `POST /api/users/signup/`
2. Login: `POST /api/users/login/` and save `access`, `refresh`
3. Call protected APIs with `Authorization: Bearer <access>`
4. When access expires, refresh via `POST /api/token/refresh/`
5. Use product endpoints (`/api/products/...`) and partner routes based on user role.

---

## Notes

- Route `paertners` is spelled exactly like that in code: `/api/users/paertners/`.
- There is no `/api/token/` route currently registered in `core/urls.py`; login/otp/google endpoints return JWTs directly.
