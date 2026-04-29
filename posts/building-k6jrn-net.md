---
title: Building k6jrn.net — A Self-Hosted Family Site on Starlink CGNAT
date: 2026-04-29
tags: homelab, self-hosting, cloudflare, docker
---

I've been running a homelab for a while now. Dell PowerEdge R720, OpenMediaVault, the whole stack. But I never had a public-facing presence — just internal dashboards and services that only made sense from inside the house.

Today that changed.

## The Problem: CGNAT

Starlink puts you behind CGNAT — Carrier-Grade NAT. You don't get a real public IP address, which means traditional port forwarding doesn't work. No dynamic DNS, no opening port 80, none of the usual tricks.

The solution is **Cloudflare Tunnel**. Instead of inbound connections, the server makes an outbound connection to Cloudflare's network. Traffic flows through that tunnel — no open ports, no public IP needed. It's free, handles HTTPS automatically, and works perfectly from Starlink.

## The Stack

The site runs as a Docker container on the R720 alongside everything else:

- **Flask** — lightweight Python web server
- **Docker + OpenMediaVault** — containerized on the homelab
- **Cloudflare Tunnel** — punches through CGNAT
- **GitHub** — blog posts as markdown files, fetched client-side

The compose file lives at `/data/compose/docker-compose.yml` with the rest of the homelab stack. The website container mounts two photo directories and reads passwords from a `.env` file.

## What's Live

**k6jrn.net** — The main family site. Hero, about, family photos, photography portfolio, projects, live printer feed, blog, and links. Built as a single-page app with smooth scroll navigation.

**movies.k6jrn.net** — A full movie library browser pulling live from Radarr. 657 movies with posters, ratings, overviews, trailer links, and a direct Play in Emby button. Search, filter by genre, sort by year or rating.

**emby.k6jrn.net** — Emby media server exposed through the tunnel. Works from anywhere — since traffic routes through the tunnel it looks local to Emby, so no transcoding penalties and no external restrictions.

**ryder.k6jrn.net** — Live feed from the Bambu Lab H2S print camera via go2rtc. Watch prints happen in real time.

## The Blog

Posts are markdown files in the `jeronthenet23` GitHub repo under `posts/`. The site fetches them via the GitHub API and renders them client-side using marked.js. No CMS, no build step — write a `.md` file, push it, it's live.

## Family First

The site isn't just about the homelab. It's about the Norrells — Jeremiah, Courtney, Wyatt (the Riot), and Weston (Messy Messy). There's a family photo gallery separate from the photography portfolio, each with their own upload password so Courtney can add photos directly from the browser.

The hero photo is a fall family portrait taken out here in the Georgetown foothills.

## What's Next

- TV show library from Sonarr
- 3D print gallery
- More posts when there's something worth writing about

No schedule. No algorithm. Just building things and documenting the journey.

— Jeremiah / K6JRN
