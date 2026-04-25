# New2Signal

AI-powered Discord bot for turning daily market news into usable signals inside Discord.

> News is noise. Signal is what matters.

## Overview

New2Signal is a production-oriented Discord bot that ingests RSS news, cleans and classifies articles, generates a structured Korean market briefing, stores the result in PostgreSQL, and delivers it through slash commands and scheduled channel delivery.

The project exists to solve a practical problem: raw financial news is abundant, repetitive, and difficult to consume consistently. New2Signal reduces that noise by producing one stored briefing that users can browse quickly in Discord, while still preserving traceability through article links, audit logs, and normalized configuration data.

It is built for two audiences at once:

- end users who want a concise daily market view directly in Discord
- operators who need a maintainable bot with clear data flow, normalized schema design, and production-safe interaction handling

## What It Does

- Collects market-relevant news from configured RSS feeds
- Normalizes, filters, and deduplicates articles before briefing generation
- Classifies news with rule-based logic and optional LLM support
- Generates a Korean daily market briefing with a single briefing-generation LLM step
- Stores briefings in PostgreSQL so user-facing reads stay fast and stable
- Serves the latest stored briefing through `/지금브리핑`
- Exposes `/서버설정` for interactive server configuration
- Exposes `/핑` for bot health and latency checks
- Delivers stored briefings automatically to each server's configured channel and time
- Records structured audit logs for important user actions and configuration changes

## Core Features

- Stored briefing model:
  user-facing commands read from the database, not from live RSS or live LLM generation
- Interactive Discord UX:
  buttons, selects, pagination, and modal-based configuration flows
- Per-guild delivery settings:
  channel, time, alert state, and briefing categories
- Normalized data model:
  users, guilds, memberships, settings, parent audit logs, and child change rows
- Production safety:
  scheduler separation, permission-aware interactions, and non-fatal audit logging

## Discord Commands

### `/지금브리핑`

Shows the latest stored market briefing in the current channel.

What users get:

- the main market-issues summary
- issue-related stock and company references
- article links where available
- a one-line market takeaway
- market-news browsing through interactive controls

This command is public by design. It reads only from the latest stored `daily_briefings` record.

### `/서버설정`

Shows the current server configuration and provides the main administration UI.

What users see:

- briefing delivery channel
- briefing time
- alert enabled state
- selected interest categories

What authorized users can change:

- `📢 채널 설정`
- `⏰ 시간 변경`
- `🔔 알림 ON/OFF`
- `🧩 관심 분야 설정`

The command is visible to everyone in the server, but modification actions are restricted to users with `Manage Server` or `Administrator`.

### `/핑`

Returns a fast liveness and websocket latency check.

## Interactive UI

New2Signal uses Discord-native UI components rather than forcing users through fragmented text commands.

### Briefing UI

- `/지금브리핑` renders the stored briefing as embeds
- `시장 뉴스 보기` opens additional browsing UI
- market-news browsing supports category switching and pagination
- browsing works on the stored final briefing content, not a regenerated response

### Settings UI

- `/서버설정` displays current guild settings publicly
- configuration changes are performed from buttons
- briefing time uses a real `discord.ui.Modal`
- briefing channel selection uses a dropdown of valid text channels
- category selection uses a select menu

### Interaction Safety

- long-running interactions are deferred where needed
- public operations use public followups
- permission-denied responses stay ephemeral
- bot send/embed permissions are checked before public interaction output

## System Architecture

### Runtime Flow

```text
RSS fetch
-> normalize
-> filter by briefing coverage date
-> deduplicate
-> rule classification
-> LLM classification
-> LLM briefing generation
-> store in PostgreSQL
-> /지금브리핑 and scheduled delivery read from DB
```

The expensive work happens in the generation path. User-facing Discord interactions stay fast because they render stored data rather than rebuilding the briefing on demand.

`/지금브리핑` does not fetch RSS, regenerate the briefing, or call the LLM. It reads the latest stored `daily_briefings` row and renders it.

### Scheduler Separation

Scheduled generation and scheduled delivery are intentionally separate:

- the daily generation scheduler creates or updates the stored briefing
- the guild delivery scheduler sends the latest stored briefing to each configured guild at its own `briefing_time`

This split keeps generation concerns independent from server-level delivery concerns.

### AI-Based News Processing

