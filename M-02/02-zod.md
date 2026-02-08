Express.js অ্যাপ্লিকেশনে **Zod** ব্যবহার করে মিডলওয়্যার তৈরি করা একটি ইন্ডাস্ট্রি-স্ট্যান্ডার্ড প্র্যাকটিস। এটি ইনকামিং রিকোয়েস্ট (body, query, params) ভ্যালিডেট করার মাধ্যমে অ্যাপ্লিকেশনকে টাইপ-সেফ এবং এরর-ফ্রি রাখে।

নিচে আমি ধাপে ধাপে TypeScript ব্যবহার করে একটি ক্লিন এবং প্রফেশনাল মিডলওয়্যার ইমপ্লিমেন্টেশন দেখাচ্ছি।

### কেন Zod মিডলওয়্যার ব্যবহার করবেন?

- **টাইপ সেফটি:** TypeScript-এর সাথে Zod নিখুঁতভাবে কাজ করে।
- **অটোমেটেড এরর হ্যান্ডলিং:** ইনপুট ভুল হলে এটি রিকোয়েস্ট হ্যান্ডলারে যাওয়ার আগেই আটকে দেয়।
- **ক্লিন কন্ট্রোলার:** কন্ট্রোলারের ভেতর বারবার ইফ-এলস দিয়ে ভ্যালিডেশন চেক করতে হয় না।

---

### স্টেপ-বাই-স্টেপ কোড ইমপ্লিমেন্টেশন

```typescript
import { Request, Response, NextFunction } from "express";
import { AnyZodObject, ZodError } from "zod";

/**
 * এটি একটি জেনেরিক ভ্যালিডেশন মিডলওয়্যার।
 * এটি রিকোয়েস্টের body, query, অথবা params-কে Zod Schema দিয়ে যাচাই করে।
 */
export const validate = (schema: AnyZodObject) => {
  return async (
    req: Request,
    res: Response,
    next: NextFunction,
  ): Promise<void> => {
    try {
      // আমরা রিকোয়েস্টের body, query এবং params তিনটিই ভ্যালিডেট করছি
      await schema.parseAsync({
        body: req.body,
        query: req.query,
        params: req.params,
      });

      // সবকিছু ঠিক থাকলে পরের ফাংশনে চলে যাবে
      next();
    } catch (error) {
      // যদি ভ্যালিডেশন ফেইল করে
      if (error instanceof ZodError) {
        res.status(400).json({
          success: false,
          message: "Validation Error",
          // এরর মেসেজগুলোকে সুন্দর করে ফরম্যাট করে পাঠানো হচ্ছে
          errors: error.errors.map((issue) => ({
            path: issue.path[issue.path.length - 1],
            message: issue.message,
          })),
        });
        return;
      }

      // অন্য কোনো অজানা এরর হলে
      res.status(500).json({
        success: false,
        message: "Internal Server Error",
      });
    }
  };
};
```

### প্র্যাকটিক্যাল ব্যবহার (Example Usage)

ধরা যাক, আমরা একজন ইউজার রেজিস্ট্রেশনের জন্য একটি স্কিমা এবং রাউট তৈরি করছি।

```typescript
import express from "express";
import { z } from "zod";
import { validate } from "./middleware/validate"; // ওপরের মিডলওয়্যারটি ইম্পোর্ট করুন

const app = express();
app.use(express.json());

// ১. একটি Zod Schema ডিফাইন করা
const createUserSchema = z.object({
  body: z.object({
    username: z.string({ required_error: "ইউজার নেম দিতেই হবে" }).min(3),
    email: z.string().email("সঠিক ইমেইল দিন"),
    password: z.string().min(6, "পাসওয়ার্ড অন্তত ৬ অক্ষরের হতে হবে"),
  }),
});

// ২. রাউটে মিডলওয়্যারটি ব্যবহার করা
app.post(
  "/api/register",
  validate(createUserSchema), // মিডলওয়্যার এখানে কল করা হয়েছে
  (req, res) => {
    // এখানে আসা মানে ডেটা ১০০% ভ্যালিড
    res.status(201).json({ message: "ইউজার তৈরি হয়েছে সফলভাবে!" });
  },
);

app.listen(3000, () => console.log("Server running on port 3000"));
```

---

১. প্রয়োজনীয় প্যাকেজ ইনস্টল করুন:

```bash
npm install express zod

```
