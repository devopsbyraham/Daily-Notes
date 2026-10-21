# AWS CloudFront (CDN) --- Complete Student Notes and Practical Lab

Beginner-friendly notes covering CDN basics, origin, edge locations,
cache, TTL, Origin Shield, security, pricing, and a hands-on deployment
using two EC2 servers, an Application Load Balancer, and CloudFront.

------------------------------------------------------------------------

## 1. What is a CDN?

**CDN means Content Delivery Network.** A CDN delivers web content
through a geographically distributed network so users can often retrieve
cacheable content from a location closer to them.

**Amazon CloudFront** is AWS's CDN service.

### Real-world examples

-   Cricket/live sports: deliver video segments, images, and supporting
    content to large audiences.
-   E-commerce sales: deliver product images, CSS, JavaScript, and other
    assets during high traffic.
-   Streaming: deliver video and audio segments.
-   Web applications: cache static assets and selected API responses
    where appropriate.

**Memory line:** CDN = deliver content from a location closer to the
user.

## 2. How CloudFront works

``` text
User / Browser
      |
      v
CloudFront Edge Location
      |
      +-- Cache HIT --> Return cached content
      |
      +-- Cache MISS --> Request content from Origin
                              |
                              v
                    ALB / S3 / API Gateway / Web Server
```

On a cache miss, CloudFront retrieves content from the origin. If the
response is cacheable, it can store it for later requests.

## 3. What is an origin?

An **origin** is the source from which CloudFront retrieves the original
version of content.

Common origins: - **Amazon S3 bucket:** images, downloads, static site
assets. - **Application Load Balancer (ALB):** web applications running
on EC2 or containers. - **API Gateway:** API endpoints. - **Custom
origin:** a supported public web server or application endpoint.

**Important:** Origin means where content/application data comes from.
It does not mean the edge location.

## 4. What is an edge location?

An **edge location** is a CloudFront site used to deliver content close
to users. CloudFront routes requests through its global network and
checks its caches.

An edge location does not necessarily run your entire application. It
primarily delivers cached content and processes requests according to
the distribution configuration.

Example: an ALB in Mumbai is the origin. A user elsewhere may receive
cacheable images from a nearby CloudFront edge location rather than
fetching every image from Mumbai.

## 5. What is a cache?

A **cache** is temporary storage for a copy of content so it can be
delivered again without repeatedly fetching it from the origin.

-   **Cache hit:** requested object is available and fresh in the cache;
    CloudFront can serve it from cache.
-   **Cache miss:** object is not available/fresh in that cache;
    CloudFront requests it from the origin or another cache layer.
-   **Cache invalidation:** request CloudFront to remove selected cached
    objects before their normal expiry.
-   **Cache policy:** controls cache key and TTL settings, including
    which query strings, headers, and cookies are included.

### Example

A product image `shoe.jpg` is requested for the first time. CloudFront
fetches it from the origin and may cache it. Subsequent requests can use
the cached copy while it remains fresh and cacheable.

## 6. What is TTL?

**TTL = Time To Live.** In CloudFront, TTL controls how long an object
can remain fresh in the cache before CloudFront needs to revalidate it
or retrieve an updated response, subject to cache policy and origin
headers.

-   **High TTL:** usually improves cache reuse and reduces origin
    requests.
-   **Low TTL:** helps content changes become visible sooner, but may
    increase origin traffic.
-   TTL is not a guarantee that CloudFront will retain an object for the
    full period; cache eviction and invalidations can happen.

For frequently changing content, choose an appropriate cache policy. For
fingerprinted/static assets such as `app.abc123.js`, a long TTL can be
useful because a new deployment creates a new filename.

## 7. What is Origin Shield?

**Origin Shield is an additional centralized caching layer between
CloudFront's other cache layers and the origin.** It can improve cache
hit ratio, consolidate requests for the same content, reduce origin
load, and help protect origin availability.

