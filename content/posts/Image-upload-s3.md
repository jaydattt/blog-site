+++
title = "Secure Way to Handle Frontend File Uploads: S3 Presigned URLs"
description = "Speed up file uploads and lighten your server load by uploading directly to S3 from the nextjs frontend using AWS presigned URLs."
date = 2026-01-12

keywords = [
  "file upload in Next.js",
  "frontend image upload",
  "upload image to S3 using presigned URL",
  "AWS S3 presigned URL Next.js"
]


[taxonomies]
tags = ["nextjs","aws","s3"]

[extra]
image = "assets/link-preview/presigned-flow.jpg"

+++

<!--more-->

## Why Use Presigned URLs?

Uploading files through your backend can quickly become a bottleneck, large
payloads, slow response times, and unnecessary server costs. A secure and
scalable alternative is uploading files directly from the frontend to Amazon S3
using presigned URLs.

### Benefits

- 🚀 Faster uploads
- 🔒 Strong security guarantees
- 💸 Lower backend costs
- 📈 Infinite scalability

## Flow Diagram :

![Flow Diagram](/assets/presigned-flow.png)

> _This diagram provides a high-level overview of how the presigned URL upload
> flow is set up._

## 1. Client requests a presigned URL from the Next.js server

- A presigned URL is generated on the server using the AWS SDK. The SDK uses AWS
  credentials from environment variables

```javascript,linenos
/*
 * From client component, requesting a presigned S3 URL
 * here we're using Next.js API Routes
 */
const presignedUrl = await fetch("/api/s3/getPresignedUrl", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    fileName: file?.name,
    fileType: file?.type,
    fileSize: file.size,
    folderName: SERVICES_FOLDER_NAME_FOR_S3,
  }),
});

/*
 * Handling response
 */
const presignedData = await presignedUrl.json()

 if (!presignedData.ok) {
  toastError(presignData?.error || "Failed to get upload URL")
  return
}

```

- Quick look of how your environment variables in `.env.local` should look like

```javascript,linenos
/*
 * S3 bucket variables
 */
AWS_S3_BUCKET_NAME=
AWS_REGION=
AWS_SECRET_ACCESS_KEY=
AWS_ACCESS_KEY_ID=
```

> ⚠️ Do not prefix them with NEXT_PUBLIC

## 2. Next.js API route creates a presigned URL using AWS SDK

path: `src/app/api/s3/getPresignedUrl/route.js`

```javascript,linenos
export const runtime = "nodejs";

import { NextResponse } from "next/server";

import { S3Client, PutObjectCommand } from "@aws-sdk/client-s3";
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";


/*
 * Initialize the S3 client.
 *
 * SDK automatically loads up your creds (AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY)
 * from environment variables not need to pass explicitly
 */
const s3 = new S3Client({
  region: process.env.AWS_REGION,
});

/**
 * POST handler to generate a presigned S3 PUT URL.
 *
 * Flow:
 * 1. Client sends file metadata (name, type, size, folder)
 * 2. Server validates input and file size
 * 3. Server generates a time-limited presigned URL
 */
export async function POST(request) {
    try {
        const { fileName, fileType, fileSize, folderName } = await request.json();

        if (!fileName || !fileType || !fileSize || !folderName) {
            return NextResponse.json(
                { error: "Missing required fields" },
                { status: 400 }
            );
        }

        if (fileSize > MAX_FILE_SIZE) {
            return NextResponse.json(
              { error: "File size exceeds 5 MB limit"},
              { status: 400}
            );
        }

        const key = `${folderName}/${Date.now()}_${fileName}`;

        const command = new PutObjectCommand({
            Bucket: process.env.AWS_S3_BUCKET_NAME,
            Key: key,
            ContentType: fileType,
        });

        const uploadUrl = await getSignedUrl(s3, command, {
            expiresIn: 60 * 5, // 5 minutes
        });

        return NextResponse.json({
            uploadUrl,
            key,
        });
    } catch (err) {
        console.error(err);
        return NextResponse.json(
            { error: "Failed to generate presigned URL" },
            { status: 500 }
        );
    }
}

```

## 3. Uploading Image to S3 from client side using that Presigned URL

```javascript,linenos
const uploadResponse = await fetch(uploadUrl, {
    method: "PUT",
    headers: {
     "Content-Type": file.type,
    },
    body: file,
 })

if (!uploadResponse.ok) {
  throw new Error("Upload failed");
}
```

> S3 validates the request against the parameters embedded in the URL, such as
> the expiration time, bucket name, object key, and access key.

> Although your`AWS_ACCESS_KEY_ID`is visible in the presigned URL, your
> `AWS_SECRET_ACCESS_KEY`is never exposed at any point in the process. The
> signature is cryptographically generated on the server and cannot be reversed
> or reused beyond the URL’s expiration time.

🎉At this point, the file is already stored in S3, your backend(if using other
then nextjs) was never involved in the upload itself.

## 4. S3 Bucket Policy & CORS Configuration :

- Make sure to give IAM user a least privilege, grant only what is strictly
  required.

### S3 bucket policy :

```json,linenos
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EnforceHTTPSOnly",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::<your-bucket-name>",
        "arn:aws:s3:::<your-bucket-name>/*"
      ],
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    },
    {
      "Sid": "AllowUploadToSpecificFolders",
      "Effect": "Allow",
      "Principal": "*",
      "Action": ["s3:PutObject", "s3:GetObject"],
      "Resource": "arn:aws:s3:::<your-bucket-name>/*" // you may specify specific folder here
    }
  ]
}
```

> `s3:GetObject`is optional and only required if you also generate presigned
> URLs for downloads

### CORS Configurations of S3 Bucket

```json,linenos
[
    {
        "AllowedHeaders": [
            "Content-Type"
        ],
        "AllowedMethods": [
            "PUT"
        ],
        "AllowedOrigins": [
            "http://localhost:3000",
            "https://your-domain.com"
        ],
        "ExposeHeaders": [
            "ETag"
        ],
        "MaxAgeSeconds": 3000
    }
]
```

- `PUT`is required for uploads
- Restrict`AllowedOrigins`to trusted domains only
- `ExposeHeaders`is optional but useful for debugging uploads

## Final Thoughts

Direct uploads to S3 using presigned URLs are one of the most secure and
scalable patterns for handling files in modern web applications.

If you’re building a Next.js application that handles user-generated content,
this should be your default upload strategy.

Happy shipping and keep your buckets locked down 🔐
