Upload a file to a directory on an RDS Oracle instance - useful for creating external tables:

```perl
#!/usr/bin/perl 
use DBI;
use warnings;
use strict;

# RDS instance info
my $numArgs = $#ARGV + 1;
my $RDS_PORT=1521;
my $RDS_HOST = "";
my $RDS_LOGIN = "";
my $RDS_SID = "";
my $dirname = "DATA_PUMP_DIR";
my $fname = "";

my $SQL_INTEGER =  4;
my $SQL_VARCHAR = 12;
my $SQL_LONGRAW = 24;


if ($numArgs < 8 || $numArgs > 12){
	print "Usage: $0  [-e endpoint]  [-f dumpfile_name] [-s db_sid] [-l username/password]\n";
	print "\nOptional parameters: [-d datapump_dir] default is DATA_PUMP_DIR\n";
	print "\t\t     [-p port] default is 1521\n";
	die "Invalid number of parameters\n";
	
}
 
while ($a = shift(@ARGV),$b = shift(@ARGV)) {
    if ($a eq "-s")		{$RDS_SID = "$b";}
	elsif ($a eq "-l")	{$RDS_LOGIN = "$b";}
	elsif ($a eq "-e")	{$RDS_HOST = "$b";}
	elsif ($a eq "-p")	{$RDS_PORT = "$b";}
	elsif ($a eq "-d")	{$dirname = "$b";}
	elsif ($a eq "-f")	{$fname = "$b";}
	else {
		print 	"$a unknown parameter\n\n";
		print 	"Allowed parameters: \n";
		print 	"-s db_sid\n";
		print 	"-p port\n";
		print 	"-e endpoint\n";
		print 	"-l username/password\n";
		print	"-f dumpfile_name\n";
		die 	"-d datapump_dir\n";
		}
}

my $data = "dummy";
my $chunk = 8192;

my $sql_open = "BEGIN perl_global.fh := utl_file.fopen(:dirname, :fname, 'wb', :chunk); END;";
my $sql_write = "BEGIN utl_file.put_raw(perl_global.fh,:data, true); END;";
my $sql_close = "BEGIN utl_file.fclose(perl_global.fh);END;";
my $sql_global = "create or replace package perl_global as fh utl_file.file_type; end;";

my $conn = DBI->connect('dbi:Oracle:host='.$RDS_HOST.';sid='.$RDS_SID.';port='.$RDS_PORT,$RDS_LOGIN, '') || die ( $DBI::errstr ."\ ") ;

my $updated = $conn->do($sql_global);
my $stmt = $conn->prepare ($sql_open);
$stmt->bind_param_inout(":dirname", \$dirname, $SQL_VARCHAR);
$stmt->bind_param_inout(":fname", \$fname, $SQL_VARCHAR);
$stmt->bind_param_inout(":chunk", \$chunk, $SQL_INTEGER);
$stmt->execute() || die ( $DBI::errstr . "\ ");

open (INF, $fname) || die "\ Can't open $fname for reading: $!\ ";
binmode(INF);
$stmt = $conn->prepare ($sql_write);
my %attrib = ('ora_type',$SQL_LONGRAW);
my $val=1;
while ($val > 0) {
 $val = read (INF, $data, $chunk);
 $stmt->bind_param(":data", $data , \%attrib);
 $stmt->execute() || die ( $DBI::errstr . "\ "); 
};

print "\n$fname successfully uploaded to $RDS_HOST...\n";

die "Problem copying: $!\ " if $!;
close INF || die "Can't close $fname: $!\n";
$stmt = $conn->prepare ($sql_close);
$stmt->execute() || die ( $DBI::errstr . "\n");
```

Copyright 2017 Amazon.com, Inc. or its affiliates. All Rights Reserved.

Licensed under the Apache License, Version 2.0 (the "License").
You may not use this file except in compliance with the License.
A copy of the License is located at <http://aws.amazon.com/apache2.0/>