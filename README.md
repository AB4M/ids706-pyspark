# ids706-pyspark
## Dataset description and source
Dataset: NYC TLC Yellow Taxi Trip Records (Jan–Jun 2023)
Public trip records for New York City yellow cabs, including timestamps, pickup/dropoff locations (zone IDs), fares, tips, trip distances, and other meter-based charges. A small zone lookup CSV maps LocationID to human-readable Borough and Zone.
## Performance analysis
Performance Analysis (Summary)<br/>
Spark (3.5.1) executed the pipeline with Adaptive Query Execution. Filters on time, distance, and fare were pushed down to Parquet scans and only needed columns were read, which reduced I/O. The small taxi_zone_lookup.csv was joined via Broadcast Hash Join (~1 MB broadcast), so shuffles stayed tiny (KB-level). Aggregations ran with AQE shuffle coalescing and reported no spill, indicating sufficient memory.<br/>

Bottlenecks & Optimizations<br/>
The main cost came from Parquet decode/scan (≈1–3 s per file across six monthly files) and hash aggregation CPU time after the monthly union. We optimized by (1) filtering early, (2) pruning columns, (3) broadcasting the small dimension table, and (4) writing results as Parquet. For larger datasets, partitioning the fact data by month and writing outputs partitioned by pickup_month would further improve pruning and scan time.<br/>

Actions vs Transformations<br/>
filter/select/withColumn are transformations (lazy). An action like count() triggers a job; you can see a new job appear in the UI after calling it.<br/>
## Key findings
Manhattan consistently dominates pickups. It ranks #1 every month in our sample (e.g., ~2.67M trips in 2023-01 and ~2.96M in 2023-03).
Queens is the steady #2, with monthly volumes in the ~1.2M–1.3M range across the period analyzed.
A small “Unknown” bucket appears each month, caused by PULocationIDs that do not map to a borough (left-join fallout / special IDs).
Monthly scale is multi-million level (~3–3.5M trips/month) for the months we processed (Jan–Jun 2023), indicating stable, high demand.
After borough enrichment, top-3 composition is stable (Manhattan → Queens → others), suggesting persistent spatial patterns in demand.
Aggregates are computed with early filters (distance>0 & fare>0) and broadcast lookups, so the results reflect cleaned, high-signal records rather than raw totals.
## Screenshots
<img width="1206" height="747" alt="image" src="https://github.com/user-attachments/assets/c9dcc7fa-71d7-42a2-85e5-4dc2bcddf4b5" />
<img width="1461" height="975" alt="image" src="https://github.com/user-attachments/assets/0c9d2242-8941-41dc-87bb-c3317b80c7d8" />
<img width="1465" height="976" alt="image" src="https://github.com/user-attachments/assets/f3f0b5a8-a5ce-40c9-a2d7-394a28dbcd8a" />
<img width="1463" height="979" alt="image" src="https://github.com/user-attachments/assets/0d7f9f8f-808c-4d37-ba78-7922d16f5ed4" />
<img width="1475" height="918" alt="image" src="https://github.com/user-attachments/assets/12f7a9f5-f145-4a9c-b1f7-2bdd06b75a98" />
<img width="1476" height="863" alt="image" src="https://github.com/user-attachments/assets/140e7f99-c251-4805-93eb-04a04fce1d06" />
<img width="1471" height="975" alt="image" src="https://github.com/user-attachments/assets/90265e10-be5f-40cc-a780-64926d3c17bc" />
<img width="1190" height="319" alt="image" src="https://github.com/user-attachments/assets/ff324385-bfd8-4cc0-a241-c3c01233e1c9" />


