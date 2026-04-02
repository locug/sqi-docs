# SQI Scripting -- AI Coding Context

Drop this file into any folder where you write `.sqi` scripts. It gives AI coding
assistants (Claude, Copilot, Cursor, etc.) the context they need to produce correct SQI.

## What is SQI?

SQI (Script Query Internal) is a text-replacement macro language used by the LOC SMS
point-of-sale suite. Scripts are `.sqi` files containing SQL intermixed with
`@COMMAND(args)` patterns. The engine resolves every `@COMMAND(...)` by replacing it
inline with its return value, repeating until only pure SQL remains.

## How the Engine Works

1. **Pre-cleanup** -- Strip comment patterns and normalize quotes.
2. **Preprocessor** -- Replace bare variables (`@STORE`, `@TER`, `@USER`, etc.) with
   their runtime values. These have no parentheses. Unknown `@` tokens are escaped.
3. **Command dispatch** -- Find the innermost `@COMMAND(args)` (rightmost `)`, then
   back to `(`, then back to `@`). Call its handler, splice the result string in place.
4. **Loop** -- Repeat step 3 until no more commands remain.
5. **Execute** -- The resulting pure SQL is sent to the database.

## Critical Rules (You WILL Get These Wrong)

### 1. The Branch Delimiter

Inside `@FMT(CMP,...)` branch strings, you CANNOT use `@` directly -- the engine
would try to resolve those commands before CMP picks a branch. Use the registered
delimiter character `®` (U+00AE) instead. After CMP selects a branch, every `®` is
replaced with `@`, and the revealed commands run on subsequent loop iterations.

WRONG -- commands inside branches use `@`:
```
@FMT(CMP,@WIZGET(MODE)=1,'@WIZSET(RESULT=yes)','@WIZSET(RESULT=no)');
```

CORRECT -- commands inside branches use `®`:
```
@FMT(CMP,@WIZGET(MODE)=1,'®WIZSET(RESULT=yes)','®WIZSET(RESULT=no)');
```

Multiple commands chain naturally:
```
@FMT(CMP,@WIZGET(MODE)=1,'®WIZSET(RESULT=yes)®EXEC(FCT=720)','®EXEC(MSG=!Error)');
```

### 2. WIZRPL Appends, Not Replaces

Despite its name, `@WIZRPL(KEY=value)` **concatenates** value onto the existing value.
Use `@WIZSET(KEY=value)` to overwrite. This is the single most common SQI bug.

| Command | Key exists | Key missing |
|---------|-----------|-------------|
| `@WIZINIT(K=V)` | No-op (keeps old) | Creates K=V |
| `@WIZSET(K=V)` | **Overwrites** to V | Creates K=V |
| `@WIZRPL(K=V)` | **Appends** V to old | Creates K=V |

### 3. Innermost-First Evaluation

Nested macros resolve from the inside out. The engine finds the rightmost `)` first.

```
@FMT(LEN,@WIZGET(NAME))
```
Step 1: `@WIZGET(NAME)` resolves to `"Smith"`.
Step 2: `@FMT(LEN,Smith)` resolves to `"5"`.

### 4. Preprocessor Variables Are Not Commands

`@STORE`, `@TER`, `@USER`, and friends have **no parentheses**. They are replaced
before any command processing. Writing `@STORE()` is wrong and will not resolve.

### 5. Unknown Commands Are Escaped, Not Errors

If the engine does not recognize a command name, it escapes the parentheses and moves
on silently. No error is raised. Your script will produce garbled output instead of
failing, making bugs hard to spot.

## Syntax Quick Reference

### Parameter Store

