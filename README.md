# Forkful 🍴

**PROG7314 / OPSC7312 - Portfolio of Evidence (Part 2)**

Forkful is a native Android culinary companion that turns *"what should I cook?"* into a fast swipe experience. It learns from every gesture, matches recipes to the ingredients in your fridge (Capture Fridge), and auto-builds a smart grocery list (Basket) that subtracts what you already own.

> **Live documentation site:** https://santerisuomi.github.io/PROG7314-POE---CraveMatch/
> **Setup guide (step by step, PowerShell included):** [SETUP.md](SETUP.md)

---

## 1. Purpose and scope

| Area | Detail |
|---|---|
| Product | Recipe discovery + pantry management + grocery aggregation |
| Platform | Native Android (Kotlin, Jetpack Compose, Material 3) |
| Architecture | Single-activity, MVVM + repository pattern, manual DI |
| Backend | Custom REST API - Supabase Edge Functions (Deno/TypeScript) + PostgreSQL |
| Auth | Google SSO (Credential Manager) + email, session JWTs issued by the API |
| Part 2 scope | SSO, Settings, REST API + the 3 user-defined features below |
| Final POE roadmap | Biometric lock, Room offline sync, FCM push, isiZulu/Afrikaans |

### User-defined features (from the Planning & Design document)

1. **Adaptive Swipe Discovery Deck** - a ranked card stack where every right/left swipe adjusts tag points in the `user_taste_profiles` table. Cards display an "engine pill" explaining why they are on top. (FR-06 .. FR-12)
2. **Capture Fridge** - pantry with expiry tracking, expiring-soon alerts, quick-add tag cloud, and a pantry-match algorithm returning match percentages + missing ingredients. (FR-17 .. FR-20)
3. **Smart Grocery Aggregator** - right-swiping a recipe auto-populates an aisle-grouped basket minus anything already in the pantry; items are checkable and shareable. (FR-21 .. FR-24)

---

## 2. Architecture

```
┌──────────────────────────────────────────────────────────────┐
│ UI (Jetpack Compose)                                        │
│  Login · Onboarding · Explore · SwipeStack · RecipeDetail   │
│  CookingMode · RecipeBox · Pantry · Basket · Settings       │
├──────────────────────────────────────────────────────────────┤
│ ViewModels (StateFlow <UiState>)                             │
├──────────────────────────────────────────────────────────────┤
│ Repositories (domain interfaces ← data implementations)      │
├──────────────────────────────────────────────────────────────┤
│ Retrofit + kotlinx.serialization · OkHttp AuthInterceptor    │
│ SessionManager (EncryptedSharedPreferences + DataStore)     │
└───────────────────────┬──────────────────────────────────────┘
                        │ HTTPS + Bearer JWT
┌───────────────────────▼──────────────────────────────────────┐
│ Supabase Edge Functions (Deno/TS) - the custom REST API      │
│  auth-sso · recipes-deck · recipe-detail · swipes ·          │
│  recipe-box · pantry · pantry-match · groceries ·            │
│  groceries-item · user-preferences                           │
├──────────────────────────────────────────────────────────────┤
│ PostgreSQL (users, recipes, recipe_tags, ingredients,        │
│  user_pantry, user_taste_profiles, user_swipes,              │
│  grocery_items)                                              │
└──────────────────────────────────────────────────────────────┘
```

Key structural decisions:

- **Manual DI** (`AppContainer` in `ForkfulApplication`) keeps wiring transparent and avoids annotation-processor complexity for a prototype of this size.
- **Optimistic swipes** (NFR-03): the deck advances instantly; the REST call syncs in the background.
- **Tokens never touch plaintext storage** (NFR-04): the session JWT lives in `EncryptedSharedPreferences` backed by the Android Keystore.
- **All screens log lifecycle + state transitions** through `ForkfulLogger` and a global `ActivityLifecycleCallbacks` implementation (rubric: functional logging).
- **All user-facing strings are externalised as new strings are added** (NFR-05); remaining literals from the Part 1 shell are being migrated ahead of the Final POE localisation work.

---

## 3. The REST API

Base URL: `https://<project-ref>.supabase.co` - source under [`supabase/`](supabase/).

