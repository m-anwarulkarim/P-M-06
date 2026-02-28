### TypeScript কোড (Bun & Node.js Compatible)

```
src/
│
├── config/
│   └── cloudinary.ts        ← cloudinary config only
│
├── middlewares/
│   └── upload/
│       └── multer.ts        ← multer + storage + fileFilter
│
├── modules/
│   └── file/
│       ├── file.controller.ts
│       └── file.route.ts
│
└── routes/
    └── index.ts
```

```bash
bun add multer cloudinary multer-storage-cloudinary
bun add -d @types/multer

```

# 🧩 1️⃣ `src/config/cloudinary.ts`

```ts
import { v2 as cloudinary } from "cloudinary";
// ১. Cloudinary কনফিগারেশন
// এই ভ্যালুগুলো অবশ্যই .env ফাইল থেকে আসা উচিত
cloudinary.config({
  cloud_name: process.env.CLOUDINARY_NAME,
  api_key: process.env.CLOUDINARY_API_KEY,
  api_secret: process.env.CLOUDINARY_API_SECRET,
});

export default cloudinary;
```

# 🧩 2️⃣ `src/middlewares/upload/multer.ts`

```ts
import { CloudinaryStorage } from "multer-storage-cloudinary";
import multer from "multer";
import { Request } from "express";
import cloudinary from "../../config/cloudinary";

// ২. স্টোরেজ ইঞ্জিন সেটআপ
// এখানে আমরা ফোল্ডার এবং অ্যালাউড ফরম্যাট ডিফাইন করে দিচ্ছি
/**
 * Cloudinary Storage Setup
 * @purpose: সরাসরি ক্লাউডিনারিতে ফাইল আপলোড করা এবং ফাইল ম্যানেজমেন্ট সহজ করা।
 */
export const storage = new CloudinaryStorage({
  cloudinary: cloudinary,
  params: async (req: Request, file: Express.Multer.File) => {
    // ১. ফাইলের এক্সটেনশন আলাদা করা (যেমন: '.png')
    const fileExtension = path.extname(file.originalname);

    // ২. এক্সটেনশন বাদে ফাইলের মূল নাম বের করা (যেমন: 'my-photo')
    // আমরা .split('.') এর বদলে path.basename ব্যবহার করি কারণ এটি বেশি রিলায়েবল
    const fileNameWithoutExt = path.basename(file.originalname, fileExtension);

    // ৩. ফাইল নেম ক্লিনিং (SEO এবং URL Friendly করার জন্য)
    // স্পেস সরিয়ে '-' দেওয়া এবং সব ছোট হাতের অক্ষরে রূপান্তর করা
    const cleanFileName = fileNameWithoutExt
      .replace(/\s+/g, "-") // সব স্পেসকে ড্যাশ দিয়ে রিপ্লেস
      .replace(/[^a-zA-Z0-9-]/g, "") // শুধুমাত্র লেটার, নাম্বার এবং ড্যাশ রাখা
      .toLowerCase();

    // ৪. ডাইনামিক ফরম্যাট হ্যান্ডলিং (image/jpeg -> jpeg)
    // Cloudinary 'format' ফিল্ড থেকে স্বয়ংক্রিয়ভাবে এক্সটেনশন বসিয়ে নেয়
    const rawFormat = file.mimetype.split("/")[1];

    // কিছু কিছু ক্ষেত্রে mimetype 'svg+xml' হতে পারে, সেটাকে ক্লিন করা
    const finalFormat = rawFormat === "svg+xml" ? "svg" : rawFormat;

    return {
      folder: "my_app_uploads", // ক্লাউডিনারি ফোল্ডারের নাম

      format: finalFormat, // এখানে ফরম্যাট বলে দিলে public_id তে এক্সটেনশন লাগে না

      // ইউনিক পাবলিক আইডি তৈরি (টাইমস্ট্যাম্প + ক্লিন নেম)
      // এতে একই নামের ফাইল আপলোড হলেও একেকটার আইডি আলাদা হবে
      public_id: `${Date.now()}-${cleanFileName}`,

      // ফাইল আপলোডের সময় কিছু বেসিক ট্রান্সফরমেশন (অপশনাল কিন্তু রিকমেন্ডেড)
      transformation: [
        { quality: "auto", fetch_format: "auto" }, // অটোমেটিক ইমেজ অপটিমাইজেশন
      ],
    };
  },
});

const fileFilter = (
  req: Request,
  file: Express.Multer.File,
  cb: multer.FileFilterCallback,
) => {
  if (file.mimetype.startsWith("image/")) {
    cb(null, true);
  } else {
    cb(new Error("ভুল ফরম্যাট! শুধুমাত্র ইমেজ আপলোড করা যাবে।") as any, false);
  }
};

// ৪. মূল Multer ইনস্ট্যান্স
export const upload = multer({
  storage,
  fileFilter,
  limits: {
    fileSize: 5 * 1024 * 1024, // সর্বোচ্চ ৫ মেগাবাইট
  },
});
```