When choosing an Origin Shield Region: - Prefer the Region with the
lowest latency to the origin. - For an origin in a supported AWS Region,
AWS recommends using the same Region where available. - Do not always
select `us-east-1` by habit. For an origin in Mumbai, check whether
`ap-south-1` is available and appropriate.

Origin Shield can have additional charges; verify current pricing before
enabling it in a cost-sensitive lab.

## 8. Advantages of CloudFront

1.  **Reduce latency:** deliver cacheable content from locations closer
    to users.
2.  **Reduce origin load:** fewer repeated requests reach the origin.
3.  **Potential cost savings:** fewer origin requests and less origin
    processing, although CloudFront itself has pricing.
4.  **Customize delivery:** cache behaviors, path patterns, headers,
    cookies, query strings, and edge functions.
5.  **Security:** HTTPS, AWS WAF integration, signed URLs/cookies, and
    geographic restrictions.
6.  **Handle traffic spikes better:** cached content can be served
    without every request reaching application servers.

CloudFront does not automatically make every dynamic request faster.
Results depend on cacheability, request routing, and origin performance.

## 9. CloudFront vs. Load Balancer

  -----------------------------------------------------------------------
  CloudFront                          Application Load Balancer
  ----------------------------------- -----------------------------------
  CDN and global delivery layer       Distributes requests across
                                      registered targets

  Caches content where allowed        Does not act as a CDN cache

  Helps reduce latency for cacheable  Routes requests to healthy
  content                             application targets

  Can reduce origin traffic           Receives application traffic and
                                      forwards it to targets

  Provides edge features and          Provides target health checks and
  geographic restrictions             routing rules
  -----------------------------------------------------------------------

A common architecture is **User → CloudFront → ALB → EC2 targets**.

## 10. HTTP vs. HTTPS --- critical distinction

There are two separate connections:

1.  **Viewer protocol:** browser/user ↔ CloudFront.
2.  **Origin protocol:** CloudFront ↔ origin (such as ALB).

For a lab where the ALB only listens on HTTP: - Origin Protocol Policy
can be **HTTP Only**. - Viewer Protocol Policy should usually be
**Redirect HTTP to HTTPS** or **HTTPS Only**.

This lets users connect securely to CloudFront over HTTPS while
CloudFront communicates with the ALB over HTTP for the lab. For
production, prefer HTTPS from CloudFront to the origin too, with a valid
origin certificate and HTTPS listener.

CloudFront can redirect viewer HTTP requests to HTTPS using its **Viewer
Protocol Policy**. This is different from the protocol used between
CloudFront and the ALB.

## 11. Practical lab architecture

Goal: deploy a sample e-commerce-style website on two EC2 instances,
distribute traffic with an ALB, and put CloudFront in front.

``` text
Browser
   |
   | HTTPS
   v
CloudFront Distribution
   |
   | HTTP for this lab (origin connection)
   v
Application Load Balancer
   |
   +------> EC2 Instance 1 (Nginx + sample site)
   |
   +------> EC2 Instance 2 (Nginx + sample site)
```

### Before you start

-   Choose one AWS Region and keep the EC2 instances, ALB, target group,
    and origin aligned in that Region.
-   Use a test account and review expected costs.
-   Use a supported Linux AMI and a sample repository that you are
    authorized to deploy.
-   For a public website lab, the ALB must be reachable by CloudFront.
    For production, consider restricting origin access to CloudFront and
    using HTTPS to the origin.
-   Do not store secrets in User Data.

## 12. Step 1 --- Create two EC2 web servers

Create two EC2 instances in suitable subnets. For a beginner lab, use a
supported Ubuntu AMI and a small permitted instance type.

Security groups: - **EC2 security group:** allow inbound HTTP TCP 80
only from the ALB security group. - **ALB security group:** allow
inbound HTTP TCP 80 from the internet for this simple lab, or from
CloudFront's origin-facing address ranges if you deliberately configure
that restriction. - SSH should be restricted to your own trusted IP, or
use Systems Manager Session Manager.

