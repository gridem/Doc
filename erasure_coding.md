# Erasure Coding

The document describes ideas related to erasure coding.

## Simple 2D Scheme

The idea is to have XOR additional parts like RAID-5 algorithm: we have a sequence of informational parts and single additional part that allows to recover single lost part.

If we have 4x4 square and each column and row represents the sequence thus we can tolerate:

1. Best case: 4*2 - 1 = 7 lost parts.
2. Worst case: 3 lost parts.
3. Best case for additional parts: 8 parts.

Overhead: (4+4) / (4*4) = 50%
Minimum required parts to recover: 4.

If we have 6x6 square:

1. Best case: 6*2 - 1 = 11 lost parts.
2. Worst case: 3 lost parts.
3. Best case for additional parts: 6*2 = 12 parts.

Overhead: (6+6) / (6*6) = 33%
Minimum required parts to recover: 6.

4x12:

1. Best case: 4+12 - 1 = 15 lost parts. (15/48 = 31%)
2. Worst case: 3 lost parts.
3. Best case for additional parts: 16 parts.

Overhead: (4+12) / (4*12) = 33%
Minimum required parts to recover: 4.

## Additional Ideas

To improve the worst case scenario we can use 3D model, e.g.:

4x12x36

Worst case: 7 lost parts: 2x2x2-1
Overhead: `(4*12+12*36+4*36)/(4*12*36)=36%`, overhead related to 2D: 3% (36-33). Or 1/36 ~= 3%

Overhead

The idea is that the part becomes bad only in rare cases and we could recover it with higher priority if we found that each dimension already has 2 or more lost elements.

## Read Overhead Reduction

> TODO section

The idea is based on splitting the part on subparts. Additional parts (parity parts) are coded by using subparts. It allows to read only subparts of all parts thus to recover 1 part we need to read e.g. 2 subparts that is equivalent to read the single part.