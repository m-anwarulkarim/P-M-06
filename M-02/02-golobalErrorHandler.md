# 🌍 Global Error Handler (Express + TypeScript)

Global Error Handler হচ্ছে এমন একটি middleware যা অ্যাপ্লিকেশনের যেকোনো জায়গা থেকে আসা error এক জায়গায় handle করে।

এতে করে:

- একই ফরম্যাটে error response যায়
- কোড clean থাকে
- Production এ sensitive তথ্য leak হয় না
- Debugging সহজ হয়

---

# 📁 File Structure Example

```
src/
 ├── middlewares/
 │     └── globalErrorHandler.ts
 ├── utils/
 │     └── AppError.ts
 ├── app.ts
```

---

# 1️⃣ Custom Error Class (`AppError.ts`)

```ts
class AppError extends Error {
  public statusCode: number;

  constructor(statusCode: number, message: string) {
    super(message);
    this.statusCode = statusCode;

    Error.captureStackTrace(this, this.constructor);
  }
}

export default AppError;
```

## 🔎 কী করছে?

- Default Error কে extend করছে
- Custom `statusCode` যোগ করছে
- Stack trace maintain করছে

এখন আমরা যেকোনো জায়গায় এভাবে error throw করতে পারি:

```ts
throw new AppError(404, "User not found");
```

---

# 2️⃣ Global Error Handler Middleware

## 📁 `globalErrorHandler.ts`

```ts
import { Request, Response, NextFunction } from "express";
import status from "http-status";
import AppError from "../errorHelpers/AppError";

const globalErrorHandler = (
  error: any,
  req: Request,
  res: Response,
  next: NextFunction,
) => {
  let statusCode = status.INTERNAL_SERVER_ERROR;
  let message = "Something went wrong";

  // 🔹 Custom AppError
  if (error instanceof AppError) {
    statusCode = error.statusCode;
    message = error.message;
  }

  // 🔹 Default Error
  else if (error instanceof Error) {
    message = error.message;
  }

  res.status(statusCode).json({
    success: false,
    message,
    stack: process.env.NODE_ENV === "development" ? error.stack : undefined,
  });
};

export default globalErrorHandler;
```

---

# 🧠 কীভাবে কাজ করে?

### Step 1: Express সব error এখানে পাঠায়

যদি কোনো controller বা middleware এ error হয়:

```ts
next(error);
```

বা

```ts
throw new AppError(...)
```

তাহলে Express স্বয়ংক্রিয়ভাবে এই middleware এ পাঠায়।

---

### Step 2: Error Type Check

আমরা চেক করছি:

- এটা কি `AppError`?
- নাকি সাধারণ `Error`?

তার ভিত্তিতে response পাঠানো হচ্ছে।

---

### Step 3: Development vs Production

```ts
stack: process.env.NODE_ENV === "development" ? error.stack : undefined;
```

- Development এ full stack trace দেখাবে
- Production এ hide থাকবে

🔐 এতে sensitive information leak হয় না

---

# 3️⃣ app.ts এ ব্যবহার

সব route এর নিচে এটা ব্যবহার করতে হবে।

```ts
import express from "express";
import globalErrorHandler from "./middlewares/globalErrorHandler";

const app = express();

// Routes
app.use("/api/users", userRoutes);

// Global Error Handler (Must be last)
app.use(globalErrorHandler);

export default app;
```

⚠️ খুব গুরুত্বপূর্ণ:
Global error handler সব route এর পরে থাকতে হবে।

---

# 🎯 কেন Global Error Handler দরকার?

| Without Global Handler       | With Global Handler      |
| ---------------------------- | ------------------------ |
| এলোমেলো error response       | Structured JSON response |
| Debug কঠিন                   | Debug সহজ                |
| Sensitive info leak হতে পারে | Production-safe          |
| Code messy                   | Clean & centralized      |

---

# 📦 Example Error Response

## Development Mode

```json
{
  "success": false,
  "message": "User not found",
  "stack": "Error: User not found at ..."
}
```

## Production Mode

```json
{
  "success": false,
  "message": "User not found"
}
```

---

# 💡 Best Practice

### ✔ Always use custom AppError for business logic error

### ✔ Never send raw error to client

### ✔ Keep handler centralized

### ✔ Log error internally (optional improvement)

---

# 🔥 Summary

Global Error Handler:

- সব error এক জায়গায় handle করে
- Clean architecture maintain করে
- Production ready করে
- Security improve করে
- Debug সহজ করে
