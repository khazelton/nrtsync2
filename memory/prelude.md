---
title: Prelude to Near-real-time Sync Study
date: 2026-07-09 09:18
draft: 0.1.0
author: Keith Hazelton, Claude Fable 5
---


Background and motivation: Why does some data need to be passed in near-real time from Systems of Record/ERPs to the IAM/IGA systems?

Timely and accurate control over who can do what is a core requirement of IAM and IGA. So when a right to access something depends on detailed and up-to-date data about the user, if that information is out of date, the user may mistakenly be provided access or mistakenly denied access to a resource or service.

This leads to a need to minimize the time between a change in access-related information in a System of Record and the update of information available to the IAM system–and other provisioned systems.

There seem to be as many ways to approach the requirement for near-real-time sync as there are systems involved. However, all these systems can be sorted into one or another of a set of categories based on their architecture.

