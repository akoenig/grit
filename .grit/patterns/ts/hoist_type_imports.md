---
title: Hoist Type Imports
tags: [typescript, ts]
---

Makes sure to hoist type imports to the top of the file.

```grit
engine marzano(0.1)
language js


function extractTypeName($import) js {
    const [, name] = $import.text.split(" ");

    return name
}

pattern hoist_type_imports() {
    file($body) where {
        $type_imports = [],
        $regular_imports = [],
        
        // First pass: collect and normalize all imports
        $body <: contains bubble($type_imports, $regular_imports) or {
            // Case 1: import type { ... }
            `import type $type_clause from $source` as $import where {
                $type_imports += `import type $type_clause from $source;`
            },
            
            // Case 2: import { type ... }
            `import { $imports } from $source` as $import where {
                $type_names = [],
                $regular_names = [],
                $imports <: some bubble($type_names, $regular_names) {
                    `$name` where { 
                        if ($name <: r"type .*") {
                            if ($type_names <: not []) { $type_names += "," },
                            $type_names += extractTypeName($name)
                        } else {
                            if ($regular_names <: not []) { $regular_names += "," },
                            $regular_names += $name 
                        }
                    }
                },
                
                if ($type_names <: not []) {
                    $type_imports += `import type { $type_names } from $source;`
                },

                if ($regular_names <: not []) {
                    $regular_imports += `import { $regular_names } from $source;`
                }
            },

            // Case 3: regular imports
            `import $clause from $source` as $import where {
                if ($clause <: not contains "type") {
                    $regular_imports += `import $clause from $source;`
                }
            }
        },

        // Second pass: build the final string
        $final_content = "",
        if ($type_imports <: not []) {
            $type_imports <: some bubble($final_content) $import where {
                if ($final_content <: not "") {
                    $final_content += "
"
                },
                $final_content += $import
            }
        },
        if ($regular_imports <: not []) {
            if ($final_content <: not "") {
                $final_content += "
"
            },
            $regular_imports <: some bubble($final_content) $import where {
                if ($final_content <: not "") {
                    $final_content += "
"
                },
                $final_content += $import
            }
        },
        $body => $final_content
    }
}

hoist_type_imports()
```

## Transformations

```ts
import { type Foo, Bar, type Zoo } from "pkg-a";
import { type X } from "pkg-b";
import { A } from "pkg-c";
```

```ts
import type { Foo, Zoo } from "pkg-a";
import type { X } from "pkg-b";

import { Bar } from "pkg-a";
import { A } from "pkg-c";
```

