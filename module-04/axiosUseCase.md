## কম্পোনেন্টে (Next.js/React) এটি কীভাবে ব্যবহার হয়?

সার্ভিস লেয়ার ব্যবহার করলে আপনার কম্পোনেন্ট ডাটা ফেচিংয়ের জটিল লজিক (যেমন Axios কনফিগারেশন বা এন্ডপয়েন্ট ইউআরএল) সম্পর্কে কিছু জানবে না। সে শুধু ডাটা পাবে এবং তা স্ক্রিনে দেখাবে।

### ১. ইউজার প্রোফাইল দেখানো (Read Case - GET)

এটি সাধারণত ড্যাশবোর্ড বা প্রোফাইল পেজে ব্যবহার করা হয় যেখানে পেজ লোড হওয়ার সাথে সাথে ডাটা ফেচ করতে হয়।

```tsx
// components/ProfileCard.tsx
"use client";

import { useEffect, useState } from "react";
import { UserService, UserProfile } from "@/services/user.service";

export default function ProfileCard() {
  // ১. স্টেট ডিক্লেয়ার করা
  const [user, setUser] = useState<UserProfile | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    // ২. ডাটা আনার জন্য একটি অ্যাসিংক্রোনাস ফাংশন
    const loadData = async () => {
      try {
        setLoading(true);
        // সার্ভিস থেকে ডাটা কল করা
        const data = await UserService.getProfile();
        setUser(data);
      } catch (error) {
        // কোনো সমস্যা হলে কনসোলে এরর দেখাবে
        console.error("ডাটা লোড করতে সমস্যা হয়েছে:", error);
      } finally {
        // কাজ শেষ হলে লোডিং স্টপ হবে
        setLoading(false);
      }
    };

    loadData();
  }, []); // ডিপেন্ডেন্সি অ্যারে খালি, তাই এটি মাউন্ট হওয়ার সময় একবারই চলবে

  // ৩. রেন্ডারিং লজিক
  if (loading) return <p className="p-4">লোড হচ্ছে...</p>;
  if (!user) return <p className="p-4 text-red-500">ইউজার পাওয়া যায়নি।</p>;

  return (
    <div className="p-5 border rounded-lg shadow-sm bg-white max-w-sm">
      <h1 className="text-xl font-bold text-gray-800">নাম: {user.name}</h1>
      <p className="text-gray-600">ইমেইল: {user.email}</p>
      {user.createdAt && (
        <small className="text-gray-400">
          মেম্বারশিপ: {new Date(user.createdAt).toLocaleDateString()}
        </small>
      )}
    </div>
  );
}
```

---

### ২. রেজিস্ট্রেশন ফর্ম (Create Case - POST)

এখানে আমরা ইউজার থেকে ইনপুট নিয়ে `createUser` সার্ভিসটি ব্যবহার করব।

```tsx
// components/RegistrationForm.tsx
"use client";

import { useState } from "react";
import { UserService, CreateUserDto } from "@/services/user.service";

export default function RegistrationForm() {
  const [formData, setFormData] = useState<CreateUserDto>({
    name: "",
    email: "",
    password: "",
  });
  const [isSubmitting, setIsSubmitting] = useState(false);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    try {
      setIsSubmitting(true);
      // সার্ভিস ব্যবহার করে POST রিকোয়েস্ট পাঠানো
      const result = await UserService.createUser(formData);
      alert(`সফল হয়েছে! ইউজার আইডি: ${result.id}`);
      // ফর্ম রিসেট করা
      setFormData({ name: "", email: "", password: "" });
    } catch (error) {
      alert("রেজিস্ট্রেশন ব্যর্থ হয়েছে! অনুগ্রহ করে আবার চেষ্টা করুন।");
    } finally {
      setIsSubmitting(false);
    }
  };

  return (
    <form
      onSubmit={handleSubmit}
      className="space-y-4 max-w-md p-6 bg-gray-50 rounded-xl"
    >
      <h2 className="text-lg font-semibold">নতুন অ্যাকাউন্ট তৈরি করুন</h2>
      <input
        type="text"
        placeholder="আপনার নাম"
        required
        value={formData.name}
        onChange={(e) => setFormData({ ...formData, name: e.target.value })}
        className="border p-2 w-full rounded focus:ring-2 focus:ring-blue-500 outline-none"
      />
      <input
        type="email"
        placeholder="ইমেইল অ্যাড্রেস"
        required
        value={formData.email}
        onChange={(e) => setFormData({ ...formData, email: e.target.value })}
        className="border p-2 w-full rounded focus:ring-2 focus:ring-blue-500 outline-none"
      />
      <button
        type="submit"
        disabled={isSubmitting}
        className="bg-blue-600 hover:bg-blue-700 text-white font-medium p-2 w-full rounded transition-colors disabled:bg-blue-300"
      >
        {isSubmitting ? "প্রসেসিং..." : "Register Now"}
      </button>
    </form>
  );
}
```

---

### ৩. প্রোফাইল আপডেট (Update Case - PATCH)

ইউজার যখন তার বর্তমান তথ্য পরিবর্তন করতে চায়, তখন আমরা `updateProfile` ব্যবহার করি।

```tsx
// components/EditProfile.tsx
"use client";

import { useState } from "react";
import { UserService } from "@/services/user.service";

export default function EditProfile() {
  const [name, setName] = useState("");
  const [updating, setUpdating] = useState(false);

  const handleUpdate = async () => {
    if (!name.trim()) return alert("নাম খালি রাখা যাবে না।");

    try {
      setUpdating(true);
      // সার্ভিস ব্যবহার করে PATCH রিকোয়েস্ট
      const updatedUser = await UserService.updateProfile({ name });
      alert("প্রোফাইল আপডেট হয়েছে: " + updatedUser.name);
      setName(""); // ফিল্ড ক্লিয়ার করা
    } catch (error) {
      console.error("Update failed", error);
      alert("আপডেট করতে সমস্যা হয়েছে।");
    } finally {
      setUpdating(false);
    }
  };

  return (
    <div className="flex flex-col gap-3 p-4 border rounded-md max-w-xs">
      <label className="text-sm font-medium">নাম পরিবর্তন করুন</label>
      <input
        type="text"
        value={name}
        onChange={(e) => setName(e.target.value)}
        placeholder="নতুন নাম দিন"
        className="border p-2 rounded outline-none focus:border-green-500"
      />
      <button
        onClick={handleUpdate}
        disabled={updating}
        className="bg-green-600 hover:bg-green-700 text-white font-semibold p-2 rounded disabled:opacity-50"
      >
        {updating ? "আপডেট হচ্ছে..." : "Update Name"}
      </button>
    </div>
  );
}
```

---

### পরবর্তী পদক্ষেপ:

এইভাবে কোড করলে আপনার অ্যাপ অনেক বেশি প্রফেশনাল দেখাবে। তবে একটি সমস্যা হলো, আপনাকে প্রতিবার ম্যানুয়ালি `loading`, `error`, এবং `useEffect` হ্যান্ডল করতে হচ্ছে।

আপনি কি চান আমি এখন দেখাই কীভাবে **TanStack Query** ব্যবহার করে এই ৩টি কম্পোনেন্টকে মাত্র অর্ধেক লাইনের কোডে নিয়ে আসা যায়?
