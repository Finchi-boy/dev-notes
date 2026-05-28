### Spring Data JPA — query method naming

Method name **is** the query — Spring Data parses it and generates SQL automatically:

```java
findByEmail(String email)
// SELECT * FROM users WHERE email = ?

findByUsernameAndEmail(String username, String email)
// SELECT * FROM users WHERE username = ? AND email = ?

findByStatusOrPriority(TaskStatus status, Priority priority)
// SELECT * FROM tasks WHERE status = ? OR priority = ?

findByDeadlineBefore(LocalDateTime date)
// SELECT * FROM tasks WHERE deadline < ?

findByTitleContaining(String keyword)
// SELECT * FROM tasks WHERE title LIKE %?%

findAllByOrderByCreatedAtDesc()
// SELECT * FROM tasks ORDER BY created_at DESC

existsByEmail(String email)
// SELECT COUNT(*) > 0 FROM users WHERE email = ?

countByStatus(TaskStatus status)
// SELECT COUNT(*) FROM tasks WHERE status = ?
```

**Supported keywords:**
- `findBy`, `getBy`, `readBy` — SELECT
- `existsBy` — returns boolean
- `countBy` — returns count
- `deleteBy` — DELETE
- `And`, `Or` — combine conditions
- `Before`, `After` — for dates
- `Containing`, `StartingWith`, `EndingWith` — LIKE
- `OrderBy...Asc/Desc` — sorting

For complex queries that don't fit into method naming — use `@Query`:

```java
@Query("SELECT t FROM Task t WHERE t.userId = :userId AND t.status = :status")
List<Task> findByUserIdAndStatus(@Param("userId") UUID userId, @Param("status") TaskStatus status);
```
