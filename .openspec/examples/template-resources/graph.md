<!-- plan snapshot; do not edit -->
# Graph — template-resources

Source: tasks.md@8888852eebf3 · 6 nodes · 5 layers · 7 edges

```
template-resources · 6 nodes · 5 layers · 3 build · 2 verify · 1 review
L1  ✓ 1  Template content                js-eng          6/6  ←None
L2  ✓ 2  Registry, resources and tests   golang-eng      4/4  ←1
L3  ✓ 3  Guide list and docs             api-documenter  2/2  ←2
L4  ✓ 4  Verify                          golang-eng      1/1  ←1,2,3   {verify}
L5  ✓ 5  Review                          code-reviewer   1/1  ←4       {review}
    ✓ 6  Render and publish              qa-eng          3/3  ←4       {qa}
next: none · deferred: none · blocked: none · critical path 1→2→3→4→5
```

## Coverage

| TC | Nodes |
|---|---|
| TC-830 | 1, 6 |
| TC-833 | 2, 3, 6 |
| TC-834 | 1, 2, 6 |
| TC-835 | 1, 2, 6 |
| TC-836 | 1, 2 |
| TC-837 | 1, 2, 6 |
| TC-838 | 1, 2 |
| TC-839 | 3 |
