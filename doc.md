## Contents

1. [Introduction](#intr)
2. [Prerequisites](#prer)
3. [Apatite Module](#apati)
4. [Registering Model](#regis)
  - 4.1 [Static method getModelDescriptor](#regisstati)
  - 4.2 [Creating ApatiteModelDescriptor](#regiscreat)
5. [Login / Logout](#login)
6. [Tables / Columns](#table)
7. [Mapping](#mappin)
  - 7.1 [Simple Mapping](#mappinsimpl)
  - 7.2 [One To One Mapping](#mappinonetoone)
  - 7.3 [One To Many Mapping](#mappinonetomany)
  - 7.4 [Inheritance Mapping](#mappininherit)
8. [Reading Objects](#readi)
  - 8.1 [Creating Query](#readicreat)
  - 8.2 [Executing Query](#readiexecu)
  - 8.3 [Simple Query Examples](#readisimpl)
  - 8.4 [One To One Query Examples](#readionetoone)
  - 8.5 [Fetching Attribute Values](#readifetch)
  - 8.6 [Fetching Function Values](#readifunc)
  - 8.7 [Exists and Not Exists Query](#readiexis)
  - 8.8 [Cursor Stream](#readicurs)
9. [Creating and Deleting Objects](#creat)
10. [Changing and Saving Objects](#chang)
11. [Session Cache](#sessi)
12. [Proxies](#proxi)
13. [Hookup Methods](#hookupm)
14. [Connection Pooling](#conne)
15. [SQLs Generation / Execution](#sqlgen)
16. [Public API](#publicapi)
17. [Tests](#tests)

## <a name="intro"></a> 1. Introduction

[Apatite](https://github.com/apatitejs/apatite) is a persistence framework for Node.js to query, update and delete objects from supported databases.


## <a name="prereq"></a> 2. Prerequisites

 - Node version >= 6.1.0.
 - One of the following modules for database:
  - [oracledb](https://github.com/oracle/node-oracledb) if you plan to use Oracle: $ npm install oracledb@4.1.0
  - [pg](https://github.com/brianc/node-postgres) if you plan to use Postgres: $ npm install pg@6.1.5
  - [mysql](https://github.com/mysqljs/mysql) if you plan to use Mysql: $ npm install mysql@2.17.1
  - [tedious](https://github.com/tediousjs/tedious) if you plan to use Microsoft SQL Server: $ npm install tedious@2.0.0, optionally for connection pool: [tedious-connection-pool](https://github.com/tediousjs/tedious-connection-pool) : $ npm install tedious-connection-pool@1.0.5
  - [sqlite3](https://github.com/mapbox/node-sqlite3) if you plan to use Sqlite: $ npm install sqlite3@3.1.8


## <a name="apati"></a> 3. Apatite Module

The apatite module exports the class Apatite.

```js
var Apatite = require('apatite')
```

You can get an instance of Apatite by calling one of the available static methods:

```js
// Oracle
Apatite.forOracle(connOptions)
```

```js
// Postgres
Apatite.forPostgres(connOptions)
```

```js
// Mysql
Apatite.forMysql(connOptions)
```

```js
// Sqlite
Apatite.forSqlite(connOptions);
```

The ```connOptions``` parameter must be an object with following properties of type string:

userName, password, connectionInfo

Example:

```js
var connOptions = { userName: 'apatite', password: 'apatite', connectionInfo: 'localhost/apatite' };
```

For Oracle, ```connOptions``` would passed to getConnection method as:

```
user: connOptions.userName,
password: connOptions.password,
connectString: connOptions.connectionInfo
```

Refer to [oracledb](https://github.com/oracle/node-oracledb) documentation for infos.

For Postgres, ```connOptions``` would passed to connect method as:

```
`postgres://${connOptions.userName}:${connOptions.password}@${connOptions.connectionInfo}`
```

Refer to [pg](https://github.com/brianc/node-postgres) documentation for infos.

For Sqlite, ```connOptions.connectionInfo``` would be passed as parameter to the Database class constructor.

```
new sqlite3.Database(connOptions.connectionInfo)
```

Refer to [sqlite3](https://github.com/mapbox/node-sqlite3) documentation for infos.

To log all sql's sent to the database, call the method enableLogging() on Apatite instance. To disable, call disableLogging().


## <a name="regis"></a> 4. Registering Model

You can register your model in two ways:

 - By creating a static method in your class and registering your model to apatite.
 - By creating a descriptor and registering the descriptor to apatite.

### <a name="regisstati"></a> 4.1 Static method getModelDescriptor

Create a static method named "getModelDescriptor" in your class. When creating a new session for the first time, this static method is called with an instance of Apatite class as parameter.

Example:

```js
class Department {
    constructor() {
    }

    static getModelDescriptor(apatite) {
        var table = apatite.newTable('DEPT');
        var modelDescriptor = apatite.newModelDescriptor(this, table);

        return modelDescriptor;
    }
}

apatite.registerModel(Department)
```

### <a name="regiscreat"></a> 4.2 Creating ApatiteModelDescriptor

```js
var descriptor = apatite.newModelDescriptor(Department, apatite.newTable('DEPT'));
apatite.registerModelDescriptor(descriptor);
```

You could also create descriptor from a simple object:

```js
var object = {
    table: 'DEPT',
    model: this,
    mappings: [
        {attr: 'oid', col: 'OID', pk: true, type: 'serial'},
        {attr: 'name', col: 'NAME', type: 'varchar', length: 100},
        {attr: 'employees', toMany: {modelName: 'Employee', toOneName: 'department', cascadeOnDelete: true, orderBy: [{attr: 'name', desc: true}]}}
    ]
}
var descriptor = apatite.newDescriptorFromObject(object)
...
...
object = {
    table: 'EMP',
    model: this,
    mappings: [
        {attr: 'oid', col: 'OID', pk: true, type: 'serial'},
        {attr: 'name', col: 'NAME', type: 'varchar', length: 100},
        {attr: 'salary', col: 'SALARY', type: 'decimal', length: 12, precision: 2},
        {attr: 'joinedOn', col: 'JOINEDON', type: 'date'},
        {attr: 'department', toOne: {modelName: 'Department', isLeftOuterJoin: true, fromCols: [{col: 'DEPTOID', type: 'int', length: 10}], toCols: [{table: 'DEPT', col: 'OID'}]}}
    ]
}
descriptor = apatite.newDescriptorFromObject(object)
```


## <a name="login"></a> 5. Login / Logout

To login to a database, you need to create an apatite session. To create a session call the method newSession on an apatite instance:

```js
apatite.newSession(function (err, session) {
})
```

The newSession takes a callback function as an arugment with two parameters. The first parameter is an error object in case of error else null. The second parameter is an instance of ApatiteSession.

To logout from database call the method end on an instance of ApatiteSession:

```js
session.end(function (err) {
})
```


## <a name="table"></a> 6. Tables / Columns

When creating a descriptor, you need an instance of ApatiteTable. Call the method newTable on an instance of Apatite. This method takes one argument, the name of the table.

```js
apatite.newTable('DEPT')
```

To add columns to the table, call the method addNewColumn. This method takes two arguments. The first argument is the name of the column, and the second argument is an instance of one of the subclasses ApatiteDatatype. The instances of subclasses of ApatiteDatatype can be created from the ApatiteDialect instance which is stored in the property of Apatite class. 

```js
var column = table.addNewColumn('OID', apatite.dialect.newSerialType())

// Integer of length 10
var column = table.addNewColumn('INTCOL', apatite.dialect.newIntegerType(10))

// Varchar of length 50
var column = table.addNewColumn('CHARCOL', apatite.dialect.newVarCharType(50))

var column = table.addNewColumn('DATECOL', apatite.dialect.newDateType())

// Decimal specifying length, precision
var column = table.addNewColumn('DECIMALCOL', apatite.dialect.newDecimalType(5,2))
```

To set the primary key for the table, call the method bePrimaryKey() on the ApatiteColumn instance.

```js
column.bePrimaryKey()
```

To specify a converter to convert the values from/to database, call the method setConverters on the ApatiteColumn instance.
```js
column.setConverters(function (value) {
    return value === 'Lion' ? 'Cat' : value // value for database
}, function (value) {
    return value === 'Cat' ? 'Lion' : value // value for object
})
```



## <a name="mappin"></a> 7. Mapping

To map model properties with table columns, you need to create an instance of one of the subclasses of ApatiteMapping. It can be created with the available methods in ApatiteModelDescriptor. An ApatiteModelDescriptor instance can be created by calling the method newModelDescriptor on Apatite instance. This method takes two arguments. The first argument is the class you would like to create the descriptor for and the second argument is the table. 

Example:

```js
apatite.newModelDescriptor(Department, table)
```

### <a name="mappinsimpl"></a> 7.1 Simple Mapping

```js
descriptor.newSimpleMapping(attributeName, column)
```

Example:

```js
modelDescriptor.newSimpleMapping('name', column)
```
attributeName: A string specifying the name of the property.
column: An instance of ApatiteColumn.

### <a name="mappinonetoone"></a> 7.2 One To One Mapping

```js
descriptor.newOneToOneMapping(attributeName, toModelName, columns, toColumns)
```

attributeName: A string specifying the name of the property.

toModelName: A string specifying the model name.

columns: Array of ApatiteColumn instances.

toColumns: Array of ApatiteColumn instances.

Example (one to one mapping from Employee to Department):

```js
...
var deptTable = apatite.getTable('DEPT')
var deptOIDColumn = deptTable.getColumn('OID')
var column = empTable.addNewColumn('DEPTOID', apatite.dialect.newIntegerType(10))
modelDescriptor.newOneToOneMapping('department', 'Department', [column], [deptOIDColumn])
...
```
If you want apatite to issue a left outer join for the one to one mapping, call the method beLeftOuterJoin() on the mapping.
```js
...
var mapping = modelDescriptor.newOneToOneMapping('department', 'Department', [column], [deptOIDColumn])
mapping.beLeftOuterJoin()
...
```


### <a name="mappinonetomany"></a> 7.3 One To Many Mapping

```js
newOneToManyMapping(attributeName, toModelName, toOneToOneAttrName)
```

attributeName: A string specifying the name of the property.

toModelName: A string specifying the model name.

toOneToOneAttrName: A string specifying the one to one attribute name in the child class.

Example (one to mapping mapping from Department to Employee):

```js
var toManyMapping = modelDescriptor.newOneToManyMapping('employees', 'Employee', 'department')

//Delete all employees when department is deleted
toManyMapping.cascadeOnDelete()

//Specify order by when fetching employees
var query = apatite.newToManyOrderQuery()
query.orderBy('firstName') // query.orderBy('firstName').desc() to order in descending
toManyMapping.setOrderByQuery(query)
``` 

### <a name="mappininherit"></a> 7.4 Inheritance Mapping

```js
inheritFrom(superDescriptor, typeFilterQuery);
```

superDescriptor: An instance of ApatiteModelDescriptor of super class.

typeFilterQuery: An instance ApatiteTypeFilterQuery specifying the type.

Example:

Shape is an abstract class with two sub classes: Circle and ShapeWithVertex. The subclasses are identified by an attribute named shapeType.

```js
class Shape {
    constructor() {
        this.oid = 0
        this.name = ''
        this.shapeType = 0
    }

    static getModelDescriptor(apatite) {
        var table = apatite.newTable('SHAPE')
        var modelDescriptor = apatite.newModelDescriptor(Shape, table)

        var column = table.addNewColumn('OID', apatite.dialect.newIntegerType(10))
        column.bePrimaryKey()
        modelDescriptor.newSimpleMapping('oid', column)

        column = table.addNewColumn('NAME', apatite.dialect.newVarCharType(100))
        modelDescriptor.newSimpleMapping('name', column)

        column = table.addNewColumn('SHAPETYPE', apatite.dialect.newIntegerType(1))
        modelDescriptor.newSimpleMapping('shapeType', column)

        return modelDescriptor;
    }
}

class Circle extends Shape {
    constructor() {
        super();
        this.shapeType = 1;
    }

    static getModelDescriptor(apatite) {
        var modelDescriptor = apatite.newModelDescriptor(Circle, apatite.getTable('SHAPE'));
        // Inherit the descriptor from Shape
        modelDescriptor.inheritFrom(apatite.getModelDescriptor(Shape), apatite.newTypeFilterQuery().attr('shapeType').eq(1));

        return modelDescriptor;
    }
}

class ShapeWithVertex extends Shape {
    constructor() {
        super();
        this.shapeType = 2;
        this.numberOfVertices = 0;
    }

    static getModelDescriptor(apatite) {
        var modelDescriptor = apatite.newModelDescriptor(ShapeWithVertex, apatite.getTable('SHAPE'));
        // Inherit the descriptor from Shape
        modelDescriptor.inheritFrom(apatite.getModelDescriptor(Shape), apatite.newTypeFilterQuery().attr('shapeType').eq(2));

        var column = apatite.getTable('SHAPE').addNewColumn('NOOFVERTICES', apatite.dialect.newIntegerType(2));
        modelDescriptor.newSimpleMapping('numberOfVertices', column);

        return modelDescriptor;
    }
}
```


## <a name="readi"></a> 8. Reading Objects

### <a name="readicreat"></a> 8.1 Creating Query

To read objects you need to create an instance of ApatiteQuery. You can create an instance of ApatiteQuery by calling the method newQuery on ApatiteSession. The method takes one argument which specifed the class for which you wish to execute the query.

```js
var query = session.newQuery(Department)
```

You could also create a query from simple array.

```js
const query = session.newQueryFromArray(Department, [['name', '=', 'Sales']])
```

### <a name="readiexecu"></a> 8.2 Executing Query

To execute the query, call the method execute on the query. The method takes one argument which is a callback function. If the callback argument of the execute method is not provided, an instance of promise is returned. The callback function is called with two arguments. The first an error object in case of error or null. The second an array containing the objects. In case of promise, array containing the objects is passed to the resolve function and error is passed to the reject function.

Example:

```js
query.execute(function(err, departments) {
    if (err) {
        console.error(err.message);
        return;
    }
    console.log(JSON.stringify(departments));
});
```

```js
//Promise
var promise = query.execute();
promise.then(function(departments) {
    console.log(JSON.stringify(departments));
}, function(err) {
    console.error(err.message);
});
```

### <a name="readisimpl"></a> 8.3 Simple Query Examples

```js
...
class User {
    constructor() {
        this.oid = 0;
        this.id = '';
        this.name = '';
    }
}
...
var query = session.newQuery(User).attr('name').eq('test');
query.orderBy('oid');
//query.orderBy('oid').desc() //descending order
//The above query form the following sql:
//SELECT T1.OID, T1.ID, T1.NAME FROM USERS T1 WHERE T1.NAME = ? ORDER BY T1.OID

//More comparision operators can be queried as

query.attr('name').like('%st')
//or query = session.newQueryFromArray(User, [['name', '~', '%st']])

query.attr('name').notLike('%st')
//or query = session.newQueryFromArray(User, [['name', '!~', '%st']])

query.attr('oid').gt(6) // or query.attr('oid').greaterThan(6) 
//or query = session.newQueryFromArray(User, [['oid', '>', 6]])

query.attr('oid').ge(6) // or query.attr('oid').greaterOrEquals(6) 
//or query = session.newQueryFromArray(User, [['oid', '>=', 6]])

query.attr('oid').lt(6) // or query.attr('oid').lessThan(6)
//or query = session.newQueryFromArray(User, [['oid', '<', 6]])

query.attr('oid').le(6) // or query.attr('oid').lessOrEquals(6) 
//or query = session.newQueryFromArray(User, [['oid', '<=', 6]])

query.attr('name').isNULL()
//or query = session.newQueryFromArray(User, [['oid', '=', null]])

query.attr('name').isNOTNULL()
//or query = session.newQueryFromArray(User, [['oid', '!=', null]])

query = session.newQuery(User);
query.attr('name').eq('test').and.attr('id').eq('tom');
//or query = session.newQueryFromArray(User, [['name', '=', 'test], '&', ['id', '=', 'tom']])

//The above query form the following sql:
//SELECT T1.OID, T1.ID, T1.NAME FROM USERS T1 WHERE T1.NAME = ? AND T1.ID = ?

query = session.newQuery(User);
query.attr('name').eq('test').or.attr('id').eq('tom');
//or query = session.newQueryFromArray(User, [['name', '=', 'test], '|', ['id', '=', 'tom']])

//The above query form the following sql:
//SELECT T1.OID, T1.ID, T1.NAME FROM USERS T1 WHERE T1.NAME = ? OR T1.ID = ?

query = session.newQuery(User);
query.enclose.attr('name').eq('tom').or.attr('name').eq('jerry');
query.and.enclose.attr('id').eq('x').or.attr('id').eq('y');
// query = session.newQueryFromArray(User, ['(', ['name', '=', 'tom'], '|', ['name', '=', 'jerry'], ')', '&', '(', ['id', '=', 'x'], '|', ['id', '=', 'y'], ')'])

//The above query form the following sql:
//SELECT T1.OID, T1.ID, T1.NAME FROM USERS T1 WHERE ( T1.NAME = ? OR T1.NAME = ? ) AND ( T1.ID = ? OR T1.ID = ? )
```

### <a name="readionetoone"></a> 8.4 One To One Query Examples

```js
...
class Location {
    constructor() {
        this.oid = 0;
        this.id = '';
        this.name = '';
    }
}
...
...
class Department {
    constructor() {
        this.oid = 0;
        this.id = '';
        this.name = '';
        this.location = null;
    }
}
...
...
class Employee {
    constructor() {
        this.oid = 0;
        this.id = '';
        this.name = '';
        this.department = null;
    }
}
...

var query = session.newQuery(Employee).attr('department.name').eq('test');

//The above query form the following sql:
//SELECT T1.OID, T1.ID, T1.NAME, T1.DEPTOID FROM EMP T1, DEPT T2 WHERE T1.DEPTOID = T2.OID AND T2.NAME = ?

query = apatite.newQuery(Employee).attr('department.location.oid').eq(2);

//The above query form the following sql:
//SELECT T1.OID, T1.ID, T1.NAME, T1.DEPTOID FROM EMP T1, DEPT T3, LOCATION T2 WHERE T1.DEPTOID = T3.OID AND T3.LOCATIONOID = T2.OID AND T3.LOCATIONOID = ?
```

### <a name="readifetch"></a> 8.5 Fetching Attribute Values

Sometimes you would want to just query the attribute values instead of querying for objects. This can be done by calling the method fetchAttr on the query instance.

Example
```js
...
// fetch employee name and department name
var query = session.newQuery(Employee)
query.fetchAttr('name')
query.fetchAttr('department.name')
query.execute(function(err, empList) {
    console.log(empList[0]) // output: {'name': 'Peter', 'department.name': 'Sales'}
})
...
...
// fetch multiple attributes using fetchAttrs method
var query = session.newQuery(Employee)
query.fetchAttrs(['name', 'department.name', 'department.location.name'])
query.execute(function(err, empList) {
    console.log(empList[0]) // output: {'name': 'Peter', 'department.name': 'Sales', 'department.location.name': 'First Floor'}
})
...
```

### <a name="readifunc"></a> 8.6 Fetching Function Values

To query a function, use one of the function methods specified in the example below. This method takes two arguments. The first, attribute name and the second an alias name.
```js
...
Example
var query = session.newQuery(Order)
query.fetchSumAs('total', 'totalSum')
query.fetchMaxAs('total', 'totalMax')
query.fetchMinAs('total', 'totalMin')
query.fetchAvgAs('total', 'totalAvg')
query.execute(function (err, result) {
    if (err)
        console.log(err)

    console.log(result) //example output: [ { totalSum: 2635.5, totalMax: 2183, totalMin: 0, totalAvg: 878.5 } ]
    endSession(session)

})
...
query.fetchDistinctAs('customer.name', 'distinctCustomerNames')
...
```

### <a name="readiexis"></a> 8.7 Exists and Not Exists Query

Exists and not exists can used in queries by specfying the join attribute name. For example, lets say we have two classes, Department and Employee. To fetch all departments which are assigned to at least one employee, you would write the following code:

```js
...
var deptQuery = session.newQuery(Department)
var empQuery = session.newQuery(Employee)
empQuery.attr('department.oid').eq(deptQuery.attrJoin('oid'))
deptQuery.exists(empQuery)
deptQuery.execute(function (err, departments) {
})
...
```

To fetch all departments which have employees with salary greater than 1000, you would do the following:

```js
...
var deptQuery = session.newQuery(Department)
var empQuery = session.newQuery(Employee)
empQuery.attr('department.oid').eq(deptQuery.attrJoin('oid'))
empQuery.and.attr('salary').gt(1000)
deptQuery.exists(empQuery)
deptQuery.execute(function (err, departments) {
})
...
```

To fetch all departments which have not been assigned to an employee, you would do:

```js
...
var deptQuery = session.newQuery(Department)
var empQuery = session.newQuery(Employee)
empQuery.attr('department.oid').eq(deptQuery.attrJoin('oid'))
deptQuery.notExists(empQuery)
deptQuery.execute(function (err, departments) {
})
...
```

The above queries would work even if you do not have a one to many relation from department to employee.

### <a name="readicurs"></a> 8.8 Cursor Stream

If you have large amounts of objects to process, you could use cursor stream to process one object at a time. Just set the returnCursorStream property of a query instance to true. The second parameter of callback function of the execute method would be an instance of ApatiteCursorStream which has 3 events: error, result, end. Example:

```js
apatite.newSession(function (err, session) {
    var query = session.newQuery(Department)
    query.returnCursorStream = true

    query.execute(function (err, cursorStream) {
        if (err) {
            console.log(err)
            return endSession(session)
        }

        cursorStream.on('error', function(cursorStreamErr) {
            console.log(cursorStreamErr)
            endSession(session)
        })
        cursorStream.on('result', function(department) {
            console.log(JSON.stringify(department))
        })
        cursorStream.on('end', function() {
            endSession(session)
        })
    })
})

function endSession(session) {
    session.end(function (endConnErr) {
        if (endConnErr)
            return console.log(endConnErr)
        console.log('Connection ended.')
    })
}
```

## <a name="creat"></a> 9. Creating and Deleting Objects

To save a new object, call the method registerNew on the session.

Example:

```js
...
var changesToDo = function (changesDone) {
    var department = new Department();
    department.name = 'Sales';
    session.registerNew(department);
    changesDone(); // must be called when you are done with all changes
}

session.doChangesAndSave(changesToDo, function (saveErr) {
    if (saveErr)
        console.error(saveErr.message);
});
...
```

To delete an object, call the method registerDelete on the session.

Example:

```js
...
var changesToDo = function (changesDone) {
    var query = session.newQuery(Department);
    query.attr('name').eq('Pre-Sales');
    query.execute(function(err, departments) {
        if (err) {
            changesDone(err)
            return;
        }
        session.registerDelete(departments[0]);
        changesDone(); // must be called when you are done with all changes
    });
}

session.doChangesAndSave(changesToDo, function (saveErr) {
    if (saveErr)
        console.error(saveErr.message);
});
...
```
 You don't need to register (new/delete) the one to many mapping objects, they would be registered automatically, just add/remove it to/from the one to many proxy.



## <a name="chang"></a> 10. Changing and Saving Objects

To do changes to your objects and save, you need to call doChangesAndSave method on the ApatiteSession instance. This method takes two arguments.

The first argument is a callback function in which you would do all your changes to objects. The callback function is passed with another callback function which must be called after you are done with your changes.

The second argument is a callback function which would be called after the changes are saved. **If the save fails, all your changes would be rolled back** and the error object would be passed in the callback function.

Example:

```js
...
// Change an existing department
var changesToDo = function (changesDone) {
    var query = session.newQuery(Department);
    query.attr('name').eq('Sales');
    query.execute(function(err, departments) {
        if (err) {
            changesDone(err)
            return;
        }
        departments[0].name = 'Pre-Sales';
        changesDone(); // must be called when you are done with all changes
    });
}

session.doChangesAndSave(changesToDo, function (saveErr) {
    if (saveErr)
        console.error(saveErr.message); // all the changes would be rolled back
});
...
```

**If a save is in progress, all subsequent calls to doChangesAndSave would be queued and would be processed once the existing save is finished.**


## <a name="sessi"></a> 11. Session Cache

The default cache size for each class is 0. All the objects loaded from the database are not cached in the session. You can change the cache size by setting the property defaultCacheSize of the Apatite instance. If the cache fill up to its size, the first object inserted is removed from the cache.

To remove all objects from cache call clearCache() on the session. If a query is executed with only primary key, object would be returned from cache if it exists.

## <a name="proxi"></a> 12. Proxies

One to one and one to many attributes are by default proxies. They are instances of ApatiteOneToOneProxy and ApatiteOneToManyProxy. To get the value of the proxy, call getValue() method on the proxy instance.

One to one proxy example:

```js
...
employee.department.getValue(function (err, department) {
  console.log(department.name);
});
...
```

```js
...
// Using promise
var promise = employee.department.getValue();
promise.then(function (department) {
  console.log(department.name);
}, function (err) {
  console.log(err.message);
});
...
```

One to many proxy example:

```js
...
department.employees.getValue(function (err, employees) {
  console.log(employees.length);
});
...
```

```js
...
// Using promise
var promise = department.employees.getValue();
promise.then(function (employees) {
  console.log(employees.length);
}, function (err) {
  console.log(err.message);
});
...
```

To add or remove objects from the one to many proxy:

```js
...
//add new employee
department.employees.add(newEmployee);

//remove existing employee
department.employees.remove(newEmployee);
...
```


## <a name="hookupm"></a> 13. Hookup Methods

Whenever apatite loads, saves, deletes or rollbacks an object, following methods would be called on the object if they are defined in the object's class:

```js
apatitePostLoad()
apatitePostSave()
apatitePostDelete()
apatitePostRollback()
```

Example:

```js
class Employee {
    constructor() {
        this.firstName = ''
        this.lastName = ''
    }

    apatitePostLoad() {
        //Called after apatite loads the object from the database 
    }

    apatitePostSave() {
        //Called after apatite sucessfully saves the object in the database
    }

    apatitePostDelete() {
        //Called after apatite successfully deletes the object from the database
    }

    apatitePostRollback() {
        //Called after apatite rolls back all the changes in case of errors during save process
    }
}
```

## <a name="conne"></a> 14. Connection Pooling

By default, connection pooling is not used. For every session you create, a new database connection is created. The connection is released when the method end() on ApatiteSession instance is called.

To use connection pooling, call the method useConnectionPool on an Apatite instance:

```js
...
apatite.useConnectionPool()
...
```

When connection pooling is used, every SQL sent to the database would use a new connection from the pool except during the save process. During the save process, the same connection is used to execute statements in transaction. To end the connection when using pool you must call the method closeConnectionPool()

```js
...
apatite.closeConnectionPool(function(connErr) {})
...
```


## <a name="sqlgen"></a> 15. SQLs Generation / Execution

SQL's can be generated from the model descriptors and additionally can be executed. To create SQL scripts, following methods are available on a session:

 ```js
...
//Returns string containing sql script
session.createSQLScriptForAllModels()
session.createSQLScriptForModel('Department')
session.createSQLScriptForAttribute('Department', 'name')

//Callback function for the method calls createDB..
function onCreated(err, result) {
    if (err)
        console.error(err)
}

//Creates sqls and executes them on the current session's connection
session.createDBTablesForAllModels(onCreated)
session.createDBTableForModel('Department', onCreated)
session.createDBColumnForAttribute('Department', 'name', onCreated)
...
```

## <a name="publicapi"></a> 16. Public API

Click [here](https://github.com/apatitejs/public-api/blob/master/public-api.md) for the public API.

## <a name="tests"></a> 17. Tests

To run the tests, install [mocha](https://github.com/mochajs/mocha), [chai](https://github.com/chaijs/chai) and then run:

```bash
$ npm test
```