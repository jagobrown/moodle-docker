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
