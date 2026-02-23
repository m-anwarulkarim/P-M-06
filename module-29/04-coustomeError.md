**Custom Error Class** বানানোর **ফুল, রিয়েল-প্রজেক্ট টাইপ উদাহরণ**

## ✅ কেন Custom Error Class লাগে?

`throw new Error("msg")` দিলে তুমি শুধু message পাওয়া যায় কিন্তু রিয়েল প্রজেক্টে আরো অনেক কিছুলাগতে পারে:

- **statusCode** (401, 404, 500)
- **errorCode** (USER_NOT_FOUND, INVALID_INPUT)
- **details** (কোন field ভুল, কোন value)
- **isOperational** (expected error নাকি crash-type)

এই সব রাখতে Custom Error Class best.

---

## ✅ 1) Basic Custom Error Class (Simple)

```js
class AppError extends Error {
  constructor(message) {
    super(message); // Error এর message সেট করে
    this.name = "AppError"; // error এর নাম
  }
}

try {
  throw new AppError("Something bad happened!");
} catch (err) {
  console.log(err.name); // AppError
  console.log(err.message); // Something bad happened!
  console.log(err.stack); // কোথায় error হলো তার trace
}
```

### 🔍 ব্যাখ্যা

- `extends Error` মানে: Error এর সব ক্ষমতা তোমার ক্লাসে চলে আসবে
- `super(message)` না দিলে `message` ঠিকভাবে বসবে না
- `err.stack` automatically থাকবে

---

## ✅ 2) Real Project Custom Error (statusCode + code + details)

এটা Express/Node API তে সবচেয়ে বেশি কাজে লাগে।

```js
class AppError extends Error {
  constructor(statusCode, message, code = "APP_ERROR", details = null) {
    super(message);

    this.name = "AppError";
    this.statusCode = statusCode;
    this.code = code;
    this.details = details;

    // stack trace clean রাখে (optional but useful)
    Error.captureStackTrace(this, this.constructor);
  }
}
```

### 🔍 এখানে কী হচ্ছে?

- `statusCode`: HTTP status (400/401/404/500)
- `code`: তোমার own error code (যেমন: `USER_NOT_FOUND`)
- `details`: extra info (কোন field ভুল, etc)
- `Error.captureStackTrace`: stack এ অপ্রয়োজনীয় constructor line কম দেখায়

---

## ✅ 3) Use case: Validation error throw

```js
function createUser(payload) {
  if (!payload.email) {
    throw new AppError(400, "Email is required", "VALIDATION_ERROR", {
      field: "email",
    });
  }

  if (!payload.password || payload.password.length < 6) {
    throw new AppError(
      400,
      "Password must be at least 6 characters",
      "VALIDATION_ERROR",
      { field: "password", minLength: 6 },
    );
  }

  return { id: 1, ...payload };
}

try {
  createUser({ email: "", password: "123" });
} catch (err) {
  console.log("Name:", err.name);
  console.log("Status:", err.statusCode);
  console.log("Code:", err.code);
  console.log("Message:", err.message);
  console.log("Details:", err.details);
}
```

### 🔍 ব্যাখ্যা (গুরুত্বপূর্ণ)

- route এ error হলে `next(err)` করা হয়
- তারপর global handler সব error এক জায়গায় handle করে
- `instanceof AppError` দিয়ে চেক করা হয় এটা আমাদের custom error কিনা
- unknown error হলে 500 দেয়, আর stack log করে

---

## ✅ Bonus: shortcut function (আরও clean)

```js
const badRequest = (message, details) =>
  new AppError(400, message, "BAD_REQUEST", details);

throw badRequest("Invalid input", { field: "email" });
```

---

## mini-challenge (practice)

একটা function বানাও `login({email, password})`:

- email না থাকলে `AppError(400, ..., VALIDATION_ERROR)`
- password ভুল হলে `AppError(401, ..., INVALID_CREDENTIALS)`
- ঠিক থাকলে `success`
