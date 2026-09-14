# Note

## Insight
- CPU internal resources (ROB size) would limit outstanding memory operations thus limiting MLP.
- Larger OoO window also means higher hardware cost and comlexity

## Part 1
- What problem they solve? - poor memory bandwidth utilization featured by indirect memory operation.
- What is the key idea? - large visible window, reorder, coalesce and interleave.
- corelation? - optimizing indirect memory access.