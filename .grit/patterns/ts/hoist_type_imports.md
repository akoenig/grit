---
title: Hoist Type Imports
tags: [typescript, ts]
---

Makes sure to hoist type imports to the top of the file.

## Pattern Definition

```grit
engine marzano(0.1)
language js

function extractTypeName($import) js {
    const [, name] = $import.text.split(" ");

    return name;
}

function extractBodyWithoutImports($code) js {
  const text = $code.text;
  let result = "";
  let inImport = false;
  let braceCount = 0;
  let i = 0;
  
  while (i < text.length) {
    // Check for import statement start
    if (!inImport && text.substring(i).trim().startsWith("import ")) {
      inImport = true;
      
      // Skip to the end of the current line or statement
      while (i < text.length) {
        // Track braces for multi-line imports
        if (text[i] === '{') braceCount++;
        if (text[i] === '}') braceCount--;
        
        // Move to next character
        i++;
        
        // Check for end of import statement
        if ((text[i-1] === ';' || text[i-1] === '\n') && braceCount === 0) {
          // If we have a semicolon or newline (and no open braces), we might be at the end
          // Look ahead to see if the next non-whitespace is not part of the import
          let j = i;
          while (j < text.length && /\s/.test(text[j])) j++;
          
          if (j >= text.length || !text.substring(j).trim().startsWith("from ")) {
            inImport = false;
            break;
          }
        }
      }
    } else if (!inImport) {
      // Not in an import statement, add to result
      result += text[i];
      i++;
    } else {
      // Still in import statement, skip
      i++;
    }
  }
  
  return result;
}

file($body) where {
	$body_without_body = "",
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
	$final_content += "

",
	$final_content += extractBodyWithoutImports($body),

	$body => $final_content
}
```

## Transformations

Converts ...

```ts
import { type Foo, Bar, type Zoo } from "pkg-a";
import { type X } from "pkg-b";
import { A } from "pkg-c";
```

... to ...

```ts
import type { Foo, Zoo } from "pkg-a";
import type { X } from "pkg-b";

import { Bar } from "pkg-a";
import { A } from "pkg-c";
```

## Test

```typescript
import { createFileRoute, Outlet, redirect } from "@tanstack/react-router";
import { createServerFn } from "@tanstack/react-start";
import { Effect } from "effect";
import type {
  Test
} from "test";
import { creatorId } from "~/middlewares/creator-id.js";
import { tenant } from "~/middlewares/tenant.js";
import { SidebarInset, SidebarProvider } from "~/components/ui/sidebar";
import { Sidebar } from "~/components/sidebar/mod.jsx";
import { type Supporter, SupporterId } from "~/domain/supporter.js";
import {
  run,
  type User, 
  Authz
} from "core/mod.js";

export class MyService extends Effect.Service<MyService>()("MyService", {
  effect: Effect.gen(function* () {
    return {
      hello: () => "world"
    } as const;
  })
}) {}
```

```typescript
import type { Test } from "test";
import type { Supporter } from "~/domain/supporter.js";
import type { User } from "core/mod.js";

import { createFileRoute, Outlet, redirect } from "@tanstack/react-router";
import { createServerFn } from "@tanstack/react-start";
import { Effect } from "effect";
import { creatorId } from "~/middlewares/creator-id.js";
import { tenant } from "~/middlewares/tenant.js";
import { SidebarInset, SidebarProvider } from "~/components/ui/sidebar";
import { Sidebar } from "~/components/sidebar/mod.jsx";
import { SupporterId } from "~/domain/supporter.js";
import { run, Authz } from "core/mod.js";

export class MyService extends Effect.Service<MyService>()("MyService", {
  effect: Effect.gen(function* () {
    return {
      hello: () => "world"
    } as const;
  })
}) {}
```
