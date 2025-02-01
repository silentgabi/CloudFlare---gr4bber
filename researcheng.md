**Hi, I'm g4bbdev.**  
I'm 13 years old, and  

Three months ago, I discovered a "0-click" deanonymization attack that allows an attacker to obtain the location of any target within a 400 km radius. If the victim has a vulnerable app installed on their phone (or a background-running program on their laptop), an attacker can send a malicious payload and track their location within seconds—without them noticing.  

I'm publishing this research as a warning, especially for journalists, activists, and hackers, about this type of undetectable attack. Hundreds of applications are vulnerable, including some of the world's most popular ones: Signal, Discord, Twitter/X, and others. Here's how it works:  

## Cloudflare  

In terms of numbers, Cloudflare is easily the most popular CDN on the market, surpassing competitors like Sucuri, Amazon CloudFront, Akamai, and Fastly. In 2019, a major Cloudflare failure took down a large portion of the internet for more than 30 minutes.  

One of Cloudflare's most widely used features is **Caching**. Cloudflare stores copies of frequently accessed content (such as images, videos, or web pages) in its data centers, reducing server load and improving website performance ([official documentation](https://developers.cloudflare.com/cache/)).  

When your device requests a cacheable resource, Cloudflare first tries to fetch it from the nearest data center. If it's not available, the resource is retrieved from the origin server, stored locally, and delivered to the user. By default, [certain file extensions](https://developers.cloudflare.com/cache/concepts/default-cache-behavior/) are automatically cached, but website operators can define new caching rules.  

Cloudflare has a massive global presence, with data centers in over **330 cities and 120 countries**—273% more than Google. In the U.S., for example, the nearest data center to me is less than 160 km away. If you live in a developed country, there's likely a Cloudflare data center within **320 km** of you.  

Then I had a "eureka" moment: **if Cloudflare caches data so close to users, could this be exploited for deanonymization attacks on sites we don't control?**  

The answer lies in Cloudflare's HTTP headers:  
![cf-cache-status](https://gist.github.com/user-attachments/assets/95e1a39a-ed25-4531-9c57-a1b43c616519)  

- `cf-cache-status` can indicate HIT/MISS (whether a file was delivered from the cache).  
- `cf-ray` contains the **nearest airport code** for the data center that processed the request.  

If we can make the victim's device load a resource hosted on a Cloudflare-backed site, we can then **map all Cloudflare data centers** to determine which one cached that resource. This gives us an extremely precise estimate of the victim's location.  

## Cloudflare Teleport  

There was one major obstacle before I could test this theory:  

**You can't send HTTP requests directly to specific Cloudflare data centers.** The entire network operates with **anycast**, meaning any TCP connection is always routed to the nearest available data center.  

However, after some research, I found [a post on Cloudflare's forum](https://community.cloudflare.com/t/how-to-run-workers-on-specific-datacenter-colos/385851) explaining a **bug** that bypassed this limitation using Cloudflare Workers.  

Basically, I discovered that by using an internal IP range from **Cloudflare WARP (Cloudflare's VPN)**, it was possible to **force** certain requests to be handled by a specific data center.  

Based on this, I developed **Cloudflare Teleport** ([GitHub](https://github.com/hackermondev/cf-teleport)), a Cloudflare Workers-based proxy that allows HTTP requests to be redirected to specific data centers. For example:  

🔹 `https://cfteleport.xyz/?proxy=https://cloudflare.com/cdn-cgi/trace&colo=SEA`  

This directs the request specifically to the **Seattle (SEA) data center**.  

Months later, Cloudflare patched this bug, making the tool obsolete, but until then, **it worked perfectly for my tests**.  

---  

## First Deanonymization Attack  

With **Cloudflare Teleport**, I could test my theory.  

I created a small CLI program that made an HTTP request to a specific URL and listed **all the data centers that had cached the resource**.  

For my first test, I chose the favicon of **Namecheap**:  

🔗 `https://www.namecheap.com/favicon.ico`  

This was a simple, static resource cached by Cloudflare.  

![cache hit](https://gist.github.com/user-attachments/assets/8da57801-ae8e-4adf-9a2e-ec6feec6086f)  

💥 It worked! I could see **all the data centers** that had cached Namecheap's favicon in the last 5 minutes.  

This proved that it was possible **to use Cloudflare's cache to track users and estimate their locations**.  

---  

## Real-World Application: Signal  

**Signal**, one of the world's most secure messaging apps, **was vulnerable** to this attack.  

### 1-Click Attack  

When a user sends an **attachment** (e.g., an image) in Signal, the file is uploaded to Cloudflare's CDN (`cdn2.signal.org`).  

As soon as the recipient **opens the chat**, their device automatically downloads the attachment, and Cloudflare caches it.  

🔹 **Solution:** I blocked the HTTP request to the CDN and sent a test attachment. Then, I used my program to map the data centers where the file was cached.  

📍 The result? I discovered that **my nearest data center was in Newark, NJ (EWR)**, **240 km from my real location**.  

---  

## 0-Click Attack  

Now, **how do you execute this attack without the victim opening the chat?**  

The secret: **push notifications**.  

📌 In Signal, when a user receives a message with an attachment, the app **automatically downloads the image** to display it in the push notification.  

This means **no user interaction is required**. We just need to send an image to the victim and wait for the notification to be delivered!  

💥 Result: **the victim is tracked without even opening the app!**  

---  

## Real-World Application: Discord  

**Discord** is also vulnerable to the same attack. But instead of attachments, we can exploit **profile avatars**.  

🔹 When someone receives a **friend request**, the sender's avatar is automatically loaded **to be displayed in the push notification**.  

If we change our avatar before the attack, we ensure that **no one else has loaded that file**, making detection precise.  

I created a bot called **GeoGuesser**, which automates the entire process and executes the attack in **seconds**.  

---  

## Conclusion  

- **Signal and Discord were alerted but did not fix the issue.**  
- **Cloudflare patched the flaw that allowed access to specific data centers, but the attack is still possible through other means.**  

This attack demonstrates that even apps that prioritize **privacy** can **leak sensitive information without the user noticing**. 🚨  
