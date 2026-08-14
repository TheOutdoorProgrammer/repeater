---
dusk: v1alpha1
namespace: stout
kind: repository
name: repeater
title: Repeater
attributes:
  iac: terraform
  cloud: aws
  public: true
---

A reusable Terraform module that puts a public front door on a webhook receiver which is not reachable from the internet.
It stands up an HTTP API Gateway on a custom domain with a single `POST /webhook` route, and a Python 3.11 Lambda behind it that forwards each request to whatever `webhook_endpoint` it was configured with and hands that endpoint's response back to the original caller verbatim.
The Lambda is attached to the subnets and security groups passed in, so reaching a private service is a matter of VPC routing rather than anything the module does itself.

`main.tf` is the entire module: six input variables, the community `terraform-aws-modules/lambda/aws` and `terraform-aws-modules/apigateway-v2/aws` modules, and nothing else.
It declares no outputs, so a consumer needing an ARN has to reach through the nested module.
`src/main.py` is the whole handler: reject anything that is not a POST with a 405, base64-decode the body when API Gateway encoded it, delete the `host` header so the downstream service never learns the repeater is in front of it, `requests.post`, then return the status, headers and body unchanged.

## Gotchas

**`src/code.zip` is a committed build artifact and the deploy uses it directly.**
The module sets `create_package = false` and points `local_existing_package` at that file, so editing `src/main.py` changes nothing until `src/build.sh` has rebuilt the zip and the new zip is committed.
That build runs in Docker on the AWS Lambda Python base image, because the dependencies have to be installed for Lambda's runtime rather than for the machine doing the building.

**Everything passing through is logged.**
The handler prints the forwarded headers, the forwarded body and the response body to CloudWatch, retained for 30 days, so any signing secret or bearer token on an incoming webhook ends up sitting in the log group.

**Nothing here is a tunnel.**
The name invites the assumption that this punches out through something like a Cloudflare tunnel, and it does not.
Public reachability is API Gateway plus an ACM certificate on a custom domain, and private reachability is the Lambda's VPC attachment.
Be careful not to confuse this module with the other similarly named webhook and SMS forwarders, which are separate things.
