# Booklore Auto-Import Feature - Technical Specification

This document contains the technical analysis of the Booklore codebase for implementing the auto-import feature from bookdrop.

---

## 1. Bookdrop Related Source Files

### Backend (booklore-api/src/main/java/com/adityachandel/booklore)

| File | Purpose |
|------|---------|
| `service/bookdrop/BookDropService.java` | Main service for import operations, file movement, and finalization |
| `service/bookdrop/BookdropMonitoringService.java` | File system watcher using Java WatchService API |
| `service/bookdrop/BookdropEventHandlerService.java` | Processes file events, triggers metadata extraction |
| `service/bookdrop/BookdropMetadataService.java` | Extracts and fetches metadata for dropped files |
| `controller/BookdropFileController.java` | REST endpoints for bookdrop operations |
| `model/entity/BookdropFileEntity.java` | JPA entity for bookdrop files (DB table: bookdrop_file) |
| `model/dto/settings/AppSettings.java` | Settings DTO including metadataDownloadOnBookdrop |
| `model/dto/settings/AppSettingKey.java` | Enum for all application settings keys |
| `service/appsettings/AppSettingService.java` | Service for managing application settings |

### Frontend (booklore-ui/src/app)

| File | Purpose |
|------|---------|
| `features/bookdrop/service/bookdrop.service.ts` | API service for bookdrop operations |
| `features/bookdrop/service/bookdrop-file-api.service.ts` | Notification API for file counts |
| `features/settings/metadata-settings/metadata-settings-component.ts` | Settings UI including metadata toggle |

---

## 2. Current Bookdrop Data Flow

```
                                    ┌─────────────────────────────────┐
                                    │  File dropped in /bookdrop      │
                                    └───────────────┬─────────────────┘
                                                    │
                                                    ▼
                            ┌───────────────────────────────────────────┐
                            │  BookdropMonitoringService                │
                            │  - WatchService detects ENTRY_CREATE      │
                            │  - Validates file extension                │
                            │  - Filters temp/hidden files               │
                            └───────────────────┬───────────────────────┘
                                                │
                                                ▼
                            ┌───────────────────────────────────────────┐
                            │  BookdropEventHandlerService.processFile()│
                            │  - Creates BookdropFileEntity             │
                            │  - Sets status = PENDING_REVIEW           │
                            └───────────────────┬───────────────────────┘
                                                │
                    ┌───────────────────────────┴───────────────────────────┐
                    │                                                       │
                    ▼                                                       ▼
    ┌───────────────────────────────┐               ┌───────────────────────────────┐
    │ metadataDownloadOnBookdrop    │               │ metadataDownloadOnBookdrop    │
    │ = TRUE                        │               │ = FALSE                       │
    └───────────────┬───────────────┘               └───────────────┬───────────────┘
                    │                                               │
                    ▼                                               ▼
    ┌───────────────────────────────┐               ┌───────────────────────────────┐
    │ attachInitialMetadata()       │               │ attachInitialMetadata() only  │
    │ + attachFetchedMetadata()     │               │ (embedded metadata only)      │
    └───────────────┬───────────────┘               └───────────────┬───────────────┘
                    │                                               │
                    └───────────────────────┬───────────────────────┘
                                            │
                                            ▼
                            ┌───────────────────────────────────────────┐
                            │  File appears in UI for manual review     │
                            │  Status: PENDING_REVIEW                   │
                            └───────────────────┬───────────────────────┘
                                                │
                                            USER ACTION
                                                │
                                                ▼
                            ┌───────────────────────────────────────────┐
                            │  BookDropService.finalizeImport()         │
                            │  1. Pause monitoring                      │
                            │  2. Move files to library                 │
                            │  3. Create BookEntity                     │
                            │  4. Update metadata                       │
                            │  5. Trigger Kobo sync                     │
                            │  6. Cleanup bookdrop data                 │
                            │  7. Resume monitoring                     │
                            └───────────────────────────────────────────┘
```

