# PR #214 — Review findings

## ALTO: Deadlock en `registry_store::key_info()`

`subkeys()` toma el lock de `KEYS`, y luego `key_info` lo vuelve a tomar.
`std::sync::Mutex` no es reentrant en Rust, asi que esto hara panic o deadlock
la primera vez que se llame `RegQueryInfoKey`.

**Fix:** extraer los datos que necesita `key_info` en una sola toma del lock,
o hacer que `subkeys()` reciba el guard ya tomado.

```
crates/libmwemu/src/api/windows/registry_store.rs
```

## MEDIO: Orden de locks en `window_table::unregister_class`

`unregister_class` toma `WINDOWS` y luego `CLASSES`.
Si algun otro path (ej. `create_window` + `register_class`) los toma en orden
inverso, hay deadlock.

**Fix:** establecer un orden canonico (siempre CLASSES antes que WINDOWS, o al
reves) y respetarlo en todos los call sites.

```
crates/libmwemu/src/api/windows/window_table.rs
```

## BAJO: `--strict-api` y `--banzai` no son mutuamente excluyentes

Ambos flags se pueden pasar a la vez. `--banzai` salta APIs no implementadas,
`--strict-api` para ante ellas. Comportamiento ambiguo.

**Fix:** validar en el CLI que no se pasen juntos, o documentar cual tiene
prioridad.

```
crates/mwemu/src/main.rs
crates/libmwemu/src/config.rs
```

## BAJO: `virtual_protect::protect` cambia toda la region

Cuando `[addr, addr+size)` cubre solo parte de una region mapeada, `protect()`
modifica los permisos de TODA la region. Las regiones de mwemu pueden ser mucho
mayores que una pagina de Windows (4 KB).

**Fix:** considerar split de la region en el rango solicitado, o al menos
documentar la limitacion.

```
crates/libmwemu/src/api/windows/common/virtual_protect.rs
```

## DISENO: Estado global (`lazy_static!`) en todos los stores

`registry_store`, `environment_store`, `profile_store`, `crt_stdio` y
`window_table` usan `lazy_static! { Mutex<...> }` como singletons.

Esto impide tener multiples instancias de `Emu` en un mismo proceso (relevante
para pymwemu). No es un bug ahora, pero limita la reutilizacion futura.

**Fix futuro:** mover el estado al struct `Emu` o a un `EmulationContext` que
se pase por referencia.

```
crates/libmwemu/src/api/windows/registry_store.rs
crates/libmwemu/src/api/windows/environment_store.rs
crates/libmwemu/src/api/windows/profile_store.rs
crates/libmwemu/src/api/windows/winapi64/crt_stdio.rs
crates/libmwemu/src/api/windows/window_table.rs
```
