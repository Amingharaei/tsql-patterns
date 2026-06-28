# T-SQL Patterns

Practical T-SQL patterns to answer problems such as running distinct counts, gaps in a sequence, de-duplication, top-N per group, dynamic pivot and unpivot, recursive hierarchies, JSON, and more.

Each file is self-contained, creates a small sample table, works through one or more approaches with their output, covers the edge cases worth knowing about, and drops the table at the end.

## Index

| Topic | File |
|-------|------|
| Running count of unique customers per salesperson | [running-unique-count.md](running-unique-count.md) |
| Finding the gaps in a sequence | [finding-gaps.md](finding-gaps.md) |
| Removing duplicate rows | [removing-duplicate-rows.md](removing-duplicate-rows.md) |
| Top-N rows per group | [top-n-rows-per-group.md](top-n-rows-per-group.md) |
| First and last value of each group | [first-and-last-value-each-group.md](first-and-last-value-each-group.md) |
| Ranking a hypothetical value within each group | [hypothetical-rank.md](hypothetical-rank.md) |
| Counting rows in a rolling three-day window | [rolling-window-count.md](rolling-window-count.md) |
| Walking an org chart with a recursive CTE | [recursive-cte-org-chart.md](recursive-cte-org-chart.md) |
| Dynamic pivot when the columns are unknown at query time | [dynamic-pivot.md](dynamic-pivot.md) |
| Dynamic unpivot when the columns are unknown at query time | [dynamic-unpivot.md](dynamic-unpivot.md) |
| Finding the islands (consecutive runs) | [finding-islands.md](finding-islands.md) |
| Null-safe comparison with IS DISTINCT FROM | [null-safe-comparison-is-distinct-from.md](null-safe-comparison-is-distinct-from.md) |
| Working with the native JSON type | [native-json-type.md](native-json-type.md) |

## Notes

- `native-json-type.md` covers the native JSON type introduced in SQL Server 2025.
- `null-safe-comparison-is-distinct-from.md` requires SQL Server 2022, where `IS DISTINCT FROM` was added.
- All sample data is synthetic and sized to make the behaviour easy to follow.
