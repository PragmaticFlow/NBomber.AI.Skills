# When the one-request check fails

Read this when the one-request check in Step 10 of SKILL.md fails.

Look at the status codes and errors in the NBomber output (and the log in the `reports` folder), and in the API's own console or log files if they're on the user's machine. Then work out whose fault it is:

- **5xx responses or server-side exceptions** (database errors, missing tables, connection failures, unhandled exceptions) mean the API or its environment is broken, not the test. Don't change the test to hide it. Say so clearly, look for the cause (config files, running containers, ports, missing setup, as in Step 5 of SKILL.md), and suggest a fix. Apply the fix only if the user agrees.
- **4xx responses** usually mean the test sent something wrong: a missing or invalid field, wrong auth, a wrong id from chaining, or a path that doesn't match. Compare the request with the spec (and with the API's validation code if it's local), fix the generated code, build, and run the check again.
- **Connection refused or timeouts** usually mean the API isn't running, or the base URL or port is wrong. Ask the user to start it or confirm the URL.

Repeat until the check passes, or until the remaining problem is on the API side and you've explained it to the user.
