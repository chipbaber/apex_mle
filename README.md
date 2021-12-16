# Intro to Javascript MLE in APEX.
This code documents the basics of javascript MLE in APEX.

## Basic Queries

- Example query no parameter
```
console.log('Basic output query');
for ( var row of apex.conn.execute( "select FIRSTNAME, LASTNAME, JERSEY_NUMBER from players").rows) {
    console.log('Name: ' + row.FIRSTNAME+' '+row.LASTNAME+'   Number: '+ row.JERSEY_NUMBER);
}
```

- Example query with parameter of jersey Number

  ```
  console.log('Basic output query with parameter');
  for ( var row of apex.conn.execute( "select FIRSTNAME, LASTNAME, JERSEY_NUMBER from players where JERSEY_NUMBER = :jersey",{jersey: 9}).rows) {
      console.log('Name: ' + row.FIRSTNAME+' '+row.LASTNAME+'   Number: '+ row.JERSEY_NUMBER);
  }
  ```
