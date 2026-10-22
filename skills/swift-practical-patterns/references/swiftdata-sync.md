# Seeded-data duplicates in SwiftData sync

Covers seeded-data identity, duplicate diagnosis, and merge decisions under SwiftData's automatic CloudKit sync. Scope: personal data sync on iOS/iPadOS 17+ and macOS 14+; shared databases, multi-user collaboration, and full migration solutions are out of scope.

Official documentation verified 2026-10-08. Local verification used Xcode 27.0, Apple Swift 6.4, the Swift 6 language mode, a macOS 27 host, and a macOS 14 deployment target; real CloudKit and multi-device behavior were not verified.

## Confirm the model and sync configuration first

Automatic SwiftData sync requires the iCloud capability with CloudKit enabled, plus Background Modes remote notifications. The default container is inferred from the app's entitlements; with more than one container, check which one is actually selected, and use `ModelConfiguration(cloudKitDatabase:)` when an explicit choice is needed. `.none` turns automatic sync off. [Apple: Syncing model data across a person's devices](https://developer.apple.com/documentation/swiftdata/syncing-model-data-across-a-persons-devices)

With CloudKit sync enabled, Apple's constraint table for SwiftData models states:

- The framework synchronizes changes concurrently and at opportune times, so CloudKit cannot enforce the `.unique` property option. Do not rely on `@Attribute(.unique)` for cloud-side uniqueness of a business attribute.
- All relationships must be optional. SwiftData sets the inverse automatically when it can reliably infer it from the schema; otherwise set the inverse explicitly before saving, because CloudKit processes changes in an indeterminate order.
- The iCloud servers do not guarantee atomic processing of relationship changes.
- The `.deny` delete rule is not supported.

These are the documented constraints for synced models, not a complete compatibility checklist. Models still need to be validated against the actual schema in the development environment; this reference does not provide a full model example that has not been validated against CloudKit. [Apple: SwiftData sync constraints](https://developer.apple.com/documentation/swiftdata/syncing-model-data-across-a-persons-devices) — the same limitations are stated for Core Data models in [Creating a Core Data model for CloudKit](https://developer.apple.com/documentation/coredata/creating-a-core-data-model-for-cloudkit).

## Distinguish three kinds of identity

1. **Business key** — an ordinary attribute that identifies the same business concept on every device, such as the "AA battery type" that each device seeds.
2. **CloudKit record identity** — a `CKRecord.ID` consists of a name string and a zone ID, and CloudKit identifies records by record ID within a specified database. Identical business attribute values do not mean identical record identity. [Apple: CKRecord.ID](https://developer.apple.com/documentation/cloudkit/ckrecord/id)
3. **Local persisted record identity** — SwiftData's `persistentModelID` on `PersistentModel`. When displaying records that may not yet be merged, do not use a possibly duplicated business key as the unique identity of each row; use an identity that distinguishes individual persisted records, and check its stability across insertion, saving, and updates. This reference does not assume that identity is the same across devices. [Apple: PersistentModel](https://developer.apple.com/documentation/swiftdata/persistentmodel)

SwiftData syncs through Core Data's `NSPersistentCloudKitContainer`, and business attributes map to ordinary CloudKit fields. Writing a fixed UUID into a business attribute therefore does not assign a shared `CKRecord.ID` to the mirrored records, and it does not promise automatic deduplication. [Apple: Reading CloudKit records for Core Data](https://developer.apple.com/documentation/coredata/reading-cloudkit-records-for-core-data)

## A minimal seeding procedure

When a device-constant list represents the same business concept on every device, a consistent business key can be used. The following is pseudocode for the local flow; implement it with the app's verified model and run it serially within the isolation domain that owns the `ModelContext`:

```text
For each default entry:
    Query the local store by the fixed business key
    If a record exists: keep it; do not re-insert; do not overwrite user edits
    If no record exists: insert the default entry
Save the context; surface fetch or save errors to the caller
```

Once inserted records have saved successfully and remain present, repeated local serial execution adds no duplicates, but "check, then insert" is not an atomic operation across concurrent contexts or devices. A fixed business key helps identify candidate duplicates; it provides no server-side uniqueness guarantee. The original author describes duplicates arising when devices seed offline before either knows the other exists; no verification record here reproduces that cross-device phenomenon.

Confirm these product and data conditions before adopting the procedure:

- If the user deletes a default entry, "insert when missing" puts it back; whether that restoration is allowed is a product decision.
- If the user has renamed or otherwise edited an entry, the existence check must not restore it to defaults.
- Records created with random IDs or an older model are not automatically migrated by a new business key; first decide how existing data is identified and mapped. Unmapped legacy records can coexist with the newly inserted defaults after the first pass. Once a fixed-key record has saved successfully and still exists, later serial queries hit it, so the same set is not inserted again on every run.
- User-created objects with the same name are not necessarily the same concept. When a reliable business key exists, decide by business semantics; do not merge on display name alone.

## Diagnosis and merge decisions

Start by counting **local** records per business key and inspecting each record's user edits and references. Multiple same-key objects are merge candidates, not proof of what can be deleted. A local count is also not direct proof of the CloudKit record count; when needed, check the corresponding database and zone in CloudKit Console.

Whether to accept duplicates, offer manual handling, or merge automatically is decided by that model's business semantics. Data referenced by other objects needs closer inspection of relationships and delete rules, but "is referenced" does not mean "must be merged".

Before implementing a merge, settle:

- How the surviving record is chosen, and whether every client and every repeated run uses a compatible strategy.
- What is kept when user edits conflict, and how silent data loss is avoided.
- How relationships are re-pointed, and which related objects the delete rules affect.
- Whether counts represent non-overlapping contributions. Only then consider adding them up; duplicated state or duplicated processing double-counts.

The original author suggests re-pointing relationships first and deleting the surplus records afterwards. That is a reasonable starting point for local operation design, **not a guarantee of multi-device atomicity or convergence**. Apple's constraint table states that the iCloud servers do not guarantee atomic processing of relationship changes; this reference provides no unverified automatic merge algorithm. [Apple: SwiftData sync constraints](https://developer.apple.com/documentation/swiftdata/syncing-model-data-across-a-persons-devices)

Do not make any flow depend on the assumption that all devices have finished syncing. No SwiftData trigger signal usable as such a guarantee has been verified. If a check is placed at launch, on entering a screen, or when handling related changes, still verify that the check and the merge can safely run more than once.

## Verification record and known boundaries

A recorded verification run — Xcode 27.0, Apple Swift 6.4, Swift 6 language mode, macOS 27 host, macOS 14 deployment target, `cloudKitDatabase: .none`, in-memory store, single-process serial execution — covered seven behavior groups:

1. Repeated seeding in the same context adds no records.
2. When a record with the same business key exists, it is not inserted again, and a user rename is not overwritten.
3. Serial seeding from different contexts on the same store adds no records.
4. After manually inserting a same-key record, local diagnostics return the duplicate count.
5. A UUID equality predicate executes correctly in that local test environment.
6. Re-seeding after deleting a default entry inserts it back.
7. Starting with two default entries carrying random IDs, seeding the two fixed-key defaults added a second set: 2 records became 4. The second serial pass added nothing, leaving 4 records.

These results support only the local conditions above. Manually inserting a duplicate object does not reproduce offline two-device sync; the sample model's CloudKit schema, iOS runtimes, migration, delete rules, and automatic merging were not verified.

To ship in an app: validate the development schema against the actual model first, then check offline first launch on two devices, user edits and deletions, relationship re-pointing, repeated merge runs, and concurrent clients. Model changes also require a compatibility assessment against the existing production schema; passing local tests does not mean the production schema or real sync behavior has passed. [Apple: schema initialization and promotion](https://developer.apple.com/documentation/swiftdata/syncing-model-data-across-a-persons-devices)

## Sources

- [Avoiding Duplicate Data When SwiftData Syncs Across Devices](https://thorsten-stark.de/posts/2026-08-06-Avoiding-Duplicate-Data-With-SwiftData-iCloud-Sync-Part-2/)
