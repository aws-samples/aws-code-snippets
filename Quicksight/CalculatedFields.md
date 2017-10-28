Some calculated fields that I found useful for splitting URIs into substrings:

```sql
firstSlash = locate({cs-uri-stem},'/', 8)

secondSlash = locate({cs-uri-stem},'/', {firstSlash} + 1)

thirdSlash = locate({cs-uri-stem},'/', {secondSlash} + 1)

fourthSlash = locate({cs-uri-stem},'/', {thirdSlash} + 1)

CNAME = substring({cs-uri-stem},8,{firstSlash} - 8)

CNAME+firstDir = substring({cs-uri-stem},8,{secondSlash}-7)

CNAME+secondDir = substring({cs-uri-stem},8,{thirdSlash}-7)
```

Copyright 2017 Amazon.com, Inc. or its affiliates. All Rights Reserved.

Licensed under the Apache License, Version 2.0 (the "License").
You may not use this file except in compliance with the License.
A copy of the License is located at <http://aws.amazon.com/apache2.0/>