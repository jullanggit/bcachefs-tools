# Testing bcachefs

## ktest CI

- **Dashboard**: https://evilpiepirate.org/~testdashboard/c
- **Test files**: `../ktest/tests/fs/bcachefs/*.ktest`

## ktest

Tests live in `../ktest/tests/fs/bcachefs/` (e.g. `subvol.ktest`).

```bash
# Run shrink tests
./scripts/install-to-kernel.sh ../bcachefs-test/
cd ../ktest
direnv exec . ./build-test-kernel run -k ../bcachefs-test tests/fs/bcachefs/shrink.ktest <specific tests> 2>&1 | tee /tmp/<log file>
```

- Test functions in the file are named `test_*`; drop the `test_` prefix
  when running (e.g. `test_foo` in the file becomes `btk run ... foo`).
- Prefix a test function with `d_` to disable it.
- Source `bcachefs-test-libs.sh` for helpers (`config-scratch-devs`,
  `bcachefs_antagonist`, etc.).
- End tests with `bcachefs_test_end_checks` for fsck.
- ktest uses its own build output dir (`ktest-out/kernel_build.$arch`),
  not `.linux`.
- With `-I` (interactive), the VM stays running after the test
  completes — must `C-c` to kill before running another.
- Alwasy redirect into a log file as there can be a lot of output and this
  enables searching through it using grep etc

## Testing C-to-Rust command conversions

When converting a command from C to Rust, always compare the output of
the old and new versions on real data before committing:

```bash
# Build the pre-conversion version in a worktree
git worktree add /tmp/bcachefs-old <commit-before-conversion>
cd /tmp/bcachefs-old && make -j

# Compare output
/tmp/bcachefs-old/target/release/bcachefs <command> <args> > /tmp/old.txt
./target/release/bcachefs <command> <args> > /tmp/new.txt
diff -u /tmp/old.txt /tmp/new.txt

# Clean up
git worktree remove /tmp/bcachefs-old
```

### What to watch for

- **Which superblock pointer?** `c->disk_sb.sb` is a selective copy
  that omits fields like `magic` and `layout` (see `__copy_super()` in
  `sb/io.c`). The per-device `ca->disk_sb.sb` has the full on-disk
  data. If the old C code iterates online members to get the sb, the
  Rust conversion must do the same — the loop isn't boilerplate.

- **Human-readable units.** The old C code sets
  `buf.human_readable_units = true` directly. In Rust, use
  `buf.set_human_readable(true)`. Missing this gives raw byte counts.

- **Structural loops.** C iteration patterns (`for_each_online_member`,
  `for_each_member_device`) often exist because they access different
  data than a direct struct field read. Don't flatten them without
  understanding why.
