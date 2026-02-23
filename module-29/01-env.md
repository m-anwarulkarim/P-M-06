# 📦 Environment Variables Loader

এই ফাইলটি আমাদের প্রজেক্টের সব **গোপন তথ্য** (যেমন: ডাটাবেস পাসওয়ার্ড, সিক্রেট কি) `.env` ফাইল থেকে লোড করে এবং চেক করে যে সব ঠিক আছে কি না।

### কেন এটি ব্যবহার করবো?

১. **নিরাপত্তা:** সরাসরি `process.env` ব্যবহার না করে একটি সেন্ট্রাল জায়গা থেকে ব্যবহার করা নিরাপদ।
২. **ভুল ধরা:** যদি কোনো ভেরিয়েবল মিসিং থাকে, তবে অ্যাপ শুরু হওয়ার আগেই এরর দিয়ে আমাদের জানিয়ে দিবে।
৩. **টাইপ সেফটি:** কোড লেখার সময় অটো-সাজেশন পাওয়া যায়।

---

## 📁 `env.ts` (ইমপ্লিমেন্টেশন)

```ts
import dotenv from "dotenv";
import path from "path";
import status from "http-status";
import AppError from "../errorHelpers/AppError";

// .env ফাইলটি খুঁজে বের করে লোড করা হচ্ছে
dotenv.config({ path: path.join(process.cwd(), ".env") });

/**
 * ১. Interface: কি কি ভেরিয়েবল থাকবে এবং তাদের টাইপ কী হবে
 */
interface EnvConfig {
  NODE_ENV: "development" | "production" | "test";
  PORT: number;
  DATABASE_URL: string;
  BETTER_AUTH_SECRET: string;
  BETTER_AUTH_URL: string;
  ACCESS_TOKEN_SECRET: string;
  REFRESH_TOKEN_SECRET: string;
  ACCESS_TOKEN_EXPIRES_IN: string;
  REFRESH_TOKEN_EXPIRES_IN: string;
}

/**
 * ২. লোড এবং ভ্যালিডেশন ফাংশন
 */
const loadEnvVariables = (): EnvConfig => {
  const requiredEnv = [
    "NODE_ENV",
    "PORT",
    "DATABASE_URL",
    "BETTER_AUTH_SECRET",
    "BETTER_AUTH_URL",
    "ACCESS_TOKEN_SECRET",
    "REFRESH_TOKEN_SECRET",
  ];

  // চেক করা হচ্ছে কোনো ভেরিয়েবল মিসিং আছে কি না
  const missing = requiredEnv.filter((v) => !process.env[v]);

  if (missing.length > 0) {
    throw new AppError(
      status.INTERNAL_SERVER_ERROR,
      `❌ .env ফাইলে এই ভেরিয়েবলগুলো নেই: ${missing.join(", ")}`,
    );
  }

  // সব ঠিক থাকলে ডাটা রিটার্ন করা হচ্ছে
  return {
    NODE_ENV: process.env.NODE_ENV as any,
    PORT: Number(process.env.PORT) || 5000, // স্ট্রিং থেকে নাম্বারে রূপান্তর
    DATABASE_URL: process.env.DATABASE_URL!,
    BETTER_AUTH_SECRET: process.env.BETTER_AUTH_SECRET!,
    BETTER_AUTH_URL: process.env.BETTER_AUTH_URL!,
    ACCESS_TOKEN_SECRET: process.env.ACCESS_TOKEN_SECRET!,
    REFRESH_TOKEN_SECRET: process.env.REFRESH_TOKEN_SECRET!,
    ACCESS_TOKEN_EXPIRES_IN: process.env.ACCESS_TOKEN_EXPIRES_IN!,
    REFRESH_TOKEN_EXPIRES_IN: process.env.REFRESH_TOKEN_EXPIRES_IN!,
  };
};

// ৩. পুরো প্রজেক্টে ব্যবহারের জন্য এক্সপোর্ট
export const envVars = loadEnvVariables();
```

---

## 🧠 কীভাবে কাজ করে? (Step-by-Step)

1. **`dotenv.config()`**: এটি আপনার `.env` ফাইলের সব লেখা পড়ে কম্পিউটারের মেমরিতে (process.env) জমা করে।
2. **`Missing Check`**: এখানে আমরা সব বাধ্যতামূলক ভেরিয়েবলের একটি লিস্ট বানিয়েছি। লুপ চালিয়ে দেখা হয় সব আছে কি না। না থাকলে সুন্দর একটি এরর মেসেজ দেয়।
3. **`Type Conversion`**: আমাদের পোর্টের ভ্যালু `.env` ফাইলে থাকে স্ট্রিং হিসেবে, আমরা এখানে সেটাকে **Number** বানিয়ে দিই।
4. **`Export`**: শেষে `envVars` কে এক্সপোর্ট করি যাতে যেকোনো ফাইলে `envVars.PORT` লিখলেই আমরা ভ্যালু পেয়ে যাই।

---

## 🎯 কেন এটা সেরা প্র্যাকটিস?

✅ **এক জায়গায় সব:** সব কনফিগ এক জায়গায় থাকে, খুঁজতে হয় না।
✅ **শুরুতেই এরর:** অ্যাপ রান হওয়ার আগেই ভুল ধরা পড়ে (Fail Early)।
✅ **স্মার্ট কোডিং:** `as string` এর বদলে ইন্টারফেস ব্যবহার করায় অটো-সাজেশন পাওয়া যায়।

---

## 📌 ব্যবহারের উদাহরণ

অন্য যেকোনো ফাইলে শুধু ইম্পোর্ট করে ব্যবহার করবেন:

```ts
import { envVars } from "./config/env";

app.listen(envVars.PORT, () => {
  console.log(`সার্ভার চলছে ${envVars.PORT} পোর্টে`);
});
```
