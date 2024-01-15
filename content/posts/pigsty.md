---
title: Pigsty
date: 2023-06-15 14:17:43
tags: 
- "PostgreSQL"
---

[Pigsty Code](https://github.com/Vonng/pigsty/)


Pigsty 是一个更好的本地开源 RDS for PostgreSQL 替代：

- 开箱即用的 [PostgreSQL](https://www.postgresql.org/) 发行版，深度整合地理时序分布式向量等核心扩展： [PostGIS](https://postgis.net/), [TimescaleDB](https://www.timescale.com/), [Citus](https://www.citusdata.com/)，[PGVector](https://github.com/pgvector/pgvector)……
- 基于现代的 [Prometheus](https://prometheus.io/) 与 [Grafana](https://grafana.com/) 技术栈，提供令人惊艳，无可比拟的数据库观测能力：[演示站点](http://demo.pigsty.cc)
- 基于 [patroni](https://patroni.readthedocs.io/en/latest/), [haproxy](http://www.haproxy.org/), 与[etcd](https://etcd.io/)，打造故障自愈的高可用架构：硬件故障自动切换，流量无缝衔接。
- 基于 [pgBackRest](https://pgbackrest.org/) 与可选的 [MinIO](https://min.io/) 集群提供开箱即用的 PITR 时间点恢复，为软件缺陷与人为删库兜底。
- 基于 [Ansible](https://www.ansible.com/) 提供声明式的 API 对复杂度进行抽象，以 **Database-as-Code** 的方式极大简化了日常运维管理操作。
- Pigsty用途广泛，可用作完整应用运行时，开发演示数据/可视化应用，大量使用 PG 的软件可用 [Docker](https://www.docker.com/) 模板一键拉起。
- 提供基于 [Vagrant](https://www.vagrantup.com/) 的本地开发测试沙箱环境，与基于 [Terraform](https://www.terraform.io/) 的云端自动部署方案，开发测试生产保持环境一致。



在用多了MySQL以后，发现MySQL有很多不如意的地方，索引类型比较单一，在数据查询方面能力表现的比较弱，在一些稀疏字段以及其他适合倒排索引的场景，几乎实现不了

pg提供的扩展能力可以很好的集成各类数据产品，优秀的外表设计以及向量查询的支持，不管在OLAP还是OLTP都比MySQL做的优秀太多

奈何大环境对pg的支持很不友好：(