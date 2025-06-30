---
title: "HomeLab Part 1: Self-Hosted Password Manager with Cloudflare, Raspberry Pi, Nginx Proxy Manager, and VaultWarden"
date: 2025-06-30
draft: false
summary: "Part 1 of my Home Lab series. My journey to homelabbing. In this blog, I will walk through setting up a self-hosted password manager using Vaultwarden on a Raspberry Pi, secured behind Nginx Proxy Manager and Cloudflare. This guide covers DNS setup, container deployment with Docker, HTTPS configuration using Let's Encrypt, and securing access to your Vaultwarden instance from anywhere. I have no idea what I'm doing, and since "
tags: ["Home Lab", "CloudFlare", "Raspberry Pi", "VaultWarden", "Nginx Proxy Manager"]
---

## 1. Materials

![MicroCenter](1.jpeg)

| Item                              | Price    |
|-----------------------------------|----------|
| Raspberry Pi 5 8 GB               | $79.99   |
| 128GB Micro SD Card               | $0.01    |
| Raspberry Pi Active Cooler        | $7.99    |
| Raspberry Pi 5 Case Black         | $9.99    |
| Raspberry Pi 45W USB-C Power Supply| $14.99  |
| Gourmet I Pocky Green Tea         | $2.99    |
| Tax                               | $6.96    |
| **Total**                         | **$122.92** |

Here are a list of all the things I bought. All of these besides the case is pretty much required to have. Assembly is pretty easy. If you've ever built a lego set in your life no instructions are needed. 

## 2. Getting Rasberry Pi