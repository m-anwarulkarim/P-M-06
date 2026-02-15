**Express + TS + Prisma + Postgres + Multer + Cloudinary + multer-storage-cloudinary** দিয়ে **image + PDF upload** (single + multiple + delete + get by id) সহ।

> ধরলাম তোমার `app.ts` থেকে তুমি `/api/files` রাউট মাউন্ট করবে।
> ধরলাম তোমার error handler / AppError আগে থেকেই আছে (তাই `error/` ফোল্ডার দেইনি)।

---

## 1) `src/config/env.ts`

```ts
// src/config/env.ts
import "dotenv/config";
import { z } from "zod";

const envSchema = z.object({
  NODE_ENV: z.string().optional().default("development"),

  DATABASE_URL: z.string().min(1),

  CLOUDINARY_CLOUD_NAME: z.string().min(1),
  CLOUDINARY_API_KEY: z.string().min(1),
  CLOUDINARY_API_SECRET: z.string().min(1),

  UPLOAD_MAX_MB: z
    .string()
    .optional()
    .transform((v) => {
      const n = Number(v ?? "10");
      return Number.isFinite(n) && n > 0 ? n : 10;
    }),
});

const parsed = envSchema.safeParse(process.env);

if (!parsed.success) {
  // eslint-disable-next-line no-console
  console.error(
    "❌ Invalid environment variables:",
    parsed.error.flatten().fieldErrors,
  );
  throw new Error("Invalid environment variables");
}

export const env = parsed.data;
```

---

## 2) `src/config/cloudinary.ts`

```ts
// src/config/cloudinary.ts
import { v2 as cloudinary } from "cloudinary";
import { env } from "./env";

cloudinary.config({
  cloud_name: env.CLOUDINARY_CLOUD_NAME,
  api_key: env.CLOUDINARY_API_KEY,
  api_secret: env.CLOUDINARY_API_SECRET,
});

export { cloudinary };
```

---

## 3) `src/lib/prisma.ts`

```ts
// src/lib/prisma.ts
import { PrismaClient } from "@prisma/client";

declare global {
  // eslint-disable-next-line no-var
  var prisma: PrismaClient | undefined;
}

export const prisma =
  global.prisma ??
  new PrismaClient({
    log: ["error", "warn"],
  });

if (process.env.NODE_ENV !== "production") global.prisma = prisma;
```

---

## 4) `src/middlewares/upload/validators.ts`

```ts
// src/middlewares/upload/validators.ts
import status from "http-status";

// তুমি চাইলে এখানে তোমার AppError use করতে পারো
export class UploadError extends Error {
  statusCode: number;
  constructor(statusCode: number, message: string) {
    super(message);
    this.statusCode = statusCode;
  }
}

export const ALLOWED_IMAGE_MIME = [
  "image/jpeg",
  "image/png",
  "image/webp",
  "image/gif",
];

export const ALLOWED_PDF_MIME = ["application/pdf"];

export const isAllowedMime = (mimeType: string) => {
  return [...ALLOWED_IMAGE_MIME, ...ALLOWED_PDF_MIME].includes(mimeType);
};

export const getFileKind = (mimeType: string) => {
  if (ALLOWED_IMAGE_MIME.includes(mimeType)) return "image";
  if (ALLOWED_PDF_MIME.includes(mimeType)) return "pdf";
  throw new UploadError(status.BAD_REQUEST, "Unsupported file type");
};

export const normalizeFolder = (folder?: string) => {
  const f = (folder ?? "app/uploads").trim();
  // basic sanitize
  return f
    .replace(/(\.\.)|[<>:"|?*]/g, "")
    .replace(/\\/g, "/")
    .replace(/\/+/g, "/");
};
```

---

## 5) `src/middlewares/upload/multer.ts`

✅ এখানে **multer-storage-cloudinary** দিয়ে upload হবে।
✅ image হলে `resource_type: "image"` + optional transformations
✅ pdf হলে `resource_type: "raw"`

