Tamamdır 🚀 Şimdi sana **Jekyll uyumlu tam bir blog post (Markdown)** vereyim. İçinde:

* **Background** ve problem tanımı
* **Çözüm 1: Python Script + CICD Secrets** (koduyla birlikte)
* **Çözüm 2: Lambda + API Gateway** (koduyla birlikte, ama bağlantıyı yüzeysel anlatıyoruz)
* **Terraform snippet** (sadece API Gateway + Lambda entegrasyonu için base)
* Diyagram için placeholder görsel

Hepsi tek bir `.md` dosyası olacak şekilde:

---

````markdown
---
layout: post
title: "How to Secure Akamai Cache Purge in a CI/CD Pipeline"
date: 2025-09-14
categories: devops akamai cicd aws
---

# How to Secure Akamai Cache Purge in a CI/CD Pipeline

## Background

Akamai is one of the largest CDN providers.  
When used as a CDN, frontend developers often need cache purges so their deployments are immediately reflected in production. For that, Akamai provides the **Invalidate by URL** API (or you can purge manually via the UI).  

Akamai’s API authentication is built on **EdgeGrid**.  
Normally, credentials are stored in a `~/.edgerc` file. This works fine for local development.

## Problem

Things start to get tricky when you want to move these purge calls into a **CI/CD pipeline**.  
Placing the `.edgerc` file in the pipeline runner OS introduces a **security risk** — we don’t want sensitive credentials stored directly on the build machine.  

On the other hand, simply attaching EdgeGrid credentials as headers to a `curl` command is not supported. So, a basic curl call won’t solve it.

---

## Solution 1: Python Script + CI/CD Secrets

A safer approach is to use the **Akamai EdgeGrid Python library**.  
We can create a session, inject credentials from CI/CD secrets, and run the purge securely.

Here’s a working Python example:

```python
import os
import json
import requests
from akamai.edgegrid import EdgeGridAuth

def purge_cache(access_token, client_token, client_secret, purge_urls):
    baseurl = "https://example.luna.akamaiapis.net"  # anonymized Akamai host
    session = requests.Session()
    session.auth = EdgeGridAuth(
        client_token=client_token,
        client_secret=client_secret,
        access_token=access_token
    )

    url = f"{baseurl}/ccu/v3/invalidate/url/production"
    payload = { "objects": purge_urls }
    headers = {
        "accept": "application/json",
        "content-type": "application/json"
    }

    response = session.post(url, json=payload, headers=headers)
    print(f"Purge Response: {response.status_code} - {response.json()}")

    if response.status_code in [200, 201]:
        print("✅ Cache purge successful!")
    else:
        print("❌ Cache purge failed.")

if __name__ == "__main__":
    purge_cache(
        access_token=os.environ["AKAMAI_ACCESS_TOKEN"],
        client_token=os.environ["AKAMAI_CLIENT_TOKEN"],
        client_secret=os.environ["AKAMAI_CLIENT_SECRET"],
        purge_urls=os.environ["PURGE_URLS"].split(",")
    )
````

In GitHub Actions (or any CI/CD system), credentials are provided via **secrets**, not via files.

---

## Solution 2: Cloud-Native with AWS Lambda + API Gateway

Another option is to push the logic into the cloud:

* Write a **Lambda function** that handles Akamai purge calls with EdgeGrid.
* Expose it via **API Gateway**.
* The CI/CD pipeline only calls the API Gateway endpoint.

In this way, credentials are managed inside Lambda, never exposed to the pipeline.

Example Lambda code (`lambda_function.py`):

```python
import os
import json
import requests
from akamai.edgegrid import EdgeGridAuth

def lambda_handler(event, context):
    baseurl = "https://example.luna.akamaiapis.net"  # anonymized Akamai host

    session = requests.Session()
    session.auth = EdgeGridAuth(
        client_token=os.environ["AKAMAI_CLIENT_TOKEN"],
        client_secret=os.environ["AKAMAI_CLIENT_SECRET"],
        access_token=os.environ["AKAMAI_ACCESS_TOKEN"]
    )

    purge_urls = []
    if "body" in event and event["body"]:
        try:
            body = json.loads(event["body"])
            purge_urls = body.get("urls", [])
        except Exception:
            pass

    if not purge_urls:
        purge_urls = os.environ.get("PURGE_URLS", "").split(",")

    url = f"{baseurl}/ccu/v3/invalidate/url/production"
    payload = { "objects": purge_urls }
    headers = {
        "accept": "application/json",
        "content-type": "application/json"
    }

    response = session.post(url, json=payload, headers=headers)

    if response.status_code in [200, 201]:
        return {
            "statusCode": 200,
            "body": json.dumps({"message": "✅ Cache purge successful!"})
        }
    else:
        return {
            "statusCode": response.status_code,
            "body": json.dumps({"message": "❌ Cache purge failed"})
        }
```

---

## Terraform: API Gateway + Lambda (base setup)

We can use Terraform to set up API Gateway and connect it to Lambda.
This is a base config (Lambda implementation details are kept abstract here):

**`vars.tf`**

```hcl
variable "lambda_function_name" {
  type        = string
  description = "Name of the Lambda function to integrate with API Gateway"
}

variable "api_name" {
  type        = string
  default     = "akamai-purge-api"
}

variable "stage_name" {
  type        = string
  default     = "dev"
}
```

**`main.tf`**

```hcl
provider "aws" {
  region = "us-east-1"
}

resource "aws_api_gateway_rest_api" "akamai_api" {
  name = var.api_name
}

resource "aws_api_gateway_resource" "purge" {
  rest_api_id = aws_api_gateway_rest_api.akamai_api.id
  parent_id   = aws_api_gateway_rest_api.akamai_api.root_resource_id
  path_part   = "purge"
}

resource "aws_api_gateway_method" "post_method" {
  rest_api_id   = aws_api_gateway_rest_api.akamai_api.id
  resource_id   = aws_api_gateway_resource.purge.id
  http_method   = "POST"
  authorization = "NONE"
}

resource "aws_api_gateway_integration" "lambda_integration" {
  rest_api_id             = aws_api_gateway_rest_api.akamai_api.id
  resource_id             = aws_api_gateway_resource.purge.id
  http_method             = aws_api_gateway_method.post_method.http_method
  integration_http_method = "POST"
  type                    = "AWS_PROXY"
  uri                     = aws_lambda_function.akamai_purge.invoke_arn
}

resource "aws_api_gateway_deployment" "deployment" {
  depends_on  = [aws_api_gateway_integration.lambda_integration]
  rest_api_id = aws_api_gateway_rest_api.akamai_api.id
  stage_name  = var.stage_name
}

resource "aws_lambda_permission" "api_gw" {
  statement_id  = "AllowAPIGatewayInvoke"
  action        = "lambda:InvokeFunction"
  function_name = var.lambda_function_name
  principal     = "apigateway.amazonaws.com"
}
```

---

## Diagram

Here’s a placeholder diagram for Solution 2:

![Cloud-Native Cache Purge Diagram](/assets/img/myimage.webp)

---

## References

* [Akamai EdgeGrid Authentication Overview](https://techdocs.akamai.com/developer/docs/authenticate-with-edgegrid)

```

---