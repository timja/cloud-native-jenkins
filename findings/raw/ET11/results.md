## ET11 results (2026-10-12T15:29Z)

| | Check |
|---|---|
| ℹ️ | 1 rounds where both changes survived on both controllers: 10/10; writes merged: 8 |
| ✅ | 1 both changes survive every round, on both controllers (10/10) |
| ✅ | 1 stored config.xml has both (round 10) |
| ℹ️ | 2 A says 'from A', B says 'from A'; conflicting writes: 0 |
| ✅ | 2 A and B agree on one of the two values |
| ✅ | 3 description (A) and trigger (B) both survive, on both controllers (3/3) |
| ℹ️ | 4 A writes, unshared: nothing |
| ℹ️ | 4 B writes, unshared: nothing |
| ✅ | 4 A's home holds nothing the cluster does not share |
| ✅ | 4 B's home holds nothing the cluster does not share |
| ℹ️ | 5 A writes, unshared: foo-bad/state.xml (1, e.g. foo-bad/state.xml) |
| ✅ | 5 the report names it (foo-bad/state.xml) |
| ✅ | 5 and it is the only finding |

Failures: 0
