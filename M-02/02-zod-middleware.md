# 🛡 Zod দিয়ে `req.body` Validation (Express + TypeScript)

Zod হলো একটি schema validation library।

এটা দিয়ে আমরা:

- Request body validate করতে পারি
- Type inference পেতে পারি
- Runtime validation করতে পারি
- Clean error message পাঠাতে পারি

---

# 📦 কেন `req.body` validate করা দরকার?

যদি frontend ভুল data পাঠায়:

```json
{
  "email": 123,
  "password": true
}
```

Validation ছাড়া:

❌ Backend crash করতে পারে
❌ Unexpected bug হতে পারে
❌ Security issue হতে পারে

Zod থাকলে:

✅ Invalid request reject হবে
✅ Clean error message যাবে
✅ Type safe হবে

---

# 📥 Installation

```bash
npm install zod
```

---

# 🧱 Step 1: Schema তৈরি করা

## 📁 `user.validation.ts`

```ts
import { z } from "zod";

export const createUserSchema = z.object({
  body: z.object({
    name: z.string().min(1, "Name is required"),

    email: z.string().email("Invalid email format"),

    password: z.string().min(6, "Password must be at least 6 characters"),

    role: z.enum(["ADMIN", "DOCTOR", "PATIENT"]),
  }),
});
```

---

# 🧠 এটা কী করছে?

```ts
z.object({
  body: z.object({
```

আমরা পুরো `req` না, শুধু `req.body` validate করছি।
তাই schema এর ভিতরে `body` রাখা হয়েছে।

---

### Field Explanation

| Field    | Validation           |
| -------- | -------------------- |
| name     | string & required    |
| email    | valid email          |
| password | minimum 6 characters |
| role     | enum type            |

---

# 🧩 Step 2: Validation Middleware তৈরি

## 📁 `validateRequest.ts`

```ts
import { Request, Response, NextFunction } from "express";
import { ZodSchema } from "zod";

const validateRequest = (schema: ZodSchema) => {
  return async (req: Request, res: Response, next: NextFunction) => {
    try {
      await schema.parseAsync({
        body: req.body,
      });

      next();
    } catch (error: any) {
      return res.status(400).json({
        success: false,
        message: "Validation Error",
        errors: error.errors,
      });
    }
  };
};

export default validateRequest;
```

---

# 🧠 এটা কী করছে?

### 1️⃣ `schema.parseAsync(...)`

Zod schema দিয়ে `req.body` validate করছে।

### 2️⃣ যদি valid হয়

```ts
next();
```

Route handler এ চলে যাবে।

### 3️⃣ যদি invalid হয়

```ts
res.status(400).json(...)
```

Clean validation error পাঠাবে।

---

# 🧪 Step 3: Route এ ব্যবহার

```ts
import express from "express";
import validateRequest from "../middlewares/validateRequest";
import { createUserSchema } from "./user.validation";

const router = express.Router();

router.post(
  "/create-user",
  validateRequest(createUserSchema),
  async (req, res) => {
    res.json({
      success: true,
      data: req.body,
    });
  },
);
```

---

# 🔎 এখন কী হবে?

### Valid Request:

```json
{
  "name": "John",
  "email": "john@gmail.com",
  "password": "123456",
  "role": "DOCTOR"
}
```

✅ Pass করবে।

---

### Invalid Request:

```json
{
  "name": "",
  "email": "wrong-email",
  "password": "123",
  "role": "INVALID"
}
```

❌ Response:

```json
{
  "success": false,
  "message": "Validation Error",
  "errors": [
    {
      "message": "Name is required",
      "path": ["body", "name"]
    },
    {
      "message": "Invalid email format",
      "path": ["body", "email"]
    }
  ]
}
```

---

# 🎯 Type Inference (Bonus 🔥)

Zod থেকে TypeScript type auto-generate করা যায়।

```ts
export type CreateUserInput = z.infer<typeof createUserSchema>["body"];
```

এখন তুমি controller এ ব্যবহার করতে পারো:

```ts
const payload: CreateUserInput = req.body;
```

✅ এখন পুরো type-safe।

---

# 🏗 Advanced Version (Global Error Handler compatible)

তুমি চাইলে validation error global error handler এ পাঠাতে পারো:

```ts
catch (error) {
  next(error);
}
```

এবং global error handler এ ZodError আলাদা handle করতে পারো।

---

# 💡 Best Practice

✔ Always validate req.body
✔ req.params ও req.query আলাদাভাবে validate করা যায়
✔ Enum ব্যবহার করো
✔ Custom error message দাও
✔ Global error handler এ Zod error handle করো

---

# 🔥 Summary

Zod দিয়ে:

- Runtime validation হয়
- Clean error পাওয়া যায়
- Type safety পাওয়া যায়
- Production ready API বানানো যায়

---
