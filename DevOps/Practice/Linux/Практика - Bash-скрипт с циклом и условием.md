cat analyze.sh && echo "---" && ./analyze.sh /etc

#!/bin/bash
if [ -z "$1" ]; then
    echo "Usage: $0 <directory>"
    exit 1
elif [ ! -d "$1" ]; then
    echo "Error: $1 is not a directory"
    exit 1
fi

files=0
dirs=0
objects=$(find /etc)
for i in $objects; do
 if [ -f $i ]; then
  ((files++))
 else
  ((dirs++))
fi
done
echo "Files: $files"
echo "Directories: $dirs"
---
Files: 3200
Directories: 451
