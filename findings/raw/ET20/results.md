## ET20 results (2026-10-12T19:02Z)

| | Check |
|---|---|
| ℹ️ | 1 B answered: hello from jenkins-a |
| ✅ | 1 a WebSocket opened on B for A's running build is answered by A |
| ✅ | 1 ... as when opened on A itself |
| ℹ️ | 2 console -f exit 0, 15 tick lines |
| ✅ | 2 console -f on B follows A's build to the end |
| ℹ️ | 2b console -f (-webSocket) exit 0, 15 tick lines |
| ✅ | 2b console -f -webSocket on B follows A's build to the end |
| ℹ️ | 3 build -s -v exit 0: Started onala #1 (on another controller), Completed onala #1 : SUCCESS |
| ❌ | 3 build -s -v on B waits for the build A ran, with its log |
| ✅ | 4 it refuses to start, saying why |
| ✅ | 4 ... and never became a member |

Failures: 1
