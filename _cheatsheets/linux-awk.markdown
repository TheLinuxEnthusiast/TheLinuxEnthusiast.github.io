---
title: "Helpful Awk commands"
layout: post
name: "linux-awk"
date: 2022-08-25 00:00:00 +0100
tags: ["Linux", "awk"]
author: "Darren Foley"
show_sidebar: false
menubar: cheatsheet_menu
---


<p>GNU awk, refered to as <b>gawk</b> for short, is a text processing tool for tabular delimited data. It is aliased as <b>awk</b> in most linux distributions so I will reference as awk throughout the code samples.</p>

[Official GNU Awk Documentation](https://www.gnu.org/software/gawk/manual/gawk.html)

<br>

<h4>Calling awk from the command line</h4>

<p>This is the most common use case, where you want to filter some output by piping into awk. The below code grabs the process id's for java processes running in the user space.</p>

```

$ ps -ef | grep "java" | awk '{ print $2 }'

# If you want to kill them for example run

$ ps -ef | grep "java" | awk '{ print $2 }' | xargs kill

```

<br>

<h4>Calling awk from a script</h4>

<p>awk is primarily a command line tool but can be called from a text file by adding your awk code to a text file and adding a shebang " <b>#!/usr/bin/awk -f</b> " pointing to your awk executable binary. If you haven't added a shebang you must call the text file using "awk -f filename.awk". The .awk extension is by convention but not manditory.
</p>

<p>Simply run the script like so (Assuming file has executable permissions)</p>

```
$ ./test.awk data.txt

or

$ awk -f test.awk data.txt

# if you don't have any input data use a here string

$ ./test.awk <<< cat ""

```

<br>

<h4>Special variables</h4>

```
$0 Current line
$1 First field
$2 Second field
$n nth field
$NF Number of Fields
$NR Number of records read so far
```

<br>

<h4>Split input by delimiter</h4>

<p>awk is line oriented and expects data to be delimited in some way. awk defaults to whitespace as default delimiter. You can specify the delimiter with "-F" flag.</p>

```

# Getting the second field of a text file with colon delimiter

$ awk -F':' '{ print $2 }' data.txt

```

<br>

<h4>Structure of awk programs</h4>

<p>awk has three code blocks; a BEGIN block, a main processing block, & an END block.</p> 

```
#!/usr/bin/awk -f

BEGIN{

# Initialise variables here

} 
{

# Do processing here

}
END{

# Print results, do something after
}
```

<p>Both BEGIN/END blocks are executed only once, once before processing in the case of BEGIN and once after in the case of END. awk is a line oriented tool and will loop through each row within the main block {}.</p>

<br>

<h4>Filter Rows by a pattern</h4>

```

# Prints second & third columns of all lines with "pattern"

$ awk ' /pattern/ { print $2,$3 } ' data.txt

```

<br>

<h4>User defined Functions</h4>

<p>awk has a large catalog of built in functions for manipulating strings, numbers and arrays but users can also define UDF's for custom logic like so.</p>

```
#!/usr/bin/awk -f

function my_func(x, y){
  return x*y
}

{
  # Main processing here
  my_func(3,4)
  
}

```

<br>

<h4>Conditional checking</h4>

```

If the first field is equal to the string "text" print entire line otherwise print "not found"

$ awk ' { if($1=="text"){ print; } else { print "not found"; }; } ' data.txt

```

<br>

<h4>Looping</h4>


```
#!/usr/bin/awk -f
# awk has "c-like" looping syntax

{
  var=4;
  for (i = 1; i <= var; i++) {
      print "do something"
      }

}

```