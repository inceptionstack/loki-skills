# Smell Priority Framework

## How to Prioritize Refactoring

Not all smells are equal. Prioritize based on **change frequency × pain level**.

### Quadrant Model

```
                    HIGH CHANGE FREQUENCY
                           │
         Fix Now           │         Fix Next Sprint
   (blocking velocity)     │       (accumulating debt)
                           │
  HIGH ────────────────────┼──────────────────── LOW
  PAIN                     │                    PAIN
                           │
       Fix When Nearby     │          Leave Alone
    (opportunistic)        │       (stable, working)
                           │
                    LOW CHANGE FREQUENCY
```

### Priority Score

```
Priority = (change_frequency × 3) + (bug_correlation × 2) + (comprehension_cost × 1)
```

Each factor scored 1-5:
- **Change frequency**: How often do we modify this code? (git log --since=3months)
- **Bug correlation**: How many bugs originate here? (issue tracker linkage)
- **Comprehension cost**: How long does a new developer need to understand it?

### Score Interpretation

| Score | Action |
|-------|--------|
| 25-30 | Emergency — blocking team velocity |
| 18-24 | Plan for this sprint |
| 12-17 | Add to backlog, fix when nearby |
| 6-11 | Leave alone (low impact) |

## Hotspot Analysis

Use git history to find refactoring targets:

```bash
# Files changed most in last 3 months
git log --since="3 months ago" --pretty=format: --name-only | sort | uniq -c | sort -rn | head -20

# Files with most distinct authors (knowledge silos)
git log --since="6 months ago" --format="AUTHOR:%an" --name-only | awk '/^AUTHOR:/{author=substr($0,8);next} NF==0{next} {a[$0][author]=1} END{for(f in a) print length(a[f]), f}' | sort -rn | head -20 

# Churn + complexity (lines × changes)
for f in $(git log --since="3 months ago" --pretty=format: --name-only | sort -u); do
  changes=$(git log --since="3 months ago" --oneline -- "$f" | wc -l)
  lines=$(wc -l < "$f" 2>/dev/null || echo 0)
  echo "$((changes * lines)) $changes $lines $f"
done | sort -rn | head -20
```

## When to Stop Refactoring

A module is "good enough" when:
1. Each file ≤400 lines
2. Each function ≤50 lines with a clear name
3. Tests exist for public API surface
4. A new developer can understand it in <15 minutes
5. Changes to one responsibility don't risk breaking others

Diminishing returns start when you're renaming things for aesthetics or extracting methods that are only called once and don't improve readability.