Attach this User Data script to both instances. Confirm that the sample
repository is available and contains the expected website files before
using it.

``` bash
#!/bin/bash
set -euxo pipefail

apt-get update
DEBIAN_FRONTEND=noninteractive apt-get install -y nginx git

rm -rf /tmp/online-clone-amazon
git clone --depth 1 https://github.com/Ironhack-Archive/online-clone-amazon.git /tmp/online-clone-amazon

# Copy website files into the Nginx document root.
cp -a /tmp/online-clone-amazon/. /var/www/html/

# Replace the default page if the repository contains its own index.html.
chown -R www-data:www-data /var/www/html
systemctl enable nginx
systemctl restart nginx
```

Notes: - EC2 User Data scripts normally run as root, so `sudo -i` is
unnecessary. - If the repository is unavailable or its layout differs,
the clone/copy step will fail. Check `/var/log/cloud-init-output.log`. -
Copying `.` also copies the repository's `.git` directory. For a cleaner
production deployment, copy only the required site files and avoid
exposing repository metadata through the web root. - For a real
deployment, pin a trusted repository revision and verify dependencies.

Verify each instance by opening its private/public test URL as
appropriate. Ensure the Nginx page loads before configuring the ALB.

## 13. Step 2 --- Create the Application Load Balancer

1.  EC2 console → **Load Balancers** → **Create load balancer**.
2.  Select **Application Load Balancer**.
3.  Name: `student-web-alb`.
4.  Scheme: **Internet-facing** for this public lab.
5.  Choose the same VPC and at least two suitable Availability
    Zones/subnets.
6.  Listener: HTTP, port 80 for this lab.
7.  Create a target group for **Instances**, protocol HTTP, port 80.
8.  Configure a health check path such as `/`.
9.  Register both EC2 instances as targets.
10. Create the ALB and wait for both targets to become **Healthy**.
11. Open the ALB DNS name in a browser and verify the site.

If targets are unhealthy, check the EC2 security group allows TCP 80
from the ALB security group, Nginx is running, and the health check path
returns a success response.

**Production recommendation:** use an HTTPS listener on the ALB with a
valid certificate, and configure CloudFront to connect to the origin
using HTTPS. The HTTP origin option is for the lab only.

## 14. Step 3 --- Create a CloudFront distribution

The console layout can change, but the main settings are:

1.  Open **CloudFront** → **Create distribution**.
2.  Select a single website/application workflow if the console presents
    that choice.
3.  **Origin domain:** choose the ALB DNS name from the origin list. Do
    not enter the CloudFront distribution's own domain.
4.  **Origin protocol policy:** choose **HTTP Only** for this lab if the
    ALB only has an HTTP listener. Production should normally use HTTPS
    Only with a valid origin certificate.
5.  Configure the default cache behavior. For a simple dynamic website,
    use a suitable cache policy; do not cache personalized or
    authenticated responses unintentionally.
6.  **Viewer protocol policy:** select **Redirect HTTP to HTTPS** or
    **HTTPS Only**.
7.  Enable AWS WAF only if required and configured; WAF may add cost
    depending on pricing plan.
8.  IPv6: leave enabled unless you have a specific reason to disable it.
    Disabling IPv6 is not required for normal CloudFront operation.
9.  Origin Shield: optional. Enable only if justified; select a Region
    suitable for the origin, not automatically `us-east-1`.
10. Review and create the distribution.
11. Wait until distribution status becomes **Deployed**. This can take
    several minutes; do not promise a fixed two-minute activation time.
12. Copy the CloudFront distribution domain name and open it in a
    browser using HTTPS.

If the site uses dynamic content, cookies, query strings, or login
sessions, review the cache policy carefully. CloudFront can cache a
response only according to its cache behavior, cache key, TTLs, and
origin headers.

## 15. Step 4 --- Test caching and verify headers

Run these commands from a terminal, replacing the domain with your
distribution domain:

