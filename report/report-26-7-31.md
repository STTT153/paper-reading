# TODOS
1. Scalar (2 elements/cycle), vector (0.83 elements/cycle) This results reveals that, even vector version can use two LSU pipelines, some resource still limit the paralelism.

2. None-blocking cache? try the throuput experiment against iteration numbers.

3. Read OpenC910 archetecture [x] according to K3's paper, they refactor the memory subsystem.

4. k3 gather use sequential processing to deal with duplicates. - check some pattern that access the same address.

5. Check out index value in spec2026

6. Other research about indexed load.