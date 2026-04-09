# EduTrack - Student performance analytics pipeline

## Problem Statement

EduTrack is a fictional college analytics platform. Every day, new enrollment and assessment records arrive as CSV files, one file per day covering students across multiple departments and courses. The raw data is messy, duplicate records, marks with % symbols, attendance values above 100, inconsistent grade formats and null values.

The goal of this project is to build a clean, reliable batch data pipeline on Databricks using the Medallion Architecture that transforms this raw data into analytics ready Delta tables, enabling educators and administrators to answer questions like -

- Which department has the highest pass rate?
- Which course has the best average score?
- How does attendance correlate with performance?
- Which students are at risk (low marks + low attendance)?
- What is the overall grade distribution across the college?

---

## Architecture

```
Raw CSVs → [Bronze] → [Silver] → [Gold] → Analytics / BI
```

| Layer  | Purpose                                                      |
|--------|--------------------------------------------------------------|
| Bronze | Raw ingestion — data stored exactly as received              |
| Silver | Cleaned & transformed — all quality issues resolved          |
| Gold   | Business-ready — metrics calculated, joins applied           |

---

## Tech Stack

| Tool         | Purpose                               |
|--------------|---------------------------------------|
| Databricks   | Cloud platform to run notebooks       |
| Apache Spark | Distributed data processing engine    |
| PySpark      | Python API for Spark                  |
| Delta Lake   | Versioned, reliable table format      |
| SQL          | Querying gold layer tables            |

---

## Dataset Overview

### Fact Table (daily, incremental)
| File | Description |
|------|-------------|
| `enrollments/landing/enrollments_YYYY-MM-DD.csv` | Daily enrollment + result records — 90 days of data (Jan–Mar 2024) |

**Columns:** `dt`, `enrollment_id`, `student_id`, `course_id`, `instructor_id`, `semester`, `marks_obtained`, `max_marks`, `attendance_pct`, `grade`, `assignment_score`, `status`

### Dimension Tables (static)
| File | Rows | Description |
|------|------|-------------|
| `students.csv` | 150 | Student master — name, department, year, city |
| `courses.csv` | 12 | Course master — name, department, credits |
| `instructors.csv` | 8 | Instructor master — name, department, experience |
| `date.csv` | 366 | Date dimension — year, month, quarter, week, weekend flag |

---

## Data Quality Issues Fixed in Silver Layer

| Table | Issue | Fix Applied |
|-------|-------|-------------|
| `enrollments` | Duplicate rows (same enrollment_id) | `dropDuplicates()` |
| `enrollments` | `marks_obtained` has `%` symbol (e.g. `85%`) | `regexp_replace()` + cast to int |
| `enrollments` | Negative marks (e.g. `-3`) | `when(col < 0, 0).otherwise(col)` |
| `enrollments` | `attendance_pct` exceeds 100 (data entry error) | Cap at 100 using `when()` |
| `enrollments` | Inconsistent grade format (`a`, `B+`, `Pass`, `PASS`) | `upper()` + `when/otherwise` mapping |
| `enrollments` | NULL `assignment_score` | `fillna(0.0)` |
| `students` | Department casing (`computer science`, `ELECTRONICS`) | `replace()` dictionary |
| `students` | City casing (`mumbai`, `DELHI`) | `initcap()` |
| `students` | NULL `phone` | `fillna('Not Available')` |
| `date` | Negative `week_of_year` | `abs()` |
| `date` | Inconsistent `day_name` casing | `initcap()` |

---

## Gold Layer — Calculated Metrics

| Column | Calculation |
|--------|-------------|
| `percentage_score` | `marks_obtained / max_marks × 100` |
| `total_score` | `marks_obtained + assignment_score` (out of 125) |
| `pass_flag` | `1` if `marks_obtained >= 40`, else `0` |
| `date_id` | Integer key (`yyyyMMdd`) for joining with date dimension |

---


## Project Structure

```
edutrack/
├── README.md
├── data/
│   ├── enrollments/landing/     ← 90 daily CSV files
│   ├── students/
│   ├── courses/
│   ├── instructors/
│   └── date/
└── notebooks/
    ├── 1_setup/
    ├── 2_dim_bronze/
    ├── 3_dim_silver/
    ├── 4_dim_gold/
    ├── 5_fact_bronze/
    ├── 6_fact_silver/
    └── 7_fact_gold/
```