``` bash
curl -I https://YOUR_DISTRIBUTION_DOMAIN/
curl -I https://YOUR_DISTRIBUTION_DOMAIN/
```

Look for headers such as `X-Cache` and `Age` where present. A response
may say `Hit from cloudfront` or `Miss from cloudfront`. The first
request is not guaranteed to be a miss, and the second is not guaranteed
to be a hit: caching depends on behavior settings, cacheability,
headers, and whether the object is already cached.

Check: - The CloudFront distribution is deployed. - The default behavior
points to the intended ALB origin. - The ALB targets are healthy. -
Viewer protocol policy redirects HTTP to HTTPS or requires HTTPS. -
Origin protocol matches the ALB listener. - Cache policy suits the type
of content. - CloudFront and ALB logs/metrics are used for deeper
troubleshooting.

## 16. How to block users from different countries

CloudFront supports country-level geographic restrictions.

1.  Open CloudFront → **Distributions**.
2.  Select the distribution.
3.  Open **Security** → **Geographic restrictions**.
4.  Select **Edit**.
5.  Choose **Allow list** to permit only selected countries, or **Block
    list** to deny selected countries.
6.  Add the country codes/countries and save changes.
7.  Wait for the distribution update to deploy and test from
    permitted/blocked locations where possible.

Geo restriction is country-level, not a precise city/IP allowlist.
CloudFront estimates viewer location from IP geolocation, so it is not a
replacement for authentication or application authorization.

## 17. Important security points

-   Use HTTPS from viewers to CloudFront.
-   Prefer HTTPS from CloudFront to the origin in production.
-   Use AWS WAF for suitable request filtering, rate-based rules, and
    managed protections.
-   For S3 origins, use Origin Access Control (OAC) to limit direct
    bucket access where supported.
-   For an ALB origin, consider restricting direct origin access to
    CloudFront and using a secret custom header or another robust
    origin-protection approach; do not rely on obscurity alone.
-   Use signed URLs or signed cookies for private content when
    appropriate.
-   Never cache personalized/private responses for shared users unless
    the cache key and policies are explicitly designed for that
    behavior.
-   Use least privilege and protect deployment credentials.

## 18. CloudFront pricing and free usage

The AWS CloudFront **pay-as-you-go Free Tier** has historically included
each month: - 1 TB of data transfer out. - 10,000,000 HTTP or HTTPS
requests. - 2,000,000 CloudFront Functions invocations.

AWS also offers separate **flat-rate CloudFront plans** with different
allowances and included features. Check the current pricing page and the
plan selected for the distribution before teaching costs. Not every
feature is free: Origin Shield, invalidations beyond any included
allowance, WAF or other related services, and origin resources may have
separate charges depending on pricing model and usage.

References: - https://aws.amazon.com/cloudfront/pricing/ -
https://aws.amazon.com/cloudfront/faqs/

## 19. Common troubleshooting

  -----------------------------------------------------------------------
  Symptom                 Possible cause          Check
  ----------------------- ----------------------- -----------------------
  CloudFront returns 502  Origin TLS/certificate  Origin protocol,
                          problem, origin         certificate, ALB
                          connectivity, or        listener, logs
                          invalid response        

  CloudFront returns 403  Access policy, WAF, geo Distribution behavior,
                          restriction, origin     WAF, S3/OAC policy, geo
                          permission, or wrong    settings
                          origin                  

  CloudFront returns 504  Origin did not respond  ALB target health, app
                          in time                 latency, security
                                                  groups, routes

  ALB works but           Incorrect origin or     Origin domain, listener
  CloudFront does not     protocol configuration  port, origin protocol

  Changes are not visible Cached object has not   TTL/cache policy;
                          expired                 invalidate only if
                                                  necessary

  Targets unhealthy       Health-check path/port  Nginx, target group
                          or security group issue health reason,
                                                  ALB-to-EC2 rule

  HTTPS fails to origin   Certificate is invalid, ALB certificate and
                          mismatched, or origin   CloudFront origin
                          does not support HTTPS  policy

  Website breaks after    Dynamic/authenticated   Cache policy, cookies,
  caching                 responses cached        headers, query strings
                          incorrectly             
  -----------------------------------------------------------------------

