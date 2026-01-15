+++
title = "Secure Way to Handle Frontend File Uploads: S3 Presigned URLs"
date = 2026-01-12

[taxonomies]
tags = [ "nextjs","aws","s3"]
+++

Speed up file uploads and lighten your server load by uploading directly to S3 from the nextjs frontend using AWS presigned URLs.

<!--more-->

## Flow Diagram :

![Flow Diagram](/assets/presigned-flow.png)

> _This diagram provides a high-level overview of how the presigned URL upload flow is set up._

## 1. Client requests a presigned URL from the Next.js server

- A presigned URL is generated on the backend using the AWS SDK. The SDK uses AWS credentials from environment variables

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
```

- Quick look of how your environment variables in `.env.local` should look like

```javascript,linenos
AWS_S3_BUCKET_NAME=
AWS_REGION=
AWS_SECRET_ACCESS_KEY=
AWS_ACCESS_KEY_ID=
```

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
            expiresIn: 300, // 5 minutes
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

# Paragraph

this is first paragraph

# Blockquotes

The blockquote element represents content that is quoted from another source, optionally with a citation which must be within a `footer` or `cite` element, and optionally with in-line changes such as annotations and abbreviations.

### Blockquote without attribution

> Tiam, ad mint andaepu dandae nostion secatur sequo quae.
> **Note** that you can use _Markdown syntax_ within a blockquote.

### Blockquote with attribution

> Don't communicate by sharing memory, share memory by communicating.
>
> — <cite>Rob Pike[^1]</cite>

[^1]: The above quote is excerpted from Rob Pik

# Tables

Tables aren't part of the core Markdown spec

| Name  | Age |
| ----- | --- |
| Bob   | 27  |
| Alice | 23  |

### Inline Markdown within tables

| Italics   | Bold     | Code   |
| --------- | -------- | ------ |
| _italics_ | **bold** | `code` |

# Code Blocks

### Inline Code

`This is Inline Code`

### Only `pre`

<pre>
This is pre text
</pre>

### Code block with backticks

```

<!DOCTYPE html>
<html lang="en">
    <head>
        <meta charset="utf-8" />
        <title>Example HTML5 Document</title>
    </head>
    <body>
        <p>Test</p>
    </body>
</html>
```

### Code block with backticks and language specified

### Code block indented with four spaces

    <!doctype html>
    <html lang="en">
    <head>
      <meta charset="utf-8">
      <title>Example HTML5 Document</title>
    </head>
    <body>
      <p>Test</p>
    </body>
    </html>

### Code block with Hugo's internal highlight shortcode

{{< highlight html >}}

<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>Example HTML5 Document</title>
</head>
<body>
  <p>Test</p>
</body>
</html>
{{< /highlight >}}

### Gist

{{< gist spf13 7896402 >}}

## List Types

### Ordered List

1. First item
2. Second item
3. Third item

### Unordered List

- List item
- Another item
- And another item

### Nested list

- Fruit
  - Apple
  - Orange
  - Banana
- Dairy
  - Milk
  - Cheese

# Other Elements — abbr, sub, sup, kbd, mark

<abbr title="Graphics Interchange Format">GIF</abbr> is a bitmap image format.

H<sub>2</sub>O

X<sup>n</sup> + Y<sup>n</sup> = Z<sup>n</sup>

Press <kbd><kbd>CTRL</kbd>+<kbd>ALT</kbd>+<kbd>Delete</kbd></kbd> to end the session.

Most <mark>salamanders</mark> are nocturnal, and hunt for insects, worms, and other small creatures.
