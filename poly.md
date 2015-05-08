Table of contents:

1. [Essential info](#essential-info)
2. [Usage](#usage)
3. [Referencing or extending a polymorphic entity](#referencing-or-extending-a-polymorphic-entity)
4. [Multiple interface implementations](#multiple-interface-implementations)
5. [Property implementation with subquery](#property-implementation-with-subquery)
6. [Limit the implementation with filter (where)](#limit-the-implementation-with-filter-where)
7. [Subtype implementation using SQL query](#subtype-implementation-using-sql-query)
8. [Writing efficient queries from client application](#writing-efficient-queries-from-client-application)

## Essential info

Polymorphic concept is intended for implementing the application design pattern where multiple entities share common interface for reading data (defined as a list of common properties).

Similar features:

* Polymorphic concept is similar to object-oriented concept of class inheritance, but class inheritance is not used here.
* Polymorphic concept can sometimes be used as an alternative to the Extends concept.

## Usage

For example, `MoneyTransaction` data structure can have multiple forms: `BorrowMoney` and `LendMoney` (see the example below).

* In this example, `MoneyTransaction` can be seen as an *interface* that is *implemented* by the concrete subtypes `BorrowMoney` and `LendMoney`.
* `MoneyTransaction` is readable data structure, it returns **union of records** from  `BorrowMoney` and `LendMoney`.

Example:

    Module Demo
    {
        Polymorphic MoneyTransaction
        {
            DateTime EventDate;
            Money Amount;
        }
        
        Entity BorrowMoney
        {
            ShortString FromWhom;
            Money Amount;
            
            Is Demo.MoneyTransaction;
            // EventDate will be automatically added.
        }
        
        Entity LendMoney
        {
            ShortString ToWhom;
            Money Amount;
            
            Is Demo.MoneyTransaction
            {
                Implements Demo.MoneyTransaction.Amount '-Amount';
                // Amount in the MoneyTransaction related to the LendMoney record will have negative value.  
            }
        }
    }

Note that:

* The `MoneyTransaction.Amount` property is automatically mapped to the `BorrowMoney.Amount` property with the same name;
* In the `LendMoney` entity, the `MoneyTransaction.Amount` property is implemented by a specific SQL expression `-Amount`. See the generated SQL view `Demo.LendMoney_As_MoneyTransaction` to check the impact of that expression.

## Referencing or extending a polymorphic entity

A polymorphic data structure may be referenced or extended by other entites (Browse is also an extension).
Rhetos will check the **reference constraint** on data modifications.

    Entity TransactionComment
    {
        Reference MoneyTransaction;
        LongString Comment;
    }

Internally, the reference is implemented as a foreign key in database: from the `TransactionComment` table to the automatically generated `MoneyTransaction_Key` table.
The `MoneyTransaction_Key` table contains union of IDs from all MoneyTransaction subtypes: `BorrowMoney` and `LendMoney`.
The redundant IDs are automatically updated when inserting or deleting the subtype entities' records.

## Multiple interface implementations

An entity may implement multiple interfaces.
It can even implement the same interface multiple times, using additional string parameter *ImplementationName* to distinguish the implementations.

For example, the `TransferMoney` entity record may represent two money transactions: subtracting from one account and adding to the other account.

    Entity TransferMoney
    {
        ShortString From;
        ShortString To;
        Money Amount;
        
        Is Demo.MoneyTransaction; // Adding money using the 'Amount' value.
        
        Is Demo.MoneyTransaction 'Subtract'
        {
            Implements Demo.MoneyTransaction.Amount '-Amount';
        }
    }

See the generated SQL view `Demo.MoneyTransaction` to check that a `TransferMoney` record is included twice in the view.

## Property implementation with subquery

An SQL subquery may be used for polymorphic property implementation.

For example, one `LendMoney` instance can have multiple additions (`LendMoneyAddendum`) that will be summed as a single `TotalAddendum` implementation of `MoneyTransaction`:

    Entity LendMoneyAddendum
    {
        Reference LendMoney;
        Money AdditionalAmount;
    }
    
    Entity LendMoney
    {
        Is Demo.MoneyTransaction 'TotalAddendum'
        {
            Implements Demo.MoneyTransaction.Amount '(SELECT -SUM(AdditionalAmount) FROM Demo.LendMoneyAddendum)';
        }
    }

See the generated SQL view `Demo.LendMoney_As_MoneyTransaction_TotalAddendum` to check the impact of the subquery.

## Limit the implementation with filter (where)

The `Where` concept can be used to limit the items when will be incluced in the polymorphic implementation. The filter is defined by an SQL expression for the SQL query *WHERE part*.

In the following example, only items from `BorrowMoney` that are not `Forgotten` will be included in the `MoneyTransaction`.

    Entity BorrowMoney
    {
        ShortString FromWhom;
        Money Amount;
        Bool Forgotten;
        
        Is Demo.MoneyTransaction
        {
            Where 'Forgotten = 0';
        }
    }

See the generated SQL view `Demo.BorrowMoney_As_MoneyTransaction` to check the impact of the `Where` concept.

## Subtype implementation using SQL query

Instead of using property implementations (`Implements` keyword), a specific SQL query may be provided to implement the mapping between the subtype and the polymorphic entity.

This example is an alternative implementation of the `LendMoney` subtype (see the original implementation above):

    Entity LendMoney
    {
        ShortString ToWhom;
        Money Amount;
        
        Is Demo.MoneyTransaction
        {
            SqlImplementation "SELECT lm.ID, lm.EventDate, Amount = -lm.Amount FROM Demo.LendMoney lm"
            {
                AutoDetectSqlDependencies;
            }
        }
    }

`SqlImplementation` may cover more complex scenarios than per-property implementations (`Implements`),
but it lacks readability and extensibility such as adding a custom property to the subtype from another package.

## Writing efficient queries from client application

When reading a polymorphic entity, filtering by subtype can be evaluated efficiently in SQL Server if the filter is based on a subtype reference property.
For example, when reading records from `MoneyTransaction` of subtype `LendMoney` through REST API, the following filter should be used:

    http://localhost/Rhetos/rest/Demo/Perf/?filters=[{"Property":"LendMoney","Operation":"NotEqual","Value":null}]

The request above will be translated to an SQL query similar to `SELECT * FROM Demo.MoneyTransaction WHERE LendMoneyID IS NOT NULL`.
SQL Server will optimize the query's execution plan so that only `Demo.LendMoney` table will be scanned.

A similar optimization is done in database when filtering by subtype: `SELECT * FROM Demo.MoneyTransaction WHERE Subtype = 'Demo.LendMoney'`.
Unfortunately, if such filter is defined in LINQ query, it will generate a different WHERE part:
`WHERE Subtype = @p0`, and the LINQ query will cause reading both `Demo.LendMoney` and `Demo.BorrowMoney` tables.
