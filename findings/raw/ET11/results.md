## ET11 results (2026-10-11T22:03Z)

| | Check |
|---|---|
| ℹ️ | 1 rounds where both changes survived on both controllers: 10/10; writes merged: 10 |
| ✅ | 1 both changes survive every round, on both controllers (10/10) |
| ✅ | 1 stored config.xml has both (round 10) |
| ℹ️ | 2 A says 'from A', B says 'from A'; conflicting writes: 1 |
| ✅ | 2 A and B agree on one of the two values |
| ℹ️ | 3 round 2: A round 1 from A|H H 2 * *, B round 1 from A|H H 2 * * |
| ❌ | 3 description (A) and trigger (B) both survive, on both controllers (2/3) |
