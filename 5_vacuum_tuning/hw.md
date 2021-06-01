
## Initial Database Size

|main   |   fsm   |   vm   |  index  | total
|-------|---------|--------|---------|-------
|17 MB  | 1080 kB | 360 kB | 5712 kB | 25 MB

|pgbench_accounts|
|----------------|
|13 MB           |

## First Run - Baseline (Default Parameters)
- `autovacuum_naptime = 1min`
- `autovacuum_vacuum_scale_factor = 0.2`
- `autovacuum_vacuum_threshold = 50`
- `autovacuum_analyze_scale_factor = 0.1`
- `autovacuum_analyze_threshold = 50`
- `autovacuum_max_workers = 3`

### Transactions
![](./images/baseline/transactions.png | width=669 height=391)
### Live Tuples
![](./images/baseline/live_tuples.png){:height="50%" width="50%"}
### Dead Tuples
![](./images/baseline/dead_tuples.png){:height="50%" width="50%"}
### Autovacuum Count
![](./images/baseline/autovacuum_count.png){:height="50%" width="50%"}
### Database Size
![](./images/baseline/database_size.png){:height="50%" width="50%"}
### Change Stats
![](./images/baseline/change_stats.png){:height="50%" width="50%"}

## Summary


## Second Run - Recommended (From Amazon Articles)
- `autovacuum_naptime = 30`
- `autovacuum_vacuum_scale_factor = 0.1`
- `autovacuum_vacuum_threshold = 50`
- `autovacuum_analyze_scale_factor = 0.1`
- `autovacuum_analyze_threshold = 50`
- `autovacuum_max_workers = 4`

### Transactions
![](./images/recommended/transactions.png)
### Live Tuples
![](./images/recommended/live_tuples.png)
### Dead Tuples
![](./images/recommended/dead_tuples.png)
### Autovacuum Count
![](./images/recommended/autovacuum_count.png)
### Database Size
![](./images/recommended/database_size.png)
### Change Stats
![](./images/recommended/change_stats.png)

## Summary


## Third Run - Aggressive
- `autovacuum_naptime = 20`
- `autovacuum_vacuum_scale_factor = 0.05`
- `autovacuum_vacuum_threshold = 20`
- `autovacuum_analyze_scale_factor = 0.05`
- `autovacuum_analyze_threshold = 20`
- `autovacuum_max_workers = 8`
- `autovacuum_vacuum_insert_threshold = 500`
- `autovacuum_vacuum_insert_scale_factor = 0.05`

### Transactions
![](./images/aggressive/transactions.png)
### Live Tuples
![](./images/aggressive/live_tuples.png)
### Dead Tuples
![](./images/aggressive/dead_tuples.png)
### Autovacuum Count
![](./images/aggressive/autovacuum_count.png)
### Database Size
![](./images/aggressive/database_size.png)
### Change Stats
![](./images/aggressive/change_stats.png)

## Summary

## Fourth Run - Disable
- `autovacuum = off`

### Transactions
![](./images/disabled/transactions.png)
### Live Tuples
![](./images/disabled/live_tuples.png)
### Dead Tuples
![](./images/disabled/dead_tuples.png)
### Autovacuum Count
![](./images/disabled/autovacuum_count.png)
### Database Size
![](./images/disabled/database_size.png)
### Change Stats
![](./images/disabled/change_stats.png)

## Summary