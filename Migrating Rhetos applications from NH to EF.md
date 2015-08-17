# Migrating Rhetos applications from NH to EF

When upgrading from **Rhetos v0.9** to **Rhetos v1.0**, the ORM framework is changed from NHibernate to Entity Framework.

## New features

* For reference properties, scalar Guid property is mapped to the database. Previously only navigation property was available in LINQ queries and in REST filter parameters. For example, the client had to use `item.Reference.ID` instead of `item.ReferenceID` in filters, now both options are available and `item.ReferenceID` is the preferred option.

## Breaking changes

## Heisenbugs

* Korištenje `updatedOld.Zip(updated` u **OnSaveUpdate** ili **OnSaveValidate** konceptu (ili u drugim konceptima koji insertaju u OnSave1Tag ili OnSave2Tag) je stohastičan jer je `updated` nakon upisa u bazu loadan sa QueryPersisted, koji ne garantira redoslijed.
    - => Koristiti `updatedNew` umjesto `updated` u OnSaveUpdate i OnSaveValidate, ako ne treba koristiti reference.
    - Isto za `inserted`.
    - Regex: `updatedOld\s+\.Zip\(updated\b`.

### Direct use of NHibernate

* `_executionContext.NHibernateSession.Clear()`
    - => `context.EntityFrameworkContext.ClearCache()` or `container.Resolve<IPersistenceCache>.ClearCache()`
* `_executionContext.NHibernateSession.Evict(item)`
    - => `context.EntityFrameworkContext.ClearCache(item)` or `container.Resolve<IPersistenceCache>.ClearCache(item)`
* `_executionContext.NHibernateSession.Query<{module}.{entity}>()`
    - => `_executionContext.EntityFrameworkContext.{module}_{entity}`
* SQL command using NHibernateSession:
    - `context.NHibernateSession.CreateSQLQuery(sql).ExecuteUpdate();`
    - => `context.SqlExecuter.ExecuteSql(sql)`
* SQL query using NHibernateSession:
    - `var sql = "SELECT * FROM mod.ent(:dateTime)"; return _executionContext.NHibernateSession.CreateSQLQuery(sql).AddEntity(typeof(mod.ent)).SetTimestamp("dateTime", parameter).List<mod.end>();`
    - => `context.SqlExecuter.ExecuteReader(...)`
    - or => `var sql = "SELECT * FROM {2}.{3}(@p0)"; return _executionContext.EntityFrameworkContext.Database.SqlQuery<{0}.{1}>(sql, parameter).ToList();`

### Querying data

* Simple instances of a data structure class do not have navigation properties, such as references, Base, Extends_, and similiar. The navigation properties are available in LINQ queries only. Simple instances are typically used in FilterBy and Computed, or created with the "new" operator.
* `IQueryable<{module}.{entity}>` => `IQueryable<Common.Queryable.{module}_{entity}>`
* Calling `AsQueryable()` on a list or an array of items:
    - To query data from database: `listOfItems.AsQuerayble()` => `entityRepository.QueryPersisted(listOfItems)`
    - To query loaded data, without using navigation properties: `listOfItems.AsQueryable()` => `entityRepository.QueryLoaded(listOfItems)`
    - Additionally, to use lazy evaluation of navigation properties (such as references) in the loaded data, add `LazyLoadReferences` to the data structure in the .rhe script.
* `Guid.Parse("...")` => `new Guid("...")`
* `SqlLike` => `Like`

### Using repositories

* `Implements` concept is now limited to scalar properties. In order to use navigation properties in the interface, use `ImplementsQueryable`.
* `IWritableRepository` interface is removed (a new one exists with a generic parameter)
    - => If referencing *ServerDom.dll*, use the entity's repository `Insert/Update/Delete/Save` functions directly. Otherwise, use the `GenericRepository`.
    - Add `using Rhetos.Dom.DefaultConcepts;` for extension methods.
* `IFilterRepository` interface is removed.
    - => If referencing *ServerDom.dll*, use the entity's repository `Filter` function directly. Otherwise, use the `GenericRepository`.
    - Add `using Rhetos.Dom.DefaultConcepts;` for extension methods.
* Avoid `GenericRepository` for querying when using navigation properties.
    -  `GenericRepository<Module.Entity>.Query(...)` => `repository.Module.Entity.Query(...)`.
* Renamed repository functions:
    - `All()` => `Load()` /// It will always return materialized items (a list or an array).
    - `Read(...)` => `Load(...)` /// It will always return materialized items (a list or an array).
    - `ReadOrQuery(...)` => `Read(...)`
    - `Load<FilterType>()` => `Load(null, typeof(FilterType))`
    - `Query<FilterType>()` => `Query(null, typeof(FilterType))`
    - `Read<FilterType>()` => `Read(null, typeof(FilterType))`

### Troubleshooting

* Error: `'moduleX.entityX' does not contain a definition for 'propertyX'`
    - In resulting query of a ComposableFilterBy or a QueryableExtension:
        + Use `new Common.Queryable.module_entity` instead of `new module.entity`.
    - Else:
        + Error `'...' does not contain a definition for '... ReferenceProperty'`
            * On assignment use `ReferencePropertyID = item.ID` instead of `ReferenceProperty = item`
            * On read use `ReferencePropertyID` or a LINQ query.
        + Error `... does not contain a definition for 'Base'` => when creating a new extension class instance in FilterBy or in Computed, set the ID property (`ID = item.ID`), not the Base property (`Base = item`).
        + Use `entityRepository.QueryLoaded(items).Select(item => item.Reference.Name)` instead of `items.Select(item => item.Reference.Name)`.
