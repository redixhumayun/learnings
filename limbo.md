## Notes For Limbo

Delete functionality doesn't work yet because it isn't implemented.
Take a look at [this issue](https://github.com/tursodatabase/limbo/issues/859) as a next step. Working on the in-mem version where the entire thing is mmap'ed sounds like a good next step.

```
sqlite> EXPLAIN DROP TABLE users;
addr  opcode         p1    p2    p3    p4             p5  comment
----  -------------  ----  ----  ----  -------------  --  -------------
0     Init           0     38    0                    0   Start at 38
1     Null           0     1     0                    0   r[1]=NULL
2     OpenWrite      0     1     0     5              0   root=1 iDb=0; sqlite_master
3     Rewind         0     11    0                    0
4       Column         0     2     2                    0   r[2]= cursor 0 column 2
5       Ne             3     10    2     BINARY-8       82  if r[2]!=r[3] goto 10
6       Column         0     0     2                    0   r[2]= cursor 0 column 0
7       Eq             4     10    2     BINARY-8       82  if r[2]==r[4] goto 10
8       Rowid          0     5     0                    0   r[5]=sqlite_master.rowid
9       Delete         0     0     0                    2
10    Next           0     4     0                    1
11    Destroy        2     2     0                    0
12    Null           0     6     7                    0   r[6..7]=NULL
13    OpenEphemeral  2     0     6                    0   nColumn=0
14    IfNot          2     22    1                    0
15    OpenRead       1     1     0     4              0   root=1 iDb=0; sqlite_master
16    Rewind         1     22    0                    0
17      Column         1     3     13                   0   r[13]= cursor 1 column 3
18      Ne             2     21    13    BINARY-8       84  if r[13]!=r[2] goto 21
19      Rowid          1     7     0                    0   r[7]= rowid of 1
20      Insert         2     6     7                    0   intkey=r[7] data=r[6]
21    Next           1     17    0                    1
22    OpenWrite      1     1     0     5              0   root=1 iDb=0; sqlite_master
23    Rewind         2     35    0                    0
24      Rowid          2     7     0                    0   r[7]= rowid of 2
25      NotExists      1     34    7                    0   intkey=r[7]
26      Column         1     0     8                    0   r[8]= cursor 1 column 0
27      Column         1     1     9                    0   r[9]= cursor 1 column 1
28      Column         1     2     10                   0   r[10]= cursor 1 column 2
29      Integer        2     11    0                    0   r[11]=2
30      Column         1     4     12                   0   r[12]= cursor 1 column 4
31      MakeRecord     8     5     15    BBBDB          0   r[15]=mkrec(r[8..12])
32      Delete         1     68    7                    0
33      Insert         1     15    7                    0   intkey=r[7] data=r[15]
34    Next           2     24    0                    0
35    DropTable      0     0     0     users          0
36    SetCookie      0     1     2                    0
37    Halt           0     0     0                    0
38    Transaction    0     1     1     0              1   usesStmtJournal=1
39    String8        0     3     0     users          0   r[3]='users'
40    String8        0     4     0     trigger        0   r[4]='trigger'
41    Goto           0     1     0                    0
```

```shell
limbo> SELECT * FROM sqlite_schema;
table|users|users|2|CREATE TABLE users (name text, age int)
```

1. Delete Schema Records (opcodes 1-10):
Creates registers r1,r2,r3,r4,r5
Jumps to instruction 38, starts a txn and sets r[3]='users' and r[4]='trigger'. Goes back and sets r[1]=Null
Open sqlite_master for writing (cursor 0)
Scans through sqlite_master looking for records where:
  * reads column 2 from cursor 0 (column 2 is the table name in the schema table) into r[2]
  * if the column just read is not equal to the table name being dropped, get out of step 1
  * read column 0 from cursor 0 (column 0  is the type in the schema table, whether table or trigger) into r[2]
  * if the column just read is equal to 'trigger', get out of step 1
  * read the row id of the current row
  * delete
This removes the table definition from the schema table

2. Call the Destroy instruction (opcode 11)
This instruction is responsible for removing the table data from the b-tree and physical representation of the b-tree.
From the SQLite docs
Delete an entire database table or index whose root page in the database file is given by P1.
The table being destroyed is in the main database file if P3==0. If P3==1 then the table to be destroyed is in the auxiliary database file that is used to store tables create using CREATE TEMPORARY TABLE.

3. Miscellaneous 
`13    OpenEphemeral  2     0     6                    0   nColumn=0`
Opens cursor 2(P1) to a table with 0(p2) columns. Points to a Btree table because p4=0
If P3 is positive, then reg[P3] is modified slightly so that it can be used as zero-length data for Insert. This is an optimization that avoids an extra Blob opcode to initialize that register.
`14    IfNot          2     22    1                    0`
Jumps to P2 if the value in register P1 is false. The value is considered false if it has a numeric value of zero. If the value in P1 is NULL then take the jump if and only if P3 is non-zero.
`15    OpenRead       1     1     0     4              0   root=1 iDb=0; sqlite_master`
Opens a read cursor(cursor 1) to the schema table

4. Read from schema and insert into transient table
  * Reads column 3(tbl_name) from cursor 1(sqlite_master read cursor) into r[13]
  * Check if r[13] != r[2]. Here r[2] is probably the value read from column 2 of sqlite_schema in the previous loop
  * If the equality check is equal, read the rowid of the current row into r[7]
  * Insert the rowid from the previous step into cursor 2 (for the ephemeral table) with empty data

5. Miscellaneous
`22    OpenWrite      1     1     0     5              0   root=1 iDb=0; sqlite_master`
Open a write cursor (cursor 1) to the sqlite_schema table

6. Insert Into Schema Table (but why?)
  * Store the current rowid pointed to by cursor 2(ephemeral table) in r[7] `24      Rowid          2     7     0                    0   r[7]= rowid of 2`
  * If the cursor p1 (sqlite_master) does not contain a record with rowid p3(r[7] which was allocated in the previous step), then jump to p2 `25      NotExists      1     34    7                    0   intkey=r[7]`
  * Allocate values to 5 columns with registers 8,9,10,11,12 and make a record with it
  * Deletes the entry pointed to by cursor 1 `32      Delete         1     68    7                    0`
  * Inserts the record with a key of `r[7]` and data of `r[15]` into cursor 1 (sqlite_schema write cursor) `33      Insert         1     15    7                    0   intkey=r[7] data=r[15]`

7. DropTable `35    DropTable      0     0     0     users          0`
Removes all the in-memory representations like foreign keys, indexes and any in-memory hash tables. This is called only after `Destroy` which removes any b-tree representation.

The first loop makes sense. The second loop is a little confusing because of
```shell
17      Column         1     3     13                   0   r[13]= cursor 1 column 3
18      Ne             2     21    13    BINARY-8       84  if r[13]!=r[2] goto 21
```
reading column 3 gives the root page number for the table but that is being compared to `r[2]` which is storing the table name. How will they ever match?


### Notes on 9th Feb

Need to issue a delete instruction for the indices associated with a table as well

For each index or table that needs to be destroyed as part of DROP TABLE, the following instructions are issued in a loop.
The only things that changes is the root page of the table that gets destroyed.

```shell
11    Destroy        3     2     0                    0
12    Null           0     6     7                    0   r[6..7]=NULL
13    OpenEphemeral  2     0     6                    0   nColumn=0
14    IfNot          2     22    1                    0
15    OpenRead       1     1     0     4              0   root=1 iDb=0; sqlite_master
16    Rewind         1     22    0                    0
17      Column         1     3     13                   0   r[13]= cursor 1 column 3
18      Ne             2     21    13    BINARY-8       84  if r[13]!=r[2] goto 21
19      Rowid          1     7     0                    0   r[7]= rowid of 1
20      Insert         2     6     7                    0   intkey=r[7] data=r[6]
21    Next           1     17    0                    1
22    OpenWrite      1     1     0     5              0   root=1 iDb=0; sqlite_master
23    Rewind         2     35    0                    0
24      Rowid          2     7     0                    0   r[7]= rowid of 2
25      NotExists      1     34    7                    0   intkey=r[7]
26      Column         1     0     8                    0   r[8]= cursor 1 column 0
27      Column         1     1     9                    0   r[9]= cursor 1 column 1
28      Column         1     2     10                   0   r[10]= cursor 1 column 2
29      Integer        3     11    0                    0   r[11]=3
30      Column         1     4     12                   0   r[12]= cursor 1 column 4
31      MakeRecord     8     5     15    BBBDB          0   r[15]=mkrec(r[8..12])
32      Delete         1     68    7                    0
33      Insert         1     15    7                    0   intkey=r[7] data=r[15]
34    Next           2     24    0                    0
```

To run a complete example, create a table with 

```shell
sqlite> CREATE TABLE users (
(x1...>     name TEXT PRIMARY KEY,
(x1...>     age INTEGER
(x1...> );
sqlite>
sqlite> EXPLAIN DROP TABLE users;
```