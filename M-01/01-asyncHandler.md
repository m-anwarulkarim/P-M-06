# 🛠 Express TypeScript Error

এই প্রজেক্টে একটি সেন্ট্রাল এরর হ্যান্ডলিং এবং রেসপন্স ম্যানেজমেন্ট আর্কিটেকচার ব্যবহার করা হয়েছে যা কোডকে DRY (Don't Repeat Yourself) রাখতে সাহায্য করে।

---

### ১. `src/utils/AppError.ts` (Custom Error Class)

> **কাজ:** এটি একটি কাস্টম এরর ক্লাস। এর মাধ্যমে আপনি সহজেই এররের ধরন (Operational vs Programming), স্ট্যাটাস কোড (404, 401) এবং মেসেজ সেট করতে পারেন। `Error.captureStackTrace` ব্যবহারের ফলে এররটি কোডের ঠিক কত নাম্বার লাইনে হয়েছে তা নির্ভুলভাবে জানা যায়।

```typescript
export class AppError extends Error {
  public readonly statusCode: number;
  public readonly status: string;
  public readonly isOperational: boolean;

  constructor(message: string, statusCode: number) {
    super(message);
    this.statusCode = statusCode;
    this.status = `${statusCode}`.startsWith("4") ? "fail" : "error";
    this.isOperational = true;
    Error.captureStackTrace(this, this.constructor);
  }
}
export default AppError;
```

---

### ২. `src/utils/asyncHandler.ts` (Clean Async Logic)

> **কাজ:** এক্সপ্রেসের অ্যাসিনক্রোনাস ফাংশনগুলোকে হ্যান্ডেল করার একটি র‍্যাপার (Wrapper)। এটি বারবার `try-catch` লেখার ঝামেলা দূর করে কোডকে ক্লিন রাখে এবং কোনো এরর হলে তা স্বয়ংক্রিয়ভাবে গ্লোবাল হ্যান্ডলারে পাঠিয়ে দেয়।

```typescript
import { Request, Response, NextFunction, RequestHandler } from "express";

export const asyncHandler = (fn: RequestHandler): RequestHandler => {
  return async (req: Request, res: Response, next: NextFunction) => {
    try {
      await fn(req, res, next);
    } catch (error) {
      next(error);
    }
  };
};
```

---

### ৩. `src/middleware/globalErrorHandler.ts` (Global Control Room)

> **কাজ:** এটি পুরো অ্যাপ্লিকেশনের **সেন্ট্রাল কন্ট্রোল রুম**। সব এরর এখানে এসে জমা হয়। এটি নির্ধারণ করে ডেভেলপমেন্ট মোডে বিস্তারিত এরর (Stack trace) এবং প্রোডাকশন মোডে ইউজার-ফ্রেন্ডলি মেসেজ দেখাবে কি না।

```ts
// src/middleware/globalErrorHandler.ts
import { ErrorRequestHandler } from "express";
import status from "http-status";
import { envVars } from "../config";

const globalErrorHandler: ErrorRequestHandler = (err, req, res, next) => {
  // যদি এররটি আমাদের AppError না হয় (যেমন DB error), তবে ডিফল্ট ৫০০ ধরা
  let statusCode = err.statusCode || 500;
  let message = err.message || "Internal Server Error";

  // Mongoose Duplicate Key Error হ্যান্ডল করার উদাহরণ (Pro-tip)
  if (err.code === 11000) {
    statusCode = 400;
    message = "Duplicate field value entered";
  }

  res.status(statusCode).json({
    success: false,
    message,
    stack: envVars.NODE_ENV === "development" ? err.stack : undefined,
  });
};
export default globalErrorHandler;
```

---

### ৪. `src/utils/sendResponse.ts` (Unified API Response)

> **কাজ:** এটি একটি জেনেরিক রেসপন্স ফাংশন। এর মাধ্যমে পুরো অ্যাপ্লিকেশনের সফল রেসপন্সগুলো (Success Responses) একটি নির্দিষ্ট ফরমেটে পাঠানো হয়, যা ফ্রন্টএন্ডের জন্য হ্যান্ডেল করা সহজ।

```typescript
import { Response } from "express";

interface IResponse<T> {
  statusCode: number;
  success: boolean;
  message?: string;
  data: T;
}

const sendResponse = <T>(res: Response, data: IResponse<T>) => {
  res.status(data.statusCode).json({
    success: data.success,
    message: data.message || "Success",
    data: data.data,
  });
};

export default sendResponse;
```

---

### ৫. `src/app.ts` (Integration)

> **কাজ:** এখানে সব মিডলওয়্যার এবং রাউটগুলোকে জোড়া দেওয়া হয়েছে। মনে রাখতে হবে, `errorHandler` সবসময় সব রাউটের শেষে বসাতে হয়।

```typescript
// ... imports
const app = express();
app.use(express.json());

app.get(
  "/api/v1/users/:id",
  asyncHandler(async (req, res) => {
    const user = null;
    if (!user) throw new AppError("User not found", 404);

    sendResponse(res, {
      statusCode: 200,
      success: true,
      message: "User fetched successfully!",
      data: user,
    });
  }),
);

app.all("*", (req, res, next) => {
  next(new AppError(`Can't find ${req.originalUrl} on this server!`, 404));
});

app.use(errorHandler);
```

---

### 🚀 মূল সুবিধাসমূহ:

- **Clean Code:** কন্ট্রোলারে বাড়তি `try-catch` বা রেসপন্স লজিক লিখতে হয় না।
- **Consistency:** পুরো অ্যাপের এরর এবং রেসপন্স ফরম্যাট সবসময় একই থাকে।
- **Easy Debugging:** ডেভেলপমেন্ট মোডে পূর্ণাঙ্গ এরর স্ট্যাক পাওয়া যায়।
