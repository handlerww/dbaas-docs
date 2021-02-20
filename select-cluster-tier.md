---
title: Select Your Cluster Tier
summary: Learn how to select your cluster tier on TiDB Cloud.
---

# Select Your Cluster Tier

This document describes how to select your cluster tier on TiDB Cloud.

## Cluster Tier overview

When creating a cluster, you need to select a cluster tier, which determines the throughput and performance of your cluster. Different cluster tiers provide different TiDB and TiKV node instance types. Different tiers are provided for AWS and GCP.

> **Note:**
>
> You cannot change the selected tier after your cluster is created.

## AWS Tiers

| Cluster tier | Node | CPU | Storage | Scenario |
| :-- | :-- | :-- | :-- | :-- |
| t1.tiny | TiKV | 2 vCPU | 50 G EBS | Testing or development |
| t1.tiny | TiDB | 2 vCPU | / | Testing or development |
| t1.standard | TiKV | 8 vCPU | 1900 G NVMe | Production |
| t1.standard | TiDB | 8 vCPU | / | Production |
| h1.standard | TiKV | 8 vCPU | 1900 G NVMe | Production |
| h1.standard | TiFlash | 8 vCPU | 1900 G NVMe | Production |
| h1.standard | TiDB | 8 vCPU | / | Production |

## GCP Tiers

| Cluster tier | Node | CPU | Storage | Scenario |
| :-- | :-- | :-- | :-- | :-- |
| t1.tiny | TiKV | 1 vCPU | 50 G | Testing or development|
| t1.tiny | TiDB | 1 vCPU | / | Testing or development|
| t1.standard | TiKV | 8 vCPU | 1875 G | Production |
| t1.standard | TiDB | 8 vCPU | / | Production |
| h1.standard | TiKV | 8 vCPU | 1875 G | Production |
| h1.standard | TiFlash | 8 vCPU | 1875 G | Production |
| h1.standard | TiDB | 8 vCPU | / | Production |