| Method | Endpoint | Body / params | Response | FR |
|---|---|---|---|---|
| POST | `/functions/v1/auth-sso` | `{ ssoProvider, idToken/email, password?, displayName?, mode? }` | `{ token, userId, isNewUser }` | 01 |
| GET | `/functions/v1/recipes-deck` | `?mood&limit&dietaryTags&maxPrepTime&minRating&difficulty&search` | ranked card array | 06-12 |
| GET | `/functions/v1/recipe-detail` | `?recipeId` | full recipe + pantry flags | 13 |
| POST | `/functions/v1/swipes` | `{ recipeId, moodProfileId, direction }` | `{ status, swipeId, updatedTagPoints }` | 07-08, 21-22 |
| GET | `/functions/v1/recipe-box` | `?status=ALL\|SAVED\|COOKED` | saved recipe array | 15-16 |
| POST | `/functions/v1/recipe-box` | `{ recipeId, cooked }` | `{ status }` | 16 |
| GET | `/functions/v1/pantry` | - | pantry rows | 17 |
| POST | `/functions/v1/pantry` | `{ name, quantity, unit, expiryDate }` | `{ pantryItemId, status }` | 17-18 |
| DELETE | `/functions/v1/pantry` | `?pantryItemId` | `{ status }` | 17 |
| POST | `/functions/v1/pantry-match` | `{ missingThreshold }` | matches + missing lists | 20 |
| GET | `/functions/v1/groceries` | - | `[{ aisle, items }]` | 23-24 |
| PUT | `/functions/v1/groceries-item` | `?itemId` + `{ isChecked }` | `{ status }` | 24 |
| GET | `/functions/v1/user-preferences` | - | preference object | 25 |
| PUT | `/functions/v1/user-preferences` | `{ language, dietaryTags, notificationsEnabled, theme, onboarded, seedCuisines, resetTasteProfiles }` | `{ status, updated }` | 25-27 |

**Taste engine:** every swipe inserts into `user_swipes` and adjusts `user_taste_profiles` points (+2 RIGHT / -1 LEFT, floor -50). Deck ranking sums the points of each recipe's tags within the active mood; onboarding seeds +10 per selected cuisine via the cuisine identity tag. **Pantry matching** uses normalised, stop-word-cleaned name containment so *"Tomatoes"* matches *"Ripe Tomatoes"*.

---

## 4. Building and running

```powershell
git clone https://github.com/SanteriSuomi/PROG7314-POE---CraveMatch.git
cd PROG7314-POE---CraveMatch
```

Open in **Android Studio**, plug in a physical device, press **Run**. Before the app can talk to the backend you must complete [SETUP.md](SETUP.md) (Supabase project + SQL migration + Edge Function deploy + Google OAuth client ≈ 30 minutes).

Unit tests:

```powershell
.\gradlew testDebugUnitTest
```

33 tests cover input validation, pantry expiry logic, DTO mappers, the REST wire contract (MockWebServer) and ViewModel state machines (onboarding, swipe deck, login validation).

---

## 5. Version control and CI

- GitHub-only workflow; **no ZIP submissions**.
- `main` is protected by convention; feature work happens on branches (`Justin`, `part2/*`).
- **GitHub Actions** (`.github/workflows/android-ci.yml`) runs on every push and PR: JDK 17 → Gradle setup → Android SDK platform 37 (canary channel) → `assembleDebug` → `testDebugUnitTest` → uploads reports on failure.
- This repository is a fork of the team org repo; the merge procedure back into `PROG7314-Open-Source-Coding-2026/PROG7314-POE---CraveMatch` is documented in [SETUP.md](SETUP.md) section 8.

## 6. Documentation site

`docs/index.html` is published via **GitHub Pages** at
https://santerisuomi.github.io/PROG7314-POE---CraveMatch/ and mirrors this README with the full setup walkthrough and endpoint reference.

## 7. Demonstration video

> _(Replace this line with the unlisted YouTube link before submission - it must cover SSO, settings, web-service round-trips, cloud data verification and the three custom features.)_

## 8. Team

| Member | Student number |
|---|---|
| Jamie Abrahams | ST10434047 |
| Justin Fussell | ST10280758 |
| Isabelle Devlin | ST10445500 |
| Matteo Nusca | ST10440432 |

**Emeris BCAD Year 3 - The Independent Institute of Education**
