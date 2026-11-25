# Storage

## What neui stores

| What | Holds | Default place | Kept until |
| --- | --- | --- | --- |
| Threads | Each thread's name, rounds, compactions and records. | Memory, one store per session. | The session ends. |
| Uploads | Each thread's files: the ones the user attached (`user_uploads/`) and the ones the agent stored (`agent_uploads/`). | A temporary directory, one per session. | The session ends. |
| Workspace | The files the agent and your tools work on, one directory per thread. | The same temporary directory. | The thread is closed. |

A session is the Doors session.

Threads and uploads go through an interface you can replace: `api.MessageStorage` and `api.UploadStorage`. The workspace is always a local directory.

The workspace is created on the first write. It is deleted when its tab closes or switches to another thread. A file that must stay with the thread goes to uploads: the agent copies it to `agent_uploads/<name>`.

One `MessageStorage` value holds one user's threads, so give each session the storage of its user. An `UploadStorage` is keyed by thread, so one value serves every user. Neui generates the thread IDs.

Deleting a thread deletes its uploads. Cloning a thread copies them. Rewinding deletes the user's files attached to the removed rounds.

## Interfaces

Both storages are called from several goroutines at once. The godocs in the `api` package hold the full rules of each method.

### MessageStorage

```go
type MessageStorage interface {
	ThreadList(ctx context.Context, from, to time.Time, asc bool, limit int, name string) ([]Thread, error)
	ThreadGet(ctx context.Context, thread ThreadID) (Thread, error)
	ThreadCreate(ctx context.Context, id ThreadID) (Thread, error)
	ThreadBump(ctx context.Context, thread ThreadID) (Thread, error)
	ThreadRename(ctx context.Context, id ThreadID, seq Seq, name string) (Seq, error)
	ThreadCut(ctx context.Context, id ThreadID, seq Seq, round RoundID) (Seq, error)
	ThreadClone(ctx context.Context, source ThreadID, target ThreadID) (Thread, error)
	ThreadDelete(ctx context.Context, id ThreadID) error

	RoundInput(ctx context.Context, thread ThreadID, seq Seq, input Input) (RoundID, Seq, error)
	RoundOk(ctx context.Context, thread ThreadID, seq Seq, round RoundID, usage EventUsage, content Indexed[Content]) (Seq, error)
	RoundRefuse(ctx context.Context, thread ThreadID, seq Seq, round RoundID, usage EventUsage, content Indexed[Content]) (Seq, error)
	RoundAbort(ctx context.Context, thread ThreadID, seq Seq, round RoundID, usage EventUsage, content Indexed[Content]) (Seq, error)
	RoundError(ctx context.Context, thread ThreadID, seq Seq, round RoundID, err string, usage EventUsage, content Indexed[Content]) (Seq, error)
	RoundResult(ctx context.Context, thread ThreadID, seq Seq, round RoundID, result map[string]ContentToolResult) (Seq, error)
	RoundRange(ctx context.Context, thread ThreadID, start RoundID, end RoundID) iter.Seq2[Round, error]
	RoundBefore(ctx context.Context, thread ThreadID, start RoundID, limit int) ([]Round, error)
	RoundAfter(ctx context.Context, thread ThreadID, start RoundID, limit int) ([]Round, error)
	RoundSearch(ctx context.Context, thread ThreadID, exclude bool, query string, from *Cursor, inclusive bool, asc bool, limit int) ([]Hit, error)

	CompactionCreate(ctx context.Context, thread ThreadID, seq Seq, compaction Compaction) (Seq, error)
	CompactionGet(ctx context.Context, thread ThreadID, beforeTail RoundID) (Compaction, error)

	RecordAdd(ctx context.Context, thread ThreadID, seq Seq, origin RoundID, kind string, attrs map[string]any, data any) (RecordID, Seq, error)
	RecordPin(ctx context.Context, thread ThreadID, seq Seq, pin RoundID, records []RecordID) (Seq, error)
	RecordList(ctx context.Context, thread ThreadID) ([]Record, error)
}
```

**Versioning.** Each thread has a write serial, `Seq`. A method that takes `seq` writes only when the thread is at that `Seq`. It then raises `Seq` and returns the new value. With any other `seq` it writes nothing and returns `api.ErrDrift`. `ThreadBump` raises `Seq` without writing content. Neui calls it when it opens a thread, so whoever held the thread before gets `ErrDrift` on its next write.

**Errors.**

- A missing thread or round is `api.ErrNotFound`. So is an unknown record ID in `RecordPin`, which then pins nothing.
- `ThreadCreate` and `ThreadClone` return `api.ErrExists` when the ID is taken.
- `ThreadDelete` of a missing thread returns nil.
- `CompactionGet` returns `api.NoCompaction` and a nil error when there is none.

**Threads.**

- Neui calls `ThreadCreate` on the thread's first message, with an ID it chose.
- `ThreadList` returns every thread of the storage, ordered by `Updated`. A content write sets `Updated`. `ThreadBump` and `ThreadRename` leave it.
- `ThreadCut` removes `round` and every round after it. It also removes the compactions with `Tail >= round` and the records with `Origin >= round`, and unpins the records with `Pin >= round`. `api.NoRoundID` cuts everything.
- `ThreadClone` copies the name, rounds, compactions and records. `ThreadDelete` removes them. Neither touches uploads: neui copies and deletes those itself.

**Rounds.**

- `RoundInput` creates a round and returns its ID. IDs grow within the thread.
- `RoundResult` replaces the round's stored results.
- `RoundRange` returns whole rounds, with `PinnedRecords` and their `Data`. `RoundBefore` and `RoundAfter` return rounds with empty `Results` and records without `Data`.
- `RoundSearch` may return no hits. Then the chat's message search finds nothing.

