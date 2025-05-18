---
title: "Analyze and Anonymize Data in Real-Time"
date: 2025-05-13T10:48:20+02:00
draft: false
toc: false
images:
tags:
  - serverless functions
  - openfaas
  - microsoft presidio
  - PII
  - data privacy
---

In today’s digital age, the intersection of artificial intelligence (AI) and
privacy is a critical concern. As organizations increasingly rely on
data-driven insights, the need to protect personally identifiable information
(PII) becomes paramount. This post explores how to redact sensitive data with
the [OpenFaaS](https://www.openfaas.com) function [Maceo](https://github.com/tschaefer/maceo).
Maceo utilizes [Microsoft Presidio](https://github.com/microsoft/presidio) to
analyze and anonymize data.

Presidio is an open-source framework for detecting, redacting, masking, and
anonymizing sensitive data (PII) across text, images, and structured data.
It supports NLP, pattern matching, and customizable pipelines.

Presidio's features two main services for anonymization PII in text:

* Presidio analyzer: Identification of PII in text
* Presidio anonymizer: De-identify detected PII entities using different operators

In most cases, we would run the Presidio analyzer to detect where PII entities
exist, and then the Presidio anonymizer to remove those using specific operators
(such as redact, replace, hash or encrypt)

![Presidio Flow](https://microsoft.github.io/presidio/assets/analyze-anonymize.png)

Maceo wraps the Presidio analyzer and anonymizer services and simplifies them
in one single step receiving a text and returning the anonymized text.

![Architecture](/images/openfaas-maceo-hetzner.png)

We will deploy Maceo and Presidio services with
[faasd](https://docs.openfaas.com/deployment/edge/), and have a look at the
function configuration.

## Prerequisites

- Up and running faasd.
- Local [faas-cli](https://docs.openfaas.com/cli/install/) and SSH client
  to deploy and manage the function and the Presidio services.

## Deploy Presidio analyzer and anonymizer service

faasd manages both functions and services. The latter are configured with a
`docker-compose.yaml` file, located in `/var/lib/faasd`. We extend the file
to deploy the Presidio analyzer and anonymizer services.

```diff
    cap_add:
      - CAP_NET_RAW
    depends_on:
      - nats
+  presidio-analyzer:
+    image: mcr.microsoft.com/presidio-analyzer:2.2.358
+    ports:
+      - "10.62.0.1:5001:3000"
+  presidio-anonymizer:
+    image: mcr.microsoft.com/presidio-anonymizer:2.2.358
+    ports:
+      - "10.62.0.1:5002:3000"
```

The services are exposed on the OpenFaaS internal network and are accessible
from any function and service. To apply the changes, restart faasd.

```bash
sudo systemctl restart faasd
```

## Deploy Maceo function

Maceo requires a configuration file provided as an OpenFaaS secret named
`maceo` in JSON format. We start with an empty configuration file and have a
detailed look at it later.

```bash
export OPENFAAS_URL=https://faasd.example.com
faas-cli secret create --from-literal '{}' maceo
```
Finally, we deploy the Maceo function.

```bash
faas-cli deploy --image ghcr.io/tschaefer/maceo:0.1.1 --name maceo
```

## Test Maceo function

### Health check

The function exposes a health check endpoint at `/health`. The check verifies
the syntax of the configuration and the availability of the Presidio services.
It returns a 200 OK status if everything is working correctly and a 500 status
if there are any issues.

```bash
curl --include https://faasd.example.com/function/maceo/health
HTTP/2 200
content-type: text/plain
date: Tue, 13 May 2025 17:42:59 GMT
x-call-id: 16d38bb5-e2a6-4537-a022-b6b13d3a4bc0
x-duration-seconds: 0.019462
x-maceo-commit: ca7eea42494e47927860f2493ef05f54fbfd3a1f
x-maceo-version: 0.1.0
x-openfaas-eula: openfaas-ce
x-served-by: openfaas-ce/0.27.12
x-start-time: 1747158179257965489
content-length: 0
```

Detailed information about any issue are available in the logs.

```bash
faas-cli logs maceo
```

Further, any response contains two headers with version information, the
release number and the tagged commit hash. This is useful for debugging and to
verify that the function is running the expected version.

### Analyze and anonymize data

Maceo expects a POST request with a plain text body. Currently only text
in English is supported and the following PII entities are detected.

- PHONE_NUMBER
- US_DRIVER_LICENSE
- US_PASSPORT
- LOCATION
- CREDIT_CARD
- CRYPTO
- UK_NHS
- US_SSN
- US_BANK_NUMBER
- EMAIL_ADDRESS
- DATE_TIME
- IP_ADDRESS
- PERSON
- IBAN_CODE
- NRP
- US_ITIN
- MEDICAL_LICENSE
- URL

```bash
curl --include --request POST https://faasd.example.com/function/maceo \
    --header "Content-Type: text/plain; charset=utf-8" \
    --data-binary  "@contrib/sample"
HTTP/2 200
content-type: text/plain; charset=utf-8
date: Tue, 13 May 2025 18:02:40 GMT
x-call-id: 7ab69f7a-0670-4d79-93dd-c9d9fd4a04fe
x-duration-seconds: 0.123645
x-maceo-commit: ca7eea42494e47927860f2493ef05f54fbfd3a1f
x-maceo-version: 0.1.0
x-openfaas-eula: openfaas-ce
x-served-by: openfaas-ce/0.27.12
x-start-time: 1747159360673811820
content-length: 1152

Sample Job Application Letter
Ms. <PERSON>
DSC Company
68 <LOCATION>, CA 09045
<PHONE_NUMBER>

<DATE_TIME>

Dear Ms. <PERSON>,

I am writing this letter to apply for a junior <IN_PAN> position <IN_PAN> in your organisation. As requested, I am enclosing a completed job application, my certificates, my resumes, and four <IN_PAN> in this letter.

The opportunity presented in this listing is exciting. I believe that my firm and <DATE_TIME> of technical experiences and education will make me a competent person for the position. The main strengths that I have, which I will <IN_PAN> to this position include:

I have designed, developed and supported many different live use applications.
I continuously work towards achieving my goals through hard work and <IN_PAN>.
I provide exceptional contributions to the needs and wants of the consumers.
I have a Bachelor of Science degree in Computer Programming. Additionally, I have in-depth knowledge of the complete cycle of a soft development project. Whenever the need arises, I learn new technologies.
I can be reached on <PHONE_NUMBER>.
Thank you for your time and consideration.

Sincerely,

<PERSON>
```

## Function configuration

The function configuration allows you to customize the behavior of Maceo.

The `upstream` section defines the Presidio services endpoints and gives the
possibility to use external Presidio services. The default endpoints are the
one of the above deployed services.


```json
{
    "upstreams": {
        "analyze": "https://presidio-analyzer.example.com",
        "anonymize": "https://presidio-anonymizer.example.com"
    }
}
```

The list of PII entities to be detected and anonymized is defined in the
section `entities` and can be a subset of the supported entities. By default,
all supported entities are used.

```json
{
    "entities": [
        "CREDIT_CARD",
        "EMAIL_ADDRESS",
        "IBAN_CODE",
        "IP_ADDRESS",
        "LOCATION",
        "PERSON",
        "PHONE_NUMBER"
    ]
}
```

The `anonymizer` section defines the anonymization operators to be used. The 
default operators is `replace`, which replaces the detected PII entity with an
entity label.

```json
"anonymizers": {
    "DEFAULT": {
        "type": "hash",
        "new_value": "sha512"
    }
}
```
The `ad_hoc_recognizers` section allows you to define custom regex and
deny-list based logic ad-hoc recognizers. By default, the section is empty.

```json
"ad_hoc_recognizers": [
    {
    "name": "Zip code Recognizer",
    "supported_language": "en",
    "patterns": [
        {
        "name": "zip code (weak)",
        "regex": "(\\b\\d{5}(?:\\-\\d{4})?\\b)",
        "score": 0.01
        }
    ],
    "context": ["zip", "code"],
    "supported_entity":"ZIP"
    }
]
```
Finally, the `score_threshold` section defines the minimum score for an entity
to be detected. The default value is 0.0, and the maximum value is 1.0.

```json
{
    "score_threshold": 0.5
}
```

To apply configuration changes, store them in a file and update the OpenFaaS
secret.

```bash
faas-cli secret update --from-file config.json maceo
```


## Conclusion

In this post, we explored how to deploy the Maceo function and the Presidio
analyzer and anonymizer services with faasd. We also looked at the function
configuration and how to test the function. Maceo is a powerful tool for
analyzing and anonymizing data in real-time, ensuring that sensitive
information is protected. This allows organizations to leverage the power of
AI while maintaining data privacy and compliance with regulations.

## References

- [OpenFaaS](https://docs.openfaas.com)
- [Microsoft Presidio](https://microsoft.github.io/presidio/)
- [Maceo](https://github.com/tschaefer/maceo)