---

## 3. Key Classes and Methods

### BookdropEventHandlerService.java (Primary Hook Point)

```java
// Line 74-160: processFile() - Main event handler
private void processFile(Path filePath, WatchEvent.Kind<?> kind) {
    // ...
    if (kind == StandardWatchEventKinds.ENTRY_CREATE) {
        // Line 83-86: Create entity with PENDING_REVIEW status
        BookdropFileEntity entity = BookdropFileEntity.builder()
            .filePath(filePath.toString())
            .fileName(filePath.getFileName().toString())
            .status(BookdropFileStatus.PENDING_REVIEW)
            .build();

        // Line 123: Check if automatic metadata fetch is enabled
        if (appSettingService.getAppSettings().isMetadataDownloadOnBookdrop()) {
            bookdropMetadataService.attachInitialMetadata(entity);
            bookdropMetadataService.attachFetchedMetadata(entity);  // <-- External API calls
        } else {
            bookdropMetadataService.attachInitialMetadata(entity);
        }

        // Line 143-148: Send notification to UI
        // *** AUTO-IMPORT LOGIC SHOULD BE INSERTED HERE ***
    }
}
```

### BookdropMetadataService.java

```java
// Line 55-88: attachFetchedMetadata() - Fetches from Google Books, Open Library
public void attachFetchedMetadata(BookdropFileEntity entity) {
    BookMetadata originalMetadata = /* extracted from file */;

    // Line 69: Calls external metadata providers
    BookMetadata fetchedMetadata = metadataRefreshService.fetchMetadataForBook(
        originalMetadata.getTitle(),
        originalMetadata.getAuthors(),
        originalMetadata.getIsbn10(),
        originalMetadata.getIsbn13(),
        // ...
    );

    // Line 77-85: Save fetched metadata
    entity.setFetchedMetadata(objectMapper.writeValueAsString(fetchedMetadata));
    entity.setStatus(BookdropFileStatus.PENDING_REVIEW);
    bookdropFileRepository.save(entity);
}
```

### BookDropService.java

```java
// Line 108-116: finalizeImport() - Called when user manually imports
public void finalizeImport(BookdropFinalizeRequest request) {
    processFileChunks(request.getProcessList(), true);
}

// Line 319-343: processFile() - Process individual file
private boolean processFile(BookdropFileProcessRequest processRequest, boolean deleteOnSuccess) {
    BookdropFileProcessingContext ctx = prepareFileProcessingContext(processRequest);
    moveFile(ctx.getOriginalPath(), ctx.getNewPath(), ctx, deleteOnSuccess);
    return true;
}

// Line 454-492: processMovedFile() - After file is moved
private void processMovedFile(BookdropFileProcessingContext ctx) {
    // Add to library
    BookFileProcessor processor = bookFileProcessorRegistry.getProcessor(fileType);
    BookEntity bookEntity = processor.processFile(library, ctx.getNewPath());

    // Update metadata
    metadataRefreshService.updateBookMetadata(bookEntity.getId(), /* metadata */);

    // Kobo integration
    koboAutoShelfService.addToShelf(/* ... */);

    // Cleanup
    cleanupBookdropData(ctx.getBookdropFile());
}
```

### AppSettings.java

```java
// Line 25: Existing bookdrop setting
private boolean metadataDownloadOnBookdrop;

// NEW FIELDS TO ADD:
private boolean autoImportEnabled;
private int autoImportMinConfidence;  // Optional: confidence threshold
private Long autoImportDefaultLibraryId;  // Optional: target library
```

### AppSettingKey.java

```java
// Line 27: Existing setting
METADATA_DOWNLOAD_ON_BOOKDROP("metadata_download_on_bookdrop", false, false,
    List.of(Permission.ADMIN, Permission.MANAGE_METADATA_CONFIG)),

// NEW SETTINGS TO ADD:
BOOKDROP_AUTO_IMPORT_ENABLED("bookdrop_auto_import_enabled", false, false,
    List.of(Permission.ADMIN, Permission.MANAGE_METADATA_CONFIG)),
```

