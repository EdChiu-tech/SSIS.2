# SSIS.2
The purpose of this package is to perform ETL methods from a flat CSV file and load into the final production table.


STEP ONE - Extract and convert data type
Using a Data Flow Task, the CSV file is loaded into the SSIS solution.  Adding a RowCount task in the beginning and saving it into a variable so that the original row count can be referenced later.  

The task Replace "null" and " " as NULL trims any spacing on the left and right side of the a string.  Blanks are replaced with the word NULL to ensure that blank data can be handled properly later on.

Data Conversion task is used to not only match the data type of the imported data matches the staging table in the database, but also help filter out initial inconsistancies such as negative age, a string age, or incorrect date format.  The errored rows are rejected and logged into the Rejected Row Logging table for future reference where as the error code is logged into the Error Log table.

The converted data is now staged onto the stg.Users table for further transformation and validation.

STEP TWO - UserID Validation
UserID Validation checks for any NULL or duplicate UserIDs.  UserIDs should be unique and should not be NULL.

The first SQL task in the UserID Validation looks for any UserID that is NULL, counts the number of rows, and stores them into a variable User::RowCountRemoveInvalidUserIDFail.  The second task logs all the rejected rows into the RejectedRows table for future reference.  The last step is to remove the invalid rows from the stg.Users table.

STEP THREE - Duplicate UserID Validation
This next step checks for any duplicate UserIDs.  Similar to the previous validation task, a row count and reject rows are logged into the RejectedRows table.  Here, we partition the duplicate UserIDs, then compare the RegistrationDate with the LastLoginDate where the RegistrationDate cannot be NULL if LastLoginDate exists, or the LastLoginDate cannot be greater than the RegistrationDate.  We want to ignore the situation where RegistrationDate is valid but LastLoginDate is null as Users can create an account but never log in for example.

STEP FOUR - Email Validation
The main condition for an email to be valid is having a "@".  As such, wildcards are used to check if an "@" sign is in an email.  According to references and research online, most if not all service providers use Unicode encoding so special characters are acceptable.  There are domain specific restrictions such as double special characters in a row, or special characters as the first or last character before the domain that may render the email invalid.  So unless we know which domain restricts the usage of special characters, we cannot adjust for all invalid special character scenarios or invalidate valid emails.  However, if we need to account for special characters, then wildcard queries or regex queries can be used to target the invalid characters

STEP FIVE - Date Validation
Thanks to the Data Conversion Task in step one, invalid date formats have been rejected already.  Similar to the queries in STEP THREE, we compare the RegistrationDate and LastLoginDate.  There are three scenarios to invalidate, the first one is where the RegistrationDate or LastLoginDate is ahead of the current date.  The second scenario is when the RegistrationDate is ahead of the LastLoginDate.  The last one is when RegistrationDate doesn't exist but LastLoginDate does.  

STEP SIX - Merge Clean data to Production
After cleaning the loaded data, the last step of ETL is to load them to the production table.  Using another Data Flow task, the cleaned stg.Users table is loaded into SSIS from the database and a Lookup Task is used to compare the stg.Users and prod.Users tables.  Using UserID as the key, UserIDs that do not exist in the production table will be loaded to the production table from the staging table.  If the UserIDs exists in both tables, we compare the LastLoginDates.  If the LastLoginDate in the staging table is greater than the LastLoginDate in the production table or the LastLoginDate doesn't exist int he production table, then we update the production table with the row from the staging table otherwise, we keep the existing row in the production table.

STEP SEVEN - Final Row Count Logging
The rows in the RejectedRows table and the Production table are now counted to give the final row count of the respective categories and logged into the FinalRowCountLog table.

STEP EIGHT - Truncate Staging Table
After the final ETL procedure is complete, the staging table is truncated to relief storage space from the database.

FINAL ROW COUNT
Rejected Rows: 9
New Rows Added to Production: 23
Updated Rows: 0
Total Rows in Production: 33
