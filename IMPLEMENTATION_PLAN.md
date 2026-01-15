# Booklore Auto-Import Feature Implementation Plan

This document outlines the complete implementation plan for forking Booklore, setting up CI/CD, and implementing the auto-import feature. Each phase is designed to be executed independently by different agents.

---

## Phase 1: Repository Setup and Forking Strategy

### Objective
Set up a proper fork structure that allows contributions back to the upstream repository while maintaining a private development workflow.

### Prerequisites
- Git CLI access with credentials for both git.taranasus.xyz and github.com
- Access to Gitea admin panel (https://git.taranasus.xyz)

### Tasks

#### 1.1 Create GitHub Fork (Public)
1. Navigate to https://github.com/booklore-app/booklore
2. Click "Fork" to create a fork under the user's GitHub account (taranasus)
3. Note the fork URL: `https://github.com/taranasus/booklore`

#### 1.2 Create Gitea Repository (Private Development)
1. Access Gitea at https://git.taranasus.xyz
2. Create a new repository named `booklore`
3. Set it as private
4. Do NOT initialize with any files

#### 1.3 Clone and Configure Remotes
Execute in `/Users/justinpopa/Repos/Booklore`:

```bash
# Clone from the original upstream
git clone https://github.com/booklore-app/booklore.git .

# Rename origin to upstream
git remote rename origin upstream

# Add GitHub fork as origin (for PRs)
git remote add origin https://github.com/taranasus/booklore.git

# Add Gitea as private remote (for CI/CD development)
git remote add gitea https://git.taranasus.xyz/taranasus/booklore.git

# Verify remotes
git remote -v
# Expected output:
# gitea    https://git.taranasus.xyz/taranasus/booklore.git (fetch)
# gitea    https://git.taranasus.xyz/taranasus/booklore.git (push)
# origin   https://github.com/taranasus/booklore.git (fetch)
# origin   https://github.com/taranasus/booklore.git (push)
# upstream https://github.com/booklore-app/booklore.git (fetch)
# upstream https://github.com/booklore-app/booklore.git (push)

# Push to Gitea (creates initial mirror)
git push gitea develop

# Push to GitHub fork
git push origin develop
```

#### 1.4 Configure Branch Protection
On Gitea (https://git.taranasus.xyz):
1. Go to Repository Settings -> Branches
2. Add protection rule for `develop` branch
3. Require pull request reviews before merging (optional)

### Deliverables
- [x] GitHub fork created at github.com/taranasus/booklore (completed 2026-01-15)
- [x] Gitea repository created at git.taranasus.xyz/taranasus/booklore (completed 2026-01-15)
- [x] Local repository with three remotes configured (completed 2026-01-15)
- [x] Initial code pushed to both Gitea and GitHub fork (completed 2026-01-15)

### Verification
```bash
git remote -v  # Should show upstream, origin, gitea
git branch -a  # Should show develop branch
```

---

## Phase 2: CI/CD Runner Preparation

### Objective
Prepare the unity-runner VM with the necessary tools to build Java/Angular applications.

### Prerequisites
- SSH access to unity-runner (192.168.1.251)
- Runner already registered with Gitea

### Tasks

#### 2.1 Install Java Development Kit
SSH into the runner and install JDK 21:

```bash
sshpass -p 'zxcvbn' ssh -o StrictHostKeyChecking=no runner@192.168.1.251

# Install JDK 21
sudo apt update
sudo apt install -y openjdk-21-jdk

# Verify installation
java -version
javac -version

# Set JAVA_HOME if not already set
echo 'export JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64' >> ~/.bashrc
source ~/.bashrc
```

#### 2.2 Install Node.js 22 (for Angular build)
```bash
# Install Node.js 22 via NodeSource
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs

# Verify
node --version  # Should be v22.x
npm --version

# Install Angular CLI globally (optional, for convenience)
sudo npm install -g @angular/cli
```

#### 2.3 Install Docker CLI
```bash
# Install Docker CLI (no daemon needed, will connect to UNRAID)
sudo apt install -y docker.io

# Add runner user to docker group (for future local testing if needed)
sudo usermod -aG docker runner
```

#### 2.4 Install Additional Build Tools
```bash
# Install gradle wrapper dependencies
sudo apt install -y unzip wget

# Install sshpass for deployment scripts
sudo apt install -y sshpass
```

#### 2.5 Verify Act Runner Configuration
```bash
# Check runner status
sudo systemctl status act_runner

# View current config
cat ~/.config/act_runner/config.yaml

# Ensure labels include capability for this build
# Should have: ubuntu-latest:host
```

### Deliverables
- [x] JDK 21 installed and JAVA_HOME configured (completed 2026-01-15)
- [x] Node.js 22 installed (completed 2026-01-15)
- [x] Docker CLI installed (already present)
- [x] sshpass installed for deployment (already present)
- [x] Runner verified as active (completed 2026-01-15)

**NOTE:** The runner VM IP is **192.168.1.250** (not .251 as originally documented)

### Verification
```bash
java -version      # OpenJDK 21.0.9
node --version     # v22.22.0
npm --version      # 10.9.4
docker --version   # Docker 28.2.2
sshpass -V         # sshpass 1.09
sudo systemctl status act_runner  # Active and running
```

---

## Phase 3: Gitea Actions Workflow Setup

### Objective
Create CI/CD pipeline that builds the application and deploys it to UNRAID.

### Prerequisites
- Phase 1 and 2 completed
- Gitea repository accessible
- Runner capable of building the project

### Tasks

#### 3.1 Create Workflow Directory
In the local repository:

```bash
mkdir -p .gitea/workflows
```

#### 3.2 Create Build and Deploy Workflow
Create `.gitea/workflows/build-and-deploy.yml`:

```yaml
name: Build and Deploy Booklore

on:
  push:
    branches:
      - develop
      - feature/*
  pull_request:
    branches:
      - develop

env:
  IMAGE_NAME: booklore-custom
  IMAGE_TAG: ${{ github.sha }}
  REGISTRY: 192.168.1.141:5000

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Set up JDK 21
        run: |
          echo "JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64" >> $GITHUB_ENV
          export JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64
          java -version

      - name: Set up Node.js 22
        run: |
          node --version
          npm --version

      - name: Cache Gradle dependencies
        uses: actions/cache@v4
        with:
          path: |
            ~/.gradle/caches
            ~/.gradle/wrapper
          key: gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
          restore-keys: |
            gradle-

      - name: Cache npm dependencies
        uses: actions/cache@v4
        with:
          path: ~/.npm
          key: npm-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            npm-

      - name: Build Angular Frontend
        working-directory: booklore-ui
        run: |
          npm ci
          npm run build -- --configuration production

      - name: Build Spring Boot Backend
        working-directory: booklore-api
        run: |
          chmod +x gradlew
          ./gradlew build -x test

      - name: Build Docker Image
        run: |
          docker build \
            --build-arg VERSION=${{ github.sha }} \
            --build-arg BUILD_DATE=$(date -u +"%Y-%m-%dT%H:%M:%SZ") \
            -t ${{ env.IMAGE_NAME }}:${{ env.IMAGE_TAG }} \
            -t ${{ env.IMAGE_NAME }}:latest \
            .

      - name: Save Docker Image
        run: |
          docker save ${{ env.IMAGE_NAME }}:latest | gzip > booklore-custom.tar.gz

      - name: Deploy to UNRAID
        if: github.ref == 'refs/heads/develop' && github.event_name == 'push'
        env:
          UNRAID_SSH_PASS: ${{ secrets.UNRAID_SSH_PASS }}
        run: |
          # Copy image to UNRAID
          sshpass -p "$UNRAID_SSH_PASS" scp -o StrictHostKeyChecking=no \
            booklore-custom.tar.gz \
            root@192.168.1.141:/tmp/

          # Load and deploy on UNRAID
          sshpass -p "$UNRAID_SSH_PASS" ssh -o StrictHostKeyChecking=no root@192.168.1.141 << 'EOF'
          set -e

          # Load the new image
          gunzip -c /tmp/booklore-custom.tar.gz | docker load

          # Tag the image
          docker tag booklore-custom:latest booklore/booklore:custom

          # Stop the existing container
          docker stop booklore || true
          docker rm booklore || true

          # Start new container with same configuration
          docker run -d \
            --name booklore \
            --restart=unless-stopped \
            -p 6060:6060 \
            -v /mnt/user/books:/books:rw \
            -v /mnt/user/appdata/booklore/bookdrop:/bookdrop:rw \
            -v /mnt/user/appdata/booklore/data:/app/data:rw \
            -e DATABASE_URL=jdbc:mariadb://192.168.1.141:3306/booklore \
            -e DATABASE_USERNAME=booklore \
            -e DATABASE_PASSWORD="${DATABASE_PASSWORD}" \
            -e BOOKLORE_PORT=6060 \
            -e TZ=Europe/London \
            --log-opt max-size=50m \
            --log-opt max-file=1 \
            --label "net.unraid.docker.managed=dockerman" \
            --label "net.unraid.docker.webui=http://192.168.1.141:6060" \
            --label "net.unraid.docker.icon=https://raw.githubusercontent.com/booklore-app/booklore/main/frontend/src/assets/favicon.svg" \
            booklore/booklore:custom

          # Cleanup
          rm /tmp/booklore-custom.tar.gz

          # Verify container is running
          docker ps | grep booklore

          echo "Deployment complete!"
          EOF
```

#### 3.3 Configure Gitea Secrets
Access Gitea repository settings and add secrets:

1. Navigate to: https://git.taranasus.xyz/taranasus/booklore/settings/actions/secrets
2. Add the following secrets:
   - `UNRAID_SSH_PASS`: `?xFQxYDd$PRaPr#4`
   - `DATABASE_PASSWORD`: `BookL0reDB2026!`

#### 3.4 Update UNRAID Docker Template
Update the UNRAID template to use custom image. SSH into UNRAID:

```bash
sshpass -p '?xFQxYDd$PRaPr#4' ssh -o StrictHostKeyChecking=no root@jack.server

# Edit the template
cat > /boot/config/plugins/dockerMan/templates-user/my-booklore.xml << 'EOF'
<?xml version="1.0"?>
<Container version="2">
  <Name>booklore</Name>
  <Repository>booklore/booklore:custom</Repository>
  <Registry>Local Build</Registry>
  <Network>bridge</Network>
  <MyIP/>
  <Shell>bash</Shell>
  <Privileged>false</Privileged>
  <Support>https://git.taranasus.xyz/taranasus/booklore/issues</Support>
  <Project>https://booklore.org/</Project>
  <Overview>BookLore (Custom Build) - A self-hosted, multi-user digital library with smart shelves, auto metadata, Kobo and KOReader sync, BookDrop imports with AUTO-IMPORT feature, OPDS support, and a built-in reader for EPUB, PDF, and comics.</Overview>
  <Category>MediaApp:Books</Category>
  <WebUI>http://192.168.1.141:6060</WebUI>
  <TemplateURL/>
  <Icon>https://raw.githubusercontent.com/booklore-app/booklore/main/frontend/src/assets/favicon.svg</Icon>
  <ExtraParams>--restart=unless-stopped</ExtraParams>
  <PostArgs/>
  <CPUset/>
  <DateInstalled>1736765000</DateInstalled>
  <DonateText/>
  <DonateLink/>
  <Requires/>
  <Config Name="WebUI" Target="6060" Default="6060" Mode="tcp" Description="Booklore web interface port" Type="Port" Display="always" Required="true" Mask="false">6060</Config>
  <Config Name="Books Library" Target="/books" Default="/mnt/user/books" Mode="rw" Description="Main ebook library location" Type="Path" Display="always" Required="true" Mask="false">/mnt/user/books</Config>
  <Config Name="BookDrop" Target="/bookdrop" Default="/mnt/user/appdata/booklore/bookdrop" Mode="rw" Description="Auto-import folder for new books" Type="Path" Display="always" Required="true" Mask="false">/mnt/user/appdata/booklore/bookdrop</Config>
  <Config Name="Appdata" Target="/app/data" Default="/mnt/user/appdata/booklore/data" Mode="rw" Description="Application data and cache" Type="Path" Display="always" Required="true" Mask="false">/mnt/user/appdata/booklore/data</Config>
  <Config Name="DATABASE_URL" Target="DATABASE_URL" Default="jdbc:mariadb://192.168.1.141:3306/booklore" Mode="{3}" Description="MariaDB connection URL" Type="Variable" Display="always" Required="true" Mask="false">jdbc:mariadb://192.168.1.141:3306/booklore</Config>
  <Config Name="DATABASE_USERNAME" Target="DATABASE_USERNAME" Default="booklore" Mode="{3}" Description="Database username" Type="Variable" Display="always" Required="true" Mask="false">booklore</Config>
  <Config Name="DATABASE_PASSWORD" Target="DATABASE_PASSWORD" Default="" Mode="{3}" Description="Database password" Type="Variable" Display="always" Required="true" Mask="true">BookL0reDB2026!</Config>
  <Config Name="BOOKLORE_PORT" Target="BOOKLORE_PORT" Default="6060" Mode="{3}" Description="Application port" Type="Variable" Display="advanced" Required="false" Mask="false">6060</Config>
  <Config Name="TZ" Target="TZ" Default="Europe/London" Mode="{3}" Description="Timezone" Type="Variable" Display="advanced" Required="false" Mask="false">Europe/London</Config>
  <TailscaleStateDir/>
</Container>
EOF
```

#### 3.5 Push Workflow and Test
```bash
git add .gitea/
git commit -m "Add Gitea Actions CI/CD workflow for build and deploy"
git push gitea develop
```

### Deliverables
- [x] `.gitea/workflows/build-and-deploy.yml` created (completed 2026-01-15)
- [x] Gitea secrets configured (UNRAID_SSH_PASS, DATABASE_PASSWORD) (completed 2026-01-15)
- [x] UNRAID template updated for custom image (completed 2026-01-15)
- [x] Initial build triggered and successful (completed 2026-01-15 - Run #167)

### Verification
1. ✅ Gitea Actions workflow triggered on push
2. ✅ Build completes without errors (~2 minutes)
3. ✅ Docker image `booklore-custom:latest` deployed to UNRAID
4. ✅ Application accessible at http://192.168.1.141:6060

---

## Phase 4: Codebase Exploration and Analysis

### Objective
Understand the existing bookdrop implementation to determine where to add auto-import functionality.

### Prerequisites
- Phase 1 completed (repository cloned)

### Tasks

#### 4.1 Identify Bookdrop Related Code
Search the codebase for bookdrop-related files:

```bash
# Find Java files related to bookdrop
grep -r -l -i "bookdrop" booklore-api/src/

# Find Angular components related to bookdrop
grep -r -l -i "bookdrop" booklore-ui/src/

# Look for file watcher or monitoring services
grep -r -l -i "watch\|monitor\|schedule" booklore-api/src/
```

#### 4.2 Map the Bookdrop Flow
Document the current bookdrop process:

1. **File Detection:** How does the system detect new files in /bookdrop?
2. **Metadata Extraction:** What service extracts metadata from dropped files?
3. **Metadata Enrichment:** How does it query Google Books and Open Library?
4. **User Review:** Where is the review UI located?
5. **Import Process:** What happens when a user manually imports a book?

#### 4.3 Identify Key Classes and Services
Expected components to find:
- FileWatcherService or similar (monitors /bookdrop directory)
- MetadataService or BookMetadataService (fetches metadata)
- BookDropService or similar (handles bookdrop operations)
- ImportService (handles the actual import process)
- Settings entity and service (for application settings)

#### 4.4 Document Settings Architecture
Find how settings are stored and managed:

```bash
# Look for settings-related code
grep -r -l -i "settings\|preferences\|config" booklore-api/src/main/java/
grep -r -l -i "settings" booklore-ui/src/app/
```

#### 4.5 Create Technical Specification Document
Based on findings, create a document detailing:
- Current data flow for bookdrop
- Classes that need modification
- New classes that need to be created
- Database changes required (if any)
- Frontend changes required

### Deliverables
- [x] List of all bookdrop-related source files (completed 2026-01-15)
- [x] Data flow diagram of current bookdrop process (completed 2026-01-15)
- [x] Identified insertion points for auto-import logic (completed 2026-01-15)
- [x] Technical specification document for the feature (completed 2026-01-15)

### Output Document
Created `TECHNICAL_SPEC.md` in the repository with findings.

### Key Findings Summary
- **Primary Hook Point:** `BookdropEventHandlerService.processFile()` after metadata attachment
- **Settings System:** Uses key-value pairs in `app_setting` table (no migration needed)
- **Monitoring:** Must pause/resume monitoring during import to prevent race conditions
- **New Service Required:** `BookdropAutoImportService` for auto-import decision logic

---

## Phase 5: Backend Implementation - Auto-Import Feature

### Objective
Implement the backend logic for automatic book import when metadata is successfully found.

### Prerequisites
- Phase 4 completed (codebase understood)
- All relevant source files identified

### Tasks

#### 5.1 Add Auto-Import Setting to Database
Create a Flyway migration (or equivalent) to add the setting:

```sql
-- Add auto_import_enabled setting
-- File: booklore-api/src/main/resources/db/migration/V{next}_add_auto_import_setting.sql

ALTER TABLE app_settings ADD COLUMN auto_import_enabled BOOLEAN DEFAULT FALSE;
-- OR if settings are stored differently, adjust accordingly
```

#### 5.2 Update Settings Entity
Add the new field to the Settings JPA entity:

```java
// In Settings.java (or equivalent entity)
@Column(name = "auto_import_enabled")
private Boolean autoImportEnabled = false;

// Add getter and setter
public Boolean getAutoImportEnabled() {
    return autoImportEnabled;
}

public void setAutoImportEnabled(Boolean autoImportEnabled) {
    this.autoImportEnabled = autoImportEnabled;
}
```

#### 5.3 Update Settings DTO and Service
Add the field to DTOs and service layer:

```java
// In SettingsDTO.java
private Boolean autoImportEnabled;

// In SettingsService.java
// Add logic to read/write the new setting
```

#### 5.4 Modify Bookdrop Processing Logic
Find the service that processes bookdrop files and add auto-import logic:

```java
// Pseudo-code for the modification
public void processDroppedBook(BookDropFile file) {
    // Existing: Extract metadata
    BookMetadata metadata = extractMetadata(file);

    // Existing: Enrich metadata from external sources
    EnrichedMetadata enriched = enrichMetadata(metadata);

    // NEW: Check if metadata was successfully found
    boolean metadataFound = isMetadataComplete(enriched);

    // NEW: Check auto-import setting
    boolean autoImportEnabled = settingsService.isAutoImportEnabled();

    if (metadataFound && autoImportEnabled) {
        // NEW: Auto-import the book
        importBook(file, enriched);
        log.info("Auto-imported book: {}", enriched.getTitle());
    } else {
        // Existing: Leave for manual review
        addToPendingReview(file, enriched);
        log.info("Book pending review: {}", file.getFilename());
    }
}

private boolean isMetadataComplete(EnrichedMetadata metadata) {
    // Define what "successful metadata" means:
    // - Title is not null/empty
    // - At least one author
    // - Confidence score above threshold (if available)
    return metadata.getTitle() != null
        && !metadata.getTitle().isEmpty()
        && metadata.getAuthors() != null
        && !metadata.getAuthors().isEmpty();
}
```

#### 5.5 Add REST Endpoint for Setting
Ensure there's an API endpoint to toggle the setting:

```java
// In SettingsController.java
@PutMapping("/settings/auto-import")
public ResponseEntity<Void> setAutoImport(@RequestParam Boolean enabled) {
    settingsService.setAutoImportEnabled(enabled);
    return ResponseEntity.ok().build();
}

@GetMapping("/settings/auto-import")
public ResponseEntity<Boolean> getAutoImport() {
    return ResponseEntity.ok(settingsService.isAutoImportEnabled());
}
```

#### 5.6 Add Logging for Auto-Import Events
Add appropriate logging for debugging and audit:

```java
// Add to the import logic
log.info("Auto-import triggered for file: {}", file.getName());
log.debug("Metadata confidence: title={}, authors={}",
    metadata.getTitle(), metadata.getAuthors());
```

#### 5.7 Write Unit Tests
Create tests for the new functionality:

```java
// AutoImportServiceTest.java
@Test
void shouldAutoImportWhenMetadataFoundAndEnabled() {
    // Setup
    when(settingsService.isAutoImportEnabled()).thenReturn(true);
    BookDropFile file = createTestFile();
    when(metadataService.enrich(any())).thenReturn(completeMetadata());

    // Execute
    bookDropService.processDroppedBook(file);

    // Verify
    verify(importService).importBook(any(), any());
}

@Test
void shouldNotAutoImportWhenDisabled() {
    // Setup
    when(settingsService.isAutoImportEnabled()).thenReturn(false);
    BookDropFile file = createTestFile();
    when(metadataService.enrich(any())).thenReturn(completeMetadata());

    // Execute
    bookDropService.processDroppedBook(file);

    // Verify
    verify(importService, never()).importBook(any(), any());
    verify(pendingReviewService).add(any());
}

@Test
void shouldNotAutoImportWhenMetadataIncomplete() {
    // Setup
    when(settingsService.isAutoImportEnabled()).thenReturn(true);
    BookDropFile file = createTestFile();
    when(metadataService.enrich(any())).thenReturn(incompleteMetadata());

    // Execute
    bookDropService.processDroppedBook(file);

    // Verify
    verify(importService, never()).importBook(any(), any());
    verify(pendingReviewService).add(any());
}
```

### Deliverables
- [x] Database migration for new setting (NOT NEEDED - uses key-value app_setting table) (completed 2026-01-15)
- [x] Updated Settings entity, DTO, and service (completed 2026-01-15)
- [x] Modified bookdrop processing with auto-import logic (completed 2026-01-15)
- [x] REST endpoints for setting management (EXISTING endpoints work - uses updateSetting) (completed 2026-01-15)
- [x] Unit tests for new functionality (completed 2026-01-15)
- [x] All backend code compiles (completed 2026-01-15)

### Files Modified/Created
- `AppSettingKey.java` - Added `BOOKDROP_AUTO_IMPORT_ENABLED`
- `AppSettings.java` - Added `autoImportEnabled` field
- `AppSettingService.java` - Added loading of new setting
- `BookdropAutoImportService.java` - **NEW** - Auto-import logic
- `BookdropEventHandlerService.java` - Calls auto-import after metadata fetch
- `BookdropAutoImportServiceTest.java` - **NEW** - Unit tests

### Verification
```bash
cd booklore-api
./gradlew test
```

---

## Phase 6: Frontend Implementation - Settings Toggle

### Objective
Add a toggle switch in the settings UI to enable/disable auto-import functionality.

### Prerequisites
- Phase 5 completed (backend API ready)

### Tasks

#### 6.1 Locate Settings Component
Find the existing settings component in the Angular codebase:

```bash
# Find settings components
find booklore-ui/src -name "*setting*" -type f
```

#### 6.2 Add Auto-Import Toggle to Settings Interface
Update the TypeScript interface for settings:

```typescript
// In settings.model.ts or similar
export interface AppSettings {
    // ... existing properties
    autoImportEnabled: boolean;
}
```

#### 6.3 Update Settings Component Template
Add the toggle to the settings UI:

```html
<!-- In settings.component.html or similar -->
<div class="setting-item">
    <div class="setting-info">
        <h4>Auto-Import from BookDrop</h4>
        <p>Automatically import books when metadata is successfully found.
           Books without complete metadata will still require manual review.</p>
    </div>
    <mat-slide-toggle
        [checked]="settings.autoImportEnabled"
        (change)="onAutoImportToggle($event)">
    </mat-slide-toggle>
</div>
```

#### 6.4 Update Settings Component Logic
Add the handler for the toggle:

```typescript
// In settings.component.ts
onAutoImportToggle(event: MatSlideToggleChange): void {
    this.settingsService.setAutoImportEnabled(event.checked)
        .subscribe({
            next: () => {
                this.notificationService.success(
                    event.checked
                        ? 'Auto-import enabled'
                        : 'Auto-import disabled'
                );
            },
            error: (err) => {
                this.notificationService.error('Failed to update setting');
                // Revert the toggle
                event.source.checked = !event.checked;
            }
        });
}
```

#### 6.5 Update Settings Service
Add API calls to the settings service:

```typescript
// In settings.service.ts
setAutoImportEnabled(enabled: boolean): Observable<void> {
    return this.http.put<void>(
        `${this.apiUrl}/settings/auto-import`,
        null,
        { params: { enabled: enabled.toString() } }
    );
}

getAutoImportEnabled(): Observable<boolean> {
    return this.http.get<boolean>(`${this.apiUrl}/settings/auto-import`);
}
```

#### 6.6 Add Styling
Ensure the toggle matches the existing design:

```scss
// In settings.component.scss or global styles
.setting-item {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 16px;
    border-bottom: 1px solid var(--border-color);

    .setting-info {
        flex: 1;
        margin-right: 16px;

        h4 {
            margin: 0 0 4px 0;
            font-weight: 500;
        }

        p {
            margin: 0;
            color: var(--text-secondary);
            font-size: 0.875rem;
        }
    }
}
```

#### 6.7 Add Import for Angular Material
If not already imported, add MatSlideToggleModule:

```typescript
// In the appropriate module
import { MatSlideToggleModule } from '@angular/material/slide-toggle';

@NgModule({
    imports: [
        // ...
        MatSlideToggleModule,
    ]
})
```

### Deliverables
- [x] Settings interface updated with autoImportEnabled (completed 2026-01-15)
- [x] Toggle added to settings component template (completed 2026-01-15)
- [x] Settings component logic updated (completed 2026-01-15)
- [x] Settings service updated with API calls (EXISTING - uses saveSettings pattern) (completed 2026-01-15)
- [x] Styling applied consistent with existing design (completed 2026-01-15)
- [x] Frontend builds without errors (completed 2026-01-15)

### Files Modified
- `app-settings.model.ts` - Added `bookdropAutoImportEnabled` field and `BOOKDROP_AUTO_IMPORT_ENABLED` enum key
- `metadata-settings-component.ts` - Added toggle handler and initialization
- `metadata-settings-component.html` - Added toggle UI in Auto-Download section
- `app-settings.service.spec.ts` - Updated mock objects with new field

### Verification
```bash
cd booklore-ui
npm run build -- --configuration production
```

---

## Phase 7: Integration Testing and Deployment

### Objective
Test the complete feature end-to-end and deploy to production.

### Prerequisites
- Phase 5 and 6 completed
- CI/CD pipeline working

### Tasks

#### 7.1 Local Integration Testing
Test the complete flow locally:

1. Build the application:
   ```bash
   cd booklore-api && ./gradlew build
   cd booklore-ui && npm run build
   ```

2. Run locally with Docker Compose (if available):
   ```bash
   docker-compose -f dev.docker-compose.yml up
   ```

3. Test scenarios:
   - [ ] Enable auto-import in settings
   - [ ] Drop a book with good metadata (ISBN in filename) -> should auto-import
   - [ ] Drop a book without metadata -> should remain for review
   - [ ] Disable auto-import
   - [ ] Drop a book with good metadata -> should remain for review

#### 7.2 Commit and Push Changes
```bash
# Create feature branch
git checkout -b feature/auto-import-bookdrop

# Add all changes
git add .

# Commit with descriptive message
git commit -m "Add auto-import feature for bookdrop

- Add database setting for auto_import_enabled
- Modify bookdrop processing to auto-import when metadata is found
- Add REST endpoint for setting management
- Add settings toggle in frontend UI
- Add unit tests for new functionality

When enabled, books dropped in the bookdrop folder will be automatically
imported if metadata (title and authors) is successfully found from
Google Books or Open Library. Books without complete metadata will
still require manual review.

Co-Authored-By: Claude Opus 4.5 <noreply@anthropic.com>"

# Push to Gitea for CI/CD
git push gitea feature/auto-import-bookdrop

# Create PR in Gitea (or push to develop for direct deploy)
```

#### 7.3 Monitor CI/CD Build
1. Watch the Gitea Actions build at: https://git.taranasus.xyz/taranasus/booklore/actions
2. Check for any build failures
3. Review build logs for warnings

#### 7.4 Merge and Deploy
```bash
# After review, merge to develop
git checkout develop
git merge feature/auto-import-bookdrop
git push gitea develop
```

#### 7.5 Verify Production Deployment
1. Wait for CI/CD to complete deployment
2. SSH to UNRAID and verify container is running:
   ```bash
   sshpass -p '?xFQxYDd$PRaPr#4' ssh root@jack.server "docker ps | grep booklore"
   ```
3. Access http://192.168.1.141:6060
4. Navigate to settings and verify auto-import toggle is present
5. Test the feature with a real book

#### 7.6 Production Testing
1. Enable auto-import in settings
2. Copy a test ebook to `/mnt/user/appdata/booklore/bookdrop/`
3. Wait for processing (check logs if needed)
4. Verify book was either:
   - Auto-imported to library (if metadata found)
   - Left in pending review (if metadata not found)

### Deliverables
- [ ] All integration tests pass locally
- [ ] Feature branch created and pushed
- [ ] CI/CD build successful
- [ ] Deployed to UNRAID
- [ ] Production testing successful
- [ ] Feature working as expected

### Verification
1. Check Gitea Actions for successful build
2. Verify container running on UNRAID
3. Verify application accessible
4. Verify auto-import toggle visible in settings
5. Verify auto-import functionality works correctly

---

## Phase 8: PR Preparation for Upstream

### Objective
Prepare the changes for submission as a Pull Request to the original booklore-app/booklore repository.

### Prerequisites
- Phase 7 completed (feature working in production)
- GitHub fork synced with upstream

### Tasks

#### 8.1 Sync Fork with Upstream
```bash
# Fetch latest from upstream
git fetch upstream

# Rebase feature branch onto latest upstream develop
git checkout feature/auto-import-bookdrop
git rebase upstream/develop

# Resolve any conflicts if necessary
```

#### 8.2 Clean Up Commits
```bash
# Interactive rebase to clean up commits
git rebase -i upstream/develop

# Squash related commits, improve commit messages
# Ensure each commit is atomic and well-documented
```

#### 8.3 Push to GitHub Fork
```bash
# Push to GitHub fork (origin)
git push origin feature/auto-import-bookdrop
```

#### 8.4 Create Pull Request
1. Navigate to https://github.com/taranasus/booklore
2. Click "Compare & pull request"
3. Select:
   - Base repository: booklore-app/booklore
   - Base: develop
   - Head repository: taranasus/booklore
   - Compare: feature/auto-import-bookdrop

4. Write PR description:

```markdown
## Summary
Add auto-import functionality for the BookDrop feature.

## Description
This PR adds a new setting that allows users to enable automatic import of books
from the BookDrop folder when metadata is successfully found. Books without
complete metadata (title and authors) will still require manual review.

## Changes
- Added `auto_import_enabled` setting to application settings
- Modified BookDrop processing to check for complete metadata
- Added auto-import logic that triggers when:
  - Auto-import is enabled in settings
  - Metadata (title + authors) was successfully found
- Added REST endpoint for managing the setting
- Added toggle switch in settings UI
- Added unit tests for new functionality

## Testing
- [x] Unit tests added and passing
- [x] Manual testing with various ebook types
- [x] Tested with books that have good metadata (ISBN in filename)
- [x] Tested with books that have poor/no metadata

## Screenshots
[Add screenshot of settings toggle]

## Related Issues
Addresses the need for a more automated BookDrop workflow.
```

#### 8.5 Address Review Feedback
- Monitor the PR for maintainer feedback
- Make requested changes
- Push updates to the same branch

### Deliverables
- [ ] Feature branch rebased on latest upstream
- [ ] Commits cleaned and well-documented
- [ ] PR created on GitHub
- [ ] PR description complete with context
- [ ] Ready for maintainer review

---

## Appendix A: Key File Locations (To Be Updated in Phase 4)

After Phase 4 exploration, update this section with actual file paths:

### Backend (booklore-api)
- Settings Entity: `booklore-api/src/main/java/com/booklore/???/Settings.java`
- Settings Service: `booklore-api/src/main/java/com/booklore/???/SettingsService.java`
- BookDrop Service: `booklore-api/src/main/java/com/booklore/???/BookDropService.java`
- Migrations: `booklore-api/src/main/resources/db/migration/`

### Frontend (booklore-ui)
- Settings Component: `booklore-ui/src/app/???/settings/`
- Settings Service: `booklore-ui/src/app/???/services/settings.service.ts`
- Settings Model: `booklore-ui/src/app/???/models/settings.model.ts`

---

## Appendix B: Troubleshooting

### CI/CD Build Failures

**Java version mismatch:**
```bash
# On runner, verify Java version
java -version
# Must be JDK 21

# If wrong version, set JAVA_HOME
export JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64
```

**Node.js build failures:**
```bash
# Clear npm cache
npm cache clean --force

# Remove node_modules and reinstall
rm -rf node_modules package-lock.json
npm install
```

**Docker build failures:**
```bash
# Check Docker disk space
docker system df

# Clean up old images
docker image prune -a
```

### Deployment Issues

**Container won't start:**
```bash
# Check logs
docker logs booklore

# Check for port conflicts
netstat -tlnp | grep 6060

# Verify database connection
docker exec booklore curl -v telnet://192.168.1.141:3306
```

**Application errors:**
```bash
# Check application logs
docker logs booklore --tail 100

# Check database
docker exec -it mariadb mysql -u booklore -p booklore
```

---

## Appendix C: Rollback Procedure

If the new version causes issues:

```bash
# SSH to UNRAID
sshpass -p '?xFQxYDd$PRaPr#4' ssh root@jack.server

# Stop custom container
docker stop booklore
docker rm booklore

# Pull and run original image
docker pull booklore/booklore:latest
docker run -d \
    --name booklore \
    --restart=unless-stopped \
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

# Verify rollback
docker ps | grep booklore
```

---

## Summary

| Phase | Description | Dependencies | Estimated Complexity |
|-------|-------------|--------------|---------------------|
| 1 | Repository Setup | None | Low |
| 2 | CI/CD Runner Prep | Phase 1 | Low |
| 3 | Gitea Actions Setup | Phase 1, 2 | Medium |
| 4 | Codebase Analysis | Phase 1 | Medium |
| 5 | Backend Implementation | Phase 4 | High |
| 6 | Frontend Implementation | Phase 5 | Medium |
| 7 | Integration & Deployment | Phase 5, 6 | Medium |
| 8 | Upstream PR | Phase 7 | Low |

Each phase should be assigned to an agent with clear start/end criteria. Phases 5 and 6 can potentially run in parallel if the API contract is defined upfront.