---

## 4. Database Schema

### Existing: bookdrop_file table (Migration V38)

```sql
CREATE TABLE bookdrop_file (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    file_path TEXT NOT NULL UNIQUE,
    file_name VARCHAR(512) NOT NULL,
    file_size BIGINT,
    status ENUM('PENDING_REVIEW', 'FINALIZED') DEFAULT 'PENDING_REVIEW',
    original_metadata JSON,
    fetched_metadata JSON,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

### Existing: app_setting table

```sql
-- Settings stored as key-value pairs
CREATE TABLE app_setting (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    setting_key VARCHAR(255) NOT NULL UNIQUE,
    setting_value TEXT,
    -- ...
);
```

### New Setting (No Migration Needed)

The app_setting table uses key-value pairs, so no schema change is required. The new setting will be created automatically via `AppSettingService.getOrCreateSetting()` when first accessed.

---

## 5. Auto-Import Implementation Plan

### 5.1 Backend Changes

#### New Setting in AppSettingKey.java

```java
BOOKDROP_AUTO_IMPORT_ENABLED(
    "bookdrop_auto_import_enabled",
    false,  // not user-specific
    false,  // not cached per-user
    List.of(Permission.ADMIN, Permission.MANAGE_METADATA_CONFIG)
),
```

#### Update AppSettings.java

```java
@Builder.Default
private boolean autoImportEnabled = false;
```

#### Update AppSettingService.java

```java
// In buildAppSettings() method, add:
builder.autoImportEnabled(Boolean.parseBoolean(
    settingPersistenceHelper.getOrCreateSetting(
        AppSettingKey.BOOKDROP_AUTO_IMPORT_ENABLED,
        "false"
    )
));
```

#### Create BookdropAutoImportService.java

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class BookdropAutoImportService {
    private final AppSettingService appSettingService;
    private final BookDropService bookDropService;
    private final LibraryRepository libraryRepository;

    /**
     * Checks if a file should be auto-imported and performs the import.
     * @return true if auto-imported, false if left for manual review
     */
    public boolean attemptAutoImport(BookdropFileEntity entity) {
        // Check if auto-import is enabled
        if (!appSettingService.getAppSettings().isAutoImportEnabled()) {
            return false;
        }

        // Check if metadata is sufficient for auto-import
        if (!hasValidMetadata(entity)) {
            log.debug("Auto-import skipped for {}: insufficient metadata", entity.getFileName());
            return false;
        }

        // Get default library (first available or user-configured)
        Library targetLibrary = getDefaultLibrary();
        if (targetLibrary == null) {
            log.warn("Auto-import skipped: no library available");
            return false;
        }

        // Perform auto-import
        try {
            BookdropFileProcessRequest request = buildAutoImportRequest(entity, targetLibrary);
            bookDropService.processFileForAutoImport(request);
            log.info("Auto-imported: {} to library {}", entity.getFileName(), targetLibrary.getName());
            return true;
        } catch (Exception e) {
            log.error("Auto-import failed for {}: {}", entity.getFileName(), e.getMessage());
            return false;
        }
    }

    private boolean hasValidMetadata(BookdropFileEntity entity) {
        // Check if we have fetched metadata with title and at least one author
        if (entity.getFetchedMetadata() == null || entity.getFetchedMetadata().isEmpty()) {
            return false;
        }

        try {
            BookMetadata metadata = objectMapper.readValue(
                entity.getFetchedMetadata(),
                BookMetadata.class
            );

            return metadata.getTitle() != null
                && !metadata.getTitle().isBlank()
                && metadata.getAuthors() != null
                && !metadata.getAuthors().isEmpty();
        } catch (Exception e) {
            return false;
        }
    }

    private Library getDefaultLibrary() {
        // Return first library that accepts this file type
        return libraryRepository.findAll().stream()
            .filter(Library::isEnabled)
            .findFirst()
            .orElse(null);
    }
}
```

