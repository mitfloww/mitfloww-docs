## Implementation to copy worker code to a file

```js
const fs = require("fs"); 
const path = require("path"); 

const SRC_DIR = path.resolve(process.cwd(), "src"); 
const OUTPUT_FILE = path.resolve(process.cwd(), "combined.txt"); 

// Ignore exact directory/file names (not substring match) 
const IGNORE = new Set(["node_modules", ".git", ".next", "dist"]); 

/** 
 * Check if a path should be ignored based on path segments 
 */ 
function shouldIgnore(fullPath) { 
 const segments = fullPath.split(path.sep); 
 return segments.some(segment => IGNORE.has(segment)); 
} 

/** 
 * Recursively collect all files (relative paths) 
 */ 
function getAllFiles(dir, baseDir) { 
 let results = []; 

 const entries = fs.readdirSync(dir).sort(); 

 for (const entry of entries) { 
 const fullPath = path.join(dir, entry); 

 if (shouldIgnore(fullPath)) continue; 

 const stat = fs.statSync(fullPath); 

 if (stat.isDirectory()) { 
 results = results.concat(getAllFiles(fullPath, baseDir)); 
 } else { 
 results.push(path.relative(baseDir, fullPath)); 
 } 
 } 

 return results; 
} 

/** 
 * Build folder tree structure (like `tree` command) 
 */ 
function buildTree(dir, prefix = "") { 
 const entries = fs.readdirSync(dir) 
 .filter(entry => !shouldIgnore(path.join(dir, entry))) 
 .sort(); 

 let tree = ""; 

 entries.forEach((entry, index) => { 
 const fullPath = path.join(dir, entry); 
 const isLast = index === entries.length - 1; 

 const connector = isLast ? "└── " : "├── "; 
 tree += `${prefix}${connector}${entry}\n`; 

 const stat = fs.statSync(fullPath); 

 if (stat.isDirectory()) { 
 const newPrefix = prefix + (isLast ? " " : "│ "); 
 tree += buildTree(fullPath, newPrefix); 
 } 
 }); 

 return tree; 
} 

/** 
 * Main execution 
 */ 
function combineFiles() { 
 const files = getAllFiles(SRC_DIR, SRC_DIR); 

 let output = ""; 

 // SECTION 1: Folder Structure 
 output += "PROJECT STRUCTURE:\n"; 
 output += "src\n"; 
 output += buildTree(SRC_DIR); 
 output += "\n=============================\n\n"; 

 // SECTION 2: File Contents 
 for (const relativeFilePath of files) { 
 const absolutePath = path.join(SRC_DIR, relativeFilePath); 

 const content = fs.readFileSync(absolutePath, "utf-8"); 

 output += `src/${relativeFilePath}\n`; 
 output += content.trim() + "\n"; 
 output += "------\n"; 
 } 

 fs.writeFileSync(OUTPUT_FILE, output, "utf-8"); 

 console.log(`Combined ${files.length} files into ${OUTPUT_FILE}`); 
} 

combineFiles();
```

---

## Run

```bash
node scripts/combine-src.js
```