**Records.** `RecordAdd` stores a record unpinned and returns its ID. `RecordList` returns the unpinned records in ID order.

### UploadStorage

```go
type UploadStorage interface {
	FilePut(ctx context.Context, threadID ThreadID, origin Origin, name string, mime string, r io.Reader, draft bool) (File, error)
	FileCommit(ctx context.Context, threadID ThreadID, origin Origin, name string) error
	FileDiscardDrafts(ctx context.Context, threadID ThreadID, origin Origin, keep []string) error
	FileStat(ctx context.Context, threadID ThreadID, origin Origin, name string) (File, error)
	FileRead(ctx context.Context, threadID ThreadID, origin Origin, name string, off int64, n int64) (io.ReadCloser, error)
	FileList(ctx context.Context, threadID ThreadID, origin Origin) iter.Seq2[File, error]
	FileDelete(ctx context.Context, threadID ThreadID, origin Origin, name string) error
	FileDeleteAll(ctx context.Context, threadID ThreadID) error
}
```

A file's key is the thread, the origin and the name. The origin is `api.OriginUser` or `api.OriginAgent`. The same name under each origin is two files.

**Writes.**

- A stored file never changes. `FilePut` on a taken name returns `api.ErrExists` and writes nothing.
- `FilePut` stores the whole file or nothing.
- `FilePut` returns the `File` with `Size` and `CreatedAt` filled.
- A name is one path element: not empty, not `.` or `..`, no separator. Return `api.ErrInvalidName` for any other name, and for one your storage can't hold.
- Refuse an empty `threadID` and an origin that is not `Valid`.

**Drafts.** A file the user attaches is stored at once with `draft` set. Sending the message calls `FileCommit`, which clears `File.Draft`. Change only the file's metadata there. `FileDiscardDrafts` deletes the origin's drafts except the names in `keep`. Neui calls it when a thread opens and when a message is sent.

**Reads.**

- `FileRead` returns the bytes `[off, off+n)`. A negative `n` reads to the end. A range past the end is cut short.
- `off` at or past the end is `api.ErrOutOfRange`. One case is not an error: `off` 0 with a negative `n` on an empty file reads empty.
- `FileList` returns the origin's files, drafts included, sorted by name in byte order.

**Missing keys.**

- `FileStat`, `FileRead` and `FileCommit` return `api.ErrNotFound`.
- `FileDelete`, `FileDeleteAll` and `FileDiscardDrafts` return nil.
- `FileList` returns nothing.

## Defaults

With no storage option, `neui.Enable` uses:

- An in-memory `MessageStorage`, one per session.
- A directory `neui-files-*` for uploads and workspaces, one per session. Neui creates it on the session's first `Enable` and removes it when the session ends. `Enable` returns an error when it can't create it.

The directory is created under `FilesConfig.TempDir`, by default `os.TempDir()`:

```go
neui.WithFilesConfig(api.FilesConfig{
	TempDir: "/var/lib/myapp/tmp",
})
```

With your own `UploadStorage`, workspaces still live there. One workspace holds up to `MaxFiles` files (default 10000) and `MaxBytes` bytes (default 1 GiB).

## Shipped storages

| Storage | Interface | Create | Enable |
| --- | --- | --- | --- |
| SQLite | `MessageStorage` | `nekit.NewSQLiteMessages(db)` | `neui.OptionMessageStorage` |
| Disk | `UploadStorage` | `nekit.NewDiskUploads(path)` | `neui.WithUploadStorage`, `neui.OptionUploadStorage` |
| S3 | `UploadStorage` | `nekit.NewS3Uploads(client, bucket, prefix)` | `neui.WithUploadStorage`, `neui.OptionUploadStorage` |

`neui.WithUploadStorage` is an app option for `doors.NewApp` and sets the uploads of every session. `neui.OptionUploadStorage` is an `Enable` option and overrides it for one session. Message storage has only the `Enable` option.

### SQLite

One database holds the threads of all users:

```go
import _ "modernc.org/sqlite"

db, err := sql.Open("sqlite", "neui.db")
...
threads, err := nekit.NewSQLiteMessages(db)
...
```

`User` returns one user's threads:

```go
err := neui.Enable(ctx, neui.OptionMessageStorage(threads.User(userID)))
```

- Import a SQLite driver of your choice. You open `db` and close it after the app stops.
- `NewSQLiteMessages` creates or migrates neui's tables and turns on WAL. It returns an error on a database written by a newer neui.
- Give neui a database file of its own. It keeps its schema version in `PRAGMA user_version`.
- `""` is a user ID like any other: every session that passes it shares the same threads.

### Disk

```go
files, err := nekit.NewDiskUploads("uploads")
...
app := doors.NewApp(..., neui.WithUploadStorage(files))
```

- `NewDiskUploads` creates the directory when it is missing. It returns an error when the path is not a writable directory.
- Open a directory from one process only. Opening it discards unfinished uploads.
- A file name longer than 255 bytes is refused with `api.ErrInvalidName`.

### S3

```go
client, err := minio.New(endpoint, &minio.Options{
	Creds:  credentials.NewStaticV4(keyID, secret, ""),
	Secure: true,
})
...
files := nekit.NewS3Uploads(client, "my-bucket", "neui")
```

- `client` is a `*minio.Client` from `github.com/minio/minio-go/v7`. It works with any S3-compatible service that supports conditional writes (`If-None-Match`), as AWS S3 and MinIO do.
- The bucket must exist. `prefix` is the key prefix of every file, `""` for the bucket root.
- Each upload buffers up to 16 MiB in memory.
- A file whose key, the prefix, thread ID and name together, is longer than 1024 bytes is refused with `api.ErrInvalidName`.
