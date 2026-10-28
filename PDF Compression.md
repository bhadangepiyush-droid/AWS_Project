# AWS Serverless PDF Compression Pipeline

A serverless PDF compression workflow built with **Amazon S3**, **AWS Lambda**, and **PyMuPDF**. When a PDF is uploaded to the source S3 bucket, an S3 `ObjectCreated` event triggers the Lambda function. The function attempts structural optimization and stores the smaller result in a separate destination bucket. If optimization makes the file larger, the original is retained.

## Architecture

```text
PDF Upload
    |
    v
Amazon S3 (Source Bucket)
    |
    | ObjectCreated event
    v
AWS Lambda (PDF-Compressor-Function)
    |
    | PyMuPDF optimization
    | Compare original and optimized sizes
    v
Amazon S3 (Destination Bucket)
    |
    v
Processed PDF
```

## AWS Services and Resources

| Resource | Example name | Purpose |
|---|---|---|
| Source S3 bucket | `student-pdf-source-unique` | Receives original PDFs |
| Destination S3 bucket | `student-pdf-output-unique` | Stores processed PDFs |
| Lambda function | `PDF-Compressor-Function` | Runs the compression workflow |
| Lambda layer | `pymupdf-layer` | Provides the PyMuPDF dependency |
| Lambda execution role | `pdf-compressor-lambda-role` | Allows CloudWatch logging and required S3 access |

Use globally unique S3 bucket names. Keep both buckets, the Lambda function, and the layer in the same AWS Region for this lab.

## How It Works

1. A user uploads a PDF to the source S3 bucket.
2. Amazon S3 emits an `ObjectCreated` event.
3. The event invokes `PDF-Compressor-Function`.
4. Lambda downloads the source PDF and attempts structural optimization using PyMuPDF.
5. The function compares the original and optimized file sizes.
6. The smaller result is uploaded to the destination S3 bucket; if optimization increases the size, the original is uploaded instead.
7. CloudWatch Logs can be used to review the invocation and troubleshoot errors.

## Prerequisites

- An AWS account with permission to create S3 buckets, IAM roles and policies, Lambda functions, layers, and S3 triggers.
- The instructor-provided `lambda_function.py` file.
- The matching instructor-provided PyMuPDF Lambda Layer ZIP.
- A small test PDF (the guide recommends under 5 MB for the first test).
- Both S3 bucket names and the AWS Region.

> **Important:** Use the supplied code and layer as a matching pair. Do not substitute a random third-party layer or unrelated code version. The layer must match the Lambda runtime and architecture.

## Setup Overview

1. **Create two private S3 buckets**: one source bucket and one destination bucket. Keep **Block all public access** enabled.
2. **Create the Lambda execution role**:
   - Attach `AWSLambdaBasicExecutionRole` for CloudWatch logging.
   - Add an inline S3 policy scoped to the two buckets.
3. **Create the Lambda function** named `PDF-Compressor-Function`, using the Python runtime compatible with the supplied code and layer.
4. **Create and attach the `pymupdf-layer` layer**. Confirm the ZIP contains a top-level `python/` directory and that the layer architecture/runtime match the function.
5. **Configure Lambda settings** for the initial test:
   - Memory: `1024 MB`
   - Timeout: `120 seconds`
   - Ephemeral storage: `1024 MB`
6. **Add environment variables**:
   - `DESTINATION_BUCKET` = exact destination bucket name
   - `MAX_FILE_MB` = `50`
7. **Deploy the instructor-provided `lambda_function.py`**.
8. **Add an S3 trigger** to the **source bucket only**, using all object-create events.
9. Upload a small PDF to the source bucket and verify the output in the destination bucket.

## IAM Permissions

The Lambda execution role should have only the permissions needed for this workflow:

- `s3:GetObject` on objects in the source bucket.
- `s3:PutObject` on objects in the destination bucket.
- `AWSLambdaBasicExecutionRole` for CloudWatch Logs.

Replace the bucket placeholders with your actual bucket names:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadSourcePDFs",
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::YOUR-SOURCE-BUCKET/*"
    },
    {
      "Sid": "WriteCompressedPDFs",
      "Effect": "Allow",
      "Action": ["s3:PutObject"],
      "Resource": "arn:aws:s3:::YOUR-DESTINATION-BUCKET/*"
    }
  ]
}
```

Avoid granting `AmazonS3FullAccess` or `AdministratorAccess` to the Lambda role for this task. Check bucket policies and encryption permissions if access is denied.

## Testing and Validation

1. Upload a small PDF (under 5 MB for the first test) to the source bucket.
2. Open the Lambda function's **Monitor** tab and view its CloudWatch Logs.
3. Open the destination bucket and check that a processed PDF with the same key/name appears.
4. Download the output and confirm it opens correctly.
5. Compare original and output file sizes and verify important text, images, links, forms, and signatures as applicable.

## Troubleshooting

| Problem | What to check |
|---|---|
| `No module named 'fitz'` or `'pymupdf'` | Confirm the layer is attached, the ZIP has a top-level `python/` directory, and the import name, runtime, and architecture match. |
| `403 Forbidden` / `HeadObject` | Check the execution role's `s3:GetObject` permission for the source bucket, bucket policies, permissions boundaries, and KMS permissions if applicable. |
| `AccessDenied` on `PutObject` | Check `s3:PutObject` on the destination bucket and its bucket policy or encryption requirements. |
| Lambda times out | Test a smaller PDF and adjust timeout, memory, or temporary storage based on observed needs. |
| Output is missing | Review CloudWatch Logs, confirm `DESTINATION_BUCKET`, and verify the S3 trigger is attached to the source bucket. |

## Security and Cost Notes

- Keep both buckets private and leave Block Public Access enabled.
- Scope IAM permissions to the required source and destination object paths.
- Configure the S3 trigger on the source bucket only. Triggering on the destination bucket can cause repeated invocations.
- Lambda, S3 storage/requests, and CloudWatch Logs may incur charges. Test with small files and review usage and costs before processing larger PDFs.

## Project Outcome

This project demonstrates an event-driven serverless pipeline that automates PDF optimization without requiring a continuously running server. Amazon S3 handles file storage and event triggering, AWS Lambda performs processing, and PyMuPDF provides the PDF optimization dependency.

## Notes

- This README describes the workflow from the supplied student practical guide.
- The actual compression implementation is provided separately in `lambda_function.py`; the code itself was not included in the uploaded guide.
