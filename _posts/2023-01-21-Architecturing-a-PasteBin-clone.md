---
title: "Architecturing: PasteBin clone" 
published: false
tags:
    - system design
    - architecture
---

There are tons of posts on internet that goes through replicating websites. Most of those are just reposts, of course. Me posting an original solution is not going to save the world, but I have other goals in mind. I'm trying to improve my system design skills. So best way to do that is actually doing the system design and document it. This is good for both my design skills, and my technical writing. So here goes nothing...

## Requirements

We will design a website to post text and retrieve it later. After a text is posted, the website will generate an URL for it and text can later be retieved using that url. System is completely anonymous, and once created bins can't be deleted or updated.

1. Creating new text files
2. Retrieving text files
3. Maybe rate limiting

## Scale

I can forecest 2 use cases for a service like this. First is write once and read few times, and the second is write once and read often, very often. I.e. we will have some hot postings that get requested A LOT.

This system can horizontally scale with minimal effort.

## Rough design

Lets first specify a few basics.

- We will build API endpoints. Website will be completely static.
- We will use a NoSQL/Distributed DB as storage. Cassandra is a good example. We want A and P of CAP here.
- A simple rate limiter based on IP can be used.

<pre class="language-mermaid">
graph TD
    U1[User 1] -->|POST text| API1(API Server 1)
    U2[User 2] -->|GET with id| API2(API Server 2)

    API1 --> RATE_LIMITER{Rate Limiter}
    API2 --> RATE_LIMITER
    RATE_LIMITER --> BACKEND[Backend Servers]

    ID_SERVER{ID Creator} -->BACKEND
    BACKEND -->|Writes| DB[Cassandra cluster]
    BACKEND -->|Reads| DB
</pre>

We will position API gateways in front of our backend servers, so we can scale our backend and API handling separately and also have robust rate limiting.

On the DB side, we will select a AP database with consistent hashing so we can also scale our storage horizontally. 

Backend servers will be pretty simple. Get the text and create a DB row. Then return the ID/url for that row.

Read operation is just a read call to DB, and error handling in case of wrong urls.

## Details

### Creating notes

When creating notes, user `POST`s to API servers. These API servers can be generic AWS API servers too. It would make more sense to use those since they have auth and rate limiting built in, so we can save some development time.

After going through optional rate limiting, requests will be handled by backend servers. These servers will create the url for the note, then write it to DB. We can use different ID generators here. Use some machine/network ID and use the time information. 64 bits of ID information should be enough.

### Retieving notes

This case should be easier. Just ask the DB and return what it gives you. We can also add a caching layer so frequently requested items do not even hot DB.

## Final System overview

<pre class="language-mermaid">
graph TD
    U1[User 1] -->|POST text| API1(API Server 1)
    U2[User 2] -->|GET with id| API2(API Server 2)

    API1 --> RATE_LIMITER{Rate Limiter}
    API2 --> RATE_LIMITER
    RATE_LIMITER --> CACHE(Cache check)
    
    CACHE -->|If misses| BACKEND[Backend Servers]

    ID_SERVER{ID Creator} -->BACKEND
    BACKEND -->|Writes| DB[Cassandra cluster]
    BACKEND -->|Reads| DB
</pre>

<script src="https://cdn.jsdelivr.net/npm/mermaid@9.3.0/dist/mermaid.min.js"></script>

<script>
    const config = {
    startOnLoad:true,
    flowchart: {
        useMaxWidth:false,
        htmlLabels:true
        }
    };
    mermaid.initialize(config);
    window.mermaid.init(undefined, document.querySelectorAll('.language-mermaid'));
</script>