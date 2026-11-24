<!-- plan snapshot; do not edit -->
# Graph — <!-- change name -->

Source: tasks.md@<!-- 12-char sha256 of tasks.md --> · <!-- n --> nodes · <!-- n --> layers · <!-- n --> edges

<!--
Written once after tasks.md exists, in the board format below. A plan snapshot, not a
live view: do not redraw it during apply and never edit the plan here. Header counts are
stages (qa counts as verify, human as review) and only non-zero ones print. Tags are
{verify}, {qa}, {review}, {human}; build is untagged. `next:` prints the first 4 ready ids
and appends `+<n>` for the rest; `deferred:` names conflict-deferred nodes only, never a size cap.
-->

```
<!-- change --> · <!-- n --> nodes · <!-- n --> layers · <!-- n --> build · <!-- n --> verify · <!-- n --> review
L1  ▪ 1  <!-- title, truncated to 36 with … -->  <!-- owner -->  <!-- done -->/<!-- total -->  ←None
L2  ⊘ 2  <!-- title -->  <!-- owner -->  0/1  ←1   {verify}
L3  ▪ 3  <!-- title -->  <!-- owner -->  0/2  ←2   {review}
next: <!-- ids or none --> · deferred: <!-- ids or none --> · blocked: <!-- ids or none --> · critical path <!-- 1→2→3 or none -->
```

## Coverage

| TC | Nodes |
|---|---|
| <!-- TC-… --> | <!-- node ids --> |
