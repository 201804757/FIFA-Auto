Build a production-ready TypeScript/Node assistant named `fifa-worldcup-video-assistant`.

Goal: each day, check FIFA World Cup completed matches and create a 30-45 second silent Remotion recap video for every completed match that has not already been covered. Do not only process the latest match; process all completed uncovered matches so the assistant recovers if a day is missed.

Hard requirements:

1. Use Remotion for video rendering.
2. The video must contain no audio. Do not include Remotion `<Audio>` components. Render with `--muted`.
3. Generate a separate narration script file named `script.md`. I will add the audio later manually.
4. Use a central Excel workbook at `data/fifa_world_cup_video_tracker.xlsx` that records every covered match, score, status, source URL, output folder, `video.mp4` link, `script.md` link, and last checked timestamp.
5. For each match, create one subfolder named after the match teams: `matches/<HomeTeam>_vs_<AwayTeam>/`.
6. Inside each match folder, put exactly two files and nothing else: `video.mp4` and `script.md`.
7. Temporary props, generated images, API responses, screenshots, and render cache must be stored outside match folders in `tmp/` or `.cache/` and cleaned after successful render.
8. The system must be idempotent. If a match is already listed as `Video Complete` and `matches/<teams>/video.mp4` exists, skip it unless `FORCE_RERENDER=true`.
9. The assistant should use data-driven/procedural visuals by default: animated scoreboard, goal timeline, stat cards, team-color motion background, group/knockout impact card, and final score card.
10. Do not use unlicensed broadcast highlight footage, copyrighted stills, official logos, or player images unless the project is explicitly configured with licensed assets. If licensed assets exist, they must live outside match folders, e.g. `assets/licensed/`.
11. Implement tests for match ID generation, folder naming, tracker updates, skip/rerender logic, and no-extra-files-in-match-folder validation.

Architecture to implement:

- `src/index.ts`: daily entry point.
- `src/matches/provider.ts`: interface for match data providers.
- `src/matches/fifaProvider.ts`: provider that can use official FIFA match centre/report pages or a configured licensed sports-data API.
- `src/tracker/excelTracker.ts`: read/write the Excel workbook using a reliable Node Excel library.
- `src/scripts/scriptWriter.ts`: produce a 30-45 second narration script in markdown.
- `src/remotion/MatchRecap.tsx`: Remotion component for visual recap.
- `src/render/renderMatch.ts`: render video via Remotion CLI or renderer API with `--muted`.
- `src/storage/storage.ts`: optional upload to S3/R2/Drive; if disabled, use local relative links.
- `src/utils/sanitize.ts`: safe team-folder names.
- `scripts/validate-output.ts`: fail if any match folder contains files other than `video.mp4` and `script.md`.

Video structure:

- 0-5s: animated title card with teams, score, stage/group, date, stadium if available.
- 5-16s: goal/key-event timeline.
- 16-28s: stat cards: shots, possession, cards, xG if licensed data provides it.
- 28-38s: group/knockout impact or match meaning.
- 38-45s: closing final-score card and next-fixture pointer if available.

Environment variables:

- `MATCH_DATA_PROVIDER`: `fifa` or `licensed_api`.
- `SPORTS_API_KEY`: required if using a licensed API.
- `OPENAI_API_KEY`: optional for AI-written script/visual prompt generation.
- `OUTPUT_ROOT`: default `matches`.
- `TRACKER_PATH`: default `data/fifa_world_cup_video_tracker.xlsx`.
- `FORCE_RERENDER`: default `false`.
- `VIDEO_DURATION_SECONDS`: default `40`, min `30`, max `45`.
- `OUTPUT_ASPECT`: default `9:16`, optionally `16:9`.
- `STORAGE_TARGET`: `local`, `s3`, `r2`, or `drive`.

GitHub Actions:

- Add a scheduled workflow and manual `workflow_dispatch`.
- Install Node dependencies.
- Run lint/tests.
- Run daily assistant.
- Validate folder contents.
- Commit tracker and outputs, or upload videos to configured storage and commit only tracker updates.

Acceptance criteria:

- Running `npm run daily` creates video and script only for uncovered completed matches.
- Running it twice without new matches skips all previously completed matches.
- Excel tracker includes the match score and links to `video.mp4` and `script.md`.
- Match folders contain exactly two files.
- Remotion video has no audio track or is rendered with audio disabled.
- The render duration is between 30 and 45 seconds.
