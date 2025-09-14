---
layout: post
title: "How to Securely Handle Akamai Cache Purge in a CI/CD Pipeline"
date: 2025-09-14
categories: [cicd, akamai, devops,aws,terraform]
image: /assets/img/view2.jpeg
---

## Background

Akamai is a well-known tech company offering CDN services.  
When using Akamai as CDN, frontend developers often need to purge cached static files so that deployments are immediately visible. For this we use Akamai’s *Invalidate by URL API* (or sometimes manually via the UI).

Akamai’s API authentication is built around **EdgeGrid**. Normally, credentials downloaded from Akamai are stored in `~/.edgerc`. That works fine in local development.

---

## The Problem

Everything works nicely until we try to make the purge API call part of a CI/CD pipeline.

- Putting the `.edgerc` file on the CI/CD runner’s OS (even in read-only mode) poses a security risk. We don’t want credentials stored in the build machine.  
- Also, sending EdgeGrid credentials directly via `curl` headers is not supported as a secure method. A simple `curl` with inline credentials doesn’t meet security best practices.

---

## Solution 1: Python Script + CI/CD Secrets

One good solution: Use the EdgeGrid library in Python.  
Create an authenticated session via EdgeGrid, then call purge safely. Credentials come via environment variables, which in GitHub Actions are stored as *secrets*.

### Python Script Example

```python
#!/usr/bin/env python3
import os
import json
import requests
from akamai.edgegrid import EdgeGridAuth

def purge_cache(access_token, client_token, client_secret, purge_urls):
    print("Starting cache purge...")

    baseurl = "https://example.luna.akamaiapis.net"  # anonymized base URL
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
    print(f"Response status: {response.status_code}")
    print(f"Response body: {response.json()}")

    if response.status_code in [200, 201]:
        print("✅ Cache purge successful!")
    else:
        print("❌ Cache purge failed!")

if __name__ == "__main__":
    purge_cache(
        access_token=os.environ["AKAMAI_ACCESS_TOKEN"],
        client_token=os.environ["AKAMAI_CLIENT_TOKEN"],
        client_secret=os.environ["AKAMAI_CLIENT_SECRET"],
        purge_urls=os.environ["PURGE_URLS"].split(",")
    )
````

In GitHub Actions workflow:

```yaml
- name: Purge Akamai Cache
  env:
    AKAMAI_ACCESS_TOKEN: ${{ secrets.AKAMAI_ACCESS_TOKEN }}
    AKAMAI_CLIENT_TOKEN: ${{ secrets.AKAMAI_CLIENT_TOKEN }}
    AKAMAI_CLIENT_SECRET: ${{ secrets.AKAMAI_CLIENT_SECRET }}
    PURGE_URLS: "https://your-app.com/index.js,https://your-app.com/index.css"
  run: python ./purge.py
```

---

## Solution 2: Cloud-Native Approach (Lambda + API Gateway)

Alternatively, use a cloud native model:

![Architecture Diagram](/assets/img/github-aws-akamai.webp)

1. Write an AWS Lambda function that does the purge with EdgeGrid.
2. Expose the function through API Gateway.
3. CI/CD pipeline only calls the API Gateway endpoint.

This way credentials live in the Lambda environment and are never exposed or stored on the pipeline runner.


---

## Additional Resources

* Akamai EdgeGrid official documentation: [Akamai EdgeGrid — Tech Docs](https://techdocs.akamai.com/developer/docs/edgegrid) ([TechDocs][1])
* Akamai EdgeGrid for Python (GitHub repository): [AkamaiOPEN-edgegrid-python](https://github.com/akamai/AkamaiOPEN-edgegrid-python) ([GitHub][2])

---

## Conclusion

* If you want a quick, secure setup: use the **Python script + CI/CD secrets** method.
* If you want a more scalable, safer architecture: go with **Lambda + API Gateway**.

Both keep your secrets safe and allow cache purging to be automated after every deployment.

```
```

[1]: https://techdocs.akamai.com/developer/docs/edgegrid?utm_source=chatgpt.com "EdgeGrid"
[2]: https://github.com/akamai/AkamaiOPEN-edgegrid-python?utm_source=chatgpt.com "akamai/AkamaiOPEN-edgegrid-python"
