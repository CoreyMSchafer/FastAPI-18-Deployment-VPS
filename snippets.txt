## S3 Configuration for .env file
S3_BUCKET_NAME=fastapi-blog-uploads
S3_REGION=us-east-1
S3_ACCESS_KEY_ID=your-access-key-id
S3_SECRET_ACCESS_KEY=your-secret-access-key


## S3 Settings for config.py
# S3 Configuration
s3_bucket_name: str
s3_region: str = "us-east-1"
s3_access_key_id: SecretStr | None = None
s3_secret_access_key: SecretStr | None = None
s3_endpoint_url: str | None = None


## _get_s3_client helper for image_utils.py
def _get_s3_client():
    return boto3.client(
        "s3",
        region_name=settings.s3_region,
        aws_access_key_id=(
            settings.s3_access_key_id.get_secret_value()
            if settings.s3_access_key_id
            else None
        ),
        aws_secret_access_key=(
            settings.s3_secret_access_key.get_secret_value()
            if settings.s3_secret_access_key
            else None
        ),
        endpoint_url=settings.s3_endpoint_url,
    )


## _upload_to_s3 and _delete_from_s3 for image_utils.py
def _upload_to_s3(file_bytes: bytes, key: str) -> None:
    s3 = _get_s3_client()
    s3.upload_fileobj(
        BytesIO(file_bytes),
        settings.s3_bucket_name,
        key,
        ExtraArgs={"ContentType": "image/jpeg"},
    )


def _delete_from_s3(key: str) -> None:
    s3 = _get_s3_client()
    s3.delete_object(Bucket=settings.s3_bucket_name, Key=key)


## Async S3 wrappers for image_utils.py
async def upload_profile_image(file_bytes: bytes, filename: str) -> None:
    key = f"profile_pics/{filename}"
    await run_in_threadpool(_upload_to_s3, file_bytes, key)


async def delete_profile_image(filename: str | None) -> None:
    if filename is None:
        return
    key = f"profile_pics/{filename}"
    await run_in_threadpool(_delete_from_s3, key)


## S3 image_path return for models.py
return f"https://{settings.s3_bucket_name}.s3.{settings.s3_region}.amazonaws.com/profile_pics/{self.image_file}"


## S3 upload try/except for routers/users.py (upload_profile_picture)
# Upload to S3 (also runs in threadpool via async wrapper)
try:
    await upload_profile_image(processed_bytes, new_filename)
except ClientError as err:
    raise HTTPException(
        status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
        detail="Failed to upload image. Please try again.",
    ) from err


## check_s3.py
from io import BytesIO

from botocore.exceptions import BotoCoreError, ClientError

from config import settings
from image_utils import _get_s3_client


def check_s3_connection():
    s3 = _get_s3_client()

    print(f"Bucket: {settings.s3_bucket_name}")
    print(f"Region: {settings.s3_region}")
    print()

    test_key = "profile_pics/test.txt"

    try:
        s3.upload_fileobj(
            BytesIO(b"test"),
            settings.s3_bucket_name,
            test_key,
            ExtraArgs={"ContentType": "text/plain"},
        )
        print("Upload: SUCCESS")
    except (BotoCoreError, ClientError) as exc:
        print(f"Upload: FAILED - {exc}")
        return

    try:
        s3.delete_object(Bucket=settings.s3_bucket_name, Key=test_key)
        print("Delete: SUCCESS")
    except (BotoCoreError, ClientError) as exc:
        print(f"Delete: FAILED - {exc}")
        return

    print()
    print("All tests passed! Your S3 configuration is working.")


if __name__ == "__main__":
    check_s3_connection()


##