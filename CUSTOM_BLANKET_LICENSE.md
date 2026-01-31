# Custom Blanket License Implementation

## Overview

This document details all custom modifications made to IOMAD's blanket license (type 4) behavior. These changes are **NOT upstream IOMAD** and represent custom functionality.

### Custom Behavior vs Upstream IOMAD

| Aspect | Upstream IOMAD | Custom Implementation |
|--------|----------------|----------------------|
| User allocation | Admin must allocate users per-course | Admin allocates users once to entire license |
| License counting | Counts course allocations | Counts unique users only |
| New courses | Require admin re-allocation | Auto-available to allocated users |
| Self-enrollment | May require `company_course_options` | Works for any course in license |

---

## Files Modified

| File | Purpose |
|------|---------|
| `blocks/iomad_company_admin/classes/forms/company_license_users_form.php` | Admin user allocation check |
| `blocks/iomad_company_admin/classes/tables/company_license_table.php` | License usage display |
| `blocks/iomad_company_admin/lib/course_selectors.php` | Admin course selector |
| `enrol/license/lib.php` | Self-enrollment check & hook |
| `enrol/license/externallib.php` | External API enrollment |
| `local/iomad/lib/company.php` | License usage calculation |
| `blocks/mycourses/classes/helper.php` | Student course visibility |

---

## Detailed Changes

### 1. Admin User Allocation Check

**File:** `blocks/iomad_company_admin/classes/forms/company_license_users_form.php`

**Purpose:** When allocating users to a blanket license, count only unique NEW users rather than user×course combinations.

**Location:** `process()` method, lines ~314-326

**Change:**
```php
// For blanket licenses (type 4), count unique new users only.
if ($this->license->type == 4) {
    $newusers = 0;
    foreach ($userstoassign as $user) {
        if (!$DB->record_exists('companylicense_users',
                ['licenseid' => $this->licenseid, 'userid' => $user->id])) {
            $newusers++;
        }
    }
    $required = $newusers;
} else {
    $required = count($userstoassign) * count($courses);
}
```

**Why:** In upstream IOMAD, allocating a user to a 10-course blanket license would consume 10 license slots. With this change, it consumes only 1 slot per unique user.

---

### 2. License Usage Calculation

**File:** `local/iomad/lib/company.php`

**Function:** `update_license_usage()` at line ~3079

**Purpose:** Calculate license usage by counting unique users instead of total allocations.

**Change:**
```php
// For blanket licenses (type 4), count unique users only.
if ($license && $license->type == 4) {
    $usertotal = $DB->count_records_sql(
        "SELECT COUNT(DISTINCT userid) FROM {companylicense_users} WHERE licenseid = :licenseid",
        ['licenseid' => $licenseid]
    );
} else if ($userusage = $DB->get_records_sql("SELECT count(id) AS total
                                              FROM {companylicense_users}
                                              WHERE licenseid = :licenseid",
                                             ['licenseid' => $licenseid])) {
    $usertotal = $userusage[0]->total;
} else {
    $usertotal = 0;
}
```

**Why:** Ensures the `used` field on blanket licenses reflects unique users, matching the allocation check logic.

---

### 3. License Usage Display

**File:** `blocks/iomad_company_admin/classes/tables/company_license_table.php`

**Function:** `col_used()` at line ~161

**Purpose:** Display blanket license usage with "users" suffix for clarity.

**Change:**
```php
public function col_used($row) {
    global $DB, $output;

    $licensecourses = $DB->get_records('companylicense_courses', array('licenseid' => $row->id));

    // Deal with allocation numbers if a program.
    if (!empty($row->program)) {
        return $row->used / count($licensecourses);
    } else if ($row->type == 4) {
        // Blanket licenses count unique users.
        return $row->used . ' ' . get_string('users');
    } else {
        return $row->used;
    }
}
```

**Why:** Makes it visually clear to admins that blanket license counts represent users, not course allocations.

---

### 4. Self-Enrollment Check

**File:** `enrol/license/lib.php`

**Function:** `can_license_enrol()` at line ~308

**Purpose:** Allow users allocated to a blanket license to self-enroll in any course within that license.

**Change:** Added blanket license fallback query when no direct license allocation exists:

