---
layout: blog
title: In search of lost nines
subtitle: Fire, water, lightning, drones, and a DNS TTL nobody thought to lower.
date: 2026-09-21 09:00:00
categories: Cloud
---

Under the legalese, every cloud contract I have signed was a promise made in nines: 99.9% for the VM, 99.99% for the load balancer, and eleven of them, 99.999999999%, for every object I put in S3. Nines are the common currency of reliability because they are easy to compare and easy to multiply. They are not a law of physics, though. They are a statistical claim, and it rests on one assumption: that things fail independently of each other.

Buildings don't read the SLA. In the last five years a data centre has burned down, another has flooded, and this month AWS confirmed that a cloud region hit by drone strikes is not coming back, and neither is the data that lived only there. Proust needed seven volumes to go looking for lost time. Lost nines are quicker to find, because they always turn up in the same place. This post goes through those incidents, plus a few others, and looks at where the nines went.

## Two kinds of nines

**Availability** is about time: the fraction of the year the service answers when you call it.

| Availability | Downtime allowed per year |
| --- | --- |
| 99% | 3 d 15 h 36 m |
| 99.9% | 8 h 46 m |
| 99.99% | 52 m 34 s |
| 99.999% | 5 m 15 s |

**Durability** is about data: the probability that an object you stored is still there at the end of the year. S3 Standard is designed for eleven nines, which in [AWS's own illustration](https://aws.amazon.com/s3/faqs/) means that if you store ten million objects, you can expect to lose one of them every ten thousand years. It gets there by spreading every object across at least three Availability Zones: separate buildings with separate power and cooling, far enough apart that no single ordinary event can reach every copy.

Both numbers are honest, within their envelope. The envelope is independence. Every incident below happened at the edge of that envelope, and every one is really about correlation: two copies that turned out to share a building, a power path, a country or a DNS record.

## Strasbourg, March 2021: fire

Shortly after half past midnight on 10 March 2021, a UPS failed in OVHcloud's SBG2 building in Strasbourg and started a fire. By morning SBG2 was gone and four of the twelve rooms in neighbouring SBG1 were destroyed. SBG3 and SBG4 were largely untouched, but they were dark anyway, because the whole site had to be de-energised. Netcraft counted 3.6 million websites on 464,000 domains knocked offline.

France's industrial accident investigators (BEA-RI) published [their report](https://www.theregister.com/2022/06/10/ovhcloud_datacenter_fire_last_year/) the following year. They could not say for certain why the UPS failed. Ambient temperature and humidity rose shortly before, and moisture inside an electrical device is a plausible cause of a short circuit. They were much clearer about why the fire won. There was no automatic extinguishing system. Cutting the electrical supply took too long, partly because the backup generators kept starting, which is exactly what they were built to do. And the building's design let the fire spread. Firefighters needed a pump boat on the Rhine and about nine and a half hours to put it out.

I wrote about Strasbourg a few months later in [Can your DB lose your data?]({% post_url 2021-09-23-DataLoss %}). The line that has aged best is the one about SBG's "availability zones" being separate buildings standing next to each other. Separate buildings protect you from a failed chiller. They do not protect you from a fire that jumps the gap.

What burned stayed burned. Facepunch, who ran Rust's EU game servers there, [confirmed](https://www.searchenginejournal.com/ovh-data-center-fire-darkens-thousands-of-sites-worldwide/398485/) "a total loss of the affected EU servers" and that "data will be unable to be restored". Customers whose backups sat on the same campus as the primary copy found out that a backup in the next building is really just a replica. OVH's own recovery effort was impressive: by late April it had [delivered 14,472 replacement bare-metal servers](https://corporate.ovhcloud.com/en/newsroom/news/informations-site-strasbourg/) and brought back 113,000 of the 120,000 affected services.

### The backup was fine. DNS wasn't.

The part of Strasbourg that interests me most is what happened to the customers who did everything right. They had backups in another data centre. OVH gave them a fresh server in Roubaix or Gravelines within hours, and they restored onto it. Then their sites stayed down for up to another two days anyway.

The problem was the new server's new IP address. Changing the A record takes thirty seconds. Getting the rest of the internet to use it takes much longer. Every resolver that has already looked up your domain keeps the old answer for as long as your published TTL tells it to, and you can't take that back. DNS has no way to tell a resolver in Warsaw that the answer you gave it yesterday is now wrong. A zone with a one-day TTL, which plenty of zones still have because nobody ever changed it, kept sending visitors to a burned-out IP address for a day. Then add the ISP resolvers that hold answers longer than they're told to, and the operating system and browser caches on top. "A day" becomes "a day and a bit, depending on who you ask".

It was worse if your DNS lived in Strasbourg too. If you ran your own nameservers on the machines that burned, which is not an unusual setup on a cheap dedicated box, the fix is to change the delegation at the registry. The registry's TTL is not yours to pick. Here is what the `.com` servers hand out today:

```bash
$ dig +norec +noall +authority +answer NS example.com @d.gtld-servers.net
example.com.		172800	IN	NS	hera.ns.cloudflare.com.
example.com.		172800	IN	NS	elliott.ns.cloudflare.com.
```

172,800 seconds is exactly 48 hours. A resolver that cached your old delegation can keep asking your dead nameservers for up to two days after you've fixed it. `.fr`, for comparison, hands out 3,600 seconds, which is one hour.

OVH clearly knew what a new IP address costs. When it rebuilt its last few thousand VPSes, it did so in SBG3 specifically ["so that customers can keep the same IP addresses"](https://corporate.ovhcloud.com/en/newsroom/news/informations-site-strasbourg/). Customers who moved elsewhere spent 48 hours waiting on caches, which is 0.55% of a year. A 99.9% budget allows 8 h 46 m of downtime per year, so that wait used up five and a half years' worth of it. And it all happened *after* the backup had been restored, because of a TTL nobody had looked at in years.

## Paris, April 2023: water

On 25 April 2023 a cooling pipe leaked in a part of a Global Switch data centre in Paris that Google didn't occupy. The water got into a UPS room and started a fire. The building was evacuated and powered down, and it took Google Cloud's europe-west9-a zone with it: [58% of its Compute Engine VMs and 55% of its persistent disks](https://status.cloud.google.com/incidents/dS9ps52MUnxQfyDGPfkY) went offline, along with part of europe-west9-c.

Losing a zone is exactly what zones are for. What makes Paris worth studying is Google's incident report, which explains how an event in one building turned into a regional outage and then a global one. The region was built as three buildings. Regional Spanner was supposed to keep one replica in each, so that losing any single building still left a quorum. But, in the report's words, "regional Spanner's replicas were not correctly distributed across the three buildings". Two of them were in the building that went dark. Spanner lost quorum at 23:04 that night, and everything in the region that depended on it went down too. Some of the control plane's fan-out calls didn't degrade gracefully when europe-west9's control plane disappeared, so Cloud Console pages and control-plane operations broke in *other* regions as well.

The zones were independent. The database holding the region together was not.

Recovery for the flooded zone took weeks. On 10 May, fifteen days in, Google was still saying there was ["no ETA for full recovery of affected instances in europe-west9-a"](https://www.theregister.com/2023/05/10/google_cloud_paris_outage_persists/). Fifteen days is 4.1% of a year, so whatever that zone's availability ended up being for 2023, it began with a single nine. Google's list of fixes included a water-intrusion risk review across every data centre hosting Google Cloud, and looking into drip trays for its space in multi-storey buildings.

## Bahrain, 2026: war

AWS has two regions in the Gulf, one per country: me-central-1 in the UAE and me-south-1 in Bahrain. Each has three Availability Zones. Both were hit.

It started on 1 March 2026, when AWS's status page reported that a zone in the UAE region had been ["impacted by objects that struck the data center, creating sparks and fire"](https://www.datacenterdynamics.com/en/news/aws-uae-outage-after-objects-struck-the-data-center-cause-fire-amid-iran-attacks/). The fire department cut power to the building and its generators. AWS took a while to say the word, but the objects were drones: ["In the UAE, two of our facilities were directly struck, while in Bahrain, a drone strike in close proximity to one of our facilities caused physical impacts to our infrastructure."](https://www.datacenterknowledge.com/outages/aws-middle-east-outage-after-data-center-hit-by-unidentified-objects) The damage included the buildings' structure, their power supply, and water from putting the fires out. AWS told customers in both regions to move their workloads elsewhere.

Here is what happened in each region:

| | UAE (me-central-1) | Bahrain (me-south-1) |
| --- | --- | --- |
| March | Drones hit two facilities directly. Two of the three zones are [badly impaired](https://www.infoq.com/news/2026/09/aws-middle-east-data-loss/). | A drone lands close to one facility. One zone is damaged. |
| April | | A second zone is damaged. The whole region goes offline. |
| July | | Iran [claims](https://www.theregister.com/off-prem/2026/07/21/iran-says-its-struck-offline-aws-facility-in-bahrain-again/5275762) another strike. It's hard to check, because the region has been offline for months. |
| 15 September | AWS writes off one zone, mec1-az2. Recovery continues in the other two. | AWS writes off the whole region. |

AWS's September statements use the same phrase for both regions. It is ["unable to restore access to the resources and data hosted exclusively"](https://www.helpnetsecurity.com/2026/09/17/aws-middle-east-outage-permanent-data-loss-bahrain-uae/) in mec1-az2 in the UAE, and in the entire region in Bahrain. For Bahrain it added why: the damage ["spanned multiple availability zones and exceeded what our regional and multi-AZ services are designed to withstand"](https://www.infoq.com/news/2026/09/aws-middle-east-data-loss/).

Put the two outcomes side by side and you get the clearest explanation of what multi-AZ does and doesn't do that a cloud provider has ever published. In the UAE, two zones were badly hit but only one is gone for good. What's lost is data that lived *only* in that zone: instances, disk volumes, anything tied to a single zone. Regional services like S3 keep copies in all three zones, so they can lose one, and AWS is not writing off regional data in the UAE. In Bahrain, the damage spread across two zones, one in March and one in April. That is more than the design covers. S3 in me-south-1 did what S3 always does and kept each object in at least three zones. It didn't help, because what happened wasn't a zone failure. It was a regional one. Eleven nines of durability describes disks, racks and buildings failing independently. It never covered a war.

For many of the customers who stayed, there's a nasty twist. Bahrain and the UAE both require some regulated data to stay in the country, and each country has exactly one AWS region. For a regulated Bahraini workload, the textbook fix of replicating to another region may have been the one option the law ruled out. Cross-region copies are the only thing that saved anyone here, and for some customers they were never an option. AWS waived March's bill in me-central-1, which was generous and beside the point.

## More lost nines

**Belgium, August 2015.** [Four successive lightning strikes](https://status.cloud.google.com/incident/compute/15056) on the electrical systems of Google's St. Ghislain data centre briefly cut power to the storage behind europe-west1-b's persistent disks. Most of it recovered. The rest, less than 0.000001% of the zone's allocated persistent-disk space, did not. As a fraction that is 10⁻⁸, or eight nines of durability for that zone that year. It still made headlines, because eight nines is not eleven.

**Daejeon, September 2025.** Technicians were relocating lithium-ion batteries at the data centre of South Korea's National Information Resources Service when the batteries caught fire. [Ninety-six systems were destroyed.](https://www.datacenterdynamics.com/en/news/858tb-of-government-data-may-be-lost-for-good-after-south-korea-data-center-fire/) One of them was G-Drive, the government's internal file store: 30 GB for each of roughly 125,000 officials, 858 TB in total, and no backup. An official told the Chosun Ilbo that it "couldn't have a backup system due to its large capacity". The other 95 systems did have backups. For scale, 858 TB fits on 48 LTO-9 cartridges. Five people have since been charged with professional negligence.

**us-east-1, October 2025.** Nothing burned this time. At 23:48 PDT on 19 October, a latent race condition between two instances of DynamoDB's DNS automation, one applying an old plan late and the other cleaning old plans up, left the regional endpoint `dynamodb.us-east-1.amazonaws.com` with [an empty DNS record](https://aws.amazon.com/message/101925/). EC2's instance management depends on DynamoDB, the load balancers' health checks got caught up in the aftermath, and the region didn't fully recover until 14:20 the next afternoon. The record itself was fixed at 02:25, and customers' DynamoDB connections came back over the next fifteen minutes, as their cached DNS answers expired. Even inside AWS, the last stretch of recovery was a wait for TTLs.

## The whole list

| When | Where | Trigger | Cost | The hidden correlation |
| --- | --- | --- | --- | --- |
| Aug 2015 | Google europe-west1-b | Four lightning strikes | < 0.000001% of persistent-disk space, permanently | One power path to the storage |
| Mar 2021 | OVHcloud SBG, Strasbourg | UPS fault, fire | Every byte with no off-site copy | "Zones" were buildings next to each other |
| Apr 2023 | Google europe-west9, Paris | Water leak, UPS-room fire | Weeks of zone downtime, a global control-plane wobble | Two of three Spanner replicas in one building |
| Sep 2025 | NIRS, Daejeon | Battery fire during relocation | 858 TB, no backup | There was only one copy |
| Oct 2025 | AWS us-east-1 | DNS automation race | About 15 hours of cascading failure | Everything resolved one DNS name |
| Mar–Apr 2026 | AWS me-south-1, me-central-1 | Drone strikes | All data held only in me-south-1 or mec1-az2 | One region per country |

## Where the nines go

The nines printed on the contract were never lies. They answered a narrower question than the one we thought we were asking. "What are the odds this disk fails?" has eleven nines in its answer. "What are the odds this building, this city or this country has a bad year?" has far fewer, and nobody prints that number on a pricing page. So here is what I take from the list above:

- **Nines only multiply when failures are independent.** Each incident here comes down to a correlation that wasn't on the architecture diagram: adjacent buildings, where the replicas were placed, a single region per country, one DNS name that everything depended on. When you add up your redundancy, look for what the redundant parts have in common.
- **Zones protect you from equipment failures. Regions protect you from events.** A failed chiller is a zone problem. A fire next door, a flood or a war is a regional one. If the data must survive the event, one copy has to live outside the region. When the region *is* the country, that copy has to be outside the country too.
- **Your recovery time has a floor, and the floor is your TTL.** Lower the TTL on anything you might need to move *before* you need to move it, because the new, shorter TTL only reaches everyone once the old one has expired. Keep authoritative DNS outside the failure domain you're protecting against, ideally with two providers. And know your registry's number: for `.com` it is 48 hours.
- **A backup you haven't restored is a hypothesis.** "Too big to back up" is a budget decision, not a law of nature. 858 TB is 48 tapes.
- **Check the residency rules against the region map.** If the law pins your data to a country with one region, your DR site is a second provider, a colo or a rack of your own. It is not another region of the same cloud.

Proust got his lost time back from the taste of a madeleine. Nothing brings lost nines back. All you can do is spend them where you intended to, rather than on a fire in the next building, a replica in the wrong room, or a TTL set years ago that nobody ever revisited.

---

*A note on how this was written: the prose of this article was created with the help of AI, working from the providers' own incident reports and status-page statements, the French BEA-RI investigation of the Strasbourg fire, contemporaneous press coverage, and my notes. Every date, count and quotation in it comes from the source linked beside it. The DNS TTLs were measured with `dig` against the `.com` and `.fr` registry servers on 21 September 2026. The only numbers that are mine rather than a provider's or an investigator's are the arithmetic that turns downtime and data loss into nines.*
