# MediaHub Infrastructure Summary

## Overview

This document describes the planned architecture for a personal self-hosted media ecosystem ("MediaHub"). The goal is to separate media serving, automation, tracking, and supporting services into logical components for easier maintenance and scalability.

---

# Objectives

* Self-hosted media streaming
* Automated media acquisition and organization
* Personal media tracking and analytics
* AI-powered recommendations (future)
* Easy backup and maintenance
* Native performance for services that interact heavily with storage

---

# Storage Layout

All media is stored under a single root directory.

```
/srv/media
├── Movies
├── TV
├── Music
├── Audiobooks
├── Podcasts
├── Books
├── Manga
├── Comics
├── Downloads
│   ├── Complete
│   └── Incomplete
└── Transcode
```

---

# Native (Host) Services

These are installed directly on the host and managed using systemd.

## Jellyfin

Purpose:

* Primary movie and TV streaming server.

Reasons for native installation:

* Direct GPU access
* Better hardware transcoding
* Simpler filesystem permissions
* Easier integration with VAAPI/QSV/NVIDIA
* Direct access to `/srv/media`

---

## qBittorrent

Purpose:

* Download client.

Responsibilities:

* Download media
* Maintain completed download directory
* Interface with automation services

---

## Sonarr

Purpose:

* TV show management.

Responsibilities:

* Monitor series
* Import completed downloads
* Rename files
* Organize TV library

---

## Radarr

Purpose:

* Movie management.

Responsibilities:

* Import movies
* Organize movie library
* Rename media

---

## Lidarr

Purpose:

* Music management.

---

## Readarr

Purpose:

* Book management.

---

## Bazarr

Purpose:

* Subtitle management.

---

## Prowlarr

Purpose:

* Centralized indexer management.

Responsibilities:

* Manage indexers
* Provide indexers to Sonarr/Radarr/etc.

---

# Docker Services

Docker is used only for supporting applications.

---

## PostgreSQL

Purpose:

* Shared database server.

Separate databases should be created for each application.

Example:

* yamtrack
* jellystat
* future applications

Do **not** share a single database between applications.

---

## Redis

Purpose:

* Cache
* Message broker
* Background job queue

Used primarily by Yamtrack.

---

## Yamtrack

Purpose:

* Personal media tracker.

Tracks:

* Movies
* TV Shows
* Anime
* Games
* Books
* Manga
* Podcasts

Future use:

* Recommendation engine data source
* Viewing history
* Ratings
* Personal statistics

Depends on:

* PostgreSQL
* Redis

---

## Navidrome

Purpose:

* Dedicated music streaming server.

Media source:

```
/srv/media/Music
```

Reason:

Provides a better music experience than Jellyfin.

---

## Audiobookshelf

Purpose:

* Audiobooks
* Podcasts

Media:

```
/srv/media/Audiobooks
/srv/media/Podcasts
```

Features:

* Progress tracking
* Chapter support
* Speed control
* Mobile clients

---

## Kavita

Purpose:

* Digital library.

Media:

```
/srv/media/Books
/srv/media/Manga
/srv/media/Comics
```

Supports:

* EPUB
* PDF
* Manga
* Comics

---

## Jellystat

Purpose:

* Jellyfin analytics.

Uses:

* Jellyfin API
* PostgreSQL

Provides:

* Watch statistics
* Viewing history
* Library usage
* Playback analytics

---

# Future Services

## Open WebUI

Purpose:

Web interface for local AI models.

---

## Ollama

Purpose:

Run local LLMs.

Potential future uses:

* Semantic media search
* Recommendation assistant
* Natural language querying
* Metadata enrichment

---

# Planned Recommendation Engine

Future machine learning project.

Possible inputs:

* Yamtrack ratings
* Yamtrack watch history
* Jellystat viewing history
* Jellyfin metadata
* Genres
* Actors
* Directors
* Runtime
* Release year
* Tags
* Embeddings generated from descriptions

Potential models:

* Collaborative filtering
* Content-based filtering
* Hybrid recommendation system

---

# Overall Architecture

```
                    Internet
                        │
                   Prowlarr
                        │
      ┌─────────────────┴─────────────────┐
      │                                   │
   Sonarr                            Radarr
      │                                   │
   Readarr                           Lidarr
      │                                   │
                qBittorrent
                     │
             Completed Downloads
                     │
             /srv/media (host)
                     │
 ┌───────────────────┼────────────────────┐
 │                   │                    │
Jellyfin        Navidrome          Audiobookshelf
 │
 │
Jellystat
 │
Yamtrack
 │
Recommendation Engine (future)
 │
Open WebUI + Ollama (future)
```

---

# Deployment Philosophy

Native services:

* High filesystem interaction
* GPU access
* Media automation
* Media serving

Docker services:

* Supporting applications
* Databases
* Analytics
* Tracking
* AI services

This separation minimizes filesystem complexity, avoids Docker permission issues, simplifies hardware acceleration, and keeps infrastructure modular and maintainable.