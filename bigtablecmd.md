The command you have looks correct! Here's the properly formatted `cbt set` command:

```bash
cbt set users user#user-042 \
  profile:name="Priya Sharma" \
  profile:email="priya@example.com" \
  profile:rating="4.8"
```

Each `family:column=value` argument sets a cell in the `profile` column family. A few things to verify before running:

**Prerequisites**
- Your `.cbtrc` is configured (or pass `-project` and `-instance` flags inline)
- The `users` table exists: `cbt createtable users`
- The `profile` column family exists: `cbt createfamily users profile`

**Verify the write worked**
```bash
cbt read users prefix=user#user-042
```

**Inline flag alternative** (without `.cbtrc`):
```bash
cbt -project oval-turbine-495504-h2 -instance rideshare-instance \
  set users user#user-042 \
  profile:name="Priya Sharma" \
  profile:email="priya@example.com" \
  profile:rating="4.8"
```

The row key pattern `user#user-042` with a `#` separator is a good practice for Bigtable — it lets you do efficient prefix scans across all user rows while keeping the key human-readable.