#### Modify BookdropEventHandlerService.java

```java
// Add injection
private final BookdropAutoImportService bookdropAutoImportService;

// In processFile() after metadata is attached (around line 143):
if (kind == StandardWatchEventKinds.ENTRY_CREATE) {
    // ... existing code to create entity and attach metadata ...

    // NEW: Attempt auto-import if enabled and metadata is valid
    boolean autoImported = bookdropAutoImportService.attemptAutoImport(entity);

    if (!autoImported) {
        // Send notification for manual review (existing behavior)
        notificationService.notifyBookdropUpdate(entity);
    }
}
```

### 5.2 Frontend Changes

#### Update metadata-settings-component.ts

```typescript
// Add property
autoImportEnabled = false;

// Add handler
onAutoImportToggle(event: MatSlideToggleChange): void {
  this.settingsHelper.saveSetting(
    AppSettingKey.BOOKDROP_AUTO_IMPORT_ENABLED,
    event.checked.toString()
  ).subscribe({
    next: () => this.toastr.success(
      event.checked ? 'Auto-import enabled' : 'Auto-import disabled'
    ),
    error: () => {
      this.toastr.error('Failed to update setting');
      event.source.checked = !event.checked;
    }
  });
}

// Load in ngOnInit
this.appSettings$.subscribe(settings => {
  this.autoImportEnabled = settings.autoImportEnabled;
});
```

#### Update metadata-settings-component.html

```html
<div class="setting-row">
  <div class="setting-info">
    <h4>Auto-Import from BookDrop</h4>
    <p>Automatically import books when metadata is successfully found from external sources.
       Books without complete metadata (title and authors) will remain for manual review.</p>
  </div>
  <mat-slide-toggle
    [checked]="autoImportEnabled"
    (change)="onAutoImportToggle($event)">
  </mat-slide-toggle>
</div>
```

#### Update AppSettingKey enum (TypeScript)

```typescript
// In app-setting-key.ts
export enum AppSettingKey {
  // ... existing keys ...
  BOOKDROP_AUTO_IMPORT_ENABLED = 'bookdrop_auto_import_enabled',
}
```

---

## 6. Critical Technical Considerations

### 6.1 Monitoring Synchronization

The auto-import must properly pause/resume library monitoring to prevent race conditions:

```java
// In BookdropAutoImportService or BookDropService
public void processFileForAutoImport(BookdropFileProcessRequest request) {
    bookdropMonitoringService.pauseMonitoring();
    try {
        // Unregister affected library from monitoring
        monitoringRegistrationService.unregister(targetLibrary);

        // Move and process file
        processFile(request, true);

        // Re-register library
        monitoringRegistrationService.register(targetLibrary);
    } finally {
        bookdropMonitoringService.resumeMonitoring();
    }
}
```

### 6.2 Thread Safety

The bookdrop event handler runs on a separate thread. Auto-import operations must be thread-safe:

- Use `@Transactional` appropriately
- Ensure file operations are atomic
- Handle concurrent access to settings cache

### 6.3 Error Recovery

If auto-import fails:
1. File remains in bookdrop folder (no data loss)
2. Entity remains with `PENDING_REVIEW` status
3. User can still manually import
4. Error logged for debugging

### 6.4 Metadata Quality Criteria

For auto-import eligibility, require:
- **Title**: Non-null and non-blank
- **Authors**: At least one author present
- **Source**: Metadata came from external fetch (not just file extraction)

Optional future enhancements:
- Confidence score threshold
- ISBN verification
- Cover image requirement

---

## 7. Testing Strategy

### Unit Tests

