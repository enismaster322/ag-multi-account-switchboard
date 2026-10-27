fix: verify uss-oauth before restart, migrate apiKey via rpc and probe cold start ports

Cause:
1. When switching accounts, setOAuthTokenInfo dispatches an asynchronous update to uss-oauth via IPC. However, switchAccount immediately updated userStatus in step 2. This caused Antigravity's maybeUpdate loop to detect an email change and restart the Language Server before uss-oauth was acknowledged in memory, spawning the new LS process with the stale accessToken passed on stdin.
2. The Language Server binary caches its active credentials and does not take credentials in RegisterGdmUser. When spawned with stale credentials, it persistently returned the previous account's UserStatus, timing out for 25s.
3. On cold start or when the IDE boots without --https_server_port in process arguments, findLSEndpoints failed to identify listening ports, returning empty endpoint lists.

Solution:
1. Added verification loop in switchAccount to await confirmation of the new accessToken in uss.OAuthPreferences.getOAuthTokenInfo() before pushing the updated userStatus.
2. Added callMigrateApiKey using the native /exa.language_server_pb.LanguageServerService/MigrateApiKey RPC with encodeString(1, accessToken), directly pushing credentials into running LS processes to immediately trigger loadCodeAssist and model retrieval.
3. Added fallback in findLSEndpoints to discover active listening TCP ports via lsof when --https_server_port is omitted from process arguments, and updated awaitLSReadiness to probe candidate ports for responsive HTTPS gRPC endpoints.
4. Added credential migration in token renewal and periodic polling retries to keep credentials synchronized without requiring process restarts.
