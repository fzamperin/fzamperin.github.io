---
title: "Scaling a Node.js Monorepo: Lessons from Building Suno's Microservices Platform"
date: 2026-02-27
description: "How we structured a 38-package monorepo with Lerna, NX, and Yarn Workspaces to power a fintech platform serving 100k+ users — from domain-driven design to automated Kubernetes deployments."
tags: ["node.js", "monorepo", "kubernetes", "devops", "architecture"]
draft: false
---

# Scaling a Node.js Monorepo at Suno

When I started at Suno the codebase was already growing fast. We had a fintech platform pulling real-time stock market data from a bunch of different APIs and we needed to serve 100k+ users without the whole thing falling apart. We went with a monorepo pretty early on. I ended up being the one maintaining it for most of my time there, so I want to share how it worked and what I learned along the way.

By the end we had **38 packages** in a single repo — APIs, background workers, shared libraries, database migrations, the whole thing. It sounds like a mess but honestly it worked really well once we got the structure right.

## Why Three Tools Instead of One

We used Lerna, Yarn Workspaces, and NX together. I know that sounds like overkill but each one did something the others couldn't:

- **Lerna 5** — versioning and publishing. It figured out what changed and bumped versions automatically
- **Yarn Workspaces** — dependency management. Hoisted shared deps so we weren't downloading the same packages 30 times
- **NX 14** — we mostly used it for build caching and the dependency graph. Made CI way faster

The big decision was going with **independent versioning** in Lerna. Each package had its own version number. So if someone pushed a fix to `shared-cache`, only that package and the ones that depend on it got a version bump. We weren't deploying the whole platform because someone fixed a typo in a utility function.

```json
{
  "version": "independent",
  "npmClient": "yarn",
  "useWorkspaces": true,
  "command": {
    "version": {
      "conventionalCommits": true
    }
  }
}
```

## How We Organized the Packages

I spent a lot of time thinking about folder structure early on, and I'm glad I did because it saved us later. Everything was grouped by business domain:

```
packages/
├── shared/              # 8 shared libraries
├── accounts/            # Auth, users, authorization
│   ├── authentication/
│   │   ├── core/        # Business logic
│   │   ├── database/    # TypeORM migrations
│   │   └── executables/
│   │       └── api-authenticator/
│   ├── users/
│   ├── notifications/
│   └── providers/       # Cognito, WordPress, etc.
├── applications/
├── boletador/           # Invoice processing
├── ratings/             # Stock ratings
├── reports/
└── suno-analytics/      # Market data
```

Every domain followed the same three-layer split:

**Core** was pure business logic. No Express, no NestJS, no HTTP stuff at all. Just functions and classes you could test without spinning up a server. We exported these as libraries so multiple services could use the same logic.

**Database** was TypeORM migrations and entity definitions. We kept these separate from everything else so database changes had their own version and their own deployment step. This was honestly one of the best decisions we made — it meant we could run migrations independently and roll them back without touching the application code.

**Executables** were the things that actually ran: NestJS APIs, SQS consumers, cron jobs. These imported from core and database but didn't share code between each other.

Any new developer could jump into a domain they'd never seen before and immediately know where to find things. That consistency was worth more than any documentation we could have written.

## The Shared Libraries

We had 8 packages under `shared/` that basically every domain depended on:

| Package | What it does |
|---------|-------------|
| `shared-cache` | Redis caching with ioredis |
| `shared-validation` | class-validator decorators |
| `shared-middlewares` | Auth middleware, error handlers, request logging |
| `shared-monitoring` | Sentry integration for errors and performance |
| `shared-exceptions` | Custom exception classes |
| `shared-storage` | S3 uploads/downloads |
| `shared-xlsx` | Excel file parsing and generation |
| `shared-zip` | ZIP handling |

All scoped under `@suno.softwares/` and consumed as normal npm deps. When you bumped `shared-cache` from `0.4.0` to `0.4.1`, Lerna would walk the dependency tree and bump everything that used it. You didn't have to think about it.

## Background Workers

Some things just can't run inside an API request. Processing uploaded invoice files could take 30 seconds. Generating a full report with charts could take a minute. So we had **SQS consumers** running as separate services.

The `boletador-consumer-files` worker was a good example of the pattern. An API endpoint would accept a file upload, dump it in S3, and push a message to SQS. The worker picked it up, downloaded the file, parsed the Excel or ZIP using our shared libraries, ran it through the `boletador-core` business logic, and wrote the results to Postgres. The API never had to wait.

We used the same approach for report generation. Each worker was its own package, its own Docker image, its own Kubernetes deployment. If the report worker was overloaded it didn't affect the API at all.

## Express to NestJS (Gradually)