```php
if (!$license = $DB->get_record_sql($sql, ['userid' => $USER->id, 'courseid' => $instance->courseid])) {
    $blanketsql = "SELECT * FROM {companylicense} cl
                   JOIN {companylicense_courses} clc ON (cl.id = clc.licenseid)
                   WHERE clc.courseid = :courseid
                   AND cl.companyid = :companyid
                   AND cl.startdate < :startdate
                   AND cl.expirydate > :expirydate
                   AND cl.type = 4
                   AND (cl.used < cl.allocation
                        OR EXISTS (SELECT 1 FROM {companylicense_users} clu
                                   WHERE clu.licenseid = cl.id AND clu.userid = :userid))";
    if (!$license = $DB->get_record_sql($blanketsql, [
                                                        'courseid' => $instance->courseid,
                                                        'companyid' => $companyid,
                                                        'startdate' => time(),
                                                        'expirydate' => time(),
                                                        'userid' => $USER->id,
                                                     ])) {
        return get_string('nolicenseinformationfound', 'enrol_license');
    }
}
```

**Key Logic:**
- `cl.type = 4` - Only blanket licenses
- `cl.used < cl.allocation` - License has capacity, OR
- `EXISTS (SELECT 1 FROM {companylicense_users} clu WHERE clu.licenseid = cl.id AND clu.userid = :userid)` - User is already allocated

**Why:** Allows users who are allocated to a blanket license to enroll in ANY course within that license, not just courses they were explicitly allocated to.

---

### 5. Self-Enrollment Hook (Course Page)

**File:** `enrol/license/lib.php`

**Function:** `enrol_page_hook()` at line ~419

**Purpose:** Handle the actual enrollment process when a user clicks "Enrol" on a course page.

**Change:** Same blanket license query pattern as `can_license_enrol()`, plus automatic license allocation:

```php
// If we are a blanket license we need to allocate the license at this time.
if ($license->type == 4) {
    $issuedate = time();
    $userlicense = (object) [
                                'licenseid' => $license->id,
                                'userid' => $USER->id,
                                'licensecourseid' => $instance->courseid,
                                'issuedate' => $issuedate,
                                'isusing' => 1,
                                'type' => $license->type,
                            ];
    $userlicense->id = $DB->insert_record('companylicense_users', $userlicense);

    // Create an event.
    $eventother = [
                    'licenseid' => $license->id,
                    'issuedate' => $issuedate,
                    'duedate' => $issuedate,
                    'noemail' => true,
                  ];
    $event = block_iomad_company_admin\event\user_license_assigned::create([...]);
    $event->trigger();
}
```

**Why:** Creates the `companylicense_users` record on-the-fly when a blanket license user enrolls, rather than requiring admin pre-allocation.

---

### 6. External API Enrollment

**File:** `enrol/license/externallib.php`

**Function:** `enrol_user()` at line ~132

**Purpose:** Allow mobile apps and external integrations to enroll users via blanket licenses.

**Change:** Same blanket license query and auto-allocation pattern:

```php
if (!$license = $DB->get_record_sql($sql, ['userid' => $USER->id,
                                           'courseid' => $course->id])) {
    // Set the companyid.
    $companyid = iomad::get_my_companyid(context_system::instance(), false);

    $blanketsql = "SELECT cl.* FROM {companylicense} cl
                   JOIN {companylicense_courses} clc ON (cl.id = clc.licenseid)
                   WHERE clc.courseid = :courseid
                   AND cl.companyid = :companyid
                   AND cl.startdate < :startdate
                   AND cl.expirydate > :expirydate
                   AND cl.type = 4
                   AND (cl.used < cl.allocation
                        OR EXISTS (SELECT 1 FROM {companylicense_users} clu
                                   WHERE clu.licenseid = cl.id AND clu.userid = :userid))";
    $license = $DB->get_record_sql($blanketsql, [...]);
}
```

**Why:** Ensures API-based enrollment (mobile app, integrations) works the same as web-based enrollment.

---

### 7. Admin Course Selector (License Capacity Check)

**File:** `blocks/iomad_company_admin/lib/course_selectors.php`

**Class:** `potential_user_license_course_selector` at line ~1231

**Purpose:** Allow admins to allocate courses to users who are already allocated to a blanket license, even when the license appears "full".

**Change:**
```php
AND (cl.used < cl.allocation
     OR EXISTS (SELECT 1 FROM {companylicense_users} clu2
                WHERE clu2.licenseid = cl.id AND clu2.userid = :existsuserid))
```

**Why:** For blanket licenses, if a user is already allocated, they should be able to access any course in the license without consuming additional slots. This `OR EXISTS` clause ensures the course selector shows available courses even when `used >= allocation`, as long as the user already has an allocation record.

---

### 8. Student Course Visibility

**File:** `blocks/mycourses/classes/helper.php`

**Function:** `get_my_available()` at line ~275

**Purpose:** Show ALL courses in a blanket license to allocated users, including courses added AFTER user allocation.

**Problem:** The original SQL had `cca.companyid = :companyid` in the WHERE clause, which turns the LEFT JOIN into an effective INNER JOIN:

