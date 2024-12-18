internal sealed class UnitOfWork : IUnitOfWork
{
    private readonly ApplicationDbContext _dbContext;
    private readonly IList<IUnitOfWorkCallback> _preCallback = new List<IUnitOfWorkCallback>();
    private readonly IList<IUnitOfWorkCallback> _postCallback = new List<IUnitOfWorkCallback>();

    public UnitOfWork(ApplicationDbContext dbContext) => _dbContext = dbContext;

    public async Task<int> SaveChangesAsync(CancellationToken cancellationToken)
    {
        var modifications = GetModifiedObjects();
        RunCallbacks(modifications, _preCallback);
        try
        {
            var changes = await _dbContext.SaveChangesAsync(cancellationToken);
            RunCallbacks(modifications, _postCallback);
            return changes;
        }
		catch (DbUpdateConcurrencyException ex)
		{
			throw new ConflictException(ex.Message, ex.InnerException);
		}
	}

    private IList<ModifiedObject> GetModifiedObjects()
    {
        var getChanges = new List<ModifiedObject>();

        var entries = _dbContext.ChangeTracker.Entries().Where(e => e.State != EntityState.Unchanged).ToList();
        foreach (var entry in entries)
        {
            var propertiesWithChanges = entry.State == EntityState.Added ?
                entry.Properties.ToArray() :
                entry.Properties.Where(prop => prop.IsModified && prop.CurrentValue != prop.OriginalValue).ToArray();

            var change = new ModifiedObject { Item = entry.Entity, Properties = propertiesWithChanges.Select(f => f.Metadata.Name).ToArray() };
            getChanges.Add(change);
        }

        return getChanges;
    }

    private class ModifiedObject
    {
        public object Item { get; set; } = null!;
        public String[] Properties { get; set; } = null!;
    }

    private void RunCallbacks(IList<ModifiedObject> changes, IList<IUnitOfWorkCallback> callbacks)
    {
        foreach (var change in changes)
        {
            foreach (var callback in callbacks)
            {
                var callbackMethods = callback.GetType().GetMethods().Select(f => new { Method = f, Attribute = (NotifiedByAttribute?)f!.GetCustomAttributes(typeof(NotifiedByAttribute), true).FirstOrDefault() }).Where(f => f.Attribute != null);
                foreach (var callbackMethod in callbackMethods)
                {
                    if (callbackMethod.Attribute!.Properties.Intersect(change.Properties).Any())
                    {
                        callbackMethod.Method.Invoke(callback, new[] { change.Item });
                    }
                }
            }
        }
    }

    public void AddPreCallback(IUnitOfWorkCallback callback)
    {
        _preCallback.Add(callback);
    }

    public void AddPostCallback(IUnitOfWorkCallback callback)
    {
        _postCallback.Add(callback);
    }
}
