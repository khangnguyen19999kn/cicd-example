# AWS S3 + CloudFront Deployment Guide (Private S3 + OAC)

## Bước 1: Tạo AWS Resources

### Tạo S3 Bucket (Private)
```bash
# 1. Tạo S3 bucket (private, không public access)
BUCKET_NAME="cicd-example-bucket-$(date +%s)"
aws s3 mb s3://$BUCKET_NAME --region ap-southeast-1

# 2. Block public access (đảm bảo bucket private)
aws s3api put-public-access-block \
  --bucket "$BUCKET_NAME" \
  --public-access-block-configuration \
  BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

### Tạo Origin Access Control (OAC)
```bash
# 3. Tạo OAC cho CloudFront
OAC_ID=$(aws cloudfront create-origin-access-control --origin-access-control-config '{
    "Name": "OAC-'$BUCKET_NAME'",
    "Description": "OAC for S3 bucket",
    "OriginAccessControlOriginType": "s3",
    "SigningBehavior": "always",
    "SigningProtocol": "sigv4"
}' --query 'OriginAccessControl.Id' --output text)

echo "OAC ID: $OAC_ID"
```

### Tạo CloudFront Distribution với OAC
```bash
# 4. Tạo CloudFront distribution với OAC
DIST_ID=$(aws cloudfront create-distribution --distribution-config '{
    "CallerReference": "cicd-'$(date +%s)'",
    "Comment": "React app with OAC",
    "DefaultRootObject": "index.html",
    "Origins": {
        "Quantity": 1,
        "Items": [{
            "Id": "S3-OAC-origin",
            "DomainName": "'$BUCKET_NAME'.s3.ap-southeast-1.amazonaws.com",
            "S3OriginConfig": {"OriginAccessIdentity": ""},
            "OriginAccessControlId": "'$OAC_ID'"
        }]
    },
    "DefaultCacheBehavior": {
        "TargetOriginId": "S3-OAC-origin",
        "ViewerProtocolPolicy": "redirect-to-https",
        "AllowedMethods": {"Quantity": 2, "Items": ["GET", "HEAD"]},
        "CachedMethods": {"Quantity": 2, "Items": ["GET", "HEAD"]},
        "TrustedSigners": {"Enabled": false, "Quantity": 0},
        "ForwardedValues": {"QueryString": false, "Cookies": {"Forward": "none"}},
        "MinTTL": 0
    },
    "CustomErrorResponses": {
        "Quantity": 1,
        "Items": [{
            "ErrorCode": 404,
            "ResponsePagePath": "/index.html",
            "ResponseCode": "200",
            "ErrorCachingMinTTL": 300
        }]
    },
    "Enabled": true,
    "PriceClass": "PriceClass_100"
}' --query 'Distribution.Id' --output text)

echo "Distribution ID: $DIST_ID"
```

### Cập nhật S3 Bucket Policy cho OAC
```bash
# 5. Lấy CloudFront service principal và tạo bucket policy
aws s3api put-bucket-policy --bucket $BUCKET_NAME --policy '{
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Allow",
        "Principal": {"Service": "cloudfront.amazonaws.com"},
        "Action": "s3:GetObject",
        "Resource": "arn:aws:s3:::'$BUCKET_NAME'/*",
        "Condition": {
            "StringEquals": {
                "AWS:SourceArn": "arn:aws:cloudfront::'$(aws sts get-caller-identity --query Account --output text)':distribution/'$DIST_ID'"
            }
        }
    }]
}'

echo "Setup completed!"
echo "S3 Bucket: $BUCKET_NAME"
echo "CloudFront Distribution ID: $DIST_ID"
echo "OAC ID: $OAC_ID"
```
```

## Bước 2: Tạo IAM User cho GitHub Actions

### IAM Policy JSON:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": ["s3:PutObject", "s3:PutObjectAcl", "s3:GetObject", "s3:DeleteObject", "s3:ListBucket"],
            "Resource": ["arn:aws:s3:::YOUR_BUCKET_NAME", "arn:aws:s3:::YOUR_BUCKET_NAME/*"]
        },
        {
            "Effect": "Allow",
            "Action": ["cloudfront:CreateInvalidation"],
            "Resource": "*"
        }
    ]
}
```

## Bước 3: Thêm GitHub Secrets
Vào GitHub repo → Settings → Secrets and variables → Actions → New repository secret:

- `AWS_ACCESS_KEY_ID`: IAM user access key
- `AWS_SECRET_ACCESS_KEY`: IAM user secret key  
- `S3_BUCKET_NAME`: Tên bucket vừa tạo
- `CLOUDFRONT_DISTRIBUTION_ID`: ID từ CloudFront distribution

## Bước 4: Push code
Workflow sẽ tự động chạy và deploy lên AWS!