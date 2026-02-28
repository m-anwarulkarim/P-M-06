**createToken, verifyToken, decodeToken** — এই ৩টা function:

**createToken, verifyToken, decodeToken কী? কেন দরকার?**

আমরা এখানে JWT (JSON Web Token) নিয়ে কথা বলছি।

---

### 🔐 ১️⃣ createToken কী?

##### 👉 কাজ:

নতুন JWT বানায়।

মানে user login করলে আমরা একটা token তৈরি করি।

#### উদাহরণ:

```ts
createToken({ userId: "123", role: "ADMIN" }, "secret_key", "15m");
```

এতে কী হয়?

একটা token তৈরি হয় যেটা দেখতে এরকম:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

#### 🧠 কেন দরকার?

- User login করার পর
- তাকে secureভাবে identify করার জন্য
- API access দেওয়ার জন্য

👉 এটা হলো “ডিজিটাল আইডি কার্ড বানানো”

---

### 🔎 ২️⃣ verifyToken কী?

#### 👉 কাজ:

Token আসল না নকল সেটা চেক করে।

এটা ৩টা জিনিস চেক করে:

1. Secret ঠিক আছে?
2. Token tamper করা হয়েছে?
3. Expire হয়েছে?

#### উদাহরণ:

```ts
verifyToken(token, "secret_key");
```

যদি token ঠিক থাকে → payload return করবে
না হলে → error দিবে

#### 🧠 কেন দরকার?

Protected route এ ঢোকার আগে verify করতে হয়।

```ts
if (!verifyToken(token)) {
  return "Unauthorized";
}
```

👉 এটা হলো “গেটে দাঁড়িয়ে আইডি কার্ড চেক করা”

---

### 📖 ৩️⃣ decodeToken কী?

#### 👉 কাজ:

Token এর ভিতরের data পড়তে পারে
কিন্তু verify করে না ❌

#### উদাহরণ:

```ts
decodeToken(token);
```

এটা শুধু ভিতরের payload দেখায়।

#### 🧠 কেন দরকার?

- Debugging
- Logging
- Inspect করা

⚠️ এটা কখনও security check এর জন্য ব্যবহার করা যাবে না।

👉 এটা হলো “কার্ড দেখে নাম পড়া, কিন্তু আসল না নকল চেক না করা”

---

### 🔥 তিনটার পার্থক্য

| Function    | কী করে          | নিরাপদ?             |
| ----------- | --------------- | ------------------- |
| createToken | Token বানায়     | ✅                  |
| verifyToken | Token যাচাই করে | 🔥 খুব গুরুত্বপূর্ণ |
| decodeToken | শুধু পড়ে        | ❌ verify না        |

### 🎯 একদম সহজ উদাহরণ

ধরো:

- createToken = আইডি কার্ড বানানো
- verifyToken = গেটে আইডি কার্ড চেক করা
- decodeToken = কার্ডে লেখা নাম পড়া

---

### 🚀 সারসংক্ষেপ

- Login করলে → createToken
- Protected route এ ঢোকার আগে → verifyToken
- শুধু ভিতরের data দেখতে চাইলে → decodeToken

এখানে জনপ্রিয় লাইব্রেরি **jsonwebtoken** ব্যবহার করবো।

###### ✅ Step 1: Package Install

```bash
npm install jsonwebtoken
npm install -D @types/jsonwebtoken
```

---

### ✅ Step 2: `token.ts` (Complete Utility File)

```ts
// src/utils/token.ts

import jwt, { JwtPayload, SignOptions } from "jsonwebtoken";

const ACCESS_SECRET = process.env.ACCESS_TOKEN_SECRET as string;
const REFRESH_SECRET = process.env.REFRESH_TOKEN_SECRET as string;

// ================================
// 1️⃣ Create Token
// ================================
export const createToken = (
  payload: object,
  secret: string,
  expiresIn: SignOptions["expiresIn"],
) => {
  if (!secret) {
    throw new Error("JWT secret is missing");
  }

  return jwt.sign(payload, secret, { expiresIn });
};

// ================================
// 2️⃣ Verify Token (Signature + Expiry check)
// ================================
export const verifyToken = <T extends JwtPayload>(
  token: string,
  secret: string,
): T => {
  if (!secret) {
    throw new Error("JWT secret is missing");
  }

  return jwt.verify(token, secret) as T;
};

// ================================
// 3️⃣ Decode Token (No verification)
// ================================
export const decodeToken = (token: string) => {
  return jwt.decode(token);
};
```

---

### ✅ Example Usage (Access + Refresh)

```ts
// Access Token (15 min)
const accessToken = createToken(
  { userId: user.id, role: user.role },
  ACCESS_SECRET,
  "15m",
);

// Refresh Token (7 days)
const refreshToken = createToken({ userId: user.id }, REFRESH_SECRET, "7d");
```

---

### ✅ Verify Example

```ts
try {
  const decoded = verifyToken(accessToken, ACCESS_SECRET);
  console.log(decoded.userId);
} catch (error) {
  console.log("Invalid or expired token");
}
```

---

### ⚠️ Important Difference

| Function    | কী করে                     | Security Level |
| ----------- | -------------------------- | -------------- |
| createToken | JWT বানায়                 | ✅             |
| verifyToken | Signature + Expiry চেক করে | 🔥 খুব নিরাপদ  |
| decodeToken | শুধু payload পড়ে          | ❌ নিরাপদ না   |

⚠️ `decodeToken` কখনো authorization এর জন্য ব্যবহার করো না।

---

### 🎯 Production Tips

- Access token → short (10–15m)
- Refresh token → long (7–30d)
- আলাদা secret ব্যবহার করো
- Refresh token DB তে store করাই best
