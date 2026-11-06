# Exam Topics Tracker

This doc explains `exam-topics.csv`, which tracks each exam topic I need to study.

## Columns

| Column | Meaning |
|--------|---------|
| `topic` | Exam topic name |
| `priority` | `high`, `medium` or `low`, based on exam weight and my confidence |
| `status` | `not started`, `in progress` or `done` |
| `target_date` | Date I plan to finish notes and practice for the topic |

## How to update

1. When I start a topic, set `status` to `in progress`.
2. When notes and practice questions are done, set it to `done`.
3. If I slip past a `target_date`, move it and note why in `log.md`.