## 20. Interview questions and answers

**Q1. How do you improve website delivery speed globally?**\
Use a CDN such as Amazon CloudFront to serve cacheable content from edge
locations closer to users.

**Q2. What is an edge location?**\
A CloudFront site used to deliver content close to viewers and serve
cached objects.

**Q3. What is an origin?**\
The source CloudFront retrieves original content from, such as S3, an
ALB, API Gateway, or a custom web server.

**Q4. Name common CloudFront origins.**\
Amazon S3, Application Load Balancer, API Gateway, and supported custom
origins.

**Q5. Can CloudFront redirect HTTP to HTTPS?**\
Yes. Set the cache behavior's Viewer Protocol Policy to **Redirect HTTP
to HTTPS**.

**Q6. What is TTL?**\
Time To Live: how long an object can remain fresh in CloudFront's cache
before it needs revalidation or a new fetch, subject to the policy and
origin headers.

**Q7. What is a cache hit?**\
The requested object is available and fresh in the cache and can be
returned without fetching it from the origin.

**Q8. What is a cache miss?**\
The object is not available/fresh in the current cache layer, so
CloudFront needs to retrieve it from the origin or another cache layer.

**Q9. What is Origin Shield?**\
An additional cache layer that can improve cache hit ratio and reduce
requests to the origin.

**Q10. What is the difference between CloudFront and an ALB?**\
CloudFront is a global CDN/cache and delivery layer; ALB routes requests
to registered application targets and performs health checks.

**Q11. Can CloudFront cache dynamic content?**\
It can cache some dynamic responses if configured and safe to do so, but
personalized or authenticated responses need careful cache policy
design.

**Q12. What is geographic restriction?**\
A CloudFront feature to allow or block delivery at country level.

**Q13. Should Origin Shield always be set to `us-east-1`?**\
No. Choose the supported Region with the lowest latency to the origin;
for AWS origins, same Region is recommended where available.

**Q14. Does CloudFront make every request faster?**\
No. Benefits depend on content cacheability, request path, origin
performance, and configuration.

**Q15. Why put CloudFront in front of an ALB?**\
To improve global delivery for cacheable content, reduce origin
requests, and use edge delivery/security features while ALB distributes
application requests to healthy targets.

## 21. One-line memory map

-   **CDN:** Globally distributed content delivery.
-   **CloudFront:** AWS CDN service.
-   **Origin:** Source of original content.
-   **Edge location:** CloudFront delivery location close to viewers.
-   **Cache:** Temporary copy of content.
-   **Cache hit:** Object served from cache.
-   **Cache miss:** Object must be retrieved from an upstream
    cache/origin.
-   **TTL:** Cache freshness lifetime.
-   **Origin Shield:** Additional centralized cache layer protecting the
    origin from repeated requests.
-   **Viewer protocol:** Browser ↔ CloudFront.
-   **Origin protocol:** CloudFront ↔ origin.
-   **Geo restriction:** Country-level allow/block control.
-   **Invalidation:** Request removal of selected cached objects.
-   **ALB:** Distributes application traffic to targets.
-   **OAC:** Restricts access to supported S3 origins through
    CloudFront.

## 22. Cleanup checklist

-   [ ] Disable/delete the CloudFront distribution when the lab is
    complete; deletion may require disabling it first and waiting for
    deployment.
-   [ ] Delete the ALB and target group if no longer needed.
-   [ ] Stop/terminate both EC2 lab instances.
-   [ ] Check EBS volumes, snapshots, Elastic IPs, and other billable
    resources.
-   [ ] Remove unused security group rules and temporary IAM
    permissions.
-   [ ] Review WAF, Origin Shield, and CloudFront pricing settings
    before leaving resources enabled.
