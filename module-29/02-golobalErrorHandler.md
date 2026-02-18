- Dependencies
  `pnpm add winston source-map-support`
- Dev Dependencies
  `pnpm add -D @types/source-map-support `

### ১. Logger কনফিগারেশন (`src/utils/logger.ts`)

এখানে `winston` ব্যবহার করা হয়েছে যা এররগুলোকে একটি ফাইল (`logs/error.log`) এবং কনসোলে সেভ করবে।

project/
├─ src/
├─ logs/
│ └─ error.log ← error log এখানে জমা হবে। মানে console এ শুধু দেখাবে না — file এও save থাকবে।

```typescript
import winston from "winston";

/**
 * কেন এই ফরম্যাট?
 * combine: একাধিক ফরম্যাটকে একসাথে করার জন্য।
 * timestamp: এররটি কখন হয়েছে তা জানার জন্য।
 * errors({ stack: true }): এররের স্ট্যাক ট্রেস (লাইন নম্বরসহ) ক্যাপচার করার জন্য।
 * json: প্রোডাকশনে লগ এনালাইসিস সহজ করার জন্য JSON ফরম্যাট।
 */
const logger = winston.createLogger({
  level: "error",
  format: winston.format.combine(
    winston.format.timestamp({ format: "YYYY-MM-DD HH:mm:ss" }),
    winston.format.errors({ stack: true }),
    winston.format.json(),
  ),
  transports: [
    // শুধুমাত্র এররগুলো এই ফাইলে জমা হবে
    new winston.transports.File({
      filename: "logs/error.log",
      level: "error",
    }),
    // সব ধরণের লগ (info, warn, error) এখানে থাকবে
    new winston.transports.File({ filename: "logs/combined.log" }),
  ],
});

// যদি ডেভেলপমেন্ট মোডে থাকি, তবে কনসোলে রঙিন আউটপুট দেখাবে
if (process.env.NODE_ENV !== "production") {
  logger.add(
    new winston.transports.Console({
      format: winston.format.combine(
        winston.format.colorize(),
        winston.format.simple(),
      ),
    }),
  );
}

export default logger;
```

---

### ২. গ্লোবাল এরর হ্যান্ডলার (`src/middlewares/globalErrorHandler.ts`)

এই ফাংশনটি অ্যাপ্লিকেশনের সব এরর এক জায়গায় রিসিভ করবে এবং লাইন নম্বরসহ লগ করবে।

```typescript
import { Request, Response, NextFunction, ErrorRequestHandler } from "express";
import logger from "../utils/logger";

/**
 * globalErrorHandler: এক্সপ্রেসের স্পেশাল মিডলওয়্যার (৪টি প্যারামিটার থাকে)।
 * এটি catchAsync থেকে আসা সব এরর হ্যান্ডেল করে।
 */
const globalErrorHandler: ErrorRequestHandler = (err, req, res, next) => {
  let statusCode = err.statusCode || 500;
  let message = err.message || "Internal Server Error";

  // ১. স্ট্যাক ট্রেস থেকে লাইন নম্বর বের করা
  // সাধারণত err.stack এর দ্বিতীয় লাইনে (index 1) ফাইলের নাম ও লাইন থাকে
  const stackLines = err.stack ? err.stack.split("\n") : [];
  const originLine =
    stackLines.length > 1 ? stackLines[1].trim() : "Unknown position";

  // ২. লগারে বিস্তারিত তথ্য পাঠানো
  logger.error({
    message: err.message,
    location: originLine, // এখানে আপনি লাইন নম্বর দেখতে পাবেন
    path: req.originalUrl, // কোন API এন্ডপয়েন্টে এরর হয়েছে
    method: req.method, // GET, POST না কি অন্য কিছু
    stack: err.stack, // পুরো স্ট্যাক ট্রেস (ডিপ ডিবাগিং এর জন্য)
  });

  // ৩. ইউজারকে রেসপন্স পাঠানো
  res.status(statusCode).json({
    success: false,
    message: message,
    // শুধুমাত্র ডেভেলপমেন্ট এনভায়রনমেন্টে লাইন নম্বর ও স্ট্যাক দেখাবো
    ...(process.env.NODE_ENV === "development" && {
      errorSource: originLine,
      stack: err.stack,
    }),
  });
};

export default globalErrorHandler;
```

---

### ৩. অ্যাপ্লিকেশনে সেটআপ এবং রান করার নিয়ম

আপনার মেইন ফাইল (যেমন: `app.ts` বা `server.ts`) নিচের মতো সাজান:

```typescript
import "source-map-support/register"; // কম্পাইল্ড কোডে অরিজিনাল TS লাইন নম্বর পাওয়ার জন্য (মাস্ট!)
import express from "express";
import globalErrorHandler from "./middlewares/globalErrorHandler";
import catchAsync from "./utils/catchAsync";

const app = express();

// একটি ডেমো রাউট যেখানে এরর হতে পারে
app.get(
  "/test-error",
  catchAsync(async (req, res) => {
    // ধরুন এখানে একটি ভুল ভ্যারিয়েবল কল করা হলো
    const user: any = undefined;
    console.log(user.profile.name); // এখানে এরর হবে এবং catchAsync এটি ধরবে
  }),
);

// সব রাউটের শেষে এটি অবশ্যই রাখতে হবে
app.use(globalErrorHandler);

app.listen(3000, () => console.log("Server running on port 3000"));
```

---

### কেন এটি সেরা প্র্যাকটিস?

- **TypeScript Friendly:** টাইপ সেফটি বজায় রাখা হয়েছে।
- **Source Map Support:** যেহেতু আপনি TS লিখছেন, তাই `source-map-support` ব্যবহারের ফলে আপনি লগে `.js` ফাইলের বদলে সরাসরি `.ts` ফাইলের লাইন নম্বর দেখতে পাবেন।
- **JSON Logging:** প্রোডাকশনে যখন আপনি `ELK Stack` বা `Winston` এনালাইজার ব্যবহার করবেন, তখন JSON ফরম্যাট অনেক উপকারে আসবে।
- **Clean Response:** সাধারণ ইউজাররা শুধু এরর মেসেজ দেখবে, কিন্তু আপনি (ডেভেলপার) পুরো ডিটেইলস দেখতে পাবেন।

### পরবর্তী ধাপ:

আপনার কি কোনো **Custom API Error Class** লাগবে? যেখানে আপনি নিজেই বলে দিতে পারবেন এটি কি ৪০০ (Bad Request) না কি ৪0৪ (Not Found) এরর? আমি কি সেটি তৈরি করে দেব?
