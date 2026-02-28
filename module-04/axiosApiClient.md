### ১. ইন্টারসেপ্টর (Interceptors) কী?

সহজ কথায়, ইন্টারসেপ্টর হলো একটি "চেকপোস্ট"।

- **Request Interceptor:** কোনো রিকোয়েস্ট সার্ভারে যাওয়ার ঠিক আগে তাকে থামিয়ে চেক করে (যেমন: "ইউজারের কি টোকেন আছে? থাকলে সেটা হেডারে বসিয়ে দাও")।
- **Response Interceptor:** সার্ভার থেকে রেসপন্স আসার পর এবং আপনার কোডে পৌঁছানোর আগে তাকে থামিয়ে চেক করে (যেমন: "সার্ভার কি ৪0১ এরর দিয়েছে? তাহলে ইউজারকে লগআউট করাও")।

---

### ২. ফোল্ডার স্ট্রাকচার (Recommended)

```text
src/
├── lib/
│   └── apiClient.ts      (Axios instance এবং interceptors)
├── services/
│   ├── auth.service.ts   (লগইন, রেজিস্ট্রেশন সংক্রান্ত)
│   └── user.service.ts   (ইউজার প্রোফাইল, আপডেট সংক্রান্ত)

```

---

### ৩. প্রফেশনাল ইন্টিগ্রেশন কোড (TypeScript)

`src/api/apiClient.ts` ফাইলে নিচের কোডটি ব্যবহার করুন:

```typescript
import axios, {
  AxiosInstance,
  InternalAxiosRequestConfig,
  AxiosResponse,
} from "axios";

const apiClient: AxiosInstance = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_URL,
  timeout: 15000,
  headers: {
    "Content-Type": "application/json",
  },
});

/**
 * ১. রিকোয়েস্ট ইন্টারসেপ্টর
 * প্রতিবার রিকোয়েস্ট পাঠানোর আগে এটি টোকেন চেক করবে।
 */
apiClient.interceptors.request.use(
  (config: InternalAxiosRequestConfig) => {
    // ব্রাউজারে থাকলে টোকেন নাও (Next.js SSR এর কথা মাথায় রেখে)
    if (typeof window !== "undefined") {
      const token = localStorage.getItem("access_token");
      if (token && config.headers) {
        config.headers.Authorization = `Bearer ${token}`;
      }
    }
    return config;
  },
  (error) => {
    return Promise.reject(error);
  },
);

/**
 * ২. রেসপন্স ইন্টারসেপ্টর
 * সার্ভার থেকে এরর আসলে এখানে গ্লোবালি হ্যান্ডেল করা হয়।
 */
apiClient.interceptors.response.use(
  (response: AxiosResponse) => {
    // রেসপন্স ডাটা ঠিক থাকলে সরাসরি রিটার্ন
    return response;
  },
  async (error) => {
    const originalRequest = error.config;

    // যদি ৪০১ (Unauthorized) এরর আসে এবং আমরা আগে ট্রাই না করে থাকি
    if (error.response?.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true;

      console.log("টোকেন এক্সপায়ার হয়েছে, লগআউট করা হচ্ছে...");

      if (typeof window !== "undefined") {
        localStorage.removeItem("access_token");
        window.location.href = "/login";
      }
    }

    return Promise.reject(error);
  },
);

export default apiClient;
```

---

### ৪. ডেটা ম্যানেজমেন্ট (Services Layer)

`src/services/userService.ts` ফাইলে নিচের কোডটি ব্যবহার করুন:

```typescript
import apiClient from "../api/apiClient";

// ১. টাইপ ডেফিনিশন
export interface UserProfile {
  id: string;
  name: string;
  email: string;
  avatar?: string;
  createdAt?: string;
}

export interface CreateUserDto {
  name: string;
  email: string;
  password?: string;
}

export interface UpdateUserDto {
  name?: string;
  email?: string;
}

// ২. সার্ভিস অবজেক্ট
export const UserService = {
  /**
   * নতুন ইউজার তৈরি করা (POST Request)
   */
  createUser: async (data: CreateUserDto): Promise<UserProfile> => {
    const response = await apiClient.post<UserProfile>("/users", data);
    return response.data;
  },

  /**
   * ইউজারের প্রোফাইল ফেচ করা (GET Request)
   */
  getProfile: async (): Promise<UserProfile> => {
    const response = await apiClient.get<UserProfile>("/users/profile");
    return response.data;
  },

  /**
   * ইউজারের তথ্য আপডেট করা (PATCH Request)
   */
  updateProfile: async (data: UpdateUserDto): Promise<UserProfile> => {
    const response = await apiClient.patch<UserProfile>("/users/update", data);
    return response.data;
  },

  /**
   * নির্দিষ্ট আইডি দিয়ে ইউজার খোঁজা
   */
  getUserById: async (id: string): Promise<UserProfile> => {
    const response = await apiClient.get<UserProfile>(`/users/${id}`);
    return response.data;
  },
};
```