# 🧩 3️⃣ `src/modules/file/file.controller.ts`

```ts
import { Request, Response } from "express";

export const uploadImageController = (req: Request, res: Response) => {
  try {
    if (!req.file) {
      return res.status(400).json({
        message: "দয়া করে একটি ফাইল সিলেক্ট করুন।",
      });
    }

    // ক্লাউডিনারি থেকে আসা ফাইলের URL
    const imageUrl = (req.file as any).path;

    return res.status(200).json({
      success: true,
      message: "ফাইল সফলভাবে আপলোড হয়েছে!",
      url: imageUrl,
    });
  } catch (error: any) {
    return res.status(500).json({
      success: false,
      message: error.message || "সার্ভার এরর!",
    });
  }
};
```

---

### কীভাবে রান করবেন?

১. **প্যাকেজ ইন্সটল (Bun দিয়ে):**
যেহেতু আপনি Bun ব্যবহার করছেন, তাই Bun-এর নেটিভ প্যাকেজ ম্যানেজার ব্যবহার করা ভালো:

```bash
bun add multer cloudinary multer-storage-cloudinary
bun add -d @types/multer
```

২. **এনভায়রনমেন্ট ভ্যারিয়েবল:**
আপনার প্রোজেক্টের রুটে একটি `.env` ফাইল তৈরি করুন:

```env
CLOUDINARY_NAME=your_name
CLOUDINARY_API_KEY=your_key
CLOUDINARY_API_SECRET=your_secret

```

৩. **রাউট সেটআপ:**

```typescript
import express from "express";
import { upload, uploadImageController } from "./upload.service";

const router = express.Router();

// 'image' হলো সেই কি (key) যা আপনি Postman বা Frontend থেকে পাঠাবেন
router.post("/upload", upload.single("image"), uploadImageController);

export default router;
```

=======================

---

### ১. ফোল্ডার স্ট্রাকচার (Folder Structure)

```text
src/
├── config/         # Cloudinary & Prisma কনফিগারেশন
├── controllers/    # রিকোয়েস্ট হ্যান্ডলিং লজিক
├── routes/         # API রাউটস
├── services/       # বিজনেস লজিক এবং ডেটাবেস অপারেশনস
├── interfaces/     # টাইপস্ক্রিপ্ট ইন্টারফেস
└── app.ts          # এক্সপ্রেস মেইন ফাইল

```

---

### ২. পূর্ণাঙ্গ কোড ইমপ্লিমেন্টেশন

#### ক) Prisma Schema (`prisma/schema.prisma`)

```prisma
model File {
  id        String   @id @default(uuid())
  url       String   // ক্লাউডিনারি ইউআরএল
  publicId  String   // ক্লাউডিনারি আইডি (ডিলিট করার জন্য)
  fileName  String   // অরিজিনাল ফাইল নেম
  createdAt DateTime @default(now())
}

```

#### খ) Cloudinary & Multer Config (`src/config/upload.config.ts`)

```typescript
import { v2 as cloudinary } from "cloudinary";
import { CloudinaryStorage } from "multer-storage-cloudinary";
import multer from "multer";
import path from "path";

cloudinary.config({
  cloud_name: process.env.CLOUDINARY_NAME,
  api_key: process.env.CLOUDINARY_API_KEY,
  api_secret: process.env.CLOUDINARY_API_SECRET,
});

const storage = new CloudinaryStorage({
  cloudinary: cloudinary,
  params: async (req, file) => {
    const fileExt = path.extname(file.originalname);
    const cleanName = path
      .basename(file.originalname, fileExt)
      .replace(/\s+/g, "-");
    return {
      folder: "my_app_uploads",
      format: file.mimetype.split("/")[1],
      public_id: `${Date.now()}-${cleanName}`,
    };
  },
});

export const upload = multer({
  storage,
  limits: { fileSize: 5 * 1024 * 1024 }, // ৫ এমবি লিমিট
});

export { cloudinary };
```

#### গ) Service Layer (`src/services/file.service.ts`)

এখানে CRUD (Create, Read, Update, Delete) লজিক রাখা হয়েছে।

