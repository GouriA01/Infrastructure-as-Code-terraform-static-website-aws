# Hosting Website on AWS with Terraform

A beginner friendly project that deploys a live HTTPS website on AWS with one command.

Built for students and freshers learning Infrastructure as Code.

## What It Creates

- Private S3 bucket that stores the website
- index.html uploaded automatically
- CloudFront distribution with free HTTPS
- Origin Access Control so only CloudFront can read the bucket
- Bucket policy that allows only this CloudFront distribution

## Environment

- Terminal AWS CloudShell (no laptop setup needed)
- Region us-east-1
- Tool Terraform 1.9.8

## Why CloudFront

- Private bucket, Many accounts block public S3 buckets with BlockPublicPolicy. CloudFront reads a private bucket through Origin Access Control, so nothing is public.
- Free HTTPS An S3 website URL is http only. CloudFront adds https with a free certificate.
- Production standard A private bucket behind CloudFront is how real sites are built.
- Faster pages CloudFront serves the site from edge locations around the world.

If your account allows public buckets, an S3 only website also works, but it is less secure and has no https.

## Cost

- CloudShell is free. You get 1 GB of home folder storage per region at no charge.
- S3 One small HTML file costs a fraction of a cent.
- CloudFront The AWS Free Tier covers far more traffic than a test site uses.
- Total Practically zero if you run terraform destroy after testing.

Set a billing alert in AWS Budgets and check the AWS Free Tier page for current limits, since they can change.

## Project Files

```
my-website/
├── main.tf
├── outputs.tf
└── index.html
```

## Step 1 Install Terraform in CloudShell

Tools installed with sudo are removed when a CloudShell session ends. Install Terraform in your home folder so it stays.

```bash
mkdir -p ~/bin
cd ~/bin
curl -LO https://releases.hashicorp.com/terraform/1.9.8/terraform_1.9.8_linux_amd64.zip
unzip -o terraform_1.9.8_linux_amd64.zip
rm terraform_1.9.8_linux_amd64.zip
echo 'export PATH=$HOME/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
terraform -version
```

## Step 2 Create the Project Folder

```bash
mkdir -p ~/my-website
cd ~/my-website
```

## Step 3 Create the Files

Create below 3 files using vi editor main.tf , outputs.tf and index.html and paste the files content as given below.

main.tf 

```hcl
provider "aws" {
  region = "us-east-1"
}

# Step 1 Create a private bucket
resource "aws_s3_bucket" "site" {
  bucket_prefix = "my-first-website-"
  force_destroy = true
}

# Step 2 Upload the web page
resource "aws_s3_object" "index" {
  bucket       = aws_s3_bucket.site.id
  key          = "index.html"
  source       = "index.html"
  content_type = "text/html"
}

# Step 3 Give CloudFront a secure way to read the bucket
resource "aws_cloudfront_origin_access_control" "site" {
  name                              = "my-first-website-oac"
  origin_access_control_origin_type = "s3"
  signing_behavior                  = "always"
  signing_protocol                  = "sigv4"
}

# Step 4 Create CloudFront
resource "aws_cloudfront_distribution" "site" {
  enabled             = true
  default_root_object = "index.html"

  origin {
    domain_name              = aws_s3_bucket.site.bucket_regional_domain_name
    origin_id                = "s3"
    origin_access_control_id = aws_cloudfront_origin_access_control.site.id
  }

  default_cache_behavior {
    target_origin_id       = "s3"
    viewer_protocol_policy = "redirect-to-https"
    allowed_methods        = ["GET", "HEAD"]
    cached_methods         = ["GET", "HEAD"]
    cache_policy_id        = "658327ea-f89d-4fab-a63d-7e88639e58f6"
  }

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  viewer_certificate {
    cloudfront_default_certificate = true
  }
}

# Step 5 Allow only CloudFront to read the files
resource "aws_s3_bucket_policy" "site" {
  bucket = aws_s3_bucket.site.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "cloudfront.amazonaws.com" }
      Action    = "s3:GetObject"
      Resource  = "${aws_s3_bucket.site.arn}/*"
      Condition = {
        StringEquals = { "AWS:SourceArn" = aws_cloudfront_distribution.site.arn }
      }
    }]
  })
}
```

outputs.tf

```hcl
output "website_url" {
  value = "https://${aws_cloudfront_distribution.site.domain_name}"
}
```

index.html

```html
<!DOCTYPE html>
<html>
<body style="text-align:center; padding-top:80px; font-family:Arial;">
  <h1>Hello from Terraform</h1>
  <p>My hosting website on AWS S3 and CloudFront</p>
</body>
</html>
```

## Step 4 Deploy

```bash
terraform init
terraform plan
terraform apply
```

Type “yes” when asked. CloudFront takes about 5 to 10 minutes. Messages like “Still creating” are normal.

## Step 5 Open the Website

Copy website_url from the output and open it in your browser. If you see “Access Denied”, wait a couple of minutes and refresh.

## Step 6 Clean Up

```bash
terraform destroy
```

Always destroy when you are done so nothing keeps running.

## Common Errors and Fixes

| Error | Cause | Fix |
|---|---|---|
| terraform command not found | New CloudShell session removed sudo installs | Install Terraform in ~/bin as shown in Step 1 |
| BlockPublicPolicy AccessDenied | Account blocks public S3 buckets | Use this private bucket plus CloudFront setup |
| Duplicate output definition | Two files such as output.tf and outputs.tf | Keep one file and delete the other |
| No configuration files | Running Terraform from the wrong folder | Run cd ~/my-website first |
| Old page still showing | CloudFront cache | Run a CloudFront invalidation with paths "/*" |

## What You Learn

- Terraform providers, resources and outputs
- The init, plan, apply and destroy workflow
- Why production sites use a private bucket behind CloudFront
- How Origin Access Control secures S3

## License

MIT


=====================================================================================
