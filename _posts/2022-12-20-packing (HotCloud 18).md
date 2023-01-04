---
layout:     post
title:      packing (HotCloud 18) 论文阅读
date:       2022-12-20
tags:
    - Cloud
categories: paperreading
comments: true
---

[A Case for Packing and Indexing in Cloud File Systems](https://www.usenix.org/conference/hotcloud18/presentation/kadekodi)

这篇文章的 packing 思想乍一看比较简单直接，但整篇文章给了我极大启发，包括但不限于：S3+packing 比其他现成的服务（e.g. EFS, DynamoDB）好在哪里，实现 packing+indexing 要考虑的方方面面，等等。

而且文笔真好啊真好啊

## Abstract

object storage 按 operation 数量付钱，所以 pack small objects 可以省钱。

We propose client-side packing of small immutable files into gigabyte-sized blobs with embedded indices to identify each file’s location.

## Introduction and motivation

- **Most files are small (KiB-sized) and that file creation occurs in bursts.** "Yet, small object creation and write bursts are improper fits for cloud storage, for which both the performance behaviors and the cost models are heavily tilted in fa- vor of using large access units and large objects. In par- ticular, with direct file-to-object mapping, per-operation costs and operation rate throttling dominate other consid- erations during bursts of small file creation."
- Cloud object storage systems like S3 are not intended to play the role of traditional file servers: there is a **significant per-operation charge**, regardless of operation size, and users are explicitly rate limited in terms of operations/s.
- "Our module packs data into GiB-sized blobs with an index of where each packed file is located in the blob. It also maintains a redundant client-side index, so that most reads can directly GET particular file data; the embedded indices are used to reconstruct this index if the client crashes rather than shutting down cleanly."

{% include figure.html path="assets/img/fig/Packing-fig1.png" class="img-fluid rounded z-depth-1" zoomable=true %}

这个图也涉及到我很想探究的一个方面：object storage (e.g. S3) performance modeling。

## Design

### Packing structures

- Blob: A packed blob is a single immutable **self-describing** object that contains several small files. A packed blob has two parts: the blob body and the blob footer. The blob body contains **concatenated byte ranges** of packed files. We can customize the packing policy used to choose the files to pack in a particular blob to suite the read pattern. The blob footer contains a list of blob extents. A blob extent maps the logical byte-range of a file that was packed to the physical byte-range in the blob body. We call this footer **the embedded index** of the blob.
- Blob Descriptor Table (BDT): While an embedded index is capable of mapping the files it contains, the process of bootstrapping with only embedded indices could entail pre-reading the footers of all blobs, potentially a very expensive task. Instead we maintain a redundant global index that stores all mappings of all blobs. We call this the blob descriptor table (BDT). BDT consistency 是个需要谨慎的问题，因为 blob creation is decentralized -> global BDT

## Operation

The master node maintains a BDT which we implemented using LevelDB for bounded memory consumption.

- Packing and pushing: Sufficient data has to accumulate to ensure sizeable (GiB-sized) blobs. Thus, buffered data waiting to be packed cannot be considered durable until it is uploaded to the cloud as a part of one or more blobs. Workers choose from buffered data using a particular packing policy (specified at mount time) to build a blob. The name of the blob is carefully selected as a combination of the following attributes:
  - Worker IP: to indicate the ownership of the blob to the other workers.
  - Footer byte offset: the embedded index byte offset in the blob. This is an important fault tolerance requirement.
  - Timestamp: the creation timestamp of the blob disambiguating it from other blobs created from the same worker having the same footer offset.
- Reads: first local, then global BDT; We then leverage the range-read feature of cloud storage systems to fetch the required bytes from the packed blob.
- Deletes and Renames: Packing (essentially buffering) complicates the delete and rename semantics. 
  - When the client issues a delete of a small file, it is **unreachable** once we remove its mapping from the global BDT. The remaining task is to **reclaim the space** occupied by the deleted small file in the cloud, which we can do so, by tracking the **utilization** of a blob in the global BDT.
  - Handling renames would involve an atomic change-of-name in the global BDT along with piggybacking renames as a part of future blobs’ embedded indices.
- Fault tolerance: whatever is pushed to the cloud can be recovered. 注意，这里假设了只有 local state 才会真的丢失，写在 local disk 的都可以通过重启 worker 之类的恢复。

### Discussion: meeting design goals

- Packing as an Alluxio **Under File System (UFS)**: transparency and modularity. The packing UFS is mounted to an Alluxio file system path, with various configurations such as the maximum blob size, data buffering timeout, number of packing threads, the packing policy and the underlying UFS that will store the data. A mount-point based access to packing allows it to co-exist with other pack- ing solutions (on different mount points, with potentially different packing policies) and with non-packing Alluxio solutions.
- Worker-local BDT: potential optimization which can reduce communication and improve scalability
- Exploiting range-reads
- Packing open files or large files
- Handling immutable files

## Packing + indexing in action

略

## Related Work

难得还挺有用的 related work