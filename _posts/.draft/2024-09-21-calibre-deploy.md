---
layout: post
title: Deploying Calibre-Web on Kubernetes Cluster
date: 2024-09-19 00:01:00
description:
tags: calibre kubernetes web
categories: IT
---

Calibre is a well-known e-book manager. Calibre-web project provides a webui for browsing and reading books in a Calibre database.

## Deploy with Docker Compose

## Deploy with Kubernetes

## Troubleshooting

### DB settings is invlaid on new instance

You need to copy an empty `metadata.db` and put it to a folder such as `/books`. Note that Calibre-web application must have write permission.

### File size may be too big

There are two possible reasons：

- Calibre-web is deployed behind a reverse proxy, which limit the max upload size.
- you are using proxy applications, such as surge/clash, and your v2ray/ss subscription prevents you from uploading large files.
