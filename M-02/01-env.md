

# 📦 Environment Variables Loader (TypeScript)

এই ফাইলটি প্রজেক্টে ব্যবহৃত সকল গুরুত্বপূর্ণ **environment variable** লোড ও ভ্যালিডেশন করার জন্য ব্যবহৃত হয়।

এটি নিশ্চিত করে যে `.env` ফাইলে প্রয়োজনীয় সব ভ্যারিয়েবল আছে কিনা।
কোনোটি না থাকলে অ্যাপ স্টার্ট হওয়ার আগেই error throw করবে।

---

## 📁 `env.ts`

```ts
import dotenv from "dotenv";
import status from "http-status";
import AppError from "../errorHelpers/AppError";

dotenv.config();

/**
 * Environment Variables Type Definition
 */
interface EnvConfig {
  NODE_ENV: string;
  PORT: string;
  DATABASE_URL: string;

  BETTER_AUTH_SECRET: string;
  BETTER_AUTH_URL: string;

  ACCESS_TOKEN_SECRET: string;
  REFRESH_TOKEN_SECRET: string;

  ACCESS_TOKEN_EXPIRES_IN: string;
  REFRESH_TOKEN_EXPIRES_IN: string;

  BETTER_AUTH_SESSION_TOKEN_EXPIRES_IN: string;
  BETTER_AUTH_SESSION_TOKEN_UPDATE_AGE: string;
}

/**
 * Load and Validate Required Environment Variables
 */
const loadEnvVariables = (): EnvConfig => {
  const requiredEnvVariables = [
    "NODE_ENV",
    "PORT",
    "DATABASE_URL",
    "BETTER_AUTH_SECRET",
    "BETTER_AUTH_URL",
    "ACCESS_TOKEN_SECRET",
    "REFRESH_TOKEN_SECRET",
    "ACCESS_TOKEN_EXPIRES_IN",
    "REFRESH_TOKEN_EXPIRES_IN",
    "BETTER_AUTH_SESSION_TOKEN_EXPIRES_IN",
    "BETTER_AUTH_SESSION_TOKEN_UPDATE_AGE",
  ];

  requiredEnvVariables.forEach((variable) => {
    if (!process.env[variable]) {
      throw new AppError(
        status.INTERNAL_SERVER_ERROR,
        `Environment variable ${variable} is missing in .env file`
      );
    }
  });

  return {
    NODE_ENV: process.env.NODE_ENV as string,
    PORT: process.env.PORT as string,
    DATABASE_URL: process.env.DATABASE_URL as string,

    BETTER_AUTH_SECRET: process.env.BETTER_AUTH_SECRET as string,
    BETTER_AUTH_URL: process.env.BETTER_AUTH_URL as string,

    ACCESS_TOKEN_SECRET: process.env.ACCESS_TOKEN_SECRET as string,
    REFRESH_TOKEN_SECRET: process.env.REFRESH_TOKEN_SECRET as string,

    ACCESS_TOKEN_EXPIRES_IN: process.env.ACCESS_TOKEN_EXPIRES_IN as string,
    REFRESH_TOKEN_EXPIRES_IN: process.env.REFRESH_TOKEN_EXPIRES_IN as string,

    BETTER_AUTH_SESSION_TOKEN_EXPIRES_IN:
      process.env.BETTER_AUTH_SESSION_TOKEN_EXPIRES_IN as string,
    BETTER_AUTH_SESSION_TOKEN_UPDATE_AGE:
      process.env.BETTER_AUTH_SESSION_TOKEN_UPDATE_AGE as string,
  };
};

export const envVars = loadEnvVariables();
```

---

# 🧠 কীভাবে কাজ করে?

## 1️⃣ `dotenv.config()`

`.env` ফাইল থেকে সব environment variable লোড করে `process.env` এ সেট করে।

---

## 2️⃣ `EnvConfig` Interface

এই interface বলে দেয় কোন কোন variable থাকতে হবে এবং সেগুলোর type কী হবে।

এতে:

* Type safety পাওয়া যায়
* ভুল variable ব্যবহার করলে TypeScript error দিবে

---

## 3️⃣ `requiredEnvVariables` Array

এখানে সব বাধ্যতামূলক `.env` variable রাখা হয়েছে।

যদি কোনোটা missing থাকে → অ্যাপ স্টার্ট হওয়ার আগেই error দিবে।

---

## 4️⃣ Validation Logic

```ts
requiredEnvVariables.forEach((variable) => {
  if (!process.env[variable]) {
    throw new AppError(...);
  }
});
```

এখানে আমরা চেক করছি:

👉 `.env` এ সব variable আছে কিনা
👉 না থাকলে custom error throw করছি

এতে production এ গিয়ে crash হওয়ার আগে development stage-এই ধরা পড়ে।

---

## 5️⃣ Final Export

```ts
export const envVars = loadEnvVariables();
```

এখন পুরো প্রজেক্টে তুমি ব্যবহার করতে পারবে:

```ts
import { envVars } from "../config/env";

console.log(envVars.PORT);
```

---

# 🎯 কেন এটা ব্যবহার করা ভালো?

✅ Centralized config
✅ Runtime validation
✅ Production-safe
✅ TypeScript support
✅ Crash early strategy

---

# 📌 Example `.env`

```env
NODE_ENV=development
PORT=5000
DATABASE_URL=postgresql://...

BETTER_AUTH_SECRET=your_secret
BETTER_AUTH_URL=http://localhost:5000

ACCESS_TOKEN_SECRET=access_secret
REFRESH_TOKEN_SECRET=refresh_secret

ACCESS_TOKEN_EXPIRES_IN=1d
REFRESH_TOKEN_EXPIRES_IN=7d

BETTER_AUTH_SESSION_TOKEN_EXPIRES_IN=7d
BETTER_AUTH_SESSION_TOKEN_UPDATE_AGE=1d
```
