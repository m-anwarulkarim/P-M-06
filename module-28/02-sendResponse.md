# Global Response Utility: Standardizing API Success Responses

### ১. `src/utils/sendResponse.ts` (শর্টকাট ইউটিলিটি)

```typescript
import { Response } from "express";

// রেসপন্স এর গঠন কেমন হবে তার একটি টাইপ ইন্টারফেস
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

### ২. এটি ব্যবহারের সুবিধা (কন্ট্রোলারে উদাহরণ)

এখন আপনার কন্ট্রোলারে আর বড় বড় কোড লিখতে হবে না। নিচের উদাহরণটি দেখুন:

```typescript
// src/controllers/userController.ts
import { asyncHandler } from "../utils/asyncHandler";
import sendResponse from "../utils/sendResponse";

export const getUser = asyncHandler(async (req, res) => {
  const user = { id: 1, name: "Gemini" };

  // শর্টকাট রেসপন্স ব্যবহার
  sendResponse(res, {
    statusCode: 200,
    success: true,
    message: "User fetched successfully!",
    data: user,
  });
});
```

---

### কেন এই শর্টকাটগুলো ব্যবহার করবেন? (ব্যাখ্যা)

- **Consistency (একই রকম ফরম্যাট):** পুরো অ্যাপের সব রেসপন্স একই রকম দেখাবে (success, message, data)। ফ্রন্টএন্ড ডেভেলপারদের জন্য এটি আশীর্বাদ।
- **Less Typing (কম কোড):** প্রতিবার `res.status().json()` লেখার বদলে জাস্ট একটা ফাংশন কল করলেই হচ্ছে।
- **Centralized Control:** যদি ভবিষ্যতে আপনি রেসপন্স ফরম্যাটে নতুন কিছু যোগ করতে চান (যেমন: API ভার্সন বা টাইমস্ট্যাম্প), তবে শুধু এক জায়গায় চেঞ্জ করলেই পুরো অ্যাপে তা আপডেট হয়ে যাবে।

**ছোট্ট টিপস:** আপনি চাইলে `http-status-codes` নামের একটি প্যাকেজ ইনস্টল করতে পারেন। এতে `200` এর জায়গায় `StatusCodes.OK` ব্যবহার করা যায়, যা কোডকে আরও রিডবল (readable) করে।

আপনি কি চান আমি **Zod Validation** এর জন্য এমন কোনো শর্টকাট মিডলওয়্যার দেখিয়ে দিই? এটি ইনপুট চেক করার কাজ অনেক সহজ করে দেয়।
