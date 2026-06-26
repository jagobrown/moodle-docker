# Moodle Tables for Course Sections

## Tables to READ

| Table | Purpose | Key columns to cache |
|---|---|---|
| `mdl_course` | Course metadata | `id`, `shortname`, `fullname`, `format` |
| `mdl_course_sections` | The section rows | `id`, `course`, `section` (order number), `name`, `summary`, `summaryformat`, `sequence`, `visible`, `availability`, `component`, `itemid`, `timemodified` |
| `mdl_course_format_options` | Format-specific section settings (e.g. `numsections`, collapsed state) | `courseid`, `format`, `sectionid`, `name`, `value` |
| `mdl_context` | Permission context for file resolution | `id`, `contextlevel=50` (course), `instanceid` = courseid |
| `mdl_files` | Embedded images in `summary` field | `contextid`, `component='course'`, `filearea='section'`, `itemid` = `course_sections.id` |

> The `sequence` column in `mdl_course_sections` is a comma-separated list of `mdl_course_modules.id` values — it defines which activities belong to the section and their order.

---

## Tables to UPDATE

| Table | When | Key columns |
|---|---|---|
| `mdl_course_sections` | Editing section name or description | `name`, `summary`, `summaryformat`, `visible`, `availability`, `timemodified` |
| `mdl_course_sections` | Reordering sections | `section` (the position integer) |
| `mdl_course_sections` | Moving a module into a section | `sequence` (append cmid) |
| `mdl_course_format_options` | Changing format-specific settings (e.g. collapsing) | `value` |
| `mdl_course` | If section count changes (older formats) | `cacherev` (Moodle invalidates its internal cache via this) |

---

## Tables to INSERT into

| Table | When | Notes |
|---|---|---|
| `mdl_course_sections` | Creating a new section | Must be unique on `(course, section)`. Always insert at end, then call `move_section_to()` internally. |
| `mdl_course_format_options` | Setting format options on the new section | `sectionid` = new section `id`, `courseid`, `format`, `name`, `value` |
| `mdl_files` | Uploading embedded images in `summary` | `component='course'`, `filearea='section'`, `itemid` = section `id`, `contextid` = course context |

---

## Section URL pattern

From the Moodle codebase (course/format/topics/lib.php):

```
/course/section.php?id={course_sections.id}       ← topics format: section on its own page
/course/view.php?id={course.id}&section={section}  ← fallback: section number as query param
```

The **stable key to cache and use in URLs** is `course_sections.id` (the database PK), not the `section` position integer (which changes when sections are reordered).

---

## Recommended caching strategy for your 3rd party app

```
local_cache.sections
├── id          → mdl_course_sections.id         (stable PK, use in URLs)
├── course_id   → mdl_course_sections.course
├── position    → mdl_course_sections.section     (volatile — changes on reorder)
├── name        → mdl_course_sections.name
├── summary     → mdl_course_sections.summary     (HTML, may contain @@PLUGINFILE@@ tokens)
├── summary_fmt → mdl_course_sections.summaryformat
├── visible     → mdl_course_sections.visible
├── url         → /course/section.php?id={id}
└── timemodified→ mdl_course_sections.timemodified  ← use this to detect changes for upsert
```

Use `timemodified` for change detection — poll `core_course_get_contents` (REST API) or query the table directly. The REST API function `core_course_get_contents` returns section `id`, `name`, `summary`, `summaryformat`, `section`, and `uservisible` all in one call, which maps directly to your cache fields.

---

## Preferred API endpoints (avoid direct DB writes)

| Operation | API function |
|---|---|
| Read sections | `core_course_get_contents` with `sectionid` or `sectionnumber` filter |
| Update section name/description | `core_courseformat_update_course` (state action `section_put`) |
| Create section | `core_courseformat_update_course` (state action `section_add`) |
| Toggle visibility | `core_course_edit_section` with `action=sectionhide/sectionshow` |

---

# Creating & Updating Course Modules (Text/Media Activities)

## Tables to READ

