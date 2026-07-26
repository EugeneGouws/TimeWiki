# Rollover — School Timetable Year Advancement Tool

## Purpose

Rollover is a C++ command-line utility that advances a school timetable from one academic year to the next. It automates the bulk-edit case: incrementing student grades, promoting lesson framework to new grades, and re-allocating teachers to lessons while detecting scheduling clashes.

**Core workflow:**
1. Load current year's timetable from MS Access database (`new2026.accdb`)
2. Increment all students and lessons to the next grade
3. Allocate teachers to lessons using constraint satisfaction (backtracking augmenting-path matching)
4. Detect and report three types of clashes: teacher double-booking, student conflicts, venue collisions
5. Emit diagnostic reports of unallocated lessons

## Data Model

Four core entity structs (all Qt-based, with minimal dependencies):

- **Student**: `id`, grade (`gr`), registration code (`reg`), names. Cap ≤500 students per build (Constants.h).
- **Teacher**: `id`, current load, list of subject codes (`sCodes`), list of grade-matched lessons, venue, names. Central to clash detection.
- **Lesson**: subject code (2-letter, e.g., "MA" for Math), grade, teacher assignment, bitset of enrolled students (efficient overlap detection), time slot, venue, allocated flag.
- **TimeSlot**: string id (e.g., "A5"), list of lessons in slot, bitset aggregating all students, list of teachers.

**Data organization** (SchoolData struct):
- `sMap`: students by id
- `tMap`: teachers by id
- `lMap`: lessons by `raw_grade_timeSlot`; `nLMap` (new lessons after rollover); `OOTMap` (out-of-time lessons)
- `tSurToId`: helper map for teacher lookup by subject+surname
- `timeSlots`: indexed by slot id (e.g., "A5") — grouping lessons that occupy same period

## Clash Detection

**ClashDetector** (clashdetector.cpp) operates post-allocation:

1. **mapTimeSlots()**: aggregates all lessons in a time slot, merging student bitsets and teacher lists for quick overlap checking.

2. **clashes()** reports three conflict types:
   - **Teacher clash**: same teacher scheduled in two lessons within one slot
   - **Student clash**: bitwise AND of student sets — any student in both lessons
   - **Venue clash**: both lessons claim non-empty same venue (room) in same slot
   
3. **missingL()** (incomplete): was intended to verify expected number of lesson instances per subject/grade (using `getM()` lookup table).

Raw mode clash detection (handling pre-rollover state) checks for grade-mismatch of same teacher in one slot — used to validate input data before promotion.

## Teacher Allocation Algorithm

**Rollover::allocateT()** uses **Kuhn's backtracking augmenting-path matching** (maximum bipartite matching):

1. Group lessons by time slot (slots are independent).
2. Filter real lessons: must have subject code, grade, and students (drop placeholders).
3. For each lesson, build domain of qualified teachers:
   - **Grade-match preferred** (teacher certified for that exact subject+grade combo)
   - **Subject-match fallback** (teacher knows subject, but different grade)
   - Sort both by current load (load-balancing heuristic).
4. **MRV heuristic**: sort lessons by fewest options first.
5. For each lesson, **tryAssign()** recursively attempts to find an augmenting path:
   - Try each qualified teacher in domain order.
   - If teacher unassigned, assign immediately.
   - If teacher assigned, recursively try to reassign their current lesson to free up the teacher.
   - Backtrack if no path found.
6. Apply all successful matches; unmatched lessons go to unallocated list (diagnostics).

## Grade Advancement Logic

**Rollover::incrementYear()** handles Python-style year cycling:
- **Gr 8, 10, 11** → advance normally (8→9, 10→11, 11→12).
- **Gr 9** → reset to 8 (incoming re-take cohort; subject choice changes).
- **Gr 12** → move to 10 (grade compression post-graduation; subject carry-over).

Rationale: South African high school has grades 8–12 on two-year subject cycles; grade 9 students who failed re-take as 8, and post-grade-12 exit, surplus students may continue as grade 10 continuants.

## Data Input

**DataParser** (dataparser.cpp) loads from Access database:
- **parseT()**: teacher roster with qualification codes and subject loads.
- **parseS()**: student roster, grades, registrations.
- **parseGr()**: lessons per grade, raw subject-teacher assignments.
- Two-parse design: `firstParse()` loads current timetable; `secondParse()` applies new grade 8 and subject-choice changes.

Data files (E:\Rollover\Data):
- `new2026.accdb`: MS Access database with all school data.
- `G10SubChoice.xlsx`: grade 10 subject choice selections (imported).
- `NewGr8s.xlsx`: new grade 8 intake (imported).

## Subject M-Slot Lookup (getM.h)

Static hash table mapping subject codes + grades to expected number of lesson instances:
- Grades 8–9: most subjects 3 or 4 instances (e.g., "GE"→3, "DR"→4).
- Grades 10–12: 7 instances standard, some 4 or 2 (e.g., "PE"→2, "LO"→2).

Used by `missingL()` to validate curriculum coverage (not fully implemented).

## Diagnostics & Output

**Diagnostic** class (diagnostic.cpp) (stub): intended for reporting completeness. Currently main.cpp outputs clash reports and unallocated-lesson warnings to console.

## Current State

- **Minimalist design** per CLAUDE.md: no scalability, simple shortest-path algorithms.
- **Incomplete features**: `missingL()` validation not finished; no JSON export (uses console diagnostics only).
- **Qt6 dependency**: all strings are QString, uses Qt containers.
- Built with CMake; Qt Creator project files present.

## Relationship to TimeEduSuite

E:\Rollover is an **earlier, standalone prototype** of the rollover logic now integrated into TimeEduSuite. TimeEduSuite's **TimeEditor** (sessions E.5–E.6) implements a more sophisticated **S0–S8 rollover pipeline** (per ROLLOVER_SPEC.md):
- S0–S4: Data preparation and grade advancement (analogous to Rollover's incrementYear)
- S5–S6: Comprehensive clash detection (3D cube checking cross-grade teacher conflicts)
- S7: Integration with `IRepairEngine` for defect resolution (beyond Rollover's simple augmenting-path)
- S8: JSON export to schema v3.0

Rollover omits clash repair (via engine) and works directly with Access; TimeEditor uses shared `core` library and JSON contracts. Build metadata suggests Rollover may have been tested as proof-of-concept before TimeEditor's more complete implementation.

## Sources

- E:\Rollover\CLAUDE.md
- E:\Rollover\Constants.h
- E:\Rollover\Student.h, Teacher.h, Lesson.h, TimeSlot.h, SchoolData.h
- E:\Rollover\clashdetector.h/.cpp
- E:\Rollover\rollover.h/.cpp
- E:\Rollover\dataparser.h
- E:\Rollover\getM.h
- E:\Rollover\main.cpp
- E:\Rollover\Data\ (directory structure)
- E:\TimeEduSuite\CLAUDE.md
- E:\TimeEduSuite\ROLLOVER_SPEC.md (S0–S8 pipeline design, context only)
