# Basic example of pl/sql use of MLE
This code serves as a basic example of pl/sql with MLE.

#Basic Hello world.
- Copy and paste into SQL Worksheet

```
DECLARE
ctx DBMS_MLE.context_handle_t;
user_code clob := q'~
    console.log('Hello World!');
~';
BEGIN
    ctx := DBMS_MLE.create_context();
    DBMS_MLE.eval(ctx, 'JAVASCRIPT', user_code);
    DBMS_MLE.drop_context(ctx);
END;
```
#Variable passing in MLE
```
DECLARE
ctx DBMS_MLE.context_handle_t;
jersey_num number;
fullName varchar2(100);

user_code clob := q'~
    console.log('Starting Javascript Code');
    //Include Javascript libraries
    let bindings = require('mle-js-bindings');
    const oracledb = require('mle-js-oracledb');

    //Set the javascript variable
     let jersey_num = bindings.importValue('num');
     console.log("Imported jersey number is: " + jersey_num);
     let fullname;


    //Build the query notcie the parameter syntax to include variables.
     const sqlQuery = 'select FIRSTNAME, LASTNAME from players where JERSEY_NUMBER = :jersey';
     console.log("Output the query formed.   ---  " + sqlQuery);

    //Create Connection and Execute the Query
     const conn = oracledb.defaultConnection();
     const getNames = conn.execute(sqlQuery, {jersey: jersey_num});
     console.log("Stringify the results to test: "+JSON.stringify(getNames.rows));
     console.log("Output Row and set javascript variable");
    for (let row of getNames.rows) {
       console.log('Name: ' + row[0] +' '+row[1]);
       fullname = row[0] +' '+row[1];
    }

    //Export the value
    bindings.exportValue('fullname', fullname);
~';

BEGIN
  ctx := dbms_mle.create_context();
  --Get the players jersey number in PL/SQL
  select JERSEY_NUMBER into jersey_num from players where firstname = 'Tank';

  --Pass the jersey number into javscript as a variable.
  dbms_mle.export_to_mle(ctx, 'num', jersey_num);

  --Execute the javascript
  dbms_mle.eval(ctx, 'JAVASCRIPT', user_code);

  --Import the name
  dbms_mle.import_from_mle(ctx, 'fullname', fullName);

  -- Drop the
  dbms_mle.drop_context(ctx);
  -- Output Full Name
  dbms_output.put_line('Output PL/SQL Variable');
  dbms_output.put_line(fullName);
EXCEPTION
  WHEN others THEN
    dbms_mle.drop_context(ctx);
    RAISE;
END;
```
