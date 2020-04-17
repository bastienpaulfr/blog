---
title: "Add a new line at the end of each text files tracked by git"
date: 2020-03-30T15:04:26+02:00
draft: false
---

# Add a new line at the end of each text files tracked by git

One of the most anoying thing I meet when I am coding software, is the « No new line at end of file » warning.

![manifest](/nl.png)

For this, I have written a little script based of my research on google. I want to rewrite all text files tracked by git and ensure that a new line is written at the end of file.

```sh
#!/bin/bash

# list all files tracked by git
for file in `git ls-files -c -m`;
do
  # Test that file does exist
  if [ -f $file ]; then
    # Test that file is not a binary one
    grep -Iq . "$file"
    if [ $? -eq 0 ]; then
      # Add a new line
      sed -i '$a\' $file
    fi
  fi
done
```

You can execute it from a git repository and commit the result.
