---
title: "What is DNS and How It Works"
layout: post
date: 2025-10-10
categories: ["Core Concepts"]
tags: ["DNS", "Networking", "Internet Basics"]
description: "A simple explanation of what DNS is, how it works, and why it's one of the key building blocks of the internet."
image: assets/img/internet/view-internet.jpeg
---

At first, it may sound like a simple question, but asking **“What is DNS?”** is actually quite important for understanding how the internet works.  

**DNS (Domain Name System)** is the system that translates domain names (for example, *erhankudretli.github.io*) into IP addresses (such as *185.199.109.153* or *2001:db8::1*) that computers can understand. In other words, it acts like the internet’s **address book**.  
We, as humans, remember website names, while computers communicate using IP addresses. DNS builds the bridge between the two.  

---

### How the DNS process works

When we try to visit a website, the process works like this:  

1. **Cache check:** When you type a domain name into your browser, it first checks the cache of your browser and operating system. If the IP address has been resolved before, it’s used directly from there.  

2. **Hosts file:** If it’s not in the cache, the operating system checks the *hosts* file — a small “manual address book” where users can define custom mappings.  

3. **DNS resolver:** If there’s still no result, your computer asks a **DNS resolver**, usually provided by your Internet Service Provider or a public service like Google DNS or Cloudflare DNS.  

4. **DNS hierarchy:** The resolver may then follow a hierarchical process:  
   - Contacting the **root DNS servers**  
   - Then the **top-level domain (TLD)** servers (like *.com*, *.org*, *.net*)  
   - And finally the **authoritative DNS server** that holds the actual record.  

5. **Response:** The correct IP address is found and returned to your computer.

---

### 🗺️ DNS Resolution Diagram

To better visualize this process, here’s a simple diagram showing how a DNS query travels through each step:

![DNS Resolution Diagram](/assets/images/internet/dns-process.png){: .center-image }

*(Image: DNS query flow — from browser cache to root, TLD, and authoritative servers.)*

---

### What happens next

DNS itself only returns the IP address.  
After that, your browser connects to the server using that address, establishes a **TCP** connection (and **TLS** if it’s HTTPS), and sends an **HTTP** request.  

At that point, the website’s content — HTML, CSS, images, and videos — starts loading.  

---

### Summary

DNS is one of the **key building blocks** of the internet.  
It turns names into numbers, making it easy for people to access websites without remembering complex IP addresses.

---

*Category:* [Core Concepts](/core-concepts)  
*Tags:* DNS, Networking, Internet Basics
