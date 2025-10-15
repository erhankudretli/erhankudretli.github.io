---
title: "AWS Data Pipeline Challenge — My Modular Terraform Approach"
date: 2025-10-15
categories: [DevOps]
tags: [terraform, aws, lambda, cicd, security, best-practices]
description: "How I approached a simple AWS data pipeline challenge in a Cloud Engineer interview using Terraform modules, security best practices, and CI/CD automation."
image: /assets/img/aws-data/view.jpg
---

A few days ago, during a **Cloud Engineer** interview process, I was asked to build a **simple data pipeline using Terraform**.  
The idea was quite straightforward:  
a file lands in an **S3 bucket**, a **Lambda** function processes it, and the result is stored in another **S3 bucket**.  

Everything was expected to be defined using **Terraform** so that it could be easily cloned or extended later.  
I completed the required task in a modular way — but I wanted to go a bit further. 🙂  

Here’s the basic diagram of the setup:  
`(/assets/img/aws-data/diagram.png){: .center-image }`

---

## Standardization and CI/CD Approach

I consider **standardization** a core DevOps principle.  
Every developer, DevOps engineer, or user should experience a **consistent, reproducible, and stable** process.  

When Terraform Enterprise isn’t available, my go-to strategy is to manage infrastructure through a **CI/CD pipeline**.  
For this challenge, I envisioned a pipeline that includes:

- **Static security scans with Trivy**  
- Deployments allowed only from specific branches  
- `terraform plan`, `terraform apply`, and **cost checks** before applying  
- **Manual approval** steps after reviewing the plan and estimated costs  
- Storing `tfplan` and `infracost` outputs as **artifacts**  
- Automatic `terraform fmt`  
- Automatic `terraform-docs` generation  
- **Branch protection** rules and extended automation features  

---

## A. Security & Logging

- **Least Privilege IAM:**  
  The Lambda role follows the principle of least privilege — it can only read from the raw bucket, write to the processed bucket, and send logs to a single Log Group.

- **Custom Log Group:**  
  Instead of letting Lambda auto-create a CloudWatch Log Group, I defined one via Terraform.  
  The IAM policy allows writes only to that group — improving both **security** and **structure**.

- **Structured Logging:**  
  I used Python’s built-in `logging` module instead of `print()`, which provides proper log levels (INFO, ERROR, etc.) and cleaner stack traces for debugging.

---

## B. Terraform Setup

- **Remote State Management:**  
  The Terraform state is stored remotely in an S3 bucket (`ek-eu-west-01-tf-state-bucket`), with DynamoDB-based locking (`terraform-locks`) to prevent concurrent modifications.

- **Modular Design:**  
  The code is organized into separate `s3`, `iam`, and `lambda` modules.  
  Deploying to a new environment is as simple as reusing these modules.

- **Provider Default Tags:**  
  The AWS provider is configured with default tags such as `env`, `owner`, `project`, and `ManagedBy`.  
  All resources automatically inherit these tags.

- **Explicit Dependencies:**  
  Added `depends_on` statements where needed to ensure resource readiness (e.g., Lambda permissions before S3 notifications).

- **Environment Management:**  
  Separate folders (`dev`, `prd`) and `terraform.tfvars` files keep environments clean and isolated.

---

## C. Lambda & Project Practices

- **Environment Variables:**  
  The Lambda receives the target bucket name via an environment variable (`TARGET_BUCKET_NAME`), keeping the Python code generic and environment-agnostic.

- **Code Formatting:**  
  Applied `terraform fmt` for consistent formatting and readability.

- **S3 Write Strategy:**  
  Lambda writes processed output under `/processed_summary/` instead of the root — this keeps the structure tidy and avoids rate limit or performance issues.

---

If you’d like to take a look at the project, you can find it here:  
👉 [GitHub – erhankudretli/iac-aws-data-pipeline](https://github.com/erhankudretli/iac-aws-data-pipeline)

---

Thanks for reading!  

---


If you’d like to know more about me or see my resume, visit the [About Me page](/about-me) or For direct contact, feel free to reach out via email at **erhankudretli@gmail.com**.
