# Discord News Briefing Bot

Production-ready Discord bot for collecting market-relevant RSS news, classifying it, generating a stored daily briefing, and delivering that briefing through Discord slash commands and scheduled guild delivery.

## What It Does

- Fetches RSS news from configured feeds.
- Normalizes, date-filters, and deduplicates articles.
- Classifies articles with rule-based logic plus optional LLM classification.
- Generates a Korean daily investment briefing with the existing briefing-generation LLM call.
- Stores generated briefings in PostgreSQL.
- Serves `/지금브리핑` from stored database data only.
- Provides `/서버설정` for interactive guild configuration with buttons and a modal.
- Provides `/핑` for fast bot health and latency checks.
- Delivers stored briefings automatically to each guild's configured channel and briefing time.
- Supports interactive Discord UI for the main briefing, market news browsing, and server settings.
- Tracks both collected article count and final reflected article count.

## Weekend Policy

Daily briefing generation uses `Asia/Seoul` by default:

- Saturday briefing covers Friday news.
- Sunday generation is skipped.
- Monday briefing covers Saturday and Sunday news together.
- Other weekdays cover the previous calendar day.

## Runtime Flow

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

`/지금브리핑` does not fetch RSS, regenerate the briefing, or call the LLM. It renders the latest stored `daily_briefings` row.

Scheduled generation and scheduled delivery are separate concerns:

- The daily generation scheduler creates or updates the stored briefing.
- The guild delivery scheduler sends the latest stored briefing to configured Discord channels at each guild's `briefing_time`.

## AI-Based News Processing

The production pipeline keeps the expensive work in the generation path and keeps the Discord read path fast:

- RSS ingestion gathers candidate articles from configured feeds.
- Normalization and filtering constrain the dataset to the intended briefing coverage window.
- Deduplication removes overlapping headlines and repeated feed entries.
- Rule classification assigns categories where deterministic logic is sufficient.
- Optional LLM classification improves article labeling when configured.
- A single LLM briefing-generation step produces the curated top-level briefing content.
- Full classified article storage is preserved for category and market-news browsing in Discord.

The interactive Discord commands always read from the database. They do not re-run RSS ingestion or LLM generation.

## Discord Commands

Current user-facing slash commands:

- `/지금브리핑` - shows the latest stored briefing with interactive controls.
- `/서버설정` - opens the current guild briefing settings and interactive controls.
- `/핑` - returns a fast health check with websocket latency.

`/서버설정` currently exposes:

- Current briefing time
- Alert enabled state
- Current interest categories
- `⏰ 시간 변경` button, which opens a real Discord modal
- `🔔 알림 ON/OFF` button for immediate toggling
- `🧩 분야 설정` button for interactive category selection

## Interactive UI

The bot uses Discord-native interaction components instead of text-only configuration:

- `/지금브리핑` renders the top-level investment issues embed and opens additional browsing views from buttons.
- Market news browsing uses a select menu for category switching and paginated browsing.
- Category and market-news browsing are backed by stored final article sets, not regenerated summaries.
- `/서버설정` uses buttons for configuration actions.
- Briefing time changes use `discord.ui.Modal` with an `HH:MM` text input.
- Category changes use a select-based interaction flow.

This keeps read operations fast and keeps settings changes close to the point of use.

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

## Database

Current application data is split between briefing storage, guild configuration, and normalized Discord domain tables.

- `daily_briefings` stores generated briefing payloads, coverage windows, generation status, and final reflected article count.
- `guild_settings` stores guild-specific delivery and briefing preferences.
- `users`, `guilds`, and `guild_members` model Discord identity and membership in a normalized way.
- `action_logs` and `action_log_changes` provide normalized audit logging with per-field change rows.

The raw collected article count is stored inside `daily_briefings.content_json` as `raw_article_count`; the final reflected article count is stored in `source_article_count`.

## Database Design

### users

Global Discord user identity.

- Internal primary key
- `discord_user_id` unique key
- Snapshot-friendly current fields such as `username` and `display_name`
- Timestamps for creation and updates

This allows one Discord user to exist once in the database even when they interact in multiple guilds.

### guilds

Global Discord guild identity.