```php
// BEFORE (Bug)
$licensecourses = $DB->get_records_sql("SELECT c.id,
                                        c.id AS courseid,
                                        c.fullname AS coursefullname,
                                        c.summary AS coursesummary,
                                        cca.mandatory
                                        FROM {course} c
                                        JOIN {companylicense_courses} clc on (c.id = clc.courseid)
                                        LEFT JOIN {company_course_options} cca ON (
                                            c.id = cca.courseid
                                            AND clc.courseid = cca.courseid)
                                        WHERE clc.licenseid = :licenseid
                                        $inprogresssql
                                        $mandatorysql
                                        AND cca.companyid = :companyid",  // <-- BUG HERE
                                       ['licenseid' => $blanketlicense->id,
                                        'companyid' => $companyid]);
```

**Fix:** Move the company check into the ON clause:

```php
// AFTER (Fixed)
$licensecourses = $DB->get_records_sql("SELECT c.id,
                                        c.id AS courseid,
                                        c.fullname AS coursefullname,
                                        c.summary AS coursesummary,
                                        cca.mandatory
                                        FROM {course} c
                                        JOIN {companylicense_courses} clc on (c.id = clc.courseid)
                                        LEFT JOIN {company_course_options} cca ON (
                                            c.id = cca.courseid
                                            AND cca.companyid = :companyid)
                                        WHERE clc.licenseid = :licenseid
                                        $inprogresssql
                                        $mandatorysql",
                                       ['licenseid' => $blanketlicense->id,
                                        'companyid' => $companyid]);
```

**Why:**
- With the bug, courses without a `company_course_options` entry would return NULL for `cca.companyid`, which fails the WHERE condition
- The fix ensures courses are returned regardless of whether they have a `company_course_options` entry
- The `mandatory` field will be NULL for courses without options, which is acceptable

---

## Database Tables Reference

### `companylicense`
| Field | Type | Description |
|-------|------|-------------|
| id | int | Primary key |
| companyid | int | Company this license belongs to |
| type | int | 0=standard, 1=reusable, 2=educator, 3=educator reusable, **4=blanket** |
| allocation | int | Total slots available |
| used | int | Slots currently used (for blanket: unique users) |
| startdate | int | License valid from |
| expirydate | int | License valid until |

### `companylicense_courses`
| Field | Type | Description |
|-------|------|-------------|
| id | int | Primary key |
| licenseid | int | FK to companylicense |
| courseid | int | FK to course |

### `companylicense_users`
| Field | Type | Description |
|-------|------|-------------|
| id | int | Primary key |
| licenseid | int | FK to companylicense |
| userid | int | FK to user |
| licensecourseid | int | FK to course |
| isusing | int | 0=allocated, 1=enrolled |
| issuedate | int | When allocated |
| timecompleted | int | When completed (NULL if not) |

### `company_course_options`
| Field | Type | Description |
|-------|------|-------------|
| id | int | Primary key |
| companyid | int | FK to company |
| courseid | int | FK to course |
| mandatory | int | Whether course is mandatory |

---

## Testing Checklist

### Admin Allocation
- [ ] Create blanket license with 5 slots and 10 courses
- [ ] Allocate User A - should show "1 users" used
- [ ] Allocate User B - should show "2 users" used
- [ ] Verify both users have access to all 10 courses

### Self-Enrollment
- [ ] Log in as allocated user
- [ ] Navigate to a course in the blanket license
- [ ] Click "Enrol me"
- [ ] Verify enrollment succeeds
- [ ] Verify `companylicense_users` record created

### New Course Visibility
- [ ] Create blanket license with courses A, B
- [ ] Allocate User to the license
- [ ] User can see courses A, B
- [ ] Admin adds course C to the license
- [ ] **User refreshes and can now see course C

### API Enrollment
- [ ] Use external API to enroll user in blanket license course
- [ ] Verify enrollment succeeds
- [ ] Verify license allocation created

---

## Commit History

1. **Admin allocation check** - `company_license_users_form.php`
2. **License usage calculation** - `company.php`
3. **Display enhancement** - `company_license_table.php`
4. **Self-enrollment check** - `enrol/license/lib.php`
5. **Self-enrollment hook** - `enrol/license/lib.php`
6. **External API** - `enrol/license/externallib.php`
7. **Admin course selector (license capacity)** - `course_selectors.php`
8. **Student course visibility** - `blocks/mycourses/classes/helper.php`

---

## Upstream Considerations

If submitting to upstream IOMAD:
1. These changes modify fundamental blanket license behavior
2. Existing installations may have blanket licenses with different expectations
3. Migration script may be needed to recalculate `used` field for existing blanket licenses
4. Consider making this a new license type rather than default behavior
