# 🚀 Express.js-এ `catchAsync` এরর হ্যান্ডলার (Note)

`catchAsync` হলো একটি **Higher-Order Function (HOF)** যা এক্সপ্রেস কন্ট্রোলারের অ্যাসিঙ্ক্রোনাস এররগুলোকে সেন্ট্রালাইজড এরর হ্যান্ডলার মিডলওয়্যারে পাঠিয়ে দেয়। এটি ব্যবহার করলে কোডে বারবার `try-catch` ব্লক লেখার প্রয়োজন হয় না।

### কেন এটি ব্যবহার করবো? (Core Logic)

১. **Higher-Order Function:** এটি আপনার কন্ট্রোলার ফাংশনকে একটি "সেফটি র‍্যাপার"-এর মধ্যে নিয়ে আসে।
২. **Promise.resolve:** যদি আপনার কন্ট্রোলারটি ভুলবশত `async` নাও হয়, তবুও এটি প্রমিস হিসেবে ট্রিট করবে, যা কোডকে নিরাপদ রাখে।
৩. **TypeScript Generics:** এখানে কোনো `any` ব্যবহার না করে সঠিক টাইপ ডেফিনিশন ব্যবহার করা হয়েছে, যা আপনার এডিটরে (VS Code) টাইপ-সেফটি এবং অটো-সাজেশন নিশ্চিত করে।
৪. **Clean Code:** এটি কোড থেকে `try-catch` ব্লকের পুনরাবৃত্তি কমিয়ে কোডকে মেইনটেইনেবল করে তোলে।

---

### ১. মডার্ন অ্যাপ্রোচ (Recommended - Promise Based)

ইন্ডাস্ট্রি স্ট্যান্ডার্ড হিসেবে এই ইমপ্লিমেন্টেশনটি সবচেয়ে সেরা কারণ এটি ক্লিন এবং দ্রুত কাজ করে।

```typescript
import { NextFunction, Request, RequestHandler, Response } from "express";

/**
 * catchAsync - অ্যাসিঙ্ক্রোনাস এরর হ্যান্ডল করার জন্য একটি গ্লোবাল র‍্যাপার।
 * @param fn - অ্যাসিঙ্ক্রোনাস এক্সপ্রেস রিকোয়েস্ট হ্যান্ডলার।
 * @returns - একটি স্ট্যান্ডার্ড এক্সপ্রেস রিকোয়েস্ট হ্যান্ডলার।
 */
const catchAsync = (fn: RequestHandler): RequestHandler => {
  return (req: Request, res: Response, next: NextFunction): void => {
    // Promise.resolve নিশ্চিত করে যে এটি সবসময় একটি প্রমিস রিটার্ন করবে।
    // কোনো এরর (reject) হলে সেটি সরাসরি .catch(next) এর মাধ্যমে
    // গ্লোবাল এরর হ্যান্ডলারে চলে যাবে।
    Promise.resolve(fn(req, res, next)).catch((err) => next(err));
  };
};

export default catchAsync;
```

---

### ২. বিকল্প পদ্ধতি (Standard Try-Catch Based)

এটি প্রথাগত পদ্ধতি। কাজ একই করলেও এখানে ফাংশনটিকে `async` হতে হয় এবং `await` ব্যবহার করতে হয়।

```typescript
import { Request, Response, NextFunction } from "express";

const catchAsyncAlternative = (fn: Function) => {
  return async (req: Request, res: Response, next: NextFunction) => {
    try {
      await fn(req, res, next);
    } catch (error) {
      // এরর হলে সেটি next() এর মাধ্যমে গ্লোবাল হ্যান্ডলারে পাঠানো হয়
      next(error);
    }
  };
};
```

---

### ৩. ব্যবহারের নিয়ম (Usage Guide)

কন্ট্রোলার ফাইলে এটি ব্যবহারের ফলে কোড একদম ক্লিন থাকে:

```typescript
import { Request, Response } from "express";
import catchAsync from "./utils/catchAsync";

// কোনো try-catch ব্লকের দরকার নেই!
export const getUserProfile = catchAsync(
  async (req: Request, res: Response) => {
    // ডাটাবেস অপারেশন
    const user = await db.user.findUnique({ where: { id: req.params.id } });

    if (!user) {
      // ম্যানুয়ালি এরর থ্রো করলে catchAsync সেটি অটোমেটিক ধরবে
      throw new Error("User not found!");
    }

    res.status(200).json({
      success: true,
      data: user,
    });
  },
);
```

---

### ৪. টেস্ট কেস এবং সুবিধা (Testing & Benefits)

| সিনারিও                          | ফলাফল                                                                      |
| -------------------------------- | -------------------------------------------------------------------------- |
| **সফল রিকোয়েস্ট (Success)**      | সরাসরি সঠিক ডেটা রিটার্ন করবে।                                             |
| **ডাটাবেস এরর (DB Error)**       | কানেকশন বা কুয়েরি ফেইল করলে সার্ভার ক্রাশ না করে গ্লোবাল হ্যান্ডলারে যাবে। |
| **ম্যানুয়াল এরর (Manual Throw)** | `throw new Error()` করলে সেটি সরাসরি লগারে ধরা পড়বে।                       |
| **এজ কেস (Large Data)**          | বড় ডাটা ফেচ করার সময় টাইম-আউট বা মেমরি ইস্যু হলে `next(err)` কল হবে।       |
