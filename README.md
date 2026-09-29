# VisionAI Dashboard — HSN Bukit Jalil

Real-time crowd-analytics dashboard for TM One Vision AI, built for Hari
Sukan Negara 2026 (3 cameras, 9–11 Oct) and later TM Elevate (1 camera,
12–14 Oct).

## Team

- **Front-end:** Farisha
- **Back-end:** Adriana, Lukman

## Stack

Laravel 13 + Laravel Sail (Docker) + MySQL + Redis + Laravel Reverb
(WebSocket broadcasting) + Chart.js + hls.js.

## Mockup
<img width="3712" height="2475" alt="Mockup dashboard v0 2 jpg" src="https://github.com/user-attachments/assets/1b1dcaa6-7034-4d35-9b31-148145a45c10" />


## How it works

Two independent pipelines feed the same page:

1. **Video** — a camera's RTSP feed is converted to HTTP (HLS) by MediaMTX
   and shown in the camera boxes. No analysis happens on this feed.
2. **Detection data** — SenseStudio's AI watches the real camera and pushes
   JSON events (gender, age, count) to our webhook. Until that's finalized,
   a dummy generator produces fake events on the same schedule so the whole
   app can be built and tested now.

Both pipelines write to the same database tables and broadcast over Reverb,
so the dashboard updates live no matter which data source is active.

## Requirements

- Windows + WSL2 (Ubuntu) + Docker Desktop
- Node (via Sail, no separate install needed)

## First-time setup

```bash
cd ~/visionai-dashboard
./vendor/bin/sail up -d
./vendor/bin/sail artisan migrate --seed --seeder=CameraSeeder
```

## Running it (3 terminals, each `cd ~/visionai-dashboard` first)

```bash
./vendor/bin/sail artisan reverb:start      # WebSocket server
./vendor/bin/sail artisan schedule:work     # generates dummy detections every 5s
./vendor/bin/sail npm run dev               # compiles JS/CSS
```

Dashboard: **http://localhost**
Live stats endpoint: **http://localhost/dashboard/stats**

## Key files

| File | Purpose |
|---|---|
| `app/Actions/IngestDetection.php` | Single place that saves a detection and broadcasts it |
| `app/Services/DummyAnalyticsProvider.php` | Fake data generator (swap out once real API is confirmed) |
| `app/Http/Controllers/SenseStudioWebhookController.php` | Receives real events from SenseStudio |
| `app/Http/Controllers/DashboardController.php` | Builds the numbers the dashboard displays |
| `resources/views/dashboard.blade.php` | The dashboard page (Farisha's design) |
| `routes/console.php` | Turns the dummy generator on/off |

## Switching from dummy to real data

1. Confirm the real event format from `storage/logs/laravel.log` and update
   `SenseStudioWebhookController::extractAttributes()` to match.
2. Replace each camera's placeholder `external_id` with its real SenseStudio
   device id.
3. Remove the `Schedule::command('analytics:poll')` line in
   `routes/console.php` to stop the dummy generator.
4. `./vendor/bin/sail artisan migrate:fresh --seed --seeder=CameraSeeder`
   to clear out fake data.

## Open questions

- Which SenseStudio scene template actually provides gender/age/count data?
- Where the app will be hosted so SenseStudio can reach the webhook?
- Whether one event = one person, or a batch (`target_count`)?