| Table | Purpose | Key columns |
|---|---|---|
| `mdl_modules` | Module type registry | `id`, `name` ('page', 'resource', 'label', 'folder', etc.) |
| `mdl_page` or `mdl_resource` | The actual content instance | `id`, `course`, `name`, `intro`, `introformat`, `content`, `contentformat` (page only) |
| `mdl_course_modules` | Links course section to module instance | `id`, `course`, `module`, `instance`, `section`, `visible`, `completion`, **`idnumber`** |
| `mdl_context` | Module permission context | `id`, `contextlevel=70` (module), `instanceid` = `course_modules.id` |
| `mdl_files` | Embedded media/images in `content` or `intro` | `contextid`, `component='mod_page'` or `'mod_resource'`, `filearea='content'` or `'intro'`, `itemid=0` |
| `mdl_course_sections` | The `sequence` field | Used to track which modules belong to section |
| `mdl_grade_items` (optional) | If module is gradeable | `itemmodule='page'` or `'resource'`, `iteminstance` = page/resource id |
| `mdl_course_modules_completion` (optional) | Completion tracking | `coursemoduleid`, `userid`, `completionstate` |

## Tables to INSERT into

| Table | When | Required fields |
|---|---|---|
| `mdl_course_modules` | Creating new activity instance | `course`, `module` (id from mdl_modules), `instance` (set to 0 initially, then updated after module instance created), `section`, `added=time()`, `visible=1`, **`idnumber`** (optional, for external ID mapping) |
| `mdl_page` or `mdl_resource` | Creating new activity instance | `course`, `name`, `intro`, `introformat`, `content` (page), `contentformat` (page), `timemodified` |
| `mdl_files` | Uploading embedded images/media | `contextid` (module context), `component='mod_page'` or `'mod_resource'`, `filearea='content'` or `'intro'`, `itemid=0`, `filename`, `filepath`, `mimetype`, `filesize`, `contenthash`, `timecreated`, `timemodified` |
| `mdl_context` | Module context (auto-created on module creation) | `contextlevel=70`, `instanceid` = new course_modules.id, `path`, `depth` |
| `mdl_grade_items` | If module has grading | `courseid`, `itemname`, `itemtype='mod'`, `itemmodule='page'`/`'resource'`, `iteminstance` = page/resource id |
| `mdl_tag_instance` | If tagging modules | `tagid`, `component='core'`, `itemtype='course_modules'`, `itemid` = course_modules.id, `tiuserid=0` |

## Tables to UPDATE

| Table | When | Key fields to modify |
|---|---|---|
| `mdl_course_modules` | After module instance created | `instance` (set to the new page/resource id), `visible`, `completion`, `completionview` |
| `mdl_course_sections` | Adding module to section | `sequence` (append course_modules.id to comma-separated list) |
| `mdl_page` or `mdl_resource` | Updating content | `name`, `intro`, `introformat`, `content`, `contentformat`, `timemodified` |
| `mdl_grade_items` | Updating grading rules | `grademax`, `grademin`, `gradepass`, `multfactor`, `plusfactor` |

## Critical Relationships

```
course_modules.module → mdl_modules.id       (which type: page=18, resource=12, etc.)
course_modules.instance → mdl_page.id        (the actual content row)
                       → mdl_resource.id

mdl_context.instanceid → course_modules.id   (module context for files)

mdl_files.contextid → mdl_context.id         (where files are stored)

course_sections.sequence → contains comma-separated course_modules.id values
```

## Module Type IDs (core modules)

Query `SELECT id, name FROM mdl_modules` to get these:

| Module | Typical ID | Table |
|---|---|---|
| Page | 18 | `mdl_page` |
| Resource | 12 | `mdl_resource` |
| Label | 5 | `mdl_label` |
| Forum | 9 | `mdl_forum` |
| Quiz | 16 | `mdl_quiz` |

## Recommended Caching Strategy for Text/Media Modules

