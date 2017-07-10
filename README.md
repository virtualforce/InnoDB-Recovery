# MySQL (InnoDB) Database Recovery

Some quick facts about this writeup to save your time (in case you have a different scenario!):

 * From MySql 5.5.5 to MySql 5.6
 * Only .frm and .idb files
 * No .ibdata files available (or backed up)
 * No .cfg files (not generated for our version)
 * Target Environment Ubuntu 14.04 __but__ Recovery on windows 10 machine using XAMPP 
 * We did not run into ___InnoDB: Error: tablespace id ...___ mismatching issues at all we did not perform those steps, though Chris has create an interesting script for it [here](http://www.chriscalender.com/tag/got-error-1-from-storage-engine/)

## Background
A couple of weeks ago we ran into database issues when trying to setup for cross network access. While  I know that this wasnt something that would have caused complete database failure, but unfortunately by the time the issues were brought to me, they were already quite far into the ordeal, so I just had to take what was at hand and start from there. Apparently the DBMS had been reinstalled already and all what I had was a backup of the data folder from MySql.

Of course we started off with the most obvious choice of googling to find a way out. There were quite a few tools that would claim to be able to recover the database, however none of them were free, so we decided to take sometime and see if there is another ___"better option"___ ... (infact atleast one showed the data but without any option to copy or export it without a license)


## Outline of process
Here is the summary of steps we needed to perform in order to recover the datbase...

 * Install __MySql 5.6__
 * Install __MySql Utilities__
 * Enable __innodb_file_per_table__
 * Set proper __innodb_file_format__
 * __Create Database__
 * Recover __Table Structures__
 * Modify to __force the ROW_FORMAT__
 * __Create Table__
 * __Drop TableSpace__
 * __Replace idb file__
 * __Import TableSpace__
 * Take __Backup__ _OR_ __Export Table & Data__
 * __Restore__ _OR_ __Import into Production__


## Lights, Camera, Action!
### Install __MySql 5.6__
Download and install mysql 5.6 version from [here](https://dev.mysql.com/downloads/windows/installer/5.6.html).

### Install __MySql Utilities__
Download and install MySql Utilities for your platform from [here](https://dev.mysql.com/downloads/utilities/)

### Enable __innodb_file_per_table__
Open my.ini file and search for _innodb__ to locate the section where innodb related settings are configured. Uncomment/Add/Adjust the following keys
```
innodb_file_per_table = 1
```
and restart mysql (or combine this step with the next one).

### Set proper __innodb_file_format__
In your my.ini 
```innodb_file_format=Barracuda```
and restart mysql (see references to know more about file_formats)

### __Create Database__
Connect to your mysql server and 
* create your database
* choose your newly created database
* create table structures as described in the sections below

### Recover __Table Structures__

__On your recovery machine__ copy over the data folder (.frm and .ibd files) from your production machine (lets call is __source__ folder)

Now goto to your ___source___ data folder and execute the following command after replacing/adjusting the following:
* UserID: replace with a valid user for your mysql DBMS (default is root). This should be the user for your recovery machine's DBMS
* Password: default is empty
* Server: default is localhost
* the_port_to_be_used: anything different from mysql port which is default 3306; I used 3307

```javascript
mysqlfrm --server=<UserID>:<password>@<Server> --port=<the_port_to_be_used> <Name_of_your_table>.frm > <Name_of_your_table>.txt --diagnostic
```

e.g.
* ``` mysqlfrm --server=root:@localhost --port=3307 users.frm > users.txt --diagnostic ``` will dump the table structure (DDL) in the file named users.txt in the same folder, which i can open and copy the script to avoid keying in everything manually. 


### Modify to __force the ROW_FORMAT__
Update the generated DDL scripts to add ROW_FORMAT=Dynamic (or whatever was your original database set for; you can discover that if you proceed without this step, the Restore/Import step will throw a mismatch error in that case, otherwise you can skip this step)
Append ROW_FORMAT at the end of DDL definition i.e. it ends with ```ENGINE=InnoDB DEFAULT CHARSET=utf8 ROW_FORMAT=Dynamic```

Or you can create the table and then use ENGINE=InnoDB DEFAULT CHARSET=utf8 ROW_FORMAT=Dynamic
```ALTER TABLE <TableName> ROW_FORMAT=Dynamic;```

### __CreateDatabase__
Now connect to your database on the target machine and create an empty database, preferrably with the same name as the one you are going to recover. This will be handy when you generate scripts for the generated database or take backup to restore it on the server.

### __Create Table__
Create the table using the modified script above using ```Engine=InnoDB```. This will create the .frm and .idb files in the data folder on the target machine.

### __Drop TableSpace__
At this point you have an empty table in the database which has same structure that you need to recover data in. Use the following command to delete the existing tablespace, this will also delete the .idb file.

```ALTER TABLE <TableName> DISCARD TABLESPACE;```

### __Replace idb file__
Now manually copy over the .idb file for your table from the source folder to the data folder of the target database. At this point you will have empty table in your database with its tablespace deleted, hence any operations on the table will report that error. The idb file is present in the folder but the DBMS doesnt know about it.

### __Import TableSpace__
Run ```ALTER TABLE <TableName> IMPORT TABLESPACE;``` command to try and link the pasted .idb file with the table in the DBMS. If any errors are reported at this point, we may need to redo some of the previous steps to fix that error and retry this. 

While there might be optimial ways of fixing the errors, if any, and resuming from here, we have been removing the table using ```DROP TABLE <TableName>``` and recreating it with updated definition to keep things simple and clean for us -- it is upto you!

Luckily, in our case, we hit ROW_FORMAT related error only. We found that even with ALTER TABLE command the definition would change for ```SHOW CREATE TABLE <TableName>``` but not in the INFORMATION SCHEMA until we changed the defaults for the DBMS in the ini file. You may run into different situations though depending upon your scenario! Do expect to need googling for your specific problem.

You can use ```SELECT COUNT(*) FROM <TableName>;``` to verify that the data has been imported and how many rows did you recover.

Now you can take __Backup__ _OR_ __Export Table & Data__ and __Restore__ _OR_ __Import into Production__


### References:

 * [MySQL Utilities](https://dev.mysql.com/downloads/utilities/) to recover the DDL from .frm file
 * About row formats [Copying MySQL Tablespaces from 5.6 to 5.7](https://medium.com/@alexquick/transporting-mysql-tablespaces-from-5-6-to-5-7-517c01345fbb)
 * 5.6 to 5.7 data recovery [Case Study](https://www.percona.com/blog/2015/12/01/how-to-transport-tablespace-from-mysql-5-6-to-mysql-5-7/)
 * Understand [InnoDB Info Schema](https://dev.mysql.com/doc/refman/5.7/en/innodb-information-schema-system-tables.html)
 * Understand [InnoDB File Formats](https://dev.mysql.com/doc/refman/5.6/en/innodb-file-format.html)
 * A handy script to [Interpret ROW_FORMAT Flags](https://bugs.mysql.com/bug.php?id=80057)
 * InnoDB [Default Row Formats](https://dba.stackexchange.com/questions/14246/innodb-file-format-barracuda)
 * Enable file format [dynamically](https://dev.mysql.com/doc/refman/5.6/en/innodb-file-format-enabling.html)
* [YouTube video](https://www.youtube.com/watch?v=HFxdibq8MHw) shows step by step procedure _(beware that step 7 listed for recovery section in the video has an ERROR which the demonstrator fixes only at the time of executing that step. It should be __IMPORT TABLESPACE__ instead of __DISCARD TABLESPACE__ in this step)_


