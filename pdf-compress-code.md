import os
import uuid
import logging
from urllib.parse import unquote_plus

import boto3
import pymupdf as fitz

s3 = boto3.client("s3")
logger = logging.getLogger()
logger.setLevel(logging.INFO)

DESTINATION_BUCKET = os.environ["DESTINATION_BUCKET"]
MAX_FILE_MB = int(os.environ.get("MAX_FILE_MB", "50"))
MAX_FILE_BYTES = MAX_FILE_MB * 1024 * 1024

def lambda_handler(event, context):
    results = []

    for record in event.get("Records", []):
        source_bucket = record["s3"]["bucket"]["name"]
        source_key = unquote_plus(
            record["s3"]["object"]["key"]
        )

        if not source_key.lower().endswith(".pdf"):
            logger.info("Skipping non-PDF: %s", source_key)
            continue

        original_path = f"/tmp/{uuid.uuid4()}.pdf"
        compressed_path = f"/tmp/{uuid.uuid4()}-compressed.pdf"

        try:
            info = s3.head_object(
                Bucket=source_bucket,
                Key=source_key
            )
            original_size = info["ContentLength"]

            if original_size > MAX_FILE_BYTES:
                raise ValueError(
                    f"PDF exceeds {MAX_FILE_MB} MB limit"
                )

            s3.download_file(
                source_bucket, source_key, original_path
            )

            with fitz.open(original_path) as pdf:
                if pdf.needs_pass:
                    raise ValueError(
                        "Password-protected PDFs are unsupported"
                    )

                pdf.save(
                    compressed_path,
                    garbage=4,
                    deflate=True,
                    deflate_images=True,
                    deflate_fonts=True,
                    clean=True
                )

            compressed_size = os.path.getsize(compressed_path)

            # Never replace the original with a larger result.
            output_path = (
                compressed_path
                if compressed_size < original_size
                else original_path
            )

            output_size = os.path.getsize(output_path)
            output_key = source_key

            s3.upload_file(
                output_path,
                DESTINATION_BUCKET,
                output_key,
                ExtraArgs={
                    "ContentType": "application/pdf",
                    "Metadata": {
                        "original-size": str(original_size),
                        "output-size": str(output_size),
                        "compression-saved-percent": str(
                            round(
                                (original_size - output_size)
                                / original_size * 100, 2
                            ) if original_size else 0
                        )
                    }
                }
            )

            saved_percent = (
                (original_size - output_size)
                / original_size * 100
                if original_size else 0
            )

            result = {
                "source": source_key,
                "output": output_key,
                "original_bytes": original_size,
                "output_bytes": output_size,
                "saved_percent": round(saved_percent, 2)
            }
            results.append(result)
            logger.info("Compression result: %s", result)

        finally:
            for path in (original_path, compressed_path):
                if os.path.exists(path):
                    os.remove(path)

    return {"processed": results}