```
local_cache.modules
├── course_modules
│   ├── id                 → mdl_course_modules.id (stable PK, use in URLs)
│   ├── course_id          → mdl_course_modules.course
│   ├── idnumber           → mdl_course_modules.idnumber (external ID for mapping to 3rd party system)
│   ├── section_id         → mdl_course_modules.section
│   ├── module_type        → mdl_modules.name ('page', 'resource')
│   ├── instance_id        → mdl_course_modules.instance (points to mdl_page.id)
│   ├── visible            → mdl_course_modules.visible
│   ├── completion         → mdl_course_modules.completion (0,1,2)
│   ├── added              → mdl_course_modules.added
│   └── url                → /course/modedit.php?update={id} or /mod/page/view.php?id={id}
│
├── content
│   ├── name               → mdl_page.name or mdl_resource.name
│   ├── intro              → mdl_page.intro or mdl_resource.intro
│   ├── intro_format       → mdl_page.introformat
│   ├── content            → mdl_page.content (text/HTML content)
│   ├── content_format     → mdl_page.contentformat
│   ├── timemodified       → mdl_page.timemodified
│   └── files              → file hashes from mdl_files for embedded media
│
└── metadata
    ├── completionview     → mdl_course_modules.completionview
    ├── completion_expect  → mdl_course_modules.completionexpected
    └── availability_json  → mdl_course_modules.availability (JSON restrictions)
```

Use `timemodified` from both `mdl_course_modules` and `mdl_page`/`mdl_resource` for change detection.

## Preferred API Endpoints for Creating/Updating Modules

| Operation | API function | Notes |
|---|---|---|
| Create page/resource | `core_courseformat_create_module` | State action: creates course_modules + page/resource instance |
| Update module content | `core_courseformat_update_course` (state action `cm_put`) | Updates module instance data |
| Update module visibility | `core_course_edit_module` with `action=hide/show` | Updates course_modules.visible |
| Delete module | `core_courseformat_update_course` (state action `cm_delete`) | Handles cleanup of all related data |
| Upload files to module | `core_courseformat_file_handlers` | Handles file upload to module context |

## Key Differences: Page vs Resource

| Aspect | Page | Resource |
|---|---|---|
| Content storage | `mdl_page.content` (HTML editor) | Typically file-based, can be URL |
| Best for | Rich formatted text, embedded media | PDFs, documents, external links |
| Intro field | Yes (`mdl_page.intro`) | Yes (`mdl_resource.intro`) |
| Completion tracking | ✓ Supported | ✓ Supported |
| Context filearea | `mod_page/content`, `mod_page/intro` | `mod_resource/intro`, legacy `mod_resource/content` |

## Creating a Module via API (Recommended Flow)

```
1. Call core_courseformat_create_module with:
   {
     "courseid": <course_id>,
     "sectionid": <course_sections.id>,
     "type": "page",           // or "resource"
     "name": "My Page",
     "intro": "<p>Description</p>",
     "introformat": 1,         // 1 = HTML
     "content": "<p>Main content with <img src='@@PLUGINFILE@@/image.png' /></p>",
     "contentformat": 1,
     "visible": 1,
     "completion": 2           // 0=off, 1=manual, 2=automatic
   }

2. Response returns:
   {
     "coursemodule": <new_course_modules.id>,
     "instanceid": <new_page.id>,
     "messages": [...]
   }

3. Then POST files to content area using contextid from step 2
   {
     "contextid": <module_context_id>,
     "itemid": 0,
     "component": "mod_page",
     "filearea": "content",
     "files": [...]
   }
```

## Change Detection for Upserts

Use these timestamps to detect changes:

- **Section changes**: `mdl_course_sections.timemodified`
- **Module metadata changes**: `mdl_course_modules.added`, plus check `mdl_course_sections.sequence` for reordering
- **Content changes**: `mdl_page.timemodified` or `mdl_resource.timemodified`
- **File changes**: `mdl_files.timemodified`

Poll `core_course_get_contents` with `cmid` filter to get combined module + section info in one call.

---

# Creating & Updating Text/Media Areas (Labels)

> **Note:** Labels are called "Text/media area" in the Moodle UI. They are the simplest content module type, ideal for embedding text, images, and media directly into a course section without a separate activity page.

## Tables to READ

