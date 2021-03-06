# Basic example of pl/sql use of MLE
This code serves as a basic example of pl/sql with MLE.

## Check to see database version if free tier look for 21c
```
SELECT banner FROM V$VERSION;
```

##Basic Hello world.
- Copy and paste into SQL Worksheet, for a basic test to see if MLE is working on your database.
```
DECLARE
ctx DBMS_MLE.context_handle_t;

user_code clob := q'~
    console.log('Hello Chip!');
~';

BEGIN
    ctx := DBMS_MLE.create_context();
    DBMS_MLE.eval(ctx, 'JAVASCRIPT', user_code);
    DBMS_MLE.drop_context(ctx);
END;
```

## Create the Core table
- Create a sample table for the queries.

```
CREATE TABLE  "PLAYERS"
   (	"FIRSTNAME" VARCHAR2(250) COLLATE "USING_NLS_COMP",
	"LASTNAME" VARCHAR2(250) COLLATE "USING_NLS_COMP",
	"JERSEY_NUMBER" NUMBER,
	"DOB" DATE,
	"TEAM_ID" NUMBER,
	"PLAYER_ID" NUMBER,
	"NICKNAME" VARCHAR2(250) COLLATE "USING_NLS_COMP",
	"EMAIL" VARCHAR2(200) COLLATE "USING_NLS_COMP",
	"CARD" BLOB,
	"MIMETYPE" VARCHAR2(50) COLLATE "USING_NLS_COMP",
	"CREATED_DATE" DATE,
	"FILENAME" VARCHAR2(200) COLLATE "USING_NLS_COMP",
	"BATS" VARCHAR2(10) COLLATE "USING_NLS_COMP",
	"THROWS" VARCHAR2(10) COLLATE "USING_NLS_COMP",
	 CONSTRAINT "PLAYERS_PK" PRIMARY KEY ("PLAYER_ID")
  USING INDEX  ENABLE,
	 CONSTRAINT "JERSEY_N" CHECK (jersey_number IS NOT NULL) ENABLE
   )  DEFAULT COLLATION "USING_NLS_COMP"
/
```

- Create a sequence for the table PK

```
CREATE SEQUENCE   "PLAYERS_SEQ"  MINVALUE 1 MAXVALUE 9999999999999999999999999999 INCREMENT BY 1 START WITH 41 CACHE 20 NOORDER  NOCYCLE  NOKEEP  NOSCALE  GLOBAL
/
```

- Create a trigger to populate the primary key

```
CREATE OR REPLACE EDITIONABLE TRIGGER  "BI_PLAYERS"
  before insert on "PLAYERS"               
  for each row  
begin   
  if :NEW."PLAYER_ID" is null then
    select "PLAYERS_SEQ".nextval into :NEW."PLAYER_ID" from sys.dual;
  end if;
end;

/
ALTER TRIGGER  "BI_PLAYERS" ENABLE
/
```

- Insert a few records for your table.


##Query the core table
- Show the raw data as an example, note player id column generated by a sequence.

```
select player_id, firstname, lastname, jersey_number from players;
```

- add in the end result with function
```
select player_id, firstname, lastname, jersey_number, getCodeName(player_id) as "Code Name" from players;
```

## Create a function with the
- Now take the code above and turn it into a function we can call via SQL.
```
create or replace function getCodeName( playerId IN number )
return varchar2
is

ctx DBMS_MLE.context_handle_t;
jersey_num number;
fullName varchar2(100);

user_code clob := q'~
    //Include Javascript libraries
    let bindings = require('mle-js-bindings');
    const oracledb = require('mle-js-oracledb');

     try {
        //Set the javascript variable from pl/sql variable
        let jersey_num = bindings.importValue('num');
        let codeName;

        //Build the query notice the parameter syntax :jersey in the query.
        const sqlQuery = 'select FIRSTNAME, LASTNAME  from players where JERSEY_NUMBER = :jersey';

        //Create Connection and Execute the Query
        const conn = oracledb.defaultConnection();

        //Notice the syntax in the json formed and passed to query.
        const getNames = conn.execute(sqlQuery, {jersey: jersey_num});

        if (getNames.rows.length<1) {
            throw new Error("Player not found error");
        }

        //Build the Code Name
        for (let row of getNames.rows) {
        codeName = row[0].charAt(0) +''+row[1].charAt(0)+jersey_num;
        }

        //Export the value from javascript back to Pl/SQL
        bindings.exportValue('fullname', codeName);

    }
    catch (error) {
        bindings.exportValue('fullname', error.message);
    }
~';
BEGIN
--create the context
ctx := dbms_mle.create_context();

--Get the players jersey number in PL/SQL
select JERSEY_NUMBER into jersey_num from players where player_id = playerId;

--Pass the jersey number into javscript as a variable.
dbms_mle.export_to_mle(ctx, 'num', jersey_num);

--Execute the javascript code block
dbms_mle.eval(ctx, 'JAVASCRIPT', user_code);

--Import the name
dbms_mle.import_from_mle(ctx, 'fullname', fullName);

-- Drop the context for MLE
dbms_mle.drop_context(ctx);

return fullname;

EXCEPTION
WHEN OTHERS THEN
--drop context if error
dbms_mle.drop_context(ctx);
    RAISE;
END;
```

##Variable passing in MLE
- You can copy and paste into sql developer and run without the function to see and play with how it works.
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
     try {
        //Set the javascript variable
        let jersey_num = bindings.importValue('num');
        console.log("Imported jersey number is: " + jersey_num);
        let codeName;


        //Build the query notcie the parameter syntax to include variables.
        const sqlQuery = 'select FIRSTNAME, LASTNAME  from players where JERSEY_NUMBER = :jersey';
        //console.log("Output the query formed.   ---  " + sqlQuery);

        //Create Connection and Execute the Query
        const conn = oracledb.defaultConnection();
        const getNames = conn.execute(sqlQuery, {jersey: jersey_num});

        console.log("Stringify the results to test: "+JSON.stringify(getNames.rows));
        console.log("Output Row and set javascript variable");

        console.log("length of array: "+ getNames.rows.length);


        if (getNames.rows.length<1) {
            throw new Error("Player not found error");
        }

        for (let row of getNames.rows) {
        console.log('Full Name is: ' + row[0] +' '+row[1]);
        codeName = row[0].charAt(0) +''+row[1].charAt(0)+jersey_num;
        }

        //Export the value
        bindings.exportValue('fullname', codeName);
    }
    catch (error) {
        bindings.exportValue('fullname', error.message);
    }
~';

BEGIN
  ctx := dbms_mle.create_context();
  --Get the players jersey number in PL/SQL
  select JERSEY_NUMBER into jersey_num from players where player_id = 1;

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

## Drop function
- Drop the function in the example above.
```
drop function getCodeName;
```
