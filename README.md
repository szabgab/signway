# Signway

[![Coverage Status](https://coveralls.io/repos/github/gabotechs/signway/badge.svg)](https://coveralls.io/github/gabotechs/signway)

A gateway that proxies signed urls to other APIs.

# Problem statement

Imagine that you have a setup that looks like this. Your backend accesses
a public and authenticated api using a API token, and the response needs 
to be streamed in chunks, because it is a lot of data or because the response
uses [SSE](https://www.w3schools.com/html/html5_serversentevents.asp).

```mermaid
sequenceDiagram
    participant frontend
    participant backend
    participant third_party api
    
    frontend->>+backend: request
    backend->>backend: authentication
    backend->>+third_party api: request + API token
    third_party api->>-backend: data stream
    backend->>-frontend: data stream
```

As you own the backend, you can safely configure there whatever is needed
for authenticating with the third party api, and re-stream back the data
as it comes from the third party api to the end user.

However, if you are using a **serverless** architecture, this gets **tricky** for two
reasons:
1. Most serverless setups don't allow you to stream the response, you can only
send back one blob.
2. Your serverless function would need to live for a very long time, even if it is just
doing slow IO data transfer, so cost may increase significantly.

This is where Signway enters the game. Signway provides you a way of letting the
end user do the request "almost directly" to the third party API in a secure way
without the need of leaking credentials to the users.

The schema using Signway looks like this:

```mermaid
sequenceDiagram
    participant frontend
    participant backend
    participant Signway
    participant third_party api

    frontend->>+backend: request
    backend->>backend: authentication
    backend->>backend: create signed url using an "id" and a "secret"
    backend->>-frontend: signed URL
    frontend->>+Signway: signed URL + request
    Signway->>Signway: verify signature for "id" using "secret"
    Signway->>+third_party api: request + API token
    third_party api->>-Signway: data stream
    Signway->>-frontend: data stream
    
```

This way you leverage heavy IO work to Signway, which is a high performant gateway server
written in Rust prepared for heavy throughput, and you are able to stream data to end users
from APIs that send you data chunk by chunk.

# Signing algorithm

The signing algorithm is inspired strongly in [AWS signature v4](https://docs.aws.amazon.com/AmazonS3/latest/API/sig-v4-authenticating-requests.html),
the same that [s3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/ShareObjectPreSignedURL.html)
uses for generated pre-signed URLs for letting clients interact with buckets directly.

Generating a signed URL requires that the signer know a public `id` and a private `secret`. The `id`
will live in plain text in the signed URL, and the `secret` will be used for creating the request's
signature, but it is not included in the URL.

Signway, who knows which `secret` is associated to which `client`, will take the request and
verify it's signature. If it is authentic and has not expired, it will redirect the request
to the requested URL, adding any preconfigured headers for that `id` like API tokens.

