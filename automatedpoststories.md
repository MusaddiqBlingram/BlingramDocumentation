# Distributed Social Automation Architecture Review

## 1. The Gateway (OAuth 2.0 & Vault)

We never store passwords. We use OAuth 2.0.

### Process

User clicks "Connect IG." They authorize Bellokey on Meta's site. Meta sends back an access_token and refresh_token.

### Storage

Securely store these in the database.

### Maintenance

A background task must "refresh" these tokens before they expire, or the automation dies.

---

## 2. The Universal Container (Database Schema)

Don't build separate tables for "IG Post" and "Reddit Post." Build one Universal Content Model.

### The Model

- PostID  
- Content (Text/Media URLs)  
- Type (Post, Story, or Ad)  
- ScheduleTime  
- PlatformTargets (JSON)

### Ads Nuance

For Ads, the JSON must include:

- budget  
- audience_id  
- objective_id  

This is a different API endpoint than organic posts.

---

## 3. The Watchman (Database + Celery Beat)

We don't clog memory with posts scheduled for next week.

### The Freezer

All scheduled content sits in the SQL Database (PostgreSQL).

### The Beat

A metronome (Celery Beat) wakes up every 60 seconds and asks:

> "Is anything due to go live right now?"

### The Handoff

If yes, it moves the data from the Database to the Redis Rail.

---

## 4. The Rail (Redis)

This is our high-speed message broker. It only holds tasks that need to be executed immediately. It’s the "hot plate" in the kitchen.

### Efficiency

Redis is in-memory (RAM), making it millions of times faster than disk storage for passing messages between the Scheduler and the Workers.

---

## 5. The Workers (Platform Adapters)

These are independent Python scripts that do the actual work.

### Translation

They pull a task from Redis, see it's for "Instagram Story," and translate our universal content into the exact multi-part form data Meta requires.

### Separation of Concerns

#### Post Worker
Handles standard feed images/videos.

#### Story Worker
Handles vertical aspect ratios and 24-hour expiry logic.

#### Ad Worker
Interfaces with the Ads Management API to set budgets and tracking pixels.

### Retries

If the API returns a 500 error, the worker catches it and puts the task back in Redis to try again in 10 minutes.

---

# The Order of Operations (What to Build First)

## Phase 1: OAuth Flow

Set up the OAuth Flow. We can't do anything without the keys.

---

## Phase 2: Universal Content Schema

Design the Database Schema for the universal content container.

---

## Phase 3: Redis + Celery Foundation

Stand up Redis and Celery with a simple "Hello World" task.

---

## Phase 4: Meta Adapter

Build the Meta Adapter (Post + Story) first, as it’s the most complex and has the longest approval time.

---

## Phase 5: Ads + Multi-Platform Expansion

Build the Ads Adapter and other platforms (X, Reddit, etc.) once the organic flow is stable.
