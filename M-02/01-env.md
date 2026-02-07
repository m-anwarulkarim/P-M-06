# set up environment variables

```typescript
import dotenv from "dotenv";
import status from "http-status";
import path from "path";
import AppError from "../errorHelpers/AppError";

// ১. প্রজেক্টের রুট থেকে .env লোড করা নিশ্চিত করা
dotenv.config({ path: path.join(process.cwd(), ".env") });

// ২. ইন্টারফেস ডিফাইন করা (টাইপ সেফটির জন্য)
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

const loadEnvVariables = (): EnvConfig => {
  // ৩. রিকোয়ার্ড ভ্যারিয়েবলগুলোর লিস্ট
  const requiredEnvVariables: (keyof EnvConfig)[] = [
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

  // ৪. লুপ চালিয়ে চেক করা এবং একটি অবজেক্টে ডেটা জমানো
  const config = {} as EnvConfig;

  for (const variable of requiredEnvVariables) {
    const value = process.env[variable];

    if (!value) {
      // ৫. কোনো ভ্যারিয়েবল মিসিং থাকলে কাস্টম অ্যাপ এরর থ্রো করা
      throw new AppError(
        status.INTERNAL_SERVER_ERROR,
        `Environment variable "${variable}" is missing in .env file!`,
      );
    }

    config[variable] = value;
  }

  return config;
};

// ৬. একবারে এক্সপোর্ট করা যাতে পুরো অ্যাপে ইমপোর্ট করে ব্যবহার করা যায়
export const envVars = loadEnvVariables();
```

---

### বিস্তারিত ব্যাখ্যা (কেন এটি ভালো?)

১. **টাইপ সেফটি (Interface):** `interface EnvConfig` ব্যবহারের ফলে আপনি যখন অ্যাপের অন্য কোথাও `envVars.` লিখবেন, তখন আপনার এডিটর (VS Code) আপনাকে অটোমেটিক সাজেস্ট করবে কোন কোন কি (key) সেখানে আছে। এতে বানান ভুল হওয়ার সম্ভাবনা থাকে না।

২. **সেন্ট্রালাইজড ভ্যালিডেশন (Validation Logic):** আপনি সরাসরি `process.env` ব্যবহার না করে একটি ফাংশনের মাধ্যমে চেক করছেন। এর বড় সুবিধা হলো—যদি কেউ ভুলবশত `.env` ফাইল থেকে কোনো সিক্রেট ডিলিট করে দেয়, তবে আপনার সার্ভারটি চালু হবে না। এটি প্রোডাকশনে বড় কোনো ক্রাশ বা সিকিউরিটি হোল থেকে বাঁচায়।

৩. **`path.join(process.cwd(), '.env')`:** কখনও কখনও ডিপ্লয়মেন্টের সময় বা আলাদা ডিরেক্টরি থেকে স্ক্রিপ্ট রান করলে `.env` ফাইল খুঁজে পায় না। `process.cwd()` ব্যবহার করলে এটি সবসময় প্রজেক্টের মেইন ফোল্ডার থেকে ফাইলটি লোড করবে, যা কোডকে আরও রোবাস্ট (Robust) করে।

৪. **ক্লিন কোড (DRY Principle):** আগের কোডে আপনি একই ভ্যারিয়েবলের নাম বারবার লিখছিলেন। রিফ্যাক্ট করা কোডে আমি শুধু একটি লুপ ব্যবহার করেছি যা অটোমেটিক `config` অবজেক্টটি তৈরি করে দিচ্ছে। এতে কোড ছোট হয়েছে এবং পড়তে সুবিধা হচ্ছে।

### কীভাবে ব্যবহার করবেন?

এখন আপনার `server.ts` বা ডাটাবেস কানেকশন ফাইলে জাস্ট এভাবে ইমপোর্ট করবেন:

```typescript
import { envVars } from "./config/envConfig";

// ব্যবহার:
console.log(envVars.PORT);
console.log(envVars.DATABASE_URL);
```