* `'..._Repository' does not contain a definition for 'Insert'` => `using Rhetos.Dom.DefaultConcepts;`
* `'..._Repository' does not contain a definition for 'Update'` => `using Rhetos.Dom.DefaultConcepts;`
* `'..._Repository' does not contain a definition for 'Delete'` => `using Rhetos.Dom.DefaultConcepts;`
* `The entity types 'x' and 'y' cannot share table 'z' because they are not in the same type hierarchy or do not have a valid one to one foreign key relationship with matching primary keys between them.`
    - If any of the structures is a LegacyEntity with an explicitly set table as a source view (LegacyEntity name 'tableX' 'tableX'), change it to a LegacyEntity with autogenerated view (LegacyEntity name 'tableX').
* Joining multiple queryes throws exception ``NotSupportedException: LINQ to Entities does not recognize the method 'System.Linq.IQueryable`1[Common.Queryable.....] Query()' method``
    - => Separately create the queries `var query = repository.module.entity.query();` before joining them.
* Error `System.NullReferenceException: Object reference not set to an instance of an object.` on reference property.
    - Problem is cause when lazy loading references after modifying data in database, or after calling ClearCache() => Use query to avoid lazy loading references.
* Error `InvalidOperationException: The '...' property on '...' could not be set to a 'System.Int32' value. You must set this property to a non-null value of type 'System.Boolean'. `
    - => Instead of selecting 1 or 0 in the SQL query, select `CONVERT(BIT, 1)` or `CONVERT(BIT, 0)`. Check the column type of the generated view; it should be `bit` instead of `int`
* Error `Unable to cast the type '...' to type 'Rhetos.Dom.DefaultConcepts.IEntity'. LINQ to Entities only supports casting EDM primitive or enumeration types.`
    - => Instead of using generic function type for GenericRepository or IQueryableRepository, use IEntity.
* Error ``LINQ to Entities does not recognize the method 'System.Linq.IQueryable`1[...] Query()' method, and this method cannot be translated into a store expression.``
    - => store the subquery in a separate variable, for example:
        + `{`
        + `    var subquery = repository.Common.PrincipalPermission.Query();`
        + `    return repository.Common.Principal.Query().Where(p => subquery.Select(pp => pp.PrincipalID).Contains(p.ID)).Dump("subquery variable");`
        + `}`
    - or => Use `entityRepository.Subquery` property for the subquery, instead of `entityRepository.Query()` function.
* Error `Unable to create a constant value of type '...'. Only primitive types or enumeration types are supported in this context.`
    - Where(item => item.Reference == other) => Where(item => item.ReferenceID == other.ID)
    - Where(item => someList.Contains(item.Reference)) => Where(item => someQuery.Contains(item.Reference)).
* Error `'Rhetos.Persistence.IPersistenceTransaction' does not contain a definition for 'ClearCache'`
    - `PersistenceTransaction.ClearCache()` => `EntityFrameworkContext.ClearCache()`
    - Or => `context.EntityFrameworkContext.ClearCache()`
    - Or => `container.Resolve<IPersistenceCache>.ClearCache()`
* Error `There is already an open DataReader associated with this Command which must be closed first.` This can happen if you execute a query while iterating over the results from another query. Ima par rješenja:
    - A) Ne koristiti lazy evaluaciju referenci, nego odmah dohvatiti potrebne referencirane podatke u inicijalnom LINQ queryiju.
    - B) Ako ne može izbjeći lazy load referenci, onda izmijeniti iteriranje (foreach ili slično) da se ne izvršava direktno na LINQ query-ju, nego na loadanim podacima. Npr. `foreach (var item in somequery)` => `foreach (var item in somequery.ToList())`.
        Koristiti `ComposableFilter` umjesto `ItemFilter`.
    - C) Uključiti MARS u connection string-u.
* Only parameterless constructors and initializers are supported in LINQ to Entities.
    - Ukloniti konstruktor s parametrom iz LINQ upita. Npr. `item => item.Created < new DateTime(DateTime.Today.Year, 1, 1);` => `{ var thisYear = new DateTime(DateTime.Today.Year, 1, 1); return item => item.Created < thisYear; }`
* NotSupportedException: The specified type member 'Base' is not supported in LINQ to Entities. Only initializers, entity members, and entity navigation properties are supported.
    - => Dodati Base property u LINQ query od extenzije, ako nedostaje. Npr.: query.Select(item => new Common.Queryable.NekaEkstenzija { ID = item.ID, Base = item, ... }.
* Using your own method in LINQ query results with NotSupportedException: `LINQ to Entities does not recognize the method '...'`.
* Error `NotSupportedException: LINQ to Entities does not recognize the method 'Boolean ExecuteValidationMethod`
    - Umjesto `ItemFilter` koristiti `FilterBy` ako ne trebaju lazy reference
    - ili `ComposableFilterBy`, ako trebaju lazy reference, sa modificiranim snippetom:
        + `(items, repository, parameter) => items.ToList().Where(item => OmegaCommonConcepts.Plugin.Utility.ExecuteValidationMethod("{0}", "{1}", item, repository)).AsQueryable()`
* Error ExtComposableFilterBy threw exception: ``System.InvalidCastException: Unable to cast object of type 'System.Data.Entity.Infrastructure.DbQuery`1[...]' to type 'System.Linq.IQueryable`1[Common.Queryable....]'.``
    - => Funkcija koja implementira composable filter mora imati parametre tipa `IQueryable<Common.Queryable.module_entity>` umjesto `IQueryable<module.entity>`.
