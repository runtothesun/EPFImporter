EPFImporter
===========

The EPFImporter is a command-line based tool, available to EPF partners, that allows for ingestion of the EPF-Relational files into a relational database. 


About security

For simplicity, the username and password used by EPFImporter to connect to the MySQL database are stored as plain text in the EPFConfigFull.json and EPFConfigIncremental.json files. Alternatively, they can be passed in as options on the command line, again in plain text.

The best security that can be provided under this implementation is to restrict access to the files themselves.  A more thorough security model would depend on the needs of the user and the platform that EPFImporter is deployed on and is outside the scope of this sample code and documentation.

Files

EPFImporter.py is executable from the command line and is the entry point for running EPFImporter. The other Python files (EPFIngester.py and EPFParser.py) must be present in the same directory but are not normally used directly.

EPFImporter also references certain configuration files (EPFConfig.json, EPFFlatConfig.json, EPFLogger.conf) that must also be in the same directory. These files can be edited by the user as needed. If they are removed or renamed, they are recreated with default values by EPFImporter the next time it is run.

Command-line help

Typing "./EPFImporter.py -h" at the command line (assuming you are in the same directory as EPFImporter.py) prints a usage and help description, including less-used options not described in this document.

Running EPFImporter

To run EPFImporter, you must first have downloaded and uncompressed one or more EPF feeds. Once you have done so, simply run EPFImporter.py with a space-separated list of the uncompressed feed directories as arguments:

./EPFImporter.py /Users/EPF/Fulls/itunes20100423 /Users/EPF/Fulls/pricing20100423

Assuming the specified directories exist, the above causes EPFImporter to perform an import of the itunes and pricing feeds from 4/23/2010. No prior setup of the database is necessary; EPFImporter creates and configures all tables as needed based on the metadata in the EPF files.

EPFImporter automatically determines if each file is a Full or an Incremental export and performs the corresponding import type. Note that for an Incremental import to succeed, tables corresponding to the imported files must already exist in the database (normally from a previous Full import).

Importing EPF Flat files

To import EPF Flat files, pass the -f flag. This causes EPFImporter to use values from EPFFlatConfig.json, corresponding to the different file format of the EPF Flat exports.

Restricting the files to be imported

You can restrict which files are imported by using the -w (whitelist) and -b (blacklist) flags. The string following the flag is treated as a regular expression and is appended to a list of such expressions.

./EFImporter.py -w 'song' -b 'translation' /Users/EPF/Fulls/itunes20100423

The above imports all files containing "song" anywhere in the filename, except for those containing "translation".

If a filename is matched by both the whitelist and blacklist, the blacklist always overrides, and the file is not imported.

To add more than one whitelist or blacklist string, pass multiple -w or -b options:

./EFImporter.py -w 'song' -w 'video' -b 'application' -b 'collection' /Users/EPF/Fulls/itunes20100423

Regular expressions can be very complex, containing many characters with special meanings. The special characters that are most commonly useful with EPF are the following:

^	Matches the beginning of a string
$	Matches the end of a string
.	Matches any single character
*	Matches any number, including zero, of the preceding character

Here are some examples of the above, including which files would be imported (or excluded, if in the blacklist) by EPFImporter:

'^song$'		only the exact file "song"
'^song'		any file beginning with "song" (for example, song, song_mix, song_imix)
'song$'		any file ending with "song" (for example, song, artist_song)
'^song.*mix$	'	any file beginning with "song" and ending with "mix" (for example, song_mix, song_imix)

If you want to restrict files for every import, you can do so by editing the whiteList and blackList lines in the appropriate EPFConfig.json or EPFFlatConfig.json file.

Resuming interrupted imports

The progress of the last import is stored in EPFSnapshot.json. If an import is interrupted for any reason (for example, by a power outage), EPFImporter can use this file to automatically resume the import where it left off. To resume an import, simply pass the -r flag. No other arguments are needed:

./EPFImporter.py -r

EPFImporter resumes the import from the beginning of whichever file was interrupted and continues through the rest of the unimported files, respecting any -w and -b flags that were passed during the original run.

Logging

The output of EPFImporter is logged both to the console and to a file. When EPFImporter is first run, it creates an EPFLog directory. Each run creates a separate log file. The latest log is called EPFLog.log; when a new run is begun, the previous log is appended with a date-time stamp.

Logging options can be configured by editing the file EPFLogger.conf.

How imports are performed

Full imports

Because Full feeds contain a complete snapshot of the EPF data, Full imports are designed to import the new data and replace the old data as efficiently as possible.

For each file in a Full import, EPFImporter creates a new, temporary table in the database and populates it with the data in the file. When the import completes successfully, it renames the previous table (if it exists), then renames the new table to match the old one. Finally, it drops the old table from the database.

This process means that any access being made to the table at the moment the "swap" takes place will fail.

Incremental imports

Incremental feeds contain data that has changed (or been added) since the last Full feed. Because Incremental imports must merge new data with existing data, they are more complex than Full imports.

Normally an Incremental import works by comparing the primary key of the row to be imported with the existing table. If no matching record exists, the new row is inserted; if a matching record is found, the new record replaces it.

Unfortunately, this process is inefficient for large files. Therefore, for Incremental files containing more than 500,000 records, EPFImporter uses a different method. This is similar to the Full import, but instead of replacing the old table, the old and new tables are merged into a third, temporary table via a complex union operation in the database. This table is then "swapped" with the existing table in the same manner as for a Full import.
