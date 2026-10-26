# Testudo: architecture case study

Testudo is a native macOS app for working with several AI models in one workspace. A single prompt can go to GPT, Claude, Gemini and Perplexity at once. Any reply can be branched, and branches can be merged into one answer. The app keeps a memory of facts across conversations.

This repo describes how the backend and billing work. The product code is private.

## 1. Local-first storage with an optional cloud

Chats are stored in a SQLite file on the user's machine. The same application code runs against Postgres in the cloud, using one Prisma schema shape for both. Users choose where their chats live. The cloud only sees what it needs for auth, billing, and provider routing.

## 2. Credits instead of per-provider token limits

Providers price tokens differently, and a plan that says "1 million tokens" hides a large cost gap between a cheap model and an expensive one. Testudo meters a single credit balance per user. Each model has a credit rate derived from its published price per million input and output tokens. An expensive model drains the balance faster, and the plan limit stays the same when a provider changes its prices. Only the rate table changes.

## 3. Reserve before you call the provider

Two requests from the same user can arrive at the same moment. A check-then-spend approach lets both pass when only one should. The balance is held in a single row per user per period, and each request reserves its estimated cost with one atomic update. When the stream finishes, the reservation is settled at the real cost. If the stream fails or is cancelled, the reservation is released.

## 4. Plan rules

Free resets daily, paid plans reset monthly. Free users get a short list of lower-cost models. Paid plans can use all models, because the credit rates already price the expensive ones. Bring-your-own-key plans skip metering entirely, since the user pays the provider directly.

## 5. Keys and secrets

User provider keys are encrypted with AES-256-GCM before they are stored, and they are decrypted only on the server at the moment of a call. The encryption key and database credentials live in AWS Secrets Manager. They are referenced from the ECS task definition and never stored as plain environment values.

## 6. Infrastructure

- ECS Fargate on ARM64 (Graviton), with the image built for that platform
- RDS PostgreSQL for the cloud data
- Cognito for sign-in
- Infrastructure defined in AWS CDK
- Billing webhooks verified by signature before any change to a user's plan

## 7. Shipping a Mac app

The app is signed with a Developer ID certificate and notarized by Apple, so it opens without warnings. Releases v1.0.0 to v1.0.3 shipped in October 2026. Building the universal disk image failed when an old copy of the app was still mounted. The workaround is documented in the build notes.

## What I'd do differently

- Per-provider token buckets were the first design. The credit model replaced them once I saw that the same budget meant very different costs across providers.
- Model allow-lists were initially too loose. A provider's default model and its most expensive model were not always the ones I expected, so I now check each allow-list against the published price table.