```
@WIZSET(KEY=value)        -- Set/overwrite key
@WIZGET(KEY)              -- Get value (empty string if missing)
@WIZINIT(KEY=value)       -- Set only if key does not exist (default)
@WIZRPL(KEY=value)        -- APPEND value to existing (creates if missing)
@WIZAPPEND(KEY=value)     -- Same as WIZRPL
@WIZCLR(KEY)              -- Delete key
@WIZCLRALL(PREFIX)        -- Delete all keys starting with PREFIX
@WIZEXIST(KEY)            -- Returns "1" if key exists, "0" if not
@WIZEXISTALL(PREFIX)      -- Returns "1" if any key starts with PREFIX
@WIZISBLANK(KEY)          -- Returns "1" if missing or empty, "0" otherwise
@WIZRESET                 -- Clear entire parameter store (preprocessor, no parens)
```

### Conditionals

```
@FMT(CMP, left_op_right, 'true_branch', 'false_branch')
```

Operators: `=`, `<>`, `>`, `<`, `>=`, `<=`

Comparison is **case-insensitive string comparison**. Both branches are optional --
omit to do nothing. Remember: use `®` not `@` inside branch strings.

### String Operations

```
@FMT(LEN,string)              -- String length
@FMT(POS,needle,haystack)     -- Find position (1-based, 0 = not found)
@FMT(POSCS,needle,haystack)   -- Case-sensitive find
@FMT(S1:5,string)             -- Substring from pos 1, length 5 (1-based)
@FMT(UPPER,string)            -- Uppercase
@FMT(LOWER,string)            -- Lowercase
@FMT(TRIM,string)             -- Trim whitespace
@FMT(TRIMZERO,string)         -- Trim leading zeros
@FMT(ORD,string)              -- ASCII code of first character
@FMT(CHR,code)                -- Character from ASCII code
@FMT(CHR,26)                  -- EXIT script (success)
@FMT(CHR,27)                  -- ABORT script (error)
```

### Number Formatting

```
@FMT(#:2,123.456)             -- "123.45" (2 decimal places)
@FMT(!0:6,42)                 -- "000042" (zero-pad to width 6)
@FMT(TRUNC,123.99)            -- "123" (integer part only)
```

### Encoding

```
@FMT(2HTML,string)     @FMT(HTML2,string)     -- HTML encode/decode
@FMT(2URL,string)      @FMT(URL2,string)      -- URL encode/decode
@FMT(2JS,string)       @FMT(JS2,string)       -- JavaScript encode/decode
```

### Preprocessor Variables (no parentheses)

| Variable | Value |
|----------|-------|
| `@STORE` | Store number |
| `@TER` | Terminal number |
| `@USER` | Operator/user ID |
| `@USERNAME` | Full username |
| `@TIME` | Current time (hhmmss) |
| `@NOW` | Current date/time |
| `@RUN` | Run directory path |
| `@OFFICE` | Office directory path |
| `@CNC` | Connection number |
| `@PROFILE` | User profile |
| `@DID` | Device ID |
| `@Dxxx` | Inline date tokens (e.g. `@Dyyyy`, `@Dmm`, `@Ddd`) |

### Database Access

`@dbHot(mode, ...)` is the primary data access command:

```
@dbHot(INI,file,section,key)          -- Read INI config value
@dbHot(SAL_HDR,field,format)          -- Sale header field
@dbHot(SAL_TTL_FIND,code,field,fmt)   -- Sale total by code
@dbHot(HOT_MEM,fieldname)             -- In-memory transaction field
@dbHot(HOT_IDX,field)                 -- Current item field
@dbHot(HOT_WIZ,GET,key)              -- Read from wizard store
@dbHot(HOT_WIZ,RPLS,key,value)       -- Write to wizard store
@dbHot(POS_TAB,field)                 -- POS item table field
@dbHot(CONFIG,key)                    -- System config value
@dbHot(FINDFIRST,pattern)             -- Check if file exists
@dbHot(DEVICE)                        -- Terminal device type
@dbHot(PERIPHERAL,type,sub,key)       -- Peripheral config
```

Other database commands:

```
@dbSelect(SELECT ...)                 -- Execute query, return single value
@dbExec(SQL statement)                -- Execute SQL (no return value)
@DBVIEW(name,SELECT ...)              -- Create named in-memory view
@DBROW(viewname,column)               -- Read column from named view
@dbFld(table,,exclude)                -- Field list (exclude semicolon-delimited)
@dbGen(field,increment)               -- Generate next sequence number
```

### Execution

```
@EXEC(FCT=code)           -- Execute POS function by code number
@EXEC(MSG=text)            -- Display message (! = error, ~ = info, = = status)
@EXEC(SQI=filename)        -- Execute another SQI script
@EXEC(CGI=name)            -- Execute CGI template
@EXEC(XCH=command)         -- Exchange/communication command
@EXEC(EXE='program')       -- Run external program
@IMPORT_SIL(filename)      -- Include another .sqi file inline
```

### Utility

```
@TOOLS(PARAM,,n,string)               -- Extract nth param from CSV string
@TOOLS(ENCODE,string)                 -- URL/SQL encode
@TOOLS(MESSAGEDLG,'text',,buttons)    -- Show dialog, return button result
@READKEY(code)                        -- Check license key ("1" if enabled)
@POST(FCT=code,CN=@CNC)              -- Post command to another terminal
@FMT(COMA,key,delim,,string)          -- Extract named value from delimited string
```

### Report Accumulators

```
@RPTCLR(varname)           -- Reset accumulator
@RPTSET(varname,value)     -- Set accumulator
@RPTTTL(varname,#:2)       -- Get accumulated total, formatted
```

### Wizard UI

```
@WIZINIT                   -- Initialize wizard form
@WIZDISPLAY                -- Show wizard (blocks until closed)
@WIZMENU(KEY=Title,Opt1=val1,Opt2=val2)  -- Selection menu, stores in KEY
@WIZXSD(schema.xsd)        -- Load XSD for form layout
```

## Common Patterns

### Set and retrieve a value
```
@WIZSET(CUSTID=12345);
SELECT * FROM CUST_TAB WHERE F01 = '@WIZGET(CUSTID)';
```

### Conditional logic with branch delimiter
```
@FMT(CMP,@WIZGET(MODE)=1,'®WIZSET(RESULT=yes)','®WIZSET(RESULT=no)');
```

### Nested conditional (abort on error)
```
@FMT(CMP,@WIZISBLANK(PAYTRACK)=1,'®EXEC(MSG=!No card data)®FMT(CHR,27)');
```

### Build a dynamic query
```
@WIZSET(FILTER=WHERE F01='@WIZGET(UPC)');
SELECT * FROM OBJ_TAB @WIZGET(FILTER);
```

### Check before acting
```
@FMT(CMP,@WIZEXIST(TRACK)=1,'®EXEC(FCT=720)');
```

### Set a default, then let the script override
```
@WIZINIT(CardType=T);
/* CardType is now T unless it was already set by a prior script */
```

### Accumulate values (careful -- WIZRPL appends!)
```
@WIZSET(LIST=);
@WIZRPL(LIST=Item1,);
@WIZRPL(LIST=Item2,);
/* LIST is now "Item1,Item2," -- not "Item2," */
```

### Early exit on condition
```
@FMT(CMP,@dbHot(HOT_MEM,TrsStep)<>2,'®FMT(CHR,26)');
/* Exit silently if no transaction in progress */
```

### Read an INI setting with fallback
```
@WIZINIT(TIMEOUT=30);
@FMT(CMP,@dbHot(INI,SYSTEM,CLIENT,Timeout)<>,'®WIZSET(TIMEOUT=®dbHot(INI,SYSTEM,CLIENT,Timeout))');
```

## Script Control Flow

- `@FMT(CHR,26)` -- Exit script with **success** (POS continues normally)
- `@FMT(CHR,27)` -- Abort script with **error** (POS cancels the operation)
- Semicolons terminate statements: `@WIZSET(X=1);`
- Comments: `/* this is a comment */`

## Full Reference

For the complete command reference and interactive examples, visit:
https://locug.github.io/sqi-docs/