```typescript
import { PrismaClient } from "@prisma/client";
import { cloudinary } from "../config/upload.config";

const prisma = new PrismaClient();

export const FileService = {
  // ১. ডেটাবেসে ফাইল সেভ করা (Add)
  async saveFile(url: string, publicId: string, fileName: string) {
    return await prisma.file.create({
      data: { url, publicId, fileName },
    });
  },

  // ২. সব ফাইল গেট করা (Get All)
  async getAllFiles() {
    return await prisma.file.findMany({ orderBy: { createdAt: "desc" } });
  },

  // ৩. নির্দিষ্ট ফাইল ডিলিট করা (Delete)
  async deleteFile(id: string) {
    const file = await prisma.file.findUnique({ where: { id } });
    if (!file) throw new Error("ফাইল পাওয়া যায়নি!");

    // ক্লাউডিনারি থেকে ডিলিট
    await cloudinary.uploader.destroy(file.publicId);

    // ডেটাবেস থেকে ডিলিট
    return await prisma.file.delete({ where: { id } });
  },

  // ৪. ফাইল আপডেট করা (Update)
  // লজিক: আগেরটি ডিলিট করে নতুনটি ক্লাউডিনারিতে আপলোড হয় (Multer এটি অলরেডি করেছে)
  async updateFileRecord(
    id: string,
    newUrl: string,
    newPublicId: string,
    newFileName: string,
  ) {
    const oldFile = await prisma.file.findUnique({ where: { id } });
    if (oldFile) {
      await cloudinary.uploader.destroy(oldFile.publicId); // পুরনো ফাইল ডিলিট
    }
    return await prisma.file.update({
      where: { id },
      data: { url: newUrl, publicId: newPublicId, fileName: newFileName },
    });
  },
};
```

#### ঘ) Controller Layer (`src/controllers/file.controller.ts`)

```typescript
import { Request, Response } from "express";
import { FileService } from "../services/file.service";

export const FileController = {
  // ফাইল আপলোড ও সেভ
  async uploadFile(req: Request, res: Response) {
    try {
      if (!req.file) throw new Error("ফাইল আপলোড হয়নি!");
      const file = await FileService.saveFile(
        (req.file as any).path,
        (req.file as any).filename,
        req.file.originalname,
      );
      res.status(201).json({ success: true, data: file });
    } catch (error: any) {
      res.status(400).json({ success: false, message: error.message });
    }
  },

  // সব ফাইল দেখা
  async getFiles(req: Request, res: Response) {
    const files = await FileService.getAllFiles();
    res.json({ success: true, data: files });
  },

  // ফাইল ডিলিট করা
  async removeFile(req: Request, res: Response) {
    try {
      await FileService.deleteFile(req.params.id);
      res.json({ success: true, message: "ফাইল ডিলিট সফল!" });
    } catch (error: any) {
      res.status(404).json({ success: false, message: error.message });
    }
  },

  // ফাইল আপডেট (Replace)
  async updateFile(req: Request, res: Response) {
    try {
      if (!req.file) throw new Error("নতুন ফাইল প্রয়োজন!");
      const updated = await FileService.updateFileRecord(
        req.params.id,
        (req.file as any).path,
        (req.file as any).filename,
        req.file.originalname,
      );
      res.json({ success: true, data: updated });
    } catch (error: any) {
      res.status(400).json({ success: false, message: error.message });
    }
  },
};
```

#### ঙ) Routes (`src/routes/file.route.ts`)

```typescript
import express from "express";
import { upload } from "../config/upload.config";
import { FileController } from "../controllers/file.controller";

const router = express.Router();

router.post("/upload", upload.single("image"), FileController.uploadFile);
router.get("/files", FileController.getFiles);
router.delete("/files/:id", FileController.removeFile);
router.put("/files/:id", upload.single("image"), FileController.updateFile);

export default router;
```

---

### ৩. কীভাবে রান করবেন ও টেস্ট করবেন?

1. **Environment Setup:** `.env` ফাইলে `DATABASE_URL`, `CLOUDINARY_NAME`, `CLOUDINARY_API_KEY`, `CLOUDINARY_API_SECRET` সেট করুন।
2. **Prisma Migration:** `npx prisma migrate dev --name init` রান করুন।
3. **Postman Testing:**

- **Add:** `POST` `/api/v1/upload` (Body: form-data, Key: image)
- **Get:** `GET` `/api/v1/files`
- **Delete:** `DELETE` `/api/v1/files/YOUR_ID`
- **Update:** `PUT` `/api/v1/files/YOUR_ID` (Body: form-data, Key: image)