Something you could clearly see in the codebase was the migration from Express to NestJS. The oldest services — authentication, analytics — were plain Express with `routing-controllers`. The newer stuff — applications, ratings, reports, boletador — was all NestJS with Fastify.

We never did a big rewrite. We just built new services in NestJS and left the old ones alone unless we had a reason to touch them. The shared libraries were framework-agnostic on purpose, so they worked with both. I think this is underrated — people want to rewrite everything at once but that almost never works. Just let the old stuff be old and build new stuff better.

## CI/CD: The Part That Actually Saved Us

With 38 packages you absolutely cannot rebuild everything on every push. That would be insane. So I built the CI pipeline around selective builds, and this ended up being probably the biggest productivity win of the whole project.

The GitLab CI pipeline had four stages:

1. **Test** — `yarn test --no-private --since` only ran tests for packages that actually changed
2. **Version Bump** — Lerna read the conventional commits and auto-bumped versions. `fix:` was a patch, `feat:` was a minor, `BREAKING CHANGE:` was a major
3. **Build** — here's where it got fun
4. **Deploy** — push to Kubernetes

For the build stage, I wrote a Node.js script that **dynamically generated a GitLab CI child pipeline**. After the version bump, the script scanned every package to see which executables needed rebuilding. It checked for packages that had a `namespace` field, were marked `private: true`, and had names containing `api`, `consumer`, `cron`, or `database`.

For each one that changed, it spit out YAML for a build job and a deploy job using shared templates. So if you only changed the ratings API, only the ratings API got built and deployed. Everything else was untouched.

This is what conventional commits were really for. Yeah the changelogs were nice:

```markdown
## [0.4.1] (2023-04-04)
### Bug Fixes
* exposed headers
* resolve eslint

## [0.4.0] (2022-11-07)
### Features
* add new route for list question users
```

But the real value was that Lerna could look at commits and automatically figure out what changed, what version it should be, and which packages downstream needed updating. The changelogs were just a side effect.

## Kubernetes

Every executable deployed to AWS EKS. We had two environments with different configs:

**Staging** got 1 replica and auto-deployed on every merge to `master`. Node affinity made sure staging pods only ran on staging nodes. Pretty standard.

**Production** used HPA (Horizontal Pod Autoscaler) with pod anti-affinity to spread across availability zones — pods scaled up and down based on actual load. Deploys were manual — you had to create a `deploy/*` tag and explicitly set `DEPLOY_TO=production`. Rolling updates with `maxUnavailable: 0` so there was never a moment where the service was down.

Database migrations ran as Kubernetes Jobs before the actual deployment. If the migration failed (backoff limit of 1, 60-second deadline), the deployment didn't proceed. We never had a situation where code was running against the wrong schema.

Docker images were multi-stage: one stage to compile TypeScript, another to copy just the `dist/` folder and production node_modules. Kept images small.

Image tags looked like `${PACKAGE_NAME}-${VERSION}-${ENV}`, so you could always tell exactly what was running where. When someone reported a bug in production you could immediately check "ok, ratings API is on 0.13.0 in prod but 0.13.2 in staging, the fix is already there, just needs promotion."

## What I'd Change

I don't want to make it sound like everything was perfect. Some things I'd do differently:

**Turborepo instead of Lerna** — When we started Lerna was the standard. Now I think Turborepo handles caching and task running better with less config. Lerna has been through a lot of ownership changes too.

**Stricter boundaries on core packages** — Some of our cores had light NestJS imports that snuck in over time. Should have enforced a ports-and-adapters pattern or at least had a lint rule for it.

**Better TypeORM entity sharing** — Each domain had its own database package which was correct, but we ended up duplicating some base entity patterns. Could have had a `shared-entities` package with common stuff like timestamps and soft deletes.

## What I Took Away From This

If you're thinking about a monorepo for your Node.js services:

Go with **independent versioning**. It's more work to set up but it means you're not deploying 38 packages because someone changed one. Your changelogs actually mean something too.

**Pick a domain structure and enforce it from day one.** The core/database/executable split was the reason new developers could onboard in a day instead of a week. Don't let people "just put it somewhere for now."

**Build your CI/CD pipeline early.** The dynamic pipeline generation script was maybe 70 lines of JavaScript and it probably saved us hundreds of hours over the life of the project. Without it we would have been sitting around waiting for builds that didn't need to happen.

**Keep shared libraries framework-agnostic.** This is what let us migrate from Express to NestJS without a rewrite. If your cache library imports from `@nestjs/common`, you're going to have a bad time.

We ended up with a system that served 100k+ users processing real-time market data across multiple microservices, and any developer on the team could deploy a single service in under 5 minutes. The monorepo structure was a big part of why that worked.
