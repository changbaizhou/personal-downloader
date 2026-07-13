# Repository upload policy

- Respond to the user in Chinese.
- Before every GitHub push or other repository upload, verify that the authenticated GitHub login is exactly `changbaizhou`.
- Only upload to repositories owned by `changbaizhou`.
- If the authenticated login cannot be verified, is not `changbaizhou`, or the remote owner is different, do not upload and report the blocker.
- Never bypass the repository's pre-push hook or use `--no-verify`.

