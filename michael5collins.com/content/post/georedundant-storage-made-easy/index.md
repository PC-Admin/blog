---
title: "Ceph's Stretch Mode: Georedundant Storage Made Easy!"
description: "Georedundant storage made easy, with Ceph's little known 'Stretch Mode' feature."
summary: "We explore Ceph's powerful 'Stretch Mode' feature, which simplifies georedundant storage setup. Learn how Stretch Mode enables efficient data replication across failure domains, ensuring data redundancy and integrity for your storage infrastructure."
date: '2023-11-28'
aliases:
- ceph-stretch-mode-georedundant-storage
author: 'Michael Collins'
usePageBundles: true

featureImage: 'ceph-cover.png'
# featureImageAlt: 'Description of image' # Alternative text for featured image.
# featureImageCap: 'This is the featured image.' # Caption (optional).
thumbnail: 'ceph-cover.png' # Image in lists of posts.
shareImage: 'ceph-cover.png' # For SEO and social media snippets.

categories:
- distributed storage
tags:
- ceph
- georedundancy
- distributed storage
---


## Geo-Redundant Storage Made Easy

---

Georedundant storage is hard, companies pay a lot of money for serious guarantees that their data will not only be replicated across multiple locations but that it'll be available to them when they need it. This is a problem that's difficult to solve, and it's why companies like AWS charge so much for their multi-region storage solutions.

Ceph is an open source distributed storage system that is designed to be highly available and fault-tolerant. It spreads data across multiple 'nodes' (hosts) and disks, so that if one node or disk fails, that data is still available. Configuring Ceph to make data redundant across one site can be a challenge, but this is made even more difficult when you want to replicate data across multiple sites while ensuring reliability. A multi-site Ceph setup requires running multiple Ceph clusters in an active-active or active-passive configuration, it is significantly more complex than a single-site setup.

This is where Ceph's "stretch mode" feature comes in. If you only want to distribute data across 2 sites and you want to avoid the complexity of a multi-site setup, stretch mode is the happy middle ground you're looking for.


## Limits of a Ceph Cluster

---

CRUSH picks OSDs at random to write data to, it doesn't guarantee that data will be replicated across multiple sites. This is a problem that is difficult to solve with standard CRUSH rules, as the documentation states:

https://docs.ceph.com/en/latest/rados/operations/stretch-mode/#stretch-cluster-issues

_"For example, in a scenario in which there are two data centers named Data Center A and Data Center B, and the CRUSH rule targets three replicas and places a replica in each data center with a min_size of 2, the PG might go active with two replicas in Data Center A and zero replicas in Data Center B. In a situation of this kind, the loss of Data Center A means that the data is lost and Ceph will not be able to operate on it. This situation is surprisingly difficult to avoid using only standard CRUSH rules."_


## How Stretch Mode Works

---

The standard configuration for stretch mode is 2 datacenters with a roughly equal amount of OSDs in each. Each site holds 2 copies of the data with a total replication size of 4. New writes are written once to both sites before they are acknowledged by Ceph. A third site is used as a tiebreaker, which doesn't contain any OSDs. In the event that the network between the 2 sites is cut, the tiebreaker site will be used to determine which site is the primary site. When the network between the 2 sites is restored, the secondary site will be brought back online and the data will be resynced.

In the event that one datacenter is lost, the cluster will enter a special "degraded stretch mode" described here:

https://docs.ceph.com/en/latest/rados/operations/stretch-mode/#entering-stretch-mode

_"If all OSDs and monitors in one of the data centers become inaccessible at once, the surviving data center enters a “degraded stretch mode”. A warning will be issued, the min_size will be reduced to 1, and the cluster will be allowed to go active with the data in the single remaining site. The pool size does not change, so warnings will be generated that report that the pools are too small -- but a special stretch mode flag will prevent the OSDs from creating extra copies in the remaining data center. This means that the data center will keep only two copies, just as before."_

![Here we see a cluster in "degraded stretch mode" that's lost one of its datacenters. It keeps serving data and allowing writes.](degraded-stretch-mode.png)


## The Benefits of Stretch Mode

---

- Cross-site data redundancy and high availability.
- It's easier to set up than a multi-site configuration; you don't need to run multiple Ceph clusters or configure zones, realms and endpoints.
- It's cheaper than a multi-site setup, as a multi-site setup requires running a duplicate amount of hardware in both locations to host 3 copies in each. While a stretch mode cluster only keeps 2 copies in each location.


## The Limitations of Stretch Mode

---

- It's only suitable for 2 sites, if you want to replicate data across more than 2 sites you'll need to create a [multi-site setup](https://docs.ceph.com/en/quincy/radosgw/multisite/).
- It's only suitable if your OSD devices are SSDs, as the latency of HDDs is too high to make this work.
- Erasure coded pools cannot be used or created while using stretch mode.
- Writes are slow, this is because Ceph needs to write an object to both sites before it can acknowledge the write.

To find out more about the limitations of stretch mode see the officical documentation here:

https://docs.ceph.com/en/latest/rados/operations/stretch-mode/#limitations-of-stretch-mode


## Install Guide for Ceph's Stretch Mode

---

I've published a guide for creating a small testing cluster with stretch mode enabled, it can be used to help you examine if a stretch mode is appropriate for your own use case:

https://github.com/PC-Admin/cephfs-stress-test/blob/main/stretch_mode_setup.md

![This is a preview of my own 2-site stretch mode cluster.](ceph-stretch-mode.gif)


## Testing Site Failure

---

I also tested an abrupt site failure on this cluster, I did this by forcibly shutting off the VMs in datacenter a1. The filesystem became unresponsive for 33 seconds, and then it was back to normal.

I then tested a sudden isolation of the network between the 2 sites, where both could talk to the tiebreaker node but not to each other. I did this by quickly erecting firewall rules on the hosts in a1 and severing any existing connections. The filesystem became unresponsive for 47 seconds before returning to normal.

I've published the steps and results of these tests here:

https://github.com/PC-Admin/cephfs-stress-test/blob/main/stretch_mode_setup_tests.md

Stretch mode seems to be very resilient and allows clients to keep accessing their data pretty quickly in the event of a site failure.


## Conclusion

---

Stretch mode is a great feature that allows you to replicate data across multiple sites without having to set up a full-blown multi-site cluster with multiple clusters and a lot of added complexity. It's cheaper than a multi-site setup, as a multi-site setup requires running a duplicate amount of hardware in both locations to host 3 copies in each. While a stretch mode cluster only keeps 2 copies in each location.

If you need assistance with your Ceph setup, or your interested in upgrading to a stretch mode cluster, please contact me for a free consultation.
