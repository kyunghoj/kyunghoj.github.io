---
title: What I read today, 2018-07-08
categories:
  - Blog
tags: [what-i-read]
---

### [Borg, Omega, and Kubernetes](https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/44843.pdf)

> and containers need to be supported by an additional security layer (such as virtual machines) to protect against the kinds of malicious actors found in the cloud. 

Kubernetes 설치 가이드 중 많은 것들이 OpenStack 기반 VM에서 설치하게끔 되어 있는 듯 한데, 이것 때문인가?

> Over time it became clear that the benefits of containerization go beyond merely enabling higher
levels of utilization. Containerization transforms the data center from being machine-oriented to being application-oriented. 

Data Center를 위한 OS라는 표현이 과하지 않다. 어찌보면 Distributed Operating Systems 에서 하려던 것들의 많은 부분을 이루었는지도.

### [The Elements of Kubernetes - Foundational Concepts for Apps Running on Kubernetes](https://www.youtube.com/watch?v=S9l2MWhIBhc)

[Slides](https://schd.ws/hosted_files/kccncna17/9f/The%20Elements%20of%20Kubernetes.pdf)
* "The Elements of Style" 이라는 책처럼, Kubernetes 에서 Application을 운영하기 위한 Best Practices 들을 정리
  - Pod to Pod messaging via `Service`s
  - Crash if you can’t connect (crash-only)
  - Look for Kubernetes resources via labels, not names
  - sig-service-catalog 에 속해있다고 하는데, Cloud 에 DB를 생성해서, auth 정보를 secret으로 만들어 app 에 binding 하는 것이라고 말함. 
  - service catalog 등 용어/개념 정리 필요 
