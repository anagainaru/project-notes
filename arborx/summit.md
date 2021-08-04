# Submission on Summit

Arborx was build and run with the following modules
```
module load cuda
module load gcc
```
All submissions are on one node.

### For fixed radix

Batch Script
```bash
for i in {1..60}; do
  cnt=$(( $i * 500 ))
  echo "Problem size $cnt"
  for _ in {1..5}; do
    jsrun -n1 -a1 -c7 -g1 -r1 ./examples/brute_force/ArborX_BruteForce.exe -p $cnt -q $cnt -r 10 -t 100
  done
done
```
Brute force example code
```c++
return attach(intersects(Sphere{{{(float)i, (float)i, (float)i}},
                        (float) d.threshold}), i);
```

### For linear radix

Same batch as for the fixed radix

Brute force example code
```c++
return attach(intersects(Sphere{{{(float)i, (float)i, (float)i}},
                        (float) i}), i);
```

### For log collision:
Same batch as for the fixed radix

Brute force example code
```c++
return attach(intersects(Sphere{{{(float)i, (float)i, (float)i}},
                        log2((float) i + 1)}), i);
```

### For fixed problem size

Brute force example code
```c++
        return attach(intersects(Sphere{{{(float)i, (float)i, (float)i}},
                        (float) d.threshold}),
              i);
```

**For large problem sizes**
```bash
for i in {1..20}; do
  cnt=$(( $i * 1000 ))
  echo "Problem size $cnt"
  for _ in {1..2}; do
    jsrun -n1 -a1 -c7 -g1 -r1 ./examples/brute_force/ArborX_BruteForce.exe -p 100000 -q 100000 -r 10 -t $cnt
  done
done
```

**For small problem sizes**
```bash
for i in {1..20}; do
  cnt=$(( $i * 1500 ))
  echo "Problem size $cnt"
  for _ in {1..2}; do
    jsrun -n1 -a1 -c7 -g1 -r1 ./examples/brute_force/ArborX_BruteForce.exe -p 15000 -q 15000 -r 10 -t $cnt
  done
done
```
