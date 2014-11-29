Cluster Provisioning
====================

---
layout: post
title: Cluster provisioning handbook
---

# Overview

- [Overview](#overview)
- [Key Concepts](#key-concepts)
- [Introduction](#Introduction)
- [Provisioning](#Provisioning)

## Introduction

This handbook walks you through provisioning a shard master capable of running suites for many boards, and a shard, capable of coordinating with the master to run suites for a single board. Any machine in the cluster has the same core configuration described by the cluster-core puppet module. This allows us to easily promote a shard to a master and viceversa. If you don't care about scaling, performance, or a distributed testing environment, you can just provision a master and fire suites at it using run_suite, suite_scheduler, test_that or the frontend.

## Key Concepts
![Architecture Diagram](https://cloud.githubusercontent.com/assets/3627706/5236006/ad969c72-77d1-11e4-9281-52b6e913fdb3.png?raw=true "Architecture overview")

To provision any machine for membership to an autotest cluster it must have the following:
* An autotest repo installation
* A mysql installation
* An apache installation

## Provisioning

```
fab -H <hostname> bootstrap
fab -H <hostname> deploy_puppet:server_type=autotest-cq
```
