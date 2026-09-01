# monad-daily-digest

Daily Monad ecosystem monitoring as a scheduled Claude skill. Scrapes X via 38 parallel queries (16 keyword, 22 account timelines), deduplicates and noise-filters, classifies posts into six narrative buckets, flags bridge opportunities (content) separately from integration targets (business development), verifies every URL and every top-story number by web search, and delivers one dashboard-style Telegram message under 4,000 characters by 8am. Archives markdown copies to Google Drive and carries corrections forward day to day.

Notable design choices: a hard qualification bar for integration targets (live product plus a named deposit asset, or it stays out), an exclusion list so tracked targets are not re-surfaced, competitive integration gaps ranked above everything else, and a changelog inside the skill so query rot gets fixed instead of silently degrading.

Before use, replace the placeholders: `{{TELEGRAM_BOT_TOKEN}}`, `{{PRIVATE_CHAT_ID}}`, `{{DRIVE_FOLDER_ID}}`. Runs on a daily schedule in Claude (Cowork scheduled tasks) with the Apify actor `danek/twitter-scraper` and a Google Drive connector.
