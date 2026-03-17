# TODO — kstack roadmap

## Phase 3.6: Visual PR Annotations + S3 Upload
  - [ ] `/setup-kstack-upload` skill (configure S3 bucket for image hosting)
  - [ ] `browse/bin/kstack-upload` helper (upload file to S3, return public URL)
  - [ ] `/ship` Step 7.5: visual verification with screenshots in PR body
  - [ ] `/review` Step 4.5: visual review with annotated screenshots in PR
  - [ ] WebM → GIF conversion (ffmpeg) for video evidence in PRs

## Phase 4: Skill + Browser Integration
  - [ ] ship + browse: post-deploy verification
    - Browse staging/preview URL after push
    - Screenshot key pages
    - Check console for JS errors
    - Compare staging vs prod via snapshot diff
    - Include verification screenshots in PR body
    - STOP if critical errors found
  - [ ] review + browse: visual diff review
    - Browse PR's preview deploy
    - Annotated screenshots of changed pages
    - Compare against production visually
    - Check responsive layouts (mobile/tablet/desktop)
    - Verify accessibility tree hasn't regressed
  - [ ] deploy-verify skill: lightweight post-deploy smoke test
    - Hit key URLs, verify 200s
    - Screenshot critical pages
    - Console error check
    - Compare against baseline snapshots
    - Pass/fail with evidence

## Phase 5: State & Sessions
  - [ ] v20 encryption format support (AES-256-GCM) — future Chromium versions may change from v10
  - [ ] Auth vault (encrypted credential storage, referenced by name, LLM never sees passwords)
  - [ ] Video recording (record start/stop)
  - [ ] retro + browse: deployment health tracking
    - Screenshot production state
    - Check perf metrics (page load times)
    - Count console errors across key pages
    - Track trends over retro window

## Future Ideas
  - [ ] Linux/Windows cookie decryption (GNOME Keyring / kwallet / DPAPI)
  - [ ] Trend tracking across QA runs — compare baseline.json over time, detect regressions
  - [ ] CI/CD integration — `/qa` as GitHub Action step, fail PR if health score drops
  - [ ] Accessibility audit mode — `--a11y` flag for focused accessibility testing

## Ideas & Notes
  - Browser is the nervous system — every skill should be able to see, interact with, and verify the web
  - Skills are the product; the browser enables them
  - One repo, one install, entire AI engineering workflow
  - Accessibility tree snapshots use ~200-400 tokens vs ~3000-5000 for full DOM — critical for AI context efficiency