```ts
// src/middlewares/upload/multer.ts
import multer from "multer";
import { CloudinaryStorage } from "multer-storage-cloudinary";
import type { Request } from "express";
import { cloudinary } from "../../config/cloudinary";
import { env } from "../../config/env";
import {
  getFileKind,
  isAllowedMime,
  normalizeFolder,
  UploadError,
} from "./validators";
import status from "http-status";
import { v4 as uuid } from "uuid";

const maxBytes = env.UPLOAD_MAX_MB * 1024 * 1024;

type UploadParams = {
  folder?: string;
  type?: "image" | "pdf";
};

const getParams = (req: Request, file: Express.Multer.File) => {
  // folder client থেকে আসলে: req.body.folder
  const bodyFolder =
    typeof req.body?.folder === "string" ? req.body.folder : undefined;
  const folder = normalizeFolder(bodyFolder);

  const kind = getFileKind(file.mimetype);

  // Cloudinary params dynamic
  if (kind === "image") {
    return {
      folder: `${folder}/images`,
      resource_type: "image" as const,
      public_id: `${uuid()}`,
      format: "webp", // চাইলে remove করো (original format রাখবে)
      transformation: [{ quality: "auto" }, { fetch_format: "auto" }],
    };
  }

  // PDF / raw
  return {
    folder: `${folder}/pdfs`,
    resource_type: "raw" as const,
    public_id: `${uuid()}`,
  };
};

const storage = new CloudinaryStorage({
  cloudinary,
  params: async (req, file) => getParams(req, file),
});

const fileFilter: multer.Options["fileFilter"] = (req, file, cb) => {
  if (!isAllowedMime(file.mimetype)) {
    return cb(
      new UploadError(
        status.BAD_REQUEST,
        "Only image or PDF files are allowed",
      ),
    );
  }
  cb(null, true);
};

export const upload = multer({
  storage,
  fileFilter,
  limits: { fileSize: maxBytes },
});

export const uploadSingle = (fieldName = "file") => upload.single(fieldName);
export const uploadMultiple = (fieldName = "files", maxCount = 10) =>
  upload.array(fieldName, maxCount);

// Multi-field example (যদি দরকার হয়)
// export const uploadFields = upload.fields([
//   { name: "avatar", maxCount: 1 },
//   { name: "documents", maxCount: 5 },
// ]);
```

---

## 6) `src/modules/file/file.interface.ts`

```ts
// src/modules/file/file.interface.ts

export type FileKind = "image" | "pdf";

export interface CreateFileAssetPayload {
  url: string;
  publicId: string;
  resourceType: string;
  format?: string | null;
  bytes?: number | null;
  folder?: string | null;
  originalName?: string | null;
  mimeType?: string | null;
  uploadedById?: string | null;
}
```

---

## 7) `src/modules/file/file.validation.ts`

> এখানে শুধু request body এর optional fields validate করছি (folder ইত্যাদি)
> multer file validate আগেই করছে।

```ts
// src/modules/file/file.validation.ts
import { z } from "zod";

export const uploadBodySchema = z.object({
  folder: z.string().min(1).optional(),
});

export const uploadMultipleBodySchema = z.object({
  folder: z.string().min(1).optional(),
});

export const deleteFileParamsSchema = z.object({
  id: z.string().uuid(),
});

export const getFileParamsSchema = z.object({
  id: z.string().uuid(),
});
```

---

## 8) `src/modules/file/file.service.ts`

✅ multer-storage-cloudinary upload করার পর `req.file` / `req.files` এ Cloudinary info থাকবে
✅ DB save
✅ delete: Cloudinary destroy + DB delete
✅ ownership check optional (তোমার auth middleware থেকে userId আনতে পারো)

