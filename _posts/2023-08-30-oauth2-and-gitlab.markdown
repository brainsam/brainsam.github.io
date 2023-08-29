---
layout: post
title:  "Gitlab As Auth Provider For Static Site"
category: devops
date:   2023-08-29 00:00:00 +0400
---

### Problem
We have a documentation site built on the docusaurus.io project. It is a static html generator with markdown support and rich features.
We want the site to be available WorldWide but with restricted access. One of the possible solutions could be Gitlab/Github pages or external web frontend services, like vercel.com.  However, I chose a self-hosting solution on private infrastructure to be able to move documentation to a restricted corporate network segment.

### Choose solution
[OAuth2 Proxy](https://oauth2-proxy.github.io/oauth2-proxy/)- is a reverse proxy and a static file server that provides authentication through Identity Providers (Github, Gitlab, Keycloak, and others)

Possible scheme of auth could be:

```mermaid
C4Context
  title Docusaurus with Oauth2 authentication
  Person(user, "User")
  Boundary(infra, "") {
    System(site,"docusaurus", $sprite="nginx_original")
    System(proxy,"Oauth2-Proxy")
    System(gitlab, "Private gitlab")
  }
  BiRel(user, proxy, "Access the site")
  Rel_R(user, gitlab, "Get Auth Token")
  BiRel(proxy, site, "Proxy to upstream")
  Rel(proxy, gitlab, "Check validity of gitlab Auth")
```

```mermaid
---
title: Sequence diagram
---
sequenceDiagram
  User->>Proxy: GET docs.example.com
  Proxy->>User: Gitlab auth token
  User->>Gitlab: OAuth
  Gitlab->>User: OAuth token
  User->>Proxy: OAuth2 token
  Proxy->>Site: HTTP Request
  Site->>Proxy: HTTP Response
  Proxy-)User: HTTP Response
```

### Implementation

1. Generate credentials for GitLab as OAuth2 Identity source as defined in [documentation](https://docs.gitlab.com/ee/integration/oauth_provider.html)
  1. Go to https://gitlab.example.com/admin/applications
  2. Create new application
  3. Select openid, profile and email permissions checkboxes
  4. Save application and secret IDs for futher oauth2-proxy config file
2. [Install](https://oauth2-proxy.github.io/oauth2-proxy/docs/) oauth2-proxy. Choose the preferred way: Kubernetes helm, docker image, or binary file. Let assume we use binary
3. Configure proxy

```
# /etc/oauth2-proxy.conf
email_domains = ["*"]
provider = "gitlab"
redirect_url = "https://docs.example.com/oauth2/callback"
oidc_issuer_url = "https://gitlab.example.com"
upstreams = "http://docusaurus.local"

```
4. Run proxy
   ```bash
   oauth2-proxy --config /etc/oauth2-proxy.conf
```

And thats it
![result](/assets/images/20230829233322.png)