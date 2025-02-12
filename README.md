# textpdf
creating pdf with large data
To implement audit logging and approval-based updates in EF Core 8.0, here's a structured approach you can follow:

1. Designing the Audit Log Tables
You'll need an audit log table to capture changes made to both parent and child entities. This table will store pending changes until they are approved.

Example Tables:
ParentTable: Your main parent entity.
ChildTable: Related to ParentTable.
AuditLog: Stores the changes before approval.
AuditLogDetail: Stores detailed property-level changes.
AuditLog Table:
csharp
Copy
Edit
public class AuditLog
{
    public int Id { get; set; }
    public string EntityName { get; set; }   // Name of the table (e.g., ParentTable or ChildTable)
    public int EntityId { get; set; }        // ID of the record being modified
    public string ActionType { get; set; }   // Insert, Update, Delete
    public DateTime CreatedAt { get; set; }
    public string CreatedBy { get; set; }    // User who made the change
    public bool IsApproved { get; set; } = false;
    public string ApprovedBy { get; set; }
    public DateTime? ApprovedAt { get; set; }

    public ICollection<AuditLogDetail> AuditLogDetails { get; set; }
}
AuditLogDetail Table:
csharp
Copy
Edit
public class AuditLogDetail
{
    public int Id { get; set; }
    public int AuditLogId { get; set; }
    public string PropertyName { get; set; }
    public string OldValue { get; set; }
    public string NewValue { get; set; }

    public AuditLog AuditLog { get; set; }
}
2. Intercepting Changes with EF Core ChangeTracker
Override the SaveChangesAsync method in your DbContext to capture changes and insert them into the AuditLog instead of updating the actual tables.

csharp
Copy
Edit
public class AppDbContext : DbContext
{
    public DbSet<ParentTable> ParentTables { get; set; }
    public DbSet<ChildTable> ChildTables { get; set; }
    public DbSet<AuditLog> AuditLogs { get; set; }
    public DbSet<AuditLogDetail> AuditLogDetails { get; set; }

    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        var auditEntries = OnBeforeSaveChanges();
        var result = await base.SaveChangesAsync(cancellationToken);
        await OnAfterSaveChanges(auditEntries);
        return result;
    }

    private List<AuditLog> OnBeforeSaveChanges()
    {
        ChangeTracker.DetectChanges();
        var auditEntries = new List<AuditLog>();

        foreach (var entry in ChangeTracker.Entries())
        {
            if (entry.Entity is AuditLog || entry.State == EntityState.Detached || entry.State == EntityState.Unchanged)
                continue;

            var auditEntry = new AuditLog
            {
                EntityName = entry.Entity.GetType().Name,
                EntityId = (int)entry.Property("Id").CurrentValue,
                ActionType = entry.State.ToString(),
                CreatedAt = DateTime.UtcNow,
                CreatedBy = "SystemUser", // Replace with current user
                AuditLogDetails = new List<AuditLogDetail>()
            };

            foreach (var property in entry.Properties)
            {
                if (property.IsTemporary)
                    continue;

                var propertyName = property.Metadata.Name;

                switch (entry.State)
                {
                    case EntityState.Modified:
                        if (property.OriginalValue?.ToString() != property.CurrentValue?.ToString())
                        {
                            auditEntry.AuditLogDetails.Add(new AuditLogDetail
                            {
                                PropertyName = propertyName,
                                OldValue = property.OriginalValue?.ToString(),
                                NewValue = property.CurrentValue?.ToString()
                            });
                        }
                        break;
                    case EntityState.Added:
                        auditEntry.AuditLogDetails.Add(new AuditLogDetail
                        {
                            PropertyName = propertyName,
                            OldValue = null,
                            NewValue = property.CurrentValue?.ToString()
                        });
                        break;
                    case EntityState.Deleted:
                        auditEntry.AuditLogDetails.Add(new AuditLogDetail
                        {
                            PropertyName = propertyName,
                            OldValue = property.OriginalValue?.ToString(),
                            NewValue = null
                        });
                        break;
                }
            }

            auditEntries.Add(auditEntry);
            entry.State = EntityState.Unchanged; // Prevent actual DB update
        }

        // Save audit logs to the database
        AuditLogs.AddRange(auditEntries);
        return auditEntries;
    }

    private async Task OnAfterSaveChanges(List<AuditLog> auditEntries)
    {
        if (auditEntries.Any())
        {
            await SaveChangesAsync();
        }
    }
}
3. Approving Changes and Applying to Database
You'll need a separate process or API endpoint for the approver to approve changes. Once approved, you'll read from the AuditLog and apply the changes to the actual tables.

Approve Changes Method:
csharp
Copy
Edit
public async Task ApproveChangesAsync(int auditLogId, string approver)
{
    var auditLog = await AuditLogs.Include(a => a.AuditLogDetails)
                                  .FirstOrDefaultAsync(a => a.Id == auditLogId && !a.IsApproved);
    if (auditLog == null) throw new Exception("Audit log not found or already approved.");

    switch (auditLog.ActionType)
    {
        case nameof(EntityState.Added):
            await ApplyAddition(auditLog);
            break;
        case nameof(EntityState.Modified):
            await ApplyModification(auditLog);
            break;
        case nameof(EntityState.Deleted):
            await ApplyDeletion(auditLog);
            break;
    }

    // Mark as approved
    auditLog.IsApproved = true;
    auditLog.ApprovedBy = approver;
    auditLog.ApprovedAt = DateTime.UtcNow;
    await SaveChangesAsync();
}

private async Task ApplyAddition(AuditLog auditLog)
{
    var entityType = Type.GetType($"YourNamespace.{auditLog.EntityName}");
    var entity = Activator.CreateInstance(entityType);

    foreach (var detail in auditLog.AuditLogDetails)
    {
        entityType.GetProperty(detail.PropertyName)?.SetValue(entity, Convert.ChangeType(detail.NewValue, entityType.GetProperty(detail.PropertyName).PropertyType));
    }

    Entry(entity).State = EntityState.Added;
    await SaveChangesAsync();
}

private async Task ApplyModification(AuditLog auditLog)
{
    var entityType = Type.GetType($"YourNamespace.{auditLog.EntityName}");
    var entity = await FindAsync(entityType, auditLog.EntityId);

    foreach (var detail in auditLog.AuditLogDetails)
    {
        entityType.GetProperty(detail.PropertyName)?.SetValue(entity, Convert.ChangeType(detail.NewValue, entityType.GetProperty(detail.PropertyName).PropertyType));
    }

    Entry(entity).State = EntityState.Modified;
    await SaveChangesAsync();
}

private async Task ApplyDeletion(AuditLog auditLog)
{
    var entityType = Type.GetType($"YourNamespace.{auditLog.EntityName}");
    var entity = await FindAsync(entityType, auditLog.EntityId);
    Entry(entity).State = EntityState.Deleted;
    await SaveChangesAsync();
}
4. Considerations
Transaction Grouping: If you need to group multiple changes under a single transaction, you can add a TransactionId field to AuditLog and log multiple entries under one ID.

Performance: This approach adds overhead to SaveChangesAsync, so ensure efficient querying, especially with large data sets.

Security: Restrict who can approve changes and log all approval actions.

