---
title: "Self-hosting Hugo"
date: 2025-03-23T20:31:58+01:00
draft: false
toc: false
description: "This is the first post on my new Hugo blog which is being hosted
on my Hetzner dedicated server, served by Traefik and Minio."
images:
author: "Tobias Schäfer"
tags:
  - hugo
  - self-hosting
  - traefik
  - minio
---

This is the first post on my new `hugo` blog which is being hosted on my
Hetzner dedicated server. Amongst others the server is running traefik and
minio as Docker containers. Both services build the backend for the blog.

I went to these steps to run the blog.

* Created a bucket on minio to store the blog content.
* Created a user and a policy to deploy the blog with the `hugo` cli.
* Added a deployment target in the `hugo` site configuration.
* Setup a traefik router, a service, and several middlewares to serve the
  blog.

## The Minio setup

Create the bucket, the user, and the policy with the minio client.


```bash
mc mb coresec/blog.tschaefer.org
mc admin user add coresec hugo secret
mc admin policy create coresec blog blog.json
mc admin policy attach coresec blog --user hugo
```
The blog policy looks like this.
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:*"
            ],
            "Resource": [
                "arn:aws:s3:::blog.tschaefer.org/*"
            ]
        }
    ]
}
```

Add a deployment target in the `hugo` site configuration.

```yaml
deployment:
  targets:
    - name: production
      url: s3://blog.tschaefer.org?endpoint=https://minio.coresec.zone
```

Deploy the blog.

```bash
AWS_ACCESS_KEY_ID=hugo \
AWS_SECRET_ACCESS_KEY=secret \
AWS_DEFAULT_REGION=ger-south-gap-1 \
hugo deploy
```

## The Traefik setup

Create a TLS terminated router and redirect all requests to the minio service.


```yaml
routers:
    blog:
        rule: Host(`blog.tschaefer.org`)
        service: minio
        middlewares: path
        tls:
            certResolver: letsencrypt
services:
    minio:
        loadBalancer:
            servers:
                - url: http://minio-service:9000
```

For the usage of the bucket and to keep pretty URLs, the request path has to
be modified by several middlewares.

First add the bucket name as a prefix to the request path.

```yaml
bucket:
    addprefix:
        prefix: /blog.tschaefer.org
```

If the request path is the root path, add the main `index.html`.

```yaml
root:
    replacepathregex:
        regex: ^/$
        replacement: /index.html
```

In any other case remove a trailing slash.

```yaml
slash:
    replacepathregex:
        regex: ^(.*)/$
        replacement: "${1}"
```

To any request path not ending with a file extension add `index.html`.

```yaml
index:
    replacepathregex:
        regex: ^(/(?:[\w-]+/)*[\w-]+)$
        replacement: ${1}/index.html
```

For error handling, redirect to the main error page.

```yaml
error:
    errors:
        status: "404"
        query: /blog.tschaefer.org/404.html
        service: minio
```

Chain all middlewares together and use them in the above router.

```yaml
path:
    chain:
        middlewares: bucket,root,slash,index,error
```

## Conclusion

The blog is up and running. I learned a lot, especially about Traefik and
it's powerfull middleware system. I'm looking forward to writing more posts
and improve the setup over time.
