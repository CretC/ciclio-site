# Harta modulelor

Generat de `npm run deps:map` (dependency-cruiser, nivel de folder). Nu edita de mână.
Săgeata `A --> B` înseamnă „A importă din B”; numărul = importuri între fișiere. Testele sunt excluse.

```mermaid
graph TD
  domain["domain"]
  data["data"]
  i18n["i18n"]
  notifications["notifications"]
  widget["widget"]
  shared["shared"]
  app["app"]
  features_account["features/account"]
  features_obligations["features/obligations"]
  features_onboarding["features/onboarding"]
  features_reminders["features/reminders"]
  features_settings["features/settings"]
  data -->|36| domain
  i18n -->|10| domain
  notifications -->|8| data
  notifications -->|6| domain
  notifications -->|3| i18n
  widget -->|2| data
  widget -->|1| domain
  widget -->|1| i18n
  shared -->|9| data
  shared -->|14| domain
  shared -->|1| i18n
  shared -->|4| notifications
  shared -->|2| widget
  app -->|3| data
  app -->|4| domain
  app -->|2| features_account
  app -->|1| features_obligations
  app -->|1| features_onboarding
  app -->|2| features_reminders
  app -->|1| features_settings
  app -->|4| notifications
  app -->|30| shared
  features_account -->|4| data
  features_account -->|1| notifications
  features_account -->|5| shared
  features_obligations -->|10| data
  features_obligations -->|30| domain
  features_obligations -->|1| notifications
  features_obligations -->|93| shared
  features_onboarding -->|1| domain
  features_onboarding -->|3| shared
  features_reminders -->|2| data
  features_reminders -->|3| domain
  features_reminders -->|2| notifications
  features_reminders -->|13| shared
  features_settings -->|4| data
  features_settings -->|5| domain
  features_settings -->|5| notifications
  features_settings -->|24| shared
  features_settings -->|1| widget
```

## Cine importă din cine

- `domain` → (nimic)
- `data` → `domain`
- `i18n` → `domain`
- `notifications` → `data`, `domain`, `i18n`
- `widget` → `data`, `domain`, `i18n`
- `shared` → `data`, `domain`, `i18n`, `notifications`, `widget`
- `app` → `data`, `domain`, `features/account`, `features/obligations`, `features/onboarding`, `features/reminders`, `features/settings`, `notifications`, `shared`
- `features/account` → `data`, `notifications`, `shared`
- `features/obligations` → `data`, `domain`, `notifications`, `shared`
- `features/onboarding` → `domain`, `shared`
- `features/reminders` → `data`, `domain`, `notifications`, `shared`
- `features/settings` → `data`, `domain`, `notifications`, `shared`, `widget`
