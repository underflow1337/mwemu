# PR #214 — Review findings

## HIGH: Deadlock in `registry_store::key_info()`

`subkeys()` acquires the `KEYS` Mutex lock, then `key_info` tries to acquire
it again. `std::sync::Mutex` is not reentrant in Rust, so this will panic or
deadlock the first time `RegQueryInfoKey` is called.

**Fix:** extract all data `key_info` needs in a single lock acquisition, or
have `subkeys()` accept an already-held guard.

```
crates/libmwemu/src/api/windows/registry_store.rs
```

## MEDIUM: Lock ordering in `window_table::unregister_class`

`unregister_class` acquires `WINDOWS` then `CLASSES`. If any other code path
(e.g. `create_window` + `register_class`) acquires them in reverse order, a
deadlock can occur.

**Fix:** establish a canonical lock order (always CLASSES before WINDOWS, or
vice versa) and enforce it across all call sites.

```
crates/libmwemu/src/api/windows/window_table.rs
```

## LOW: `--strict-api` and `--banzai` are not mutually exclusive

Both flags can be passed at the same time. `--banzai` skips unimplemented APIs,
`--strict-api` stops on them. Behavior is ambiguous when combined.

**Fix:** validate in the CLI that they cannot be used together, or document
which one takes priority.

```
crates/mwemu/src/main.rs
crates/libmwemu/src/config.rs
```

## LOW: `virtual_protect::protect` changes the entire region

When `[addr, addr+size)` covers only part of a mapped region, `protect()`
modifies the permissions of the ENTIRE region. mwemu regions can be much larger
than a Windows page (4 KB).

**Fix:** consider splitting the region at the requested range boundaries, or at
least document the limitation.

```
crates/libmwemu/src/api/windows/common/virtual_protect.rs
```

## DESIGN: Global state (`lazy_static!`) in all stores

`registry_store`, `environment_store`, `profile_store`, `crt_stdio`, and
`window_table` all use `lazy_static! { Mutex<...> }` as singletons.

This prevents running multiple `Emu` instances in the same process (relevant
for pymwemu). Not a bug today, but limits future reusability.

**Future fix:** move the state into the `Emu` struct or an `EmulationContext`
passed by reference.

```
crates/libmwemu/src/api/windows/registry_store.rs
crates/libmwemu/src/api/windows/environment_store.rs
crates/libmwemu/src/api/windows/profile_store.rs
crates/libmwemu/src/api/windows/winapi64/crt_stdio.rs
crates/libmwemu/src/api/windows/window_table.rs
```
