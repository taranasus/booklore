# Booklore Fork - Status & Change Summary

**Last Updated:** 2026-01-16

## Repository Overview

This is a customized fork of [booklore-app/booklore](https://github.com/booklore-app/booklore) with enhancements for automated workflows.

### Repository Links

| Location | URL | Purpose |
|----------|-----|---------|
| **Upstream** | https://github.com/booklore-app/booklore | Original project |
| **GitHub Fork** | https://github.com/taranasus/booklore | Public fork for PRs |
| **Gitea (Private)** | https://git.taranasus.xyz/taranasus/booklore | CI/CD & development |
| **Production** | http://192.168.1.141:6060 | UNRAID deployment |

### Git Remotes
```
upstream  https://github.com/booklore-app/booklore.git    (original)
origin    https://github.com/taranasus/booklore.git       (public fork)
gitea     https://git.taranasus.xyz/taranasus/booklore.git (private CI/CD)
```

---

## Active Pull Requests

### PR #2279 - Auto-Import Feature
- **URL:** https://github.com/booklore-app/booklore/pull/2279
- **Branch:** `feat/bookdrop-auto-import`
- **Status:** OPEN
- **Created:** 2026-01-15
- **Description:** Adds ability to automatically import books from BookDrop when metadata is successfully found

### PR #2280 - Amazon Parser Scoring Fix
- **URL:** https://github.com/booklore-app/booklore/pull/2280
- **Branch:** `fix/metadata-selection-scoring`
- **Status:** OPEN
- **Created:** 2026-01-15
- **Description:** Improves Amazon parser result selection with scoring algorithm and fixes provider priority order

---

## Custom Changes (Not in Upstream)

### 1. CI/CD Pipeline
**File:** `.gitea/workflows/build-and-deploy.yml`

Gitea Actions workflow that:
- Builds Angular frontend and Spring Boot backend
- Creates Docker image
- Deploys to UNRAID server automatically on push to `develop`

### 2. Auto-Import Feature
**Primary Files:**
- `booklore-api/.../service/bookdrop/BookdropAutoImportService.java` (NEW)
- `booklore-api/.../service/bookdrop/BookdropEventHandlerService.java` (modified)
- `booklore-api/.../model/dto/settings/AppSettingKey.java` (added enum)
- `booklore-api/.../model/dto/settings/AppSettings.java` (added field)
- `booklore-api/.../service/appsettings/AppSettingService.java` (loads setting)
- `booklore-ui/.../metadata-settings/metadata-settings-component.html` (toggle UI)
- `booklore-ui/.../metadata-settings/metadata-settings-component.ts` (handler)
- `booklore-ui/.../shared/model/app-settings.model.ts` (model field)

**Functionality:**
- Adds setting `BOOKDROP_AUTO_IMPORT_ENABLED` (default: false)
- When enabled, books dropped in BookDrop folder are automatically imported IF:
  - Title is found
  - At least one author is found
- Books without complete metadata remain for manual review
- Setting accessible via: Settings → Metadata Settings → Auto-Download section

### 3. Amazon Parser Scoring
**Files:**
- `booklore-api/.../service/metadata/parser/AmazonBookParser.java`
- `booklore-api/.../service/metadata/MetadataRefreshService.java`

**Functionality:**
- Improved result selection using scoring algorithm based on title/author similarity
- Fixed provider priority order to match UI documentation

### 4. Automated Upstream Sync
**File:** `.gitea/workflows/sync-upstream.yml`

Gitea Actions workflow that:
- Runs daily at 3 AM UTC (or manually triggered)
- Fetches latest from upstream (booklore-app/booklore)
- Attempts automatic merge
- If successful → pushes to develop (triggers build/deploy)
- If conflicts → creates a Gitea issue for manual resolution

### 5. Documentation Files
- `CLAUDE.md` - Agent context file with infrastructure details
- `IMPLEMENTATION_PLAN.md` - Original implementation plan (completed)
- `TECHNICAL_SPEC.md` - Technical specification for auto-import feature
- `FORK_STATUS.md` - This file

---

## Upstream Sync Strategy

### How It Works

This fork uses an **automated merge-based sync** strategy:

1. **Daily Schedule:** A Gitea Action runs at 3 AM UTC
2. **Check for Updates:** Compares local HEAD with upstream/develop
3. **Auto-Merge:** If no conflicts, merges and pushes automatically
4. **Conflict Alert:** If conflicts exist, creates a Gitea issue

### Why Merge (Not Rebase)

- **No force-push required** - CI/CD pipeline isn't disrupted
- **Preserves history** - Both upstream and custom commits visible
- **Contained conflicts** - Isolated to single merge commit
- **CI/CD friendly** - Successful merges trigger build/deploy automatically

### Manual Conflict Resolution

When you receive a "Upstream sync conflict" issue:

```bash
# 1. Fetch latest from all remotes
git fetch upstream
git fetch gitea

# 2. Start the merge
git checkout develop
git merge upstream/develop

# 3. Resolve conflicts
# - Edit conflicting files
# - Keep your customizations where they matter
# - Accept upstream for everything else

# 4. Complete the merge
git add .
git commit -m "Merge upstream develop - resolved conflicts"

# 5. Push (triggers build/deploy)
git push gitea develop
```

### Git Rerere (Recommended)

Enable rerere to remember conflict resolutions:

```bash
git config rerere.enabled true
```

Git will then auto-resolve recurring conflicts using your previous resolutions.

---

## Sync Status

### Current Position
- **Base commit:** `99c8c131` (fix comicvine metadata search)
- **Our commits ahead:** 13 commits
- **Upstream commits behind:** 12 commits (as of 2026-01-16)

### Upstream Changes Not Yet Merged
The following upstream changes are NOT in our fork yet:
- `3a41e25a` - fix(chart-ui): resolve glitches in chart rendering
- `69ad5787` - Fix flyway
- `c207d4f6` - Fix out-of-order migration file versions
- `d8e7e0b5` - Fix Angular tests
- `36d9f748` - Implement Public Shelves
- `c20b1972` - Remove support for the legacy ePub reader
- `abceba34` - Update Angular dependencies
- `8c35f241` - New eBook reader (EPUB, MOBI, AZW3, FB2)
- `775887b5` - Shelf filtering options
- `869bea9c` - Auto-save metadata feature
- `709f90dc` - Increase parser request interval
- `574c01b4` - Fix ReadingSessionRepository
- `2be02017` - Extract file-specific information from book (refactor)

**Note:** The upstream has a major refactor (`2be02017`) that changes how book files are structured. Merging this may require conflict resolution.

---

## Deployment Information

### Docker Container
- **Name:** booklore
- **Image:** booklore-custom:latest (built by CI/CD)
- **Port:** 6060

### UNRAID Template
- **Location:** `/boot/config/plugins/dockerMan/templates-user/my-booklore.xml`
- **Repository:** `booklore/booklore:custom`

### Environment Variables
```
DATABASE_URL=jdbc:mariadb://192.168.1.141:3306/booklore
DATABASE_USERNAME=booklore
DATABASE_PASSWORD=BookL0reDB2026!
BOOKLORE_PORT=6060
TZ=Europe/London
```

### Volume Mounts
```
/mnt/user/books → /books
/mnt/user/appdata/booklore/bookdrop → /bookdrop
/mnt/user/appdata/booklore/data → /app/data
```

---

## Common Tasks

### Sync with Upstream (Automated)
Upstream sync runs automatically daily at 3 AM UTC.

**Manual trigger:** Go to Gitea Actions → "Sync with Upstream" → Run workflow

**Manual sync (if needed):**
```bash
git fetch upstream
git checkout develop
git merge upstream/develop
# Resolve any conflicts
git push gitea develop
```

### Deploy Changes
Simply push to `develop` branch on gitea:
```bash
git push gitea develop
```
CI/CD will automatically build and deploy.

### Check CI/CD Status
- Gitea Actions: https://git.taranasus.xyz/taranasus/booklore/actions

### View Container Logs
```bash
sshpass -p '?xFQxYDd$PRaPr#4' ssh root@jack.server "docker logs booklore --tail 100"
```

### Rollback to Upstream Version
```bash
sshpass -p '?xFQxYDd$PRaPr#4' ssh root@jack.server << 'EOF'
docker stop booklore && docker rm booklore
docker pull booklore/booklore:latest
docker run -d --name booklore --restart=unless-stopped \
  -p 6060:6060 \
  -v /mnt/user/books:/books:rw \
  -v /mnt/user/appdata/booklore/bookdrop:/bookdrop:rw \
  -v /mnt/user/appdata/booklore/data:/app/data:rw \
  -e DATABASE_URL=jdbc:mariadb://192.168.1.141:3306/booklore \
  -e DATABASE_USERNAME=booklore \
  -e DATABASE_PASSWORD=BookL0reDB2026! \
  -e BOOKLORE_PORT=6060 \
  -e TZ=Europe/London \
  booklore/booklore:latest
EOF
```

---

## Feature Testing

### Auto-Import Feature
1. Navigate to Settings → Metadata Settings
2. Enable "Auto-Import from BookDrop" toggle in the Auto-Download section
3. Drop a book file into `/mnt/user/appdata/booklore/bookdrop/`
4. If metadata is found (title + author), book auto-imports to library
5. If metadata incomplete, book remains in BookDrop for manual review

---

## Commit History (Our Changes)

```
0a29b42c chore(metadata): clean up debug logging from metadata selection
ebcf38c1 fix(metadata): correct provider priority order to match UI documentation
e05e8180 debug(metadata): add logging to show provider titles and priority selection
3b42db7a debug(metadata): add detailed scoring logs for Amazon candidate selection
8f01783f fix(metadata): use filename fallback for scoring when PDF metadata is empty
41d11969 feat(metadata): improve Amazon parser result selection with scoring
03810408 Fix auto-import lazy loading exception
05e56a6e fix(frontend): align autoImportEnabled field name with backend
70efc639 fix(bookdrop): break circular dependency with @Lazy annotation
4b887ffb fix(ci): write tar to /tmp to avoid race condition
e3916e33 feat(bookdrop): add auto-import feature for BookDrop files
ea15f1ab Fix CI workflow - simplify SSH commands
4f3a55a9 Add Gitea Actions CI/CD workflow for build and deploy
```