```java
@Test
void shouldAutoImportWhenEnabledAndMetadataValid() {
    // Given: Auto-import enabled and valid metadata
    when(appSettingService.getAppSettings().isAutoImportEnabled()).thenReturn(true);
    BookdropFileEntity entity = createEntityWithValidFetchedMetadata();

    // When
    boolean result = autoImportService.attemptAutoImport(entity);

    // Then
    assertTrue(result);
    verify(bookDropService).processFileForAutoImport(any());
}

@Test
void shouldNotAutoImportWhenDisabled() {
    when(appSettingService.getAppSettings().isAutoImportEnabled()).thenReturn(false);
    BookdropFileEntity entity = createEntityWithValidFetchedMetadata();

    boolean result = autoImportService.attemptAutoImport(entity);

    assertFalse(result);
    verify(bookDropService, never()).processFileForAutoImport(any());
}

@Test
void shouldNotAutoImportWhenMetadataIncomplete() {
    when(appSettingService.getAppSettings().isAutoImportEnabled()).thenReturn(true);
    BookdropFileEntity entity = createEntityWithNoFetchedMetadata();

    boolean result = autoImportService.attemptAutoImport(entity);

    assertFalse(result);
    verify(bookDropService, never()).processFileForAutoImport(any());
}
```

### Integration Tests

1. Drop file with ISBN in filename → should auto-import
2. Drop file without metadata → should remain for review
3. Disable auto-import → all files should require review
4. Multiple files at once → all should be processed correctly

---

## 8. Summary of Changes

| Layer | File | Change Type |
|-------|------|-------------|
| Backend | `AppSettingKey.java` | Add `BOOKDROP_AUTO_IMPORT_ENABLED` |
| Backend | `AppSettings.java` | Add `autoImportEnabled` field |
| Backend | `AppSettingService.java` | Load new setting in `buildAppSettings()` |
| Backend | `BookdropAutoImportService.java` | **NEW FILE** - Auto-import logic |
| Backend | `BookdropEventHandlerService.java` | Call auto-import service after metadata |
| Backend | `BookDropService.java` | Add `processFileForAutoImport()` method |
| Frontend | `AppSettingKey` (enum) | Add `BOOKDROP_AUTO_IMPORT_ENABLED` |
| Frontend | `metadata-settings-component.ts` | Add toggle handler |
| Frontend | `metadata-settings-component.html` | Add toggle UI |

---

## Appendix: File Locations Summary

### Backend Key Files

- **Settings Entity**: `booklore-api/src/main/java/com/adityachandel/booklore/model/dto/settings/AppSettings.java`
- **Settings Keys**: `booklore-api/src/main/java/com/adityachandel/booklore/model/dto/settings/AppSettingKey.java`
- **Settings Service**: `booklore-api/src/main/java/com/adityachandel/booklore/service/appsettings/AppSettingService.java`
- **BookDrop Service**: `booklore-api/src/main/java/com/adityachandel/booklore/service/bookdrop/BookDropService.java`
- **Event Handler**: `booklore-api/src/main/java/com/adityachandel/booklore/service/bookdrop/BookdropEventHandlerService.java`
- **Metadata Service**: `booklore-api/src/main/java/com/adityachandel/booklore/service/bookdrop/BookdropMetadataService.java`
- **Monitoring Service**: `booklore-api/src/main/java/com/adityachandel/booklore/service/bookdrop/BookdropMonitoringService.java`
- **Entity**: `booklore-api/src/main/java/com/adityachandel/booklore/model/entity/BookdropFileEntity.java`

### Frontend Key Files

- **Settings Component**: `booklore-ui/src/app/features/settings/metadata-settings/metadata-settings-component.ts`
- **Settings Template**: `booklore-ui/src/app/features/settings/metadata-settings/metadata-settings-component.html`
- **Bookdrop Service**: `booklore-ui/src/app/features/bookdrop/service/bookdrop.service.ts`
