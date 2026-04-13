# Detect-DSE-Bypass
How EAC Can Detect DSE Bypass



# 🧠 How EAC Can Detect DSE Bypass

Been looking into **EasyAntiCheat_EOS.sys** for a bit and the usual idea:

> `patch ci.dll → load → restore → done`

…doesn’t really hold up once you dig deeper.

---

## What actually happens

The basic flow is correct:

1. Patch `g_CiOptions` or hook `CiValidateImageHeader`
2. Call `NtLoadDriver` → ends up in `MmLoadSystemImage`

Which automatically:

* Inserts your driver into `PsLoadedModuleList`
* Creates a `DRIVER_OBJECT` (`\Driver\Name`)
* Uses a service key from the registry

3. `DriverEntry` runs
4. You restore whatever you patched

On paper this looks clean — but the issue is everything that stays behind.

---

## Where it starts breaking

### PsLoadedModuleList

EAC walks the module list — expected.

But it’s not just presence, they also look at:

* Name validity
* Certificate info
* Structure sanity

So even properly loaded drivers get inspected.

---

### SystemModuleInformation

```c
NtQuerySystemInformation(SystemModuleInformation, ...)
```

Same list, different access path.

If you modify one and not the other → inconsistency.

---

### Driver object still exists

Even if you unlink from the module list:

```text
\Driver\YourDriverName
```

Still exists as a separate object.

---

### Code Integrity (timing issue)

```c
NtQuerySystemInformation(SystemCodeIntegrityInformation)
```

Exposes `g_CiOptions`.

If your timing isn’t perfect → easy catch.

---

### Certificate validation

This part is deeper than most expect:

* Parses certificate chain
* Detects self-signed certs
* Validates structure

So fake signing isn’t reliable.

---

## Important point

Restoring doesn’t mean invisible.

Even if you:

* Restore `ci.dll` immediately
* Unlink from module list
* Clean registry
* Remove driver object

You’re just reducing visibility — not eliminating it.

Too many structures need to stay consistent.

---

## Loading before EAC?

Helps a bit:

* Avoids some load-time callbacks

But you still deal with:

* Module enumeration
* Object lookups
* General integrity checks

So it’s not really a bypass — just less exposure during load.

---

## Final note

DSE bypass isn’t just about loading anymore.

Once the driver goes through normal kernel paths, it leaves traces in multiple places.

Cleaning up helps, but it’s not a guaranteed “gg”.
