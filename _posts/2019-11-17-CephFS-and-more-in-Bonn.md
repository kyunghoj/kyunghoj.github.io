---
layout: post
title: "CephFS and more in Bonn"
categories: [tech-talks]
tags: [storage, filesystem, ceph, cephfs]
---

논문은 잘 읽지 못하고, 대신 Technical Talk을 동영상으로 보거나, 슬라이드를 읽는 경우가 많다. 간단히 정리하는 것이 기억하는 데에 도움이 될 것 같아 포스트로 남긴다. 

## 개요

* [Video](https://cds.cern.ch/record/2691580)
* [Slides](https://indico.cern.ch/event/765214/contributions/3517132/attachments/1907986/3151718/15-Oliver-Freyermuth-Bonn.pdf)
* 독일 Bonn 대학의 Physics Institute 에서의 Ceph(FS) 사용하는 것을 소개.
* Ceph 가 제공하는 주요 기능 세 가지를 모두 사용하고 있음
  - CephFS: High Throughput Computing (HTC) cluster 용. 
    - Erasure Coding (k = 4, m =2)
    - Snappy compression 사용
  - RBD: 가상화 서버 클러스터에서 사용됨. 3개의 건물에 3 replica 구성
  - RGW: Backup storage 로 테스트 중

## CephFS

* Lustre 를 사용하고 있었음 (10 Gb/s ethernet)
* InfiniBand 56 Gb/s 로 업그레이드
* 원래 계획은 Lustre / BeeGFS 였으나 
  - 무료 라이센스는 ACL이나 Quota 를 지원하지 않음
  - Code contribution 이 어려운 문제도 있음
    - (Lustre 나 BeeGFS 는 오픈소스이긴 하지만 한 회사가 주도하여 개발하는 형태)
* 2018년 1분기에 CephFS 로 계획 변경
* Metadata on SSD / NVMe devices, Data on HDD
  - 3 MON + MDS + OSD nodes: 2개의 SSD/NVMe
  - 7 OSD nodes: 대략 30개의 HDD, 2 개의 SSD/NVMe
* NFS Ganesha (NFS v4.2)를 통해 desktop machine 에서 액세스 할 수 있음
* IPoIB 사용 (RDMA에서는 문제가 있었음): tuning 후 성능 좋음
* Erasure Coding (k=4, m=2), Snappy compression 사용
  - [Red Hat Ceph Storage 3.3 Bluestore compression performance](https://www.redhat.com/en/blog/red-hat-ceph-storage-33-bluestore-compression-performance)
* CentOS 7.6
* 각 사용자에 500 GB, 파일 갯수 100,000개 제한
* Use case 는 큰 파일, write-once, read-many
* 성능: effective sequential read throughput 3GB/s 이상

## Ceph RBD and Backup

* `libvirt`

## Ceph-based Backup

* 사용자의 데스크탑 백업 서비스 제공
* Linux에서는 [Restic](https://restic.net/), Linux, Windows, Mac 에서는 [Duplicati](https://www.duplicati.com/) 사용


