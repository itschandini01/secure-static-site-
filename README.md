# secure-static-site-
terraform code sample 
# variables.tf

variable "domain_name" {
  description = "The root domain, e.g., example.com"
  type        = string
}

variable "www_domain_name" {
  description = "The www subdomain, e.g., www.example.com"
  type        = string
}

variable "aws_region" {
  description = "AWS region for most resources"
  type        = string
  default     = "us-east-1"
}

variable "acm_region" {
  description = "Region for ACM certificate (must be us-east-1 for CloudFront)"
  type        = string
  default     = "us-east-1"
}
# main.tf

provider "aws" {
  region = var.aws_region
}

provider "aws" {
  alias  = "us_east_1"
  region = var.acm_region
}

# S3 bucket for website assets (private)
resource "aws_s3_bucket" "website_bucket" {
  bucket = var.www_domain_name
  acl    = "private"

  versioning {
    enabled = true
  }

  # block all public access
  block_public_acls   = true
  block_public_policy = true
  ignore_public_acls  = true
  restrict_public_buckets = true

  tags = {
    Name = "website-assets-${var.www_domain_name}"
  }
}

# S3 bucket for redirection (root domain -> www)
resource "aws_s3_bucket" "redirect_bucket" {
  bucket = var.domain_name
  acl    = "private"

  website {
    redirect_all_requests_to = {
      host_name = var.www_domain_name
      protocol  = "https"
    }
  }

  block_public_acls   = true
  block_public_policy = true
  ignore_public_acls  = true
  restrict_public_buckets = true

  tags = {
    Name = "redirect-${var.domain_name}"
  }
}

# Get ACM certificate in us-east-1 for CloudFront
resource "aws_acm_certificate" "cert" {
  provider          = aws.us_east_1
  domain_name       = var.www_domain_name
  subject_alternative_names = [var.domain_name]
  validation_method = "DNS"

  tags = {
    Name = "cert-${var.www_domain_name}"
  }
}

# DNS validation for certificate
resource "aws_route53_zone" "primary" {
  name = var.domain_name
}

resource "aws_route53_record" "cert_validation" {
  provider = aws.us_east_1
  for_each = {
    for dvo in aws_acm_certificate.cert.domain_validation_options : dvo.domain_name => dvo
  }
  name    = each.value.resource_record_name
  type    = each.value.resource_record_type
  zone_id = aws_route53_zone.primary.zone_id
  records = [each.value.resource_record_value]
  ttl     = 60
}

resource "aws_acm_certificate_validation" "cert_validation_complete" {
  provider                = aws.us_east_1
  certificate_arn         = aws_acm_certificate.cert.arn
  validation_record_fqdns = [for rec in aws_route53_record.cert_validation : rec.fqdn]
}

# CloudFront origin access identity (if using OAI)
resource "aws_cloudfront_origin_access_identity" "oai" {
  comment = "OAI for ${var.www_domain_name}"
}

# Bucket policy to allow CloudFront (via OAI) to get objects
data "aws_iam_policy_document" "bucket_policy" {
  statement {
    actions   = ["s3:GetObject"]
    principals {
      type        = "AWS"
      identifiers = [aws_cloudfront_origin_access_identity.oai.iam_arn]
    }
    resources = ["${aws_s3_bucket.website_bucket.arn}/*"]
  }
}

resource "aws_s3_bucket_policy" "website_bucket_policy" {
  bucket = aws_s3_bucket.website_bucket.id
  policy = data.aws_iam_policy_document.bucket_policy.json
}

# CloudFront distribution
resource "aws_cloudfront_distribution" "cdn" {
  enabled             = true
  is_ipv6_enabled     = true
  comment             = "CDN for ${var.www_domain_name}"

  aliases = [var.www_domain_name, var.domain_name]

  origins {
    origin_id   = "website-origin"
    domain_name = aws_s3_bucket.website_bucket.bucket_regional_domain_name
    s3_origin_config {
      origin_access_identity = aws_cloudfront_origin_access_identity.oai.cloudfront_access_identity_path
    }
  }

  default_root_object = "index.html"

  default_cache_behavior {
    allowed_methods  = ["GET", "HEAD"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "website-origin"

    viewer_protocol_policy = "redirect-to-https"

    # optional: caching settings
    compress = true

    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
    }
  }

  # custom error responses
  custom_error_response {
    error_caching_min_ttl = 60
    error_code             = 404
    response_code          = 404
    response_page_path     = "/404.html"
  }

  # TLS certificate
  viewer_certificate {
    acm_certificate_arn            = aws_acm_certificate.cert.arn
    ssl_support_method             = "sni-only"
    minimum_protocol_version       = "TLSv1.2_2021"
  }

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  price_class = "PriceClass_100"

  tags = {
    Name = "cdn-${var.www_domain_name}"
  }
}

# Route53 records
resource "aws_route53_record" "www_A_record" {
  zone_id = aws_route53_zone.primary.zone_id
  name    = var.www_domain_name
  type    = "A"
  alias {
    name                   = aws_cloudfront_distribution.cdn.domain_name
    zone_id                = aws_cloudfront_distribution.cdn.hosted_zone_id
    evaluate_target_health = false
  }
}

resource "aws_route53_record" "root_A_record" {
  zone_id = aws_route53_zone.primary.zone_id
  name    = var.domain_name
  type    = "A"
  alias {
    name                   = aws_cloudfront_distribution.cdn.domain_name
    zone_id                = aws_cloudfront_distribution.cdn.hosted_zone_id
    evaluate_target_health = false
  }
}
