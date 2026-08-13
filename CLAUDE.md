# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

A Laravel 6 social-network demo app ("larasns"-style): users write **Articles**, tag them, **like** articles, and **follow** other users. Auth supports both standard Laravel email/password and Google OAuth (via Laravel Socialite). Backend is server-rendered Blade; a handful of Vue 2 components are mounted as interactive islands for AJAX actions (like/unlike, follow/unfollow, tag input) rather than a full SPA.

Stack: PHP ^7.2, Laravel ^6.2, Laravel Socialite ^4.3, PHPUnit ^8, Vue 2 + Laravel Mix/webpack for assets, MySQL/PostgreSQL/SQLite depending on environment.

## Commands

```bash
# Install
composer install
npm ci

# First-time setup
cp .env.example .env
php artisan key:generate
php artisan migrate           # add --seed to run database/seeds/DatabaseSeeder.php

# Run the app
php artisan serve
npm run watch                 # or `npm run hot` for HMR, `npm run dev` for a single build

# Build frontend assets
npm run dev                   # development build
npm run prod                  # production build (also runs on Heroku via heroku-postbuild)

# Tests (PHPUnit; testsuites are Unit and Feature, see phpunit.xml)
vendor/bin/phpunit
vendor/bin/phpunit tests/Feature/ArticleControllerTest.php   # single file
vendor/bin/phpunit --filter testIndex                        # single test method
```

There is no local linter/formatter command in this repo — PHP style is enforced by **StyleCI** (`.styleci.yml`, Laravel preset) as a hosted service on push, not a CLI tool.

Test environment: `phpunit.xml` forces `DB_CONNECTION=sqlite` / `DB_DATABASE=:memory:` regardless of `.env`, so `vendor/bin/phpunit` needs no local database. CI (CircleCI) instead runs against real Postgres. Feature tests use Laravel's `RefreshDatabase` trait and model factories (`database/factories`).

## Architecture

### Domain model (app/*.php)

Business logic lives on the three Eloquent models, not in controllers/services:

- `Article` belongsTo `User`; belongsToMany `User` through the `likes` pivot; belongsToMany `Tag` through `article_tag`. Exposes `isLikedBy(?User)` and a `count_likes` accessor (`getCountLikesAttribute`).
- `User` hasMany `Article`; belongsToMany `Article` through `likes` (as the liker); and is **self-referencing** through the `follows` pivot for social graph: `followers()` uses `followee_id`/`follower_id`, `followings()` uses the reverse. Exposes `isFollowedBy(?User)`, `count_followers`, `count_followings`.
- `Tag` belongsToMany `Article`; exposes a `#name` accessor (`hashtag`).

Controllers stay thin: they load relations eagerly (`->load([...])`), sort in PHP (`sortByDesc('created_at')`), and pass view models straight to Blade — there's no repository/service layer.

### Routing & authorization (routes/web.php)

All routes are in `web.php`; `routes/api.php` is unused (default stub). Key patterns:

- `Route::resource('/articles', ...)` is split: `index`/`show` are public, everything else (`create/store/edit/update/destroy`) requires `auth` middleware.
- Like/unlike and follow/unfollow are separate `PUT`/`DELETE` routes returning small JSON payloads (`['id' => ..., 'countLikes' => ...]`) consumed by the Vue components below — not Blade views.
- `ArticleController::__construct()` calls `$this->authorizeResource(Article::class, 'article')`, which wires every resourceful action to `App\Policies\ArticlePolicy` automatically (edit/update/destroy require `$user->id === $article->user_id`; everything else is open).

### Auth (app/Http/Controllers/Auth)

Standard Laravel scaffolding (`Auth::routes()`) plus a custom social-login flow layered on top:

- `GET/POST /login/{provider}` and `/register/{provider}` (see `routes/web.php`) drive `LoginController` and `RegisterController`.
- `LoginController::handleProviderCallback` looks up the user by the provider's email; if found, logs them in; if not, redirects to the provider-specific registration form carrying the OAuth token.
- `RegisterController::registerProviderUser` re-resolves the user from that token (`Socialite::driver($provider)->userFromToken($token)`) and creates a `User` with `password = null`.
- Only `google` is configured (`config/services.php`, env `GOOGLE_CLIENT_ID`/`GOOGLE_CLIENT_SECRET`), but the routes/controllers are provider-agnostic — adding a provider means adding Socialite config, not new routes.
- Password reset sends `PasswordResetNotification` wrapping an empty `App\Mail\BareMail` mailable (`User::sendPasswordResetNotification`).

### Frontend (resources/)

Blade views under `resources/views/{articles,users,tags,auth}` render full pages. `resources/js/components/*.vue` (`ArticleLike`, `FollowButton`, `ArticleTagsInput`) are individually mounted Vue 2 widgets for AJAX interactions — there is no client-side router or single global Vue app tree; `resources/js/app.js` just registers/mounts components used on a given page. `resources/lang/{en,ja}` provides English and Japanese translations (Japanese is used for user-facing validation attribute names in `ArticleRequest::attributes()`).

`ArticleRequest` (form request) also does post-validation transformation: tags arrive as a JSON string from `ArticleTagsInput.vue`, validated with `json` + no-whitespace/no-slash regexes, then in `passedValidation()` decoded, capped at 5, and reduced to plain tag-name strings for the controller to `firstOrCreate`.

## Deployment

Multiple deployment targets coexist in this repo — check which is relevant before assuming Heroku/AWS-only:

- **CircleCI** (`.circleci/config.yml`): `build` job runs against PHP 7.3 + Postgres 11.6, runs `npm run dev` then `vendor/bin/phpunit`. On `master`, a `deploy` job git-pushes to an AWS CodeCommit repo (triggering CodePipeline/CodeDeploy downstream); `deploy_old` is a legacy S3+CodeDeploy path left in place.
- **AWS CodeDeploy** (`appspec.yml` + `scripts/after_install.sh`): deploys to `/home/webapp/laravel-ci`, then runs `php artisan migrate --force` and `php artisan config:cache`.
- **AWS CodeBuild** (`buildspec.yml`): `composer install --no-dev` + `npm ci` + `npm run prod`.
- **Heroku** (`Procfile`): `heroku-php-apache2 public/`, using the `heroku-postbuild` npm script.
- **AWS infra** (`aws/ec2.yml`): CloudFormation template for the VPC/EC2 layout these deploy targets run on.
