# 📬 Express.js-এ `sendResponse` ইউটিলিটি (Note)

`sendResponse` হলো একটি **Utility Function** যা পুরো অ্যাপ্লিকেশনের সাকসেস রেসপন্সগুলোকে একটি নির্দিষ্ট এবং স্ট্যান্ডার্ড ফরম্যাটে ক্লায়েন্টের কাছে পাঠাতে ব্যবহার করা হয়।

### কেন এই ফাংশনটি প্রয়োজন? (Core Logic)

১. **Consistency:** আপনার অ্যাপ্লিকেশনের সব API যেন একই ধরণের ডাটা স্ট্রাকচার (যেমন: `success`, `statusCode`, `data`) রিটার্ন করে।
২. **DRY (Don't Repeat Yourself):** প্রতি কন্ট্রোলারে বারবার `res.status(...).json(...)` লেখার ঝামেলা কমায়।
৩. **TypeScript Generics (`<T>`):** এটি ব্যবহারের ফলে `data` ফিল্ডে আপনি যে ধরণের ডাটা পাঠাবেন (User, Product, বা Array), TypeScript সেটি অটো-ডিটেক্ট করতে পারবে।
৪. **Separation of Concerns:** এক্সপ্রেসের `res` অবজেক্ট এবং আপনার পাঠানো `data`-কে আলাদা রেখে কোডকে আরও রিডেবল করে।

---

### ১. `sendResponse` ইমপ্লিমেন্টেশন

এই কোডটি আপনার `src/utils/sendResponse.ts` ফাইলে রাখতে পারেন।

```typescript
import { Response } from "express";

/**
 * IApiResponse Interface:
 * রেসপন্স অবজেক্টের গঠন কেমন হবে তা এখানে ডিফাইন করা হয়েছে।
 * <T> হলো একটি জেনেরিক টাইপ, যা যেকোনো ধরণের ডাটা গ্রহণ করতে পারে।
 */
type IApiResponse<T> = {
  statusCode: number; // HTTP স্ট্যাটাস কোড (যেমন: 200, 201)
  success: boolean; // রিকোয়েস্ট সফল কি না (true/false)
  message?: string | null; // ইউজার ফ্রেন্ডলি মেসেজ
  meta?: {
    // প্যাজিনেশনের জন্য অতিরিক্ত তথ্য (অপশনাল)
    page: number;
    limit: number;
    total: number;
  };
  data?: T | null; // মূল ডেটা যা আমরা পাঠাতে চাই
};

/**
 * sendResponse ফাংশন:
 * @param res - এক্সপ্রেসের রেসপন্স অবজেক্ট (এটি ট্রাকের মতো যা ডাটা বহন করে)
 * @param data - আমাদের কাস্টম ডাটা অবজেক্ট (IApiResponse টাইপের)
 */
const sendResponse = <T>(res: Response, data: IApiResponse<T>): void => {
  // একটি স্ট্যান্ডার্ড ফরম্যাটে রেসপন্স বডি তৈরি করা হচ্ছে
  const responseData: IApiResponse<T> = {
    success: data.success,
    statusCode: data.statusCode,
    message: data.message || null,
    meta: data.meta || undefined, // যদি মেটা না থাকে তবে এটি রেসপন্সে দেখাবে না
    data: data.data || null,
  };

  // এক্সপ্রেসের মাধ্যমে রেসপন্স পাঠানো হচ্ছে
  res.status(data.statusCode).json(responseData);
};

export default sendResponse;
```

---

### ২. কেন এখানে ২টা প্যারামিটার (`res` এবং `data`)?

- **প্যারামিটার ১ (`res`):** এটি এক্সপ্রেস থেকে আসা অরিজিনাল **Response Object**। এর কাজ হলো নেটওয়ার্কের মাধ্যমে ক্লায়েন্টের কাছে ডাটা পৌঁছে দেওয়া। এটি ছাড়া আমরা `res.status()` কল করতে পারব না।
- **প্যারামিটার ২ (`data`):** এটি আমাদের তৈরি করা **Custom Object**। এর ভেতর আমরা বলে দেই স্ট্যাটাস কোড কত হবে, মেসেজ কী হবে এবং আসল ডাটা (`payload`) কী থাকবে।

---

### ৩. ব্যবহারের উদাহরণ (Usage Example)

কন্ট্রোলারে এটি যেভাবে ব্যবহার করবেন:

```typescript
import { Request, Response } from "express";
import catchAsync from "./catchAsync";
import sendResponse from "./sendResponse";

// একটি ডেমো ইউজার লিস্ট API
export const getAllUsers = catchAsync(async (req: Request, res: Response) => {
  const result = [{ id: 1, name: "Gemini" }];

  // sendResponse কল করা হচ্ছে
  sendResponse(res, {
    statusCode: 200,
    success: true,
    message: "Users fetched successfully!",
    meta: {
      page: 1,
      limit: 10,
      total: 1,
    },
    data: result,
  });
});
```

---

### ৪. সুবিধা ও টেস্ট কেস

| সুবিধা                 | ব্যাখ্যা                                                                       |
| ---------------------- | ------------------------------------------------------------------------------ |
| **Type Safety**        | ভুল টাইপের ডাটা পাঠালে TypeScript কম্পাইল টাইমে এরর দেবে।                      |
| **Meta Support**       | লিস্ট জাতীয় ডাটার ক্ষেত্রে `total` বা `page` সংখ্যা সহজে পাঠানো যায়।           |
| **Front-end Friendly** | ফ্রন্ট-এন্ড ডেভেলপাররা সবসময় একই ফরম্যাটে ডাটা পায়, তাই তাদের কোড করতে সহজ হয়। |
