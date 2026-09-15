# Backup encryption

`roles/rclone` renders an `[r2crypt]` remote when both crypt passwords are set: rclone's `crypt` overlay over
`rclone_crypt_target` (default `r2:backups`), so R2 stores only ciphertext. A service opts in by writing to
`r2crypt:<path>` instead of `r2:<path>`.

```
r2crypt:<service>/<scope>/  →  r2:backups/<service>/<scope>/<encrypted-filename>
```

Directory names stay plain so R2 lifecycle prefix rules keep matching; filenames and contents are encrypted.

> [!WARNING] Lose both passwords and every encrypted backup is unreadable. No escrow, no recovery.

## Turn it on

1. On the controller, generate two independent passwords and obscure each:

   ```bash
   rclone obscure '<password-1>'
   rclone obscure '<password-2>'
   ```

   Obscuring is reversible, not encryption; the consumer's vault is what protects them.

2. Escrow both plaintexts out of band — somewhere that survives losing the vault and your password manager.

3. Set the obscured pair in the consumer's vault, both or neither (the role asserts it):

   ```yaml
   rclone_crypt_password: "<obscured-1>"
   rclone_crypt_password2: "<obscured-2>"
   ```

4. Converge. Point services at `r2crypt:`, then delete the old plaintext objects.