| Table | Purpose | Key columns |
|---|---|---|
| `mdl_label` | The actual label content | `id`, `course`, `name`, `intro`, `introformat`, `timemodified` |
| `mdl_course_modules` | Links section to label instance | `id`, `course`, `module`, `instance`, `section`, `visible`, **`idnumber`** |
| `mdl_modules` | Module type registry | `id`, `name='label'` |
| `mdl_context` | Label permission context | `id`, `contextlevel=70`, `instanceid` = `course_modules.id` |
| `mdl_files` | Embedded images/media in label content | `contextid`, `component='mod_label'`, `filearea='intro'`, `itemid=0` |
| `mdl_course_sections` | The `sequence` field | Used to track which labels belong to section |

> **Key difference from Page/Resource:** Labels store their **main content in the `intro` field**, not in a separate `content` field. There is no `mdl_label.content` — the `intro` field IS the content.

## Tables to INSERT into

| Table | When | Required fields |
|---|---|---|
| `mdl_course_modules` | Creating new label | `course`, `module` (id from mdl_modules where name='label'), `instance=0` (updated after label created), `section`, `added=time()`, `visible=1`, **`idnumber`** (optional, for external ID mapping) |
| `mdl_label` | Creating new label | `course`, `name`, `intro` (the actual text/HTML content), `introformat` (0=plain, 1=HTML), `timemodified=time()` |
| `mdl_files` | Uploading embedded images/media to label | `contextid` (label context), `component='mod_label'`, `filearea='intro'`, `itemid=0`, `filename`, `filepath`, `mimetype`, `filesize`, `contenthash`, `timecreated`, `timemodified` |
| `mdl_context` | Label context (auto-created on module creation) | `contextlevel=70`, `instanceid` = new course_modules.id, `path`, `depth` |

## Tables to UPDATE

| Table | When | Key fields to modify |
|---|---|---|
| `mdl_label` | Editing label content | `name`, `intro` (the main content), `introformat`, `timemodified=time()` |
| `mdl_course_modules` | After label instance created | `instance` (set to new label id), `visible`, **`idnumber`** |
| `mdl_course_sections` | Adding label to section | `sequence` (append course_modules.id to comma-separated list) |

## Label-Specific Considerations

### No separate intro/content split
```
mdl_label.intro   ← This is the MAIN CONTENT (not a description)
mdl_label.name    ← The label title (optional, often blank or auto-hidden)
```

This differs from `mdl_page` and `mdl_resource`, which have:
```
mdl_page.intro      ← Description above the main content
mdl_page.content    ← The main content
```

### No grading, completion, or advanced settings
Labels are **display-only** and don't support:
- Grades or `mdl_grade_items`
- Completion tracking (`mdl_course_modules_completion`)
- Availability conditions
- Conditional access logic

This makes them lightweight and ideal for spacing, dividers, and informational text.

### File storage structure
```
Files in label are stored under:
  component='mod_label'
  filearea='intro'
  itemid=0

So the file path in Moodle will be:
  @@PLUGINFILE@@/image.png
  @@PLUGINFILE@@/video.mp4
```

## Recommended Caching Strategy for Labels

```
local_cache.labels
├── id                → mdl_course_modules.id (stable PK, use in URLs)
├── course_id         → mdl_course_modules.course
├── idnumber          → mdl_course_modules.idnumber (external ID for mapping to 3rd party system)
├── section_id        → mdl_course_modules.section
├── instance_id       → mdl_course_modules.instance (points to mdl_label.id)
├── visible           → mdl_course_modules.visible
├── label_id          → mdl_label.id
├── name              → mdl_label.name (label title, often empty)
├── content           → mdl_label.intro (the main text/HTML content)
├── content_format    → mdl_label.introformat (0=plain, 1=HTML)
├── timemodified      → mdl_label.timemodified
└── embedded_files    → file hashes from mdl_files with @@PLUGINFILE@@ tokens
```

**For change detection**, monitor:
- `mdl_label.timemodified` — content changes
- `mdl_course_modules.added` — position/section changes
- `mdl_course_sections.sequence` — reordering

## Preferred API Endpoints for Labels