```ts
// src/modules/file/file.service.ts
import status from "http-status";
import { prisma } from "../../lib/prisma";
import { cloudinary } from "../../config/cloudinary";
import type { CreateFileAssetPayload } from "./file.interface";

// তোমার AppError থাকলে এটা replace করে দিও
class AppError extends Error {
  statusCode: number;
  constructor(statusCode: number, message: string) {
    super(message);
    this.statusCode = statusCode;
  }
}

type CloudinaryMulterFile = Express.Multer.File & {
  path?: string; // cloudinary secure url often mapped to "path"
  filename?: string; // cloudinary public_id mapped to "filename"
};

const mapMulterFileToDBPayload = (
  file: CloudinaryMulterFile,
  uploadedById?: string | null,
): CreateFileAssetPayload => {
  const url = (file.path ?? "") as string;
  const publicId = (file.filename ?? "") as string;

  if (!url || !publicId) {
    throw new AppError(
      status.BAD_REQUEST,
      "Upload failed: missing cloudinary file info",
    );
  }

  return {
    url,
    publicId,
    resourceType: (file as any).resource_type ?? "image", // storage params থেকে আসতে পারে
    format: (file as any).format ?? null,
    bytes: typeof file.size === "number" ? file.size : null,
    folder: (file as any).folder ?? null,
    originalName: file.originalname ?? null,
    mimeType: file.mimetype ?? null,
    uploadedById: uploadedById ?? null,
  };
};

export const FileService = {
  async createSingle(file: CloudinaryMulterFile, uploadedById?: string) {
    const payload = mapMulterFileToDBPayload(file, uploadedById ?? null);

    const created = await prisma.fileAsset.create({
      data: payload,
    });

    return created;
  },

  async createMany(files: CloudinaryMulterFile[], uploadedById?: string) {
    if (!files.length) throw new AppError(status.BAD_REQUEST, "No files found");

    // createMany return count only — তাই loop create (or transaction)
    const created = await prisma.$transaction(
      files.map((f) =>
        prisma.fileAsset.create({
          data: mapMulterFileToDBPayload(f, uploadedById ?? null),
        }),
      ),
    );

    return created;
  },

  async getById(id: string) {
    const file = await prisma.fileAsset.findUnique({ where: { id } });
    if (!file) throw new AppError(status.NOT_FOUND, "File not found");
    return file;
  },

  async deleteById(id: string, requesterId?: string) {
    const file = await prisma.fileAsset.findUnique({ where: { id } });
    if (!file) throw new AppError(status.NOT_FOUND, "File not found");

    // Ownership check (optional) — তোমার auth থাকলে requesterId পাঠাও
    if (requesterId && file.uploadedById && file.uploadedById !== requesterId) {
      throw new AppError(
        status.FORBIDDEN,
        "You are not allowed to delete this file",
      );
    }

    // cloudinary destroy: raw/image আলাদা resource_type লাগতে পারে
    // Prisma তে resourceType স্টোর করেছি: "image" / "raw"
    const resourceType = file.resourceType === "raw" ? "raw" : "image";

    await cloudinary.uploader.destroy(file.publicId, {
      resource_type: resourceType,
    });

    await prisma.fileAsset.delete({ where: { id } });

    return { success: true };
  },
};
```

---

## 9) `src/modules/file/file.controller.ts`

```ts
// src/modules/file/file.controller.ts
import type { Request, Response, NextFunction } from "express";
import status from "http-status";
import { FileService } from "./file.service";
import {
  uploadBodySchema,
  uploadMultipleBodySchema,
  deleteFileParamsSchema,
  getFileParamsSchema,
} from "./file.validation";

// helper: তোমার auth middleware থাকলে req.user থেকে userId আনবে
const getUserId = (req: Request) => {
  // উদাহরণ: (req as any).user?.id
  return (req as any)?.user?.id as string | undefined;
};

export const FileController = {
  async uploadSingle(req: Request, res: Response, next: NextFunction) {
    try {
      uploadBodySchema.parse(req.body);

      const file = req.file as any;
      if (!file) {
        return res.status(status.BAD_REQUEST).json({
          success: false,
          message: "No file provided",
        });
      }

      const userId = getUserId(req);
      const created = await FileService.createSingle(file, userId);

      return res.status(status.CREATED).json({
        success: true,
        message: "File uploaded successfully",
        data: created,
      });
    } catch (err) {
      next(err);
    }
  },

  async uploadMultiple(req: Request, res: Response, next: NextFunction) {
    try {
      uploadMultipleBodySchema.parse(req.body);

      const files = (req.files ?? []) as any[];
      if (!files.length) {
        return res.status(status.BAD_REQUEST).json({
          success: false,
          message: "No files provided",
        });
      }

      const userId = getUserId(req);
      const created = await FileService.createMany(files, userId);

      return res.status(status.CREATED).json({
        success: true,
        message: "Files uploaded successfully",
        data: created,
      });
    } catch (err) {
      next(err);
    }
  },

  async getById(req: Request, res: Response, next: NextFunction) {
    try {
      const { id } = getFileParamsSchema.parse(req.params);

      const file = await FileService.getById(id);

      return res.status(status.OK).json({
        success: true,
        data: file,
      });
    } catch (err) {
      next(err);
    }
  },

  async deleteById(req: Request, res: Response, next: NextFunction) {
    try {
      const { id } = deleteFileParamsSchema.parse(req.params);

      const userId = getUserId(req);
      const result = await FileService.deleteById(id, userId);

      return res.status(status.OK).json({
        success: true,
        message: "File deleted successfully",
        data: result,
      });
    } catch (err) {
      next(err);
    }
  },
};
```

