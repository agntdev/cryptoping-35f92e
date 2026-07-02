# CryptoPing — Bot specification

**Archetype:** finance

**Voice:** professional and concise — write every user-facing message, button label, error, and empty state in this voice.

CryptoPing is a personal Telegram bot that allows users to maintain a private list of cryptocurrencies and receive notifications for price thresholds and percentage changes over a 1-hour period. It features interactive inline buttons for managing the watchlist, a /price command for instant price checks, optional morning summaries, quiet hours, and spam protection. The owner can access anonymized statistics about user counts and notification frequencies.

> This is the complete contract for the bot. Implement EVERY entry point, flow, feature, integration, and edge case below. The completeness review checks the bot against this document after each build pass.

## Primary audience

- retail crypto investors and traders
- bot owner

## Success criteria

- Users can add, remove, and configure notification rules for cryptocurrencies in their watchlist
- Users receive accurate and timely notifications for price thresholds and percentage changes
- Owner can view anonymized statistics about user activity and notification triggers

## Entry points

Every feature must be reachable from the bot's command/button surface (button-first; only /start and /help are slash commands).

- **/start** (command, actor: user, command: /start) — Open the main menu and begin onboarding
- **/price** (command, actor: user, command: /price) — Check current price of a specific cryptocurrency or all in the watchlist
- **Manage Watchlist** (button, actor: user, callback: watchlist:manage) — Open the watchlist management interface with inline buttons
- **Enable Morning Summary** (button, actor: user, callback: summary:enable) — Configure and enable the optional daily morning summary
- **Set Quiet Hours** (button, actor: user, callback: quiet:hours) — Configure quiet hours for notification suppression

## Flows

### Onboarding
_Trigger:_ /start

1. Greet user and offer quick setup
2. Set timezone (auto-detected or user override)
3. Set quiet hours (default 23:00-07:00)
4. Enable/disable morning summary
5. Show popular coin buttons for quick addition

_Data touched:_ User profile

### Add Coin to Watchlist
_Trigger:_ Add coin button or /price command

1. Prompt for ticker or show popular coin buttons
2. Validate ticker and suggest corrections if needed
3. Add coin to watchlist
4. Prompt to set notification rules for the new coin

_Data touched:_ Watchlist item, Notification rule

### Configure Notification Rules
_Trigger:_ Configure rule button

1. Select coin to configure
2. Set price threshold rules (above/below with target price)
3. Set percent-change rules (enable and set threshold)
4. Set cooldown duration for each rule

_Data touched:_ Notification rule

### Price Check
_Trigger:_ /price command

1. Check current price of specified coin or all in watchlist
2. Show price, 24h change, and last-tracked price if available

_Data touched:_ Watchlist item

### Morning Summary
_Trigger:_ User-local time

1. Generate summary of current prices
2. Highlight coins with > configured percent-change in last 24h
3. List rules that would currently trigger without sending alerts

_Data touched:_ Watchlist item, Notification rule

### Notification Trigger
_Trigger:_ Price threshold or percent-change rule

1. Check if current time is within quiet hours
2. If not, send notification with price change details
3. Apply cooldown to prevent repeat notifications
4. If during quiet hours, queue notification for later delivery

_Data touched:_ Notification rule, User profile

## Data entities

Durable data (must survive a restart) uses the toolkit's persistent store, never in-memory maps.

- **User profile** _(retention: persistent)_ — User's Telegram ID, display name, locale/timezone, settings (quiet hours, morning summary), and personal watchlist
  - fields: Telegram ID, display name, locale/timezone, quiet hours start/end, morning summary time, summary enabled/disabled, watchlist
- **Watchlist item** _(retention: persistent)_ — Cryptocurrency ticker, display name, tracking modes, and last-notified state
  - fields: ticker, display name, price thresholds, percent-change rule, last price, last notification timestamp, notification cooldown state
- **Notification rule** _(retention: persistent)_ — Type of rule (price threshold or percent-change), enabled/disabled status, and cooldown duration
  - fields: type, enabled, target price, percent threshold, cooldown duration, last triggered timestamp
- **Owner analytics** _(retention: persistent)_ — Aggregated metrics about active users and notification triggers
  - fields: total active users, notification triggers by coin, notification triggers by rule type, daily/weekly aggregates

## Integrations

- **Telegram** (required) — Bot API messaging and inline buttons
- **Price data provider** (required) — External market data API for cryptocurrency prices
Call external APIs against their real contract (correct endpoints, ids, params); credentials from env. Do not fake responses.

## Owner controls

- View anonymized statistics about user counts and notification frequencies
- Configure default settings (timezone, quiet hours, cooldowns)

## Notifications

- Price threshold notifications
- Percent-change notifications
- Morning summary
- Quiet hours summary at end of quiet period

## Permissions & privacy

- User data is private and not shared with third parties
- Owner analytics only show aggregated, anonymized metrics
- No user identifiers are exposed in analytics

## Edge cases

- Invalid or unknown tickers with fuzzy matching and suggestions
- Price-source failures with silent retries and no notifications during stale data
- Quiet hours with queued notifications and aggregated delivery at end of period

## Required tests

- Verify that users can add and remove coins from their watchlist
- Test notification rules for price thresholds and percent changes
- Validate morning summary delivery at user-local time
- Ensure quiet hours behavior with queued notifications
- Confirm owner analytics show correct aggregated metrics

## Assumptions

- Default timezone is auto-detected from Telegram locale
- Default quiet hours are 23:00-07:00 local
- Morning summary is optional and default disabled
- Default percent-change rule is 5% over 1 hour
- Default cooldowns are 6 hours for price-threshold and 1 hour for percent-change
- Notifications during quiet hours are queued and delivered at end of period
