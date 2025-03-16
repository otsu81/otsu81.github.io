---
layout: post
title: "Cloudflare Gotchas"
---

We're switching from Akamai to Cloudflare for our CDN. I wasn't involved in the decision to make the switch, but I'm part of the team migrating our properties. Our backend is entirely AWS. Overall it's been a pleasant experience and I can honestly say the developer experience is superior to Akamai. While Akamai oftentimes felt like pushing buttons on a nuclear power plant control board, Cloudflare is more like skateboarding. In a minefield.



Here are some gotchas I've encountered. It's not an exhaustive list and I'm fully aware I might misunderstand or be incorrect about some of my assumptions. In somce sense I'm kinda hoping I am (see [Cunningham's Law](https://meta.wikimedia.org/wiki/Cunningham%27s_Law))

## Cloudflare's IaC tooling... or rather lack thereof
You can use Terraform, Pulumi, or Cloudflare's own Wrangler to deploy and manage your Cloudflare resources. But why isn't a single one entirely comprehensive? Biggest complaint: I need to use several tools to put my configuration into code.

* **Wrangler** is Cloudflare's own tool but primarily covers Workers, Pages, KV, Durable Objects, and R2. Crucially, I can't use Wrangler to manage e.g. DNS, Zero Trust, or Access.

* **Terraform** is fairly comprehensive, but is riddled with its own set of gotchas. For example I can get a nasty surprise like this, without any warning from the resource specification:

```
│ Warning: Resource Destruction Considerations
│
│ with cloudflare_zone_setting.always_use_https["dev.************.net"],
│ on cf-zone.tf line 20, in resource "cloudflare_zone_setting" "always_use_https":
│ 20: resource "cloudflare_zone_setting" "always_use_https" {
│
│ This resource cannot be destroyed from Terraform. If you create this resource, it will be present in the API until manually
│ deleted.
│
│ (and 8 more similar warnings elsewhere)
```

... or that many of the different types of rules do not allow me to set priority (kind of important that e.g. the auth flow runs at a specific priority). Examples, as of v.5.1.0:
  * [`cloudflare_access_rule`](https://registry.terraform.io/providers/cloudflare/cloudflare/latest/docs/data-sources/access_rules)
  * [`cloudflare_user_agent_blocking_rules`](https://registry.terraform.io/providers/cloudflare/cloudflare/latest/docs/data-sources/user_agent_blocking_rules)
  * [`cloudflare_load_balancer_pool`](https://registry.terraform.io/providers/cloudflare/cloudflare/latest/docs/resources/cloud_connector_rules)

For me the main selling point of Terraform is its ability to manage a state and guarantee that my configuration will always be the same, even if there's been out of band changes. But if the Cloudflare provider isn't even able to delete what it has created, what's the point?

* **Pulumi** on the surface looks promising but it's not nearly as widespread as Terraform. Also, am I mistaken or is Pulumi working its way towards being a [Terraform wrapper](https://github.com/pulumi/pulumi-terraform-bridge)? Is the Pulumi provider just a wrapper to [use the Terraform provider](https://github.com/pulumi/pulumi-cloudflare/issues/986) with a more popular language than HCL? As we're a fairly small devops team that are responsible for the whole wheat to chaff, I'm not sure I want to introduce a new tool that might be harder to find help for, or worse be stuck handling two separate set of gotchas from two different tools.

## Sharing secrets between workers
There is no easy way to share secrets between Workers,e.g. API keys. If we want to maintain secrets we need to maintain it for every worker, for every environment. When that number starts to approach tripple digits, combined with the lack of comprehensive IaC tooling, it's not a great look.

At the time of writing the [Cloudflare Secrets Store has been in beta for almost two years](https://blog.cloudflare.com/secrets-store/). When asking about it our TAM simply said, "we'll keep you posted" (or something to that effect).

My team's workaround has been to use KV, but it's not ideal as it's currently not possible to mask values. That means all secrets are stored in plaintext. I guess the upside is that it has forced us to be very strict with permission scoping, and introduced a Lambda that rotates the IAM keys several times a day.

## Cloudflare's liberal borrowing of syntax from other sources

There's a habit of wanting to make it easier for customers to switch to Cloudflare by making API and syntax "compatible" with other products. Sometimes it's great, other times not so much. The key takeaway is that just because it's easier to get started you still have to dig through Cloudflare documentation to learn their specifics and gotchas. Examples:
* R2 famously is S3 "compatible". You can even use the AWS CLI to interact with it. The problem is that it means the CLI expects keys to be present in the same format, either through environment variables or config file. The result is that it's a chore to be working with Cloudflare R2 in parallel with AWS. Cloudflare really wants you to use R2 instead of S3. But good luck using my favorite CLI command `aws s3 sync` between R2 and S3 buckets.
* The `fetch` API is a core component of Cloudflare workers, and supposedly compatible with the [Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API). Unfortunately it seems to be Cloudflare's own implementation, and it's not well documented what compatibility is achieved (there's no mention of Cloudflare on the browser compatibility chart). You can't make assumptions of which headers are available, or how they are formatted. It's a bit of a guessing game, trial and error. Intimate familiarity with how Cloudflare's [Request interface](https://developers.cloudflare.com/workers/runtime-apis/request/) is implemented is recommended.
* The Cloudflare Rules language is based on Wireshark display filters. However it's not an exact mapping, and Cloudflare uses its own set of fields (like `http.request.uri.path`). It's not a big deal, but it does mean that even if you're familiar with Wireshark, you need to learn the Cloudflare syntax.

## Certificate priority and wildcards
You cannot specify what certificate a Cloudflare worker should use (🤯). You are expected to be intrically familiar with the [certificate priority](https://developers.cloudflare.com/ssl/reference/certificate-and-hostname-priority/) and how it works. Specifically, a specific subdomain certificate will **always** be used over a wildcard certificate, regardless of the priority liost. Conversely, if you want to use the hassle-free cost-free auto-renewing certificate "Universal" service combined with third party certificates (DigiCert, anyone?), you will never be able to use a bought wildcard certificate! For us it was a big gotcha, as we are dependent on our own CA as a firmware OTA endpoint, and had used gotten used to using a wildcard certificate for all our subdomains. Common Cloudflare. Why not just add the option of letting me choose which certificate to use?!

## Worker CPU runtime, memory, and AWS SDK
Cloudflare workers are measured primarily on CPU time, secondly on execution time. You must be very familiar with the limits, what applies for your specific Cloudflare plan, and how each and every dependency in your worker impacts execution time. Idle time is not counted; if you're using a worker as a reverse proxy I strongly advise to use the native `fetch` API instead of third party libraries. For our usecase of serving large firmware files, we quickly discovered that the AWS SDK and its S3 client consumes a lot of CPU time while processing. Also, trying to fit a downloaded object into memory proved to be a challenge; there's some kind of swap logic that consumes more CPU. The easiest way to profile CPU usage is through the [Wrangler CLI's native capability](https://developers.cloudflare.com/workers/observability/dev-tools/cpu-usage/).

Our workaround was to use S3 pre-signed URLs and let the native `fetch` API handle the download. It makes the most efficient type of download with the least amount of fuss. It does however mean that we have considerably less flexibility compared to how we are used to working with Lambda.

## Split horizon DNS

Our DNS is primarily managed by Route 53. To leverage the DNS services and automagic mapping inside Cloudflare to workers and resources, we have configured [Cloudflare DNS in a partial setup](https://developers.cloudflare.com/dns/zone-setups/partial-setup/). In theory we get the best of both worlds - free TLS certificates and automatic management of worker endpoints, while retaining the tight coupling with Route 53 inside AWS. In practice it's a bit of a hassle to manage, as we need to keep track of which records are managed by Cloudflare and which are managed by Route 53. Each resource in Cloudflare must be mapped in Route 53 with its own CNAME. Due to the previously mentioned lack of comprehensive IaC tooling we end up with using a secondary tool (AWS CDK) *just to manage the Route 53 records mapping to Cloudflare*.

Another gotcha is that workers do not look outside of Cloudflare for DNS resolution. This means that if you have a worker that needs to resolve a DNS record pointing to a resource outside of Cloudflare on the same subdomain, it will not resolve without making a modification to the request (see [`resolveOverride` in the Request](https://developers.cloudflare.com/workers/runtime-apis/request/)).

## The 413 Request Entity Too Large error
The `413` error is a custom Cloudflare error. We monitored the [instant logs](https://developers.cloudflare.com/logs/instant-logs/) after deploying some workers, and noticed this error a lot. We were very confused - we were not uploading anything. After frantic searching and conversation with support we learned that hidden deep in the documentation this was [expected behavior when bypassing the Cloudflare cache](https://developers.cloudflare.com/workers/runtime-apis/cache/#put, e.g. through a cache rule.

Lesson learned: Error messages can be confusing, unhelpful, and by design.
