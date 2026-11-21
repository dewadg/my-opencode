<!-- plan snapshot; do not edit -->
# Graph — glab50-template-resources

Source: tasks.md@438245ee1adf · 5 nodes · 4 layers · 8 edges

```
glab50-template-resources · 5 nodes · 4 layers
L1  ✓ 1  Template content                js-eng          6/6  ←None
L2  ✓ 2  Registry, resources and tests   golang-eng      4/4  ←1
L3  ✓ 3  Guide list and docs             api-documenter  2/2  ←2
L4  ✓ 4  Review                          code-reviewer   1/1  ←1,2,3   {review}
    ✓ 5  Render and publish              qa-eng          3/3  ←1,2,3   {qa}
next: none · deferred: none · blocked: none · critical path 1→2→3→4
```

## Coverage

| TC | Nodes |
|---|---|
| TC-830 | 1, 5 |
| TC-833 | 2, 3 |
| TC-834 | 1, 2 |
| TC-835 | 1, 2, 5 |
| TC-836 | 1, 2 |
| TC-837 | 1, 2, 5 |
| TC-838 | 1, 2 |
| TC-839 | 3 |