The production pipeline keeps the Discord read path lightweight while allowing richer classification and summarization upstream.

- RSS ingestion gathers candidate articles from configured feeds
- normalization and filtering constrain the dataset to the intended briefing coverage window
- deduplication removes overlapping headlines and repeated feed entries
- rule classification handles deterministic category assignments
- optional LLM classification improves article labeling when configured
- a single LLM briefing-generation step produces the final curated briefing content
- full classified article context remains available for category and market-news browsing

## Database Design

Current application data is split between briefing storage, guild configuration, normalized Discord domain tables, and normalized audit logs.

### `daily_briefings`

Stores generated briefing payloads, coverage windows, generation status, and article-count metadata.

- final reflected article count is stored in `source_article_count`
- raw collected article count is stored inside `content_json.raw_article_count`

### `users`

Global Discord user identity.

- internal primary key
- unique `discord_user_id`
- current `username`
- current `display_name`
- created and updated timestamps

This allows a single Discord user to exist once even when they interact across multiple guilds.

### `guilds`

Global Discord guild identity.

- internal primary key
- unique `discord_guild_id`
- current guild name
- created and updated timestamps

This avoids using raw Discord IDs as the only relational key across the system.

### `guild_members`

Bridge table between users and guilds.

- internal primary key
- foreign keys to `guilds.id` and `users.id`
- guild-specific nickname
- optional `joined_at`
- `last_seen_at`
- unique constraint on `(guild_id, user_id)`

This is where guild nickname belongs, because the same user can have different identities across servers.

### `guild_settings`

Per-guild briefing configuration.

- one row per guild
- foreign key to `guilds.id`
- delivery channel ID
- briefing time
- enabled flag
- category list
- briefing mode
- summary length
- timezone

The repository layer still exposes guild-centric operations while using normalized guild references underneath.

### `action_logs`

Parent audit log table for user actions.

- references normalized guild, user, and optional guild-member rows
- stores immutable snapshot fields such as `username`, `display_name`, `guild_nickname`, and `guild_name`
- stores action name and creation timestamp

Snapshot fields remain necessary because normalized tables represent current state, while audit history must preserve what was true at the time of the action.

### `action_log_changes`

Child audit log table for structured field-level changes.

- foreign key to `action_logs.id`
- `field_name`
- `before_value`
- `after_value`

This keeps change history queryable and relational. The project deliberately avoids a generic JSON metadata blob for structured audit trails.

## Audit Logging

Interactive user actions can be recorded through the normalized audit layer.

Current integration includes:

- `/지금브리핑`
- `/서버설정`
- market-news opening from the briefing UI
- briefing time changes
- alert toggles
- category changes
- channel changes
- permission-denied server-settings modification attempts
- `/핑` health checks

Audit logging is intentionally fail-safe. Logging failures do not break the primary bot interaction.

## Usage Notes

### Normal users

- can run `/지금브리핑`
- can run `/서버설정`
- can view current server settings
- can see the settings buttons
- cannot change settings without appropriate server permissions

### Server managers and administrators

Users with `Manage Server` or `Administrator` can modify settings through `/서버설정`.

That includes:

- changing the delivery channel
- changing briefing time
- toggling alerts
- changing interest categories

### Public vs private responses

- main briefing output is public
- server settings view is public
- successful configuration changes are public
- permission-denied responses are private
- operational error messages may remain private where appropriate

### Weekend policy

Daily briefing generation uses `Asia/Seoul` by default:

- Saturday briefing covers Friday news
- Sunday generation is skipped
- Monday briefing covers Saturday and Sunday news together
- other weekdays cover the previous calendar day

## Tech Stack

- Python 3.11
- `discord.py`
- PostgreSQL
- SQLAlchemy ORM
- Alembic
- Docker / Docker Compose
- optional OpenAI-compatible LLM APIs for classification and briefing generation

## Setup

### Requirements

- Docker and Docker Compose
- Python 3.11 if running locally outside Docker
- Discord bot token
- PostgreSQL database

Docker Compose is the recommended path because it starts both the bot and PostgreSQL with the expected service networking.

### Environment

Create a `.env` file in the project root.

A minimal local Docker setup:

```env
DISCORD_BOT_TOKEN=your_discord_bot_token
DATABASE_URL=postgresql+psycopg://news_briefing:news_briefing@db:5432/news_briefing
POSTGRES_DB=news_briefing
POSTGRES_USER=news_briefing
POSTGRES_PASSWORD=news_briefing
LOG_LEVEL=INFO
DEFAULT_TIMEZONE=Asia/Seoul
DEFAULT_BRIEFING_TIME=07:00
```

Optional variables:

```env
DISCORD_GUILD_ID=123456789012345678
NEWS_RSS_FEED_URLS=https://example.com/feed.xml,https://example.com/another.xml
OPENAI_API_KEY=your_openai_api_key
OPENAI_BASE_URL=https://api.openai.com/v1
OPENAI_MODEL=gpt-4o-mini
TEST_BRIEFING_SCHEDULE_ENABLED=false
```

Do not commit `.env` or real secrets.

### Run With Docker

Start PostgreSQL and the bot:

```bash
docker compose up --build
```

Apply database migrations:

```bash
docker compose run --rm bot alembic upgrade head
```

If the bot was already running before migrations were applied:

```bash
docker compose restart bot
```

### Run Locally Without Docker

Create a virtual environment and install dependencies:

```bash
python3.11 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

Start PostgreSQL separately or through Docker:

```bash
docker compose up -d db
```

For host-local execution, point `DATABASE_URL` at a host-reachable database:

```env
DATABASE_URL=postgresql+psycopg://news_briefing:news_briefing@localhost:5432/news_briefing
```

Apply migrations:

```bash
alembic upgrade head
```

Start the bot:

```bash
python src/main.py
```

### Alembic

Create a migration after model changes:

```bash
alembic revision --autogenerate -m "describe change"
```

Apply migrations:

```bash
alembic upgrade head
```

### Tests

Run tests with:

```bash
PYTHONPATH=src pytest
```

If `pytest` is unavailable, install `requirements.txt` first.

## Project Structure

```text
src/
  bot/
    commands/              Discord slash commands and rendering
    views/                 Interactive Discord button views
    briefing_delivery.py   Scheduled Discord delivery
    interactive_briefing.py
  core/
    config.py              Environment configuration
    logger.py              JSON logging setup
  db/
    models.py              SQLAlchemy models
    repositories.py        Database access helpers
    session.py             SQLAlchemy engine/session setup
  news/
    providers.py           RSS provider
    normalizer.py          Raw RSS item normalization
    filtering.py           Date filtering
    deduplicator.py        Duplicate removal
    classifier.py          Rule classification
    llm_classifier.py      LLM classification
    briefing_generator.py  LLM briefing generation
    pipeline.py            Article preparation pipeline
  scheduler/
    daily_briefing.py      Daily generation scheduler
    guild_delivery.py      Guild-specific delivery scheduler
  services/
    audit_logs.py          Audit logging and Discord snapshot helpers
    briefings.py           Briefing generation/storage service
  main.py                  Bot entrypoint

alembic/                   Database migrations
tests/                     Unit tests
```

## Notes

- Discord global slash-command sync can take time. Use `DISCORD_GUILD_ID` during development for faster guild sync.
- Historical-date testing with live RSS feeds can be misleading because RSS usually exposes current items only. Use fake providers or fixtures for stable historical tests.
- `/지금브리핑` renders only stored DB data. If there is no ready briefing row, wait for generation or create one through the generation flow.
- `/서버설정` is the unified production settings entry point.
- Scheduled delivery requires both a configured guild channel and sufficient bot permissions in that channel.
- Missing `OPENAI_API_KEY` causes LLM-supported steps to fall back where supported, but production-quality summaries require valid LLM configuration.
- Keep secrets out of git. Use `.env` locally and proper secret management in deployment.

## Disclaimer

New2Signal is an informational market-briefing bot. It does not provide financial advice, investment advice, legal advice, or tax advice.

Outputs may rely on third-party feeds, external APIs, AI models, and Discord platform behavior. News classification and summarization may be incomplete, delayed, inaccurate, or misleading.

Users are responsible for their own decisions. Nothing in the bot should be interpreted as a recommendation to buy, sell, or hold any asset.

## Policy & Support

- Privacy Policy: `PRIVACY.md`
- Terms of Service: `TERMS.md`
- Support email: [95doubleh@gmail.com](mailto:95doubleh@gmail.com)
- Support server: https://discord.gg/c4c6nBt5

For privacy, support, or data deletion requests, include the relevant Discord server ID or Discord user ID where possible.