| Operation | API function | Notes |
|---|---|---|
| Create label | `core_courseformat_create_module` | Type: `label`, only needs `courseid`, `sectionid`, `name`, `intro`, `introformat` |
| Update label | `core_courseformat_update_course` (state action `cm_put`) | Update `name`, `intro`, `introformat` |
| Toggle visibility | `core_course_edit_module` with `action=hide/show` | Updates course_modules.visible |
| Delete label | `core_courseformat_update_course` (state action `cm_delete`) | Cleans up files and module record |
| Upload embedded media | `core_courseformat_file_handlers` | Upload images/media to label intro filearea |

## Creating a Label via API (Minimal Example)

```json
{
  "courseid": 2,
  "sectionid": 5,
  "type": "label",
  "name": "Section introduction",
  "intro": "<h3>Welcome to this section</h3><p>This is a text/media area with <img src='@@PLUGINFILE@@/welcome.png' /> embedded image.</p>",
  "introformat": 1
}
```

**Response:**
```json
{
  "coursemodule": 42,
  "instanceid": 15,
  "messages": []
}
```

Then upload the image to the label's file area using the contextid derived from course_modules.id=42.

## Label vs Page vs Resource Comparison

| Feature | Label | Page | Resource |
|---|---|---|---|
| **Main content field** | `intro` | `content` | File-based |
| **Has dedicated content table** | Yes (`mdl_label`) | Yes (`mdl_page`) | Yes (`mdl_resource`) |
| **Separate intro/description** | No — intro IS content | Yes, has both | Yes, has intro only |
| **Gradeable** | ✗ No | ✗ No | ✗ No |
| **Completion tracking** | ✗ No | ✓ Yes | ✓ Yes |
| **Display options** | Limited (just visible/hidden) | Advanced (pop-up, etc.) | Advanced (embed, download, etc.) |
| **Best use case** | Text sections, embedded media, dividers | Formatted pages, books | Document distribution |
| **Complexity** | Minimal | Medium | Medium-High |

## SQL Query Examples

### Get all labels in a course
```sql
SELECT cm.id, cm.instance, l.name, l.intro, l.introformat, cs.section
FROM mdl_course_modules cm
JOIN mdl_modules m ON cm.module = m.id
JOIN mdl_label l ON cm.instance = l.id
JOIN mdl_course_sections cs ON cm.section = cs.id
WHERE cm.course = 2 AND m.name = 'label'
ORDER BY cs.section, cm.added;
```

### Get all files embedded in a label
```sql
SELECT f.filename, f.filepath, f.mimetype, f.filesize, f.contenthash
FROM mdl_files f
JOIN mdl_context ctx ON f.contextid = ctx.id
JOIN mdl_course_modules cm ON ctx.instanceid = cm.id
WHERE cm.id = 42 
  AND f.component = 'mod_label'
  AND f.filearea = 'intro'
  AND f.filename != '.'
ORDER BY f.filepath, f.filename;
```

### Check if a label has changed
```sql
SELECT cm.id, l.timemodified, cm.added
FROM mdl_course_modules cm
JOIN mdl_label l ON cm.instance = l.id
WHERE cm.id = 42;
```

---

# Using idnumber for External ID Mapping

The `idnumber` field is available on key Moodle tables to provide a direct link between Moodle records and your 3rd party cached data. In this Moodle version, activity-level mapping should use `mdl_course_modules.idnumber`.

## Tables with idnumber Support

| Table | Has idnumber? | Field type | Purpose |
|---|---|---|---|
| `mdl_course_sections` | ❌ No | — | Use `mdl_course_sections.id` (or store external section IDs in `mdl_course_format_options`) |
| `mdl_course_modules` | ✅ Yes | `VARCHAR(100)` | Map activities to external content items |
| `mdl_page` | ❌ No | — | Use `mdl_course_modules.idnumber` instead |
| `mdl_resource` | ❌ No | — | Use `mdl_course_modules.idnumber` instead |
| `mdl_label` | ❌ No | — | Use `mdl_course_modules.idnumber` instead |

## Recommended External ID Mapping Strategy

Instead of relying on Moodle's auto-increment `id` fields (which can be unpredictable across environments), use `idnumber` as a stable, human-readable reference:

### For Sections
```sql
-- mdl_course_sections has no idnumber in this version.
-- Use Moodle section id as canonical key.
SELECT id, course, section, name, timemodified
FROM mdl_course_sections
WHERE course = 2;
```

### For Course Modules (Activities)
```sql
INSERT INTO mdl_course_modules 
  (course, module, instance, section, visible, idnumber, added)
VALUES 
  (2, 18, 0, 5, 1, 'EXT_ACT_PAGE_001', UNIX_TIMESTAMP());

-- Update instance after creating page
UPDATE mdl_course_modules 
SET instance = 42 
WHERE idnumber = 'EXT_ACT_PAGE_001';

-- Later, lookup by idnumber
SELECT id, instance FROM mdl_course_modules 
WHERE course = 2 AND idnumber = 'EXT_ACT_PAGE_001';
```

## Local Cache Structure with idnumber Mapping

```javascript
local_cache.sections = {
  [section_42]: {
    moodle_id: 42,
    course_id: 2,
    section_num: 1,
    name: 'Introduction',
    content: '...',
    timemodified: 1703001600
  },
  [ext_id_002]: { ... }
}

local_cache.modules = {
  [ext_id_001]: {
    moodle_id: 15,
    idnumber: 'EXT_ACT_PAGE_001',
    course_module_id: 42,    // mdl_course_modules.id
    section_moodle_id: 42,
    content: '...',
    timemodified: 1703001600
  }
}
```

## Upsert Pattern Using idnumber

This pattern ensures reliable synchronization between systems:

```sql
-- Check if section exists by Moodle section id (idnumber is not available)
SELECT id INTO @section_id FROM mdl_course_sections 
WHERE course = 2 AND id = 42;

IF @section_id IS NULL THEN
  -- Create new section
  INSERT INTO mdl_course_sections 
    (course, section, name, summary, summaryformat, visible, timemodified)
  VALUES 
    (2, 1, 'Introduction', '<p>Welcome</p>', 1, 1, UNIX_TIMESTAMP());
  SET @section_id = LAST_INSERT_ID();
ELSE
  -- Update existing section
  UPDATE mdl_course_sections 
  SET name = 'Introduction', 
      summary = '<p>Welcome</p>', 
      timemodified = UNIX_TIMESTAMP()
  WHERE id = @section_id;
END IF;
```

## Best Practices for idnumber Values

- **Prefix by type**: Use prefixes like `EXT_ACT_` for activities to avoid collisions
- **Keep it stable**: Never change an idnumber once set — treat it as the "primary key" for external mapping
- **Uniqueness per course**: Enforce uniqueness within a course for `mdl_course_modules.idnumber`
- **Human-readable**: Use sequential IDs or slugs that align with your external system (e.g., `EXT_SEC_001`, `COURSE_2_MOD_42`)
- **Null as default**: Leave idnumber NULL for Moodle-created content that isn't mapped to external systems


## SQL Queries for External Sync

### Find all sections mapped to external system
```sql
SELECT id, course, section, name, timemodified
FROM mdl_course_sections
WHERE course = 2
ORDER BY section;
```

### Find all activities mapped to external system
```sql
SELECT cm.id, cm.idnumber, m.name AS module_type, cm.instance, cm.timemodified
FROM mdl_course_modules cm
JOIN mdl_modules m ON cm.module = m.id
WHERE cm.course = 2 AND cm.idnumber IS NOT NULL
ORDER BY cm.added;
```

### Detect changes since last sync
```sql
SELECT id, section, timemodified
FROM mdl_course_sections
WHERE course = 2 
  AND timemodified > UNIX_TIMESTAMP('2026-06-26 10:00:00')
UNION ALL
SELECT cm.id, cm.idnumber, cm.added AS timemodified
FROM mdl_course_modules cm
WHERE cm.course = 2 
  AND cm.idnumber IS NOT NULL 
  AND (cm.added > UNIX_TIMESTAMP('2026-06-26 10:00:00') 
       OR (SELECT timemodified FROM mdl_page WHERE id = cm.instance) > UNIX_TIMESTAMP('2026-06-26 10:00:00'))
ORDER BY timemodified DESC;
```