---

## 10) `src/modules/file/file.route.ts`

```ts
// src/modules/file/file.route.ts
import { Router } from "express";
import { uploadMultiple, uploadSingle } from "../../middlewares/upload/multer";
import { FileController } from "./file.controller";

const router = Router();

/**
 * Single file upload
 * form-data: file (field name = "file")
 * optional: folder
 */
router.post("/upload", uploadSingle("file"), FileController.uploadSingle);

/**
 * Multiple file upload
 * form-data: files (field name = "files")
 * optional: folder
 */
router.post(
  "/upload/multiple",
  uploadMultiple("files", 10),
  FileController.uploadMultiple,
);

/**
 * get file metadata
 */
router.get("/:id", FileController.getById);

/**
 * delete file (cloudinary + db)
 */
router.delete("/:id", FileController.deleteById);

export const FileRoutes = router;
```

---

# 11) `app.ts` এ কীভাবে মাউন্ট করবে (শুধু নির্দেশনা)

তুমি বলেছ `app.ts` দিব না—তাই শুধু বলছি:

```ts
// app.ts এর ভিতরে
app.use("/api/files", FileRoutes);
```

---

# 12) Postman / Client টেস্ট

### Single

- Method: `POST /api/files/upload`
- Body: `form-data`
  - key: `file` (type file) → image/pdf
  - key: `folder` (text) → `medi-store` (optional)

### Multiple

- Method: `POST /api/files/upload/multiple`
- Body: `form-data`
  - key: `files` (type file) → multiple select
  - `folder` optional

project-root/
│
├── prisma/
│ ├── schema.prisma
│ └── migrations/
│
├── src/
│
│ ├── app.ts
│ ├── server.ts
│
│ ├── config/
│ │ ├── env.ts
│ │ └── cloudinary.ts
│ │
│ ├── lib/
│ │ └── prisma.ts
│ │
│ ├── interfaces/
│ │ └── index.d.ts
│ │
│ ├── utils/
│ │ ├── catchAsync.ts
│ │ ├── sendResponse.ts
│ │ └── helpers.ts
│ │
│ ├── error/
│ │ ├── AppError.ts
│ │ ├── globalErrorHandler.ts
│ │ └── handleValidationError.ts
│ │
│ ├── middlewares/
│ │
│ │ ├── auth/
│ │ │ └── auth.ts
│ │ │
│ │ ├── upload/
│ │ │ ├── multer.ts
│ │ │ └── validators.ts
│ │ │
│ │ ├── validateRequest.ts
│ │ └── notFound.ts
│ │
│ ├── routes/
│ │ └── index.ts
│ │
│ └── modules/
│
│ ├── file/
│ │ ├── file.interface.ts
│ │ ├── file.validation.ts
│ │ ├── file.service.ts
│ │ ├── file.controller.ts
│ │ └── file.route.ts
│ │
│ ├── auth/
│ │ ├── auth.interface.ts
│ │ ├── auth.validation.ts
│ │ ├── auth.service.ts
│ │ ├── auth.controller.ts
│ │ └── auth.route.ts
│ │
│ ├── user/
│ │ ├── user.interface.ts
│ │ ├── user.validation.ts
│ │ ├── user.service.ts
│ │ ├── user.controller.ts
│ │ └── user.route.ts
│ │
│ └── medicine/
│ ├── medicine.interface.ts
│ ├── medicine.validation.ts
│ ├── medicine.service.ts
│ ├── medicine.controller.ts
│ └── medicine.route.ts
│
├── .env
├── package.json
├── tsconfig.json
└── README.md

```

```