- Internal primary key
- `discord_guild_id` unique key
- Current guild name
- Timestamps for creation and updates

This separates domain identity from guild-specific settings rows and avoids using raw Discord IDs as the only relational key.

### guild_members

Guild membership bridge between `users` and `guilds`.

- Internal primary key
- Foreign keys to `guilds.id` and `users.id`
- Guild-specific nickname
- Optional `joined_at`
- `last_seen_at`
- Unique constraint on `(guild_id, user_id)`

This is the correct place to store guild nickname because the same user can have different nicknames in different guilds.

### guild_settings

Per-guild briefing configuration.

- One row per guild
- Foreign key to `guilds.id`
- Delivery channel ID
- Briefing time
- Enabled flag
- Category list
- Briefing mode
- Summary length
- Timezone

The application repository layer still exposes guild-centric behavior while storing the relationship through normalized guild rows.

### action_logs

Parent audit log table for user actions.

- References normalized guild, user, and optional guild member rows
- Keeps immutable snapshot fields such as `username`, `display_name`, `guild_nickname`, and `guild_name`
- Stores the action name and creation timestamp

Snapshots remain necessary because normalized reference tables show current state, while audit logs must preserve what was true at the time of the action.

### action_log_changes

Child audit log table for structured field-level changes.

- Foreign key to `action_logs.id`
- `field_name`
- `before_value`
- `after_value`

This keeps change history relational and queryable. The project deliberately avoids using a generic JSON blob for structured audit trails.

## Audit Logging

Interactive user actions can be recorded through the normalized audit layer.

Current integration includes:

- `/지금브리핑`
- `/서버설정`
- Market news opening from the briefing UI
- Briefing time changes
- Alert toggles
- Category changes
- `/핑` health checks

Audit logging is intentionally fail-safe. Logging failures should not break the primary bot interaction.

## Setup

### Requirements

- Docker and Docker Compose
- Python 3.11 if running locally outside Docker
- Discord bot token
- PostgreSQL database

Docker Compose is the recommended path because it starts both the bot and PostgreSQL with the expected service networking.

### Environment

Create a `.env` file in the project root. A minimal local Docker setup looks like:

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

Do not commit `.env` or any real secrets.

## Run With Docker

Start PostgreSQL and the bot:

```bash
docker compose up --build
```

Apply database migrations:

```bash
docker compose run --rm bot alembic upgrade head
```

If the bot is already running before migrations are applied, restart it after migration:

```bash
docker compose restart bot
```

## Run Locally Without Docker

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

For local execution from the host, set `DATABASE_URL` to a host-reachable database URL, for example:

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

## Alembic

Create a migration after model changes:

```bash
alembic revision --autogenerate -m "describe change"
```

Apply migrations:

```bash
alembic upgrade head
```

## Tests

Run tests with:

```bash
PYTHONPATH=src pytest
```

If `pytest` is unavailable in your environment, install `requirements.txt` first.

## Notes And Troubleshooting

- Discord global slash-command sync can take time. Set `DISCORD_GUILD_ID` during development for faster guild command sync.
- Historical-date manual testing with the live RSS provider can be misleading because RSS feeds usually expose current items only. Use an injected fake provider or fixtures when testing old dates.
- `/지금브리핑` shows only stored DB data. If there is no ready briefing row, wait for generation or create one through the scheduled generation path.
- `/서버설정` is the production settings entry point. It is intended to replace fragmented legacy settings commands.
- Scheduled delivery requires a guild channel and enabled guild settings.
- Missing `OPENAI_API_KEY` causes LLM classification and briefing generation to fall back where supported; production-quality summaries require valid LLM configuration.
- Keep secrets out of git. Use `.env` locally and secret management in deployment.

## Policy And Support

New2Signal is an informational market-briefing bot. It is not financial advice and does not guarantee accuracy, completeness, or market outcomes.

* Privacy Policy: `PRIVACY.md`
* Terms of Service: `TERMS.md`
* Support email: [95doubleh@gmail.com](mailto:95doubleh@gmail.com)
* Support server: https://discord.gg/c4c6nBt5

For data deletion or privacy-related requests, contact the support channel above and include the relevant Discord server ID or Discord user ID where possible.
